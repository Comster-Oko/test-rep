# Spring Boot API 開発標準

> 対象チーム：小規模（〜5人）  
> 対象フレームワーク：Java / Spring Boot  
> 最終更新：2026年3月

---

## 目次

1. [プロジェクト構成・アーキテクチャ](#1-プロジェクト構成アーキテクチャ)
2. [コーディング規約](#2-コーディング規約)
3. [API設計規約](#3-api設計規約)
4. [エラーハンドリング・例外設計](#4-エラーハンドリング例外設計)
5. [セキュリティ](#5-セキュリティ)
6. [テスト規約](#6-テスト規約)
7. [ログ・監視](#7-ログ監視)
8. [CI/CD・デプロイ](#8-cicdデプロイ)

---

## 1. プロジェクト構成・アーキテクチャ

### 1.1 レイヤードアーキテクチャ

以下の4層構造を基本とする。

| レイヤー | パッケージ | 責務 |
|---|---|---|
| Presentation | `controller` | リクエスト受付・レスポンス整形 |
| Application | `service` | ビジネスロジック |
| Domain | `domain` | エンティティ・ドメインロジック |
| Infrastructure | `repository`, `client` | DB・外部API連携 |

### 1.2 ディレクトリ構成

```
src/
├── main/
│   ├── java/com/example/app/
│   │   ├── config/          # 設定クラス（Bean定義、Security等）
│   │   ├── controller/      # REST コントローラー
│   │   ├── service/         # サービス層（インターフェース + 実装）
│   │   ├── domain/
│   │   │   ├── model/       # エンティティ、値オブジェクト
│   │   │   └── repository/  # リポジトリインターフェース
│   │   ├── infrastructure/
│   │   │   ├── repository/  # リポジトリ実装（JPA等）
│   │   │   └── client/      # 外部API クライアント
│   │   ├── dto/
│   │   │   ├── request/     # リクエストDTO
│   │   │   └── response/    # レスポンスDTO
│   │   ├── exception/       # カスタム例外クラス
│   │   └── util/            # 共通ユーティリティ
│   └── resources/
│       ├── application.yml
│       ├── application-dev.yml
│       └── application-prod.yml
└── test/
    └── java/com/example/app/
        ├── controller/
        ├── service/
        └── repository/
```

### 1.3 依存関係のルール

- 依存方向は **上位レイヤー → 下位レイヤー** のみ許可する
- Controller は Service のみに依存し、Repository に直接依存しない
- Domain層は他レイヤーに依存しない

### 1.4 推奨ライブラリ・バージョン

| 用途 | ライブラリ |
|---|---|
| フレームワーク | Spring Boot 3.x |
| ORM | Spring Data JPA + Hibernate |
| バリデーション | Spring Validation (jakarta.validation) |
| マッピング | MapStruct |
| テスト | JUnit 5, Mockito, AssertJ |
| APIドキュメント | springdoc-openapi (Swagger UI) |
| ビルドツール | Gradle (Kotlin DSL 推奨) |

---

## 2. コーディング規約

### 2.1 命名規則

| 対象 | 規則 | 例 |
|---|---|---|
| クラス | UpperCamelCase | `UserService`, `OrderController` |
| メソッド・変数 | lowerCamelCase | `findUserById`, `orderList` |
| 定数 | UPPER_SNAKE_CASE | `MAX_RETRY_COUNT` |
| パッケージ | すべて小文字 | `com.example.app.service` |
| テストクラス | 対象クラス名 + `Test` | `UserServiceTest` |

### 2.2 クラス設計

```java
// ✅ Good: コンストラクタインジェクション（フィールドインジェクション禁止）
@Service
@RequiredArgsConstructor
public class UserService {
    private final UserRepository userRepository;
    // ...
}

// ❌ Bad: フィールドインジェクション
@Service
public class UserService {
    @Autowired
    private UserRepository userRepository;
}
```

### 2.3 DTO の扱い

- Controller層のリクエスト・レスポンスには必ずDTOを使用し、エンティティを直接公開しない
- DTOにはRecordクラス（Java 16+）またはLombokの `@Value` を使用する

```java
// リクエストDTO
public record CreateUserRequest(
    @NotBlank String name,
    @Email String email
) {}

// レスポンスDTO
public record UserResponse(
    Long id,
    String name,
    String email
) {}
```

### 2.4 コードフォーマット

- インデント：スペース4つ
- 1行の最大文字数：120文字
- フォーマッターは **Google Java Style Guide** をベースに設定する
- IDEの自動フォーマットを必ず有効化する（`Checkstyle` / `Spotless` プラグイン推奨）

### 2.5 コメント規約

- Javadocは公開APIおよびサービス層の主要メソッドに記述する
- 「何をしているか」ではなく「なぜそうしているか」をコメントに書く
- TODO/FIXMEコメントにはチケット番号を付記する

```java
// TODO: TICKET-123 パフォーマンス改善のためキャッシュを導入する
```

---

## 3. API設計規約

### 3.1 URL設計

- リソース名は**複数形の名詞**を使用する
- 階層は深くなりすぎないようにする（最大3階層まで）
- バージョンはパスに含める（`/api/v1/`）

```
GET    /api/v1/users          # ユーザー一覧取得
GET    /api/v1/users/{id}     # ユーザー取得
POST   /api/v1/users          # ユーザー作成
PUT    /api/v1/users/{id}     # ユーザー全更新
PATCH  /api/v1/users/{id}     # ユーザー部分更新
DELETE /api/v1/users/{id}     # ユーザー削除
```

### 3.2 HTTPメソッドとステータスコード

| 操作 | メソッド | 成功時ステータス |
|---|---|---|
| 取得 | GET | 200 OK |
| 作成 | POST | 201 Created |
| 全更新 | PUT | 200 OK |
| 部分更新 | PATCH | 200 OK |
| 削除 | DELETE | 204 No Content |

### 3.3 リクエスト・レスポンスの形式

- Content-Type は `application/json` を使用する
- レスポンスのJSONキーは **lowerCamelCase** とする
- 日時は **ISO 8601形式**（`2026-03-30T12:00:00Z`）で統一する

**レスポンスの基本構造（ラッパー不要。リソースを直接返す）：**

```json
// 単一リソース
{
  "id": 1,
  "name": "山田太郎",
  "email": "yamada@example.com",
  "createdAt": "2026-03-30T12:00:00Z"
}

// リスト（ページネーションあり）
{
  "content": [...],
  "totalElements": 100,
  "totalPages": 10,
  "page": 0,
  "size": 10
}
```

### 3.4 バリデーション

- 入力値バリデーションは `jakarta.validation` アノテーションを使用し、DTOに定義する
- コントローラーで `@Valid` を付与して有効化する

```java
@PostMapping("/users")
public ResponseEntity<UserResponse> createUser(
    @Valid @RequestBody CreateUserRequest request) {
    // ...
}
```

### 3.5 APIドキュメント

- springdoc-openapi を使用してSwagger UIを自動生成する
- 各エンドポイントに `@Operation`、`@ApiResponse` でドキュメントを記述する
- 開発環境・ステージング環境のみSwagger UIを公開し、本番では無効化する

---

## 4. エラーハンドリング・例外設計

### 4.1 例外クラスの設計

独自の業務例外クラスを定義し、エラーの種類を分類する。

```java
// 基底クラス
public abstract class AppException extends RuntimeException {
    private final ErrorCode errorCode;
    // ...
}

// ビジネスロジック例外（400系）
public class BusinessException extends AppException { ... }

// リソース未発見（404）
public class ResourceNotFoundException extends AppException { ... }

// 認可エラー（403）
public class ForbiddenException extends AppException { ... }
```

### 4.2 エラーコード定義

```java
public enum ErrorCode {
    USER_NOT_FOUND("E001", "ユーザーが見つかりません"),
    EMAIL_ALREADY_EXISTS("E002", "このメールアドレスはすでに使用されています"),
    INVALID_REQUEST("E003", "リクエストが不正です");

    private final String code;
    private final String message;
}
```

### 4.3 グローバル例外ハンドラー

`@RestControllerAdvice` で一元管理する。

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(ResourceNotFoundException e) {
        return ResponseEntity.status(HttpStatus.NOT_FOUND)
            .body(ErrorResponse.of(e.getErrorCode()));
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidation(MethodArgumentNotValidException e) {
        // バリデーションエラーの詳細をレスポンスに含める
        ...
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleUnexpected(Exception e) {
        log.error("予期せぬエラーが発生しました", e);
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
            .body(ErrorResponse.ofUnexpected());
    }
}
```

### 4.4 エラーレスポンス形式

```json
{
  "code": "E001",
  "message": "ユーザーが見つかりません",
  "timestamp": "2026-03-30T12:00:00Z",
  "path": "/api/v1/users/999"
}
```

**バリデーションエラーの場合：**

```json
{
  "code": "E003",
  "message": "リクエストが不正です",
  "timestamp": "2026-03-30T12:00:00Z",
  "path": "/api/v1/users",
  "errors": [
    { "field": "email", "message": "メールアドレスの形式が不正です" },
    { "field": "name", "message": "名前は必須です" }
  ]
}
```

---

## 5. セキュリティ

### 5.1 認証・認可

- 認証には **JWT（JSON Web Token）** を基本とする
- Spring Security を使用してエンドポイントの認可を制御する
- JWTの有効期限：アクセストークン 30分、リフレッシュトークン 7日

```java
@Configuration
@EnableMethodSecurity
public class SecurityConfig {
    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        return http
            .csrf(AbstractHttpConfigurer::disable)
            .sessionManagement(s -> s.sessionCreationPolicy(STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/v1/auth/**").permitAll()
                .anyRequest().authenticated()
            )
            .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class)
            .build();
    }
}
```

### 5.2 入力値の検証

- SQLインジェクション対策：JPA / Prepared Statement を使用し、生クエリを避ける
- XSS対策：レスポンスHTMLをエスケープする（APIの場合はJSON返却のためリスクは低いが意識すること）
- ユーザー入力をそのままログに出力しない（ログインジェクション対策）

### 5.3 機密情報の管理

- パスワードや秘密鍵を **ソースコードにハードコードしない**
- 環境変数または Vault / AWS Secrets Manager を使用する
- `application.yml` に機密情報を直接記載しない

```yaml
# ✅ Good: 環境変数から取得
spring:
  datasource:
    password: ${DB_PASSWORD}

# ❌ Bad: 直接記載
spring:
  datasource:
    password: mysecretpassword
```

### 5.4 依存ライブラリの脆弱性管理

- `./gradlew dependencyCheckAnalyze`（OWASP Dependency Check）を定期実行する
- 既知の脆弱性があるバージョンは速やかにアップデートする

### 5.5 HTTPセキュリティヘッダー

以下のヘッダーをデフォルトで有効化する。

| ヘッダー | 設定値 |
|---|---|
| `X-Content-Type-Options` | `nosniff` |
| `X-Frame-Options` | `DENY` |
| `Strict-Transport-Security` | `max-age=31536000` |

---

## 6. テスト規約

### 6.1 テスト種別と目標カバレッジ

| 種別 | 対象 | 目標カバレッジ |
|---|---|---|
| 単体テスト | Service, Domain | 80%以上 |
| 結合テスト | Controller（@WebMvcTest） | 主要エンドポイント全件 |
| E2Eテスト | 主要フロー | クリティカルパスのみ |

### 6.2 テストの書き方

**Given-When-Then パターン**を必ず守る。

```java
@Test
void ユーザーが存在する場合にIDで取得できること() {
    // Given（前提条件）
    var userId = 1L;
    var expected = new User(userId, "山田太郎", "yamada@example.com");
    when(userRepository.findById(userId)).thenReturn(Optional.of(expected));

    // When（操作）
    var result = userService.findById(userId);

    // Then（検証）
    assertThat(result.name()).isEqualTo("山田太郎");
}
```

### 6.3 コントローラーのテスト

`@WebMvcTest` + `MockMvc` を使用する。

```java
@WebMvcTest(UserController.class)
class UserControllerTest {

    @Autowired MockMvc mockMvc;
    @MockBean UserService userService;

    @Test
    void ユーザー取得APIが200を返すこと() throws Exception {
        when(userService.findById(1L)).thenReturn(new UserResponse(1L, "山田太郎", "yamada@example.com"));

        mockMvc.perform(get("/api/v1/users/1"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.name").value("山田太郎"));
    }
}
```

### 6.4 リポジトリのテスト

`@DataJpaTest` を使用してDBレイヤーを個別にテストする。

```java
@DataJpaTest
class UserRepositoryTest {
    @Autowired UserRepository userRepository;

    @Test
    void メールアドレスでユーザーを検索できること() {
        // ...
    }
}
```

### 6.5 テストの禁止事項

- 本番DBや外部APIに依存したテストを書かない（モック・インメモリDBを使用する）
- テスト間で状態を共有しない（各テストは独立して実行できること）
- `Thread.sleep()` による待機を使わない

---

## 7. ログ・監視

### 7.1 ログレベルの使い分け

| レベル | 用途 |
|---|---|
| `ERROR` | システムの異常・予期せぬ例外（即時対応が必要） |
| `WARN` | 業務例外・リトライ発生など（対応が望ましい） |
| `INFO` | リクエスト受付・主要処理の開始・終了 |
| `DEBUG` | 開発時のデバッグ情報（本番では出力しない） |

### 7.2 ログ出力ルール

```java
// ✅ Good: ロガーはフィールドで定義（SLF4J + Lombok）
@Slf4j
@Service
public class UserService {
    public UserResponse findById(Long id) {
        log.info("ユーザー取得開始 userId={}", id);  // 構造化パラメータ
        // ...
    }
}

// ❌ Bad: System.out.println の使用禁止
System.out.println("ユーザー取得: " + id);
```

### 7.3 ログに含めること / 含めないこと

**含めること：**
- リクエストID（トレーシング用）
- 処理時間
- エラーコード・スタックトレース（ERRORレベル）

**含めないこと：**
- パスワード、クレジットカード番号
- 個人情報（マスキング処理を行うこと）
- JWTトークン全体

### 7.4 リクエストログ

`HandlerInterceptor` または `Filter` でリクエスト・レスポンスのログを自動出力する。

```
[INFO] method=GET path=/api/v1/users/1 status=200 duration=45ms requestId=abc-123
```

### 7.5 ログフォーマット

**本番環境：** JSON形式（ログ収集システムへの取り込みを容易にする）  
**開発環境：** テキスト形式（可読性優先）

```yaml
# logback-spring.xml で環境ごとに切り替え
logging:
  pattern:
    console: "%d{yyyy-MM-dd HH:mm:ss} [%thread] %-5level %logger{36} - %msg%n"
```

### 7.6 ヘルスチェック・メトリクス

- Spring Boot Actuator を使用してヘルスエンドポイントを公開する
- `/actuator/health` は外部に公開し、`/actuator/metrics` は内部ネットワークのみに制限する

---

## 8. CI/CD・デプロイ

### 8.1 ブランチ戦略

**GitHub Flow** を採用する（小規模チーム向け）。

```
main            # 本番相当。直接pushは禁止
└── feature/*   # 機能開発ブランチ
└── fix/*        # バグ修正ブランチ
└── hotfix/*     # 緊急修正ブランチ
```

- `main` への統合は必ずPull Request経由とする
- PRのマージには最低1名のレビュー承認を必須とする
- CIが全て通過しないとマージできないよう設定する

### 8.2 CI パイプライン（GitHub Actions 例）

```yaml
name: CI

on:
  pull_request:
    branches: [main]

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
      - name: Build
        run: ./gradlew build
      - name: Test
        run: ./gradlew test
      - name: Test Coverage Check
        run: ./gradlew jacocoTestCoverageVerification
      - name: Code Style Check
        run: ./gradlew spotlessCheck
      - name: Security Check
        run: ./gradlew dependencyCheckAnalyze
```

### 8.3 環境構成

| 環境 | 用途 | デプロイトリガー |
|---|---|---|
| ローカル | 開発・デバッグ | 手動 |
| 開発（dev） | 結合確認 | `main` マージ時 自動 |
| ステージング（stg） | 本番前動作確認 | 手動（リリース前） |
| 本番（prod） | 本番稼働 | 手動承認後 |

### 8.4 設定ファイルの管理

- 環境ごとの設定は `application-{profile}.yml` で管理する
- 機密情報は環境変数で渡す（ソースに含めない）

```yaml
# application.yml（共通設定）
spring:
  profiles:
    active: ${SPRING_PROFILES_ACTIVE:dev}

# application-prod.yml（本番設定）
logging:
  level:
    root: WARN
```

### 8.5 Dockerイメージのビルド

```dockerfile
# マルチステージビルドで軽量なイメージを作成
FROM eclipse-temurin:21-jdk-alpine AS builder
WORKDIR /app
COPY gradlew build.gradle.kts settings.gradle.kts ./
COPY gradle ./gradle
RUN ./gradlew dependencies --no-daemon
COPY src ./src
RUN ./gradlew bootJar --no-daemon

FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
COPY --from=builder /app/build/libs/*.jar app.jar
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### 8.6 ロールバック手順

1. 問題のデプロイを確認したら即座にチームに共有する
2. 直前の安定バージョンのイメージを再デプロイする
3. 原因調査はロールバック後に行う（ダウンタイム最小化を優先）
4. 事後にインシデントレポートを作成し、再発防止策を検討する

---

## 付録：チェックリスト

### コードレビューチェックリスト

- [ ] コーディング規約に準拠しているか
- [ ] ビジネスロジックがService層に書かれているか
- [ ] エンティティがAPIレスポンスに直接露出していないか
- [ ] 適切な例外クラスを使用しているか
- [ ] 機密情報がコードやログに含まれていないか
- [ ] 単体テストが追加・更新されているか
- [ ] Javadoc / コメントが適切に記述されているか

### リリース前チェックリスト

- [ ] 全テストが通過していること
- [ ] ステージング環境での動作確認が完了していること
- [ ] 本番環境の設定ファイル・環境変数を確認していること
- [ ] ロールバック手順を確認していること
- [ ] 監視・アラートが正常に機能していること
