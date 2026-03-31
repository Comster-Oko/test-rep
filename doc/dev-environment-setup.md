# 開発環境構築手順

> 対象OS：Windows  
> 構成：Java 21 / Spring Boot 3.x / Docker Compose / PostgreSQL  
> 最終更新：2026年3月

---

## 目次

1. [前提ソフトウェアのインストール](#1-前提ソフトウェアのインストール)
2. [Javaのセットアップ](#2-javaのセットアップ)
3. [IDEのセットアップ（IntelliJ IDEA）](#3-ideのセットアップintelliJ-idea)
4. [Dockerのセットアップ](#4-dockerのセットアップ)
5. [リポジトリのクローンと初期設定](#5-リポジトリのクローンと初期設定)
6. [Docker Composeでミドルウェアを起動する](#6-docker-composeでミドルウェアを起動する)
7. [アプリケーションの起動確認](#7-アプリケーションの起動確認)
8. [よくあるトラブルと対処法](#8-よくあるトラブルと対処法)

---

## 1. 前提ソフトウェアのインストール

以下のソフトウェアをインストールする。インストール済みの場合はスキップしてよい。

| ソフトウェア | 推奨バージョン | 入手先 |
|---|---|---|
| Git | 最新版 | https://git-scm.com/ |
| Docker Desktop for Windows | 最新版 | https://www.docker.com/products/docker-desktop/ |
| IntelliJ IDEA | 最新版（Community可） | https://www.jetbrains.com/idea/ |
| Java 21（Temurin） | 21 LTS | https://adoptium.net/ |

### Wingetでまとめてインストールする場合（Windows 11推奨）

PowerShell（管理者）を開いて実行する。

```powershell
winget install Git.Git
winget install Docker.DockerDesktop
winget install EclipseAdoptium.Temurin.21.JDK
winget install JetBrains.IntelliJIDEA.Community
```

---

## 2. Javaのセットアップ

### 2.1 インストールの確認

コマンドプロンプトまたはPowerShellで以下を実行する。

```powershell
java -version
```

以下のように表示されれば正常。

```
openjdk version "21.x.x" ...
Eclipse Adoptium
```

### 2.2 JAVA_HOME の設定

環境変数が設定されていない場合は手動で設定する。

1. 「スタートメニュー」→「環境変数を編集」を開く
2. 「システム環境変数」に以下を追加する

| 変数名 | 値 |
|---|---|
| `JAVA_HOME` | `C:\Program Files\Eclipse Adoptium\jdk-21.x.x.x-hotspot` |

3. `Path` に `%JAVA_HOME%\bin` を追加する
4. 設定後、PowerShellを再起動して `java -version` で確認する

---

## 3. IDEのセットアップ（IntelliJ IDEA）

### 3.1 推奨プラグイン

IntelliJ IDEA を起動し、`Settings > Plugins` から以下をインストールする。

| プラグイン | 用途 |
|---|---|
| Lombok | アノテーション処理の有効化 |
| Checkstyle-IDEA | コーディング規約チェック |
| .env files support | .envファイルのシンタックスハイライト |
| Docker | Docker統合 |

### 3.2 Lombokの有効化

`Settings > Build, Execution, Deployment > Compiler > Annotation Processors` で  
**「Enable annotation processing」** にチェックを入れる。

### 3.3 コードフォーマッターの設定

1. [Google Java Style Guide の XML設定ファイル](https://github.com/google/styleguide/blob/gh-pages/intellij-java-google-style.xml) をダウンロードする
2. `Settings > Editor > Code Style > Java` → 右上の歯車アイコン → `Import Scheme` でインポートする
3. 保存時に自動フォーマットされるよう `Settings > Tools > Actions on Save` で  
   **「Reformat code」** を有効にする

---

## 4. Dockerのセットアップ

### 4.1 Docker Desktop の起動確認

Docker Desktop を起動し、タスクトレイのアイコンが **Running** になっていることを確認する。

```powershell
docker --version
docker compose version
```

以下のように表示されれば正常。

```
Docker version 27.x.x
Docker Compose version v2.x.x
```

### 4.2 WSL2 バックエンドの確認（Windows 11 推奨設定）

Docker Desktop の `Settings > General` で  
**「Use the WSL 2 based engine」** が有効になっていることを確認する。

> **Note:** WSL2が未インストールの場合は、PowerShell（管理者）で `wsl --install` を実行してPCを再起動する。

---

## 5. リポジトリのクローンと初期設定

### 5.1 リポジトリのクローン

```powershell
git clone https://github.com/your-org/your-project.git
cd your-project
```

### 5.2 環境変数ファイルの作成

リポジトリに含まれる `.env.example` をコピーして `.env` を作成する。

```powershell
Copy-Item .env.example .env
```

`.env` を編集して各自の環境に合わせた値を設定する。

```dotenv
# .env
SPRING_PROFILES_ACTIVE=dev

# データベース
DB_HOST=localhost
DB_PORT=5432
DB_NAME=appdb
DB_USERNAME=appuser
DB_PASSWORD=your_password_here

# JWT
JWT_SECRET=your_jwt_secret_here
JWT_EXPIRATION_MS=1800000
```

> **⚠️ 注意:** `.env` は `.gitignore` に含まれているため、**絶対にコミットしないこと。**

### 5.3 Gradle Wrapper の動作確認

```powershell
./gradlew --version
```

---

## 6. Docker Composeでミドルウェアを起動する

### 6.1 docker-compose.yml の構成

プロジェクトルートの `docker-compose.yml` は以下の構成になっている。

```yaml
version: '3.9'

services:
  db:
    image: postgres:16-alpine
    container_name: app-postgres
    environment:
      POSTGRES_DB: appdb
      POSTGRES_USER: appuser
      POSTGRES_PASSWORD: your_password_here
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./docker/init:/docker-entrypoint-initdb.d  # 初期化SQL
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U appuser -d appdb"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  postgres_data:
```

### 6.2 コンテナの起動

```powershell
# バックグラウンドで起動
docker compose up -d

# 起動状態を確認
docker compose ps
```

以下のように `healthy` または `running` になっていれば正常。

```
NAME            STATUS
app-postgres    running (healthy)
```

### 6.3 PostgreSQL への接続確認

```powershell
docker exec -it app-postgres psql -U appuser -d appdb
```

接続できたら `\q` で終了する。

### 6.4 コンテナの停止・削除

```powershell
# 停止（データは保持）
docker compose stop

# 停止 + コンテナ削除（データは保持）
docker compose down

# データも含めて完全削除（注意）
docker compose down -v
```

---

## 7. アプリケーションの起動確認

### 7.1 ビルドの実行

```powershell
./gradlew build
```

### 7.2 アプリケーションの起動

**IntelliJ IDEA から起動する場合：**

1. `src/main/java/.../Application.java` を開く
2. クラス横の ▶ ボタンをクリックする
3. 起動設定に環境変数を追加する  
   `Run > Edit Configurations > Environment variables` に `.env` の内容を設定する

**コマンドラインから起動する場合：**

```powershell
./gradlew bootRun
```

または `.env` を読み込んで起動する場合：

```powershell
# PowerShellで.envを読み込んでから起動
Get-Content .env | ForEach-Object {
    if ($_ -match '^([^#][^=]*)=(.*)$') {
        [System.Environment]::SetEnvironmentVariable($matches[1], $matches[2], 'Process')
    }
}
./gradlew bootRun
```

### 7.3 起動確認

ブラウザまたはcurlで以下にアクセスし、正常なレスポンスが返ることを確認する。

```powershell
# ヘルスチェック
curl http://localhost:8080/actuator/health

# Swagger UI（ブラウザで開く）
start http://localhost:8080/swagger-ui.html
```

**ヘルスチェックの期待レスポンス：**

```json
{
  "status": "UP",
  "components": {
    "db": { "status": "UP" }
  }
}
```

---

## 8. よくあるトラブルと対処法

### ❶ `docker compose up` でポート5432が競合する

**原因：** ローカルにPostgreSQLがインストールされていてポートを使用している。

**対処：**  
`docker-compose.yml` のポートマッピングを変更する。

```yaml
ports:
  - "5433:5432"  # ホスト側を5433に変更
```

`application-dev.yml` のDB接続ポートも合わせて変更する。

---

### ❷ `gradlew build` で `JAVA_HOME` エラーが出る

**原因：** 環境変数 `JAVA_HOME` が正しく設定されていない。

**対処：**

```powershell
# 現在の設定を確認
echo $env:JAVA_HOME

# PowerShellで一時的に設定（セッション内のみ有効）
$env:JAVA_HOME = "C:\Program Files\Eclipse Adoptium\jdk-21.x.x.x-hotspot"
```

恒久的に設定する場合は「2.2 JAVA_HOME の設定」を参照すること。

---

### ❸ Lombok のアノテーションが認識されない

**原因：** IntelliJ のアノテーションプロセッサが無効になっている。

**対処：**  
「3.2 Lombokの有効化」の手順を再確認し、IntelliJ を再起動する。

---

### ❹ Docker Desktop が起動しない（WSL2エラー）

**原因：** WSL2のインストールが不完全。

**対処：**

```powershell
# PowerShell（管理者）で実行
wsl --update
wsl --set-default-version 2
```

実行後、PCを再起動してDocker Desktopを再度起動する。

---

### ❺ アプリ起動時に `Connection refused` でDBに繋がらない

**原因：** PostgreSQLコンテナが起動していない、またはヘルスチェックが完了していない。

**対処：**

```powershell
# コンテナの状態を確認
docker compose ps

# ログを確認
docker compose logs db

# コンテナが停止していたら再起動
docker compose up -d
```

---

## 付録：開発フロー チートシート

```powershell
# 毎朝の起動手順
docker compose up -d          # ミドルウェア起動
./gradlew bootRun              # アプリ起動

# 終業時の停止手順
# Ctrl+C でアプリを停止
docker compose stop            # ミドルウェア停止

# テストの実行
./gradlew test                 # 全テスト実行
./gradlew test --tests "*.UserServiceTest"  # 特定クラスのみ

# ビルド（テストをスキップ）
./gradlew build -x test

# DBの中身をリセットしたい場合
docker compose down -v         # ボリューム削除
docker compose up -d           # 再起動（初期化SQLが再実行される）
```
