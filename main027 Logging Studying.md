# MAIN027 Logging 정리

***

## 들어가기 전에

* 저희 팀은 각자 맡은 파트의 세부적인 내용까지 타 팀원에게 공유가 될 수 있도록 메뉴얼 수준의 자세한 설명서를 만들었습니다. 
  * 프로젝트 중 인원에 공백이 생겨 자리를 메워야 하는 상황이 발생했을 때, 프로젝트가 끝나고 타 팀원이 제 코드를 공부할 때 도움을 주기 위한 목적입니다.
* 이 글에서는 main027(에코그린 서울) 팀에서 타 팀원이 프로젝트에 적용한 Logging을 분석하는 내용이 주를 이룹니다.
* 학습을 하면서 든 개인적인 생각 등도 함께 적혀있습니다.
<br></br>


## 목차

1. [main027에서의 로깅은 스프링 AOP를 이용하여 만들어졌다.](#main027에서의-로깅은-스프링-aop를-이용하여-만들어졌다)
2. [AOP 관련 코드 분석](#aop-관련-코드-분석)
3. [Logging 회고](#logging-회고)
***
<br></br>
<br></br>



## main027에서의 로깅은 스프링 AOP를 이용하여 만들어졌다.

### AOP(Aspect Oriented Programming)란?

 AOP는 관점 지향 프로그래밍이라고 불린다. 관점 지향 프로그래밍은 어떤 로직을 핵심 기능과 부가 기능으로 나누고 그걸 기준으로 각각 모듈화한다. main027에서 **핵심 기능**은 핵심 비즈니스 로직을 생각하면 된다. **부가 기능**은 비즈니스 로직을 수행하는 데 필요한 로깅, 보안, 트랜잭션 등 공통적으로 적용하는 기능을 뜻한다.
<br></br>


### 우리는 왜 로깅을 적용했나?

 로깅의 중요성은 백 번 강조해도 모자라다. 로그를 남기는 건 백엔드 개발자에게 굉장히 중요하다고 배웠다. 이유는 간단하다. 비즈니스 로직이 돌아가는 것을 기록으로 남김으로써 이후의 유지보수, 에러 수정 작업 등에 시간을 단축시키고, 성능 개선 작업에 있어 효율적이면서 올바른 선택을 할 수 있도록 도와주기 때문이다. 

 백엔드팀은 기본 기능 구현이 끝나고 성능 최적화를 통해 프로젝트의 완성도를 높이고자 했다. 성능 최적화를 모든 로직에 적용하면 좋겠지만 시간 관계상 그건 불가능했다. 따라서 로깅을 통해 API 요청 비율과 API 응답 속도를 남겨 이를 활용하여 우선적으로 적용할 메서드를 찾으려고 했다. 백엔드 팀원 중에 AOP를 이용하여 로깅을 남겨 본 경험이 없어 시간 관계상 한 명이 학습 후 구현을 하고 이후에 스터디를 통해 학습 및 프로젝트에 적용한 내용을 공유하는 식으로 진행했다.
<br></br>
<br></br>




## AOP 관련 코드 분석

### @TimeTrace 애너태이션 생성

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface TimeTrace {

    /**
     *  Aspect에서 log.WARN을 표시할 시간 기준 ms
     */
    int millis() default 50;

}
```

- `@Target(ElementType.METHOD)`: 사용자가 생성한 애너테이션이 적용될 타입을 메서드로 한다.
- `@Retention(RetentionPolicy.RUNTIME)`: 생성한 애너테이션의 라이프 사이클을 런타임이 끝낼 때까지 유지한다.
- `int millis() default 50`: `TimeTraceAspect` 클래스의 `doLogTime` 메서드에서 사용.
  - 메서드 소요시간이 defalut로 설정된 50보다 길면 log.warn을 띄움
  - `@TimeTrace(millis = 100)` 이런 식으로 메서드마다 설정 시간을 바꿀 수도 있음
<br></br>


### TimeTraceAspect 클래스

```java
@Slf4j
@Aspect
@Component
public class TimeTraceAspect {

    private final DataHolder dataHolder;

    public TimeTraceAspect(DataHolder dataHolder) {
        this.dataHolder = dataHolder;
    }

    @Around("@annotation(timeTrace)")
    public Object doTimeTrace(ProceedingJoinPoint joinPoint, TimeTrace timeTrace) throws Throwable {

        String name = joinPoint.getSignature().toShortString().split("\\(")[0] + "()";
        String uri = dataHolder.getUri();
        int millis = timeTrace.millis();
        final int limitMillis = 300;

        dataHolder.stop();

        doLogTime("-->", uri, name, dataHolder.getLastTaskTimeMillis(), millis, limitMillis);

        dataHolder.start();

        Object result = joinPoint.proceed();

        dataHolder.stop();

        doLogTime("<--", uri, name, dataHolder.getLastTaskTimeMillis(), millis, limitMillis);

        dataHolder.start();

        return result;
    }

    private void doLogTime(String arrow, String uri, String name, long lastStopWatchTime,
                           int millis, int limitMillis) {
        if (lastStopWatchTime <= millis) {
            log.info(arrow + "[{}][{}ms][{}]", uri, lastStopWatchTime, name);
        } else if (lastStopWatchTime <= limitMillis) {
            log.warn(arrow + "[{}][{}ms][{}]", uri, lastStopWatchTime, name);
        } else log.error(arrow + "[{}][{}ms][{}]", uri, lastStopWatchTime, name);
    }
}
```

**빠르게 개념을 정리해보자.**

* Pointcut : 공통 사항을 적용할 위치 / Advice : 어떤 공통 사항인가?(어떻게 처리, 어떻게 조작할 것인가)
* Aspect는 여러 개의 PointCut과 여러 개의 Advice로 구성되어 있다.
  * Advisor는 하나의 Pointcut과 하나의 Advice로 구성되어 있다.(Aspect와 비슷한 개념이다.)
* 빈 후처리기는 어떤 클래스를(Pointcut을 이용) 어떻게 조작할 지를(Advice를 이용) 확인하고 변경하여, 스프링 빈 컨테이너에 프록시 객체를 등록한다.
  * 빈 후처리기는 빈에 실제 객체 대신 사용자가 의도적으로 변경한 프록시 객체(가짜 객체)를 등록한다.
  * main027에서는 빈에 저장될 프록시 객체에 로깅을 입힌다고 이해하면 될 것 같다.
<br></br>




### @Aspect

```java
@Slf4j
@Aspect
@Component
public class TimeTraceAspect {
```

* `@Aspect` 애너테이션을 붙이면 스프링은 빈 후처리기를 통해서 클래스에 붙어있는 `@Aspect`를 인식하여 그 클래스가 AOP와 관련된 클래스라는 것을 알게 된다.
  * main027에서는 `TimeTraceAspect` 클래스에 붙어있는 `@Aspect`를 인식해서 `TimeTraceAspect`가 AOP 관련된 Class라는 것을 알게 된다.
  * 즉, 스프링 빈 컨테이너에 프록시 객체가 등록되기 전에 TimeTraceAspect 클래스가 어느 클래스를 어떻게 조작할 것인지를 알려줄 것이다.





```java
@Around("@annotation(timeTrace)")
public Object doTimeTrace(ProceedingJoinPoint joinPoint, TimeTrace timeTrace) throws Throwable {

    String name = joinPoint.getSignature().toShortString().split("\\(")[0] + "()";
    String uri = dataHolder.getUri();
    int millis = timeTrace.millis();
    final int limitMillis = 300;

    dataHolder.stop();

    doLogTime("-->", uri, name, dataHolder.getLastTaskTimeMillis(), millis, limitMillis);

    dataHolder.start();

    Object result = joinPoint.proceed();

    dataHolder.stop();

    doLogTime("<--", uri, name, dataHolder.getLastTaskTimeMillis(), millis, limitMillis);

    dataHolder.start();

    return result;
}
```

* `@Around("@annotation(timeTrace)")`애너테이션을 통해 Pointcut을 설정했다.

  * `@TimeTrace` 애너테이션이 달린 메서드에 로깅 로직을 적용하겠다는 것을 명시하고 있다.

  * main027에서는 `@Around` 애너테이션을 통해 `@TimeTrace` 애너테이션이 붙은 메서드에 이 로깅 로직을 적용해줘 라고 포인트컷이 설정되어 있다.
    * 개념 설명 : 스프링이 실행될 때 빈 후처리기가 해당 메서드가 적용되어 있는 클래스에 AOP를 적용해서 프록시 객체로 바꿔서 스프링 빈 컨테이너에 담게 된다.

* DI 받은 `dataholder`를 이용하여 API 요청 및 응답 시간을 잰다.

```java
private void doLogTime(String arrow, String uri, String name, long lastStopWatchTime,
                       int millis, int limitMillis) {
    if (lastStopWatchTime <= millis) {
        log.info(arrow + "[{}][{}ms][{}]", uri, lastStopWatchTime, name);
    } else if (lastStopWatchTime <= limitMillis) {
        log.warn(arrow + "[{}][{}ms][{}]", uri, lastStopWatchTime, name);
    } else log.error(arrow + "[{}][{}ms][{}]", uri, lastStopWatchTime, name);
}
```

* `doLogTime` 메서드를 이용하여 로깅한다.
  * `doLogTime("-->", uri, name, dataHolder.getLastTaskTimeMillis(), millis, limitMillis);`
  * `doLogTime("<--", uri, name, dataHolder.getLastTaskTimeMillis(), millis, limitMillis);`
<br></br>




### DataHolder 클래스

```java
@Slf4j
@Component
@Scope(value = "request", proxyMode = ScopedProxyMode.TARGET_CLASS)
@Getter
@Setter
public class DataHolder extends StopWatch {

    private Long memberId;
    private List<String> roles;
    private String uri;
    private String method;

    @PostConstruct
    private void init() {
        this.start();
    }

    public void setUri(String uri) {
        this.uri = uri;
        log.info("START [{}] [{}]", uri, method);
    }

    @PreDestroy
    private void destroy() {
        this.stop();
        log.info("END [{}] [{}] [{}]ms", uri, method, this.getTotalTimeMillis());
    }
}
```

* `init()`: 해당 클래스가 생성될 때 StopWatch 시작 후 로깅
* `destroy()`: 해당 클래스가 없어지기 전에 StopWatch 종료, 생성된 시간은 로깅 후 destroy



```java
@Slf4j
@Component
@Scope(value = "request", proxyMode = ScopedProxyMode.TARGET_CLASS)
@Getter
@Setter
public class DataHolder extends StopWatch {
  ...
```

* 클라이언트 요청이 동시에 들어왔을 때 StopWatch가 싱글톤일 경우 필드를 공유하는 문제가 생긴다.
  * 이를 해결하기 위해 클라이언트 요청마다 새로운 StopWatch 객체를 생성해줘야 함
  * `@Scope(value = "request", proxyMode = ScopedProxyMode.TARGET_CLASS)`를 통해 HTTP 요청마다 새로운 객체를 생성하도록 설정
  
  

```java
 	private Long memberId; 
```

* `memberId`: 로그인 권한이 필요한 api 요청 시 접근 권한이 있는 유저인지 확인할 때 필요하다. AccessToken의 claims에서 추출하여 DataHolder를 통해 사용한다.

```java
// MemberController
@TimeTrace
@GetMapping
public ResponseEntity getMyInfo(){
    Member getMember = memberService.findMember(dataHolder.getMemberId());
    return new ResponseEntity<>(mapper.memberToMemberResponseDto(getMember),HttpStatus.OK);
}	
```

* 실제 사용된 메서드로 `dataHolder`를 통해 AccessToken에서 `memberId`를 가져와 유저 정보를 확인할 수 있다.



  ```java
  	private List<String> roles;
  ```

* `roles`: Admin의 권한으로 유저의 데이터를 변경하거나, 삭제하기 위해 필요하다. AccessToken의 claims에서 roles를 추출하여 DataHolder를 통해 사용한다.

```java
@TimeTrace
public void deletePlace(Long memberId, List<String> roles, Long placeId) {
    Place findPlace = placeVerifier.findVerifiedPlace(placeId);
    if (Objects.equals(findPlace.getMember().getMemberId(), memberId) || roles.contains("ADMIN")) {
        placeRepository.delete(findPlace);
    } else throw new PermissionDeniedException(ExceptionCode.PERMISSION_DENIED);
}
```

* `Objects.equals(findPlace.getMember().getMemberId(), memberId) || roles.contains("ADMIN")`
  * 장소를 등록한 멤버이거나, 권한이 ADMIN일 경우 장소를 삭제할 수 있다.
    * 그러나, 실제 프로젝트에서 장소 삭제 기능은 ADMIN만 보이게 설정되어있다.



  ```java
  	private String uri; 
    private String method; 
  ```

* `uri`: 어떤 uri인지 데이터 수집(api 요청 비율 데이터 수집에 사용됨)
* `method`: 어떤 method인지 데이터 수집
<br></br>




### InterceptorConfig

```java
@Configuration
@RequiredArgsConstructor
public class InterceptorConfig implements WebMvcConfigurer {

    private final JwtTokenizer jwtTokenizer;
    private final DataHolder dataHolder;

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new DataInterceptor(jwtTokenizer, dataHolder))
                .order(1)
                .addPathPatterns("/**");
    }
}
```

* 컨트롤러단으로 넘어가기 전에 인터셉터를 거는 `InterceptorConfiguration`
* `jwtTokenizer`와 `dataholder`를 파라미터로 받는 `DataInterceptor` 객체를 생성하고 모든 uri에 인터셉터를 적용한다.

* `.order(1)`은 해당 인터셉터를 첫번째 인터셉터로 등록하겠다는 의미로 추후에 여러 개의 인터셉터를 등록하게 될 때 번호를 매겨 순를 조정할 수 있다.
<br></br>




### DataHolderInterceptor

```java
@Slf4j
@RequiredArgsConstructor
public class DataInterceptor implements HandlerInterceptor {

    private final JwtTokenizer jwtTokenizer;
    private final DataHolder dataHolder;

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler){
        dataHolder.setMethod(request.getMethod());
        dataHolder.setUri(request.getRequestURI());
        if (request.getHeader("Authorization") == null) {
            log.info("로그인 되지 않은 사용자 요청");
            return true;
        }
        try {
            Map<String, Object> claims = verifyJws(request);
            dataHolder.setMemberId(Long.valueOf((Integer) claims.get("memberId")));
            dataHolder.setRoles((List<String>) claims.get("roles"));
            log.info("로그인 된 사용자 요청");
        } catch (Exception e) {
            log.info("로그인 되지 않은 사용자 요청");
            return true;
        }
        return true;
    }
		...
}
```

* 인터셉터가 들어가면 `dataHolder`에 request로 들어온 method와 uri를 저장한다.
  * `if` 헤더에 AccessToken이 포함되어있지 않다면 "로그인 되지 않은 사용자 요청"이라고 로그를 찍고 그대로 넘어간다.
  * `try` 요청에서 claims를 추출하여 dataHolder에 claims로부터 추출한 memberId와 roles를 저장하고 "로그인 된 사용자 요청"이라고 로그를 찍고 넘어간다.
    * `catch` 만약 검증되지 않은 사용자라면 에러가 터지며 "로그인 되지 않은 사용자 요청"이라고 로그를 찍고 넘어간다.
      * but, 이미 필터단에서 vrifyjws가 있어 검증이 될 예정이라 의미없는 로직이긴 하다.
<br></br>
<br></br>


### `getPlaces` 메서드를 수행 했을 때 찍히는 로그 흐름 정리

***

o.s.web.servlet.DispatcherServlet        : Completed initialization in 3 ms

m.server.global.aop.logging.DataHolder   : START [/places] [GET]
m.s.global.interceptor.DataInterceptor   : 로그인 되지 않은 사용자 요청
m.s.g.a.logging.aspect.TimeTraceAspect   : -->[/places] [212ms] [PlaceController.getPlaces()]
m.s.g.a.logging.aspect.TimeTraceAspect   : -->[/places] [8ms] [PlaceServiceImpl.findPlaces()]
m.s.g.a.logging.aspect.TimeTraceAspect   : -->[/places] [1ms] [PlaceRepository.findAllLikeCount()]
m.s.g.a.logging.aspect.TimeTraceAspect   : <--[/places] [52ms] [PlaceRepository.findAllLikeCount()]
m.s.g.a.logging.aspect.TimeTraceAspect   : <--[/places] [0ms] [PlaceServiceImpl.findPlaces()]
m.s.g.a.logging.aspect.TimeTraceAspect   : <--[/places] [8ms] [PlaceController.getPlaces()]
m.server.global.aop.logging.DataHolder   : END [/places] [GET] [336]ms

***

* DispatcherServlet에서 넘어온 request를 DataHolder에서 받아 START 시간 기록을 시작
* DispatcherServlet에서 컨트롤러단으로 넘어가기 전에 DataInterceptor에서 로그인 된 사용자인지 아닌지를 로그에 남김
* TimeTraceAspect에서 API 요청, 응답 시간을 메서드 별로 로깅
* StopWatch가 끝나기 전에 END 로그 남기고 destroy
<br></br>
<br></br>




## Logging 회고

 우리 팀에게 **로깅 적용은 첫 팀 회의에서부터 이야기 한 프로젝트 목표와 가장 어울리는 작업**이었다. 팀원 전원이 비전공자였고, 프로젝트 경험 또한 처음이었기 때문에 꽤나 어려워 보이는 기능을 구현하여 프로젝트의 난이도를 올리는 것은 모두가 원하지 않았다. 오히려 **이번 기회를 백엔드 개발자로서의 기본 역량을 갖출 수 있는 초석으로 삼자는 것이 모두의 의견이었다.** 따라서 **배포부터 유지보수까지 이어지는 안정적인 서비스 운영을 최우선 목표로 삼았다.** 그러기 위해서는 로깅 적용이 필수적이었다. 로깅 기능으로 에러를 손쉽게 잡아내는 것은 물론이고, 성능 개선이 필요한 부분까지 쉽게 확인할 수 있기 때문이다. 



 로깅이 적용된 이후 팀 회의를 거쳐 로그의 시각화를 하기로 결정했다. 로그 중에서 필요한 것만 골라내어 성능 개선에 사용하기 위해서였다. 이는 AWS CloudWatch와 Grafana를 통해 구현되었다. **API 요청 비율과 응답 속도를 한 눈에 확인할 수 있게 된 덕분에 N+1 문제 해결 후 성능 개선 정도를 수치로 확인하고, 요청 비율이 많은 메서드에 대한 캐시 서버 우선 적용이 이루어질 수 있었다.**



 로깅을 구현한 팀원의 주도 하에 진행된 로깅 스터디가 끝이 나고, 개인 공부를 통해 중요한 내용을 많이 배울 수 있었다. **나는 그동안 스프링의 기능 구현을 익히는 데에만 집중을 해 기본 개념에 거의 신경을 쓰지 못했다.** 이는 곧 JWT를 이용한 로그인/로그아웃 구현 등 컨트롤러단 이전에 서블릿, 필터, 인터셉터 쪽에서 일어나는 일에 대해 이해를 못하는 상황으로 이어졌다. JWT 관련 구현을 위해 컨트롤러단 이전에서 일어나는 일을 공부하면서 엄청 힘들었던 경험이 AOP를 공부하는 데에 많은 도움이 되었다. **기존에 크게 신경쓰지 않았던 웹서버 - WAS- DB로 이어지는 아키텍처부터 서블릿, 필터, 인터셉터로 이어지는 공부까지 정말 많은 걸 배울 수 있는 소중한 시간이었고 반대로 부족함도 많이 느껴 더욱 열심히 공부해야겠다고 마음을 먹었다.** 특히 스프링이 스프링 빈 컨테이너의 인스턴스를 싱글톤으로 관리한다든가 하는 기본적인 내용에 대해서도 어느 순간에 잊고 있었구나 하는 생각에 아찔했다. **기본이 중요하다. 기본을 다지자.**
