# RuboCop - Linter / Formatter

## 概要

RuboCopはRubyの静的コード解析ツールです。コーディングスタイルの一貫性を保ち、潜在的なバグを検出します。

```bash
# インストール
gem install rubocop

# Gemfile
group :development do
  gem 'rubocop', require: false
  gem 'rubocop-performance', require: false
  gem 'rubocop-rspec', require: false
  gem 'rubocop-rails', require: false  # Railsプロジェクト
end
```

---

## 基本的な使い方

### コマンド

```bash
# 現在のディレクトリをチェック
rubocop

# 特定のファイル/ディレクトリ
rubocop app/models/user.rb
rubocop app/models/

# 自動修正（安全な修正のみ）
rubocop -a
rubocop --auto-correct

# 自動修正（すべて）
rubocop -A
rubocop --auto-correct-all

# 特定のCopのみ
rubocop --only Style/StringLiterals

# 特定のCopを除外
rubocop --except Style/Documentation

# 設定ファイル生成
rubocop --init
```

### 出力フォーマット

```bash
# デフォルト（シンプル）
rubocop

# 詳細
rubocop --format offenses

# JSON
rubocop --format json

# HTML
rubocop --format html -o report.html

# GitHub Actions用
rubocop --format github

# 複数フォーマット
rubocop --format simple --format html -o report.html
```

---

## 設定ファイル

### .rubocop.yml の基本

```yaml
# .rubocop.yml
AllCops:
  TargetRubyVersion: 3.3
  NewCops: enable
  Exclude:
    - 'vendor/**/*'
    - 'db/schema.rb'
    - 'bin/*'
    - 'node_modules/**/*'

# 拡張Gemの読み込み
require:
  - rubocop-performance
  - rubocop-rspec
  - rubocop-rails
```

### Copの設定

```yaml
# 無効化
Style/Documentation:
  Enabled: false

# パラメータ変更
Metrics/MethodLength:
  Max: 20
  CountAsOne:
    - array
    - hash
    - heredoc

Layout/LineLength:
  Max: 120
  AllowedPatterns:
    - '^\s*#'  # コメント行

# 特定ファイルで除外
Metrics/BlockLength:
  Exclude:
    - 'spec/**/*'
    - 'config/routes.rb'

# 特定ファイルのみ適用
Rails/HttpPositionalArguments:
  Include:
    - 'spec/controllers/**/*'
    - 'spec/requests/**/*'
```

### 継承

```yaml
# 他の設定ファイルを継承
inherit_from:
  - .rubocop_todo.yml
  - .rubocop_custom.yml

# リモートの設定を継承
inherit_from:
  - https://example.com/rubocop.yml

# Gemの設定を継承
inherit_gem:
  rubocop-shopify: rubocop.yml
```

### TODO ファイルの生成

```bash
# 現在の違反をTODOファイルに記録
rubocop --auto-gen-config

# 生成される .rubocop_todo.yml を継承
# .rubocop.yml
inherit_from: .rubocop_todo.yml
```

---

## 主要な Cop カテゴリ

### Layout（レイアウト）

```yaml
# インデント
Layout/IndentationWidth:
  Width: 2

# 空行
Layout/EmptyLines:
  Enabled: true

Layout/EmptyLinesAroundClassBody:
  EnforcedStyle: no_empty_lines

# スペース
Layout/SpaceAroundOperators:
  Enabled: true

Layout/SpaceInsideHashLiteralBraces:
  EnforcedStyle: space
```

```ruby
# 良い例
def method
  { key: value }
  array = [1, 2, 3]
  x = 1 + 2
end
```

### Style（スタイル）

```yaml
# 文字列リテラル
Style/StringLiterals:
  EnforcedStyle: double_quotes  # または single_quotes

# シンボル配列
Style/SymbolArray:
  EnforcedStyle: brackets  # %i[] vs [:a, :b]

# 条件式
Style/ConditionalAssignment:
  EnforcedStyle: assign_to_condition

# frozen_string_literal
Style/FrozenStringLiteralComment:
  EnforcedStyle: always

# ドキュメント
Style/Documentation:
  Enabled: false  # よく無効化される
```

```ruby
# frozen_string_literal: true

# 良い例
class User
  ROLES = %i[admin user guest].freeze

  def status
    active? ? "active" : "inactive"
  end
end
```

### Naming（命名）

```yaml
# メソッド名
Naming/MethodName:
  EnforcedStyle: snake_case

# 変数名
Naming/VariableName:
  EnforcedStyle: snake_case

# 定数名
Naming/ConstantName:
  Enabled: true

# 述語メソッド
Naming/PredicateName:
  ForbiddenPrefixes:
    - is_
    - has_
    - have_
```

```ruby
# 良い例
class UserAccount
  MAX_LOGIN_ATTEMPTS = 5

  def active?      # is_active ではなく
    @active
  end

  def admin_user?  # has_admin ではなく
    role == :admin
  end
end
```

### Metrics（メトリクス）

```yaml
# メソッドの長さ
Metrics/MethodLength:
  Max: 15
  CountComments: false

# クラスの長さ
Metrics/ClassLength:
  Max: 200
  CountComments: false

# 循環的複雑度
Metrics/CyclomaticComplexity:
  Max: 10

# ABC メトリクス
Metrics/AbcSize:
  Max: 20

# ブロックの長さ
Metrics/BlockLength:
  Max: 25
  Exclude:
    - 'spec/**/*'
```

### Lint（潜在的バグ）

```yaml
# 使用されていない変数
Lint/UnusedMethodArgument:
  Enabled: true
  AllowUnusedKeywordArguments: true

# デバッグコード
Lint/Debugger:
  Enabled: true

# 冗長な条件
Lint/RedundantStringCoercion:
  Enabled: true
```

```ruby
# Lint が検出する問題例

# Lint/UselessAssignment
x = 1
x = 2  # 最初の代入が無意味

# Lint/UnreachableCode
def method
  return
  puts "到達しない"
end

# Lint/ShadowingOuterLocalVariable
x = 1
[1, 2].each { |x| puts x }  # 外側のxを隠蔽
```

### Performance（パフォーマンス）

```yaml
require:
  - rubocop-performance

Performance/StringReplacement:
  Enabled: true

Performance/Casecmp:
  Enabled: true
```

```ruby
# 悪い例
str.gsub('a', 'b')
str.downcase == other.downcase

# 良い例
str.tr('a', 'b')
str.casecmp(other).zero?
```

---

## インラインでの無効化

```ruby
# 行単位
some_code # rubocop:disable Style/SomeCop

# ブロック単位
# rubocop:disable Metrics/MethodLength
def long_method
  # ...
end
# rubocop:enable Metrics/MethodLength

# 複数のCop
# rubocop:disable Style/StringLiterals, Layout/LineLength
code_here
# rubocop:enable Style/StringLiterals, Layout/LineLength

# すべてのCop
# rubocop:disable all
legacy_code
# rubocop:enable all
```

---

## CI/CD との連携

### GitHub Actions

```yaml
# .github/workflows/rubocop.yml
name: RuboCop

on: [push, pull_request]

jobs:
  rubocop:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: ruby/setup-ruby@v1
        with:
          ruby-version: '3.3'
          bundler-cache: true
      - name: Run RuboCop
        run: bundle exec rubocop --format github
```

### Pre-commit フック

```bash
# .git/hooks/pre-commit
#!/bin/sh
bundle exec rubocop --force-exclusion $(git diff --cached --name-only --diff-filter=ACM | grep '\.rb$')
```

### VSCode 連携

```json
// .vscode/settings.json
{
  "ruby.rubocop.executePath": "",
  "ruby.rubocop.onSave": true,
  "editor.formatOnSave": true,
  "[ruby]": {
    "editor.defaultFormatter": "misogi.ruby-rubocop"
  }
}
```

---

## おすすめ設定

### 新規プロジェクト

```yaml
# .rubocop.yml
AllCops:
  TargetRubyVersion: 3.3
  NewCops: enable
  SuggestExtensions: false
  Exclude:
    - 'vendor/**/*'
    - 'tmp/**/*'

require:
  - rubocop-performance

# よく調整するもの
Layout/LineLength:
  Max: 120

Metrics/MethodLength:
  Max: 20

Metrics/BlockLength:
  Exclude:
    - 'spec/**/*'
    - 'Rakefile'
    - '**/*.rake'

Style/Documentation:
  Enabled: false

Style/FrozenStringLiteralComment:
  Enabled: true
  EnforcedStyle: always

Style/StringLiterals:
  EnforcedStyle: double_quotes

Style/TrailingCommaInArrayLiteral:
  EnforcedStyleForMultiline: trailing_comma

Style/TrailingCommaInHashLiteral:
  EnforcedStyleForMultiline: trailing_comma
```

### レガシープロジェクト

```bash
# 1. 現状をTODOに記録
rubocop --auto-gen-config --auto-gen-only-exclude

# 2. .rubocop.yml で継承
inherit_from: .rubocop_todo.yml

# 3. 少しずつ修正していく
rubocop -a  # 安全な自動修正
```

---

## 拡張 Gem

| Gem | 説明 |
|-----|------|
| rubocop-performance | パフォーマンス関連 |
| rubocop-rspec | RSpec用ルール |
| rubocop-rails | Rails用ルール |
| rubocop-minitest | Minitest用 |
| rubocop-rake | Rake用 |
| rubocop-thread_safety | スレッドセーフティ |
| rubocop-shopify | Shopifyのスタイルガイド |
| standard | StandardRB（厳選ルール） |

---

## 参考リンク

- [RuboCop公式ドキュメント](https://docs.rubocop.org/)
- [Ruby Style Guide](https://rubystyle.guide/)
- [全Cop一覧](https://docs.rubocop.org/rubocop/cops.html)

← [Gem一覧に戻る](README.md)
