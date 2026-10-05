# Java Code review: `service`, `api`, `web`

I split the findings into three categories:  high, medium and low. The high category contains issues that are likely to cause a failure in production, or a security vulnerability. Medium issues are less likely to cause a failure, but they are still worth fixing. Low issues are mostly about clarity and style.

## High

| # | Location | Principle | Finding |
|---|---|---|---|
| 1 | `YesResultSetExtractor.extractData` | Proper function | When a column holds a refcursor, its rows are added and then the empty outer `row` is still added too. The API therefore returns an extra `{}`. The frontend sorts empty values first, so a blank row appears on page 1. `TestTransactionService` asserts 5,000 rows against a 4,999-row fixture, so the test only passes because of this bug. Fix: skip `results.add(row)` when you expand a cursor, then correct the test. |
| 2 | `YesPreparedStatementCreator` | Proper function, scalability | `conn.setAutoCommit(false)` runs on a pooled connection and is never committed or reset. Connections go back to the pool still inside an open transaction ("idle in transaction"), and later users inherit that state. Use a `DataSourceTransactionManager` or `TransactionTemplate` instead of changing connection state by hand. |
| 3 | `StoredProcExecutor.fetch` | Proper function, security | `"select " + storedProcedureName` puts the function name straight into the SQL. Nothing enforces the "constants only" rule. Check the name against a whitelist or the pattern `^[a-z_][a-z0-9_]*$`. |
| 4 | Whole request path | Scalability | Every request returns all rows, and the client sorts and pages them. `setFetchSize(20)` only suggests streaming; it doesn't stream. Move sorting and paging into the database function with `ORDER BY`/`LIMIT`/`OFFSET` parameters. |
| 5 | `api_dispatch.xml` | Proper function | `<tx:annotation-driven transaction-manager="myTransactionManager"/>` refers to a bean that doesn't exist. The first `@Transactional` method will fail. |
| 6 | `applicationContext-security.xml` | Proper function | `access="ROLE_ANONYMOUS"` on `/**` blocks every authenticated user. The XML also hardcodes a user and password, and CSRF is disabled. |
| 7 | StoredProcExecutor | Design flaws | `StoredProcExecutor` is not a working template for fetching in batches. The whole result set is loaded into memory at three separate points, and `setFetchSize(20)` has no effect on any of them. All 4,999 rows reach the JVM in one round trip. No fetch size is involved (res.getObject(col))|
| 8 | XML Configuration | Design flaws | Consider using Spring Boot: Convention over  configuration over XML configuration |
| 9 | Every layer (`ITransactionService`, `IStoredProcExecutor`, `StoredProcExecutor`, `YesResultSetExtractor`, `TransactionService`, `ITransactionServiceFacade`, `TransactionServiceFacade`, tests) | Design flaws, clarity, testability | **No domain model.** Every layer passes `ArrayList<HashMap<String, Object>>` instead of a typed model. Java is statically typed, but this design throws the type system away: column names are unchecked string keys, values need casts (`(String) results.get(0).get("initcap")`), and the database column list *is* the API contract with no compile-time check. A renamed or removed column (such as the missing `participantfullname`) compiles and passes, then fails silently in the UI. Type drift is already present: `TestTransactionController` mocks `hours` as `BigInteger`, but PostgreSQL `bigint` is returned as `Long`. Fix: add a `Transaction` POJO (or `record` after upgrading from Java 8 to 16+), map rows with a `RowMapper<Transaction>` in a `TransactionRepository`, return `List<Transaction>` from the service and controller, and remove the generic `StoredProcExecutor`/`YesResultSetExtractor`. See [Typed model and RowMapper](#typed-model-and-rowmapper-finding-9). |



## Medium

| # | Location | Principle | Finding |
|---|---|---|---|
| 10 | Every layer | Logging | There's no logging anywhere, and no `@ExceptionHandler`. A database failure becomes a raw 500 error from TomEE. `log4j.version` is declared but never used. |
| 11 | `YesResultSetExtractor` | Resources, clarity | The nested cursor `ResultSet` is never closed. CLOBs longer than `Integer.MAX_VALUE` are dropped without warning. The `clobVal != null` check runs after the value is already used, so it does nothing. The `rows` counter is never read. Checking for `PgResultSet` ties the code to the Postgres driver; checking for `ResultSet` would work instead. Memory leaks. Suggest try-with-resources as ResultSet is Autoclosable|
| 12 | `YesPreparedStatementCreator` | Proper function | It has no `Long` or `Boolean` branch, so those values fall through to `setString`, which Postgres rejects as a type mismatch. It calls `prepareCall` for a plain `SELECT` where `prepareStatement` fits. Its fields are package-private and could be `private final`. |
| 13 | `YesJSONView` | DRY, clarity | The `ThreadLocal<ObjectMapper>` setup is copied in the constructor and in `render`, and the constructor's copy only seeds the thread that built the view. `ObjectMapper` is thread-safe, so one `private static final` instance is enough. A `ThreadLocal` also risks a classloader leak on redeploy. Any model value that isn't an `ArrayList` is silently dropped. |
| 14 | Three XML files | DRY | The `dataSource` and its credentials (`postgres`/`postgres`) are copied in `application-config.xml` and both test XMLs. Externalize them to a properties file with a placeholder. `maxActive=200` is higher than Postgres's default limit of 100 connections. |
| 15 | POMs | No warnings | `tomcat-catalina` and `tomcat-jdbc` are declared twice in both the parent and service POMs (confirmed by Maven warnings). `dependencyManagement` lists `com.yesenergy.web:web`, but the artifact is actually `PS`. `${web.version}` is undefined. `spring-instrument-tomcat` is a 4.3 artifact in a Spring 5 project. Several comments are mislabeled (for example "snowflake-jdbc" and "org.json"). |
| 16 | POM scopes | Proper packaging | `junit` and `spring-test` (service) and `mockito-core` and `org.json` (api) use compile scope, so they ship inside the WAR. The service module also depends on `spring-webmvc` and `spring-security`, which it doesn't need. |
| 17 | `api_dispatch.xml`, `application-config.xml` | No warnings | `useSuffixPatternMatch` and `PathExtensionContentNegotiationStrategy` are deprecated in 5.3. The handler-mapping class name has a trailing space. MVC view beans live in the root context. `proxy-target-class` is set inconsistently between the two contexts. |

## Low

| # | Location | Principle | Finding |
|---|---|---|---|
| 18 | `YesResultSetExtractor` | No code in comments | Remove the commented-out `SimpleDateFormat` line. |
| 19 | `TestTransactionController` | Tests | It covers only the happy path: no empty-list or error case, and no check for the extra `{}` row. A message says "portfolioid" but the test checks the participant. `toLowerCase()` on the path is unexplained. It loads a full Spring context just to get `YesJSONView`. |
| 20 | `TestTransactionService` | Tests | The hardcoded count is fragile, as #1 shows. Assert against `SELECT count(*)` instead. |
| 21 | General | Clarity | Bean id typo `transactionServiceFascade`. Tabs and spaces are mixed. `StoredProcExecutor` builds its placeholder list by concatenating strings in a loop; `String.join(",", Collections.nCopies(n, "?"))` is simpler. Interfaces expose concrete `ArrayList` instead of `List` (the `Map` element type is covered by #9). The web module lacks the `-Werror` lint settings the other two modules have. |


## Typed model and RowMapper (finding #9)

### Where `Map<String, Object>` is used today

| Layer | Signature |
|---|---|
| `IStoredProcExecutor` / `StoredProcExecutor` | `ArrayList<HashMap<String, Object>> fetch(String, Object...)` |
| `YesResultSetExtractor` | `ResultSetExtractor<ArrayList<HashMap<String, Object>>>` |
| `ITransactionService` / `TransactionService` | `ArrayList<HashMap<String, Object>> getAwardedTransactions()` |
| `ITransactionServiceFacade` / `TransactionServiceFacade` | `ArrayList<HashMap<String, Object>> getTransactionList()` |
| `TestStoredProcExecutor`, `TestTransactionService`, `TestTransactionController` | Assert on string keys and cast values |

The one acceptable `Map<String, Object>` is the `model` parameter of `YesJSONView.renderMergedOutputModel`, because Spring's `AbstractView` API defines it. With a typed controller response that view is no longer needed at all.

### What goes wrong without types

- **No compile-time contract.** `row.get("participantfullname")` compiles even though the SQL never selects that column.
- **Casts everywhere.** Callers must know each value's JDBC type (`Float` for `real`, `Long` for `bigint`) and cast it; a wrong guess is a runtime `ClassCastException`.
- **Mocks drift from reality.** `TestTransactionController` builds `hours` as `BigInteger`, but the driver returns `Long`. A typed model makes that impossible.
- **Leaky API.** Every column the SQL selects is sent to the browser, including the six unused ones. Adding a column to the query silently changes the public JSON.
- **Hard to read and refactor.** The IDE can't find usages of a field, rename it, or show which fields exist.

### Fix

The project targets Java 8, so `record` isn't available. Use an immutable POJO now, and switch to a `record` after upgrading to Java 16+ (Java 17 LTS recommended).

**Domain model (`service` module):**

```java
package com.yesenergy.service.model;

public final class Transaction {
    private final String participantShortName;
    private final String participantFullName;
    private final String tradeType;
    private final String peakType;
    private final String hedgeType;
    private final long sourceId;
    private final long sinkId;
    private final float contractSize;
    private final float cost;
    private final float revenue;
    private final float profit;

    public Transaction(String participantShortName, String participantFullName, String tradeType,
            String peakType, String hedgeType, long sourceId, long sinkId,
            float contractSize, float cost, float revenue, float profit) {
        this.participantShortName = participantShortName;
        this.participantFullName = participantFullName;
        this.tradeType = tradeType;
        this.peakType = peakType;
        this.hedgeType = hedgeType;
        this.sourceId = sourceId;
        this.sinkId = sinkId;
        this.contractSize = contractSize;
        this.cost = cost;
        this.revenue = revenue;
        this.profit = profit;
    }

    public String getParticipantShortName() { return participantShortName; }
    public String getParticipantFullName() { return participantFullName; }
    public String getTradeType() { return tradeType; }
    public String getPeakType() { return peakType; }
    public String getHedgeType() { return hedgeType; }
    public long getSourceId() { return sourceId; }
    public long getSinkId() { return sinkId; }
    public float getContractSize() { return contractSize; }
    public float getCost() { return cost; }
    public float getRevenue() { return revenue; }
    public float getProfit() { return profit; }
}
```

On Java 16+ the same model is one declaration:

```java
public record Transaction(String participantShortName, String participantFullName,
        String tradeType, String peakType, String hedgeType, long sourceId, long sinkId,
        float contractSize, float cost, float revenue, float profit) {}
```

If the columns can be `NULL`, use `Long`/`Float` (or `BigDecimal` for money) instead of primitives, and read them with `rs.getObject(col, Long.class)`.

**RowMapper:** keeps the column-name strings in one place:

```java
public final class TransactionRowMapper implements RowMapper<Transaction> {
    @Override
    public Transaction mapRow(ResultSet rs, int rowNum) throws SQLException {
        return new Transaction(
            rs.getString("participantshortname"),
            rs.getString("participantfullname"),
            rs.getString("tradetype"),
            rs.getString("peaktype"),
            rs.getString("hedgetype"),
            rs.getLong("sourceid"),
            rs.getLong("sinkid"),
            rs.getFloat("contractsize"),
            rs.getFloat("cost"),
            rs.getFloat("revenue"),
            rs.getFloat("profit"));
    }
}
```

**Repository:** replaces the generic `StoredProcExecutor` and `YesResultSetExtractor`. It assumes the set-returning function proposed in `pr_comments_db.md`; no refcursor means no `setAutoCommit(false)` and no recursive `PgResultSet` unwrapping:

```java
public class JdbcTransactionRepository extends JdbcDaoSupport implements TransactionRepository {
    private static final String SQL = "select * from get_awarded_transactions(?, ?)";
    private static final RowMapper<Transaction> MAPPER = new TransactionRowMapper();

    @Override
    public List<Transaction> findAwarded(int limit, int offset) {
        return getJdbcTemplate().query(SQL, MAPPER, limit, offset);
    }
}
```

**Service and controller:** typed end to end:

```java
public interface ITransactionService {
    List<Transaction> getAwardedTransactions(int limit, int offset);
}

@Controller
public class TransactionServiceFacade {
    @Autowired
    private ITransactionService transactionService;

    @RequestMapping(method = RequestMethod.GET, value = "/ftr/transactions")
    @ResponseBody
    public List<Transaction> getTransactionList(
            @RequestParam(defaultValue = "20") int limit,
            @RequestParam(defaultValue = "0") int offset) {
        return transactionService.getAwardedTransactions(limit, offset);
    }
}
```

`@ResponseBody` uses the `MappingJackson2HttpMessageConverter` already registered in `api_dispatch.xml`, so `YesJSONView` and its flattening logic can be removed.

### JSON field names

Jackson names properties after the getters (`participantShortName`), but the frontend reads lowercase database names (`participantshortname`). Either:
- update `Row.jsx`, `ControlBar.jsx` and the sort values to camelCase in the same PR (recommended, because it makes the API contract explicit), or
- keep the current JSON by annotating each getter, e.g. `@JsonProperty("participantshortname")`. That puts a Jackson dependency on the `service` model; to avoid it, map to a separate response DTO in the `api` module.

### Wiring and tests

- Declare `JdbcTransactionRepository` in `application-config.xml` **and** both `test-application-configuration.xml` files, replacing the `storedProcExecutor` bean.
- `TestTransactionService` asserts on getters (`first.getSourceId()`) instead of `instanceof` checks on map values.
- `TestTransactionController` mocks `List<Transaction>` using the constructor, so a wrong type no longer compiles.
- Add a unit test for `TransactionRowMapper` with a mocked `ResultSet`. It doesn't need a database.
