# 04 Stripe Checkout：決済画面への遷移と注文画面への戻り

状態：学習中。以下は本文作成用の見出しであり、完了記録ではない。
学習済みの項目から自分の言葉で記入し、未確認の項目は「未確認」とする。

## この章の目的

## 注文作成後から決済画面へ移動し、注文画面へ戻るまでの全体フロー

## Stripeへ送る商品情報とrowsからの取得方法

## Orderの合計金額とStripeの請求情報を同じrowsから作る理由

## Checkout Sessionの作成とRailsのSessionとの違い

## session_obj.idとsession_obj.urlの役割

## OrderとCheckout Sessionの紐付け

## 302 Responseを受け取ったBrowserの動作

## allow_other_host: trueを指定する理由

## success_urlとcancel_urlの役割

## 注文詳細画面でのpaidパラメータと注文状態の確認

## カートを削除する条件

## 支払い確認待ちと判定する条件

## URLのpaid=1だけで支払い完了と判断してはいけない理由

## CheckoutとWebhookの役割の違い

## 正常時・異常時・支払い確認待ちの動作

## 実際のRails Logによる確認

## 理解できたこと・まだ確認が必要なこと
