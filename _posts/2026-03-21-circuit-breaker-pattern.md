---
title: Circuit Breaker 패턴 - 연쇄 장애를 막는 안전장치
date: 2026-03-21 10:19:10 +/-0800
categories: [Tech Notes, Spring]
tags: [resilience4j, circuit-breaker, cascading-failure, fault-tolerance]
---

외부 API가 죽었을 때, 내 서비스까지 죽지 않으려면 어떻게 해야 할까? Circuit Breaker 패턴의 동작 원리, 3가지 상태 전이, Sliding Window 방식별 트레이드오프, 그리고 Resilience4j로의 구현까지 정리한다.

---

## Cascading Failure (연쇄 장애)

Circuit Breaker를 이해하려면 먼저 **왜 필요한지**를 알아야 한다.

```
서비스 A → 서비스 B → 서비스 C (장애 발생)
```

서비스 C가 응답하지 않으면
1. 서비스 B는 C의 응답을 기다리며 스레드를 점유한다
2. B에 요청이 쌓이면서 B의 스레드 풀이 고갈된다
3. B도 응답하지 못하게 되고, A도 같은 이유로 죽는다

**하나의 장애가 호출 체인을 따라 전파되어 전체 시스템이 무너지는 현상**이 Cascading Failure(연쇄 장애)다.

### 실제 예시

공공 API가 30분간 다운된 상황을 가정해보자.

Retry만 적용된 서비스에서는
- 매 요청마다 3회 재시도 x 지수 백오프 = 요청당 ~1.5초 소모
- 100명 동시 요청 → 100개 스레드가 각각 1.5초간 점유
- 톰캣 기본 스레드 풀(200개)의 절반이 "실패할 것을 알면서도" 대기
- 약 검색뿐 아니라 로그인, 결제 등 다른 기능까지 스레드 부족으로 영향

**이미 죽은 서비스에 계속 요청을 보내는 것은 자원 낭비다.** Circuit Breaker는 이 문제를 해결한다.

---

## Circuit Breaker 패턴

Martin Fowler가 정리한 이 패턴은 전기 회로의 차단기에서 이름을 따왔다. 과부하가 걸리면 회로를 끊어서 화재를 방지하듯, 소프트웨어에서도 장애가 감지되면 호출을 차단한다.

### 3가지 상태

<style>
  @keyframes pulse { 0%,100%{opacity:1} 50%{opacity:.55} }
  .active-state rect { animation: pulse 2s ease-in-out infinite; }
  .state-btn { cursor:pointer; padding:8px 16px; border-radius:var(--border-radius-md); border:0.5px solid var(--color-border-secondary); background:var(--color-background-secondary); color:var(--color-text-secondary); font-size:13px; transition:all .2s; }
  .state-btn.active { background:var(--color-background-info); color:var(--color-text-info); border-color:var(--color-border-info); font-weight:500; }
  .info-panel { margin-top:16px; padding:16px 20px; border-radius:var(--border-radius-lg); background:var(--color-background-secondary); font-size:13px; line-height:1.7; color:var(--color-text-secondary); min-height:90px; transition:opacity .3s; }
  .info-title { font-weight:500; font-size:14px; color:var(--color-text-primary); margin-bottom:6px; }
</style>

<div style="display:flex; gap:8px; justify-content:center; margin-bottom:4px;">
  <button class="state-btn active" onclick="setState('closed')" id="btn-closed">Closed</button>
  <button class="state-btn" onclick="setState('open')" id="btn-open">Open</button>
  <button class="state-btn" onclick="setState('half')" id="btn-half">Half-open</button>
</div>

<svg width="100%" viewBox="0 0 680 340">
  <defs>
    <marker id="arrow" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse">
      <path d="M2 1L8 5L2 9" fill="none" stroke="context-stroke" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"/>
    </marker>
  </defs>

  <!-- Closed state -->
  <g id="g-closed" class="active-state" style="cursor:pointer" onclick="setState('closed')">
    <rect id="r-closed" x="40" y="60" width="160" height="64" rx="8" fill="#E1F5EE" stroke="#0F6E56" stroke-width="1"/>
    <text class="th" x="120" y="84" text-anchor="middle" dominant-baseline="central" fill="#085041">Closed</text>
    <text class="ts" x="120" y="104" text-anchor="middle" dominant-baseline="central" fill="#0F6E56">요청 통과</text>
  </g>

  <!-- Open state -->
  <g id="g-open" style="cursor:pointer" onclick="setState('open')">
    <rect id="r-open" x="460" y="60" width="160" height="64" rx="8" fill="#FCEBEB" stroke="#A32D2D" stroke-width="0.5"/>
    <text class="th" x="540" y="84" text-anchor="middle" dominant-baseline="central" fill="#791F1F">Open</text>
    <text class="ts" x="540" y="104" text-anchor="middle" dominant-baseline="central" fill="#A32D2D">요청 차단</text>
  </g>

  <!-- Half-open state -->
  <g id="g-half" style="cursor:pointer" onclick="setState('half')">
    <rect id="r-half" x="250" y="230" width="160" height="64" rx="8" fill="#FAEEDA" stroke="#854F0B" stroke-width="0.5"/>
    <text class="th" x="330" y="254" text-anchor="middle" dominant-baseline="central" fill="#633806">Half-open</text>
    <text class="ts" x="330" y="274" text-anchor="middle" dominant-baseline="central" fill="#854F0B">시험 요청</text>
  </g>

  <!-- Arrow: Closed → Open -->
  <path id="a-co" d="M200 80 L460 80" fill="none" stroke="#A32D2D" stroke-width="0.5" marker-end="url(#arrow)" opacity="0.4"/>
  <text id="l-co" class="ts" x="330" y="72" text-anchor="middle" fill="var(--color-text-secondary)" opacity="0.5">실패 임계치 초과</text>

  <!-- Arrow: Open → Half-open -->
  <path id="a-oh" d="M540 124 L540 262 L410 262" fill="none" stroke="#854F0B" stroke-width="0.5" marker-end="url(#arrow)" opacity="0.4"/>
  <text id="l-oh" class="ts" x="548" y="200" text-anchor="start" fill="var(--color-text-secondary)" opacity="0.5">타임아웃 만료</text>

  <!-- Arrow: Half-open → Closed -->
  <path id="a-hc" d="M250 262 L120 262 L120 124" fill="none" stroke="#0F6E56" stroke-width="0.5" marker-end="url(#arrow)" opacity="0.4"/>
  <text id="l-hc" class="ts" x="112" y="200" text-anchor="end" fill="var(--color-text-secondary)" opacity="0.5">시험 성공</text>

  <!-- Arrow: Half-open → Open -->
  <path id="a-ho" d="M410 270 Q440 270 460 240 Q480 210 500 180 L540 124" fill="none" stroke="#A32D2D" stroke-width="0.5" marker-end="url(#arrow)" opacity="0.4"/>
  <text id="l-ho" class="ts" x="490" y="220" text-anchor="start" fill="var(--color-text-secondary)" opacity="0.5">시험 실패</text>

  <!-- Self-loop: Closed -->
  <path id="a-cc" d="M60 60 Q60 30 90 30 Q120 30 120 56" fill="none" stroke="#0F6E56" stroke-width="0.5" marker-end="url(#arrow)" opacity="0.3"/>
  <text id="l-cc" class="ts" x="90" y="22" text-anchor="middle" fill="var(--color-text-secondary)" opacity="0.3">성공</text>
</svg>

<div class="info-panel" id="info-panel">
  <div class="info-title" id="info-title">Closed — 정상 상태</div>
  <div id="info-body">모든 요청이 정상적으로 대상 서비스로 전달됩니다. 실패 횟수를 카운트하며, 설정된 임계치를 초과하면 Open 상태로 전환됩니다.</div>
</div>

<script>
const info = {
  closed: {
    title: 'Closed — 정상 상태',
    body: '모든 요청이 정상적으로 대상 서비스로 전달됩니다. 실패 횟수를 카운트하며, 설정된 임계치를 초과하면 Open 상태로 전환됩니다.'
  },
  open: {
    title: 'Open — 차단 상태',
    body: '모든 요청을 즉시 실패 처리(fast-fail)합니다. 장애가 전파되는 것을 막아 시스템을 보호합니다. 설정된 타임아웃이 지나면 Half-open으로 전환됩니다.'
  },
  half: {
    title: 'Half-open — 탐색 상태',
    body: '제한된 수의 시험 요청만 통과시켜 서비스 복구 여부를 확인합니다. 시험 요청이 성공하면 Closed로, 실패하면 다시 Open으로 돌아갑니다.'
  }
};

const arrowMap = {
  closed: ['a-co','a-cc'],
  open:   ['a-oh'],
  half:   ['a-hc','a-ho']
};
const labelMap = {
  closed: ['l-co','l-cc'],
  open:   ['l-oh'],
  half:   ['l-hc','l-ho']
};

function setState(s) {
  ['closed','open','half'].forEach(k => {
    const g = document.getElementById('g-'+k);
    const r = document.getElementById('r-'+k);
    const btn = document.getElementById('btn-'+k);
    if (k === s) {
      g.classList.add('active-state');
      r.setAttribute('stroke-width', '1');
      btn.classList.add('active');
    } else {
      g.classList.remove('active-state');
      r.setAttribute('stroke-width', '0.5');
      btn.classList.remove('active');
    }
  });

  ['a-co','a-oh','a-hc','a-ho','a-cc'].forEach(id => {
    document.getElementById(id).setAttribute('opacity','0.15');
  });
  ['l-co','l-oh','l-hc','l-ho','l-cc'].forEach(id => {
    document.getElementById(id).setAttribute('opacity','0.2');
  });

  (arrowMap[s]||[]).forEach(id => {
    document.getElementById(id).setAttribute('opacity','0.7');
  });
  (labelMap[s]||[]).forEach(id => {
    document.getElementById(id).setAttribute('opacity','0.8');
  });

  document.getElementById('info-title').textContent = info[s].title;
  document.getElementById('info-body').textContent = info[s].body;
}
</script>

### 핵심: Open 상태의 가치

Open 상태에서는 외부 API를 **아예 호출하지 않는다**. 이것이 Retry와의 결정적 차이다.

- **Retry**: 실패해도 일단 시도한다 → 일시적 장애에 효과적
- **Circuit Breaker**: 실패가 반복되면 시도 자체를 중단한다 → 장기 장애에 효과적

둘은 대체재가 아니라 **보완재**다. Retry로 일시적 장애를 복구하고, Circuit Breaker로 장기 장애 시 자원을 보호한다.

---

## Sliding Window: 실패율을 어떻게 판단할까

Circuit Breaker가 Closed → Open으로 전환하려면 "실패율이 임계치를 넘었는지" 판단해야 한다. 이때 **어떤 범위의 호출을 기준으로 실패율을 계산할 것인가**가 Sliding Window 방식이다.

### Count-based (최근 N회)

최근 N번의 호출 결과를 원형 배열(circular array)에 기록한다.

```
slidingWindowSize: 10 → 최근 10회 호출 중 실패율 계산
failureRateThreshold: 50 → 10회 중 5회 이상 실패하면 Open
```

**장점**: 구현이 단순하고, 메모리 사용이 일정하다 (N개 고정)

**단점**: 트래픽이 적을 때 문제가 된다. 1시간에 10번 호출되는 API에서 연속 5번 실패하면 서킷이 열리는데, 그 5번이 30분에 걸쳐 발생한 것일 수 있다.

### Time-based (최근 N초)

최근 N초 동안의 호출 결과를 기록한다.

```
slidingWindowSize: 60 → 최근 60초간의 호출 결과로 실패율 계산
failureRateThreshold: 50 → 60초 내 호출 중 50% 이상 실패하면 Open
```

**장점**: 시간 기반이므로 "최근 장애"를 더 정확하게 반영한다

**단점**: 트래픽에 따라 윈도우 내 호출 수가 가변적이다. 60초간 2번만 호출되고 둘 다 실패하면 실패율 100%로 서킷이 열릴 수 있다.

### 트레이드오프: 어떤 것을 선택할까?

| 기준 | Count-based | Time-based |
|------|-------------|------------|
| 트래픽이 안정적일 때 | 적합 | 적합 |
| 트래픽이 불규칙할 때 | 부적합 (오래된 결과 포함) | **적합** |
| 설정 난이도 | 낮음 (N회만 결정) | 중간 (시간 + minimumNumberOfCalls) |
| 메모리 | 고정 (N개) | 가변 |

**`minimumNumberOfCalls`이 핵심 안전장치다.** 이 값을 설정하면 최소 호출 수를 충족하기 전에는 실패율을 계산하지 않는다. Time-based의 "2번 중 2번 실패 = 100%" 문제를 방지한다.

---

## Failure Rate Threshold (실패율 임계치)

```yaml
failureRateThreshold: 50  # 50% 이상 실패하면 Open
```

이 값을 어떻게 잡느냐에 따라 서킷브레이커의 민감도가 달라진다.

| 임계치 | 민감도 | 적합한 상황 |
|--------|--------|-----------|
| 30% | 높음 (빨리 열림) | 실패 시 비용이 큰 서비스 (결제) |
| 50% | 균형 | 일반적인 외부 API |
| 70% | 낮음 (늦게 열림) | 간헐적 실패가 정상인 서비스 |

너무 낮으면 정상적인 간헐 실패에도 서킷이 열리고, 너무 높으면 이미 장애가 심각해진 후에야 열린다. 서비스 특성에 맞게 조정해야 한다.

---

## Resilience4j로 구현하기

### application.yml 설정

```yaml
resilience4j:
  circuitbreaker:
    instances:
      drugApi:
        slidingWindowType: COUNT_BASED
        slidingWindowSize: 10          # 최근 10회 호출 기준
        minimumNumberOfCalls: 5        # 최소 5회 호출 후 판단
        failureRateThreshold: 50       # 50% 실패 시 Open
        waitDurationInOpenState: 30s   # Open 후 30초 대기
        permittedNumberOfCallsInHalfOpenState: 3  # Half-Open에서 3회 시험
        retryExceptions 활용과 동일하게 예외 필터링 가능
```

### 상태 전이 시나리오

```
1. 정상 운영 (Closed)
   → 10회 중 2회 실패 (실패율 20%) → 여전히 Closed

2. 장애 발생 (Closed → Open)
   → 10회 중 6회 실패 (실패율 60% > 50%) → Open 전환
   → 이후 모든 요청은 외부 호출 없이 즉시 fallback 반환

3. 복구 시도 (Open → Half-Open)
   → 30초 경과 → Half-Open 전환
   → 3회 시험 호출 허용

4-a. 복구 성공 (Half-Open → Closed)
   → 시험 3회 중 2회 이상 성공 → Closed 복귀

4-b. 복구 실패 (Half-Open → Open)
   → 시험 3회 중 2회 이상 실패 → 다시 Open (30초 대기)
```

### @CircuitBreaker 적용

```java
@CircuitBreaker(name = "drugApi", fallbackMethod = "searchFallback")
@Retry(name = "drugApi")  // Retry와 조합
public String searchEasyDrug(String itemName, int page, int size) {
    // 외부 API 호출
}

private String searchFallback(String itemName, int page, int size, Exception e) {
    log.warn("CircuitBreaker fallback: {}", e.getMessage());
    return null;
}
```

### 어노테이션 실행 순서

Resilience4j에서 여러 어노테이션을 조합하면 실행 순서가 중요하다:

```
Retry ( CircuitBreaker ( 실제 호출 ) )
```

1. 실제 호출 실패 → CircuitBreaker가 실패 기록
2. CircuitBreaker가 예외를 던짐 → Retry가 재시도
3. 재시도 모두 실패 → fallback 호출

**CircuitBreaker가 Open이면 Retry도 의미가 없다.** Open 상태에서는 CircuitBreaker가 `CallNotPermittedException`을 던지고, Retry는 이 예외를 재시도하지 않도록 설정해야 한다.

---

## Retry vs Circuit Breaker 비교

| 관점 | Retry | Circuit Breaker |
|------|-------|----------------|
| 대응하는 장애 유형 | 일시적 (순간 네트워크 끊김) | 장기적 (서비스 다운) |
| 실패 시 동작 | 다시 시도 | 시도 자체를 차단 |
| 자원 소모 | 재시도만큼 추가 소모 | Open 시 자원 소모 0 |
| 외부 서비스 부하 | 재시도로 부하 증가 가능 | Open 시 부하 0 |
| 복구 감지 | 없음 (매번 시도) | Half-Open으로 자동 감지 |

조합하면: **일시적 장애 → Retry가 복구. 반복 실패 → Circuit Breaker가 차단. 대기 후 → Half-Open이 복구 확인.**

---

## 실무에서의 주의점

### 우아한형제들 사례

배민상회에서는 Resilience4j CircuitBreaker를 적용하고, Prometheus + Grafana로 서킷 상태를 모니터링한다. 서킷이 Open되면 Grafana 대시보드에서 즉시 확인할 수 있어, 장애 대응 속도가 빨라진다.

### fallback 설계가 핵심이다

Circuit Breaker의 가치는 Open 상태에서 **fallback이 얼마나 유용한가**에 달려있다.

| fallback 전략 | 예시 | 적합한 상황 |
|--------------|------|-----------|
| 빈 결과 반환 | `Collections.emptyList()` | 검색 결과 (없어도 앱은 동작) |
| 캐시된 값 반환 | 마지막 성공 응답 | 자주 변하지 않는 데이터 |
| 기본값 반환 | 하드코딩된 인기 약 목록 | 최소한의 사용자 경험 보장 |
| 에러 메시지 | "잠시 후 다시 시도해주세요" | 결제 등 대체 불가한 기능 |

### 모니터링 없는 Circuit Breaker는 위험하다

서킷이 Open된 걸 모르면, 사용자는 "검색이 안 된다"만 경험하고 개발팀은 아무것도 모른다. **서킷 상태 변화는 반드시 알림으로 연결**해야 한다.

---

## 정리

- **Cascading Failure**: 하나의 장애가 호출 체인을 따라 전파되어 전체 시스템이 무너지는 현상
- **Circuit Breaker**: 실패율이 임계치를 넘으면 호출을 차단하여 장애 전파를 막는 패턴
- **3가지 상태**: Closed(정상) → Open(차단) → Half-Open(시험) → 복구 시 Closed
- **Sliding Window**: Count-based(최근 N회)는 단순, Time-based(최근 N초)는 시간 기반으로 정확
- **Retry와 조합**: Retry는 일시적 장애, Circuit Breaker는 장기 장애에 대응. 둘은 보완재

---

## 참고 자료

- [Martin Fowler - Circuit Breaker](https://martinfowler.com/bliki/CircuitBreaker.html)
- [Resilience4j CircuitBreaker 공식 문서](https://resilience4j.readme.io/docs/circuitbreaker)
- [우아한형제들 기술블로그 - 서킷브레이커 적용기](https://techblog.woowahan.com/15694/)
- [화해 기술블로그 - 내부통신에 서킷브레이커 적용하기](https://blog.hwahae.co.kr/all/tech/14541)
- [Reflectoring - Implementing a Circuit Breaker with Resilience4j](https://reflectoring.io/circuitbreaker-with-resilience4j/)
