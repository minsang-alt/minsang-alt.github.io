---
title: 캐싱 전략 - Cache-Aside, Spring Cache, Caffeine, Redis
date: 2026-03-15 14:00:00 +/-0800
categories: [Tech Notes, Spring]
tags: [spring, cache, caffeine, redis, ttl]
---

## 캐싱 패턴 3가지

### Cache-Aside (Lazy Loading)

가장 일반적인 패턴입니다. **애플리케이션이 직접** 캐시를 관리합니다.

```
읽기 흐름:
1. 캐시에서 조회 → 있으면 반환 (Cache Hit)
2. 없으면 DB/API에서 조회 (Cache Miss)
3. 결과를 캐시에 저장
4. 결과를 반환

클라이언트 → 애플리케이션 → 캐시 확인
                          ↓ (miss)
                      DB/외부 API 조회
                          ↓
                      캐시에 저장 → 반환
```

```java
// Cache-Aside 의사코드
public Drug findDrug(String name) {
    Drug cached = cache.get(name);    // 1. 캐시 조회
    if (cached != null) return cached; // 2. Hit → 바로 반환

    Drug drug = api.search(name);      // 3. Miss → 원본 조회
    cache.put(name, drug);             // 4. 캐시에 저장
    return drug;
}
```

**특징:**
- 실제로 요청된 데이터만 캐싱 → 메모리 효율적
- 첫 요청은 느림 (Cold Start)
- 캐시와 원본 데이터 사이에 **불일치 가능** (캐시에 저장된 후 원본이 바뀌면)

### Read-Through

Cache-Aside와 비슷하지만, **캐시가 직접** 원본 데이터를 조회합니다.

```
클라이언트 → 캐시 → (miss일 때) 캐시가 직접 DB/API 조회 → 저장 → 반환
```

```java
// Read-Through 의사코드
// 캐시 자체에 loader를 등록
Cache<String, Drug> cache = Caffeine.newBuilder()
    .build(name -> api.search(name));  // miss 시 캐시가 직접 호출

Drug drug = cache.get("타이레놀");  // 애플리케이션은 캐시만 호출하면 됨
```

**Cache-Aside와의 차이:**

| 구분 | Cache-Aside | Read-Through |
|------|------------|-------------|
| 원본 조회 주체 | 애플리케이션 | 캐시 |
| 코드 복잡도 | 조건 분기 필요 | 단순 (캐시만 호출) |
| 캐시 라이브러리 요구사항 | 없음 | loader 지원 필요 |

Spring의 `@Cacheable`은 사실상 **Read-Through**입니다. 캐시 miss 시 메서드가 자동으로 실행되고 결과가 캐시에 저장됩니다.

### Write-Through

**쓰기 시** 캐시와 원본 데이터를 **동시에 갱신**합니다.

```
쓰기 흐름:
클라이언트 → 캐시에 쓰기 → 캐시가 DB에도 쓰기 → 완료

읽기 흐름:
클라이언트 → 캐시에서 읽기 → 항상 최신 데이터
```

**특징:**
- 캐시와 원본이 항상 일치 → 일관성 높음
- 쓰기가 느림 (캐시 + DB 두 번 쓰기)
- 읽기가 많고 쓰기가 적은 데이터에 적합

### 약 검색 서비스에는 어떤 패턴이 적합한가?

약 정보는 **읽기만 있고 쓰기가 없는** 데이터입니다 (공공 API에서 가져올 뿐, 우리가 수정하지 않음). 따라서 **Cache-Aside (또는 Read-Through)**가 적합합니다. Write-Through는 쓰기가 있는 데이터에만 의미가 있습니다.

## Spring Cache Abstraction

### @EnableCaching

캐싱 기능을 활성화하는 어노테이션입니다.

```java
@Configuration
@EnableCaching
public class CacheConfig {
    @Bean
    public CacheManager cacheManager() {
        return new CaffeineCacheManager("drug-search", "drug-detail");
    }
}
```

### @Cacheable — 결과를 캐시에 저장

```java
@Cacheable(value = "drug-search", key = "#itemName + '_' + #page + '_' + #size")
public List<DrugSearchResponse> searchByName(String itemName, int page, int size) {
    // 첫 호출: 이 메서드가 실행되고 결과가 캐시에 저장됨
    // 두 번째 호출: 이 메서드가 실행되지 않고 캐시에서 바로 반환
    return callExternalApi(itemName, page, size);
}
```

동작 흐름:

```
1번째 호출: searchByName("타이레놀", 1, 10)
  → 캐시 key "타이레놀_1_10" 조회 → miss
  → 메서드 실행 → 외부 API 호출 → 결과 반환 + 캐시에 저장
  → 응답 시간: ~2000ms

2번째 호출: searchByName("타이레놀", 1, 10)
  → 캐시 key "타이레놀_1_10" 조회 → hit!
  → 메서드 실행 안 함 → 캐시에서 바로 반환
  → 응답 시간: ~1ms
```

### @CacheEvict — 캐시 삭제

```java
// 특정 키의 캐시를 삭제
@CacheEvict(value = "drug-search", key = "#itemName + '_' + #page + '_' + #size")
public void evictSearchCache(String itemName, int page, int size) { }

// 캐시 전체를 삭제
@CacheEvict(value = "drug-search", allEntries = true)
public void evictAllSearchCache() { }
```

관리자 API에서 "캐시 초기화" 기능을 제공할 때 사용합니다.

### @CachePut — 캐시를 강제로 갱신

```java
@CachePut(value = "drug-search", key = "#itemName + '_' + #page + '_' + #size")
public List<DrugSearchResponse> refreshSearch(String itemName, int page, int size) {
    // @Cacheable과 달리, 항상 메서드를 실행하고 결과를 캐시에 덮어쓴다
    return callExternalApi(itemName, page, size);
}
```

`@Cacheable`은 캐시가 있으면 메서드를 실행하지 않지만, `@CachePut`은 **항상 실행**하고 결과를 캐시에 갱신합니다.

### 비교 정리

| 어노테이션 | 캐시 hit 시 | 캐시 miss 시 | 용도 |
|-----------|-----------|------------|------|
| `@Cacheable` | 메서드 실행 안 함, 캐시 반환 | 메서드 실행, 결과 캐싱 | 읽기 캐싱 |
| `@CacheEvict` | 캐시 삭제 | 캐시 삭제 | 캐시 무효화 |
| `@CachePut` | 메서드 실행, 캐시 덮어쓰기 | 메서드 실행, 결과 캐싱 | 캐시 강제 갱신 |

## Caffeine Cache

### 특징

Caffeine은 Java 기반의 **로컬 캐시** 라이브러리입니다. Google Guava Cache의 후속작으로, 현재 Java 생태계에서 가장 빠른 로컬 캐시입니다.

```java
Cache<String, List<Drug>> cache = Caffeine.newBuilder()
    .maximumSize(1000)                   // 최대 1000건
    .expireAfterWrite(24, TimeUnit.HOURS) // 쓰기 후 24시간 만료
    .recordStats()                        // 히트율 통계 수집
    .build();
```

### 핵심 설정

| 설정 | 의미 | 예시 |
|------|------|------|
| `maximumSize` | 캐시에 저장할 최대 항목 수 | 1000 |
| `expireAfterWrite` | 쓰기 후 만료 시간 (TTL) | 24시간 |
| `expireAfterAccess` | 마지막 접근 후 만료 시간 | 1시간 |
| `refreshAfterWrite` | 쓰기 후 자동 갱신 시간 | 12시간 |
| `recordStats` | 히트율 통계 수집 활성화 | - |

### expireAfterWrite vs expireAfterAccess

```
expireAfterWrite(1h):
  PUT "타이레놀" ──── 1시간 후 만료 (사용 여부 무관)
                     ↑ 중간에 아무리 GET해도 만료 시간 안 바뀜

expireAfterAccess(1h):
  PUT "타이레놀" ── GET ── GET ── GET ── 1시간 후 만료
                   ↑      ↑      ↑
              접근할 때마다 만료 시간 리셋
```

약 검색처럼 **데이터 신선도가 중요한 경우** `expireAfterWrite`가 적합합니다. `expireAfterAccess`는 인기 데이터가 영원히 갱신되지 않을 수 있습니다.

### Caffeine의 eviction 정책: Window TinyLFU

캐시가 가득 차면 어떤 항목을 제거할지 결정해야 합니다.

- **LRU** (Least Recently Used): 가장 오래 전에 사용된 항목 제거 → 최근 한 번만 접근된 항목이 자주 접근되는 항목을 밀어낼 수 있음
- **LFU** (Least Frequently Used): 가장 적게 사용된 항목 제거 → 과거에 많이 쓰이다 안 쓰이는 항목이 남을 수 있음
- **Window TinyLFU** (Caffeine): LRU와 LFU의 장점을 결합. 최근 접근 빈도와 전체 접근 빈도를 함께 고려하여 **가장 높은 히트율**을 달성

Caffeine이 "가장 빠른 로컬 캐시"라고 불리는 이유 중 하나입니다.

### Spring Boot에서 Caffeine 설정

```java
@Configuration
@EnableCaching
public class CacheConfig {

    @Bean
    public CacheManager cacheManager() {
        CaffeineCacheManager manager = new CaffeineCacheManager();
        manager.setCaffeine(Caffeine.newBuilder()
            .maximumSize(1000)
            .expireAfterWrite(24, TimeUnit.HOURS)
            .recordStats());
        return manager;
    }
}
```

캐시별로 다른 설정을 적용하려면 `SimpleCacheManager`를 사용합니다.

```java
@Bean
public CacheManager cacheManager() {
    SimpleCacheManager manager = new SimpleCacheManager();
    manager.setCaches(List.of(
        buildCache("drug-search", 1000, 24),    // 검색 결과: 1000건, 24시간
        buildCache("drug-detail", 5000, 24)     // 상세 정보: 5000건, 24시간
    ));
    return manager;
}

private CaffeineCache buildCache(String name, int maxSize, int ttlHours) {
    return new CaffeineCache(name, Caffeine.newBuilder()
        .maximumSize(maxSize)
        .expireAfterWrite(ttlHours, TimeUnit.HOURS)
        .recordStats()
        .build());
}
```

## Redis

### Caffeine과의 핵심 차이

Caffeine은 **JVM 힙 메모리**에 데이터를 저장합니다. 서버가 2대이면 각 서버의 캐시가 독립적입니다.

```
[Caffeine - 로컬 캐시]
서버 A: { "타이레놀": [결과] }   ← 서버 A에서 캐싱
서버 B: { }                    ← 서버 B는 모름, 다시 API 호출 필요

[Redis - 분산 캐시]
서버 A ──┐
         ├── Redis: { "타이레놀": [결과] }   ← 둘 다 같은 캐시 사용
서버 B ──┘
```

### 비교

| 기준 | Caffeine (로컬) | Redis (분산) |
|------|----------------|-------------|
| 조회 속도 | **~ns** (같은 JVM 힙) | ~1ms (네트워크 왕복) |
| 서버 간 공유 | 불가 (각 서버 독립) | **가능** (중앙 저장소) |
| 데이터 일관성 | 서버마다 다를 수 있음 | **항상 일관** |
| 서버 재시작 시 | 캐시 소멸 | **유지** |
| 인프라 비용 | 0원 | Redis 서버 필요 |
| 직렬화 | 불필요 (객체 참조) | **필요** (JSON/바이트 변환) |
| 적합한 상황 | 단일 서버, 읽기 위주 | 다중 서버, 일관성 필요 |

### 단일 서버라면 Caffeine이 정답

현재 프로젝트가 단일 서버라면 Caffeine만으로 충분합니다.

- 네트워크 지연 없음 (ns vs ms)
- 직렬화/역직렬화 비용 없음
- Redis 인프라 불필요
- 서버 재시작 시 캐시 소멸 → 약 검색은 분기별 갱신이니 문제없음

서버가 2대 이상으로 늘어나면 Redis 도입을 검토합니다.

## TTL (Time To Live)과 캐시 히트율

### TTL이란

캐시에 저장된 데이터가 **유효한 시간**입니다. TTL이 지나면 캐시에서 자동 삭제됩니다.

```
TTL = 24시간
PUT "타이레놀" (10:00) → 다음 날 10:00에 만료
                        → 10:01에 조회하면 Cache Miss → 외부 API 재호출 → 새로운 데이터로 캐싱
```

### TTL을 어떻게 결정하는가

| 고려 요소 | TTL 길게 | TTL 짧게 |
|----------|---------|---------|
| 데이터 신선도 | 오래된 데이터 가능성 | 항상 최신 |
| 캐시 히트율 | 높음 (오래 유지) | 낮음 (자주 만료) |
| 외부 API 호출 수 | 적음 | 많음 |
| 메모리 사용 | 많음 (오래 유지) | 적음 |

약 정보는 **분기별 업데이트**이므로 TTL 24시간이면 충분합니다. 만료 후 자연스럽게 최신 데이터로 갱신됩니다.

### 캐시 히트율 (Hit Rate)

```
히트율 = Cache Hit 수 / 전체 요청 수 × 100

예: 100번 요청 중 85번이 캐시에서 반환 → 히트율 85%
```

히트율이 높을수록 외부 API 호출이 줄고 응답이 빨라집니다. Caffeine의 `recordStats()`로 히트율을 모니터링할 수 있습니다.

```java
CaffeineCache cache = (CaffeineCache) cacheManager.getCache("drug-search");
CacheStats stats = cache.getNativeCache().stats();

log.info("히트율: {}%", stats.hitRate() * 100);
log.info("히트 수: {}, 미스 수: {}", stats.hitCount(), stats.missCount());
```

## 캐시 스탬피드 (Cache Stampede)

### 문제 상황

TTL이 만료되는 순간, **동시에 많은 요청**이 같은 데이터를 조회하면

```
TTL 만료! "타이레놀" 캐시 사라짐

요청 1 → 캐시 miss → 외부 API 호출 시작
요청 2 → 캐시 miss → 외부 API 호출 시작  (요청 1이 아직 안 끝남!)
요청 3 → 캐시 miss → 외부 API 호출 시작
...
요청 100 → 캐시 miss → 외부 API 호출 시작

→ 100번의 동일한 외부 API 호출이 동시에 발생! (Thundering Herd)
→ 외부 API rate limit 초과, 서버 과부하
```

### 해결 방법

**1. Caffeine의 refreshAfterWrite**

```java
Caffeine.newBuilder()
    .expireAfterWrite(24, TimeUnit.HOURS)    // 24시간 후 만료
    .refreshAfterWrite(12, TimeUnit.HOURS)   // 12시간 후 백그라운드 갱신
    .build(key -> loadFromApi(key));          // CacheLoader 필요
```

12시간이 지나면 **다음 접근 시** 하나의 스레드만 백그라운드에서 데이터를 갱신하고, 나머지 요청은 **기존 캐시 데이터를 반환**합니다. 24시간이 지나면 완전히 만료됩니다.

```
시간 흐름:
0h    → 캐싱됨
12h   → refreshAfterWrite 도달
12h+1 → 요청 A: 기존 캐시 반환 + 백그라운드에서 갱신 시작
12h+1 → 요청 B: 기존 캐시 반환 (갱신 중복 없음!)
12h+2 → 갱신 완료, 새 데이터로 교체
24h   → expireAfterWrite 도달, 완전 만료
```

**2. 분산 락 (Redis 환경)**

Redis를 사용한다면, 캐시 miss 시 **하나의 요청만 원본을 조회**하고 나머지는 대기합니다.

```java
if (cache.get(key) == null) {
    if (redis.tryLock(key + ":lock", 5, SECONDS)) {
        try {
            result = loadFromApi(key);
            cache.put(key, result);
        } finally {
            redis.unlock(key + ":lock");
        }
    } else {
        // 다른 스레드가 조회 중, 잠시 대기 후 캐시에서 다시 조회
        Thread.sleep(100);
        return cache.get(key);
    }
}
```

현재 단일 서버 + Caffeine 구성이라면 `refreshAfterWrite`로 충분합니다.

## Negative Caching

### 문제

존재하지 않는 약을 검색하면?

```
검색: "asdfghjk" (존재하지 않는 약)
  → 캐시 miss → 외부 API 호출 → 빈 결과 반환
  → 캐시에 저장하지 않음!

같은 검색 반복:
  → 또 캐시 miss → 또 외부 API 호출 → 또 빈 결과
  → 무의미한 API 호출 반복
```

악의적인 사용자가 존재하지 않는 검색어를 반복 호출하면, 캐시를 우회하여 외부 API에 직접 부하를 줄 수 있습니다.

### 해결: 빈 결과도 캐싱

```java
@Cacheable(value = "drug-search", key = "#itemName + '_' + #page + '_' + #size")
public List<DrugSearchResponse> searchByName(String itemName, int page, int size) {
    List<DrugSearchResponse> results = callExternalApi(itemName, page, size);
    // 빈 리스트도 정상적으로 반환 → @Cacheable이 빈 리스트도 캐싱함
    return results;
}
```

Spring의 `@Cacheable`은 기본적으로 **null이 아닌 모든 반환값을 캐싱**합니다. 빈 리스트(`Collections.emptyList()`)도 캐싱됩니다. 다만 null을 반환하면 캐싱되지 않으므로, 결과가 없을 때 빈 리스트를 반환하는 것이 중요합니다.

### 주의: Negative Cache의 TTL

빈 결과의 TTL을 정상 결과와 같게 잡으면, 나중에 해당 약이 추가되어도 24시간 동안 검색되지 않습니다. 빈 결과는 **짧은 TTL** (예: 1시간)을 적용하는 것이 이상적이지만, 캐시 설정이 복잡해집니다.

단순하게 가려면 동일한 TTL을 적용하되, 관리자 API로 특정 캐시를 수동 초기화할 수 있게 만드는 것이 실용적입니다.

## 참고 자료

- [Spring Cache Abstraction 공식 문서](https://docs.spring.io/spring-framework/reference/integration/cache.html)
- [Caffeine GitHub - Wiki](https://github.com/ben-manes/caffeine/wiki)
- [Redis 공식 문서](https://redis.io/docs/)
