# Rails 第1章: 概要と環境構築

Ruby on Rails は Ruby 製のフルスタック Web アプリケーションフレームワークです。本章では Rails の設計思想、MVC アーキテクチャ、環境構築、`rails new` によるプロジェクト生成、ディレクトリ構成、主要コマンドまでを一通り解説します。最後に scaffold を使った最小の動作確認まで行い、Rails 開発の全体像をつかみます。

---

## 目次
1. [Railsとは・設計思想](#1-railsとは設計思想)
2. [MVCアーキテクチャとリクエストのライフサイクル](#2-mvcアーキテクチャとリクエストのライフサイクル)
3. [前提環境とインストール](#3-前提環境とインストール)
4. [rails new とオプション](#4-rails-new-とオプション)
5. [ディレクトリ構成](#5-ディレクトリ構成)
6. [よく使う rails コマンド](#6-よく使う-rails-コマンド)
7. [開発サーバとコンソール](#7-開発サーバとコンソール)
8. [環境（development / test / production）](#8-環境development--test--production)
9. [設定と機密情報の管理](#9-設定と機密情報の管理)
10. [Gemfile と Bundler](#10-gemfile-と-bundler)
11. [最小の動作確認（scaffold）](#11-最小の動作確認scaffold)

---

## 1. Railsとは・設計思想

Ruby on Rails（以下 Rails）は、2004 年に David Heinemeier Hansson（DHH）が公開した Ruby 製の Web アプリケーションフレームワークです。データベース連携、ルーティング、HTML の描画、メール送信、バックグラウンドジョブ、WebSocket まで、Web アプリに必要な要素を標準で備えた**フルスタック**フレームワークです。

### フルスタックの構成要素

Rails は複数のサブフレームワーク（コンポーネント）の集合体です。

| コンポーネント | 役割 |
|--------------|------|
| Action Controller | リクエストの受付とレスポンスの制御（コントローラ層） |
| Action View | HTML などのビュー描画（テンプレート） |
| Active Record | データベースとの ORM（モデル層） |
| Active Model | モデルの共通機能（バリデーション等）を非DBクラスにも提供 |
| Action Mailer | メール送信 |
| Action Mailbox | 受信メールの処理 |
| Active Job | バックグラウンドジョブの抽象化 |
| Active Storage | ファイルアップロードとクラウドストレージ連携 |
| Action Cable | WebSocket によるリアルタイム通信 |
| Action Text | リッチテキスト編集 |
| Active Support | 各種ユーティリティ（Ruby の拡張） |

### 3つの設計思想

Rails の生産性は、以下の設計哲学に支えられています。

- **DRY（Don't Repeat Yourself）**
  同じ知識や情報をシステム内で二重に持たない。一箇所を直せば全体に反映されるようにコードを組む。

- **設定より規約（CoC: Convention over Configuration）**
  命名や配置の「規約」に従えば設定を書く必要がない。たとえば `Article` モデルは `articles` テーブルに自動でひも付き、設定ファイルは不要。規約から外れたいときだけ設定を書く。

- **RESTful 設計**
  リソース（記事、ユーザーなど）を中心に、HTTP メソッド（GET / POST / PATCH / DELETE）と URL の組み合わせで操作を表現する。ルーティングやコントローラの構造がこの思想に沿って設計されている。

```
規約に従う例:
  モデル名   Article        （単数・キャメルケース）
  テーブル名 articles       （複数・スネークケース）※自動対応
  コントローラ ArticlesController
  ビュー     app/views/articles/
```

**🎯 Django との比較（フレームワーク全体像）:**

RailsもDjangoも、ORM・ルーティング・テンプレート・管理機能などを標準搭載した**フルスタック**フレームワークです。ただし設計思想が対照的です。

- **Rails:「設定より規約」** — 規約に従えば設定はほぼ不要（`Article` → `articles` テーブルが自動対応）。
- **Django:「明示的設定」** — 接続や機能の有効化を `settings.py` に**明示的に書く**ことを重視します。また 1 プロジェクトを機能単位の**アプリ（app）**に分割し、`INSTALLED_APPS` に登録して組み立てるのが基本スタイルです。

---

## 2. MVCアーキテクチャとリクエストのライフサイクル

Rails は **MVC（Model-View-Controller）** パターンを採用しています。役割を分離することで、保守性とテスト容易性を高めます。

| 層 | 役割 | 主なディレクトリ |
|----|------|----------------|
| Model | データとビジネスロジック、DB とのやり取り | `app/models/` |
| View | ユーザーに見せる画面（HTML など）の生成 | `app/views/` |
| Controller | リクエストを受け、モデルとビューを仲介 | `app/controllers/` |

**🎯 Django（MTV アーキテクチャ）との比較:**

Django も MVC 系ですが、独自に **MTV（Model-Template-View）** と呼びます。用語が Rails とズレるため注意が必要です。

| Django の呼称 | 実際の役割 | Rails での相当 |
|--------------|-----------|---------------|
| Model（`models.py`） | データと DB とのやり取り | Model |
| Template（`.html`） | 画面（HTML）の生成 | **View** |
| View（`views.py`） | リクエストを受け処理を仲介 | **Controller** |

つまり Django の「View（`views.py`）」は Rails の**コントローラ**に相当し、Rails の「View（テンプレート）」は Django の**Template**に相当します。ルーティングは Rails が `config/routes.rb` に集約するのに対し、Django は各アプリの `urls.py` で `path()` を使って定義します。

### MVCの関係図

```
        ┌─────────────────────────────────────────────┐
        │                 ブラウザ                     │
        └───────────────┬──────────────▲──────────────┘
                        │ リクエスト   │ レスポンス(HTML)
                        ▼              │
        ┌───────────────────────────────────────────┐
        │              Router (routes.rb)            │
        │        URL + HTTP メソッド → 振り分け      │
        └───────────────┬───────────────────────────┘
                        ▼
        ┌───────────────────────────────────────────┐
        │              Controller                    │
        │        アクション（index, show ...）        │
        └───────┬───────────────────────▲───────────┘
                │ 問い合わせ            │ データ
                ▼                       │
        ┌───────────────┐       ┌──────┴────────────┐
        │     Model     │◀─────▶│      View         │
        │ (Active Record)│      │ (ERB テンプレート) │
        └───────┬───────┘       └───────────────────┘
                │ SQL
                ▼
        ┌───────────────┐
        │   Database    │
        └───────────────┘
```

### リクエストのライフサイクル

1 回のリクエストがレスポンスになるまでの流れは次のとおりです。

```
① ブラウザ
     │  GET /articles/1 を送信
     ▼
② Router（config/routes.rb）
     │  URL とメソッドを解析し、articles#show に振り分け
     ▼
③ Controller（ArticlesController#show）
     │  Middleware を通過後、アクションを実行
     ▼
④ Model / DB（Article.find(1)）
     │  Active Record が SQL を発行してレコードを取得
     ▼
⑤ View（app/views/articles/show.html.erb）
     │  取得したデータを埋め込み HTML を組み立て
     ▼
⑥ レスポンス
        HTTP ステータスと HTML をブラウザへ返す
```

途中で Rack ミドルウェア（ログ、セッション、CSRF 対策など）を通過します。ミドルウェアの一覧は `bin/rails middleware` で確認できます。

---

## 3. 前提環境とインストール

### 必要なソフトウェア

| ソフトウェア | 用途 | 目安バージョン |
|------------|------|--------------|
| Ruby | 実行環境 | 3.2 以上（Rails 8 は 3.2 以上が必須） |
| SQLite / PostgreSQL / MySQL | データベース | いずれか。開発は SQLite で十分 |
| Node.js | JS のビルドが必要な場合のみ | 18 以上（importmap 構成では不要） |
| Git | バージョン管理 | 任意（推奨） |

Rails 7 以降は JavaScript のバンドラとして **importmap** を既定に採用しており、この構成なら Node.js は不要です。`--javascript=esbuild` などバンドラを使う構成や、`--css=bootstrap` のように CSS バンドラを使う構成を選んだ場合にのみ Node.js（と Yarn/npm）が必要になります。

### Rails のインストール

Rails は gem として配布されます。

```bash
# Rails をインストール（最新安定版）
gem install rails

# バージョンを指定してインストール
gem install rails -v 8.0.0

# インストールされた Rails のバージョンを確認
rails -v
# => Rails 8.0.0
```

```bash
# Ruby / RubyGems / Bundler のバージョンも確認しておく
ruby -v
gem -v
bundle -v
```

**🎯 Django のインストール:**

Django は pip で配布されるパッケージです。プロジェクトごとに**仮想環境（venv）**を作って隔離するのが一般的です。

```bash
# 仮想環境を作成して有効化
python -m venv .venv
source .venv/bin/activate     # Windows は .venv\Scripts\activate

# Django をインストール（バージョン指定も可能）
pip install django
pip install "django==5.0"

# バージョン確認
django-admin --version
```

Rails の rbenv/asdf ＋ bundler が「Ruby 本体のバージョン管理」と「gem のプロジェクト隔離」を担うのに対し、Python では venv がプロジェクトごとの隔離を担います。

> rbenv や asdf などのバージョン管理ツールを使っている場合、`gem install rails` の後に `rbenv rehash` が必要になることがあります。

---

## 4. rails new とオプション

新規プロジェクトは `rails new` コマンドで生成します。指定した名前のディレクトリが作られ、雛形一式の生成と `bundle install` が自動実行されます。

```bash
# 最小構成の例（SQLite・importmap がデフォルト）
rails new myapp

# PostgreSQL を使い、API 専用アプリとして作成
rails new myapp --database=postgresql --api

# CSS に Tailwind、テストファイルは生成しない
rails new myapp --css=tailwind --skip-test
```

**🎯 Django のプロジェクト/アプリ生成:**

Django では**プロジェクト**の生成と、機能単位の**アプリ**の生成が 2 段階に分かれます。

```bash
# プロジェクト（設定・全体の入れ物）を生成
django-admin startproject myapp
cd myapp

# 機能単位のアプリを生成（例: blog）
python manage.py startapp blog
```

`rails new` が 1 コマンドで一式（`bundle install` 込み）を用意するのに対し、Django は「プロジェクト作成 → アプリ作成 → `settings.py` の `INSTALLED_APPS` に登録」という明示的な組み立てが前提です。`--api` のような専用構成オプションはなく、必要な機能を後から足していきます。

### 主要オプション一覧

| オプション | 短縮形 | 説明 |
|-----------|-------|------|
| `--database=NAME` | `-d` | DB を指定（`sqlite3`（既定）/ `postgresql` / `mysql` / `trilogy` など） |
| `--api` | | API 専用構成。ビューやアセット関連を省いた軽量な雛形 |
| `--css=NAME` | `-c` | CSS 処理系（`tailwind` / `bootstrap` / `bulma` / `postcss` / `sass`） |
| `--javascript=NAME` | `-j` | JS 手法（`importmap`（既定）/ `esbuild` / `bun` / `webpack` / `rollup`） |
| `--asset-pipeline=NAME` | `-a` | アセットパイプライン（`propshaft`（既定）/ `sprockets`） |
| `--skip-test` | `-T` | 標準のテスト（Minitest）ファイルを生成しない |
| `--minimal` | | 最小構成。多くのコンポーネントを省いた雛形を生成 |
| `--skip-bundle` | | 生成後に `bundle install` を実行しない |
| `--skip-active-record` | `-O` | Active Record（DB 連携）を除外 |
| `--skip-git` | | `.gitignore` などの Git 関連ファイルを生成しない |
| `--devcontainer` | | Dev Container 用の設定ファイルを生成（Rails 7.2 以降） |

利用可能なオプションは `rails new --help` で全て確認できます。

```bash
# オプションの全一覧を表示
rails new --help
```

Rails 7.2 以降は既定で `Dockerfile`、GitHub Actions 用の CI 設定、RuboCop・Brakeman（静的解析）の設定が生成されます。Rails 8 ではさらに、認証機能のジェネレータや、キャッシュ・ジョブ・キューを DB で扱う Solid 系の gem、デプロイツール Kamal 2 の設定が標準で含まれます。

---

## 5. ディレクトリ構成

`rails new` で生成される主なディレクトリとファイルは次のとおりです。

```
myapp/
├── app/                  # アプリケーション本体（MVC の中心）
│   ├── assets/           # CSS・画像などの静的アセット
│   ├── channels/         # Action Cable（WebSocket）
│   ├── controllers/      # コントローラ
│   ├── helpers/          # ビュー用の補助メソッド
│   ├── javascript/       # JavaScript（importmap のエントリ等）
│   ├── jobs/             # Active Job のジョブ
│   ├── mailers/          # Action Mailer のメーラー
│   ├── models/           # モデル（Active Record）
│   └── views/            # ビューテンプレート（.erb など）
├── bin/                  # 実行スクリプト（rails, setup など）
├── config/               # 設定ファイル群
│   ├── environments/     # 環境ごとの設定（development/test/production）
│   ├── initializers/     # 起動時に読み込む初期化コード
│   ├── locales/          # 国際化（i18n）の翻訳ファイル
│   ├── application.rb    # アプリ全体の設定
│   ├── routes.rb         # ルーティング定義
│   ├── database.yml      # DB 接続設定
│   └── credentials.yml.enc # 暗号化された機密情報
├── db/                   # DB 関連（マイグレーション・スキーマ・seed）
│   ├── migrate/          # マイグレーションファイル
│   ├── schema.rb         # 現在のスキーマ（自動生成）
│   └── seeds.rb          # 初期データ投入スクリプト
├── lib/                  # 自作ライブラリ・Rake タスク
├── log/                  # ログファイル
├── public/               # 公開ディレクトリ（そのまま配信される）
├── storage/              # Active Storage のローカル保存先
├── test/                 # テストコード（Minitest）
├── tmp/                  # 一時ファイル・キャッシュ
├── vendor/               # サードパーティのコード
├── Gemfile               # 依存 gem の定義
├── Gemfile.lock          # 依存 gem の確定版バージョン
└── config.ru             # Rack の起動設定
```

**🎯 Django のディレクトリ構成:**

Django は「プロジェクト」ディレクトリと「アプリ」ディレクトリに分かれ、MVC の各要素は**アプリ内のファイル**として配置されます。

```
myapp/                     # プロジェクトルート
├── manage.py              # 各種コマンドの入口（bin/rails 相当）
├── myapp/                 # プロジェクト設定パッケージ
│   ├── settings.py        # 全体設定（config/ 相当を1ファイルに集約）
│   ├── urls.py            # ルート URL 定義（config/routes.rb 相当）
│   └── wsgi.py / asgi.py  # アプリケーションサーバの入口（config.ru 相当）
└── blog/                  # 機能単位のアプリ
    ├── models.py          # モデル（app/models/ 相当）
    ├── views.py           # ビュー＝処理（app/controllers/ 相当）
    ├── urls.py            # このアプリの URL 定義
    ├── admin.py           # 管理画面へのモデル登録
    ├── migrations/        # マイグレーション（db/migrate/ 相当）
    └── templates/         # HTML テンプレート（app/views/ 相当）
```

Rails が層（models/controllers/views）ごとにディレクトリを分けるのに対し、Django は機能（アプリ）ごとに `models.py` / `views.py` / `urls.py` などをまとめる点が対照的です。

### app/ 配下の役割

| ディレクトリ | 役割 |
|------------|------|
| `models/` | データとビジネスロジック。1 モデル = 原則 1 テーブル |
| `controllers/` | リクエストを処理するアクションを定義 |
| `views/` | 画面（HTML）のテンプレート。コントローラごとにサブフォルダを持つ |
| `helpers/` | ビューで使う整形メソッドなどを定義 |
| `jobs/` | 非同期処理（メール送信、集計など）のジョブ |
| `mailers/` | メール本文と送信ロジック |
| `channels/` | Action Cable のチャンネル（リアルタイム通信） |

### 主要ファイル

| パス | 役割 |
|------|------|
| `config/routes.rb` | URL とコントローラ・アクションの対応を定義 |
| `config/application.rb` | 全環境共通のアプリ設定 |
| `config/database.yml` | 環境ごとの DB 接続情報 |
| `db/schema.rb` | DB の現在の構造（マイグレーションから自動生成） |
| `db/seeds.rb` | 初期データを投入するスクリプト |
| `Gemfile` | 使用する gem の一覧 |

### zeitwerk による自動読み込み

Rails は **zeitwerk** というオートローダーを採用しています。ファイル名とクラス名を規約どおりに対応させておけば、`require` を書かずにクラスを参照できます。

```
app/models/article.rb          → Article
app/models/blog/comment.rb     → Blog::Comment
app/controllers/articles_controller.rb → ArticlesController
```

ファイル名（スネークケース）とクラス名（キャメルケース）を一致させることが規約です。命名がずれると `NameError` になります。

---

## 6. よく使う rails コマンド

Rails のコマンドは、グローバルの `rails` と、プロジェクト内の `bin/rails` の 2 通りで実行できます。**プロジェクト内では `bin/rails` を使うのが推奨**です。プロジェクトが依存する Rails のバージョン（Gemfile.lock）で確実に実行されるためです。

| コマンド | 短縮形 | 説明 |
|---------|-------|------|
| `bin/rails server` | `s` | 開発サーバを起動 |
| `bin/rails console` | `c` | 対話コンソールを起動 |
| `bin/rails generate` | `g` | モデル・コントローラなどの雛形を生成 |
| `bin/rails destroy` | `d` | `generate` で作ったファイルを削除 |
| `bin/rails db:migrate` | | マイグレーションを実行し DB を更新 |
| `bin/rails db:seed` | | `db/seeds.rb` を実行し初期データを投入 |
| `bin/rails routes` | | ルーティング一覧を表示 |
| `bin/rails dbconsole` | `db` | DB のコマンドラインクライアントを起動 |
| `bin/rails test` | `t` | テストを実行 |
| `bin/rails runner` | `r` | アプリ環境下で Ruby コードを実行 |
| `bin/rails credentials:edit` | | 暗号化された機密情報を編集 |
| `bin/rails about` | | Ruby / Rails のバージョンなど環境情報を表示 |

**🎯 Django のコマンド（`python manage.py`）:**

Django ではプロジェクト内のコマンドを `python manage.py <サブコマンド>` で実行します（`bin/rails` 相当）。

| Rails | Django | 説明 |
|-------|--------|------|
| `bin/rails server` | `python manage.py runserver` | 開発サーバ起動 |
| `bin/rails console` | `python manage.py shell` | 対話コンソール |
| `bin/rails generate ...` | `python manage.py startapp <app>` | Django は雛形の自動生成が乏しく、多くは手動でファイルを作成 |
| （マイグレーション生成） | `python manage.py makemigrations` | モデルの変更から migration を生成 |
| `bin/rails db:migrate` | `python manage.py migrate` | マイグレーションを適用 |
| `bin/rails routes` | 標準にはなし（`django-extensions` の `show_urls`） | URL 一覧表示 |
| `bin/rails runner` | `python manage.py shell -c "..."` | アプリ環境下でコード実行 |

Rails が migration の生成を `generate migration`、適用を `db:migrate` で行うのに対し、Django は `makemigrations`（生成）と `migrate`（適用）の 2 コマンドで対応します。

```bash
# ジェネレータの例
bin/rails generate model Article title:string body:text
bin/rails generate controller Articles index show

# runner でワンライナー実行（アプリのモデルが使える）
bin/rails runner "puts Article.count"

# ルーティング一覧（特定のコントローラで絞り込み）
bin/rails routes -c articles
```

```bash
# データベース関連の主なタスク
bin/rails db:create     # DB を作成
bin/rails db:migrate    # マイグレーションを適用
bin/rails db:rollback   # 直前のマイグレーションを取り消し
bin/rails db:seed       # 初期データを投入
bin/rails db:prepare    # 未作成なら作成し、必要なら migrate まで実行
```

`bin/rails --help` でタスクの一覧を確認できます。

---

## 7. 開発サーバとコンソール

### 開発サーバの起動

```bash
# サーバを起動（既定は localhost:3000）
bin/rails server

# 短縮形
bin/rails s
```

起動後、ブラウザで `http://localhost:3000` にアクセスすると Rails の初期ページが表示されます。停止は `Ctrl + C` です。

```bash
# ポートを変更する（-p / --port）
bin/rails server -p 4000

# バインドするアドレスを変更する（-b / --binding）
# 0.0.0.0 にすると同一ネットワークの他端末やコンテナ外からもアクセス可能
bin/rails server -b 0.0.0.0

# 環境を指定して起動（-e / --environment）
bin/rails server -e production
```

> **🎯 Django:** `python manage.py runserver`（`rails server` 相当。既定ポートは **8000**）。ポートやバインド先は `python manage.py runserver 0.0.0.0:8000` のように引数で指定します。

| オプション | 短縮形 | 説明 |
|-----------|-------|------|
| `--port` | `-p` | 待ち受けポート（既定 3000） |
| `--binding` | `-b` | バインドする IP アドレス |
| `--environment` | `-e` | 実行環境（既定 development） |
| `--daemon` | `-d` | バックグラウンド（デーモン）で起動 |

### コンソール

`bin/rails console` は、アプリの全クラスを読み込んだ状態の対話環境（IRB ベース）です。モデルを直接操作でき、動作確認やデバッグに便利です。

```bash
# コンソールを起動
bin/rails console

# 短縮形
bin/rails c
```

```ruby
# コンソール内でモデルを操作する例
irb> Article.count
=> 0
irb> article = Article.create(title: "初投稿", body: "本文です")
irb> Article.first.title
=> "初投稿"
```

> **🐍 Python:** Django の対話コンソールは `python manage.py shell`（`rails console` 相当）。素の Python シェルに Django 環境を読み込んだもので、`from blog.models import Article` のようにモデルを import して使います。`django-extensions` を入れると、モデルを自動 import する `shell_plus` も利用できます。なお、終了時に自動ロールバックする Rails の `--sandbox` に相当する標準機能はありません。

#### サンドボックスモード

`--sandbox` を付けると、コンソール終了時にすべての DB 変更が自動でロールバックされます。本番データを壊さずに試したいときに有効です。

```bash
# 終了時に変更を巻き戻す
bin/rails console --sandbox
```

```ruby
# sandbox 内で作成したレコードは exit 時に取り消される
irb> Article.create(title: "実験", body: "test")
irb> exit   # ここで作成が rollback される
```

---

## 8. 環境（development / test / production）

Rails には 3 つの標準環境があり、それぞれ独立した設定・データベースを持ちます。

| 環境 | 用途 | 特徴 |
|------|------|------|
| development | 開発 | コード変更を都度読み込み、エラー詳細を表示、キャッシュ無効 |
| test | 自動テスト | テスト専用の DB を使用、実行のたびにデータをリセット |
| production | 本番 | 高速化のためコードをキャッシュ、エラー詳細は非表示 |

### 環境ごとの設定ファイル

環境ごとの設定は `config/environments/` 配下にあります。

```
config/environments/
├── development.rb   # 開発環境の設定
├── test.rb          # テスト環境の設定
└── production.rb    # 本番環境の設定
```

```ruby
# config/environments/development.rb（抜粋・イメージ）
Rails.application.configure do
  # クラスの変更を都度読み込む（開発を快適にする）
  config.cache_classes = false

  # 詳細なエラーページを表示
  config.consider_all_requests_local = true
end
```

### RAILS_ENV

どの環境で動かすかは環境変数 `RAILS_ENV`（既定は `development`）で決まります。多くのコマンドは `-e` オプションでも指定できます。

```bash
# 環境変数で本番環境を指定してマイグレーション
RAILS_ENV=production bin/rails db:migrate

# コマンドのオプションで指定
bin/rails console -e test
```

現在の環境はコード中で `Rails.env` から参照できます。

```ruby
Rails.env             # => "development"
Rails.env.production? # => false
```

**🎯 Django の環境の扱い:**

Django には Rails のような development/test/production の 3 環境が**あらかじめ用意されているわけではありません**。代わりに、どの設定モジュールを読み込むかを環境変数 `DJANGO_SETTINGS_MODULE`（例: `myapp.settings.production`）で切り替え、`settings.py` を `base.py` / `development.py` / `production.py` などに分割するのが定石です。開発時のエラー詳細表示などは `settings.py` の `DEBUG = True/False` で制御します。

---

## 9. 設定と機密情報の管理

### アプリケーション設定

全環境共通の設定は `config/application.rb`、環境固有の設定は `config/environments/*.rb` に書きます。設定値は `Rails.application.config`（コード中では `config`）を通じて読み書きします。

```ruby
# config/application.rb
module Myapp
  class Application < Rails::Application
    # タイムゾーンを東京に
    config.time_zone = "Tokyo"

    # デフォルトのロケールを日本語に
    config.i18n.default_locale = :ja
  end
end
```

```ruby
# 設定値をコード中で参照する
Rails.application.config.time_zone   # => "Tokyo"
```

**🎯 Django:** Django の設定は原則すべて `settings.py`（1 ファイル、または分割したモジュール群）に集約します。上記のタイムゾーンやロケールは、`settings.py` の `TIME_ZONE = "Asia/Tokyo"`、`LANGUAGE_CODE = "ja"`、`USE_TZ = True` などの変数で指定します。Rails が「規約＋環境別ファイル」で設定を分散させるのに対し、Django は「Python の変数として明示的に書く」点が対照的です。

### 機密情報（credentials）

API キーやパスワードなどの秘密情報は、暗号化された **credentials** に保存します。平文をリポジトリにコミットせずに済む仕組みです。

```
┌──────────────────────────────┐
│  config/master.key           │  ← 復号鍵（.gitignore 済み・共有しない）
└──────────────┬───────────────┘
               │ 復号 / 暗号化
               ▼
┌──────────────────────────────┐
│  config/credentials.yml.enc  │  ← 暗号化された機密情報（コミット可）
└──────────────────────────────┘
```

```bash
# credentials を編集（既定エディタが開く）
bin/rails credentials:edit
```

```yaml
# credentials:edit で開かれる中身（復号された状態）のイメージ
secret_key_base: xxxxxxxxxxxxxxxxxxxx
aws:
  access_key_id: AKIAxxxxxxxx
  secret_access_key: yyyyyyyyyyyy
```

保存すると内容は再び暗号化され、`config/credentials.yml.enc` に書き込まれます。コード中では次のように参照します。

```ruby
# ネストした値はドットでたどる
Rails.application.credentials.aws[:access_key_id]
Rails.application.credentials.secret_key_base
```

- `config/master.key`（または環境変数 `RAILS_MASTER_KEY`）が復号に必要です。**このキーはリポジトリにコミットしない**（既定で `.gitignore` に登録済み）。
- 環境ごとに分けたい場合は `bin/rails credentials:edit --environment production` を使うと、`config/credentials/production.yml.enc` と対応する鍵が作られます。

DB のパスワードなど、鍵管理サービスと連携させたい値は環境変数（`ENV["..."]`）で渡す方法もよく使われます。用途に応じて credentials と環境変数を使い分けます。

**🎯 Django の機密情報管理:**

Django には credentials.yml.enc / master.key のような**暗号化ファイルの仕組みは標準にはありません**。プロジェクト生成時に `settings.py` へ自動生成される `SECRET_KEY`（Rails の `secret_key_base` 相当）をはじめ、API キーや DB パスワードは**環境変数**で渡し、`os.environ` や `django-environ`（`.env` ファイルを読み込む）経由で参照するのが定石です。

```python
# settings.py（django-environ を使う例）
import environ
env = environ.Env()
SECRET_KEY = env("SECRET_KEY")
DATABASES = {"default": env.db()}   # DATABASE_URL 環境変数から構築
```

`.env` や鍵ファイルをリポジトリにコミットしない点は、Rails の master.key と同じ考え方です。

---

## 10. Gemfile と Bundler

Rails アプリの依存ライブラリ（gem）は **Bundler** で管理します。使用する gem は `Gemfile` に列挙します。

```ruby
# Gemfile（抜粋）
source "https://rubygems.org"

# Rails 本体
gem "rails", "~> 8.0.0"

# 既定のデータベース
gem "sqlite3", ">= 2.1"

# アプリケーションサーバ
gem "puma", ">= 5.0"

# JS を importmap で扱う
gem "importmap-rails"

# アセット配信
gem "propshaft"

# 特定の環境でのみ使う gem はグループにまとめる
group :development, :test do
  # デバッグ用
  gem "debug", platforms: %i[ mri windows ]
end
```

### 主なコマンド

```bash
# Gemfile の内容をインストールし、Gemfile.lock を生成/更新
bundle install

# gem を最新化（バージョン制約の範囲で更新）
bundle update

# プロジェクトの gem バージョンで実行する（重要）
bundle exec rake db:migrate
```

- `bundle install` を実行すると、依存関係を解決した確定版バージョンが **`Gemfile.lock`** に記録されます。このファイルもコミットすることで、チーム全員・本番環境で同じバージョンが再現されます。
- `bundle exec` は、システムにインストールされた別バージョンではなく、`Gemfile.lock` で固定されたバージョンで gem のコマンドを実行します。
- `bin/rails` は内部で Bundler を経由するため、Rails のコマンドを打つときは通常 `bundle exec` を明示する必要はありません。

**🎯 Django（Python）の依存管理:**

Python では依存パッケージを `requirements.txt` に列挙し、pip でインストールするのが基本です（Rails の Gemfile/Bundler 相当）。

```bash
# 現在の環境の依存を書き出す（バージョン固定に使う）
pip freeze > requirements.txt

# requirements.txt からインストール
pip install -r requirements.txt
```

`Gemfile.lock` による厳密なバージョン固定に対応させたい場合は、**Poetry** や **pipenv**（`poetry.lock` / `Pipfile.lock`）を使うことが多く、これらは依存解決とロックファイル生成を Bundler に近い形で行います。仮想環境（venv）による隔離が、Bundler のプロジェクト単位の管理に相当します。

---

## 11. 最小の動作確認（scaffold）

最後に、**scaffold** を使って CRUD（作成・参照・更新・削除）画面を一気に生成し、Rails が動くことを確認します。scaffold はモデル・コントローラ・ビュー・ルーティング・マイグレーションをまとめて作るジェネレータです。

```bash
# 1. Article リソースの一式を生成
bin/rails generate scaffold Article title:string body:text
```

このコマンドで、モデル `Article`、`ArticlesController`、一覧・詳細・新規・編集のビュー、`resources :articles` のルーティング、`articles` テーブル用のマイグレーションが生成されます。

```bash
# 2. マイグレーションを実行して DB にテーブルを作成
bin/rails db:migrate
```

```bash
# 3. サーバを起動
bin/rails server
```

ブラウザで `http://localhost:3000/articles` にアクセスすると、記事の一覧・作成・編集・削除ができる CRUD 画面が動作します。

```
┌──────────────────────────────────────────────┐
│  http://localhost:3000/articles              │
├──────────────────────────────────────────────┤
│  Articles                                    │
│                                              │
│  [ New article ]                             │
│                                              │
│  ・初めての記事   [Show] [Edit] [Destroy]    │
│  ・二つ目の記事   [Show] [Edit] [Destroy]    │
└──────────────────────────────────────────────┘
```

**🎯 Django に scaffold はある？ — 代わりに「管理画面（admin）」:**

Django には Rails の scaffold のように CRUD 画面一式を生成するコマンドは**ありません**。その代わり、標準で**自動生成される管理画面（admin）**があり、モデルを登録するだけで CRUD 可能な UI がすぐに手に入ります。

```python
# blog/admin.py
from django.contrib import admin
from .models import Article

admin.site.register(Article)   # これだけで /admin に CRUD 画面が現れる
```

```bash
# マイグレーション適用と管理ユーザー作成 → /admin にアクセス
python manage.py migrate
python manage.py createsuperuser
python manage.py runserver
```

ただし admin は**運用・管理者向け**の画面である点に注意が必要です。エンドユーザー向けの画面は、Rails の scaffold が生成するような一般公開ページとして、別途 `views.py` とテンプレートで作り込みます。

生成されたルーティングは `bin/rails routes -c articles` で確認できます。

```bash
bin/rails routes -c articles
# GET    /articles          articles#index
# POST   /articles          articles#create
# GET    /articles/new      articles#new
# GET    /articles/:id/edit articles#edit
# GET    /articles/:id      articles#show
# PATCH  /articles/:id      articles#update
# DELETE /articles/:id      articles#destroy
```

生成物を取り消したい場合は `destroy` を使います（`migrate` 済みなら先に `db:rollback` を実行）。

```bash
# 生成した scaffold を削除
bin/rails destroy scaffold Article
```

これで Rails 開発の全体像と基本操作を確認できました。次章では、この scaffold が自動生成したルーティングの仕組みを詳しく見ていきます。

---

[Rails目次に戻る](README.md) | [次の章: ルーティング →](02_ルーティング.md)
