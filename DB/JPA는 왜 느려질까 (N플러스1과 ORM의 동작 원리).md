
### 전제
- ORM 사용
- 수만 개 이상의 데이터 존재
- LAZY Loading 사용
- 객체 간 연관관계가 존재
- 반복문/다수 데이터에서 연관 객체에 접근

# N+1 문제
- 연관된 엔티티를 반복 조회할 때 발생함
- 예시 상황 : 쇼핑몰의 각 주문에 대해 유저 이름을 가져오기 
	```
	List<Order> orders = orderRepository.findAll();
	
	for (Order order : orders) {
	    System.out.println(order.getUser().getName());
	}
	```
### 예시 상황 살펴보기
- 쇼핑몰 주문 조회 시 필요한 정보
	- 주문번호
	- 주문자 이름
	- 주문 시간
	- 주문상품들
 - Entity
	- Order
	- User
	- OrderItem
- 관계
	- Order : User = 1 : N
	- Order : OrderItem = 1 : N

-  `주문들을 가져왔으니까 주문 안의 user는 메모리에 있겠지?`
	→ 그렇지 않음
	
### N+1의 의미
- 최초 조회 : 1번 발생
	`SELECT * FROM orders;`
- 반복문 진입 시 추가 SQL이 발생함
	- `order.getUser().getName()`
	- forEach 이므로
		```
		SELECT * FROM users WHERE id = 1;
		SELECT * FROM users WHERE id = 2;
		SELECT * FROM users WHERE id = 3;
		...
		```
### 왜 이런 일이 발생할까 ?

#### **EAGER Loading (즉시 로딩) vs Lazy Loading (지연 로딩)**
- **EAGER Loading : Entity 조회 시, 연관된 Entity의 데이터도 함께 조회** (기본값)
	- `@ManyToOne`,`@OneToOne` 관계에서 `fetch = FetchType.EAGER` 인 경우
	- Entity의 연관관계는 무수히 많이 맺어져 있을 수 있음
	- 연관관계가 많을 경우 시스템 성능저하 발생

- **Lazy Loading : 연관된 Entity를 *"실제로 사용/접근할 때"* 연관된 Entity를 조회하는 것 (로딩을 지연)**
	- `@ManyToOne`,`@OneToOne` 관계에서 `@ManyToOne(fetch = FetchType.LAZY)` 인 경우
	- 작동 방식
		1. Entity Manager가 Entity를 로드 할 때
			JPA가 프록시 객체(HibernateProxy)를 실제 Entity를 대신해서 반환
			가짜 객체, 껍데기
		2. 프록시 객체 사용시점까지 실제 데이터를 로드하지 않음
			프록시 객체는 연관된 Entity에 대한 참조를 가지고 있지만, 그 Entity의 데이터는 로드하지 않음
			```
				Order
				 └── user = ProxyUser
			```
		3. 프록시 객체의 메서드 호출 시 실제 데이터 로딩
			프록시 객체의 메서드 중에서 실제 데이터가 필요한 시점(getter 호출 등)에 데이터베이스에서 실제 데이터를 가져와서 프록시 객체를 실제 객체로 대체한다.
			```
			order.getUser().getName()
			```
		4. 데이터가 로드되면 프록시 객체는 실제 객체로 교체
			한 번 프록시 객체가 초기화되면, 그 이후에는 프록시 대신 실제 객체가 사용됨.
			
- **기본적으로 Lazy Loading을 사용해야하는 이유**
	- 연관 데이터가 필요없는 경우에도 항상 조회하기 때문에 성능 저하 발생 가능
	- EAGER Loading은 예상하지 못한 SQL이 발생할 수 있음
	- 실무에서 대부분의 연관관게는 기본적으로 Lazy Loading을 사용하고, 필요한 곳에만 EAGER Loading을 사용
	- Lazy Loading을 사용할 때는 N+1문제에 대한 보완책을 사용함

### 어떻게 발견할 수 있을까?
1. SQL 로그 확인
	```YAML
	logging.level.org.hibernate.SQL=debug
	```
2. Query Count 측정
	```
	{
	  "queryCount": 101
	}
	```
3. API 갑자기 느려짐 (특히 리스트 조회 API)

## 여러가지 해결 방법
### 방법 1 Fetch Join
- 원래는 Order 조회 이후 User를 따로 조회 했다면
- Fetch Join 방법을 사용해 Order + User를 한번에 조회
- 쿼리 개수 N+1개 → 1개
- DB는 JOIN 최적화를 잘하므로 DB에게 맡긴다.
```Java
@Query("""
    SELECT o
    FROM Order o
    JOIN FETCH o.user
""")
List<Order> findAllWithUser();
```

```SQL
SELECT o.*, u.*
FROM orders o
JOIN users u ON o.user_id = u.id
```

#### Fetch Join 단점
##### 1. 중복된 행 발생
- Order : OrderItem = 1:N 이면 Join을 할 때 Order 하나에 여러 OrderItem이 존재함
- 중복 행 발생 시
	- 메모리 증가
	- 네트워크 전송량 증가
	- pagination 깨짐 (아래에서 다룸)
	- Cartesian Explosion : Join 연산이므로 여러 컬렉션 fetch join 시 연산이 엄청 많아짐
- 
  ```
Order1 ProductA
Order1 ProductB
Order1 ProductC
  ```
##### 2. Pagination 문제
- `@OneToMany fetch join` 과 pagination을 함께 사용할 때 메모리에서 pagination이 될 수 있는데 데이터가 많을 경우 성능 문제 발생 가능 (아래에서 다룸)
- 보통 OneToMany의 경우 다른 방법을 사용해 해결 (Batch size, DTO 조회, 두 번 조회)
	- `@ManyToOne fetch join`은 문제 없음
	- 두 번 조회 : 실무에서도 흔하게 사용함
		```
1. Order ID pagination
2. IN 조회로 연관 데이터 가져오기
		```

### 방법 2 @EntityGraph
- 이 조회에서는 연관 객체도 같이 가져오라고 선언적으로 지정하는 기능
- 기본 조회 : `findAll()` → Order만 조회
- EntityGraph 추가 → Order + User 함께 조회
	- 내부적으로는 fetch join 비슷하게 동작
		```java
		@EntityGraph(attributePaths = {"user"})
		List<Order> findAll();
		```
- 장점
	- 코드 간단
	- 선언적
	- Repository 레벨에서 쉽게 적용 가능
- 단점
	- 복잡한 조건/동적 조합 어려움
- 실무에서는
  간단한 조회는 EntityGraph 사용
  복잡한 조회는 Querydsl fetch join 사용
- QueryDSL : 타입 안전한 동적 쿼리 라이브러리
	- 문자열 JPQL 대신 Java 코드로 쿼리를 작성할 수 있음
	- 일반 JPQL은 문자열 기반이므로 오타 가능성 존재
		- `@Query("SELECT o FROM Order o JOIN FETCH o.user")`
	- QueryDSL 사용해 Java코드로 여러 복잡한 조건들을 동적으로 조립 가능함
		```
queryFactory
    .selectFrom(order)
    .join(order.user, user).fetchJoin()
    .fetch();
		```
### 방법 3 Batch Size
- 한 번에 가져오는 데이터 행(Row) 수를 늘려 쿼리 수를 줄임 (컬렉션은 따로 조회함)
- 장점
	- pagination 가능
	- 실무에서 제일 많이 씀
- 단점
	- 완전한 해결은 아님
	- 여전히 추가 쿼리가 존재함
- 설정
	```YAML
	spring.jpa.properties.hibernate.default_batch_fetch_size=100
	```
- Batch 적용 시 SQL
	```SQL
	SELECT * FROM users
	WHERE id IN (1,2,3,...)
	```

### 방법 4 DTO Projection
- 기본 컨셉 : 엔티티를 조회하지 말고 화면에 필요한 데이터만 가져오자
- 장점
	- 성능 좋음
	- 필요한 컬럼만 조회
	- API 최적화
- 단점
	- 재사용성 감소
	- 코드 증가
- ```Java
  select new OrderSummaryDto(
    o.id,
    u.name
)
  ```

### 해결 방법 선택 방법
- 해결책마다 Trade-Off가 존재함
- 조회 목적, 데이터 크기, 페이징 여부, 연관관계 방향 등을 종합적으로 고려해 전략을 선택해야 함


## ORM의 문제점
> [!warning] 객체 세계의 단위 ≠ DB 세계의 단위

- 그 전에 알고 갈 개념 Pagination
	- 대량 데이터를 효율적으로 처리하기 위한 전략
	- 메모리, DB, UI에서 사용
	- Pagination을 구현하는 대표적인 DB 기법이 LIMIT/OFFSET
		ex) 3페이지 데이터 가져오기
		```SQL
SELECT * FROM orders
LIMIT 20 OFFSET 40
		```

### 예시 : 주문 2개만 pagination해서 가져오기 => LIMIT 2
```
Order1 -> ItemA, ItemB, ItemC
Order2 -> ItemD
Order3 -> ItemE, ItemF
```
- JPQL 
	```JPQL
SELECT o  
FROM Order o  
JOIN FETCH o.orderItems
	```
- SQL
	``` SQL
SELECT *  
FROM orders o  
JOIN order_item oi ON oi.order_id = o.id  
LIMIT 2
	```
- Join 결과 행들을 기준으로 페이지네이션 결과에서 Order1만 가져옴
	``` 결과
	Order1 ItemA
	Order1 ItemB
	Order1 ItemC
	Order2 ItemD
	Order3 ItemE
	Order3 ItemF
	...
	```
- Limit 결과
	```
Order1 ItemA
Order1 ItemB
	```


Join 데이터를 부모/자식으로 볼 때
	→ DB 관점에서는 부모 row 증식

| 객체 관점<br>→ Order 객체 1개                                  | DB 관점<br>→ 결과 row 집합                         |
| ------------------------------------------------------- | -------------------------------------------- |
| Order1 (부모)<br> ├─ ItemA (자식)<br> ├─ ItemB<br> └─ ItemC | Order1 ItemA<br>Order1 ItemB<br>Order1 ItemC |
#### DB는 객체를 모르고 Row만 안다
우리가 원하는 객체 기준 중첩 그래프와 평면 Row 데이터를 억지로 연결함

| 우리가 원하는 것           | DB가 보는 것               |
| ------------------- | ---------------------- |
| Order 기준 3개 가져와라    | JOIN 결과 row 기준 3개      |
| 부모 객체 기준 pagination | JOIN row 기준 pagination |
| 객체의 중첩 그래프          | 평면 row                 |

#### ORM (JPA)의 역할과 한계
- DB에서 반환된 Row 결과를 보고 객체 그래프로 조립해야 함
	(객체 그래프 ↔ 평면 row 간의 변환)
- ORM은 `Order1.orderItems = [ ItemA, ItemB ]` 라고 객체를 만듦
- 실제 DB에는 ItemC도 존재하지만 DB에서는 Limit 때문에 ORM은 그 사실을 모름
- 객체는 참조(reference) 로 연결되고 그런 것처럼 보임, DB는 객체 그래프가 없이 Table, Row, Foreign Key만 존재함.
	- 객체는 필요하면 참조를 따라가면 되는 방식
		```Java
order.getUser().getAddress().getCity()
		```
	- DB입장에서는 처음부터 어떤 데이터를 가져올 지 명시해야 함
		```SQL
JOIN user
JOIN address
		```

### 발생하는 문제
- 객체 관점
	- order.getOrderItems()
	  → Order의 전체 상품 목록
	- 우리는 객체가 "완전한 상태"라고 믿고 사용함

- 실제로는 일부만 들어있음
	- ex) `if(order.getOrderItems().size() == 3)`
	- 개발자는 상품 3개 주문
	- 실제로는 2개만 로딩된 상태

- ORM 객체는 그래프 완전성(completeness)을 기대하지만 DB LIMIT는 ROW를 자른다.
	- Hibernate도 DB LIMIT를 사용하면 컬렉션 일부만 잘릴 위험이 있다는 사실을 알고 있음
	- 그래서 전부 가져와서 Java에서 객체 기준으로 안전하게 자르는 방법을 선택함.
	  → 메모리 Pagination
	- 이렇게 해야 Order 객체 하나가 완전한 컬렉션 상태를 보장할 수 있음

- 객체지향 코드에서 SQL이 숨어버리는데, ORM은 객체 그래프 탐색과 SQL 실행을 자동 변환한다.
	- 객체지향처럼 편리하게 사용할 수 있는 문제가 DB 성능 문제로 이어질 수 이어짐
	- 객체는 자유롭게 탐색 가능하지만, DB는 그때마다 조회가 필요하다
	- ORM의 근본적인 문제점임

### @OneToMany fetch join과 Pagination을 함께 사용했을 때의 위험성
- 평면 Row를 중첩 객체로 복원해야 함
- LIMIT로 중간 Row를 잘라버리기 때문에 객체 그래프가 깨지고 부분 객체(Partial Object)가 생길 위험이 있음.
- 이 때 "LIMIT 2"가 문제가 아니라, 무엇을 기준으로 2개인지가 중요함
	- 우리가 원하는 것은 Order 객체 2개이지만
	- DB가 실제로 하는 것은 JOIN row 2개임
-  Hibernate는 보통 DB pagination을 포기하고 메모리에서 pagination 수행
	- 컬렉션 일부만 로딩되는 것과 같이 객체 완전성 깨질 위험은 사라짐
	- 주문 10만개, 주문 상품 100만개 모두를 메모리에 가져와서 pagination을 해야 함 → 성능 문제 발생


## 결론
ORM은 SQL을 없애는 기술이 아니라 객체와 SQL 사이를 번역하는 기술로, N+1 문제는 단순한 JPA 실수가 아니라 객체 그래프와 관계형 DB의 구조 차이에서 발생하는 근본적인 차이에서 발생한다.

"객체 탐색 = SQL 실행 가능성" 이라는 관점을 반드시 가지고 JPA를 사용해야 한다

객체는 자유롭게 탐색할 수 있지만, DB 조회는 공짜가 아니라는 사실을 기억하자