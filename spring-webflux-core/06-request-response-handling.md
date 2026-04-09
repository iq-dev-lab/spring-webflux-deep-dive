# 요청/응답 처리 — ServerRequest와 Reactive 기반 본문 읽기

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `ServerRequest`와 `ServerHttpRequest`는 어떻게 다른가?
- `multipart/form-data`를 WebFlux에서 스트리밍 방식으로 처리하는 방법은?
- 대용량 파일을 `Flux<DataBuffer>`로 스트리밍하여 메모리 부족을 방지하는 방법은?
- `DataBufferUtils.release()`를 빼먹으면 어떤 일이 발생하는가?
- 파일 다운로드를 응답 스트리밍으로 구현하는 방법은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

파일 업로드/다운로드를 잘못 구현하면 두 가지 심각한 문제가 생깁니다. 첫째, 대용량 파일을 통째로 메모리에 올리면 OOM이 발생합니다. 둘째, `DataBuffer`를 해제하지 않으면 Netty의 기본 메모리 풀(PooledByteBufAllocator)에서 메모리가 반환되지 않아 점진적 메모리 누수가 생깁니다. `Flux<DataBuffer>` 스트리밍과 `DataBufferUtils`의 올바른 사용법을 이해하면 이 문제를 예방할 수 있습니다.

---

## 😱 흔한 실수 (Before — 요청/응답 처리를 잘못할 때)

```
실수 1: 대용량 파일 업로드를 통째로 메모리에 로드

  @PostMapping("/upload")
  public Mono<String> upload(@RequestBody Mono<byte[]> body) {
      return body.map(bytes -> {
          // 전체 파일이 byte[]로 메모리에!
          saveToStorage(bytes);  // 1GB 파일 → 1GB Heap 사용
          return "OK";
      });
  }

실수 2: DataBuffer를 해제하지 않음 (메모리 누수)

  Flux<DataBuffer> buffers = request.getBody();
  buffers.subscribe(buffer -> {
      byte[] bytes = new byte[buffer.readableByteCount()];
      buffer.read(bytes);
      process(bytes);
      // DataBufferUtils.release(buffer) 누락!
      // Netty 버퍼 풀에 반환 안 됨 → 메모리 누수
  });

실수 3: multipart FilePart를 블로킹으로 처리

  @PostMapping("/upload")
  public Mono<String> upload(@RequestPart FilePart file) {
      File dest = new File("/upload/" + file.filename());
      file.transferTo(dest).block();  // EventLoop 블로킹!
      return Mono.just("OK");
  }
```

---

## ✨ 올바른 접근 (After — 스트리밍 기반 처리)

```
올바른 파일 업로드 패턴:

@PostMapping("/upload")
public Mono<String> upload(@RequestPart("file") FilePart filePart) {
    Path dest = Path.of("/upload/" + filePart.filename());
    return filePart.transferTo(dest)  // 논블로킹 파일 쓰기
        .thenReturn("업로드 완료: " + filePart.filename());
}

올바른 DataBuffer 처리:
  DataBufferUtils.join(request.getBody())  // 청크 합산 (주의: 전체 메모리)
  .map(buffer -> {
      try {
          return processBuffer(buffer);
      } finally {
          DataBufferUtils.release(buffer);  // 반드시 해제
      }
  })

스트리밍 처리 (메모리 최소화):
  request.getBody()  // Flux<DataBuffer>
      .map(buffer -> {
          // 청크 단위로 처리
          DataBuffer processed = transform(buffer);
          DataBufferUtils.release(buffer);  // 입력 버퍼 해제
          return processed;
      })
      .as(body -> response.writeWith(body));  // 바로 출력
```

---

## 🔬 내부 동작 원리

### 1. ServerRequest vs ServerHttpRequest

```
ServerRequest:
  함수형 엔드포인트(RouterFunction)의 요청 추상화
  HandlerFunction에 주입됨
  
  주요 메서드:
  request.pathVariable("id")          // URI 변수
  request.queryParam("page")          // 쿼리 파라미터
  request.headers().contentType()     // Content-Type
  request.bodyToMono(User.class)      // 본문 역직렬화
  request.bodyToFlux(DataBuffer.class) // 원시 버퍼 스트림
  request.multipartData()             // multipart 데이터
  request.exchange()                  // 하위 ServerWebExchange 접근

ServerHttpRequest:
  어노테이션 방식(@Controller)의 요청 추상화
  ServerWebExchange.getRequest()로 접근
  또는 @Controller 파라미터로 직접 주입

  ServerHttpRequest request = exchange.getRequest();
  request.getURI()
  request.getHeaders()
  request.getCookies()
  request.getBody()  // Flux<DataBuffer> (원시 바이트)
  request.getQueryParams()  // MultiValueMap<String, String>

차이점:
  ServerRequest: 상위 추상화 (bodyToMono 등 편의 메서드 제공)
  ServerHttpRequest: 하위 추상화 (원시 DataBuffer 수준)
  → 어노테이션 방식: @RequestBody, @RequestParam 등으로 자동 바인딩
  → 함수형 방식: ServerRequest로 명시적 처리
```

### 2. DataBuffer와 메모리 관리

```
DataBuffer:
  Netty의 ByteBuf를 스프링이 래핑한 추상화
  기본 구현: NettyDataBuffer (PooledByteBufAllocator 사용)
  풀링(Pooling): 동일 버퍼를 재사용하여 GC 압박 감소

참조 카운팅:
  ByteBuf는 참조 카운트 기반 메모리 관리
  생성 시: refCount = 1
  retain(): refCount++
  release(): refCount-- (0이 되면 풀에 반환)

DataBufferUtils.release(buffer):
  NettyDataBuffer → 내부 ByteBuf.release() 호출
  refCount 감소 → 0이 되면 풀로 반환

  해제 안 하면:
    refCount 영구적으로 > 0
    → 풀에 반환 안 됨
    → 다른 버퍼가 새로 할당
    → 시간이 지남에 따라 메모리 고갈
    → Netty: "LEAK: ByteBuf.release() was not called" 경고

안전한 DataBuffer 처리 패턴:
  1. DataBufferUtils.join() — 여러 청크 합산
     DataBufferUtils.join(publisher)  // Mono<DataBuffer>
     → 모든 청크를 하나로 합산 (메모리 주의)
     → 반환된 DataBuffer는 사용 후 반드시 release()

  2. DataBufferUtils.toByteArray() — byte[]로 추출
     DataBufferUtils.join(publisher)
         .map(DataBufferUtils::toByteArrayAndRelease)
         // 내부에서 toByteArray + release 처리

  3. 청크 단위 처리 — 스트리밍
     publisher.map(buffer -> {
         try {
             return processChunk(buffer);
         } finally {
             DataBufferUtils.release(buffer);
         }
     })
```

### 3. 파일 업로드 — multipart 처리

```
@RequestPart (어노테이션 방식):
  @PostMapping("/upload")
  public Mono<String> upload(
          @RequestPart("file") FilePart file,
          @RequestPart("metadata") Mono<FileMetadata> metadata) {
      return metadata
          .flatMap(meta -> {
              Path dest = uploadPath.resolve(meta.getFolder())
                  .resolve(file.filename());
              return file.transferTo(dest)
                  .thenReturn("업로드: " + file.filename());
          });
  }

multipartData() (함수형 방식):
  public Mono<ServerResponse> upload(ServerRequest request) {
      return request.multipartData()
          .flatMap(parts -> {
              FilePart file = (FilePart) parts.getFirst("file");
              FormFieldPart name = (FormFieldPart) parts.getFirst("name");

              String fileName = name.value() + "_" + file.filename();
              return file.transferTo(Path.of("/upload/" + fileName))
                  .then(ServerResponse.ok().bodyValue(fileName));
          });
  }

대용량 파일 청크 스트리밍:
  @PostMapping("/upload/stream")
  public Mono<String> uploadStream(
          @RequestPart("file") FilePart filePart) {
      // FilePart.content() = Flux<DataBuffer> (청크 스트림)
      return DataBufferUtils.write(
          filePart.content(),
          Path.of("/upload/" + filePart.filename()),
          StandardOpenOption.CREATE, StandardOpenOption.WRITE
      )
      .thenReturn("스트리밍 업로드 완료");
      // 청크를 메모리에 쌓지 않고 바로 파일에 쓰기
  }
```

### 4. 파일 다운로드 — 스트리밍 응답

```
대용량 파일 다운로드 (메모리 효율적):

@GetMapping("/download/{filename}")
public Mono<Void> download(
        @PathVariable String filename,
        ServerWebExchange exchange) {

    Path filePath = uploadPath.resolve(filename);
    long fileSize;
    try {
        fileSize = Files.size(filePath);
    } catch (IOException e) {
        return Mono.error(new FileNotFoundException(filename));
    }

    ServerHttpResponse response = exchange.getResponse();
    response.getHeaders().setContentType(MediaType.APPLICATION_OCTET_STREAM);
    response.getHeaders().setContentLength(fileSize);
    response.getHeaders().setContentDispositionFormData("attachment", filename);

    // Flux<DataBuffer>로 파일을 청크 단위로 읽어 스트리밍
    Flux<DataBuffer> fileContent = DataBufferUtils.read(
        filePath,
        exchange.getResponse().bufferFactory(),
        4096  // 청크 크기 4KB
    );

    return response.writeWith(fileContent);
    // 전체 파일을 메모리에 올리지 않고 4KB씩 읽어 전송
}

ResponseEntity 방식:
@GetMapping("/download/{filename}")
public ResponseEntity<Flux<DataBuffer>> downloadFile(
        @PathVariable String filename) {
    Flux<DataBuffer> content = DataBufferUtils.read(
        uploadPath.resolve(filename),
        new DefaultDataBufferFactory(),
        4096
    );
    return ResponseEntity.ok()
        .header(HttpHeaders.CONTENT_DISPOSITION,
            "attachment; filename=\"" + filename + "\"")
        .contentType(MediaType.APPLICATION_OCTET_STREAM)
        .body(content);
}
```

### 5. 요청 본문 제한 — maxInMemorySize 설정

```
기본 maxInMemorySize: 256KB
대용량 요청 시 DataBufferLimitException 발생

설정 방법 (Spring Boot):
  application.yml:
    spring:
      codec:
        max-in-memory-size: 10MB  # 전체 코덱 기본값
      webflux:
        multipart:
          max-in-memory-size: 1MB  # multipart 파트당
          max-disk-usage-per-part: 100MB  # 파트당 최대 디스크
          max-parts: 10  # 파트 수 제한

코드 설정:
  @Configuration
  public class WebFluxConfig implements WebFluxConfigurer {
      @Override
      public void configureHttpMessageCodecs(ServerCodecConfigurer configurer) {
          configurer.defaultCodecs()
              .maxInMemorySize(10 * 1024 * 1024);  // 10MB
      }
  }

스트리밍으로 제한 우회:
  @PostMapping(value = "/upload",
               consumes = MediaType.MULTIPART_FORM_DATA_VALUE)
  public Mono<String> upload(@RequestPart("file") FilePart file) {
      // FilePart.transferTo() 또는 FilePart.content()
      // → 청크 스트리밍 → maxInMemorySize 적용 안 됨
      // → 메모리 제한 없이 대용량 처리 가능
      return file.transferTo(dest).thenReturn("OK");
  }
  // 단, bodyToMono(byte[].class)는 전체를 메모리에 → 제한 적용됨
```

---

## 💻 실전 코드

### 실험 1: 이미지 업로드 + 리사이징 파이프라인

```java
@PostMapping(value = "/images",
             consumes = MediaType.MULTIPART_FORM_DATA_VALUE)
public Mono<ImageResponse> uploadImage(
        @RequestPart("image") FilePart imagePart,
        @RequestPart("width") FormFieldPart widthPart,
        @RequestPart("height") FormFieldPart heightPart) {

    int width = Integer.parseInt(widthPart.value());
    int height = Integer.parseInt(heightPart.value());

    return DataBufferUtils.join(imagePart.content())
        .map(buffer -> {
            try {
                byte[] imageBytes = new byte[buffer.readableByteCount()];
                buffer.read(imageBytes);
                return imageBytes;
            } finally {
                DataBufferUtils.release(buffer);  // 해제 필수
            }
        })
        .subscribeOn(Schedulers.boundedElastic())  // 이미지 처리는 블로킹 가능
        .flatMap(bytes -> imageService.resize(bytes, width, height))
        .flatMap(resizedBytes -> storageService.save(resizedBytes, imagePart.filename()))
        .map(url -> new ImageResponse(url));
}
```

### 실험 2: CSV 파일 스트리밍 파싱

```java
@PostMapping(value = "/csv/import",
             consumes = MediaType.MULTIPART_FORM_DATA_VALUE)
public Flux<ImportResult> importCsv(@RequestPart("file") FilePart csvFile) {
    return csvFile.content()  // Flux<DataBuffer>
        .map(buffer -> {
            // DataBuffer → String 변환 (청크 단위)
            String chunk = buffer.toString(StandardCharsets.UTF_8);
            DataBufferUtils.release(buffer);  // 해제
            return chunk;
        })
        .flatMap(chunk -> Flux.fromArray(chunk.split("\n")))
        .skip(1)  // 헤더 행 스킵
        .filter(line -> !line.isBlank())
        .map(line -> line.split(","))
        .flatMap(
            fields -> importService.processRecord(fields),
            10  // 동시 10개 처리
        );
}
```

### 실험 3: 파일 프록시 (외부 → 클라이언트 스트리밍)

```java
@GetMapping("/proxy/file")
public Mono<Void> proxyFile(
        @RequestParam String fileUrl,
        ServerWebExchange exchange) {

    return webClient.get()
        .uri(fileUrl)
        .retrieve()
        .bodyToFlux(DataBuffer.class)
        .as(dataBuffers -> {
            ServerHttpResponse response = exchange.getResponse();
            // 외부 파일을 메모리에 올리지 않고 바로 클라이언트에 전달
            return response.writeWith(dataBuffers);
        });
}
```

---

## 📊 성능 비교

```
파일 업로드 방식별 메모리 사용:

방식                        | 1GB 파일 처리 시 메모리  | OOM 위험
───────────────────────────┼──────────────────────┼─────────
@RequestBody byte[]         | ~1GB                 | 높음
DataBufferUtils.join()      | ~1GB                 | 높음
FilePart.transferTo()       | ~수 MB (청크 크기)     | 낮음
Flux<DataBuffer> 스트리밍   | ~수 KB~MB (청크 기반)  | 없음

DataBuffer release 효과:
  1만 요청/시간, 각 요청 10개 DataBuffer, 각 4KB
  release 안 함: 1만 × 10 × 4KB = 400MB 누수/시간
  release 정상: 메모리 풀 재사용 → 증가 없음

스트리밍 다운로드 (4KB 청크):
  maxMemory: 4KB × EventLoop 스레드 수 × 동시 다운로드 수
  10 스레드 × 100 동시: 10 × 100 × 4KB = 4MB (매우 효율적)
  vs 전체 파일 메모리 로드: 100 × 파일 크기
```

---

## ⚖️ 트레이드오프

```
스트리밍 처리 트레이드오프:

청크 스트리밍:
  장점: 메모리 효율, OOM 위험 없음
  단점: 청크 경계에서 데이터 분할 가능 (UTF-8 문자 분할 등)
  해결: DataBufferUtils.join()으로 특정 크기까지 합산

DataBufferUtils.join():
  장점: 전체 데이터를 한 번에 처리 (파싱 용이)
  단점: 전체 데이터가 메모리에 → 대용량 시 OOM 위험
  → maxInMemorySize 제한 적용됨

FilePart.transferTo() vs content():
  transferTo(): 단순 파일 저장에 최적화 (내부적으로 스트리밍)
  content(): 데이터 가공 필요 시 (청크 수신 후 처리)

DataBuffer 해제 책임:
  Reactor 파이프라인에서 자동 해제 되는 경우:
    bodyToMono(), bodyToFlux() — 내부에서 자동 처리
  수동 해제 필요한 경우:
    직접 getBody() 또는 content()로 DataBuffer 구독 시
    → 항상 release() 필수
```

---

## 📌 핵심 정리

```
요청/응답 처리 핵심:

ServerRequest vs ServerHttpRequest:
  ServerRequest: 함수형 엔드포인트, 상위 추상화
  ServerHttpRequest: 어노테이션 방식, 하위 추상화

DataBuffer 관리:
  Netty 메모리 풀 기반 → release() 필수
  누락 시 점진적 메모리 누수
  안전 패턴: try-finally로 항상 release()

파일 업로드:
  소규모: @RequestPart FilePart → transferTo()
  대용량: FilePart.content() → Flux<DataBuffer> 스트리밍
  직접 DataBuffer: join() 후 release() 필수

파일 다운로드:
  DataBufferUtils.read(path, factory, chunkSize)
  → Flux<DataBuffer> → response.writeWith()
  전체를 메모리에 올리지 않고 청크 단위 스트리밍

maxInMemorySize:
  bodyToMono(byte[].class): 제한 적용
  FilePart.transferTo() / content(): 제한 비적용
```

---

## 🤔 생각해볼 문제

**Q1.** `DataBufferUtils.release()`를 `try-finally`가 아닌 `doFinally()`에서 호출해도 안전한가요?

<details>
<summary>해설 보기</summary>

`doFinally()`는 Reactive 스트림이 완료(`onComplete`), 에러(`onError`), 취소(`cancel`) 중 어떤 방식으로 종료되어도 실행됩니다. 안전하게 사용할 수 있습니다.

```java
request.getBody()
    .flatMap(buffer ->
        Mono.fromCallable(() -> processBuffer(buffer))
            .subscribeOn(Schedulers.boundedElastic())
            .doFinally(signal -> DataBufferUtils.release(buffer))
    );
```

단, `flatMap` 내부에서 예외가 발생해도 `doFinally`가 실행되므로 안전합니다.

`try-finally`와 `doFinally`의 차이:
- `try-finally`: 동기 코드에서 즉시 실행 보장
- `doFinally`: Reactive 파이프라인의 생명주기에 맞춰 실행

비동기 처리 시에는 `doFinally`가 더 적합합니다. 동기 처리라면 `try-finally`도 가능합니다.

</details>

---

**Q2.** 클라이언트가 파일 다운로드 중 연결을 끊으면 서버에서 어떻게 처리되나요?

<details>
<summary>해설 보기</summary>

클라이언트가 연결을 끊으면:
1. Netty가 TCP RST 또는 FIN을 감지
2. `response.writeWith(fileContentFlux)`의 쓰기 작업 실패
3. `Flux<DataBuffer>` 파이프라인에 `cancel` 신호 전파
4. `DataBufferUtils.read()`가 파일 읽기를 중단

이 과정은 Reactor의 Backpressure와 취소 메커니즘으로 자동 처리됩니다. `doOnCancel()`로 추가 정리 로직을 넣을 수 있습니다.

```java
Flux<DataBuffer> fileContent = DataBufferUtils.read(
    filePath, factory, 4096
)
.doOnCancel(() ->
    log.info("파일 다운로드 클라이언트가 연결 종료: {}", filePath)
);
```

핵심은 **자동으로 정리**된다는 점입니다. 파일 디스크립터, 버퍼 등이 cancel 신호를 받으면 Netty가 정리합니다.

</details>

---

**Q3.** `FilePart.transferTo(Path)`는 내부적으로 어떻게 동작하나요? 블로킹인가요?

<details>
<summary>해설 보기</summary>

`FilePart.transferTo(Path)`는 `Mono<Void>`를 반환하는 **논블로킹 작업**입니다. 내부적으로 청크(DataBuffer)를 받아 `AsynchronousFileChannel`로 비동기 파일 쓰기를 수행합니다.

```
내부 동작:
  FilePart.content()  →  Flux<DataBuffer>
    각 DataBuffer:
      AsynchronousFileChannel.write(ByteBuffer, position, ...)
      → Java NIO 비동기 파일 쓰기 (OS AIO 또는 스레드 풀)
      → 완료 콜백 → Mono 신호
```

단, `File` 객체를 인자로 받는 deprecated `transferTo(File)` 버전은 내부적으로 `Files.copy()`를 사용할 수 있어 블로킹일 수 있습니다. `Path` 버전을 사용하는 것이 권장됩니다.

실제로 `AsynchronousFileChannel`은 OS에 따라 커널 AIO(Linux io_uring) 또는 스레드 풀을 사용할 수 있지만, 어느 쪽이든 호출 스레드(EventLoop)는 블로킹되지 않습니다.

</details>

---

<div align="center">

**[⬅️ 이전: SSE와 WebSocket](./05-streaming-sse-websocket.md)** | **[홈으로 🏠](../README.md)** | **[다음: WebFilter ➡️](./07-web-filter.md)**

</div>
