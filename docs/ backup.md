# S3 バックアップ / 復元手順書

## 0. 方針

### Render Postgres

Render の Cron Job で `backup.sh` を実行し、その中で `pg_dump` を使って SQL ダンプを gzip 圧縮して S3 に保存する。Render の公式サンプル `postgres-s3-backups` もこの方式で、`backup.sh` では `pg_dump ... | gzip` を実行して `.sql.gz` を S3 にアップロードしている。

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

## 2. 実務上の運用ルール

* Render Postgres のバックアップは Render Cron Job が `backup.sh` を実行し、その中で `pg_dump` を取得して gzip 圧縮し、S3 に保存する。
* 復元は、新しい Render Postgres を作成し、ローカルに落とした `.sql.gz` を `psql` で流し込む。
* ローカル端末から接続する場合は External Database URL を使う。

---
