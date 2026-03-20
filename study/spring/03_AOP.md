# 스프링 핵심 - AOP (Aspect Oriented Programming)

## AOP란?
### 정의
스프링 AOP를 사용하면 반복되는 공통 코드(로그, 트랜잭션 등)를 비즈니스 로직에서 분리해 한곳에서 관리할 수 있다.  
관점(Aspect) 기준으로 프로그래밍하는 기법이라 하여, 보통 코드를 작성할 때에는 크게 두가지로 나뉘는데 핵심 관심사, 횡단 관심사라 한다.

- 핵심 관심사 : 주문하기, 회원가입하기 등 실제 비즈니스 로직
- 횡단 관심사 : 로그 남기기, 실행 시간 측정, 트랜잭션 관리, 보안 등 비즈니스 로직에 공통으로 들어가는 기능

AOP는 이 횡단 관심사를 핵심에서 분리하는 기능을 한다.  
AOP가 없으면 어떤 문제점이 있는지 알아보자

### 1. AOP가 없다면?
예를 들어 서비스 메서드마다 아래와 같은 코드가 반복된다고 하자.

```
public void order() {
    log.info("start");
    long start = System.currentTimeMillis();

    // 핵심 비즈니스 로직

    long end = System.currentTimeMillis();
    log.info("end = {}", end - start);
}
```
문제는 다음과 같다.

- 핵심 로직과 부가 로직이 섞인다.
- 같은 코드가 여러 클래스에 반복된다.
- 그러므로 수정시 여러 곳을 같이 수정해야 한다.

AOP는 이러한 문제들을 해결할 수 있다.

### 2. AOP가 있다면?
```
@Service
public class OrderService {
	public void order() {
		// 로그 start
		// 핵심 비즈니스 로직
		// 로그 end
	}
}
```
코드는 사라지지만, 위와 똑같이 비즈니스 로직 전후로 로깅 기능이 수행된다.
즉, ```//로그 start, end```라고 주석이 있지만 그 부분은 사라지고 핵심 비즈니스 로직 코드만 남게 된다.

#### 예제로 보면
```
@Aspect
@Component
public class LogAspect {

    @Before("execution(* com.example.service.OrderService.order(..))")
    public void log() {
        System.out.println("before order");
    }
}
```

```
@Service
public class OrderService {

    public void order() {
        System.out.println("order");
    }

    public void cancel() {
        System.out.println("cancel");
    }
}
```
```LogAspect``` 라는 AOP 클래스를 만들어주고 어노테이션(**PointCut**)을 지정해주면 스프링 AOP가 자동으로 해당 메서드 실행시 AOP(```log()```)를 적용해준다.

## 그렇다면 어떻게 AOP가 가능할까?
AOP를 이해하련 프록시(Proxy) 패턴을 알아야 한다. 프록시 패턴이란 어떤 객체를 사용하고자 할 때, 객체를 직접 참조하는 것이 아니라 그 객체를 대행하는 객체(Proxy)를 통해 접근하는 설계 패턴이다. 말이 좀 어려운데 원본 객체 앞에 다른 **대리 객체**(원본 객체를 실행)가 하나 더 있다고 생각하면 된다.

프록시 패턴이 왜 필요할까?

- 제어 : 권한이 있는 사용자만 원본 객체를 사용하게 제한하고 싶을때
- 지연로딩 : 생성 비용이 매우 비싼 객체(큰 이미지, 무거운 DB 연결 등)를 실제로 사용할 때까지 생성을 미루고 싶을 때
- 부가 기능 추가 : 원본 코드에 로그를 남기거가 실행 시간을 측정하는 코드를 넣고 싶을 때

결국은 책임 분리이다. 핵심 로직과 부가 기능 로직을 분리하기 위한 용도라고 볼 수 있다.

### 프록시 패턴 흐름
프록시 패턴이 성립하려면 원본 객체와 프록시 객체가 같은 인터페이스를 구현해야 한다.
그렇게 해야 클라언트가 자기가 쓰는 게 원본인지 프록시인지 모른 채 사용할 수 있다.

```
[클라이언트] ----> [인터페이스(Subject)]
                         ^
                         |
           +-------------+-------------+
           |                           |
    [프록시(Proxy)] ----------> [원본 객체(RealSubject)]
      (부가 기능 수행)           (핵심 로직 수행)
```

#### 이미지 예시 (java)

```
public interface Image {
    void display();
}
```

```
public class RealImage implements Image {
    private String fileName;

    public RealImage(String fileName) {
        this.fileName = fileName;
        loadFromDisk(); // 생성 시점에 무거운 로딩 발생
    }

    private void loadFromDisk() {
        System.out.println("디스크에서 " + fileName + " 로딩 중... (무거운 작업)");
    }

    @Override
    public void display() {
        System.out.println("화면에 " + fileName + " 출력");
    }
}
```


```
public class ProxyImage implements Image {
    private RealImage realImage;
    private String fileName;

    public ProxyImage(String fileName) {
        this.fileName = fileName;
    }

    @Override
    public void display() {
        // 실제로 보여줄 때만 원본 객체를 생성함 (지연 로딩)
        if (realImage == null) {
            realImage = new RealImage(fileName);
        }
        System.out.println("보안 검사 및 로그 남기기..."); // 부가 기능
        realImage.display();
    }
}
```

프록시 패턴은 단순히 '대신 전달한다'는 의미를 넘어, 객체 지향 원칙 중 **OCP** 원칙을 지키는 패턴이다.

스프링 AOP는 이런 프록시 패턴을 사용하여 구현되고 있고, 자동으로 만들어지고 사용된다.

## AOP 동작 과정
프록시 패턴이라는 방식은 이해했으니, 이제 스프링이 프록시를 어떻게 사용하는지 알아보자.

### 전체 흐름
전체 흐름은 다음과 같다.

```
@Bean / @Component 스캔
        ↓
원본 빈 생성
        ↓
BeanPostProcessor가 검사
        ↓
AOP 적용 대상이면 프록시 생성
        ↓
컨테이너에는 원본 대신 프록시 빈이 등록됨
        ↓
클라이언트는 프록시를 주입받아 사용
        ↓
메소드 호출시 포인트 컷 대상 메소드인지 확인
        ↓
포인트 컷 매칭인 경우 Interceptor/chain 구성
        ↓
구성된 Advice 목록을 차례대로 호출
        ↓
실제 메소드 호출
        ↓
Advice 목록을 거꾸로 다시 타고 올라가며 반환
```

### 프록시 생성하기
Spring AOP에서 프록시는 개발자가 직접 ```new``` 하듯이 만들어지지 않는다.
스프링 컨테이너가 빈을 생성하는 과정 중간에, AOP 대상이면 원본 객체 대신 프록시 객체를 만들어 빈으로 등록한다.

그래서 우리가 ```@Autowired```로 받는 객체는 겉보기엔 **OrderService** 같아도, 실제로는 프록시인 경우가 많다.

프록시 객체를 만드는 기능은 Spring의 ```BeanPostProcessor```에 의해서 프록시 객체인지 확인하고 생성하게 된다.
> 스프링에는 빈 생성 전후에 후처리를 할 수 있는데, 공식 문서에도 **Auto-proxy 기능**이 이 지점에서 동작한다고 설명한다. (bean post processor infrastructure)

다음은 **BeanPostProcessor** 구현체가 프록시를 빈으로 등록하는 부분이다.
```
public abstract class AbstractAutoProxyCreator extends ProxyProcessorSupport
		implements SmartInstantiationAwareBeanPostProcessor, BeanFactoryAware {
		
	protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
			// 중략...
			
			// Create proxy if we have advice.
			Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
			if (specificInterceptors != DO_NOT_PROXY) {
				this.advisedBeans.put(cacheKey, Boolean.TRUE);
				Object proxy = createProxy(
						bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
				this.proxyTypes.put(cacheKey, proxy.getClass());
				return proxy;
			}
	}
}
```

빈은 직접 판단하지 않고, 아래 추상 메서드에 위임한다.
```getAdvicesAndAdvisorsForBean(....)```



스프링은 **Aspect + Pointcut + Advice** 정보를 바탕으로, 특정 빈의 특정 메서드에 AOP를 판단한다.

```@Aspect, @Arount, @Before``` 어노테이션 등이 이에 해당한다.

예시 : ```@Around("execution(* com.example.service..*(..))")```
-> **service** 패키지의 빈 메서드들을 프록시로 등록한다.

Spring AOP의 **auto-proxy**는 **"selected bean definitions"** 를 자동 프록시로 감싼다고 설명한다.

```
@Aspect
@Component
public class LogAspect {

    @Around("execution(* com.example.service..*(..))")
    public Object log(ProceedingJoinPoint joinPoint) throws Throwable {
        System.out.println("before");
        try {
            return joinPoint.proceed();
        } finally {
            System.out.println("after");
        }
    }
}
```

```
@Service
public class OrderService {
    public void order() {
        System.out.println("주문 처리");
    }
}
```

이 경우 스프링은 OrderService를 ```order()``` 호출을 가로챌 수 있는 프록시를 만든다.

그리고 스프링 AOP는 프록시 객체를 생성할 때 JDK 동적 프록시, CGLIB 두 가지 방식을 사용한다.

**JDK 동적 프록시 방식**
- 대상이 인터페이스를 구현하면 사용 가능
- 인터페이스 타입 기준으로 프록시 생성

**CGLIB 프록시**
- 대상 클래스를 상속한 서브클래스 형태로 프록시 생성
- 인터페이스가 없어도 가능
- 다만 final class, final method 같은 상속 불가 요소는 제약이 생긴다. 공식 문서도 CGLIB는 클래스 프록시용이며, 프록시 제약이 있다고 설명한다.

스프링은 인터페이스가 있으면 JDK 동적 프록시를 없으면 CGLIB를 채택하지만,
스프링 부트는 기본으로 CGLIB 프록시를 사용한다.

또한, 설정을 통해서 동작 방식을 변경할 수 있다.

```
spring.aop.proxy-target-class=false
```
```application.yml```에 해당 설정을 하면, JDK 프록시를 사용할 수 있다.   
그리고 AspectJ가 클래스패스에 있으면 @EnableAspectJAutoProxy를 별도로 쓰지 않아도 자동으로 활성화 된다.

### 프록시 호출
위와 같은 과정으로 프록시를 생성하게 되면 어노테이션에 기술한 PointCut에 맞는 메서드 호출시 빈으로 등록한 프록시를 호출하게 된다.

```
Controller
   ↓
OrderService 프록시
   ↓   [Advice 실행]
실제 OrderService 대상 객체
```
프록시는 메서드 호출을 먼저 받고, 적용할 Advice 체인을 실행한 뒤, 마지막에 실제 대상 메서드를 호출한다.  예를 들어 @Around면 내부적으로 다음과 같다.

```
proxy.order()

// 프록시 내부
beforeAdvice();
target.order();
afterAdvice();
```

!!! info "포인트 컷에 해당하지 않는 메소드는?"
포인트 컷에 해당하지 않는 메소드는 프록시 객체를 거치지만 ```interceptor/advice``` 체인에 들어가지 않는다. 매칭되지 않은 메서드는 별도 어드바이스 없이 바로 타킷 메소드로 위임된다. Spring AOP는 Advisor를 기반으로 프록시를 만들고, 메소드 매칭은 poincut의 maches(Method, Class)같은 방식으로 판단한다.

	```
	빈이 AOP 후보인가?  → yes → 프록시 생성
	현재 호출한 메서드가 포인트컷에 맞는가? → yes면 advice 실행, no면 그냥 통과
	```
실제 호출 시 그 advisor 안의 Method/matcher/poincut 으로 현재 호출 메소드가 적용 대상인지 다시 판단한다. 그리고 interceptor/chain을 구성하여 advice 목록을 구성한다.

### Spring AOP 문제점
Spring AOP는 프록시 기반이기 때문에 다음과 같은 문제점이 발생한다.

#### 1. self-Invocation 문제
같은 클래스 내에서 ```@Trasactional```이 없는 메소드가 있는 메소드를 호출하면 트랜잭션이 적용되지 않는다.

```
@Service
public class OrderService {

    public void createOrder() {
        validate(); // 내부 호출 -> 프록시 안 탐
    }

    @Transactional
    public void validate() {
        // 트랜잭션 기대했지만 안 걸릴 수 있음
    }
}
```
**왜 AOP가 동작하지 않을까?**  
AOP의 동작 방식을 다시 살펴보자.  
AOP는 적용 대상에 실제 객체 대신 프록시 객체를 사용한다고 했다. 외부에서 AOP 대상 메소드를 호출하면 프록시가 가로채어 부가 기능을 수앵한 후 실제 객체를 호출하는 방식이다.

즉, 프록시 객체가 실제 객체를 감싸는 형태가 된다. 그리고 한 가지 더해서 AOP는 메서드를 호출하기 전에AOP 대상 메서드인지 확인하는 검증 단계를 거치게 된다.

그에 따라서 advice가 호출 되거나 그렇지 않게 되는데, AOP 대상이 아닌 메소드도 프록시 객체 안에서 이미 검증 단계를 거치고 들어온 상태이다.

그렇기 때문에 그 안에서 다시 대상 메소드를 호출시에는 this.method()로 호출 되기 때문에 AOP가 동작하지 않게 된다.

즉, 내부에서 Transactional이 걸린 메소드를 호출해버리는 케이스를 조심해야 한다.


#### 2. **메소드 실행에만 적용 가능**
Spring AOP는 메소드 실행 join point만 지원한다. 즉, 생성자 호출, 필드 접근, 객체 생성 시점, static 초기화 같은 지점에는 일반 Spring AOP만으로 적용할 수 없다. 그럴 경우 AspectJ가 필요하다.

#### 3. **private / final 관련 제약**
JDK 프록시는 인터페이스 기반, CGLIB는 클래스 상속 기반이기 때문에 final class/method, private 메서드에는 적용되지 않는다.

#### 4. **Spring Bean에만 적용된다.**
Spring Container가 관리하는 빈에 대해서만 프록시를 붙인다. 직접 new로 객체를 생성한 경우 AOP가 적용 되지 않는다.

## 실제 사용해보기

### 용어 알기

| 용어 | 영문명 | 설명 | 비유 (요리사 예시) |
| :--- | :--- | :--- | :--- |
| **타겟** | **Target** | 부가 기능을 부여할 대상 (핵심 로직을 담은 클래스/메서드) | **요리 재료** (고기, 채소 등) |
| **애스펙트** | **Aspect** | 여러 곳에 적용되는 공통 관심사를 모듈화한 것 (Advice + Pointcut) | **요리 레시피 꾸러미** |
| **어드바이스** | **Advice** | 실질적으로 수행할 부가 기능 코드와 실행 시점 (Before, After, Around 등) | **조리법** (언제 소금을 뿌릴지) |
| **조인 포인트** | **Join Point** | 어드바이스가 적용될 수 있는 모든 실행 지점 (메서드 실행, 생성자 호출 등) | **조리 과정 중 모든 순간** |
| **포인트컷** | **Pointcut** | 조인 포인트 중에서 실제로 어드바이스를 적용할 지점을 선별하는 규칙 | **"고기를 굽기 직전"**이라는 특정 규칙 |
| **위빙** | **Weaving** | 포인트컷으로 지정된 타겟에 어드바이스를 적용하여 프록시를 생성하는 과정 | **실제로 재료에 양념을 하는 행위** |
| **어드바이저** | **Advisor** | 하나의 어드바이스와 하나의 포인트컷으로 구성된 가장 단순한 애스펙트 | **단순한 조리 팁 하나** |


### @어노테이션

| 어노테이션 | 역할 | 실행 시점 및 특징 |
| :--- | :--- | :--- |
| **@Aspect** | **Aspect 선언** | 해당 클래스가 에스펙트(공통 로직 담은 클래스)임을 명시한다. |
| **@Before** | **Advice (이전)** | 대상 메서드가 실행되기 **전**에 부가 기능을 실행한다. |
| **@After** | **Advice (이후)** | 대상 메서드의 결과와 상관없이(성공/예외) 실행 후 **무조건** 실행한다. |
| **@AfterReturning** | **Advice (정상 종료)** | 대상 메서드가 **성공적으로 반환**된 경우에만 실행한다. (결과값 참조 가능) |
| **@AfterThrowing** | **Advice (예외 발생)** | 대상 메서드 실행 중 **예외가 발생**했을 때만 실행한다. (예외 객체 참조 가능) |
| **@Around** | **Advice (전후)** | 메서드 실행 전후, 예외 발생 시점 등 **모든 시점**을 제어한다. |
| **@Pointcut** | **지점 정의** | 공통적으로 사용될 포인트컷 표현식을 변수처럼 정의해 재사용할 때 쓴다. |

### 예시
```
@Aspect
@Component
public class MyAspect {

    // 포인트컷 재사용을 위해 정의
    @Pointcut("execution(* com.example.service..*(..))")
    private void serviceMethods() {}

    @Before("serviceMethods()")
    public void doBefore() {
        System.out.println("비즈니스 로직 실행 전!");
    }

    @Around("serviceMethods()")
    public Object doAround(ProceedingJoinPoint joinPoint) throws Throwable {
        // 전처리
        Object result = joinPoint.proceed(); // 실제 메서드 호출
        // 후처리
        return result;
    }
}
```

1. 클래스 레벨 선언  
   @Aspect 이 클래스는 공통 관심사(횡단 관심사)를 모듈화한 에스펙트 클래스다"라고 스프링에게 알려주는 핵심 어노테이션이다.

2. 메소들 레벨 선언  
   @PointCut 어디에 이 기능을 적용할지 필터링하는 규칙을 정하는 곳. 위에 예시처럼 한 곳에 규칙을 정의 해두면 아래 ```Before```, ```Around``` 에서 해당 규칙을 재사용할 수 있다.

**실행 시나리오**

1. ```doAround```의 전처리 부분이 시작
2. ```doBefore```가 호출
3. 실제 비즈니스 로직 호출 ```joinPoint.proceed()```
4. doAround의 후처리 부분 실행

---
## 출처
- [스프링 공식 문서](https://docs.spring.io/spring-framework/reference/core/aop/introduction-proxies.html?utm_source=chatgpt.com)

- chatGPT, 제미나이