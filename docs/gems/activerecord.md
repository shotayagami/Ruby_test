# ActiveRecord - ORM（単体使用）

## 概要

ActiveRecordはRuby on Railsに含まれるORM（Object-Relational Mapping）ですが、単体でも使用可能です。データベースとRubyオブジェクトを結びつけ、SQLを書かずにデータ操作ができます。

```bash
# インストール
gem install activerecord

# Gemfile
gem 'activerecord'

# データベースアダプタ（いずれか）
gem 'sqlite3'
gem 'pg'
gem 'mysql2'
```

---

## セットアップ（Rails以外）

### 基本的な接続

```ruby
require 'active_record'

# SQLite
ActiveRecord::Base.establish_connection(
  adapter: 'sqlite3',
  database: 'db/development.sqlite3'
)

# PostgreSQL
ActiveRecord::Base.establish_connection(
  adapter: 'postgresql',
  host: 'localhost',
  database: 'myapp_development',
  username: 'postgres',
  password: 'password'
)

# MySQL
ActiveRecord::Base.establish_connection(
  adapter: 'mysql2',
  host: 'localhost',
  database: 'myapp_development',
  username: 'root',
  password: 'password'
)
```

### 環境変数から接続

```ruby
# DATABASE_URL環境変数を使用
ActiveRecord::Base.establish_connection(ENV['DATABASE_URL'])

# または詳細設定
ActiveRecord::Base.establish_connection(
  adapter: ENV.fetch('DB_ADAPTER', 'postgresql'),
  host: ENV.fetch('DB_HOST', 'localhost'),
  database: ENV.fetch('DB_NAME', 'myapp'),
  username: ENV.fetch('DB_USER', 'postgres'),
  password: ENV.fetch('DB_PASSWORD', ''),
  pool: ENV.fetch('DB_POOL', 5).to_i
)
```

### ロガー設定

```ruby
# SQLログを出力
ActiveRecord::Base.logger = Logger.new(STDOUT)

# ファイルに出力
ActiveRecord::Base.logger = Logger.new('log/database.log')

# ログレベル設定
ActiveRecord::Base.logger.level = Logger::DEBUG
```

---

## マイグレーション

### マイグレーションファイルの作成

```ruby
# db/migrate/001_create_users.rb
class CreateUsers < ActiveRecord::Migration[7.0]
  def change
    create_table :users do |t|
      t.string :name, null: false
      t.string :email, null: false
      t.integer :age
      t.boolean :active, default: true
      t.timestamps
    end

    add_index :users, :email, unique: true
  end
end

# db/migrate/002_create_posts.rb
class CreatePosts < ActiveRecord::Migration[7.0]
  def change
    create_table :posts do |t|
      t.references :user, null: false, foreign_key: true
      t.string :title, null: false
      t.text :body
      t.datetime :published_at
      t.timestamps
    end

    add_index :posts, :published_at
  end
end
```

### マイグレーションの実行（Rails以外）

```ruby
# スタンドアロンでの実行
require 'active_record'

ActiveRecord::Base.establish_connection(
  adapter: 'sqlite3',
  database: 'db/development.sqlite3'
)

# マイグレーションの実行
ActiveRecord::MigrationContext.new('db/migrate').migrate

# 特定バージョンまで
ActiveRecord::MigrationContext.new('db/migrate').migrate(2)

# ロールバック
ActiveRecord::MigrationContext.new('db/migrate').rollback
```

### Rake タスクを作成

```ruby
# Rakefile
require 'active_record'
require 'rake'

namespace :db do
  desc 'Create database'
  task :create do
    # データベース作成処理
  end

  desc 'Run migrations'
  task :migrate do
    ActiveRecord::Base.establish_connection(database_config)
    ActiveRecord::MigrationContext.new('db/migrate').migrate
    puts "Migrated to version #{ActiveRecord::Migrator.current_version}"
  end

  desc 'Rollback migration'
  task :rollback do
    ActiveRecord::Base.establish_connection(database_config)
    ActiveRecord::MigrationContext.new('db/migrate').rollback
  end

  desc 'Drop database'
  task :drop do
    # データベース削除処理
  end
end

def database_config
  {
    adapter: 'sqlite3',
    database: 'db/development.sqlite3'
  }
end
```

---

## モデル定義

### 基本的なモデル

```ruby
class User < ActiveRecord::Base
  # テーブル名は自動的に users
end

class Post < ActiveRecord::Base
  # テーブル名は自動的に posts
end
```

### テーブル名のカスタマイズ

```ruby
class User < ActiveRecord::Base
  self.table_name = 'app_users'
end

class LegacyPost < ActiveRecord::Base
  self.table_name = 'old_posts'
  self.primary_key = 'post_id'
end
```

---

## CRUD操作

### Create（作成）

```ruby
# new + save
user = User.new(name: 'Alice', email: 'alice@example.com')
user.save

# create（new + save を一度に）
user = User.create(name: 'Bob', email: 'bob@example.com')

# create!（バリデーション失敗で例外）
user = User.create!(name: 'Charlie', email: 'charlie@example.com')

# 複数レコード作成
User.create([
  { name: 'Dave', email: 'dave@example.com' },
  { name: 'Eve', email: 'eve@example.com' }
])

# find_or_create_by
user = User.find_or_create_by(email: 'alice@example.com') do |u|
  u.name = 'Alice'
end

# first_or_create
user = User.where(email: 'alice@example.com').first_or_create(name: 'Alice')
```

### Read（読み取り）

```ruby
# 単一レコード
user = User.find(1)                    # ID指定（見つからないと例外）
user = User.find_by(email: 'alice@example.com')  # 条件指定（見つからないとnil）
user = User.find_by!(email: 'alice@example.com') # 見つからないと例外

# 最初/最後
user = User.first
user = User.last
user = User.first(3)                   # 最初の3件
user = User.last(3)                    # 最後の3件

# 全件
users = User.all

# 条件検索
users = User.where(active: true)
users = User.where('age > ?', 18)
users = User.where('age > :age AND name LIKE :name', age: 18, name: '%Alice%')
users = User.where(age: 18..30)        # BETWEEN
users = User.where(age: [18, 20, 25])  # IN
users = User.where.not(active: false)  # NOT

# 並び替え
users = User.order(:name)              # ASC
users = User.order(name: :desc)        # DESC
users = User.order(created_at: :desc, name: :asc)

# 制限
users = User.limit(10)
users = User.limit(10).offset(20)

# 選択
users = User.select(:id, :name)
users = User.select('id, name, email')

# 重複除去
emails = User.select(:email).distinct

# グループ化
counts = User.group(:age).count
```

### Update（更新）

```ruby
# 単一レコード
user = User.find(1)
user.name = 'Alice Updated'
user.save

# update メソッド
user.update(name: 'Alice Updated', age: 26)
user.update!(name: 'Alice Updated')    # バリデーション失敗で例外

# 複数レコード
User.where(active: false).update_all(active: true)
User.where('created_at < ?', 1.year.ago).update_all(archived: true)

# インクリメント/デクリメント
user.increment!(:login_count)
user.decrement!(:points, 10)

# 特定カラムのみ更新（バリデーションをスキップ）
user.update_column(:login_count, 5)
user.update_columns(login_count: 5, last_login_at: Time.current)
```

### Delete（削除）

```ruby
# 単一レコード
user = User.find(1)
user.destroy                           # コールバック実行
user.delete                            # コールバックなし

# 条件指定で削除
User.where(active: false).destroy_all  # コールバック実行
User.where(active: false).delete_all   # コールバックなし

# ID指定
User.destroy(1)
User.destroy([1, 2, 3])
```

---

## クエリインターフェース

### チェーン

```ruby
# メソッドチェーン
users = User
  .where(active: true)
  .where('age > ?', 18)
  .order(created_at: :desc)
  .limit(10)

# スコープの組み合わせ
class User < ActiveRecord::Base
  scope :active, -> { where(active: true) }
  scope :adult, -> { where('age >= ?', 18) }
  scope :recent, -> { order(created_at: :desc) }
end

users = User.active.adult.recent.limit(10)
```

### 集計

```ruby
User.count                             # 件数
User.where(active: true).count

User.average(:age)                     # 平均
User.maximum(:age)                     # 最大
User.minimum(:age)                     # 最小
User.sum(:points)                      # 合計

User.group(:country).count             # グループ別カウント
User.group(:country).average(:age)     # グループ別平均
```

### 存在確認

```ruby
User.exists?(1)                        # ID指定
User.exists?(email: 'alice@example.com')  # 条件指定
User.where(active: true).exists?

User.any?                              # 1件以上存在
User.none?                             # 0件
User.empty?                            # 0件
```

### プラック

```ruby
# 単一カラム
emails = User.pluck(:email)            # => ['alice@...', 'bob@...']

# 複数カラム
data = User.pluck(:id, :name)          # => [[1, 'Alice'], [2, 'Bob']]

# IDs
ids = User.ids                         # => [1, 2, 3]
```

### バッチ処理

```ruby
# find_each（1000件ずつ処理）
User.find_each do |user|
  user.send_newsletter
end

# バッチサイズ指定
User.find_each(batch_size: 500) do |user|
  user.process
end

# find_in_batches（配列で取得）
User.find_in_batches(batch_size: 1000) do |users|
  users.each { |user| user.process }
end

# in_batches（Relationで取得）
User.in_batches(of: 1000) do |relation|
  relation.update_all(processed: true)
end
```

---

## アソシエーション

### 1対多（has_many / belongs_to）

```ruby
class User < ActiveRecord::Base
  has_many :posts, dependent: :destroy
end

class Post < ActiveRecord::Base
  belongs_to :user
end

# 使用
user = User.find(1)
user.posts                             # ユーザーの投稿一覧
user.posts.create(title: 'New Post')   # 関連付けて作成

post = Post.find(1)
post.user                              # 投稿者
```

### 1対1（has_one / belongs_to）

```ruby
class User < ActiveRecord::Base
  has_one :profile, dependent: :destroy
end

class Profile < ActiveRecord::Base
  belongs_to :user
end

# 使用
user.profile
user.create_profile(bio: 'Hello')
```

### 多対多（has_many :through）

```ruby
class User < ActiveRecord::Base
  has_many :memberships
  has_many :groups, through: :memberships
end

class Group < ActiveRecord::Base
  has_many :memberships
  has_many :users, through: :memberships
end

class Membership < ActiveRecord::Base
  belongs_to :user
  belongs_to :group
end

# 使用
user.groups
user.groups << group
group.users
```

### 多対多（has_and_belongs_to_many）

```ruby
class Post < ActiveRecord::Base
  has_and_belongs_to_many :tags
end

class Tag < ActiveRecord::Base
  has_and_belongs_to_many :posts
end

# 中間テーブル（マイグレーション）
create_table :posts_tags, id: false do |t|
  t.belongs_to :post
  t.belongs_to :tag
end

# 使用
post.tags
post.tags << tag
tag.posts
```

### アソシエーションオプション

```ruby
class User < ActiveRecord::Base
  # 依存関係
  has_many :posts, dependent: :destroy     # 削除時に関連も削除
  has_many :comments, dependent: :nullify  # 外部キーをnullに

  # 条件付き
  has_many :published_posts, -> { where(published: true) }, class_name: 'Post'

  # 並び順
  has_many :posts, -> { order(created_at: :desc) }

  # カウンターキャッシュ
  has_many :posts, counter_cache: true  # users.posts_count を自動更新
end

class Post < ActiveRecord::Base
  belongs_to :user, counter_cache: true

  # オプショナル
  belongs_to :category, optional: true

  # カスタム外部キー
  belongs_to :author, class_name: 'User', foreign_key: 'author_id'
end
```

### Eager Loading

```ruby
# N+1問題を解決
users = User.includes(:posts)
users.each do |user|
  puts user.posts.count  # 追加クエリなし
end

# 複数の関連
users = User.includes(:posts, :profile)

# ネストした関連
users = User.includes(posts: :comments)

# preload（別クエリ）
users = User.preload(:posts)

# eager_load（JOIN）
users = User.eager_load(:posts)

# 条件付きeager loading
users = User.includes(:posts).where(posts: { published: true })
```

---

## バリデーション

### 基本的なバリデーション

```ruby
class User < ActiveRecord::Base
  # 存在確認
  validates :name, presence: true

  # 一意性
  validates :email, uniqueness: true
  validates :email, uniqueness: { case_sensitive: false }
  validates :email, uniqueness: { scope: :organization_id }

  # 長さ
  validates :name, length: { minimum: 2, maximum: 50 }
  validates :bio, length: { in: 10..500 }

  # 数値
  validates :age, numericality: { only_integer: true, greater_than: 0 }
  validates :price, numericality: { greater_than_or_equal_to: 0 }

  # フォーマット
  validates :email, format: { with: URI::MailTo::EMAIL_REGEXP }
  validates :phone, format: { with: /\A\d{10,11}\z/ }

  # 含まれる値
  validates :status, inclusion: { in: %w[draft published archived] }
  validates :role, exclusion: { in: %w[banned deleted] }

  # 確認
  validates :password, confirmation: true  # password_confirmation と比較

  # 受け入れ
  validates :terms, acceptance: true
end
```

### 条件付きバリデーション

```ruby
class User < ActiveRecord::Base
  validates :phone, presence: true, if: :requires_phone?
  validates :company, presence: true, unless: :personal_account?

  # Procを使用
  validates :age, presence: true, if: -> { adult_content? }

  private

  def requires_phone?
    verified?
  end
end
```

### カスタムバリデーション

```ruby
class User < ActiveRecord::Base
  validate :email_domain_check

  private

  def email_domain_check
    return if email.blank?

    unless email.end_with?('@company.com')
      errors.add(:email, 'must be a company email')
    end
  end
end

# カスタムバリデータクラス
class EmailValidator < ActiveModel::EachValidator
  def validate_each(record, attribute, value)
    unless value =~ URI::MailTo::EMAIL_REGEXP
      record.errors.add(attribute, options[:message] || 'is not a valid email')
    end
  end
end

class User < ActiveRecord::Base
  validates :email, email: true
end
```

### バリデーションの確認

```ruby
user = User.new(name: '')
user.valid?                            # => false
user.invalid?                          # => true
user.errors.full_messages             # => ["Name can't be blank"]
user.errors[:name]                    # => ["can't be blank"]
user.errors.add(:base, 'Something went wrong')
```

---

## コールバック

### 利用可能なコールバック

```ruby
class User < ActiveRecord::Base
  # 作成・更新・削除の前後
  before_validation :normalize_email
  after_validation :log_errors

  before_save :encrypt_password
  after_save :clear_cache

  before_create :set_defaults
  after_create :send_welcome_email

  before_update :track_changes
  after_update :notify_changes

  before_destroy :check_dependencies
  after_destroy :cleanup

  # トランザクション関連
  after_commit :sync_to_external_service
  after_rollback :log_failure

  private

  def normalize_email
    self.email = email&.downcase&.strip
  end

  def set_defaults
    self.role ||= 'user'
  end

  def send_welcome_email
    UserMailer.welcome(self).deliver_later
  end
end
```

### 条件付きコールバック

```ruby
class User < ActiveRecord::Base
  before_save :update_slug, if: :name_changed?
  after_create :send_notification, unless: :admin?

  after_save :sync_data, if: -> { saved_change_to_email? }
end
```

### コールバックのスキップ

```ruby
# バリデーションとコールバックをスキップ
user.update_column(:name, 'New Name')
user.update_columns(name: 'New Name', updated_at: Time.current)

# 削除時のコールバックをスキップ
user.delete  # destroy の代わりに
```

---

## スコープ

### スコープの定義

```ruby
class Post < ActiveRecord::Base
  # 基本的なスコープ
  scope :published, -> { where(published: true) }
  scope :draft, -> { where(published: false) }
  scope :recent, -> { order(created_at: :desc) }

  # 引数付きスコープ
  scope :by_author, ->(user_id) { where(user_id: user_id) }
  scope :created_after, ->(date) { where('created_at > ?', date) }

  # 条件付きスコープ
  scope :published_after, ->(date) {
    where(published: true).where('published_at > ?', date) if date.present?
  }

  # デフォルトスコープ
  default_scope { where(deleted: false) }
end

# 使用
Post.published.recent.limit(10)
Post.by_author(1).published
Post.created_after(1.week.ago)

# デフォルトスコープを無視
Post.unscoped.all
```

### クラスメソッドとしてのスコープ

```ruby
class Post < ActiveRecord::Base
  def self.search(query)
    return all if query.blank?

    where('title LIKE ? OR body LIKE ?', "%#{query}%", "%#{query}%")
  end

  def self.filter_by_status(status)
    case status
    when 'published'
      published
    when 'draft'
      draft
    else
      all
    end
  end
end
```

---

## トランザクション

### 基本的なトランザクション

```ruby
ActiveRecord::Base.transaction do
  user.save!
  account.save!
  # 例外が発生するとロールバック
end

# モデルから呼び出し
User.transaction do
  user.update!(balance: user.balance - 100)
  other_user.update!(balance: other_user.balance + 100)
end
```

### ネストしたトランザクション

```ruby
User.transaction do
  user.save!

  User.transaction(requires_new: true) do
    # 独立したトランザクション
    other_user.save!
  end
end
```

### ロールバック

```ruby
User.transaction do
  user.save!

  if some_condition
    raise ActiveRecord::Rollback  # 静かにロールバック
  end
end

# 例外でロールバック
begin
  User.transaction do
    user.save!
    raise StandardError, 'Something went wrong'
  end
rescue StandardError => e
  puts "Transaction failed: #{e.message}"
end
```

### ロック

```ruby
# 悲観的ロック
User.transaction do
  user = User.lock.find(1)  # SELECT ... FOR UPDATE
  user.update!(balance: user.balance - 100)
end

# 特定の行をロック
user = User.find(1)
user.with_lock do
  user.update!(balance: user.balance - 100)
end

# 楽観的ロック（lock_version カラムが必要）
class User < ActiveRecord::Base
  # lock_version カラムを追加するマイグレーションが必要
end

user = User.find(1)
user.update!(name: 'New Name')  # lock_version が自動的にインクリメント
```

---

## 実践的な例

### シンプルなアプリケーション

```ruby
# app.rb
require 'active_record'
require 'sqlite3'

# 接続設定
ActiveRecord::Base.establish_connection(
  adapter: 'sqlite3',
  database: ':memory:'
)

# マイグレーション
ActiveRecord::Schema.define do
  create_table :users do |t|
    t.string :name
    t.string :email
    t.timestamps
  end

  create_table :posts do |t|
    t.references :user
    t.string :title
    t.text :body
    t.timestamps
  end
end

# モデル定義
class User < ActiveRecord::Base
  has_many :posts

  validates :name, presence: true
  validates :email, presence: true, uniqueness: true
end

class Post < ActiveRecord::Base
  belongs_to :user

  validates :title, presence: true
end

# 使用
user = User.create!(name: 'Alice', email: 'alice@example.com')
user.posts.create!(title: 'First Post', body: 'Hello, World!')

puts User.includes(:posts).find_by(email: 'alice@example.com').posts.count
```

### Sinatra との組み合わせ

```ruby
# app.rb
require 'sinatra'
require 'active_record'
require 'json'

# 接続設定
ActiveRecord::Base.establish_connection(
  adapter: 'sqlite3',
  database: 'db/app.sqlite3'
)

class User < ActiveRecord::Base
  validates :name, presence: true
  validates :email, presence: true, uniqueness: true
end

# API エンドポイント
get '/users' do
  content_type :json
  User.all.to_json
end

get '/users/:id' do
  content_type :json
  user = User.find_by(id: params[:id])
  halt 404, { error: 'Not found' }.to_json unless user
  user.to_json
end

post '/users' do
  content_type :json
  user = User.new(JSON.parse(request.body.read))
  if user.save
    status 201
    user.to_json
  else
    status 422
    { errors: user.errors.full_messages }.to_json
  end
end
```

---

## パフォーマンス

### インデックス

```ruby
# マイグレーション
class AddIndexes < ActiveRecord::Migration[7.0]
  def change
    add_index :users, :email, unique: true
    add_index :posts, :user_id
    add_index :posts, [:user_id, :published], name: 'index_posts_on_user_and_published'
  end
end
```

### クエリの最適化

```ruby
# 必要なカラムのみ取得
User.select(:id, :name)

# カウントはSQLで
User.where(active: true).count  # SELECT COUNT(*) ...
User.where(active: true).size   # キャッシュがあれば使用

# exists?を使用
User.where(email: 'test@example.com').exists?  # LIMIT 1

# pluckで配列を取得
User.pluck(:email)  # SELECT email FROM users
```

### 接続プール

```ruby
ActiveRecord::Base.establish_connection(
  adapter: 'postgresql',
  database: 'myapp',
  pool: 10,
  checkout_timeout: 5
)
```

---

## 参考リンク

- [Active Record Basics](https://guides.rubyonrails.org/active_record_basics.html)
- [Active Record Query Interface](https://guides.rubyonrails.org/active_record_querying.html)
- [Active Record Associations](https://guides.rubyonrails.org/association_basics.html)
- [Active Record Validations](https://guides.rubyonrails.org/active_record_validations.html)
- [Active Record Callbacks](https://guides.rubyonrails.org/active_record_callbacks.html)

← [Gem一覧に戻る](README.md)
