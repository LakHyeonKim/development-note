# 📒 spring.main.allow-bean-definition-overriding=true 로 인한 교훈

최근 프로젝트에서 Mongodb 관련 설정을 하고 Bean으로 등록하던 도중 만났던 문제 였다.



해당 설정은 mongodb 도큐먼트에 대한 커스텀 컨버터를 등록하고, local 테스트 할때 의도한 방향으로 전부 동작을 확인 하였고 쿠버네티스에 배포하고 퇴근을 하였다.



다음날 슬랙알림을 보니 수차례 내가 배포한 모듈이 재시작을 하고 있었다.

바로 들어가 로그를 확인하니 mongodb에서 해당 필드의 데이터가 정상적으로 컨버팅이 되고 있지 않았다.



아니 어제 분명 잘되었는데...

다시 로컬 테스트코드를 돌려보니 잘된다..



원인을 찾던 도중 내가 올린 빈 이름과, mongodb AutoConfiguration으로 인하여 기본 Bean들 중 이름이 같은게 있는 것을 발견하였다. &#x20;

* `MongoConfigurationSupport` 클래스 일부분 `customConversions` Bean이 이름이 같았다.

```java
/**
 * Base class for Spring Data MongoDB to be extended for JavaConfiguration usage.
 *
 * @author Mark Paluch
 * @since 2.0
 */
public abstract class MongoConfigurationSupport {
...

     /**
	 * Register custom {@link Converter}s in a {@link CustomConversions} object if required. These
	 * {@link CustomConversions} will be registered with the
	 * {@link org.springframework.data.mongodb.core.convert.MappingMongoConverter} and {@link MongoMappingContext}.
	 * Returns an empty {@link MongoCustomConversions} instance by default.
	 * <p>
	 * <strong>NOTE:</strong> Use {@link #configureConverters(MongoConverterConfigurationAdapter)} to configure MongoDB
	 * native simple types and register custom {@link Converter converters}.
	 *
	 * @return must not be {@literal null}.
	 */
	@Bean
	public MongoCustomConversions customConversions() {
		return MongoCustomConversions.create(this::configureConverters);
	}

...
}
```



같은건 같은건데... 발견을 하지 못 한 이유가 아래 옵션 때문이다.

```yaml
spring.main.allow-bean-definition-overriding=true
```

해당 옵션은 bean 이름이 충돌되었을때 빈 정의를 변경하지 않고 빈 오버라이딩을 허용한다는 옵션이다.



즉 AutoConfiguration으로 인해 올라가는 빈과, 내가 정의 한 빈 이름이 같아 둘중 하나가 오버라이딩 된 것 이였다.

아래 사이트 참고

{% embed url="https://www.baeldung.com/spring-boot-bean-definition-override-exception" %}

위 링크 6.4 부분을 보면 써드 파티 라이브러이에서 bean 충돌시 이 옵셥으로 해결 할 수 있다고 되어있지만

빈이 생성되는 순서가 런타임시 종속관계 의해 로드 되게 때문에 무엇이 우선으로 생성되고 오버라이드 될지 예측이 어렵다고 한다.

<figure><img src=".gitbook/assets/image (6).png" alt=""><figcaption></figcaption></figure>

간단한 하게 현상을 만들어보자



**AbstractTest.class**

```java
@Slf4j
@Configuration(proxyBeanMethods = false)
public abstract class AbstractTest {
    
    @Bean
    public User userBean(){
        log.info("userBean");
        return new User();
    }

    @Bean
    public User1 userBean1(){
        log.info("userBean1");
        return new User1();
    }
    
    protected User3 userBean3() {
        return null;
    }
    
}
```



**Test\_A.class**

```java
@Slf4j
@Configuration
public class Test_A extends AbstractTest {
    
    @Override
    public User userBean() {
        log.info("override userBean");
        return new User();
    } 
    
}
```



**Test\_B.class**

```java
@Slf4j
@Configuration
public class Test_B {
    @Bean
    public User1 userBean1(){
        log.info("userBean1C");
        return new User1();
    }
}

```

위 예제는 AbstractTest.class 추상클래스 내부의 빈을 재정의 하는 방식 이다.

​

Test\_A.class 는 보통 일반적으로 오버라이드 하는 방식이고

Test\_B.class 는 내가 했던 방식이다.

​

실행을 해보면

<figure><img src=".gitbook/assets/image (7).png" alt=""><figcaption></figcaption></figure>

"userBean1C" 잘나오지만

Test\_B.class 이 클래스 이름을 Test\_A 이름보다 빠른 순서로 바꾸면 (Test\_B.class -> ATest\_B.class)

<figure><img src=".gitbook/assets/image (8).png" alt=""><figcaption></figcaption></figure>

AbstractTest.class 에 존재하는 빈이 나중에 뜨게된다.



종속관계가 단순 클래스명도 영향을 미친다고 하면 Bean 오버라이드 설정을 가능하게 두는 것은 확실한 경우가 아니면 하지 않는 방향이 좋을 것 같다.



