# OmniAuth Azure AD - Microsoft Entra ID 連携

## 概要

`omniauth-azure-activedirectory-v2`は、Microsoft Entra ID（旧Azure Active Directory）を使用したOAuth2認証を実現するOmniAuth戦略です。企業のシングルサインオン（SSO）やMicrosoftアカウントでのログインを実装できます。

> **🐍 Python相当:** MSAL for Python（microsoft-authentication-library, msal）。汎用OAuthなら python-social-auth / Authlib、DjangoならallauthのMicrosoftプロバイダ。

**🐍 Python:** OmniAuthはRackミドルウェアとして認可コードフロー全体を代行するのに対し、MSALはアプリケーションオブジェクト（`msal.ConfidentialClientApplication`）を通じてURL生成・トークン交換を明示的に呼び出すライブラリ。ミドルウェアではないため、リダイレクトやコールバックのルーティングはDjango/Flask側で自分で書く。

```bash
# インストール
gem install omniauth-azure-activedirectory-v2

# Gemfile
gem 'omniauth-azure-activedirectory-v2'
gem 'omniauth-rails_csrf_protection'  # Rails使用時は必須
```

---

## Azure Portal での設定

### 1. アプリの登録

1. [Azure Portal](https://portal.azure.com) にアクセス
2. 「Microsoft Entra ID」→「アプリの登録」→「新規登録」
3. 以下を設定：
   - **名前**: アプリケーション名
   - **サポートされているアカウントの種類**:
     - 単一テナント: 自組織のみ
     - マルチテナント: 任意の組織
     - マルチテナント + 個人アカウント: Microsoft アカウント含む
   - **リダイレクトURI**: `https://your-app.com/auth/azure_activedirectory_v2/callback`

### 2. クライアントシークレットの作成

1. 登録したアプリの「証明書とシークレット」
2. 「新しいクライアントシークレット」をクリック
3. 説明と有効期限を設定して追加
4. **値をコピー**（一度しか表示されない）

### 3. 必要な情報

- **アプリケーション（クライアント）ID**: 概要ページに表示
- **ディレクトリ（テナント）ID**: 概要ページに表示
- **クライアントシークレット**: 上記で作成した値

---

## 基本設定

### Rails での設定

```ruby
# config/initializers/omniauth.rb
Rails.application.config.middleware.use OmniAuth::Builder do
  provider :azure_activedirectory_v2,
    client_id: ENV['AZURE_CLIENT_ID'],
    client_secret: ENV['AZURE_CLIENT_SECRET'],
    tenant_id: ENV['AZURE_TENANT_ID']
end

# CSRF対策
OmniAuth.config.allowed_request_methods = [:post, :get]
```

**🐍 Python:** MSALでは同じ設定値（client_id / client_secret / tenant）を`ConfidentialClientApplication`のコンストラクタに渡し、認可URLを組み立てる。`tenant_id`はauthorityのURLとして表現される。

```python
import msal

app = msal.ConfidentialClientApplication(
    client_id="AZURE_CLIENT_ID",
    client_credential="AZURE_CLIENT_SECRET",
    authority="https://login.microsoftonline.com/AZURE_TENANT_ID",  # 'common'/'consumers'も可
)

# 認可コードフローの開始（OmniAuthの /auth/... 相当）
auth_url = app.get_authorization_request_url(
    scopes=["User.Read"],
    redirect_uri="https://your-app.com/auth/callback",
)
```

### 環境変数

```bash
# .env
AZURE_CLIENT_ID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
AZURE_CLIENT_SECRET=your-client-secret
AZURE_TENANT_ID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
```

### テナントオプション

```ruby
# 単一テナント（自組織のみ）
provider :azure_activedirectory_v2,
  client_id: ENV['AZURE_CLIENT_ID'],
  client_secret: ENV['AZURE_CLIENT_SECRET'],
  tenant_id: ENV['AZURE_TENANT_ID']

# マルチテナント（任意の組織）
provider :azure_activedirectory_v2,
  client_id: ENV['AZURE_CLIENT_ID'],
  client_secret: ENV['AZURE_CLIENT_SECRET'],
  tenant_id: 'common'

# 個人アカウントも含む
provider :azure_activedirectory_v2,
  client_id: ENV['AZURE_CLIENT_ID'],
  client_secret: ENV['AZURE_CLIENT_SECRET'],
  tenant_id: 'consumers'  # 個人のみ
  # または 'common' でマルチテナント+個人
```

---

## Devise との連携

> **🐍 Python相当:** MSALを直接使う代わりに、フレームワーク統合ライブラリでDevise + OmniAuthのような「ボタン一つでSSO」を実現できる。Djangoなら**django-allauth**（Microsoftプロバイダ）や**python-social-auth**（`social-auth-app-django`）、Flaskなら**Flask-Dance**。汎用OIDCクライアントとして**Authlib**も利用可能。

### 設定

```ruby
# config/initializers/devise.rb
Devise.setup do |config|
  config.omniauth :azure_activedirectory_v2,
    client_id: ENV['AZURE_CLIENT_ID'],
    client_secret: ENV['AZURE_CLIENT_SECRET'],
    tenant_id: ENV['AZURE_TENANT_ID']
end
```

### User モデル

```ruby
# app/models/user.rb
class User < ApplicationRecord
  devise :database_authenticatable, :registerable,
         :recoverable, :rememberable, :validatable,
         :omniauthable, omniauth_providers: [:azure_activedirectory_v2]

  def self.from_azure_oauth(auth)
    where(provider: auth.provider, uid: auth.uid).first_or_create do |user|
      user.email = auth.info.email
      user.password = Devise.friendly_token[0, 20]
      user.name = auth.info.name
      # Azure AD固有の情報
      user.azure_id = auth.uid
      user.tenant_id = auth.extra.raw_info.tid
    end
  end

  def self.from_omniauth(auth)
    case auth.provider
    when 'azure_activedirectory_v2'
      from_azure_oauth(auth)
    else
      # 他のプロバイダー
    end
  end
end
```

### マイグレーション

```ruby
# db/migrate/xxx_add_azure_fields_to_users.rb
class AddAzureFieldsToUsers < ActiveRecord::Migration[7.0]
  def change
    add_column :users, :provider, :string
    add_column :users, :uid, :string
    add_column :users, :azure_id, :string
    add_column :users, :tenant_id, :string

    add_index :users, [:provider, :uid], unique: true
  end
end
```

### コールバックコントローラー

```ruby
# app/controllers/users/omniauth_callbacks_controller.rb
class Users::OmniauthCallbacksController < Devise::OmniauthCallbacksController
  def azure_activedirectory_v2
    @user = User.from_omniauth(request.env['omniauth.auth'])

    if @user.persisted?
      sign_in_and_redirect @user, event: :authentication
      set_flash_message(:notice, :success, kind: 'Microsoft') if is_navigational_format?
    else
      session['devise.azure_data'] = request.env['omniauth.auth'].except(:extra)
      redirect_to new_user_registration_url, alert: @user.errors.full_messages.join("\n")
    end
  end

  def failure
    redirect_to root_path, alert: "認証に失敗しました: #{failure_message}"
  end
end
```

### ルーティング

```ruby
# config/routes.rb
Rails.application.routes.draw do
  devise_for :users, controllers: {
    omniauth_callbacks: 'users/omniauth_callbacks'
  }
end
```

### ビュー

```erb
<%# app/views/devise/sessions/new.html.erb %>
<h2>ログイン</h2>

<%= form_for(resource, as: resource_name, url: session_path(resource_name)) do |f| %>
  <div class="field">
    <%= f.label :email %>
    <%= f.email_field :email %>
  </div>

  <div class="field">
    <%= f.label :password %>
    <%= f.password_field :password %>
  </div>

  <div class="actions">
    <%= f.submit "ログイン" %>
  </div>
<% end %>

<hr>

<div class="social-login">
  <h3>または</h3>
  <%= button_to "Microsoftアカウントでログイン",
                user_azure_activedirectory_v2_omniauth_authorize_path,
                method: :post,
                data: { turbo: false },
                class: "btn btn-microsoft" %>
</div>
```

---

## スコープとアクセス許可

### 基本スコープ

```ruby
provider :azure_activedirectory_v2,
  client_id: ENV['AZURE_CLIENT_ID'],
  client_secret: ENV['AZURE_CLIENT_SECRET'],
  tenant_id: ENV['AZURE_TENANT_ID'],
  scope: 'openid profile email'
```

### 追加スコープ

```ruby
# Microsoft Graph API へのアクセス
provider :azure_activedirectory_v2,
  client_id: ENV['AZURE_CLIENT_ID'],
  client_secret: ENV['AZURE_CLIENT_SECRET'],
  tenant_id: ENV['AZURE_TENANT_ID'],
  scope: 'openid profile email User.Read Calendars.Read'
```

### 主なスコープ一覧

| スコープ | 説明 |
|---------|------|
| openid | OpenID Connect認証 |
| profile | ユーザーの基本プロフィール |
| email | メールアドレス |
| User.Read | ユーザー情報の読み取り |
| User.ReadBasic.All | 組織内ユーザーの基本情報 |
| Calendars.Read | カレンダーの読み取り |
| Mail.Read | メールの読み取り |
| offline_access | リフレッシュトークン取得 |

---

## 認証レスポンス

### auth ハッシュの構造

```ruby
# request.env['omniauth.auth'] の内容
{
  provider: 'azure_activedirectory_v2',
  uid: 'xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx',
  info: {
    name: '山田 太郎',
    email: 'taro.yamada@company.com',
    nickname: 'taro.yamada',
    first_name: '太郎',
    last_name: '山田'
  },
  credentials: {
    token: 'アクセストークン',
    refresh_token: 'リフレッシュトークン',  # offline_access スコープ時
    expires_at: 1234567890,
    expires: true
  },
  extra: {
    raw_info: {
      aud: 'クライアントID',
      iss: 'https://login.microsoftonline.com/テナントID/v2.0',
      iat: 1234567890,
      nbf: 1234567890,
      exp: 1234567890,
      name: '山田 太郎',
      oid: 'オブジェクトID',
      preferred_username: 'taro.yamada@company.com',
      sub: 'サブジェクト',
      tid: 'テナントID',
      ver: '2.0'
    }
  }
}
```

### 情報の取得

```ruby
def azure_activedirectory_v2
  auth = request.env['omniauth.auth']

  # 基本情報
  name = auth.info.name
  email = auth.info.email
  uid = auth.uid

  # Azure AD固有情報
  tenant_id = auth.extra.raw_info.tid
  object_id = auth.extra.raw_info.oid

  # トークン（API呼び出し用）
  access_token = auth.credentials.token
  refresh_token = auth.credentials.refresh_token
  expires_at = Time.at(auth.credentials.expires_at)

  # ...
end
```

**🐍 Python:** MSALではコールバックで受け取った認可コードを`acquire_token_by_authorization_code`でトークンに交換する。`request.env["omniauth.auth"]`のinfo/extraに相当する情報は、応答の`id_token_claims`（`name` / `preferred_username` / `oid` / `tid` など）と`access_token`から取り出す。

```python
result = app.acquire_token_by_authorization_code(
    code=request.GET["code"],          # コールバックの認可コード
    scopes=["User.Read"],
    redirect_uri="https://your-app.com/auth/callback",
)

claims = result["id_token_claims"]     # auth.extra.raw_info 相当
name = claims["name"]
email = claims.get("preferred_username")
uid = claims["oid"]                    # auth.uid 相当
tenant_id = claims["tid"]
access_token = result["access_token"]  # auth.credentials.token 相当
```

---

## Microsoft Graph API の利用

### アクセストークンの保存

```ruby
# app/models/user.rb
class User < ApplicationRecord
  def self.from_azure_oauth(auth)
    user = where(provider: auth.provider, uid: auth.uid).first_or_initialize

    user.assign_attributes(
      email: auth.info.email,
      name: auth.info.name,
      azure_token: auth.credentials.token,
      azure_refresh_token: auth.credentials.refresh_token,
      azure_token_expires_at: Time.at(auth.credentials.expires_at)
    )

    user.password = Devise.friendly_token[0, 20] if user.new_record?
    user.save!
    user
  end

  def azure_token_expired?
    azure_token_expires_at < Time.current
  end
end
```

### Graph API の呼び出し

```ruby
# app/services/microsoft_graph_service.rb
require 'faraday'

class MicrosoftGraphService
  BASE_URL = 'https://graph.microsoft.com/v1.0'

  def initialize(user)
    @user = user
    @token = user.azure_token
  end

  def me
    get('/me')
  end

  def profile_photo
    response = connection.get('/me/photo/$value')
    response.body if response.success?
  end

  def calendar_events(start_time: Time.current, end_time: 1.week.from_now)
    get('/me/calendarview', {
      startDateTime: start_time.iso8601,
      endDateTime: end_time.iso8601
    })
  end

  def send_mail(to:, subject:, body:)
    post('/me/sendMail', {
      message: {
        subject: subject,
        body: { contentType: 'Text', content: body },
        toRecipients: [{ emailAddress: { address: to } }]
      }
    })
  end

  private

  def connection
    @connection ||= Faraday.new(url: BASE_URL) do |f|
      f.request :json
      f.response :json
      f.headers['Authorization'] = "Bearer #{@token}"
    end
  end

  def get(path, params = {})
    response = connection.get(path, params)
    handle_response(response)
  end

  def post(path, body)
    response = connection.post(path, body)
    handle_response(response)
  end

  def handle_response(response)
    case response.status
    when 200..299
      response.body
    when 401
      raise AuthenticationError, 'Token expired or invalid'
    else
      raise ApiError, response.body['error']['message']
    end
  end

  class AuthenticationError < StandardError; end
  class ApiError < StandardError; end
end
```

**🐍 Python:** MSALはトークン取得までを担当し、Graph API呼び出し自体は`requests`などのHTTPクライアントでBearerトークンを付与して行う（Faradayでの呼び出しに相当）。Microsoft Graph SDK for Python（`msgraph-sdk`）を使う選択肢もある。

```python
import requests

headers = {"Authorization": f"Bearer {access_token}"}
me = requests.get("https://graph.microsoft.com/v1.0/me", headers=headers).json()
```

### 使用例

```ruby
# コントローラーでの使用
class ProfileController < ApplicationController
  def show
    graph = MicrosoftGraphService.new(current_user)
    @profile = graph.me
    @events = graph.calendar_events
  rescue MicrosoftGraphService::AuthenticationError
    # トークンの更新が必要
    redirect_to user_azure_activedirectory_v2_omniauth_authorize_path
  end
end
```

---

## トークンの更新

### リフレッシュトークンの使用

```ruby
# app/services/azure_token_refresh_service.rb
class AzureTokenRefreshService
  TOKEN_URL = "https://login.microsoftonline.com/%s/oauth2/v2.0/token"

  def initialize(user)
    @user = user
  end

  def refresh!
    return unless @user.azure_refresh_token.present?

    response = Faraday.post(token_url, {
      client_id: ENV['AZURE_CLIENT_ID'],
      client_secret: ENV['AZURE_CLIENT_SECRET'],
      refresh_token: @user.azure_refresh_token,
      grant_type: 'refresh_token',
      scope: 'openid profile email User.Read offline_access'
    })

    if response.success?
      data = JSON.parse(response.body)
      @user.update!(
        azure_token: data['access_token'],
        azure_refresh_token: data['refresh_token'],
        azure_token_expires_at: Time.current + data['expires_in'].seconds
      )
      true
    else
      false
    end
  end

  private

  def token_url
    TOKEN_URL % @user.tenant_id
  end
end
```

**🐍 Python:** MSALはトークンキャッシュを内蔵しており、リフレッシュを手動でエンドポイントに投げる代わりに`acquire_token_silent`を呼ぶと、キャッシュされたリフレッシュトークンを使って自動的に更新する。

```python
accounts = app.get_accounts()
result = app.acquire_token_silent(scopes=["User.Read"], account=accounts[0])
# result が None または "access_token" を含まない場合は再認証が必要
access_token = result["access_token"]
```

### 自動更新

```ruby
# app/models/user.rb
class User < ApplicationRecord
  def ensure_fresh_azure_token!
    return azure_token unless azure_token_expired?

    if AzureTokenRefreshService.new(self).refresh!
      reload.azure_token
    else
      raise TokenRefreshError, 'Failed to refresh token'
    end
  end

  class TokenRefreshError < StandardError; end
end
```

---

## グループとロール

### グループクレームの有効化

Azure Portal で設定：

1. アプリの登録 → トークン構成
2. 「グループ要求を追加」
3. 「セキュリティグループ」を選択

### グループ情報の取得

```ruby
# 設定
provider :azure_activedirectory_v2,
  client_id: ENV['AZURE_CLIENT_ID'],
  client_secret: ENV['AZURE_CLIENT_SECRET'],
  tenant_id: ENV['AZURE_TENANT_ID'],
  scope: 'openid profile email GroupMember.Read.All'
```

```ruby
# コールバックでグループを取得
def azure_activedirectory_v2
  auth = request.env['omniauth.auth']
  groups = auth.extra.raw_info.groups || []

  @user = User.from_azure_oauth(auth)
  @user.update_azure_groups(groups)

  # ...
end
```

### ロールベースアクセス制御

```ruby
# app/models/user.rb
class User < ApplicationRecord
  ADMIN_GROUP_ID = 'xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx'
  MANAGER_GROUP_ID = 'yyyyyyyy-yyyy-yyyy-yyyy-yyyyyyyyyyyy'

  def admin?
    azure_groups.include?(ADMIN_GROUP_ID)
  end

  def manager?
    azure_groups.include?(MANAGER_GROUP_ID)
  end

  def update_azure_groups(group_ids)
    update!(azure_groups: group_ids)
  end
end
```

---

## シングルサインアウト

### ログアウト処理

```ruby
# app/controllers/users/sessions_controller.rb
class Users::SessionsController < Devise::SessionsController
  def destroy
    # Azure ADからもログアウト
    azure_logout_url = build_azure_logout_url if current_user&.provider == 'azure_activedirectory_v2'

    super

    redirect_to azure_logout_url, allow_other_host: true if azure_logout_url
  end

  private

  def build_azure_logout_url
    tenant_id = ENV['AZURE_TENANT_ID']
    post_logout_uri = CGI.escape(root_url)

    "https://login.microsoftonline.com/#{tenant_id}/oauth2/v2.0/logout" \
    "?post_logout_redirect_uri=#{post_logout_uri}"
  end
end
```

---

## セキュリティ

### 推奨設定

```ruby
# config/initializers/omniauth.rb
OmniAuth.config.allowed_request_methods = [:post]  # GETを無効化
OmniAuth.config.silence_get_warning = false

Rails.application.config.middleware.use OmniAuth::Builder do
  provider :azure_activedirectory_v2,
    client_id: ENV['AZURE_CLIENT_ID'],
    client_secret: ENV['AZURE_CLIENT_SECRET'],
    tenant_id: ENV['AZURE_TENANT_ID'],
    callback_path: '/auth/azure_activedirectory_v2/callback',
    authorize_params: {
      prompt: 'select_account'  # 常にアカウント選択を表示
    }
end
```

### state パラメータ

```ruby
# カスタムstate（CSRF対策は自動）
provider :azure_activedirectory_v2,
  client_id: ENV['AZURE_CLIENT_ID'],
  client_secret: ENV['AZURE_CLIENT_SECRET'],
  tenant_id: ENV['AZURE_TENANT_ID'],
  provider_ignores_state: false  # デフォルトで有効
```

---

## トラブルシューティング

### よくあるエラー

```ruby
# AADSTS50011: リダイレクトURIが一致しない
# → Azure Portalでリダイレクトを正確に設定

# AADSTS700016: アプリが見つからない
# → テナントIDとクライアントIDを確認

# AADSTS7000218: クライアントシークレットが無効
# → シークレットの有効期限を確認
```

### デバッグ

```ruby
# コールバックで詳細を確認
def azure_activedirectory_v2
  auth = request.env['omniauth.auth']
  Rails.logger.debug "Auth hash: #{auth.to_json}"

  error = request.env['omniauth.error']
  Rails.logger.error "Auth error: #{error}" if error

  # ...
end
```

---

## Sinatra での使用

```ruby
# app.rb
require 'sinatra/base'
require 'omniauth-azure-activedirectory-v2'

class MyApp < Sinatra::Base
  enable :sessions
  set :session_secret, ENV['SESSION_SECRET']

  use OmniAuth::Builder do
    provider :azure_activedirectory_v2,
      client_id: ENV['AZURE_CLIENT_ID'],
      client_secret: ENV['AZURE_CLIENT_SECRET'],
      tenant_id: ENV['AZURE_TENANT_ID']
  end

  get '/auth/:provider/callback' do
    auth = request.env['omniauth.auth']
    session[:user_id] = auth.uid
    session[:user_name] = auth.info.name
    redirect '/'
  end

  get '/auth/failure' do
    "認証エラー: #{params[:message]}"
  end

  get '/login' do
    redirect '/auth/azure_activedirectory_v2'
  end

  get '/logout' do
    session.clear
    redirect '/'
  end

  get '/' do
    if session[:user_id]
      "Welcome, #{session[:user_name]}!"
    else
      '<a href="/login">Microsoftでログイン</a>'
    end
  end
end
```

---

## 関連 Gem

| Gem | 説明 |
|-----|------|
| omniauth | OmniAuthコア |
| omniauth-rails_csrf_protection | CSRF対策 |
| devise | Rails認証 |
| faraday | HTTP クライアント（Graph API用） |
| microsoft_graph | Graph API クライアント |

---

## 参考リンク

- [omniauth-azure-activedirectory-v2 GitHub](https://github.com/RIPAGlobal/omniauth-azure-activedirectory-v2)
- [Microsoft Entra ID ドキュメント](https://learn.microsoft.com/ja-jp/entra/identity/)
- [Microsoft Graph API リファレンス](https://learn.microsoft.com/ja-jp/graph/api/overview)
- [Azure Portal](https://portal.azure.com)

← [Gem一覧に戻る](README.md)
