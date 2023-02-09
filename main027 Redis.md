# MAIN027 REDIS 정리

***

## 들어가기 전에

* 프로젝트에서 제가 맡은 파트의 세부적인 내용까지 타 팀원에게 공유가 될 수 있도록 메뉴얼 수준의 자세한 설명서를 만들었습니다. 
  * 프로젝트 중 인원에 공백이 생겨 자리를 메워야 하는 상황이 발생했을 때, 프로젝트가 끝나고 타 팀원이 제 코드를 공부할 때 도움을 주기 위한 목적입니다.
* 이 글에서는 main027(에코그린 서울) 팀에서 프로젝트에 적용한 Redis에 대한 개념 및 관련 코드(토큰 관리 & 캐시 서버 구현)를 설명하는 내용이 주를 이룹니다.
* 프로젝트 중 생긴 에러의 핸들링 및 이슈에 대한 개인적인 생각 등도 함께 적혀있습니다.
<br></br>


## 목차

1. [What is a Redis?](#what-is-a-redis)
2. [Redis를 활용한 Token 기반 로그인/로그아웃 구현](#redis를-활용한-token-기반-로그인로그아웃-구현)
3. [캐시서버로서의 Redis](#캐시서버로서의-redis)
4. [캐시 서버 구현하기](#캐시-서버-구현하기)
5. [Redis 회고](#redis-회고)
***
<br></br>
<br></br>



## What is a Redis?

* Key, Value 구조의 비정형 데이터를 저장하고 관리하는 NOSQL DBMS이다.(비관계형 데이터베이스 관리 시스템)
* 오픈 소스이므로 무료로 사용이 가능하다.
* 인메모리 데이터 구조이면서 다양한 데이터 구조체를 지원한다. 
* 일반 데이터베이스처럼 디스크에 데이터를 쓰는 구조가 아닌 메모리에서 데이터를 처리하기 때문에 속도가 굉장히 빠르다.
  * 위와 같은 장점으로 DB, 캐시서버, 메시지 큐, Shared Memory 등의 용도로 사용된다.
    * 데이터의 빠른 전송이 필요할 때 사용
* 샤딩(Sharding)을 지원하여 쓰기 성능을 향상시킬 수 있다.
* 메모리 기반이지만 메모리에 있는 데이터를 디스크에 저장하는 방식을 통해 영속적인 데이터 보존이 가능하다.(Redis Persistence)
  * 데이터를 저장하는 방식에는 RDB(snapshotting), AOF(Append only file) 두가지가 있다.
* 싱글 스레드를 지원하므로 안정적인 인프라 구축을 위해 Replication(Master-Slave 구조)를 사용한다.
<br></br>


### Redis VS Memcached

* **공통점**
  * 인메모리 기반이므로 디스크 기반의 일반 데이터베이스에 비해 압도적으로 빠르다.
  * Key - Value의 저장 방식이다.
  * 문법적으로 사용하기 쉬우며 개발에 필요한 코드 양도 적다.
  * 파티셔닝, 샤딩 등을 통해 데이터의 분산 저장이 가능하다.
  * Java, Python, C, JavaScript 등 다양한 프로그래밍 언어를 지원한다.
<br></br>


* **Memcached의 특징**
  * 멀티 스레드를 지원하여 멀티 프로세스 코어를 사용할 수 있다.
    * 이를 이용하여 스케일 업을 통해 보다 많은 작업을 처리할 수 있다.
  * String 타입으로 저장한다.
  * 데이터 복제가 불가능하다.
  * 데이터를 저장할 수 없기 때문에 캐시서버로 밖에 활용할 수 없다.
<br></br>


* **Memcahced와 비교했을 때 레디스의 장점**
  * 레디스가 훨씬 빠르다.
  * 다양한 자료 구조를 활용할 수 있다.(String, Hash, Set, List 등) 
  * 하나의 Key에 저장할 수 있는 Value의 크기가 크다.
  * Master - Slave 구조를 이용하여 클러스터를 만들 수 있다.
  * 디스크에 데이터를 기록하기 때문에 데이터 복구가 가능하다.
<br></br>


* **왜 Redis를 썼는가?**
  * 팀원들과 회의를 통해 Redis와 Memcached 중 어느 것을 사용할 지에 대해 논의했다.
  * Redis의 장점이 명확했기 때문에 결론을 빠르게 도출할 수 있었다.
    * Main027 프로젝트 수준에서는 Memcached의 유일한 장점이라고 할 수 있는 멀티 스레드를 쓸 필요가 없었다.
    * 트래픽이 멀티 스레드를 써야할 만큼 발생하지 않기 때문에 멀티 스레드를 제외한 대부분의 기능에서 보다 좋은 성능을 가지고 있는 Redis를 사용하기로 결정했다.
<br></br>
<br></br>


## Main027에서 Redis가 하는 일

1. 로그인 시 생성되는 RefreshToken을 저장하는 DB 역할을 한다.
 * Redis에 RefreshToken을 저장하면 만료 기한도 함께 저장이 되는데 기한이 만료되면 자동 삭제된다.
   * Redis에 expiration 기한이 되면 자동 삭제되는 기능이 있음을 테스트로 확인했다. 


2. 로그아웃 후에 만료 기한이 남아있는 AccessToken을 저장하여 재로그인을 못 하게 막는 역할을 한다.
 * 로그아웃을 하게 되면 요청으로 받은 AccessToken을 이용해 DB의 RefreshToken을 삭제하고, AccessToken을 blackList라는 이름으로 대신 넣는다.
   * 이는 RefreshToken이 DB에서 삭제되는 시점에 AccessToken의 기한이 남아있다면 재로그인이 가능하기 때문에 BlackList에 저장해두었다가 로그아웃된 AccessToken이 요청으로 들어오면 BlackList의 AccessToken과 비교 후에 로그인을 거부하기 위한 것이다

3. 캐시 서버 역할을 한다.
 * api 요청이 많은 일부 메서드를 선정해 캐시 서버에 저장하여 응답 속도를 높였다.
 * Grafana를 통해 확인한 api 사용량을 토대로 캐시 서버를 활용할 메서드를 선정했다. 캐시 서버를 활용할 메서드는 다음과 같다.
   * ReviewController - getReviews
   * PlaceController - getplace
   * PlaceController - getPlaces
<br></br>
<br></br>

### AccessToken과 RefreshToken 짚고 넘어가기

#### AccessToken

* 로그인 시 발급 받음
* 접근 권한이 필요한 요청 시 자격 증명으로 사용
* 만료 기한이 있음
* 서버에서 관리를 하지 않음
<br></br>

#### RefreshToken

* 로그인 시 발급 받음
* AccessToken의 만료 기한이 다 되어 더 이상 사용할 수 없게 되었을 때 AccessToken의 재발급 용도로 사용
* 만료 기한이 있음(AccessToken보다 길게 설정함)
* 서버 DB에서 관리함
* 메인프로젝트에서는 일반적인 DB 대신에 인메모리 DB인 Redis를 사용
<br></br>


#### 흐름

1. 클라이언트는 로그인을 통해 AccessToken과 RefreshToken을 발급받는다.

2. 접근 권한이 필요한 요청에는 요청 헤더(Request Header)에 AccessToken만 담아 보낸다.
 * 서버에서는 AccessToken을 검증하고 요청 수행 후 응답을 보냄
3. AccessToken을 재발급받을 때는 RefreshToken만 요청 헤더에 담아 보낸다.
 
4. 로그아웃을 할 때는 AccessToken과 RefreshToken을 둘 다 요청 헤더에 담아 보낸다.
 * 요청 헤더의 AccessToken은 BlackList에 넣는 용도로, RefreshToken은 redis에 저장된 RefreshToken을 삭제할 때 사용된다.
<br></br>


#### 경우의 수

1. AccessToken이 만료되고 RefreshToken이 만료되지 않았을 때
 * RefreshToken으로 AccessToken을 재발급받는다.

2. AccessToken, RefreshToken이 모두 만료되었을 때
 * 재로그인을 통해 둘 다 재발급받는다.

3. RefreshToken이 삭제된 상태에서 AccessToken이 만료되지 않았을 때(로그아웃 후 기한이 남이있는 AccessToken으로 로그인 시도)
 * DB에 저장되어있는 BlackList와 비교를 통해 걸러지게 된다.(로그인 거부)
<br></br>


#### 질문

1. 요청마다 헤더에 AccessToken이 포함되는지 RefreshToken이 포함되는지 헷갈리니 그냥 모든 요청에 AccessToken과 RefreshToken을 다 보내면 안되나?

 * 모든 요청에 RefreshToken을 보내면 Redis를 쓸 필요가 없다.(DB에서 토큰을 관리할 필요가 없다) 대신 RefreshToken을 매번 클라이언트에게 받아서 필요할 때 AccessToken을 재발급하게 된다. 이는 DB를 사용하지 않기 때문에 성능이 향상되는 장점이 있지만, RefreshToken이 외부에 지속적으로 노출이 되기 때문에 보안에 매우 치명적이다. 다르게 이야기하면 성능을 위해 보안을 포기한 것과 같기 때문에 권장하지 않는다.
<br></br>
<br></br>



## Redis를 활용한 Token 기반 로그인/로그아웃 구현

### RedisConfiguration

```java
@RequiredArgsConstructor
@Configuration
@EnableRedisRepositories
public class RedisConfiguration {
  
  ...
    
  @Bean
    public RedisConnectionFactory redisConnectionFactory() {
        return new LettuceConnectionFactory(redisProperties.getHost(),
                                            redisProperties.getPort());
    }

    @Bean
    public RedisTemplate<String, String> redisTemplate() {
        RedisTemplate<String, String> redisTemplate= new RedisTemplate<>();
        redisTemplate.setKeySerializer(new StringRedisSerializer());
        redisTemplate.setValueSerializer(new StringRedisSerializer());
        redisTemplate.setConnectionFactory(redisConnectionFactory());
        return redisTemplate;
    }
}
```

* `RedisConnectionFactory` : redis와 connection을 생성해주는 객체, 이 메서드를 통해 Redis 서버와 연결이 가능하다.
* `RedisTemplate` : redis 서버와 통신을 처리하고, 사용자로 하여금 redis 모듈을 사용하기 쉽도록 다양한 기능을 제공한다.
* ```redisTemplate.setKeySerializer(new StringRedisSerializer());```, ```redisTemplate.setValueSerializer(new StringRedisSerializer());``` Redis에 저장할 때 올바른 값으로 저장하기 위해 넣었다. 이게 없으면 /00/x0 같은 문자가 함께 저장된다.



### RedisService

```java
@Service
@RequiredArgsConstructor
public class RedisService {
    private final RedisTemplate<String, String> redisTemplate;

    public void setRefreshToken(String key, String value, long expirationMinutes) {
        if (key.startsWith("Bearer")) throw new TokenException(ExceptionCode.INVALID_REFRESH_TOKEN);
        ValueOperations<String, String> values = redisTemplate.opsForValue();
        values.set(key, value, Duration.ofMinutes(expirationMinutes));
    }

    public String getToken(String key) {
        ValueOperations<String, String> values = redisTemplate.opsForValue();
        return values.get(key);
    }

    public void setBlackList(String key, String value, long expirationMinutes) {
        if (!key.startsWith("Bearer")) throw new TokenException(ExceptionCode.INVALID_ACCESS_TOKEN);
        ValueOperations<String, String> values = redisTemplate.opsForValue();
        values.set(key, value, Duration.ofMinutes(expirationMinutes));
    }

    public void deleteRefreshToken(String key) {
        redisTemplate.delete(key);
    }
}
```

* `setRefreshToken` : RefreshToken을 redis에 저장
  * ```if (key.startsWith("Bearer")) ``` AccessToken에는 Bearer라는 토큰 타입을 붙여 AccessToken과 RefreshToken을 구분함
    * Bearer로 시작한다면 AccessToken이라는 의미이므로 에러 던짐
  * key : RefreshToken
  * value : 이메일
  * expirationMinutes : 만료 기한(다 되면 자동으로 삭제)
* `getToken` : key로 email(value) 가져옴
* `setBlackList` : 기한이 남아있는 AccessToken을 BlackList로 redis에 저장
  * ```if (!key.startsWith("Bearer")) ``` Bearer로 시작하지 않는다면 RefreshToken이라는 의미이므로 에러 던짐
  * key : AccessToken
  * value : 이메일
  * expirationMinutes : 만료 기한(다 되면 자동으로 삭제)
* `deleteRefreshToken` : RefreshToken 삭제
<br></br>
<br></br>



## 캐시서버로서의 Redis

### Why is Single-threaded Redis So fast? (멘토님이 보내주신 Redis 영상 정리, 의역 다수 포함)



* **Redis is rock solid. easy to use and fast.**(Redis는 매우 견고하며, 사용하기 쉽고 빠르다.)
* 이는 왜 Redis가 매년 스택오버플로우에서 유저 투표로 선정하는 데이터베이스 순위에서 가장 사랑받는 데이터베이스 서비스인지를 설명한다.
* Redis가 빠른 이유 3 가지

  * (1) RAM을 사용하는 인메모리 데이터베이스이기 때문이다.

    * **Memory access is several orders of magnitude faster than Random disk.** (I/O 메모리 접근이 일반적인 디스크 입출력에 비해 훨씬 빠르다.)

    * **Pure memory access provides high read and write throuput and low latency.**(순수한 메모리 접근은 높은 입출력 처리율과 낮은 지연시간을 제공한다.)

    * **The trade-off is that dataset cannot be larger than memory.**(단점으로는 데이터셋이 일반적인 메모리보다 크지 않다.)

    * 인메모리 데이터베이스 구조는 디스크보다 구성하기 쉽다. 이는 Redis의 견고성을 유지하는 데에 도움을 준다.
  * (2) 싱글 스레드이기 때문이다. 왜 싱글 스레드가 좋은 성능을 보여주는가?
    * **Multi-threaded applications require locks or other synchronization mechanisms. They are notoriously hard to reason about.**(멀티 스레드는 동기화 메커니즘을 요구한다. 이는 단순히 추론하기 매우 어렵다.)
    * **In many applications, the added complexity is bug prone and sacrifices stability, making it difficult to justify the performance gain.**(일반적으로 애플리케이션에서 복잡성이 추가되면 버그가 일어나기 쉽고 견고함을 잃게 만드는데 이는 성능 향상을 어렵게 만든다.
    * 그러나 Redis의 경우 싱글 스레드이기 때문에 사용자가 이해하기 쉽다.(메커니즘이 복잡하지 않다.)
    * 어떻게 싱글 스레드가 수많은 요청과 응답을 동시에 수행할 수 있을까?
      * 이는 I/O multiplexing과 싱글 스레드의 조합 때문에 가능하다.
      * **With I/O multiflexing, the operating system allows a single thread to wait on many socket connections simultaneously.**(I/O multiplexing에서 동작 시스템은 동시적으로 연결되어있는 수많은 소켓 커넥션들 사이에서 싱글 스레드가 자신의 차례를 기다리도록 허락한다.
  * (3) Low-level 데이터 구조를 사용하기 떄문이다.
    * **It could leverage several efficient low-level data structures without worrying about how to persist then to disk efficiently - linked list skip list and hash table are some examples.**
      * 레디스는 키밸류 쌍을 그대로 저장한다. 그래서 키값만 알면 시간복잡도를 n(1)에 액세스 할 수 있다. 그저 객체화 된 로우 레벨의 데이터구조이자 인메모리이기 때문에 퍼포먼스가 잘 나온다.
      * 그러나 일반 DB의 경우 데이터를 디스크에 저장하기 때문에 디스크의 어디에 저장이 되어있는지 찾아야하고 그걸 꺼내오기 위한 i/o 연산을 빠르게 하기 위해서 복잡한 자료구조가 불가피해 상대적으로 느리다.
<br></br>
<br></br>



## 캐시 서버 구현하기

### ServerApplication

```java
@EnableCaching //추가
@EnableJpaAuditing 
@SpringBootApplication
@PropertySource("classpath:env.yml")
public class ServerApplication {
```

* 애플리케이션 실행 클래스에 `@EnableCaching` 애너테이션을 추가한다.



### RedisConfiguration

```java
@RequiredArgsConstructor
@Configuration
@EnableRedisRepositories
public class RedisConfiguration {
  
  ...
    
  @Bean
  public CacheManager cacheManager() {
        RedisCacheManager.RedisCacheManagerBuilder builder =
                RedisCacheManager.RedisCacheManagerBuilder.fromConnectionFactory(redisConnectionFactory());
        RedisCacheConfiguration configuration =
                RedisCacheConfiguration.defaultCacheConfig()
                                       .serializeKeysWith(RedisSerializationContext.SerializationPair.fromSerializer(
                                               new StringRedisSerializer()))
                                       .serializeValuesWith(RedisSerializationContext.SerializationPair.fromSerializer(
                                               new GenericJackson2JsonRedisSerializer(objectMapper())))
                                       .entryTtl(Duration.ofMinutes(30));

        builder.cacheDefaults(configuration);
        return builder.build();
    }
}

```

* cacheManager를 통해 캐시 정책을 설정한다.
* `serializeKeysWith(...)`: Key에 대한 직렬화 설정
* `serializeValuesWith(...)`: Value에 대한 직렬화 설정
* `entryTtl(Duration.ofMinutes(30))`: 캐시 저장 기간 설정(30분)
<br></br>
<br></br>


#### 에러 해결 과정

```java
.serializeValuesWith(RedisSerializationContext.SerializationPair.fromSerializer(
                                               new GenericJackson2JsonRedisSerializer()))
```

* 처음에 `objectMapper()`를 사용하지 않고 진행을 했더니, `SerializationException: Could not write JSON: Java 8 date/time type java.time.LocalDateTime not supported by default` 에러가 터졌다.
  * 이 에러는 Main027의 응답 객체이 들어있는 LocalDateTime과 관련된 필드 때문인데, 이 필드를 직렬화하기 위해서는 `objectMapper()` 메서드를 만들고 메서드 안에서 timeStamp에 대해 직렬화 작업을 해줘야하는 것 같다.



```java
public ObjectMapper objectMapper() {
        ObjectMapper mapper = new ObjectMapper();

        mapper.disable(SerializationFeature.WRITE_DATE_KEYS_AS_TIMESTAMPS);
        mapper.registerModules(new JavaTimeModule(), new Jdk8Module());

        return mapper;
    }
```

```java
.serializeValuesWith(RedisSerializationContext.SerializationPair.fromSerializer(
                                               new GenericJackson2JsonRedisSerializer(objectMapper())))
```

* `objectMapper()` 메서드 안에 timeStamp 관련 내용을 직렬화 해주는 코드가 들어갔다. 그러나 오류가 다른 것으로 바뀌었다.
* `Jackson: java.util.LinkedHashMap cannot be cast to X` 
  * 이 오류는 아무래도 Redis에 저장된 Value를 불러올 때 모종의 이유로 객체로 보는 것이 아닌 LinkedHashMap 타입으로 읽어들이는 것 때문에 발생하는 것 같다. 
    * 스프링 3.x 버전의 버그라고 하는데 LocalDateTime에 대한 역직렬화 문제인 것 같기도 해서 정확한 원인을 알 수 없다. 아무튼 스프링 4.0부터는 해결이 된 것 같다.



```java
    public ObjectMapper objectMapper() {
        PolymorphicTypeValidator ptv = BasicPolymorphicTypeValidator
                .builder()
                .allowIfSubType(Object.class)
                .build(); // 추가

        ObjectMapper mapper = new ObjectMapper();

        mapper.disable(SerializationFeature.WRITE_DATE_KEYS_AS_TIMESTAMPS);
        mapper.registerModules(new JavaTimeModule(), new Jdk8Module());
        mapper.activateDefaultTyping(ptv, ObjectMapper.DefaultTyping.NON_FINAL); // 추가

        return mapper;
    }
```

* 인터넷을 뒤져 위 코드를 찾아 Mapper에 추가해주니 해결되었고 저장된 캐시를 정상적으로 불러올 수 있었다.
  * 안타깝지만 이 부분은 어떻게 해결된건지 정확히 모른다. 아마도 불러온 Value 값을 LinkedHashmap이 아닌 객체로 인식하는 것 같다.
  * 매핑작업을 통해 직렬화와 역직렬화 시 객체로 읽어 들이게(?) 하는 작업을 통해서 해결할 수 있었다.
* 이렇게 RedisConfiguration을 구성하니 더이상 오류는 나타나지 않았다.
<br></br>
<br></br>


### 캐시 애너테이션

* 캐시 적용을 위해서는 캐시 애너테이션을 메서드나 클래스단에 붙이면 된다.



#### Main027의 placeController를 보며 이해 필요.

```java
@Cacheable(value = "place", key = "#placeId")
```

* 캐시를 적용할 클래스나 메서드에 붙임
* 캐시 서버에 데이터가 없으면 저장을 하고, 데이터가 있으면 캐시 데이터를 반환한다.
* Value : 저장될 캐시 이름
* Key : 이 값을 기준으로 캐시가 구분 됨



```java
@CacheEvict
```

* 이 애너테이션이 붙은 메서드가 실행되면 관련 캐시를 삭제함
* 캐시에 데이터를 저장한 후 데이터베이스 안의 데이터에 변화가 생기면, 캐시에 저장된 데이터와 데이터베이스에 저장된 데이터가 서로 불일치하는 정합성 문제가 발생할 수 있음 

* 예를 들어 데이터 베이스의 데이터를 삭제한 경우 캐시 서버 안에 있는 데이터도 삭제해야 함. 그래야 데이터가 일치하기 때문이다.

* 이때 @CacheEvict를 사용



```java
@CachePut
```

* 이 애너테이션이 붙은 메서드가 실행되면 실행 결과를 캐시에 저장함
* 이때 기존에 저장된 캐시 데이터와 관계없이 진행함
* 이를 활용하여 저장된 캐시 데이터에 변경된 값을 반영해야 할 때 사용함(정합성 문제 해결)
* main027에서는 @CachePut을 사용하지 않고 @CacheEvict를 사용했음
  * 캐시 데이터에 변경, 삭제 등의 이슈가 있을 때는 관련 캐시 데이터를 전부 삭제하는 방식을 택함
    * 아직 캐시 관련해서 학습이 부족하다고 판단하여 짧은 프로젝트 기간에 에러를 최대한 줄이기 위해서 전부 삭제하도록 구성했다.
<br></br>
<br></br>



## Redis 회고

 데이터베이스에 대해 코드스테이츠 코스에서 배운 건 H2 인메모리 DB를 이용한 로컬에서의 서버 구현과, Mysql을 활용한 데이터베이스 연결 정도였다. 정보처리기사 공부를 통해 캐시와 캐시 서버에 대한 개념은 알고 있었지만 **데이터베이스라는 학문에 대해 제대로 공부하지 않았기 때문에 팀원들과 스터디를 통해 데이터베이스에 대해 공부했다.**([블로그 정리](https://suzuworld.tistory.com/category/%EB%84%93%EA%B3%A0%20%EC%96%95%EC%9D%80%20%EB%8D%B0%EC%9D%B4%ED%84%B0%EB%B2%A0%EC%9D%B4%EC%8A%A4%20%EC%A7%80%EC%8B%9D)) 



 JWT를 이용한 로그인 구현 이후 팀원들과 RefreshToken을 어떻게 관리할 것인가에 대해 이야기를 나눴다. 우리 팀은 토큰 관리를 위해서 인메모리 DB를 이용해보자는 의견을 모았다. **속도가 더 빠른 것도 있고, 토큰과 같은 중요 정보는 별도의 데이터베이스에 저장하는게 더 안전하다고 생각했다.** 팀원들의 만장일치로 Redis를 활용한 토큰 관리 구현을 시작했다. 그러나, Reids를 자유자재로 사용하기에는 학습 시간이 턱없이 부족했다. **온갖 블로그와 스택오버플로우를 찾아 이것저것 시도해보며 겨우 완성했다.** 특히 Redis에 저장될 때 바이트 코드로 저장되는 것 때문에 초반에 굉장히 애를 먹었다. 이를 해결하기 위해서는 String 형태로 직렬화를 적용하는 코드를 삽입했다. 또 다른 문제는 캐시 서버를 구현할 때 모종의 이유로 Redis에서 데이터를 불러올 때 Value 값을 객체로 인식하는 것이 아닌 LinkedHashMap 타입으로 인식하는 것이었다. Redis를 열어 확인해보니 저장은 스트링 형태로 잘 저장된 것으로 보아 불러올 때 문제임이 확실해 보였다. **매핑 작업을 통해 직렬화, 역직렬화를 하는 작업을 거치도록 하니 문제가 해결되었다.** 



 시간이 부족해 이런 식으로 구현할 수 밖에 없어 아쉬웠지만, **결과적으로 캐시 서버 도입을 통해 유의미한 성능 개선을 눈으로 확인할 수 있어서 학습에 큰 도움이 되었다.** 데이터베이스에 대해 공부하면서 알게 된 내용으로 클러스터를 만들어 Replication을 적용하여 캐시 서버의 안정성을 높이는 등의 작업도 해보고 싶었지만 프로젝트 규모 상, 그리고 시간 관계 상 시도해 볼 수 없어 아쉬웠다. **내게 Redis를 활용한 토큰 관리와 캐시 서버 구현은 대규모 서비스를 지탱하기 위해 데이터베이스 설계가 얼마나 중요한 지 알게 된 소중한 경험이었다.**
