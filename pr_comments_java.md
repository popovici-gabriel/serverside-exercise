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



## Medium

| # | Location | Principle | Finding |
|---|---|---|---|
| 9 | Every layer | Logging | There's no logging anywhere, and no `@ExceptionHandler`. A database failure becomes a raw 500 error from TomEE. `log4j.version` is declared but never used. |
| 10 | `YesResultSetExtractor` | Resources, clarity | The nested cursor `ResultSet` is never closed. CLOBs longer than `Integer.MAX_VALUE` are dropped without warning. The `clobVal != null` check runs after the value is already used, so it does nothing. The `rows` counter is never read. Checking for `PgResultSet` ties the code to the Postgres driver; checking for `ResultSet` would work instead. Memory leaks. Suggest try-with-resources as ResultSet is Autoclosable|
| 11 | `YesPreparedStatementCreator` | Proper function | It has no `Long` or `Boolean` branch, so those values fall through to `setString`, which Postgres rejects as a type mismatch. It calls `prepareCall` for a plain `SELECT` where `prepareStatement` fits. Its fields are package-private and could be `private final`. |
| 12 | `YesJSONView` | DRY, clarity | The `ThreadLocal<ObjectMapper>` setup is copied in the constructor and in `render`, and the constructor's copy only seeds the thread that built the view. `ObjectMapper` is thread-safe, so one `private static final` instance is enough. A `ThreadLocal` also risks a classloader leak on redeploy. Any model value that isn't an `ArrayList` is silently dropped. |
| 13 | Three XML files | DRY | The `dataSource` and its credentials (`postgres`/`postgres`) are copied in `application-config.xml` and both test XMLs. Externalize them to a properties file with a placeholder. `maxActive=200` is higher than Postgres's default limit of 100 connections. |
| 14 | POMs | No warnings | `tomcat-catalina` and `tomcat-jdbc` are declared twice in both the parent and service POMs (confirmed by Maven warnings). `dependencyManagement` lists `com.yesenergy.web:web`, but the artifact is actually `PS`. `${web.version}` is undefined. `spring-instrument-tomcat` is a 4.3 artifact in a Spring 5 project. Several comments are mislabeled (for example "snowflake-jdbc" and "org.json"). |
| 15 | POM scopes | Proper packaging | `junit` and `spring-test` (service) and `mockito-core` and `org.json` (api) use compile scope, so they ship inside the WAR. The service module also depends on `spring-webmvc` and `spring-security`, which it doesn't need. |
| 16 | `api_dispatch.xml`, `application-config.xml` | No warnings | `useSuffixPatternMatch` and `PathExtensionContentNegotiationStrategy` are deprecated in 5.3. The handler-mapping class name has a trailing space. MVC view beans live in the root context. `proxy-target-class` is set inconsistently between the two contexts. |

## Low

| # | Location | Principle | Finding |
|---|---|---|---|
| 17 | `YesResultSetExtractor` | No code in comments | Remove the commented-out `SimpleDateFormat` line. |
| 18 | `TestTransactionController` | Tests | It covers only the happy path: no empty-list or error case, and no check for the extra `{}` row. A message says "portfolioid" but the test checks the participant. `toLowerCase()` on the path is unexplained. It loads a full Spring context just to get `YesJSONView`. |
| 19 | `TestTransactionService` | Tests | The hardcoded count is fragile, as #1 shows. Assert against `SELECT count(*)` instead. |
| 20 | General | Clarity | Bean id typo `transactionServiceFascade`. Tabs and spaces are mixed. `StoredProcExecutor` builds its placeholder list by concatenating strings in a loop; `String.join(",", Collections.nCopies(n, "?"))` is simpler. Interfaces expose `ArrayList`/`HashMap` where `List`/`Map` would do. The web module lacks the `-Werror` lint settings the other two modules have. |

