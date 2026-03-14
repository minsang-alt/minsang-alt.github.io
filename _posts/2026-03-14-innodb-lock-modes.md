---
title: InnoDB 락 모드. RECORD, X, GAP, REC_NOT_GAP의 관계
date: 2026-03-14 10:19:10 +/-0800
categories: [Tech Notes, MySQL]
tags: [mysql, innodb, lock, gap-lock, record-lock, next-key-lock]
---

`performance_schema.data_locks`를 조회하면 `LOCK_TYPE`과 `LOCK_MODE`라는 두 컬럼이 나온다.
이 글을 읽으면 각 값이 **무엇을 잠그는지**, 서로 어떤 관계인지, 왜 이렇게 설계되었는지를 이해할 수 있다.

---

## 먼저 혼동을 정리하자

`data_locks` 테이블의 결과를 처음 보면 이런 의문이 생긴다.

```
| LOCK_TYPE | LOCK_MODE     |
|-----------|---------------|
| RECORD    | X,GAP         |
| RECORD    | X,REC_NOT_GAP |
| RECORD    | X             |
```

**"LOCK_TYPE이 RECORD인데, LOCK_MODE에도 REC가 나오고 GAP도 나온다. 뭐가 뭐지?"**

답은 간단하다. **LOCK_TYPE과 LOCK_MODE는 서로 다른 차원**을 표현한다.

| 컬럼 | 의미 | 가능한 값 |
|------|------|-----------|
| **LOCK_TYPE** | 락이 걸리는 **수준** | `TABLE` (테이블 전체) / `RECORD` (인덱스 레코드 수준) |
| **LOCK_MODE** | 락의 **종류와 범위** | `X`, `S`, `X,GAP`, `S,GAP`, `X,REC_NOT_GAP`, `S,REC_NOT_GAP` 등 |

`LOCK_TYPE = RECORD`는 "이 락이 인덱스 레코드 수준에서 동작한다"는 분류일 뿐이다. **실제로 레코드를 잠그는지, gap을 잠그는지는 LOCK_MODE가 결정한다.**

---

## InnoDB가 인덱스를 보는 방식

LOCK_MODE를 이해하려면 InnoDB가 인덱스를 어떻게 바라보는지 알아야 한다.

InnoDB는 인덱스를 **레코드(점)와 gap(빈 공간)의 연속**으로 본다.

```
  gap    rec    gap    rec    gap    rec    gap
──( )── [10]  ──( )── [20]  ──( )── [30] ──( )── [supremum]
```

각 레코드 앞에는 반드시 gap이 존재한다. InnoDB는 **하나의 레코드와 그 앞의 gap을 묶어서 하나의 리소스**로 관리한다.

```
리소스 1: (gap before 10) + [10]
리소스 2: (gap before 20) + [20]
리소스 3: (gap before 30) + [30]
```

이 설계가 LOCK_MODE의 근거다. 하나의 리소스에 대해 **어느 부분을 잠글지** 모드로 지정한다.

---

## LOCK_MODE 6가지 분해

### 조합 규칙

```
X  =  X,REC_NOT_GAP  +  X,GAP    (레코드 + gap 모두 잠금)
S  =  S,REC_NOT_GAP  +  S,GAP    (레코드 + gap 모두 잠금)
```

| LOCK_MODE | 잠그는 대상 | InnoDB 공식 명칭 |
|-----------|------------|-----------------|
| `X` | 레코드 **+** 앞 gap | **Next-Key Lock** (Exclusive) |
| `S` | 레코드 **+** 앞 gap | **Next-Key Lock** (Shared) |
| `X,REC_NOT_GAP` | 레코드**만** (gap 제외) | **Record Lock** (Exclusive) |
| `S,REC_NOT_GAP` | 레코드**만** (gap 제외) | **Record Lock** (Shared) |
| `X,GAP` | 앞 gap**만** (레코드 제외) | **Gap Lock** (Exclusive) |
| `S,GAP` | 앞 gap**만** (레코드 제외) | **Gap Lock** (Shared) |

### 시각화

인덱스에 10, 20, 30이 있을 때, 레코드 20에 대한 각 모드가 잠그는 범위:

```
               (10, 20) gap      [20] record
               ─────────────     ──────────
X              ████████████████  ██████████    ← 둘 다
X,GAP          ████████████████  ··········    ← gap만
X,REC_NOT_GAP  ················  ██████████    ← 레코드만
```

---

## 왜 이렇게 설계했는가?

### 메모리 효율

레코드 락과 gap 락을 별도 객체로 관리하면 **매번 2개의 락 객체**가 필요하다. InnoDB는 하나의 리소스에 모드 비트만 바꿔서 관리한다. 일반적인 인덱스 스캔에서는 레코드와 gap을 동시에 잠그는 경우(Next-Key Lock)가 가장 많으므로, `X` 하나로 처리하는 것이 효율적이다.

### Lock Splitting (8.0.18+)

트랜잭션이 이미 `X,REC_NOT_GAP`을 가지고 있는데 `X`(전체)가 필요한 상황이 생길 수 있다.

```
필요한 것 - 이미 가진 것 = 추가로 필요한 것
X   - X,REC_NOT_GAP = X,GAP
```

InnoDB는 이 연산으로 부족한 부분(`X,GAP`)만 추가 요청한다. `*,GAP` 요청은 **절대 대기하지 않으므로** 즉시 획득된다. 불필요한 대기와 데드락을 방지하는 최적화다.

---

## 호환성 매트릭스

두 트랜잭션이 같은 리소스에 동시에 락을 걸 수 있는지를 결정하는 표다.

| 요청 ↓ \ 보유 → | S,REC_NOT_GAP | X,REC_NOT_GAP | *,GAP | S | X | *,INSERT_INTENTION |
|---|:---:|:---:|:---:|:---:|:---:|:---:|
| **S,REC_NOT_GAP** | ✅ | ⏳ | ✅ | ✅ | ⏳ | ✅ |
| **X,REC_NOT_GAP** | ⏳ | ⏳ | ✅ | ⏳ | ⏳ | ✅ |
| **\*,GAP** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| **S** | ✅ | ⏳ | ✅ | ✅ | ⏳ | ✅ |
| **X** | ⏳ | ⏳ | ✅ | ⏳ | ⏳ | ✅ |
| **\*,INSERT_INTENTION** | ✅ | ✅ | ⏳ | ⏳ | ⏳ | ✅ |

✅ = 즉시 획득 / ⏳ = 대기

### 핵심 규칙 3가지

**1. Gap Lock끼리는 절대 충돌하지 않는다**

`X,GAP`과 `X,GAP`이 같은 gap에 동시에 걸릴 수 있다. Gap Lock의 목적은 "이 범위에 INSERT를 막겠다"이므로, 여러 트랜잭션이 동시에 같은 gap을 보호해도 문제없다.

```sql
-- Session A: DELETE WHERE id = 15  → [20] 앞 gap에 X,GAP 획득
-- Session B: DELETE WHERE id = 17  → [20] 앞 gap에 X,GAP 획득 ← 충돌 없음!
```

**2. Insert Intention Lock은 Gap Lock과 충돌한다**

Insert Intention Lock은 "이 gap에 INSERT하겠다"는 의도다. Gap Lock이 "이 gap에 아무도 INSERT하지 마라"이므로 서로 비호환이다. **이것이 데드락의 핵심 원인이다.**

```sql
-- Session A: [20] 앞 gap에 X,GAP 보유
-- Session B: [20] 앞 gap에 INSERT 시도 → Insert Intention Lock 필요 → 대기!
```

**3. 이 매트릭스는 비대칭이다**

Insert Intention Lock은 Gap Lock을 **기다리지만**, Gap Lock은 Insert Intention Lock을 **기다리지 않는다**. 방향이 다르다.

---

## EXPLAIN과의 관계

EXPLAIN의 `type` 컬럼은 **데이터 접근 방식**을 알려준다. 이것으로 어떤 LOCK_MODE가 걸릴지 **추정**할 수 있지만, 직접적인 대응은 아니다.

| EXPLAIN type | 접근 방식 | REPEATABLE READ에서 예상 락 |
|---|---|---|
| `eq_ref` | UNIQUE 인덱스로 정확히 1건 조회 | `X,REC_NOT_GAP` (레코드만) |
| `ref` | 비유니크 인덱스 스캔 | `X` (Next-Key Lock, 레코드 + gap) |
| `range` | 범위 스캔 | `X` (Next-Key Lock) |
| `ALL` | 풀 테이블 스캔 | 모든 레코드에 `X` |

### 주의: EXPLAIN은 추정일 뿐이다

EXPLAIN이 `eq_ref`를 보여줘도 **실제로 매칭되는 레코드가 없으면** gap lock이 걸린다. 존재하지 않는 값을 조건으로 DELETE하면 해당 값이 들어갈 위치의 gap에 `X,GAP`이 걸린다.

```sql
-- 인덱스에 10, 20, 30이 있고, id=15인 레코드는 없는 상태
DELETE FROM t WHERE id = 15;
-- EXPLAIN: eq_ref (UNIQUE KEY 사용)
-- 실제 락: [20] 앞 gap에 X,GAP  ← 레코드가 없으니 gap만 잠금
```

이것이 이전 포스트에서 `eq_ref`인데도 `X,GAP`이 관찰된 이유다. DELETE 대상 레코드가 삭제되면 해당 위치는 gap이 되고, 다음 레코드를 기준점으로 gap lock이 남는다.

---

## 실전: data_locks 읽는 법

이전 포스트의 실험 결과를 다시 보자.

```
| 테이블 | 인덱스    | LOCK_TYPE | LOCK_MODE | 건수 |
|--------|----------|-----------|-----------|------|
| rsgucp | (TABLE)  | TABLE     | IX        | 1    |
| rsgucp | UK_oahu5r| RECORD    | X,GAP     | 1    |
```

이것을 해석하면:

1. **TABLE / IX**: 테이블 수준의 Intention Exclusive Lock. "이 테이블 안에서 행 수준 X Lock을 걸 예정"이라는 선언이다. 모든 행 수준 락 전에 자동으로 획득된다.

2. **RECORD / X,GAP**: UK_oahu5r(UNIQUE 인덱스)의 특정 레코드를 기준점으로, **그 앞의 gap만** exclusive로 잠근다. 레코드 자체는 이미 삭제되었으므로 레코드 락은 없고 gap만 남은 상태다.

### DELETE 후 왜 X,GAP만 남는가?

```
삭제 전:  gap [700001] gap [700002] gap [700003] gap ...
삭제 후:  gap ~~~~~~~~ gap [700002] gap ~~~~~~~~ gap ...
                                         ↑
                          삭제된 700003의 흔적 → [700004] 앞 gap에 X,GAP
```

DELETE는 레코드를 제거하면서 그 자리를 gap으로 만든다. REPEATABLE READ에서는 이 gap에 다른 트랜잭션이 INSERT하지 못하도록 gap lock을 유지한다. `data_locks`에는 **다음 존재하는 레코드를 기준점으로** `X,GAP`이 표시된다.

---

## 격리 수준에 따른 차이

| 동작 | REPEATABLE READ | READ COMMITTED |
|------|----------------|----------------|
| 인덱스 스캔 시 기본 락 | `X` (Next-Key Lock) | `X,REC_NOT_GAP` (Record Lock만) |
| 매칭 안 되는 레코드 | gap lock 유지 | **즉시 해제** |
| DELETE 시 gap lock | O | **X** (FK 체크, 중복 체크 시에만) |
| Insert Intention Lock 충돌 | O (gap lock과 비호환) | **X** (gap lock 없으므로) |

READ COMMITTED에서는 gap lock을 걸지 않으므로 Insert Intention Lock과의 충돌이 원천적으로 사라진다. 이것이 이전 포스트에서 격리 수준 변경으로 데드락을 해결할 수 있었던 이유다.

---

## 정리

| 혼동 포인트 | 정리 |
|------------|------|
| LOCK_TYPE = RECORD | 락의 **수준**이 행 수준이라는 의미. 레코드를 잠근다는 뜻이 아니다 |
| LOCK_MODE = X,GAP | 레코드 **앞의 gap만** 잠근다. 레코드 자체는 잠기지 않는다 |
| LOCK_MODE = X | gap + 레코드 **모두** 잠근다 (= Next-Key Lock) |
| LOCK_MODE = X,REC_NOT_GAP | 레코드**만** 잠근다. gap은 잠기지 않는다 |
| EXPLAIN의 eq_ref | 접근 방식일 뿐, 실제 락 모드와 1:1 대응하지 않는다 |
| X,GAP끼리 충돌? | **충돌하지 않는다.** Insert Intention Lock과만 충돌한다 |

---

## 참고 자료

- [MySQL 공식 문서 - InnoDB Locking](https://dev.mysql.com/doc/refman/8.0/en/innodb-locking.html)
- [InnoDB Data Locking - Part 2 "Locks"](https://dev.mysql.com/blog-archive/innodb-data-locking-part-2-locks/) — Lock 종류와 data_locks 테이블 해설
- [InnoDB Data Locking - Part 2.5 "Locks" (Deeper dive)](https://dev.mysql.com/blog-archive/innodb-data-locking-part-2-5-locks-deeper-dive/) — Lock Splitting, 호환성 매트릭스 상세
- [MySQL 공식 문서 - data_locks Table](https://dev.mysql.com/doc/mysql-perfschema-excerpt/8.0/en/performance-schema-data-locks-table.html)
- [MySQL 공식 문서 - InnoDB Transaction Isolation Levels](https://dev.mysql.com/doc/refman/9.1/en/innodb-transaction-isolation-levels.html)
