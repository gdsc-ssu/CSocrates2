---
tags:
  - topic/database
  - project/csocrates2
Date: 2026-03-29
---
# WAL & Crash Recovery

1. 커밋했는데 서버가 꺼지면?
2. 커밋 중에 죽으면, 절반만 쓰인 데이터는?
##### Durability
"커밋했다"는 약속은 서버가 꺼져도 유효해야 한다.
##### Atomicity
트랜잭션은 전부 반영되거나, 전혀 반영되지 않아야 한다.

## 1
### 1.1. Failure Classification
시스템에서 발생할 수 있는 failure는 크게 세 종류가 있다.

| Type                | Description                                 |
| ------------------- | ------------------------------------------- |
| Transaction Failure | 트랜잭션 자체의 실패 (e.g., Logical error, Deadlock) |
| System Crash        | 하드웨어 오류나 OS 버그로 메모리 내용이 소멸하는 경우             |
| Disk Failure        | 디스크가 물리적으로 손상되는 경우                          |
WAL과 Recovery는 System Crash를 대상으로 한다. 디스크는 살아있고 메모리만 날아갔다는 가정이다.

### 1.2. Failure Scenarios

- **Buffer Pool**
	- 디스크 접근을 줄이기 위해 데이터를 메모리에 올려두는 캐시

DB의 모든 읽기와 쓰기는 Buffer Pool 위에서 일어난다.
- 사용자가 `UPDATE accounts SET balance = 950 WHERE id = 'A'`를 실행하면
- DB는 해당 page를 Buffer Pool에 올리고
- 메모리 위에서 값을 바꾼다. 
- 디스크에는 나중에 몰아서 반영한다. (Lazy Loading)
	- 매번 디스크에 쓰면 너무 느리기 때문이다.

이때, 두 가지 문제가 생긴다.
#### Scenario 1: Durability 위반
- 트랜잭션이 `COMMIT`을 완료했다.
- DB는 클라이언트에게 성공이라고 응답했다.
- 그런데 그 데이터는 아직 메모리에만 있다. 디스크에는 아직 반영되지 않은 것이다.

이 상태에서 System Crash가 발생하면 메모리는 사라지고 디스크에는 변경 전 값이 남아있다.
커밋을 확인받은 사용자 입장에서는 데이터가 사라진 것이다.
$\rightarrow$ **Durability**를 보장하지 못하는 상황

#### Scenario 2: Atomicity 위반
- 트랜잭션이 진행 중이다. 아직 `COMMIT`은 하지 않았다.
- 그런데 DB는 메모리가 부족해지면 미완료 트랜잭션의 변경 사항이라도 디스크로 밀어낼 수 있다.

이 상태에서 System Crash가 발생하면
디스크에는 미완료 트랜잭션의 변경이 일부 반영된 채로 남는다.
$\rightarrow$ **Atomicity**를 보장하지 못하는 상황

## 2. Buffer Management
Buffer Pool을 어떻게 관리하는지에 따라 복구 방식이 결정된다.
- Dirty Page: 
	- Buffer Pool에서 수정됐지만 아직 디스크에 반영되지 않은 페이지
	- 값을 바꿨으나 디스크와 메모리가 아직 불일치한 상태
### 2.1. Database Buffering
데이터베이스의 데이터는 디스크에 저장되며
데이터가 필요할 때마다 메인 메모리(Buffer Pool)에 블록(Page) 단위로 로드한다.

문제는 디스크 크기에 비해 메모리는 현저히 작다는 점이다.
메모리가 꽉 차면 불가피하게 기존에 로드된 페이지를 지우고, (Evict)
새로운 페이지를 덮어써야 한다. (Overwrite)

| Category | Policy       | Description                                                      |
| -------- | ------------ | ---------------------------------------------------------------- |
| Evict    | **Steal**    | 커밋되지 않은 트랜잭션의 버퍼 페이지라도 메모리가 부족하면 디스크로 밀어냄.<br>**유연한 메모리 활용** 가능. |
|          | No-Steal     | 커밋 전에는 해당 트랜잭션이 수정한 페이지를 절대 디스크에 쓰지 않음.                          |
| Flush    | Force        | 커밋 시점에 변경된 모든 버퍼 페이지를 디스크의 데이터 파일에 강제 기록                         |
|          | **No-Force** | 커밋 응답을 반환하더라도 실제 디스크 기록은 나중으로 미룸.<br>**빠른 커밋 응답 속도** (I/O 최적화)   |
데이터베이스는 논리적으로 안전한 No-Steal & Force 조합 대신
디스크 I/O 성능을 극대화하기 위해 로그 관리를 감수하고 **Steal & No-Force** 정책을 표준으로 채택한다.

### 2.2. Steal 정책
Buffer Pool에 공간이 부족하면 DB는 어떤 페이지를 밀어내야 한다.
이때 아직 커밋되지 않은 트랜잭션의 Dirty Page도 디스크로 밀어낼 수 있게 허용하는 정책이 Steal Policy이다.
##### Steal Policy $\rightarrow$ Undo Log 필요
- 커밋되지 않은 데이터가 디스크에 쓰일 수 있다.
- 나중에 해당 트랜잭션이 롤백되거나, 크래시로 인해 취소되어야 할 수 있다.
- 이미 디스크에 쓰인 값을 되돌려야 한다.
### 2.3. No-Force 정책
커밋할 때 모든 Dirty Page를 즉시 디스크에 강제로 쓰지 않는 정책이 No-Force Policy이다.
커밋 응답을 빠르게 반환하기 위해 실제 디스크 쓰기는 나중으로 미룬다.
##### No-Force Policy $\rightarrow$ Redo Log 필요
- 커밋했더라도 데이터가 아직 디스크에 없을 수 있다.
- 서버 크래시가 나면 그 데이터는 사라진다.

## 3. WAL: Write-Ahead Logging

> 데이터 페이지를 디스크에 쓰기 전에
> 반드시 로그를 먼저 디스크에 써야 한다.

실제 데이터를 바꾸기 전에
"나는 이런 변경을 하려 한다"는 기록을 안전한 곳에 남겨두는 것이다.
이렇게 하면 크래시가 나도 로그가 살아있으면 복구할 수 있다.

### 3.1. Log Record
로그에는 무엇을 어떻게 바꿨는지가 기록되어 있다.
로그를 보면 되돌아가야 할 상태와 앞으로 가야 할 상태 모두를 알 수 있다.

로그 레코드는 다음과 같은 형식으로 기록된다.
```
<Ti, Xj, V1, V2>
 Ti  = 트랜잭션 식별자
 Xj  = 수정한 데이터 항목
 V1  = 수정 전 값 (old value)
 V2  = 수정 후 값 (new value)
```

예를 들어, 계좌 A에서 B로 50달러를 이체하는 트랜잭션 $T_0$ 의 로그는 다음과 같이 쌓인다.
```
<T0 start>
<T0, A, 1000, 950>    ← A를 1000에서 950으로 변경
<T0, B, 2000, 2050>   ← B를 2000에서 2050으로 변경
<T0 commit>
```

트랜잭션이 `write()`를 수행할 때마다
위와 같은 로그가 **실제 데이터 페이지보다 먼저** 디스크에 기록된다.

### 3.2. LSN: Log Sequence Number
각 로그 레코드에는 순서를 추적하기 위한 번호가 붙는다. (globally unique)
LSN으로부터 어떤 로그 레코드가 먼저 쓰였는지, 어떤 데이터 페이지가 마지막으로 수정되었는지 추적할 수 있다.

| LSN             | Location            | Description                           |
| --------------- | ------------------- | ------------------------------------- |
| **Flushed LSN** | Log Buffer (Memory) | 디스크 상의 WAL 파일에 `fsync()`가 완료된 마지막 오프셋 |
| **Page LSN**    | 각 Data Page         | 해당 데이터 페이지가 마지막으로 수정된 시점의 LSN         |
| Checkpoint LSN  | 시스템 메타              | 이 시점의 트랜잭션 상태와 Dirty Page 목록이 기록      |
- `pageLSN`은 트랜잭션이 데이터 페이지의 레코드를 수정할 때마다,
- `flushedLSN`은 DBMS가 WAL buffer의 내용을 디스크에 쓸 때마다 업데이트된다.

##### **WAL의 핵심 원칙**: `pageLSN ≤ FlushedLSN`
Buffer Pool Manager가 메모리의 특정 Dirty Page를 디스크로 flush하려 할 때
반드시 해당 페이지의 수정 내역을 담은 로그가 먼저 디스크에 안전하게 기록되어 있어야만 페이지 쓰기를 허용한다.
(=`flushedLSN` 갱신이 되어 있어야 페이지 쓰기를 허용한다.)

로그 없이 데이터 페이지가 먼저 쓰이면
크래시 시 변경 전후 상태를 알 수 없어 undo/redo 모두 불가능하다.
## 4. Recovery Algorithm
### 4.1. Redo Phase
No-Force 정책에 따라 커밋된 트랜잭션이라도 Dirty Page가 아직 디스크에 없을 수 있다.
이때 크래시가 나면 메모리에만 있던 최신 데이터가 사라지며 Durability가 깨진다.
```
로그 기록 있음 + 커밋 완료 → 디스크에 재실행 (Redo)
```
- 커밋했는데 디스크에 없으면
- 로그를 순방향으로 읽으며
- 로그에 기록된 `V2`(new value)를 순서대로 재실행해서 디스크에 반영한다.
### 4.2. Undo Phase
Steal 정책에 따라 커밋하지 않은 트랜잭션의 변경이 이미 디스크에 쓰였을 수 있다.
크래시 이후 그 트랜잭션은 커밋될 수 없으나 그 값은 디스크에 남아있다.
```
로그 기록 있음 + 커밋 미완료 → 변경 취소 (Undo)
```
- 커밋하지 않았는데 디스크에 있으면
- (복원된 상태에서) 로그를 역방향으로 읽으며
- 로그에 기록된 `V1`(old value)을 역순으로 복원한다.

### 4.3. Scenario A
크래시 직전에 다음과 같은 로그가 기록되어 있었다고 가정한다.
```
<T0 start>
<T0, A, 1000, 950>
<T0, B, 2000, 2050>
<T0 commit>
<T1 start>
<T1, C, 700, 600>
```
- `T0`은 `<T0 commit>` 레코드가 있지만, `T1`은 `<T1 commit>`이 없다.
	- `T0`: 커밋 완료되었으므로 Redo $\rightarrow$ A=950, B=2050을 디스크에 재반영한다.
	- `T1`: 커밋 미완료되었으므로 Undo $\rightarrow$ C=700으로 복원한다. (Atomicity)

**복구 후 최종 상태**: A=950, B=2050, C=700으로 커밋된 것만 남는다.

### 4.4. Scenario B
```
<T0 start>
<T0, A, 1000, 950>
          ↑ 여기서 크래시
```
- `T0`는 `<T0 commit>`이 없지만 Steal 정책에 의해 A=950이 이미 디스크에 쓰였을 수도 있다.
	- `T0`: 커밋 미완료되었으므로 Undo $\rightarrow$ A=1000으로 복원

**복구 후 최종 상태**: A=1000로 트랜잭션 T0의 흔적은 디스크에 남지 않는다.

### 4.5. Checkpoint

**Checkpoint가 없으면** 
로그를 처음부터 스캔해야 한다.
하지만 서비스가 오래 운영될수록 로그는 계속 쌓이므로 Checkpoint라는 개념이 필요하다.
- DB는 일정 시점마다 Buffer Pool의 Dirty Page를 디스크에 동기화하고
- 이 시점의 트랜잭션 상태와 Dirty Page 목록을 스냅샷으로 남긴다.

복구할 때는 로그 전체를 읽을 필요 없이  
- Dirty Page Table에 기록된 가장 오래된 `recLSN`부터 읽으면 된다.

$\therefore$ `pageLSN`과 `flushedLSN`은 특정 페이지를 flush해도 안전한지 판단하는 기준이 되고,
`checkpointLSN`이 복구 스캔의 시작점을 좁혀준다.

## 5. ARIES: Recovery Algorithm

ARIES(Algorithm for Recovery and Isolation Exploiting Semantics)는 이 복구 과정을 체계화한 알고리즘이다.
![](img/Pasted-image-20260329215951.png)
ARIES는 세 단계로 복구를 진행한다.
### 1) Analysis Pass
Checkpoint 지점부터 로그 끝까지 스캔한다. 이 스캔의 목적은 두 가지다.
1. 크래시 당시 살아있던 트랜잭션 목록을 파악한다.
	- 어떤 트랜잭션이 아직 커밋하지 못한 채 죽었는지 알아야 **Undo 대상**을 정할 수 있다.
2. Dirty Page 목록을 파악한다.
	- 어떤 페이지가 아직 디스크에 반영되지 않았는지 알아야 **Redo 범위**를 정할 수 있다.
### 2) Redo Pass
Dirty Page Table에서 가장 작은 recLSN부터
**커밋 여부와 무관하게** 로그를 **순서대로 재실행**한다.
- Undo를 정확히 수행하기 위해 크래시 직전 상태를 디스크에 재현한다.
### 3) Undo Pass
Analysis에서 확인된 미완료 트랜잭션들을 **역순으로 취소**한다.
Undo 로그에 기록된 `V1`(old value) 값으로 디스크의 변경사항을 되돌린다.
- 가장 나중에 일어난 변경부터 거꾸로 취소한다.

```
Analysis → "지금 뭐가 문제인지 파악"
Redo     → "일단 크래시 직전으로 되돌아가기"
Undo     → "완료 못 한 것들 정리"
```
이 세 단계가 끝나면 DB는 크래시 직전의 커밋된 상태와 동일한 상태가 된다.

## 6. InnoDB 실제 구현
아래 설정값은 "커밋 시점에 로그를 디스크에 얼마나 적극적으로 내려쓸지"를 결정한다.
```sql
innodb_flush_log_at_trx_commit
```

| Value | Action                             | Performance | Reliability                           |
| ----- | ---------------------------------- | ----------- | ------------------------------------- |
| 0     | 커밋마다 디스크에 쓰지 않음. <br>1초마다 flush    | 가장 빠름       | 가장 낮음<br>(DB 프로세스가 죽으면 1초치 데이터 손실 가능) |
| 1     | 커밋마다 디스크에 <br>즉시 flush (default)   | 가장 느림       | 가장 높음<br>(완전한 Durability)             |
| 2     | 커밋마다 OS 버퍼에 쓰되,<br>디스크 flush는 1초마다 | 중간          | 중간<br>(OS가 살아있으면 안전)                  |

![](img/Pasted-image-20260329221304.png)
## References

- Silberschatz et al., _Database System Concepts_, Chapter 19: Recovery System
- [15-445/645 Database Systems - #21: Database Crash Recovery](https://15445.courses.cs.cmu.edu/fall2020/notes/21-recovery.pdf)
- https://dev.mysql.com/doc/refman/9.6/en/innodb-parameters.html#sysvar_innodb_flush_log_at_trx_commit
- https://sqlconjuror.com/mariadb-mysql-innodb_flush_log_at_trx_commit/#google_vignette
- [Kakao Tech - MySQL InnoDB Log에 대한 이해 (1)](https://tech.kakao.com/posts/721)
