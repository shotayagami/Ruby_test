# Devise - 認証ソリューション

## 概要

DeviseはRailsアプリケーション向けの柔軟な認証ソリューションです。ユーザー登録、ログイン、パスワードリセットなど、認証に必要な機能を包括的に提供します。

> **🐍 Python相当:** Djangoは標準の django.contrib.auth＋django-allauth。FlaskならFlask-Login＋Flask-Security。

**🐍 Python:** 位置づけの差が重要です。DeviseはRailsのデファクトだが「追加するgem」であるのに対し、**Djangoは認証（Userモデル・`authenticate`/`login`/`logout`・パスワードハッシュ・管理画面ログイン）をフレームワークに標準内蔵**しています。Flaskはマイクロフレームワークなので、セッション管理を担う **Flask-Login** に、登録・確認メール・ロール等をまとめた **Flask-Security-Too** や **Flask-Bcrypt** を組み合わせるのが一般的です。

```bash
# インストール
gem install devise

# Gemfile
gem 'devise'
```

**注意:** DeviseはRails専用のGemです。

---

## セットアップ

### 初期設定

```bash
# Deviseのインストール
rails generate devise:install

# 作成されるファイル
# config/initializers/devise.rb - 設定ファイル
# config/locales/devise.en.yml - 翻訳ファイル
```

### インストール後の設定

```ruby
# config/environments/development.rb
config.action_mailer.default_url_options = { host: 'localhost', port: 3000 }

# config/environments/production.rb
config.action_mailer.default_url_options = { host: 'example.com' }
```

### User モデルの生成

```bash
# Userモデルを生成
rails generate devise User

# 作成されるファイル
# app/models/user.rb
# db/migrate/xxx_devise_create_users.rb
# config/routes.rb に devise_for :users が追加される

# マイグレーション実行
rails db:migrate
```

### 生成されるマイグレーション

```ruby
# db/migrate/xxx_devise_create_users.rb
class DeviseCreateUsers < ActiveRecord::Migration[7.0]
  def change
    create_table :users do |t|
      ## Database authenticatable
      t.string :email,              null: false, default: ""
      t.string :encrypted_password, null: false, default: ""

      ## Recoverable
      t.string   :reset_password_token
      t.datetime :reset_password_sent_at

      ## Rememberable
      t.datetime :remember_created_at

      ## Trackable（オプション）
      # t.integer  :sign_in_count, default: 0, null: false
      # t.datetime :current_sign_in_at
      # t.datetime :last_sign_in_at
      # t.string   :current_sign_in_ip
      # t.string   :last_sign_in_ip

      ## Confirmable（オプション）
      # t.string   :confirmation_token
      # t.datetime :confirmed_at
      # t.datetime :confirmation_sent_at
      # t.string   :unconfirmed_email

      ## Lockable（オプション）
      # t.integer  :failed_attempts, default: 0, null: false
      # t.string   :unlock_token
      # t.datetime :locked_at

      t.timestamps null: false
    end

    add_index :users, :email,                unique: true
    add_index :users, :reset_password_token, unique: true
    # add_index :users, :confirmation_token,   unique: true
    # add_index :users, :unlock_token,         unique: true
  end
end
```

---

## モジュール

### 利用可能なモジュール

```ruby
# app/models/user.rb
class User < ApplicationRecord
  devise :database_authenticatable,  # パスワード認証
         :registerable,              # ユーザー登録
         :recoverable,               # パスワードリセット
         :rememberable,              # ログイン状態の保持
         :validatable,               # メール・パスワードのバリデーション
         :confirmable,               # メール確認
         :lockable,                  # アカウントロック
         :timeoutable,               # セッションタイムアウト
         :trackable,                 # ログイン追跡
         :omniauthable               # OmniAuth連携
end
```

### 各モジュールの説明

| モジュール | 説明 |
|-----------|------|
| database_authenticatable | パスワードをハッシュ化してDBに保存、認証処理 |
| registerable | ユーザー登録、編集、削除 |
| recoverable | パスワードリセット |
| rememberable | Cookieでログイン状態を保持 |
| validatable | メール・パスワードの標準バリデーション |
| confirmable | メールアドレスの確認 |
| lockable | 一定回数のログイン失敗でロック |
| timeoutable | 一定時間後にセッション無効化 |
| trackable | ログイン回数、日時、IPを記録 |
| omniauthable | OmniAuth（SNS認証）連携 |

**🐍 Python:** Deviseは「モジュールを`devise :...`で列挙して機能を足す」設計ですが、Django/Flaskでは相当機能が別々の場所（標準機能・追加パッケージ）に分散します。対応の目安は次の通りです。

| Deviseモジュール | Django | Flask |
|-----------------|--------|-------|
| database_authenticatable | 標準の `authenticate()` とパスワードハッシュ（`set_password`/`check_password`） | Flask-Bcrypt / werkzeug の `generate_password_hash` |
| registerable | フォーム自作、または django-allauth の登録フロー | Flask-Security-Too の登録ビュー |
| recoverable | 標準の `PasswordResetView` ほかパスワードリセットビュー群 | Flask-Security-Too の recoverable |
| rememberable | セッションの `set_expiry` / Cookie 設定 | Flask-Login の `remember=True` |
| confirmable | django-allauth の `ACCOUNT_EMAIL_VERIFICATION` | Flask-Security-Too の confirmable |
| lockable | django-axes 等の追加パッケージ | Flask-Security-Too の login 試行制限 |
| trackable | `last_login` は標準記録、詳細は自作 | 自作 |
| omniauthable | django-allauth のソーシャルアカウント | Flask-Dance / Authlib |

django-allauth は、ソーシャルログイン・メール確認・登録フローをまとめて提供する事実上の標準パッケージです。

---

## 設定

### devise.rb の主要設定

```ruby
# config/initializers/devise.rb
Devise.setup do |config|
  # メール送信元
  config.mailer_sender = 'noreply@example.com'

  # ORMの設定
  require 'devise/orm/active_record'

  # 認証キー
  config.authentication_keys = [:email]

  # 大文字小文字を区別しない
  config.case_insensitive_keys = [:email]

  # 空白を除去
  config.strip_whitespace_keys = [:email]

  # パスワードの最小文字数
  config.password_length = 8..128

  # メール形式のバリデーション
  config.email_regexp = /\A[^@\s]+@[^@\s]+\z/

  # ログイン失敗時のメッセージ
  config.paranoid = true  # ユーザーの存在を隠す

  # セッションタイムアウト（timeoutableモジュール）
  config.timeout_in = 30.minutes

  # ロック設定（lockableモジュール）
  config.lock_strategy = :failed_attempts
  config.unlock_strategy = :both  # :time, :email, :both
  config.maximum_attempts = 5
  config.unlock_in = 1.hour

  # パスワードリセットトークンの有効期限
  config.reset_password_within = 6.hours

  # サインアウト時のHTTPメソッド
  config.sign_out_via = :delete
end
```

---

## コントローラー

### ヘルパーメソッド

```ruby
# コントローラーで使用可能なヘルパー

# 認証が必要なアクション
before_action :authenticate_user!

# 現在のユーザー
current_user

# ログイン済みか確認
user_signed_in?

# 現在のセッション
user_session
```

**🐍 Python:** `current_user` はDjangoでは `request.user`（未ログイン時は `AnonymousUser`）、Flask-Loginでは `flask_login.current_user` に相当します。`before_action :authenticate_user!` はDjangoの `@login_required` デコレータ、Flask-Loginの `@login_required` に対応します。`user_signed_in?` はDjangoなら `request.user.is_authenticated`、Flaskなら `current_user.is_authenticated` です。

```python
# Django（関数ビュー）
from django.contrib.auth.decorators import login_required

@login_required
def index(request):
    user = request.user  # current_user 相当
    ...
```

```python
# Flask（Flask-Login）
from flask_login import login_required, current_user

@app.route("/dashboard")
@login_required
def dashboard():
    return f"Hello {current_user.email}"
```

### フィルターの例

```ruby
class ApplicationController < ActionController::Base
  before_action :authenticate_user!
  before_action :configure_permitted_parameters, if: :devise_controller?

  protected

  def configure_permitted_parameters
    # サインアップ時に許可するパラメータ
    devise_parameter_sanitizer.permit(:sign_up, keys: [:name, :avatar])

    # アカウント更新時に許可するパラメータ
    devise_parameter_sanitizer.permit(:account_update, keys: [:name, :avatar])
  end
end
```

### 特定のアクションのみ認証

```ruby
class ArticlesController < ApplicationController
  before_action :authenticate_user!, except: [:index, :show]

  # または
  before_action :authenticate_user!, only: [:new, :create, :edit, :update, :destroy]
end
```

### カスタムコントローラー

```bash
# コントローラーを生成
rails generate devise:controllers users
```

```ruby
# app/controllers/users/sessions_controller.rb
class Users::SessionsController < Devise::SessionsController
  # ログイン前の処理
  before_action :check_maintenance, only: [:create]

  # ログイン後のリダイレクト先
  def after_sign_in_path_for(resource)
    dashboard_path
  end

  # ログアウト後のリダイレクト先
  def after_sign_out_path_for(resource_or_scope)
    root_path
  end

  private

  def check_maintenance
    if Settings.maintenance_mode?
      redirect_to maintenance_path, alert: 'メンテナンス中です'
    end
  end
end

# app/controllers/users/registrations_controller.rb
class Users::RegistrationsController < Devise::RegistrationsController
  # サインアップ後のリダイレクト先
  def after_sign_up_path_for(resource)
    welcome_path
  end

  # アカウント更新後のリダイレクト先
  def after_update_path_for(resource)
    edit_user_registration_path
  end
end
```

### ルーティング設定

```ruby
# config/routes.rb
Rails.application.routes.draw do
  # デフォルト
  devise_for :users

  # カスタムコントローラーを使用
  devise_for :users, controllers: {
    sessions: 'users/sessions',
    registrations: 'users/registrations',
    passwords: 'users/passwords',
    confirmations: 'users/confirmations',
    unlocks: 'users/unlocks'
  }

  # パスをカスタマイズ
  devise_for :users, path: 'auth', path_names: {
    sign_in: 'login',
    sign_out: 'logout',
    sign_up: 'register',
    password: 'forgot_password'
  }

  # スコープを追加
  devise_scope :user do
    get 'login', to: 'devise/sessions#new'
  end
end
```

---

## ビュー

### ビューの生成

```bash
# すべてのビューを生成
rails generate devise:views

# 特定のモデル用
rails generate devise:views users

# 特定のモジュールのみ
rails generate devise:views -v sessions registrations
```

### ビューのカスタマイズ例

```erb
<%# app/views/devise/sessions/new.html.erb %>
<div class="login-container">
  <h2>ログイン</h2>

  <%= form_for(resource, as: resource_name, url: session_path(resource_name)) do |f| %>
    <div class="field">
      <%= f.label :email, "メールアドレス" %>
      <%= f.email_field :email, autofocus: true, autocomplete: "email" %>
    </div>

    <div class="field">
      <%= f.label :password, "パスワード" %>
      <%= f.password_field :password, autocomplete: "current-password" %>
    </div>

    <% if devise_mapping.rememberable? %>
      <div class="field">
        <%= f.check_box :remember_me %>
        <%= f.label :remember_me, "ログイン状態を保持" %>
      </div>
    <% end %>

    <div class="actions">
      <%= f.submit "ログイン", class: "btn btn-primary" %>
    </div>
  <% end %>

  <div class="links">
    <%= render "devise/shared/links" %>
  </div>
</div>
```

```erb
<%# app/views/devise/registrations/new.html.erb %>
<div class="registration-container">
  <h2>新規登録</h2>

  <%= form_for(resource, as: resource_name, url: registration_path(resource_name)) do |f| %>
    <%= render "devise/shared/error_messages", resource: resource %>

    <div class="field">
      <%= f.label :name, "名前" %>
      <%= f.text_field :name, autofocus: true %>
    </div>

    <div class="field">
      <%= f.label :email, "メールアドレス" %>
      <%= f.email_field :email, autocomplete: "email" %>
    </div>

    <div class="field">
      <%= f.label :password, "パスワード" %>
      <% if @minimum_password_length %>
        <em>(<%= @minimum_password_length %>文字以上)</em>
      <% end %>
      <%= f.password_field :password, autocomplete: "new-password" %>
    </div>

    <div class="field">
      <%= f.label :password_confirmation, "パスワード（確認）" %>
      <%= f.password_field :password_confirmation, autocomplete: "new-password" %>
    </div>

    <div class="actions">
      <%= f.submit "登録", class: "btn btn-primary" %>
    </div>
  <% end %>

  <div class="links">
    <%= render "devise/shared/links" %>
  </div>
</div>
```

---

## OmniAuth 連携

> **🐍 Python相当:** OmniAuth（Google/GitHub等のソーシャルログイン）は、Djangoでは **django-allauth** の `SocialAccount` プロバイダ設定、Flaskでは **Flask-Dance** や **Authlib** に相当します。allauthは `INSTALLED_APPS` に `allauth.socialaccount.providers.google` 等を追加し、管理画面かsettingsでクライアントID/シークレットを登録するだけで、コールバック処理まで内蔵されます（Deviseのように自前のコールバックコントローラーを書く必要は基本的にありません）。

### セットアップ

```ruby
# Gemfile
gem 'devise'
gem 'omniauth'
gem 'omniauth-google-oauth2'
gem 'omniauth-github'
gem 'omniauth-rails_csrf_protection'
```

### 設定

```ruby
# config/initializers/devise.rb
Devise.setup do |config|
  config.omniauth :google_oauth2,
                  ENV['GOOGLE_CLIENT_ID'],
                  ENV['GOOGLE_CLIENT_SECRET'],
                  scope: 'email,profile'

  config.omniauth :github,
                  ENV['GITHUB_CLIENT_ID'],
                  ENV['GITHUB_CLIENT_SECRET'],
                  scope: 'user:email'
end
```

### モデルの設定

```ruby
# app/models/user.rb
class User < ApplicationRecord
  devise :database_authenticatable, :registerable,
         :recoverable, :rememberable, :validatable,
         :omniauthable, omniauth_providers: [:google_oauth2, :github]

  def self.from_omniauth(auth)
    where(provider: auth.provider, uid: auth.uid).first_or_create do |user|
      user.email = auth.info.email
      user.password = Devise.friendly_token[0, 20]
      user.name = auth.info.name
      user.avatar_url = auth.info.image
    end
  end
end
```

### コールバックコントローラー

```ruby
# app/controllers/users/omniauth_callbacks_controller.rb
class Users::OmniauthCallbacksController < Devise::OmniauthCallbacksController
  def google_oauth2
    handle_auth("Google")
  end

  def github
    handle_auth("GitHub")
  end

  def failure
    redirect_to root_path, alert: "認証に失敗しました"
  end

  private

  def handle_auth(provider)
    @user = User.from_omniauth(request.env["omniauth.auth"])

    if @user.persisted?
      sign_in_and_redirect @user, event: :authentication
      set_flash_message(:notice, :success, kind: provider) if is_navigational_format?
    else
      session["devise.#{provider.downcase}_data"] = request.env["omniauth.auth"].except(:extra)
      redirect_to new_user_registration_url, alert: @user.errors.full_messages.join("\n")
    end
  end
end
```

### ルーティング

```ruby
# config/routes.rb
devise_for :users, controllers: {
  omniauth_callbacks: 'users/omniauth_callbacks'
}
```

### ビューにボタンを追加

```erb
<%# app/views/devise/shared/_links.html.erb %>
<%- if devise_mapping.omniauthable? %>
  <div class="social-login">
    <%- resource_class.omniauth_providers.each do |provider| %>
      <%= button_to "#{provider.to_s.titleize}でログイン",
                    omniauth_authorize_path(resource_name, provider),
                    method: :post,
                    data: { turbo: false },
                    class: "btn btn-#{provider}" %>
    <% end %>
  </div>
<% end %>
```

---

## API 認証

### JWT 認証（devise-jwt）

**🐍 Python:** `devise-jwt` によるAPIトークン認証は、Djangoでは **djangorestframework-simplejwt**（`TokenObtainPairView` / `TokenRefreshView`）、Flaskでは **Flask-JWT-Extended**（`create_access_token` / `@jwt_required`）に相当します。いずれもトークンの発行・検証・有効期限を担い、Deviseの `dispatch_requests` / `revocation_requests` にあたる発行・失効の制御を提供します。

```ruby
# Gemfile
gem 'devise'
gem 'devise-jwt'
```

```ruby
# config/initializers/devise.rb
Devise.setup do |config|
  config.jwt do |jwt|
    jwt.secret = ENV['DEVISE_JWT_SECRET_KEY']
    jwt.dispatch_requests = [
      ['POST', %r{^/api/login$}]
    ]
    jwt.revocation_requests = [
      ['DELETE', %r{^/api/logout$}]
    ]
    jwt.expiration_time = 1.day.to_i
  end
end
```

```ruby
# app/models/user.rb
class User < ApplicationRecord
  include Devise::JWT::RevocationStrategies::JTIMatcher

  devise :database_authenticatable, :registerable,
         :jwt_authenticatable, jwt_revocation_strategy: self
end
```

### API コントローラー

```ruby
# app/controllers/api/base_controller.rb
class Api::BaseController < ActionController::API
  before_action :authenticate_user!

  private

  def authenticate_user!
    unless current_user
      render json: { error: 'Unauthorized' }, status: :unauthorized
    end
  end
end

# app/controllers/api/sessions_controller.rb
class Api::SessionsController < Devise::SessionsController
  respond_to :json

  private

  def respond_with(resource, _opts = {})
    render json: {
      status: { code: 200, message: 'Logged in successfully.' },
      data: UserSerializer.new(resource).as_json
    }
  end

  def respond_to_on_destroy
    if current_user
      render json: {
        status: 200,
        message: 'Logged out successfully.'
      }
    else
      render json: {
        status: 401,
        message: 'Couldn\'t find an active session.'
      }, status: :unauthorized
    end
  end
end
```

---

## テスト

### RSpec でのテスト

```ruby
# spec/rails_helper.rb
RSpec.configure do |config|
  config.include Devise::Test::ControllerHelpers, type: :controller
  config.include Devise::Test::IntegrationHelpers, type: :request
  config.include Devise::Test::IntegrationHelpers, type: :system
end
```

**🐍 Python:** テストでログイン状態を作る `sign_in user` は、Djangoのテストクライアントでは `self.client.force_login(user)`（またはパスワード検証を通す `self.client.login(...)`）、Flaskでは Flask-Login の `login_user()` をテストリクエスト内で呼ぶ形に相当します。

### コントローラーテスト

```ruby
# spec/controllers/articles_controller_spec.rb
RSpec.describe ArticlesController, type: :controller do
  let(:user) { create(:user) }

  describe 'GET #index' do
    context 'when not logged in' do
      it 'redirects to login page' do
        get :index
        expect(response).to redirect_to(new_user_session_path)
      end
    end

    context 'when logged in' do
      before { sign_in user }

      it 'returns success' do
        get :index
        expect(response).to be_successful
      end
    end
  end
end
```

### リクエストスペック

```ruby
# spec/requests/articles_spec.rb
RSpec.describe 'Articles', type: :request do
  let(:user) { create(:user) }

  describe 'GET /articles' do
    context 'when not logged in' do
      it 'redirects to login' do
        get articles_path
        expect(response).to redirect_to(new_user_session_path)
      end
    end

    context 'when logged in' do
      before { sign_in user }

      it 'returns success' do
        get articles_path
        expect(response).to be_successful
      end
    end
  end
end
```

### FactoryBot

```ruby
# spec/factories/users.rb
FactoryBot.define do
  factory :user do
    sequence(:email) { |n| "user#{n}@example.com" }
    password { 'password123' }
    password_confirmation { 'password123' }

    trait :confirmed do
      confirmed_at { Time.current }
    end

    trait :admin do
      admin { true }
    end
  end
end
```

---

## 日本語化

### ロケールファイル

```yaml
# config/locales/devise.ja.yml
ja:
  devise:
    confirmations:
      confirmed: "メールアドレスが確認されました。"
      send_instructions: "アカウントの有効化について数分以内にメールでご連絡します。"
    failure:
      already_authenticated: "すでにログインしています。"
      inactive: "アカウントが有効化されていません。"
      invalid: "%{authentication_keys}またはパスワードが違います。"
      locked: "アカウントはロックされています。"
      not_found_in_database: "%{authentication_keys}またはパスワードが違います。"
      timeout: "セッションがタイムアウトしました。もう一度ログインしてください。"
      unauthenticated: "ログインもしくはアカウント登録してください。"
      unconfirmed: "メールアドレスの本人確認が必要です。"
    mailer:
      confirmation_instructions:
        subject: "メールアドレスの確認"
      reset_password_instructions:
        subject: "パスワードの再設定"
      unlock_instructions:
        subject: "アカウントのロック解除"
    passwords:
      no_token: "パスワード再設定メールからアクセスされた場合のみ有効です。"
      send_instructions: "パスワードの再設定について数分以内にメールでご連絡します。"
      updated: "パスワードが正しく変更されました。"
    registrations:
      destroyed: "アカウントを削除しました。"
      signed_up: "アカウント登録が完了しました。"
      updated: "アカウント情報を変更しました。"
    sessions:
      signed_in: "ログインしました。"
      signed_out: "ログアウトしました。"
    unlocks:
      send_instructions: "アカウントのロック解除方法を数分以内にメールでご連絡します。"
      unlocked: "アカウントをロック解除しました。"
  errors:
    messages:
      already_confirmed: "は既に登録済みです。ログインしてください。"
      confirmation_period_expired: "の期限が切れました。%{period}以内に確認してください。新しくリクエストしてください。"
      expired: "の有効期限が切れました。新しくリクエストしてください。"
      not_found: "は見つかりませんでした。"
      not_locked: "はロックされていません。"
      not_saved:
        one: "エラーが発生したため、%{resource}は保存されませんでした:"
        other: "%{count}件のエラーが発生したため、%{resource}は保存されませんでした:"
```

---

## ベストプラクティス

### セキュリティ

```ruby
# config/initializers/devise.rb
Devise.setup do |config|
  # パラノイドモード（ユーザーの存在を隠す）
  config.paranoid = true

  # パスワードのストレッチング回数
  config.stretches = Rails.env.test? ? 1 : 12

  # セッションタイムアウト
  config.timeout_in = 30.minutes

  # パスワードの最小長
  config.password_length = 12..128

  # リセットトークンの有効期限
  config.reset_password_within = 2.hours

  # アカウントロック
  config.lock_strategy = :failed_attempts
  config.maximum_attempts = 5
  config.unlock_in = 1.hour
end
```

### Strong Parameters

```ruby
class ApplicationController < ActionController::Base
  before_action :configure_permitted_parameters, if: :devise_controller?

  protected

  def configure_permitted_parameters
    # 登録時
    devise_parameter_sanitizer.permit(:sign_up, keys: [:name])

    # 更新時
    devise_parameter_sanitizer.permit(:account_update, keys: [:name, :avatar])
  end
end
```

---

## 関連 Gem

| Gem | 説明 |
|-----|------|
| devise-jwt | JWT認証 |
| devise_invitable | 招待機能 |
| devise-two-factor | 二要素認証 |
| devise_token_auth | トークン認証API |
| devise-security | セキュリティ強化 |
| omniauth | SNS認証 |

---

## 参考リンク

- [Devise GitHub](https://github.com/heartcombo/devise)
- [Devise Wiki](https://github.com/heartcombo/devise/wiki)
- [Devise README](https://github.com/heartcombo/devise#readme)

← [Gem一覧に戻る](README.md)
