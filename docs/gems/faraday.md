# Faraday - HTTP クライアント

## 概要

FaradayはRubyの柔軟なHTTPクライアントライブラリです。ミドルウェアアーキテクチャにより、リクエスト/レスポンスの処理をカスタマイズできます。

> **🐍 Python相当:** `requests`（定番・同期）／ `httpx`（`requests`互換APIで非同期にも対応した現代的な選択肢）。

```bash
# インストール
gem install faraday

# Gemfile
gem 'faraday'
gem 'faraday-retry'      # リトライ機能
gem 'faraday-multipart'  # ファイルアップロード
```

---

## 基本的な使い方

### シンプルなリクエスト

```ruby
require 'faraday'

# GET
response = Faraday.get('https://api.example.com/users')
response.status  # => 200
response.body    # => レスポンスボディ
response.headers # => ヘッダー

# POST
response = Faraday.post('https://api.example.com/users',
  { name: 'Alice', email: 'alice@example.com' }.to_json,
  'Content-Type' => 'application/json'
)

# 他のHTTPメソッド
Faraday.put(url, body, headers)
Faraday.patch(url, body, headers)
Faraday.delete(url)
Faraday.head(url)
```

**🐍 Python:**

```python
import requests

# GET
resp = requests.get("https://api.example.com/users")
resp.status_code  # => 200      (Faraday の response.status に相当)
resp.text         # => 生のレスポンスボディ (Faraday の response.body)
resp.json()       # => JSON をパース済み dict/list
resp.headers      # => ヘッダー

# POST (json= で自動的に JSON 化 + Content-Type 付与)
resp = requests.post(
    "https://api.example.com/users",
    json={"name": "Alice", "email": "alice@example.com"},
)

# 他の HTTP メソッド
requests.put(url, json=body)
requests.patch(url, json=body)
requests.delete(url)
requests.head(url)
```

`requests` はボディを `resp.text`（文字列）と `resp.json()`（パース済み）に分けて公開します。Faradayの`f.response :json`ミドルウェアのように、`json=`引数を渡すだけでJSONエンコードとヘッダー付与が自動で行われます。

### コネクションの作成

```ruby
# 基本的なコネクション
conn = Faraday.new(url: 'https://api.example.com') do |f|
  f.request :json                    # リクエストをJSON化
  f.response :json                   # レスポンスをパース
  f.adapter Faraday.default_adapter  # デフォルトアダプタ
end

# リクエスト
response = conn.get('/users')
response = conn.get('/users/1')
response = conn.post('/users', { name: 'Alice' })
```

**🐍 Python:**

```python
import requests

# Session はベースヘッダーや接続プールを共有する Faraday の conn 相当
session = requests.Session()
session.headers.update({"Authorization": "Bearer token123"})

resp = session.get("https://api.example.com/users")
resp = session.post("https://api.example.com/users", json={"name": "Alice"})

# httpx なら base_url を持てるので Faraday の url: 指定により近い
import httpx
client = httpx.Client(base_url="https://api.example.com")
resp = client.get("/users")
```

Faradayの`conn`はミドルウェアスタックとベース設定を束ねますが、`requests`ではベースURLの概念がなく、共有設定は`Session`に集約します。ベースURL付きのクライアントが欲しい場合は`httpx.Client(base_url=...)`が最も近い書き味です。

### 詳細な設定

```ruby
conn = Faraday.new(
  url: 'https://api.example.com',
  headers: {
    'Authorization' => 'Bearer token123',
    'User-Agent' => 'MyApp/1.0'
  },
  params: { api_version: 'v2' }  # 全リクエストに付加
) do |f|
  # リクエストミドルウェア
  f.request :json
  f.request :authorization, 'Bearer', 'token123'

  # レスポンスミドルウェア
  f.response :json
  f.response :logger  # ログ出力

  # タイムアウト設定
  f.options.timeout = 30
  f.options.open_timeout = 10

  # アダプタ
  f.adapter :net_http
end
```

---

## リクエスト

### GET リクエスト

```ruby
conn = Faraday.new(url: 'https://api.example.com')

# パスのみ
response = conn.get('/users')

# クエリパラメータ
response = conn.get('/users', { page: 1, per_page: 20 })
# => /users?page=1&per_page=20

# ブロック形式
response = conn.get('/users') do |req|
  req.params['page'] = 1
  req.headers['X-Custom-Header'] = 'value'
end
```

### POST リクエスト

```ruby
# ハッシュを送信（form-urlencoded）
response = conn.post('/users', name: 'Alice', email: 'alice@example.com')

# JSONを送信
conn = Faraday.new(url: 'https://api.example.com') do |f|
  f.request :json
  f.response :json
end

response = conn.post('/users', { name: 'Alice', email: 'alice@example.com' })

# ブロック形式
response = conn.post('/users') do |req|
  req.headers['Content-Type'] = 'application/json'
  req.body = { name: 'Alice' }.to_json
end
```

### ファイルアップロード

```ruby
require 'faraday'
require 'faraday/multipart'

conn = Faraday.new(url: 'https://api.example.com') do |f|
  f.request :multipart
  f.request :url_encoded
  f.adapter :net_http
end

payload = {
  file: Faraday::Multipart::FilePart.new(
    '/path/to/file.pdf',
    'application/pdf',
    'document.pdf'
  ),
  description: 'My document'
}

response = conn.post('/upload', payload)
```

---

## レスポンス

### レスポンスの処理

```ruby
response = conn.get('/users')

# ステータス
response.status         # => 200
response.success?       # => true (2xx)
response.finished?      # => true

# ボディ
response.body           # => パース済みまたは生データ

# ヘッダー
response.headers        # => ヘッダーハッシュ
response.headers['content-type']

# 環境情報
response.env            # => リクエスト/レスポンスの詳細
```

### エラーハンドリング

```ruby
begin
  response = conn.get('/users')

  case response.status
  when 200..299
    response.body
  when 400
    raise "Bad request: #{response.body}"
  when 401
    raise "Unauthorized"
  when 404
    raise "Not found"
  when 500..599
    raise "Server error: #{response.status}"
  end
rescue Faraday::ConnectionFailed => e
  puts "Connection failed: #{e.message}"
rescue Faraday::TimeoutError => e
  puts "Timeout: #{e.message}"
rescue Faraday::ClientError => e
  puts "Client error: #{e.message}"
end
```

### 例外を発生させる

```ruby
conn = Faraday.new(url: 'https://api.example.com') do |f|
  f.request :json
  f.response :json
  f.response :raise_error  # 4xx/5xxで例外
end

begin
  response = conn.get('/not-found')
rescue Faraday::ResourceNotFound => e
  puts "404: #{e.message}"
rescue Faraday::ClientError => e
  puts "4xx: #{e.message}"
rescue Faraday::ServerError => e
  puts "5xx: #{e.message}"
end
```

---

## ミドルウェア

### リクエストミドルウェア

```ruby
conn = Faraday.new do |f|
  # JSON エンコード
  f.request :json

  # フォームエンコード
  f.request :url_encoded

  # 認証
  f.request :authorization, 'Bearer', 'token'
  f.request :authorization, 'Basic', 'user', 'pass'

  # リトライ（faraday-retry gem）
  f.request :retry, {
    max: 3,
    interval: 0.5,
    interval_randomness: 0.5,
    backoff_factor: 2,
    retry_statuses: [500, 502, 503],
    methods: [:get, :post],
    retry_if: ->(env, exc) { exc.is_a?(Faraday::TimeoutError) }
  }

  # インストルメンテーション
  f.request :instrumentation
end
```

### レスポンスミドルウェア

```ruby
conn = Faraday.new do |f|
  # JSON パース
  f.response :json, content_type: /\bjson$/

  # ログ出力
  f.response :logger

  # 詳細なログ
  f.response :logger, Logger.new(STDOUT), bodies: true

  # エラー時に例外
  f.response :raise_error

  # キャッシュ（別gem必要）
  # f.response :caching, store: cache_store
end
```

### カスタムミドルウェア

```ruby
class MyMiddleware < Faraday::Middleware
  def initialize(app, options = {})
    super(app)
    @options = options
  end

  def on_request(env)
    # リクエスト前の処理
    env.request_headers['X-Request-ID'] = SecureRandom.uuid
    puts "Request: #{env.method.upcase} #{env.url}"
  end

  def on_complete(env)
    # レスポンス後の処理
    puts "Response: #{env.status}"
  end
end

# 登録
Faraday::Request.register_middleware(my_middleware: MyMiddleware)

# 使用
conn = Faraday.new do |f|
  f.request :my_middleware, option: 'value'
end
```

**🐍 Python:**

```python
import requests

session = requests.Session()

# リクエスト前フック相当: 全リクエストに共通ヘッダーを付与
session.headers["X-Request-ID"] = "..."

# レスポンス後フック: response イベントに登録
def log_response(resp, *args, **kwargs):
    print(f"Response: {resp.status_code}")

session.hooks["response"].append(log_response)

# httpx はより明示的なイベントフックを持つ
import httpx
client = httpx.Client(
    event_hooks={
        "request": [lambda req: print(f"Request: {req.method} {req.url}")],
        "response": [lambda resp: print(f"Response: {resp.status_code}")],
    },
)
```

Faradayのようなリクエスト/レスポンスを積み重ねるミドルウェアスタックは`requests`/`httpx`には無く、代わりに`Session`のフック（`session.hooks`はレスポンス専用）や`httpx`の`event_hooks`（request/responseの両方）で近いことを実現します。低レベルな差し替えは`requests`のカスタム`HTTPAdapter`が担います。

---

## 認証

### Bearer Token

```ruby
conn = Faraday.new(url: 'https://api.example.com') do |f|
  f.request :authorization, 'Bearer', 'your-token'
end

# または動的に
conn = Faraday.new(url: 'https://api.example.com') do |f|
  f.request :authorization, 'Bearer', -> { fetch_token }
end

def fetch_token
  # トークンを取得するロジック
  'dynamic-token'
end
```

### Basic 認証

```ruby
conn = Faraday.new(url: 'https://api.example.com') do |f|
  f.request :authorization, :basic, 'username', 'password'
end

# または
conn = Faraday.new(url: 'https://api.example.com') do |f|
  f.request :basic_auth, 'username', 'password'
end
```

### OAuth

```ruby
# 手動でヘッダー設定
conn = Faraday.new(url: 'https://api.example.com',
  headers: { 'Authorization' => "OAuth #{oauth_token}" }
)
```

---

## 実践的な例

### API クライアントクラス

```ruby
class ApiClient
  BASE_URL = 'https://api.example.com'

  def initialize(api_key)
    @conn = Faraday.new(url: BASE_URL) do |f|
      f.request :json
      f.request :authorization, 'Bearer', api_key
      f.response :json
      f.response :raise_error
      f.options.timeout = 30
    end
  end

  def get_users(page: 1, per_page: 20)
    response = @conn.get('/users', page: page, per_page: per_page)
    response.body
  end

  def get_user(id)
    response = @conn.get("/users/#{id}")
    response.body
  end

  def create_user(attributes)
    response = @conn.post('/users', attributes)
    response.body
  end

  def update_user(id, attributes)
    response = @conn.patch("/users/#{id}", attributes)
    response.body
  end

  def delete_user(id)
    @conn.delete("/users/#{id}")
    true
  end
end

# 使用
client = ApiClient.new('your-api-key')
users = client.get_users(page: 1)
user = client.create_user(name: 'Alice', email: 'alice@example.com')
```

### リトライ付きクライアント

```ruby
require 'faraday'
require 'faraday/retry'

class RobustApiClient
  def initialize
    @conn = Faraday.new(url: 'https://api.example.com') do |f|
      f.request :json
      f.request :retry, {
        max: 3,
        interval: 1,
        backoff_factor: 2,
        retry_statuses: [429, 500, 502, 503, 504],
        retry_if: ->(env, exc) {
          exc.is_a?(Faraday::TimeoutError) ||
          exc.is_a?(Faraday::ConnectionFailed)
        }
      }
      f.response :json
      f.options.timeout = 10
    end
  end

  def fetch(path)
    response = @conn.get(path)
    response.body
  rescue Faraday::Error => e
    { error: e.message }
  end
end
```

**🐍 Python:**

```python
import requests
from requests.adapters import HTTPAdapter
from urllib3.util.retry import Retry

retry = Retry(
    total=3,
    backoff_factor=1,                         # Faraday の backoff_factor に相当
    status_forcelist=[429, 500, 502, 503, 504],
    allowed_methods=["GET", "POST"],
)

session = requests.Session()
session.mount("https://", HTTPAdapter(max_retries=retry))

# タイムアウトは呼び出しごとに指定 (接続, 読み取り)
resp = session.get("https://api.example.com/users", timeout=(10, 10))

# httpx は transport でリトライ、client でタイムアウトを設定
import httpx
client = httpx.Client(
    transport=httpx.HTTPTransport(retries=3),
    timeout=10.0,
)
```

Faradayの`f.request :retry`ミドルウェアに相当するのは`urllib3`の`Retry`を`HTTPAdapter`に載せて`Session`に`mount`する方法です。ただし`urllib3`のリトライはステータスコードや接続エラーが対象で、`retry_if`のような任意条件は自前実装（例: `tenacity`）が必要です。タイムアウトはFaradayのように接続へ固定するのではなく、`requests`では呼び出しごとの`timeout`引数で指定します。

### 並列リクエスト

```ruby
require 'faraday'

conn = Faraday.new(url: 'https://api.example.com')

# 並列実行（要: concurrent-ruby or typhoeus）
responses = []

conn.in_parallel do
  responses << conn.get('/users/1')
  responses << conn.get('/users/2')
  responses << conn.get('/users/3')
end

responses.each do |response|
  puts response.body
end
```

---

## テスト

### WebMock との連携

```ruby
require 'webmock/rspec'

RSpec.describe ApiClient do
  let(:client) { ApiClient.new('test-key') }

  before do
    stub_request(:get, 'https://api.example.com/users')
      .to_return(
        status: 200,
        body: [{ id: 1, name: 'Alice' }].to_json,
        headers: { 'Content-Type' => 'application/json' }
      )
  end

  it 'fetches users' do
    users = client.get_users
    expect(users).to eq([{ 'id' => 1, 'name' => 'Alice' }])
  end
end
```

### Faraday のテストアダプタ

```ruby
stubs = Faraday::Adapter::Test::Stubs.new do |stub|
  stub.get('/users') { [200, {}, '[{"id":1}]'] }
  stub.post('/users') { [201, {}, '{"id":2}'] }
end

conn = Faraday.new do |f|
  f.adapter :test, stubs
end

response = conn.get('/users')
# => status: 200, body: '[{"id":1}]'
```

---

## 参考リンク

- [Faraday公式ドキュメント](https://lostisland.github.io/faraday/)
- [Faraday GitHub](https://github.com/lostisland/faraday)
- [利用可能なミドルウェア一覧](https://github.com/lostisland/awesome-faraday)

← [Gem一覧に戻る](README.md)
