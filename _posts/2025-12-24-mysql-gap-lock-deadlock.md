---
title: MySQL Gap Lock 데드락. 원인 분석과 해결 방법
date: 2025-12-24 10:19:10 +/-0800
categories: [MySQL]
tags: mysql
---

## TL;DR

- 서로 다른 트랜잭션이 Unique Key 인덱스의 **같은 갭**에 INSERT하면 데드락이 발생한다.
- Gap Lock끼리는 호환되지만, Insert Intention Lock과는 충돌한다.
- 해결책은 **분산 락으로 직렬화**하거나 **READ COMMITTED 격리 수준**을 사용하는 것이다.

---

## 문제 상황

운영 환경에서 다음과 같은 데드락이 발생했다.

```sql
-- TX A
INSERT INTO user_score
  (user_id, ...)
VALUES (765331, ...), (765332, ...);

-- TX B
INSERT INTO user_score
  (user_id, ...)
VALUES (765326, ...), (765327, ...), (765328, ...), (765329, ...), (765330, ...);
```

두 트랜잭션은 서로 다른 그룹의 점수를 계산한다. 삽입하는 `user_id`도 다르다. 그런데 데드락이 발생했다.

---

## 데드락 로그 분석

MySQL의 `SHOW ENGINE INNODB STATUS` 출력 중 핵심 부분을 살펴보자.

```
*** (1) HOLDS THE LOCK(S):
RECORD LOCKS space id 1151 page no 1740 n bits 776
index UK_user_id
lock_mode X locks gap before rec

*** (1) WAITING FOR THIS LOCK TO BE GRANTED:
lock_mode X locks gap before rec insert intention waiting
```

| 키워드 | 의미 |
|--------|------|
| `UK_xxxx...` | `user_id` Unique Key 인덱스 |
| `locks gap before rec` | 특정 레코드 직전의 갭에 락 |
| `insert intention waiting` | INSERT를 위한 락 대기 중 |

두 트랜잭션 모두 **같은 갭**에 Gap Lock을 보유하면서, **같은 갭**에 Insert Intention Lock을 요청하고 있다.

---

## Gap Lock이란?

InnoDB는 REPEATABLE READ 격리 수준에서 Phantom Read를 방지하기 위해 Gap Lock을 사용한다.

### Gap Lock의 범위

Gap Lock은 인덱스 레코드 **사이의 빈 공간**을 잠근다.

```
인덱스 상태:
┌───────┬───────┬───────────────────────┬───────┐
│  100  │  200  │        [GAP]          │  700  │
└───────┴───────┴───────────────────────┴───────┘
                ↑                       ↑
                301~699는 존재하지 않음
                이 구간 전체가 하나의 "갭"

INSERT 400 → 700 직전 갭에 락 필요
INSERT 500 → 700 직전 갭에 락 필요 (같은 갭!)
INSERT 600 → 700 직전 갭에 락 필요 (같은 갭!)
```

### 락 호환성

| | Gap Lock | Insert Intention Lock |
|---|:---:|:---:|
| **Gap Lock** | O | X |
| **Insert Intention Lock** | X | O |

Gap Lock끼리는 호환된다. 여러 트랜잭션이 같은 갭에 Gap Lock을 동시에 보유할 수 있다.

하지만 Gap Lock과 Insert Intention Lock은 충돌한다. Gap Lock을 보유한 상태에서 다른 트랜잭션의 Insert Intention Lock 요청을 막는다.

Gap Lock은 Primary Key뿐만 아니라 Secondary Index에도 동일하게 사용된다는 것도 기억해 두도록 하자.

---

## 데드락 발생 원리

### 인덱스 상태

```
UK 인덱스: user_id

... ── 765325 ══════════ [GAP] ══════════ 765333 ── ...
                ↑                    ↑
         TX B: 765326~330     TX A: 765331~332
                └────────────────────┘
                     같은 갭!
```

765326부터 765332까지의 ID는 아직 존재하지 않는다. 다음으로 존재하는 레코드는 765333이다. 따라서 모든 INSERT는 **765333 직전의 갭**을 대상으로 한다.

### 데드락 로그에서 갭 위치 확인하기

데드락 로그의 hex 값을 해석하면 정확히 어떤 레코드 직전의 갭인지 알 수 있다.

```
Record lock, heap no 698 PHYSICAL RECORD: n_fields 2; compact format; info bits 0
0: len 8; hex 80000000000bad95; asc         ;;
1: len 8; hex 800000000003c59b; asc         ;;
```

| 필드 | Hex 값 | 변환 | 의미 |
|------|--------|------|------|
| 0 | `80000000000bad95` | 765333 | UK 인덱스 값 (다음 레코드) |
| 1 | `800000000003c59b` | 247195 | PK 값 (해당 레코드의 id) |

**Hex 변환 방법**: `0x80000000000bad95`에서 최상위 비트(부호 비트)를 제거하면 `0x0bad95` = 765333

이 로그는 **765333 직전의 갭**에 Gap Lock이 걸려있음을 보여준다. TX A와 TX B 모두 이 갭에 INSERT하려고 했기 때문에 데드락이 발생했다.

### 시간 순서

```
T1: TX B - Gap Lock 획득 (765333 직전)
T2: TX A - Gap Lock 획득 (765333 직전) ← Gap Lock끼리 호환되므로 성공
T3: TX A - Insert Intention Lock 요청 → TX B의 Gap Lock과 충돌 → 대기
T4: TX B - Insert Intention Lock 요청 → TX A의 Gap Lock과 충돌 → 대기
T5: DEADLOCK!
```

```
     TX A                              TX B
      │                                 │
      ▼                                 ▼
┌───────────┐                    ┌───────────┐
│ Gap Lock  │◄─── 호환 OK ──────► │ Gap Lock  │
│  (보유)    │                    │  (보유)    │
└─────┬─────┘                    └─────┬─────┘
      │                                │
      │ Insert Intention               │ Insert Intention
      │ Lock 요청                       │ Lock 요청
      ▼                                ▼
┌───────────┐                    ┌───────────┐
│   대기     │◄───── 충돌! ──────► │   대기      │
└───────────┘                    └───────────┘
              💀 DEADLOCK
```

---

## 해결 방법

### 방법 1: 분산 락으로 직렬화

동시 실행 자체를 막는다.

```java
@Component
@RequiredArgsConstructor
public class CalculationPointService {

    private final RedissonClient redissonClient;

    public void saveCalculationPoints(List<Point> points) {
        RLock lock = redissonClient.getLock("calculation-point-lock");

        try {
            if (lock.tryLock(10, 60, TimeUnit.SECONDS)) {
                repository.saveAll(points);
            }
        } finally {
            if (lock.isHeldByCurrentThread()) {
                lock.unlock();
            }
        }
    }
}
```

**장점**: 데드락 완전 방지

**단점**: 처리량 감소

```
분산 락 없이 (병렬 처리):
┌─────────────────────────────────────────────────────────┐
│ 시간 →                                                  │
│                                                         │
│ TX A: ████████████                                      │
│ TX B:     ████████████                                  │
│ TX C:         ████████████                              │
│                                                         │
│ → 동시에 여러 트랜잭션 실행 (빠름)                        │
│ → 단, 같은 갭에서 데드락 가능                            │
└─────────────────────────────────────────────────────────┘

분산 락 적용 (직렬화):
┌─────────────────────────────────────────────────────────┐
│ 시간 →                                                  │
│                                                         │
│ TX A: ████████████                                      │
│ TX B:             ████████████                          │
│ TX C:                         ████████████              │
│                                                         │
│ → 한 번에 하나씩만 실행 (느림)                           │
│ → 데드락 없음                                           │
└─────────────────────────────────────────────────────────┘
```

트래픽이 많을 때 병목이 될 수 있다.

지금 상황에서는 그룹마다 계산 로직이 수행되는 시간이 매우 길기 때문에, 적합한 선택이 아니다. 

### 방법 2: READ COMMITTED 격리 수준

Gap Lock을 사용하지 않도록 격리 수준을 READ COMMITTED로 낮춘다.

```java
@Transactional(isolation = Isolation.READ_COMMITTED)
public void saveCalculationPoints(List<Point> points) {
    bulkRepository.saveAllInBulk(points);
}
```

**장점**: 동시성 유지. Gap Lock이 발생하지 않아 데드락 가능성이 낮아진다.

**단점**: Phantom Read가 발생할 수 있다.

### REPEATABLE READ vs READ COMMITTED

| 격리 수준 | INSERT 시 락 | Phantom Read |
|-----------|-------------|--------------|
| REPEATABLE READ | Gap Lock + Record Lock | 방지 |
| READ COMMITTED | Record Lock만 | 발생 가능 |

### Phantom Read가 문제되지 않는 경우

다음과 같은 패턴에서는 READ COMMITTED를 안전하게 사용할 수 있다.

```java
@Transactional(isolation = Isolation.READ_COMMITTED)
public void calculateScore(Long groupId) {
    // 1. 기존 데이터 삭제
    bulkRepository.deleteByGroupId(groupId);

    // 2. 새로 계산해서 INSERT
    List<Score> scores = calculate();
    bulkRepository.saveAllInBulk(scores);

    // 같은 데이터를 두 번 조회하지 않으므로
    // Phantom Read 영향 없음
}
```

### 방법 3: 현행 유지 + 재시도에 의존

데드락이 모니터링으로 감지되고 있고, 자주 발생하지는 않는다.

주로 하나의 고객사에서 2개 이상의 그룹을 거의 동시에 종료했을 때 발생한다.

데드락이 발생하면 MySQL은 하나의 트랜잭션을 롤백하고, 다른 트랜잭션은 커밋시킨다.

롤백된 트랜잭션은 카프카 컨슈머에서 실행되므로, 카프카가 자동으로 재시도한다.

```
메시지 처리 실패
    ↓
재시도 실패 시 RETRY 토픽으로 이동
    ↓
최종 실패 시 DLT(Dead Letter Topic)로 이동
```


---

## 최종 결정: 방법 2

최근 고객사에서 Open API를 활용해 여러 그룹을 동시에 종료하는 경우가 잦아졌다.

데드락 알림이 빈번해지고, 매번 점수가 제대로 계산됐는지 확인하는 절차가 번거로워졌다. 격리 수준을 낮추고 현황을 지켜보기로 했다.

```java
@Repository
public class UserScoreBulkRepository {
    @Transactional(isolation = Isolation.READ_COMMITTED)
    public void saveAllInBulk(List<UserScore> scores) {
        // bulk insert 로직
    }
}
```

---

## 정리

| 항목 | 내용 |
|------|------|
| **원인** | 같은 갭에 Gap Lock을 보유한 상태에서 Insert Intention Lock 상호 대기 |
| **조건** | REPEATABLE READ + Unique Key + 인접한 ID에 동시 INSERT |
| **해결책 1** | 분산 락으로 직렬화 |
| **해결책 2** | READ COMMITTED로 Gap Lock 제거 |
| **해결책 3** | 현행 유지 + 카프카 재시도에 의존 |

Gap Lock 데드락은 **인접한 ID**를 **동시에 INSERT**할 때 발생한다. AUTO_INCREMENT를 사용하는 테이블에서 벌크 INSERT가 많다면, 격리 수준을 READ COMMITTED로 낮추는 것을 검토하자.
