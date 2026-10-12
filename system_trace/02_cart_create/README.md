# 02_cart_create

## 目的

商品詳細ページから商品Variantと数量を送信し、カートへ追加したあとカート画面へ移動するまでの処理を理解する。

## 全体フロー

```text
商品詳細ページ
↓
POST /cart_items

product_variant_id = "8"
quantity = "3"

↓
RailsがHTTP Requestを受信
↓
routes
↓
CartItemsController#create
↓
params[:product_variant_id] = "8"
params[:quantity] = "3"
↓
ProductVariant取得処理
対象Variantが存在し、
紐づくProductが公開・非削除、
Variant自身も非削除であることを確認
↓
ActiveRecord
SQLを生成
↓
PostgreSQL
ProductVariantとProductをJOINして検索
↓
ActiveRecord
検索結果をProductVariantオブジェクトへ変換
↓
Controller
variantへ代入
↓
quantityをIntegerへ変換
0以下・不正な値なら1にする
↓
existing_qtyを取得

ログイン済み
→ CartItemから現在の数量を取得

未ログイン
→ session[:cart]から現在の数量を取得
↓
new_qty = existing_qty + qty
↓
在庫チェック
↓
在庫不足
→ redirect_back
→ alert表示
→ 処理終了

在庫あり
→ カート情報を保存

ログイン済み
→ CartItemをDBへ保存

未ログイン
→ session[:cart]へ保存
↓
redirect_to cart_path
↓
302 Redirect Response
↓
Browserが新しく
GET /cart
を送信
↓
CartsController#show
↓
カート画面のHTMLを生成
↓
200 OK
↓
Browserに表示
```

## Request
Browser
↓
POST /cart_items

product_variant_id = "8"
quantity = "3"
↓
Rails
## Routing
routes.rbには、

resources :cart_items, only: %i[create update destroy]

が定義されている。

POST /cart_items
↓
CartItemsController#create

へ処理が渡される。
## Controller
## Controller

`CartItemsController#create` では、カート追加に必要な処理を順番に実行する。

```text
1. ProductVariantを取得する
↓
ProductVariantとProductをJOINし、
選択されたproduct_variant_idのVariantを検索する。

このとき、
Productが公開状態であること
Productが削除状態ではないこと
ProductVariantが削除状態ではないこと
を確認する。

↓
2. quantityを整数へ変換する
↓
params[:quantity]をIntegerへ変換する。
数値に変換できない場合は1、
0以下の場合も1にする。

↓
3. existing_qtyを取得する
↓
ログイン済みの場合
→ ActiveRecordを使ってcart_itemsテーブルから
   現在のquantityを取得する。

未ログインの場合
→ session[:cart]から現在のquantityを取得する。

↓
4. new_qtyを計算する
↓
new_qty = existing_qty + qty

↓
5. 在庫チェックを行う
↓
在庫が0以下の場合は処理を中断する。

new_qtyが在庫数を超える場合も
カートへ追加せず処理を中断する。

↓
6. カート情報を保存する
↓
ログイン済みの場合
→ CartItemへnew_qtyを保存する。

未ログインの場合
→ session[:cart]へnew_qtyを保存する。

↓
7. カート画面へRedirectする
↓
redirect_to cart_path

BrowserがRedirect Responseを受け取り、
新しくGET /cartを送信する。
```
## ProductVariant取得

Controllerに届いた `params[:product_variant_id]` を使って、
対象のProductVariantを取得する。

ProductVariantとProductをJOINし、

- Productが公開状態である
- Productが非削除状態である
- ProductVariantが非削除状態である
- ProductVariantのIDが指定されたIDと一致する

ことを確認する。

ActiveRecordがSQLを生成し、
PostgreSQLがProductVariantとProductを検索する。

条件を満たすデータがあれば、
ProductVariantオブジェクトとして `variant` に代入される。
## Quantity処理
params[:quantity] = "3"
↓
Integerで3へ変換

変換できない
→ 1

0以下
→ 1
## Existing Quantity

ログインユーザーと未ログインユーザーで、現在のカート数量の取得方法が変わる。

### ログインユーザーの場合

ActiveRecordを使って `cart_items` テーブルから、

- 現在のユーザー
- 選択されたProductVariant

に一致するCartItemを検索し、`quantity` を取得する。

取得した現在の数量を `existing_qty` とする。

```text
既存数量
existing_qty = 2

今回追加する数量
qty = 3
```

その後、

```ruby
new_qty = existing_qty + qty
```

として、追加後の合計数量を計算する。

この場合、

```text
new_qty = 2 + 3
        = 5
```

となる。

### 未ログインユーザーの場合

`cart_items` テーブルは検索せず、`session[:cart]` から対象Variantの現在数量を取得し、`existing_qty` とする。

その後はログインユーザーと同様に、

```ruby
new_qty = existing_qty + qty
```

で追加後の合計数量を計算する。
## 在庫チェック

ProductVariantの在庫数 `stock` と、追加後の合計数量 `new_qty` を比較する。

### 在庫がない場合

```ruby
variant.stock <= 0
```

の場合は在庫切れとして、カートへ追加せず処理を終了する。

### 追加後の数量が在庫を超える場合

```ruby
new_qty > variant.stock
```

の場合もカートへ追加せず処理を終了する。

どちらにも該当しなければ、在庫数の範囲内なのでカート保存処理へ進む。

```text
stock = 5
existing_qty = 2
qty = 3
new_qty = 5
→ 保存可能

stock = 5
existing_qty = 2
qty = 4
new_qty = 6
→ 在庫超過のため保存しない
```
## ログイン済みカート

ログイン済みユーザーの場合は、カート情報を `cart_items` テーブルへ保存する。

まず `find_or_initialize_by` を使って、

- 現在のユーザー
- 選択されたProductVariant

に一致するCartItemを探す。

```text
既存のCartItemがある
→ そのCartItemを取得する

既存のCartItemがない
→ 新しいCartItemオブジェクトを作る
  ※この時点ではまだDBには保存されていない
```

その後、

```ruby
item.quantity = new_qty
```

で追加後の合計数量を設定する。

最後に、

```ruby
item.save!
```

を実行してデータベースへ保存する。

既存のCartItemならUPDATE、
新規のCartItemならINSERTが実行される。
## 未ログインSessionカート
## 未ログインSessionカート

未ログインユーザーは `cart_items` テーブルを使わず、`session[:cart]` にカート情報を保持する。

```ruby
session[:cart] ||= {}
session[:cart][variant.id.to_s] = new_qty
```

`session[:cart]` がまだ存在しない場合は、空のHashを作る。

```text
{}
```

その後、ProductVariantのIDをキー、カート内の数量を値として保存する。

例：

```text
{
  "8" => 3,
  "12" => 2
}
```

これは、

```text
Variant ID 8  → 3個
Variant ID 12 → 2個
```

という意味になる。

すでに同じVariantがカートに入っている場合は、先にSessionから現在数量を `existing_qty` として取得し、

new_qty = existing_qty + qty

を計算したあと、その値でSessionを更新する。

未ログインユーザーではCartItemをDBへ保存せず、Sessionを使って複数のHTTP Request間でカート状態を保持する。
## Redirect後の画面遷移

カートへの保存が完了すると、Controllerでは次を実行する。

```ruby id="r7m6i2"
redirect_to cart_path, notice: "カートに追加しました"
```

Railsはその場でカート画面を表示するのではなく、BrowserへRedirect Responseを返す。

```text id="prx9st"
POST /cart_items
↓
CartItemsController#create
↓
カート保存
↓
302 Redirect Response
Location: /cart
↓
Browser
```

BrowserはRedirect Responseを受け取ると、新しいHTTP Requestとして、

```text id="l6f39p"
GET /cart
```

を送信する。

その後、

```text id="q5mzbe"
GET /cart
↓
CartsController#show
↓
カート画面を生成
↓
200 OK
↓
Browserに表示
```

となる。

つまり `POST /cart_items` と `GET /cart` は別々のHTTP Requestである。
## 異常系・失敗時の処理
① ProductVariantが存在しない
→ findでActiveRecord::RecordNotFound
→ rescue
→ 商品一覧などへredirect
→ 「商品が見つかりません」

② 在庫が0以下
→ Controllerのif条件で検知
→ redirect_back
→ 「在庫がありません」
→ returnで処理終了

③ new_qtyがstockを超える
→ Controllerのif条件で検知
→ redirect_back
→ 「在庫数を超えています」
→ returnで処理終了
## 実際のRails Logで確認できたこと
POST /cart_items
↓
CartItemsController#create
↓
ProductVariant検索SQL
↓
302 Redirect
↓
GET /cart
↓
CartsController#show
↓
ProductVariant / Product検索
↓
View生成
↓
200 OK