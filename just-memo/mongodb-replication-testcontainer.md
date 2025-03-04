---
description: 이런건 테스트가 어렵더라...
---

# 📒 Mongodb replication Testcontainer

몽고디비 테스트 컨테이너 기본적으로 1개 노드로 replica set으로 실행된다

허나 실제 여러 노드를 set으로 구성하여 트랜잭션이 없는 로직에서는 secondry 에서 읽기 테스트를 하거나

읽기, 쓰기 고려 설정에 따라 로직이 영향을 받을 수 있는지 테스트 할땐 실제 set을  구성하여 여러 상황에 따라 테스트 해볼 필요가 있다



일단 mongodb set 구성은 검색 조금만 해보면 스크립드가 널리 알려져 있다

이 스크립트 베이스로 컨테이너를 구성해보자

바로 로직을 보자



**MongoDBReplicaSetTestContainer.java**

```java
public class MongoDBReplicaSetTestContainer {

  private final Network network = newNetwork();
  private final String rsName;
  private final String mongoImageVersionTag;
  private final List<PortAndContainer> nodes = new ArrayList<>();
  public MongoDBReplicaSetTestContainer() {
    this("rs0", "mongo:latest");
  }

  public MongoDBReplicaSetTestContainer(String rsName, String mongoImageVersionTag) {
    this.rsName = rsName;
    this.mongoImageVersionTag = mongoImageVersionTag;
    init(MongoSetConfig.MONGO_ALIAS_PREFIX_AND_NUMBER_OF_REPLICAS);
  }

  public String getConnectionString() {
    return nodes.stream()
      .map(node -> String.format("%s:%d", node.container().getHost(), node.port))
      .collect(Collectors.joining(",", "mongodb://", "/?replicaSet=" + rsName));
  }

  public void stop() {
    nodes.forEach(node -> node.container().stop());
  }

  public void start() {
    nodes.forEach(node -> node.container().start());

    var replicaMembersCfg = nodes.stream()//
      .map(node -> format("{\"_id\": %d, \"host\": \"%s:%d\"}", nodes.indexOf(node),
        node.mongoAlias, node.port))//
      .collect(Collectors.joining(","));
    var rsInitiate = format("rs.initiate({\"_id\": \"%s\", \"members\": [%s]})", rsName,
      replicaMembersCfg);

    var node = nodes.get(0);
    var nodeUrl = node.mongoAlias + ":" + node.port;

    execInContainer(node.container(), "mongosh", nodeUrl + "/admin", "--quiet", "--eval",
      rsInitiate);
    execInContainer(node.container(), "/bin/bash", "-c",
      "until mongosh " + nodeUrl + "/admin"
        + " --eval \"printjson(rs.isMaster())\" | grep ismaster | grep true > /dev/null 2>&1;"
        + "do sleep 1;done");
  }

  private void init(MongoSetConfig mongoSetConfig) {
    int n = mongoSetConfig.getNumberOfReplicas();
    if (n < 1) {
      throw new IllegalArgumentException("At least one node is required");
    }

    for (int i = 0; i < n; i++) {
      var mongoAlias = mongoSetConfig.getMongoIndex(i);
      var port = findAvailableTcpPort();
      var node = new GenericContainer<>(mongoImageVersionTag)
        .withNetwork(network)
        .withExposedPorts(port)
        .withCreateContainerCmdModifier(cmd -> Objects.requireNonNull(cmd.getHostConfig())
          .withPortBindings(
            new PortBinding(Binding.bindPort(port), new ExposedPort(port))
          ))
        .withNetworkAliases(mongoAlias)
        .waitingFor(Wait.forLogMessage(".*Waiting for connections.*\\n", 1))
        .withCommand("mongod", "--replSet", rsName, "--bind_ip_all", "--port",
          String.valueOf(port));
      nodes.add(new PortAndContainer(port, mongoAlias, node));
    }
  }

  private int findAvailableTcpPort() {
    try (ServerSocket socket = new ServerSocket(0)) {
      return socket.getLocalPort();
    } catch (IOException e) {
      throw new IllegalStateException("Failed to find available port", e);
    }
  }

  private void execInContainer(GenericContainer<?> container, String... command) {
    try {
      var result = container.execInContainer(command);
      if (result.getExitCode() != 0) {
        throw new RuntimeException(format("Failed execution of command. Code: %s, output: %s ",
          result.getExitCode(), result.getStdout() + "\n" + result.getStderr()));
      }
    } catch (UnsupportedOperationException | IOException | InterruptedException e) {
      throw new RuntimeException("Failed to execute command", e);
    }
  }

  record PortAndContainer(int port, String mongoAlias, GenericContainer<?> container) {

  }
}
```

이렇게 구성하면된다&#x20;



여기서 다른건 다 필요없고 .withExposedPorts(port) 사용하면 보통 이 포트에 대한 host port를 자동으로 맵팽 해주게 되는데 이건 컨테이너 실행하고 나서 알 수 있어서  host 사용가능한 포트를 직접 지정 해줬다.



&#x20;그리고 withCreateContainerCmdModifier() 로 host port, container port 바인드 꼭 해줘야 한다. 그렇지 않으면 당연히 host에서 접속을 못 하겠지?



네트워크 설정은 기본 브릿지 모드일테니 mongoAlias 각 노드에 줘서 이걸로 컨테이너 끼리 통신 가능하다.



**MongoSetConfig.java**

```java
public enum MongoSetConfig {
  MONGO_ALIAS_PREFIX_AND_NUMBER_OF_REPLICAS("lucida-mongo", 3);

  private final String prefix;
  private final int numberOfReplicas;

  MongoSetConfig(String prefix, int numberOfReplicas) {
    this.prefix = prefix;
    this.numberOfReplicas = numberOfReplicas;
  }

  public String getPrefix() {
    return prefix;
  }

  public int getNumberOfReplicas() {
    return numberOfReplicas;
  }

  public String getMongoIndex(int index) {
    return prefix + (index + 1);
  }
}
```

테스트 환경 마다 hosts 파일을 수정 해야하니 번거롭다.

요렇게 enum으로 mongoAlias 관리해주고&#x20;



이렇게만 하면 뭐 거의 끝났다 볼 수 있지만..getConnectionString() url 로 접속해보면 안된다.&#x20;

아마도 보통 localhsot:port1,localhsot:port2,localhsot:port3 이렇게 접속 하면 되긴 하는데 mongodb set url 일 경우 서로 도메인을 다 알아야 한다.. host 쪽에서도 mongoAlias 에대한 ip 정보 알아야 접속이 가능하다



리눅스 기준 /etc/hosts 에 127.0.0.1 mongoAlias1 mongoAlias2 mongoAlias3 이렇게 넣어주면 되긴하는데 OS 별로 hosts 파일을 수정하기 번거롭다.

그래서 이를 자바에서 해결 가능하다

JAVA 실행 시 도메인 이름에대한 ip 정보를 집어 넣을 수 있다. [InetAddressResolver](https://docs.oracle.com/en/java/javase/19/docs/api/java.base/java/net/spi/InetAddressResolver.html) 를 구현해주면 된다.

하지만... 이 인터페이스는 java 18 부터 사용 할 수 있고 나의 프로젝트는 17이라 없다



하지만 이 mongodb 드라이버는 이럴줄 알고 java 낮은 버전에서 사용 할 수 있게 해둔게 있더라고 역시 천조국 개발자👍



**LocalHostMongoResolver.java**

```java
import com.mongodb.spi.dns.InetAddressResolver; // 갬동 ㅎㅎㅎ
import java.net.InetAddress;
import java.net.UnknownHostException;
import java.util.Arrays;
import java.util.Collections;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import org.jetbrains.annotations.NotNull;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class LocalHostMongoResolver implements InetAddressResolver {

  private static final Logger LOG = LoggerFactory.getLogger(LocalHostMongoResolver.class);

  private static final String LOCAL_IP = "127.0.0.1";
  private final Map<String, String> customDnsMapping = new HashMap<>();

  public LocalHostMongoResolver() {
    MongoSetConfig mongoSetConfig = MongoSetConfig.MONGO_ALIAS_PREFIX_AND_NUMBER_OF_REPLICAS;
    int numberOfReplicas = mongoSetConfig.getNumberOfReplicas();
    for (int i = 0; i < numberOfReplicas; i++) {
      customDnsMapping.put(mongoSetConfig.getMongoIndex(i), LOCAL_IP);
    }
    LOG.info("LocalHostMongoResolver: {}", customDnsMapping);
  }

  @Override
  public @NotNull List<InetAddress> lookupByName(@NotNull String host)
    throws UnknownHostException {
    if (customDnsMapping.containsKey(host)) {
      return Collections.singletonList(InetAddress.getByName(customDnsMapping.get(host)));
    }
    return Arrays.asList(InetAddress.getAllByName(host));
  }
}
```

com.mongodb.spi.dns.InetAddressResolver 이건 진짜 우연하게 찾았는데 hosts 파일 수정 없이 어떻게 ip 등록할까 고민하다가  이친구들 github 들어가서 심심해서 인터페이스 이름으로 검색을 했는데 있었다 👊



마지막으로 이걸 어떻게 등록 하냐면 InetAddressResolverProvider 이거 까지 구현해서 resources/META-INF/services/com.mongodb.spi.dns.InetAddressResolverProvider 파일 만들고&#x20;

내가 작성한 프로바이더 클래스 이름을 적어주면 등록된다.

**CustomInetAddressResolverProvider.java**

```java
public class CustomInetAddressResolverProvider implements InetAddressResolverProvider {

  @Override
  public @NotNull InetAddressResolver create() {
    return new LocalHostMongoResolver();
  }
}
```

**resources/META-INF/services/com.mongodb.spi.dns.InetAddressResolverProvider**&#x20;

```java
com.nkia.lucida.builder.mongodb.CustomInetAddressResolverProvider
```



**사용법**

```javascript
@ExtendWith(SpringExtension.class)
@ContextConfiguration(classes = {MongoTestApplication.class})
public abstract class MongoReplicaSetManualDynamicPropertiesTest {

  protected static final MongoDBReplicaSetTestContainer mongoDBReplicaSet =
    new MongoDBReplicaSetTestContainer("rs0", "mongo:7.0.9");
  protected static final Logger LOG = LoggerFactory.getLogger(
    MongoReplicaSetManualDynamicPropertiesTest.class);

  @DynamicPropertySource
  static void setProperties(DynamicPropertyRegistry registry) {
    mongoDBReplicaSet.start();
    String replicaSetUri = mongoDBReplicaSet.getConnectionString();
    registry.add("spring.data.mongodb.uri", () -> replicaSetUri);
    registry.add("spring.data.mongodb.database", () -> "test");
    LOG.info("MongoDB Replica Set URI: {}", replicaSetUri);
  }
}

```

```java
  @Test
  void saveEndReadTest() {
    String databaseName = "test-database-1";
    testService.save(databaseName);

    // then
    org.bson.Document explainResult1 = testService.findEntity1(databaseName);
    LOG.info("Test entity 1: {}", explainResult1);

    org.bson.Document explainResult2 = testService.findEntity2(databaseName);
    LOG.info("Test entity 2: {}", explainResult2);

    String serverHost1 = explainResult1.get("serverInfo", org.bson.Document.class).getString("host");
    int serverPort1 = explainResult1.get("serverInfo", org.bson.Document.class).getInteger("port");

    String serverHost2 = explainResult2.get("serverInfo", org.bson.Document.class).getString("host");
    int serverPort2 = explainResult2.get("serverInfo", org.bson.Document.class).getInteger("port");

    LOG.info("Test Entity 1 실행된 노드: {}:{}", serverHost1, serverPort1);
    LOG.info("Test Entity 2 실행된 노드: {}:{}", serverHost2, serverPort2);

    Map<Integer, ServerType> serverTypeMap = mongoClient.getClusterDescription().getServerDescriptions().stream()
      .collect(Collectors.toMap(serverDescription -> serverDescription.getAddress().getPort(),
        ServerDescription::getType));

    assertEquals(ServerType.REPLICA_SET_SECONDARY, serverTypeMap.get(serverPort1));
    assertEquals(ServerType.REPLICA_SET_SECONDARY, serverTypeMap.get(serverPort2));
  }
```

테스트 돌려보면 읽기는 REPLICA\_SET\_SECONDARY 에서 읽게되는데 의도 한 대로 잘 동작한다

<figure><img src="../.gitbook/assets/image (13).png" alt=""><figcaption></figcaption></figure>

크으으으으 굿\~

이제 이코드 베이스로 여러 상황에 맞는 테스트 코드를 작성 해볼수 있다👍
