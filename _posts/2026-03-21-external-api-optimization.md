---
title: 외부 공공 API 응답 시간 13초 → 4ms로 개선한 과정
date: 2026-03-21 14:00:00 +/-0800
categories: [Project Insights, Spring]
tags: [spring, performance, caffeine, resilience4j, virtual-thread, circuit-breaker]
---

약 검색 API의 응답 시간을 13초에서 4ms로 개선한 과정을 정리한다. 측정 → 타임아웃 → 병렬화 → 캐싱 → 재시도 → 서킷브레이커, 6단계에 걸쳐 외부 API 의존성을 최적화했다.

---

## 배경

사이드 프로젝트에서 공공 데이터 포털(data.go.kr)의 약 검색 API 3종을 호출하는 기능을 만들었다. 문제는 **한 번의 검색에 외부 API를 최소 3번, 최대 13번 호출**한다는 것이었다.

```
searchByName("타이레놀")
├── searchEasyDrug()           ← 공공 API 1회
├── searchPillIdentification() ← 공공 API 1회
└── enrichWithPermitDetail()   ← pill 결과 N건 × 공공 API 각 1회
```

체감상 느렸지만, 정확히 어디가 얼마나 느린지는 몰랐다.

---

## 1단계: 측정 — 감이 아니라 숫자로

### StopWatch로 구간별 측정

`Spring StopWatch`로 각 단계의 소요 시간을 찍었다.

```
StopWatch 'DrugSearch-타이레놀': 5.460s
  searchEasyDrug:           0.588s   11%
  searchPillIdentification: 0.986s   18%
  enrichWithPermitDetail:   3.886s   71%   ← 여기가 병목
```

`enrichWithPermitDetail`이 전체의 **71~86%**를 차지했다. N건의 상세 정보를 **순차적으로** 호출하는 N+1 패턴이 원인이었다.

### k6 부하 테스트로 Baseline 확보

```
k6 결과 (1 VU, 1분간, 4개 검색어 랜덤):
  p50: 13,148ms
  p95: 16,201ms
  TPS: 0.086 req/s  ← 1분에 6건 처리
```

**단일 사용자도 1분에 6건밖에 처리하지 못했다.**

---

## 2단계: 타임아웃 — 무한 대기 차단

측정하면서 발견한 가장 위험한 문제는 `new RestTemplate()`에 **타임아웃이 없다**는 것이었다. 공공 API가 응답하지 않으면 톰캣 스레드가 무한 대기한다.

### 변경

- `RestTemplateConfig`에서 용도별 빈 분리 (약검색용, AI용, 결제용)
- Connect Timeout 3초, Read Timeout 5초 설정
- Apache HttpClient 기반으로 커넥션 풀 관리

```java
@Bean("drugApiRestTemplate")
public RestTemplate drugApiRestTemplate() {
    HttpComponentsClientHttpRequestFactory factory =
        new HttpComponentsClientHttpRequestFactory();
    factory.setConnectTimeout(Duration.ofSeconds(3));
    factory.setReadTimeout(Duration.ofSeconds(5));
    return new RestTemplate(factory);
}
```

### 효과

외부 API 장애 시 무한 대기 → **최대 10초로 제한**. 직접적인 성능 개선은 아니지만, 이후 모든 최적화의 전제 조건이 됐다.

---

## 3단계: 병렬화 — 순차를 동시로

### 독립적인 호출 식별

`searchEasyDrug`과 `searchPillIdentification`은 서로 독립적이다. `enrichWithPermitDetail` 내부의 각 `fetchPermitDetail`도 서로 독립적이다. 순차적으로 호출할 이유가 없었다.

### CompletableFuture + Virtual Thread

```java
private final ExecutorService executor =
    Executors.newVirtualThreadPerTaskExecutor();

// 두 검색 API 병렬 호출
CompletableFuture<List<DrugSearchResponse>> easyFuture =
    CompletableFuture.supplyAsync(
        () -> searchEasyDrug(itemName, page, size), executor);
CompletableFuture<List<DrugSearchResponse>> pillFuture =
    CompletableFuture.supplyAsync(
        () -> searchPillIdentification(itemName, page, size), executor);

CompletableFuture.allOf(easyFuture, pillFuture).join();
```

N+1 병렬화에는 `Semaphore(5)`로 동시 호출 수를 제한했다. 공공 API의 rate limit을 보호하면서도 병렬성을 확보하는 조합이다.

### Virtual Thread를 선택한 이유

4가지 방식(ForkJoinPool, Virtual Thread, ThreadPoolTaskExecutor, @Async)을 모두 구현하고 측정했다. I/O 대기가 지배적인 이 워크로드에서는 Virtual Thread가 설정 없이 최적의 성능을 냈다.

| 방식 | 평균 응답 시간 | 스레드 수 |
|------|-------------|----------|
| Before (순차) | 3,417ms | 26 |
| Virtual Thread | **1,884ms** | 29 |
| ThreadPoolTaskExecutor | 2,047ms | 35 |

ThreadPoolTaskExecutor는 비슷한 성능에 core/max/queue 튜닝이 필요하고, `RejectedExecutionException` 위험이 있다. Virtual Thread는 한 줄로 끝난다.

### 효과

응답 시간 3,417ms → **1,884ms (-45%)**. 하지만 같은 검색어를 반복해도 매번 외부 API를 호출하는 문제는 여전했다.

---

## 4단계: 캐싱 — 가장 빠른 API 호출은 호출하지 않는 것

약 검색 데이터는 분기별로 업데이트된다. 같은 검색어로 100번 요청해도 결과는 동일한데, 매번 외부 API를 호출하고 있었다.

### Caffeine Cache 도입

```java
@Configuration
@EnableCaching
public class CacheConfig {
    @Bean
    public CacheManager cacheManager() {
        SimpleCacheManager manager = new SimpleCacheManager();
        manager.setCaches(List.of(
            buildCache("drug-search", 24, 1000),
            buildCache("drug-detail", 24, 5000)
        ));
        return manager;
    }

    private CaffeineCache buildCache(String name, int ttlHours, int maxSize) {
        return new CaffeineCache(name, Caffeine.newBuilder()
            .expireAfterWrite(Duration.ofHours(ttlHours))
            .maximumSize(maxSize)
            .recordStats()
            .build());
    }
}
```

### 캐시 키 정규화

"타이레놀"과 " 타이레놀 "이 같은 캐시 키로 매핑되도록 정규화했다.

```java
public static String normalizeKey(String input) {
    if (input == null) return "";
    return input.strip().toLowerCase().replaceAll("\\s+", "");
}

@Cacheable(value = "drug-search",
    key = "T(...).normalizeKey(#itemName) + '_' + #page + '_' + #size")
public List<DrugSearchResponse> searchByName(...) { ... }
```

### 왜 Redis가 아니라 Caffeine인가

| 기준 | Caffeine | Redis |
|------|----------|-------|
| 조회 지연 | ~ns (힙 내) | ~ms (네트워크) |
| 인프라 비용 | 0원 | ElastiCache 월 ~$15 |
| 서버 간 일관성 | 불일치 가능 | 일관 |

현재 단일 서버이고, 약 데이터는 분기별 업데이트라 서버 간 불일치 위험이 낮다. 인프라 비용 0원으로 충분한 효과를 얻을 수 있었다.

### 효과

| 지표 | Before | After | 개선 |
|------|--------|-------|------|
| 캐시 히트 응답 시간 | ~2,100ms | **4ms** | **500배** |
| TPS (1 VU 기준) | 0.086 | **9.47** | **110배** |
| 30초간 처리 건수 (1 VU) | ~3건 | **285건** | 95배 |

---

## 5단계: 재시도 — 일시적 장애 복구

캐시 미스 시에는 여전히 외부 API를 호출한다. 공공 API가 순간적으로 과부하 상태라면, 한 번 더 시도하면 성공할 수 있다.

### Resilience4j Retry + 지수 백오프

```yaml
resilience4j:
  retry:
    instances:
      drugApi:
        max-attempts: 3
        wait-duration: 500ms
        enable-exponential-backoff: true
        exponential-backoff-multiplier: 2
        retry-exceptions:
          - org.springframework.web.client.ResourceAccessException
          - org.springframework.web.client.HttpServerErrorException
        ignore-exceptions:
          - org.springframework.web.client.HttpClientErrorException
```

### DrugApiClient 분리

`@Retry`는 Spring AOP 기반이라 private 메서드에 적용할 수 없다. 외부 API 호출을 `DrugApiClient`로 분리하여 해결했다.

```java
@Component
public class DrugApiClient {
    @Retry(name = "drugApi", fallbackMethod = "searchEasyDrugFallback")
    public String searchEasyDrug(String itemName, int page, int size) {
        return restTemplate.getForObject(uri, String.class);
    }
}
```

### 재시도 대상 구분

| 예외 | 재시도 | 이유 |
|------|--------|------|
| 5xx, 타임아웃 | O | 일시적 장애, 재시도하면 복구 가능 |
| 4xx (400, 401, 404) | X | 요청 자체가 잘못됨, 재시도해도 동일 |

### 효과 (WireMock 측정)

| 시나리오 | 응답 시간 | API 호출 수 |
|---------|----------|------------|
| 정상 | 4ms | 1회 |
| 1회 실패 후 성공 | 511ms | 2회 |
| 2회 실패 후 성공 | 1,019ms | 3회 |
| 3회 모두 실패 | 1,027ms | 3회 (fallback) |
| 4xx | 5ms | 1회 (즉시 실패) |

---

## 6단계: 서킷브레이커 — 장기 장애 차단

Retry는 일시적 장애에 효과적이다. 하지만 공공 API가 30분간 완전히 죽으면, **매 요청마다 3회 재시도 × 백오프 = 요청당 ~1초 소모**. 이미 죽은 서비스에 계속 요청을 보내는 것은 자원 낭비다.

### Resilience4j CircuitBreaker

```yaml
resilience4j:
  circuitbreaker:
    instances:
      drugApi:
        sliding-window-type: COUNT_BASED
        sliding-window-size: 10
        failure-rate-threshold: 50
        wait-duration-in-open-state: 60s
        permitted-number-of-calls-in-half-open-state: 3
```

10회 중 50% 이상 실패하면 서킷이 열리고, 이후 요청은 **외부 호출 없이 즉시 fallback을 반환**한다.

### @Retry + @CircuitBreaker 조합

```java
@Retry(name = "drugApi", fallbackMethod = "searchEasyDrugFallback")
@CircuitBreaker(name = "drugApi")
public String searchEasyDrug(...) { ... }
```

주의할 점: `@CircuitBreaker`에 `fallbackMethod`를 넣으면 내부에서 바로 fallback을 호출하여 Retry가 예외를 받지 못한다. `fallbackMethod`는 바깥 데코레이터인 `@Retry`에 두어야 한다.

동작 흐름:
- **CB Closed**: 실패 → CB가 기록 + 예외 throw → Retry 재시도
- **CB Open**: `CallNotPermittedException` → Retry가 받지만 재시도 대상 아님 → 즉시 fallback (**2ms**)

### 3중 방어 구조

```
요청 → @Cacheable (캐시 HIT? → 즉시 반환, 4ms)
         ↓ 캐시 MISS
      @Retry (일시적 실패? → 재시도)
         ↓ 반복 실패
      @CircuitBreaker (장기 장애? → 즉시 차단, 2ms)
         ↓ fallback
      빈 결과 반환
```

캐시가 1차 방어로 대부분의 요청을 처리하고, 캐시 미스 시에만 Retry와 CircuitBreaker가 동작한다.

---

## 전체 개선 결과

| 지표 | Before (1단계) | After (6단계) | 개선 |
|------|---------------|-------------|------|
| 응답 시간 (캐시 히트) | 13,148ms | **4ms** | **3,000배** |
| 응답 시간 (캐시 미스) | 13,148ms | **1,884ms** | **7배** |
| TPS (1 VU 기준) | 0.086 req/s | **9.47 req/s** | **110배** |
| 장애 시 응답 시간 | 무한 대기 | **2ms** (CB Open) | - |
| 장애 전파 | 전체 서비스 영향 | 약 검색만 fallback | - |

> TPS는 k6 단일 VU(순차 요청) 기준이다. VU를 늘리면 비례하여 증가하며, 캐시 히트 시 서버 자체는 수백 req/s 이상 처리 가능하다.

### 각 단계의 기여

| 단계 | 기법 | 핵심 효과 |
|------|------|----------|
| 1. 측정 | StopWatch + k6 | 병목 식별 (enrichWithPermitDetail 86%) |
| 2. 타임아웃 | RestTemplate 빈 분리 | 무한 대기 → 최대 10초 |
| 3. 병렬화 | Virtual Thread + Semaphore | 3,417ms → 1,884ms (-45%) |
| 4. 캐싱 | Caffeine Cache-Aside | 2,100ms → 4ms (500배) |
| 5. 재시도 | Resilience4j Retry + 지수 백오프 | 일시적 장애 자동 복구 |
| 6. 서킷브레이커 | Resilience4j CircuitBreaker | 장기 장애 시 즉시 차단 (2ms) |

---

## 배운 것

**측정 먼저.** 1단계에서 StopWatch를 찍지 않았으면 `enrichWithPermitDetail`이 병목인지 몰랐을 것이다. 병렬화(3단계)도 StopWatch 결과가 있었기 때문에 어디를 병렬화할지 판단할 수 있었다.

**단계적으로.** 캐시(4단계)부터 적용하고 싶었지만, 타임아웃(2단계)이 없으면 캐시 미스 시 무한 대기가 발생하고, 병렬화(3단계)가 없으면 캐시 미스의 응답 시간이 여전히 느리다. 순서가 중요했다.

**방어는 층으로.** Retry만으로는 장기 장애를 막을 수 없고, CircuitBreaker만으로는 일시적 장애를 복구할 수 없다. 캐시 → Retry → CircuitBreaker, 각각 다른 장애 시나리오에 대응하는 3중 방어가 완성됐을 때 안정성이 확보됐다.

---

## 참고 자료

- [Martin Fowler - Circuit Breaker](https://martinfowler.com/bliki/CircuitBreaker.html)
- [Resilience4j 공식 문서](https://resilience4j.readme.io/docs)
- [Java 21 Virtual Threads - JEP 444](https://openjdk.org/jeps/444)
- [Caffeine Cache Wiki](https://github.com/ben-manes/caffeine/wiki)
- [우아한형제들 기술블로그 - 서킷브레이커 적용기](https://techblog.woowahan.com/15694/)
- [AWS - Exponential Backoff And Jitter](https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/)
