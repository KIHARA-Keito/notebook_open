# Cloud Run 学習プラン（初心者向け）

Web サービスを「作る」から「動かし続ける」へ進むための、Google Cloud Run 学習プラン。
用語を理解する → 最小構成で動かす → 実サービスに必要な要素を足す、の順で進める。

想定スタック: Node.js (Express) → 慣れたら Next.js
想定期間: 週 5〜8 時間 × 約 8〜10 週間（Step 0〜9）
関連メモ: [gcs_upload.md](gcs_upload.md) / [gcs_upload_nextjs.md](gcs_upload_nextjs.md)

> ⚠️ 課金の事故防止のため、**Step 1 の予算アラート設定を必ず先に済ませる**こと。
> 学習が終わったら [Step 10 の後片付け](#step-10-後片付けと運用の習慣化2〜3-時間) を実施する。

---

## 前提: 用語の整理

| 用語             | 意味                                           | 補足                            |
| ---------------- | ---------------------------------------------- | ------------------------------- |
| コンテナ         | アプリと実行環境をまとめた箱                   | どのマシンでも同じように動く    |
| イメージ         | コンテナの設計図（ビルド成果物）               | Dockerfile から作る             |
| Cloud Run        | コンテナを渡すと HTTPS の URL が生える実行環境 | サーバの構築・管理が不要        |
| サービス         | 常時 URL を持つ Cloud Run のデプロイ単位       | Web アプリはこちら              |
| ジョブ           | 実行して終了する処理を動かす単位               | バッチ、日次集計など            |
| リビジョン       | デプロイのたびに作られるスナップショット       | 切り戻しの単位                  |
| コールドスタート | 停止状態から起動する際の初回遅延               | インスタンス 0 まで縮むため発生 |

まず押さえるべき原則:

- Cloud Run のアプリは**ステートレス**。ローカルディスクに書いた内容は次のインスタンスに残らない。
- アプリ側の必須ルールはほぼ 1 つ。**環境変数 `PORT`（既定 8080）を `0.0.0.0` で listen する**。
- **リクエストが来たときだけ動く**。既定の `min-instances=0` なら、使わない時間は課金されない。
- 設定は**環境変数とシークレット**で外に出す。イメージに秘密情報を焼き込まない。

---

## 学習マップ

```
Step 0  なぜ Cloud Run か（サーバの選択肢を知る）
   ↓
Step 1  Google Cloud のセットアップ + 予算アラート  ← 最初に必ず
   ↓
Step 2  Docker の基礎（ローカルでコンテナを動かす）
   ↓
Step 3  最小アプリをデプロイする  ★ 最重要
   ↓
Step 4  Cloud Run の中核概念（リビジョン / スケーリング / 同時実行）
   ↓
Step 5  設定とシークレットの管理
   ↓
Step 6  データの永続化（Cloud SQL / GCS）
   ↓
Step 7  アクセス制御と IAM
   ↓
Step 8  ログ・監視・トラブルシュート
   ↓
Step 9  CI/CD と独自ドメイン
   ↓
Step 10 後片付けと運用の習慣化
```

---

## Step 0. なぜ Cloud Run か（2〜3 時間）

いきなり手を動かす前に、「どこにアプリを置くか」の選択肢を整理する。ここが分かると
Cloud Run が何を肩代わりしてくれているかが腹落ちする。

- [ ] レンタルサーバ / VPS（自分で OS もミドルウェアも管理する）
- [ ] IaaS = Compute Engine, EC2（仮想マシンを借りる）
- [ ] PaaS / サーバーレス = Cloud Run, App Engine, AWS Lambda（実行環境を借りる）
- [ ] マネージド Kubernetes = GKE（多数のコンテナを束ねる。個人開発には過剰）
- [ ] Vercel / Netlify との違い（フロント特化 vs 任意のコンテナが動く）

**Cloud Run が向いている場面**: 常時起動が不要な Web API、個人開発、社内ツール、
アクセスが読めないサービス。**向かない場面**: WebSocket を長時間張り続ける用途、
GPU 常時利用、ローカルディスクに状態を持つ設計。

**課題**: 自分が作りたいサービスを 1 つ想定し、上の選択肢のどれが適切か理由付きで 3 行メモする。

---

## Step 1. Google Cloud のセットアップ（3〜4 時間）

### アカウントとプロジェクト

- [ ] Google Cloud アカウントを作成する
- [ ] 請求先アカウントを作成する（無料枠内でも Cloud Run のデプロイには紐付けが必要）
- [ ] プロジェクトを作成する（プロジェクト ID は全 Google Cloud で一意）

```bash
gcloud projects create YOUR_PROJECT_ID
gcloud config set project YOUR_PROJECT_ID
```

### 予算アラート（最優先）

課金画面から、作成したプロジェクトに紐づく請求先アカウントに予算を設定する。
金額は学習用なら 1,000 円程度で十分。50% / 90% / 100% でメール通知が来るようにしておく。

> 予算アラートは**通知するだけで課金は止まらない**。上限として過信しない。

### gcloud CLI

- [ ] gcloud CLI をインストールする
- [ ] `gcloud auth login` でログインする
- [ ] `gcloud config list` で既定プロジェクトを確認する

```bash
gcloud version
gcloud auth login
gcloud config set run/region asia-northeast1   # 東京リージョン
```

### API の有効化

```bash
gcloud services enable \
  run.googleapis.com \
  cloudbuild.googleapis.com \
  artifactregistry.googleapis.com
```

| API              | 役割                                 |
| ---------------- | ------------------------------------ |
| run              | Cloud Run 本体                       |
| cloudbuild       | ソースコードからイメージをビルドする |
| artifactregistry | ビルドしたイメージの保管庫           |

**課題**: `gcloud config list` の出力をメモに残し、プロジェクト ID とリージョンを固定する。

---

## Step 2. Docker の基礎（6〜8 時間）

Cloud Run は「コンテナを動かすサービス」なので、コンテナが分からないとすべてが呪文になる。
ここは Cloud Run から離れて、ローカルだけで完結させる。

- [ ] イメージとコンテナの違い（設計図と実体）
- [ ] `docker build` / `docker run` / `docker ps` / `docker logs`
- [ ] Dockerfile の基本命令（`FROM` `WORKDIR` `COPY` `RUN` `CMD`）
- [ ] レイヤーキャッシュ（依存インストールを先に書くとビルドが速くなる理由）
- [ ] ポートフォワード `-p 8080:8080` の意味
- [ ] `.dockerignore` で `node_modules` や `.env` を除外する

```dockerfile
# Dockerfile
FROM node:20-slim
WORKDIR /app

# 依存だけ先に入れてレイヤーキャッシュを効かせる
COPY package*.json ./
RUN npm ci --omit=dev

COPY . .
ENV PORT=8080
CMD ["node", "index.js"]
```

```bash
docker build -t sample-app .
docker run --rm -p 8080:8080 -e PORT=8080 sample-app
# http://localhost:8080 で確認
```

**課題**: Express の "Hello" アプリをコンテナ化し、ローカルで表示できるところまで到達する。
`PORT` を 3000 に変えて起動し、環境変数でポートが変わることを確認する。

---

## Step 3. 最小アプリをデプロイする（4〜6 時間）★ 最重要

ここが学習の山場。**まず URL が生えるところまで一気に行く。**

### アプリ側の作法

```js
// index.js
const express = require("express");
const app = express();

app.get("/", (req, res) => res.send("Hello Cloud Run!"));

// Cloud Run は PORT を渡してくる。0.0.0.0 で listen すること
const port = process.env.PORT || 8080;
app.listen(port, "0.0.0.0", () => console.log(`listening on ${port}`));
```

### 方式 A: ソースから直接デプロイ（初学者はこちら）

Dockerfile がなくてもよい。Cloud Build がビルドまで面倒を見てくれる。

```bash
gcloud run deploy sample-app \
  --source . \
  --region asia-northeast1 \
  --allow-unauthenticated
```

- [ ] 発行された URL をブラウザで開く
- [ ] コードを書き換えて再デプロイし、リビジョンが増えることを確認する

```bash
gcloud run services describe sample-app --region asia-northeast1 --format='value(status.url)'
```

### 方式 B: 自分でイメージをビルドして push

Dockerfile を自分で管理したくなったらこちら。CI/CD もこの流れになる。

```bash
# 保管庫を作る
gcloud artifacts repositories create my-repo \
  --repository-format=docker --location=asia-northeast1

# docker が push できるよう認証を通す
gcloud auth configure-docker asia-northeast1-docker.pkg.dev

IMAGE=asia-northeast1-docker.pkg.dev/YOUR_PROJECT_ID/my-repo/sample-app:v1
docker build -t $IMAGE .
docker push $IMAGE

gcloud run deploy sample-app --image $IMAGE \
  --region asia-northeast1 --allow-unauthenticated
```

> Apple Silicon の Mac でローカルビルドする場合は `--platform linux/amd64` を付ける。
> アーキテクチャ不一致は初心者が最も踏みやすい落とし穴。

**課題**: 方式 A と方式 B の両方でデプロイし、それぞれ何を自分で管理しているか比較してメモする。

---

## Step 4. Cloud Run の中核概念（6〜8 時間）

URL が生えた後に効いてくる知識。ここを飛ばすと、本番で必ず詰まる。

### リビジョンとトラフィック

- [ ] デプロイのたびに新しいリビジョンが作られる
- [ ] トラフィックを複数リビジョンに分割できる（カナリアリリース）
- [ ] 直前のリビジョンに戻せる（＝ロールバックが容易）

```bash
gcloud run revisions list --service sample-app --region asia-northeast1

# 新リビジョンに 10% だけ流す
gcloud run services update-traffic sample-app \
  --region asia-northeast1 --to-revisions sample-app-00002-abc=10

# 切り戻し（最新リビジョンへ 100%）
gcloud run services update-traffic sample-app \
  --region asia-northeast1 --to-latest
```

### スケーリングと同時実行

| 設定                 | 意味                             | 学習時の目安                              |
| -------------------- | -------------------------------- | ----------------------------------------- |
| `--min-instances`    | 常時待機させる数                 | 0（コールドスタートを許容し課金を抑える） |
| `--max-instances`    | 上限                             | 小さめに固定して暴走課金を防ぐ            |
| `--concurrency`      | 1 インスタンスが同時に処理する数 | 既定のまま。負荷試験後に調整              |
| `--cpu` / `--memory` | 割り当てリソース                 | まず既定。OOM が出たら上げる              |
| `--timeout`          | リクエストの最大時間             | 長時間処理はジョブに切り出す              |

```bash
gcloud run services update sample-app \
  --region asia-northeast1 --min-instances 0 --max-instances 3
```

- [ ] コールドスタートを体感する（しばらく放置してからアクセスし、初回だけ遅いことを確認）
- [ ] CPU の割り当てが「リクエスト処理中のみ」であることを理解する
      （バックグラウンド処理が止まる原因になる）
- [ ] ステートレス前提を理解する（インスタンス内のメモリやファイルは共有されない）

**課題**: `--max-instances` を 1 にして連続アクセスし、レスポンスがどう変わるか観察する。

---

## Step 5. 設定とシークレットの管理（4〜5 時間）

- [ ] 環境変数で設定を外出しする（`--set-env-vars`）
- [ ] API キーや DB パスワードは Secret Manager に置く
- [ ] `.env` やサービスアカウントキーを**イメージと Git に含めない**

```bash
gcloud services enable secretmanager.googleapis.com

echo -n "super-secret-value" | \
  gcloud secrets create API_KEY --data-file=-

gcloud run services update sample-app \
  --region asia-northeast1 \
  --set-env-vars "APP_ENV=production" \
  --set-secrets "API_KEY=API_KEY:latest"
```

アプリ側では通常の環境変数として `process.env.API_KEY` で読める。

**課題**: 秘密の値を環境変数直書きから Secret Manager 参照に移行し、
`gcloud run services describe` の出力に値が出ないことを確認する。

---

## Step 6. データの永続化（8〜10 時間）

Cloud Run 単体では何も残らない。保存先を外に持つ設計を学ぶ。

- [ ] 画像・ファイル → Cloud Storage（[gcs_upload.md](gcs_upload.md) 参照）
- [ ] リレーショナルデータ → Cloud SQL（PostgreSQL / MySQL）
- [ ] 接続方式の違い（Cloud SQL 接続の付与 / VPC 経由 / パブリック IP）
- [ ] コネクションプールの扱い（インスタンスが増減する前提で最大接続数に注意）

```bash
gcloud run services update sample-app \
  --region asia-northeast1 \
  --add-cloudsql-instances YOUR_PROJECT_ID:asia-northeast1:my-instance \
  --set-secrets "DB_PASSWORD=DB_PASSWORD:latest"
```

> Cloud SQL は**起動しているだけで課金される**。学習で使ったら必ず停止・削除する。
> 無料枠を重視するなら、まずは Firestore や GCS で代替できないか検討する。

**課題**: Cloud Run から Cloud SQL に接続し、1 テーブルの読み書きができる API を作る。

---

## Step 7. アクセス制御と IAM（5〜6 時間）

- [ ] `--allow-unauthenticated` の意味（誰でも呼べる公開サービスになる）
- [ ] 認証必須にする（`--no-allow-unauthenticated` + `roles/run.invoker` の付与）
- [ ] サービスアカウントとは何か。Cloud Run に専用のものを割り当てる
- [ ] 最小権限の原則（既定のサービスアカウントは権限が広いので使わない）
- [ ] サービス間呼び出し（ID トークンを付けて呼ぶ）

```bash
# 専用サービスアカウントを作って割り当てる
gcloud iam service-accounts create sample-app-sa

gcloud run services update sample-app \
  --region asia-northeast1 \
  --service-account sample-app-sa@YOUR_PROJECT_ID.iam.gserviceaccount.com

# 特定のユーザーだけ呼べるようにする
gcloud run services add-iam-policy-binding sample-app \
  --region asia-northeast1 \
  --member "user:someone@example.com" --role roles/run.invoker
```

アプリのログイン機能そのもの（認証・認可の設計）は Cloud Run の管轄外。
アプリ側で別途実装する。

**課題**: 公開設定を外し、認証付きでしか叩けないことを `curl` で確認する。

```bash
curl -H "Authorization: Bearer $(gcloud auth print-identity-token)" "$SERVICE_URL"
```

---

## Step 8. ログ・監視・トラブルシュート（4〜6 時間）

- [ ] 標準出力・標準エラーがそのまま Cloud Logging に流れることを理解する
- [ ] JSON 形式で出力すると構造化ログとして扱える
- [ ] メトリクス（リクエスト数、レイテンシ、インスタンス数、エラー率）の見方
- [ ] 起動失敗時のログの読み方（デプロイ失敗の大半はここで分かる）

```bash
gcloud run services logs read sample-app --region asia-northeast1 --limit 50
gcloud run services logs tail sample-app --region asia-northeast1
```

**課題**: わざと例外を投げるエンドポイントを作り、Cloud Logging でスタックトレースを追う。

---

## Step 9. CI/CD と独自ドメイン（8〜10 時間）

- [ ] GitHub にプッシュしたら自動デプロイする（Cloud Build トリガー or GitHub Actions）
- [ ] サービスアカウントキーを使わない認証（Workload Identity 連携）
- [ ] ステージング用と本番用でサービスを分ける
- [ ] 独自ドメインを割り当てる（ドメインマッピング、または外部ロードバランサ経由）

最初は Cloud Build トリガーが簡単。GitHub リポジトリを連携し、
ブランチへの push をトリガーに `gcloud run deploy` 相当を実行する。

**課題**: main ブランチへの push で自動デプロイされる状態を作り、
コミット → 数分後に本番 URL が変わることを確認する。

---

## Step 10. 後片付けと運用の習慣化（2〜3 時間）

学習で作ったリソースは、放置すると少額でも課金され続ける。

```bash
# 1. Cloud Run サービス
gcloud run services delete sample-app --region asia-northeast1

# 2. Artifact Registry（イメージが溜まる）
gcloud artifacts repositories list --location asia-northeast1
gcloud artifacts repositories delete my-repo --location asia-northeast1

# 3. Cloud Build がソース保管に使ったバケット
gcloud storage buckets list

# 4. Cloud SQL インスタンス（作った場合。最も高額になりやすい）
gcloud sql instances list
```

**一番確実なのはプロジェクトごと削除する方法。**

```bash
gcloud projects delete YOUR_PROJECT_ID
```

- [ ] 課金レポートで、翌日以降の発生額が 0 になっていることを確認する

---

## 料金の考え方

- 課金は主に**リクエスト数**と**リクエスト処理中の CPU / メモリ使用時間**で決まる。
- `min-instances=0` なら、アクセスがない時間帯はほぼ課金されない。
- `min-instances` を 1 以上にした瞬間、常時課金になる。コールドスタート対策と課金はトレードオフ。
- Cloud Run には無料枠があるが、**金額・数量は改定されるので必ず公式の料金ページで確認する**。
- 意外な課金源: Cloud SQL の常時起動、Artifact Registry に溜まった古いイメージ、
  ロードバランサの固定費。

---

## つまずきポイント集

| 症状                     | 原因                                     | 対処                                          |
| ------------------------ | ---------------------------------------- | --------------------------------------------- |
| デプロイは成功するが 503 | `PORT` を listen していない              | `process.env.PORT` を `0.0.0.0` で listen     |
| コンテナが起動しない     | ローカルとアーキテクチャ不一致           | `--platform linux/amd64` でビルド             |
| 403 が返る               | 公開設定になっていない                   | `--allow-unauthenticated` または invoker 権限 |
| 初回だけ極端に遅い       | コールドスタート                         | イメージ軽量化、必要なら `min-instances`      |
| 非同期処理が途中で止まる | レスポンス後に CPU が絞られる            | 処理を Cloud Tasks / ジョブに切り出す         |
| ファイルが消える         | ステートレス前提                         | GCS などの外部ストレージへ保存                |
| 環境変数が反映されない   | 新リビジョンにトラフィックが向いていない | `update-traffic --to-latest`                  |

---

## 成果物（ポートフォリオ化）

Step 9 まで終えたら、この 1 つを作りきると学習が定着する。

- Next.js の Web アプリを Cloud Run にデプロイする
- 画像アップロードは Cloud Storage、データは Cloud SQL または Firestore
- シークレットは Secret Manager
- main ブランチへの push で自動デプロイ
- README に構成図（誰が誰を呼ぶか）と月額試算を書く

---

## 進捗メモ

| Step | 内容                        | 状態 | メモ |
| ---- | --------------------------- | ---- | ---- |
| 0    | なぜ Cloud Run か           |      |      |
| 1    | セットアップ + 予算アラート |      |      |
| 2    | Docker 基礎                 |      |      |
| 3    | 最小アプリのデプロイ        |      |      |
| 4    | 中核概念                    |      |      |
| 5    | 設定とシークレット          |      |      |
| 6    | データの永続化              |      |      |
| 7    | アクセス制御と IAM          |      |      |
| 8    | ログ・監視                  |      |      |
| 9    | CI/CD と独自ドメイン        |      |      |
| 10   | 後片付け                    |      |      |

---

## 参考リンク

- Cloud Run 公式ドキュメント: https://cloud.google.com/run/docs
- Cloud Run クイックスタート: https://cloud.google.com/run/docs/quickstarts
- 料金: https://cloud.google.com/run/pricing
- gcloud CLI インストール: https://cloud.google.com/sdk/docs/install
- Docker 公式チュートリアル: https://docs.docker.com/get-started/
- Secret Manager: https://cloud.google.com/secret-manager/docs
- Cloud SQL への接続: https://cloud.google.com/sql/docs/mysql/connect-run
