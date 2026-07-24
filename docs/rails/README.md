# Ruby on Rails 入門ガイド

このディレクトリでは、Rubyのフルスタックフレームワーク **Ruby on Rails** を体系的に学べる資料をまとめています。Ruby本体の基礎（[../../RUBY_GUIDE.md](../../RUBY_GUIDE.md)）を一通り理解していることを前提に、Webアプリケーション開発に必要な知識を章立てで解説します。

対象バージョン: **Rails 7.2 / 8 系**

---

## 目次

| 章 | タイトル | 内容 | ドキュメント |
|----|---------|------|-------------|
| 1 | 概要と環境構築 | Railsの思想（DRY / CoC）、MVC、`rails new`、ディレクトリ構成、主要コマンド | [01_Rails概要.md](01_Rails概要.md) |
| 2 | ルーティング | `config/routes.rb`、`resources`、RESTful、名前付きルート、namespace | [02_ルーティング.md](02_ルーティング.md) |
| 3 | コントローラ | アクション、Strong Parameters、`render`/`redirect_to`、フィルタ、セッション | [03_コントローラ.md](03_コントローラ.md) |
| 4 | モデルとActiveRecord | マイグレーション、CRUD、クエリ、バリデーション、コールバック | [04_モデルとActiveRecord.md](04_モデルとActiveRecord.md) |
| 5 | アソシエーション | `has_many`/`belongs_to`、多対多、ポリモーフィック、N+1問題 | [05_アソシエーション.md](05_アソシエーション.md) |
| 6 | ビューとフォーム | ERB、レイアウト、パーシャル、ヘルパー、`form_with` | [06_ビューとフォーム.md](06_ビューとフォーム.md) |
| 7 | 認証と認可 | `has_secure_password`、Devise、認可、Railsのセキュリティ機構 | [07_認証と認可.md](07_認証と認可.md) |
| 8 | テスト | Minitest（標準）、RSpec、システムテスト、FactoryBot | [08_テスト.md](08_テスト.md) |
| 9 | 発展機能 | Active Job / Action Mailer / Active Storage / APIモード / I18n / デプロイ | [09_発展機能.md](09_発展機能.md) |

---

## 学習の進め方

```
┌──────────────────────────────────────────────────────────┐
│  1. 概要・環境構築   … Railsの全体像を掴み、雛形を動かす      │
│         ↓                                                 │
│  2〜3. ルーティング / コントローラ … リクエストの流れを理解    │
│         ↓                                                 │
│  4〜5. モデル / アソシエーション   … データベースとの連携      │
│         ↓                                                 │
│  6. ビューとフォーム … 画面の表示と入力                      │
│         ↓                                                 │
│  7〜8. 認証・認可 / テスト … 実用的なアプリに必要な要素        │
│         ↓                                                 │
│  9. 発展機能 … メール・非同期処理・API・デプロイ              │
└──────────────────────────────────────────────────────────┘
```

まずは第1章の `rails new` と scaffold で「動くもの」を作り、第2章以降で仕組みを掘り下げていくのがおすすめです。

---

## 関連する既存資料

Railsの内部やエコシステムを理解する上で、本リポジトリの以下の資料も役立ちます。

| テーマ | 資料 |
|--------|------|
| Ruby言語の基礎全般 | [../../RUBY_GUIDE.md](../../RUBY_GUIDE.md) |
| テストの基礎（Minitest / RSpec） | [../13_テスト.md](../13_テスト.md) |
| ActiveRecord（Rails非依存の単体利用） | [../gems/activerecord.md](../gems/activerecord.md) |
| 認証gem Devise | [../gems/devise.md](../gems/devise.md) |
| Microsoft Entra ID連携（OmniAuth） | [../gems/omniauth-azure-ad.md](../gems/omniauth-azure-ad.md) |
| RSpecの詳細 | [../gems/rspec.md](../gems/rspec.md) |
| 非同期ジョブ Sidekiq | [../gems/sidekiq.md](../gems/sidekiq.md) |
| オブジェクトストレージ Cloudflare R2 | [../gems/cloudflare-r2.md](../gems/cloudflare-r2.md) |
| 軽量Webフレームワーク Sinatra（比較用） | [../gems/sinatra.md](../gems/sinatra.md) |

---

## 公式リファレンス

- [Ruby on Rails Guides（公式ガイド・英語）](https://guides.rubyonrails.org/)
- [Railsガイド（日本語）](https://railsguides.jp/)
- [Rails API ドキュメント](https://api.rubyonrails.org/)
- [Ruby on Rails 公式サイト](https://rubyonrails.org/)

---

← [Ruby基礎ガイドに戻る](../../RUBY_GUIDE.md)
