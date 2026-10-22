# 04 Stripe Checkout：決済画面への遷移と注文画面への戻り

状態：学習中。以下は本文作成用の見出しであり、完了記録ではない。
学習済みの項目から自分の言葉で記入し、未確認の項目は「未確認」とする。

## この章の目的

決済画面へ移動後、決済を完了して、Railsへ戻ってくるまでの動きを把握する

## 注文作成後から決済画面へ移動し、注文画面へ戻るまでの全体フロー

```text
Order・OrderItemの作成完了
注文状態はpending（支払い待ち）
↓
CheckoutsController#createで、
RailsがStripe APIへCheckout Sessionの作成を依頼する。
rowsをもとに商品名・単価・数量を送り、
success_url・cancel_url・metadata.order_idも指定する。
↓
StripeがCheckout Sessionを作成し、情報をRailsへ返す。
Railsは返された情報をsession_objに保持する。
↓
Railsがsession_obj.idを
Orderのstripe_checkout_session_idに保存する。
↓
redirect_toを実行する。
Railsが302 Responseを返し、
Locationヘッダーでsession_obj.urlをBrowserへ伝える。
↓
BrowserがStripeの決済画面URLへGET Requestを送る。
↓
Stripeが決済画面を返し、Browserが表示する。
↓
利用者が決済操作を行い、Checkoutが成功する。
↓
Browserがsuccess_urlへGET Requestを送る。
例：GET /orders/73?paid=1
↓
RailsのRoutingでOrdersController#showへ振り分ける。
↓
閲覧権限を確認し、
ActiveRecordを通じて取得した既存の注文を@orderに保持する。
↓
paidパラメータの有無と、取得した注文状態を確認する。

【paidパラメータがあり、注文状態がpaidの場合】
・カートを削除する。
・@waiting_payment_confirmationをfalseにする。

【paidパラメータがあり、注文状態がpendingの場合】
・カートを残す。
・@waiting_payment_confirmationをtrueにする。
・画面に支払い確認中の案内を含める。
↓
どちらの場合もWebhookを待って処理を停止せず、
注文詳細画面のHTMLを200 Responseとして返す。
↓
Browserが注文詳細画面を表示する。
```

### Webhookによる注文更新と画面表示の関係

StripeからのWebhookは、Browserが注文詳細画面へ戻るRequestとは別にRailsへ届く。

Webhook側の処理によって注文状態が更新されるが、
Browserが戻る時点で、その更新が完了しているとは限らない。

確認待ちの画面を表示した後にDBが更新された場合は、
Browserでリロードして注文を再取得し、更新後の状態で画面を表示する。

URLのpaid=1だけでは支払い完了を確定しない。
また、200 Responseは注文詳細画面へのRequestを正常に処理したことを示し、
決済成功そのものを示すものではない。

### 現在の実装で見つかった表示の問題

注文詳細画面のViewには、paidパラメータがあるだけで
「支払い完了」と表示する箇所もある。

そのため、注文状態がpendingでも「支払い確認中」と
「支払い完了」が同時に表示される。
これは表示の不整合であり、注文状態がpaidになったことを意味しない。

## Stripeへ送る商品情報とrowsからの取得方法
CheckoutsController#createからbuild_cart_rows!を呼び出し、
商品ごとのProductVariantオブジェクト・単価・注文数量・小計を
Hashにまとめ、それらをArrayにしたrowsを作る。

単価はProductのprice_centsから取得する。
数量は、ログイン済みならcart_itemsテーブルのquantity、
未ログインならsession[:cart]から取得する。

rows.mapで各HashをStripe用のHashへ変換する。
商品名はr[:variant].product.name、
単価はr[:unit_price_cents]、数量はr[:qty]から取得する。
変換結果のArrayをline_itemsとしてStripeへ送信する。
## Orderの合計金額とStripeの請求情報を同じrowsから作る理由
Orderの合計金額とStripeへ送る単価・数量を同じrowsから作ることで、Rails側の注文金額とStripe側の請求金額の不一致を防ぐ。別々に価格を取得した場合、その間にProductのprice_centsが更新されると、金額がずれる可能性がある。
## Checkout Sessionの作成とRailsのSessionとの違い
| 対象 | 役割 |
|---|---|
| StripeのCheckout Session | Stripe側で購入手続きを管理し、決済画面へ進めるようにする |
| Railsの `session[:cart]` | 未ログインユーザーのカート情報をRequest間で保持する |
## session_obj.idとsession_obj.urlの役割
session_obj.id はStripeのCheckout Sessionを識別するIDで、Orderとの紐付けに使う。session_obj.url はBrowserを誘導するStripeの決済画面URLである。
## OrderとCheckout Sessionの紐付け
Checkout Session作成時に、Rails側の注文IDを metadata.order_id としてStripeへ渡す。後のWebhookでは、このIDを使って更新対象の注文を特定する。Stripeから返されたCheckout Session IDは、Rails側のOrderの stripe_checkout_session_id に保存する。
## 302 Responseを受け取ったBrowserの動作
現在「allow_other_host」の項目にある、302から画面表示までの説明をこちらへ移してください。「allow_other_host」の項目には、外部ホストをリダイレクト先として許可する説明だけを残します。
## allow_other_host: trueを指定する理由
Railsは302 Responseを返し、LocationヘッダーでStripeの決済画面URLをBrowserへ伝える。BrowserはそのURLへGET Requestを送り、Stripeから返された決済画面を表示する。allow_other_host: true は、Railsが外部ホストのURLをリダイレクト先に指定することを許可する設定である。
## success_urlとcancel_urlの役割
success_url はCheckout成功後に注文詳細画面へ戻るためのURLである。ただし、このURLへのアクセスだけで支払い完了とは判断しない。cancel_url は利用者が決済画面からキャンセルして戻る場合のURLで、このアプリではカート画面を指定している。決済失敗時すべての遷移先ではない。
## 注文詳細画面でのpaidパラメータと注文状態の確認
paid パラメータは、Requestに含まれる値が空でないかを確認する。@order.paid? は、DBから取得した注文の状態が支払い済みかを確認する。利用者はURLに paid=1 を直接指定できるため、それだけでは支払い完了と判断せず、注文状態も確認する。
## カートを削除する条件
paidパラメータが空でなく、取得した注文状態がpaidの場合に削除する。
ログイン済みなら、そのユーザーのCartItemを削除する。
未ログインなら、session[:cart]を削除する。
## 支払い確認待ちと判定する条件
paidパラメータが空でなく、取得した注文状態がpaidではない場合、
@waiting_payment_confirmationがtrueになる。

これは画面表示のための判定であり、Webhookが届くまで
処理を停止して待機するものではない。
## URLのpaid=1だけで支払い完了と判断してはいけない理由
利用者はURLに paid=1 を直接指定できるため、支払いが成功した証拠にはならない。DBから取得した注文状態も確認する必要がある。
## CheckoutとWebhookの役割の違い

Checkoutでは、RailsがStripeへCheckout Sessionの作成を依頼し、
返されたURLへBrowserを誘導して決済画面を表示する。
Sessionを作成しただけでは、支払い完了にはならない。

Webhookでは、StripeのサーバーがRailsへイベントを通知する。
Railsは通知を検証し、対象の注文状態や在庫を更新する。

WebhookはBrowserが注文詳細画面へ戻る処理とは独立しているため、
利用者がBrowserを閉じても、通知を受信して注文更新の処理を実行できる。
## 正常時・異常時・支払い確認待ちの動作
| ケース | 表示する画面 | カート |
|---|---|---|
| ① 注文状態が `paid` | 注文73の詳細画面 | 空にする |
| ② 注文状態が `pending` | **注文73の詳細画面（支払い確認中の案内あり）** | 残す |
| ③ 決済画面からキャンセルして戻る | カート画面 | 残す |
## 実際のRails Logによる確認
注文73の stripe_checkout_session_id が更新され、COMMITされたことを確認した。その後、RailsがStripeの決済画面URLへの302 Responseを返している。このログでは、支払い完了や注文状態のpaidへの更新は確認できない。
注文詳細画面へのGET、既存注文・明細のSELECT、HTML生成、200 Responseの流れは模擬ログで確認した。決済後の注文状態更新とカート削除について、今回新たな実操作による検証は行っていない。
## 理解できたこと・まだ確認が必要なこと

### 理解できたこと
- Checkout Sessionの作成依頼と、Browserの画面遷移は別の通信である。
- URLのpaidパラメータと、保存された注文状態は別の情報である。
- WebhookはBrowserが戻る処理とは独立して届く。
- pendingでもRailsは待機せず、その時点の情報で画面を返す。

### まだ確認が必要なこと
- 全体フローを補助なしで説明できるか。
- SQLの更新項目とHTTPステータスを区別して説明できるか。
- Checkout SessionとPaymentIntentの違い。
- 決済後の処理と異常系の実ログ確認。
- Viewの「支払い確認中」と「支払い完了」が同時に出る表示の不整合。