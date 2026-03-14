---
title: MySQL Gap Lock 데드락. 가설 반증과 격리 수준 변경으로 해결
date: 2025-12-24 10:19:10 +/-0800
categories: [Tech Notes, MySQL]
tags: [mysql, deadlock, gap-lock, isolation-level]
---

## TL;DR

Kafka 컨슈머들이 동일 테이블에 DELETE + INSERT를 병렬로 수행하면서 **REPEATABLE READ의 gap lock** 때문에 데드락이 발생했다.

처음에는 서브쿼리가 원인이라고 판단했다. 하지만 `performance_schema.data_locks`로 실제 락을 비교한 결과, **서브쿼리 제거는 효과가 없었다**. 근본 원인은 격리 수준 자체였고, **READ COMMITTED**로 변경하여 해결했다.

---

## 1. 문제 현황

### 증상

- **에러**: `CannotAcquireLockException: Deadlock found when trying to get lock; try restarting transaction`
- **발생 위치**: `ReviewSubGroupUserCalculationPointBulkRepository.processChunk():43` (INSERT)
- **발생 빈도**: 2025년 12월 15일부터 거의 매일 반복
- **영향**: Kafka 이벤트(`CalculateOverallGradeEvent`) 처리 실패 → 종합등급 계산 중단

### 아키텍처 흐름

```
Kafka Consumer (CalculateOverallGradeEventHandler)
  └─ @Transactional
       ├─ 1. deleteExistingPoints(reviewGroup)     ← DELETE
       ├─ 2. calculateOverallGrade(reviewGroup)     ← INSERT (BulkInsert)
       └─ 3. markComplete()
```

각 컨슈머는 **서로 다른 ReviewGroup**을 처리하지만, 모두 같은 테이블에 DELETE + INSERT를 수행한다.

### 테이블 구조

```sql
CREATE TABLE review_sub_group_user_calculation_point (
  id bigint NOT NULL AUTO_INCREMENT,
  review_sub_group_user_id bigint,       -- UNIQUE KEY
  numerator decimal(38,0) NOT NULL,
  denominator decimal(38,0) NOT NULL,
  ...
  PRIMARY KEY (id),
  UNIQUE KEY UK_oahu5r (review_sub_group_user_id),   -- ★ 핵심 인덱스
  KEY FK5h2bd (overall_grade_id)
);
```

`review_sub_group_user_id`의 **UNIQUE KEY**가 이후 gap lock 분석의 핵심이 된다.

---

## 2. 초기 가설: 서브쿼리가 원인?

### 가설

서브쿼리가 불필요한 gap lock을 유발한다고 판단했다. 서브쿼리를 제거하고 리터럴 IN절로 바꾸면 데드락이 해결될 것이라고 예상했다.

**기존 쿼리**

```sql
DELETE FROM review_sub_group_user_calculation_point p
WHERE p.review_sub_group_user_id IN (
    SELECT rsgu.id
    FROM review_sub_group_user rsgu
    JOIN review_sub_group rsg ON rsgu.sub_group_id = rsg.id
    WHERE rsg.review_group_id = ?
)
```

**변경 쿼리** (서브쿼리 제거)

```sql
-- Step 1: Java에서 ID 조회
SELECT rsgu.id FROM review_sub_group_user rsgu WHERE ...

-- Step 2: 리터럴 IN절 DELETE
DELETE FROM review_sub_group_user_calculation_point p
WHERE p.review_sub_group_user_id IN (450800, 450801, 450802, ...)
```

### EXPLAIN 분석

```
테이블                                          | type    | key
review_sub_group (rsg)                          | ref     | ix_unique_review_sub_name
review_sub_group_user (rsgu)                    | ref     | FKhkw0i6oid0tg9546898y1kvw5
review_sub_group_user_calculation_point (p)     | eq_ref  | UK_oahu5r (UNIQUE KEY)
```

`review_sub_group_user_calculation_point`는 **eq_ref**(UNIQUE KEY 포인트 룩업)로 접근한다. MySQL 공식 문서에 따르면 eq_ref는 **record lock만 사용하고 gap lock을 걸지 않는다**. 이 분석 결과만 보면 gap lock이 발생할 이유가 없었고, 실제로 로컬에서 데드락이 재현되지 않았다.

---

## 3. 가설 반증: `performance_schema.data_locks` 실험

### 실험 방법

트랜잭션을 커밋하지 않은 상태에서 `performance_schema.data_locks`를 조회하여 실제 락을 비교했다.

### 실험 1: 서브쿼리 DELETE의 락

```sql
SET autocommit = 0;
DELETE FROM review_sub_group_user_calculation_point
WHERE review_sub_group_user_id IN (
    SELECT rsgu.id FROM review_sub_group_user rsgu
    JOIN review_sub_group rsg ON rsgu.sub_group_id = rsg.id
    WHERE rsg.review_group_id = 372517
);
-- 4901건 삭제, 커밋하지 않고 data_locks 조회
```

**결과**:

| 테이블 | 인덱스 | LOCK_TYPE | LOCK_MODE | 건수 |
| --- | --- | --- | --- | --- |
| rsgucp | (TABLE) | TABLE | IX | 1 |
| rsgucp | UK_oahu5r | RECORD | **X,GAP** | 1 |
| rsgu | FKhkw0i | RECORD | S | 4908 |
| rsgu | FKhkw0i | RECORD | S,GAP | 1 |
| rsg | ix_unique | RECORD | S | 1 |

### 실험 2: 리터럴 IN절 DELETE의 락

```sql
SET autocommit = 0;
DELETE FROM review_sub_group_user_calculation_point
WHERE review_sub_group_user_id IN (450800, 450801, ..., 450820);
-- 20건 삭제, 커밋하지 않고 data_locks 조회
```

**결과**

| 테이블 | 인덱스 | LOCK_TYPE | LOCK_MODE | 건수 |
| --- | --- | --- | --- | --- |
| rsgucp | (TABLE) | TABLE | IX | 1 |
| rsgucp | UK_oahu5r | RECORD | **X,GAP** | 1 |

### 핵심 발견

두 방식 모두 대상 테이블의 UNIQUE 인덱스에 **동일한 X,GAP 락**을 건다.

유일한 차이는 서브쿼리 방식이 `rsgu` 테이블에 추가 공유 락(S lock)을 거는 것뿐이다. 이 락은 데드락과 무관하다.

> 서브쿼리를 제거해도 데드락은 해결되지 않는다.

---

## 4. 실제 데드락 재현

### 재현 조건: 인터리브된 ID

로컬에서 재현되지 않은 이유가 있었다. **테스트 데이터의 rsgu_id가 ReviewGroup별로 연속**되어 있었기 때문이다.

```
ReviewGroup A의 rsgu_id: 433321 ~ 438220  (연속)
ReviewGroup B의 rsgu_id: 450800 ~ 455701  (연속)
→ ID 범위가 겹치지 않음 → gap lock 범위가 겹치지 않음 → 데드락 불가
```

하지만 프로덕션에서는 여러 ReviewGroup에 사용자가 시간 순서대로 추가된다. 전역 AUTO_INCREMENT로 ID가 발급되므로 **ReviewGroup 간 ID가 인터리브**된다

```
ReviewGroup A의 rsgu_id: 1, 3, 5, 7, 9, 11, ...  (홀수)
ReviewGroup B의 rsgu_id: 2, 4, 6, 8, 10, 12, ...  (짝수)
→ ID 범위가 완전히 겹침 → gap lock 충돌 가능
```

### 인터리브 테스트 데이터 생성

```sql
-- ReviewGroup A (sub_group 24704) → 홀수 ID
INSERT INTO review_sub_group_user (id, sub_group_id, user_id) VALUES
  (700001, 24704, 1), (700003, 24704, 1), ..., (700019, 24704, 1);

-- ReviewGroup B (sub_group 31494) → 짝수 ID
INSERT INTO review_sub_group_user (id, sub_group_id, user_id) VALUES
  (700002, 31494, 1), (700004, 31494, 1), ..., (700020, 31494, 1);

-- 양쪽 모두 calculation_point 생성
INSERT INTO review_sub_group_user_calculation_point
  (review_sub_group_user_id, numerator, denominator, is_excluded_from_rating) VALUES
  (700001, 100, 100, 0), (700002, 100, 100, 0), ..., (700020, 100, 100, 0);
```

### 동시 세션 데드락 재현

```
Session A (ReviewGroup A - 홀수)              Session B (ReviewGroup B - 짝수)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

① BEGIN
   DELETE ... WHERE rsgu_id IN
   (700001,700003,...,700019)
   → 삭제된 레코드에 X lock
   → 인접 gap에 X,GAP lock 획득
                                           ② BEGIN (0.5초 후)
                                              DELETE ... WHERE rsgu_id IN
                                              (700002,700004,...,700020)
                                              → 삭제된 레코드에 X lock
                                              → 인접 gap에 X,GAP lock 획득

③ INSERT (2초 후)                           ④ INSERT (1.5초 후)
   (700001,700003,...,700019)                  (700002,700004,...,700020)
   → Insert Intention Lock 필요               → Insert Intention Lock 필요
   → Session B의 gap lock과 충돌! ⏸️          → Session A의 gap lock과 충돌! ⏸️

                    💥 DEADLOCK DETECTED!
```

### 재현 결과

**REPEATABLE READ (기본)** — 리터럴 IN절 DELETE:

```
Round 1: DEADLOCK
Round 2: DEADLOCK
Round 3: OK
Round 4: DEADLOCK
Round 5: OK
결과: 3/5 데드락 발생
```

**REPEATABLE READ (기본)** — 서브쿼리 DELETE:

```
결과: 동일하게 데드락 발생
```

> **서브쿼리든 리터럴 IN이든 상관없이 데드락이 발생한다.**

---

## 5. InnoDB 데드락 로그 분석

재현에 성공한 후 `SHOW ENGINE INNODB STATUS`로 데드락 로그를 확인했다

```
*** (1) TRANSACTION: (TxB - 짝수 INSERT)
HOLDS THE LOCK(S):
  index UK_oahu5r  lock mode S locks gap before rec
  → rsgu_id=700003 앞의 gap에 S,GAP lock 보유

WAITING FOR THIS LOCK TO BE GRANTED:
  index UK_oahu5r  lock_mode X locks gap before rec insert intention waiting
  → rsgu_id=700003 위치에 INSERT 시도 → Insert Intention Lock 대기

*** (2) TRANSACTION: (TxA - 홀수 INSERT)
HOLDS THE LOCK(S):
  index UK_oahu5r  lock_mode X locks gap before rec
  → 여러 위치에 X,GAP lock 보유

WAITING FOR THIS LOCK TO BE GRANTED:
  insert intention waiting
  → 다른 위치에 INSERT 시도 → TxB의 gap lock과 충돌
```

### 데드락 메커니즘 정리

```
Gap Lock과 Insert Intention Lock의 비호환성:

┌──────────────────────────────────────────────────────┐
│  Gap Lock (X,GAP / S,GAP)                            │
│  → "이 범위에 아무도 INSERT하지 마라"                    │
│                                                      │
│  Insert Intention Lock                               │
│  → "이 위치에 INSERT하겠다"                             │
│                                                      │
│  ★ 이 두 락은 서로 호환되지 않는다 (MySQL 공식 문서)      │
│  → Gap Lock이 걸린 범위에 INSERT 시도 시 대기 발생       │
│  → 양방향으로 대기하면 DEADLOCK                         │
└──────────────────────────────────────────────────────┘
```

---

## 6. 근본 원인

근본 원인은 서브쿼리가 아니다. **REPEATABLE READ 격리 수준에서 DELETE가 거는 gap lock**이 원인이다.

| 요소 | 설명 |
| --- | --- |
| **격리 수준** | REPEATABLE READ → DELETE 시 gap lock 획득 |
| **인덱스 구조** | `review_sub_group_user_id`에 UNIQUE KEY |
| **데이터 분포** | 서로 다른 ReviewGroup의 rsgu_id가 전역 AUTO_INCREMENT로 인터리브 |
| **동시성** | Kafka 컨슈머가 서로 다른 ReviewGroup을 병렬 처리 |
| **작업 패턴** | 같은 트랜잭션 내에서 DELETE → INSERT (같은 rsgu_id 재삽입) |

### 서브쿼리 제거가 효과 없는 이유

1. UNIQUE 인덱스의 eq_ref 접근은 서브쿼리 유무와 무관하게 **동일한 gap lock**을 건다
2. `performance_schema.data_locks` 비교 결과, 두 방식 모두 `X,GAP` 패턴이 동일하다
3. 인터리브 데이터로 테스트하면 두 방식 모두 같은 빈도로 데드락이 발생한다

---

## 7. 해결: READ COMMITTED 격리 수준

### READ COMMITTED에서 gap lock이 사라지는 이유

MySQL 공식 문서는 READ COMMITTED의 락 동작을 다음과 같이 설명한다:

> "For locking reads, UPDATE statements, and DELETE statements, InnoDB locks only index records, not the gaps before them, and thus permits the free insertion of new records next to locked records. **Gap locking is only used for foreign-key constraint checking and duplicate-key checking.**"
>
> — [InnoDB Transaction Isolation Levels](https://dev.mysql.com/doc/refman/9.1/en/innodb-transaction-isolation-levels.html)

|  | REPEATABLE READ | READ COMMITTED |
| --- | --- | --- |
| DELETE 시 gap lock | O (Phantom Read 방지) | **X** (record lock만) |
| INSERT intention lock 충돌 | O (gap lock과 비호환) | **X** (gap lock 없으므로) |
| 데드락 가능성 | **높음** | **낮음** |

### 검증 실험

**READ COMMITTED** — 동일한 인터리브 테스트 데이터:

```
Round 1: OK
Round 2: OK
Round 3: OK
Round 4: OK
Round 5: OK
결과: 0/5 데드락 발생
```

### 코드 변경

```java
// CalculateOverallGradeEventHandler.java
@Transactional(isolation = Isolation.READ_COMMITTED)  // 변경
@KafkaHandler
public void handle(final CalculateTotalOverallGradeEvent event) { ... }

@Transactional(isolation = Isolation.READ_COMMITTED)  // 변경
@KafkaHandler
public void handle(final CalculateOverallGradeEvent event) { ... }

// OverallGradeCalculationService.java
@Transactional(isolation = Isolation.READ_COMMITTED)  // 변경
public void calculateOverallGrade(long reviewGroupId) { ... }

@Transactional(isolation = Isolation.READ_COMMITTED)  // 변경
public void calculateOverallGrade(OverallGradeCalculationInfo request, ...) { ... }
```

변경 범위는 **2개 파일, 4개 메서드**다. `@Transactional`의 isolation 속성만 수정했고, 비즈니스 로직은 건드리지 않았다.

### READ COMMITTED 사용 시 고려사항

| 항목 | 영향 |
| --- | --- |
| Phantom Read | 같은 ReviewGroup을 동시에 처리하지 않으므로 영향 없음 |
| Non-repeatable Read | DELETE → INSERT를 한 트랜잭션에서 순차 실행하므로 영향 없음 |
| FK 제약조건 | READ COMMITTED에서도 FK 체크 시 gap lock은 유지된다 (MySQL 보장) |
| 성능 | gap lock이 사라지면서 동시성이 향상된다 |

---

## 8. 실험 결과 요약

| # | 테스트 | 격리 수준 | DELETE 방식 | 데드락 |
| --- | --- | --- | --- | --- |
| 1 | 서브쿼리 DELETE | REPEATABLE READ | 서브쿼리 | **발생** |
| 2 | 리터럴 IN DELETE | REPEATABLE READ | 리터럴 IN | **발생 (3/5)** |
| 3 | 리터럴 IN DELETE | **READ COMMITTED** | 리터럴 IN | **없음 (0/5)** |

> **DELETE 방식(서브쿼리 vs IN절)은 데드락과 무관하다.** 격리 수준을 REPEATABLE READ에서 READ COMMITTED로 바꾸는 것이 유일한 해결책이었다.

---

## 참고 문서

- [MySQL InnoDB Locking](https://dev.mysql.com/doc/refman/9.1/en/innodb-locking.html)
- [InnoDB Gap Locks](https://dev.mysql.com/doc/refman/9.1/en/innodb-locking.html#innodb-gap-locks)
- [Locks Set by Different SQL Statements](https://dev.mysql.com/doc/refman/9.1/en/innodb-locks-set.html)
- [InnoDB Transaction Isolation Levels](https://dev.mysql.com/doc/refman/9.1/en/innodb-transaction-isolation-levels.html)
- [MySQL performance_schema.data_locks](https://dev.mysql.com/doc/refman/8.0/en/performance-schema-data-locks-table.html)
