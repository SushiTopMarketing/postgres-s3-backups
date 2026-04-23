# S3 バックアップ / 復元手順書

## 0. 方針

### Render Postgres

Render の Cron Job で `backup.sh` を実行し、その中で `pg_dump` を使って SQL ダンプを gzip 圧縮して S3 に保存する。Render の公式サンプル `postgres-s3-backups` もこの方式で、`backup.sh` では `pg_dump ... | gzip` を実行して `.sql.gz` を S3 にアップロードしている。

### PlanetScale / Vitess

Render の Cron Job で `backup.sh` を実行し、その中で `pscale database dump --output-format=csv` を使って dump ディレクトリを作成し、tar.gz 化して S3 に保存する。

---

## 1. Render Postgres → S3 → Render Postgres

### 1-1. バックアップ概要

Render Cron Job が `backup.sh` を実行し、その中で Render Postgres に接続して `pg_dump` を取得し、gzip 圧縮した `.sql.gz` を S3 に保存する。接続先の `DATABASE_URL` には **PgBouncer ではなく Postgres 本体の URL** を使う。

### 1-2. 復元手順

#### 1. 復元先の Render Postgres を新規作成

ローカル端末から接続する場合は、復元先 DB の **External Database URL** を使う。

#### 2. S3 からバックアップをローカルへダウンロード

対象ファイルは `.sql.gz`。

#### 3. ローカルから `psql` で復元

```bash id="2zjlwm"
export DATABASE_URL='Render Postgres の External Database URL'
gunzip -c backup.sql.gz | psql -X --set ON_ERROR_STOP=on "$DATABASE_URL"
```

`psql` は PostgreSQL の標準 CLI クライアント。SQL ダンプは `psql` に流し込んで復元する。

#### 4. 確認

テーブル数、件数、主要データを確認する。

#### 5. 切り替え

確認後、アプリの `DATABASE_URL` を新 DB に切り替える。

---

## 2. PlanetScale / Vitess → CSV dump → S3 → Render Postgres

### 2-1. バックアップ概要

Render Cron Job が `backup.sh` を実行し、その中で PlanetScale に接続して `pscale database dump` を **CSV 形式**で実行し、出力した dump ディレクトリを tar.gz 化して S3 に保存する。

以下は PlanetScale / Vitess 向けに `backup.sh` へ組み込む想定の処理例。

```bash id="tghbi0"
mkdir -p backup

pscale database dump "$PSCALE_DB_NAME" main \
  --org "$PSCALE_ORG" \
  --output backup \
  --output-format=csv

tar -C backup -czf "backup-$(date +%Y-%m-%d-%H-%M-%S).tar.gz" .
aws s3 cp "backup-$(date +%Y-%m-%d-%H-%M-%S).tar.gz" \
  "s3://$S3_BUCKET_NAME/$(date +%Y/%m/%d/)"
```

### 2-2. 復元手順

#### 1. 復元先の Render Postgres を新規作成

ローカル端末から接続するため、**External Database URL** を使う。

#### 2. Prisma で schema を作成

Prisma migration を管理している前提で、先に schema を適用する。

```bash id="g1ehj9"
export DATABASE_URL='Render Postgres の External Database URL'
npx prisma migrate deploy
```

`prisma migrate deploy` は、デプロイ先 DB に pending migration を適用するための Prisma のコマンド。

#### 3. S3 から tar.gz をローカルへダウンロードして展開

```bash id="gmpopx"
mkdir -p restore_work
tar -xzf backup-2026-04-22-03-00-00.tar.gz -C restore_work
```

#### 4. `\copy` で CSV を投入

ローカルに展開した CSV を Render Postgres に投入する。`psql` の `\copy` はローカルファイルを使って取り込める。

```bash id="i1kj95"
psql "$DATABASE_URL" -c "\copy public.users (id, email, created_at) FROM '/absolute/path/restore_work/users.csv' WITH (FORMAT csv, HEADER true)"
```

必要に応じて `NULL` などを指定する。

```bash id="i7iwnw"
psql "$DATABASE_URL" -c "\copy public.users (id, email, created_at) FROM '/absolute/path/restore_work/users.csv' WITH (FORMAT csv, HEADER true, NULL '', QUOTE '\"', ESCAPE '\"')"
```

PostgreSQL の `COPY` / `\copy` は `FORMAT csv`、`HEADER`、`NULL`、`QUOTE`、`ESCAPE` を指定できる。

#### 5. 複数テーブルを投入

```bash id="takw06"
for f in /absolute/path/restore_work/*.csv; do
  table=$(basename "$f" .csv)
  psql "$DATABASE_URL" -c "\copy public.\"$table\" FROM '$f' WITH (FORMAT csv, HEADER true)"
done
```

列順が一致している前提。必要なら列リストを明示する。

#### 6. sequence を補正

`id` を明示投入したテーブルは sequence を補正する。

```sql id="pw2l0x"
SELECT setval(
  pg_get_serial_sequence('public.users', 'id'),
  COALESCE((SELECT MAX(id) FROM public.users), 1)
);
```

`pg_get_serial_sequence` で対象 sequence 名を取得し、`setval` で現在値を調整する。

#### 7. 確認

件数、主キー、時刻列、NULL 値などを確認する。

---

## 3. 実務上の運用ルール

### Render Postgres → Render Postgres

* Render Postgres のバックアップは Render Cron Job が `backup.sh` を実行し、その中で `pg_dump` を取得して gzip 圧縮し、S3 に保存する。
* 復元は、新しい Render Postgres を作成し、ローカルに落とした `.sql.gz` を `psql` で流し込む。
* ローカル端末から接続する場合は External Database URL を使う。

### PlanetScale / Vitess → Render Postgres

* PlanetScale / Vitess のデータは Render Cron Job が `backup.sh` を実行し、その中で `pscale database dump --output-format=csv` を実行して CSV 出力した dump を tar.gz 化し、S3 に保存する。
* 復元先の Render Postgres には、先に Prisma の `migrate deploy` で schema を作成する。
* その後、ローカルに展開した CSV を `psql` の `\copy` で投入する。
* `id` を明示投入したテーブルは、`setval` で sequence を補正する。

---
