# Spring Bean 알아보기

## Bean 이란?
빈은 스프링 컨테이너가 생성하고 관리하는 객체를 의미한다.

직접 new로 만든 객체가 아닌, 스프링이 대신 생성한 객체로 의존성을 넣어주고, 생명주기(Life Cycle)까지 관리하는 객체를 빈이라고 한다.

```
@Service
public class MemberService {
}
```

빈 등록 방식은 이렇게 어노테이션 또는 xml 방식 등 다양한 방법이 있다.
빈 등록 방식과 같은 방식 공식 문서 또는 블로그를 보면 쉽게 찾을 수 있다. 여기서는 등록 방식과 같은 내용 보다는 빈이 어떻게 관리되고, 왜 이렇게 사용하는지 알아볼 것이다.

### 객체를 Spring Context가 관리하는 이유
[Spring IoC](https://yeonsup.com/blog/Spring/43)에 나와있긴 하지만, 정리하면 다음과 같다.

1. 객체 생성과 관리 책임을 분리
   개발자가 직접 new를 해서 생성하는게 아닌 빈으로 등록하면 스프링이 객체 생성 시점을 관리한다.

2. 의존성 주입이 가능해진다.
   개발자는 의존성 주입에 대한 관리를 신경쓰지 않아도 된다.

3. 생명주기와 부가 기능 관리
   스프링은 빈에 대한 다양한 관리 기능이 있다.
    - 생성/초기화/소멸 관리
    - 싱글톤 관리
    - AOP 프록시 적용
    - 트랜잭션 처리
    - 테스트 시 교체 용이

빈은 단순 객체가 아니라 스프링 기능이 붙을 수 있는 관리 대상이다.

### Bean의 생성과 소멸까지 - Bean Life Cycle

빈 생명주기란 빈의 생성 - 주입 - 초기화 - 사용 - 소멸 까지의 빈이 살아있는 과정을 말한다. 이 과정에서 스프링은 여러 콜백 지점을 제공한다.

#### 생성
스프링 컨테이너가 BeanDefinition 정보를 보고 객체를 생성한다.  
이때 **생성자 호출** 또는 **팩토리 메소드** 방식으로 인스턴스가 말들어진다.

#### 의존성 주입
@Autowired, 생성자 주입, setter 주입 등을 통해 필요한 의존 객체가 주입된다. 객체는 먼저 만들어지고, 그 다음 필요한 협력 객체들이 연결된다.

#### Aware 인터페이스 콜백
빈이 BeanNameAware, ApplicationContextAware 같은 Aware 인터페이스를 구현했다면, 컨테이너가관련 정보를 주입한다. 예를 들어 빈 이름이나 ApplicationContext를 알 수 있게 된다.

#### BeanPostProcessor - 초기화 전
BeanPostProcessor.postProcessBeforeInitialization()이 호출된다. 여기서 빈을 가공하거나 추가 처리를 할 수 있다. @PoscConstruct도 내부적으로 이런 메커니즘으로 동작한다.

**BeanPostProceesor를 사용하는 이유**  
빈이 생성되고 의존성 주입까지 끝난 뒤에 그 빈을 공통 규칙으로 가공하거나 감싸거나 추가 기능을 붙일 수 있게 하기 위해서이다. BeanPostProcessor를 새로 만들어진 빈 인스턴스를 커스터마이징할 수 있는 factory hook으로 설명하고, 예시로 marker inteface 검사나 프록시 wrapping을 든다.

스프링이 빈을 만들 때, 단순히 객체만 생성하면 끝나는 경우도 있지만 실제 프레임워크는 그보다 더 많은 일을 한다.

- @Autowired 붙은 필드나 메서드에 의존성 주입
- @PostConstruct 호출
- @PreDestory 같은 공통 어노테이션 처리
- 원본 빈을 최종 빈으로 변경 가능 (AOP, Transactional 등)
- 특정 규칙을 만족하는 빈만 공통 래핑

이런 작업은 각 빈 클래스 안에 직접 넣는게 아니라, 컨테이너가 공통적으로 처리하는 편이 낫다. 실제로 스프링은 내부적으로 여러 lifecycle callback과 공통 어노테이션 처리를 BeanPostProcessor 구현체들로 수행한다고 문서에 나와있다.

#### 초기화 콜백
의존성 주입이 끝난 뒤 초기화 로직이 실행된다.

- @PostContruct
- InitializaingBean.afterPropertiesSet()
- @Bean(initMethod = "...") 또는 XML의 ini_method

스프링 공식 문서는 @PostConstruct를 권장한다.  
이유는 스프링 인터페이스에 직접 결합되지 않기 때문이다.

#### BeanPostProcessor - 초기화 후
BenaPostProcessor.postProcessAfterInitialization()이 호출된다.  
AOP 프록시 생성도 보통 이 단계 이후에 적용된다.  
즉, 실제 타깃 빈이 만들어진 뒤 프록시가 깜사는 구조로 이해하면 된다.

#### 빈 사용
이후 애플리케이션은 컨테이너가 관리하는 빈을 사용한다.  
싱글톤이면 같은 인스턴스를 계속 재사용하고, 프로토타입이면 요청할 때마다 새 개게를 받는다.

#### 소멸 콜백
컨테이너 종료 시점에 종료 메소드가 호출된다.

- @PreDestroy
- DisposableBean.destroy()
- @Bean(destroyMethod = "..." 또는 XML의 destory-method

단, 이 소멸 콜백은 모든 스코프에 동일하게 자동 보장되지 않는다. 특히 protorype은 다르게 동작한다.

![빈 생성 다이어그램](https://suplab-bucket.s3.ap-northeast-2.amazonaws.com/images/suplab-bucketfa12a05c-3efd-475f-b102-a519e47dd7d2-image.png)

### 흐름 정리

정확히 정리하면 보통 이렇게 이해하면 된다.
```
	1.	빈 인스턴스 생성
	2.	의존성 주입
	3.	Aware 콜백
	4.	postProcessBeforeInitialization()
	5.	@PostConstruct / afterPropertiesSet() / init-method
	6.	postProcessAfterInitialization()
	7.	필요 시 프록시로 감싸짐
	8.	최종 빈 사용
	9.  빈 destroy()
```

## Bean Scrope
빈 스코프(scope)는 스프링 컨테이너가 해당 빈 인스턴스를 몇 개 만들고, 어느 범위까지 유지하는 지 정의하는 개념이다. 스프링 공식 문서에 나와있는 기본 스코프는 Singleton이고, 웹 환경에서는 추가 스코프가 제공된다고 나와있다.

### 싱글톤(Singleton)
스프링 컨테이너당 해당 빈 인스턴스를 하나만 생성해서 공유한다.

- 가장 많이 사용되며 여러 곳에서 같은 객체를 사용한다.
- 상태를 가지면 동시성 문제가 생길 수 있어 보통 **stateless** 로 설계한다.

```
@Service
@Scope("singleton")
public class OrderService {
}
```
기본이라 @Scope 하지 않아도 된다.

### 프로토타입(Prototype)
빈을 조회할 때마다 새로운 객체를 생성한다.

- 상태를 가지는 객체에 쓸 수 있다.
- 스프링이 생성까지만 관리하고, 이후 소멸은 기본적으로 직접 관리하지 않는다.
- 그래서 @PreDestory 같은 종료 콜백이 자동으로 기대한 대로 관리되지 않는다.

!!! warn "주의할 점"
싱글톤 빈 안에 프로토타입 빈을 그냥 주입하면, 주입 시점에 한 번만 만들어져서 사실상 같은 객체처럼 동작할 수 있다. 이 경우 ObjectProvider, Provider, lookup method 같은 방식으로 필요할 때마다 조회해야 한다.

```
@Component
@Scope("prototype")
public class TempOb₩ject {
}
```

### Reqest
HTTP 요청 1개당 빈 1개를 생성한다. 웹 환경에서만 가능하다.

```
@Component
@RequestScope
public class RequestLog {
}
```

### Session
HTTP 세션당 빈 1개를 생성한다.

```
@Component
@SessionScope
public class UserSessionData {
}
```

### application
ServletContext 기준으로 1개를 생성한다. 웹 애플리케이션 전체 범위에서 공유된다.

### WebSocket
WebSocket 세션 단위로 빈을 관리한다. 웹소켓 연결 생명주기에 맞춰 유지된다.

### Thread Scope
공싱 문서상 사용 가능하지만 기본 등록은 되어 있지 않다. 필요하면 별도 등록해서 사용해야 한다.

## Life Cycle과 Scope
이 둘은 따로 외우면 혼동되기 쉽고, 같이 묶어서 이해해야 한다

- life cycle : 빈이 언제 생성되고 언제 초기화되고 소멸되는가
- scope : 그 빈이 몇 개 생성되고 살아있는 범위를 정의

예시
- singleton : 컨테이너 시작 시 생성되고 컨테이너 종료시 종료
- prototype : 요청할 때마다 생성되지만, 생성 이후 소멸 관리는 스프링이 끝까지 책임지지 않음
- request : HTTP 요청시 생성되고 HTTP 응답이 완료되면 소멸
- session : 사용자 세션 동안 유지


## 추가로 알아야 하는 것들

### 왜 대부분 싱글톤인가?

스프링은 컨트롤러, 서비스, 리포지토리 같은 핵심 빈을 싱글톤으로 관리한다. 이들은 stateless이기 때문에 객체를 새로 만들 필요가 없고, 컨테이너 기반 DI와 잘 맞기 때문이다.

즉, 싱글톤은 공유되는 변경 가능한 상태값이 없으면 위험하지 않다.
그래서 싱글톤 빈은 보통 상태를 저장하지 않는 무상태 객체이다.

### 프로토타입이 많이 쓰이지 않는 이유
필요할 때만 제한적으로 사용하는데, 생성 이후 관리가 복잡하고, 싱글톤과 같이 사용할 경우 의도와 다른 동작을 하기 쉽기 때문이다.

### 싱글톤과 application 차이
**공통점**
- 둘 다 보통 하나만 생성되어 공유됨
- 웹 앱에서는 둘 다 거의 앱 전체 동안 살아 있는 것처럼 보인다.

**차이점**
- 싱글톤은 스프링 컨테이너 생명주기
- application은 ServletContext 생명주기

그래서 application 스코프 빈은 웹 환경에서만 의미 있다. 또한, 저장 위치 개념도 다르다고 설명된다.

- singleton : 스트링 컨테이너의 싱글톤 캐시에 저장
- application : ServletContext attribute 처럼 웹 애플리케이션 범위에 연결

### AOP 프록시는 생명주기 어디쯤인가
공식 문서 기준으로 target 빈이 먼저 생성되고 초기화된 후에 프록시가 적용되는 방식이라고 나와있다. 그래서 초기화 메소드는 보통 원본 빈 기준으로 실행된다고 보는 것이 맞다고 한다.

## Bean 생명주기 코드로 살펴보기
```
@Component
public class OrderService implements InitializingBean {

    public OrderService() {
        System.out.println("1. 생성자");
    }

    @Autowired
    public void setRepository(OrderRepository repository) {
        System.out.println("2. 의존성 주입");
    }

    @PostConstruct
    public void postConstruct() {
        System.out.println("4. @PostConstruct");
    }

    @Override
    public void afterPropertiesSet() {
        System.out.println("5. afterPropertiesSet");
    }

    @PreDestroy
    public void preDestroy() {
        System.out.println("8. @PreDestroy");
    }
}
```



---
## 출처
- [스프링 공식문서](https://docs.spring.io/spring-framework/reference/core/beans/introduction.html)
