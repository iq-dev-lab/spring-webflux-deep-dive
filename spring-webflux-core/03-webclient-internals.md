# WebClient 완전 분해 — RestTemplate과 근본적으로 다른 이유

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `RestTemplate`의 블로킹 모델과 `WebClient`의 논블로킹 모델은 내부에서 어떻게 다른가?
- `retrieve()`와 `exchangeToMono()`는 에러 처리에서 왜 다르게 동작하는가?
- `WebClient`에서 재시도(`retryWhen`)와 타임아웃(`responseTimeout`)을 올바르게 설정하는 방법은?
- `WebClient.Builder`로 공통 설정을 어떻게 관리하는가?
- 4xx/5xx 에러를 `retrieve()`에서 커스텀 예외로 변환하는 방법은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

MSA 환경에서 외부 서비스 호출은 가장 빈번한 I/O 작업입니다. `RestTemplate`을 그대로 WebFlux에 가져오면 EventLoop를 블로킹하여 처리량이 급감합니다. `WebClient`를 제대로 사용하면 동일한 스레드 수로 수십 배 많은 외부 API 호출을 처리할 수 있습니다. `retrieve()`와 `exchangeToMono()`의 차이를 모르면 4xx 에러를 정상으로 처리하는 버그가 생깁니다.

---

## 😱 흔한 실수 (Before — WebClient를 잘못 사용할 때)

```
실수 1: retrieve()로 4xx 응답을 정상 처리

  webClient.get().uri("/user/999")
      .retrieve()
      .bodyToMono(User.class)
      .subscribe(user -> log.info("사용자: {}", user));
  // 서버가 404 반환 시:
  // retrieve()는 기본적으로 4xx → WebClientResponseException.NotFound 발생
  // 하지만 onError 없이 subscribe하면 에러 무시
  // subscribe에 에러 핸들러 항상 필요!

실수 2: exchangeToMono() 후 응답 바디를 소비하지 않음

  webClient.get().uri("/data")
      .exchangeToMono(response -> {
          if (response.statusCode().is2xxSuccessful()) {
              return response.bodyToMono(Data.class);
          }
          return Mono.error(new ServiceException("실패"));
          // 에러 시 response.bodyToMono() 안 호출 → 바디 미소비
          // → HTTP 연결 풀 고갈 가능
      });
  
  올바른 방식:
      return Mono.error(new ServiceException("실패"))
          .doFinally(_ -> response.releaseBody().subscribe());
  또는:
      return response.createException().flatMap(Mono::error);

실수 3: baseUrl 없이 WebClient.create() 후 상대 URI 사용

  WebClient client = WebClient.create();
  client.get().uri("/users/1").retrieve()...
  // baseUrl 없으면 "/users/1"은 상대 URI → 실제 URL 형성 안 됨
  // 반드시 baseUrl 설정하거나 절대 URI 사용
```

---

## ✨ 올바른 접근 (After — WebClient 올바른 사용)

```
WebClient 기본 패턴:

@Bean
WebClient webClient(WebClient.Builder builder) {
    return builder
        .baseUrl("https://api.example.com")
        .defaultHeader(HttpHeaders.CONTENT_TYPE, APPLICATION_JSON_VALUE)
        .build();
}

호출 패턴:
  webClient.get()
      .uri("/users/{id}", id)
      .retrieve()
      .onStatus(HttpStatusCode::is4xxClientError,
          response -> response.createException()
              .flatMap(Mono::error))    // WebClientResponseException으로 변환
      .bodyToMono(User.class)
      .timeout(Duration.ofSeconds(5))  // 응답 타임아웃
      .retryWhen(Retry.backoff(2, Duration.ofMillis(500))
          .filter(e -> !(e instanceof WebClientResponseException.NotFound))
      );
```

---

## 🔬 내부 동작 원리

### 1. RestTemplate vs WebClient — 내부 I/O 모델

```
RestTemplate (블로킹):

  Thread-1 (요청 스레드):
    1. HTTP 요청 구성
    2. HttpUrlConnection.connect() ← TCP 연결 대기
    3. OutputStream.write(requestBody) ← 요청 전송
    4. InputStream.read() ← 응답 대기 (블로킹!)
       [이 동안 Thread-1은 아무것도 못 함]
    5. 응답 역직렬화
    6. 반환

  → Thread-1은 외부 API 응답 시간만큼 블로킹
  → 100ms 외부 API, 200 스레드 → 초당 2,000 req/s 한계

────────────────────────────────────────

WebClient (논블로킹):

  EventLoop-1:
    1. HTTP 요청 구성 → Netty에 비동기 write
    2. 즉시 반환 (Mono<ClientResponse> 반환)
    3. 다른 요청 처리 중...

  Netty I/O (비동기):
    TCP 연결 → 요청 전송 → 응답 대기

  EventLoop-1 (응답 도착 시):
    4. 응답 수신 이벤트 → Subscriber에 onNext
    5. 역직렬화 → 하위 파이프라인 실행

  → EventLoop-1은 외부 API 대기 중에도 다른 요청 처리
  → 16 EventLoop로 수만 req/s 처리 가능 (외부 API가 한계)
```

### 2. retrieve() vs exchangeToMono() — 에러 처리 차이

```
retrieve():
  응답을 받아 2xx 여부 체크 후 바디 추출
  4xx → WebClientResponseException (기본) 또는 onStatus로 커스터마이징
  5xx → WebClientResponseException (기본)
  응답 바디를 프레임워크가 자동 소비/해제

  webClient.get().uri("/user/{id}", id)
      .retrieve()
      .onStatus(HttpStatus.NOT_FOUND::equals,
          response -> Mono.error(new UserNotFoundException(id)))
      .onStatus(HttpStatusCode::is5xxServerError,
          response -> response.bodyToMono(ErrorResponse.class)
              .flatMap(err -> Mono.error(new ServiceException(err.getMessage())))
      )
      .bodyToMono(User.class)

  장점: 간결, 응답 바디 자동 관리
  단점: 응답 헤더, 상태 코드 접근 제한

────────────────────────────────────────

exchangeToMono():
  원시 ClientResponse 완전 제어
  상태 코드, 헤더, 바디 모두 접근 가능
  단, 반드시 바디를 소비해야 함 (연결 풀 반환)

  webClient.get().uri("/user/{id}", id)
      .exchangeToMono(response -> {
          HttpStatusCode status = response.statusCode();
          HttpHeaders headers = response.headers().asHttpHeaders();

          if (status.is2xxSuccessful()) {
              return response.bodyToMono(User.class);
          } else if (status.equals(HttpStatus.NOT_FOUND)) {
              // 바디 소비 필수! (연결 반환)
              return response.bodyToMono(Void.class)
                  .then(Mono.error(new UserNotFoundException(id)));
          } else {
              // createException(): 바디를 읽어 WebClientResponseException 생성
              return response.createException().flatMap(Mono::error);
          }
      });

  장점: 완전한 응답 제어, 응답 헤더 접근
  단점: 바디 소비 관리 책임, 코드 장황
```

### 3. WebClient.Builder — 공통 설정 관리

```
WebClient.Builder:
  Spring Boot가 자동으로 Bean 등록
  Builder를 주입받아 커스터마이징 후 build()

  @Bean
  public WebClient externalApiClient(WebClient.Builder builder) {
      return builder
          // 기본 URL
          .baseUrl("https://api.external.com")

          // 기본 헤더
          .defaultHeader("X-API-Version", "v2")
          .defaultHeader(HttpHeaders.ACCEPT, APPLICATION_JSON_VALUE)

          // 기본 Cookie
          .defaultCookie("session", "abc123")

          // 공통 에러 처리
          .defaultStatusHandler(
              HttpStatusCode::isError,
              response -> response.createException()
                  .flatMap(Mono::error)
          )

          // HTTP 클라이언트 커스터마이징 (연결 풀, 타임아웃)
          .clientConnector(new ReactorClientHttpConnector(
              HttpClient.create(connectionProvider)
                  .responseTimeout(Duration.ofSeconds(10))
          ))
          .build();
  }

Spring Boot의 WebClient.Builder 자동 구성:
  - ObjectMapper Jackson 설정 자동 반영
  - Actuator/Micrometer 메트릭 자동 통합
  - HttpMessageCodecs 자동 구성
  → WebClient.create()보다 WebClient.Builder 사용 권장
```

### 4. 타임아웃 설정 — 세 가지 레벨

```
타임아웃 종류:

① 연결 타임아웃 (Connection Timeout):
  TCP 연결 수립까지 최대 대기 시간
  HttpClient.option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 3000)

② 응답 타임아웃 (Response Timeout):
  연결 후 응답 도착까지 최대 대기 시간
  HttpClient.responseTimeout(Duration.ofSeconds(5))
  또는 체인에서: .timeout(Duration.ofSeconds(5))

③ 읽기/쓰기 타임아웃:
  Netty 핸들러 레벨
  .doOnConnected(conn -> conn
      .addHandlerLast(new ReadTimeoutHandler(5))
      .addHandlerLast(new WriteTimeoutHandler(3))
  )

타임아웃 설정 완전 예시:
  HttpClient httpClient = HttpClient.create()
      .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 3_000)
      .responseTimeout(Duration.ofSeconds(5))
      .doOnConnected(conn -> conn
          .addHandlerLast(new ReadTimeoutHandler(5))
          .addHandlerLast(new WriteTimeoutHandler(3))
      );

  WebClient client = WebClient.builder()
      .clientConnector(new ReactorClientHttpConnector(httpClient))
      .build();

  // 요청별 타임아웃 (연결 타임아웃 설정 재정의)
  client.get().uri("/data")
      .retrieve()
      .bodyToMono(String.class)
      .timeout(Duration.ofSeconds(3));  // 이 요청만 3초 타임아웃
```

### 5. 재시도 전략 — retryWhen 실전 패턴

```
WebClient + retryWhen 완전 패턴:

webClient.get()
    .uri("/external/resource")
    .retrieve()
    .bodyToMono(Resource.class)
    .retryWhen(Retry.backoff(3, Duration.ofMillis(300))
        .maxBackoff(Duration.ofSeconds(5))
        .jitter(0.3)
        .filter(throwable -> {
            // 5xx만 재시도, 4xx는 재시도 불필요
            if (throwable instanceof WebClientResponseException ex) {
                return ex.getStatusCode().is5xxServerError();
            }
            // 타임아웃, 연결 오류는 재시도
            return throwable instanceof TimeoutException
                || throwable instanceof ConnectException;
        })
        .doBeforeRetry(signal ->
            log.warn("재시도 #{}: {}",
                signal.totalRetries() + 1,
                signal.failure().getMessage())
        )
        .onRetryExhaustedThrow((retrySpec, retrySignal) ->
            new ExternalServiceException(
                "외부 서비스 일시 불가", retrySignal.failure())
        )
    );
```

---

## 💻 실전 코드

### 실험 1: 완전한 WebClient Bean 설정

```java
@Configuration
public class WebClientConfig {

    @Bean
    public ConnectionProvider externalApiConnectionProvider() {
        return ConnectionProvider.builder("external-api")
            .maxConnections(50)
            .maxIdleTime(Duration.ofSeconds(20))
            .maxLifeTime(Duration.ofSeconds(60))
            .pendingAcquireTimeout(Duration.ofSeconds(30))
            .metrics(true)
            .build();
    }

    @Bean
    public WebClient externalApiWebClient(
            WebClient.Builder builder,
            ConnectionProvider externalApiConnectionProvider) {

        HttpClient httpClient = HttpClient.create(externalApiConnectionProvider)
            .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 3_000)
            .responseTimeout(Duration.ofSeconds(10))
            .doOnConnected(conn -> conn
                .addHandlerLast(new ReadTimeoutHandler(10))
                .addHandlerLast(new WriteTimeoutHandler(5))
            );

        return builder
            .baseUrl("https://api.external-service.com")
            .clientConnector(new ReactorClientHttpConnector(httpClient))
            .defaultHeader(HttpHeaders.CONTENT_TYPE,
                MediaType.APPLICATION_JSON_VALUE)
            .defaultHeader("X-Client-ID", "my-service")
            .build();
    }
}
```

### 실험 2: 서비스 레이어에서 WebClient 사용

```java
@Service
@RequiredArgsConstructor
public class ExternalUserService {
    private final WebClient externalApiWebClient;

    public Mono<ExternalUser> findUser(Long userId) {
        return externalApiWebClient.get()
            .uri("/users/{id}", userId)
            .retrieve()
            .onStatus(HttpStatus.NOT_FOUND::equals,
                response -> Mono.error(new UserNotFoundException(userId)))
            .onStatus(HttpStatusCode::is5xxServerError,
                response -> response.createException()
                    .flatMap(Mono::error))
            .bodyToMono(ExternalUser.class)
            .timeout(Duration.ofSeconds(5))
            .retryWhen(Retry.backoff(2, Duration.ofMillis(300))
                .filter(e -> e instanceof WebClientResponseException ex
                    && ex.getStatusCode().is5xxServerError())
            )
            .doOnError(e -> log.error("외부 사용자 조회 실패: id={}", userId, e));
    }

    public Mono<ExternalUser> createUser(CreateUserRequest request) {
        return externalApiWebClient.post()
            .uri("/users")
            .bodyValue(request)
            .retrieve()
            .onStatus(HttpStatus.CONFLICT::equals,
                response -> Mono.error(new DuplicateUserException()))
            .bodyToMono(ExternalUser.class);
    }
}
```

### 실험 3: multipart 파일 업로드

```java
public Mono<UploadResult> uploadFile(FilePart filePart) {
    MultipartBodyBuilder builder = new MultipartBodyBuilder();
    builder.asyncPart("file", filePart.content(), DataBuffer.class)
        .filename(filePart.filename())
        .contentType(filePart.headers().getContentType());

    return webClient.post()
        .uri("/upload")
        .body(BodyInserters.fromMultipartData(builder.build()))
        .retrieve()
        .bodyToMono(UploadResult.class);
}
```

---

## 📊 성능 비교

```
RestTemplate vs WebClient 처리량 비교:
(외부 API 응답 시간: 100ms, 동시 요청: 1000개)

RestTemplate (Spring MVC):
  스레드 200개 (기본) → 초당 2,000 req/s
  1,000 동시 요청 시: 800개 큐 대기
  CPU 사용률: 낮음 (대부분 I/O 대기)
  메모리: 스레드 × 512KB = 100MB 스택

WebClient (WebFlux):
  EventLoop 16개 + ConnectionProvider
  1,000 동시 요청: EventLoop에서 비동기 처리
  외부 API 한계까지 처리 (연결 풀 크기에 따름)
  CPU 사용률: 높음 (EventLoop 활성)
  메모리: 8MB 스택 (스레드 16개)

실제 처리량 차이:
  I/O 집약 (외부 API 호출): WebClient 5~20배 우위
  CPU 집약 (데이터 변환): 차이 없음
  혼합: 상황에 따름
```

---

## ⚖️ 트레이드오프

```
retrieve() vs exchangeToMono():
  retrieve(): 대부분의 경우 충분, 간결
  exchangeToMono(): 응답 헤더 접근, 복잡한 상태 코드 처리 필요 시
  주의: exchangeToMono()는 바디 소비 책임

타임아웃 설정:
  너무 짧음: 정상 응답도 타임아웃 → 불필요한 재시도
  너무 긺: 장애 서비스에 연결 풀 고갈 위험
  권장: SLA의 50~70% (SLA 10초 → 타임아웃 5~7초)

재시도:
  4xx는 재시도 불필요 (클라이언트 에러, 재시도해도 동일)
  5xx + 타임아웃만 재시도
  재시도 횟수 × 최대 지연 < 전체 타임아웃
  jitter 필수 (동시 재시도 충돌 방지)
```

---

## 📌 핵심 정리

```
WebClient 핵심:

RestTemplate vs WebClient:
  RestTemplate: 블로킹, Thread-per-Request
  WebClient: 논블로킹, EventLoop 비동기

retrieve() vs exchangeToMono():
  retrieve(): 간결, 바디 자동 관리, 4xx/5xx 자동 에러
  exchangeToMono(): 전체 응답 제어, 바디 소비 필수

타임아웃 (3단계):
  연결 타임아웃 → 응답 타임아웃 → 읽기 타임아웃

재시도 원칙:
  5xx + 연결 오류만 재시도
  지수 백오프 + jitter
  retryWhen(Retry.backoff(...).filter(...))

WebClient Bean 관리:
  WebClient.Builder 주입 → 자동 설정 유지
  ConnectionProvider 별도 Bean으로 커스터마이징
  매 요청마다 create() 금지
```

---

## 🤔 생각해볼 문제

**Q1.** `retrieve()` 후 `bodyToMono(Void.class)`와 `bodyToMono(String.class)`의 동작 차이는 무엇인가요?

<details>
<summary>해설 보기</summary>

**`bodyToMono(Void.class)`**: 응답 바디를 무시하고 바로 완료(`onComplete`) 신호. 바디가 있어도 읽지 않고 버립니다. 응답 바디가 필요 없는 경우(DELETE, 일부 POST) 사용.

**`bodyToMono(String.class)`**: 응답 바디를 UTF-8 문자열로 읽어 반환. 바디가 없으면 `empty()`(onComplete, 값 없음).

```java
// 바디 무시 (204 No Content 등)
webClient.delete().uri("/users/{id}", id)
    .retrieve()
    .bodyToMono(Void.class);  // 바디 무시, 완료만 확인

// 바디 읽기
webClient.get().uri("/users/1")
    .retrieve()
    .bodyToMono(String.class);  // JSON 문자열로 읽기
```

단, 에러 상황에서 `bodyToMono(Void.class)`를 쓰면 에러 상세 정보를 잃습니다. `onStatus`로 에러를 먼저 처리한 후 `bodyToMono(Void.class)`를 쓰는 것이 안전합니다.

</details>

---

**Q2.** `WebClient`에서 요청마다 다른 인증 토큰을 헤더에 추가하는 방법은 무엇인가요?

<details>
<summary>해설 보기</summary>

`defaultHeader`는 모든 요청에 고정 값을 추가합니다. 요청마다 다른 값은 두 가지 방법으로 처리합니다.

```java
// 방법 1: 요청 시 헤더 직접 추가
webClient.get()
    .uri("/protected/resource")
    .header(HttpHeaders.AUTHORIZATION, "Bearer " + getToken())
    .retrieve()
    .bodyToMono(Resource.class);

// 방법 2: ExchangeFilterFunction으로 동적 헤더 주입
WebClient client = webClient.mutate()
    .filter(ExchangeFilterFunction.ofRequestProcessor(request -> {
        return tokenService.getToken()  // Mono<String>
            .map(token -> ClientRequest.from(request)
                .header(HttpHeaders.AUTHORIZATION, "Bearer " + token)
                .build());
    }))
    .build();
// 이 WebClient의 모든 요청에 동적으로 토큰 주입
```

`ExchangeFilterFunction`을 사용하면 모든 요청에 일관되게 적용되어 반복 코드를 줄일 수 있습니다. Reactor Context에서 SecurityContext를 꺼내 토큰을 주입하는 패턴도 자주 사용됩니다.

</details>

---

**Q3.** `WebClient`로 대용량 응답을 스트리밍 방식으로 처리하려면 어떻게 하나요?

<details>
<summary>해설 보기</summary>

`bodyToFlux(DataBuffer.class)`로 청크 단위로 스트리밍하거나, 줄 단위로 읽으려면 `bodyToFlux(String.class)`를 사용합니다.

```java
// 대용량 파일 다운로드 스트리밍
public Flux<DataBuffer> downloadFile(String fileId) {
    return webClient.get()
        .uri("/files/{id}/download", fileId)
        .retrieve()
        .bodyToFlux(DataBuffer.class);
    // 응답 전체를 메모리에 올리지 않고 청크 단위로 처리
}

// 컨트롤러에서 직접 스트리밍 응답
@GetMapping("/proxy/large-file")
public Mono<Void> proxyLargeFile(ServerWebExchange exchange) {
    return webClient.get()
        .uri("https://external/large-file")
        .retrieve()
        .bodyToFlux(DataBuffer.class)
        .as(dataBufferFlux ->
            exchange.getResponse().writeWith(dataBufferFlux)
        );
    // 외부 응답을 그대로 클라이언트에 스트리밍 (프록시)
    // 전체를 메모리에 올리지 않음
}

// NDJSON 스트리밍 (줄 단위 JSON)
webClient.get()
    .uri("/events/stream")
    .accept(MediaType.APPLICATION_NDJSON)
    .retrieve()
    .bodyToFlux(Event.class);  // 각 줄을 Event로 역직렬화
```

`DataBuffer`를 사용할 때는 `DataBufferUtils.release(buffer)`로 버퍼를 해제해야 메모리 누수가 없습니다.

</details>

---

<div align="center">

**[⬅️ 이전: 어노테이션 vs 함수형 엔드포인트](./02-annotation-vs-functional.md)** | **[홈으로 🏠](../README.md)** | **[다음: WebClient 고급 패턴 ➡️](./04-webclient-advanced-patterns.md)**

</div>
