---
title: Spring 트랜잭션 전파와 동작 원리. @Transactional을 제대로 이해하기
date: 2025-03-12 10:19:10 +/-0800
categories: [Tech Notes, Spring]
tags: [spring, transaction, propagation, aop]
---

Spring에서 `@Transactional`이 어떻게 동작하는지, 전파 속성별 트레이드오프는 무엇인지, 실무에서 자주 발생하는 함정은 무엇인지를 정리한다. 이 글을 읽으면 트랜잭션 전파 설정을 상황에 맞게 선택할 수 있다.

---

## 왜 트랜잭션 전파를 이해해야 하는가?

`@Transactional`을 붙이면 트랜잭션이 적용된다는 것은 대부분 안다. 문제는 **서비스 A가 서비스 B를 호출할 때** 발생한다.

```java
@Service
public class OrderService {
    @Transactional
    public void createOrder() {
        orderRepository.save(order);
        paymentService.processPayment(order); // PaymentService도 @Transactional
    }
}
```

PaymentService에서 예외가 발생하면 OrderService의 저장도 롤백될까? 아니면 독립적으로 동작할까? 이 질문에 정확히 답하려면 트랜잭션 전파 속성을 이해해야 한다.

## @Transactional의 동작 원리: AOP 프록시

Spring은 `@Transactional`이 붙은 빈을 **프록시 객체**로 감싼다. 외부에서 메서드를 호출하면 프록시가 가로채서 트랜잭션을 시작하고, 메서드 실행 후 커밋 또는 롤백한다.

```
Client → Proxy(TransactionInterceptor) → 실제 Bean 메서드
         ↑ 트랜잭션 시작                    ↑ 비즈니스 로직 실행
         ↓ 커밋 또는 롤백
```

핵심은 **프록시를 통해 호출되어야** 트랜잭션이 적용된다는 점이다. 이 구조를 모르면 뒤에서 다룰 self-invocation 문제를 이해할 수 없다.

## 트랜잭션 전파 속성 7가지

### REQUIRED (기본값)

기존 트랜잭션이 있으면 참여하고, 없으면 새로 생성한다.

```java
@Transactional(propagation = Propagation.REQUIRED) // 기본값이므로 생략 가능
public void methodA() { ... }
```

- **장점:** 하나의 트랜잭션으로 묶여 데이터 일관성을 보장한다
- **단점:** 내부 메서드에서 예외가 발생하면 전체가 롤백된다. 예외를 catch해도 이미 rollback-only로 마킹되어 커밋할 수 없다

### REQUIRES_NEW

항상 새로운 트랜잭션을 생성한다. 기존 트랜잭션은 일시 보류한다.

```java
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void methodB() { ... }
```

- **장점:** 독립적인 커밋/롤백이 가능하다. methodB가 실패해도 methodA는 커밋할 수 있다
- **단점:** 커넥션을 2개 점유한다. 커넥션 풀 고갈 위험이 있다. 외부 트랜잭션과 데이터 정합성이 깨질 수 있다

### NOT_SUPPORTED

트랜잭션 없이 실행한다. 기존 트랜잭션이 있으면 보류한다.

```java
@Transactional(propagation = Propagation.NOT_SUPPORTED)
public void sendSlackNotification() { ... }
```

- **적합한 상황:** 외부 API 호출처럼 DB 트랜잭션이 불필요한 작업
- **주의:** 트랜잭션이 보류되는 동안 커넥션은 유지된다

### SUPPORTS

기존 트랜잭션이 있으면 참여하고, 없으면 트랜잭션 없이 실행한다.

### MANDATORY

기존 트랜잭션이 반드시 있어야 한다. 없으면 `IllegalTransactionStateException`을 던진다. 단독 실행을 방지할 때 유용하다.

### NESTED

기존 트랜잭션 안에서 세이브포인트를 생성한다. 내부에서 롤백해도 세이브포인트까지만 롤백된다.

- **주의:** JPA에서는 지원하지 않는다. JDBC 기반에서만 동작한다

### NEVER

트랜잭션이 존재하면 예외를 던진다. 트랜잭션 없이 실행되어야 하는 작업을 강제할 때 사용한다.

## 트레이드오프: REQUIRED vs REQUIRES_NEW

실무에서 가장 많이 고민하는 선택이다.

| 기준 | REQUIRED | REQUIRES_NEW |
|------|----------|--------------|
| 데이터 일관성 | 하나의 트랜잭션으로 보장 | 독립적이므로 불일치 가능 |
| 장애 격리 | 하나가 실패하면 전체 롤백 | 실패를 격리할 수 있음 |
| 커넥션 사용 | 1개 | 2개 (풀 고갈 위험) |
| 롤백 제어 | catch해도 롤백됨 | 독립 롤백 가능 |

**선택 기준:**
- 주문과 결제처럼 **반드시 함께 성공/실패해야 하는 작업** → `REQUIRED`
- 알림 발송처럼 **실패해도 핵심 로직에 영향을 주면 안 되는 작업** → `REQUIRES_NEW` 또는 `NOT_SUPPORTED`
- 로그 기록처럼 **DB 트랜잭션 자체가 불필요한 작업** → `NOT_SUPPORTED`

## 실무에서 빠지는 3가지 함정

### 1. rollback-only 마킹 문제

가장 흔한 장애 패턴이다. REQUIRED로 참여한 내부 메서드에서 예외가 발생하면, 외부에서 catch하더라도 이미 트랜잭션은 rollback-only로 마킹된다.

```java
@Transactional
public void outerMethod() {
    try {
        innerService.innerMethod(); // RuntimeException 발생 → rollback-only 마킹
    } catch (Exception e) {
        // 예외를 잡았지만 소용없다
        log.warn("내부 실패, 계속 진행");
    }
    repository.save(entity); // 저장은 되지만...
    // 메서드 종료 시 커밋 시도 → UnexpectedRollbackException 발생
}
```

**해결 방법:**
- 내부 메서드를 `REQUIRES_NEW`로 분리한다 → 독립 트랜잭션이므로 외부에 영향 없음
- 트랜잭션이 필요 없다면 `NOT_SUPPORTED`로 설정한다
- 내부 메서드에서 예외를 던지지 않도록 설계를 변경한다

### 2. self-invocation (내부 호출) 문제

같은 클래스 내에서 `@Transactional` 메서드를 호출하면 프록시를 거치지 않아 **트랜잭션이 적용되지 않는다.**

```java
@Service
public class UserService {
    public void register(User user) {
        saveUser(user); // ❌ 프록시를 거치지 않음 → 트랜잭션 미적용
    }

    @Transactional
    public void saveUser(User user) {
        userRepository.save(user);
    }
}
```

`register()`가 외부에서 프록시를 통해 호출되더라도, `saveUser()`는 `this.saveUser()`로 직접 호출되기 때문에 프록시를 우회한다.

**해결 방법:**
- 트랜잭션이 필요한 로직을 별도 클래스로 분리한다 (권장)
- `register()` 자체에 `@Transactional`을 붙인다
- `ApplicationContext`에서 자기 자신의 빈을 가져와 호출한다 (비권장)

### 3. checked exception은 롤백하지 않는다

Spring은 기본적으로 **RuntimeException과 Error만 롤백**한다. checked exception은 커밋된다.

```java
@Transactional
public void transfer() throws InsufficientBalanceException {
    accountRepository.withdraw(from, amount);
    if (balance < 0) {
        throw new InsufficientBalanceException(); // ❌ checked exception → 커밋됨
    }
}
```

**해결 방법:**

```java
@Transactional(rollbackFor = InsufficientBalanceException.class)
public void transfer() throws InsufficientBalanceException { ... }
```

`rollbackFor`로 명시하거나, 비즈니스 예외를 RuntimeException으로 설계하는 것이 일반적이다.

## 격리 수준 (Isolation Level)

Spring의 `@Transactional`에서 격리 수준도 설정할 수 있다. 다만 대부분의 경우 **DB의 기본 격리 수준을 그대로 사용**하는 것이 권장된다.

| 격리 수준 | Dirty Read | Non-Repeatable Read | Phantom Read |
|-----------|-----------|-------------------|-------------|
| READ_UNCOMMITTED | O | O | O |
| READ_COMMITTED | X | O | O |
| REPEATABLE_READ | X | X | O (InnoDB는 X) |
| SERIALIZABLE | X | X | X |

- MySQL InnoDB 기본값: `REPEATABLE_READ`
- PostgreSQL 기본값: `READ_COMMITTED`

**트레이드오프:** 격리 수준이 높을수록 데이터 일관성은 보장되지만, 동시성은 떨어진다. 대부분의 웹 애플리케이션에서는 DB 기본값으로 충분하다. Spring에서 `isolation`을 별도로 지정하는 경우는 금융 거래처럼 정합성이 극도로 중요한 상황에 한정한다.

```java
@Transactional(isolation = Isolation.SERIALIZABLE)
public void processPayment() { ... } // 금전 처리에서만 고려
```

## 정리

| 상황 | 추천 전파 속성 | 이유 |
|------|---------------|------|
| 일반적인 서비스 로직 | REQUIRED (기본값) | 하나의 트랜잭션으로 일관성 보장 |
| 실패해도 되는 부가 기능 | REQUIRES_NEW / NOT_SUPPORTED | 핵심 로직과 격리 |
| 외부 API 호출 | NOT_SUPPORTED | DB 트랜잭션 불필요, 롤백 영향 차단 |
| 반드시 기존 트랜잭션 필요 | MANDATORY | 단독 실행 방지 |
| 로그/감사 독립 저장 | REQUIRES_NEW | 핵심 롤백과 무관하게 기록 유지 |

핵심은 세 가지다:
1. REQUIRED는 catch해도 rollback-only가 풀리지 않는다
2. 같은 클래스 내부 호출은 프록시를 우회한다
3. checked exception은 기본적으로 롤백하지 않는다

이 세 가지를 모르면 "왜 롤백이 됐지?", "왜 트랜잭션이 안 걸리지?"라는 질문에 답할 수 없다.

## 참고 자료
- [Spring 공식 문서 - Using @Transactional](https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative/annotations.html)
- [우아한형제들 기술블로그 - 응? 이게 왜 롤백되는거지?](https://woowabros.github.io/experience/2019/01/29/exception-in-transaction.html)
- [Hyperconnect Tech Blog - Spring Transactional Rollback Deep Dive](https://hyperconnect.github.io/2025/02/10/spring-transactional-rollback.html)
- [Spring Transaction Management: @Transactional In-Depth](https://www.marcobehler.com/guides/spring-transaction-management-transactional-in-depth)
