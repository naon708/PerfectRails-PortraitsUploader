# Active Storage
- https://railsguides.jp/active_storage_overview.html
- https://gorails.com/tool_categories/file-uploading/tools

## 2つのモデルで構成される

```bash
SomeModel >-- Attachment --< Blob
```

- Blob
    - 「塊、かたまり」という意味
    - **ファイルの実態を表すオブジェクト**
    - ファイルのメタデータを持ち、ファイルの格納場所やファイル名などを管理する役割(= バイナリデータ)
- Attachment
    - Blobにアクセスするための中間テーブル
- ポリモーフィック関連を採用している
    - 他のGemだと対象モデルに画像を扱う用のカラムを生やすことが多い

## ファイルアップロードの流れ

```bash
User Create
ActiveStorage::Blob Create
ActiveStorage::Attachment Create
User Update

Disk Storage (3.1ms) Uploaded file to key: akn5g65qcknemhjff4wckq7u735j (checksum: pgr4sMPfB8TTMYIoGVMS6w==)

[ActiveJob] Enqueued ActiveStorage::AnalyzeJob

Redirected to http://localhost:3000/users/1
Completed 302
```

- Userレコード作成→Blobレコード作成→Attachmentレコード作成
- Userのupdated_atが更新される(アップロードが成功したタイミングで)
- ファイルアップロード
    - Disk Storage: デフォルトではRailsのstorageディレクトリ。productionではS3とかを指定する。
    - key: ファイルに割り当てられる一意な識別子
    - checksum: データが壊れてないかチェックするための値
- AnalyzeJobがエンキューされた
- 302レスポンス

```bash
ActiveStorage::Blob Load

[ActiveJob] Performing ActiveStorage::AnalyzeJob
[ActiveJob] Disk Storage (0.6ms) Downloaded file from key: akn5g65qcknemhjff4wckq7u735j
[ActiveJob] ActiveStorage::Blob Update (2.1ms)  UPDATE "active_storage_blobs" SET "metadata" = ? WHERE "active_storage_blobs"."id" = ?  [["metadata", "{\"identified\":true,\"analyzed\":true}"], ["id", 1]]
[ActiveJob] Performed ActiveStorage::AnalyzeJob

User Load
ActiveStorage::Attachment Load
ActiveStorage::Blob Load

Rendered users/show.html.erb
Completed 200
```

- Blobを取得
- ジョブが実行されてDisk Storageからkeyを指定してファイルをダウンロード
- ファイルのメタデータが更新された
    - `analyzed: true` : 画像の解析が完了したことを表す(画像のフォーマットや幅・高さなどの情報)
- User, AttachmentからBlobを取得
- 200レスポンス

## アップロードしたファイルを表示したとき

```bash
Started GET "/rails/active_storage/blobs/eyJfcmFpbHMiOnsibWVzc2FnZSI6IkJBaHBCZz09IiwiZXhwIjpudWxsLCJwdXIiOiJibG9iX2lkIn19--1135d6255dfc40eeb97271a21bd3663aacb20cca/sample.png"

Processing by ActiveStorage::BlobsController#show as PNG

Parameters: {"signed_id"=>"eyJfcmFpbHMiOnsibWVzc2FnZSI6IkJBaHBCZz09IiwiZXhwIjpudWxsLCJwdXIiOiJibG9iX2lkIn19--1135d6255dfc40eeb97271a21bd3663aacb20cca", "filename"=>"sample"}

Blob Load

Disk Storage (8.8ms) Generated URL for file at key

Redirected
Completed 302

Started GET "/rails/active_storage/disk/eyJfcmFpbHMiOnsibWVzc2FnZSI6IkJBaDdDRG9JYTJWNVNTSWhZV3R1TldjMk5YRmphMjVsYldocVptWTBkMk5yY1RkMU56TTFhZ1k2QmtWVU9oQmthWE53YjNOcGRHbHZia2tpUldsdWJHbHVaVHNnWm1sc1pXNWhiV1U5SW5CcGJtRndYMjluY0M1d2JtY2lPeUJtYVd4bGJtRnRaU285VlZSR0xUZ25KM0JwYm1Gd1gyOW5jQzV3Ym1jR093WlVPaEZqYjI1MFpXNTBYM1I1Y0dWSklnNXBiV0ZuWlM5d2JtY0dPd1pVIiwiZXhwIjoiMjAyMy0wNC0wMVQwNTozNTo1OS42NThaIiwicHVyIjoiYmxvYl9rZXkifX0=--248a1c1badd7903ae9cacade7121973a8c99f47f/pinap_ogp.png?content_type=image%2Fpng&disposition=inline%3B+filename%3D%22pinap_ogp.png%22%3B+filename%2A%3DUTF-8%27%27sample.png"

Processing by ActiveStorage::DiskController#show as PNG

Parameters: {"content_type"=>"image/png", "disposition"=>"inline; filename=\"pinap_ogp.png\"; filename*=UTF-8''pinap_ogp.png", "encoded_key"=>"eyJfcm....", "filename"=>"sample"}

Completed 200
```

- Blobにアクセス
- ファイルのKeyをDisk Storageに渡して、生成されたファイルのURLを受け取る
- 302リダイレクト(デフォルトで5分の期限が付くらしい)
- Diskにアクセス(ファイルそのものの住所)
- 200レスポンス

## ファイルの保存場所の設定

- config/storage.yml で保存場所を定義する
- config/environments/**.rb で↑で定義したものを指定する
    
    ```ruby
    config.active_storage.service = :local
    ```
    

## ファイルへのアクセス制限

- すべての画像ファイルそのものにアクセス制限をかけるのはつらすぎる
- 画像を表示するためのページにアクセス制限をかけるのが妥当
- 生成されたファイルのURLには期限が設定される(URLが漏洩した際の被害を抑えるため)

## ダイレクトアップロード

- サーバーを経由せず直接クラウドストレージにアップロードできる
- アップロード時間短縮・サーバーへの負荷軽減

```ruby
Started POST "/rails/active_storage/direct_uploads"

Processing by ActiveStorage::DirectUploadsController#create as JSON

Parameters: {"blob"=>{"filename"=>"sample.jpg", "content_type"=>"image/jpeg", "byte_size"=>275507, "checksum"=>"WK8YEszi7qla+wEjgFYYrA=="}, "direct_upload"=>{"blob"=>{"filename"=>"sample.jpg", "content_type"=>"image/jpeg", "byte_size"=>275507, "checksum"=>"WK8YEszi7qla+wEjgFYYrA=="}}}

ActiveStorage::Blob Create

Completed 200 OK in 17ms (Views: 0.4ms | ActiveRecord: 8.1ms | Allocations: 5787)

# ~~~~~~~~~~~~~~~~~~~~~

Started PUT "/rails/active_storage/disk/eyJfcmFpbHMi

# ~~~~~~~~~~~~~~~~~~~~~

Started POST "/users"
Processing by UsersController#create as HTML

User Create
ActiveStorage::Blob Load
ActiveStorage::Attachment Create
User Update
ActiveStorage::Blob Update
....

Completed 200

```

- ダイレクトにBlobが作成される
- Diskが更新される
- User作成→作成済みのBlobをLoad→Attachment作成→User, Blobの値を更新して完了
- [https://railsguides.jp/active_storage_overview.html#ダイレクトアップロード](https://railsguides.jp/active_storage_overview.html#%E3%83%80%E3%82%A4%E3%83%AC%E3%82%AF%E3%83%88%E3%82%A2%E3%83%83%E3%83%97%E3%83%AD%E3%83%BC%E3%83%89)

## バリデーション機能がない

- バリデーションを追加する場合は以下の選択肢になる
    - active_storage_validations などのGemを使う
    - 他のライブラリを使う(e.g. CarrirerWave, Shrine)

## キャッシュ機能がない

- キャッシュが必要なケース
    - 再アップロードが必要なとき
    - 非同期でアップロードする場合
