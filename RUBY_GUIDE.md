# Ruby 基礎ガイド

## 目次
1. [環境構築](#1-環境構築)
2. [基本文法](#2-基本文法)
3. [データ型](#3-データ型)
4. [制御構文](#4-制御構文)
5. [メソッド](#5-メソッド)
6. [クラスとオブジェクト](#6-クラスとオブジェクト)
7. [モジュール](#7-モジュール)
8. [例外処理](#8-例外処理)
9. [ファイル操作](#9-ファイル操作)
10. [Gem（パッケージ管理）](#10-gemパッケージ管理)
11. [よく使うイディオム](#11-よく使うイディオム)
12. [デバッグ方法](#12-デバッグ方法)

## 詳細ドキュメント

各章の詳細は [docs/](docs/) ディレクトリを参照してください。

- [13. テスト](docs/13_テスト.md) - Minitest、RSpec、テストのベストプラクティス

---

## 1. 環境構築

### Rubyのインストール

**Windows:**
```bash
# RubyInstallerを使用（推奨）
# https://rubyinstaller.org/ からダウンロード

# インストール確認
ruby -v
```

**Mac:**
```bash
# Homebrewを使用
brew install ruby

# rbenvを使用（バージョン管理推奨）
brew install rbenv
rbenv install 3.3.0
rbenv global 3.3.0
```

**Linux:**
```bash
# Ubuntu/Debian
sudo apt-get install ruby-full

# rbenvを使用（推奨）
git clone https://github.com/rbenv/rbenv.git ~/.rbenv
```

### 実行方法

```bash
# ファイルを実行
ruby script.rb

# 対話型シェル（irb）
irb

# ワンライナー
ruby -e 'puts "Hello"'
```

---

## 2. 基本文法

### Hello World

```ruby
puts "Hello, World!"
```

### コメント

```ruby
# 単一行コメント

=begin
複数行コメント
ここに説明を書く
=end
```

### 変数

```ruby
# ローカル変数（小文字またはアンダースコアで始まる）
name = "太郎"
user_age = 25

# インスタンス変数（@で始まる）
@name = "太郎"

# クラス変数（@@で始まる）
@@count = 0

# グローバル変数（$で始まる）- 使用は推奨されない
$global_var = "global"

# 定数（大文字で始まる）
MAX_SIZE = 100
PI = 3.14159
```

### 文字列

```ruby
# シングルクォート（エスケープシーケンス無効）
str1 = 'Hello'

# ダブルクォート（式展開・エスケープシーケンス有効）
name = "Ruby"
str2 = "Hello, #{name}!"  # => "Hello, Ruby!"
str3 = "改行\nタブ\t"

# ヒアドキュメント
text = <<~TEXT
  複数行の
  テキストを
  書ける
TEXT
```

### 出力

```ruby
puts "改行あり"      # 末尾に改行を追加
print "改行なし"     # 改行なし
p [1, 2, 3]          # デバッグ用（inspect形式）
pp { a: 1, b: 2 }    # 整形出力（Pretty Print）
```

---

## 3. データ型

### 数値

```ruby
# 整数（Integer）
num = 42
big_num = 1_000_000  # アンダースコアで区切り可能

# 浮動小数点（Float）
pi = 3.14159

# 演算
1 + 2      # => 3
10 / 3     # => 3（整数同士は整数）
10.0 / 3   # => 3.333...
10 % 3     # => 1（剰余）
2 ** 10    # => 1024（累乗）
```

### 文字列（String）

```ruby
str = "Hello"

str.length          # => 5
str.upcase          # => "HELLO"
str.downcase        # => "hello"
str.reverse         # => "olleH"
str.include?("ell") # => true
str.gsub("l", "L")  # => "HeLLo"
str.split("")       # => ["H", "e", "l", "l", "o"]

# 文字列結合
"Hello" + " " + "World"  # => "Hello World"
"Ruby" * 3               # => "RubyRubyRuby"
```

### 配列（Array）

```ruby
# 作成
arr = [1, 2, 3, 4, 5]
arr = %w[apple banana cherry]  # => ["apple", "banana", "cherry"]

# アクセス
arr[0]      # => 最初の要素
arr[-1]     # => 最後の要素
arr[1..3]   # => インデックス1から3まで

# 操作
arr.push(6)      # 末尾に追加
arr << 7         # 末尾に追加（同上）
arr.pop          # 末尾から削除
arr.shift        # 先頭から削除
arr.unshift(0)   # 先頭に追加

# よく使うメソッド
arr.length       # 要素数
arr.first        # 最初の要素
arr.last         # 最後の要素
arr.include?(3)  # 含まれるか
arr.sort         # ソート
arr.uniq         # 重複削除
arr.reverse      # 逆順
arr.compact      # nil除去
arr.flatten      # ネスト解除
```

### ハッシュ（Hash）- 他言語のMap/Dictionary

```ruby
# 作成（シンボルキー推奨）
user = { name: "太郎", age: 25, city: "東京" }

# 文字列キー
config = { "host" => "localhost", "port" => 3000 }

# アクセス
user[:name]           # => "太郎"
user[:email]          # => nil
user.fetch(:email, "default@example.com")  # デフォルト値付き

# 操作
user[:email] = "taro@example.com"  # 追加/更新
user.delete(:city)                  # 削除

# よく使うメソッド
user.keys            # => [:name, :age, :city]
user.values          # => ["太郎", 25, "東京"]
user.key?(:name)     # => true
user.merge({ job: "Engineer" })  # マージ
```

### シンボル（Symbol）

```ruby
# 文字列に似ているが、イミュータブルで高速
:name
:user_id

# 文字列との変換
"hello".to_sym  # => :hello
:hello.to_s     # => "hello"

# ハッシュのキーとしてよく使用
{ name: "太郎" }  # name: は :name => の省略形
```

### 範囲（Range）

```ruby
(1..5)   # 1, 2, 3, 4, 5（終端含む）
(1...5)  # 1, 2, 3, 4（終端含まない）

(1..5).to_a        # => [1, 2, 3, 4, 5]
("a".."e").to_a    # => ["a", "b", "c", "d", "e"]

# 範囲チェック
(1..10).include?(5)  # => true
(1..10).cover?(5)    # => true（より高速）
```

### nil と真偽値

```ruby
# nilはオブジェクト
nil
nil.nil?   # => true

# 真偽値
true
false

# Rubyでの偽: false と nil のみ
# それ以外はすべて真（0や""も真）
```

---

## 4. 制御構文

### 条件分岐

```ruby
# if文
if score >= 80
  puts "優秀"
elsif score >= 60
  puts "合格"
else
  puts "不合格"
end

# 後置if（1行で書ける場合）
puts "成人" if age >= 18

# unless（ifの逆）
unless logged_in?
  redirect_to login_path
end
puts "ログインしてください" unless logged_in?

# 三項演算子
status = age >= 18 ? "成人" : "未成年"

# case文（switch相当）
case grade
when "A"
  puts "素晴らしい"
when "B", "C"
  puts "良い"
when "D".."F"
  puts "要努力"
else
  puts "不明"
end

# case文（パターンマッチング Ruby 3.0+）
case user
in { name:, age: 18.. }
  puts "成人: #{name}"
in { name:, age: }
  puts "未成年: #{name}"
end
```

### ループ

```ruby
# each（最もよく使う）
[1, 2, 3].each do |num|
  puts num
end

# 1行で書く場合
[1, 2, 3].each { |num| puts num }

# each_with_index
["a", "b", "c"].each_with_index do |item, index|
  puts "#{index}: #{item}"
end

# times
5.times do |i|
  puts i  # 0, 1, 2, 3, 4
end

# upto / downto
1.upto(5) { |i| puts i }    # 1, 2, 3, 4, 5
5.downto(1) { |i| puts i }  # 5, 4, 3, 2, 1

# while
i = 0
while i < 5
  puts i
  i += 1
end

# until（whileの逆）
i = 0
until i >= 5
  puts i
  i += 1
end

# for（あまり使わない）
for i in 1..5
  puts i
end

# loop（無限ループ）
loop do
  break if condition
end
```

### ループ制御

```ruby
# break: ループを抜ける
[1, 2, 3, 4, 5].each do |n|
  break if n > 3
  puts n
end

# next: 次のイテレーションへ
[1, 2, 3, 4, 5].each do |n|
  next if n.even?
  puts n  # 1, 3, 5
end

# redo: 同じイテレーションを再実行
```

---

## 5. メソッド

### 基本

```ruby
# 定義
def greet(name)
  "Hello, #{name}!"
end

# 呼び出し（括弧は省略可能）
greet("Ruby")
greet "Ruby"

# 最後の式の値が戻り値（returnは省略可能）
def add(a, b)
  a + b
end

# 明示的なreturn
def check(n)
  return "negative" if n < 0
  return "zero" if n == 0
  "positive"
end
```

### 引数

```ruby
# デフォルト値
def greet(name = "Guest")
  "Hello, #{name}!"
end

# キーワード引数（推奨）
def create_user(name:, age:, city: "東京")
  { name: name, age: age, city: city }
end
create_user(name: "太郎", age: 25)

# 可変長引数
def sum(*numbers)
  numbers.sum
end
sum(1, 2, 3, 4, 5)  # => 15

# ハッシュ引数
def configure(**options)
  options
end
configure(host: "localhost", port: 3000)
```

### ブロック

```ruby
# ブロックを受け取るメソッド
def with_logging
  puts "開始"
  result = yield  # ブロックを実行
  puts "終了"
  result
end

with_logging { puts "処理中"; 42 }

# ブロックに引数を渡す
def repeat(n)
  n.times { |i| yield(i) }
end

repeat(3) { |i| puts "#{i}回目" }

# ブロックを明示的に受け取る
def process(&block)
  block.call(10)
end
```

### メソッド名の慣習

```ruby
# ?で終わる: 真偽値を返す
"".empty?      # => true
[1,2].include?(1)  # => true

# !で終わる: 破壊的メソッド（オブジェクトを変更）
arr = [3, 1, 2]
arr.sort   # => [1, 2, 3]（元のarrは変わらない）
arr.sort!  # arrが直接ソートされる

# =で終わる: セッター
def name=(value)
  @name = value
end
```

---

## 6. クラスとオブジェクト

### 基本的なクラス

```ruby
class User
  # クラス変数
  @@count = 0

  # 初期化メソッド（コンストラクタ）
  def initialize(name, age)
    @name = name  # インスタンス変数
    @age = age
    @@count += 1
  end

  # インスタンスメソッド
  def greet
    "こんにちは、#{@name}です"
  end

  # ゲッター
  def name
    @name
  end

  # セッター
  def name=(value)
    @name = value
  end

  # クラスメソッド
  def self.count
    @@count
  end
end

# インスタンス化
user = User.new("太郎", 25)
user.greet      # => "こんにちは、太郎です"
user.name       # => "太郎"
user.name = "次郎"
User.count      # => 1
```

### アクセサ（ゲッター/セッター自動生成）

```ruby
class User
  attr_reader :name      # ゲッターのみ
  attr_writer :email     # セッターのみ
  attr_accessor :age     # 両方

  def initialize(name, age)
    @name = name
    @age = age
  end
end

user = User.new("太郎", 25)
user.name           # => "太郎"
user.age = 26       # セッター
user.age            # => 26
```

### 継承

```ruby
class Animal
  def initialize(name)
    @name = name
  end

  def speak
    "..."
  end
end

class Dog < Animal
  def speak
    "ワン！"
  end

  def fetch
    "#{@name}がボールを取ってきた"
  end
end

class Cat < Animal
  def speak
    "ニャー"
  end
end

dog = Dog.new("ポチ")
dog.speak   # => "ワン！"
dog.fetch   # => "ポチがボールを取ってきた"
```

### アクセス制御

```ruby
class MyClass
  def public_method
    "誰でも呼べる"
  end

  protected

  def protected_method
    "サブクラスから呼べる"
  end

  private

  def private_method
    "クラス内からのみ呼べる"
  end
end
```

---

## 7. モジュール

### 名前空間として使用

```ruby
module MyApp
  class User
    # ...
  end

  class Admin
    # ...
  end
end

user = MyApp::User.new
```

### ミックスイン（多重継承の代替）

```ruby
module Loggable
  def log(message)
    puts "[LOG] #{message}"
  end
end

module Cacheable
  def cache(key, value)
    @cache ||= {}
    @cache[key] = value
  end
end

class Service
  include Loggable    # インスタンスメソッドとして追加
  extend Cacheable    # クラスメソッドとして追加

  def process
    log("処理開始")
  end
end

service = Service.new
service.log("テスト")     # include したメソッド
Service.cache(:key, 123)  # extend したメソッド
```

### 標準モジュール

```ruby
# Comparable: 比較演算子を提供
class Version
  include Comparable
  attr_reader :number

  def initialize(number)
    @number = number
  end

  def <=>(other)
    number <=> other.number
  end
end

v1 = Version.new(1)
v2 = Version.new(2)
v1 < v2   # => true

# Enumerable: each があれば map, select などが使える
class MyCollection
  include Enumerable

  def each
    yield 1
    yield 2
    yield 3
  end
end

MyCollection.new.map { |n| n * 2 }  # => [2, 4, 6]
```

---

## 8. 例外処理

### 基本

```ruby
begin
  # 危険な処理
  result = 10 / 0
rescue ZeroDivisionError => e
  puts "エラー: #{e.message}"
rescue StandardError => e
  puts "その他のエラー: #{e.message}"
else
  puts "成功: #{result}"
ensure
  puts "必ず実行される"
end
```

### 例外を発生させる

```ruby
def divide(a, b)
  raise ArgumentError, "ゼロで割れません" if b == 0
  a / b
end

# カスタム例外
class MyError < StandardError; end

raise MyError, "カスタムエラー"
```

### retry

```ruby
attempts = 0
begin
  attempts += 1
  # 失敗する可能性のある処理
  connect_to_server
rescue ConnectionError
  retry if attempts < 3
  raise
end
```

---

## 9. ファイル操作

### 読み込み

```ruby
# 全体を読み込み
content = File.read("file.txt")

# 行ごとに読み込み
File.foreach("file.txt") do |line|
  puts line
end

# 配列として読み込み
lines = File.readlines("file.txt")

# ブロック形式（自動でclose）
File.open("file.txt", "r") do |file|
  file.each_line { |line| puts line }
end
```

### 書き込み

```ruby
# 上書き
File.write("file.txt", "Hello, World!")

# 追記
File.write("file.txt", "追加テキスト", mode: "a")

# ブロック形式
File.open("file.txt", "w") do |file|
  file.puts "1行目"
  file.puts "2行目"
end
```

### ファイル・ディレクトリ操作

```ruby
# 存在確認
File.exist?("file.txt")
Dir.exist?("my_dir")

# ファイル情報
File.size("file.txt")
File.mtime("file.txt")  # 更新日時

# ディレクトリ操作
Dir.mkdir("new_dir")
Dir.glob("*.rb")        # パターンマッチ
Dir.entries(".")        # ディレクトリ内一覧

# パス操作
File.join("path", "to", "file.txt")  # => "path/to/file.txt"
File.dirname("/path/to/file.txt")    # => "/path/to"
File.basename("/path/to/file.txt")   # => "file.txt"
File.extname("file.txt")             # => ".txt"
```

---

## 10. Gem（パッケージ管理）

### Gemのインストール

```bash
# 単体インストール
gem install rails

# バージョン指定
gem install rails -v 7.0.0

# インストール済み一覧
gem list
```

### Bundler（プロジェクト単位の依存管理）

```bash
# Bundlerインストール
gem install bundler

# Gemfile作成
bundle init
```

**Gemfile:**
```ruby
source "https://rubygems.org"

gem "rails", "~> 7.0"
gem "pg"
gem "puma"

group :development, :test do
  gem "rspec"
  gem "rubocop"
end
```

```bash
# 依存関係インストール
bundle install

# Gemfile.lockに基づいてインストール
bundle install --deployment

# Gem追加
bundle add nokogiri

# 実行
bundle exec ruby script.rb
bundle exec rails server
```

---

## 11. よく使うイディオム

### 配列操作

```ruby
numbers = [1, 2, 3, 4, 5]

# map: 変換
numbers.map { |n| n * 2 }           # => [2, 4, 6, 8, 10]

# select: 絞り込み
numbers.select { |n| n.even? }      # => [2, 4]

# reject: 除外
numbers.reject { |n| n.even? }      # => [1, 3, 5]

# find: 最初の一致
numbers.find { |n| n > 3 }          # => 4

# reduce: 畳み込み
numbers.reduce(0) { |sum, n| sum + n }  # => 15
numbers.sum                              # => 15（よりシンプル）

# any? / all? / none?
numbers.any? { |n| n > 3 }          # => true
numbers.all? { |n| n > 0 }          # => true
numbers.none? { |n| n < 0 }         # => true

# グループ化
words = %w[apple apricot banana blueberry cherry]
words.group_by { |w| w[0] }
# => {"a"=>["apple", "apricot"], "b"=>["banana", "blueberry"], "c"=>["cherry"]}

# ソート
users.sort_by { |u| u[:age] }
users.sort_by(&:age)  # メソッド参照
```

### 安全なナビゲーション

```ruby
# &. (ぼっち演算子): nilチェック
user&.name          # userがnilならnil、そうでなければuser.name
user&.address&.city # チェーン可能

# || : デフォルト値
name = params[:name] || "Guest"

# ||= : 未定義/nil時に代入
@users ||= User.all

# &: : メソッドをブロックに変換
names = users.map(&:name)  # users.map { |u| u.name } と同じ
```

### 文字列操作

```ruby
# 空白処理
"  hello  ".strip      # => "hello"
"  hello  ".lstrip     # => "hello  "

# 分割・結合
"a,b,c".split(",")     # => ["a", "b", "c"]
["a", "b", "c"].join("-")  # => "a-b-c"

# 置換
"hello".gsub(/l/, "L")  # => "heLLo"

# フォーマット
"Name: %s, Age: %d" % ["太郎", 25]
sprintf("%.2f", 3.14159)  # => "3.14"
```

### ハッシュ操作

```ruby
hash = { a: 1, b: 2, c: 3 }

# 変換
hash.transform_keys(&:to_s)     # => {"a"=>1, "b"=>2, "c"=>3}
hash.transform_values { |v| v * 2 }  # => {a: 2, b: 4, c: 6}

# フィルタ
hash.select { |k, v| v > 1 }    # => {b: 2, c: 3}
hash.slice(:a, :b)              # => {a: 1, b: 2}
hash.except(:c)                 # => {a: 1, b: 2}

# デフォルト値付きハッシュ
counts = Hash.new(0)
counts[:a] += 1  # => 1（存在しないキーは0として扱う）
```

---

## 12. デバッグ方法

### 基本的な出力

```ruby
# 値の確認
p variable           # inspect形式で出力
pp complex_object    # 整形出力
puts variable.inspect

# 型の確認
variable.class       # => String, Array など
variable.is_a?(String)  # => true/false
variable.methods     # 利用可能なメソッド一覧
```

### binding.irb（Ruby 2.5+）

```ruby
def complex_method
  x = calculate_something
  binding.irb  # ここで実行が止まり、対話的にデバッグ可能
  process(x)
end
```

### デバッガ（debug gem, Ruby 3.1+標準）

```ruby
require "debug"

def my_method
  debugger  # ブレークポイント
  # ...
end
```

```bash
# コマンド
# n: 次の行
# s: ステップイン
# c: 続行
# p variable: 変数の値を表示
# q: 終了
```

### ログ出力

```ruby
require "logger"

logger = Logger.new(STDOUT)
logger.level = Logger::DEBUG

logger.debug "デバッグ情報"
logger.info "情報"
logger.warn "警告"
logger.error "エラー"
```

---

## クイックリファレンス

| 概念 | Ruby | 他言語との比較 |
|------|------|----------------|
| 配列 | `[1, 2, 3]` | JavaScript/Python と同様 |
| ハッシュ | `{ key: value }` | JavaScript のオブジェクト、Python の辞書 |
| nil | `nil` | JavaScript の `null`、Python の `None` |
| 文字列展開 | `"Hello #{name}"` | JavaScript の `` `Hello ${name}` `` |
| 範囲 | `1..5` | Python の `range(1, 6)` |
| シンボル | `:name` | Ruby固有（イミュータブルな識別子） |
| ブロック | `{ \|x\| x * 2 }` | JavaScript のアロー関数に近い |
| メソッド定義 | `def method_name; end` | `function` キーワードなし |
| クラス継承 | `class Child < Parent` | `extends` の代わりに `<` |
| モジュール | `include Module` | Java の interface + default method に近い |

---

## 参考リンク

- [Ruby公式ドキュメント](https://docs.ruby-lang.org/ja/)
- [Ruby Style Guide](https://github.com/rubocop/ruby-style-guide)
- [Ruby on Rails Guides](https://guides.rubyonrails.org/)
- [RubyGems](https://rubygems.org/)
