# CGLIB VS JDK Dynamic Proxy
Spring AOP는 런타임 브록시 기반이다. 즉, 실제 객체를 직접 호출하는게 아니라 프록시가 먼저 호출을 받고, 드 귀에 Advice를 실행한 후 타깃 메소드를 호출한다. 이 과정 중 프록시를 생성할 때 두 가지 방식인 CGLIB와 JDK Dynamic Proxy를 사용한다.

- CGLIB : 클래스의 바이트코드를 조작하여 Proxy 객체를 생성해주는 방식
- JDK Dynamic Proxy : Interface를 기반으로 Proxy를 생성해주는 방식

이 두 가지 생성 방식과 차이점을 알아야 Spring AOP의 한계도 같이 이해하게 된다.
- @Transactional이 어떤 상황에서 동작이 수행되지 않는지 알 수 있다.
- self-invocation에서 AOP가 타지 않는 상황을 이해할 수 있다.
- private 메서드에 프록시는 왜 적용이 안되는지 알 수 있다.

Spring 문서는 프록시 기반 특성 때문에 타깃 객체 내부 호출은 가로채지 못한다고 설명한다. 또한 JDK 프록시는 프록시의 public interface 메소드 호출만, CGLIB는 public/protected 및 필요 시 package-visible 호출까지 가로챌 수 있다고 설명한다.

## JDK Dynamic Proxy
JDK에서 제공하는 Dynamic Proxy는 인터페이스를 기반으로 프록시를 생성한다. 인터페이스 판단 유무를 거쳐 인터페이스가 존재하면 JDK Dynamic Proxy 방식으로 프록시 객체를 생성할 수 있다.

- JDK Dynamic Proxy는 리플레션을 사용해서 동적 프록시 객체를 생성한다.
- JDK Dynamic Proxy는 인터페이스 메소드 호출시 하나의 InvocationHandler로 위임을 한다.
- JDK Dynamic Proxy는 JDK 1.3 버전부터 도입되었다.

### JDK 핵심 구성요소

#### 인터페이스
프록시는 구현 클래스가 아니라 인터페이스 기준이고, 자바 공식 문서도 프로기 클래스는 런타임에 지정된 인터페이스들의 구현체라고 설명한다.

#### 프록시
프록시 클래스를 만들고 인스턴스를 생성하는 진입점이다. 보통 Proxy.newProxyInstance()를 사용한다. 이 메소드는 내부적으로 프록시 클래스 생성 + 생성자 호출+ 핸들러 연결을 한번에 처리한다.

#### InvocationHandler
프록시로 들어온 모든 메소드 호출을 한 곳에서 받는 핸들러이다. 메소드 이름, 파라미터, 리턴 처리, 예외 처리까지 여기서 제어한다.

### 동작 원리

#### 생성 예시
```
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;

public class Main {
    public static void main(String[] args) {
        OrderService target = new OrderServiceImpl();

        OrderService proxy = (OrderService) Proxy.newProxyInstance(
                OrderService.class.getClassLoader(),
                new Class[]{OrderService.class},
                new InvocationHandler() {
                    @Override
                    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                        System.out.println("before");
                        Object result = method.invoke(target, args);
                        System.out.println("after");
                        return result;
                    }
                }
        );

        proxy.order();
    }
}
```

#### 실행 흐름
```
클라이언트
  ↓
프록시 객체
  ↓
InvocationHandler.invoke(...)
  ↓
target.order()
```

핵심은 프록시가 직접 비즈니스 로직을 처리하는게 아닌, 모든 호출을 InvocationHandler로 위임한다.

### 특징
- 인터페이스가 있어야 한다.
- equals, hashCode, toString도 핸들러로 들어온다.
- 기본 메소드도 고려 대상이다.
  최신 JDK 문서에는 프록시 인터페이스의 default method를 Invocationhandler::invokeDefualt로 호출할 수 있다고 설명한다.

#### 장점
1. JDK 내장된 기능이라 별도 바이트코드 라이브러리 없이 사용할 수 있다.
2. 인터페이스 기반 설계와 자연스럽다.
3. 생성자 이슈가 상대적으로 적다.

#### 단점
1. 인터페이스가 없으면 사용 불가하다.
2. 인터페이스 메소드 범위에서 생각해야 한다.
3. Self-invocation 문제가 있다.
4. CGLIB에 비해서 리플렉션을 사용하기 때문에 느릴 수 있다.
   --> JDK 18부터는 Method Handle 기반으로 재구현되어 리플렉션 반복 호출에 대한 성능이 개선됐다고 할 수 있다. 하지만 여전히 직접 호출보다는 느릴 수 있다.

## CGLIB 방식
CGLIB는 런타임에 바이트코드 조작 라이브러리(ASM)를 사용하여 자바 클래스의 서브클래스를 동적으로 생성하는 기술이다. JDK 동적 프록시와는 다르게 인터페이스 없이 구체 클래스만으로도 프록시 생성이 가능하며, 메소드를 재정의하여 AOP 기능을 구현한다.

즉, 인터페이스가 없는 클래스에 AOP를 적용하기 위해 Spring AOP는 이 방식을 사용한다.

!!! info "바이트코드를 조작한다"
바이트코드 조작은 .java를 고치는게 아니라 .class 파일 내용을 읽어서 일부를 바꾸거나, 아예 새로운 .class를 만들어 JVM에 올리는 것이다. Oracle 문서에서는 클래스는 반드시 파일로 존재할 필요는 없고, 클래스 로더가 생성한 유효한 class file format 표현이어도 된다고 설명한다.

예시를 보자
```
public class PaymentService {
    public void pay() {
        System.out.println("결제 실행");
    }
}
```

```
public class PaymentService$$EnhancerBySpringCGLIB extends PaymentService {
    @Override
    public void pay() {
        System.out.println("before");
        super.pay();
        System.out.println("after");
    }
}
```
CGLIB가 PaymentService를 대상으로 프록시 객체를 생성한 예시이다. 실제 생성 코드는 훨씬 복잡하지만 구조는 이와 같다.

두 코드를 비교로 봐야할 것은
- PaymentService를 상속한다.
- 오버라이드 가능한 메서드만 가로챈다.

스프링 문서에도 동적으로 생성된 서브클래스가 non-final 메서드들을 override하고, 사용자 정의 interceptor로 연결되는 훅을 가진다고 설명한다.

### 조금 더 깊게 알아기

#### Enhancer
CGLIB는 Enhancer 클래스를 사용하여 프록시 객체를 생성한다.
Enhancer는 프록시 클래스를 만드는 공장이라 할 수 있다. 대상 슈퍼클래스, 인터셉터, 콜백 설정을 바탕으로 동적 서브클래스를 생성한다.

#### 예시
```
Enhancer enhancer = new Enhancer();
enhancer.setSuperclass(PaymentService.class);
enhancer.setCallback(new MyMethodInterceptor());

PaymentService proxy = (PaymentService) enhancer.create();
```
PaymentService를 상송하근 새 클래스를 만들고 메서드가 호출되면 이 인터셉터에게 넘기는 설정하고, 프프록시를 생성하게 된다.

#### MethodInterceptor
MethodInterceptor는 프록시 메서드 호출 시 연결되는 콜백 역할을 한다.

```
public class MyMethodInterceptor implements MethodInterceptor {
    @Override
    public Object intercept(
            Object obj,
            Method method,
            Object[] args,
            MethodProxy proxy) throws Throwable {

        System.out.println("before");
        Object result = proxy.invokeSuper(obj, args);
        System.out.println("after");
        return result;
    }
}
```
위에 코드와 같이 되어있다고 보면 되는데, intercept이 하나로 모든 메서드 호출을 제어할 수 있다.
여기서 한가지 더 봐야할 것은 MethodProxy이다.

- **MethodProxy** :  원본 메서드를 빠르게 호출하기 위한 도구

CGLIB는 단순히 자바 리플렉션 Method.invoke()만 쓰는게 아니라, 내부적으로 더 최적한 방식으로 super 메서드를 호출하도록 돕는다.

!!! info "CGLIB 오해 "
CGLIB는 리플렉션을 아예 배제하는 기술이 아니다.
프록시 클래스 생성은 바이트코드 생성 방식으로 하고, 메서드 호출 처리에서는 Method 같은 리플렉션 정보도 사용하지만, 실제 호출은 MethodProxy를 통해 더 빠르게 처리할 수 있다.

### 버전 차이
CGLIB 방식은 버전별 차이는 없다. 가장 큰 차이점은 Spring 3.2 버전 부터 Spring-core 안에 repackaging해서 포함하기 시작했다. 즉, 외부 cglib 의존성을 굳이 별도로 넣지 않아도 Spring AOP의 CGLIB 프록시가 동작하게 됐다.

#### 생서자 2번 호출 이슈

**예전 스프링 4.0 문서**에는 CGLIB 프록시에서 생성자가 두 번 호출될 수 있다고 적혀있다. 당시 문서 설명은 프록시 객체와 실제 타깃 관련 객체 생성 때문에 이런 현상이 자연스럽다는 방향이다.

**현재 문서에는** Objenesis를 사용하기 때문에 보통 생성자가 두 번 호출되지 않는다고 설명한다. 다만 JVM이 constructor bypass를 허용하지 않으면 드물게 이중 호출처럼 보이는 현상이 나타날 수 있다고 안내한다.

> **Objenesis** : 생성자 호출 없이 Java 객체를 인스턴스화할 수 있게 해주는 라이브러리

## 스프링의 프록시 생성 기본값은?

기본 규칙은 이렇다.
- 대상 객체가 인터페이스를 구현하면 → JDK 동적 프록시 가능
- 대상 객체가 인터페이스를 구현하지 않으면 → CGLIB 사용

Spring 공식 문서가 이 규칙을 그대로 설명한다.

추가로 Spring Boot에서는 AOP 자동 설정 시 기본적으로 클래스 기반 프록시(CGLIB) 를 쓰는 쪽으로 동작해, 인터페이스가 있어도 CGLIB가 사용될 수 있다. 이건 Boot 쪽 설정 문제와 연결된다.

- JPA Hibernate에서도 기본적으로 CGLib을 사용한다고 한다.

## 어느 것을 선택해야 할까?
**JDK 동적 프록시를 선호하는 경우**
- 서비스 계약을 인터페이스로 명확히 나누고 싶을 때
- 구현체 타입 의존을 줄이고 싶을 때
- 프록시 제약을 덜 복잡하게 가져가고 싶을 때

**CGLIB를 선호하는 경우**
- 인터페이스가 없을 때
- 레거시 코드라 인터페이스 도입 비용이 클 때
- Boot 기본 설정을 그대로 쓰는 게 더 단순할 때

**Spring boot는?**
Spring boot가 CGLIB를 기본값으로 한 이유는 초기 설정 없이도 더 넓은 범위를 커버할 수 있어서 볼 수 있지 않을까. 공식 문서가 이유를 한 줄로 직접 설명하진 않지만, 프록시 메커니즘 특성과 Boot 기본 설정을 합치면 나올 수 있는 해석이라고 한다.

**Github 이슈에도 관련 논의가 있었다고 한다.**
- Spring Boot팀은 이미 트랜잭션 프록시 쪽에서는 CGLIB를 사용하고 있었는데, 일반 AOP 기본값으 ㄴ여전히 달라서 일관성이 없었고, 다음 버전에서 기본값을 true(CGLIB 기본 설정으로)로 바꾸자고 논의했다. 관련 이슈에 “transactions since 1.4”, “AOP default value change for next version”가 명시돼 있다. [Github 이슈](https://github.com/spring-projects/spring-boot/issues/8786?utm_source=chatgpt.com)

---

## 출처
[Spring 공식 문서](https://docs.spring.io/spring-framework/reference/core/aop/proxying.html?utm_source=chatgpt.com)
