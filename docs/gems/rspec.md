# RSpec - BDDテストフレームワーク

## 概要

RSpecはRubyで最も広く使われているテストフレームワークです。BDD（振る舞い駆動開発）スタイルで、人間が読みやすいテストコードを書くことができます。

> **🐍 Python相当:** pytest（BDD風の記述・強力なfixture）

```bash
# インストール
gem install rspec

# プロジェクトで使用
# Gemfile
group :test do
  gem 'rspec', '~> 3.12'
end
```

---

## セットアップ

### 初期化

```bash
# RSpecの初期化
rspec --init

# 作成されるファイル
# .rspec              - コマンドラインオプション
# spec/spec_helper.rb - 基本設定
```

### .rspec ファイル

```text
--require spec_helper
--format documentation
--color
```

### spec_helper.rb

```ruby
RSpec.configure do |config|
  # 期待値の構文設定
  config.expect_with :rspec do |expectations|
    expectations.include_chain_clauses_in_custom_matcher_descriptions = true
  end

  # モックの設定
  config.mock_with :rspec do |mocks|
    mocks.verify_partial_doubles = true
  end

  # 共有コンテキストのメタデータ継承
  config.shared_context_metadata_behavior = :apply_to_host_groups

  # フィルタリング
  config.filter_run_when_matching :focus

  # 出力設定
  config.disable_monkey_patching!
  config.warnings = true

  # ランダム順序
  config.order = :random
  Kernel.srand config.seed
end
```

---

## 基本構文

### describe と it

```ruby
# spec/calculator_spec.rb
RSpec.describe Calculator do
  describe '#add' do
    it '2つの数を足し算する' do
      calculator = Calculator.new
      result = calculator.add(2, 3)
      expect(result).to eq(5)
    end

    it '負の数も扱える' do
      calculator = Calculator.new
      expect(calculator.add(-1, 1)).to eq(0)
    end
  end

  describe '#divide' do
    it '2つの数を割り算する' do
      calculator = Calculator.new
      expect(calculator.divide(10, 2)).to eq(5)
    end

    it 'ゼロで割ると例外を発生させる' do
      calculator = Calculator.new
      expect { calculator.divide(10, 0) }.to raise_error(ZeroDivisionError)
    end
  end
end
```

**🐍 Python:**

```python
# test_calculator.py
import pytest

class TestCalculator:
    def test_2つの数を足し算する(self):
        calculator = Calculator()
        assert calculator.add(2, 3) == 5

    def test_ゼロで割ると例外を発生させる(self):
        calculator = Calculator()
        with pytest.raises(ZeroDivisionError):
            calculator.divide(10, 0)
```

pytestでは `describe`/`it` の代わりに `class TestX:` とプレーンな `def test_...` 関数を使い、マッチャーの代わりに素の `assert` 文で検証します。

### context でグループ化

```ruby
RSpec.describe User do
  describe '#adult?' do
    context '18歳以上の場合' do
      it 'trueを返す' do
        user = User.new(age: 20)
        expect(user.adult?).to be true
      end
    end

    context '18歳未満の場合' do
      it 'falseを返す' do
        user = User.new(age: 15)
        expect(user.adult?).to be false
      end
    end
  end
end
```

**🐍 Python:**

```python
class TestUserAdult:
    class TestWhen18OrOver:
        def test_trueを返す(self):
            user = User(age=20)
            assert user.adult() is True

    class TestWhenUnder18:
        def test_falseを返す(self):
            user = User(age=15)
            assert user.adult() is False
```

pytestに `context` はありませんが、ネストしたクラスでグループ化できます（`pytest-describe` を使えば `describe`/`context` 風の記述も可能）。

### subject

```ruby
RSpec.describe User do
  # 暗黙のsubject
  subject { User.new(name: 'Alice', age: 25) }

  it { is_expected.to be_valid }

  # 名前付きsubject
  subject(:user) { User.new(name: 'Bob', age: 30) }

  it 'has a name' do
    expect(user.name).to eq('Bob')
  end
end
```

**🐍 Python:**

```python
import pytest

@pytest.fixture
def user():
    return User(name='Bob', age=30)

def test_has_a_name(user):
    assert user.name == 'Bob'
```

`subject`/`let` はどちらも fixture に相当します。fixture名を引数で受け取ると、遅延評価されたその値が渡されます。

### `let` と `let!`

```ruby
RSpec.describe Order do
  # let: 遅延評価（使用時に初めて実行）
  let(:user) { User.create(name: 'Alice') }
  let(:product) { Product.create(name: 'Book', price: 1000) }
  let(:order) { Order.new(user: user, product: product) }

  # let!: 即時評価（各テスト前に実行）
  let!(:admin) { User.create(name: 'Admin', role: :admin) }

  it 'calculates total' do
    expect(order.total).to eq(1000)
  end
end
```

**🐍 Python:**

```python
@pytest.fixture
def user():           # let 相当（遅延評価）
    return User.create(name='Alice')

@pytest.fixture(autouse=True)
def admin():          # let! 相当（各テスト前に必ず実行）
    return User.create(name='Admin', role='admin')
```

遅延評価の `let` は普通の fixture、即時評価の `let!` は `autouse=True` の fixture が近い挙動になります。

### before / after

```ruby
RSpec.describe Database do
  before(:all) do
    # 全テスト前に1回だけ実行
    DatabaseCleaner.clean_with(:truncation)
  end

  before(:each) do
    # 各テスト前に実行
    @connection = Database.connect
  end

  after(:each) do
    # 各テスト後に実行
    @connection.close
  end

  after(:all) do
    # 全テスト後に1回だけ実行
    Database.disconnect_all
  end

  # before(:suite) / after(:suite) は spec_helper で設定
end
```

**🐍 Python:**

```python
class TestDatabase:
    @classmethod
    def setup_class(cls):      # before(:all) 相当
        DatabaseCleaner.clean()

    def setup_method(self):    # before(:each) 相当
        self.connection = Database.connect()

    def teardown_method(self): # after(:each) 相当
        self.connection.close()

    @classmethod
    def teardown_class(cls):   # after(:all) 相当
        Database.disconnect_all()
```

`before`/`after` フックは `setup_method`/`teardown_method`（各テスト）や `setup_class`/`teardown_class`（クラス全体）に対応します。fixtureの `yield` で前後処理をまとめる書き方も一般的です。

---

## マッチャー

### 等価性

```ruby
expect(actual).to eq(expected)      # ==
expect(actual).to eql(expected)     # eql?
expect(actual).to equal(expected)   # equal?（同一オブジェクト）
expect(actual).to be(expected)      # equal?（同上）
```

**🐍 Python:**

```python
assert actual == expected     # eq 相当
assert actual is expected     # equal / be 相当（同一オブジェクト）
```

pytestは専用マッチャーを持たず、素の `assert` と比較演算子で表現します（`==` が eq、`is` が equal/be に対応）。

### 比較

```ruby
expect(10).to be > 5
expect(10).to be >= 10
expect(10).to be < 20
expect(10).to be <= 10
expect(10).to be_between(5, 15).inclusive
expect(10).to be_between(5, 15).exclusive
```

### 真偽値

```ruby
expect(true).to be true
expect(false).to be false
expect(nil).to be_nil
expect(nil).to be_falsey
expect(1).to be_truthy
```

**🐍 Python:**

```python
assert value is True      # be true
assert value is False     # be false
assert value is None      # be_nil
assert not value          # be_falsey
assert value              # be_truthy
```

### 型チェック

```ruby
expect('hello').to be_a(String)
expect('hello').to be_an_instance_of(String)
expect([]).to be_an(Array)
expect(1).to be_a_kind_of(Numeric)
```

### コレクション

```ruby
# 含む
expect([1, 2, 3]).to include(2)
expect([1, 2, 3]).to include(1, 3)
expect({ a: 1, b: 2 }).to include(:a)
expect({ a: 1, b: 2 }).to include(a: 1)

# サイズ
expect([1, 2, 3]).to have_attributes(size: 3)
expect([]).to be_empty
expect([1, 2, 3]).not_to be_empty

# 配列マッチング
expect([1, 2, 3]).to match_array([3, 1, 2])  # 順序無視
expect([1, 2, 3]).to contain_exactly(1, 2, 3)  # 順序無視
expect([1, 2, 3]).to eq([1, 2, 3])  # 順序一致

# 開始/終了
expect([1, 2, 3]).to start_with(1)
expect([1, 2, 3]).to end_with(3)
expect('hello').to start_with('he')
expect('hello').to end_with('lo')

# すべての要素
expect([2, 4, 6]).to all(be_even)
expect(['a', 'b', 'c']).to all(be_a(String))
```

**🐍 Python:**

```python
assert 2 in [1, 2, 3]                      # include
assert 'a' in {'a': 1, 'b': 2}             # include（キー）
assert [] == []                             # be_empty は len で: assert len(x) == 0
assert sorted([1, 2, 3]) == sorted([3, 1, 2])  # match_array 相当
assert all(n % 2 == 0 for n in [2, 4, 6])  # all 相当
```

`include` は `in` 演算子、順序無視の比較は `sorted()` 同士の比較、`all` は組み込みの `all()` で表現します。

### 文字列

```ruby
expect('hello world').to include('world')
expect('hello').to start_with('he')
expect('hello').to end_with('lo')
expect('hello').to match(/\Ahe/)
```

### 例外

```ruby
# 例外の発生
expect { raise 'error' }.to raise_error
expect { raise ArgumentError }.to raise_error(ArgumentError)
expect { raise 'oops' }.to raise_error('oops')
expect { raise ArgumentError, 'bad' }.to raise_error(ArgumentError, 'bad')
expect { raise ArgumentError, 'bad' }.to raise_error(ArgumentError, /bad/)

# 例外が発生しない
expect { safe_operation }.not_to raise_error
```

**🐍 Python:**

```python
with pytest.raises(ValueError):
    do_something()

# メッセージのマッチ
with pytest.raises(ValueError, match='bad'):
    do_something()
```

`raise_error` は `pytest.raises` コンテキストマネージャで表現し、`match=` に正規表現を渡してメッセージを検証できます。「例外が発生しない」ことの専用構文はなく、通常はそのままコードを実行して例外が出なければ合格とします。

### 変化

```ruby
# 値の変化
expect { user.grow_up }.to change(user, :age).by(1)
expect { user.grow_up }.to change(user, :age).from(20).to(21)
expect { user.grow_up }.to change { user.age }.by(1)

# 配列の変化
expect { list << 'item' }.to change { list.size }.by(1)
expect { list.clear }.to change { list.size }.to(0)
```

### 出力

```ruby
expect { puts 'hello' }.to output("hello\n").to_stdout
expect { warn 'warning' }.to output(/warning/).to_stderr
```

### 述語マッチャー

```ruby
# be_xxx は xxx? メソッドを呼ぶ
expect(user).to be_valid       # user.valid?
expect(user).to be_admin       # user.admin?
expect([]).to be_empty         # [].empty?
expect(1).to be_positive       # 1.positive?

# have_xxx は has_xxx? を呼ぶ
expect(user).to have_permission(:read)  # user.has_permission?(:read)
```

---

## モックとスタブ

### スタブ（戻り値の設定）

```ruby
RSpec.describe PaymentProcessor do
  it 'processes payment' do
    gateway = instance_double(PaymentGateway)

    # 戻り値を設定
    allow(gateway).to receive(:charge).and_return(true)

    # 引数に応じた戻り値
    allow(gateway).to receive(:charge).with(100).and_return(true)
    allow(gateway).to receive(:charge).with(0).and_return(false)

    # 複数回の呼び出し
    allow(gateway).to receive(:status).and_return('pending', 'complete')

    processor = PaymentProcessor.new(gateway)
    expect(processor.process(100)).to be true
  end
end
```

**🐍 Python:**

```python
from unittest.mock import create_autospec

def test_processes_payment():
    gateway = create_autospec(PaymentGateway)
    gateway.charge.return_value = True          # 戻り値を設定
    gateway.status.side_effect = ['pending', 'complete']  # 複数回の戻り値

    processor = PaymentProcessor(gateway)
    assert processor.process(100) is True
```

`allow(...).to receive(...).and_return(...)` は `mock.method.return_value = ...`、連続した戻り値は `side_effect` にリストを渡します（`pytest-mock` の `mocker` fixture も同様に使えます）。

### モック（呼び出しの検証）

```ruby
RSpec.describe OrderService do
  it 'sends confirmation email' do
    mailer = instance_double(OrderMailer)

    # 呼び出されることを期待
    expect(mailer).to receive(:send_confirmation).with(order)

    service = OrderService.new(mailer: mailer)
    service.complete(order)
  end

  it 'calls method specific number of times' do
    logger = instance_double(Logger)

    expect(logger).to receive(:info).exactly(3).times
    expect(logger).to receive(:error).once
    expect(logger).to receive(:debug).at_least(:twice)
    expect(logger).to receive(:warn).at_most(5).times

    # テスト対象の処理
  end
end
```

**🐍 Python:**

```python
def test_sends_confirmation_email(mocker):
    mailer = mocker.create_autospec(OrderMailer)
    service = OrderService(mailer=mailer)
    service.complete(order)
    mailer.send_confirmation.assert_called_once_with(order)

def test_call_counts(mocker):
    logger = mocker.create_autospec(Logger)
    # ... テスト対象の処理 ...
    assert logger.info.call_count == 3
    logger.error.assert_called_once()
```

呼び出しの検証は、RSpecのように事前に `expect(...).to receive` するのではなく、実行後に `assert_called_once_with` / `call_count` などで確認するのがPython流です（`unittest.mock` または `pytest-mock` の `mocker` fixture）。

### 部分的なダブル

```ruby
RSpec.describe User do
  it 'sends welcome email' do
    user = User.new(email: 'test@example.com')

    # 既存のオブジェクトの一部をモック
    allow(user).to receive(:send_email).and_return(true)

    expect(user.welcome).to be true
  end
end
```

### スパイ

```ruby
RSpec.describe Logger do
  it 'logs messages' do
    logger = spy('logger')

    service = Service.new(logger)
    service.perform

    # 後から検証
    expect(logger).to have_received(:info).with('Started')
    expect(logger).to have_received(:info).with('Completed')
  end
end
```

---

## 共有例

### shared_examples

```ruby
# spec/support/shared_examples/api_response.rb
RSpec.shared_examples 'a successful API response' do
  it 'returns status 200' do
    expect(response.status).to eq(200)
  end

  it 'returns JSON content type' do
    expect(response.content_type).to include('application/json')
  end
end

# 使用
RSpec.describe UsersController do
  describe 'GET /users' do
    before { get '/users' }

    it_behaves_like 'a successful API response'
  end
end

# パラメータ付き
RSpec.shared_examples 'paginated response' do |per_page|
  it "returns #{per_page} items per page" do
    expect(json_response['data'].size).to be <= per_page
  end
end

# 使用
it_behaves_like 'paginated response', 20
```

**🐍 Python:**

```python
import pytest

@pytest.mark.parametrize('per_page', [10, 20, 50])
def test_paginated_response(per_page):
    assert len(json_response['data']) <= per_page
```

同じ検証を複数の値で繰り返すパラメータ化は、`@pytest.mark.parametrize` デコレータで表現します。

### shared_context

```ruby
RSpec.shared_context 'authenticated user' do
  let(:current_user) { create(:user) }

  before do
    sign_in(current_user)
  end
end

# 使用
RSpec.describe 'Protected resources' do
  include_context 'authenticated user'

  it 'allows access' do
    get '/dashboard'
    expect(response).to be_successful
  end
end
```

---

## 実行とフィルタリング

### 実行コマンド

```bash
# 全テスト実行
rspec

# 特定のファイル
rspec spec/models/user_spec.rb

# 特定の行
rspec spec/models/user_spec.rb:10

# ディレクトリ
rspec spec/models/

# パターン
rspec spec/**/*_spec.rb

# タグでフィルタ
rspec --tag slow
rspec --tag ~slow  # slow以外

# 失敗したテストのみ再実行
rspec --only-failures

# ランダム順序を固定
rspec --seed 1234
```

**🐍 Python:**

```bash
pytest                              # 全テスト実行
pytest tests/test_user.py           # 特定のファイル
pytest tests/test_user.py::test_x   # 特定のテスト
pytest -k "slow"                    # 名前でフィルタ
pytest --lf                         # 失敗したテストのみ再実行（--last-failed）
```

`rspec` に対応するコマンドは `pytest` です。行番号指定 (`:10`) の代わりに `::テスト名` で個別テストを指定します。

### タグ

```ruby
RSpec.describe 'slow test', :slow do
  # ...
end

it 'is a slow test', :slow do
  # ...
end

it 'focuses on this', :focus do
  # ...
end

it 'is pending', :pending do
  # ...
end

it 'skips', skip: 'reason' do
  # ...
end
```

### 出力フォーマット

```bash
rspec --format documentation  # 詳細
rspec --format progress       # ドット表示
rspec --format json           # JSON出力
rspec --format html --out report.html  # HTMLレポート
```

---

## ベストプラクティス

### 1テスト1アサーション

```ruby
# 悪い例
it 'creates a user' do
  user = User.create(name: 'Alice', email: 'alice@example.com')
  expect(user).to be_persisted
  expect(user.name).to eq('Alice')
  expect(user.email).to eq('alice@example.com')
end

# 良い例
describe 'creating a user' do
  subject(:user) { User.create(name: 'Alice', email: 'alice@example.com') }

  it { is_expected.to be_persisted }
  it { is_expected.to have_attributes(name: 'Alice') }
  it { is_expected.to have_attributes(email: 'alice@example.com') }
end
```

### 説明的な記述

```ruby
# 悪い例
it 'works' do
  expect(user.valid?).to be true
end

# 良い例
it 'is valid with valid attributes' do
  expect(user).to be_valid
end
```

### ファクトリーの使用

```ruby
# FactoryBot との組み合わせ
RSpec.configure do |config|
  config.include FactoryBot::Syntax::Methods
end

# テスト内
let(:user) { create(:user) }
let(:admin) { create(:user, :admin) }
let(:order) { create(:order, user: user) }
```

---

## 関連Gem

| Gem | 説明 |
|-----|------|
| factory_bot | テストデータ生成 |
| faker | ダミーデータ |
| shoulda-matchers | Rails用マッチャー |
| database_cleaner | テストDB清掃 |
| simplecov | コードカバレッジ |
| webmock | HTTPモック |
| vcr | HTTPレスポンス記録 |

---

## 参考リンク

- [RSpec公式ドキュメント](https://rspec.info/)
- [Better Specs](https://www.betterspecs.org/) - RSpecベストプラクティス
- [RSpec Style Guide](https://rspec.rubystyle.guide/)

← [Gem一覧に戻る](README.md)
