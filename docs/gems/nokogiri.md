# Nokogiri - HTML/XML パーサー

## 概要

NokogiriはRubyで最も使われているHTML/XMLパーサーです。Webスクレイピング、XMLデータの処理、HTMLの生成など、幅広い用途で使用されます。

> **🐍 Python相当:** `BeautifulSoup`（bs4・使いやすさ重視）／ `lxml`（C実装で高速、XPathフル対応）。Nokogiriはこのlxmlと同じくlibxml2ベースです。

```bash
# インストール
gem install nokogiri

# Gemfile
gem 'nokogiri'
```

---

## 基本的な使い方

### HTML のパース

```ruby
require 'nokogiri'

# 文字列から
html = '<html><body><h1>Hello</h1><p>World</p></body></html>'
doc = Nokogiri::HTML(html)

# ファイルから
doc = Nokogiri::HTML(File.open('page.html'))

# URLから（open-uriと組み合わせ）
require 'open-uri'
doc = Nokogiri::HTML(URI.open('https://example.com'))

# フラグメント（部分的なHTML）
fragment = Nokogiri::HTML.fragment('<p>Hello</p><p>World</p>')
```

**🐍 Python:**

```python
# BeautifulSoup (bs4): 第2引数でパーサーを選ぶ
from bs4 import BeautifulSoup
html = "<html><body><h1>Hello</h1><p>World</p></body></html>"
soup = BeautifulSoup(html, "html.parser")   # 標準ライブラリのパーサー
soup = BeautifulSoup(html, "lxml")          # lxml があればより高速・寛容

# ファイルから
with open("page.html", encoding="utf-8") as f:
    soup = BeautifulSoup(f, "html.parser")

# lxml を直接使う場合 (Nokogiri に最も近い書き味)
import lxml.html
doc = lxml.html.fromstring(html)
```

`Nokogiri::HTML(html)`は`BeautifulSoup(html, "html.parser")`や`lxml.html.fromstring(html)`に相当します。bs4はパーサーを差し替え可能で、`"lxml"`を選ぶとNokogiriと同じlibxml2ベースになります。

### XML のパース

```ruby
# 文字列から
xml = '<root><item id="1">First</item><item id="2">Second</item></root>'
doc = Nokogiri::XML(xml)

# ファイルから
doc = Nokogiri::XML(File.open('data.xml'))

# 厳密なパース（エラー時に例外）
doc = Nokogiri::XML(xml) { |config| config.strict }
```

**🐍 Python:**

```python
from lxml import etree

xml = '<root><item id="1">First</item><item id="2">Second</item></root>'

# 文字列から (XMLは厳密パースがデフォルト、不正なら例外)
root = etree.fromstring(xml.encode("utf-8"))

# ファイルから
tree = etree.parse("data.xml")
root = tree.getroot()
```

Nokogiriの`Nokogiri::XML`は`lxml.etree`が最も近く、こちらも既定で厳密パース（不正なXMLで`XMLSyntaxError`）です。bs4でも`BeautifulSoup(xml, "xml")`でXMLを扱えますが、XPathを使うなら`lxml.etree`が適しています。

---

## 要素の検索

### CSS セレクタ

```ruby
html = <<~HTML
  <html>
    <body>
      <div id="content">
        <h1 class="title">Title</h1>
        <ul class="list">
          <li>Item 1</li>
          <li>Item 2</li>
          <li class="special">Item 3</li>
        </ul>
        <a href="https://example.com">Link</a>
      </div>
    </body>
  </html>
HTML

doc = Nokogiri::HTML(html)

# 基本的なセレクタ
doc.css('h1')                    # タグ名
doc.css('#content')              # ID
doc.css('.title')                # クラス
doc.css('div.container')         # タグ + クラス
doc.css('ul li')                 # 子孫
doc.css('ul > li')               # 直接の子
doc.css('li.special')            # タグ + クラス

# 属性セレクタ
doc.css('a[href]')               # 属性が存在
doc.css('a[href="https://example.com"]')  # 属性値が一致
doc.css('a[href^="https"]')      # 前方一致
doc.css('a[href$=".com"]')       # 後方一致
doc.css('a[href*="example"]')    # 部分一致

# 擬似クラス
doc.css('li:first-child')        # 最初の子
doc.css('li:last-child')         # 最後の子
doc.css('li:nth-child(2)')       # n番目の子
doc.css('li:not(.special)')      # 否定
```

**🐍 Python:**

```python
# BeautifulSoup: select() が CSS セレクタで NodeSet 相当を返す
soup.select("h1")                     # タグ名
soup.select("#content")               # ID
soup.select(".title")                 # クラス
soup.select("ul li")                  # 子孫
soup.select("ul > li")                # 直接の子
soup.select('a[href^="https"]')       # 属性・前方一致
soup.select("li:nth-child(2)")        # 擬似クラス
soup.select_one("h1")                 # 最初の1件 (Nokogiri の at_css)

# lxml でも CSS セレクタが使える (内部で XPath に変換)
doc.cssselect("ul > li")
```

Nokogiriの`doc.css("a")`は`soup.select("a")`に相当し、単一取得の`at_css`は`select_one`にあたります。bs4は`find`/`find_all`という独自APIも持ちますが、CSSセレクタなら`select`が最も対応が明快です。

### XPath

```ruby
# 基本的なXPath
doc.xpath('//h1')                # すべてのh1
doc.xpath('//div[@id="content"]')  # ID指定
doc.xpath('//li[@class="special"]')  # クラス指定
doc.xpath('//ul/li')             # 直接の子
doc.xpath('//ul//li')            # 子孫

# 属性
doc.xpath('//a/@href')           # href属性の値

# テキスト
doc.xpath('//li/text()')         # テキストノード

# 条件
doc.xpath('//li[position()=1]')  # 位置指定
doc.xpath('//li[last()]')        # 最後
doc.xpath('//li[contains(@class, "special")]')  # 部分一致

# 複数条件
doc.xpath('//div[@id="content" and @class="container"]')

# 関数
doc.xpath('//li[contains(text(), "Item")]')
doc.xpath('//li[starts-with(text(), "Item")]')
doc.xpath('count(//li)')         # カウント
```

**🐍 Python:**

```python
# lxml が XPath 1.0 をフルサポート (bs4 単体は XPath 非対応)
from lxml import html
tree = html.fromstring(page_html)

tree.xpath("//h1")                        # すべての h1 (要素のリスト)
tree.xpath('//div[@id="content"]')        # ID 指定
tree.xpath("//a/@href")                   # href 属性の値 (文字列のリスト)
tree.xpath("//li/text()")                 # テキストノード
tree.xpath('//li[contains(@class, "special")]')
tree.xpath("count(//li)")                 # カウント (float を返す)
```

Nokogiriの`doc.xpath("//a")`は`lxml`の`tree.xpath("//a")`に相当します。XPathを使いたい場合はbs4単体では非対応のため`lxml`を選びます（`count()`などの関数は数値や文字列を、要素式は要素のリストを返す点はNokogiriと同じ挙動です）。

### 検索メソッド

```ruby
# css / xpath - NodeSetを返す
nodes = doc.css('li')

# at_css / at_xpath - 最初の1つを返す
node = doc.at_css('h1')
node = doc.at_xpath('//h1')

# search - css と xpath の両方を受け付ける
doc.search('h1')
doc.search('//h1')

# at - search の単一版
doc.at('h1')
```

---

## 要素の操作

### 要素の情報取得

```ruby
node = doc.at_css('a')

# テキスト
node.text                        # テキスト内容
node.content                     # 同上
node.inner_text                  # 同上

# HTML
node.inner_html                  # 内部HTML
node.to_html                     # 外部HTML含む
node.to_s                        # 同上

# 属性
node['href']                     # 属性値
node.get_attribute('href')       # 同上
node.attributes                  # すべての属性
node.attribute('href').value     # Attrオブジェクト経由

# 要素情報
node.name                        # タグ名
node.path                        # XPathパス
node.line                        # 行番号（パース時に保持）
```

**🐍 Python:**

```python
# BeautifulSoup
el = soup.select_one("a")
el.get_text()          # テキスト内容 (Nokogiri の node.text)
el.get_text(strip=True)  # 前後空白を除去
str(el)                # 外側 HTML (node.to_html)
el.decode_contents()   # 内部 HTML (node.inner_html)
el["href"]             # 属性値 (存在しないと KeyError)
el.get("href")         # 属性値 (無ければ None)
el.attrs               # すべての属性 (dict)
el.name                # タグ名

# lxml
el = tree.xpath("//a")[0]
el.text_content()      # テキスト内容
el.get("href")         # 属性値
el.tag                 # タグ名
```

Nokogiriの`node.text`はbs4の`el.get_text()`、`node["href"]`は`el["href"]`（安全に取るなら`el.get("href")`）に相当します。lxmlでは`el.text_content()`と`el.get("href")`を使います。

### NodeSet の操作

```ruby
nodes = doc.css('li')

# イテレーション
nodes.each { |node| puts node.text }
nodes.map(&:text)
nodes.select { |n| n.text.include?('Item') }

# インデックスアクセス
nodes[0]
nodes.first
nodes.last

# 検索の連鎖
nodes.css('.special')
nodes.xpath('.//span')  # 相対パス

# 集合操作
nodes1 = doc.css('.class1')
nodes2 = doc.css('.class2')
nodes1 + nodes2                  # 和
nodes1 - nodes2                  # 差
nodes1 & nodes2                  # 積
```

### ナビゲーション

```ruby
node = doc.at_css('li.special')

# 親
node.parent
node.ancestors

# 子
node.children
node.first_element_child
node.last_element_child

# 兄弟
node.next_sibling
node.previous_sibling
node.next_element
node.previous_element

# ナビゲーションの連鎖
node.parent.css('li')
node.next_element.text
```

---

## 要素の作成と変更

### 新しい要素の作成

```ruby
doc = Nokogiri::HTML('<html><body></body></html>')

# 新しい要素
new_element = Nokogiri::XML::Node.new('div', doc)
new_element['class'] = 'container'
new_element.content = 'Hello'

# ドキュメントに追加
doc.at_css('body').add_child(new_element)

# フラグメントから作成
fragment = doc.create_element('p')
fragment.content = 'Paragraph'
doc.at_css('body') << fragment
```

### 要素の変更

```ruby
node = doc.at_css('h1')

# テキストの変更
node.content = 'New Title'

# 属性の変更
node['class'] = 'main-title'
node['id'] = 'header'
node.remove_attribute('class')

# 内部HTMLの変更
node.inner_html = '<span>Inner</span> Content'

# 要素名の変更
node.name = 'h2'
```

### 要素の追加と削除

```ruby
parent = doc.at_css('ul')
new_li = doc.create_element('li')
new_li.content = 'New Item'

# 追加
parent.add_child(new_li)         # 末尾に追加
parent << new_li                 # 同上
parent.prepend_child(new_li)     # 先頭に追加

# 特定の位置に挿入
reference = doc.at_css('li:nth-child(2)')
reference.add_next_sibling(new_li)
reference.add_previous_sibling(new_li)

# 置換
old_node = doc.at_css('li.old')
old_node.replace(new_li)

# 削除
node = doc.at_css('li.special')
node.remove
node.unlink                      # 同上

# ラップ
node = doc.at_css('p')
node.wrap('<div class="wrapper"></div>')
```

---

## 実践的な例

### Web スクレイピング

```ruby
require 'nokogiri'
require 'open-uri'

# ページを取得してパース
url = 'https://example.com/articles'
doc = Nokogiri::HTML(URI.open(url))

# 記事一覧を抽出
articles = doc.css('article.post').map do |article|
  {
    title: article.at_css('h2.title')&.text&.strip,
    url: article.at_css('a.read-more')&.[]('href'),
    date: article.at_css('time')&.[]('datetime'),
    summary: article.at_css('p.summary')&.text&.strip
  }
end

puts articles

# リンクをすべて抽出
links = doc.css('a[href]').map { |a| a['href'] }
external_links = links.select { |href| href.start_with?('http') }
```

### HTML の整形

```ruby
# 整形されたHTMLを出力
html = '<html><body><div><p>Hello</p></div></body></html>'
doc = Nokogiri::HTML(html)
puts doc.to_html

# インデント付き
puts doc.to_xhtml(indent: 2)

# XMLの整形
xml = '<root><item>1</item><item>2</item></root>'
doc = Nokogiri::XML(xml)
puts doc.to_xml(indent: 2)
```

### XML データの処理

```ruby
xml = <<~XML
  <?xml version="1.0"?>
  <catalog>
    <book id="1">
      <title>Ruby Programming</title>
      <author>Alice</author>
      <price currency="USD">29.99</price>
    </book>
    <book id="2">
      <title>Web Development</title>
      <author>Bob</author>
      <price currency="USD">39.99</price>
    </book>
  </catalog>
XML

doc = Nokogiri::XML(xml)

# データ抽出
books = doc.xpath('//book').map do |book|
  {
    id: book['id'],
    title: book.at_xpath('title').text,
    author: book.at_xpath('author').text,
    price: book.at_xpath('price').text.to_f,
    currency: book.at_xpath('price')['currency']
  }
end

# 条件付き検索
expensive = doc.xpath('//book[price > 30]')

# 新しい要素を追加
catalog = doc.at_xpath('//catalog')
new_book = Nokogiri::XML::Node.new('book', doc)
new_book['id'] = '3'
new_book.add_child('<title>New Book</title>')
new_book.add_child('<author>Charlie</author>')
new_book.add_child('<price currency="USD">19.99</price>')
catalog.add_child(new_book)

puts doc.to_xml(indent: 2)
```

### HTML の生成

```ruby
builder = Nokogiri::HTML::Builder.new do |doc|
  doc.html {
    doc.head {
      doc.title 'My Page'
      doc.link(rel: 'stylesheet', href: 'style.css')
    }
    doc.body {
      doc.div(class: 'container') {
        doc.h1 'Welcome'
        doc.p 'This is a paragraph.'
        doc.ul {
          ['Item 1', 'Item 2', 'Item 3'].each do |item|
            doc.li item
          end
        }
      }
    }
  }
end

puts builder.to_html
```

### 名前空間付き XML

```ruby
xml = <<~XML
  <?xml version="1.0"?>
  <root xmlns:ns="http://example.com/ns">
    <ns:item>Value</ns:item>
  </root>
XML

doc = Nokogiri::XML(xml)

# 名前空間を指定して検索
doc.xpath('//ns:item', 'ns' => 'http://example.com/ns')

# または
doc.remove_namespaces!  # 名前空間を削除
doc.xpath('//item')
```

---

## エラー処理

### パースエラー

```ruby
# エラーの確認
doc = Nokogiri::HTML(broken_html)
doc.errors.each do |error|
  puts "Line #{error.line}: #{error.message}"
end

# 厳密モード（XMLのみ）
begin
  doc = Nokogiri::XML(xml) { |config| config.strict }
rescue Nokogiri::XML::SyntaxError => e
  puts "Parse error: #{e.message}"
end

# 警告を無視
doc = Nokogiri::XML(xml) { |config| config.noerror.nowarning }
```

### 要素が見つからない場合

```ruby
# at_css / at_xpath は nil を返す
node = doc.at_css('nonexistent')
if node
  puts node.text
else
  puts 'Not found'
end

# 安全なアクセス
title = doc.at_css('h1')&.text || 'Default'

# css / xpath は空のNodeSetを返す
nodes = doc.css('nonexistent')
nodes.empty?  # => true
```

---

## パフォーマンス

### メモリ効率

```ruby
# 大きなファイルの処理
# SAXパーサーを使用
class MyHandler < Nokogiri::XML::SAX::Document
  def start_element(name, attrs = [])
    puts "Start: #{name}"
  end

  def end_element(name)
    puts "End: #{name}"
  end

  def characters(string)
    puts "Text: #{string}"
  end
end

parser = Nokogiri::XML::SAX::Parser.new(MyHandler.new)
parser.parse(File.open('large.xml'))
```

### キャッシュ

```ruby
# CSSセレクタは内部でXPathに変換される
# 同じセレクタを繰り返し使う場合はXPathを直接使う
xpath = Nokogiri::CSS.xpath_for('div.content > p.intro').first
doc.xpath(xpath)
```

---

## 参考リンク

- [Nokogiri公式サイト](https://nokogiri.org/)
- [Nokogiri チュートリアル](https://nokogiri.org/tutorials/toc.html)
- [CSSセレクタリファレンス](https://nokogiri.org/tutorials/searching_a_xml_html_document.html)
- [XPathリファレンス](https://www.w3schools.com/xml/xpath_intro.asp)

← [Gem一覧に戻る](README.md)
