# 01_product_show

## 目的

商品詳細ページへアクセスしたとき、商品情報をPostgreSQLから取得し、HTMLとしてBrowserへ表示するまでの処理を理解する。

## 全体フロー

```text
http://localhost:3000/products/20
へBrowserからアクセス
↓
HTTP GET Request
GET /products/20
↓
routes
GET /products/:id に一致
↓
ProductsController#show
params[:id] = "20"
↓
Product.find_by!(...)
↓
ActiveRecord
Rubyの検索処理からSQLを生成する
↓
SQL
PostgreSQLへ検索条件を伝える
↓
PostgreSQL
productsテーブルから条件に一致する商品を検索する
↓
ActiveRecord
検索結果をProductオブジェクトへ変換する
↓
@product
取得したProductオブジェクトを保持する
↓
@product.product_variants
ProductとProductVariantのAssociationを利用する
↓
@variants
対象商品のProductVariantを取得するRelationを保持する
↓
show.html.erb
@productと@variantsを使ってHTMLを生成する
↓
HTTP Response
生成したHTMLをBrowserへ返す
↓
Browser
HTMLを解析して商品詳細画面を表示する
↓
200 OK
```

## Request

Browserから、

```text
http://localhost:3000/products/20
```

へアクセスする。

BrowserからRailsへ、

```text
GET /products/20
```

というHTTP Requestが送られる。

この時点ではまだ `params[:id]` ではなく、URLの `/products/20` として送られている。

## Routing

`routes.rb` には、

```ruby
resources :products, only: %i[index show]
```

が定義されている。

これによって、

```text
GET /products/:id
↓
ProductsController#show
```

というRouteが作られる。

今回の、

```text
GET /products/20
```

では、

```text
:id = "20"
```

となり、Controllerから、

```ruby
params[:id]
```

として取得できる。

## Controller

実際のControllerは、

```ruby
def show
  @product = Product.find_by!(
    id: params[:id],
    published: true,
    deleted: false
  )

  @variants = @product.product_variants
                      .where(deleted: false)
                      .order(:color, :size)
end
```

となっている。

まず、

```ruby
Product.find_by!(
  id: params[:id],
  published: true,
  deleted: false
)
```

によって、

```text
id = 20
AND published = true
AND deleted = false
```

を満たすProductを取得する。

取得されたProductオブジェクトを、

```ruby
@product
```

へ代入する。

次に、

```ruby
@product.product_variants
```

によって、そのProductに紐づくProductVariantを対象にする。

さらに、

```ruby
.where(deleted: false)
```

で論理削除されていないVariantだけに絞り、

```ruby
.order(:color, :size)
```

でcolor、sizeの順に並べる。

そのRelationを、

```ruby
@variants
```

へ代入する。
## Model / Association

ProductModelとProductVariantModelはAssociationで関連付いている。

ProductModel

has_many :product_variants

ProductVariantModel

belongs_to :product

関係は、

Product 1件
↓
ProductVariant 複数件

products.id
= Primary Key

product_variants.product_id
= Foreign Key

product_variants.product_id が products.id を参照することで、
どのProductVariantがどのProductに属しているかを判断できる。

そのため、

@product.product_variants

と書くことで、現在のProductに紐づくProductVariantを取得できる。
## ActiveRecord / SQL

ActiveRecordはRailsのRubyコードを、
PostgreSQLが実行できるSQLへ変換する。

Controllerでは、

Product.find_by!(
  id: params[:id],
  published: true,
  deleted: false
)

を実行している。

ActiveRecordは概念的に、

SELECT *
FROM products
WHERE id = 20
  AND published = true
  AND deleted = false
LIMIT 1;

のようなSQLを生成する。

役割は、

Product.find_by!(...)
→ Rails / Ruby側の検索処理

ActiveRecord
→ Rubyの検索処理をSQLへ変換

SQL
→ PostgreSQLへ検索条件を指示する言語
## PostgreSQL

ActiveRecordから送られたSQLを実行する。

今回の場合は、

id = 20
published = true
deleted = false

をすべて満たすproductsテーブルのRowを1件検索する。

該当するRowが見つかれば、その検索結果をActiveRecordへ返す。

その後、ActiveRecordが検索結果をProductオブジェクトへ変換する。
## View

Controllerから受け取った、

@product
@variants

をshow.html.erbで使用する。

<%= @product.name %>
<%= @product.description %>

などによって、Productオブジェクトの値をHTMLへ埋め込む。

@variantsはeachで1件ずつ取り出し、
color、size、stockなどをHTMLへ反映する。

最終的にBrowserが表示できるHTMLを生成する。
## Response

Viewで生成したHTMLをHTTP ResponseとしてBrowserへ返す。

正常に処理が完了した場合、
HTTP Status Codeとして200 OKを返す。

Browserは受け取ったHTMLを解析して商品詳細ページを表示する。
## Error Test

/products/1 へアクセスした。

routes、Controller、ActiveRecordによるSQL生成までは正常に処理される。

PostgreSQLで、

id = 1
published = true
deleted = false

をすべて満たすProductを検索したが、該当するRowが0件だった。

find_by!を使用しているため、
ActiveRecordはnilを返さず、

ActiveRecord::RecordNotFound

を発生させる。

そのためViewまでは進まず、

404 Not Found

となる。

## 実際のRails Logで確認できたこと

GET /products/20 にアクセスすると、

Started GET "/products/20"

が出力され、RailsがHTTP GET Requestを受け取ったことを確認できた。

その後、

Processing by ProductsController#show as HTML

が出力され、routesによってProductsController#showへ処理が渡ったことを確認できた。

Parameters: {"id"=>"20"}

から、URLの20がparams[:id]としてControllerへ渡っていることを確認できた。

Product Load

では、Product.find_by!に対応するSQLが実行されていた。

また、

ProductVariant Load

では、@product.product_variants.where(...).order(...) に対応するSQLが実行されていた。

ProductVariantのSQLはControllerではなくView内のnone?でデータが必要になった時点で実行されており、
ActiveRecord::Relationが遅延評価されることを確認できた。

最後に、

Completed 200 OK

が出力され、HTML生成まで正常に完了してHTTP Responseが返されたことを確認できた。