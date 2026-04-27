---
name: java-code-review
description: Java / Spring Boot 程式碼審查流程。當使用者要求 review java、審查 java、檢查 spring boot 程式碼、code review、看一下這段 java 等情境時觸發。會依固定步驟讀取目標檔案，按照 Java 通用規則 + Spring Boot 專屬規則檢查，並以「嚴重問題 / 警告 / 建議」三級格式輸出。
---

# Java / Spring Boot Code Review Workflow

## 0. 觸發確認
- 觸發詞（中英混用皆可）：`review java`、`審查 java`、`檢查 java`、`spring boot review`、`看一下這段 java`
- 若使用者沒指定檔案，主動詢問要 review 的範圍：
    - 單一檔案路徑
    - 某個套件 / 資料夾
    - 「目前未 commit 的 Java 變更」（此時用 `git diff` 取得範圍）

## 1. 前置資訊收集
1. 用 `Read` / `Glob` 讀取目標 Java 檔案
2. 若是 Spring Boot 專案，先看一次 `pom.xml` 或 `build.gradle` 確認：
    - Spring Boot 版本（影響可用註解、deprecated API）
    - JDK 版本（17+ 可用 record / sealed class / pattern matching）
    - 用了哪些 starter（web、data-jpa、security、validation⋯）
3. 若有 `application.yml` / `application.properties`，掃一次確認是否有設定相關的 smell

## 2. 檢查類別（請逐項過，命中才寫入報告）

### A. Java 語言層級（通用）
- **Null safety**：高風險物件是否用 `Optional` 包裝？避免 chain call 直接 NPE
- **資源管理**：`InputStream`、`Connection`、`FileReader` 是否使用 `try-with-resources`
- **例外處理**：
    - 是否 `catch (Exception e)` 吞掉所有錯誤
    - 是否 `catch` 之後只 `e.printStackTrace()` 不處理也不往上拋
    - 是否在 `catch` 區塊吃掉了應該回滾交易的例外（與 D 段 `@Transactional` 連動）
- **equals / hashCode**：override 一個就要 override 另一個；Lombok `@Data` 是否誤用在 JPA Entity（會引發 lazy loading + 無限遞迴）
- **不可變性**：DTO / VO 欄位優先 `final`；集合回傳考慮 `List.copyOf` 防止外部修改
- **String 操作**：迴圈內字串拼接改用 `StringBuilder`
- **Stream API**：
    - 不要在 `forEach` 裡做有副作用的事（要用就用 `peek` 為 debug 用途）
    - terminal operation 是否漏寫（`stream()` 後沒 `collect` / `toList`）
    - 不要用 stream 改寫單純 for 迴圈反而讓可讀性下降
- **集合**：用 `isEmpty()` 而不是 `size() == 0`；`Map.getOrDefault` / `computeIfAbsent` 取代手動判 null
- **Magic number / String**：抽成 `private static final` 常數
- **方法長度**：> 50 行或 cyclomatic complexity 過高 → 建議拆分

### B. 並行 / 執行緒
- 共享可變狀態是否用 `ConcurrentHashMap` / `AtomicXxx` / `synchronized`
- `SimpleDateFormat` 不是 thread-safe → 用 `DateTimeFormatter`
- `@Async` 方法是否回傳 `CompletableFuture` 才能拿到結果
- ThreadLocal 用完是否 `remove()`（避免 thread pool 記憶體洩漏）

### C. Logging
- 使用 SLF4J（`org.slf4j.Logger`），不要用 `System.out.println` 或 `e.printStackTrace()`
- 參數化訊息：`log.info("user={} login", userId)`，**不要**字串拼接
- 不要 log 密碼、token、身分證、信用卡
- 例外要連 stack trace 一起 log：`log.error("msg", e)`，不要 `log.error("msg" + e)`

---

### D. Spring Boot 專屬規則（重點）

#### D1. 依賴注入
- **強制使用 constructor injection**，搭配 Lombok `@RequiredArgsConstructor` 與 `private final` 欄位
- 不要 `@Autowired` 在欄位上（難測試、無法 final）
- 若有循環依賴 → 重構，**不要**用 `@Lazy` 繞過

#### D2. Controller 層
- 不要把 JPA Entity 直接回傳給前端 → 用 DTO / Response 物件
- request body 要加 `@Valid`，並對 DTO 加 `@NotNull` / `@Size` / `@Email` 等
- 統一例外處理用 `@RestControllerAdvice` + `@ExceptionHandler`，不要每個 controller 自己 try-catch
- HTTP status 要正確：建立成功用 201、找不到用 404、權限不足用 403
- 路徑命名遵循 RESTful：名詞複數、動詞用 HTTP method 表達（`POST /users` 而非 `POST /createUser`）

#### D3. Service 層 / 交易
- `@Transactional` 只能放 **public method**（self-invocation 不會觸發 proxy）
- 預設 `@Transactional(readOnly = true)` 在 class 上，寫入方法另外加 `@Transactional`
- 不要在 `@Transactional` 方法內 `catch` 掉應該觸發 rollback 的例外
- 跨 service 呼叫要注意 propagation（`REQUIRED` vs `REQUIRES_NEW`）

#### D4. JPA / 資料庫
- **N+1 查詢**：lazy 關聯欄位在 loop 中存取 → 改用 `@EntityGraph` 或 `JOIN FETCH`
- 不要在 controller 直接用 `EntityManager` / `Repository`，要走 service
- 自訂查詢優先 `@Query` JPQL，避免字串拼接 SQL（SQL injection）
- 大量資料要分頁（`Pageable`），不要 `findAll()` 全撈
- Entity 不要用 Lombok `@Data` / `@EqualsAndHashCode` 全欄位（lazy loading 觸發、雙向關聯無限遞迴）→ 用 `@Getter` + 自己寫 equals 基於 id
- 主鍵 `@GeneratedValue` 策略要明確（`IDENTITY` vs `SEQUENCE`），別用預設

#### D5. 設定與安全
- 機密（DB 密碼、API key、JWT secret）**禁止**寫死在 `application.yml` → 用環境變數或 Spring Cloud Config
- 一組相關設定用 `@ConfigurationProperties`，不要散落 `@Value`
- 開放的 endpoint 要在 Spring Security 設定中明確列出（白名單 > 黑名單）
- CORS 不要設 `*` 上 production
- 密碼一律 `BCryptPasswordEncoder`，不要 MD5 / SHA-1

#### D6. Bean 與 Scope
- `@Component` / `@Service` / `@Repository` / `@Controller` 用對語意（`@Repository` 會做 SQLException 轉換）
- 預設 singleton；要 prototype 必須有理由並註解說明
- `@PostConstruct` 不要做太重的事（會拖慢啟動）

---

### E. 測試
- 用 JUnit 5 (`org.junit.jupiter`)，不要混用 JUnit 4
- 斷言用 AssertJ (`assertThat(...).isEqualTo(...)`)，可讀性 > Hamcrest
- **slice test 優先**：`@WebMvcTest`（controller）、`@DataJpaTest`（repository），不要動不動 `@SpringBootTest`（啟動慢）
- Mockito：`@ExtendWith(MockitoExtension.class)` + `@Mock` / `@InjectMocks`
- 不要為了覆蓋率測 getter / setter / framework 行為
- 測試命名：`方法_情境_預期結果`，例如 `findUser_whenNotExist_throwsNotFound`

---

## 3. 輸出格式

每個問題用以下結構，並依嚴重度分群：

```
### [嚴重問題] 標題（檔案:行號）
- 問題：<具體描述>
- 風險：<會造成什麼後果，例如：production NPE、資料外洩、N+1 查詢>
- 建議：<具體修法，必要時附 1~5 行 code snippet>
```

嚴重度定義：
| 等級 | 定義 | 範例 |
|------|------|------|
| 嚴重問題 | 會造成 production bug、資安漏洞、資料錯誤 | SQL injection、密碼明文、@Transactional 失效、N+1、機密寫死 |
| 警告 | 不會立刻爆，但累積會出事或難維護 | field injection、catch Exception、Entity 直接回傳前端、缺 `@Valid` |
| 建議 | 風格 / 可讀性 / 微小優化 | 用 isEmpty()、抽常數、Stream 改寫 |

最後附一段 **Summary**：總共幾項嚴重 / 警告 / 建議，以及最該優先修的 Top 3。

## 4. 邊界
- 不主動修改程式碼，除非使用者在看完報告後明確要求 `幫我修` / `apply`
- 若檔案非 `.java`（例如 `.kt`、`.groovy`），中止並告知此 skill 僅針對 Java
- 規則命中很多時，每個類別最多列 5 項代表性問題，其餘以「其他類似問題 N 處」帶過，避免報告爆炸
