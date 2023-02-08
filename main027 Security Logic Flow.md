# MAIN027 Security Logic Flow

***

## 들어가기 전에

* 프로젝트에서 제가 맡은 파트의 세부적인 내용까지 타 팀원에게 공유가 될 수 있도록 메뉴얼 수준의 자세한 설명서를 만들었습니다. 
  * 프로젝트 중 인원에 공백이 생겨 자리를 메워야 하는 상황이 발생했을 때, 프로젝트가 끝나고 타 팀원이 제 코드를 공부할 때 도움을 주기 위한 목적입니다.
* 이 글에서는 main027(에코그린 서울) 팀에서 프로젝트에 적용한 Security 관련 코드를 설명하는 내용이 주를 이룹니다.
* 프로젝트 중 생긴 에러의 핸들링 및 이슈에 대한 개인적인 생각 등도 함께 적혀있습니다.
* 코드스테이츠 백엔드 부트캠프의 학습 컨텐츠인 유어클래스 내 시큐리티 챕터를 학습 후 참고하여 작성한 코드입니다.
  * Security 전체 흐름이 숙지되어 있지 않다면 유어클래스 내 시큐리티 챕터를 학습 후 보시는 걸 권장합니다.



## 목차

1. [회원가입 흐름](#회원가입-흐름)
2. [로그인(JWT 발급까지 흐름)](로그인(#jwt-발급까지-흐름))
3. [JWT 검증 작업](#jwt-검증-작업)
4. [JWT 재발급 구현](#jwt-재발급-구현)
5. [JWT 로그아웃 구현](#jwt-로그아웃-구현)

***





## 회원가입 흐름

* **MemberController** -> **postMember**

* **MemberService** -> **MemberServiceImpl** -> **createMember** 서비스 계층에서 회원가입 진행

  ```java
  @TimeTrace
      public Member createMember(Member member) {
          memberVerifier.verifyExistsEmailAndNickName(member);
  
          /**
           * 패스워드 암호화
           */
          String encryptedPassword = passwordEncoder.encode(member.getPassword());
          member.setPassword(encryptedPassword);
  
          /**
           * User Role 생성
           */
          List<String> roles = authorityUtils.createRoles(member.getEmail());
          member.setRoles(roles);
  
          return memberRepository.save(member);
      }
  ```

  

  **User Role 생성**

  ```java
  public List<String> createRoles(String email) {
          if (email.equals(adminMail)) {
              return ADMIN_ROLES_STRING;
          }
          return USER_ROLES_STRING;
      }
  ```

  -> `CustomAuthorityUtils`의 `createRoles` 메서드에서 역할 부여

  -> ```memberRepository.save(member)```로 DB에 저장

* 회원가입 완료





## 로그인(JWT 발급까지 흐름)

* 클라이언트가 서버 측에 로그인 인증을 요청함(username / password를 서버 측에 전송)

  ```java
  public class JwtAuthenticationFilter extends UsernamePasswordAuthenticationFilter {
  ```

  * 로그인 인증을 담당하는 **Security Filter**가 클라이언트의 인증 정보를 수신하며 발급 과정 시작

  ```java
  @SneakyThrows
  @Override
  public Authentication attemptAuthentication(HttpServletRequest request, HttpServletResponse response) {
      ObjectMapper objectMapper = new ObjectMapper();
      LoginDto loginDto = objectMapper.readValue(request.getInputStream(), LoginDto.class);
  
      UsernamePasswordAuthenticationToken authenticationToken =
              new UsernamePasswordAuthenticationToken(loginDto.getUsername(), loginDto.getPassword());
  
      return authenticationManager.authenticate(authenticationToken);
  }
  ```

* **request**에 담긴 로그인 인증 정보(username & password)를 **LoginDto** 타입으로 변환하여 저장 

* 아직 인증되지 않은 **authenticationToken** 객체 생성

* **AuthenticationManager**에게 전달해 인증 처리 위임

  ```java
  public class MemberDetailsService implements UserDetailsService {
  		
  		...
    
  		@Override
      public MemberDetails loadUserByUsername(String email) throws 	UsernameNotFoundException {
          Optional<Member> optionalMember = memberRepository.findByEmail(email);
          Member findMember = optionalMember.orElseThrow(() -> new TokenException(ExceptionCode.UNREGISTERED_MEMBER));
  
          return new MemberDetails(findMember);
      }
    
      ...
  ```
  
  * **AuthenticationManager** `MemberDetailsService`에게 사용자의 `MemberDetails` 생성을 위임
  * `MemberDetailsService`가 사용자의 Credential을 DB에서 조회
    * 만약, db에 사용자의 Credential이 없다면 `UNREGISTERED_MEMBER` 에러를 던짐(아래 실패 시 핸들러 처리에서 재설명)
  
  
  ```java
  public class MemberDetails extends Member implements UserDetails {
  		MemberDetails(Member member) {
        setMemberId(member.getMemberId());
        setNickName(member.getNickName());
        setEmail(member.getEmail());
        setPassword(member.getPassword());
        setRoles(member.getRoles());
      }
  ```

  * `MemberDetailsService`가 DB에서 조회한 사용자의 정보를 기반으로 `MemberDetails`를 생성

  * **AuthenticationManager**에게 생성한 `MemberDetails`를 전달
  
    * `MemberDetails`는 데이터베이스 등에 저장된 사용자의 Username과, 사용자의 자격을 증명해주는 Credential인 Password와, 사용자의 권한 정보 등을 포함하고 있는 컴포넌트임
  
      

```java
public class DaoAuthenticationProvider extends AbstractUserDetailsAuthenticationProvider {
  
  ...
    
  @Override
	@SuppressWarnings("deprecation")
	protected void additionalAuthenticationChecks(UserDetails userDetails,
			UsernamePasswordAuthenticationToken authentication) throws AuthenticationException {
		if (authentication.getCredentials() == null) {
			this.logger.debug("Failed to authenticate since no credentials provided");
			throw new BadCredentialsException(this.messages
					.getMessage("AbstractUserDetailsAuthenticationProvider.badCredentials", "Bad credentials"));
		}
		String presentedPassword = authentication.getCredentials().toString();
		if (!this.passwordEncoder.matches(presentedPassword, userDetails.getPassword())) {
			this.logger.debug("Failed to authenticate since password does not match stored value");
			throw new BadCredentialsException(this.messages
					.getMessage("AbstractUserDetailsAuthenticationProvider.badCredentials", "Bad credentials"));
		}
	}
  
  ...
```

* **AuthenticationManager**가 로그인 시 입력한 비밀번호와 `MemberDetails`의 정보를 비교해 인증처리 후 `JwtAuthenticaitionFilter`로 전달

  * `if (!this.passwordEncoder.matches(presentedPassword, userDetails.getPassword()))` 
  
  

```java
public class JwtAuthenticationFilter extends UsernamePasswordAuthenticationFilter {
  
  ...
    
  @Override
  protected void successfulAuthentication(HttpServletRequest request,
                                            HttpServletResponse response,
                                            FilterChain chain,
                                            Authentication authResult) throws ServletException, IOException {
        Member member = (Member) authResult.getPrincipal();

        String accessToken = delegateAccessToken(member);
        String refreshToken = delegateRefreshToken(member);

        /**
         * 레디스에 (key :refreshToken), (value: email), (duration: expirationMinutes)으로 refreshToken 저장
         */
        redisService.setRefreshToken(refreshToken, member.getEmail(), 	jwtTokenizer.getRefreshTokenExpirationMinutes());

        response.setHeader("Authorization", "Bearer" + accessToken);
        response.setHeader("Refresh", refreshToken);

        this.getSuccessHandler().onAuthenticationSuccess(request, response, authResult);
    }
  
  ...
```

* `JwtAuthenticationFilter`의 `successfulAuthentication` 메서드에서 **AccessToken**, **RefreshToken**을 발급하고 **responseHeader**에 넣음
* **RefreshToken**은 **Redis**에 따로 저장
* 성공 시에 `getSuccessHandler` 호출





### 성공

```java
@Slf4j
public class MemberAuthenticationSuccessHandler implements AuthenticationSuccessHandler {
    @Override
    public void onAuthenticationSuccess(HttpServletRequest request,
                                        HttpServletResponse response,
                                        Authentication authentication) throws IOException {
        Member member = (Member) authentication.getPrincipal();
        ObjectMapper objectMapper = new ObjectMapper();

        LinkedHashMap<String, Object> loginResponse = new LinkedHashMap<>();
        loginResponse.put("memberId", member.getMemberId());
        loginResponse.put("email", member.getEmail());
        loginResponse.put("nickName", member.getNickName());
        loginResponse.put("roles", member.getRoles());

        String responsebody = objectMapper.writeValueAsString(loginResponse);
        response.setContentType(MediaType.APPLICATION_JSON_VALUE);
      	response.setCharacterEncoding("UTF-8");

        response.getWriter().write(responsebody);

        log.info("# Authenticated successfully!");
        log.info("nickName: {}, email: {}, role: {}", member.getNickName(), member.getEmail(), member. getRoles());
    }
}
```

* `MemberAuthenticationSuccessHandler`를 호출해서 성공 응답을 보내고, 로그에 기록



### 실패

```java
@Slf4j
public class MemberAuthenticationFailureHandler implements AuthenticationFailureHandler {
  
  ...
    
      private void sendErrorResponse(HttpServletResponse response, AuthenticationException exception) throws IOException {
        Gson gson = new Gson();
        ErrorResponse errorResponse;

        if (exception.getClass().equals(BadCredentialsException.class)) {
            String message = "password do not match";
            errorResponse = ErrorResponse.of(HttpStatus.BAD_REQUEST, message);
            response.setStatus(HttpStatus.BAD_REQUEST.value());
        } else if (exception.getClass().equals(InternalAuthenticationServiceException.class)){
            errorResponse = ErrorResponse.of(HttpStatus.NOT_FOUND, exception.getMessage());
            response.setStatus(HttpStatus.NOT_FOUND.value());
        } else {
            errorResponse = ErrorResponse.of(HttpStatus.UNAUTHORIZED);
            response.setStatus(HttpStatus.UNAUTHORIZED.value());
        }

        response.setContentType(MediaType.APPLICATION_JSON_VALUE);
        response.getWriter().write(gson.toJson(errorResponse, ErrorResponse.class));
    }
}
```

* 실패 시에는 필터를 타고 넘어가 `MemberAuthenticationFailureHandler`에서 에러 처리

* 패스워드 틀렸을 경우 : 

  * ``` if (exception.getClass().equals(BadCredentialsException.class))``` 
  * 오류 메시지로 "password do not match" 출력

* 등록되지 않은 이메일로 로그인 시도할 경우 :

  *  `else if (exception.getClass().equals(InternalAuthenticationServiceException.class))`

  ```java
      @Override
      public MemberDetails loadUserByUsername(String email) throws UsernameNotFoundException {
          Optional<Member> optionalMember = memberRepository.findByEmail(email);
          Member findMember = optionalMember.orElseThrow(() -> new TokenException(ExceptionCode.UNREGISTERED_MEMBER));
  
          return new MemberDetails(findMember);
      }
  ```

  * `MemberDetailsService`의 `loadUserByUsername` 메서드에서 `UNREGISTERED_MEMBER` 에러를 던짐

  * Postman에서 응답값으로 Status는 404, 메세지는 `UNREGISTERED_MEMBER`의 메시지("Unregistered Member")가 출력되지만, 스프링 내 exception은 예상과 달리 **TokenExeption**이 아닌 **InternalAuthenticationServiceException**이다. 이상하다고 생각하여 이에 대해 자세히 찾아보았다.

    * 에러메시지를 자세히 읽어보니 `Caused by: main027.server.global.advice.exception.TokenException: Unregistered Member` 라고 나온다. 내 생각대로 `loadUserByUsername`에서 에러가 나는게 맞다. 
    * 그런데 에러를 던지는건 `DaoAuthenticationProvider.retrieveUser(DaoAuthenticationProvider.java:109)` 여기서부터였다.
    * 즉, `loadUserByUsername`에서 난 에러가 원인이 되어 `DaoAuthenticationProvider.retrieveUser`에서 그 에러를 잡아 다시 던지고 최종적으로 `MemberAuthenticationFailureHandler`에서 처리하는 것이다.

    ```java
    @Override
    	protected final UserDetails retrieveUser(String username, UsernamePasswordAuthenticationToken authentication)
    			throws AuthenticationException {
        
        	...
            
          catch (Exception ex) {
    			throw new InternalAuthenticationServiceException(ex.getMessage(), ex);
    		}
    	}
    ```

    * `loadUserByUsername`에서 던진 에러를 잡아 새롭게 `InternalAuthenticationServiceException` 에러를 던지기 때문에 `MemberAuthenticationFailureHandler`의 `else if (exception.getClass().equals(InternalAuthenticationServiceException.class))`에서 에러가 걸리는 것이고, 파라미터값은 기존의 `ExceptionCode.UNREGISTERED_MEMBER`가 들어가 "Unregistered Member"라고 출력되는 것이었다.

* 그 외 : ``` else ``` 이 외에 다른 에러는 권한 없음으로 처리.





## JWT 검증 작업

### JwtVerificationFilter

* 요청으로 들어온 **AccessToken**을 검증함

* JWT 검증 작업에서 하는 일은 크게 두 가지임.

  1. **AccessToken**이 요청으로 들어오면 먼저 로그아웃 된 **AccessToken**인지 검증 작업을 함(**BlackList**)

  2. 그 다음 시스템에서 발급한 토큰이 맞는지 검증 작업을 함(**verifyJws**)

```java
@Slf4j
@RequiredArgsConstructor
public class JwtVerificationFilter extends OncePerRequestFilter {
  
  ...
  
    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response,
                                    FilterChain filterChain) throws ServletException, IOException {
        try {
            String blackList = redisService.getToken(request.getHeader("Authorization"));
            if (blackList != null) throw new TokenException(ExceptionCode.LOGOUT_MEMBER);

            Map<String, Object> claims = verifyJws(request);
            setAuthenticationToContext(claims);
        } catch (SignatureException se) {
            request.setAttribute("exception", se);
        } catch (ExpiredJwtException ee) {
            request.setAttribute("exception", ee);
        } catch (TokenException te){
            request.setAttribute("exception", te);
        } catch (Exception e) {
            request.setAttribute("exception", e);
        }
        filterChain.doFilter(request, response);
    }
  
  ...
```



```java
String blackList = redisService.getToken(request.getHeader("Authorization"));
if (blackList != null) throw new TokenException(ExceptionCode.LOGOUT_MEMBER);
```

* **Redis**의 Blacklist에 저장되어있는 **AccessToken**인지 확인
  * 있다면, 탈취된 AccessToken일 확률이 높음(아무튼 사용되서는 안되는 토큰이니 로그인 거부)




```java
private Map<String, Object> verifyJws(HttpServletRequest request) {
  	String jws = request.getHeader("Authorization").replace("Bearer", "");
  	String base64EncodedSecretKey = jwtTokenizer.encodeBase64SecretKey(jwtTokenizer.getSecretKey());
  	Map<String, Object> claims = jwtTokenizer.getClaims(jws, base64EncodedSecretKey).getBody();

  	return claims;
}
```

* `verifyJws` 메서드는 Bearer을 제외한 **AccessToken**의 실제 토큰 부분만을 가지고 SecretKey로 parsing하는 작업을 통해 claims를 얻는 역할을 한다.
  * 이 메서드가 성공적으로 수행되면 SecretKey로 **AccessToken** 안의 claims를 얻을 수 있다는 의미이기 때문에 별도의 검증 메서드 없이 서버에서 발급한 AccessToken임을 확인할 수 있다.
  * `jwtTokenizer.getClaims(jws, base64EncodedSecretKey)`
    * `parseClaimsJws()`에서 Claims를 파싱할 때 내부적으로 Signature를 먼저 검증하게 되는데 이때 만료기한까지 검증하여 기한이 만료되었다면 `ExpiredJwtException`이 나온다.




```java
    private void setAuthenticationToContext(Map<String, Object> claims) {
        String email = (String) claims.get("email");
        List<GrantedAuthority> authorities = authorityUtils.createAuthorities((List) claims.get("roles"));
        Authentication authentication = new UsernamePasswordAuthenticationToken(email, null, authorities);
        SecurityContextHolder.getContext().setAuthentication(authentication);
        log.info("securityContext에 email ={}, authorities ={} 저장 완료", email, authorities);
    }
```

* 검증이 완료되면 `SecurityContext`에 Authentication 객체를 저장하게 된다. 
* Authentication에는 email과 authorities(권한)이 저장되며 `SecurityFilterChain`에서 아직 처리가 진행되지 않은 Filter 어딘가에서 사용하기 위해 저장하는 것이다.
* 클라이언트의 요청이 모두 처리되면(필터에서 더 이상 Authentication 객체가 필요 없게 되면) `SecurityContextPersistenceFilter`에서 Authentication 객체를 삭제한다.
  * `SecurityContextPersistenceFilter`는 `SecurityContext`를 저장, 로드, 삭제 등을 하는 역할을 한다.



```java
    @Override
    protected boolean shouldNotFilter(HttpServletRequest request) throws ServletException {
        String authorization = request.getHeader("Authorization");
        return authorization == null || !authorization.startsWith("Bearer");
    }
```

* `shouldNotFilter`는 return 문이 true일 경우 해당 필터를 수행하지 않고 다음 필터로 넘어가게 하는 역할을 함
* 검증 작업에서는 요청 헤더에 **AccessToken**이 없거나(== null), **RefreshToken**이 들어온 경우(Bearer로 시작하지 않는 Token) true가 되어 `JwtVerificationFilter`가 수행되지 않음





## JWT 재발급 구현

### JwtReissueFilter

```java
@RequiredArgsConstructor
public class JwtReissueFilter extends OncePerRequestFilter {
  
  ...
    
  @Override
  protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain) throws ServletException, IOException {
     try {
       String requestRefreshToken = request.getHeader("Refresh");
       MemberDetailsService.MemberDetails memberDetails = memberDetailsService.loadUserByUsername(
         redisService.getToken(requestRefreshToken));

       String base64EncodedSecretKey = jwtTokenizer.encodeBase64SecretKey(jwtTokenizer.getSecretKey());
       jwtTokenizer.getClaims(requestRefreshToken, base64EncodedSecretKey);
       String newAccessToken = delegateNewAccessToken(memberDetails, base64EncodedSecretKey);

       response.setHeader("Authorization", "Bearer" + newAccessToken);
       response.setContentType(MediaType.APPLICATION_JSON_VALUE);
       response.getWriter().write("Reissued Complete.");

     } catch (TokenException e) {
       response.setContentType(MediaType.APPLICATION_JSON_VALUE);
       response.setStatus(ExceptionCode.INVALID_REFRESH_TOKEN.getStatus());
       response.getWriter().write(ExceptionCode.INVALID_REFRESH_TOKEN.getMessage());
     }
  }
  
  ...
```

* Jwt를 재발급받기 위해서는 **RefreshToken**을 요청 헤더에 실어 보내야한다.



```java
String requestRefreshToken = request.getHeader("Refresh");
       MemberDetailsService.MemberDetails memberDetails = memberDetailsService.loadUserByUsername(
         redisService.getToken(requestRefreshToken));
```

* **RefreshToken**으로 `redisService` 클래스에서 value인 email을 찾아 `memberDetailsService`를 통해 `memberDetails` 생성



```java
String base64EncodedSecretKey = jwtTokenizer.encodeBase64SecretKey(jwtTokenizer.getSecretKey());
       jwtTokenizer.getClaims(requestRefreshToken, base64EncodedSecretKey);
       String newAccessToken = delegateNewAccessToken(memberDetails, base64EncodedSecretKey);
```

* 생성된 `memberDetails`를 이용해 새로운 **AccessToken** 발급



```java
catch (TokenException e) {
       response.setContentType(MediaType.APPLICATION_JSON_VALUE);
       response.setStatus(ExceptionCode.INVALID_REFRESH_TOKEN.getStatus());
       response.getWriter().write(ExceptionCode.INVALID_REFRESH_TOKEN.getMessage());
     }
```

* 유효하지 않은 **RefreshToken**이라면 `INVALID_REFRESH_TOKEN` 에러를 응답으로 보냄



```java
    @Override
    protected boolean shouldNotFilter(HttpServletRequest request) throws ServletException {
        String authorization = request.getHeader("Refresh");
        String uri = request.getRequestURI();

        return authorization == null || !(uri.equals("/reissue")) || !(request.getMethod().equals("POST"));
    }
```

* **RefreshToken**이 null이거나, uri가 "/reissue"가 아니거나, 메서드 요청이 POST가 아니라면 필터를 수행하지 않고 다음 필터로 넘어감



## JWT 로그아웃 구현

### JwtLogoutFilter

```java
@RequiredArgsConstructor
public class JwtLogoutFilter extends OncePerRequestFilter {
  
  ...
   
  @Override
  protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response,
                                  FilterChain filterChain) throws ServletException, IOException {

      String requestAccessToken = request.getHeader("Authorization");
      String subject = jwtTokenizer.getSubject(requestAccessToken.replace("Bearer", ""));
      long expiration = jwtTokenizer.getAccessTokenExpirationMinutes();
      redisService.setBlackList(requestAccessToken, subject, expiration);

      String requestRefreshToken = request.getHeader("Refresh");
      redisService.deleteRefreshToken(requestRefreshToken);

      response.setContentType(MediaType.APPLICATION_JSON_VALUE);
      response.getWriter().write("logout Complete.");
      response.setHeader("Location", "/auth/login");
  }
  
  ...
```

* 로그아웃을 할 때는 **AccessToken**과 **RefreshToken**이 모두 필요하다.



```java
String requestAccessToken = request.getHeader("Authorization");
String subject = jwtTokenizer.getSubject(requestAccessToken.replace("Bearer", ""));
long expiration = jwtTokenizer.getAccessTokenExpirationMinutes();
redisService.setBlackList(requestAccessToken, subject, expiration);
```

* 요청 헤더로 받은 AccessToken에서 `jwtTokenizer`의 `getSubject` 메서드를 이용하여 subject(email)를 얻음
* `redisService`에 `setBlackList` 메서드를 이용하여 사용돼서는 안되는 AccessToken 저장



```java
public String getSubject(String token) {
    String subject = Jwts
            .parserBuilder()
            .setSigningKey(getKeyFromBase64EncodedKey(encodeBase64SecretKey(getSecretKey())))
            .build()
            .parseClaimsJws(token)
            .getBody()
            .getSubject();

    return subject;
}
```

* `getSubject` 메서드는 구글링을 통해 스택오버플로우에서 찾은 메서드를 이것저것 만져보다가 만들어낸 것.
  * `Jwts.`라는 기능이 있는 모양이다.
  * 파싱을 통해 SecretKey를 세팅하고 거기서 claims를 얻는다. claims에서 Subject를 get하여 BlakList 등록에 사용한다.
  * `.setSigningKey(getKeyFromBase64EncodedKey(encodeBase64SecretKey(getSecretKey())))` 이 부분은 **getSecretKey()**를 인코딩했다가 다시 디코딩하는 의미없는 일을 하지만 이렇게 하지 않고 그냥 **getSecretKey()**를 넣으면 **Deprecated** 뜸(임시 방편으로 해놓은 것이다)
  



```java
String requestRefreshToken = request.getHeader("Refresh");
        redisService.deleteRefreshToken(requestRefreshToken);
```

* 요청 헤더로 받은 **RefreshToken**은 **Redis**에 저장된 **RefreshToken**의 삭제에 이용됨



```
response.setHeader("Location", "/auth/login");
```

* 응답 헤더에 리다이렉트 할 Location 추가
* Postman으로 응답 헤더에 추가된 Location 확인 가능



```java
@Override
    protected boolean shouldNotFilter(HttpServletRequest request) throws ServletException {
        String requestAccessToken = request.getHeader("Authorization");
        String requestRefreshToken = request.getHeader("Refresh");
        String uri = request.getRequestURI();

        return requestAccessToken == null || requestRefreshToken == null || !(uri.equals("/auth/logout")) || !(request.getMethod().equals("POST"));
    }
}
```

* **AccessToken**이 null이거나, **RefreshToken**이 null이거나, uri가 "/auth/logout"이 아니거나, 메서드 요청이 POST가 아니라면 필터를 실행하지 않고 다음 필터로 넘어감
