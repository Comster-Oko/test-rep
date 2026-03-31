# Spring Boot 開発標準（共通）

このドキュメントは、Spring Boot（主に 3.x / Java 17+）で一般的に適用される開発標準です。  
**「最低限守るべき内容（必須）」**と、チーム状況により採用する**「推奨」**を分けています。

---

## 最低限守るべき（必須）

### 0) 守れない場合の扱い
- 本標準の**必須**に例外が必要な場合は、PR（または変更管理）で**理由・影響・代替策**を明記すること（口頭合意のみは禁止）。

### 1) 変更管理（Git / PR）
- 変更は**必ず PR 経由**（直コミット・直マージ禁止）。
- PR には以下を必ず含める。
  - 変更概要（何を、なぜ）
  - 影響範囲（API/DB/設定/互換性）
  - テスト観点（実行したテスト、未実施なら理由）

### 2) コーディング規約（可読性・一貫性）
- **フォーマッタと静的解析を必ず適用**し、CI で落ちない状態にする。
  - Java: Spotless / Checkstyle / Error Prone など（プロジェクト採用ツールに従う）
- 例外的なスタイル抑制（`@SuppressWarnings` 等）は**最小限**にし、対象を絞る。
- public な API（Controller/DTO 等）の破壊的変更は**互換性**（バージョニングまたは移行期間）を考慮する。

### 3) 設計（責務分離）
- 層の責務を混ぜない。
  - **Controller**: 入出力・認可・バリデーションの入り口（業務ロジックを置かない）
  - **Service**: 業務ロジック・トランザクション境界
  - **Repository**: 永続化アクセスのみ
- DB への更新を伴う処理は、原則として **Service でトランザクション管理**（`@Transactional`）する。

### 4) 例外処理（API 品質）
- 例外を握りつぶさない（空 catch 禁止）。
- REST API のエラー応答は、原則として **`@RestControllerAdvice` で一元化**し、以下を守る。
  - HTTP ステータスが妥当である（例: バリデーション 400、認可 403、存在しない 404、競合 409、想定外 500）
  - 返却形式は一貫（`code` / `message` / `details` 等、プロジェクトで統一）
  - 想定外エラーは内部情報を返さない（スタックトレース・SQL 等の露出禁止）

### 5) ログ（運用性・監査性）
- 標準は **SLF4J**（`LoggerFactory.getLogger` / Lombok の `@Slf4j` 等）。`System.out.println` 禁止。
- 個人情報・秘密情報（パスワード、トークン、鍵、クレカ等）を**ログに出さない**。
- 例外ログは「必要十分」：想定外は stacktrace、想定内（業務例外等）は過剰に stacktrace を出さない。

### 6) 設定管理（環境差分）
- 環境依存値（URL/パスワード/外部 API）は **`application.yml` + 環境変数（または Secret 管理）**へ。
- **設定のコード埋め込み禁止**（例: DB URL を Java に直書きしない）。
- `application-prod.yml` 等の本番向け設定は、誤ってローカルに適用されないようにプロファイル運用を明確化する。

### 7) テスト（最低限の品質担保）
- 新規/変更したロジックには、少なくとも以下のいずれかを追加する。
  - **ユニットテスト**（Service/Domain）
  - **Web 層テスト**（Controller: `@WebMvcTest`）
  - **結合テスト**（`@SpringBootTest`、必要なら Testcontainers）
- 再現したバグは **再発防止としてテスト化**する（原則）。

### 8) 依存関係・脆弱性
- Spring Boot の BOM に従い、依存バージョンをむやみに固定しない（例外は理由を残す）。
- 重大な脆弱性（Critical/High）は放置しない（アップデートまたは軽減策を取る）。

---

## 推奨（チームで採用すると強い）

### A) パッケージ構成（例）
以下のいずれかで統一し、混在させない。

- 機能別（推奨されがち）
  - `com.example.app.feature.user.controller`
  - `com.example.app.feature.user.service`
  - `com.example.app.feature.user.repository`
  - `com.example.app.feature.user.dto`
- 層別（小規模なら可）
  - `com.example.app.controller`
  - `com.example.app.service`
  - `com.example.app.repository`
  - `com.example.app.dto`

### B) DTO / Validation
- API 入出力は Entity を直返ししない（DTO を使う）。
- Validation は `jakarta.validation` を使い、Controller 入口で検証する。
  - 例: `@Valid` + `@NotNull` / `@Size` / `@Email` 等

### C) OpenAPI（API の合意形成）
- 外部公開または複数クライアントがいる場合、OpenAPI（Swagger）を導入し、API 仕様をレビュー対象にする。

### D) データアクセス
- ページングは `Pageable`（または明示的な limit/offset）で統一し、無制限取得を避ける。
- N+1 問題を意識し、必要に応じて fetch 戦略やクエリを調整する。

### E) セキュリティ
- Spring Security を導入する場合、認可ルールを Controller に散らさず、原則として設定（FilterChain）やメソッドセキュリティで統一する。
- CSRF/CORS/セッション/JWT などは「どの方式か」をプロジェクト標準に明記する。

### F) 観測性（Observability）
- 本番運用するなら `spring-boot-starter-actuator` を導入し、ヘルスチェックとメトリクスを整備する。
- 可能なら相関 ID（Correlation ID）をリクエスト単位で付与し、ログに出す。

### G) CI（最低ライン）
- PR で以下を必ず実行する。
  - ビルド
  - テスト
  - 静的解析（採用していれば）

---

## チェックリスト（PR 作成時）
- [ ] 必須ルールに違反していない（例外なら理由・影響・代替策を書いた）
- [ ] エラー応答/例外処理が一貫している
- [ ] ログに秘密情報が含まれていない
- [ ] 設定をコードに直書きしていない
- [ ] 変更に見合うテストが追加/更新されている

---

## 参考実装（このリポジトリ内）

本標準に沿った **Spring Boot Web（Thymeleaf）画面サンプル** は `web-sample/` を参照してください（README に準拠ポイントと起動手順を記載）。

