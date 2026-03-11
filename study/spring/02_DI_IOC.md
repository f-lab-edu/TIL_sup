# IoC 컨테이너
## 개요
스프링에서 객체는 Bean(빈)이라고 한다. 객체 생성을 "빈을 생성하다"라고 하는데, IoC 컨테이너는 이러한 빈 생성, 객체 간의 구성 및 조립을 담당한다. 컨테이너는 의존 관계 등을 나열한 구성 메타데이터를 읽어 빈 생성, 의존 관계 정보 등을 알 수 있다. 구성 메타데이터는 어노테이션, 팩토리 메서드, 외부 XML 파일 또는 Groovy 스크립트로 표현될 수 있다.

좀 더 구체적으로 IoC 컨테이너가 어떻게 빈을 인스턴스화하고, 다른 빈에 주입할 수 있는 이유는 DI와 IoC의 개념을 알아야 한다.

### DI (Dependency Injection)
DI는 어느 객체가 다른 객체를 사용해야할 때 객체 생성을 내부가 아닌 외부에 위임하여 객체의 생성자 인자, 팩토리 메서드 인자 또는 객체 인스턴스 생성/반환 후 설정되는 프로퍼터(Setter)를 통해서만 의존성을 정의하는 방식이다. 간단하게 생각하면 외부 인자를 통해서 들어온 객체를 내부 인스턴스 변수에 할당하는 것을 말한다.

#### 생성자 인자 DI
```
public class UserService {
    private final UserRepository userRepository;

    // 생성자를 통해 의존성 주입
    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }
}
```

코드 예시와 같이 객체의 생성자 인자로 객체를 받아서 내부 변수에 할당하는 방식이다. 이렇게 하변 빈의 불변성을 보장할 수 있다.

#### Factory Method 기반 DI
```
public class ServiceFactory {
    public static UserService createUserService(UserRepository repository) {
        return new UserService(repository); // 팩토리 메서드 내에서 의존성 주입
    }
}
```

객체 생성을 외부 Factory Class에 위임하는 방식이다. 예시와 같이 별도의 팩토리 클래스나 메서드를 통해 빈을 생성하고 의존성을 주입하게 된다. 주로 외부 라이브러리나 복잡한 생성 과정이 필요한 경우 사용한다.

#### Setter 기반 DI
```
public class UserService {
    private UserRepository userRepository;

    // 객체 생성 후 프로퍼티 설정
    public void setUserRepository(UserRepository userRepository) {
        this.userRepository = userRepository;
    }
}
```

객체 생성 후, setter 메서드를 통해서 의존성을 주입한다. 생성 후에 의존성을 변경해야 하거나 선택적인 의존성이 있을 때 사용한다.

#### DI 장점
1. 객체 간의 결합도를 낮춘다
   클래스가 직접 의존 객체를 생성하지 않으므로, 의존하는 클래스의 구현이 바뀌어도 해당 클래스를 수정할 필요가 없다.

2. 테스트 용이성
   의존하는 객체를 인터페이스로 관리하면, 테스트 시 실제 객체 대신 Mock 객체를 쉽게 사용할 수 있어 단위 테스트가 용이해진다.

3. 중복 코드 제거
   객체 생성을 위한 중복된 ```new``` 키워드와 복잡한 설정 코드들이 대폭 줄어든다.

### 스프링에서 DI가 필요한 이유
IoC는 프로그램의 제어 흐름을 개발자가 아닌 외부가 결정하는 것을 의미하는데 외부에서 의존성을 주입하는 방식인 DI는 IoC를 실현하기 위한 핵심 수단이 된다.

!!! info "제어의 흐름을 외부로 둔 이유?"
1. **객체 관리 자동화**  
수많은 객체를 일일이 개발자가 생성하고 관리한다면 비효율적이고, 실수가 잦아지게 된다. 스프링 컨테이너가 객체의 생명주기를 책임짐으로써, 개발자는 비즈니스 로직에 집중할 수 있게 된다.
2. **설정 분리**  
코드 내부에 의존 관계를 하드 코딩하지 않고, 설정 정보를 통해 외부에 주입하여 객체 간의 관계를 런타임에 변경할 수 있게 된다.
3. **트랜잭션, 로깅 등 AOP**
IoC 컨테이너와 프록시 패턴을 사용하면 AOP가 가능하게 된다. 스프링은 개발자가 직접 ```new```로 생성한 객체를 사용하는 대신, 프록시라는 대리인을 만들어서 대신 주입한다. 이 프록시 객체가 진짜 비즈니스 로직을 호출하는 것이다.

## IoC란? - 제어 역전
결국 IoC는 스프링이 제어권을 가지게 되는 것인데, 의존 방향을 생각해야 한다.

객체가 new 생성하여 다른 객체를 생성하는 방식과 DI를 통한 방식을 다시 한번 보자.

### **객체가 내부에서 객체를 생성한 방식**
```
+----------------+        +--------------------------+
|  OrderService  | ---->  |   MemoryOrderRepository  | (직접 생성: new)
+----------------+        +--------------------------+
      ^
      | (OrderService가 Repository의 구체적인 타입까지 알아야 함 -> 강한 결합)
```
화살표가 의존방향이며, OrderService가 MemoryOrderRepository를 바라보고 있다.

이 구조에서 문제점은 다음과 같다.

1. OrderService가 MemoryOrderRepository라는 구체 클래스를 알고 있다.
    - OrderService는 구현제에 의존하게 되어 강한 결합이 된다.

이러한 경우 MemoryOrderRepository가 아닌 다른 Repsitory 구현체로 변경한다면
OrderService 내부 코드가 변경된다. (수정 영향을 받게 됨)

### **DI 적용후 의존 방향**

```
+----------------+
|  OrderService  |
+--------+-------+
         |
         v
+----------------------+
|   OrderRepository    | (Interface)
+----------+-----------+
           ^
           |
+----------+-----------+
| MemoryOrderRepository|
+----------------------+
```
DI를 적용하게 되면 의존 방향이 구현체가 아니라 추상화(인터페이스)로 향한다.
여기서 핵심은 OrderService는 구현체를 알 수 없다.

OrderService의 의존 방향은 OrderRepository 인터페이스로 향한다.
```
OrderService → OrderRepository (Interface)
```

이렇듯 의존 방향이 역전되는 것을 DIP라고 한다.


### **스프링 컨테이너의 역할 - IoC**
```
+-----------------------+
|  Spring Container     | (객체 생성, 관리, 주입 담당)
+-----------+-----------+
            |
      (주입: Injection)
            v
+----------------+        +--------------------------+
|  OrderService  | ---->  | <<Interface>>            |
+----------------+        |  OrderRepository         |
                          +------------+-------------+
                                       ^
                                       | (구현체 교체 용이)
                          +------------+-------------+
                          | MemoryOrderRepository    |
                          +--------------------------+
```
DI가 적용된 곳에 추가로 스프링 컨테이너가 의존성을 주입해주는 것을 볼 수 있다. DI가 됨으로써 외부에서 구현된 객체를 주입할 수 있게 되었고, 그 역할을 스프링 컨테이너가 하고 있다.

즉, IoC란 객체 생성과 의존관계 설정에 대한 **제어권**을 애플리케이션 코드에서 프레임워크(스프링 컨테이너)로 넘기는 설계 원칙이며, 스프링에서는 이를 DI를 통해 구현한다.



---



## 출처
- [스프링 공식 문서](https://docs.spring.io/spring-framework/reference/core/beans/basics.html)
