# Spring Data

## Spring data란
Spring Data는 단순 CRUD 자동화 도구가 아니라, 어떤 철학과 구조로 데이터 접근 계층을 추상화하는지를 이해해야 한다. Spring 공식 문서에도 Spring Data의 목표를 반복적인 데이터 접근 보일러플레이트를 크게 줄이는 것이라고 설명한다.

Spring Data란 데이터 접근 계층을 더 적은 코드로, 더 일관되게 만들기 위한 추상화 프레임워크이다. 특히 Repository 추상화를 중심으로 CRUD, 페이징, 정렬, 쿼리 메서드, 프로젝션, 검사 같은 기능을 공통된 방식으로 제공한다.

## Spring Data를 사용하는 이유
### 1. 중복되는 코드가 많다.
전통적인 방식에서는 DAO/Repository 구현 클래스를 직접 만들고, EntityManager나 JDBC 코드로 조회/저장/삭제를 계속 작성해야 했다.

예를 들어
```
@Repository
public class UserRepository {

    @PersistenceContext
    private EntityManager em;

    public User findById(Long id) {
        return em.find(User.class, id);
    }

    public List<User> findByEmail(String email) {
        return em.createQuery("select u from User u where u.email = :email", User.class)
                 .setParameter("email", email)
                 .getResultList();
    }

    public void save(User user) {
        em.persist(user);
    }
}
```
위에 긴 코드가 다음과 같이 짧게 작성할 수 있게 된다.

```
public interface UserRepository extends JpaRepository<User, Long> {
    List<User> findByEmail(String email);
}
```

Spring Data의 공식 목표 자체가 데이터 접근 레이어의 보일러플레이트를 크게 줄이는 것이다.

### 2. 저장소 기술이 달라도 접근 방식이 비슷해진다.
Spring Data는 JPA만 있는게 아니다. MongoDB, Redis, JDBC, R2DBC 등 여러 저장소에 대해 비슷한 Repository 추상화를 제공한다. 공식 문서에도 Spring Data를 여러 Persistence store에 대한 공통 repository/object-mapping 추상화로 소개한다.

즉, 개발자는 도메인 중심으로 저장소 인터페이스를 정의할 수 있게 된다.

### 3. 선언형 개발이 가능하다.
Spring Data JPA는 쿼리 생성 방식으로 메서드 이름을 기반 쿼리 파생(quert derivation)과 직접 쿼리 정의(@Query)를 지원한다.

### 4. 공통 기능이 잘 갖춰져 있다.
- CRUD
- 페이징
- 정렬
- 프로젝션
- 감사
- Query by Example
- Specification
- 커스텀 Repository 확장

## Spring Data의 핵심

### 1. Repository 추상화
가장 중심 인터페이스는 Repository이다. 이 인터페이스는 도메인 타입과 ID 타입을 받는 마커 인터페이스 역할을 하며, 그 위에 CrudRepository, ListCrudRepository 같은 기능성 인터페이스가 쌓인다.

구조는 다음과 같다고 보면 된다.
```
Repository<T, ID>   // 가장 기본
  └─ CrudRepository<T, ID>
      └─ PagingAndSortingRepository<T, ID>
          └─ JpaRepository<T, ID>   // JPA 사용 시 흔히 사용
```
보통 JpaRepository<Entity, ID>를 상속해서 쓴다.

### 2. 인터페이스만 선언하면 구현체를 런타임에 만들어준다.
Spring Data가 애플리케이션 시작 시점에 Repository 인터페이스를 스캔하고, 그에 대한 프록시 객체를 만들어 빈으로 등록한다. 공식 문서의 repository configuration/definition 설명도 repository interface를 기반으로 인스턴스를 생성하는 구조를 전제로 한다.

즉, 우리가 쓰는 UserRepository는 보통 실제 구현 클래스가 아니라 Spring Data가 만든 프록시 빈이다.

### 3. 쿼리 생성을 전략적으로 한다.
Spring Data JPA는 쿼리를 만드는 방식을 여러 가지로 지원한다.
- 메서드 이름으로 유도
- @Query로 직접 JPQL/네이티브 SQL 작성
- Named Query 사용

공식 문서는 이를 Query Lookup Strategies라고 설명한다. 즉, 이 메서드 호출에 대응하는 쿼리를 어떤 방식으로 찾을 것인가를 선택하는 것이다. 보통 다음 2가지로 처리한다.

1. 선언된 쿼리 사용
	- @Query로 직접 작성한 쿼리
	- JPA named query 같은 미리 선언된 쿼리
2. 메서드 이름으로 쿼리 생성
  - finyByUser(...)
  - 메서드 이름을 파싱해서 쿼리를 자동 생성

Query Lookup Strategies 옵션에는 CREATE, USE_DECLARED_QUERY, CREATE_IF_NOT_FOUND 3가지가 있고, 기본은 CREATE IF_NOT_FOUND이다.
- CREATE_IF_NOT_FOUND : 먼저 선언된 쿼리를 찾고, 없으면 메서드 이름 기반으로 쿼리를 생성한다.

## Spring Data의 동작 과정
### 전체 흐름
다음과 같은 코드가 있다고 한다면,
```
public interface UserRepository extends JpaRepository<User, Long> {
    List<User> findByEmail(String email);
}

userRepository.findByEmail("test@test.com");
```

#### 1. 애플리케이션 시작시 Repository 인터페이스 스캔
Spring Boot가 Spring Data JPA 설정을 로드하면 Repository 인터페이스를 찾아낸다. 이 인터페이스는 Repository 계열을 상속하고 있으므로 Spring Data가 관리 대상임을 안다.

#### 2. Repository Factory가 프록시 객체를 만든다.
실제 구현 클래스가 없으니, Spring Data는 내부 팩토리를 통해 Repository 프록시를 생성한다.
이 프록시는 호출을 가로채고 어떤 저장소 연산을 수행할지 판단한다.

#### 3.메서드가 기본 CRUD인지, 쿼리 메서드인지 구분한다.

예를 들어서
- save, findById, deleteById : 이미 JpaRepository에 정의된 공통 메소드
- findByEmail : 사용자가 선언한 쿼리 메서드

Spring Data는 메서드 메타데이터를 보고 이 메서드가 어떤 종류인지 해석한다.

#### 4. 쿼리 생성 전략을 적용한다.
findByEmail을 분석한다고 했을때,
- findBy : 조회
- Email : 조건 필드

메서드 이름에서 파싱한 결과를 기준으로 "email=?" 형태의 조건 쿼리로 해석한다.

#### 5. 실제 Jpa 구현체에게 위임한다.
Spring Data JPA는 JPA 위에서 동작한다.
실제 DB 접근은 Hibernate 같은 Jpa Provider가 처리한다. 즉, Spring Data JPA는 DB를 직접 다루는 것이 아닌, JPA 위에 있는 추상화이다.

```
Service
  -> UserRepository(프록시)
      -> Spring Data Repository Infrastructure
          -> JPA(EntityManager)
              -> Hibernate
                  -> SQL 실행
                      -> DB
```
#### 6. 결과를 도메인 객체나 Projection으로 반환한다.
조회 결과는 Entity일 수도 있고, 인터페이스 기반 프로젝션이나 DTO 기반 프로젝션일 수도 있다.

> Projection : 필요한 컬럼만 조회하는 기술

## Spring Data에서 중요한 내용
### 1. Projection 적극적으로 사용
엔티티 전체가 필요 없는데도 전체 엔티티를 조회하면 성느과 설계 모두 분리해질 수 있다.

예를 들어 사용자 목록 화면에 이름/이메일만 필요하면 다음과 같이 사용할 수 있다.
```
public interface UserSummary {
    String getName();
    String getEmail();
}

List<UserSummary> findByStatus(Status status);
```
Spring Data JPA는 인터페이스 기반과 클래스 기반 프로젝션을 지원한다. 다만 문서에는 중첩 속성은 조인 전체가 물질화될 수 있어 주의가 필요하다고 한다.

중첩 필드는 다음과 같은 상황
```
@Entity
class Order {
    @ManyToOne
    Member member;
}

@Entity
class Member {
    String name;
}
```

그래서 projection을 사용한다면,
```
interface OrderView {
    String getMemberName(); // ❌ 안됨
}
```
```
interface OrderView {
    MemberView getMember();

    interface MemberView {
        String getName();
    }
}
```
아래와 같은 방식으로 사용해야 한다.  
여기서 JPA는 쿼리를 생성할 때 다음과 같이 생성할 수도 있다.
```
SELECT o.*, m.*
FROM order o
JOIN member m
```
즉, 조인 전체를 조회하게 된다.

### 2. 트랜잭션을 Spring Data가 다 해주지는 않는다.
- 오해 : Repsitory니까 다 자동 트랜잭션이다
- 실제 : 기본 제공 메서드와 선언한 쿼리 메서드는 다르다.

공식 문서에 따르면 선언한 쿼리 메서드는 기본 트랜잭션 설정이 자동 적용 되지 않는다. 필요하면 @Transactional을 직접 지정해야 한다.(사실 트랜잭션 필요하면 서비스에 하쥐...)

### 3. Auditing은 실무에서 꽤 유용하다.
Auditing이란 데이터를 누가, 언제 생성하고 수정했는지 자동으로 기록하는 기능이다.
예:
	•	createdAt
	•	updatedAt
	•	createdBy
	•	lastModifiedBy  
이러한 것들은 보통 모든 테이블에 들어가게 되는데 이를 매번 서비스 코드에 직접 넣으면 중복 코드가 많아지고, 누락이 되거나, 일관성이 깨진다.  
Spring Data는 이런 감사 기능을 인프라 차원에서 제공한다.

- 누가 생성했는가: @CreatedBy
- 언제 생성됐는가: @CreatedDate
- 누가 수정했는가: @LastModifiedBy
- 언제 수정됐는가: @LastModifiedDate

즉, 단순히 시간만 찍는 기능이 아니라
**“작성자/수정자 + 생성/수정 시각”**까지 함께 다룬다.

### 4. Null 처리와 Optional을 조심해야 한다.
조회 결과가 없을 수 있는 단건 조회는 ```Optional<T>```로 표현하는 경우가 많다. 예전 방식처럼 단건 조회를 그냥 엔티티 타입으로 받으면, 조회 결과가 없을때 null이 들어올 수 있다. 그러면 서비스 계층이나 컨트롤러에서 널처리를 해주어야 한다. Optional은 널 처리를 타입 수준에서 드러내는 도구이다.

단, Optional을 필드나 파라미터로 남용하면 안된다.
```
void save(Optional<User> user)
class UserDto { Optional<String> name; }
```
이러 식의 파라미터/필드로 사용하는 경우 코드 가독성을 오히려 떨어뜨릴 수 있다.

## 주의할 점
### 1. Spring Data는 JPA가 아니다.
- JPA : 자바 ORM 표준
- Hibernate : JPA 구현체
- Spring Data JPA : JPA를 더 쉽게 쓰도록 도와주는 추상화

즉, Spring Data JPA는 JPA 위에 올라간 편의 레이어이다.

### 2. N+1 문제를 Spring Data가 자동 해결해주지 않는다.
Spring Data는 Repository를 편하게 만들어주는 도구이지, 연관관계 조회 성능 문제를 자동 해결해주는 도구가 아니다. N+1은 JPA fetch 전략, fetch join, entity graph, projection 설계로 풀어야 한다. 

!!! "N+1이란?"
	```
	@Entity
	class Order {
    @Id Long id;

    @ManyToOne(fetch = FetchType.LAZY)
		private Member member;
	}

	@Entity
	class Member {
			@Id Long id;
			private String name;
	}
	```
	```
	List<Order> orders = orderRepository.findAll();
	for (Order order : orders) {
    System.out.println(order.getMember().getName());
  }
	```
	코드가 위와 같을 때 다음과 같이 조회 쿼리가 실행된다.
	
	```
	-- 1번
  select * from orders;

  -- order 개수만큼 추가 조회
  select * from member where id = ?;
  select * from member where id = ?;
  select * from member where id = ?;
  ...
	```
	처음 1번 + 연관 관계 객체 수만큼 N번 추가 조회가 나간다. Hibernate 문서는 연관 객체를 select fetching 하는 기본 방식이 N+1 selects 문제에 매우 취약하다고 설명한다.

### 3. 삭제/수정 쿼리는 읽기 쿼리와 다르다.
수정/삭제 쿼리는 영속성 컨텍스트와의 정합성, flush/clear 여부를 신경 써야 한다.
그 이유는 수정/삭제 쿼리는 영속성 컨텍스트를 건너뛰고 DB를 직접 때린다. 그래서 영속성 컨텍스트 상태와 DB 상태가 불일치하는 결과를 가져온다.

#### 읽기 쿼리
```
Order order = em.find(Order.class, 1L);
```
영속성 컨텍스트 확인 - 없으면 DB 조회 - 결과를 영속성 컨텍스트에 저장  
읽기 쿼리는 항상 영속성 컨텍스트 기준으로 동작한다.

#### 수정/삭제 쿼리
```
@Modifying
@Query("update Order o set o.status = 'CANCEL'")
int updateAll();
```
JPQL - 바로 SQL 실행
수정/삭제 쿼리는 바로 DB 수행을 한다.

#### 문제 상황
```
Order order = em.find(Order.class, 1L);

orderRepository.updateAll(); // bulk update

System.out.println(order.getStatus());
```
- 기대 : CANCEL
- 실제 : OLD STATUS

이유는 order 객체는 이미 영속성 컨텍스트에 있기 때문이다.  
DB는 바뀌었지만, 메모리는 안 바뀌었다. 그래서 정합성이 깨진다.

---

## 정리
Spring Data는 데이터 접근 계층을 인터페이스 중심으로 추상화해서 개발 생산성을 높이는 프레임워크이다.

1.	왜 쓰는가  
	반복적인 DAO 구현 코드를 줄이고, CRUD/페이징/정렬/쿼리 기능을 일관되게 제공하기 위해 쓴다.
2.	어떻게 동작하는가  
	Repository 인터페이스를 스캔하고, 프록시 빈을 만들고, 메서드 호출을 해석해서 실제 저장소 기술로 위임한다.
3.	핵심이 무엇인가  
	Repository 추상화, 런타임 프록시, 선언형 쿼리 작성이다.
4.	실무에서 진짜 중요한 것  
	Spring Data가 편의성을 주는 건 맞지만, JPA의 본질과 성능 문제까지 대신 해결해주지는 않는다는 점이다.

따라서 실무에서는 Repository를 잘 쓰는 것보다, 언제 단순 메서드 파생을 쓰고 언제 커스텀 쿼리로 넘어가야 하는지를 구분하는 능력이 더 중요하다.
