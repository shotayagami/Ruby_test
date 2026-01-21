# Sidekiq - バックグラウンドジョブ処理

## 概要

SidekiqはRedisを使用した高性能なバックグラウンドジョブ処理システムです。マルチスレッドで動作し、非同期処理を効率的に実行できます。

```bash
# インストール
gem install sidekiq

# Gemfile
gem 'sidekiq'
```

**必要な依存:**
- Redis サーバー

---

## 基本的な使い方

### Worker の作成

```ruby
# app/workers/hard_worker.rb
class HardWorker
  include Sidekiq::Worker

  def perform(name, count)
    # 重い処理
    count.times do |i|
      puts "Processing #{name}: #{i + 1}/#{count}"
      sleep 1
    end
  end
end
```

### ジョブの実行

```ruby
# 非同期実行（即座にキューに追加）
HardWorker.perform_async('task', 5)

# 遅延実行（5分後）
HardWorker.perform_in(5.minutes, 'task', 5)

# 指定時刻に実行
HardWorker.perform_at(Time.now + 1.hour, 'task', 5)

# 同期実行（テスト用）
HardWorker.new.perform('task', 5)
```

---

## 設定

### Sidekiq の設定

```ruby
# config/initializers/sidekiq.rb
Sidekiq.configure_server do |config|
  config.redis = { url: ENV.fetch('REDIS_URL', 'redis://localhost:6379/0') }

  # コネクションプールサイズ
  config.redis = { url: ENV['REDIS_URL'], size: 25 }
end

Sidekiq.configure_client do |config|
  config.redis = { url: ENV.fetch('REDIS_URL', 'redis://localhost:6379/0') }
end
```

### sidekiq.yml

```yaml
# config/sidekiq.yml
:concurrency: 10
:timeout: 25
:queues:
  - [critical, 3]
  - [default, 2]
  - [low, 1]

# 環境別設定
production:
  :concurrency: 25
development:
  :concurrency: 5
```

### Sidekiq の起動

```bash
# 基本起動
bundle exec sidekiq

# 設定ファイル指定
bundle exec sidekiq -C config/sidekiq.yml

# キュー指定
bundle exec sidekiq -q critical -q default -q low

# 並行数指定
bundle exec sidekiq -c 20

# デーモンモード
bundle exec sidekiq -d -L log/sidekiq.log
```

---

## Worker のオプション

### 基本オプション

```ruby
class ConfiguredWorker
  include Sidekiq::Worker

  # オプション設定
  sidekiq_options queue: :critical,       # キュー名
                  retry: 5,               # リトライ回数
                  backtrace: true,        # バックトレース保存
                  dead: true,             # Dead Job Queue に移動
                  tags: ['important']     # タグ

  def perform(id)
    # 処理
  end
end
```

### リトライ設定

```ruby
class RetryWorker
  include Sidekiq::Worker

  # リトライ回数（デフォルト: 25）
  sidekiq_options retry: 5

  # リトライ無効
  sidekiq_options retry: false

  # リトライ間隔のカスタマイズ
  sidekiq_retry_in do |count, exception|
    case exception
    when RateLimitError
      60 * 60  # 1時間後
    else
      (count ** 4) + 15  # デフォルトの指数バックオフ
    end
  end

  # 最終リトライ失敗時の処理
  sidekiq_retries_exhausted do |job, exception|
    Rails.logger.error "Job #{job['jid']} failed: #{exception.message}"
    # 通知を送るなど
  end

  def perform(id)
    # 処理
  end
end
```

### ユニークジョブ

```ruby
# sidekiq-unique-jobs gem を使用
class UniqueWorker
  include Sidekiq::Worker

  sidekiq_options lock: :until_executed,
                  on_conflict: :reject

  def perform(user_id)
    # 同じ user_id のジョブは1つだけ実行
  end
end
```

---

## キュー管理

### キューの優先度

```ruby
# 優先度の高い順に処理
# config/sidekiq.yml
:queues:
  - critical    # 最優先
  - default
  - low         # 最後

# または重み付け
:queues:
  - [critical, 3]  # 3倍の頻度で処理
  - [default, 2]
  - [low, 1]
```

### Worker でのキュー指定

```ruby
class CriticalWorker
  include Sidekiq::Worker
  sidekiq_options queue: :critical

  def perform(data)
    # 重要な処理
  end
end

class LowPriorityWorker
  include Sidekiq::Worker
  sidekiq_options queue: :low

  def perform(data)
    # 優先度の低い処理
  end
end
```

### 動的キュー指定

```ruby
# 実行時にキューを指定
Sidekiq::Client.push(
  'class' => HardWorker,
  'args' => ['task', 5],
  'queue' => 'critical'
)

# set メソッドを使用
HardWorker.set(queue: :critical).perform_async('task', 5)
```

---

## ジョブの状態管理

### ジョブ情報の取得

```ruby
# ジョブIDの取得
jid = HardWorker.perform_async('task', 5)
puts "Job ID: #{jid}"

# キュー内のジョブ確認
queue = Sidekiq::Queue.new('default')
queue.each do |job|
  puts "Job: #{job.klass}, Args: #{job.args}"
  puts "Enqueued at: #{job.enqueued_at}"
end

# キューのサイズ
puts "Queue size: #{queue.size}"

# 全キューのサイズ
Sidekiq::Queue.all.each do |q|
  puts "#{q.name}: #{q.size}"
end
```

### 実行中のジョブ

```ruby
workers = Sidekiq::Workers.new

workers.each do |process_id, thread_id, work|
  puts "Process: #{process_id}"
  puts "Thread: #{thread_id}"
  puts "Class: #{work['payload']['class']}"
  puts "Args: #{work['payload']['args']}"
  puts "Started at: #{Time.at(work['run_at'])}"
end

puts "Total workers: #{workers.size}"
```

### スケジュールされたジョブ

```ruby
scheduled = Sidekiq::ScheduledSet.new

scheduled.each do |job|
  puts "#{job.klass}: scheduled for #{job.at}"
end

# 特定のジョブを削除
scheduled.each do |job|
  job.delete if job.klass == 'HardWorker'
end

# 全削除
scheduled.clear
```

### リトライ待ちジョブ

```ruby
retry_set = Sidekiq::RetrySet.new

retry_set.each do |job|
  puts "#{job.klass}: #{job.error_message}"
  puts "Retry count: #{job['retry_count']}"
end

# 即座にリトライ
retry_set.each do |job|
  job.retry if job.klass == 'ImportantWorker'
end

# 全削除
retry_set.clear
```

### Dead Job Queue

```ruby
dead = Sidekiq::DeadSet.new

dead.each do |job|
  puts "#{job.klass}: #{job.error_message}"
  puts "Died at: #{job.at}"
end

# 復活させる
dead.each do |job|
  job.retry if job.klass == 'RecoverableWorker'
end

# サイズ確認
puts "Dead jobs: #{dead.size}"
```

---

## バッチ処理（Sidekiq Pro）

### 基本的なバッチ

```ruby
# Sidekiq Pro が必要
batch = Sidekiq::Batch.new
batch.description = "Import users"
batch.on(:success, BatchCallbacks, user_id: 123)
batch.on(:complete, BatchCallbacks, user_id: 123)

batch.jobs do
  users.each do |user|
    ImportWorker.perform_async(user.id)
  end
end

puts "Batch ID: #{batch.bid}"
```

### バッチコールバック

```ruby
class BatchCallbacks
  def on_success(status, options)
    # 全ジョブ成功時
    puts "All #{status.total} jobs succeeded!"
  end

  def on_complete(status, options)
    # 全ジョブ完了時（成功・失敗問わず）
    puts "Completed: #{status.total} total, #{status.failures} failures"
  end
end
```

---

## Web UI

### 設定

```ruby
# Gemfile
gem 'sidekiq'

# config/routes.rb (Rails)
require 'sidekiq/web'

Rails.application.routes.draw do
  # 認証付き（推奨）
  authenticate :user, ->(u) { u.admin? } do
    mount Sidekiq::Web => '/sidekiq'
  end

  # または Basic 認証
  Sidekiq::Web.use Rack::Auth::Basic do |username, password|
    ActiveSupport::SecurityUtils.secure_compare(
      ::Digest::SHA256.hexdigest(username),
      ::Digest::SHA256.hexdigest(ENV['SIDEKIQ_USERNAME'])
    ) & ActiveSupport::SecurityUtils.secure_compare(
      ::Digest::SHA256.hexdigest(password),
      ::Digest::SHA256.hexdigest(ENV['SIDEKIQ_PASSWORD'])
    )
  end
  mount Sidekiq::Web => '/sidekiq'
end
```

### Sinatra での設定

```ruby
# config.ru
require 'sidekiq/web'

map '/sidekiq' do
  use Rack::Auth::Basic, 'Protected' do |username, password|
    username == 'admin' && password == 'secret'
  end
  run Sidekiq::Web
end
```

---

## テスト

### RSpec での設定

```ruby
# spec/rails_helper.rb
require 'sidekiq/testing'

RSpec.configure do |config|
  config.before(:each) do
    Sidekiq::Worker.clear_all
  end
end
```

### テストモード

```ruby
# Fake モード（デフォルト）- ジョブをキューに追加するだけ
Sidekiq::Testing.fake! do
  HardWorker.perform_async('task', 5)
  expect(HardWorker.jobs.size).to eq(1)

  # ジョブの内容確認
  job = HardWorker.jobs.first
  expect(job['args']).to eq(['task', 5])

  # 全ジョブ実行
  HardWorker.drain
end

# Inline モード - 即座に同期実行
Sidekiq::Testing.inline! do
  HardWorker.perform_async('task', 5)
  # すぐに実行される
end

# Disable モード - 実際にRedisに送信
Sidekiq::Testing.disable! do
  HardWorker.perform_async('task', 5)
end
```

### ジョブのテスト例

```ruby
# spec/workers/hard_worker_spec.rb
require 'rails_helper'

RSpec.describe HardWorker, type: :worker do
  describe '#perform' do
    it 'enqueues a job' do
      expect {
        HardWorker.perform_async('test', 3)
      }.to change(HardWorker.jobs, :size).by(1)
    end

    it 'executes the job' do
      expect_any_instance_of(HardWorker).to receive(:perform).with('test', 3)
      HardWorker.new.perform('test', 3)
    end
  end

  describe 'job options' do
    it 'uses the correct queue' do
      expect(HardWorker.get_sidekiq_options['queue']).to eq('default')
    end
  end
end
```

### shoulda-matchers 風のマッチャー

```ruby
# spec/support/sidekiq_matchers.rb
RSpec::Matchers.define :have_enqueued_sidekiq_job do |*expected_args|
  match do |worker_class|
    worker_class.jobs.any? do |job|
      expected_args.empty? || job['args'] == expected_args
    end
  end
end

# 使用例
expect(HardWorker).to have_enqueued_sidekiq_job('task', 5)
```

---

## エラーハンドリング

### カスタムエラーハンドラ

```ruby
# config/initializers/sidekiq.rb
Sidekiq.configure_server do |config|
  config.error_handlers << proc do |exception, context|
    # エラー通知
    Bugsnag.notify(exception) do |report|
      report.add_metadata(:sidekiq, context)
    end
  end
end
```

### Worker 内でのエラー処理

```ruby
class SafeWorker
  include Sidekiq::Worker

  def perform(id)
    user = User.find(id)
    process_user(user)
  rescue ActiveRecord::RecordNotFound => e
    # リトライしない
    logger.warn "User #{id} not found, skipping"
  rescue ExternalService::Error => e
    # 手動でリトライをスケジュール
    SafeWorker.perform_in(1.hour, id)
  rescue StandardError => e
    # 通常のリトライフローに任せる
    raise
  end
end
```

### 死活監視

```ruby
# ヘルスチェックエンドポイント
class HealthController < ApplicationController
  def sidekiq
    stats = Sidekiq::Stats.new

    if stats.processes_size > 0
      render json: {
        status: 'ok',
        processes: stats.processes_size,
        queues: stats.queues,
        enqueued: stats.enqueued,
        processed: stats.processed,
        failed: stats.failed
      }
    else
      render json: { status: 'error', message: 'No Sidekiq processes' },
             status: :service_unavailable
    end
  end
end
```

---

## 実践的なパターン

### メール送信

```ruby
class EmailWorker
  include Sidekiq::Worker
  sidekiq_options queue: :mailers, retry: 3

  def perform(user_id, template)
    user = User.find(user_id)
    UserMailer.send(template, user).deliver_now
  end
end

# 使用
EmailWorker.perform_async(user.id, :welcome)
```

### 外部API連携

```ruby
class ApiSyncWorker
  include Sidekiq::Worker
  sidekiq_options queue: :api, retry: 5

  sidekiq_retry_in do |count, exception|
    case exception
    when RateLimitError
      exception.retry_after || 60
    else
      (count ** 4) + 15
    end
  end

  def perform(resource_id)
    resource = Resource.find(resource_id)
    ExternalApi.sync(resource)
  rescue ExternalApi::RateLimitError => e
    raise RateLimitError.new(e.retry_after)
  end
end

class RateLimitError < StandardError
  attr_reader :retry_after

  def initialize(retry_after = nil)
    @retry_after = retry_after
    super("Rate limit exceeded")
  end
end
```

### データインポート

```ruby
class ImportWorker
  include Sidekiq::Worker
  sidekiq_options queue: :imports, retry: 1

  def perform(file_path, user_id)
    user = User.find(user_id)

    CSV.foreach(file_path, headers: true).each_slice(100) do |batch|
      ProcessBatchWorker.perform_async(batch.map(&:to_h), user_id)
    end

    NotificationWorker.perform_async(user_id, 'Import started')
  ensure
    File.delete(file_path) if File.exist?(file_path)
  end
end

class ProcessBatchWorker
  include Sidekiq::Worker
  sidekiq_options queue: :imports

  def perform(records, user_id)
    records.each do |record|
      Product.create!(record)
    end
  end
end
```

### 定期実行（sidekiq-cron）

```ruby
# Gemfile
gem 'sidekiq-cron'

# config/initializers/sidekiq_cron.rb
Sidekiq::Cron::Job.create(
  name: 'Daily cleanup',
  cron: '0 3 * * *',  # 毎日午前3時
  class: 'CleanupWorker'
)

Sidekiq::Cron::Job.create(
  name: 'Hourly sync',
  cron: '0 * * * *',  # 毎時
  class: 'SyncWorker',
  args: { full: false }
)

# YAMLで設定
# config/schedule.yml
daily_cleanup:
  cron: "0 3 * * *"
  class: "CleanupWorker"

hourly_sync:
  cron: "0 * * * *"
  class: "SyncWorker"
  args:
    full: false

# 読み込み
schedule = YAML.load_file('config/schedule.yml')
Sidekiq::Cron::Job.load_from_hash(schedule)
```

---

## パフォーマンス最適化

### 並行数の調整

```yaml
# config/sidekiq.yml
# CPUバウンドな処理が多い場合
:concurrency: <%= ENV.fetch('SIDEKIQ_CONCURRENCY', 5) %>

# I/Oバウンドな処理が多い場合
:concurrency: <%= ENV.fetch('SIDEKIQ_CONCURRENCY', 25) %>
```

### メモリ管理

```ruby
class MemoryEfficientWorker
  include Sidekiq::Worker

  def perform(large_ids)
    # バッチ処理でメモリを節約
    large_ids.each_slice(100) do |batch|
      process_batch(batch)
      GC.start  # 必要に応じて
    end
  end
end
```

### コネクションプール

```ruby
# config/initializers/sidekiq.rb
Sidekiq.configure_server do |config|
  config.redis = {
    url: ENV['REDIS_URL'],
    size: ENV.fetch('SIDEKIQ_REDIS_SIZE', 10).to_i
  }

  # ActiveRecord コネクションプール
  database_url = ENV['DATABASE_URL']
  if database_url
    ActiveRecord::Base.establish_connection(
      database_url,
      pool: ENV.fetch('SIDEKIQ_DB_POOL', 10).to_i
    )
  end
end
```

---

## デプロイメント

### Systemd

```ini
# /etc/systemd/system/sidekiq.service
[Unit]
Description=sidekiq
After=syslog.target network.target

[Service]
Type=simple
WorkingDirectory=/var/www/app/current
ExecStart=/bin/bash -lc 'bundle exec sidekiq -C config/sidekiq.yml'
User=deploy
Group=deploy
UMask=0002

Environment=RAILS_ENV=production
Environment=MALLOC_ARENA_MAX=2

RestartSec=1
Restart=on-failure

StandardOutput=syslog
StandardError=syslog
SyslogIdentifier=sidekiq

[Install]
WantedBy=multi-user.target
```

### Docker

```dockerfile
# Dockerfile
FROM ruby:3.3

WORKDIR /app
COPY . .
RUN bundle install

CMD ["bundle", "exec", "sidekiq", "-C", "config/sidekiq.yml"]
```

```yaml
# docker-compose.yml
services:
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

  sidekiq:
    build: .
    depends_on:
      - redis
    environment:
      - REDIS_URL=redis://redis:6379/0
    volumes:
      - .:/app
```

---

## 関連 Gem

| Gem | 説明 |
|-----|------|
| sidekiq-cron | 定期実行 |
| sidekiq-unique-jobs | ユニークジョブ |
| sidekiq-failures | 失敗ジョブの管理UI |
| sidekiq-limit_fetch | キューの同時実行数制限 |
| sidekiq-scheduler | スケジューリング |

---

## 参考リンク

- [Sidekiq Wiki](https://github.com/sidekiq/sidekiq/wiki)
- [Sidekiq Best Practices](https://github.com/sidekiq/sidekiq/wiki/Best-Practices)
- [Sidekiq Pro/Enterprise](https://sidekiq.org/)

← [Gem一覧に戻る](README.md)
