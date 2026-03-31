# Docker で PostgreSQL を起動する手順（Windows）

このドキュメントは **Docker を使ってローカルに PostgreSQL を立ち上げる** ための手順です。Windows（PowerShell）想定です。

---

## 前提

- **Docker Desktop** がインストール済みで、起動していること
- ターミナルで `docker` コマンドが使えること

確認:

```powershell
docker --version
docker ps
```

---

## 1) コンテナを起動（`docker run`）

以下は例です。ポート・ユーザー名・パスワード・DB名は必要に応じて変更してください。

| 項目 | 例の値 |
|------|--------|
| コンテナ名 | `dev-pg` |
| PostgreSQL バージョン | **最新**（イメージ `postgres:latest`） |
| ホスト側ポート | `5432` |
| データベース名 | `devdb` |
| ユーザー / パスワード | `postgres` / `postgres` |

```powershell
docker run -d `
  --name dev-pg `
  -e POSTGRES_USER=postgres `
  -e POSTGRES_PASSWORD=postgres `
  -e POSTGRES_DB=devdb `
  -p 5432:5432 `
  -v pg-local-data:/var/lib/postgresql/data `
  postgres:17
```

`postgres:latest` は **Docker Hub で公開されている公式イメージの最新タグ** です。`docker pull` した時点で指すメジャー／マイナー版が変わり得るため、チームや本番では特定バージョン（例: `postgres:17`）に固定する運用も検討してください。

- **`-d`**: バックグラウンド実行  
- **`-p 5432:5432`**: ホストの 5432 をコンテナの 5432 に転送  
- **`-v pg-local-data:...`**: 名前付きボリュームでデータを永続化（コンテナ削除後も残る）

起動確認:

```powershell
docker ps
docker logs dev-pg
```

### 初期化（initialize / `initdb`）は手動で必要か

**通常は不要です。** 公式 `postgres` イメージは、データ領域（`/var/lib/postgresql/data`）が **空のときだけ** 初回起動で **`initdb` 相当の処理を自動実行** し、`POSTGRES_USER`・`POSTGRES_PASSWORD`・`POSTGRES_DB` に従ってユーザーとデータベースを作ります。

- **既に初期化済みのボリューム** をマウントしている場合は、以降の起動ではその処理はスキップされ、既存データがそのまま使われます。
- **初回だけ追加の SQL やシェル** を実行したい場合（スキーマ作成・マスタ投入など）は、任意で `/docker-entrypoint-initdb.d/` に `.sql` や `.sh` を置く方法が公式イメージでサポートされています（空のデータ領域での初回起動時のみ実行）。

---

## 2) 接続情報（アプリ・クライアント用）

**§1 の `docker run` で立ち上げたコンテナと同じ値**です（§1 の `POSTGRES_*` や `-p` を変えた場合は、ここも同じように読み替えてください）。

### JDBC URL（Spring Boot など）

```
jdbc:postgresql://localhost:5432/devdb
```

### 接続パラメータの例

| キー | 値（§1 の例どおりの場合） |
|------|---------------------------|
| ホスト | `localhost` |
| ポート | `5432` |
| データベース | `devdb` |
| ユーザー | `postgres` |
| パスワード | `postgres` |

---

## 3) Windows ホストから A5:SQL Mk-2（a5m2）で接続

[A5:SQL Mk-2](https://a5m2.mmatsubara.com/)（通称 **a5m2**）は Windows 向けの無料データベースクライアントです。PostgreSQL に接続してクエリや ER 図などが利用できます。

### 前提

- **Docker で PostgreSQL コンテナが起動している**こと（`docker ps` で `dev-pg` などが表示される）。
- **`-p 5432:5432` のようにポートがホストに公開されている**こと（§1 の例どおりなら、ホストの `localhost:5432` がコンテナに届きます）。

### 接続設定（§1 の例どおりの場合）

A5M2 で **サーバ登録**（または **データベースへの接続**）を開き、種類を **PostgreSQL** にして、次のように入力します。

| 項目 | 設定値 |
|------|--------|
| ホスト名 | `localhost`（または `127.0.0.1`） |
| ポート | `5432`（§1 で `-p` を変えた場合はそのホスト側ポート） |
| データベース名 | `devdb` |
| ユーザー名 | `postgres` |
| パスワード | `postgres` |
| SSL | **利用しない**（ローカル Docker では通常オフのまま） |

接続テストや OK で登録し、データベース一覧から `devdb` に接続します。

### つながらないとき

- コンテナが止まっていないか（`docker start dev-pg` など）。
- ポートが他アプリと競合していないか（§6 の「ポート 5432 が既に使用中」）。
- 接続情報が §2 の表と一致しているか（ユーザー名・DB 名の typo、別ポートにした場合のポート番号）。

---

## 4) 動作確認（コンテナ内で `psql`）

```powershell
docker exec -it dev-pg psql -U postgres -d devdb -c "SELECT version();"
```

---

## 5) 停止・削除

### 停止

```powershell
docker stop dev-pg
```

### 再起動

```powershell
docker start dev-pg
```

### コンテナを削除（データはボリューム `pg-local-data` に残る）

```powershell
docker rm -f dev-pg
```

### 名前付きボリュームも削除する（**データが消えます**）

```powershell
docker volume rm pg-local-data
```

---

## 6) よくあるつまずき

- **`Database is uninitialized and superuser password is not specified`**
  - **原因**: データ領域がまだ空の **初回初期化** のとき、公式イメージはスーパーユーザ用パスワードの指定を必須にしています。`POSTGRES_PASSWORD` が未設定・空文字だとこのエラーになります。
  - **対応（推奨）**: `docker run` または Compose の `environment` に **`POSTGRES_PASSWORD` を空でない値で指定** する（本ドキュメントの例: `-e POSTGRES_PASSWORD=postgres`）。
  - **Compose / `.env` の場合**: `environment` に書き忘れ、または `.env` の変数名が違う・値が空になっていないか確認する。
  - **メッセージに出る `POSTGRES_HOST_AUTH_METHOD=trust`**: パスワードなし接続を許可する設定で、**ローカル検証以外では非推奨**（セキュリティ上の理由）。通常はパスワードを設定する方で解決する。

- **ポート 5432 が既に使用中**
  - ホスト側の別ポートにマッピングする（例: `-p 15432:5432`）。その場合 JDBC や A5M2 のポートも `15432` に合わせる。
- **`psql` がホストにない**
  - 上記のとおり `docker exec` でコンテナ内から実行する。
- **パスワードを変えたい**
  - 既存ボリュームがあると初期化済みのため、環境変数だけ変えても反映されないことがあります。開発用ならボリューム削除後に再作成するか、新しいボリューム名で別コンテナを立てる。

---

## 7) （任意）`docker compose` での例

`docker-compose.yml` の一例:

```yaml
services:
  db:
    image: postgres:latest
    container_name: dev-pg
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: devdb
    ports:
      - "5432:5432"
    volumes:
      - pg-local-data:/var/lib/postgresql/data

volumes:
  pg-local-data:
```

起動・停止:

```powershell
docker compose up -d
docker compose down
```

`down` ではデフォルトで名前付きボリュームは削除されません。データごと消す場合は `docker compose down -v`（**データ消失に注意**）。
