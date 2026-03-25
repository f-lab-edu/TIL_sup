## 개요
지난 글에 Spring AOP의 개념과 문제점에 대해서 알아봤다. 이번 글에서는 Spring 문제점을 어떻게 해결할 수 있을지 @Transactional을 예시로 들며 해결 방식들을 알아볼 것이다.

---

## @Transactional
트랜잭셔널 어노테이션은 스프링에서 트랜잭션 처리를 적용하기 위해 사용하는 어노테이션으로, 해당 어노테이션을 선언하면, 스프링 AOP를 통해서 트랜잭션 처리가 동작하게 된다.

### 트랜잭션이란?
우선 트랜잭션의 개념부터 알아보자.  
트트랜잭션은 데이터베이스 작업을 하나의 논리적 실행 단위로 묶는 개념이다. 가장 중요한 특징은 **"All or Nothing"** 이라 하여, 여러 작업을 전부 성공시키거나 중간에 문제가 생기면 전부 원래대로 원복하는 것이다.

!!! info "예시 : 온라인 쇼핑몰 결제"
	고객이 상품을 결제하는 과정이다.
	1. 고객의 계좌에서 돈이 출금된다.
	2. 쇼핑몰의 데이터베이스에 주문 내역이 생성된다.
	
	만약 1번은 성공했지만 서버 오류로 2번이 실패한다면 고객은 돈만 잃게 된다. 이런 데이터 정합성 문제를 막기 위해, 두 작업을 하나의 트랜잭션 단위로 묶는다. 즉, 둘 다 성공해야지 실제 데이터에 반영(commit)이 되고 하나라도 실패하면 이전 상태로 돌아가게 된다.(rollback)
	
즉, @Transactional은 트랜잭션 기능을 적용한다는 마커의 개념과 트랜잭션 속성을 선언하는 어노테이션이라고 보면 된다.  
과거에는 개발자가 직접 DB 커넥션을 가져와서 ```setAutoCommit(false)```를 하고, ```commit()```이나 ```rollback()```을 필요한 모든 곳에 작성해줘야 했다. 하지만 ```@Transactional```을 사용하면 비즈니스 로직에 집중하고 트랜잭션 제어는 스프링에게 선언만으로 위임할 수 있으며, 이를 선언적 트랜잭션이라고 한다.

### 사용법
```
@Service
@RequiredArgsConstructor
@Transactional
public class OrderService {

    private final OrderRepository orderRepository;
    private final PaymentService paymentService;

    // 정상적으로 트랜잭션 프록시가 작동합니다.
    @Transactional
    public void createOrder(OrderRequest request) {
        Order order = orderRepository.save(new Order(request));
        
        // 결제 로직 중 RuntimeException이 발생하면 orderRepository.save 작업도 롤백됩니다.
        paymentService.process(order); 
    }
}
```
@Transactional은 클래스나 메서드에 적용할 수 있다. 위와 같이 선언만 하는 것으로 트랜잭션은 동작하게 된다. 주문 생성이 paymentRepository.process가 실패하게 되면 저장했던 주문도 같이 롤백이 된다.

### 동작 방식 - AOP 프록시 기반
@Transactional은 Spring AOP로 동작한다고 했다. 즉, 이 전에 배운 프록시를 기반으로 동작을 하게 된다.  
스프링은 ```@Transactional```이 붙은 클래스를 빈으로 등록할 때, 실제 객체 대신 프록시 객체를 생성하여 등록하게 된다.

!!! info "여기서 @Transactional은 JDK 동적 프록시 or CGLIB?"
	프록시 생성 방식을 결정하는 것은 ```@Transactional```이 아닌 구현된 실제 객체에 따라서 달라진다. 두 방식의 차이점을 잘 이해해야 한다. JDK 동적 프록시는 ```인터페이스```, CGLIB는 ```상속```이다. 그래서 실제 객체가 인터페이스 기반이면 JDK 동적 프록시를 기본 클래스이면 CGLIB 방식이다.
	단, 스프링 부트는 기본이 CGLIB 방식이다.

1. 메소드 호출 : 클라이언트가 메소드를 호출하면, 실제 빈이 아니라 Proxy가 가로챈다.
2. 트랜잭션 시작 : Proxy는 실제 메소드 실행 전에 데이터베이스 커넥션을 가져와 트랜잭션을 시작합니다.
3. 실제 로직 실행 : Proxy가 실제 객체의 메소드를 호출하여 비즈니스 로직을 실행한다.
4. 정상 종료 시 : 로직이 예외 없이 무사히 끝나면 Proxy가 트랜잭션을 커밋한다.
5. 예외 발생 시 : 로직 실행 중 예외가 발생하면 Proxy가 트랜잭션을 롤백한다. 커넥션을 닫고 반환한다.

#### 클래스 레벨과 메서드 레벨 우선순위
클래스 레벨과 메서드 레벨 둘 다 @Transactional이 있는 경우 좀 더 구체적인 메서드 레벨이 우선으로 적용된다.

### 실무에서 사용시 주의사항 (장애 유발 포인트)
1. 같은 클래스 내부 호출 (Self-invocation) :
	- 이 문제는 AOP의 문제로 AOP 동작 방식을 이해 해야 한다. 같은 클래스 내의 다른 메서드를 호출할 때는 프록시를 거치치 않고 실제 객체의 내부에서 직접 호출하게 된다. 프록시가 적용되는 지점은 외부에서 프록시 객체를 통해 들어오는 호출이다. 같은 객체 내부에서 ```this.method()``` 형태로 호출하면 프록시를 거치지 않고 대상 객체를 직접 호출하므로 advice가 적용되지 않는다. 따라서 호출당하는 메서드에 ```@Transactional```이 있어도 트랜잭션이 적용되지 않는다.

2. private/ 메서드, final 클래스
	- 프록시 기반 ```@Transactional```은 실무적으로 **public 메서드에만 적용하는 것을 원칙**으로 보는 것이 안전하다. non-public 메서드는 프록시 방식과 스프링 버전에 따라 기대와 다르게 동작할 수 있으며, private 메서드는 프록시 기반 AOP로 적용되지 않는다.
	- 또한, CGLIB는 상속 기반 프록시이므로 final class 프록시 생성이 어렵고, final method는 오버라이드 할 수 없어 트랜잭션 advice가 적용되지 않는다.

3. Rollback 기준
	- 스프링은 기본적으로 RuntimeException과 Error가 발생했을 때만 롤백한다.
	- IOException 이나 SQLException 같은 Checked Exception이 발생하면 기본적으로 커밋해버린다. 롤백이 필요하다면 @Transactional(rollbackFor = Exception.class) 처럼 명시해 주어야 한다.

4. ```readOnly = true``` 의미
	- readOnly = true는 “이 트랜잭션은 조회 중심”이라는 힌트를 주는 옵션이다. JPA/Hibernate에서는 flush 전략이나 dirty checking 최적화에 도움이 될 수 있고, 일부 환경에서는 DB 수준 최적화에 연결될 수 있다. 다만 자동 읽기 전용 DB 라우팅이나 쓰기 차단은 별도 인프라 설정이 있어야 한다.

### 문제 해결
#### 1. self invocation 문제 해결
클래스 분리하여 self Invocation을 해결할 수 있다. 내부 호출이 일어난다는 것은, 하나의 클래스가 너무 많은 역할을 가지고 있다는 신호일 수 있다. 문제가 되는 메서드를 아예 새로운 클래스(Service)로 분리하여 사용하면 문제를 해결할 수 있다.

```
// [개선 전] 문제가 발생하는 구조
@Service
public class OrderService {
    public void processOrder() {
        // 내부 호출 -> 트랜잭션 안 먹힘!
        pay(); 
    }

    @Transactional
    public void pay() { ... }
}

// ==========================================================

// [개선 후] 클래스를 분리하여 프록시를 타게 만듦
@Service
@RequiredArgsConstructor
public class OrderService {
    // 분리한 클래스를 주입받음
    private final PaymentService paymentService; 

    public void processOrder() {
        // 외부 클래스의 메서드 호출이 되므로 정상적으로 프록시를 타고 트랜잭션 적용!
        paymentService.pay(); 
    }
}

@Service
public class PaymentService {
    @Transactional
    public void pay() { ... }
}
```
클래스 개수가 늘어날 수 있지만, 단일 책임 원칙을 지킬 수 있고, 코드가 깔끔해진다.

아니면, 자기 자신을 필드로 참조해 호출하는 방식이 있다.
```
@Service
public class OrderService {

    // 자기 자신(프록시 객체)을 주입받되, @Lazy로 초기화를 지연시킴
    private final OrderService self;

    public OrderService(@Lazy OrderService self) {
        this.self = self;
    }

    public void processOrder() {
        // this.pay()가 아니라 프록시 객체인 self.pay()를 호출!
        self.pay(); 
    }

    @Transactional
    public void pay() { ... }
}
```
클래스를 당장 분리하기엔 로직이 너무 강하게 결합되어 있거나 시간이 촉박한 경우 사용하는 우회 방법이다. 스프링 컨테이너에서 자기 자신의 프록시 객체를 주입받아 호출한다.

하지만, 생성자 주입을 사용하면 순환 참조 오류가 발생하므로, 반드시 ```@Lazy```를 사용해 지연 주입을 받거나 필드 주입을 사용해야 한다.

#### 2. 예외적으로 private에서 트랜잭션을 적용하고 싶을 때
보통 다음과 같은 고민이 생길 때 ```private``` 트랜잭션을 떠올린다.

1. 비즈니스 로직의 일부가 데이터 정합성이 매우 중요하지만, 이 메서드가 서비스 외부로 노출되는 것을 막고 싶을 때
2. 거대한 메서드의 부분 트랜잭션을 원할 때

하지만, private을 억지로 하는 것보다는 상황에 따라서 다른 방식을 사용하는 것이 좋다.  
다음과 같은 방식으로 문제를 해결할 수 있다.
1. Public 위임
2. 소규모 서비스 분리
3. TransactionTemplate 사용

> **다시 생각해보기**  
> 만약 독립적인 트랜잭션이 필요한 만큼 중요하다면, 그러면 별도의 책임으로 분리가 필요한 리팩토링 상황일 수 있다.

**1. Public 위임 (Facade 패턴)**
private 로직을 감싸는 public 메서드를 만들고, 그 public 메서드에 @Transactional을 부여
```
@Service
public class UserService {

    // 외부에서는 이 메서드만 호출 가능
    @Transactional
    public void registerUser(UserDto dto) {
        // 내부의 private 로직들을 하나의 트랜잭션으로 묶어서 실행
        saveUser(dto);
        sendWelcomeEmail(dto);
    }

    private void saveUser(UserDto dto) { ... }
    private void sendWelcomeEmail(UserDto dto) { ... }
}
```
하지만, 내부의 특정 private 로직만 독립적으로 트랜잭션을 걸 수 없다.

**2. 소규모 서비스 분리 (도메인 서비스)**
만약 특정 private 로직이 자신만의 트랜잭션 전파 속성을 가져야 한다면, 그 로직을 별도의 도메인 서비스나 컴포넌트로 분리한다.

```
@Service
@RequiredArgsConstructor
public class OrderService {
    private final InventoryHandler inventoryHandler; // 분리된 컴포넌트

    @Transactional
    public void placeOrder(Order order) {
        // 재고 차감은 별도의 트랜잭션으로 처리하고 싶다면?
        inventoryHandler.reduceStock(order.getItemId()); 
    }
}

@Component
public class InventoryHandler {
    // 내부 로직이었던 것을 public으로 바꾸고 클래스를 분리
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void reduceStock(Long itemId) { ... }
}
```
**Propagation 속성**  
- REQUIRED : 기존 트랜잭션 참여, 없으면 새로 시작  
- REQUIRES_NEW : 항상 새로운 트랜잭션 시작



**3. TransactionTemplate 사용**
private 메서드 내부에서 직접 트랜잭션 코드를 작성한다.
```
@Service
@RequiredArgsConstructor
public class LogService {
    private final TransactionTemplate transactionTemplate;

    public void externalMethod() {
        // 비트랜잭션 로직들...
        doInternalLog();
    }

    private void doInternalLog() {
        // private 메서드 안에서도 트랜잭션을 수동으로 제어 가능
        transactionTemplate.execute(status -> {
            // 이 안의 코드는 트랜잭션 내에서 실행됨
            logRepository.save(new Log("..."));
            return null;
        });
    }
}
```
프록시 제약에서 완전히 자유로울 수 있지만, 비즈니스 로직에 스프링 종속적인 코드가 섞여 가독성이 떨어진다.

```private```에 트랜잭션을 거는 것 보단, 트랜잭션이 필요한 단위로 클래스를 분리하는 것이 훨씬 유지보수하기 좋은 코드를 만든다.


## 궁금한 것들
### 그러면 조회 메소드에는 모두 readOnly를 적용해야 하나?
조회만 하는 메서드에는 ```readonly=true```를 붙이는 것을 기본 원칙으로 삼는 것이 좋다. 무조건은 아니지만, 실무적으로 얻는 이점이 훨씬 크다.

#### JPA 성능 최적화
JPA는 트랜잭션 내에서 데이터를 조회할 때, 나중에 데이터가 수정될 것을 대비해 원본 상태를 복사해 두는 스냅샷을 영속성 컨텍스트를 저장한다. ```readonly=true```를 설정하면 JPA는 조회만하는 메소드를 라는 것을 인식하고 스냅샷을 만들지 않게 된다. 결과적으로 불필요한 메모리 사용량을 줄이고 CPU 연산 비용을 아낄 수 있다.

#### DB 부하 분산
트래픽이 많은 실무 환경에서는 데이터베이스를 쓰기 전용과 읽기 전용 복제본으로 분리하는 경우가 많다. 이 때 ```readOnly = true```를 설정되어 있으면, 스프링과 DB 커넥션 풀이 이를 인식하여 쿼리를 쓰기가 아닌 읽기 전용 DB로 자동 라우팅하여 부하를 분산시킨다.

#### 명시적 이유
트래픽이 많은 실무 환경에서는 데이터베이스를 쓰기 DB와 읽기 전용 복제본으로 분리해 운영하는 경우가 많다. 이때 readOnly = true는 해당 트랜잭션이 조회 목적임을 나타내는 힌트로 활용될 수 있으며, 별도의 라우팅 설정이 되어 있다면 읽기 전용 DB로 분산 처리하는 데 도움을 줄 수 있다.

#### 예외적인 상황
트랜잭션을 시작하고 종료하는 과정 자체에서 미세한 리소스를 소모한다. 따라서 로직이 복잡하지 않은 단순한 1줄짜리 단건 쿼리의 경우, 아예 ```@Transactinal``` 자체를 붙이지 않고 조회하는 것이 속도 면에서는 미세하게 빠를 수 있다.

하지만 미세한 성능 차이보다는 프로젝트 전체의 일관성과 연관된 객체를 가져오는 지연 로딩의 안정성을 위해 ```readOnly```를 사용한다.

!!! info "정리"
	@Transactional은 단순히 트랜잭션을 켜는 마커가 아니라, 트랜잭션의 전파 방식, 격리 수준, 읽기 전용 여부, 롤백 규칙 등을 선언하는 메타데이터다. 스프링은 이 정보를 바탕으로 프록시를 통해 메서드 호출 전후에 트랜잭션을 시작하고 commit 또는 rollback을 수행한다. 다만 이 방식은 프록시 기반이므로, 같은 클래스 내부 호출이나 private 메서드처럼 프록시를 거치지 않는 경우에는 기대한 대로 동작하지 않을 수 있다. 

---

## 출처
- 스프링 공식 문서
- ChatGPT, 제미나이[TOC]