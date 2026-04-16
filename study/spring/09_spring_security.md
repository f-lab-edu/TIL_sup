# Spring Security

## Spring Security 정의
스프링 기반의 애플리케이션의 보안을 담당하는 프레임워크.
Spring Security는 인증(Authentication), 인가(Authorization), 그리고 CSRF 같은 일반적인 웹 공격 방어를 제공하는 보안 프레임워크이다. Servlet 기반 애플리케이션에는 표준 Servlet Filter를 통해 서블릿 컨테이너와 통합된다. Spring MVC의 DispatcherServlet 안쪽이 아니라, 그 앞단 요청 흐름에서 보안 처리를 시작하는 특징이 핵심이다.

## Spring Security가 필터를 사용하는 이유
Spring Security가 필터를 사용하는 이유는 컨트롤러에 도달하기 요청을 가로채서 보안 판단을 해야 하기 때문이다.

예를 들어:
- 로그인 안 한 사용자가 ```/admin``` 요청
- 권한 없는 사용자가 ```/api/orders/123/cancel``` 요청
- 위조 요청이 들어온 POST 요청

이런 요청은 컨트롤러로 보내지 않는 게 맞다. 필터는 서블릿 체인 앞단에서 동작하므로 이 역할에 적합하다.

또한, 보안은 특정 컨트롤러 몇 개가 아니라 거의 모든 요청에 공통 적용된다.
- 인증 확인
- 권한 확인
- SecurityContext 로드/정리
- 세션 확인
- 예외를 401/403으로 반환
- CSRF 검사
- 헤더 보안 처리

이러한 처리들은 공통으로 처리되어야 하고, 필터 체인으로 쌓아두는 방식이 가장 유지보수하기 좋다.

마지막으로, Spring Security는 Servlet Filter 기반이다. 그렇기 때문에 Spring MVC가 있어야 동작하는 구조가 아니다. 공식 문서에서도 Servlet Container 위에서 동작한다고 설명하고 있다. 즉, 웹 공통 보안을 다루기에 범용성이 좋은 프레임워크이다.

##  또 다른 이유들 - Filter의 장점

### 1. DispatcherServlet 전에 차단 가능
보안상 가장 이상적인 위치인 컨트롤러 진입 전에 인증/인가 처리가 가능하다.

### 2. 요청 단위 체인 구성 가능

Spring Security는 요청마다 같은 필터를 무조건 다 돌리지 않는다. 어떤 요청에 어떤 보안 필터 체인을 적용할 선택을 할 수 있다. 이렇게 할 수 있는 이유는 SecurityFilterChain이 있기 때문이다.

예를 들어서:
- /api/** : JWT 기반 stateless 체인
- /admin/** : form login + session 체인
- /css/** , /js/** : 보안 제외 또는 최소화

이렇게 나눌 수 있다. 공식 문서에도 첫 번째로 매칭되는 SecurityFilterChain 하나만 적용된다고 나와있다.

### 3. 순서 제어
인증은 인가보다 먼저 오면 안된다. Spring Security는 필터가 특정 순서로 실행되도록 보장된다. 공식 문서도 인증 필터가 인가 필터보다 먼저 와야 한다고 설명한다.

## Spring Security 전체 동작 순서

1. 클라이언트 요청
2. Servlet Container가 FilterChain 생성
3. DelegatingFilterProxy가 Spring Bean 쪽으로 위임
4. 실제 보안 지입점인 FilterChainProxy가 동작
5. 현재 요청에 맞는 SecurityFilterChain 선책
6. 그 안의 Security Filter들이 순서대로 실해
7. 인증 성공 시 SecuriyContextHolder에 인증 정보 저장
8. 인가 단계에서 권한 검사
9. 통과하면 컨트롤러까지 진입
10. 요청 종료 시 보안 컨텍스트 정리

## 내부 구조

### DelegatingFilterProxy
서블릿 컨테이너는 Spring Bean을 모른다.  
그래서 중간 다리가 필요하다.

DelegatingFilterProxy는 서블릿 컨테이너의 Filter 등록 방식과 Spring ApplicationContext를 연결하는 브리지이다. 컨테이너에는 이 프록시가 등록되고, 실제 일은 Spring Bean에게 넘기게 된다.

즉, 톰캣은 필터 하나가 등록됐다고만 여긴다. 그리고 그 필터가 Spring Bean을 실행하게 된다.

1. 클라이언트 요청
2. 서블릿 컨테이너의 Filter Chain에서 DelegatingFilterProxy가 호출된다.
3. DelegatingFilterProxy가 스프링 ApplicationContext에서 특적 이름의 Filter Bean을 찾는다.
4. 찾은 실제 Filter Bean에게 doFilter() 처리를 위임한다.

이 DelegatingFilterProxy가 springSeucirtyFilterChain이라는 빈으로 요청을 넘기는 것이다. 그리고 이 Chain이 FilterChainProxy다.

```
[Servlet Container Filter Chain]
        ↓
DelegatingFilterProxy
        ↓
springSecurityFilterChain
        ↓
FilterChainProxy
        ↓
여러 개의 Security Filter
(예: UsernamePasswordAuthenticationFilter, BasicAuthenticationFilter 등)
```

즉, FilterChainProy는 Security의 핵심이고, DelegatingFilterProxy는 그 앞단 연결용이라고 보면 된다.

### FilterChainProxy
FilterChainProxt가 Spring Security의 핵심 엔진이다.

Filter Chain Proxy는 여러 본안 필터를 직접 관리하고, 어떤 요청에 어떤 체인을 적용할지 결정한다. 또한 SecurityContext 정리나 HttpFirewall 적용 같은 공통 보안 처리도 많다.

FilterChainProxy는 Security에서 총괄 관리자라고 보면 된다.
- 개발 필터 = 실제 작업자

### SecurityFilterChain
요청 URL이나 RequestMacher 조건에 따라 적용될 보안 필터 묶음이다.
공식 문서상 FilterChainProxy는 요청마다 적절한 SecurityFilterChain을 선택한다.

```
@Bean
SecurityFilterChain apiSecurity(HttpSecurity http) throws Exception {
    http
        .securityMatcher("/api/**")
        .authorizeHttpRequests(auth -> auth
            .requestMatchers("/api/public/**").permitAll()
            .anyRequest().authenticated()
        )
        .httpBasic();
    return http.build();
}
```
이 의미는 ```/api/**``` 요청에 대해서만 이 체인을 적용하겠다는 뜻이다. HttpSecurity는 웹 요청 보안 구성을 위한 빌더이며, 기본적으로 모든 요청에 적용될 수 있지만 제한할 수도 있다.

### 인증 처리 방식
인증에서 핵심 객체는 다음과 같다.
```
- Authentication
- AuthenticationManager
- AuthenticationProvider
- SecurityContextHolder
```

1. 로그인 요청이 들어온다.
   form login이라 가정했을때 UsernamePasswordAuthenticationFilter가 요청에서 username과 password를 꺼내서 UsernamePasswordAuthenticationToken을 만든다.

2. AuthenticationManager에게 인증을 맡긴다.
   AutenticationManager는 인증을 수행하는 API이다.  
   대표적인 구현체는 ProviderManager이다.

3. ProviderManager가 AuthenticationProvider들에게 위임한다.
   각 AuthenticationProvider는 자신이 처리 가능한 인증 타입을 검사하고, 가능하면 인증을 수행한다. 공식 문서상 여러 Provider가 순서대로 시도될 수 있다.

    - 아이디/비밀번호 인증 -> DaoAuthenticationProvider
    - JWT 인증 -> 커스텀 Provider 또는 커스텀 필터 + 토큰 검증 로직
    - SAML/OAuth2 -> 해당 방식용 Provider

4. 인증 성공 시 SecurityContextHolder에 저장
   인증이 성공하면 현재 요청을 처리하는 스레드에서 쓸 수 있도록 인증 정보가 보관된다. 인증 결과는 SecurityContextHolder에 설정된다.

5. 이후 인가에서 사용된다.
   이후에 AuthrizationFilter나 메서드 보안이 이 인증 정보를 통해 권한을 판단한다.

### 인가 처리 방식
Spring Security는 웹 요청이나 메서드 호출에 대해 접근 제어를 수행하며, 현재는 AuthorizationManager가 핵심 역할을 맡는다. 공식 문서상 AuthorizationManager는 기존 AccessDecisionManage / AccessDescisionVoter를 대체하는 방향이다.

```
.authorizeHttpRequests(auth -> auth
    .requestMatchers("/admin/**").hasRole("ADMIN")
    .requestMatchers("/mypage/**").authenticated()
    .anyRequest().permitAll()
)
```
- /admin/** -> ADMIN 권한 필요
- /mypage/** -> 로그인만 하면 가능
- 나머지 -> 허용

### 인증 안 된 사용자가 권한이 필요한 요청에 접근한다면
공식 문서의 form login 흐름 기준으로 보면
1. 인증 안 된 사용자가 보호 리소스 요청
2. AuthorizationFilter가 접근 불가 판단
3. ExceptionTranslationFilter가 예외를 잡음
4. 인증이 안 된 상태면 AuthenticationEntryPoint를 호출
5. 로그인 페이지로 리다이렉트하거나 WWW-Authenticate 헤더를 내림

즉, 실제로 401/403 처리 흐름을 정리해주는 필터는 ExceptionTranslationFilter이다.  
이 필터는 인증 시작(AuthenticationException), 403 처리(AccessDeniedException)를 담당한다.

### 주의할 점
#### 1. 필터는 순서를 잘 못 넣으면 인증 불가
커스텀 JWT 필터를 너무 뒤에 넣으면 이미 인가 판단이 끝나 있을 수 있다.

JWT 인증 필터를 너무 뒤에 넣으면, 권한 판단에 필요한 Authentication이 아직 SecurityContext에 없는 상태로 인가 필터가 먼저 실행될 수 있다. 그 결과, 토큰이 정상이어도 401 또는 403으로 막힐 수 있다. Spring Security 공식 문서도 인증을 수행하는 필터는 인가를 수행하는 필터보다 먼저 실행되어야 한다고 설명한다. 또한 AuthorizationFilter는 기본적으로 체인 마지막에 위치한다.

Spring Security는 하나의 거대한 보안 로직이 아니라, 여러 개의 필터가 정해진 순서대로 실행되는 구조다. 각 필터는 당연이 역할이 다 다르다.

즉, Spring Security에서 HTTP 요청 권한 검사는 현대 버전 기준 AuthorizationFilter가 담당하며, 이 필터는 기본적으로 마지막에 위치한다. 즉, 그 앞에서 인증이 끝나 있어야 한다. 따라서 JWT 필터가 AuthorizationFilter 뒤에 있거나, 실질적으로 인가 판단 이후에 동작하는 위치에 들어가면 이런 일이 생긴다.
- /api/user/me 요청
- 헤더에 Authrization: Bearer [JWT] 존재
- 인가 필터 먼저 실행
- SecurityContext는 비어있기 때문에 authenticated() 또는 hasRole 검사 실패
- 요청 차단
- 뒤에 위치한 JWT 필터는 실행 기회가 없거나, 실행돼도 이미 응답이 결정됨

JWT는 보통 요청 헤더를 읽어서 미리 인증 객체를 세팅해야 하므로, 일반적으로 세션 로그인과 같이 사용하게 된다면, UsernamePasswordAuthenticationFilter 보다 앞이나 그 근처에 둔다.

예를 들면
```
http.addFilterBefore(jwtAuthenticationFilter, UsernamePasswordAuthenticationFilter.class);
```
- JWT 기반 인증을 먼저 시도
- 뒤쪽 인가 필터가 SecurityContext에 들어 있는 인증 정보를 사용할 수 있다.
- 폼 로그인과 혼용하는 경우에도 충돌을 줄이기 쉽다.

#### 2. 인증 방식과 저장 방식을 섞어 생각하면 안된다.
- 세션 기반인지
- JWT stateless 인지
- 둘 다 사용하는 지

먼저 정해야 한다.

- 인증 방식 : 사용자를 어떻게 확인할 것 인가
  예 : 폼 로그인, OAuth2, Bearer JWT
- 인증 상태 저장 방식 : 인증 결과를 다음 요청까지 어디에 유지할 것인가
  예 : 서버 세션(HttpSession), 저장하지 않음(stateless), 별도 저장소

Spring Security도 SecurityContextHolder에 인증 정보가 있으며 사용하고, 그 인증 정보를 어떻게 저장하고 다시 불러올지는 구성 요소가 담당한다. 즉, 어떻게 로그인하느냐와 로그인 결과를 어디에 유지하느냐는 분리해서 봐야 한다.

#### 3. 401과 403을 구분해야 한다.
- 401 : 인증 안됨
- 403 : 인증은 됐지만 권한 없음

#### 4. Spring Security 6에서는 세션 관리 관련 동작이 예전과 다르다.
공식 문서상 Spring Seucrity 6에서는 기본적으로 SessionManagementFilter를 사용하지 않으며, 인증 메커니즘이 직접 SessionAuthenticationStrategy를 호출하는 방식으로 바뀌었다.

