# Cafemisty Misskey カスタムビルド手順

この作業ツリーは、公式 Misskey の `2026.5.4` を基準にしています。

このリポジトリは、運用用リポジトリ `~/misskey` とは分けて管理します。
運用用リポジトリ側では、ビルド済み Docker イメージのタグを
`MISSKEY_IMAGE` で参照するだけにします。

## 現在の基準

```text
公式タグ: 2026.5.4
作業ブランチ: custom/cafemisty-2026.5.4
```

## リモート設定

`upstream` は公式 Misskey リポジトリを指します。誤 push を避けるため、
`upstream` への push は無効化しています。

GitHub 上に fork を作成した後、fork を `origin` として追加します。

```bash
cd ~/misskey-src
git remote add origin git@github.com:YOUR_USER/misskey.git
git push -u origin custom/cafemisty-2026.5.4
```

## ローカルイメージのビルド

本番やテストで使うタグは固定します。`latest` は使わないでください。

```bash
cd ~/misskey-src
sudo docker build -t cafemisty/misskey:2026.5.4-custom.1 .
```

## 運用 Compose への反映

本番コンテナを変更する前に、必ず PostgreSQL バックアップを作成します。

```bash
cd ~/misskey
scripts/backup-postgres.sh
```

次に、`~/misskey/.env` の `MISSKEY_IMAGE` を変更します。

```env
MISSKEY_IMAGE=cafemisty/misskey:2026.5.4-custom.1
```

反映します。

```bash
cd ~/misskey
sudo docker compose up -d
sudo docker compose ps
sudo docker compose logs --tail=100 misskey-web
curl -I http://127.0.0.1:${MISSKEY_PORT:-3000}/
```

## ロールバック

`~/misskey/.env` の `MISSKEY_IMAGE` を変更前の公式イメージへ戻します。

```env
MISSKEY_IMAGE=misskey/misskey:2026.5.4
```

反映します。

```bash
cd ~/misskey
sudo docker compose up -d
sudo docker compose ps
```

## 別ローカルサーバーでのテスト手順

本番サーバーとは別のローカルサーバーでテストできます。基本方針は、`misskey-src`
でカスタム Docker イメージを作り、テストサーバー側では本番とは独立した
Compose 環境、PostgreSQL volume、Redis volume、MinIO bucket を使うことです。

本番の PostgreSQL volume、`.env`、MinIO bucket はテストに流用しないでください。

### 1. テスト用イメージを作る

`~/misskey-src` があるサーバーで実行します。

```bash
cd ~/misskey-src
sudo docker build -t cafemisty/misskey:2026.5.4-test.1 .
sudo docker save cafemisty/misskey:2026.5.4-test.1 | gzip > misskey-test-image.tar.gz
```

### 2. テストサーバーへ転送する

```bash
scp misskey-test-image.tar.gz USER@TEST_SERVER:~/
```

テストサーバー側で読み込みます。

```bash
gunzip -c ~/misskey-test-image.tar.gz | sudo docker load
```

### 3. テスト用 Compose 環境を作る

テストサーバー側に、運用リポジトリ相当のディレクトリを用意します。
既存の `~/misskey` リポジトリをコピーする場合も、`.env` はテスト用に作り直します。

```bash
mkdir -p ~/misskey-test
cd ~/misskey-test
```

`.env` は本番と分けます。例:

```env
MISSKEY_IMAGE=cafemisty/misskey:2026.5.4-test.1
MISSKEY_URL=http://192.168.10.100:3000
MISSKEY_BIND_ADDRESS=192.168.10.100
MISSKEY_PORT=3000

POSTGRES_IMAGE=postgres:16-bookworm
POSTGRES_DB=misskey_test
POSTGRES_USER=misskey
POSTGRES_PASSWORD=CHANGE_ME_TEST_ONLY

REDIS_IMAGE=redis:7-bookworm
```

MinIO を使う場合は、本番 bucket ではなくテスト用 bucket を使います。

```env
MISSKEY_S3_BUCKET=misskey-drive-test
```

### 4. 初期化と起動

テスト用 PostgreSQL volume が空であることを確認してから実行します。

```bash
cd ~/misskey-test
sudo docker compose config
sudo docker compose run --rm misskey-web pnpm run init
sudo docker compose up -d
sudo docker compose ps
curl -I http://192.168.10.100:3000/
```

`192.168.10.100` はテストサーバーの実 IP に置き換えてください。

### 5. 停止と片付け

停止だけなら volume は残します。

```bash
cd ~/misskey-test
sudo docker compose down
```

テスト用データを削除する場合でも、本番と取り違えないように `sudo docker volume ls`
で対象 volume 名を確認してから行ってください。`down -v` は PostgreSQL データを削除
するため、テスト環境であることが明確な場合だけ使います。

## 同じ PC 上でテストする場合

同じ Docker ホスト上でもテストできます。すでに `~/misskey-test` に、
同一 PC 用の独立した Compose 一式を作成済みです。

このテスト環境は、本番側と次の名前を分けています。

```text
公開URL: http://127.0.0.1:3100
Compose project: misskey-test
Web container: misskey-test-web
Worker container: misskey-test-worker
PostgreSQL container: misskey-test-postgres
Redis container: misskey-test-redis
Network: misskey-test-internal
PostgreSQL volume: misskey-test-postgres-data
Redis volume: misskey-test-redis-data
Files volume: misskey-test-files
```

### 1. テスト用イメージを作る

`~/misskey-test/.env` は `cafemisty/misskey:2026.5.4-test.1` を使う設定です。
先にこのイメージを作成します。

```bash
cd ~/misskey-src
sudo docker build -t cafemisty/misskey:2026.5.4-test.1 .
```

### 2. 初期化と起動

初回だけ DB 初期化を行います。

```bash
cd ~/misskey-test
sudo docker compose config
sudo docker compose run --rm misskey-web pnpm run init
sudo docker compose up -d
sudo docker compose ps
curl -I http://127.0.0.1:3100/
```

### 3. 停止と削除

停止だけなら volume は残します。

```bash
cd ~/misskey-test
sudo docker compose down
```

テスト用データを消す場合は、対象が `misskey-test-*` であることを確認してから行います。

```bash
sudo docker volume ls | grep misskey-test
```

削除してよいことが明確な場合のみ:

```bash
cd ~/misskey-test
sudo docker compose down -v
```

## 運用ルール

- カスタムイメージの運用手順が固まるまで、DB migration を伴う変更は避けます。
- カスタムイメージのタグは固定し、用途が分かる名前にします。
- 最初は UI だけの変更をテストし、backend の挙動変更は後回しにします。
- secrets、`.config/default.yml`、DB dump、media file は commit しません。
- 改変版 Misskey を公開運用する場合、上流ライセンスに従ってソース公開が必要になる可能性があります。
  実運用する改変版の fork は参照可能な状態にしておきます。
