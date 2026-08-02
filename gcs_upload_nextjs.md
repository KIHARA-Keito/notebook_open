# Next.js で GCS に画像をアップロードする

Google Cloud Storage (GCS) に画像を保存する方法のメモ。**サーバー経由**と**署名付き URL(クライアント直接)** の 2 パターン。

## 事前準備

```bash
npm install @google-cloud/storage
```

`.env.local`:

```bash
GCP_PROJECT_ID=your-project-id
GCS_BUCKET=your-bucket-name
# サービスアカウントキーのパス、または内容を直接持たせる
GOOGLE_APPLICATION_CREDENTIALS=./service-account.json
```

Storage クライアントは使い回すので共通化しておく。

```ts
// lib/gcs.ts
import { Storage } from "@google-cloud/storage";

export const storage = new Storage({
  projectId: process.env.GCP_PROJECT_ID,
  // GOOGLE_APPLICATION_CREDENTIALS を使わず環境変数でキーを渡す場合:
  // credentials: {
  //   client_email: process.env.GCP_CLIENT_EMAIL,
  //   private_key: process.env.GCP_PRIVATE_KEY?.replace(/\\n/g, '\n'),
  // },
});

export const bucket = storage.bucket(process.env.GCS_BUCKET!);
```

---

## パターン A: サーバー経由でアップロード (Route Handler)

小〜中サイズの画像で、サーバー側でバリデーションやリサイズもしたい場合向け。

```ts
// app/api/upload/route.ts
import { NextRequest, NextResponse } from "next/server";
import { randomUUID } from "crypto";
import { bucket } from "@/lib/gcs";

export async function POST(req: NextRequest) {
  const formData = await req.formData();
  const file = formData.get("file");

  if (!(file instanceof File)) {
    return NextResponse.json({ error: "file is required" }, { status: 400 });
  }
  if (!file.type.startsWith("image/")) {
    return NextResponse.json({ error: "only images allowed" }, { status: 400 });
  }

  const buffer = Buffer.from(await file.arrayBuffer());
  const ext = file.name.split(".").pop() ?? "png";
  const objectName = `images/${randomUUID()}.${ext}`;

  const blob = bucket.file(objectName);
  await blob.save(buffer, {
    contentType: file.type,
    resumable: false,
  });

  // 公開バケットなら公開URLを返す
  const publicUrl = `https://storage.googleapis.com/${bucket.name}/${objectName}`;

  return NextResponse.json({ name: objectName, url: publicUrl });
}
```

クライアント側:

```tsx
"use client";
import { useState } from "react";

export default function Uploader() {
  const [url, setUrl] = useState<string>();

  async function handleChange(e: React.ChangeEvent<HTMLInputElement>) {
    const file = e.target.files?.[0];
    if (!file) return;

    const fd = new FormData();
    fd.append("file", file);

    const res = await fetch("/api/upload", { method: "POST", body: fd });
    const data = await res.json();
    setUrl(data.url);
  }

  return (
    <div>
      <input type="file" accept="image/*" onChange={handleChange} />
      {url && <img src={url} alt="uploaded" width={200} />}
    </div>
  );
}
```

> ⚠️ Route Handler のボディサイズには制限がある(特に Vercel は制限が厳しい)。大きい画像を扱うなら次のパターン B を推奨。

---

## パターン B: 署名付き URL でクライアントから直接アップロード(推奨)

サーバーは署名付き URL を発行するだけで、画像本体はブラウザ →GCS へ直接送るのでサーバー負荷・帯域を節約できる。大きいファイルにも強い。

```ts
// app/api/upload-url/route.ts
import { NextRequest, NextResponse } from "next/server";
import { randomUUID } from "crypto";
import { bucket } from "@/lib/gcs";

export async function POST(req: NextRequest) {
  const { contentType } = await req.json();

  if (!contentType?.startsWith("image/")) {
    return NextResponse.json({ error: "invalid contentType" }, { status: 400 });
  }

  const objectName = `images/${randomUUID()}`;
  const file = bucket.file(objectName);

  const [uploadUrl] = await file.getSignedUrl({
    version: "v4",
    action: "write",
    expires: Date.now() + 15 * 60 * 1000, // 15分
    contentType,
  });

  return NextResponse.json({
    uploadUrl,
    objectName,
    publicUrl: `https://storage.googleapis.com/${bucket.name}/${objectName}`,
  });
}
```

クライアント側:

```tsx
"use client";
import { useState } from "react";

export default function DirectUploader() {
  const [url, setUrl] = useState<string>();

  async function handleChange(e: React.ChangeEvent<HTMLInputElement>) {
    const file = e.target.files?.[0];
    if (!file) return;

    // 1. サーバーから署名付きURLを取得
    const res = await fetch("/api/upload-url", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ contentType: file.type }),
    });
    const { uploadUrl, publicUrl } = await res.json();

    // 2. GCSへ直接PUT (Content-Type は署名時と一致させる)
    await fetch(uploadUrl, {
      method: "PUT",
      headers: { "Content-Type": file.type },
      body: file,
    });

    setUrl(publicUrl);
  }

  return (
    <div>
      <input type="file" accept="image/*" onChange={handleChange} />
      {url && <img src={url} alt="uploaded" width={200} />}
    </div>
  );
}
```

---

## 補足

- **CORS**: パターン B でブラウザから直接 PUT する場合、バケットに CORS 設定が必要。

  ```json
  [
    {
      "origin": ["http://localhost:3000"],
      "method": ["PUT"],
      "responseHeader": ["Content-Type"],
      "maxAgeSeconds": 3600
    }
  ]
  ```

  ```bash
  gsutil cors set cors.json gs://your-bucket-name
  ```

- **runtime**: `@google-cloud/storage` は Node.js 依存なので、Route Handler に `export const runtime = 'nodejs';` を明示しておくと安全(Edge では動かない)。
- **認証**: Cloud Run / GCE 上なら鍵ファイル不要で、アタッチされたサービスアカウントが自動で使われる。ローカルは `gcloud auth application-default login` でも可。
