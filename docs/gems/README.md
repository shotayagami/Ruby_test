# よく使われるGem 解説

このディレクトリでは、Ruby開発で頻繁に使用される重要なGemについて詳しく解説します。

## 目次

### テスト・品質管理
| Gem | 説明 | ドキュメント |
|-----|------|-------------|
| RSpec | BDDテストフレームワーク | [rspec.md](rspec.md) |
| RuboCop | Linter/Formatter | [rubocop.md](rubocop.md) |

### 開発ツール
| Gem | 説明 | ドキュメント |
|-----|------|-------------|
| Pry | 高機能REPL・デバッガ | [pry.md](pry.md) |
| Rake | タスクランナー | [rake.md](rake.md) |

### Web・API
| Gem | 説明 | ドキュメント |
|-----|------|-------------|
| Sinatra | 軽量Webフレームワーク | [sinatra.md](sinatra.md) |
| Faraday | HTTPクライアント | [faraday.md](faraday.md) |
| Nokogiri | HTML/XMLパーサー | [nokogiri.md](nokogiri.md) |

### データベース・ORM
| Gem | 説明 | ドキュメント |
|-----|------|-------------|
| ActiveRecord | ORM（単体使用） | [activerecord.md](activerecord.md) |

### 認証・セキュリティ
| Gem | 説明 | ドキュメント |
|-----|------|-------------|
| Devise | 認証ソリューション | [devise.md](devise.md) |
| OmniAuth Azure AD | Microsoft Entra ID連携 | [omniauth-azure-ad.md](omniauth-azure-ad.md) |

### バックグラウンドジョブ
| Gem | 説明 | ドキュメント |
|-----|------|-------------|
| Sidekiq | 非同期ジョブ処理 | [sidekiq.md](sidekiq.md) |

### クラウドストレージ
| Gem | 説明 | ドキュメント |
|-----|------|-------------|
| Cloudflare R2 | S3互換オブジェクトストレージ | [cloudflare-r2.md](cloudflare-r2.md) |

---

## Gemの選定基準

### 1. メンテナンス状況
- 最終更新日
- Issue/PRの対応状況
- メンテナの活動状況

### 2. コミュニティ
- GitHubスター数
- ダウンロード数
- ドキュメントの充実度

### 3. 依存関係
- 依存Gemの数
- セキュリティ脆弱性

### 確認コマンド
```bash
# Gemの情報確認
gem info gem_name

# 依存関係の確認
gem dependency gem_name

# 脆弱性チェック
bundle audit check --update
```

---

## カテゴリ別おすすめGem一覧

### テスト関連
```ruby
group :test do
  gem 'rspec'           # テストフレームワーク
  gem 'factory_bot'     # テストデータ生成
  gem 'faker'           # ダミーデータ生成
  gem 'webmock'         # HTTPモック
  gem 'vcr'             # HTTPレスポンス記録
  gem 'simplecov'       # コードカバレッジ
  gem 'timecop'         # 時間操作
end
```

### コード品質
```ruby
group :development do
  gem 'rubocop'         # Linter
  gem 'rubocop-rspec'   # RSpec用ルール
  gem 'rubocop-performance'  # パフォーマンス
  gem 'brakeman'        # セキュリティスキャン
  gem 'reek'            # コードの匂い検出
end
```

### デバッグ・開発
```ruby
group :development, :test do
  gem 'pry'             # 高機能REPL
  gem 'pry-byebug'      # デバッガ
  gem 'awesome_print'   # 見やすい出力
  gem 'dotenv'          # 環境変数管理
end
```

### HTTP・API
```ruby
gem 'faraday'           # HTTPクライアント
gem 'httparty'          # シンプルなHTTP
gem 'rest-client'       # REST API
gem 'oauth2'            # OAuth2クライアント
gem 'jwt'               # JWT処理
```

### データ処理
```ruby
gem 'nokogiri'          # HTML/XMLパース
gem 'oj'                # 高速JSON
gem 'csv'               # CSV処理（標準）
gem 'rubyzip'           # ZIP処理
gem 'roo'               # Excel読み込み
```

### データベース
```ruby
gem 'pg'                # PostgreSQL
gem 'mysql2'            # MySQL
gem 'sqlite3'           # SQLite
gem 'redis'             # Redis
gem 'sequel'            # 軽量ORM
```

### バックグラウンド処理
```ruby
gem 'sidekiq'           # Redis使用
gem 'delayed_job'       # DB使用
gem 'resque'            # Redis使用
gem 'good_job'          # PostgreSQL使用
```

### ユーティリティ
```ruby
gem 'activesupport'     # Rails拡張（単体使用可）
gem 'dry-rb'            # 関数型ツール群
gem 'hashie'            # ハッシュ拡張
gem 'chronic'           # 自然言語日時パース
gem 'money'             # 通貨処理
```

---

## Python相当ライブラリ早見表

本ディレクトリで扱う各Gemに対応する、代表的なPythonライブラリの対応表です。

| Gem | 用途 | Python相当 |
|-----|------|-----------|
| RSpec / Minitest | テストフレームワーク | pytest / unittest |
| RuboCop | Linter/Formatter | Ruff（旧来の flake8 + black） |
| Pry | 高機能REPL・デバッガ | IPython + pdb / ipdb |
| Rake | タスクランナー | invoke / nox / Make |
| Sinatra | 軽量Webフレームワーク | Flask |
| Faraday | HTTPクライアント | requests / httpx |
| Nokogiri | HTML/XMLパーサー | BeautifulSoup / lxml |
| ActiveRecord | ORM | SQLAlchemy（Django ORM） |
| Devise | 認証ソリューション | django.contrib.auth / django-allauth / Flask-Login |
| OmniAuth Azure AD | Microsoft Entra ID連携 | MSAL for Python（msal）/ python-social-auth |
| Sidekiq | 非同期ジョブ処理 | Celery / RQ |
| Cloudflare R2 | S3互換オブジェクトストレージ | boto3（S3互換API） |

対応はあくまで用途ベースの目安で、機能や設計思想は完全一致しません。特にRailsのActiveRecord/DeviseはフルスタックのRailsに密結合している一方、Python側は用途に応じてフレームワーク（Django / Flask）やライブラリを選択する点が異なります。

---

← [第10章: Gem に戻る](../10_Gem.md)
