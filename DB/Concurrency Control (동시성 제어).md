
# 1. 왜 동시성 제어가 필요할까?
## 1.1 DB 동시성

- DB는 항상 다음과 같은 상황에 놓여있음
	- 여러 사용자가 "동시에 요청"을 보냄
	- 여러 트랜잭션이 "같은 데이터"를 동시에 접근
	- 예시
		- A: 상품 재고 조회
		- B: 상품 재고 감소
		- C: 다시 재고 감소

## 1.2 동시에 실행되면 무슨 일이 생길까?

| ✅ 트랜잭션이 순서대로<br>    실행되면 문제 없음 | T1 → T2 → T3                                        |
| ------------------------------ | --------------------------------------------------- |
| **❗ 실제는 이렇게<br>   동시에 실행됨**    | **T1 -----  <br>    T2 -----  <br>       T3 -----** |
### A. Concurrency Anomalies
트랙잭션을 올바르게 수행하지 못하는 경우 발생할 수 있는 현상
1. **Dirty Read (다른 트랜잭션에서 수정 후 Commit되지 않은 값 읽기)**
2. **Lost Update (갱신 손실 = 덮어쓰기)**
3. **Non-Repeatable Read (하나의 트랜잭션, 두 번 이상 읽기 시, 읽을 때마다 다른 값 읽음)**

### B. 예시 : Lost Update (갱신 손실)

| Transaction A    | Transaction B    |                  |
| ---------------- | ---------------- | ---------------- |
| 재고 조회<br>→ 10개   |                  |                  |
|                  | 재고 조회<br>→ 10개   |                  |
| 재고 1개 감소<br>→ 9개 |                  |                  |
|                  | 재고 1개 감소<br>→ 9개 |                  |
|                  | 저장 (Commit)      |                  |
| 저장 (Commit)      |                  | → Lost Update 발생 |
- 상황 : **초기 재고 10개**
- 최종 재고 : 9개 → **잘못된 값 (8이 되어야 함)**
- 왜 이런 일이 발생했나?
	- 두 트랜잭션이 **같은 값을 읽음**
	- 서로의 변경을 **덮어씀**

## C. 핵심 포인트

> **데이터가 서로 간섭하면서 일관성이 깨짐** (ACID)
- 누군가의 작업이 다른 작업에 의해 덮어씌워짐
- 중간 상태를 읽어버림
- 결과가 실행 순서에 따라 달라짐

> DB는 기본적으로 "동시에 처리하려고" 설계된 것
- → 동시성 자체는 문제 X
- → 통제 (제어) 되지 않은 동시성이 문제 O

> 동시에 실행은 허용하되 결과는 **문제 없이 유지** 되도록 하는 것이 중요   


---

# 2. Serializability (직렬화, 직렬가능성)

## 2.1 Schedule (스케줄)
> **Schedule = 트랜잭션 연산들의 실행 순서**

예를 들어 2개의 트랜잭션 연산
``` 
T1: Read → Write  
T2: Read → Write
```

이 두 개가 섞이면 전체 실행 순설를 "하나의 Schedule"이라고 부름
```
T1 Read -> T2 Read -> T1 Write -> T2 Write
```

## 2.2 Serial vs Non-Serial
### A. Serial Schedule (직렬 실행)
```
  T1 → T2
```
- 한 번에 하나의 트랜잭션만 실행, 절대 섞이지 않음
	→ 항상 안전함
- 👉문제 : **"너무 느림 (성능↓)"** → 사용 불가
### B. Non-Serial Schedule (비직렬 실행)
```
  T1 Read
  T2 Read
  T1 Write
  T2 Write
```
- 트랜잭션이 섞여서 실행됨
- 대부분 실제 DB 상황
- 👉 문제:
	- 어떤 건 안전
	- 어떤 건 위험
 
## 2.3 Serializability의 개념
> Non-Serial Schedule이지만,
   결과가 어떤 Serial Schedule과 같다면 안전
   
이 결과를 만들어낼 수 있는 ‘순차 실행’이 존재하는가?
- 존재하면 → Serializable → 안전✔️
- 존재하지 않으면 → Not Serializable → 위험❌

## 2.4 Conflict Serializability
> 실제로는 충돌(Conflict)을 기준으로 Serializable 여부를 판단함
- Conflict Serializability
- 구현상 DBMS에서는 Transaction이 Conflict Serializable인지 확인하는 방식이 X
- 여러 Transaction을 동시에 실행해도 Schedule이 Conflict Serializable를 보장하도록 동작

![[Types-of-Schedules-in-DBMS.png]]

### 이론 : 분석/증명용 개념
- 어떤 Schedule이 주어짐
	- 실행 다 한 후 그래프를 그림
	- **Serializable인지 판단** → 아니면 롤백
### 실제 : Serializable Schedule만 실행
- 모든 트랜잭션 실행 기록 저장 후 검사 X
	- 너무 느림
	- 실시간 처리 불가
- 안전하지 않은 실행 자체를 못 하게 막음
	- 문제가 될 수 있는 실행 → 차단
	- 허용하는 실행 → 항상 Serializable

---

# 3. 동시성 제어 방식

> DB는 어떤 방식으로 실행을 통제할까?
> Non-Serial 실행을 Serial처럼 보이게 만들어야 함
  👉 크게 두 가지 방법을 사용 ( Lock, MVCC )
  
## 3.1 Lock 기반
> 충돌이 일어날 것 같으면, 기다리게 하자
- 핵심 : 같은 데이터에 동시에 접근하지 못하게 막음
- 구분 : 비관적 락 (Pessimistic Locking)
- 특징
	- 자연스럽게 Serial처럼 됨 (Read를 막음)
	- 강력하게 제어 가능
- 단점
	- 대기 발생 → 시간↑= 성능↓(상대적)
	- Deadlock 가능

## 3.2 MVCC 기반 (Multi-Version Concurrency Control)
> 충돌을 막지 말고, 애초에 안 생기게 하자
- 핵심 : 데이터를 여러 버전을 두어 서로 간섭이 없게 하자
- 구분 : 낙관적 락 (Optimistic Locking)
- 특징
	- Read가 Block되지 않음
	- 성능 좋음
- 단점
	- 구현 복잡한 편
	- 완전한 Serializable 보장은 아님

## 3.3 실제 DB
- 대부분의 DB는 하나만 사용하지 않고 둘 다 사용함

---

# 4. Lock 기반 동시성 제어
## 4.1 개요
- Lock이란 : 특정 데이터에 대한 접근 권한을 잠그는 것
- 충돌이 발생할 것 같으면 기다리게 함
- 순서를 강제하는 것
	- 실행 : 동시 허용
	- 충돌 : 직렬처럼 만들어버림 (순서를 강제)
## 4.2 Lock 종류
### A. Shared Lock (공유 락, 읽기 작업용)
- 여러 트랜잭션이 동시에 읽기 가능
- 쓰기는 동시에 불가능
### B. Exclusive Lock (배타 락, 쓰기 작업용)
- 완전 독점 : 나만 접근 가능
- 동시 읽기 & 쓰기 모두 불가능
### C. MySQL InnoDB 스토리지 엔진의 Lock
- Record Lock → 특정 행(Row) 하나를 잠금
	- 대부분의 DB에도 존재함
- Gap Lock → 값 사이 범위를 잠금
	- ex) id : 1,5 → gap = (1~5) 사이
- **Next-Key Lock → 행+범위**
	- InnoDB의 핵심
	- 행+ 그 앞의 범위까지 같이 잠금
	- Repeatable Read에서 **Phantom Read를 막기 위한 방법**
- 다른 DBMS도 구현방식은 다르지만, Phantom Read를 막기 위한 방법 존재
## 4.3 왜 Lock이 필요한가?
> 데이터베이스 무결성을 보장하기 위한 DB 핵심 메커니즘

위 Lost Update 예시
- Lock 사용, Serializable 보장
- 순서 T1 → T2

| Transaction 1    |              | Transaction 2    |
| ---------------- | :----------: | ---------------- |
| 재고 조회<br>→ 10개   | ← 재고 Lock 획득 |                  |
| 재고 1개 감소<br>→ 9개 |              | (대기)             |
| 저장 (Commit)      | ← 재고 Unlock  | (대기)             |
|                  |              | 재고 조회<br>→ 9개    |
|                  |              | 재고 1개 감소<br>→ 8개 |
|                  |              | 저장 (Commit)      |

## 4.4 Two-Phase Locking (2PL)
> Lock은 한 번에 다 잡고, 나중에 한 번에 풀어라

>Why?
>  → 2PL을 따르면 Conflict Serializable이 보장됨
### A. Growing Phase (확장 단계)
- Lock 획득만 가능
- 해제 ❌
### B. Shrinking Phase (축소 단계)
- Lock 해제만 가능
- 새 Lock 획득 ❌

![[two-phase locking.png]]
## 4.5 Deadlock (교착상태)
> Lock의 가장 큰 문제
> → 발생할 수 있다고 가정하고 처리해야 함.

- 서로 기다림 → 영원히 멈춤
	- T1: A Lock → B 기다림
	- T2: B Lock → A 기다림
- 발생 원인
	- Lock 순서가 다름
	- 서로 자원을 잡고 있음
- 해결 방법
	1. Detection 탐지
		- 하나를 강제 롤백
	2. Timeout
		- 일정 시간 기다리다 실패 처리

## 4.6 Lock 방식의 특징과 한계
> Lock은 충돌을 막기 위해 실행 순서를 강제로 만든다

| 장점         | 단점       |
| ---------- | -------- |
| 강력한 일관성 보장 | 대기 발생    |
| 직관적        | Deadlock |
|            | 성능저하     |
### Lock의 현실적 한계 = 성능저하
1. Lock Contention (경합)
	- 여러 트랜잭션이 하나의 데이터를 기다림
	- 대기 시간 증가
2. Blocking : 읽기 쓰기 모두 Block 가능
3. Deadlock

---

#  5. MVCC (Multi-Version Concurrency Control)

> Lock은 안전하지만 (데이터 일관성↑)
> 너무 자주 기다려야 하고 성능이 느림
> → 굳이 읽기까지 막아야 할까?

> 💡 MVCC의 핵심 아이디어
>  막지 말고, 각자 다른 데이터를 보게하자

![[mvcc.png]]

## 5.1 동작 방식
### A. Lock과의 비교
- 기존 Lock 기반 방식
	`T1: Write 중`
	`T2: Read → 대기`
- MVCC 기반
	`T1: Write 중 (새 버전 생성)`
	`T2: Read (이전 버전 읽음)`
### B. 동작 방식
#### a. Undo Log
> 이전 버전 데이터 저장
- 데이터 수정 시 (Update / Delete ) 수행
- 과거 값을 Undo Log에 보관해 MVCC에서 조회
#### b. Read View
> 트랜잭션이 어떤 버전을 볼 지 결정하는 기준
- 트랜잭션 시작 시 생성
- 이 시점까지 커밋된 데이터만 본다 를 정의
#### c. Redo Log (보조)
> 장애 복구용
- 최신 데이터 복구용
- MVCC와 직접적 관계 X
#### d. Buffer Pool (보조)
> 캐시 메모리
- 디스크 접근 줄임
- 성능 개선 목적
#### e. 핵심 흐름
```
1. 트랜잭션 시작
2. Read View 생성
3. 데이터 조회
   → 최신이 아니라
   → 조건에 맞는 과거 버전 선택
```

## 5.2 MVCC 버전 관리 방식
### A. 스냅샷 기반 (Snapshot)
> 트랜잭션 시작 시점 기준
- 가장 일반적인 방식
- MySQL InnoDB, PostgreSQL
### B. 타임스탬프 기반 (TimeStamp)
> 각 데이터에 시간 정보 부여
- 읽을 때 시점보다 이전 것만 읽을 수 있음
### C. 이력 기반 (History-based)
> 버전 체인을 따라가며 조회
- Undo Log 기반 구조
### D. 하이브리드
- 여러 방식 혼합
- 
## 5.3 MVCC의 특징과 장단점
> 시간에 따라 도일 데이터의 버전을 여러 개 유지
> 각 트랜잭션은 자기 시점의 데이터만 읽기
### A. 특징
- 읽기는 쓰기를 차단하지 않음
- 쓰기는 읽기를 차단하지 않음
- 트랜잭션 격리성 보장
- 일관성 읽기 제공
### B. 장점
- 읽기 성능 개선
	- Non-Blocking Read
	- 읽기가 절대 막히지 않음
- 성능 향상
	- 락 경합 감소 → 동시성 향상
### C. 한계
- 완벽하지 않음
- Write-Write 충돌은 막지 못함
> Lock과 함께 사용됨
> : MVCC는 Lock을 보완하는 기법
- Pantom Read 방지는 Serializable Isolation에서만 가능
	> Phantom Read ?
	>  : 범위 쿼리 시 새로운 데이터 등장

	- Next-Key Lock (InnoDB 방식)
- 오래된 버전 누적 (Storage 문제)
	- Undo Log가 계속 쌓이면 공간 증가
		- GC (가비지 컬렉터) 필요
			- ex. PorstgreSQL - VACUUM
			  MySQL InnoDB - Purge Thread
	- 오버헤드, 데이터베이스 용량 확장 필요
- Snapshot Too Old (Snapshot Anomaly)
	- 트랜잭션이 오래된 스냅샷을 참조
	- 최신 데이터와 불일치 발생
	- 더 높은 격리수준을 사용
	  / 애플리케이션 레벨에서 데이터 동기화 처리 필요

## 5.4 DB별 MVCC 구현 차이

MVCC는 개념이 있지만, DB마다 구현 방식이 다름

### A. 🏗️ MySQL (InnoDB)
>MVCC + Lock 혼합
- Undo Log 기반
	- Purge 자동정리 가능 → 효율적
- Read View 사용
- Repeatable Read에서 Phantom 방지 (Next-Key Lock)
- Buffer Pool과 통합해 좋은 성능
### B. 🐘 PostgreSQL
> 데이터 자체가 버전
- Tuple 자체에 버전 저장
  → 간편/단순구조, 빠른 읽기 가능
- Vacuum으로 정리 → 오버헤드 발생
- 용량 문제 발생
### C. 🏢 Oracle
> Snapshot 기반 강력
- Undo Segment 기반
	- 순환 재사용으로 공간 효율적
- Read Consistency 강함
- 복잡하지만 효율적, 안정적 성능




# 결론
> 동시성 제어는 “일관성과 성능의 트레이드오프”
- Lock → 안전하지만 느림
- MVCC → 빠르지만 복잡