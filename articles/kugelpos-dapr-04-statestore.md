---
title: "【KugelPOS】Dapr活用 第4回/全5回 — State Storeでインフラ非依存の状態管理とキャッシュ"
emoji: "💾"
type: "tech"
topics: ["Dapr", "マイクロサービス", "POS", "Redis", "キャッシュ"]
published: true
---

![](/images/kugelpos-dapr-04/slide-1.png)

## はじめに

前回はPub/Subによる非同期メッセージングを取り上げ、**泣きどころ②「同期の呪縛」**を解決しました。今回はDaprの State Store（状態ストア）を深掘りし、**泣きどころ③「インフラ依存」**を解決します。

State Storeは、データの一時保存を「インフラを意識せずに」行える仕組みです。KugelPOSではカートデータのキャッシュやメッセージの重複処理防止など、複数の用途で活用しています。それぞれの活用方法と、なぜこの仕組みが重要なのかをお話しします。

## なぜキャッシュが必要か

![](/images/kugelpos-dapr-04/slide-2.png)

### レジの操作は高頻度

スーパーマーケットのレジを想像してください。お客様が30品目の商品を持ってレジに並んでいます。店員が1品ずつバーコードをスキャンするたびに、システムは以下の処理を行います。

1. カートに商品を追加する
2. 小計・税額を再計算する
3. 更新されたカート情報を保存する
4. レジ画面に結果を表示する

30品目なら、この処理が30回繰り返されます。スキャンのたびにカートの読み出しと書き込みが発生し、その応答時間が一定でないと、レジ操作のリズムが崩れます。繁忙時間帯のレジでは、このもたつきがストレスになります。

### MongoDBへの高頻度アクセスが問題になる

KugelPOSではMongoDBを全7サービスで共有する永続層として使っています。ここに、カートの読み書きという高頻度アクセスを直接集中させると、DB全体のスループットが圧迫され、CartサービスだけでなくReport・Journal・Stockといった他のサービスの応答にも影響が出てしまいます。

ここでキャッシュの出番です。頻繁に更新されるカートデータをRedis（インメモリのキャッシュ）に保存し、精算が確定したときだけMongoDBに書き込む2層構造にすることで、MongoDBへの負荷を大幅に削減できます。結果としてレジの応答速度も改善し、他サービスへの影響も排除できます。

## DaprのState Storeの仕組み

![](/images/kugelpos-dapr-04/slide-3.png)

### 保存先を知らなくていい

Dapr State Storeの最大の特徴は、アプリケーションが保存先の具体的な技術を知らなくてよいことです。

アプリケーションがDaprに伝えるのは以下の3つだけです。

- **どのストアに** — ストアの名前（例: cartstore）
- **どのキーで** — データを識別するキー（例: カートID）
- **何を保存するか** — 保存するデータの中身

保存先がRedisなのかAWSのDynamoDBなのかAzureのCosmos DBなのか、アプリケーションは関知しません。保存先の切り替えは、Daprの設定ファイルを変更するだけで完了します。

## KugelPOSでの実践

### 3つのState Store、3つの役割

![](/images/kugelpos-dapr-04/slide-4.png)

KugelPOSでは用途に応じて3つのState Storeを使い分けています。

**1. cartstore — カートデータのキャッシュ**

| 項目 | 内容 |
|---|---|
| 用途 | レジのカート情報の高速な読み書き |
| データの寿命 | 10時間（営業時間をカバー） |
| 利用サービス | Cart |

**2. statestore — メッセージの重複処理防止**

| 項目 | 内容 |
|---|---|
| 用途 | Pub/Subメッセージの冪等性保証 |
| データの寿命 | 1時間 |
| 利用サービス | Report, Journal, Stock |

**3. terminalstore — ターミナル状態の管理**

| 項目 | 内容 |
|---|---|
| 用途 | レジ端末の状態管理 |
| データの寿命 | 1時間 |
| 利用サービス | Terminal |

### State Storeの読み書き — CartRepositoryの実装

```python
# cart/app/models/repositories/cart_repository.py
base_url_cartstore = f"{settings.BASE_URL_DAPR}/state/cartstore"

async def __cache_cart_async(self, cart: CartDocument, isNew: bool = False):
    """カートデータをState Storeに保存"""
    cart_data = cart.model_dump()
    state_post_data = [{"key": cart.cart_id, "value": cart_data}]

    session = await get_dapr_statestore_session()
    async with session.post(self.base_url_cartstore, json=state_post_data) as response:
        if response.status != 204:
            raise UpdateNotWorkException("Failed to cache cart", ...)
```

| 操作 | HTTP メソッド | URL |
|---|---|---|
| 保存 | POST | `http://localhost:3500/v1.0/state/cartstore` |
| 取得 | GET | `http://localhost:3500/v1.0/state/cartstore/{cart_id}` |
| 削除 | DELETE | `http://localhost:3500/v1.0/state/cartstore/{cart_id}` |

このコードのどこにも「Redis」という文字は出てきません。保存先がRedisからDynamoDBやCosmos DBに変わっても、このコードは一切変更不要です。

### なぜ3つに分けるのか

![](/images/kugelpos-dapr-04/slide-5.png)

**データの寿命が異なる。** カートデータは10時間保持する必要がありますが、冪等性のキーは1時間で十分です。

**障害の影響を限定する。** もしcartstoreに問題が発生しても、statestoreのメッセージ重複防止機能には影響しません。

**運用の見通しが良い。** どのストアにどの程度のデータが蓄積されているか、どのストアに負荷が集中しているかが明確になります。

## State Storeがもたらす効果

![](/images/kugelpos-dapr-04/slide-6.png)

**レジの応答安定化。** カートデータのキャッシュにより、商品スキャンごとの応答時間のばらつきを抑えています。体感できるほどの劇的な高速化というよりも、MongoDB への負荷集中を避けることで応答が安定し、スキャンリズムが崩れにくくなることが主な効果です。

**データの一貫性。** Pub/Subの冪等性保証により、ネットワーク障害時の再配信でもデータが二重処理されません。

**インフラの柔軟性。** 保存先の技術をアプリケーションコードから切り離しているため、将来のインフラ変更に対してコードの修正が不要です。

**運用の安定性。** 用途別にState Storeを分離することで、障害の影響範囲を限定し、運用の見通しを良くしています。

## おわりに

State Storeは一見地味な機能ですが、KugelPOSにおいては複数の重要な役割を担っています。レジの応答速度を支えるキャッシュ、データの二重処理を防ぐ冪等性保証、端末状態の管理。これらすべてが、インフラの具体的な技術を知らなくてよいという共通の仕組みの上に成り立っています。

次回は、サーキットブレーカーを取り上げます。キャッシュやPub/Subの仕組みがあっても、それら自体に障害が起きたらどうするのか。「障害を前提とした設計」がKugelPOSでどのように実現されているかをお話しします。

**KugelPOS Backend Services**
GitHub: https://github.com/kugel-masa/kugelpos-backend
ライセンス: Apache 2.0
