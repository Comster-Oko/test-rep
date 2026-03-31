# Spring Boot 環境構築手順（Windows）

## 前提（何を作るか）
このドキュメントは **Spring Boot の新規プロジェクトを作成し、ローカルで起動して動作確認するまで** の最短手順です（Windows想定）。

---

## 1) Java（JDK）をインストール
- **推奨**: Java **21**（または 17）

インストール後、PowerShell で確認します。

```powershell
java -version
```

---

## 2) プロジェクトを作成（Spring Initializr）
1. `https://start.spring.io/` を開く
2. 例（おすすめ設定）
   - **Project**: Maven
   - **Language**: Java
   - **Spring Boot**: 3.x
   - **Java**: 21（または 17）
   - **Dependencies**: `Spring Web`（まずはこれだけでOK）
3. **Generate** を押して zip をダウンロード
4. zip を任意のフォルダへ展開

---

## 3) 起動して動作確認
展開したプロジェクトフォルダで PowerShell を開いて実行します。

### Maven の場合（Maven Wrapper）
```powershell
.\mvnw spring-boot:run
```

起動ログに以下のようなメッセージが出たらOKです。
- `Tomcat started on port(s): 8080`

ブラウザで `http://localhost:8080/` にアクセスします（最初は 404 でも「起動できた」確認としてはOK）。

---

## 4) “Hello” を返すエンドポイントを追加（確認用）
`src/main/java/.../` 配下にコントローラを追加します（パッケージは既存の構成に合わせてください）。

```java
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class HelloController {
  @GetMapping("/hello")
  public String hello() { return "hello"; }
}
```

アプリを再起動し、`http://localhost:8080/hello` にアクセスして `hello` が返れば環境構築は完了です。

---

## よくあるつまずき
- **`java` が見つからない**
  - JDK インストール後に PATH が通っていない可能性があります（PowerShell/PC の再起動、環境変数の見直し）。
- **ポート 8080 が使用中**
  - `src/main/resources/application.properties`（または `application.yml`）に設定を追加します。

例:
```properties
server.port=8081
```

