# Cloudflare R2 - S3互換オブジェクトストレージ

## 概要

Cloudflare R2はS3互換のオブジェクトストレージサービスです。RubyからはAWS SDK（`aws-sdk-s3`）を使用してアクセスできます。エグレス料金が無料という特徴があり、コスト効率の良いストレージソリューションです。

```bash
# インストール
gem install aws-sdk-s3

# Gemfile
gem 'aws-sdk-s3'
```

**注意:** Cloudflare D1（SQLiteベースのデータベース）は、現時点ではCloudflare Workers専用であり、Ruby用の公式Gemは存在しません。

---

## Cloudflare での準備

### 1. R2 バケットの作成

1. [Cloudflare Dashboard](https://dash.cloudflare.com/) にログイン
2. 左メニューから「R2 Object Storage」を選択
3. 「Create bucket」をクリック
4. バケット名を入力して作成

### 2. API トークンの取得

1. R2ページの右サイドバーで「Manage R2 API Tokens」をクリック
2. 「Create API Token」を選択
3. トークン名を入力し、権限を設定：
   - **Object Read & Write**: 読み書き両方
   - **Object Read only**: 読み取りのみ
4. 「Create API Token」をクリック
5. 表示される情報を保存：
   - **Access Key ID**
   - **Secret Access Key**（一度しか表示されない）

### 3. アカウントIDの確認

1. Cloudflare Dashboardの右サイドバー
2. または URL: `https://dash.cloudflare.com/<ACCOUNT_ID>/r2`

---

## 基本設定

### クライアントの初期化

```ruby
require 'aws-sdk-s3'

# 基本設定
r2_client = Aws::S3::Client.new(
  access_key_id: ENV['R2_ACCESS_KEY_ID'],
  secret_access_key: ENV['R2_SECRET_ACCESS_KEY'],
  endpoint: "https://#{ENV['R2_ACCOUNT_ID']}.r2.cloudflarestorage.com",
  region: 'auto'  # SDK必須だがR2では使用されない
)
```

### 環境変数

```bash
# .env
R2_ACCOUNT_ID=your_account_id
R2_ACCESS_KEY_ID=your_access_key_id
R2_SECRET_ACCESS_KEY=your_secret_access_key
R2_BUCKET_NAME=your_bucket_name
```

### Resource インターフェース

```ruby
# Resourceを使用（より高レベルなAPI）
r2_resource = Aws::S3::Resource.new(
  access_key_id: ENV['R2_ACCESS_KEY_ID'],
  secret_access_key: ENV['R2_SECRET_ACCESS_KEY'],
  endpoint: "https://#{ENV['R2_ACCOUNT_ID']}.r2.cloudflarestorage.com",
  region: 'auto'
)

bucket = r2_resource.bucket(ENV['R2_BUCKET_NAME'])
```

---

## バケット操作

### バケット一覧の取得

```ruby
# すべてのバケットを一覧
response = r2_client.list_buckets
response.buckets.each do |bucket|
  puts "#{bucket.name} - Created: #{bucket.creation_date}"
end
```

### バケットの作成

```ruby
# 新しいバケットを作成
r2_client.create_bucket(bucket: 'my-new-bucket')
puts "Bucket created successfully"
```

### バケットの削除

```ruby
# バケットを削除（空である必要がある）
r2_client.delete_bucket(bucket: 'my-bucket')
puts "Bucket deleted successfully"
```

### バケットの存在確認

```ruby
def bucket_exists?(client, bucket_name)
  client.head_bucket(bucket: bucket_name)
  true
rescue Aws::S3::Errors::NotFound
  false
end

if bucket_exists?(r2_client, 'my-bucket')
  puts "Bucket exists"
else
  puts "Bucket not found"
end
```

---

## オブジェクト操作

### オブジェクトのアップロード

```ruby
# 文字列をアップロード
r2_client.put_object(
  bucket: ENV['R2_BUCKET_NAME'],
  key: 'path/to/file.txt',
  body: 'Hello, R2!'
)

# ファイルをアップロード
File.open('local_file.txt', 'rb') do |file|
  r2_client.put_object(
    bucket: ENV['R2_BUCKET_NAME'],
    key: 'uploads/file.txt',
    body: file
  )
end

# メタデータ付きでアップロード
r2_client.put_object(
  bucket: ENV['R2_BUCKET_NAME'],
  key: 'documents/report.pdf',
  body: File.read('report.pdf'),
  content_type: 'application/pdf',
  metadata: {
    'uploaded-by' => 'user123',
    'category' => 'reports'
  }
)
```

### オブジェクトのダウンロード

```ruby
# オブジェクトを取得
response = r2_client.get_object(
  bucket: ENV['R2_BUCKET_NAME'],
  key: 'path/to/file.txt'
)

# 内容を読む
content = response.body.read
puts content

# ファイルに保存
r2_client.get_object(
  bucket: ENV['R2_BUCKET_NAME'],
  key: 'path/to/file.txt',
  response_target: 'downloaded_file.txt'
)
```

### オブジェクトの一覧

```ruby
# 最初の20件を取得
response = r2_client.list_objects(
  bucket: ENV['R2_BUCKET_NAME'],
  max_keys: 20
)

response.contents.each do |object|
  puts "#{object.key} (#{object.size} bytes)"
end

# プレフィックスでフィルタ
response = r2_client.list_objects(
  bucket: ENV['R2_BUCKET_NAME'],
  prefix: 'uploads/'
)

# ページネーション
r2_client.list_objects(bucket: ENV['R2_BUCKET_NAME']).each do |response|
  response.contents.each do |object|
    puts object.key
  end
end

# list_objects_v2を使用（推奨）
response = r2_client.list_objects_v2(
  bucket: ENV['R2_BUCKET_NAME'],
  prefix: 'images/',
  max_keys: 100
)

response.contents.each do |object|
  puts "#{object.key} - #{object.last_modified}"
end
```

### オブジェクトの削除

```ruby
# 単一オブジェクトを削除
r2_client.delete_object(
  bucket: ENV['R2_BUCKET_NAME'],
  key: 'path/to/file.txt'
)

# 複数オブジェクトを一括削除
r2_client.delete_objects(
  bucket: ENV['R2_BUCKET_NAME'],
  delete: {
    objects: [
      { key: 'file1.txt' },
      { key: 'file2.txt' },
      { key: 'folder/file3.txt' }
    ]
  }
)
```

### オブジェクトのコピー

```ruby
# 同じバケット内でコピー
r2_client.copy_object(
  bucket: ENV['R2_BUCKET_NAME'],
  copy_source: "#{ENV['R2_BUCKET_NAME']}/original.txt",
  key: 'copy.txt'
)

# 別のバケットにコピー
r2_client.copy_object(
  bucket: 'destination-bucket',
  copy_source: "#{ENV['R2_BUCKET_NAME']}/original.txt",
  key: 'copied-file.txt'
)
```

### オブジェクトの存在確認

```ruby
def object_exists?(client, bucket, key)
  client.head_object(bucket: bucket, key: key)
  true
rescue Aws::S3::Errors::NotFound
  false
end

if object_exists?(r2_client, ENV['R2_BUCKET_NAME'], 'my-file.txt')
  puts "Object exists"
end
```

### メタデータの取得

```ruby
response = r2_client.head_object(
  bucket: ENV['R2_BUCKET_NAME'],
  key: 'path/to/file.txt'
)

puts "Size: #{response.content_length}"
puts "Content-Type: #{response.content_type}"
puts "Last Modified: #{response.last_modified}"
puts "Metadata: #{response.metadata}"
```

---

## 署名付きURL

### ダウンロード用URL

```ruby
# Signerを使用
signer = Aws::S3::Presigner.new(client: r2_client)

# 1時間有効なダウンロードURL
download_url = signer.presigned_url(
  :get_object,
  bucket: ENV['R2_BUCKET_NAME'],
  key: 'path/to/file.txt',
  expires_in: 3600  # 秒
)

puts download_url
```

### アップロード用URL

```ruby
# 直接アップロード用のURL
upload_url = signer.presigned_url(
  :put_object,
  bucket: ENV['R2_BUCKET_NAME'],
  key: 'uploads/new-file.txt',
  expires_in: 3600,
  content_type: 'text/plain'
)

puts upload_url

# クライアントからのアップロード例（curl）
# curl -X PUT -H "Content-Type: text/plain" --data-binary @file.txt "#{upload_url}"
```

---

## マルチパートアップロード

大きなファイルを効率的にアップロードするためのマルチパートアップロード。

```ruby
# 自動マルチパートアップロード（推奨）
object = r2_resource.bucket(ENV['R2_BUCKET_NAME']).object('large-file.zip')

# 5MB以上で自動的にマルチパートに切り替わる
object.upload_file('large-file.zip')

# オプション指定
object.upload_file(
  'large-file.zip',
  multipart_threshold: 15 * 1024 * 1024,  # 15MB以上でマルチパート
  part_size: 10 * 1024 * 1024              # パートサイズ10MB
)
```

### 手動マルチパートアップロード

```ruby
# マルチパートアップロード開始
multipart = r2_client.create_multipart_upload(
  bucket: ENV['R2_BUCKET_NAME'],
  key: 'large-file.zip'
)

parts = []
part_number = 1

File.open('large-file.zip', 'rb') do |file|
  while (chunk = file.read(5 * 1024 * 1024))  # 5MBずつ
    response = r2_client.upload_part(
      bucket: ENV['R2_BUCKET_NAME'],
      key: 'large-file.zip',
      upload_id: multipart.upload_id,
      part_number: part_number,
      body: chunk
    )

    parts << { etag: response.etag, part_number: part_number }
    part_number += 1
  end
end

# アップロード完了
r2_client.complete_multipart_upload(
  bucket: ENV['R2_BUCKET_NAME'],
  key: 'large-file.zip',
  upload_id: multipart.upload_id,
  multipart_upload: { parts: parts }
)
```

---

## Rails との連携

### Active Storage 設定

```ruby
# Gemfile
gem 'aws-sdk-s3'
```

```yaml
# config/storage.yml
cloudflare_r2:
  service: S3
  access_key_id: <%= ENV['R2_ACCESS_KEY_ID'] %>
  secret_access_key: <%= ENV['R2_SECRET_ACCESS_KEY'] %>
  region: auto
  bucket: <%= ENV['R2_BUCKET_NAME'] %>
  endpoint: https://<%= ENV['R2_ACCOUNT_ID'] %>.r2.cloudflarestorage.com
  force_path_style: true
```

```ruby
# config/environments/production.rb
config.active_storage.service = :cloudflare_r2
```

### CarrierWave 設定

```ruby
# config/initializers/carrierwave.rb
CarrierWave.configure do |config|
  config.fog_credentials = {
    provider: 'AWS',
    aws_access_key_id: ENV['R2_ACCESS_KEY_ID'],
    aws_secret_access_key: ENV['R2_SECRET_ACCESS_KEY'],
    region: 'auto',
    endpoint: "https://#{ENV['R2_ACCOUNT_ID']}.r2.cloudflarestorage.com",
    path_style: true
  }
  config.fog_directory = ENV['R2_BUCKET_NAME']
  config.fog_public = false
end
```

### Shrine 設定

```ruby
# config/initializers/shrine.rb
require 'shrine/storage/s3'

Shrine.storages = {
  cache: Shrine::Storage::S3.new(
    bucket: ENV['R2_BUCKET_NAME'],
    prefix: 'cache',
    **r2_options
  ),
  store: Shrine::Storage::S3.new(
    bucket: ENV['R2_BUCKET_NAME'],
    prefix: 'store',
    **r2_options
  )
}

def r2_options
  {
    access_key_id: ENV['R2_ACCESS_KEY_ID'],
    secret_access_key: ENV['R2_SECRET_ACCESS_KEY'],
    region: 'auto',
    endpoint: "https://#{ENV['R2_ACCOUNT_ID']}.r2.cloudflarestorage.com",
    force_path_style: true
  }
end
```

---

## サービスクラスの例

### R2 ストレージサービス

```ruby
# app/services/r2_storage_service.rb
class R2StorageService
  def initialize
    @client = Aws::S3::Client.new(
      access_key_id: ENV['R2_ACCESS_KEY_ID'],
      secret_access_key: ENV['R2_SECRET_ACCESS_KEY'],
      endpoint: "https://#{ENV['R2_ACCOUNT_ID']}.r2.cloudflarestorage.com",
      region: 'auto'
    )
    @bucket = ENV['R2_BUCKET_NAME']
  end

  def upload(key, file_or_content, options = {})
    body = file_or_content.respond_to?(:read) ? file_or_content : file_or_content

    @client.put_object(
      bucket: @bucket,
      key: key,
      body: body,
      content_type: options[:content_type],
      metadata: options[:metadata] || {}
    )

    key
  end

  def download(key)
    response = @client.get_object(bucket: @bucket, key: key)
    response.body.read
  rescue Aws::S3::Errors::NoSuchKey
    nil
  end

  def delete(key)
    @client.delete_object(bucket: @bucket, key: key)
    true
  rescue Aws::S3::Errors::NoSuchKey
    false
  end

  def exists?(key)
    @client.head_object(bucket: @bucket, key: key)
    true
  rescue Aws::S3::Errors::NotFound
    false
  end

  def url(key, expires_in: 3600)
    signer = Aws::S3::Presigner.new(client: @client)
    signer.presigned_url(:get_object, bucket: @bucket, key: key, expires_in: expires_in)
  end

  def list(prefix: nil, max_keys: 1000)
    options = { bucket: @bucket, max_keys: max_keys }
    options[:prefix] = prefix if prefix

    response = @client.list_objects_v2(options)
    response.contents.map do |object|
      {
        key: object.key,
        size: object.size,
        last_modified: object.last_modified
      }
    end
  end
end
```

### 使用例

```ruby
# サービスの使用
storage = R2StorageService.new

# アップロード
storage.upload('images/photo.jpg', File.read('photo.jpg'), content_type: 'image/jpeg')

# ダウンロード
content = storage.download('images/photo.jpg')
File.write('downloaded.jpg', content)

# 署名付きURL取得
url = storage.url('images/photo.jpg', expires_in: 7200)

# 一覧取得
files = storage.list(prefix: 'images/')
files.each { |f| puts "#{f[:key]} - #{f[:size]} bytes" }

# 削除
storage.delete('images/photo.jpg')
```

---

## エラーハンドリング

```ruby
begin
  r2_client.get_object(bucket: bucket, key: key)
rescue Aws::S3::Errors::NoSuchKey
  puts "Object not found"
rescue Aws::S3::Errors::NoSuchBucket
  puts "Bucket not found"
rescue Aws::S3::Errors::AccessDenied
  puts "Access denied - check your credentials"
rescue Aws::S3::Errors::ServiceError => e
  puts "S3 error: #{e.message}"
end
```

---

## R2 と S3 の違い・制限事項

### サポートされていない機能

| 機能 | 状態 |
|------|------|
| オブジェクトのACL | 非サポート |
| バケットポリシー | 非サポート |
| オブジェクトロック | 非サポート |
| バージョニング | 非サポート |
| レプリケーション | 非サポート |

### R2 の特徴

- **エグレス料金無料**: データ転送出力に料金がかからない
- **S3互換API**: 既存のS3コードをそのまま使用可能
- **グローバルエッジ**: Cloudflareのエッジネットワークを活用

---

## 参考リンク

- [Cloudflare R2 公式ドキュメント](https://developers.cloudflare.com/r2/)
- [aws-sdk-ruby with R2](https://developers.cloudflare.com/r2/examples/aws/aws-sdk-ruby/)
- [R2 S3 API互換性](https://developers.cloudflare.com/r2/api/s3/api/)
- [aws-sdk-s3 Gem](https://rubygems.org/gems/aws-sdk-s3)

← [Gem一覧に戻る](README.md)
