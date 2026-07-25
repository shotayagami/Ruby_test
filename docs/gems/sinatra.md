# Sinatra - 軽量Webフレームワーク

## 概要

SinatraはRubyの軽量Webフレームワークです。最小限の設定でWebアプリケーションやAPIを素早く構築できます。シンプルさが特徴で、学習コストが低く、小規模なプロジェクトやAPIサーバーに最適です。

> **🐍 Python相当:** Flask（軽量Webフレームワーク。FastAPIも近い）

```bash
# インストール
gem install sinatra
gem install puma  # Webサーバー（推奨）

# Gemfile
gem 'sinatra'
gem 'puma'
```

---

## 基本的な使い方

### Hello World

```ruby
# app.rb
require 'sinatra'

get '/' do
  'Hello, World!'
end
```

**🐍 Python:** Flaskはルートをデコレータで宣言し、関数の戻り値がレスポンスになります（Sinatraのブロックの代わりに関数、末尾は `return`）。

```python
# app.py
from flask import Flask

app = Flask(__name__)

@app.route("/")
def index():
    return "Hello, World!"
```

```bash
# 起動
ruby app.rb
# => http://localhost:4567 でアクセス可能
```

**🐍 Python:** 開発サーバーは `flask run`（既定 <http://localhost:5000）。本番は> `gunicorn app:app` などのWSGIサーバーを使う（SinatraのPumaに相当）。

### モジュラースタイル（推奨）

```ruby
# app.rb
require 'sinatra/base'

class MyApp < Sinatra::Base
  get '/' do
    'Hello, World!'
  end

  # 開発時のみ直接実行
  run! if app_file == $0
end
```

**🐍 Python:** Flaskではモジュール化にBlueprintを使い、`create_app()` のアプリケーションファクトリで組み立てるのが定番です。

```python
# users/routes.py
from flask import Blueprint

users_bp = Blueprint("users", __name__)

@users_bp.route("/")
def index():
    return "Hello, World!"

# app.py
from flask import Flask
from users.routes import users_bp

def create_app():
    app = Flask(__name__)
    app.register_blueprint(users_bp)
    return app
```

### config.ru を使用

```ruby
# config.ru
require './app'

run MyApp
```

```bash
# 起動
bundle exec rackup -p 4567
# または
bundle exec puma config.ru -p 4567
```

---

## ルーティング

### 基本的なルート

```ruby
class MyApp < Sinatra::Base
  # GET
  get '/' do
    'Home page'
  end

  # POST
  post '/users' do
    'Create user'
  end

  # PUT
  put '/users/:id' do
    'Update user'
  end

  # PATCH
  patch '/users/:id' do
    'Partial update'
  end

  # DELETE
  delete '/users/:id' do
    'Delete user'
  end

  # OPTIONS
  options '/users' do
    'Options'
  end
end
```

**🐍 Python:** Flaskは1つの `@app.route` に `methods=[...]` を渡してHTTPメソッドを指定します（Sinatraのように動詞ごとのメソッドはありませんが、`@app.get(...)` / `@app.post(...)` のショートカットもあります）。

```python
@app.route("/", methods=["GET"])
def home():
    return "Home page"

@app.route("/users", methods=["POST"])
def create_user():
    return "Create user"

@app.route("/users/<id>", methods=["PUT", "PATCH"])
def update_user(id):
    return "Update user"

@app.route("/users/<id>", methods=["DELETE"])
def delete_user(id):
    return "Delete user"
```

### パラメータ

```ruby
class MyApp < Sinatra::Base
  # URLパラメータ
  get '/users/:id' do
    "User ID: #{params[:id]}"
  end

  # 複数のパラメータ
  get '/users/:user_id/posts/:post_id' do
    "User: #{params[:user_id]}, Post: #{params[:post_id]}"
  end

  # オプショナルパラメータ
  get '/posts/?:year?/?:month?' do
    year = params[:year] || 'all'
    month = params[:month] || 'all'
    "Posts for #{year}/#{month}"
  end

  # クエリパラメータ
  get '/search' do
    # /search?q=ruby&page=1
    "Query: #{params[:q]}, Page: #{params[:page]}"
  end

  # ワイルドカード
  get '/files/*.*' do
    # /files/path/to/file.txt
    # params[:splat] => ['path/to/file', 'txt']
    "Path: #{params[:splat][0]}, Extension: #{params[:splat][1]}"
  end

  # 正規表現
  get %r{/posts/(\d+)} do |id|
    "Post ID: #{id}"
  end
end
```

**🐍 Python:** 動的セグメントは `/:id` ではなく `/<id>`（型変換は `/<int:id>`）で書き、関数の引数として受け取ります。クエリ文字列は `request.args.get("q")` で取得します（`params[:q]` に相当）。

```python
from flask import request

# URLパラメータ（型変換あり）
@app.route("/users/<int:id>")
def show_user(id):
    return f"User ID: {id}"

# 複数のパラメータ
@app.route("/users/<user_id>/posts/<post_id>")
def user_post(user_id, post_id):
    return f"User: {user_id}, Post: {post_id}"

# クエリパラメータ  /search?q=ruby&page=1
@app.route("/search")
def search():
    return f"Query: {request.args.get('q')}, Page: {request.args.get('page')}"
```

### リクエストボディ

```ruby
class MyApp < Sinatra::Base
  post '/users' do
    # フォームデータ
    name = params[:name]
    email = params[:email]

    # JSONボディ
    request.body.rewind
    data = JSON.parse(request.body.read)
    name = data['name']

    "Created: #{name}"
  end
end
```

**🐍 Python:** フォームデータは `request.form`、JSONボディは `request.get_json()` で受け取ります（Sinatraの `params[:name]` / 手動 `JSON.parse` に相当）。

```python
@app.route("/users", methods=["POST"])
def create_user():
    # フォームデータ
    name = request.form.get("name")
    email = request.form.get("email")

    # JSONボディ
    data = request.get_json()
    name = data["name"]

    return f"Created: {name}"
```

---

## レスポンス

### 基本的なレスポンス

```ruby
class MyApp < Sinatra::Base
  # 文字列を返す
  get '/' do
    'Hello'
  end

  # ステータスコード
  get '/not_found' do
    status 404
    'Not Found'
  end

  # ヘッダー
  get '/custom' do
    headers 'X-Custom-Header' => 'value'
    'Custom response'
  end

  # Content-Type
  get '/json' do
    content_type :json
    { message: 'Hello' }.to_json
  end

  # 配列で返す [status, headers, body]
  get '/full' do
    [201, { 'Content-Type' => 'text/plain' }, 'Created']
  end
end
```

**🐍 Python:** Flaskはステータスや ヘッダーを戻り値のタプルで表します（`return body, 404` や `return body, status, headers`）。Sinatraの `status 404` に相当する明示的な呼び出しではなく、戻り値で指定するのが基本です。

```python
@app.route("/not_found")
def not_found():
    return "Not Found", 404

@app.route("/custom")
def custom():
    return "Custom response", 200, {"X-Custom-Header": "value"}

@app.route("/full")
def full():
    return "Created", 201, {"Content-Type": "text/plain"}
```

### JSON API

```ruby
require 'sinatra/base'
require 'json'

class ApiApp < Sinatra::Base
  before do
    content_type :json
  end

  get '/users' do
    users = [
      { id: 1, name: 'Alice' },
      { id: 2, name: 'Bob' }
    ]
    users.to_json
  end

  post '/users' do
    request.body.rewind
    data = JSON.parse(request.body.read)

    # バリデーション
    halt 400, { error: 'Name is required' }.to_json if data['name'].nil?

    user = { id: 3, name: data['name'] }
    status 201
    user.to_json
  end

  not_found do
    { error: 'Not Found' }.to_json
  end

  error do
    { error: 'Internal Server Error' }.to_json
  end
end
```

**🐍 Python:** Flaskは `jsonify()` で辞書やリストをJSONレスポンスに変換します（Content-Typeも自動設定。Sinatraの `content_type :json` + `.to_json` に相当）。エラーは `@app.errorhandler` で定義します。

```python
from flask import jsonify

@app.route("/users")
def users():
    return jsonify([{"id": 1, "name": "Alice"}, {"id": 2, "name": "Bob"}])

@app.route("/users", methods=["POST"])
def create_user():
    data = request.get_json()
    if not data.get("name"):
        return jsonify(error="Name is required"), 400
    return jsonify(id=3, name=data["name"]), 201

@app.errorhandler(404)
def handle_404(e):
    return jsonify(error="Not Found"), 404
```

### リダイレクト

```ruby
class MyApp < Sinatra::Base
  get '/old' do
    redirect '/new'
  end

  get '/permanent' do
    redirect '/new', 301  # 恒久的リダイレクト
  end

  get '/back' do
    redirect back  # 前のページへ
  end
end
```

### ファイル送信

```ruby
class MyApp < Sinatra::Base
  get '/download' do
    send_file 'path/to/file.pdf',
              filename: 'document.pdf',
              type: 'application/pdf'
  end

  get '/image' do
    send_file 'public/images/logo.png',
              type: 'image/png',
              disposition: 'inline'  # ブラウザで表示
  end
end
```

---

## ビューとテンプレート

### ERB テンプレート

```ruby
# app.rb
class MyApp < Sinatra::Base
  get '/' do
    @title = 'Home'
    @users = User.all
    erb :index
  end

  get '/users/:id' do
    @user = User.find(params[:id])
    erb :show
  end
end
```

**🐍 Python:** FlaskはJinja2テンプレートを標準搭載し、`render_template("index.html", ...)` で `templates/` 配下のファイルを描画します（ERB/hamlに相当。インスタンス変数ではなくキーワード引数で値を渡す）。

```python
from flask import render_template

@app.route("/")
def index():
    return render_template("index.html", title="Home", users=User.all())

@app.route("/users/<id>")
def show(id):
    return render_template("show.html", user=User.find(id))
```

```erb
<%# views/layout.erb %>
<!DOCTYPE html>
<html>
<head>
  <title><%= @title %></title>
</head>
<body>
  <%= yield %>
</body>
</html>

<%# views/index.erb %>
<h1>Users</h1>
<ul>
  <% @users.each do |user| %>
    <li><%= user.name %></li>
  <% end %>
</ul>

<%# views/show.erb %>
<h1><%= @user.name %></h1>
<p>Email: <%= @user.email %></p>
```

### レイアウト

```ruby
class MyApp < Sinatra::Base
  # デフォルトレイアウト（views/layout.erb）
  get '/' do
    erb :index
  end

  # レイアウトを指定
  get '/admin' do
    erb :admin, layout: :admin_layout
  end

  # レイアウトなし
  get '/partial' do
    erb :partial, layout: false
  end
end
```

### パーシャル

```erb
<%# views/index.erb %>
<h1>Users</h1>
<%= erb :_user_list, locals: { users: @users } %>

<%# views/_user_list.erb %>
<ul>
  <% users.each do |user| %>
    <li><%= user.name %></li>
  <% end %>
</ul>
```

### その他のテンプレートエンジン

```ruby
# Gemfile
gem 'slim'
gem 'haml'

# app.rb
class MyApp < Sinatra::Base
  get '/slim' do
    slim :index
  end

  get '/haml' do
    haml :index
  end
end
```

---

## フィルター

### before / after

```ruby
class MyApp < Sinatra::Base
  # すべてのリクエスト前
  before do
    @start_time = Time.now
    content_type :json if request.accept?('application/json')
  end

  # すべてのリクエスト後
  after do
    duration = Time.now - @start_time
    logger.info "Request took #{duration}s"
  end

  # 特定のパスのみ
  before '/admin/*' do
    authenticate!
  end

  before '/api/*' do
    content_type :json
  end

  # 特定のHTTPメソッド
  before :post, :put, :patch do
    request.body.rewind
  end

  get '/' do
    'Hello'
  end

  private

  def authenticate!
    halt 401, { error: 'Unauthorized' }.to_json unless current_user
  end
end
```

---

## ヘルパー

### ヘルパーメソッドの定義

```ruby
class MyApp < Sinatra::Base
  helpers do
    def current_user
      @current_user ||= User.find_by(id: session[:user_id])
    end

    def logged_in?
      !current_user.nil?
    end

    def authenticate!
      halt 401, 'Unauthorized' unless logged_in?
    end

    def h(text)
      Rack::Utils.escape_html(text)
    end
  end

  get '/profile' do
    authenticate!
    erb :profile
  end
end
```

### モジュールとしてのヘルパー

```ruby
# helpers/auth_helpers.rb
module AuthHelpers
  def current_user
    @current_user ||= User.find_by(id: session[:user_id])
  end

  def authenticate!
    halt 401, 'Unauthorized' unless current_user
  end
end

# app.rb
class MyApp < Sinatra::Base
  helpers AuthHelpers

  get '/admin' do
    authenticate!
    'Admin area'
  end
end
```

---

## セッションとCookie

### セッション

```ruby
class MyApp < Sinatra::Base
  enable :sessions
  set :session_secret, ENV.fetch('SESSION_SECRET') { SecureRandom.hex(64) }

  get '/login' do
    erb :login
  end

  post '/login' do
    user = User.authenticate(params[:email], params[:password])
    if user
      session[:user_id] = user.id
      redirect '/dashboard'
    else
      @error = 'Invalid credentials'
      erb :login
    end
  end

  get '/logout' do
    session.clear
    redirect '/'
  end

  get '/dashboard' do
    @user = User.find(session[:user_id])
    erb :dashboard
  end
end
```

### Cookie

```ruby
class MyApp < Sinatra::Base
  get '/remember' do
    response.set_cookie('remember_token',
                        value: 'abc123',
                        expires: Time.now + 30 * 24 * 60 * 60,  # 30日
                        httponly: true,
                        secure: production?)
    'Cookie set'
  end

  get '/forget' do
    response.delete_cookie('remember_token')
    'Cookie deleted'
  end

  get '/check' do
    token = request.cookies['remember_token']
    "Token: #{token}"
  end
end
```

---

## エラーハンドリング

### エラーページ

```ruby
class MyApp < Sinatra::Base
  # 404エラー
  not_found do
    erb :not_found
  end

  # 500エラー
  error do
    @error = env['sinatra.error']
    erb :error
  end

  # 特定の例外
  error ActiveRecord::RecordNotFound do
    status 404
    erb :not_found
  end

  error ActiveRecord::RecordInvalid do
    status 422
    @errors = env['sinatra.error'].record.errors
    erb :validation_error
  end

  # 特定のステータスコード
  error 403 do
    erb :forbidden
  end
end
```

### halt

```ruby
class MyApp < Sinatra::Base
  get '/protected' do
    halt 403, 'Access Denied' unless authorized?
    'Secret content'
  end

  get '/api/resource' do
    halt 404, { error: 'Not found' }.to_json unless resource
    resource.to_json
  end
end
```

---

## 設定

### 環境設定

```ruby
class MyApp < Sinatra::Base
  # 全環境共通
  configure do
    enable :logging
    set :views, 'app/views'
    set :public_folder, 'public'
  end

  # 開発環境
  configure :development do
    enable :reloader  # sinatra/reloader が必要
    set :show_exceptions, true
  end

  # 本番環境
  configure :production do
    disable :show_exceptions
    enable :dump_errors
  end

  # テスト環境
  configure :test do
    disable :logging
  end
end
```

### 設定オプション

```ruby
class MyApp < Sinatra::Base
  # 基本設定
  set :app_file, __FILE__
  set :root, File.dirname(__FILE__)
  set :views, Proc.new { File.join(root, 'views') }
  set :public_folder, Proc.new { File.join(root, 'public') }

  # セッション
  enable :sessions
  set :session_secret, ENV['SESSION_SECRET']

  # セキュリティ
  set :protection, except: [:json_csrf]

  # メソッドオーバーライド（_method パラメータ）
  enable :method_override

  # 静的ファイル
  enable :static

  # ロギング
  enable :logging

  # カスタム設定
  set :api_key, ENV['API_KEY']
  set :max_upload_size, 10 * 1024 * 1024  # 10MB
end
```

---

## ミドルウェア

### Rack ミドルウェア

```ruby
class MyApp < Sinatra::Base
  # 組み込みミドルウェア
  use Rack::Deflater  # 圧縮
  use Rack::CommonLogger  # ログ
  use Rack::Session::Cookie, secret: 'secret'

  # カスタムミドルウェア
  use RequestTimer

  get '/' do
    'Hello'
  end
end

# カスタムミドルウェア
class RequestTimer
  def initialize(app)
    @app = app
  end

  def call(env)
    start = Time.now
    status, headers, body = @app.call(env)
    duration = Time.now - start
    headers['X-Response-Time'] = "#{duration}s"
    [status, headers, body]
  end
end
```

---

## 拡張機能

### Sinatra Contrib

```ruby
# Gemfile
gem 'sinatra-contrib'
```

```ruby
require 'sinatra/base'
require 'sinatra/reloader'  # 開発時の自動リロード
require 'sinatra/json'      # JSONヘルパー
require 'sinatra/namespace' # 名前空間

class MyApp < Sinatra::Base
  configure :development do
    register Sinatra::Reloader
  end

  register Sinatra::Namespace

  # JSON ヘルパー
  get '/api/users' do
    json users: User.all
  end

  # 名前空間
  namespace '/api/v1' do
    get '/users' do
      json User.all
    end

    get '/users/:id' do
      json User.find(params[:id])
    end
  end

  namespace '/api/v2' do
    get '/users' do
      json data: User.all, version: 2
    end
  end
end
```

---

## テスト

### RSpec でのテスト

```ruby
# Gemfile
group :test do
  gem 'rspec'
  gem 'rack-test'
end
```

```ruby
# spec/spec_helper.rb
ENV['RACK_ENV'] = 'test'

require_relative '../app'
require 'rspec'
require 'rack/test'

RSpec.configure do |config|
  config.include Rack::Test::Methods
end

def app
  MyApp
end
```

```ruby
# spec/app_spec.rb
require 'spec_helper'

RSpec.describe MyApp do
  describe 'GET /' do
    it 'returns success' do
      get '/'
      expect(last_response).to be_ok
      expect(last_response.body).to include('Hello')
    end
  end

  describe 'GET /users/:id' do
    it 'returns user' do
      get '/users/1'
      expect(last_response).to be_ok
    end

    it 'returns 404 for non-existent user' do
      get '/users/999'
      expect(last_response.status).to eq(404)
    end
  end

  describe 'POST /users' do
    it 'creates user' do
      post '/users', { name: 'Alice', email: 'alice@example.com' }.to_json,
           'CONTENT_TYPE' => 'application/json'

      expect(last_response.status).to eq(201)
      expect(JSON.parse(last_response.body)['name']).to eq('Alice')
    end
  end

  describe 'with authentication' do
    it 'requires login' do
      get '/admin'
      expect(last_response.status).to eq(401)
    end

    it 'allows authenticated access' do
      # セッションを設定
      env 'rack.session', { user_id: 1 }
      get '/admin'
      expect(last_response).to be_ok
    end
  end
end
```

---

## 実践的な例

### RESTful API

```ruby
require 'sinatra/base'
require 'sinatra/namespace'
require 'json'

class ApiApp < Sinatra::Base
  register Sinatra::Namespace

  before do
    content_type :json
  end

  # エラーハンドリング
  error JSON::ParserError do
    status 400
    { error: 'Invalid JSON' }.to_json
  end

  not_found do
    { error: 'Not Found' }.to_json
  end

  helpers do
    def json_body
      request.body.rewind
      JSON.parse(request.body.read)
    rescue JSON::ParserError
      halt 400, { error: 'Invalid JSON' }.to_json
    end
  end

  namespace '/api/v1' do
    # GET /api/v1/users
    get '/users' do
      users = User.all
      { data: users, count: users.count }.to_json
    end

    # GET /api/v1/users/:id
    get '/users/:id' do
      user = User.find_by(id: params[:id])
      halt 404 unless user
      { data: user }.to_json
    end

    # POST /api/v1/users
    post '/users' do
      user = User.new(json_body)
      if user.save
        status 201
        { data: user }.to_json
      else
        status 422
        { errors: user.errors.full_messages }.to_json
      end
    end

    # PUT /api/v1/users/:id
    put '/users/:id' do
      user = User.find_by(id: params[:id])
      halt 404 unless user

      if user.update(json_body)
        { data: user }.to_json
      else
        status 422
        { errors: user.errors.full_messages }.to_json
      end
    end

    # DELETE /api/v1/users/:id
    delete '/users/:id' do
      user = User.find_by(id: params[:id])
      halt 404 unless user

      user.destroy
      status 204
    end
  end
end
```

### Webアプリケーション

```ruby
require 'sinatra/base'
require 'sinatra/flash'

class WebApp < Sinatra::Base
  register Sinatra::Flash

  enable :sessions
  set :session_secret, ENV['SESSION_SECRET']

  helpers do
    def current_user
      @current_user ||= User.find_by(id: session[:user_id])
    end

    def logged_in?
      !current_user.nil?
    end

    def require_login!
      unless logged_in?
        flash[:error] = 'ログインが必要です'
        redirect '/login'
      end
    end
  end

  # ホーム
  get '/' do
    @posts = Post.recent.limit(10)
    erb :index
  end

  # 認証
  get '/login' do
    erb :login
  end

  post '/login' do
    user = User.authenticate(params[:email], params[:password])
    if user
      session[:user_id] = user.id
      flash[:success] = 'ログインしました'
      redirect '/'
    else
      flash[:error] = 'メールアドレスまたはパスワードが違います'
      erb :login
    end
  end

  get '/logout' do
    session.clear
    flash[:success] = 'ログアウトしました'
    redirect '/'
  end

  # 投稿
  get '/posts/new' do
    require_login!
    @post = Post.new
    erb :'posts/new'
  end

  post '/posts' do
    require_login!
    @post = current_user.posts.build(params[:post])
    if @post.save
      flash[:success] = '投稿しました'
      redirect "/posts/#{@post.id}"
    else
      erb :'posts/new'
    end
  end

  get '/posts/:id' do
    @post = Post.find_by(id: params[:id])
    halt 404 unless @post
    erb :'posts/show'
  end
end
```

---

## デプロイ

### Puma 設定

```ruby
# config/puma.rb
workers ENV.fetch('WEB_CONCURRENCY', 2).to_i
threads_count = ENV.fetch('RAILS_MAX_THREADS', 5).to_i
threads threads_count, threads_count

preload_app!

port ENV.fetch('PORT', 4567)
environment ENV.fetch('RACK_ENV', 'development')

on_worker_boot do
  ActiveRecord::Base.establish_connection if defined?(ActiveRecord)
end
```

### Procfile

```text
web: bundle exec puma -C config/puma.rb
```

### Docker

```dockerfile
FROM ruby:3.3-slim

WORKDIR /app

COPY Gemfile Gemfile.lock ./
RUN bundle install --without development test

COPY . .

EXPOSE 4567

CMD ["bundle", "exec", "puma", "-C", "config/puma.rb"]
```

---

## 関連 Gem

| Gem | 説明 |
|-----|------|
| sinatra-contrib | 拡張機能セット |
| sinatra-activerecord | ActiveRecord連携 |
| sinatra-flash | フラッシュメッセージ |
| warden | 認証 |
| rack-protection | セキュリティ |
| rack-cors | CORS対応 |

---

## 参考リンク

- [Sinatra公式サイト](http://sinatrarb.com/)
- [Sinatra README](http://sinatrarb.com/intro.html)
- [Sinatra Contrib](https://github.com/sinatra/sinatra/tree/main/sinatra-contrib)
- [Sinatra Recipes](http://recipes.sinatrarb.com/)

← [Gem一覧に戻る](README.md)
