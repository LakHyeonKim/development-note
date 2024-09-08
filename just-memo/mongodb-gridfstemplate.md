# 📒 Mongodb GridFsTemplate 사용기

## GridFsTemplate 이란





## Spring boot 3.1.1 -> 3.2.2 버전 주의 사항



해당 솔루션은 조직 개념이 있어 조직별 mongodb 데이터베이스가 분리되는 구성을 가지고 있다.

ThreadLocal을 이용하여 조직별 데이터베이스로 가야하는 처리가&#x20;

업데이트 이후로 되지 않던 이유가 있었다.



### Mongodb 4.2.2 업데이트 시 GridFsTemplate 사용 이슈



Spring boot 3.2.2 버전으로 업데이트로 하게되면 spring-data-mongodb 4.2.2 버전을 의존하게 됨

해당 버전(4.2.2)이 MongoTemplate에서는 문제가 없지만 GridFsTemplate 인스턴스 초기화 방식이 Laze + 싱글톤으로 변경이 됨에 따라 요청 중 mongodb database 변경이 되지 않는다.



* 프로젝트 코드 일부분 (getTenantDatabase 호출을 하여 조직 데이터 베이스 이름을 가져오고 mongo client 객체에서 데이터베이스 객체를 얻어오는 로직)

<figure><img src="../.gitbook/assets/image (1) (1) (1) (1).png" alt=""><figcaption></figcaption></figure>

### 관련 코드

#### **기존 4.1.1 버전 GridFsTemplate**

* 생성자

<figure><img src="../.gitbook/assets/image (2) (1) (1).png" alt=""><figcaption></figcaption></figure>

* getGridFs()

<figure><img src="../.gitbook/assets/image (3) (1).png" alt=""><figcaption></figcaption></figure>



#### 4.2.2 버전 GridFsTemplate

* 생성자

<figure><img src="../.gitbook/assets/image (4).png" alt=""><figcaption></figcaption></figure>

* getGridFs()

<figure><img src="../.gitbook/assets/image (5).png" alt=""><figcaption></figcaption></figure>

* org.springframework.data.util.Lazy.of()

생성자 부분에서 Supplier function interface를 재정의 한 Lazy 클래스로 get()을 호출하는 부분에 기존 인스턴스가 최초 로딩되면 호출하지 않게 됨

* get() -> getNullable()&#x20;

<figure><img src="../.gitbook/assets/image (6).png" alt=""><figcaption></figcaption></figure>

### 해결방법

* Lazy 클래스를 제거하여 빈을 새롭게 정의 해서 등록

```java

@Bean("builderGridFsTemplate")
  public GridFsTemplate builderGridFsTemplate(@Autowired MappingMongoConverter mongoConverter) {
    return new GridFsTemplate(mongoConverter, () -> {
      MongoDatabaseFactory dbFactory = mongoDbFactory();
      Assert.notNull(dbFactory, "MongoDatabaseFactory must not be null");
      MongoDatabase db = dbFactory.getMongoDatabase();
      return GridFSBuckets.create(db, BUILDER_GRID_FS_BUCKET);
    });
  }
```

