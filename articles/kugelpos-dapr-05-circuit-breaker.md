---
title: "【KugelPOS】Dapr活用 第5回/全5回 — サーキットブレーカーで障害を前提とした設計"
emoji: "⚡"
type: "tech"
topics: ["Dapr", "マイクロサービス", "POS", "耐障害性", "サーキットブレーカー"]
published: true
---

![](/images/kugelpos-dapr-05/slide-1.png)

## はじめに

これまでの連載で、Service Invocation（泣きどころ①）、Pub/Sub（泣きどころ②）、State Store（泣きどころ③）というマイクロサービスの課題を解決する3つの仕組みを紹介してきました。しかし、これらの仕組み自体に障害が起きたらどうなるでしょうか。

キャッシュとして使っているRedisがダウンしたら？ Pub/Subのメッセージブローカーが応答しなくなったら？ マイクロサービスの世界では、依存するコンポーネントの障害は「起きるかもしれない」ではなく「いつか必ず起きる」ものとして設計する必要があります。

今回は、障害の連鎖を食い止める「サーキットブレーカー」というパターンと、KugelPOSでの実践をお話しします。

## 障害が連鎖するメカニズム

![](/images/kugelpos-dapr-05/slide-2.png)

### 1つの障害が全体を巻き込む

具体的なシナリオで考えてみましょう。

KugelPOSのCartサービスは、カートデータをRedis（State Store）にキャッシュしています。通常はRedisへの読み書きが高速に完了するため、レジの応答は快適です。

ある日、Redisに障害が発生しました。Redisは応答しませんが、接続自体はタイムアウトするまで待たされます。タイムアウトが3秒に設定されていて、リトライが3回なら、1回のカート操作で最大9秒待たされます。

レジで商品をスキャンするたびに9秒待たされる。30品目の精算に4分半。これは事実上、レジが使えない状態です。

さらに悪いことに、タイムアウト待ちのリクエストがCartサービス内に滞留し、最終的にシステム全体が連鎖的に低下していきます。

## サーキットブレーカーの考え方

### 電気のブレーカーと同じ発想

家庭の分電盤にあるブレーカーは、過電流を検知すると回路を遮断して電気機器を保護します。ソフトウェアのサーキットブレーカーも同じ発想です。

### 3つの状態

![](/images/kugelpos-dapr-05/slide-3.png)

**CLOSED（閉じている）— 通常状態**
電気が流れている正常な状態です。リクエストは通常通り対象のコンポーネントに送られます。ただし、失敗の回数はカウントされています。

**OPEN（開いている）— 遮断状態**
ブレーカーが落ちた状態です。連続的な失敗が一定回数に達すると、この状態に遷移します。対象のコンポーネントへのリクエストは即座に遮断され、フォールバック処理が実行されます。

**HALF-OPEN（半開き）— 回復試行状態**
ブレーカーが落ちてから一定時間が経過すると、試しに1つリクエストを送ってみます。成功すればCLOSEDに戻り、失敗すれば再びOPENに戻ります。

## KugelPOSでの実践

### CartサービスのRedisフォールバック

![](/images/kugelpos-dapr-05/slide-4.png)

```python
# cart/app/models/repositories/cart_repository.py
class CartRepository(AbstractRepository[CartDocument]):
    def __init__(self, db, terminal_info=None):
        super().__init__(settings.DB_COLLECTION_NAME_CACHE_CART, CartDocument, db)
        self._circuit_open = False
        self._failure_count = 0
        self._last_failure_time = 0
        self._failure_threshold = 3       # 3回連続失敗でOPEN
        self._reset_timeout = 60          # 60秒後にHALF_OPENへ

    async def cache_cart_async(self, cart: CartDocument, isNew: bool = False):
        # サーキットブレーカーがOPENなら、Redisをバイパス
        if not self._check_circuit_breaker():
            logger.warning("Circuit is open. Using database directly.")
            await self.__save_cart_to_db_async(cart)
            return

        try:
            await self.__cache_cart_async(cart, isNew)
            self._record_success()          # 成功を記録
        except Exception as e:
            self._record_failure()          # 失敗を記録
            await self.__save_cart_to_db_async(cart)  # MongoDBにフォールバック
```

通常時はRedis（インメモリキャッシュ）に保存し、Redis障害時はサーキットブレーカーがOPENになった瞬間からMongoDB（永続DB）に直接保存します。応答速度は若干低下しますが、レジは止まりません。

### 設定値 — なぜその値か

**失敗閾値: 3回**
1回の失敗はネットワークの一時的な揺らぎかもしれません。2回連続はまだ偶然の可能性があります。3回連続となると、「コンポーネントの障害」と判断するのが妥当です。一方で、閾値を大きくしすぎると障害の検知が遅れ、タイムアウト待ちでレジが遅延し続けます。

**リセットタイムアウト: 60秒**
Redisのような軽量なインメモリストアは、再起動すれば比較的短時間で復旧します。60秒は復旧を待つのに十分な時間であり、かつMongoDBフォールバックの状態が長引きすぎない適切なバランスです。

### 2つのサーキットブレーカー

![](/images/kugelpos-dapr-05/slide-5.png)

KugelPOSには実は2種類のサーキットブレーカーが存在します。

**CartRepositoryのサーキットブレーカー** — Cartサービス内でState Storeの読み書きを保護。障害時はMongoDBにフォールバック。

**DaprClientHelperのサーキットブレーカー** — 共通ライブラリとして、Daprサイドカーとの通信全般を保護。

前者は「State Storeが落ちたらデータベースで代替する」というCartサービス固有のフォールバック戦略を持ち、後者は「Daprサイドカーとの通信障害全般を検知する」という汎用的な保護を提供します。

## 「障害を前提とする」という設計思想

![](/images/kugelpos-dapr-05/slide-6.png)

### 楽観と悲観の使い分け

**悲観的に備える:** Redisは落ちる。ネットワークは切れる。Daprサイドカーは応答しなくなる。これらの障害は「いつか必ず起きる」ものとして、すべてにフォールバックとサーキットブレーカーを用意する。

**楽観的に運用する:** 通常時は高速なRedisキャッシュとPub/Subの恩恵を最大限に受ける。障害が起きたときだけ代替手段に切り替わり、復旧すれば自動的に通常運転に戻る。

### ヘルスチェックとの連携

KugelPOSの各サービスは、以下のヘルスチェックを定期的に実行しています。

| チェック対象 | 確認方法 |
|---|---|
| Daprサイドカー | ヘルスエンドポイントへの応答確認 |
| State Store | テストデータの書き込み・読み取り・削除 |
| Pub/Sub | テストメッセージの発行確認 |
| MongoDB | データベース接続の確認 |

ヘルスチェックの役割は、各コンポーネントの状態を可視化することです。サーキットブレーカーが障害を検知してフォールバックで業務を継続している間に、運用者がヘルスチェックの情報をもとに原因を特定し、復旧対応を行う。この連携によって、システム全体の可用性を維持しています。

## おわりに

サーキットブレーカーは「障害を防ぐ」仕組みではありません。障害が起きたときに、その影響が広がることを防ぐ仕組みです。

マイクロサービスは多くのコンポーネントで構成されるため、どこかで障害が起きる確率は高くなります。その現実を受け入れた上で、「障害が起きても業務を継続する」ための設計が重要です。サーキットブレーカーによるフォールバックが業務を継続し、ヘルスチェックが障害の状況を可視化する。これらが連携して、KugelPOSの可用性を支えています。

本連載を通じて、マイクロサービスが抱える課題とDaprによる解決策をお話ししてきました。Service Invocationによるサービス間通信の簡素化、Pub/Subによる非同期処理、State Storeによるインフラ非依存の状態管理、そしてサーキットブレーカーによる耐障害性。これらはいずれも、アプリケーションのコードからインフラの複雑さを切り離し、開発者がビジネスロジックに集中するための仕組みです。

![](/images/kugelpos-dapr-05/slide-7.png)

**KugelPOS Backend Services**
GitHub: https://github.com/kugel-masa/kugelpos-backend
ライセンス: Apache 2.0
