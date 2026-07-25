# Pry - 高機能 REPL / デバッガ

## 概要

PryはIRBの代替となる高機能なREPL（対話型シェル）です。構文ハイライト、コード補完、ソースコード閲覧、デバッグ機能など、開発に便利な機能を多数備えています。

> **🐍 Python相当:** REPL部分は **IPython**（高機能な対話型シェル）、デバッガ部分は **pdb / ipdb**（`pry-byebug`に相当）。

```bash
# インストール
gem install pry
gem install pry-byebug  # デバッグ機能追加

# Gemfile
group :development, :test do
  gem 'pry'
  gem 'pry-byebug'    # ステップ実行
  gem 'pry-rails'     # Rails統合
  gem 'pry-doc'       # C言語ソース閲覧
end
```

---

## 基本的な使い方

### REPL として使用

```bash
# 起動
pry

# ファイルを読み込んで起動
pry -r ./lib/my_app.rb

# 式を評価して終了
pry -e "puts 1 + 1"
```

```ruby
pry> 1 + 1
=> 2

pry> "hello".upcase
=> "HELLO"

pry> [1, 2, 3].map { |n| n * 2 }
=> [2, 4, 6]

# 最後の結果を参照
pry> _ * 2
=> [4, 8, 12]

# 複数行入力
pry> def hello
pry>   puts "Hello"
pry> end
=> :hello
```

### コード内にブレークポイント

```ruby
require 'pry'

def complex_calculation(x, y)
  result = x * y
  binding.pry  # ここで実行が停止
  result + 10
end

complex_calculation(5, 3)
```

**🐍 Python:** 任意の場所での対話停止は複数の方法がある。

```python
def complex_calculation(x, y):
    result = x * y
    breakpoint()  # Python 3.7+ の組み込み。pdb が起動する
    # import ipdb; ipdb.set_trace()  # ipdb を使う場合
    # import IPython; IPython.embed()  # 対話シェルとして埋め込む場合
    return result + 10
```

`binding.pry` はデバッグ操作と対話シェルを兼ねるが、Pythonでは `breakpoint()`（デバッグ操作）と `IPython.embed()`（対話シェル）で用途が分かれる点に注意。

---

## ナビゲーションコマンド

### ls - オブジェクトの調査

```ruby
# メソッド一覧
pry> ls String
# String#methods: ...
# String.methods: ...

# インスタンスの調査
pry> user = User.new(name: "Alice")
pry> ls user
# User#methods: name, email, save, ...
# instance variables: @name, @email

# 詳細オプション
pry> ls -m User           # メソッドのみ
pry> ls -M User           # クラスメソッドのみ
pry> ls -i user           # インスタンス変数のみ
pry> ls -c User           # 定数のみ
pry> ls -G pattern        # パターンでフィルタ
pry> ls -v User           # 詳細表示
```

**🐍 Python:** IPythonでは `dir(obj)` でメソッド/属性一覧、`vars(obj)` でインスタンス変数（`obj.__dict__`）を確認する。フィルタはタブ補完や `obj.<TAB>` で行う。

### cd - コンテキスト移動

```ruby
pry> user = User.new(name: "Alice")
pry> cd user
pry(#<User>)> @name
=> "Alice"
pry(#<User>)> name
=> "Alice"

# 戻る
pry(#<User>)> cd ..
pry>

# クラスに移動
pry> cd User
pry(User)> self
=> User

# ネスト
pry> nesting
# => [[0, #<User>], [1, User]]
```

### show-source - ソースコード表示

```ruby
# メソッドのソースを表示
pry> show-source User#save

# クラスのソースを表示
pry> show-source User

# 行番号付き
pry> show-source User#save -l

# 特定の行から
pry> show-source User#save -b
```

**🐍 Python:** IPythonでは `obj.method??`（ソース表示）、`obj.method?`（ドキュメント/シグネチャ表示）を使う。プログラム的には `import inspect; inspect.getsource(func)` でソースを取得できる。

### show-doc - ドキュメント表示

```ruby
# メソッドのドキュメント
pry> show-doc Array#map

# クラスのドキュメント
pry> show-doc String

# ? は show-doc の別名
pry> ? Array#map
```

### find-method - メソッド検索

```ruby
# メソッド名で検索
pry> find-method each Array
# => Array#each, Array#each_index, ...

# パターンで検索
pry> find-method /^to_/ String
```

---

## デバッグコマンド（pry-byebug）

### ステップ実行

```ruby
require 'pry-byebug'

def outer
  x = 10
  binding.pry
  inner(x)
end

def inner(n)
  n * 2
end
```

```ruby
# コマンド
pry> next          # 次の行（ステップオーバー）
pry> n             # next の短縮形

pry> step          # ステップイン（メソッド内に入る）
pry> s             # step の短縮形

pry> finish        # 現在のメソッドを抜ける
pry> f             # finish の短縮形

pry> continue      # 次のブレークポイントまで実行
pry> c             # continue の短縮形

pry> up            # コールスタックを上に移動
pry> down          # コールスタックを下に移動
pry> frame         # 現在のフレーム情報

pry> backtrace     # コールスタック表示
pry> bt            # backtrace の短縮形
```

**🐍 Python:** pdb / ipdb のコマンドはほぼ同じ短縮形を持つ。

```bash
n   # next: ステップオーバー
s   # step: ステップイン
c   # continue: 続行
r   # return: 現在の関数を抜ける（pry の finish に相当）
u / d  # up / down: コールスタック移動
w   # where: コールスタック表示（bt に相当）
```

### ブレークポイント

```ruby
# 行番号でブレークポイント設定
pry> break 20

# ファイルと行番号
pry> break app/models/user.rb:30

# メソッドにブレークポイント
pry> break User#save

# 条件付きブレークポイント
pry> break User#save if user.admin?

# ブレークポイント一覧
pry> breakpoints
pry> breaks

# 削除
pry> breakpoints delete 1
pry> breakpoints clear
```

### ウォッチ

```ruby
# 変数の監視
pry> watch @counter

# 式の監視
pry> watch user.status

# 一覧
pry> watch list

# 削除
pry> watch delete 1
```

---

## 編集コマンド

### edit - エディタで編集

```ruby
# メソッドを編集
pry> edit User#save

# ファイルを編集
pry> edit app/models/user.rb

# 現在のコンテキストで式を編集
pry> edit

# エディタ指定
pry> edit -e vim User#save
```

### play - コードを実行

```ruby
# ファイルの特定行を実行
pry> play -l 10..20 app/models/user.rb

# メソッドを実行
pry> play -m User#setup

# 最後の式を再実行
pry> play -r
```

### hist - 履歴

```ruby
# 履歴表示
pry> hist
pry> hist --tail 10

# 履歴を実行
pry> hist --replay 5

# 履歴検索
pry> hist -G pattern

# 履歴をクリア
pry> hist -c
```

---

## シェルコマンド

```ruby
# シェルコマンド実行
pry> .ls -la
pry> .git status

# シェルに切り替え
pry> shell-mode

# 環境変数
pry> ENV['PATH']

# システムコマンドの出力を使用
pry> files = `ls *.rb`.split("\n")
```

---

## カスタマイズ

### .pryrc 設定ファイル

```ruby
# ~/.pryrc
Pry.config.editor = 'code'  # エディタ設定

# プロンプトのカスタマイズ
Pry.config.prompt = Pry::Prompt.new(
  'custom',
  'custom prompt',
  [
    proc { |obj, nest, pry| "#{obj}> " },
    proc { |obj, nest, pry| "#{obj}* " }
  ]
)

# 履歴の保存
Pry.config.history_file = '~/.pry_history'
Pry.config.history_save = true

# カラーテーマ
Pry.config.color = true
Pry.config.theme = 'solarized'

# エイリアス
Pry.commands.alias_command 'c', 'continue'
Pry.commands.alias_command 'n', 'next'
Pry.commands.alias_command 's', 'step'

# 起動時に実行
Pry.config.hooks.add_hook(:when_started, :say_hi) do
  puts "Welcome to Pry!"
end
```

### カスタムコマンド

```ruby
# .pryrc にカスタムコマンドを追加
Pry::Commands.create_command 'hello' do
  description 'Say hello'

  def process
    output.puts "Hello, #{args.first || 'World'}!"
  end
end

# 使用
pry> hello Ruby
# Hello, Ruby!
```

---

## 実践的な使い方

### Rails での使用

```ruby
# Gemfile
group :development, :test do
  gem 'pry-rails'
  gem 'pry-byebug'
end
```

```bash
# rails console が pry になる
rails console

# コントローラでデバッグ
class UsersController < ApplicationController
  def show
    @user = User.find(params[:id])
    binding.pry  # ここで停止
    render :show
  end
end
```

### テストでのデバッグ

```ruby
# RSpec
it 'creates a user' do
  user = build(:user)
  binding.pry  # テスト中に停止
  expect(user).to be_valid
end
```

```bash
# テスト実行
bundle exec rspec spec/models/user_spec.rb
```

### オブジェクトの詳細調査

```ruby
pry> user = User.first

# 全メソッドを見る
pry> ls user -v

# 特定のパターンで検索
pry> ls user -G admin

# 継承チェーン
pry> user.class.ancestors

# ソースの場所
pry> user.method(:save).source_location

# ソースを見る
pry> show-source user.save
```

### 例外のデバッグ

```ruby
# 例外発生時にPryを起動
begin
  risky_operation
rescue => e
  binding.pry  # 例外の状態を調査
  raise
end

# pry-rescue gem を使用すると自動的に停止
# gem 'pry-rescue'
# rescue pry でプログラム実行
```

---

## 便利なコマンド一覧

| コマンド | 説明 |
|---------|------|
| `ls` | オブジェクトのメソッドや変数を表示 |
| `cd` | コンテキスト移動 |
| `show-source` | ソースコード表示 |
| `show-doc` | ドキュメント表示 |
| `edit` | エディタで編集 |
| `whereami` | 現在位置を表示 |
| `wtf?` | 最後の例外のバックトレース |
| `hist` | 履歴 |
| `exit` | Pryを終了 |
| `exit!` | プログラムを終了 |
| `!!!` | exit! の別名 |
| `.command` | シェルコマンド実行 |

### デバッグコマンド（pry-byebug）

| コマンド | 短縮形 | 説明 |
|---------|--------|------|
| `next` | `n` | ステップオーバー |
| `step` | `s` | ステップイン |
| `finish` | `f` | メソッドを抜ける |
| `continue` | `c` | 続行 |
| `break` | - | ブレークポイント設定 |
| `breakpoints` | `breaks` | ブレークポイント一覧 |
| `backtrace` | `bt` | コールスタック |
| `up` | - | スタック上へ |
| `down` | - | スタック下へ |

---

## 関連 Gem

| Gem | 説明 |
|-----|------|
| pry-byebug | デバッグ機能 |
| pry-rails | Rails統合 |
| pry-doc | Cソース閲覧 |
| pry-rescue | 例外時に自動停止 |
| pry-stack_explorer | スタック探索 |
| awesome_print | 見やすい出力 |

---

## 参考リンク

- [Pry公式Wiki](https://github.com/pry/pry/wiki)
- [Pry-byebug](https://github.com/deivid-rodriguez/pry-byebug)
- [Pry Cheatsheet](https://gist.github.com/lfender6445/9919357)

← [Gem一覧に戻る](README.md)
