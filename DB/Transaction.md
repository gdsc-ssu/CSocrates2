- 트랜잭션의 기본 개념
- ACID 특성
- 트랜잭션 연산
- 트랜잭션 상태
- MySQL에서 트랜잭션이 어떻게 동작하는지
- 스토리지 엔진(MyISAM vs InnoDB)에 따른 동작 차이
- 트랜잭션 격리 수준 4단계와 관련 문제(Dirty Read, Phantom Read 등)
- Spring에서 `@Transactional`로 격리 수준을 설정하는 방법

## 트랜잭션(Transaction)이란?

### 정의
데이터베이스에서 하나의 논리적 작업 단위를 구성하는 연산들의 집합
모두 성공하거나 모두 실패해야 한다 (All or Nothing)

### 대표 예시
A 계좌에서 10만원 출금 → B 계좌에 10만원 입금 

중간에 오류가 나면? 출금만 되고 입금이 안 되면 돈이 사라진다
트랜잭션은 이 두 연산을 하나로 묶어 일관성을 보장한다


> [!info] INFO
> 트랜잭션의 목표 : **데이터 무결성을 지키는 것!**

## ACID 특성

트랜잭션을 제대로 이해하려면 **ACID 네 가지 속성**을 함께 살펴보는 것이 중요하다

### 1. 원자성 (Atomicity)
> "트랜잭션의 연산은 전부 실행되거나 전혀 실행되지 않아야 한다"

트랜잭션에 포함된 작업은 **하나의 단위처럼** 처리됨
즉, 모든 작업이 성공하면 **COMMIT**, 하나라도 실패하면 **ROLLBACK**

- 예: 송금 시 출금은 성공했지만 입금이 실패한 경우  
    → 전체 작업을 되돌려야 정상

**왜 중요한가?** 원자성이 없다면 절반만 실행된 상태가 DB에 남게 된다. 이는 데이터 불일치를 유발하며, 이후 비즈니스 로직에서 예측 불가능한 버그로 이어진다. 결제, 재고 차감, 포인트 지급처럼 여러 테이블을 동시에 변경하는 상황에서 원자성은 필수다

### 2. 일관성 (Consistency)
> "트랜잭션 전후로 DB의 무결성 제약 조건이 항상 만족되어야 한다"

트랜잭션이 끝난 후 데이터베이스는 정의된 모든 규칙(제약조건) 을 만족해야 함

- 예: 재고를 -1 감소시켰는데 주문 생성이 실패한 경우  
    → 데이터가 규칙을 어기게 되며, 트랜잭션은 이를 방지

**왜 중요한가?** 일관성은 단순히 오류를 막는 것을 넘어, 비즈니스 규칙 자체를 DB 레벨에서 강제한다는 의미다. NOT NULL, UNIQUE, FOREIGN KEY 같은 제약 조건이 항상 지켜지기 때문에, 잘못된 데이터가 쌓이는 것을 원천 차단할 수 있다.

### 3.  격리성 (Isolation)
> "동시에 실행되는 트랜잭션끼리 서로 간섭하지 않아야 한다"

여러 트랜잭션이 동시에 실행되더라도 **서로 간섭하지 않아야** 함
중간 결과가 노출되면 잘못된 데이터가 저장될 수 있음

- 예: 호텔 101호를 동시에 두 사용자가 예약한 상황  
    → 격리성이 충분하지 않다면 두 사람 모두 예약에 성공하는 문제가 발생

**왜 중요한가?** 실제 서비스는 수많은 요청이 동시에 들어온다. 격리성이 없다면 한 트랜잭션이 아직 완료되지 않은 데이터를 다른 트랜잭션이 읽거나 덮어쓰는 상황이 발생한다. 격리 수준을 어떻게 설정하느냐에 따라 성능과 정확성 사이의 트레이드오프가 결정된다. (격리 수준은 아래에서 자세히 다룬다.)

### 4. 지속성 (Durability)
> "커밋된 트랜잭션의 결과는 장애가 발생해도 영구적으로 반영된다"

트랜잭션이 커밋된 이후에는 결과가 **영구적으로 저장**됨
서버가 장애를 일으키더라도 데이터는 유지되어야 함

- 예: 결제 완료 기록이 장애로 사라져서는 안 됨
- InnoDB는 WAL(redo log)을 사용해 커밋된 내용을 안전하게 보존

**왜 중요한가?** 사용자 입장에서 "완료되었습니다" 메시지를 받은 이후 데이터가 사라지는 것은 심각한 신뢰 문제다. 지속성은 이를 DB 레벨에서 보장해준다. InnoDB는 커밋 전에 redo log에 먼저 기록하는 WAL(Write-Ahead Logging) 방식을 사용하기 때문에, 커밋 직후 서버가 다운되더라도 재시작 시 해당 내용을 복구할 수 있다.

## 트랜잭션 연산

트랜잭션은 크게 5가지 핵심 연산으로 구성된다.

### 1. BEGIN (= START TRANSACTION)

트랜잭션의 시작을 선언한다. 이 시점부터 이후의 모든 SQL 작업이 하나의 트랜잭션 단위로 묶인다.

```sql
BEGIN;
-- 또는
START TRANSACTION;
```

MySQL에서는 기본적으로 `autocommit = 1` 상태이기 때문에, 명시적으로 트랜잭션을 시작하지 않으면 SQL 하나하나가 자동으로 커밋된다. `BEGIN`을 사용하면 autocommit을 일시적으로 비활성화하고 수동 제어 모드로 전환된다.

### 2. COMMIT

트랜잭션 내의 모든 변경 사항을 DB에 영구적으로 반영한다. COMMIT이 완료된 이후에는 ROLLBACK으로 되돌릴 수 없다.

```sql
START TRANSACTION;

UPDATE accounts SET balance = balance - 10000 WHERE id = 1;
UPDATE accounts SET balance = balance + 10000 WHERE id = 2;

COMMIT; -- 두 작업 모두 영구 반영
```

COMMIT 시점에 InnoDB는 redo log에 기록하고, 이후 실제 데이터 파일에 반영한다. 덕분에 COMMIT 직후 장애가 발생해도 redo log를 통해 복구가 가능하다.

### 3. ROLLBACK

트랜잭션 시작 이후의 모든 변경 사항을 취소하고 원래 상태로 되돌린다.

```sql
START TRANSACTION;

UPDATE accounts SET balance = balance - 10000 WHERE id = 1;
-- 여기서 오류 발생

ROLLBACK; -- 출금 작업 취소, 원상 복구
```

ROLLBACK은 undo log를 기반으로 동작한다. InnoDB는 데이터를 변경하기 전에 원본 값을 undo log에 기록해두기 때문에, ROLLBACK 시 undo log를 참조해 이전 상태로 복원한다.

### 4. SAVEPOINT

트랜잭션 중간에 임시 저장 지점을 만든다. 전체를 ROLLBACK하지 않고 특정 지점까지만 되돌리고 싶을 때 사용한다.

```sql
START TRANSACTION;

UPDATE orders SET status = 'processing' WHERE id = 1;

SAVEPOINT order_updated; -- 중간 저장 지점 설정

UPDATE inventory SET stock = stock - 1 WHERE product_id = 10;
-- 재고 차감에서 오류 발생

ROLLBACK TO SAVEPOINT order_updated; -- 재고 차감만 취소, 주문 상태 변경은 유지

COMMIT; -- 주문 상태 변경만 최종 반영
```

SAVEPOINT는 복잡한 트랜잭션에서 부분적인 실패를 세밀하게 제어할 때 유용하다. 단, 남용하면 트랜잭션 흐름이 복잡해져 오히려 유지보수가 어려워질 수 있다.


### 5. RELEASE SAVEPOINT

더 이상 필요 없는 SAVEPOINT를 삭제한다. 해당 지점으로의 ROLLBACK이 불가능해지며, 메모리를 절약할 수 있다.

```sql
RELEASE SAVEPOINT order_updated;
```

### 연산 흐름 정리
```
BEGIN
  │
  ├─ SQL 작업 1
  ├─ SQL 작업 2
  ├─ SAVEPOINT sp1
  ├─ SQL 작업 3  ← 실패 시 ROLLBACK TO sp1
  ├─ SQL 작업 4
  │
  ├─ 성공 → COMMIT   (영구 반영)
  └─ 실패 → ROLLBACK (전체 취소)
````

### autocommit 주의사항

```sql
-- 현재 설정 확인
SHOW VARIABLES LIKE 'autocommit';

-- autocommit 끄기 (수동 커밋 모드)
SET autocommit = 0;

-- autocommit 켜기 (기본값)
SET autocommit = 1;
```

`autocommit = 1` 상태에서 `BEGIN` 없이 SQL을 실행하면 각 SQL이 즉시 커밋된다. 실수로 잘못된 UPDATE나 DELETE를 실행했을 때 되돌릴 수 없으므로, 중요한 작업 전에는 반드시 `BEGIN`으로 트랜잭션을 명시적으로 시작하는 습관이 중요하다.


## 트랜잭션 상태

트랜잭션은 시작부터 종료까지 5가지 상태를 거친다.
![[Pasted image 20260323014825.png]]

### 5가지 상태 설명

- **Active** — 트랜잭션이 시작되어 SQL 연산을 실행 중인 상태. `BEGIN` 이후 모든 작업이 여기에 해당한다. 정상 실행 시 Partially Committed으로, 오류 발생 시 Failed로 전이된다.
- **Partially Committed** — 마지막 연산까지 모두 실행 완료된 상태. 아직 `COMMIT`이 되지 않았기 때문에 변경 사항이 디스크에 영구 반영되지는 않은 시점이다. `COMMIT` 명령을 실행하면 Committed로, 이 과정에서 오류가 생기면 Failed로 전이된다.
- **Committed** — `COMMIT`이 완료되어 모든 변경 사항이 DB에 영구 반영된 상태. 이 시점 이후에는 `ROLLBACK`으로 되돌릴 수 없다. InnoDB는 이 시점에 redo log에 기록하여 장애 복구를 보장한다.
- **Failed** — 실행 도중 오류가 발생하거나 시스템 장애가 생긴 상태. 더 이상 정상적으로 진행할 수 없으며, 반드시 `ROLLBACK`을 통해 Aborted 상태로 전이되어야 한다.
- **Aborted** — `ROLLBACK`이 완료되어 트랜잭션 시작 이전 상태로 완전히 복구된 상태. 이 시점에서 두 가지 선택이 가능하다. 오류 원인을 수정한 후 트랜잭션을 재시작하거나, 트랜잭션 자체를 종료한다.

## MySQL에서의 트랜잭션

```sql
START TRANSACTION;

-- 작업 수행
UPDATE accounts SET balance = balance - 10000 WHERE id = 1;
UPDATE accounts SET balance = balance + 10000 WHERE id = 2;

-- 성공 시
COMMIT;

-- 실패 시
ROLLBACK;
```

트랜잭션은 필요한 최소 범위에서만 사용해야 한다. 
범위가 넓어질수록 락이 오래 유지되어 동시성 저하와 성능 문제가 발생할 수 있기 때문이다.

> [!inotice] TIP
트랜잭션 안에서 외부 API 호출, 파일 I/O, 이메일 발송 같은 작업은 하지 않는 것이 좋다. 이런 작업은 시간이 오래 걸리기 때문에 락을 불필요하게 오래 점유하게 된다

## MySQL 스토리지 엔진 비교

스토리지 엔진에 따라 트랜잭션 지원 여부와 동작 방식이 달라진다.
대표적으로 MyISAM과 InnoDB가 자주 비교된다

|기능|MyISAM|InnoDB|
|---|---|---|
|트랜잭션 지원|X|O|
|락 방식|테이블 락|행(Row) 락|
|외래 키 지원|X|O|
|동시성 처리|낮음|높음|
|충돌 후 복구|불가|자동 복구|
- **MyISAM** → 읽기 속도가 빠르지만 트랜잭션과 무결성 검증이 필요 없는 경우에만 적합
- **InnoDB** → 트랜잭션, 외래 키, 자동 복구 등 대부분의 서비스에서 요구하는 기능을 제공

### 왜 차이가 중요할까?

ROLLBACK 동작만 봐도 두 엔진의 차이가 명확하다.

- **MyISAM**: ROLLBACK을 실행해도 데이터가 그대로 남음
- **InnoDB**: ROLLBACK 시 변경 사항이 정상적으로 되돌아감

즉, 실무에서 트랜잭션 안정성이 필요하다면 선택지는 사실상 InnoDB다.

**락 방식의 차이도 중요하다.** MyISAM은 테이블 전체에 락을 걸기 때문에, 한 사용자가 데이터를 수정하는 동안 다른 모든 사용자가 해당 테이블에 접근하지 못한다. InnoDB는 수정 대상 행에만 락을 걸기 때문에 동시에 여러 사용자가 다른 행을 수정할 수 있어 처리량이 훨씬 높다.

## 트랜잭션 격리 수준 (Isolation Level)

격리성을 얼마나 엄격하게 적용할지를 단계별로 설정할 수 있다. 격리 수준이 높을수록 정확성이 올라가지만 성능은 떨어진다.

### 발생 가능한 문제 3가지

**Dirty Read** — 아직 커밋되지 않은 데이터를 다른 트랜잭션이 읽는 현상. T1이 값을 변경했지만 아직 커밋하지 않은 상태에서 T2가 그 값을 읽고, 이후 T1이 롤백하면 T2는 존재하지 않는 데이터를 기반으로 작업한 셈이 된다.

**Non-Repeatable Read** — 같은 트랜잭션 안에서 같은 데이터를 두 번 읽었을 때 값이 달라지는 현상. T1이 첫 번째로 읽은 뒤 T2가 해당 값을 변경·커밋하면, T1이 두 번째로 읽을 때 다른 값이 나온다.

**Phantom Read** — 같은 조건으로 조회했을 때 첫 번째에는 없던 행이 두 번째 조회에서 나타나는 현상. T1이 특정 조건으로 조회하는 사이 T2가 새 행을 삽입·커밋하면, T1의 두 번째 조회에서 행이 추가되어 있다.

### 격리 수준 4단계

|격리 수준|Dirty Read|Non-Repeatable Read|Phantom Read|
|---|---|---|---|
|READ UNCOMMITTED|발생|발생|발생|
|READ COMMITTED|방지|발생|발생|
|REPEATABLE READ|방지|방지|발생 가능|
|SERIALIZABLE|방지|방지|방지|
- **READ UNCOMMITTED** — 커밋되지 않은 데이터도 읽을 수 있다. 데이터 정확성이 거의 보장되지 않아 실무에서는 거의 사용하지 않는다.
- **READ COMMITTED** — 커밋된 데이터만 읽는다. PostgreSQL의 기본값이며, 대부분의 일반적인 웹 서비스에서 사용하기 적합한 수준이다.
- **REPEATABLE READ** — 같은 트랜잭션 내에서 같은 데이터를 읽으면 항상 같은 값을 보장한다. MySQL InnoDB의 기본값이며, MVCC(Multi-Version Concurrency Control)를 통해 구현된다. InnoDB는 이 수준에서 Phantom Read도 대부분 방지한다.
- **SERIALIZABLE** — 트랜잭션을 순차적으로 실행한 것처럼 동작한다. 모든 문제를 방지하지만 동시성이 가장 낮아 성능 부담이 크다. 금융처럼 정합성이 절대적으로 중요한 경우에만 제한적으로 사용한다

## Spring에서 `@Transactional` 격리 수준 설정

Spring에서는 `@Transactional` 어노테이션의 `isolation` 속성으로 격리 수준을 간편하게 지정할 수 있다.

```java
@Transactional(isolation = Isolation.READ_COMMITTED)
public void transferMoney(Long fromId, Long toId, int amount) {
    Account from = accountRepository.findById(fromId).orElseThrow();
    Account to = accountRepository.findById(toId).orElseThrow();

    from.withdraw(amount);
    to.deposit(amount);
}
```

### 실무에서의 선택 기준

격리 수준은 상황에 따라 다르게 설정하는 것이 좋다.

- **일반적인 조회 · 목록 API** → `READ COMMITTED`로도 충분한 경우가 많다. 성능 우선.
- **같은 트랜잭션 안에서 동일 데이터를 여러 번 읽어야 하는 경우** → `REPEATABLE READ`로 값이 바뀌지 않음을 보장한다.
- **재고 차감, 좌석 예약처럼 동시 접근이 치명적인 경우** → `SERIALIZABLE` 또는 비관적 락(`@Lock(LockModeType.PESSIMISTIC_WRITE)`)을 함께 고려한다. 

> [!info] **주의**
@Transactional`은 Spring AOP 기반으로 동작하기 때문에, 같은 클래스 내부에서 메서드를 직접 호출하면 트랜잭션이 적용되지 않는다. 반드시 외부에서 빈(Bean)을 통해 호출해야 한다.



