# GCS に画像を保存するリクエストのサンプル

Google Cloud Storage (GCS) に画像をアップロードする各種方法のメモ。
Next.js 版は [gcs_upload_nextjs.md](gcs_upload_nextjs.md) を参照。

## 1. REST API を直接叩く (curl)

```bash
# 単純アップロード (media upload)
curl -X POST \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "Content-Type: image/png" \
  --data-binary @./sample.png \
  "https://storage.googleapis.com/upload/storage/v1/b/YOUR_BUCKET/o?uploadType=media&name=images/sample.png"
```

- `YOUR_BUCKET`: バケット名
- `name=images/sample.png`: オブジェクトのパス(URL エンコードが必要な文字がある場合は注意)

## 2. Node.js (`@google-cloud/storage`)

```js
const { Storage } = require("@google-cloud/storage");
const storage = new Storage(); // 認証は環境変数 GOOGLE_APPLICATION_CREDENTIALS など

async function uploadImage() {
  const bucket = storage.bucket("YOUR_BUCKET");

  // ローカルファイルをアップロード
  await bucket.upload("./sample.png", {
    destination: "images/sample.png",
    metadata: { contentType: "image/png" },
  });

  // または Buffer から直接保存
  const file = bucket.file("images/from-buffer.png");
  await file.save(imageBuffer, {
    contentType: "image/png",
    resumable: false, // 小さいファイルは false が速い
  });

  console.log("uploaded");
}
```

## 3. Python (`google-cloud-storage`)

```python
from google.cloud import storage

client = storage.Client()
bucket = client.bucket("YOUR_BUCKET")
blob = bucket.blob("images/sample.png")

# ファイルから
blob.upload_from_filename("./sample.png", content_type="image/png")

# または bytes から
# blob.upload_from_string(image_bytes, content_type="image/png")
```

## 4. 署名付き URL 経由でクライアントから直接アップロード

サーバーから URL を発行し、ブラウザ/アプリが直接 PUT するパターン(サーバーに画像を経由させない)。

```js
// サーバー側: 署名付きURLを生成
const [url] = await storage
  .bucket("YOUR_BUCKET")
  .file("images/sample.png")
  .getSignedUrl({
    version: "v4",
    action: "write",
    expires: Date.now() + 15 * 60 * 1000, // 15分
    contentType: "image/png",
  });
```

```bash
# クライアント側: 発行されたURLへPUT
curl -X PUT -H "Content-Type: image/png" --data-binary @./sample.png "SIGNED_URL"
```
