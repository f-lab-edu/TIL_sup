# 영속성 컨텍스트

Spring Data JPA를 잘 쓰려면, Repository 문법보다 영속성 컨텍스트를 먼저 이해해야 한다. Spring Data JPA의 save(), 조회, 변경 감지, 지연 로딩, flush, merge 같은 동작이 전부 JPA의 EntityManager와 영속성 컨텍스트 위에서 돌아가기 때문이다. 이전에 설명했던 것과 같이 Spring Data JPA는 결국 DB관련해서 더 편하게 사용하려고 한 추상화일 뿐이지, 영속성 컨텍스트 자체를 대체하는 기술은 아니다.

자주 헷갈리는 지점이 EntityManager와 영속성 컨텍스트를 동일시 하는 경우가 있는데, 둘은 다르다. EntityMaager는 영속성 컨텍스트를 관리하는 도구이고, 영속성 컨텍스트는 실제 엔티티가 있는 살아있는 공간이다. 이 차이를 바로 잡고 가야 뒤가 안 헷갈린다.

![](https://suplab-bucket.s3.ap-northeast-2.amazonaws.com/images/suplab-bucket0ce45a24-097d-4fbe-be3c-b135449ab272-image.png)

## Spring Data Jpa, Hibernate, 영속성 컨텍스트의 관계
JPA 명세에서 EntityManager는 영속성 컨텍스트와 직접 연결되는 API이고, 영속성 컨텍스트는 엔티티 타입 + 식별자(PK)가 같다면 항상 하나의 엔티티 인스턴스만 관리되는 영역으로 정의된다. Hibernate 문서도 JPA의 EntityManager와 Hibernate의 Session이 모두 persistence context를 다루는 API라고 설명한다.

- Spring Data JPA : Repostiory 인터페이스를 편하게 작성하게 해주는 스프링 기술
- JPA : 자바 ORM 표준
- Hibernate : JPA를 구현한 구현체
- EntityManager : 영속성 컨텍스트에 접근하고 조작하는 핵심 객체

### JPA란
일단 JPA는 Java Persistenct API라 하는데, 중요한 포인트는 JPA는 이렇게 영속성 기술을 만들어라고 정의한 규약이지, 실행 엔진 자체가 아니다.

> 즉, JPA 자체가 SQL을 날리는 프로그램이 아니다. 인터페이스와 규칙, 동작 방식의 표준을 정의한다.  
예를 들어: **설계도와 가깝다.**
- Entity는 어떻게 표시하는가
- EntityManager는 어떤 역할을 하는가
- 영속성 컨텍스트는 어떤 개념인가
- flush, detacch, merge 같은 상태 전이는 어떻게 되는가
- JPQL은 어떻게 동작하는가

### Hibernate란
Hibernate는 JPA명세를 실제로 구현한 ORM 프레임워크이다.

JPA가 이렇게 동작해야 한다라고 정의했다면, Hibernate는 그 규약을 토대로 실제 코드로 구현해서 SQL 생성, DB 반영, 캐시 관리, dirty checking 등을 수행한다.

코드에서는 ```EntityManager```를 쓰지만, 실제 내부에서는 Hibernate의 Session, Persistence Context, SQL Generator 등이 동작한다. 이외에 EclipseLink, OpenJPA라는 다른 구현체가 있지만 Hibernate가 실무에서 표준처럼 많이 쓰인다.

### Spring Data JPA란
Spring Data는 더 상위에서 데이터 접근 코드를 쉽게 만들기 위한 프로젝트이다.
이전에도 Spring Data JPA에 대해 알아봤지만, 이는 JPA가 아닌 JPA 그리고 DB를 더 편리하게 사용하기 위한 스프링의 프레임워크이다.

그래서 Spring Data JPA를 사용하면,
- Repository 인터페이스 정의로 구현체 자동생성
- 메서드 이름 기반 쿼리 자동 생성
- 페이징/정렬/기본 CRUD 제공

DB 관련된 편리한 기능을 사용할 수 있다.

이제 다음 영속성 컨텍스트에 대해 알아볼텐데 각각의 역할과 경계를 잘 이해하면 영속성 컨텍스트를 더 잘 이해할 수 있다.

## 영속성 컨택스트란?
> 영속성 컨텍스트(Persistence Context)는 엔티티를 영구 저장하는 환경이다. 쉽게 말하면, DB와 애플리케이션 사이의 중간 저장소이다.

JPA 명세에서 정의하는 영속성 컨텍스트는 엔티티 인스턴스들의 집합으로서, 주어진 영속성 컨텍스트 내에서 엔티티 인스턴스와 그 라이프사이클을 관리하는 것이라고 나와있다. 실무적으로 표현한다면, 트랜잭션이 시작되면 생성되고, 트랜잭션이 끝나면 사라지는 DB 앞의 1차 캐시 공간이다.

**더 정확히 말하면** :
DB 테이블의 row를 그대로 다루는 것이 아니라 자바 객체인 **Entity**를 애플리케이션 실행 중 특정 범위 안에서 **"관리 상태(managed state)"** 로 보관하고 추적하는 컨텍스트이다.

### 영속성 컨텍스트가 필요한 이유
DB와 직접 매번 통신하면 다음과 같은 문제가 생긴다.
- 같은 데이터를 한 트랜잭션 안에서 여러 번 조회할 때 비효율적이다.
- 객체 변경 여부를 개발자가 일일이 SQL로 반영해야 한다.
- 객체 동일성 보장이 어렵다.
- 연관 객체 탐색과 쓰기 작업이 복잡하다.

영속성 컨텍스트는 이 문제를 해결하려고 존재한다.  
즉, 객체를 단순하기 읽는 것을 넘어서 관리하기 위해 필요한 기술이다.

간단하게 예를 들어서 DB를 사용하려면 다음과 같은 것들을 직접 코드로 작성해야 했다.
- 커넥션 획득
- SQL 작성
- PreparedStatement 생성
- 파라미터 바인딩
- ResultSet 조회
- 결과를 객체로 변환
- 자원 반납

이 과정은 반복이 많고, 생산성이 떨어지고, 객체 지향적으로 개발하기 어렵다.

자바 개발자는 보통 객체를 다루고 싶은데, DB의 데이터 구조는 테이블 형태이다. 여기서 생기는 문제가 객체-관계 불일치이다.

- 객체의 상속
- 객체간의 참조를 통한 연관관계
- 객채의 그래프 탐색
- DB 테이블, FK, 조인 중심

이 간극을 줄이려고 나온 것이 ORM 기술이다.

### 특징
#### 1. 1차 캐시의 역할
한 번 조회한 데이터는 영속성 컨텍스트에서 관리하게 되는데, 이후에 다시 또 조회하면 DB에서 조회를 하지 않고 컨텍스트에서 관리하는 저장된 Entity를 사용하면 되기 때문에 1차 캐시의 역할을 가진다.

```
Map<@Id, Entity> 형태로 엔티티를 저장
```
영속성 컨텍스트는 위와 같은 형태로 엔티티를 저장하게 되고, 저장된 동일한 엔티티가 있다면 그것을 사용한다.

#### 2. 변경 감지(Dirty Checking)
관리 상태의 Entity는 값이 바뀌면, 트랜잭션 flush 시점에 변경 내용을 감지해 UPDATE SQL을 반영할 수 있다. Hibernate는 영속 상태 점검과 flush 과정을 통해 이를 수행한다.

#### 3. 쓰기 지연이 가능하다.
엔티티를 persist 했다고 즉시 DB에 INSERT가 날아가는 게 아니라, 트랜잭션/flush 시점에 SQL이 반영될 수 있다. JPA는 flush를 통해 영속성 컨텍스트와를 DB와 동기화한다. 이를 통해 불필요한 DB I/O를 최소화하며, 배치 처리가 가능하다.

#### 4. 엔티티 동일성을 보장한다.
같은 영속성 컨텍스트에서 같은 엔티티 식별자를 조회하면, 논리적으로 같은 데이터가 아니라 실제로 같은 객체 참조를 기대할 수 있다. 이는 객체 그래프를 안정적으로 다루게 해준다.

즉, 같은 PK로 조회를 했다면, 항상 객체는 == 비교도 true를 보장한다. (a == b가 True)

## 영속성 컨텍스트 핵심

### 1. 엔티티 생명주기
영속성 컨텍스트를 이해하려면 엔티티 상태를 알아야 한다.

![claude "Entity 상태 관계 그래프"](https://suplab-bucket.s3.ap-northeast-2.amazonaws.com/images/suplab-bucket4d42f4d4-a95b-4e8e-bc18-233f16f71994-image.png)

#### 1) 비영속 (Transient)
아직 영속성 컨텍스트와 관계없는 순수한 새 객체 상태를 말한다. new 키워드로 객체를 생성했을 때의 초기 상태이다. DB에도 없고, 영속성 컨텍스트에도 없다. 이 상태에서 변경해봤자 DB에 아무 영향이 가지 않는다.


```
Member member = new Member();
member.setName("kim");
```
단순히 new를 통해 객체를 생성만 했다면 그냥 자바 객체일 뿐이다.

#### 2) 영속 (Managed)
영속성 컨텍스트 관리하는 상태이다. persist(), findById(), save() 후의 상태이다. 이 상태의 엔티티를 수정하면 트랜잭션 커밋 시 자동으로 UPDATE가 발생한다. 1차 캐시에 존재하며 스냅샷이 찍혀있다.

```
entityManager.persist(member);
```

#### 3) 준영속 (Detached)
영속성 컨텍스트에서 분리된 상태이다. 더 이상 변경 감지가 되지 않는다. detach(), clear(), close() 후 또는 트랜잭션이 끝난 후 반환된 엔티티가 여기에 해당된다. LazyLoading도 불가하다. merge()로 다시 영속 상태로 만들 수 있다.

!!! info "준영속과 merge"
Spring Data JPA의 save()는 새 엔티티면 persist(), 기존 엔티티면 merge()를 호출한다. 즉, save()는 단순한 "저장 버튼"이 아니라 상황에 따라 동작이 달라진다. 그래서 화면에서 받는 객체나, 트랜잭션 밖에서 들고 있던 객체를 그대로 save()하면 내부적으로 merge가 개입할 수 있다.

#### 4) 삭제 (Removed)
삭제 예약 상태이다. flush/commit 시 DELETE가 반영된다. remove()가 호출된 상태이며, 트랜잭션이 롤백 되면 삭제도 취소된다.

#### 상태 전환 요약
- ```new Member()``` : 비영속이며 DB와 무관한 상태
- persist() / save() : 영속이며 1차 캐시 등록, 변경 감지 시작
- detach()/ clear() / 트랜잭션 종료 : 준영속이며 컨텍스트에서 빠져나온 상태, LazyLoading 불가
- merge() : 준영속 -> 영속으로 복귀
- remove() : 삭제 - flush + commit시 실제 DELETE 실행

!!! danger "주의할 점"
실무에서 가장 많이 실수하는 지점은 ```트랜잭션 종료 -> 준영속``` 전환이다. 서비스 메서드가 끝나는 순간 모든 엔티티가 준영속이 되기 때문에, Controller에서 Lazy 필드에 접근하면 ```LazyInitializationException```이 터진다.

```
@Transactional
public void entityLifecycleExample() {
    // 1. 비영속 상태 — new로 생성, DB와 무관
    Member member = new Member();
    member.setName("홍길동");

    // 2. 영속 상태 — 영속성 컨텍스트가 관리 시작
    em.persist(member); // 이 시점엔 INSERT SQL 아직 안 나감!

    // 3. 변경 감지 — setter만 호출해도 됨
    member.setName("이순신"); // UPDATE 자동 예약됨

    // 4. 준영속 상태 — 컨텍스트에서 분리
    em.detach(member); // 이후 변경은 DB에 반영 안 됨

    // 5. 다시 영속 상태로 — merge로 재진입
    Member merged = em.merge(member);

    // 6. 삭제 상태
    em.remove(merged); // flush 시 DELETE SQL 실행
} // 트랜잭션 커밋 → flush 자동 발생
```

### 2. 영속성 컨텍스트는 보통 트랜잭션 범위와 함께 움직인다.
JPA에는 transaction-scoped persistence context와  extended persistence context 개념이 있다. 일반적인 Spring에서는 보통 트랜잭션 범위의 영속성 컨텍스트를 사용한다.

- 트랜잭션 시작
- 영속성 컨텍스트 사용
- 트랜잭션 종료
- 영속성 컨텍스트 종료 또는 범위 종료

그래서 @Transactional 안에서 조회한 엔티티와, 그 밖에서 다루는 엔티티의 행동이 달라진다.

### 3. DB와 즉시 동기화 되지 않는다.
영속성 컨텍스트는 메모리에서 엔티티 상태를 관리하고, DB 반영은 flush 시점에 일어난다. flush는 보통 트랜잭션 커밋 전, 혹은 쿼리 실행 전 등 특정 시점에 발생한다.

즉, 객체를 바꿨다고 바로 UPDATE가 되는 것이 아니라

- 객체를 바꿨다.
- 영속성 컨텍스트가 변경을 기억한다.
- flush 시점에 UPDATE SQL을 만든다.
- coommit 시 최종 반영된다.

이 흐름으로 동작한다.

## 영속성 컨텍스트의 아키텍쳐
> Spring Data JPA의 전체 레이어에서 영속성 컨텍스트가 어떤 위치에 있는지 이해하면, 언제 어디서 문제가 발생했는지 바로 파악할 수 있다.

![](https://suplab-bucket.s3.ap-northeast-2.amazonaws.com/images/suplab-bucketb93f7d34-d73c-4b1f-9cdb-37297cb544cd-image.png)

!!! info "핵심 포인트"
@Transactional이 붙은 메서드가 시작되면 영속성 컨텍스트가 생성되고, 메서드가 끝나면 flush + commit 후 영속성 컨텍스트가 소멸된다.

```
[Spring Data Repository]
        ↓
[EntityManager]
        ↓
[Persistence Context]
   ├─ 1차 캐시
   ├─ 엔티티 식별자 관리
   ├─ 스냅샷/변경 감지
   ├─ 쓰기 지연 SQL 저장
   └─ 엔티티 상태 관리
        ↓ flush
[SQL 생성 / JDBC]
        ↓
[Database]
```
Spring Data JPA의 Repository는 직접 DB를 치는 것이 아니라, 내부적으로 JPA EntityManager를 통해 동작한다. Spring Data 공식 문서도 save()가 내부적으로 persist() 또는 merge()를 호출한다고 설명한다. EntityManager는 다시 영속성 컨텍스트와 연결되어 있다.

## 동작 과정
### 1. 조회 흐름
1. emfind(Member.class, 1L) 호출  
   애플리케이션이 엔티티 조회를 요청한다.
2. 1차 캐시 확인 (Map<1L, member>)  
   영속성 컨텍스트의 Map에서 해당 ID가 있는지 먼저 확인한다. 있으면 DB 조회 없이 바로 반환
3. 없으면 DB SELECT 실행  
   1차 캐시에 없을 때만 실제 DB에 SELECT 쿼리를 날립니다
4. 결과를 1차 캐시에 저장 + 스냅샷 생성  
   조회된 엔티티를 1차 캐시에 저장하고, 변경 감지를 위한 스냅샷(원본 상태)을 찍어 둡니다
5. 엔티티 반환  
   애플리케이션에 영속 상태의 엔티티를 반환합니다

### 2. 저장 / 변경 흐름 (flush 과정)
1. 엔티티 변경 (member.setName(...))  
   영속 상태 엔티티의 필드를 수정합니다
2. flush 발생 시 스냅샷과 비교 (dirty checking)  
   트랜잭션 커밋 직전, JPA가 스냅샷과 현재 엔티티를 필드 단위로 비교합니다
3. 변경된 필드가 있으면 UPDATE SQL 생성  
   쓰기 지연 SQL 저장소에 UPDATE 쿼리를 등록합니다
4. 쓰기 지연 저장소의 SQL 일괄 실행  
   INSERT, UPDATE, DELETE가 모두 DB로 전송됩니다 (아직 커밋 전)
5. 트랜잭션 커밋  
   DB 트랜잭션이 커밋되어 변경이 영구 반영됩니다

!!! info "flush != commit"
flush는 영속성 컨텍스트의 변경 내용을 DB에 동기화하는 것이고, commit은 트랜잭션을 확정하는 것이다. flush후에도 롤백이 가능하다.

#### flush 발생 시점 3가지
- 자동 flush : 트랜잭션 커밋 직전에 자동으로 발생한다. 가장 일반적인 경우이다.
- JPQL 실행 전 : JPQL/HQL 쿼리 실행 직전에 자동 flush 된다. 데이터 정합성 보장을 위해서이다.
- 직접 flush : em.flush(), repository.flush()를 명시적으로 호출한 경우이다.

## 문제 상황

### N+1 문제
![](https://suplab-bucket.s3.ap-northeast-2.amazonaws.com/images/suplab-bucket2cb57ea7-de0a-4299-9928-31aa0930b9ff-image.png)

클로드가 아주 잘 그려줬다.

Orders는 3건을 한 번에 가져왔지만, 각 ```order.getMember()``` 호출 순간마다 LAZY 로딩이 개별 SELECT를 날린다. 주문이 100건이면 쿼리가 101번, 이걸 고치는 방법은 3가지가 있다.

#### 방법 1. Fetch Join
JPQL에 JOIN FETCCH를 명시해서 연관 엔티티를 한 번의 JOIN 쿼리로 가져오는 방법이다. 가장 직관적이고 강력하다.

```
@Query("SELECT o FROM Order o JOIN FETCH o.member")
List<Order> findAllWithMember();

// 여러 연관관계 동시에
@Query("SELECT o FROM Order o " +
       "JOIN FETCH o.member " +
       "JOIN FETCH o.items")
List<Order> findAllWithMemberAndItems();
```
이렇게 하면 쿼리는 단 1번 실행된다. 그릐고 SQL 직접 제어 가능하며, ToOne 한정으로 페이징과 함께 사용 가능하다.

하지만, 컬렉션(ToMany) fetch join은 페이징이 불가하고,  
둘 이상 컬렉션 fetch join이 불가하다.

> **컬렉션 + 페이징 조합 금지**: @OneToMany fetch join에 페이징을 붙이면 JPA가 메모리에서 페이징 처리하면서 HHH90003004 경고를 낸다. 이때는 @BatchSize를 사용한다.

#### 방법 2. @EntityGraph
```
// 방식 1: 메서드에 직접 적용
@EntityGraph(attributePaths = {"member"})
@Query("SELECT o FROM Order o")
List<Order> findAllWithMember();

// 방식 2: 기본 메서드 오버라이드
@EntityGraph(attributePaths = {"member", "items"})
@Override
List<Order> findAll();

// 방식 3: Named EntityGraph (엔티티에 정의)
@NamedEntityGraph(
    name = "Order.withMember",
    attributeNodes = @NamedAttributeNode("member")
)
@Entity
public class Order { ... }

@EntityGraph("Order.withMember")
List<Order> findByStatus(OrderStatus status);
```
@EntityGraph는 JPQL 없이 선언적으로 사용할 수 있으며,  
기존 Repository 메서드에 추가만 하면 된다.

하지만, 항상 LEFT OUTER JOIN (불필요한 null)로 조회하며  
복잡한 조건에서 Fetch Join보다 제어력이 낮다.  
또한, 컬렉션 페이징 동일하게 사용 불가하다.

> **Fetch Join vs EntityGraph 선택 기준**: 단순 연관관계 조회는 @EntityGraph가 편하다. 복잡한 WHERE 조건이나 다중 JOIN이 필요하면 @Query + fetch join이 낫다.

#### 방법 3. @BatchSize
```
# application.yml — 전체에 적용
spring:
  jpa:
    properties:
      hibernate:
        default_batch_fetch_size: 100  # 100개씩 IN 절로 묶음
```
**또는**

```
@Entity
public class Order {
    @BatchSize(size = 100)
    @OneToMany(mappedBy = "order")
    private List<OrderItem> items;

    @BatchSize(size = 100)
    @ManyToOne(fetch = FetchType.LAZY)
    private Member member;
}
```
**쿼리 실행**
```
SELECT * FROM member WHERE id IN (1, 2, 3, ... 100)
```
쿼리가 2번 실행 되지만, 컬렉션 페이징과 함께 사용 가능하다.  
코드 변경을 최소화하며, 기존 LAZY 로딩 구조는 유지할 수 있다.

다만, size 설정이 크면 IN 절 성능이 저하될 수 있다.

> **실무 패턴**: default_batch_fetch_size: 100을 기본으로 깔아두고, 성능이 중요한 API는 fetch join으로 추가 최적화한다.

#### 방법 4. 엔티티 자체 가져오지 않기
엔티티를 사용하지 말고 프로젝션을 사용하는 방법이다.
```
// 4. 순수 DTO 조회 — N+1 자체가 없음 (가장 빠름)
@Query("SELECT new com.example.OrderDto(o.id, m.name) " +
       "FROM Order o JOIN o.member m")
List<OrderDto> findAllDto();
```
프로젝션을 사용하면 필요한 컬럼만 가져와 사용할 수 있고 N+1 자체가 없다.  
DTO 활용은 영속성 컨텍스트를 전혀 거치치 않기 때문에 1차 캐시, 스냅샷, 변경 감지 등의 추가 비용이 들지 않는다.

![](https://suplab-bucket.s3.ap-northeast-2.amazonaws.com/images/suplab-bucket2efdb39a-d883-45d8-91cd-9e1f6b1b0f52-image.png)
> 클로드 코드 응답 결과 화면 캡쳐

!!! info "체감 성능 차이"
건당 차이는 작지만 100건, 1000건 리스트 API에서는 체감이 난다. 특히 컬럼이 많은 테이블(이미지 URL, 긴 텍스트 등)에서는 SELECT * 대비 2~5배 빠른 경우도 있다.

## 정리

영속성 컨텍스트는 JPA가 엔티티를 관리하는 실행 중 메모리 공간이며, 1차 캐시, 동일성 보장, 변경 감지, 쓰기 지연을 통해 객체 중심의 데이터 처리를 가능하게 하는 핵심 메커니즘이다.

---

## 출처
- AI : ChatGPT, 제미나이, Claude
- https://docs.hibernate.org/orm/5.1/userguide/html_single/chapters/pc/PersistenceContext.html?utm_source=chatgpt.com
- https://dev-jwblog.tistory.com/125?utm_source=chatgpt.com