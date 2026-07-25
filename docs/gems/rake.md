# Rake - タスクランナー

## 概要

RakeはRuby版のMakeです。ビルド、テスト、デプロイなどの定型タスクを自動化するために使用します。Rubyの標準ライブラリに含まれています。

```bash
# 通常はRubyに付属
gem install rake  # 必要な場合

# Gemfile
gem 'rake'
```

> **🐍 Python相当:** タスクランナーは **invoke**（`tasks.py` に `@task` を定義）が最も近い。環境ごとにタスクを回すなら **nox** / **tox**、伝統的な **Makefile** も選択肢。なお `rake` にあるパッケージビルド機能は Python では別途 build / hatch などで担う。

**🐍 Python:**

```bash
pip install invoke  # または nox / tox
```

---

## 基本的な使い方

### Rakefile

```ruby
# Rakefile
desc '挨拶を表示する'
task :hello do
  puts 'Hello, Rake!'
end

# 実行
# $ rake hello
# Hello, Rake!
```

> **🐍 Python相当:** `Rakefile` にタスクを書く ↔ invoke では `tasks.py` に関数を定義しデコレータを付ける。`rake hello` ↔ `invoke hello`。

**🐍 Python:**

```python
# tasks.py
from invoke import task

@task
def hello(c):
    """挨拶を表示する"""
    print("Hello, Invoke!")

# $ invoke hello
```

### タスクの一覧

```bash
# すべてのタスク
rake -T
rake --tasks

# 詳細表示
rake -T -A

# 特定のパターン
rake -T test
```

> **🐍 Python相当:** `rake -T`（説明付き一覧）↔ `invoke --list`（または `invoke -l`）。docstring がそのまま説明として表示される。

**🐍 Python:**

```bash
invoke --list   # タスク一覧
invoke -l
```

---

## タスクの定義

### 基本的なタスク

```ruby
# シンプルなタスク
task :build do
  puts 'Building...'
end

# 説明付き
desc 'Run all tests'
task :test do
  puts 'Testing...'
end

# 引数付きタスク
task :greet, [:name] do |t, args|
  puts "Hello, #{args[:name] || 'World'}!"
end
# $ rake greet[Alice]

# 複数引数
task :create_user, [:name, :email] do |t, args|
  puts "Creating user: #{args[:name]}, #{args[:email]}"
end
# $ rake create_user[Alice,alice@example.com]
```

### デフォルト引数と環境変数

```ruby
# デフォルト値
task :greet, [:name] do |t, args|
  args.with_defaults(name: 'World')
  puts "Hello, #{args[:name]}!"
end

# 環境変数を使用
task :deploy do
  env = ENV['RAILS_ENV'] || 'development'
  puts "Deploying to #{env}..."
end
# $ RAILS_ENV=production rake deploy
```

### 依存タスク

```ruby
# 別のタスクに依存
task :build => :clean do
  puts 'Building...'
end

task :clean do
  puts 'Cleaning...'
end

# 複数の依存
task :deploy => [:test, :build] do
  puts 'Deploying...'
end

# 依存タスクの連鎖
task :release => [:test, :build, :deploy]

# 実行順序: clean -> build -> test -> deploy -> release
```

> **🐍 Python相当:** 依存タスク（prerequisites）↔ invoke の `@task(pre=[...])`。事前に実行するタスクを列挙する。`post=[...]` で後処理も指定できる。

**🐍 Python:**

```python
from invoke import task

@task
def clean(c):
    print("Cleaning...")

@task(pre=[clean])
def build(c):
    print("Building...")

@task(pre=[build, test])   # 複数の依存
def deploy(c):
    print("Deploying...")
```

### 名前空間

```ruby
namespace :db do
  desc 'データベースを作成'
  task :create do
    puts 'Creating database...'
  end

  desc 'マイグレーションを実行'
  task :migrate do
    puts 'Running migrations...'
  end

  desc 'シードデータを投入'
  task :seed => :migrate do
    puts 'Seeding database...'
  end

  desc 'データベースをリセット'
  task :reset => [:drop, :create, :migrate, :seed]

  task :drop do
    puts 'Dropping database...'
  end
end

# $ rake db:create
# $ rake db:migrate
# $ rake db:reset

# ネストした名前空間
namespace :db do
  namespace :schema do
    task :dump do
      puts 'Dumping schema...'
    end

    task :load do
      puts 'Loading schema...'
    end
  end
end

# $ rake db:schema:dump
```

> **🐍 Python相当:** 名前空間（`namespace :db`）↔ invoke の **Collection**。タスクを `Collection` にまとめると `invoke db.migrate` のようにドット区切りで呼び出せる。

**🐍 Python:**

```python
# tasks.py
from invoke import task, Collection

@task
def create(c):
    print("Creating database...")

@task
def migrate(c):
    print("Running migrations...")

db = Collection("db")
db.add_task(create)
db.add_task(migrate)

ns = Collection()
ns.add_collection(db)

# $ invoke db.create
# $ invoke db.migrate
```

---

## ファイルタスク

### file タスク

```ruby
# ファイルが存在しない、または依存ファイルより古い場合に実行
file 'output.txt' => ['input.txt'] do |t|
  puts "Creating #{t.name} from input.txt"
  File.write(t.name, File.read('input.txt').upcase)
end

# $ rake output.txt

# 複数の依存ファイル
file 'app.js' => ['src/module1.js', 'src/module2.js'] do |t|
  content = t.prerequisites.map { |f| File.read(f) }.join("\n")
  File.write(t.name, content)
end
```

### directory タスク

```ruby
# ディレクトリの作成
directory 'tmp/cache'
directory 'log'

task :setup => ['tmp/cache', 'log'] do
  puts 'Setup complete'
end
```

### ルールベースのタスク

```ruby
# パターンマッチングでタスクを生成
rule '.o' => '.c' do |t|
  sh "gcc -c -o #{t.name} #{t.source}"
end

# .md から .html を生成
rule '.html' => '.md' do |t|
  sh "markdown #{t.source} > #{t.name}"
end
```

---

## FileList

```ruby
# ファイルリストの作成
files = FileList['**/*.rb']
files = FileList['app/**/*.rb', 'lib/**/*.rb']

# 除外
files = FileList['**/*.rb'].exclude('vendor/**/*')
files.exclude('test/**/*')

# タスクで使用
task :count_lines do
  files = FileList['lib/**/*.rb']
  total = files.sum { |f| File.readlines(f).size }
  puts "Total lines: #{total}"
end

# clean タスク用
CLEAN = FileList['**/*.o', '**/*.log']
CLOBBER = FileList['build/**/*']

require 'rake/clean'
# $ rake clean    # CLEAN のファイルを削除
# $ rake clobber  # CLEAN と CLOBBER のファイルを削除
```

---

## シェルコマンド

### sh メソッド

```ruby
task :build do
  sh 'mkdir -p build'
  sh 'gcc -o build/app src/main.c'
end

# 失敗時の処理
task :deploy do
  sh 'git push origin main' do |ok, res|
    if ok
      puts 'Pushed successfully'
    else
      puts "Push failed: #{res.exitstatus}"
    end
  end
end
```

### ruby メソッド

```ruby
task :run do
  ruby 'script.rb'
  ruby '-e', 'puts "Hello"'
end
```

### その他のヘルパー

```ruby
task :example do
  # ディレクトリ変更
  cd 'subdir' do
    sh 'pwd'
  end

  # ファイル操作
  mkdir_p 'path/to/dir'
  cp 'src/file.txt', 'dest/'
  mv 'old.txt', 'new.txt'
  rm_rf 'tmp'

  # 環境変数
  ENV['PATH'] = "/custom/bin:#{ENV['PATH']}"
end
```

---

## テストタスク

### Rake::TestTask

```ruby
require 'rake/testtask'

Rake::TestTask.new(:test) do |t|
  t.libs << 'test'
  t.libs << 'lib'
  t.test_files = FileList['test/**/*_test.rb']
  t.verbose = true
end

# $ rake test
```

### RSpec タスク

```ruby
require 'rspec/core/rake_task'

RSpec::Core::RakeTask.new(:spec) do |t|
  t.pattern = 'spec/**/*_spec.rb'
  t.rspec_opts = '--format documentation'
end

# デフォルトタスクに設定
task default: :spec
```

---

## マルチタスク

```ruby
# 並列実行
multitask :parallel => [:task1, :task2, :task3]

task :task1 do
  sleep 2
  puts 'Task 1 done'
end

task :task2 do
  sleep 2
  puts 'Task 2 done'
end

task :task3 do
  sleep 2
  puts 'Task 3 done'
end

# 約2秒で完了（直列なら6秒）
```

---

## 実践的な例

### プロジェクトセットアップ

```ruby
# Rakefile
namespace :setup do
  desc '開発環境のセットアップ'
  task :dev => [:bundle, :db, :env] do
    puts 'Development environment ready!'
  end

  task :bundle do
    sh 'bundle install'
  end

  task :db do
    sh 'bundle exec rake db:create db:migrate db:seed'
  end

  task :env do
    unless File.exist?('.env')
      cp '.env.example', '.env'
      puts 'Created .env file - please update with your settings'
    end
  end
end
```

### デプロイタスク

```ruby
namespace :deploy do
  desc 'ステージングにデプロイ'
  task :staging do
    deploy_to('staging')
  end

  desc '本番にデプロイ'
  task :production do
    confirm('production')
    deploy_to('production')
  end

  def deploy_to(env)
    puts "Deploying to #{env}..."
    sh "cap #{env} deploy"
  end

  def confirm(env)
    print "Deploy to #{env}? [y/N]: "
    input = STDIN.gets.chomp
    abort 'Aborted' unless input.downcase == 'y'
  end
end
```

### ビルドタスク

```ruby
BUILD_DIR = 'build'
SRC_FILES = FileList['src/**/*.rb']

directory BUILD_DIR

desc 'アプリケーションをビルド'
task :build => [BUILD_DIR, :compile, :assets]

task :compile => BUILD_DIR do
  SRC_FILES.each do |src|
    dest = src.sub('src/', "#{BUILD_DIR}/")
    mkdir_p File.dirname(dest)
    cp src, dest
  end
end

task :assets => BUILD_DIR do
  sh "cp -r assets #{BUILD_DIR}/"
end

desc 'ビルドをクリーン'
task :clean do
  rm_rf BUILD_DIR
end

desc 'クリーンしてビルド'
task :rebuild => [:clean, :build]
```

### CI/CD タスク

```ruby
namespace :ci do
  desc 'CIパイプラインを実行'
  task :run => [:lint, :test, :security]

  task :lint do
    sh 'bundle exec rubocop'
  end

  task :test do
    sh 'bundle exec rspec'
  end

  task :security do
    sh 'bundle exec brakeman -q'
    sh 'bundle audit check --update'
  end
end

# デフォルトタスク
task default: 'ci:run'
```

---

## Rake の読み込み

### 複数ファイルに分割

```ruby
# Rakefile
Dir.glob('lib/tasks/**/*.rake').each { |r| load r }

# または
import 'lib/tasks/db.rake'
import 'lib/tasks/deploy.rake'
```

```ruby
# lib/tasks/db.rake
namespace :db do
  task :backup do
    # ...
  end
end
```

---

## コマンドラインオプション

```bash
# タスク一覧
rake -T

# 詳細なタスク一覧
rake -T -A

# ドライラン（実行せずに表示）
rake --dry-run task_name
rake -n task_name

# 依存関係を表示
rake -P

# トレース（デバッグ用）
rake --trace task_name
rake -t task_name

# 特定のRakefileを使用
rake -f path/to/Rakefile

# ディレクトリを指定
rake -C subdir task_name
```

---

## 参考リンク

- [Rake公式ドキュメント](https://ruby.github.io/rake/)
- [Rake GitHub](https://github.com/ruby/rake)

← [Gem一覧に戻る](README.md)
