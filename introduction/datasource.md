---
description: 사소한게 이렇게 장애를 만든다.
---

# 📒 동적으로 생성되는 Datasource 인한 장애

입사 초기 말도 안되는 장애를 만난적이 있다.

`spring framework 3`버전 `java 7`로 개발된 레거시 제품에서 발생 하였다.



Spring boot를 사용하게 되면 data-jpa 의존하여 `DataSourceAutoConfiguration.class, HibernateJpaAutoConfiguration.class, JpaRepositoriesAutoConfiguration.class` auto 설정들에 의해 속성으로 간단하게 datasource를 생성 할 수 있지만 해당 솔루션은 웹 페이지에서 db 커낵션을 연결 할 수 기능이 있고, 내부 로직에 동적으로 datasource를 만들어 커낵션 풀을 생성 하는 로직이 존재 했었다.

## 지금까지 해당 솔루션을 잘 사용하다가 왜 문제가 발생했나?



내부 코드는 오픈 할 수 없어 그때 수정하기전 로직을 간단히 작성 해보았다.

아래 로직을 보자!

* Start.class

```java
// auto config 제외
@SpringBootApplication(exclude = {DataSourceAutoConfiguration.class})
public class Start {

    public static void main(String[] args) {
        SpringApplication.run(Start.class, args);
    }

}
```

* `DynamicDataSourceComponent.class`

```java
@Component
public class DynamicDataSourceComponent {
  private final Map<String, DataSource> dataSourceMap = new HashMap<>();

  public DataSource getDataSource(String key, String jdbcUrl, String username, String password) {
    if (!dataSourceMap.containsKey(key)) {
      DataSource dataSource = createDataSource(jdbcUrl, username, password);
      dataSourceMap.put(key, dataSource);
    }
    return dataSourceMap.get(key);
  }

  private DataSource createDataSource(String jdbcUrl, String username, String password) {
    HikariConfig config = new HikariConfig();
    config.setJdbcUrl(jdbcUrl);
    config.setUsername(username);
    config.setPassword(password);
    config.setDriverClassName("org.postgresql.Driver");
    config.setMinimumIdle(10);
    config.setMaximumPoolSize(20);

    return new HikariDataSource(config);
  }
}
```



* `DynamicDataSourceService.class`

```java
@Slf4j
@Service
@RequiredArgsConstructor
public class DynamicDataSourceService {
  private final DynamicDataSourceComponent dynamicDataSourceComponent;

  public void executeQuery(String key, String jdbcUrl, String username, String password)
    throws SQLFeatureNotSupportedException {
    DataSource dataSource = dynamicDataSourceComponent.getDataSource(key, jdbcUrl, username, password);
    JdbcTemplate jdbcTemplate = new JdbcTemplate(dataSource);

    // 예시 쿼리 실행
    String sql = "SELECT COUNT(*) FROM some_table";
    Integer count = jdbcTemplate.queryForObject(sql, Integer.class);
    log.info("Thread : {}, count : {}", Thread.currentThread().getName(), count);
  }
}
```



간단하다. `DynamicDataSourceComponent.class` 는 datasource를 key에 따라 존재하면 찾아와서 사용하고 그렇지 않으면 생성하여 가져오는 로직이다. 그리고 `DynamicDataSourceService.class`에서 해당 컴포넌트를 의존하여 사용한다.



위 로직은 정말 어마어마한 장애를 낸다...



`DynamicDataSourceService` 클래스의 `executeQuery`가 여러 요청에 의해 동시 호출이 발생했던 것이 문제였다.



DynamicDataSourceComponent의 HashMap 이부분이 동시성이 고려되지 않았기 때문인데..

사실... 데이터베이스 커낵션을 설정화면에서 만 해당 로직을 사용했다면 문제가 크게 없겠지만....



저 로직을 개발 하신분은 lazy 하게 생성하고 싶었나보다.. 특정 db 의 데이터를 조회해서 가져올 시점에서 datasource가 생성되어 있지않으면 생성하고 가져오게 로직을 작성한 것 이였다.&#x20;

아무리 생각해봐도 db 커낵션 pool을 lazy 하게 생성할 필요가 있을까??...



## 동시성을 고려하지 않았을 때 결과를 보자

테스트 코드는 아래와 같이 간단하게 작성해보았다.



```java
@SpringBootTest
@Testcontainers
class DynamicDataSourceServiceTest {

  @Container
  public static PostgreSQLContainer<?> postgreSQLContainer = new PostgreSQLContainer<>("postgres:latest")
    .withDatabaseName("test")
    .withUsername("username")
    .withPassword("password");

  @Autowired
  private DynamicDataSourceService dynamicDataSourceService;

  @BeforeAll
  public static void setUp() {
    postgreSQLContainer.start();
    createSomeTable();
  }

  @AfterAll
  public static void tearDown() {
    System.out.println("postgresql container shutting down");
    postgreSQLContainer.stop();
  }

  private static void createSomeTable() {
    try {
      Connection connection = DriverManager.getConnection(
        postgreSQLContainer.getJdbcUrl(),
        postgreSQLContainer.getUsername(),
        postgreSQLContainer.getPassword()
      );
      Statement statement = connection.createStatement();
      statement.execute("CREATE TABLE some_table (id SERIAL PRIMARY KEY, name VARCHAR(255), value INT)");
      statement.execute("INSERT INTO some_table (name, value) VALUES ('test1', 10), ('test2', 20)");
      statement.close();
      connection.close();
    } catch (Exception e) {
      e.printStackTrace();
    }
  }

  @Test
  void testExecuteQuery() throws InterruptedException {
    String jdbcUrl = postgreSQLContainer.getJdbcUrl();
    String username = postgreSQLContainer.getUsername();
    String password = postgreSQLContainer.getPassword();
    int threadCount = 5;  // 동시에 실행할 스레드 수
    ExecutorService executor = Executors.newFixedThreadPool(threadCount);

    for (int i = 0; i < threadCount; i++) {
      executor.submit(() -> {
        try {
          dynamicDataSourceService.executeQuery("test", jdbcUrl, username, password);
        } catch (SQLFeatureNotSupportedException e) {
          throw new RuntimeException(e);
        }
      });
    }

    executor.shutdown(); // 커낵션 풀 종료
    executor.awaitTermination(1, TimeUnit.MINUTES);

    // postgresql 커낵션 상태 확인
    Thread.sleep(2000);
    checkActiveConnections(jdbcUrl, username, password);
    getMaxConnections(jdbcUrl, username, password);

    Thread.sleep(5000);
  }

  private void checkActiveConnections(String jdbcUrl, String username, String password) {
    String sql = "SELECT count(*) FROM pg_stat_activity WHERE state = 'idle'";
    try (Connection connection = DriverManager.getConnection(jdbcUrl, username, password);
      Statement statement = connection.createStatement();
      java.sql.ResultSet resultSet = statement.executeQuery(sql)) {

      if (resultSet.next()) {
        int activeConnections = resultSet.getInt(1);
        System.out.println("Active connections: " + activeConnections);
      }
    } catch (Exception e) {
      e.printStackTrace();
    }
  }

  public void getMaxConnections(String jdbcUrl, String username, String password) {
    String sql = "SHOW max_connections";
    try (Connection connection = DriverManager.getConnection(jdbcUrl, username, password);
      Statement statement = connection.createStatement();
      java.sql.ResultSet resultSet = statement.executeQuery(sql)) {

      if (resultSet.next()) {
        int activeConnections = resultSet.getInt(1);
        System.out.println("Max connections: " + activeConnections);
      }
    } catch (Exception e) {
      e.printStackTrace();
    }
  }
}
```



동시에 test 라는 데이터베이스에서 데이터를 조회하는 요청이 들어왔다고 보고, postgresql 커낵션 상태와 max 커낵션 상태를 출력해보았다.



Datasource 설정 옵션은 maxConnectionPoolSize 20, 최소 idle connection 은 10으로 설정 하였을 때 동시 요청이 5개가 왔다고 가정해보자

```
2024-07-22T09:39:19.408+09:00 DEBUG 27838 --- [pool-2-thread-4] com.zaxxer.hikari.HikariConfig           : HikariPool-4 - configuration:
2024-07-22T09:39:19.408+09:00 DEBUG 27838 --- [pool-2-thread-1] com.zaxxer.hikari.HikariConfig           : HikariPool-2 - configuration:
2024-07-22T09:39:19.408+09:00 DEBUG 27838 --- [pool-2-thread-2] com.zaxxer.hikari.HikariConfig           : HikariPool-5 - configuration:
2024-07-22T09:39:19.408+09:00 DEBUG 27838 --- [pool-2-thread-3] com.zaxxer.hikari.HikariConfig           : HikariPool-1 - configuration:
2024-07-22T09:39:19.408+09:00 DEBUG 27838 --- [pool-2-thread-5] com.zaxxer.hikari.HikariConfig           : HikariPool-3 - configuration:
```

test 키에 pool이 5개나 생성되는 것을 볼 수 있고

```
2024-07-22T09:39:19.829+09:00 DEBUG 27838 --- [onnection adder] com.zaxxer.hikari.pool.HikariPool        : HikariPool-2 - After adding stats (total=10, active=0, idle=10, waiting=0)
2024-07-22T09:39:19.830+09:00 DEBUG 27838 --- [onnection adder] com.zaxxer.hikari.pool.HikariPool        : HikariPool-4 - After adding stats (total=10, active=0, idle=10, waiting=0)
2024-07-22T09:39:19.830+09:00 DEBUG 27838 --- [onnection adder] com.zaxxer.hikari.pool.HikariPool        : HikariPool-5 - After adding stats (total=10, active=0, idle=10, waiting=0)
2024-07-22T09:39:19.830+09:00 DEBUG 27838 --- [onnection adder] com.zaxxer.hikari.pool.HikariPool        : HikariPool-1 - After adding stats (total=10, active=0, idle=10, waiting=0)
2024-07-22T09:39:19.830+09:00 DEBUG 27838 --- [onnection adder] com.zaxxer.hikari.pool.HikariPool        : HikariPool-3 - After adding stats (total=10, active=0, idle=10, waiting=0)
```

각 datasource가 최소 idle 을 10개씩 가지고 초기화가 완료되었다.



이후 postgresql 커낵션 상태를 보면

<figure><img src="../.gitbook/assets/스크린샷 2024-07-22 오전 9.46.38.png" alt=""><figcaption></figcaption></figure>

최대 연결 수량은 100인데.. 50개를 활성하고 있다.

말이 안되는 상황이다.



또한, datasource를 lazy 했을때 와 그렇지 않을때 차이를 보면

**Lazy**

```
wrk -t10 -c10 -d10s http://localhost:8080/execute\?key\=test

Running 10s test @ http://localhost:8080/execute?key=test
  10 threads and 10 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     8.29ms   32.40ms 269.51ms   95.10%
    Req/Sec     0.88k   307.92     1.37k    67.89%
  83580 requests in 10.02s, 11.73MB read
Requests/sec:   8339.89
Transfer/sec:      1.17MB
```

**non-Lazy**

```
wrk -t10 -c10 -d10s http://localhost:8080/execute\?key\=test

Running 10s test @ http://localhost:8080/execute?key=test
  10 threads and 10 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     2.30ms    9.65ms 135.81ms   97.91%
    Req/Sec     1.03k   268.87     1.37k    79.60%
  102502 requests in 10.02s, 14.39MB read
Requests/sec:  10232.85
Transfer/sec:      1.44MB
```

10명의 사용자가 동시 요청을 하였을때 평균 응답 시간이 4배 정도 차이나는 것을 알 수 있다.



## 해결

간단하게 ConcurrentHashMap을 사용하는 게 편하나 Lazy 로 db 커낵션을 생성해야 할 이점이 없고

동적으로 웹 UI 에서 설정할 때 커낵션 정보들이 db에 테이블로 관리 되고 있어 어플리케이션 시작시 target database들의 datasource를 미리 생성해두는 방식으로 변경하였다.













