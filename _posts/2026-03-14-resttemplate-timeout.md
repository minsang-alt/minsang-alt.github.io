---
title: RestTemplate 타임아웃과 연결 관리 - 사전 학습
date: 2026-03-14 10:00:00 +/-0800
categories: [Tech Notes, Spring]
tags: [spring, resttemplate, timeout, connection-pool]
---

## TCP 3-way Handshake와 Connect Timeout

### TCP 3-way Handshake란

HTTP 요청을 보내기 전에, 클라이언트와 서버는 먼저 TCP 연결을 수립해야 합니다. 이 과정이 3-way Handshake입니다.

```
클라이언트                    서버
    |  ---- SYN -------->    |   1) "연결하고 싶어"
    |  <--- SYN + ACK ---    |   2) "알겠어, 나도 준비됐어"
    |  ---- ACK -------->    |   3) "확인, 연결하자"
    |                        |
    |  ---- HTTP 요청 --->   |   이제야 데이터 전송 시작
```

이 3단계가 완료되어야 비로소 HTTP 요청을 보낼 수 있습니다.

### Connect Timeout이란

Connect Timeout은 **이 3-way Handshake가 완료되기까지 기다리는 최대 시간**입니다.

서버가 다운되었거나, 네트워크가 불안정하거나, 방화벽이 패킷을 drop하는 경우 SYN을 보내도 SYN+ACK가 돌아오지 않습니다. Connect Timeout이 없으면 클라이언트는 이 응답을 **무한히 기다립니다**.

```java
// Connect Timeout = 3초로 설정하면
// 3초 안에 TCP 연결이 안 되면 ConnectTimeoutException 발생
factory.setConnectTimeout(3000);
```

일반적으로 같은 네트워크 내 서버라면 TCP 연결은 수 ms면 충분합니다. 외부 API라 해도 수백 ms를 넘기면 비정상입니다. 따라서 Connect Timeout은 **1~3초** 정도로 짧게 잡는 것이 일반적입니다.

## Read Timeout

### Connect Timeout과의 차이

TCP 연결이 성공한 **이후**, HTTP 요청을 보내고 **응답 데이터가 돌아오기까지 기다리는 시간**이 Read Timeout입니다.

```
[Connect Timeout 구간]          [Read Timeout 구간]
       |                              |
클라이언트 --- SYN/ACK --> 서버    클라이언트 --- HTTP 요청 --> 서버
       TCP 연결 수립                서버가 처리 중... 대기...
                                   <--- HTTP 응답 ---
```

| 구분 | Connect Timeout | Read Timeout |
|------|----------------|-------------|
| 언제 | TCP 연결 수립 단계 | HTTP 요청 후 응답 대기 단계 |
| 원인 | 서버 다운, 네트워크 장애, 방화벽 | 서버 처리 지연, 과부하 |
| 보통 값 | 1~3초 | API 특성에 따라 다름 (3~30초) |
| 예외 | `ConnectTimeoutException` | `SocketTimeoutException` |

### Read Timeout을 API마다 다르게 설정해야 하는 이유

모든 API가 같은 속도로 응답하지 않습니다.

- **공공 약 검색 API**: 평균 500ms ~ 1초 → Read Timeout 5초면 충분
- **Claude AI API**: LLM 추론에 10~20초 걸릴 수 있음 → Read Timeout 30초 필요
- **Toss 결제 API**: 결제 승인에 2~3초 → Read Timeout 10초

Read Timeout을 너무 짧게 잡으면 정상 응답을 버리고, 너무 길게 잡으면 스레드가 오래 점유됩니다.

## SimpleClientHttpRequestFactory vs HttpComponentsClientHttpRequestFactory

RestTemplate은 내부적으로 `ClientHttpRequestFactory`를 사용해 HTTP 요청을 만듭니다. 두 가지 주요 구현체가 있습니다.

### SimpleClientHttpRequestFactory (기본값)

```java
// new RestTemplate()은 내부적으로 이걸 사용
RestTemplate restTemplate = new RestTemplate();
// == new RestTemplate(new SimpleClientHttpRequestFactory())
```

- Java의 `HttpURLConnection`을 사용
- **요청마다 새로운 TCP 연결을 생성하고 종료** (커넥션 재사용 없음)
- 커넥션 풀이 없음
- 간단한 테스트용으로는 괜찮지만, 운영에서는 비효율적

### HttpComponentsClientHttpRequestFactory

```java
// Apache HttpClient를 사용
CloseableHttpClient httpClient = HttpClients.custom()
    .setMaxConnTotal(100)         // 전체 최대 커넥션 수
    .setMaxConnPerRoute(20)       // 호스트당 최대 커넥션 수
    .build();

HttpComponentsClientHttpRequestFactory factory =
    new HttpComponentsClientHttpRequestFactory(httpClient);
factory.setConnectTimeout(3000);
factory.setReadTimeout(5000);

RestTemplate restTemplate = new RestTemplate(factory);
```

- Apache HttpClient 라이브러리 기반
- **커넥션 풀**을 지원하여 TCP 연결을 재사용
- Keep-Alive, 재시도, 커넥션 관리 등 고급 기능 제공
- 운영 환경에서 권장

### 비교 정리

| 기준 | SimpleClientHttpRequestFactory | HttpComponentsClientHttpRequestFactory |
|------|-------------------------------|---------------------------------------|
| 내부 구현 | `HttpURLConnection` (JDK 내장) | Apache HttpClient (외부 라이브러리) |
| 커넥션 풀 | 없음 (매번 새 연결) | 있음 (재사용) |
| 성능 | 낮음 (Handshake 반복) | 높음 (연결 재사용) |
| 설정 유연성 | 제한적 | 세밀한 제어 가능 |
| 의존성 | 없음 | `httpclient5` 추가 필요 |
| 적합한 환경 | 테스트, 간단한 호출 | 운영, 빈번한 외부 API 호출 |

## Connection Pool (Apache HttpClient의 커넥션 재사용)

### 왜 커넥션 풀이 필요한가

`SimpleClientHttpRequestFactory`는 요청마다 TCP 3-way Handshake를 반복합니다.

```
요청 1: SYN → SYN+ACK → ACK → HTTP 요청 → 응답 → 연결 종료
요청 2: SYN → SYN+ACK → ACK → HTTP 요청 → 응답 → 연결 종료  (또 Handshake!)
요청 3: SYN → SYN+ACK → ACK → HTTP 요청 → 응답 → 연결 종료  (또!)
```

커넥션 풀을 사용하면 한 번 맺은 연결을 재사용합니다.

```
요청 1: SYN → SYN+ACK → ACK → HTTP 요청 → 응답 → [연결 유지]
요청 2: HTTP 요청 → 응답 → [연결 유지]  (Handshake 생략!)
요청 3: HTTP 요청 → 응답 → [연결 유지]  (Handshake 생략!)
```

### 주요 설정값

```java
CloseableHttpClient httpClient = HttpClients.custom()
    .setMaxConnTotal(100)        // 풀 전체 최대 커넥션 수
    .setMaxConnPerRoute(20)      // 동일 호스트(route)당 최대 커넥션 수
    .evictIdleConnections(30, TimeUnit.SECONDS)  // 유휴 커넥션 정리
    .build();
```

- **MaxConnTotal**: 풀에 유지할 전체 커넥션 수. 너무 작으면 대기 발생, 너무 크면 서버에 부담.
- **MaxConnPerRoute**: 하나의 호스트에 동시에 열 수 있는 최대 커넥션 수. 공공 API 서버 하나에 동시 20개 연결까지 허용하는 식.

### 커넥션 풀은 어떻게 연결을 재사용하는가

커넥션 풀은 요청의 내용(검색어 등)을 구분하지 않습니다. **목적지(host:port)** 기준으로 연결을 관리합니다.

```
풀 내부 상태:

| 커넥션 | 목적지                        | 상태   |
|--------|-------------------------------|--------|
| conn-1 | apis.data.go.kr:443           | 사용중 |
| conn-2 | apis.data.go.kr:443           | 유휴   | ← 재사용 가능!
| conn-3 | api.tosspayments.com:443      | 유휴   |
```

동작 흐름은 다음과 같습니다.

```
요청 1: "타이레놀 검색" → apis.data.go.kr로 보내야 함
  1) 풀에서 apis.data.go.kr:443 유휴 커넥션 확인
  2) 없음 → TCP 3-way Handshake로 새 연결 생성 (conn-1)
  3) conn-1로 HTTP 요청 전송 → 응답 수신
  4) conn-1을 풀에 반환 (연결 끊지 않고 "유휴" 상태로 보관)

요청 2: "아스피린 검색" → apis.data.go.kr로 보내야 함
  1) 풀에서 apis.data.go.kr:443 유휴 커넥션 확인
  2) conn-1이 유휴 상태 → 꺼냄
  3) conn-1로 HTTP 요청 전송 → 응답 수신 (Handshake 생략!)
  4) conn-1을 풀에 반환
```

검색어가 다르더라도 **같은 서버로 가는 요청**이면 같은 TCP 연결을 재사용합니다.

### HTTP Keep-Alive

TCP 연결을 재사용하려면 서버도 동의해야 합니다. 이것이 `Connection: keep-alive` 헤더의 역할입니다.

```
HTTP/1.1 200 OK
Connection: keep-alive      ← "이 연결 끊지 마, 다음 요청도 받을게"
Keep-Alive: timeout=5       ← "5초 동안 유지할게"
```

HTTP/1.1에서는 keep-alive가 기본값이므로 대부분의 서버가 지원합니다.

### 커넥션 풀 사이즈와 톰캣 스레드의 관계

톰캣 기본 스레드 수는 200개입니다. 만약 200개 스레드가 모두 외부 API를 호출하는데 커넥션 풀이 20개라면, 180개 스레드는 커넥션을 얻기 위해 대기합니다. 반대로 풀 사이즈를 200으로 잡으면 외부 API 서버에 200개 동시 요청이 가서 상대 서버에 부담을 줄 수 있습니다.

## ResourceAccessException

### 타임아웃 시 발생하는 예외

RestTemplate에서 타임아웃이 발생하면 `ResourceAccessException`이 throw됩니다.

```java
try {
    restTemplate.getForObject(url, String.class);
} catch (ResourceAccessException e) {
    // e.getCause()로 원인을 구분할 수 있다
    if (e.getCause() instanceof ConnectTimeoutException) {
        // TCP 연결 자체가 실패 (서버 다운, 네트워크 장애)
    } else if (e.getCause() instanceof SocketTimeoutException) {
        // 연결은 됐지만 응답이 늦음 (서버 과부하)
    }
}
```

### 예외 계층 구조

```
RestClientException
  └── ResourceAccessException    ← 타임아웃, 네트워크 오류
        ├── cause: ConnectTimeoutException   ← Connect Timeout 초과
        └── cause: SocketTimeoutException    ← Read Timeout 초과
  └── HttpClientErrorException   ← 4xx 응답
  └── HttpServerErrorException   ← 5xx 응답
```

주의할 점은 `ResourceAccessException`은 **응답 자체를 받지 못한 경우**이고, `HttpServerErrorException`은 **응답은 받았지만 서버 에러인 경우**입니다. 둘은 다른 상황이므로 다르게 처리해야 합니다.

## RestTemplate vs WebClient vs RestClient

Spring에서 HTTP 클라이언트는 세 가지 선택지가 있습니다.

### RestTemplate (Spring 3.0~)

```java
RestTemplate restTemplate = new RestTemplate();
String result = restTemplate.getForObject("/api/data", String.class);
```

- **동기/블로킹** 방식
- 요청을 보내면 응답이 올 때까지 스레드가 대기
- 직관적이고 사용이 간단
- Spring 5.0부터 "유지보수 모드" — 새 기능 추가 없음, 버그 수정만

### WebClient (Spring 5.0~)

```java
WebClient webClient = WebClient.create();
Mono<String> result = webClient.get()
    .uri("/api/data")
    .retrieve()
    .bodyToMono(String.class);
```

- **비동기/논블로킹** 방식 (Reactor 기반)
- 스레드를 점유하지 않고 응답을 기다림 → 높은 동시성
- `Mono`, `Flux` 등 리액티브 타입 사용 필수
- 러닝 커브가 높고, 리액티브 스택(`spring-webflux`) 의존성 필요
- 동기식으로도 사용 가능하지만 (`.block()`) 이러면 WebClient의 장점이 사라짐

### RestClient (Spring 6.1+ / Boot 3.2+)

```java
RestClient restClient = RestClient.create();
String result = restClient.get()
    .uri("/api/data")
    .retrieve()
    .body(String.class);
```

- **동기/블로킹**이지만 fluent API 제공
- RestTemplate의 후계자 — Spring이 **신규 프로젝트에 권장**
- WebClient처럼 체이닝 방식이지만, 리액티브 의존성 불필요
- 내부적으로 RestTemplate과 같은 `ClientHttpRequestFactory` 사용

### 선택 기준

| 기준 | RestTemplate | WebClient | RestClient |
|------|-------------|-----------|-----------|
| 동기/비동기 | 동기 | 비동기 (동기 가능) | 동기 |
| 러닝 커브 | 낮음 | 높음 | 낮음 |
| 리액티브 의존성 | 불필요 | 필수 | 불필요 |
| Spring 권장도 | 유지보수 모드 | Reactive 스택이면 권장 | 신규 추천 |
| 기존 코드 전환 비용 | - | 높음 (Mono/Flux 전환) | 중간 |

### 현재 프로젝트에서의 선택

기존 코드가 RestTemplate 기반이고 리액티브 스택을 사용하지 않는 상황이라면, **RestTemplate을 유지하면서 타임아웃과 커넥션 풀만 제대로 설정하는 것**이 가장 현실적입니다. WebClient로 전환하면 리액티브 학습 + 전체 호출 코드 변경이 필요하고, RestClient는 Spring Boot 3.2+ 에서만 사용 가능하므로 프로젝트 버전을 확인해야 합니다.

"완벽한 기술 선택"보다 **"지금 가장 적은 비용으로 문제를 해결하는 선택"**이 중요합니다.
