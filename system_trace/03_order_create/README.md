# 03_order_create

## 目的

カート画面の「決済へ進む」ボタンを押してから、カート情報をもとに注文データ（Order・OrderItem）を作成し、データベースへ保存するまでの処理を理解する。

この段階では注文ステータスは `pending`（支払い待ち）であり、決済は完了していない。

## 注文作成までの全体フロー

**① BrowserからRequestを送信**

カート画面で「決済へ進む」を押す。

`POST /checkout`

↓

**② Routing**

RailsがRequestを受け取り、routes.rbを使って `CheckoutsController#create` を実行する。

↓

**③ カート情報を取得**

`build_cart_rows!` を実行する。

- ログイン済み：CartItemからカート情報を取得
- 未ログイン：Sessionからカート情報を取得

↓

**④ ProductVariant・Productの情報を取得**

ControllerからActiveRecordを使ってDBを検索する。

- ActiveRecordがSQLを発行
- PostgreSQLが商品情報・在庫数などを検索
- ActiveRecordが検索結果をRubyのオブジェクトとして扱えるようにする

↓

**⑤ 商品状態・数量・在庫を確認**

- 商品が公開・非削除状態か
- 注文数量が1以上か
- 注文数量が在庫数以下か

異常があれば例外を発生させ、rescueで捕捉してカート画面へ戻す。

↓

**⑥ rowsを作成**

商品ごとに以下の情報をまとめる。

- ProductVariant
- 注文数量 `qty`
- 商品単価 `unit_price_cents`
- 小計 `subtotal_cents`

↓

**⑦ 注文の合計金額を計算**

各商品の小計を合算して `total_cents` を作る。

↓

**⑧ Orderを作成**

`Order.create!`

注文全体の情報をPostgreSQLのordersテーブルに保存する。

保存内容は、注文ステータス、合計金額、購入者情報、配送先など。

↓

**⑨ OrderItemを作成**

`OrderItem.create!`

rows内の商品ごとに注文明細を作成し、PostgreSQLのorder_itemsテーブルへ保存する。

各OrderItemは `order_id` によって、⑧で作成したOrderに紐づく。

↓

**⑩ 注文データ作成完了**

Orderのステータスは `pending`（支払い待ち）。

この後、Stripe Checkout Sessionを作成し、Stripeの決済画面へRedirectする。この部分は04_stripe_checkoutで学習する。
## Browserから送信するRequestとRouting

カート画面で「決済へ進む」ボタンを押すと、
BrowserからRailsへ以下のHTTP Requestが送信される。

HTTPメソッド：POST
URL：http://localhost:3000/checkout

Railsはroutes.rbの以下の定義を参照する。

resource :checkout, only: [:create]

POST /checkout
↓
CheckoutsController#create

これにより、注文作成処理が開始される。
## ログイン状態に応じたカート情報の取得

### ログイン済みユーザー

`current_customer.cart_items` を使って、現在ログインしているユーザーのカート情報を取得する。

`includes(product_variant: :product)` によって、CartItemに紐づくProductVariantとProductも事前に読み込み、関連データを取得する際のSQL発行回数を減らす。

### 未ログインユーザー

`session[:cart]` からカート情報を取得する。

未ログインユーザーのカート情報はCartItemテーブルではなくSessionで管理しているため、Session内のVariant IDと数量を使用する。

どちらの場合も、取得したカート情報をもとに商品ごとの数量を整理し、後続の在庫確認・注文データ作成へ進む。
## 商品状態・注文数量・在庫の再確認

`build_cart_rows!` で注文作成前に商品状態・数量・在庫を確認する。

**① RecordNotFound**

- ProductVariantが削除済み
- Productが削除済み
- Productが非公開

上記のいずれかに該当する場合、`ActiveRecord::RecordNotFound` を発生させる。

**② ArgumentError**

注文数量 `qty` が1未満の場合に発生する。

**③ stock shortage**

注文数量 `qty` がProductVariantの在庫数 `stock` を超えた場合に発生する。

**④ 再確認する理由**

カート追加時から注文作成時までに、他の注文などによって在庫が変動する可能性があるため、注文作成直前に再確認する。

異常があれば例外を発生させ、Controllerの `rescue` で捕捉してカート画面へ戻す。
## rowsの作成と合計金額の計算

商品状態・数量・在庫のチェックを通過した後、
各商品の情報をHashとしてまとめ、Array形式のrowsを作成する。

各Hashには以下の情報を格納する。

- variant：ProductVariantオブジェクト
- qty：注文数量
- unit_price_cents：商品単価
- subtotal_cents：単価 × 数量

その後、rows内の各subtotal_centsを合計して、
注文全体のtotal_centsを算出する。
## Orderへの注文全体の保存

`Order.create!` を実行し、ActiveRecordを通してPostgreSQLの `orders` テーブルへ注文全体の情報を保存する。

主な保存内容は以下のとおり。

- `customer`：注文に紐づくCustomer
- `status`：注文ステータス
- `total_cents`：注文全体の合計金額
- `customer_name`：購入者氏名
- `customer_email`：メールアドレス
- `postal_code` など：配送先住所

`status: :pending` は支払い待ちを意味し、Order作成時点ではStripe決済は完了していない。

### ログイン済みユーザー

`current_customer` を使って現在のCustomerにOrderを紐付け、登録済みの氏名・住所などをOrderにも保存する。

### 未ログインユーザー

`customer` は `nil` とし、`SecureRandom.alphanumeric(12)` でゲスト注文番号を生成する。

購入者の氏名・メールアドレス・配送先住所は、ブラウザから送信された `params` を使用してOrderへ保存する。

`Order.create!` が成功すると、新しいOrderレコードが作成され、注文IDが生成される。
## OrderItemへの商品明細の保存

`rows.each` によって商品ごとのデータを取り出し、`OrderItem.create!` を実行して `order_items` テーブルへ保存する。

商品が3種類ある場合、OrderItemは3件作成される。

`order: order` は、事前に `Order.create!` で作成したOrderと各OrderItemを紐付けるための指定である。

これにより、`orders.id` と `order_items.order_id` が対応する。

OrderItemには以下の情報を保存する。

- `product_variant`：注文した商品のVariant
- `quantity`：注文数量
- `unit_price_cents`：注文時点の単価
- `subtotal_cents`：数量と単価から計算した小計

Productの現在価格は注文後に変更される可能性があるため、OrderItemへ購入時点の単価を保存し、過去の注文履歴に影響しないようにしている。
## ゲスト注文とSessionの紐付け

未ログインユーザーが注文を作成した場合、後からその注文を参照できるように、作成したOrder IDをSessionに保存する。

`unless customer_signed_in?` によって、未ログインユーザーの場合のみ処理を実行する。

`Array(session[:guest_order_ids])` で既存の注文ID情報を配列形式に整える。

`<< order.id` で今回作成したOrder IDを配列に追加する。

`uniq!` によって配列内の重複したOrder IDを削除する。

その後、注文詳細の表示時に、このSessionに対象のOrder IDが含まれているか確認することで、ゲストが参照できる注文を制限している。

## 在庫不足・数量不正などの異常系

`CheckoutsController#create` では、異常が発生した場合に例外を捕捉し、カート画面へ戻す処理を実行する。

- `ActiveRecord::RecordInvalid`：入力内容や保存データが不正な場合
- `ActiveRecord::RecordNotFound`：対象商品が存在しない、削除済み、非公開などの場合
- `ArgumentError`：注文数量が1未満の場合
- `stock shortage`：注文数量が在庫を超える場合

`build_cart_rows!` で異常が発生した場合は、後続のOrder・OrderItem作成処理を実行しない。

`rescue` が例外を捕捉すると、Railsはカート画面への302 Redirect ResponseをBrowserに返す。

Browserは新しく `GET /cart` を送信し、`CartsController#show` によってカート画面とエラーメッセージが表示される。
## Rails Logによる動作確認

カート画面から `POST /checkout` を送信し、Rails Logで注文作成処理を確認した。

### 確認した処理

1. `POST /checkout` により `CheckoutsController#create` が実行された。
2. `build_cart_rows!` でProductVariant ID 31とProduct ID 20を取得した。
3. `Order Create` によって、Order ID 73が作成された。
4. Orderの合計金額は12,800円、ステータスは `pending`（支払い待ち）だった。
5. `OrderItem Create` によって、Variant ID 31・数量1・単価12,800円の注文明細が作成された。
6. `order_items.order_id = 73` によってOrderとOrderItemが紐付いていることを確認した。
7. Stripe Checkout Session作成後、`Order Update` でStripe Session IDを保存した。
8. 最後にRailsが302 Redirect Responseを返し、Stripeの決済画面へ誘導した。

### 確認結果

カート情報の取得からOrder・OrderItemのDB保存、StripeへのRedirectまでの一連の処理を確認できた。

ただし、Stripeでの支払い完了および在庫減算は、このログでは確認していない。