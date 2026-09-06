# GCP Cloud Run 入門（学習用ハンズオン）

Node.js + Express の最小アプリを Cloud Run にデプロイするまでの手順メモ。
GCS 関連は [gcs_upload.md](gcs_upload.md) / [gcs_upload_nextjs.md](gcs_upload_nextjs.md) を参照。

このメモの前提：

- サービス名 `sample-app` / リージョン `asia-northeast1`（東京）で統一
- プロジェクトIDは `YOUR_PROJECT_ID` と表記（自分のIDに読み替える）
- **学習後に必ず「8. 後片付け」を実施すること**

---

## 1. Cloud Run とは

コンテナイメージを渡すと HTTPS の URL が生えるフルマネージド実行環境。サーバーの管理は不要。

- **リクエストが来たときだけコンテナが起動する。** アイドル時はインスタンス数 0 になるので、使わなければ課金されない（既定 `min-instances=0`）
- アプリ側の必須ルールはほぼ 1 つだけ：**環境変数 `PORT`（既定 8080）を `0.0.0.0` で listen する**
- デプロイ単位は **リビジョン**。デプロイのたびに新しいリビジョンが作られ、トラフィックの向き先を切り替えられる（＝ロールバックが簡単）
- ステートレス前提。ローカルディスクに書いても次のインスタンスには残らない（永続化は GCS や Cloud SQL へ）

---

## 2. 事前準備

### gcloud CLI

```bash
# インストール確認（なければ https://cloud.google.com/sdk/docs/install から）
gcloud version

# ログイン
gcloud auth login
```

### プロジェクト作成

```bash
# プロジェクトIDは全GCPでユニークである必要がある
gcloud projects create YOUR_PROJECT_ID

# 以降のコマンドの既定プロジェクトにする
gcloud config set project YOUR_PROJECT_ID
```

> ⚠️ **課金アカウントのリンクが必須。** 無料枠の範囲で使う場合でも、請求先アカウントを紐付けないと Cloud Run はデプロイできない。
> コンソール（https://console.cloud.google.com/billing）から作成したプロジェクトに請求先アカウントをリンクしておく。

### API 有効化

```bash
gcloud services enable \
  run.googleapis.com \
  cloudbuild.googleapis.com \
  artifactregistry.googleapis.com
```

- `run` … Cloud Run 本体
- `cloudbuild` … ソースからのビルド
- `artifactregistry` … ビルドしたコンテナイメージの保管先

### 既定リージョン

```bash
gcloud config set run/region asia-northeast1
```

設定しておくと以降の `--region` を省略できる（このメモでは明示的に書く）。

---

## 3. サンプルアプリ

```bash
mkdir cloud-run-sample && cd cloud-run-sample
npm init -y
npm install express
```

`package.json`（`start` スクリプトが必須。Buildpacks はこれを見て起動する）:

```json
{
  "name": "cloud-run-sample",
  "version": "1.0.0",
  "main": "index.js",
  "scripts": {
    "start": "node index.js"
  },
  "engines": {
    "node": ">=22"
  },
  "dependencies": {
    "express": "^5.1.0"
  }
}
```

`index.js`:

```js
const express = require('express');
const app = express();

app.get('/', (req, res) => {
  res.send('Hello Cloud Run!');
});

app.get('/healthz', (req, res) => {
  res.json({ status: 'ok', revision: process.env.K_REVISION ?? 'local' });
});

// Cloud Run は PORT 環境変数でポートを指定してくる。ハードコード禁止
const port = process.env.PORT || 8080;
app.listen(port, '0.0.0.0', () => {
  console.log(`listening on ${port}`);
});
```

- `K_REVISION` は Cloud Run が自動で入れる環境変数。どのリビジョンが応答したか確認できる

### ローカル確認

```bash
npm start
curl http://localhost:8080
# => Hello Cloud Run!
```

---

## 4. 方式A: ソースから直接デプロイ

Dockerfile を書かずに、ソースコードをそのまま投げる方法。まずはこれで動かす。

```bash
gcloud run deploy sample-app \
  --source . \
  --region asia-northeast1 \
  --allow-unauthenticated
```

初回は「Artifact Registry のリポジトリを作るか？」と聞かれるので `y`。

裏で起きていること：

1. ソースが Cloud Storage にアップロードされる
2. Cloud Build が **Buildpacks** で言語を自動判定してコンテナイメージをビルド（`package.json` の `start` が起動コマンドになる）
3. イメージが Artifact Registry の `cloud-run-source-deploy` リポジトリに push される
4. そのイメージで Cloud Run のリビジョンが作られる

`--allow-unauthenticated` は「誰でもURLを叩ける」設定。付けないと IAM 認証が必要になり、ブラウザで開くと 403 になる。

### 動作確認

```bash
# 発行された URL を取得
gcloud run services describe sample-app \
  --region asia-northeast1 \
  --format 'value(status.url)'

curl "$(gcloud run services describe sample-app --region asia-northeast1 --format 'value(status.url)')"
# => Hello Cloud Run!
```

### 更新してリビジョンを増やす

`index.js` のメッセージを書き換えて、同じコマンドをもう一度実行する。

```bash
gcloud run deploy sample-app --source . --region asia-northeast1
# ※ --allow-unauthenticated は初回に設定済みなので2回目以降は不要

gcloud run revisions list --service sample-app --region asia-northeast1
```

新しいリビジョンが作られ、トラフィックが 100% そちらに向いていることを確認する。

---

## 5. 方式B: Dockerfile + Artifact Registry

方式Aが自動でやっていたことを手動で分解する。イメージの中身を自分で制御したい場合はこちら。

### Dockerfile

```dockerfile
FROM node:22-slim

WORKDIR /app

# 依存だけ先に入れてレイヤーキャッシュを効かせる
COPY package*.json ./
RUN npm ci --omit=dev

COPY . .

# Cloud Run は PORT を渡してくるので EXPOSE は必須ではない（ドキュメント目的）
EXPOSE 8080

CMD ["node", "index.js"]
```

`.dockerignore`:

```
node_modules
npm-debug.log
.git
.env*
Dockerfile
.dockerignore
```

### ローカルで動作確認

```bash
docker build -t sample-app .
docker run --rm -e PORT=8080 -p 8080:8080 sample-app

curl http://localhost:8080
```

### Artifact Registry へ push

```bash
# リポジトリ作成（方式Aが作る cloud-run-source-deploy とは別に自分で用意する）
gcloud artifacts repositories create sample-repo \
  --repository-format=docker \
  --location=asia-northeast1 \
  --description="Cloud Run 学習用"

# docker コマンドが Artifact Registry に push できるよう認証設定
gcloud auth configure-docker asia-northeast1-docker.pkg.dev

# タグ付け
docker build --platform linux/amd64 \
  -t asia-northeast1-docker.pkg.dev/YOUR_PROJECT_ID/sample-repo/sample-app:v1 .

# push
docker push asia-northeast1-docker.pkg.dev/YOUR_PROJECT_ID/sample-repo/sample-app:v1
```

> ⚠️ **Apple Silicon (M1/M2/M3...) は要注意。** そのまま build すると arm64 イメージになり、Cloud Run（amd64）で起動に失敗する。`--platform linux/amd64` を必ず付ける。

### デプロイ

```bash
gcloud run deploy sample-app \
  --image asia-northeast1-docker.pkg.dev/YOUR_PROJECT_ID/sample-repo/sample-app:v1 \
  --region asia-northeast1 \
  --allow-unauthenticated
```

### 代替: ローカル build せず Cloud Build に投げる

ローカルに Docker がない場合や、プラットフォームの差異を気にしたくない場合。

```bash
gcloud builds submit --tag asia-northeast1-docker.pkg.dev/YOUR_PROJECT_ID/sample-repo/sample-app:v2
```

---

## 6. 運用の基本

### サービスとリビジョン

```bash
# サービス一覧
gcloud run services list --region asia-northeast1

# 詳細（URL、現在のイメージ、リソース設定など）
gcloud run services describe sample-app --region asia-northeast1

# リビジョン一覧
gcloud run revisions list --service sample-app --region asia-northeast1
```

### ロールバック

```bash
# 特定リビジョンに 100% 戻す
gcloud run services update-traffic sample-app \
  --region asia-northeast1 \
  --to-revisions sample-app-00001-abc=100

# 常に最新リビジョンへ戻す
gcloud run services update-traffic sample-app \
  --region asia-northeast1 \
  --to-latest
```

カナリアリリースもトラフィック分割でできる（`--to-revisions REV_A=90,REV_B=10`）。

### ログ

```bash
gcloud run services logs read sample-app --region asia-northeast1 --limit 50

# 追従
gcloud beta run services logs tail sample-app --region asia-northeast1
```

コンソールなら Cloud Logging（https://console.cloud.google.com/logs）でフィルタして見るのが楽。
アプリが標準出力に書いたものがそのままログになる。

### リソース設定

```bash
gcloud run services update sample-app \
  --region asia-northeast1 \
  --min-instances 0 \
  --max-instances 3 \
  --memory 512Mi \
  --cpu 1 \
  --concurrency 80 \
  --timeout 300
```

| オプション | 意味 |
| --- | --- |
| `--min-instances` | 常時起動しておくインスタンス数。**1以上にすると常時課金**（コールドスタート対策） |
| `--max-instances` | 上限。暴走時の課金上限にもなるので学習中は小さくしておく |
| `--memory` / `--cpu` | 1インスタンスあたりの割り当て。課金単価に直結 |
| `--concurrency` | 1インスタンスが同時に受けるリクエスト数（既定 80）。小さいほどインスタンスが増える |
| `--timeout` | 1リクエストの最大処理時間（秒） |

---

## 7. 料金と無料枠

課金軸は主に **リクエスト数 / vCPU 秒 / メモリ GiB 秒**。リクエスト処理中だけカウントされるので、**アイドル時は 0 円**。

- 毎月一定の無料枠がある。**枠の数値は変わるので公式を確認する** → https://cloud.google.com/run/pricing
- `--min-instances` を 1 以上にすると、リクエストが無くても課金され続ける。学習中は **0 のまま** にしておく
- Cloud Run 以外にも少額の課金要素がある：
  - **Artifact Registry** … 保存したイメージの容量
  - **Cloud Build** … ビルド時間（無料枠あり）
  - **Cloud Storage** … `--source` デプロイ時のソース保管
  - **Cloud Logging** … ログ保存量

### 予算アラートを設定しておく

コンソールの Billing →「予算とアラート」で、少額（例：1,000円）の予算としきい値通知を作っておくと事故が防げる。
課金を自動で止める機能ではないが、気付ける。

---

## 8. 後片付け（重要）

学習が終わったら、作ったリソースを消す。上から順に実行する。

```bash
# 1. Cloud Run サービス
gcloud run services delete sample-app --region asia-northeast1

# 2. Artifact Registry（方式Aが自動生成したもの）
gcloud artifacts repositories delete cloud-run-source-deploy --location asia-northeast1

# 3. Artifact Registry（方式Bで自分が作ったもの）
gcloud artifacts repositories delete sample-repo --location asia-northeast1

# 4. Cloud Build がソース保管に使ったバケットを確認して削除
gcloud storage buckets list --format 'value(name)'
# YOUR_PROJECT_ID_cloudbuild や run-sources-YOUR_PROJECT_ID-asia-northeast1 などが該当
gcloud storage rm -r gs://YOUR_PROJECT_ID_cloudbuild
```

### 一番確実な方法：プロジェクトごと削除

消し忘れが怖いので、学習用に作ったプロジェクトなら丸ごと消すのが安全。

```bash
gcloud projects delete YOUR_PROJECT_ID
```

削除は 30 日間の猶予があり、その間は `gcloud projects undelete YOUR_PROJECT_ID` で復元できる。

---

## 9. つまずきポイント

| 症状 | 原因と対処 |
| --- | --- |
| `The user-provided container failed to start and listen on the port defined by the PORT environment variable` | ポートをハードコードしている。`process.env.PORT` を使う |
| デプロイは成功するが応答がない | `localhost` / `127.0.0.1` で listen している。`0.0.0.0` にバインドする |
| コンテナが起動しない（Apple Silicon から push） | arm64 イメージになっている。`docker build --platform linux/amd64` |
| `403 Forbidden` がブラウザに出る | `--allow-unauthenticated` を付け忘れ。`gcloud run services add-iam-policy-binding sample-app --region asia-northeast1 --member=allUsers --role=roles/run.invoker` で後付けも可 |
| `API [run.googleapis.com] not enabled` | 「2. 事前準備」の API 有効化を実行する |
| `billing account ... is not found` 系 | プロジェクトに請求先アカウントがリンクされていない |
| ビルドは通るのに起動が遅い（初回だけ数秒待つ） | コールドスタート。気になるなら `--min-instances 1`（ただし常時課金） |

---

## 10. 次にやること

- 環境変数と **Secret Manager** 連携（`--set-env-vars` / `--set-secrets`）
- **カスタムドメイン**のマッピング
- **GitHub push による継続的デプロイ**（Cloud Build トリガー）
- **Cloud SQL** への接続
- **Next.js** のデプロイ（`output: 'standalone'` ビルド + Dockerfile）
- Cloud Run **ジョブ**（HTTP サーバーではないバッチ処理用）
