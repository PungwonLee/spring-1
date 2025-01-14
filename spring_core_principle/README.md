# 출처
- url: https://www.inflearn.com/course/%EC%8A%A4%ED%94%84%EB%A7%81-%ED%95%B5%EC%8B%AC-%EC%9B%90%EB%A6%AC-%EA%B3%A0%EA%B8%89%ED%8E%B8
- 강사: 김영한님

# [1. 예제 만들기](./1.example-make)

<details> <summary> 1. 프로젝트 생성 </summary>

## 1. 프로젝트 생성

[https://start.spring.io/](https://start.spring.io/)

</details>


<details> <summary> 2. 예제 프로젝트 만들기 - V0 </summary>

## 2. 예제 프로젝트 만들기 - V0

- 학습을 위한 간단한 예제 프로젝트를 만들어보자.
- 상품을 주문하는 프로세스로 가정하고, 일반적인 웹 애플리케이션에서 Controller -> Service -> Repository로 이어지는 흐름을 최대한 단순하게 만들어보자.

**OrderRepositoryV0**
```java
package hello.advanced.app.v0;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Repository;
@Repository
@RequiredArgsConstructor
public class OrderRepositoryV0 {
 public void save(String itemId) {
 //저장 로직
 if (itemId.equals("ex")) {
 throw new IllegalStateException("예외 발생!");
 }
 sleep(1000);
 }
 private void sleep(int millis) {
 try {
 Thread.sleep(millis);
 } catch (InterruptedException e) {
 e.printStackTrace();
 }
 }
}
```
- `@Repository`: 컴포넌트 스캔의 대상이 된다. 따라서 스프링 빈으로 자동 등록된다. 
- `sleep(1000)`: 리포지토리는 상품을 저장하는데 약 1초 정도 걸리는 것으로 가정하기 위해 1초 지연을 주었다 (1000ms)
- 예외가 발생하는 상황도 확인하기 위해 파라미터(`itemId`)의 값이 `ex`로 넘어오면 `IllegalStateException` 예외가 발생하도록 했다.

**OrderServiceV0**
```java
package hello.advanced.app.v0;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;
@Service
@RequiredArgsConstructor
public class OrderServiceV0 {
 private final OrderRepositoryV0 orderRepository;
 public void orderItem(String itemId) {
 orderRepository.save(itemId);
 }
}
```
- `@Service`: 컴포넌트 스캔의 대상이 된다.
- 실무에서는 복잡한 비즈니스 로직이 서비스 계층에 포함되지만, 애졔에서는 단순함을 위해서 리포지토리에 저장을 호출하는 코드만 있다.

**OrderControllerV0**
```java
package hello.advanced.app.v0;
import lombok.RequiredArgsConstructor;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;
@RestController
@RequiredArgsConstructor
public class OrderControllerV0 {
 private final OrderServiceV0 orderService;
 @GetMapping("/v0/request")
 public String request(String itemId) {
 orderService.orderItem(itemId);
 return "ok";
 }
}
```
- `@RestController`: 컴포넌트 스캔과 스프링 Rest 컨트롤러로 인식된다.
- `/v0/request` 메서드는 HTTP 파라미터로 `itemId`를 받을 수 있다.


- 실행: http://localhost:8080/v0/request?itemId=hello
- 결과: `ok`

실무에서 일반적으로 사용하는 컨트롤러 -> 서비스 -> 리포지토리의 기본 흐름을 만들었다.
지금부터 이 흐름을 기반으로 예제를 점진적으로 발전시켜 나가면서 학습을 진행하겠다.

</details>

<details> <summary> 3. 로그 추적기 - 요구사항 분석 </summary>

## 3. 로그 추적기 - 요구사항 분석

새로운 회사에 입사 했는데, 수 년간 운영중인 거대한 프로젝트에 투입되었다. 전체 소스 코드는 수십만 라인이고, 클래스 수도 수 백개 이상이다.

우리가 처음 맡겨진 요구사항은 로그 추적기를 만드는 것이다.

애플리케이션이 커지면서 점점 모니터링과 운영이 중요해지는 단계이다. 특히 최근 자주 병목이 발생하고 있다.
어떤 부분에서 병목이 발생하는지, 그리고 어떤 부분에서 예외가 발생하는지를 로그를 통해 확인하는 것이 점점 중요해지고 있다.

기존에는 개발자가 문제가 발생한 다음에 관련 부분을 어렵게 찾아서 로그를 하나하나 직접 만들어서 남겼다.

로그를 미리 남겨둔다면 이런 부분을 손쉽게 찾을 수 있을 것이다. 

이부분을 개선하고 자동화하는 것이 우리의 미션이다.

**요구사항**
- 모든 PUBLIC 메서드의 호출과 응답 정보를 로그로 출력
- 애플리케이션의 흐름을 변경하면 안됨
  - 로그를 남긴다고 해서 비즈니스 로직의 동작에 영향을 주면 안됨
- 메서드 호출에 걸린 시간
- 정상 흐름과 예외 흐름 구분
  - 예외 발생시 예외 정보가 남아야 함
- 메서드 호출의 깊이 표현
- HTTP 요청을 구분
  - HTTP 요청 단위로 특정 ID를 남겨서 어떤 HTTP 요청에서 시작된 것인지 명확하게 구분이 가능해야 함
  - 트랜잭션 ID (DB 트랜잭션X), 여기서는 하나의 HTTP 요청이 시작해서 끝날 때 까지를 하나의 트랜잭션이라 함

**예시**
```
정상 요청
[796bccd9] OrderController.request()
[796bccd9] |-->OrderService.orderItem()
[796bccd9] | |-->OrderRepository.save()
[796bccd9] | |<--OrderRepository.save() time=1004ms
[796bccd9] |<--OrderService.orderItem() time=1014ms
[796bccd9] OrderController.request() time=1016ms

예외 발생
[b7119f27] OrderController.request()
[b7119f27] |-->OrderService.orderItem()
[b7119f27] | |-->OrderRepository.save()
[b7119f27] | |<X-OrderRepository.save() time=0ms
ex=java.lang.IllegalStateException: 예외 발생!
[b7119f27] |<X-OrderService.orderItem() time=10ms
ex=java.lang.IllegalStateException: 예외 발생!
[b7119f27] OrderController.request() time=11ms
ex=java.lang.IllegalStateException: 예외 발생!
``` 

**참고**
> 모니터링 툴을 도입하면 많은 부분이 해결되지만, 지금은 학습이 목적이라는 사실을 기억하자.

</details>

<details> <summary> 4. 로그 추적기V1 - 프로토타입 개발 </summary>

## 4. 로그 추적기V1 - 프로토타입 개발

애플리케이션의 모든 로직에 직접 로그를 남겨도 되지만, 그것보다는 더 효율적인 개발 방법이 필요하다.

특히 트랜잭션ID와 깊이를 표현하는 방법은 기존 정보를 이어 받아야 하기 떄문에 단순히 로그만 남긴다고 해결할 수 있는 것은 아니다.

요구사항에 맞추어 애플리케이션에 효과적으로 로그를 남기기 위한 로그 추적기를 개발해보자.

먼저 프로토타입 버전을 개발해보자. 아마 코드를 모두 작성하고 테스트 코드까지 실행해보아야 어떤 것을 하는지 감이 올 것이다.

먼저 로그 추적기를 위한 기반 데이터를 가지고 있는 `TraceId`, `TraceStatus` 클래스를 만들어보자. 

**TraceId**
```java
package hello.advanced.trace;
import java.util.UUID;
public class TraceId {
 private String id;
 private int level;
 public TraceId() {
 this.id = createId();
 this.level = 0;
 }
 private TraceId(String id, int level) {
 this.id = id;
 this.level = level;
 }
 private String createId() {
 return UUID.randomUUID().toString().substring(0, 8);
 }
 public TraceId createNextId() {
 return new TraceId(id, level + 1);
 }
 public TraceId createPreviousId() {
 return new TraceId(id, level - 1);
 }
 public boolean isFirstLevel() {
 return level == 0;
 }
 public String getId() {
 return id;
 }
 public int getLevel() {
 return level;
 }
}
```

**TraceId 클래스**
- 로그 추적기는 트랜잭션 ID와 깊이를 표현하는 방법이 필요하다.
- 여기서는 트랜잭션ID와 깊이를 표현하는 level을 묶어서 `TraceId`라는 개념을 만들었다.
- `TraceId`는 단순히 `id`(트랜잭션ID)와 `level`정보를 함께 가지고 있다.

```
[796bccd9] OrderController.request() //트랜잭션ID:796bccd9, level:0
[796bccd9] |-->OrderService.orderItem() //트랜잭션ID:796bccd9, level:1
[796bccd9] | |-->OrderRepository.save()//트랜잭션ID:796bccd9, level:2
```

**UUID**
- `TraceId`를 처음 생성하면 `createId()`를 사용해서 UUID를 만들어낸다.
- UUID가 너무 길어서 여기서는 앞 8자리만 사용한다.
- 이 정도면 로그를 충분히 구분할 수 있다.
- 여기서는 이렇게 만들어진 값을 트랜잭션ID로 사용한다.
```
ab99e16f-3cde-4d24-8241-256108c203a2 //생성된 UUID
ab99e16f //앞 8자리만 사용
``` 

**createNextId()**
- 다음 `TraceId`를 만든다. 
- 예제 로그를 잘 보면 깊이가 증가해도 트랜잭션ID는 같다.
- 대신에 깊이가 하나 증가한다.
- 실행 코드: `new TraceId(id, level + 1)`
```
[796bccd9] OrderController.request()
[796bccd9] |-->OrderService.orderItem() //트랜잭션ID가 같다. 깊이는 하나 증가한다
``` 
- 따라서 `createNextId()`를 사용해서 현재 `TraceId`를 기반으로 다음 `TraceId`를 만들면 `id`는 기존과 같고, `level`은 하나 증가한다.

**createPreviousId()**
- `createNextId()`의 반대 역할을 한다.
- `id`는 기존과 같고, `level`은 하나 감소한다.

**isFirstLevel()**
- 첫 번째 레벨 여부를 편리하게 확인할 수 있는 메서드

**TraceStatus**
```java
package hello.advanced.trace;
public class TraceStatus {
 private TraceId traceId;
 private Long startTimeMs;
 private String message;
 public TraceStatus(TraceId traceId, Long startTimeMs, String message) {
 this.traceId = traceId;
 this.startTimeMs = startTimeMs;
 this.message = message;
 }
 public Long getStartTimeMs() {
 return startTimeMs;
 }
 public String getMessage() {
 return message;
 }
 public TraceId getTraceId() {
 return traceId;
 }
}
```

**TraceStatus 클래스**: 로그의 상태 정보를 나타낸다.
- 로그를 시작하면 끝이 있어야 한다.
```
[796bccd9] OrderController.request() //로그 시작
[796bccd9] OrderController.request() time=1016ms //로그 종료
``` 
- `TraceStatus`는 로그를 시작할 때의 상태 정보를 가지고 있다. 
- 이 상태 정보는 로그를 종료할 때 사용 된다.
- `traceId`: 내부에 트랜잭션 ID와 level을 가지고 있다. 
- `startTimeMs`: 로그 시작시간이다. 로그 종료시 이 시작 시간을 기준으로 시작~종료까지 전체 수행 시간을 구할 수 있다.
- `message`: 시작 시 사용한 메시지이다. 이후 로그 종료시에도 이 메시지를 사용해서 출력한다.


`TraceId`,`TraceStatus`를 사용해서 실제 로그를 생성하고, 처리하는 기능을 개발해보자

**HelloTraceV1**
```java
package hello.advanced.trace.hellotrace;
import hello.advanced.trace.TraceId;
import hello.advanced.trace.TraceStatus;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Component;
@Slf4j
@Component
public class HelloTraceV1 {
 private static final String START_PREFIX = "-->";
 private static final String COMPLETE_PREFIX = "<--";
 private static final String EX_PREFIX = "<X-";
 public TraceStatus begin(String message) {
 TraceId traceId = new TraceId();
 Long startTimeMs = System.currentTimeMillis();
 log.info("[{}] {}{}", traceId.getId(), addSpace(START_PREFIX,
traceId.getLevel()), message);
 return new TraceStatus(traceId, startTimeMs, message);
 }
 public void end(TraceStatus status) {
 complete(status, null);
 }
 public void exception(TraceStatus status, Exception e) {
 complete(status, e);
 }
 private void complete(TraceStatus status, Exception e) {
 Long stopTimeMs = System.currentTimeMillis();
 long resultTimeMs = stopTimeMs - status.getStartTimeMs();
 TraceId traceId = status.getTraceId();
 if (e == null) {
 log.info("[{}] {}{} time={}ms", traceId.getId(),
addSpace(COMPLETE_PREFIX, traceId.getLevel()), status.getMessage(),
resultTimeMs);
 } else {
 log.info("[{}] {}{} time={}ms ex={}", traceId.getId(),
addSpace(EX_PREFIX, traceId.getLevel()), status.getMessage(), resultTimeMs,
e.toString());
 }
 }
 private static String addSpace(String prefix, int level) {
 StringBuilder sb = new StringBuilder();
 for (int i = 0; i < level; i++) {
 sb.append( (i == level - 1) ? "|" + prefix : "| ");
 }
 return sb.toString();
 }
}
```
- `HelloTraceV1`을 사용해서 실제 로그를 시작하고 종료할 수 있다. 그리고 로그를 출력하고 실행시간도 측정할 수 있다.
- `@Component`: 싱글톤으로 사용하기 위해 스프링 빈으로 등록한다. 컴포넌트 스캔의 대상이 된다.

**공개 메서드**

로그 추적기에서 사용되는 공개 메서드는 다음 3가지이다.
- `begin(..)`
- `end(..)`
- `exception(..)`

**하나씩 자세히 알아보자**
- `TraceStatus begin(String message)`
  - 로그를 시작한다.
  - 로그 메시지를 파라미터로 받아서 시작 로그를 출력한다.
  - 응답 결과로 현재 로그의 상태인 `TRaceStatus`를 반환한다.
- `void end(TraceStatus status)`
  - 로그를 정상 종료한다.
  - 파라미터로 시작 로그의 상태(`TraceStatus`)를 전달 받는다. 이 값을 활용해서 실행 시간을 계산하고, 종료시에도 시작할 때와 동일한 로그 메시지를 출력할 수 있다.
  - 정상 흐름에서 호출한다.
- `void exception(TraceStatus status, Exception e)`
  - 로그를 예외 상황으로 종료한다.
  - `TraceStatus`, `Exception` 정보를 함께 전달 받아서 실행시간, 예외 정보를 포함한 결과 로그를 출력한다.
  - 예외가 발생했을 때 호출한다.

**비공개 메서드**
- `complete(TraceStatus status, Exception e)`
  - `end()`, `exception()` 의 요청 흐름을 한곳에서 편리하게 처리한다. 실행 시간을 측정하고 로그를 남긴다.
- `String addSpace(String prefix, int level)`: 다음과 같은 결과를 출력한다.
  - prefix: `-->`
    - level 0:
    - level 1: `-->`
    - level 2: `| |-->`
  - prefix: `<--`
    - level 0:
    - level 1: `|<--`
    - level 2: `| |<---`
  - prefix: `<X-`
    - level 0:
    - level 1: `|<X-`
    - level 2: `| |<X`
참고로 `HelloTraceV1`는 아직 모든 요구사항을 만족하지 못한다. 이후에 기능을 하나씩 추가할 예정이다.

테스트 작성

**HelloTraceV1Test**

주의! 테스트 코드는 `test/java/` 하위에 위치함
```java
package hello.advanced.trace.hellotrace;
import hello.advanced.trace.TraceStatus;
import org.junit.jupiter.api.Test;
class HelloTraceV1Test {
 @Test
 void begin_end() {
 HelloTraceV1 trace = new HelloTraceV1();
 TraceStatus status = trace.begin("hello");
 trace.end(status);
 }
 @Test
 void begin_exception() {
 HelloTraceV1 trace = new HelloTraceV1();
 TraceStatus status = trace.begin("hello");
 trace.exception(status, new IllegalStateException());
 }
}
```

테스트 코드를 보면 로그 추적기를 어떻게 실행해야 하는지, 그리고 어떻게 동작하는지 이해가 될 것이다.

**begin_end() - 실행 로그**
```
[41bbb3b7] hello
[41bbb3b7] hello time=5ms
```

**begin_exception() - 실행 로그**
```
[898a3def] hello
[898a3def] hello time=13ms ex=java.lang.IllegalStateException
```

이제 실제 애플리케이션에 적용해보자.

> 참고: 이것은 온전한 테스트 코드가 아니다. 일반적으로 테스트라고 하면 자동으로 검증하는 과정이
> 필요하다. 이 테스트는 검증하는 과정이 없고 결과를 콘솔로 직접 확인해야 한다. 이렇게 응답값이 없는
> 경우를 자동으로 검증하려면 여러가지 테스트 기법이 필요하다. 이번 강의에서는 예제를 최대한 단순화
> 하기 위해 검증 테스트를 생략했다.

> 주의: 지금까지 만든 로그 추적기가 어떻게 동작하는지 확실히 이해해야 다음 단계로 넘어갈 수 있다.
> 복습과 코드를 직접 만들어보면서 확실하게 본인 것으로 만들고 다음으로 넘어가자

</details>

<details> <summary> 5. 로그 추적기V1 - 적용 </summary>

## 5. 로그 추적기V1 - 적용 

이제 애플리케이션에 우리가 개발한 로그 추적기를 적용해보자.

기존 `v0` 패키지에 코드를 직접 작성해도 되지만, 기존 코드를 유지하고, 비교하기 위해서 `v1` 패키지를 새로 만들고 기존 코드를 복사하자. 복사하는 과정은 다음을 참고하자.

### V0 -> V1 복사 
- `hello.advanced.app.v1` 패키지 생성
- 복사
  - `v0.OrderRepositoryV0` -> `v1.OrderRepositoryV1`
  - `v0.OrderServiceV0` -> `v1.OrderServiceV1`
  - `v0.OrderControllerV0` -> `v1.OrderControllerV1`
- 코드 내부 의존관계 클래스를 V1으로 변경
  - `OrderControllerV1`: `OrderServiceV0` -> `OrderServiceV1`
  - `OrderServiceV1`: `OrderRepositoryV0` -> `OrderRepositoryV1`
- `OrderControllerV1` 매핑 정보 변경
  - `@GetMapping("/v1/request")`

실행해서 정상 동작하는지 확인하자.
- 실행: http://localhost:8080/v1/request?itemId=hello
- 결과: `ok`

### V1 적용하기
`OrderControllerV1`, `OrderServiceV1`, `OrderRepositoryV1`에 로그 추적기를 적용해보자.

먼저 컨트롤러에 우리가 개발한 `HelloTraceV1`을 적용해보자.

**OrderControllerV1**
```java
package hello.advanced.app.v1;
import hello.advanced.trace.TraceStatus;
import hello.advanced.trace.hellotrace.HelloTraceV1;
import lombok.RequiredArgsConstructor;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;
@RestController
@RequiredArgsConstructor
public class OrderControllerV1 {
 private final OrderServiceV1 orderService;
 private final HelloTraceV1 trace;
 @GetMapping("/v1/request")
 public String request(String itemId) {
 TraceStatus status = null;
 try {
 status = trace.begin("OrderController.request()");
 orderService.orderItem(itemId);
 trace.end(status);
 return "ok";
 } catch (Exception e) {
 trace.exception(status, e);
 throw e; //예외를 꼭 다시 던져주어야 한다.
 }
 }
}
```
- `HelloTraceV1 trace`: `HelloTraceV1`을 주입 받는다. 참고로 `HelloTraceV1`은 `@Component` 애노테이션을 가지고 있기 때문에 컴포넌트 스캔의 대상이 된다. 따라서 자동으로 스프링 빈으로 등록된다.
- `trace.begin("OrderController.request()")`: 로그를 시작할 때 메시지 이름으로 컨트롤러 이름 + 메서드 이름을 주었다. 이렇게 하면 어떤 컨트롤러와 메서드가 호출되었는지 로그로 편리하게 확인할 수 있다. 물론 수작업이다.
- 단순하게 `trace.begin()`, `trace.end()` 코드 두 줄만 적용하면 될 줄 알았지만, 실상은 그렇지 않다. `trace.exception()`으로 예외까지 처리해야 하므로 지저분한 `try`, `catch` 코드가 추가된다.
- `begin()`의 결과 값으로 받은 `TraceStatus status` 값을 `end()`, `exception()`에 넘겨야 한다. 결국 `try`, `catch` 블록 모두에 이 값을 넘겨야 한다. 따라서 `try` 상위에 `TraceStatus status`코드를 선언해야 한다. 만약 `try`안에서 `TraceStatus status`를 선언하면 `try` 블록안에서만 해당 변수가 유효하기 때문에 `catch`블록에 넘길 수 없다. 따라서 컴파일 오류가 발생한다.
- `throw e`: 예외를 꼭 다시 던져주어야 한다. 그렇지 않으면 여기서 예외를 먹어버리고, 이후에 정상 흐름으로 동작한다. 로그는 애플리케이션에 흐름에 영향을 주면 안된다. 로그 때문에 예외가 사라지면 안된다.

실행
- 정상: http://localhost:8080/v1/request?itemId=hello
- 예외: http://localhost:8080/v1/request?itemId=ex

실행해보면 정상 흐름과 예외 모두 로그로 잘 출력되는 것을 확인할 수 있다. 나머지 부분도 완성하자.

**OrderServiceV1**
```java
package hello.advanced.app.v1;
import hello.advanced.trace.TraceStatus;
import hello.advanced.trace.hellotrace.HelloTraceV1;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;
@Service
@RequiredArgsConstructor
public class OrderServiceV1 {
 private final OrderRepositoryV1 orderRepository;
 private final HelloTraceV1 trace;
 public void orderItem(String itemId) {
 TraceStatus status = null;
 try {
 status = trace.begin("OrderService.orderItem()");
 orderRepository.save(itemId);
 trace.end(status);
 } catch (Exception e) {
 trace.exception(status, e);
 throw e;
 }
 }
}
```

**OrderRepositoryV1**
```java
package hello.advanced.app.v1;
import hello.advanced.trace.TraceStatus;
import hello.advanced.trace.hellotrace.HelloTraceV1;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Repository;
@Repository
@RequiredArgsConstructor
public class OrderRepositoryV1 {
 private final HelloTraceV1 trace;
 public void save(String itemId) {
 TraceStatus status = null;
 try {
 status = trace.begin("OrderRepository.save()");
 //저장 로직
 if (itemId.equals("ex")) {
 throw new IllegalStateException("예외 발생!");
 }
 sleep(1000);
 trace.end(status);
 } catch (Exception e) {
 trace.exception(status, e);
 throw e;
 }
 }
 private void sleep(int millis) {
 try {
 Thread.sleep(millis);
 } catch (InterruptedException e) {
 e.printStackTrace();
 }
 }
}
```

정상 실행
- http://localhost:8080/v1/request?itemId=hello


**정상 실행 로그**
```
[11111111] OrderController.request()
[22222222] OrderService.orderItem()
[33333333] OrderRepository.save()
[33333333] OrderRepository.save() time=1000ms
[22222222] OrderService.orderItem() time=1001ms
[11111111] OrderController.request() time=1001ms
```
![image](https://user-images.githubusercontent.com/28394879/140095299-02d7c3f7-0a51-46f6-9cc3-dd9e7c7062ad.png)

> 참고: 아직 level 관련 기능을 개발하지 않았다. 
> 따라서 level값은 항상 0이다.
> 그리고 트랜잭션ID 값도 다르다. 
> 이부분은 아직 개발하지 않았다.


**예외 실행**
- http://localhost:8080/v1/request?itemId=ex

**예외 실행 로그**
```
[5e110a14] OrderController.request()
[6bc1dcd2] OrderService.orderItem()
[48ddffd6] OrderRepository.save()
[48ddffd6] OrderRepository.save() time=0ms ex=java.lang.IllegalStateException:
예외 발생!
[6bc1dcd2] OrderService.orderItem() time=6ms
ex=java.lang.IllegalStateException: 예외 발생!
[5e110a14] OrderController.request() time=7ms
ex=java.lang.IllegalStateException: 예외 발생!
```

`HelloTraceV1` 덕분에 직접 로그를 하나하나 남기는 것 보다는 편하게 여러가지 로그를 남길 수 있었다.
하지만 로그를 남기기 위한 코드가 생각보다 너무 복잡하다. 지금은 우선 요구사항과 동작하는 것에만 집중하자.


### 남은 문제
**요구사항**
- ~~모든 PUBLIC 메서드의 호출과 응답 정보를 로그로 출력~~
- ~~애플리케이션의 흐름을 변경하면 안됨~~
  - ~~로그를 남긴다고 해서 비즈니스 로직의 동작에 영향을 주면 안됨~~
- ~~메서드 호출에 걸린 시간~~
- ~~정상 흐름과 예외 흐름 구분~~
  - ~~예외 발생시 예외 정보가 남아야 함~~
- 메서드 호출의 깊이 표현
- HTTP 요청을 구분
  - HTTP 요청 단위로 특정 ID를 남겨서 어떤 HTTP 요청에서 시작된 것인지 명확하게 구분이 가능해야 함
  - 트랜잭션 ID (DB 트랜잭션 말고..)
  
아직 구현하지 못한 요구사항은 메서드 호출의 깊이를 표현하고, 같은 HTTP 요청이면 같은 트랜잭션 ID를 남기는 것이다.

이 기능은 직전 로그의 깊이와 트랜잭션 ID가 무엇인지 알아야 할 수 있는 일이다.

예를 들어서 `OrderController.request()`에서 로그를 남길 때 어떤 깊이와 어떤 트랜잭션 ID를 사용했는지를 그 다음에 로그를 남기는 `OrderService.orderItem()`에서 로그를 남길 때 알아야 한다.
결국 현재 로그의 상태 정보인 `트랜잭션ID`와 `level`이 다음으로 전달되어야 한다.
정리하면 로그에 대한 문액(`Context`) 정보가 필요하다.



</details>

<details> <summary> 6. 로그 추적기V2 - 파라미터로 동기화 개발 </summary>

## 6. 로그 추적기V2 - 파라미터로 동기화 개발

트랜잭션ID와 메서드 호출의 깊이를 표현하는 가장 단순한 방법은 첫 로그에서 사용한 `트랜잭션ID`와 `level`을 다음 로그에 넘겨주면 된다.

현재 로그의 상태 정보인 `트랜잭션ID`와 `level`은 `TraceId`에 포함되어 있다. 따라서 `TraceId`를 다음 로그에 넘겨주면 된다. 이 기능을 추가한 `HelloTraceV2`를 개발해보자. 

**HelloTraceV2**
```java
package hello.advanced.trace.hellotrace;
import hello.advanced.trace.TraceId;
import hello.advanced.trace.TraceStatus;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Component;
@Slf4j
@Component
public class HelloTraceV2 {
 private static final String START_PREFIX = "-->";
 private static final String COMPLETE_PREFIX = "<--";
 private static final String EX_PREFIX = "<X-";
 public TraceStatus begin(String message) {
 TraceId traceId = new TraceId();
 Long startTimeMs = System.currentTimeMillis();
 log.info("[" + traceId.getId() + "] " + addSpace(START_PREFIX,
traceId.getLevel()) + message);
 return new TraceStatus(traceId, startTimeMs, message);
 }
 //V2에서 추가
 public TraceStatus beginSync(TraceId beforeTraceId, String message) {
 TraceId nextId = beforeTraceId.createNextId();
 Long startTimeMs = System.currentTimeMillis();
 log.info("[" + nextId.getId() + "] " + addSpace(START_PREFIX,
nextId.getLevel()) + message);
 return new TraceStatus(nextId, startTimeMs, message);
 }
 public void end(TraceStatus status) {
 complete(status, null);
 }
 public void exception(TraceStatus status, Exception e) {
 complete(status, e);
 }
 private void complete(TraceStatus status, Exception e) {
 Long stopTimeMs = System.currentTimeMillis();
 long resultTimeMs = stopTimeMs - status.getStartTimeMs();
 TraceId traceId = status.getTraceId();
 if (e == null) {
 log.info("[" + traceId.getId() + "] " + addSpace(COMPLETE_PREFIX,
traceId.getLevel()) + status.getMessage() + " time=" + resultTimeMs + "ms");
 } else {
 log.info("[" + traceId.getId() + "] " + addSpace(EX_PREFIX,
traceId.getLevel()) + status.getMessage() + " time=" + resultTimeMs + "ms" + "
ex=" + e);
 }
 }
 private static String addSpace(String prefix, int level) {
 StringBuilder sb = new StringBuilder();
 for (int i = 0; i < level; i++) {
 sb.append( (i == level - 1) ? "|" + prefix : "| ");
 }
 return sb.toString();
 }
}
```

`HelloTraceV2`는 기존 코드인 `HelloTraceV2`와 같고, `beginSync(..)`가 추가되었다.

```java
//V2에서 추가
public TraceStatus beginSync(TraceId beforeTraceId, String message) {
 TraceId nextId = beforeTraceId.createNextId();
 Long startTimeMs = System.currentTimeMillis();
 log.info("[" + nextId.getId() + "] " + addSpace(START_PREFIX,
nextId.getLevel()) + message);
 return new TraceStatus(nextId, startTimeMs, message);
}
```

**beginSync(..)**
- 기존 `TraceId`에서 `createNextId()`를 통해 다음 ID를 구한다.
- `createNextId()`의 `TraceId` 생성 로직은 다음과 같다.
  - 트랜잭션ID는 기존과 같이 유지한다.
  - 깊이를 표현하는 Level은 하나 증가한다. (`0 -> 1`)

테스트 코드를 통해 잘 동작하는지 확인해보자.

**HelloTraceV2Test**
```java
package hello.advanced.trace.hellotrace;
import hello.advanced.trace.TraceStatus;
import org.junit.jupiter.api.Test;
class HelloTraceV2Test {
 @Test
 void begin_end_level2() {
 HelloTraceV2 trace = new HelloTraceV2();
 TraceStatus status1 = trace.begin("hello1");
 TraceStatus status2 = trace.beginSync(status1.getTraceId(), "hello2");
 trace.end(status2);
 trace.end(status1);
 }
 @Test
 void begin_exception_level2() {
 HelloTraceV2 trace = new HelloTraceV2();
 TraceStatus status1 = trace.begin("hello");
 TraceStatus status2 = trace.beginSync(status1.getTraceId(), "hello2");
 trace.exception(status2, new IllegalStateException());
 trace.exception(status1, new IllegalStateException());
 }
}
```

처음에는 `begin(..)`을 사용하고, 이후에는 `beginSync(..)`를 사용하면 된다. `beginSync(..)`를 호출할 때 직전 로그의 `traceId` 정보를 넘겨주어야 한다.

**begin_end_level2() - 실행 로그**
```
[0314baf6] hello1
[0314baf6] |-->hello2
[0314baf6] |<--hello2 time=2ms
[0314baf6] hello1 time=25ms
```

**begin_exception_level2() - 실행 로그**
```
[37ccb357] hello
[37ccb357] |-->hello2
[37ccb357] |<X-hello2 time=2ms ex=java.lang.IllegalStateException
[37ccb357] hello time=25ms ex=java.lang.IllegalStateException
```

실행 로그를 보면 `트랜잭션ID`를 유지하고 `level`을 통해 메서드 호출의 깊이를 표현하는 것을 확인할 수 있다. 

</details>

<details> <summary> 7. 로그 추적기 V2 - 적용 </summary>

## 7. 로그 추적기 V2 - 적용

로그 추적기를 애플리케이션에 적용하자

**v1 -> v2 복사**

로그 추적기 V2를 적용하기 전에 먼저 기존 코드를 복사하자 
- `hello.advanced.app.v2`패키지 생성
- 복사
  - `v1.OrderControllerV1` -> `v2.OrderControllerV2`
  - `v1.OrderServiceV1` -> `v2.OrderServiceV2`
  - `v1.OrderRepositoryV1` -> `v2.OrderRepositoryV2`
- 코드 내부 의존관계 클래스를 V2로 변경
  - `OrderControllerV2`: `OrderServiceV1` -> `OrderServiceV2`
  - `OrderServiceV2`: `OrderRepositoryV1` -> `OrderRepositoryV2`
- `OrderControllerV2`: 매핑 정보 변경
  - `@GetMapping("/v2/request")
- `app.v2`에서는 `HelloTraceV1` -> `HelloTraceV2`를 사용하도록 변경
  - `OrderControllerV2`
  - `OrderServiceV2`
  - `OrderRepositoryV2`
 

### V2 적용하기

메서드 호출의 깊이를 표현하고, HTTP 요청도 구분해보자

이렇게 하려면 처음 로그를 남기는 `OrderController.request()`에서 로그를 남길 때 어떤 깊이와 어떤 트랜잭션 ID를 사용했는지 다음 차례인 `OrderService.orderItem()`에서 로그를 남기는 시점에 알아야 한다.

결국 현재 로그의 상태 정보인 `트랜잭션ID`와 `level`이 다음으로 전달되어야 한다.
이 정보는 `TraceStatus.traceId`에 담겨있다. 따라서 `traceId`를 컨트롤러에서 서비스를 호출할 때 넘겨주면 된다.

![image](https://user-images.githubusercontent.com/28394879/140507295-bcc66f99-fce9-42a5-9e08-d372f5a40ade.png)

`traceId`를 넘기도록 V2 전체 코드를 수정하자

**OrderControllerV2**
```java
package hello.advanced.app.v2;
import hello.advanced.trace.TraceStatus;
import hello.advanced.trace.hellotrace.HelloTraceV2;
import lombok.RequiredArgsConstructor;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;
@RestController
@RequiredArgsConstructor
public class OrderControllerV2 {
 private final OrderServiceV2 orderService;
 private final HelloTraceV2 trace;
 @GetMapping("/v2/request")
 public String request(String itemId) {
 TraceStatus status = null;
 try {
 status = trace.begin("OrderController.request()");
 orderService.orderItem(status.getTraceId(), itemId);
 trace.end(status);
 return "ok";
 } catch (Exception e) {
 trace.exception(status, e);
 throw e;
 }
 }
}
```
- `TraceStatus status = trace.begin()`에서 반환 받은 `TraceStatus`에는 `트랜잭션ID`와 `level`정보가 있는 `TraceId`가 있다.
- `orderService.orderItem()`을 호출할 때 `TraceId`를 파라미터로 전달한다.
- `TraceId`를 파라미터로 전달하기 위해 `OrderServiceV2.orderItem()`의 파라미터에 `TraceId`를 추가해야 한다.


**OrderServiceV2**
```java
package hello.advanced.app.v2;
import hello.advanced.trace.TraceId;
import hello.advanced.trace.TraceStatus;
import hello.advanced.trace.hellotrace.HelloTraceV2;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;
@Service
@RequiredArgsConstructor
public class OrderServiceV2 {
 private final OrderRepositoryV2 orderRepository;
 private final HelloTraceV2 trace;
 public void orderItem(TraceId traceId, String itemId) {
 TraceStatus status = null;
 try {
 status = trace.beginSync(traceId, "OrderService.orderItem()");
 orderRepository.save(status.getTraceId(), itemId);
 trace.end(status);
 } catch (Exception e) {
 trace.exception(status, e);
 throw e;
 }
 }
}
```
- `orderItem()`은 파라미터로 전달 받은 `traceId`를 사용해서 `trace.beginSync()`를 실행한다.
- `beginSync()`는 내부에서 다음 `traceId`를 생성하면서 트랜잭션ID는 유지하고 `level`은 하나 증가시킨다.
- `beginSync()`가 반환한 새로운 `TraceStatus`를 `orderRepository.save()`를 호출하면서 파라미터로 전달한다.
- `TraceId`를 파라미터로 전달하기 위해 `orderRepository.save()`의 파라미터에 `TraceId`를 추가해야 한다.

**OrderRepositoryV2**
```java
package hello.advanced.app.v2;
import hello.advanced.trace.TraceId;
import hello.advanced.trace.TraceStatus;
import hello.advanced.trace.hellotrace.HelloTraceV2;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Repository;
@Repository
@RequiredArgsConstructor
public class OrderRepositoryV2 {
 private final HelloTraceV2 trace;
 public void save(TraceId traceId, String itemId) {
 TraceStatus status = null;
 try {
 status = trace.beginSync(traceId, "OrderRepository.save()");
 //저장 로직
 if (itemId.equals("ex")) {
 throw new IllegalStateException("예외 발생!");
 }
 sleep(1000);
 trace.end(status);
 } catch (Exception e) {
 trace.exception(status, e);
 throw e;
 }
 }
 private void sleep(int millis) {
 try {
 Thread.sleep(millis);
 } catch (InterruptedException e) {
 e.printStackTrace();
 }
 }
}
```

- `save()`는 파라미터로 전달 받은 `traceId`를 사용해서 `trace.beginSync()`를 실행한다.
- `beginSync()`는 내부에서 다음 `traceId`를 생성하면서 트랜잭션ID는 유지하고 `level`은 하나 증가시킨다.
- `beginSync()`는 이렇게 갱신된 `traceId`로 새로운 `TraceStatus`를 반환한다.
- `trace.end(status)`를 호출하면서 반환된 `TraceStatus`를 전달한다.

**정상 실행**
- http://localhost:8080/v2/request?itemId=hello

**정상 실행 로그**
```
[c80f5dbb] OrderController.request()
[c80f5dbb] |-->OrderService.orderItem()
[c80f5dbb] | |-->OrderRepository.save()
[c80f5dbb] | |<--OrderRepository.save() time=1005ms
[c80f5dbb] |<--OrderService.orderItem() time=1014ms
[c80f5dbb] OrderController.request() time=1017ms
```

**예외 실행**
- http://localhost:8080/v2/request?itemId=ex

**예외 실행 로그**
```
[ca867d59] OrderController.request()
[ca867d59] |-->OrderService.orderItem()
[ca867d59] | |-->OrderRepository.save()
[ca867d59] | |<X-OrderRepository.save() time=0ms
ex=java.lang.IllegalStateException: 예외 발생!
[ca867d59] |<X-OrderService.orderItem() time=7ms 
ex=java.lang.IllegalStateException: 예외 발생!
[ca867d59] OrderController.request() time=7ms
ex=java.lang.IllegalStateException: 예외 발생!
```

실행 로그를 보면 같은 HTTP 요청에 대해서 `트랜잭션ID`가 유지되고, `level`도 잘 표현되는 것을 확인할 수 있다.


</details>

<details> <summary> 8. 정리 </summary>

## 8. 정리 

**요구사항**

- ~~모든 PUBLIC 메서드의 호출과 응답 정보를 로그로 출력~~
- ~~애플리케이션의 흐름을 변경하면 안됨~~
  - ~~로그를 남긴다고 해서 비즈니스 로직의 동작에 영향을 주면 안됨~~
- ~~메서드 호출에 걸린 시간~~
- ~~정상 흐름과 예외 흐름 구분~~
  - ~~예외 발생시 예외 정보가 남아야 함~~
- ~~메서드 호출의 깊이 표현~~
- ~~HTTP 요청을 구분~~
  - ~~HTTP 요청 단위로 특정 ID를 남겨서 어떤 HTTP 요청에서 시작된 것인지 명확하게 구분이 가능해야 함~~
  - ~~트랜잭션 ID (DB 트랜잭션 말고..)~~

**남은 문제**
- HTTP 요청을 구분하고 깊이를 표현하기 위해서 `TraceId` 동기화가 필요하다.
- `TraceId`의 동기화를 위해서 관련 메서드의 모든 파라미터를 수정해야 한다.
  - 만약 인터페이스가 있다면 인터페이스까지 모두 고쳐야 하는 상황이다.
- 로그를 처음 시작할 때는 `begin()`을 호출하고, 호출이 아닐때는 `beginSync()`를 호출해야 한다.
  - 만약에 컨트롤러를 통해서 서비스를 호출하는 것이 아니라, 다른 곳에서 서비스를 처음으로 호출하는 상황이라면 파라미터로 넘길 `TraceId`가 없다.

HTTP 요청을 구분하고 깊이를 표현하기 위해서 `TraceId`를 파라미터로 넘기는 것 말고 다른 대안은 없을까?


</details>

# [2. 쓰레드 로컬 - ThreadLocal](./2.threadLocal)

<details> <summary> 1. 필드 동기화 - 개발 </summary>

## 1. 필드 동기화 - 개발

앞서 로그 추적기를 만들면서 다음 로그를 출력할 때 `트랜잭션ID`와 `level`을 동기화 하는 문제가 있었다.

이 문제를 해결하기 위해 `TraceId`를 파라미터로 넘기도록 구현했다.

이렇게 해서 동기화는 성공했지만, 로그를 출력하는 모든 메서드에 `TraceId` 파라미터를 추가해야 하는 문제가 발생했다.

`TraceId`를 파라미터로 넘기지 않고 이 문제를 해결할 수 있는 방법은 없을까?

이런 문제를 해결할 목적으로 새로운 로그 추적기를 만들어보자.
이제 프로토타입 버전이 아닌 정식 버전으로 제대로 개발해보자.

향후 다양한 구현체로 변경할 수 있도록 `LogTrace` 인터페이스를 먼저 만들고, 구현 해보자.

**LogTrace 인터페이스**
```java
package hello.advanced.trace.logtrace;
import hello.advanced.trace.TraceStatus;
public interface LogTrace {
 TraceStatus begin(String message);
 void end(TraceStatus status);
 void exception(TraceStatus status, Exception e);
}
```

**FieldLogTrace**
```java
package hello.advanced.trace.logtrace;
import hello.advanced.trace.TraceId;
import hello.advanced.trace.TraceStatus;
import lombok.extern.slf4j.Slf4j;
@Slf4j
public class FieldLogTrace implements LogTrace {
 private static final String START_PREFIX = "-->";
 private static final String COMPLETE_PREFIX = "<--";
 private static final String EX_PREFIX = "<X-";
 private TraceId traceIdHolder; //traceId 동기화, 동시성 이슈 발생
 @Override
 public TraceStatus begin(String message) {
 syncTraceId();
 TraceId traceId = traceIdHolder;
 Long startTimeMs = System.currentTimeMillis();
 log.info("[{}] {}{}", traceId.getId(), addSpace(START_PREFIX,
traceId.getLevel()), message);
 return new TraceStatus(traceId, startTimeMs, message);
 }
 @Override
 public void end(TraceStatus status) {
 complete(status, null);
 }
 @Override
 public void exception(TraceStatus status, Exception e) {
 complete(status, e);
 }
 private void complete(TraceStatus status, Exception e) {
 Long stopTimeMs = System.currentTimeMillis();
 long resultTimeMs = stopTimeMs - status.getStartTimeMs();
 TraceId traceId = status.getTraceId();
 if (e == null) {
 log.info("[{}] {}{} time={}ms", traceId.getId(),
addSpace(COMPLETE_PREFIX, traceId.getLevel()), status.getMessage(),
resultTimeMs);
 } else {
 log.info("[{}] {}{} time={}ms ex={}", traceId.getId(),
addSpace(EX_PREFIX, traceId.getLevel()), status.getMessage(), resultTimeMs,
e.toString());
 }
 releaseTraceId();
 }
 private void syncTraceId() {
 if (traceIdHolder == null) {
 traceIdHolder = new TraceId();
 } else {
 traceIdHolder = traceIdHolder.createNextId();
 }
 }
 private void releaseTraceId() {
 if (traceIdHolder.isFirstLevel()) {
 traceIdHolder = null; //destroy
 } else {
 traceIdHolder = traceIdHolder.createPreviousId();
 }
 }
 private static String addSpace(String prefix, int level) {
 StringBuilder sb = new StringBuilder();
 for (int i = 0; i < level; i++) {
 sb.append( (i == level - 1) ? "|" + prefix : "| ");
 }
 return sb.toString();
 }
}
```
`FieldLogTrace`는 기존에 만들었던 `HelloTraceV2`와 거의 같은 기능을 한다.

`TraceId`를 동기화하는 부분만 파라미터를 사용하는 것에서 `TraceId traceIdHolder`필드를 사용하도록 변경되었다.

이제 직전 로그의 `TraceId`는 파라미터로 전달되는 것이 아니라 `FieldLogTrace`의 필드인 `traceIdHolder`에 저장된다.

여기서 중요한 부분은 로그를 싲가할 때 호출하는 `syncTraceId()`와 로그를 종료할 때 호출하는 `releaseTraceId()`이다.

- `syncTraceId()`
  - `TraceId`를 새로 만들거나 앞선 로그의 `TraceId`를 참고해서 동기화하고, `level`도 증가한다.
  - 최초 호출이면 `TraceId`를 새로 만든다.
  - 직전 로그가 있으면 해당 로그의 `TraceId`를 참고해서 동기화하고, `level`도 하나 증가한다.
  - 결과를 `traceIdHolder`에 보관한다.
- `releaseTraceId()`
  - 메서드를 추가로 호출할 때는 `level`이 하나 증가해야 하지만, 메서드 호출이 끝나면 `level`이 하나 감소해야 한다.
  - `releaseTraceId()`는 `level`을 하나 감소한다.
  - 만약 최초 호출(`level==0`)이면 내부에서 관리하는 `traceId`를 제거한다

```
[c80f5dbb] OrderController.request() //syncTraceId(): 최초 호출 level=0
[c80f5dbb] |-->OrderService.orderItem() //syncTraceId(): 직전 로그 있음 level=1
증가
[c80f5dbb] | |-->OrderRepository.save() //syncTraceId(): 직전 로그 있음 level=2
증가
[c80f5dbb] | |<--OrderRepository.save() time=1005ms //releaseTraceId(): 
level=2->1 감소
[c80f5dbb] |<--OrderService.orderItem() time=1014ms //releaseTraceId():
level=1->0 감소
[c80f5dbb] OrderController.request() time=1017ms //releaseTraceId():
level==0, traceId 제거
```

### 테스트 코드

**FieldLogTraceTest**
```java
package hello.advanced.trace.logtrace;
import hello.advanced.trace.TraceStatus;
import org.junit.jupiter.api.Test;
class FieldLogTraceTest {
 FieldLogTrace trace = new FieldLogTrace();
 @Test
 void begin_end_level2() {
 TraceStatus status1 = trace.begin("hello1");
 TraceStatus status2 = trace.begin("hello2");
 trace.end(status2);
 trace.end(status1);
 }
 @Test
 void begin_exception_level2() {
 TraceStatus status1 = trace.begin("hello");
 TraceStatus status2 = trace.begin("hello2");
 trace.exception(status2, new IllegalStateException());
 trace.exception(status1, new IllegalStateException());
 }
}
```

**begin_end_level2() - 실행 결과**
```
[ed72b67d] hello1
[ed72b67d] |-->hello2
[ed72b67d] |<--hello2 time=2ms
[ed72b67d] hello1 time=6ms
```

**begin_exception_level2() - 실행 결과**
```
[59770788] hello
[59770788] |-->hello2
[59770788] |<X-hello2 time=3ms ex=java.lang.IllegalStateException
[59770788] hello time=8ms ex=java.lang.IllegalStateException
```

실행 결과를 보면 `트랜잭션ID`도 동일하게 나오고, `level`을 통한 깊이도 잘 표현된다.
`FieldLogTrace.traceIdHolder` 필드를 사용해서 `TraceId`가 잘 동기화 되는 것을 확인할 수 있다.
이제 불필요하게 `TraceId`를 파라미터로 전달하지 않아도 되고, 애플리케이션의 메서드 파라미터도 변경하지 않아도 된다.


</details>


<details> <summary> 2. 필드 동기화 - 적용 </summary>

## 2. 필드 동기화 - 적용

지금까지 만든 `FieldLogTrace`를 애플리케이션에 적용해보자

### LogTrace 스프링 빈 등록
`FieldLogTrace`를 수동으로 스프링 빈으로 등록하자. 수동으로 등록하면 향후 구현체를 편리하게 변경할 수 있다는 장점이 있다. 

**LogTraceConfig**
```java
package hello.advanced;
import hello.advanced.trace.logtrace.FieldLogTrace;
import hello.advanced.trace.logtrace.LogTrace;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
@Configuration
public class LogTraceConfig {
 @Bean
 public LogTrace logTrace() {
 return new FieldLogTrace();
 }
}
```

**v2 -> v3 복사**

로그 추적기 V3를 적용하기 전에 먼저 기존 코드를 복사하자.
- `hello.advanced.app.v3` 패키지 생성
- 복사
  - `v2.OrderControllerV2` -> `v3.OrderControllerV3`
  - `v2.OrderServiceV2` -> `v3.OrderServiceV3`
  - `v2.OrderRepositoryV2` -> `v3.OrderRepositoryV3`
- 코드 내부 의존관계 클래스를 V3으로 변경
  - `OrderControllerV3`: `OrderServiceV2` -> `OrderServiceV3`
  - `OrderServiceV3`: `OrderRepositoryV2` -> `OrderRepositoryV3`
- `OrderControllerV3` 매핑 정보 변경
  - `@GetMapping("/v3/request")`
- `HelloTraceV2` -> `LogTrace` 인터페이스 사용 -> **주의!**
- `TraceId traceId` 파라미터를 모두 제거
- `beginSync()` -> `begin`으로 사용다로고 변경 

전체 코드는 다음과 같다.

**OrderControllerV3**
```java
package hello.advanced.app.v3;
import hello.advanced.trace.TraceStatus;
import hello.advanced.trace.logtrace.LogTrace;
import lombok.RequiredArgsConstructor;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;
@RestController
@RequiredArgsConstructor
public class OrderControllerV3 {
 private final OrderServiceV3 orderService;
 private final LogTrace trace;
 @GetMapping("/v3/request")
 public String request(String itemId) {
 TraceStatus status = null;
 try {
 status = trace.begin("OrderController.request()");
 orderService.orderItem(itemId);
 trace.end(status);
 return "ok";
 } catch (Exception e) {
 trace.exception(status, e);
 throw e; //예외를 꼭 다시 던져주어야 한다.
 }
 }
}
```

**OrderServiceV3**
```java
package hello.advanced.app.v3;
import hello.advanced.trace.TraceId;
import hello.advanced.trace.TraceStatus;
import hello.advanced.trace.logtrace.LogTrace;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;
@Service
@RequiredArgsConstructor
public class OrderServiceV3 {
 private final OrderRepositoryV3 orderRepository;
 private final LogTrace trace;
 public void orderItem(String itemId) {
 TraceStatus status = null;
 try {
 status = trace.begin("OrderService.orderItem()");
 orderRepository.save(itemId);
 trace.end(status);
 } catch (Exception e) {
 trace.exception(status, e);
 throw e;
 }
 }
}
```

**OrderRepositoryV3**
```java
package hello.advanced.app.v3;
import hello.advanced.trace.TraceId;
import hello.advanced.trace.TraceStatus;
import hello.advanced.trace.logtrace.LogTrace;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Repository;
@Repository
@RequiredArgsConstructor
public class OrderRepositoryV3 {
 private final LogTrace trace;
 public void save(String itemId) {
 TraceStatus status = null;
 try {
 status = trace.begin("OrderRepository.save()");
 //저장 로직
 if (itemId.equals("ex")) {
 throw new IllegalStateException("예외 발생!");
 }
 sleep(1000);
 trace.end(status);
 } catch (Exception e) {
 trace.exception(status, e);
 throw e;
 }
 }
 private void sleep(int millis) {
 try {
 Thread.sleep(millis);
 } catch (InterruptedException e) {
 e.printStackTrace();
 }
 }
}
```

**정상 실행**
- http://localhost:8080/v3/request?itemId=hello

**정상 실행 로그**
```
[f8477cfc] OrderController.request()
[f8477cfc] |-->OrderService.orderItem()
[f8477cfc] | |-->OrderRepository.save()
[f8477cfc] | |<--OrderRepository.save() time=1004ms
[f8477cfc] |<--OrderService.orderItem() time=1006ms
[f8477cfc] OrderController.request() time=1007ms
```

**예외 실행**
- http://localhost:8080/v3/request?itemId=ex

**예외 실행 로그**
```
[c426fcfc] OrderController.request()
[c426fcfc] |-->OrderService.orderItem()
[c426fcfc] | |-->OrderRepository.save()
[c426fcfc] | |<X-OrderRepository.save() time=0ms
ex=java.lang.IllegalStateException: 예외 발생!
[c426fcfc] |<X-OrderService.orderItem() time=7ms
ex=java.lang.IllegalStateException: 예외 발생!
[c426fcfc] OrderController.request() time=7ms
ex=java.lang.IllegalStateException: 예외 발생!
```

`traceIdHolder` 필드를 사용한 덕분에 파라미터 추가 없는 깔끔한 로그 추적기를 완성했다. 이제 실제 서비스에 배포한다고 가정해보자. 


</details>


<details> <summary> 3. 필드 동기화 - 동시성 문제 </summary>

## 3. 필드 동기화 - 동시성 문제

잘 만든 로그 추적기를 실제 서비스에 배포 했다고 가정해보자.

테스트 할 때는 문제가 없는 것처럼 보인다. 

하지만 `FieldLogTrace`는 심각한 동시성 문제를 가지고 있다.

동시성 문제를 확인하려면 다음과 같이 동시에 여러번 호출해보면 된다.

**동시성 문제 확인**  
다음 로직을 1초안에 2번 실행해보자.
- http://localhost:8080/v3/request?itemId=hello
- http://localhost:8080/v3/request?itemId=hello

**기대하는 결과**
```
[nio-8080-exec-3] [52808e46] OrderController.request()
[nio-8080-exec-3] [52808e46] |-->OrderService.orderItem()
[nio-8080-exec-3] [52808e46] | |-->OrderRepository.save()
[nio-8080-exec-4] [4568423c] OrderController.request()
[nio-8080-exec-4] [4568423c] |-->OrderService.orderItem()
[nio-8080-exec-4] [4568423c] | |-->OrderRepository.save()
[nio-8080-exec-3] [52808e46] | |<--OrderRepository.save() time=1001ms
[nio-8080-exec-3] [52808e46] |<--OrderService.orderItem() time=1001ms
[nio-8080-exec-3] [52808e46] OrderController.request() time=1003ms
[nio-8080-exec-4] [4568423c] | |<--OrderRepository.save() time=1000ms
[nio-8080-exec-4] [4568423c] |<--OrderService.orderItem() time=1001ms
[nio-8080-exec-4] [4568423c] OrderController.request() time=1001ms
```

동시에 여러 사용자가 요청하면 여러 쓰레드가 동시에 애플리케이션 로직을 호출하게 된다.  
따라서 로그는 이렇게 섞어서 출력된다.

**기대하는 결과 - 로그 분리해서 확인하기**
```
[52808e46]
[nio-8080-exec-3] [52808e46] OrderController.request()
[nio-8080-exec-3] [52808e46] |-->OrderService.orderItem()
[nio-8080-exec-3] [52808e46] | |-->OrderRepository.save()
[nio-8080-exec-3] [52808e46] | |<--OrderRepository.save() time=1001ms
[nio-8080-exec-3] [52808e46] |<--OrderService.orderItem() time=1001ms
[nio-8080-exec-3] [52808e46] OrderController.request() time=1003ms
[4568423c]
[nio-8080-exec-4] [4568423c] OrderController.request()
[nio-8080-exec-4] [4568423c] |-->OrderService.orderItem()
[nio-8080-exec-4] [4568423c] | |-->OrderRepository.save()
[nio-8080-exec-4] [4568423c] | |<--OrderRepository.save() time=1000ms
[nio-8080-exec-4] [4568423c] |<--OrderService.orderItem() time=1001ms
[nio-8080-exec-4] [4568423c] OrderController.request() time=1001ms
```

로그가 섞어서 출력되더라도 특정 트랜잭션ID로 구분해서 직접 분류해보면 이렇게 깔끔하게 분리된 것을 확인할 수 있다.  
그런데 실제 결과를 기대한 것과 다르게 다음과 같이 출력 된다.  

**실제 결과**
```
[nio-8080-exec-3] [aaaaaaaa] OrderController.request()
[nio-8080-exec-3] [aaaaaaaa] |-->OrderService.orderItem()
[nio-8080-exec-3] [aaaaaaaa] | |-->OrderRepository.save()
[nio-8080-exec-4] [aaaaaaaa] | | |-->OrderController.request()
[nio-8080-exec-4] [aaaaaaaa] | | | |-->OrderService.orderItem()
[nio-8080-exec-4] [aaaaaaaa] | | | | |-->OrderRepository.save()
[nio-8080-exec-3] [aaaaaaaa] | |<--OrderRepository.save() time=1005ms
[nio-8080-exec-3] [aaaaaaaa] |<--OrderService.orderItem() time=1005ms
[nio-8080-exec-3] [aaaaaaaa] OrderController.request() time=1005ms
[nio-8080-exec-4] [aaaaaaaa] | | | | |<--OrderRepository.save()
time=1005ms
[nio-8080-exec-4] [aaaaaaaa] | | | |<--OrderService.orderItem()
time=1005ms
[nio-8080-exec-4] [aaaaaaaa] | | |<--OrderController.request() time=1005ms
```

**실제 결과 - 로그 분리해서 확인하기**
```
[nio-8080-exec-3]
[nio-8080-exec-3] [aaaaaaaa] OrderController.request()
[nio-8080-exec-3] [aaaaaaaa] |-->OrderService.orderItem()
[nio-8080-exec-3] [aaaaaaaa] | |-->OrderRepository.save()
[nio-8080-exec-3] [aaaaaaaa] | |<--OrderRepository.save() time=1005ms
[nio-8080-exec-3] [aaaaaaaa] |<--OrderService.orderItem() time=1005ms
[nio-8080-exec-3] [aaaaaaaa] OrderController.request() time=1005ms
[nio-8080-exec-4]
[nio-8080-exec-4] [aaaaaaaa] | | |-->OrderController.request()
[nio-8080-exec-4] [aaaaaaaa] | | | |-->OrderService.orderItem()
[nio-8080-exec-4] [aaaaaaaa] | | | | |-->OrderRepository.save()
[nio-8080-exec-4] [aaaaaaaa] | | | | |<--OrderRepository.save()
time=1005ms
[nio-8080-exec-4] [aaaaaaaa] | | | |<--OrderService.orderItem()
time=1005ms
[nio-8080-exec-4] [aaaaaaaa] | | |<--OrderController.request() time=1005ms
```

기대한 것과 전혀 다른 문제가 발생한다. `트랜잭션 ID`도 동일하고, `level`도 뭔가 많이 꼬인 것 같다.   

**동시성 문제**  
이 문제가 동시성 문제다.
`FieldLogTrace`는 싱글톤으로 등록된 스프링 빈이다. 이 객체의 인스턴스가 애플리케이션에 딱 1개 존재한다는 뜻이다.  
이렇게 하나만 있는 인스턴스의 `FieldLogTrace.traceIdHolder` 필드를 여러 쓰레드가 동시에 접근하기 때문에 문제가 발생한다.  
실무에서 한번 나타나면 개발자를 가장 괴롭히는 문제도 바로 이러한 동시성 문제이다.


</details>

<details> <summary> 4. 동시성 문제 - 예제 코드 </summary>

## 4. 동시성 문제 - 예제 코드

동시성 문제가 어떻게 발생하는지 단순화해서 알아보자. 

테스트에서도 lombok을 사용하기 위해 다음 코드를 추가하자  
`build.gradle`
```
dependencies {
 ...
 //테스트에서 lombok 사용
 testCompileOnly 'org.projectlombok:lombok'
 testAnnotationProcessor 'org.projectlombok:lombok'
}
```
이렇게 해야 테스트 코드에서 `@Slfj4` 같은 애노테이션이 작동한다.

**FieldService**  
주의: 테스트코드(src/test)에 위치한다. 
```java
package hello.advanced.trace.threadlocal.code;
import lombok.extern.slf4j.Slf4j;
@Slf4j
public class FieldService {
 private String nameStore;
 public String logic(String name) {
 log.info("저장 name={} -> nameStore={}", name, nameStore);
 nameStore = name;
 sleep(1000);
 log.info("조회 nameStore={}",nameStore);
 return nameStore;
 }
 private void sleep(int millis) {
 try {
 Thread.sleep(millis);
 } catch (InterruptedException e) {
 e.printStackTrace();
 }
 }
}
```
매우 단순한 로직이다. 파라미터로 넘어온 `name`을 필드인 `nameStore`에 저장한다. 그리고 1초간 쉰 다음 필드에 저장된 `nameStore`를 반환한다.

**FieldServiceTest**
```java
package hello.advanced.trace.threadlocal;
import hello.advanced.trace.threadlocal.code.FieldService;
import lombok.extern.slf4j.Slf4j;
import org.junit.jupiter.api.Test;
@Slf4j
public class FieldServiceTest {
 private FieldService fieldService = new FieldService();
 @Test
 void field() {
 log.info("main start");
 Runnable userA = () -> {
 fieldService.logic("userA");
 };
 Runnable userB = () -> {
 fieldService.logic("userB");
 };
 Thread threadA = new Thread(userA);
 threadA.setName("thread-A");
 Thread threadB = new Thread(userB);
 threadB.setName("thread-B");
 threadA.start(); //A실행
 sleep(2000); //동시성 문제 발생X
// sleep(100); //동시성 문제 발생O
 threadB.start(); //B실행
 sleep(3000); //메인 쓰레드 종료 대기
 log.info("main exit");
 }
 private void sleep(int millis) {
 try {
 Thread.sleep(millis);
 } catch (InterruptedException e) {
 e.printStackTrace();
 }
 }
}
```

**순서대로 실행**  
`sleep(2000)`을 설정해서 `thread-A`의 실행이 끝나고 나서 `thread-B`가 실행되도록 해보자.  
참고로 `FieldService.logic()`메서드는 내부에 `sleep(1000)`으로 1초의 지연이 있다.  
따라서 1초 이후에 호출하면 순서대로 실행할 수 있다. 여기서는 넉넉하게 2초(2000ms)를 설정했다.
```java
sleep(2000); //동시성 문제 발생X
//sleep(100); //동시성 문제 발생O
```

**실행 결과**
```java
[Test worker] main start
[Thread-A] 저장 name=userA -> nameStore=null
[Thread-A] 조회 nameStore=userA
[Thread-B] 저장 name=userB -> nameStore=userA
[Thread-B] 조회 nameStore=userB
[Test worker] main exit
```

실행 결과를 보면 문제가 없다. 


**동시성 문제 발생 코드**  
이번에는 `sleep(100)`을 설정해서 `thread-A`의 작업이 끝나기 전에 `thread-B`가 실행되도록 해보자.  
참고로 `FieldService.logic()` 메서드는 내부에 `sleep(1000)`으로 1초의 지연이 있다.  
따라서 1초 이후에 호출하면 순서대로 실행할 수 있다.  
다음에 설정할 100(ms)는 0.1초 이기 때문에 `thread-A`의 작업이 끝나기 전에 `thread-B`가 실행된다.
```java
//sleep(2000); //동시성 문제 발생X
sleep(100); //동시성 문제 발생O
```

**실행 결과**
```
[Test worker] main start
[Thread-A] 저장 name=userA -> nameStore=null
[Thread-B] 저장 name=userB -> nameStore=userA
[Thread-A] 조회 nameStore=userB
[Thread-B] 조회 nameStore=userB
[Test worker] main exit
```

실행 결과를 보면 저장하는 부분은 문제가 없는데, 조회하는 부분에서 발생한다.
- `thread-A`의 호출이 끝나면서 `nameStore`의 결과를 반환하는데, 이때 `nameStore`는 앞의 2번에서  
`userB`의 값으로 대체되었다. 따라서 기대했던 `userA`의 값이 아니라 `userB`의 값이 반환된다.
- `thread-B`의 호출이 끝나면서 `nameStore`의 결과인 `userB`를 반환받는다. 

**동시성 문제**  
결과적으로 `thread-A`입장에서는 저장한 데이터와 조회한 데이터가 다른 문제가 발생한다.  
이처럼 여러 쓰레드가 동시에 같은 인스턴스의 필드 값을 변경하면서 발생하는 문제를 동시성 문제라 한다.  
이런 동시성 문제는 여러 쓰레드가 같은 인스턴스의 필드에 접근해야 하기 때문에 트래픽이 적은 상황에서는 확률상 잘 나타나지 않고, 트래픽이 점점 많아질 수록 자주 발생한다.  
특히 스프링 빈 처럼 싱글톤 객체의 필드를 변경하며 사용할 때 이러한 동시성 문제를 조심해야 한다. 

> 참고  
> 이런 동시성 문제는 지역 변수에서는 발생하지 않는다. 지역 변수는 쓰레드마다 각각 다른 메모리 영역이 할당된다.  
> 동시성 문제가 발생하는 곳은 같은 인스턴스의 필드(주로 싱글톤에서 자주 발생), 또는 static 같은 공용 필드에 접근할 때 발생한다.  
> 동시성 문제는 값을 읽기만 하면 발생하지 않는다. 어디선가 값을 변경하기 때문에 발생한다. 

그렇다면 지금처럼 싱글톤 객체의 필드를 사용하면서 동시성 문제를 해결하려면 어떻게 해야할까? 다시 파라미터를 전달하는 방식으로 돌아가야 할까? 이럴 때 사용하는 것이 바로 쓰레드 로컬이다. 

</details>


<details> <summary> 5. ThreadLocal - 소개 </summary>

## 5. ThreadLocal - 소개

쓰레드 로컬은 해당 쓰레드만 접근할 수 있는 특별한 저장소를 말한다. 쉽게 이야기해서 물건 보관 창구를 떠올리면 된다.  
여러 사람이 같은 물건 보관 창구르 사용하더라도 창구 직원은 사용자를 인식해서 사용자별로 확실하게 물건을 구분해준다.  
사용자A, 사용자B 모두 창구 직원을 통해서 물건을 보관하고, 꺼내지만 창구 지원이 사용자에 따라 보관한 물건을 구분해주는 것이다. 

**일반적인 변수 필드**  
여러 쓰레드가 같은 인스턴스의 필드에 접근하면 처음 쓰레드가 보관한 데이터가 사라질 수 있다. (이전까지 설명했던 동시성 문제)

**쓰레드 로컬**  
쓰레드 로컬을 사용하면 각 쓰레드마다 별도의 내부 저장소를 제공한다. 따라서 같은 인스턴스의 쓰레드 로컬 필드에 접근해도 문제 없다. 

자바는 언어차원에서 쓰레드 로컬을 지원하기 위한 `java.lang.ThreadLocal` 클래스를 제공한다. 

</details>


<details> <summary> 6. ThreadLocal - 예제 코드 </summary>

## 6. ThreadLocal - 예제 코드

예제 코드를 통해서 `ThreadLocal`을 학습해보자.

**ThreadLocalService**  
주의: 테스트 코드(src/test)에 위치한다.
```java
package hello.advanced.trace.threadlocal.code;
import lombok.extern.slf4j.Slf4j;
@Slf4j
public class ThreadLocalService {
 private ThreadLocal<String> nameStore = new ThreadLocal<>();
 public String logic(String name) {
 log.info("저장 name={} -> nameStore={}", name, nameStore.get());
 nameStore.set(name);
 sleep(1000);
 log.info("조회 nameStore={}",nameStore.get());
 return nameStore.get();
 }
 private void sleep(int millis) {
 try {
 Thread.sleep(millis);
 } catch (InterruptedException e) {
 e.printStackTrace();
 }
 }
}
```
기존에 있던 `FieldService`와 거의 같은 코드인데, `nameStore`필드가 일반 `String`타입에서 `ThreadLocal`을 사용하도록 변경되었다. 

**ThreadLocal 사용법**
- 값 저장: `ThreadLocal.set(xxx)`
- 값 조회: `ThreadLocal.get()`
- 값 제거: `ThreadLocal.remove()`

> 주의  
> 해당 쓰레드가 쓰레드 로컬을 모두 사용하고 나면 `ThreadLocal.remove()`를 호출해서 쓰레드 로컬에  
> 저장된 값을 제거해주어야 한다. 제거하는 구체적인 예제는 조금 뒤에 설명한다.

**ThreadLocalServiceTest**
```java
package hello.advanced.trace.threadlocal;
import hello.advanced.trace.threadlocal.code.ThreadLocalService;
import lombok.extern.slf4j.Slf4j;
import org.junit.jupiter.api.Test;
@Slf4j
public class ThreadLocalServiceTest {
 private ThreadLocalService service = new ThreadLocalService();
 @Test
 void threadLocal() {
 log.info("main start");
 Runnable userA = () -> {
 service.logic("userA");
 };
 Runnable userB = () -> {
 service.logic("userB");
 };
 Thread threadA = new Thread(userA);
 threadA.setName("thread-A");
 Thread threadB = new Thread(userB);
 threadB.setName("thread-B");
 threadA.start();
 sleep(100);
 threadB.start();
 sleep(2000);
 log.info("main exit");
 }
 private void sleep(int millis) {
 try {
 Thread.sleep(millis);
 } catch (InterruptedException e) {
 e.printStackTrace();
 }
 }
}
```

**실행 결과**
```
[Test worker] main start
[Thread-A] 저장 name=userA -> nameStore=null
[Thread-B] 저장 name=userB -> nameStore=null
[Thread-A] 조회 nameStore=userA
[Thread-B] 조회 nameStore=userB
[Test worker] main exit
```

쓰레드 로컬 덕분에 쓰레드 마다 각 별도의 데이터 저장소를 가지게 되었다. 결과적으로 동시성 문제도 해결되었다.



</details>


<details> <summary> 7. 쓰레드 로컬 동기화 - 개발 </summary>

## 7. 쓰레드 로컬 동기화 - 개발

`FieldLogTrace`에서 발생했던 동시성 문제를 `ThreadLocal`로 해결해보자  
`TraceId traceIdHolder` 필드를 쓰레드 로컬을 사용하도록 `ThreadLocal<TraceId> traceIdHolder`로 변경하면 된다.

필드 대신에 쓰레드 로컬을 사용해서 데이터를 동기화하는 `ThreadLocalLogTrace`를 새로 만들자. 

**ThreadLocalLogTrace**
```java
package hello.advanced.trace.logtrace;
import hello.advanced.trace.TraceId;
import hello.advanced.trace.TraceStatus;
import lombok.extern.slf4j.Slf4j;
@Slf4j
public class ThreadLocalLogTrace implements LogTrace {
 private static final String START_PREFIX = "-->";
 private static final String COMPLETE_PREFIX = "<--";
 private static final String EX_PREFIX = "<X-";
 private ThreadLocal<TraceId> traceIdHolder = new ThreadLocal<>();
 @Override
 public TraceStatus begin(String message) {
 syncTraceId();
 TraceId traceId = traceIdHolder.get();
 Long startTimeMs = System.currentTimeMillis();
 log.info("[{}] {}{}", traceId.getId(), addSpace(START_PREFIX,
traceId.getLevel()), message);
 return new TraceStatus(traceId, startTimeMs, message);
 }
 @Override
 public void end(TraceStatus status) {
 complete(status, null);
 }
 @Override
 public void exception(TraceStatus status, Exception e) {
 complete(status, e);
 }
 private void complete(TraceStatus status, Exception e) {
 Long stopTimeMs = System.currentTimeMillis();
 long resultTimeMs = stopTimeMs - status.getStartTimeMs();
 TraceId traceId = status.getTraceId();
 if (e == null) {
 log.info("[{}] {}{} time={}ms", traceId.getId(),
addSpace(COMPLETE_PREFIX, traceId.getLevel()), status.getMessage(),
resultTimeMs);
 } else {
 log.info("[{}] {}{} time={}ms ex={}", traceId.getId(),
addSpace(EX_PREFIX, traceId.getLevel()), status.getMessage(), resultTimeMs,
e.toString());
 }
 releaseTraceId();
 }
 private void syncTraceId() {
 TraceId traceId = traceIdHolder.get();
 if (traceId == null) {
 traceIdHolder.set(new TraceId());
 } else {
 traceIdHolder.set(traceId.createNextId());
 }
 }
 private void releaseTraceId() {
 TraceId traceId = traceIdHolder.get();
 if (traceId.isFirstLevel()) {
 traceIdHolder.remove();//destroy
 } else {
 traceIdHolder.set(traceId.createPreviousId());
 }
 }
 private static String addSpace(String prefix, int level) {
 StringBuilder sb = new StringBuilder();
 for (int i = 0; i < level; i++) {
 sb.append( (i == level - 1) ? "|" + prefix : "| ");
 }
 return sb.toString();
 }
}
```

`traceIdHolder`가 필드에서 `ThreadLocal`로 변경되었다. 따라서 값을 저장할 때는 `set(..)`을 사용하고, 값을 조회할 때는 `get()`을 사용한다.

**ThreadLocal.remove()**  
추가로 쓰레드 로컬을 모두 사용하고 나면 꼭 `ThreadLocal.remove()`를 호출해서 쓰레드 로컬에 저장된 값을 제거해주어야 한다.  
쉽게 이야기해서 다음의 마지막 로그를 출력하고 나면 쓰레드 로컬의 값을 제거해야 한다.
```java
[3f902f0b] hello1
[3f902f0b] |-->hello2
[3f902f0b] |<--hello2 time=2ms
[3f902f0b] hello1 time=6ms //end() -> releaseTraceId() -> level==0,
ThreadLocal.remove() 호출
```

여기서는 `releaseTraceId()`를 통해 `level`이 점점 낮아져서 2->1->0이 되면 로그를 처음 호출한 부분으로 돌아온 것이다.  
따라서 이 경우 연관된 로그 출력이 끝난 것이다. 이제 더이상 `TraceId` 값을 추적하지 않아도 된다.  
그래서 `traceId.isFirstLevel()``(level==0)`인 경우 `ThreadLocal.remove()`를 호출해서 쓰레드 로컬에 저장된 값을 제거해준다. 

코드에 문제가 없는지 간단한 테스트를 만들어서 확인해보자 

**ThreadLocalLogTraceTest**
```java
package hello.advanced.trace.logtrace;
import hello.advanced.trace.TraceStatus;
import lombok.extern.slf4j.Slf4j;
import org.junit.jupiter.api.Test;
@Slf4j
class ThreadLocalLogTraceTest {
 ThreadLocalLogTrace trace = new ThreadLocalLogTrace();
 @Test
 void begin_end_level2() {
 TraceStatus status1 = trace.begin("hello1");
 TraceStatus status2 = trace.begin("hello2");
 trace.end(status2);
 trace.end(status1);
 }
 @Test
 void begin_exception_level2() {
 TraceStatus status1 = trace.begin("hello");
 TraceStatus status2 = trace.begin("hello2");
 trace.exception(status2, new IllegalStateException());
 trace.exception(status1, new IllegalStateException());
 }
}
```

**begin_end_level2() - 실행 결과**
```
[3f902f0b] hello1
[3f902f0b] |-->hello2
[3f902f0b] |<--hello2 time=2ms
[3f902f0b] hello1 time=6ms
```

**begin_exception_level2() - 실행 결과**
```
[3dd9e4f1] hello
[3dd9e4f1] |-->hello2
[3dd9e4f1] |<X-hello2 time=3ms ex=java.lang.IllegalStateException
[3dd9e4f1] hello time=8ms ex=java.lang.IllegalStateException
```

멀티쓰레드 상황에서 문제가 없는지는 애플리케이션에 `ThreadLocalLogTrace`를 적용해서 확인해보자. 


</details>


<details> <summary> 8. 쓰레드 로컬 동기화 - 적용 </summary>

## 8. 쓰레드 로컬 동기화 - 적용

**LogTraceConfig - 수정**
```java
package hello.advanced;
import hello.advanced.trace.logtrace.LogTrace;
import hello.advanced.trace.logtrace.ThreadLocalLogTrace;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
@Configuration
public class LogTraceConfig {
 @Bean
 public LogTrace logTrace() {
 //return new FieldLogTrace();
 return new ThreadLocalLogTrace();
 }
}
```
동시성 문제가 있는 `FieldLogTrace` 대신에 문제를 해결한 `ThreadLocalLogTrace`를 스프링 빈으로 등록하자.

**정상 실행**
- http://localhost:8080/v3/request?itemId=hello

**정상 실행 로그**
```
[f8477cfc] OrderController.request()
[f8477cfc] |-->OrderService.orderItem()
[f8477cfc] | |-->OrderRepository.save()
[f8477cfc] | |<--OrderRepository.save() time=1004ms
[f8477cfc] |<--OrderService.orderItem() time=1006ms
[f8477cfc] OrderController.request() time=1007ms
```

**예외 실행**
- http://localhost:8080/v3/request?itemId=ex

**예외 실행 로그**
```
[c426fcfc] OrderController.request()
[c426fcfc] |-->OrderService.orderItem()
[c426fcfc] | |-->OrderRepository.save()
[c426fcfc] | |<X-OrderRepository.save() time=0ms
ex=java.lang.IllegalStateException: 예외 발생!
[c426fcfc] |<X-OrderService.orderItem() time=7ms
ex=java.lang.IllegalStateException: 예외 발생!
[c426fcfc] OrderController.request() time=7ms
ex=java.lang.IllegalStateException: 예외 발생!
```

### 동시 요청 
**동시성 문제 확인**  
다음 로직을 1초 안에 2번 실행해보자. 
- http://localhost:8080/v3/request?itemId=hello
- http://localhost:8080/v3/request?itemId=hello

**실행 결과**
```
[nio-8080-exec-3] [52808e46] OrderController.request()
[nio-8080-exec-3] [52808e46] |-->OrderService.orderItem()
[nio-8080-exec-3] [52808e46] | |-->OrderRepository.save()
[nio-8080-exec-4] [4568423c] OrderController.request()
[nio-8080-exec-4] [4568423c] |-->OrderService.orderItem()
[nio-8080-exec-4] [4568423c] | |-->OrderRepository.save()
[nio-8080-exec-3] [52808e46] | |<--OrderRepository.save() time=1001ms
[nio-8080-exec-3] [52808e46] |<--OrderService.orderItem() time=1001ms
[nio-8080-exec-3] [52808e46] OrderController.request() time=1003ms
[nio-8080-exec-4] [4568423c] | |<--OrderRepository.save() time=1000ms
[nio-8080-exec-4] [4568423c] |<--OrderService.orderItem() time=1001ms
[nio-8080-exec-4] [4568423c] OrderController.request() time=1001ms
```

**로그 분리해서 확인하기**
```
[nio-8080-exec-3]
[nio-8080-exec-3] [52808e46] OrderController.request()
[nio-8080-exec-3] [52808e46] |-->OrderService.orderItem()
[nio-8080-exec-3] [52808e46] | |-->OrderRepository.save()
[nio-8080-exec-3] [52808e46] | |<--OrderRepository.save() time=1001ms
[nio-8080-exec-3] [52808e46] |<--OrderService.orderItem() time=1001ms
[nio-8080-exec-3] [52808e46] OrderController.request() time=1003ms
[nio-8080-exec-4]
[nio-8080-exec-4] [4568423c] OrderController.request()
[nio-8080-exec-4] [4568423c] |-->OrderService.orderItem()
[nio-8080-exec-4] [4568423c] | |-->OrderRepository.save()
[nio-8080-exec-4] [4568423c] | |<--OrderRepository.save() time=1000ms
[nio-8080-exec-4] [4568423c] |<--OrderService.orderItem() time=1001ms
[nio-8080-exec-4] [4568423c] OrderController.request() time=1001ms
```

로그를 직접 분리해서 확인해보면 각각의 쓰레드 `nio-8080-exec-3`, `nio-8080-exec-4` 별로 로그가 정확하게 나누어진 것을 확인할 수 있다. 

</details>


<details> <summary> 9. 쓰레드 로컬 - 주의 사항 </summary>

## 9. 쓰레드 로컬 - 주의 사항

쓰레드 로컬의 값을 사용 후 제거하지 않고 그냥 두면 WAS(톰캣)처럼 쓰레드 풀을 사용하는 경우에 심각한 문제가 발생할 수 있다.  
다음 예시를 통해서 알아보자. 

**사용자 A 저장 요청**  
![image](https://user-images.githubusercontent.com/28394879/140542343-8c6aba7b-cba1-4263-8735-beb92cfd3409.png)
1. 사용자 A가 저장 HTTP를 요청했다.
2. WAS는 쓰레드 풀에서 쓰레드를 하나 조회한다.
3. 쓰레드 `thread-A`가 할당되었다.
4. `thread-A`는 `사용자A`의 데이터를 쓰레드 로컬에 저장한다.
5. 쓰레드 로컬의 `thread-A` 전용 보관소에 `사용자A` 데이터를 보관한다. 

**사용자 A 저장 요청 종료**
![image](https://user-images.githubusercontent.com/28394879/140542562-94a81298-ec27-4779-b22d-b3acfd1b8d89.png)
1. 사용자 A의 HTTP 응답이 끝난다.
2. WAS는 사용이 끝난 `thread-A`를 쓰레드 풀에 반환한다. 쓰레드를 생성하는 비용은 비싸기 때문에 쓰레드를 제거하지 않고, 보통 쓰레드 풀을 통해서 쓰레드를 재사용한다.
3. `thread-A`는 쓰레드풀에 아직 살아 있다. 따라서 쓰레드 로컬의 `thread-A` 전용 보관소에 `사용자A` 데이터도 함께 살아있게 된다.

**사용자 B 조회 요청**
![image](https://user-images.githubusercontent.com/28394879/140542808-f4f8bc71-570b-42e7-b931-c7ab4225727b.png)
1. 사용자B가 조회를 위한 새로운 HTTP 요청을 한다.
2. WAS는 쓰레드 풀에서 쓰레드를 하나 조회한다.
3. 쓰레드 thread-A 가 할당되었다. (물론 다른 쓰레드가 할당될 수 도 있다.)
4. 이번에는 조회하는 요청이다. thread-A 는 쓰레드 로컬에서 데이터를 조회한다.
5. 쓰레드 로컬은 thread-A 전용 보관소에 있는 사용자A 값을 반환한다.
6. 결과적으로 사용자A 값이 반환된다.
7. 사용자B는 사용자A의 정보를 조회하게 된다.

결과적으로 사용자B는 사용자A의 데이터를 확인하게 되는 심각한 문제가 발생하게 된다.  
이런 문제를 예방하려면 사용자A의 요청이 끝날 떄 쓰레드 로컬의 값을 `ThreadLocal.remove()`를 통해서 꼭 제거해야 한다.  
쓰레드 로컬을 사용할 때는 이 부분을 꼭 기억해야 한다. 


</details>


# [3. 템플릿 메서드 패턴과 콜백 패턴](./3.template-method-pattern-and-callback-pattern)

<details> <summary> 1. 템플릿 메서드 패턴 - 시작 </summary>

## 1. 템플릿 메서드 패턴 - 시작 

지금까지 로그 추적기를 열심히 잘 만들었다. 요구사항도 만족하고, 파라미터를 넘기는 불편함을 제거하기 위해 쓰레드 로컬도 도입했다.  
그런데 로그 추적기를 막상 프로젝트에 도입하려고 하니 개발자들의 반대의 목소리가 높다.  
로그 추적기 도입 전과 도입 후의 코드를 비교해보자 

**로그 추적기 도입 전 - V0 코드**
```java
//OrderControllerV0 코드
@GetMapping("/v0/request")
public String request(String itemId) {
 orderService.orderItem(itemId);
 return "ok";
}
//OrderServiceV0 코드
public void orderItem(String itemId) {
 orderRepository.save(itemId);
}
```

**로그 추적기 도입 후 - V3 코드**
```java
//OrderControllerV3 코드
@GetMapping("/v3/request")
public String request(String itemId) {
 TraceStatus status = null;
 try {
 status = trace.begin("OrderController.request()");
 orderService.orderItem(itemId); //핵심 기능
 trace.end(status);
 } catch (Exception e) {
 trace.exception(status, e);
 throw e;
 }
 return "ok";
}
//OrderServiceV3 코드
public void orderItem(String itemId) {
 TraceStatus status = null;
 try {
 status = trace.begin("OrderService.orderItem()");
 orderRepository.save(itemId); //핵심 기능
 trace.end(status);
 } catch (Exception e) {
 trace.exception(status, e);
 throw e;
 }
}
```

V0 시절 코드와 비교해서 V3 코드를 보자.  
V0는 해당 메서드가 실제 처리해야 하는 핵심 기능만 깔끔하게 남아있다. 반면에 V3에는 핵심 기능보다 로그를 출력해야 하는 부가 기능 코드가 훨씬 더 많고 복잡하다.  
앞으로 코드를 설명할 때 핵심 기능과 부가 기능으로 구분해서 정리하겠다.

**핵심 기능 vs 부가 기능**
- 핵심 기능
  - 해당 객체가 제공하는 고유의 기능 
  - 예) `orderService`의 핵심 기능은 주문 로직이다. 
  - 메서드 단위로 보면 `orderService.orderItem()`의 핵심 기능은 주문 데이터를 저장하기 위해 리포지토리를 호출하는 `orderRepository.save(itemId)`코드가 핵심 기능이다.
- 부가 기능
  - 핵심 기능을 보조하기 위해 제공되는 기능 
  - 예) 로그 추적 로직, 트랜잭션 기능 
  - 단독으로 사용되지 않고, 핵심 기능과 함께 사용된다. 
  - 예) 로그 추적기능은 어떤 핵심 기능이 호출되었는지 로그를 남기기 위해 사용한다. 그러니까 핵심 기능을 보조하기 위해 존재한다.

V0는 핵심 기능만 있지만, 로그 추적기를 추가한 V3 코드는 핵심 기능과 부가 기능이 함께 섞여있다.  
V3를 보면 로그 추적기의 도입으로 ㅎ개심 기능 코드보다 부가 기능을 처리하기 위한 코드가 더 많아졌다.  
소위 배보다 배꼽이 더 큰 상황이다. 만약 클래스가 수백 개라면 어떻게 하겠는가?  


이 문제를 좀 더 효율적으로 처리할 수 있는 방법이 있을까?  
V3 코드를 유심히 잘 살펴보면 다음과 같이 동일한 패턴이 있다.  
```java
TraceStatus status = null;
try {
 status = trace.begin("message");
 //핵심 기능 호출
 trace.end(status);
} catch (Exception e) {
 trace.exception(status, e);
 throw e;
}
```
`Controller`,`Service`,`Repository`의 코드를 잘 보면, 로그 추적기를 사용하는 구조는 모두 동일하다.  
중간에 핵심 기능을 사용하는 코드만 다를 뿐이다.  
부가 기능과 관련된 코드가 중복이니 중복을 별도의 메서드로 뽑아내면 될 것 같다.  
그런데, `try ~ catch`는 물론이고 ,핵심 기능 부분이 중간에 있어서 단순하게 메서드로 추출하는 것은 어렵다.  

**변하는 것과 변하지 않는 것을 분리**  
좋은 설계는 변하는 것과 변하지 않는 것을 분리하는 것이다.  
여기서 핵심 기능 부분은 변하고, 로그 추적기를 사용하는 부분은 변하지 않는 부분이다.  
이 둘을 분리해서 모듈화 해야 한다.  

템플릿 메서드 패턴(Template Method Pattern)은 이런 문제를 해결하는 디자인 패턴이다.



</details>

<details> <summary> 2. 템플릿 메서드 패턴 - 예제1 </summary>

## 2. 템플릿 메서드 패턴 - 예제1

템플릿 메서드 패턴을 쉽게 이해하기 위해 단순한 예제 코드를 만들어보자 

**TemplateMethodTest**
```java
package hello.advanced.trace.template;
import lombok.extern.slf4j.Slf4j;
import org.junit.jupiter.api.Test;
@Slf4j
public class TemplateMethodTest {
 @Test
 void templateMethodV0() {
 logic1();
 logic2();
 }
 private void logic1() {
 long startTime = System.currentTimeMillis();
 //비즈니스 로직 실행
 log.info("비즈니스 로직1 실행");
 //비즈니스 로직 종료
 long endTime = System.currentTimeMillis();
 long resultTime = endTime - startTime;
 log.info("resultTime={}", resultTime);
 }
 private void logic2() {
 long startTime = System.currentTimeMillis();
 //비즈니스 로직 실행
 log.info("비즈니스 로직2 실행");
 //비즈니스 로직 종료
 long endTime = System.currentTimeMillis();
 long resultTime = endTime - startTime;
 log.info("resultTime={}", resultTime);
 }
}
```
`logic1()`,`logic2()`를 호출하는 단순한 테스트 코드이다.

**실행 결과**
```
비즈니스 로직1 실행
resultTime=5
비즈니스 로직2 실행
resultTime=1
```

`logic1()`과 `logic2()`는 시간을 측정하는 부분과 비즈니스 로직을 실행하는 부분이 함께 존재한다.

- 변하는 부분: 비즈니스 로직
- 변하지 않는 부분: 시간 측정

이제 템플릿 메서드 패턴ㅇ르 사용해서 변하는 부분과 변하지 않는 부분을 분리해보자.




</details>

<details> <summary> 3. 템플릿 메서드 패턴 - 예제2 </summary>

## 3. 템플릿 메서드 패턴 - 예제2

**템플릿 메서드 패턴 구조 그림**  
![image](https://user-images.githubusercontent.com/28394879/140595800-4311a773-1bb6-4d29-bac9-f11582999dd6.png)


**AbstractTemplate**  
주의: 테스트 코드(src/test)에 위치한다.
```java
package hello.advanced.trace.template.code;
import lombok.extern.slf4j.Slf4j;
@Slf4j
public abstract class AbstractTemplate {
 public void execute() {
 long startTime = System.currentTimeMillis();
 //비즈니스 로직 실행
 call(); //상속
 //비즈니스 로직 종료
 long endTime = System.currentTimeMillis();
 long resultTime = endTime - startTime;
 log.info("resultTime={}", resultTime);
 }
 protected abstract void call();
}
```

템플릿 메서드 패턴은 이름 그대로 템플릿을 사용하는 방식이다. 템플릿은 기준이 되는 거대한 틀이다.  
템플릿이라는 틀에 변하지 않는 부분을 몰아둔다. 그리고 일부 변하는 부분을 별도로 호출해서 해결한다.  

`AbstractTemplate`코드를 보자. 변하지 않는 부분인 시간 측정 로직을 몰아둔 것을 확인할 수 있다.  
이제 이것의 하나의 템플릿이 된다. 그리고 템플릿 안에서 변하는 부분은 `call()` 메서드를 호출해서 처리한다.  
템플릿 메서드 패턴은 부모 클래스에 변하지 않는 템플릿 코드를 둔다. 그리고 변하는 부분은 자식 클래스에 두고 상속과 오버라이딩을 사용해서 처리한다.

**SubClassLogic1**  
주의: 테스트 코드 (src/test)에 위치한다.
```java
package hello.advanced.trace.template.code;
import lombok.extern.slf4j.Slf4j;
@Slf4j
public class SubClassLogic1 extends AbstractTemplate {
 @Override
 protected void call() {
 log.info("비즈니스 로직1 실행");
 }
}
```
변하는 부분인 비즈니스 로직1을 처리하는 자식 클래스이다. 템플릿이 호출하는 대상인 `call()`메서드를 오버라이딩 한다. 

**SubClassLogic2**  
주의: 테스트 코드 (src/test)에 위치한다.
```java
package hello.advanced.trace.template.code;
import lombok.extern.slf4j.Slf4j;
@Slf4j
public class SubClassLogic2 extends AbstractTemplate {
 @Override
 protected void call() {
 log.info("비즈니스 로직2 실행");
 }
}
```
변하는 부분인 비즈니스 로직2를 처리하는 자식 클래스이다. 템플릿이 호출하는 대상인 `call()`메서드를 오버라이딩 한다.

**TemplateMethodTest - templateMethodV1() 추가**
```java
/**
 * 템플릿 메서드 패턴 적용
 */
@Test
void templateMethodV1() {
 AbstractTemplate template1 = new SubClassLogic1();
 template1.execute();
 AbstractTemplate template2 = new SubClassLogic2();
 template2.execute();
}
```
템플릿 메서드 패턴으로 구현한 코드를 실행해보자.

**실행 결과**  
```
비즈니스 로직1 실행
resultTime=0
비즈니스 로직2 실행
resultTime=1
```

**템플릿 메서드 패턴 인스턴스 호출 그림**  
![image](https://user-images.githubusercontent.com/28394879/140595929-9ae574a1-4f08-481a-b968-361814118f4b.png)

`template1.execute()`를 호출하면 템플릿 로직인 `AbstractTemplate.execute()`를 실행한다.   
여기서 중간에 `call()`메서드를 호출하는데, 이 부분이 오버라이딩 되어 있다.  
따라서 현재 인스턴스인 `SubClassLogic1`인스턴스의 `SubClassLogic1.call()`메서드가 호출된다.

템플릿 메서드 패턴은 이렇게 다형성을 사용해서 변하는 부분과 변하지 않는 부분을 분리하는 방법이다. 



</details>

<details> <summary> 4. 템플릿 메서드 패턴 - 예제3 </summary>

## 4. 템플릿 메서드 패턴 - 예제3

**익명 내부 클래스 사용하기**  
템플릿 메서드 패턴은 `SubClassLogic1`, `SubClassLogic2`처럼 클래스를 계속 만들어야 하는 단점이 있다.  
익명 내부 클래스를 사용하면 이런 단점을 보완할 수 있다.  
익명 내부 클래슬르 사용하면 객체 인스턴스를 생성하면서 동시에 생성한 클래스를 상속 받은 자식 클래스를 정의할 수 있다.  
이 클래스는 `SubClassLogic1`처럼 직접 지정하는 이름이 없고 클래스 내부에 선언되는 클래스여서 익명 내부 클래스라 한다.  
익명 내부 클래스에 대한 자세한 내용은 자바 기본 문법을 참고하자. 

**TemplateMethodTest - templateMethodV2() 추가**
```java
/**
 * 템플릿 메서드 패턴, 익명 내부 클래스 사용
 */
@Test
void templateMethodV2() {
 AbstractTemplate template1 = new AbstractTemplate() {
 @Override
 protected void call() {
 log.info("비즈니스 로직1 실행");
 }
 };
 log.info("클래스 이름1={}", template1.getClass());
 template1.execute();
 AbstractTemplate template2 = new AbstractTemplate() {
 @Override
 protected void call() {
 log.info("비즈니스 로직1 실행");
 }
 };
 log.info("클래스 이름2={}", template2.getClass());
 template2.execute();
}
```

**실행 결과**
```
클래스 이름1 class hello.advanced.trace.template.TemplateMethodTest$1
비즈니스 로직1 실행
resultTime=3
클래스 이름2 class hello.advanced.trace.template.TemplateMethodTest$2
비즈니스 로직2 실행
resultTime=0
```

실행 결과를 보면 자바가 임의로 만들어주는 익명 내부 클래스 이름은 `TemplateMethodTest$1`, `TemplateMethodTest$2`인 것을 확인할 수 있다. 



</details>

<details> <summary> 5. 템플릿 메서드 패턴 - 적용1 </summary>

## 5. 템플릿 메서드 패턴 - 적용1

이제 우리가 만든 애플리케이션의 로그 추적기 로직에 템플릿 메서드 패턴을 적용해보자. 

**AbstractTemplate**
```java
package hello.advanced.trace.template;
import hello.advanced.trace.TraceStatus;
import hello.advanced.trace.logtrace.LogTrace;
public abstract class AbstractTemplate<T> {
 private final LogTrace trace;
 public AbstractTemplate(LogTrace trace) {
 this.trace = trace;
 }
 public T execute(String message) {
 TraceStatus status = null;
 try {
 status = trace.begin(message);
 //로직 호출
 T result = call();
 trace.end(status);
 return result;
 } catch (Exception e) {
 trace.exception(status, e);
 throw e;
 }
 }
 protected abstract T call();
}
```

- `AbstractTemplate`은 템플릿 메서드 패턴에서 부모 클래스이고, 템플릿 역할을 한다.
- `<T>` 제네릭을 사용했다. 반환 타입을 정의한다.
- 객체를 생성할 때 내부에서 사용할 `LogTrace trace`를 전달 받는다.
- 로그에 출력할 `message`를 외부에서 파라미터로 전달 받는다.
- 템플릿 코드 중간에 `call()` 메서드를 통해서 변하는 부분을 처리한다.
- `abstract T call()`은 변하는 부분을 처리하는 메서드이다. 이 부분은 상속으로 구현해야 한다.

**v3 -> v4 복사**  
먼저 기존 프로젝트 코드를 유지하기 위해 v4 애플리케이션을 복사해서 만들자.

- `hello.advanced.app.v4` 패키지 생성
- 복사
  - `v3.OrderControllerV3` -> `v4.OrderControllerV4`
  - `v3.OrderServiceV3` -> `v4.OrderServiceV4`
  - `v3.OrderRepositoryV3` -> `v4.OrderRepositoryV4`
- 코드 내부 의존관계 클래스를 V4으로 변경
  - `OrderControllerV4`: `OrderServiceV3` -> `OrderServiceV4`
  - `OrderServiceV4`: `OrderRepositoryV3` -> `OrderRepositoryV4`
- `OrderControllerV4` 매핑 정보 변경
  - `@GetMapping("/v4/request")`
- `AbstractTemplate`을 사용하도록 코드 변경 

**OrderControllerV4**
```java
package hello.advanced.app.v4;
import hello.advanced.trace.logtrace.LogTrace;
import hello.advanced.trace.template.AbstractTemplate;
import lombok.RequiredArgsConstructor;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;
@RestController
@RequiredArgsConstructor
public class OrderControllerV4 {
 private final OrderServiceV4 orderService;
 private final LogTrace trace;
 @GetMapping("/v4/request")
 public String request(String itemId) {
 AbstractTemplate<String> template = new AbstractTemplate<>(trace) {
 @Override
 protected String call() {
 orderService.orderItem(itemId);
 return "ok";
 }
 };
 return template.execute("OrderController.request()");
 }
}
```

- `AbstractTemplate<String>`
  - 제네릭을 `String`으로 설정했다. 따라서 `AbstractTemplate`의 반환 타입은 `String`이 된다.
- 익명 내부 클래스
  - 익명 내부 클래스를 사용한다. 객체를 생성하면서 `AbstractTemplate`를 상속받은 자식 클래스를 정의했다.
  - 따라서 별도의 자식 클래스를 직접 만들지 않아도 된다.
- `template.execute("OrderController.request()")`
  - 템플릿을 실행하면서 로그를 남길 `message`를 전달한다. 

**OrderServiceV4**
```java
package hello.advanced.app.v4;
import hello.advanced.trace.logtrace.LogTrace;
import hello.advanced.trace.template.AbstractTemplate;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;
@Service
@RequiredArgsConstructor
public class OrderServiceV4 {
 private final OrderRepositoryV4 orderRepository;
 private final LogTrace trace;
 public void orderItem(String itemId) {
 AbstractTemplate<Void> template = new AbstractTemplate<>(trace) {
 @Override
 protected Void call() {
 orderRepository.save(itemId);
 return null;
 }
 };
 template.execute("OrderService.orderItem()");
 }
}
```

- `AbstractTemplate<Void>`
  - 제네릭에서 반환 타입이 필요한데, 반환할 내용이 없으면 `Void`타입을 사용하고 `null`을 반환하면 된다. 참고로 제네릭은 기본 타입인 `void`,`int`등을 선언할 수 없다.

**OrderRepositoryV4**
```java
package hello.advanced.app.v4;
import hello.advanced.trace.logtrace.LogTrace;
import hello.advanced.trace.template.AbstractTemplate;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Repository;
@Repository
@RequiredArgsConstructor
public class OrderRepositoryV4 {
 private final LogTrace trace;
 public void save(String itemId) {
 AbstractTemplate<Void> template = new AbstractTemplate<>(trace) {
 @Override
 protected Void call() {
 //저장 로직
 if (itemId.equals("ex")) {
 throw new IllegalStateException("예외 발생!");
 }
 sleep(1000);
 return null;
 }
 };
 template.execute("OrderRepository.save()");
 }
 private void sleep(int millis) {
 try {
 Thread.sleep(millis);
 } catch (InterruptedException e) {
 e.printStackTrace();
 }
 }
}
``` 

</details>

<details> <summary> 6. 템플릿 메서드 패턴 - 적용2 </summary>

## 6. 템플릿 메서드 패턴 - 적용2

템플릿 메서드 패턴 덕분에 변하는 코드와 변하지 않는 코드를 명확하게 분리했다.  
로그 출력, 템플릿 역할을 하고 변하지 않는 코드 모두 `AbstractTemplate`에 담아두고, 변하는 코드는 자식클래스를 만들어서 분리했다.

**지금까지 작성한 코드를 비교해보자**
```java
//OrderServiceV0 코드
public void orderItem(String itemId) {
 orderRepository.save(itemId);
}
//OrderServiceV3 코드
public void orderItem(String itemId) {
 TraceStatus status = null;
 try {
 status = trace.begin("OrderService.orderItem()");
 orderRepository.save(itemId); //핵심 기능
 trace.end(status);
 } catch (Exception e) {
 trace.exception(status, e);
 throw e;
 }
}
//OrderServiceV4 코드
AbstractTemplate<Void> template = new AbstractTemplate<>(trace) {
 @Override
 protected Void call() {
 orderRepository.save(itemId);
 return null;
 }
};
template.execute("OrderService.orderItem()");
```

- `OrderServiceV0`: 핵심 기능만 있다.
- `OrderServiceV3`: 핵심 기능과 부가 기능이 함께 섞여 있다.
- `OrderServiceV4`: 핵심 기능과 템플릿을 호출하는 코드가 섞여 있다.

V4는 템플릿 메서들 패턴을 사용한 덕분에 핵심 기능에 좀 더 집중할 수 있게 되었다.

**좋은 설계란?**  
- 좋은 설계란 **변경**이 일어날때 자연스럽게 드러난다.
- 지금까지 로그를 남기는 부분을 모아서 하나로 모듈화하고, 비즈니스 로직 부분을 분리했다.
- 여기서 만약 로그를 남기는 로직을 변경해야 한다고 생각해보자.
- 그래서 `AbstractTemplate` 코드를 변경해야 한다 가정해보자.
- 단순히 `AbstractTemplate` 코드만 변경하면 된다.
- 템플릿이 없는 `V3` 상태에서 로그를 남기는 로직을 변경해야 한다고 생각해보자. 
- 이 경우 모든 클래스를 다 찾아서 고쳐야 한다. 
- 클래스가 수백 개라면 생각만해도 끔찍한 상황이다.

**단일 책임 원칙(SRP)**
- `V4`는 단순히 템플릿 메서드 패턴을 적용해서 소스코드 몇줄을 줄인 것이 전부가 아니다.
- 로그를 남기는 부분에 대한 단일 책임 원칙(SRP)을 지킨것이다.
- 변경 지점을 하나로 모아서 변경에 쉽게 대처할 수 있는 구조를 만든 것이다. 



</details>

<details> <summary> 7. 템플릿 메서드 패턴 - 정의 </summary>

## 7. 템플릿 메서드 패턴 - 정의

GOF 디자인 패턴에서는 템플릿 메서드 패턴을 다음과 같이 정의했다.

> 템플릿 메서드 디자인 패턴의 목적은 다음과 같다  
> "작업에서 알고리즘의 골격을 정의하고 일분 단계를 하위 클래스로 연기합니다. 템플릿 메서드를 사용하면  
> 하위 클래스가 알고리즘의 구조를 변경하지 않고도 알고리즘의 특정 단게를 재정의 할 수 있다. [GOF]

**GOF 템플릿 메서드 패턴 정의**  
![image](https://user-images.githubusercontent.com/28394879/140649861-e9b4bbcc-0923-4201-ad57-b4f09f7a84fd.png)

풀어서 설명하면 다음과 같다.  
부모 클래스에 알고리즘의 골격인 템플릿을 정의하고, 일부 변경되는 로직은 자식 클래스에 정의하는 것이다.  
이렇게 하면 자식 클래스가 알고리즘의 전체 구조를 변경하지 않고, 특정 부분만 재정의할 수 있다.  
결국 상속과 오버라이딩을 통한 다형성으로 문제를 해결하는 것이다.

**하지만**  
템플릿 메서드 패턴은 상속을 사용한다. 따라서 상속에서 오는 단점들을 그대로 안고간다.  
특히 자식 클래스가 부모 클래스의 컴파일 시점에 강하게 결합되는 문제가 있다.   
이것은 의존관계에 대한 문제이다.  
자식 클래스 입장에서는 부모 클래스의 기능을 전혀 사용하지 않는다.  
이번 장에서 지금까지 작성했던 코드를 떠올려보자. 자식 클래스를 작성할 떄 부모 클래스의 기능을 사용한것이 있었던가?  
그럼에도 불구하고 템플릿 메서드 패턴을 위해 자식 클래스는 부모 클래스를 상속 받고 있다.  

상속을 받는다는 것은 특정 부모 클래스를 의존하고 있다는 것이다. 자식 클래스의 `extends`다음에 바로 부모 클래스가 코드상에 지정되어 있다.  
따라서 부모 클래스의 기능을 사용하든 사용하지 않든간에 부모 클래스를 강하게 의존하게 된다.  
여기서 강하게 의존한다는 뜻은 자식 클래스의 코드에 부모 클래스의 코드가 명확하게 적혀 있다는 뜻이다.  
UML에서 상속을 받으면 삼각형 화살표가 `자식 -> 부모`를 향하고 있는 것은 이런 의존관계를 반영하는 것이다.

자식 클래스 입장에서는 부모 클래스의 기능을 전혀 사용하지 않는데, 부모 클래스를 알아야한다. 이것은 좋은 설계가 아니다.  
그리고 이런 잘못된 의존관계 떄문에 부모 클래스를 수정하면, 자식 클래스에도 영향을 줄 수 있다.  

추가로 템플릿 메서드 패턴은 상속 구조를 사용하기 때문에, 별도의 클래스나 익명 내부 클래스를 만들어야 하는 부분도 복잡하다.  
지금까지 설명한 이런 부분들을 더 깔끔하게 개선하려면 어떻게 해야할까?  

템플릿 메서드 패턴과 비슷한 역할을 하면서 상속의 단점을 제거할 수 있는 디자인 패턴이 바로 전략 패턴(Strategy Pattern)이다.


</details>

<details> <summary> 8. 전략 패턴 - 시작 </summary>

## 8. 전략 패턴 - 시작

전략 패턴의 이해를 돕기 위해 템플릿 메서드 패턴에서 만들었던 동일한 예제를 만들어보자.

**ContextV1Text**
```java
package hello.advanced.trace.strategy;
import lombok.extern.slf4j.Slf4j;
import org.junit.jupiter.api.Test;
@Slf4j
public class ContextV1Test {
 @Test
 void strategyV0() {
 logic1();
 logic2();
 }
 private void logic1() {
 long startTime = System.currentTimeMillis();
 //비즈니스 로직 실행
 log.info("비즈니스 로직1 실행");
 //비즈니스 로직 종료
 long endTime = System.currentTimeMillis();
 long resultTime = endTime - startTime;
 log.info("resultTime={}", resultTime);
 }
 private void logic2() {
 long startTime = System.currentTimeMillis();
 //비즈니스 로직 실행
 log.info("비즈니스 로직2 실행");
 //비즈니스 로직 종료
 long endTime = System.currentTimeMillis();
 long resultTime = endTime - startTime;
 log.info("resultTime={}", resultTime);
 }
}
```

잘 동작하면 동일한 문제를 전략 패턴으로 풀어보자.

**실행 결과**
```
비즈니스 로직1 실행
resultTime=5
비즈니스 로직2 실행
resultTime=1
```

</details>

<details> <summary> 9. 전략 패턴 - 예제1 </summary>

## 9. 전략 패턴 - 예제1

이번에는 동일한 문제를 전략 패턴을 사용해서 해결해보자.  
템플릿 메서드 패턴은 부모 클래스에 변하지 않는 템플릿을 두고, 변하는 부분을 자식 클래스에 두어서 상속을 사용해서 문제를 해결한 것이다.  
전략 패턴은 변하지 않는 부분을 `Context`라는 곳에 두고, 변하는 부분을 `Strategy`라는 인터페이스를 만들고 해당 인터페이스를 구현하도록 문제를 해결한다.  
상속이 아니라 위임으로 문제를 해결하는 것이다.  
전략 패턴에서 `Context`는 변하지 않는 템플릿 역할을 하고, `Strategy`는 변하는 알고리즘 역할을 한다. 

GOF 디자인 패턴에서 정의한 전략 패턴의 의도는 다음과 같다.  
> 알고리즘 제품군을 정의하고 각각을 캡슐화하여 상호 교환 가능하게 만들자. 전략을 사용하면 알고리즘을 사용하는 클라이언트와  
> 독립적으로 알고리즘을 변경할 수 있다.  

![image](https://user-images.githubusercontent.com/28394879/140736668-251f4df8-2f3c-4d25-880f-fdbfcfc396b8.png)

**Strategy 인터페이스**  
주의: 테스트 코드 (src/test/)에 위치한다.
```java
package hello.advanced.trace.strategy.code.strategy;
public interface Strategy {
 void call();
}
```
이 인터페이스는 변하는 알고리즘 역할을 한다.

**StrategyLogic1**  
주의: 테스트 코드 패키지에 위치한다.
```java
package hello.advanced.trace.strategy.code.strategy;
import lombok.extern.slf4j.Slf4j;
@Slf4j
public class StrategyLogic1 implements Strategy {
 @Override
 public void call() {
 log.info("비즈니스 로직1 실행");
 }
}
```
변하는 알고리즘은 `Strategy` 인터페이스를 구현하면 된다. 여기서는 비즈니스 로직1을 구현했다.

**StrateLogic2**  
주의: 테스트 코드 패키지에 위치한다.
```java
package hello.advanced.trace.strategy.code.strategy;
import lombok.extern.slf4j.Slf4j;
@Slf4j
public class StrategyLogic2 implements Strategy {
 @Override
 public void call() {
 log.info("비즈니스 로직2 실행");
 }
}
```
비즈니스 로직2를 구현했다. 

**ContextV1**  
주의: 테스트 코드 패키지에 위치한다. 
```java
package hello.advanced.trace.strategy.code.strategy;
import lombok.extern.slf4j.Slf4j;
/**
 * 필드에 전략을 보관하는 방식
 */
@Slf4j
public class ContextV1 {
 private Strategy strategy;
 public ContextV1(Strategy strategy) {
 this.strategy = strategy;
 }
 public void execute() {
 long startTime = System.currentTimeMillis();
 //비즈니스 로직 실행
 strategy.call(); //위임
 //비즈니스 로직 종료
 long endTime = System.currentTimeMillis();
 long resultTime = endTime - startTime;
 log.info("resultTime={}", resultTime);
 }
}
```
`ContextV1`은 변하지 않는 로직을 가지고 있는 템플릿 역할을 하는 코드이다. 전략 패턴에서는 이것을 컨텍스트(문맥)이라 한다.  
쉽게 이야기해서 컨텍스트(문맥)는 크게 변하지 않지만, 그 문맥 속에서 `strategy`를 통해 일부 전략이 변경된다 생각하면 된다.  

`Context`는 내부에 `Strategy strategy`필드를 가지고 있다. 이 필드에 변하는 부분인 `Strategy`의 구현체를 주입하면 된다.   
전략 패턴의 핵심은 `Context`는 `Strategy` 인터페이스에만 의존한다는 점이다. 덕분에 `Strategy`의 구현체를 변경하거나 새로 만들어도 `Context` 코드에는 영향을 주지 않는다.  

어디서 많이 본 코드 같지 않은가? 그렇다. 바로 스프링에서 의존관계 주입에서 사용하는 방식이 바로 전략 패턴이다.  

**ContextV1Test - 추가**
```java
/**
 * 전략 패턴 적용
 */
@Test
void strategyV1() {
 Strategy strategyLogic1 = new StrategyLogic1();
 ContextV1 context1 = new ContextV1(strategyLogic1);
 context1.execute();
 Strategy strategyLogic2 = new StrategyLogic2();
 ContextV1 context2 = new ContextV1(strategyLogic2);
 context2.execute();
}
```
전략 패턴을 사용해보자.  
코드를 보면 의존관계 주입을 통해 `ContextV1`에 `Strategy`의 구현체인 `strategyLogic1`를 주입하는 것을 확인할 수 있다.  
이렇게해서 `Context`안에 원하는 전략을 주입한다. 이렇게 원하는 모양으로 조립을 완료하고 난 다음에 `context1.execute()`를 호출해서 `context`를 실행한다. 

**전략 패턴 실행 그림**  
![image](https://user-images.githubusercontent.com/28394879/140737368-d376bf85-15b2-4b5d-ae45-5f666a5e7edf.png)
1. `Context`에 원하는 `Strategy` 구현체를 주입한다.
2. 클라이언트는 `context`를 실행한다.
3. `context`는 `context`로직을 시작한다.
4. `context`로직 중간에 `strategy.call()`을 호출해서 주입 받는 `strategy` 로직을 실행한다.
5. `context`는 나머지 로직을 실행한다.

**실행 결과**
```
StrategyLogic1 - 비즈니스 로직1 실행
ContextV1 - resultTime=3
StrategyLogic2 - 비즈니스 로직2 실행
ContextV1 - resultTime=0
```


</details>

<details> <summary> 10. 전략 패턴 - 예제2 </summary>

## 10. 전략 패턴 - 예제2

전략 패턴도 익명 내부 클래스를 사용할 수 있다. 

**ContextV1Test - 추가**
```java
/**
 * 전략 패턴 익명 내부 클래스1
 */
@Test
void strategyV2() {
 Strategy strategyLogic1 = new Strategy() {
 @Override
 public void call() {
 log.info("비즈니스 로직1 실행");
 }
 };
 log.info("strategyLogic1={}", strategyLogic1.getClass());
 ContextV1 context1 = new ContextV1(strategyLogic1);
 context1.execute();
 Strategy strategyLogic2 = new Strategy() {
 @Override
 public void call() {
 log.info("비즈니스 로직2 실행");
 }
 };
 log.info("strategyLogic2={}", strategyLogic2.getClass());
 ContextV1 context2 = new ContextV1(strategyLogic2);
 context2.execute();
}
```

**실행 결과**
```
ContextV1Test - strategyLogic1=class
hello.advanced.trace.strategy.ContextV1Test$1
ContextV1Test - 비즈니스 로직1 실행
ContextV1 - resultTime=0
ContextV1Test - strategyLogic2=class
hello.advanced.trace.strategy.ContextV1Test$2
ContextV1Test - 비즈니스 로직2 실행
ContextV1 - resultTime=0
```
실행 결과를 보면 `ContextV1Test$1`, `ContextV1Test$2`와 같이 익명 내부 클래스가 생성된 것을 확인할 수 있다.

**ContextV1Test - 추가**
```java
/**
 * 전략 패턴 익명 내부 클래스2
 */
@Test
void strategyV3() {
 ContextV1 context1 = new ContextV1(new Strategy() {
 @Override
 public void call() {
 log.info("비즈니스 로직1 실행");
 }
 });
 context1.execute();
 ContextV1 context2 = new ContextV1(new Strategy() {
 @Override
 public void call() {
 log.info("비즈니스 로직2 실행");
 }
 });
 context2.execute();
}
```
익명 내부 클래스를 변수에 담아두지 말고, 생성하면서 바로 `ContextV1`에 전달해도 된다. 

**ContextV1Test - 추가**
```java
/**
 * 전략 패턴, 람다
 */
@Test
void strategyV4() {
 ContextV1 context1 = new ContextV1(() -> log.info("비즈니스 로직1 실행"));
 context1.execute();
 ContextV1 context2 = new ContextV1(() -> log.info("비즈니스 로직2 실행"));
 context2.execute();
}
```
익명 내부 클래스를 자바8부터 제공하는 람다로 변경할 수 있다. 람다로 변경하려면 인터페이스에 메서드가 1개만 있으면 되는데, 여기에서 제공하는 `Strategy` 인터페이스는 메서드가 1개만 있으므로 람다로 사용할 수 있다.  
람다에 대한 부분은 자바 기본 문법이므로 자바 문법 관련 내용을 찾아보자. 

**정리**  
지금까지 일반적으로 이야기하는 전략 패턴에 대해서 알아보았다. 변하지 않는 부분을 `Context`에 두고 변하는 부분을 `Strategy`를 구현해서 만든다. 그리고 `Context`의 내부 필드에 `Strategy`를 주입해서 사용했다.

**선 조립, 후 실행**  
여기서 이야기하고 싶은 부분은 `Context`의 내부 필드에 `Strategy`를 두고 사용하는 부분이다.  
이 방식은 `Context`와 `Strategy`를 실행 전에 원하는 모양으로 조립해두고, 그 다음에 `Context`를 실행하는 선 조립, 후 실행 방식에서 매우 유용하다.  
`Context`와 `Strategy`를 한번 조립하고 나면 이후로는 `Context`를 실행하기만 하면 된다.  
우리가 스프링으로 애플리케이션을 개발할 때 애플리케이션 로딩 시점에 의존관계 주입을 통해 필요한 의존관계를 모두 맺어두고 난 다음에 실제 요청을 처리하는 것 과 같은 원리이다.  

이 방식의 단점은 `Context`와 `Strategy`를 조립한 이후에는 전략을 변경하기가 번거롭다는 점이다. 물론 `Context`에 `setter`를 제공해서 `Strategy`를 넘겨 받아 변경하면 되지만, `Context`를 싱글톤으로 사용할 때는 동시성 이슈 등 고려할 점이 많다. 그래서 전략을 실시간으로 변경해야 하면 차라리 이전에 개발한 테스트 코드 처럼 `Context`를 하나더 생성하고 그곳에 다른 `Strategy`를 주입하는 것이 더 나은 선택일 수 있다.  

이렇게 먼저 조립하고 사용하는 방식보다 더 유연하게 전략 패턴을 사용하는 방법을 없을까? 

</details>

<details> <summary> 11. 전략 패턴 - 예제3 </summary>

## 11. 전략 패턴 - 예제3

이번에는 전략 패턴을 조금 다르게 사용해보자. 이전에는 `Context`의 필드에 `Strategy`를 주입해서 사용했다. 이번에는 전략을 실행할 때 직접 파라미터로 전달해서 사용해보자. 

**ContextV2**  
주의: 테스트 패키지에 위치 
```java
package hello.advanced.trace.strategy.code.strategy;
import lombok.extern.slf4j.Slf4j;
/**
 * 전략을 파라미터로 전달 받는 방식
 */
@Slf4j
public class ContextV2 {
 public void execute(Strategy strategy) {
 long startTime = System.currentTimeMillis();
 //비즈니스 로직 실행
 strategy.call(); //위임
 //비즈니스 로직 종료
 long endTime = System.currentTimeMillis();
 long resultTime = endTime - startTime;
 log.info("resultTime={}", resultTime);
 }
}
```
`ContextV2`는 전략을 필드로 가지지 않는다. 대신에 전략을 `execute(...)`가 호출될 떄 마다 항상 파라미터로 전달 받는다. 

**ContextV2Test**
```java
package hello.advanced.trace.strategy;
import hello.advanced.trace.strategy.code.strategy.ContextV2;
import hello.advanced.trace.strategy.code.strategy.Strategy;
import hello.advanced.trace.strategy.code.strategy.StrategyLogic1;
import hello.advanced.trace.strategy.code.strategy.StrategyLogic2;
import lombok.extern.slf4j.Slf4j;
import org.junit.jupiter.api.Test;
@Slf4j
public class ContextV2Test {
 /**
 * 전략 패턴 적용
 */
 @Test
 void strategyV1() {
 ContextV2 context = new ContextV2();
 context.execute(new StrategyLogic1());
 context.execute(new StrategyLogic2());
 }
}
```

`Context`와 `Strategy`를 선 조립 후 실행하는 방식이 아니라 `Context`를 실행할 때 마다 전략을 인수로 전달한다.  
클라이언트는 `Context`를 실행하는 시점에 원하는 `Strategy`를 전달할 수 있다. 따라서 이전 방식과 비교해서 원하는 전략을 더욱 유연하게 변경할 수 있다.  
테스트 코드를 보면 하나의 `Context`만 생성한다. 그리고 하나의 `Context`에 실행 시점에 여러 전략을 인수로 전달해서 유연하게 실행하는 것을 확인할 수 있다. 

**전략 패턴 파라미터 실행 그림**  
![image](https://user-images.githubusercontent.com/28394879/140746112-9b680db0-9f46-4297-a3b2-b3aa99c8acda.png)  
1. 클라이언트는 `Context`를 실행하면서 인수로 `Strategy`를 전달한다.
2. `Context`는 `execute()` 로직을 실행한다.
3. `Context`는 파라미터로 넘어온 `strategy.call()` 로직을 실행한다.
4. `Context`의 `execute()` 로직이 종료된다.

**ContextV2Test - 추가**
```java
/**
 * 전략 패턴 익명 내부 클래스
 */
@Test
void strategyV2() {
 ContextV2 context = new ContextV2();
 context.execute(new Strategy() {
 @Override
 public void call() {
 log.info("비즈니스 로직1 실행");
 }
 });
 context.execute(new Strategy() {
 @Override
 public void call() {
 log.info("비즈니스 로직2 실행");
 }
 });
}
```

여기도 물론 익명 내부 클래스를 사용할 수 있다. 코드 조각을 파라미터로 넘긴다고 생각하면 더 자연스럽다.

**ContextV2Test - 추가**
```java
/**
 * 전략 패턴 익명 내부 클래스2, 람다
 */
@Test
void strategyV3() {
 ContextV2 context = new ContextV2();
 context.execute(() -> log.info("비즈니스 로직1 실행"));
 context.execute(() -> log.info("비즈니스 로직2 실행"));
}
```
람다를 사용해서 코드를 더 단순하게 만들 수 있다.

**정리**  
- `ContextV1`은 필드에 `Strategy`를 저장하는 방식으로 전략 패턴을 구사했다.
  - 선 조립, 후 실행 방법에 적합하다.
  - `Context`를 실행하는 시점에는 이미 조립이 끝났기 떄문에 전략을 신경쓰지 않고 단순히 실행만 하면 된다.
- `ContextV2`는 파라미터에 `Strategy`를 전달하는 방식으로 전략 패턴을 구사했다.
  - 실행할 때 마다 전략을 유연하게 변경할 수 있다.
  - 단점 역시 실행할 때 마다 전략을 계속 지정해주어야 한다는 점이다.

**템플릿**  
지금 우리가 해결하고 싶은 문제는 변하는 부분과 변하지 않는 부분을 분리하는 것이다.  
변하지 않는 부분을 템플릿이라고 하고, 그 템플릿 안에서 변하는 부분에 약간 다른 코드 조각을 넘겨서 실행하는 것이 목적이다.  
`ContextV1`, `ContextV2` 두 가지 방식 다 문제를 해결할 수 있지만, 어떤 방식이 조금 더 나아 보이는가?  
지금 우리가 원하는 것은 애플리케이션 의존 관계를 설정하는 것처럼 선 조립, 후 실행이 아니다.  
단순히 코드를 실행할 때 변하지 않는 템플릿이 있고, 그 템플릿 안에서 원하는 부분만 살짝 다른 코드를 실행하고 싶을 뿐이다.  
따라서 우리가 고민하는 문제는 실행 시점에 유연하게 실행 코드 조각을 전달하는 `ContextV2`가 더 적합하다.


</details>

<details> <summary> 12. 템플릿 콜백 패턴 - 시작 </summary>

## 12. 템플릿 콜백 패턴 - 시작

`ContextV2`는 변하지 않는 템플릿 역할을 한다. 그리고 변하는 부분은 파라미터로 넘어온 `Strategy`의 코드를 실행해서 처리한다.  
이렇게 다른 코드의 인수로서 넘겨주는 실행 가능한 코드를 콜백(callback)이라고 한다.

> 콜백 정의  
> 프로그래밍에서 콜백(callback)또는 콜애프터 함수(call-after function)는 다른 코드의 인수로서   
> 넘겨주는 실행 가능한 코드를 말한다. 콜백을 넘겨받은 코드는 이 콜백을 필요에 따라 즉시 실행할 수도  
> 있고, 아니면 나중에 실행할 수도 있다. 

쉽게 이야기해서 `callback`은 코드가 호출(`call`)은 되는데 코드를 넘겨준 곳의 뒤(`back`)에서 실행된다는 뜻이다.
- `ContextV2`예제에서 콜백은 `Strategy`이다.
- 여기에서는 클라이언트에서 직접 `Strategy`를 실행하는 것이 아니라, 클라이언트가 `ContextV2.execute(..)`를 실행할 때 `Strategy`를 넘겨주고, `ContextV2`뒤에서 `Strategy`가 실행된다.

**자바 언어에서 콜백**  
- 자바 언어에서 실행 가능한 코드를 인수로 넘기려면 객체가 필요하다. 자바8부터는 람다를 사용할 수 있다.
- 자바 8 이전에는 보통 하나의 메소드를 가진 인터페이스를 구현하고, 주로 익명 내부 클래스를 사용했다.
- 최근에는 주로 람다를 사용한다.

**템플릿 콜백 패턴**  
- 스프링에서는 `ContextV2`와 같은 방식의 전략 패턴을 템플릿 콜백 패턴이라 한다. 전략 패턴에서 `Context`가 템플릿 역할을 하고, `Strategy`부분이 콜백으로 넘어온다 생각하면 된다.
- 참고로 템플릿 콜백 패턴은 GOF 패턴은 아니고, 스프링 내부에서 이런 방식을 자주 사용하기 때문에, 스프링 안에서만 이렇게 부른다. 전략 패턴에서 템플릿과 콜백 부분이 강조된 패턴이라 생각하면 된다.
- 스프링에서는 `JdbcTemplate`, `RestTemplate`, `TransactionTemplate`, `RedisTemplate`처럼 다양한 템플릿 콜백 패턴이 사용된다. 스프링에서 이름에 `XxxTemplate`가 있다면 템플릿 콜백 패턴으로 만들어져 있다 생각하면 된다. 

![image](https://user-images.githubusercontent.com/28394879/140760267-27a3ce2f-6565-47f1-8fb4-8a03bedaa864.png)




</details>

<details> <summary> 13. 템플릿 콜백 패턴 - 예제 </summary>

## 13. 템플릿 콜백 패턴 - 예제

템플릿 콜백 패턴을 구현해보자. `ContextV2`와 내용이 같고 이름만 다르므로 크게 어려움은 없을 것이다. 
- `Context` -> `Template`
- `Strategy` -> `Callback`

**Callback - 인터페이스**  
주의: 테스트 패키지에 위치
```java
package hello.advanced.trace.strategy.code.template;
public interface Callback {
 void call();
}
```
콜백 로직을 전달할 인터페이스이다.

**TimeLogTemplate**  
주의: 테스트 패키지에 위치 
```java
package hello.advanced.trace.strategy.code.template;
import lombok.extern.slf4j.Slf4j;
@Slf4j
public class TimeLogTemplate {
 public void execute(Callback callback) {
 long startTime = System.currentTimeMillis();
 //비즈니스 로직 실행
 callback.call(); //위임
 //비즈니스 로직 종료
 long endTime = System.currentTimeMillis();
 long resultTime = endTime - startTime;
 log.info("resultTime={}", resultTime);
 }
}
```

**TemplateCallbackTest**
```java
package hello.advanced.trace.strategy;
import hello.advanced.trace.strategy.code.template.Callback;
import hello.advanced.trace.strategy.code.template.TimeLogTemplate;
import lombok.extern.slf4j.Slf4j;
import org.junit.jupiter.api.Test;
@Slf4j
public class TemplateCallbackTest {
 /**
 * 템플릿 콜백 패턴 - 익명 내부 클래스
 */
 @Test
 void callbackV1() {
 TimeLogTemplate template = new TimeLogTemplate();
 template.execute(new Callback() {
 @Override
 public void call() {
 log.info("비즈니스 로직1 실행");
 }
 });
 template.execute(new Callback() {
 @Override
 public void call() {
 log.info("비즈니스 로직2 실행");
 }
 });
 }
 /**
 * 템플릿 콜백 패턴 - 람다
 */
 @Test
 void callbackV2() {
 TimeLogTemplate template = new TimeLogTemplate();
 template.execute(() -> log.info("비즈니스 로직1 실행"));
 template.execute(() -> log.info("비즈니스 로직2 실행"));
 }
}
```
별도의 클래스를 만들어서 전달해도 되지만, 콜백을 사용할 경우 익명 내부 클래스나 람다를 사용하는 것이 편리하다.  
물론 여러곳에서 함께 사용되는 경우 재사용을 위해 콜백을 별도의 클래스로 만들어도 된다. 

</details>

<details> <summary> 14. 템플릿 콜백 패턴 - 적용 </summary>

## 14. 템플릿 콜백 패턴 - 적용

이제 템플릿 콜백 패턴을 애플리케이션에 적용해보자.

**TraceCallback 인터페이스**
```java
package hello.advanced.trace.callback;
public interface TraceCallback<T> {
 T call();
}
```
- 콜백을 전달하는 인터페이스이다.
- `<T>` 제네릭을 사용했다. 콜백의 반환 타입을 정의한다.

**TraceTemplate**
```java
package hello.advanced.trace.callback;
import hello.advanced.trace.TraceStatus;
import hello.advanced.trace.logtrace.LogTrace;
public class TraceTemplate {
 private final LogTrace trace;
 public TraceTemplate(LogTrace trace) {
 this.trace = trace;
 }
 public <T> T execute(String message, TraceCallback<T> callback) {
 TraceStatus status = null;
 try {
 status = trace.begin(message);
 //로직 호출
 T result = callback.call();
 trace.end(status);
 return result;
 } catch (Exception e) {
 trace.exception(status, e);
 throw e;
 }
 }
}
```
- `TraceTemplate`는 템플릿 역할을 한다.
- `execute(..)`를 보면 `message` 데이터와 콜백인 `TraceCallback callback`을 전달 받는다.
- `<T>` 제네릭을 사용했다. 반환 타입을 정의한다.

**OrderControllerV5**
```java
package hello.advanced.app.v5;
import hello.advanced.trace.callback.TraceCallback;
import hello.advanced.trace.callback.TraceTemplate;
import hello.advanced.trace.logtrace.LogTrace;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;
@RestController
public class OrderControllerV5 {
 private final OrderServiceV5 orderService;
 private final TraceTemplate template;
 public OrderControllerV5(OrderServiceV5 orderService, LogTrace trace) {
 this.orderService = orderService;
 this.template = new TraceTemplate(trace);
 }
 @GetMapping("/v5/request")
 public String request(String itemId) {
 return template.execute("OrderController.request()", new
TraceCallback<>() {
 @Override
 public String call() {
 orderService.orderItem(itemId);
 return "ok";
 }
 });
 }
}
```
- `this.template = new TraceTemplate(trace)`: `trace` 의존관계 주입을 받으면서 필요한 `TraceTemplate` 템플릿을 생성한다. 참고로 `TraceTemplate`를 처음부터 스프링 빈으로 등록하고 주입 받아도 된다. 이부분은 선택이다.
- `template.execute(..., new TraceCallback(){..})`: 템플릿을 실행하면서 콜백을 전달한다. 여기서는 콜백으로 익명 내부 클래스를 사용했다. 

**OrderServiceV5**
```java
package hello.advanced.app.v5;
import hello.advanced.trace.callback.TraceTemplate;
import hello.advanced.trace.logtrace.LogTrace;
import org.springframework.stereotype.Service;
@Service
public class OrderServiceV5 {
 private final OrderRepositoryV5 orderRepository;
 private final TraceTemplate template;
 public OrderServiceV5(OrderRepositoryV5 orderRepository, LogTrace trace) {
 this.orderRepository = orderRepository;
 this.template = new TraceTemplate(trace);
 }
 public void orderItem(String itemId) {
 template.execute("OrderController.request()", () -> {
 orderRepository.save(itemId);
 return null;
 });
 }
}
```
- `template.execute(..., new TraceCallback(){..})`: 템플릿을 실행하면서 콜백을 전달한다. 여기서는 콜백으로 람다를 전달했다.

**OrderRepositoryV5**
```java
package hello.advanced.app.v5;
import hello.advanced.trace.callback.TraceTemplate;
import hello.advanced.trace.logtrace.LogTrace;
import org.springframework.stereotype.Repository;
@Repository
public class OrderRepositoryV5 {
 private final TraceTemplate template;
 public OrderRepositoryV5(LogTrace trace) {
 this.template = new TraceTemplate(trace);
 }
 public void save(String itemId) {
 template.execute("OrderRepository.save()", () -> {
 //저장 로직
 if (itemId.equals("ex")) {
 throw new IllegalStateException("예외 발생!");
 }
 sleep(1000);
 return null;
 });
 }
 private void sleep(int millis) {
 try {
 Thread.sleep(millis);
 } catch (InterruptedException e) {
 e.printStackTrace();
 }
 }
}
```
앞의 로직과 같다.


</details>

<details> <summary> 15. 정리 </summary>

## 15. 정리

지금까지 우리는 변하는 코드와 변하지 않는 코드를 분리하고, 더 적은 코드로 로그 추적기를 적용하기 위해
고군분투 했다.  
템플릿 메서드 패턴, 전략 패턴, 그리고 템플릿 콜백 패턴까지 진행하면서 변하는 코드와 변하지 않는 코드를
분리했다.  
그리고 최종적으로 템플릿 콜백 패턴을 적용하고 콜백으로 람다를 사용해서 코드 사용도 최소화
할 수 있었다.

**한계**
그런데 지금까지 설명한 방식의 한계는 아무리 최적화를 해도 결국 로그 추적기를 적용하기 위해서 원본
코드를 수정해야 한다는 점이다. 클래스가 수백개이면 수백개를 더 힘들게 수정하는가 조금 덜 힘들게
수정하는가의 차이가 있을 뿐, 본질적으로 코드를 다 수정해야 하는 것은 마찬가지이다.

개발자의 게으름에 대한 욕심은 끝이 없다. 수 많은 개발자가 이 문제에 대해서 집요하게 고민해왔고,
여러가지 방향으로 해결책을 만들어왔다. 지금부터 원본 코드를 손대지 않고 로그 추적기를 적용할 수 있는
방법을 알아보자. 그러기 위해서 프록시 개념을 먼저 이해해야 한다.

> 참고  
> 지금까지 설명한 방식은 스프링 안에서 많이 사용되는 방식이다.   
> `XxxTemplate`를 만나면 이번에 학습한 내용을 떠올려 보면 어떻게 돌아가는지 쉽게 이해할 수 있을 것이다.


</details>


# [4. 프록시 패턴과 데코레이터 패턴](./4.proxy-pattern-decorator-pattern)

<details> <summary> 1. 프로젝트 생성 </summary>

## 1. 프로젝트 생성

</details>

<details> <summary> 2. 예제 프로젝트 만들기 v1 </summary>

## 2. 예제 프로젝트 만들기 v1

다양한 상황에서 프록시 사용법을 이해하기 위해 다음과 같은 기준으로 기본 예제 프로젝트를 만들어보자. 

**예제는 크게 3가지 상황으로 만든다**
- v1 - 인터페이스와 구현 클래스 - 스프링 빈으로 수동 등록
- v2 - 인터페이스 없는 구체 클래스 - 스프링 빈으로 수동 등록 
- v3 - 컴포넌트 스캔으로 스프링 빈 자동 등록 

실무에서는 스프링 빈으로 등록할 클래스는 인터페이스가 있는 경우도 있고 없는 경우도 있다. 그리고 스프링 빈을 수동으로 직접 등록하는 경우도 있고, 컴포넌트 스캔으로 자동으로 등록하는 경우도 있다.   
이런 다양한 케이스에 프록시를 어떻게 적용하는지 알아보기 위해 다양한 예제를 준비해보자.


### v1 - 인터페이스와 구현 클래스 - 스프링 빈으로 수동 등록 
지금까지 보아왔던 `Controller`, `Service`, `Repository`에 인터페이스를 도입하고, 스프링 빈으로 수동 등록해보자. 

**OrderRepositoryV1**
```java
package hello.proxy.app.v1;
public interface OrderRepositoryV1 {
 void save(String itemId);
}
```

**OrderRepositoryV1Impl**
```java
package hello.proxy.app.v1;
public class OrderRepositoryV1Impl implements OrderRepositoryV1 {
 @Override
 public void save(String itemId) {
 //저장 로직
 if (itemId.equals("ex")) {
 throw new IllegalStateException("예외 발생!");
 }
 sleep(1000);
 }
 private void sleep(int millis) {
 try {
 Thread.sleep(millis);
 } catch (InterruptedException e) {
 e.printStackTrace();
 }
 }
}
```

**OrderServiceV1**
```java
package hello.proxy.app.v1;
public interface OrderServiceV1 {
 void orderItem(String itemId);
}
```

**OrderServiceV1Impl**
```java
package hello.proxy.app.v1;
public class OrderServiceV1Impl implements OrderServiceV1 {
 private final OrderRepositoryV1 orderRepository;
 public OrderServiceV1Impl(OrderRepositoryV1 orderRepository) {
 this.orderRepository = orderRepository;
 }
 @Override
 public void orderItem(String itemId) {
 orderRepository.save(itemId);
 }
}
```

**OrderControllerV1**
```java
package hello.proxy.app.v1;
import org.springframework.web.bind.annotation.*;
@RequestMapping //스프링은 @Controller 또는 @RequestMapping 이 있어야 스프링 컨트롤러로
인식
@ResponseBody
public interface OrderControllerV1 {
 @GetMapping("/v1/request")
 String request(@RequestParam("itemId") String itemId);
 @GetMapping("/v1/no-log")
 String noLog();
}
```

- `@RequestMapping`: 스프링MVC는 타입에 `@Controller` 또는 `@RequestMapping` 애노테이션이 있어야 스프링 컨트롤러로 인식한다. 그리고 스프링 컨트롤러로 인식해야, HTTP URL이 매핑되고 동작한다. 이 애노테이션은 인터페이스에 사용해도 된다. 
- `@ResponseBody`: HTTP 메시지 컨버터를 사용해서 응답한다. 이 애노테이션은 인터페이스에 사용해도 된다.
- `@RequestParam("itemId") String itemId`: 인터페이스에는 `@RequestParam("itemId")`의 값을 생략하면 `itemId`단어를 컴파일 이후 자바 버전에 따라 인식하지 못할 수 있다. 인터페이스에는 꼭 넣어주자. 클래스에는 생략해도 대부분 잘 지원된다.
- 코드를 보면 `request()`, `noLog()` 두가지 메서드가 있다. `request()`는 `LogTrace`를 적용할 대상이고, `noLog()`는 단순히 `LogTrace`를 적용하지 않을 대상이다. 

**OrderControllerV1Impl**
```java
package hello.proxy.app.v1;
import lombok.extern.slf4j.Slf4j;
@Slf4j
public class OrderControllerV1Impl implements OrderControllerV1 {
 private final OrderServiceV1 orderService;
 public OrderControllerV1Impl(OrderServiceV1 orderService) {
 this.orderService = orderService;
 }
 @Override
 public String request(String itemId) {
 orderService.orderItem(itemId);
 return "ok";
 }
 @Override
 public String noLog() {
 return "ok";
 }
}
```
- 컨트롤러 구현체이다. `OrderControllerV1`인터페이스에 스프링 MVC 관련 애노테이션이 정의되어 있따. 

**AppV1Config**  
이제 스프링 빈으로 수동 등록해보자.
```java
package hello.proxy.config;
import hello.proxy.app.v1.*;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
@Configuration
public class AppV1Config {
 @Bean
 public OrderControllerV1 orderControllerV1() {
 return new OrderControllerV1Impl(orderServiceV1());
 }
 @Bean
 public OrderServiceV1 orderServiceV1() {
 return new OrderServiceV1Impl(orderRepositoryV1());
 }
 @Bean
 public OrderRepositoryV1 orderRepositoryV1() {
 return new OrderRepositoryV1Impl();
 }
}
```

**ProxyApplication - 코드 추가**
```java
package hello.proxy;
import hello.proxy.config.AppV1Config;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.annotation.Import;
@Import(AppV1Config.class)
@SpringBootApplication(scanBasePackages = "hello.proxy.app") //주의
public class ProxyApplication {
 public static void main(String[] args) {
 SpringApplication.run(ProxyApplication.class, args);
 }
}
```
- `@Import(AppV1Config.class)`: 클래스를 스프링 빈으로 등록한다. 여기서는 `AppV1Config.class`를 스프링 빈으로 등록한다. 일반적으로 `@Configuration`같은 설정 파일을 등록할 때 사용하지만, 스프링 빈을 등록할 때도 사용할 수 있다.
- `@SpringBootApplication(scanBasePackages = "hello.proxy.app")`: `@ComponentScan`의 기능과 같다. 컴포넌트 스캔을 시작할 위치를 지정한다. 이 값을 설정하면 해당 패키지와 그 하위 패키지를 컴포넌트 스캔한다. 이 값을 사용하지 않으면 `ProxyApplication`이 있는 패키지와 그 하위 패키지를 스캔한다. 참고로 `v3`에서 지금 설정한 컴포넌트 스캔 기능을 사용한다. 

> 주의  
> `@Configuration`을 사용한 수동 빈 등록 설정을 `hello.proxy.config`위치에 두고 점진적으로 변경할 예정이다.   
> 지금은 `AppV1Config.class`를 `@Import`를 사용해서 설정하지만 이후에 다른것을 설정한다는 이야기이다.
> 
> 
> `@Configuration`은 내부에 `@Component` 애노테이션을 포함하고 있어서 컴포넌트 스캔의 대상이 된다.
> 따라서 컴포넌트 스캔에 의해 `hello.proxy.config` 위치의 설정 파일들이 스프링 빈으로 자동 등록 되지 않도록
> 컴포넌트 스캔의 시작 위치를 `scanBasePackages=hello.proxy.app`로 설정해야 한다.

**실행**  
http://localhost:8080/v1/request?itemId=hello




</details>

<details> <summary> 3. 예제 프로젝트 만들기 v2 </summary>

## 3. 예제 프로젝트 만들기 v2

v2 - 인터페이스 없는 구체 클래스 - 스프링 빈으로 수동 등록  
이번에는 인터페이스가 없는 `Controller`, `Service`, `Repository`를 스프링 빈으로 수동 등록해보자.

**OrderRepositoryV2**
```java
package hello.proxy.app.v2;
public class OrderRepositoryV2 {
 public void save(String itemId) {
 //저장 로직
 if (itemId.equals("ex")) {
 throw new IllegalStateException("예외 발생!");
 }
 sleep(1000);
 }
 private void sleep(int millis) {
 try {
 Thread.sleep(millis);
 } catch (InterruptedException e) {
 e.printStackTrace();
 }
 }
}
```

**OrderServiceV2**
```java
package hello.proxy.app.v2;
public class OrderServiceV2 {
 private final OrderRepositoryV2 orderRepository;
 public OrderServiceV2(OrderRepositoryV2 orderRepository) {
 this.orderRepository = orderRepository;
 }
 public void orderItem(String itemId) {
 orderRepository.save(itemId);
 }
}
```

**OrderControllerV2**
```java
package hello.proxy.app.v2;
import lombok.extern.slf4j.Slf4j;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.ResponseBody;
@Slf4j
@RequestMapping
@ResponseBody
public class OrderControllerV2 {
 private final OrderServiceV2 orderService;
 public OrderControllerV2(OrderServiceV2 orderService) {
 this.orderService = orderService;
 }
 @GetMapping("/v2/request")
 public String request(String itemId) {
 orderService.orderItem(itemId);
 return "ok";
 }
 @GetMapping("/v2/no-log")
 public String noLog() {
 return "ok";
 }
}
```

- `@RequestMapping`: 스프링MVC는 타입에 `@Controller` 또는 `@RequestMapping` 애노테이션이 있어야 스프링 컨트롤러로 인식한다. 그리고 스프링 컨트롤러로 인식해야, HTTP URL이 매핑되고 동작한다. 그런데 여기서는 `@Controller`를 사용하지 않고, `@RequestMapping`애노테이션을 사용했다. 그 이유는 `@Controller`를 사용하면 자동 컴포넌트 스캔의 대상이 되기 때문이다. 여기서는 컴포넌트 스캔을 통한 자동 빈 등록이 아니라 수동 빈 등록을 하는 것이 목표다. 따라서 컴포넌트 스캔과 관계 없는 `@RequestMapping`를 타입에 사용했다.

**AppV2Config**
```java
package hello.proxy.config;
import hello.proxy.app.v2.OrderControllerV2;
import hello.proxy.app.v2.OrderRepositoryV2;
import hello.proxy.app.v2.OrderServiceV2;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
@Configuration
public class AppV2Config {
 @Bean
 public OrderControllerV2 orderControllerV2() {
 return new OrderControllerV2(orderServiceV2());
 }
 @Bean
 public OrderServiceV2 orderServiceV2() {
 return new OrderServiceV2(orderRepositoryV2());
 }
 @Bean
 public OrderRepositoryV2 orderRepositoryV2() {
 return new OrderRepositoryV2();
 }
}
```
- 수동 빈 등록을 위한 설정 

**ProxyApplication**
```java
package hello.proxy;
import hello.proxy.config.AppV1Config;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.annotation.Import;
@Import({AppV1Config.class, AppV2Config.class})
@SpringBootApplication(scanBasePackages = "hello.proxy.app")
public class ProxyApplication {
 public static void main(String[] args) {
 SpringApplication.run(ProxyApplication.class, args);
 }
}
```

**변경 사항**
- 기존: `@Import(AppV1Config.class)`
- 변경: `@Import({AppV1Config.class, AppV2Config.class})
- `@Import`안에 배열로 등록하고 싶은 설정파일을 다양하게 추가할 수 있다. 


</details>

<details> <summary> 4. 예제 프로젝트 만들기 v3 </summary>

## 4. 예제 프로젝트 만들기 v3

v3 - 컴포넌트 스캔으로 스프링 빈 자동 등록 

**OrderRepositoryV3**
```java
package hello.proxy.app.v3;
import org.springframework.stereotype.Repository;
@Repository
public class OrderRepositoryV3 {
 public void save(String itemId) {
 //저장 로직
 if (itemId.equals("ex")) {
 throw new IllegalStateException("예외 발생!");
 }
 sleep(1000);
 }
 private void sleep(int millis) {
 try {
 Thread.sleep(millis);
 } catch (InterruptedException e) {
 e.printStackTrace();
 }
 }
}
```

**OrderServiceV3**
```java
package hello.proxy.app.v3;
import org.springframework.stereotype.Service;
@Service
public class OrderServiceV3 {
 private final OrderRepositoryV3 orderRepository;
 public OrderServiceV3(OrderRepositoryV3 orderRepository) {
 this.orderRepository = orderRepository;
 }
 public void orderItem(String itemId) {
 orderRepository.save(itemId);
 }
}
```

**OrderControllerV3**
```java
package hello.proxy.app.v3;
import lombok.extern.slf4j.Slf4j;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;
@Slf4j
@RestController
public class OrderControllerV3 {
 private final OrderServiceV3 orderService;
 public OrderControllerV3(OrderServiceV3 orderService) {
 this.orderService = orderService;
 }
 @GetMapping("/v3/request")
 public String request(String itemId) {
 orderService.orderItem(itemId);
 return "ok";
 }
}
```

`ProxyApplication`에서 `@SpringBootApplication(scanBasePackages = "hello.proxy.app")`를 사용했고, 각각 `@RestController`, `@Service`, `@Repository` 애노테이션을 가지고 있기 떄문에 컴포넌트 스캔의 대상이 된다. 


</details>

<details> <summary> 5. 요구사항 추가 </summary>

## 5. 요구사항 추가

지금까지 로그 추적기를 만들어서 기존 요구사항을 모두 만족했다.

**기존 요구사항**
- 모든 PUBLIC 메서드의 호출과 응답 정보를 로그로 출력
- 애플리케이션의 흐름을 변경하면 안됨
  - 로그를 남긴다고 해서 비즈니스 로직의 동작에 영향을 주면 안됨
- 메서드 호출에 걸린 시간
- 정상 흐름과 예외 흐름 구분
  - 예외 발생시 예외 정보가 남아야 함
- 메서드 호출의 깊이 표현
- HTTP 요청을 구분
  - HTTP 요청 단위로 특정 ID를 남겨서 어떤 HTTP 요청에서 시작된 것인지 명확하게 구분이 가능해야
함
  - 트랜잭션 ID (DB 트랜잭션X)

**예시**
```
정상 요청
[796bccd9] OrderController.request()
[796bccd9] |-->OrderService.orderItem()
[796bccd9] | |-->OrderRepository.save()
[796bccd9] | |<--OrderRepository.save() time=1004ms
[796bccd9] |<--OrderService.orderItem() time=1014ms
[796bccd9] OrderController.request() time=1016ms
예외 발생
[b7119f27] OrderController.request()
[b7119f27] |-->OrderService.orderItem()
[b7119f27] | |-->OrderRepository.save()
[b7119f27] | |<X-OrderRepository.save() time=0ms
ex=java.lang.IllegalStateException: 예외 발생!
[b7119f27] |<X-OrderService.orderItem() time=10ms
ex=java.lang.IllegalStateException: 예외 발생!
[b7119f27] OrderController.request() time=11ms
ex=java.lang.IllegalStateException: 예외 발생!
```

**하지만**  
하지만 이 요구사항을 만족하기 위해서 기존 코드를 많이 수정해야 한다.   
코드 수정을 최소화 하기 위해 템플릿 메서드 패턴과 콜백 패턴도 사용했지만, 결과적으로 로그를 남기고 싶은 클래스가 수백개라면 수백개의 클래스를 모두 고쳐야한다.  
로그를 남길 떄 기존 원본 코드를 변경해야 한다는 사실 그 자체가 개발자에게는 가장 큰 문제로 남는다.

기존 요구사항에 다음 요구사항이 추가되었다.

**요구사항 추가**
- 원본 코드를 전혀 수정하지 않고, 로그 추적기를 적용해라.
- 특정 메서드는 로그를 출력하지 않는 기능
  - 보안상 일부는 로그를 출력하면 안된다.
- 다음과 같은 다양한 케이스에 적용할 수 있어야 한다.
  - v1 - 인터페이스가 있는 구현 클래스에 적용
  - v2 - 인터페이스가 없는 구체 클래스에 적용
  - v3 - 컴포넌트 스캔 대상에 기능 적용 

가장 어려운 문제는 **원본 코드를 전혀 수정하지 않고, 로그 추적기를 도입**하는 것이다. 이 문제를 해결하려면 프록시(Proxy)의 개념을 먼저 이해해야 한다. 

</details>

<details> <summary> 6. 프록시, 프록시 패턴, 데코레이터 패턴 - 소개  </summary>

## 6. 프록시, 프록시 패턴, 데코레이터 패턴 - 소개 

프록시에 대해서 알아보자. 

**클라이언트와 서버**  
![image](https://user-images.githubusercontent.com/28394879/141047469-5a8cbe46-93ab-431d-af9a-9658f4b0fe72.png)

클라이언트(`Client`)와 서버(`Server`)라고 하면 개발자들은 보통 서버 컴퓨터를 생각한다.  
사실 클라이어트와 서버의 개념은 상당히 넓게 사용된다. 클라이언트는 의뢰인이라는 뜻이고, 서버는 '서비스나 상품을 제공하는 사람이나 물건'을 뜻한다.  
따라서 클라이언트와 서버의 기본 개념을 정의하면 "클라이언트는 서버에 필요한 것을 요청하고, 서버는 클라이언트의 요청을 처리" 하는 것이다.

이 개념을 우리가 익숙한 컴퓨터 네트워크에 도입하면 클라이언트는 웹 브라우저가 되고, 요청을 처리하는 서버는 웹 서버가 된다.  
이 개념을 객체에 도입하면, 요청하는 객체는 클라이언트가 되고, 요청을 처리하는 객체는 서버가 된다. 


**직접 호출과 간접 호출**  
![image](https://user-images.githubusercontent.com/28394879/141047516-128e42f4-192c-42c4-99f7-2eb5b9d0288f.png)

클라이언트와 서버 개념에서 일반적으로 클라이언트가 서버를 직접 호출하고, 처리 결과를 직접 받는다. 이것을 직접 호출이라 한다.


![image](https://user-images.githubusercontent.com/28394879/141047548-07197883-3c33-4046-ab6c-bf1db0694932.png)

그런데 클라이언트가 요청한 결과를 서버에 직접 요청하는 것이 아니라 어떤 대리자를 통해서 대신 간접적으로 서버에 요청할 수 있다. 예를 들어서 내가 직접 마트에서 장을 볼 수 있지만, 누군가에게 대신 장을 봐달라고 부탁할 수도 있가.  
여기서 대신 장을 보는 **대리자를 영어로 프록시(Proxy)**라 한다.

**예시**  
재미있는 점은 직접 호출과 다르게 간접 호출을 하면 대리자가 중간에서 여러가지 일을 할 수 있다는 점이다.
- 엄마에게 라면을 사달라고 부탁 했는데, 엄마는 그 라면은 이미 집에 있다고 할 수도 있다. 그러면 기대한 것 보다 더 빨리 라면을 먹을 수 있다.(접근 제어, 캐싱)
- 아버지께 자동차 주유를 부탁했는데, 아버지가 주유 뿐만 아니라 세차까지 하고 왔다. 클라이언트가 기대한 것 외에 세차라는 부가 기능까지 얻게 되었다. (부가 기능 추가)
- 그리고 대리자가 또 다른 대리자를 부를 수도 있다. 예를 들어서 내가 동생에게 라면을 사달라고 했는데, 동생은 또 다른 누군가에게 라면을 사달라고 다시 요청할 수도 있다. 중요한 점은 클라이언트는 대리자를 통해서 요청했기 때문에 그 이후 과정은 모른다는 점이다. 동생을 통해서 라면이 나에게 도착하기만 하면 된다.(프록시 체인) 


![image](https://user-images.githubusercontent.com/28394879/141047581-0616a5fa-a91a-415c-a3d4-00529901cec6.png)

재미로 이야기해보았지만, 실제 프록시의 기능도 이와 같다. 객체에서 프록시의 역할을 알아보자. 

**대체 가능**  
그런데 여기까지 듣고 보면 아무 객체나 프록시가 될 수 있는 것 같다.  
객체에서 프록시가 되려면, 클라이언트는 서버에게 요청을 한 것인지, 프록시에게 요청을 한 것인지 조차 몰라야 한다.  
쉽게 이야기해서 서버와 프록시는 같은 인터페이스를 사용해야 한다. 그리고 클라이언트가 사용하느 서버 객체를 프록시 객체로 변경해도 클라이언트 코드를 변경하지 않고 동작할 수 있어야 한다. 


![image](https://user-images.githubusercontent.com/28394879/141047606-638a0a6a-995b-4b51-ba7d-1189c3e650db.png)  
**서버와 프록시가 같은 인터페이스 사용**

클래스 의존관계를 보면 클라이언트는 서버 인터페이스(`ServerInterface`)에만 의존한다. 그리고 서버와 프록시가 같은 인터페이스를 사용한다. 따라서 DI를 사용해서 대체 가능하다.



![image](https://user-images.githubusercontent.com/28394879/141047618-ed291c50-a2fe-47f2-a3f0-a31b16192bb9.png)


이번에는 런타임 객체 의존관계를 살펴보자. 런타임(애플리케이션 실행 시점)에 클라이언트 객체에 DI를 사용해서 `Client-> Server`에서 `Client -> Proxy`로 객체 의존관계를 변경해도 클라이언트 코드를 전혀 변경하지 않아도 된다.  
클라이언트 입장에서는 변경 사실 조차 모른다.  
DI를 사용하면 클라이언트 코드의 변경 없이 유연하게 프록시를 주입할 수 있다. 

**프록시의 주요 기능**  
프록시를 통해서 할 수 있는 일은 크게 2가지로 구분할 수 있다.
- 접근 제어
  - 권한에 따른 접근 차단
  - 캐싱
  - 지연 로딩
- 부가 기능 추가
  - 원래 서버가 제공하는 기능에 더해서 부가 기능을 수행한다.
  - 예) 요청 값이나, 응답 값을 중간에 변형한다.
  - 예) 실행 시간을 측정해서 추가 로그를 남긴다.

프록시 객체가 중간에 있으면 크게 **접근 제어**와 **부가기능 추가**를 수행할 수 있다. 

**GOF 디자인 패턴**  
둘다 프록시를 사용하는 방법이지만 GOF 디자인 패턴에서는 이 둘을 의도(intent)에 따라서 프록시 패턴과 데코레이터 패턴으로 구분한다.  
- 프록시 패턴: 접근 제어가 목적
- 데코레이터 패턴: 새로운 기능 추가가 목적

둘다 프록시를 사용하지만, 의도가 다르다는 점이 핵심이다. 용어가 프록시 패턴이라고 해서 이 패턴만 프록시를 사용하는 것은 아니다. 데코레이터 패턴도 프록시를 사용한다.

이왕 프록시를 학습하기로 했으니 GOF 디자인 패턴에서 설명하는 프록시 패턴과 데코레이터 패턴을 나누어 학습해보자.

> **참고**: 프록시라는 개념은 클라이언트 서버라는 큰 개념안에서 자연스럽게 발생할 수 있다. 프록시는 객체안에서의 개념도 있고, 웹 서버에서의 프록시도 있다.  
> 객체안에서 객체로 구현되어 있는가, 웹 서버로 구현되어 있는가 처럼 규모의 차이가 있을 뿐 근본적인 역할은 같다. 

</details>

<details> <summary> 7. 프록시 패턴 - 예제 코드1 </summary>

## 7. 프록시 패턴 - 예제 코드1

### 테스트 코드에 Lombok 적용하기

테스트 코드에 Lombok을 사용하려면 `build.gradle`에 테스트에서 lombok을 사용할 수 있도록 의존관계를 추가해야 한다.

**build.gradle**에 추가  
```java
dependencies {
 ...
 //테스트에서 lombok 사용
 testCompileOnly 'org.projectlombok:lombok'
 testAnnotationProcessor 'org.projectlombok:lombok'
}
```
이렇게 해야 테스트 코드에서 `@Slfj4`같은 애노테이션이 작동한다.

### 프록시 패턴 - 예제 코드 작성   
프록시 패턴을 이해하기 위한 예제 코드를 작성해보자. 먼저 프록시 패턴을 도입하기 전 코드를 아주 단순하게 만들어보자.

![image](https://user-images.githubusercontent.com/28394879/141050401-45069328-c6c9-4057-837a-57119b800395.png)

![image](https://user-images.githubusercontent.com/28394879/141050433-f78b821e-ac98-4032-8b40-dc8213a5b4a3.png)

**Subject 인터페이스**  
주의: 테스트 패키지에 위치한다.
```java
package hello.proxy.pureproxy.proxy.code;
public interface Subject {
 String operation();
}
```
예제에서 `Subject` 인터페이스는 단순히 `operation()` 메서드 하나만 가지고 있다.

**RealSubject**  
주의: 테스트 패키지에 위치한다.
```java
package hello.proxy.pureproxy.proxy.code;
import lombok.extern.slf4j.Slf4j;
@Slf4j
public class RealSubject implements Subject {
 @Override
 public String operation() {
 log.info("실제 객체 호출");
 sleep(1000);
 return "data";
 }
 private void sleep(int millis) {
 try {
 Thread.sleep(millis);
 } catch (InterruptedException e) {
 e.printStackTrace();
 }
 }
}
```
`RealSubject`는 `Subject` 인터페이스를 구현했다. `operation()`은 데이터 조회를 시뮬레이션 하기 위해 1초 쉬도록 했다. 예를 들어서 데이터를 DB나 외부에서 조회하는데 1초 걸린다고 생각하면 된다.   
호출할 때 마다 시스템에 큰 부하를 주는 데이터 조회라고 가정하자.

**ProxyPatternClient**  
주의: 테스트 패키지에 위치한다.  
```java
package hello.proxy.pureproxy.proxy.code;
public class ProxyPatternClient {
 private Subject subject;
 public ProxyPatternClient(Subject subject) {
 this.subject = subject;
 }
 public void execute() {
 subject.operation();
 }
}
```
`Subject` 인터페이스에 의존하고, `Subject`를 호출하는 클라이언트 코드이다.  
`execute()`를 실행하면 `subject.operation()`를 호출한다.

**ProxyPatternTest**
```java
package hello.proxy.pureproxy.proxy;
import hello.proxy.pureproxy.proxy.code.ProxyPatternClient;
import hello.proxy.pureproxy.proxy.code.RealSubject;
import hello.proxy.pureproxy.proxy.code.Subject;
import org.junit.jupiter.api.Test;
public class ProxyPatternTest {
 @Test
 void noProxyTest() {
 RealSubject realSubject = new RealSubject();
 ProxyPatternClient client = new ProxyPatternClient(realSubject);
 client.execute();
 client.execute();
 client.execute();
 }
}
```

테스트 코드에서는 `client.execute()`를 3번 호출한다. 데이터를 조회하는데 1초가 소모되므로 총 3초의 시간이 걸린다.

**실행 결과**
```java
RealSubject - 실제 객체 호출
RealSubject - 실제 객체 호출
RealSubject - 실제 객체 호출
```

**client.execute()을 3번 호출하면 다음과 같이 처리된다.**
1. `client -> realSubject`을 호출해서 값을 조회한다. (1초)
2. `client -> realSubject`을 호출해서 값을 조회한다. (1초)
3. `client -> realSubject`을 호출해서 값을 조회한다. (1초)


그런데 이 데이터가 한번 조회하면 변하지 않는 데이터라면 어딘가에 보관해두고 이미 조회한 데이터를 사용하는 것이 성능상 좋다. 이런 것을 캐시라고 한다.  
프록시 패턴의 주요 기능은 접근 제어이다. 캐시도 접근 자체를 제어하는 기능 중 하나이다. 

이미 개발된 로직을 전혀 수정하지 않고, 프록시 객체를 통해서 캐시를 적용해보자. 



</details>

<details> <summary> 8. 프록시 패턴 - 예제 코드2 </summary>

## 8. 프록시 패턴 - 예제 코드2

![image](https://user-images.githubusercontent.com/28394879/141100911-5794679f-a4c3-4ec5-9935-aed52156fbd0.png)

**CacheProxy**  
주의: 테스트 패키지에 위치함
```java
package hello.proxy.pureproxy.proxy.code;
import lombok.extern.slf4j.Slf4j;
@Slf4j
public class CacheProxy implements Subject {
 private Subject target;
 private String cacheValue;
 public CacheProxy(Subject target) {
 this.target = target;
 }
 @Override
 public String operation() {
 log.info("프록시 호출");
 if (cacheValue == null) {
 cacheValue = target.operation();
 }
 return cacheValue;
 }
}
```
앞서 설명한 것 처럼 프록시도 실제 객체와 그 모양이 같아야 하기 떄문에 `Subject` 인터페이슬르 구현해야 한다.

- `private Subject target`: 클라이언트가 프록시를 호출하면 프록시가 최종적으로 실제 객체를 호출해야 한다. 따라서 내부에 실제 객체의 참조를 가지고 있어야 한다. 이렇게 프록시가 호출하는 대상을 `target`이라 한다.
- `operation()`: 구현한 코드를 보면 `cacheValue`에 값이 없으면 실제 객체 (`target`)를 호출해서 값을 구한다. 그리고 구한 값을 `cacheValue`에 저장하고 반환한다. 만약 `cacheValue`에 값이 있으면 실제 객체를 전혀 호출하지 않고, 캐시 값을 그대로 반환한다. 따라서 처음 조회 이후에는 캐시 (`cacheValue`)에서 매우 빠르게 데이터를 조회할 수 있다. 

**ProxyPatternTest-cacheProxyTest() 추가**
```java
package hello.proxy.pureproxy.proxy;
import hello.proxy.pureproxy.proxy.code.CacheProxy;
import hello.proxy.pureproxy.proxy.code.ProxyPatternClient;
import hello.proxy.pureproxy.proxy.code.RealSubject;
import hello.proxy.pureproxy.proxy.code.Subject;
import org.junit.jupiter.api.Test;
public class ProxyPatternTest {
 @Test
 void noProxyTest() {
 RealSubject realSubject = new RealSubject();
 ProxyPatternClient client = new ProxyPatternClient(realSubject);
 client.execute();
 client.execute();
 client.execute();
 }
 @Test
 void cacheProxyTest() {
 Subject realSubject = new RealSubject();
 Subject cacheProxy = new CacheProxy(realSubject);
 ProxyPatternClient client = new ProxyPatternClient(cacheProxy);
 client.execute();
 client.execute();
 client.execute();
 }
}
```

**cacheProxyTest()**  
`realSubject`와 `cacheProxy`를 생성하고 둘을 연결한다. 결과적으로 `cacheProxy`가 `realSubject`를 참조하는 런타임 객체 의존관계가 완전된다.   
그리고 마지막으로 `client`에 `realSubject`가 아닌 `cacheProxy`를 주입한다.  
이 과정을 통해서 `client -> cacheProxy -> realSubject` 런타임 객체 의존관계가 완성된다.

`cacheProxyTest()`는 `client.execute()`을 총 3번 호출한다. 이번에는 클라이언트가 실제 `realSubject`를 호출하는 것이 아니라 `cacheProxy`를 호출하게 된다.

**실행 결과**
```
CacheProxy - 프록시 호출
RealSubject - 실제 객체 호출
CacheProxy - 프록시 호출
CacheProxy - 프록시 호출
```

**client.execute()을 3번 호출하면 다음과 같이 처리 된다.**
1. client의 cacheProxy 호출 cacheProxy에 캐시 값이 없다. realSubject를 호출, 결과를 캐시에
저장 (1초)
2. client의 cacheProxy 호출 cacheProxy에 캐시 값이 있다. cacheProxy에서 즉시 반환 (0초)
3. client의 cacheProxy 호출 cacheProxy에 캐시 값이 있다. cacheProxy에서 즉시 반환 (0초)

결과적으로 캐시 프록시를 도입하기 전에는 3초가 걸렸지만, 캐시 프록시 도입 이후에는 최초에 한번만 1초가 걸리고, 이후에는 거의 즉시 반환한다.

**정리**  
프록시 패턴의 핵심은 `RealSubject` 코드와 클라이언트 코드를 전혀 변경하지 않고, 프록시를 도입해서 접근 제어를 했다는 점이다.  
그리고 클라이언트 코드의 변경 없이 자유롭게 프록시를 넣고 뺄 수 있다. 실제 클라이언트 입장에서는 프록시 객체가 주입되었는지, 실제 객체가 주입되었는지 알지 못한다.


</details>

<details> <summary> 9. 데코레이터 패턴 - 예제 코드1 </summary>

## 9. 데코레이터 패턴 - 예제 코드1

데코레이터 패턴을 이해하기 위한 예제 코드를 작성해보자. 먼저 데코레이터 패턴을 도입하기 전 코드를 아주 단순하게 만들어보자.  

![image](https://user-images.githubusercontent.com/28394879/141103041-7b1be727-4404-4e45-b740-3c7e99986ea2.png)

**Component 인터페이스**  
주의: 테스트 패키지에 위치 
```java
package hello.proxy.pureproxy.decorator.code;
public interface Component {
 String operation();
}
```

`Component` 인터페이스는 단순히 `String operation()` 메서드를 가진다.

**RealComponent**  
주의: 테스트 패키지에 위치 
```java
package hello.proxy.pureproxy.decorator.code;
import lombok.extern.slf4j.Slf4j;
@Slf4j
public class RealComponent implements Component {
 @Override
 public String operation() {
 log.info("RealComponent 실행");
 return "data";
 }
}
```
- `RealComponent`는 `Component` 인터페이스를 구현한다.
- `operation()`: 단순히 로그를 남기고 `"data"` 문자를 반환한다.

**DecoratorPatternClient**  
주의: 테스트 패키지에 위치 
```java
package hello.proxy.pureproxy.decorator.code;
import lombok.extern.slf4j.Slf4j;
@Slf4j
public class DecoratorPatternClient {
 private Component component;
 public DecoratorPatternClient(Component component) {
 this.component = component;
 }
 public void execute() {
 String result = component.operation();
 log.info("result={}", result);
 }
}
```
- 클라이언트 코드는 단순히 `Component` 인터페이스를 의존한다.
- `execute()`를 실행하면 `component.operation()`을 호출하고, 그 결과를 출력한다.

**DecoratorPatternTest**
```java
package hello.proxy.pureproxy.decorator;
import hello.proxy.pureproxy.decorator.code.*;
import lombok.extern.slf4j.Slf4j;
import org.junit.jupiter.api.Test;
@Slf4j
public class DecoratorPatternTest {
 @Test
 void noDecorator() {
 Component realComponent = new RealComponent();
 DecoratorPatternClient client = new
DecoratorPatternClient(realComponent);
 client.execute();
 }
}
```

테스트 코드는 `client -> realComponent`의 의존관계를 설정하고, `client.execute()`을 호출한다.

**실행 결과**
```
RealComponent - RealComponent 실행
DecoratorPatternClient - result=data
```

여기까지는 앞서 프록시 패턴에서 설명한 내용과 유사하고 단순해서 이해하는데 어려움은 없을 것이다. 




</details>

<details> <summary> 10. 데코레이터 패턴 - 예제 코드2 </summary>

## 10. 데코레이터 패턴 - 예제 코드2

**부가 기능 추가**  
앞서 설명한 것 처럼 프록시를 통해서 할 수 있는 기능은 크게 접근 제어와 부가 기능 추가라는 2가지로 구분한다.  
앞서 프록시 패턴에서 캐시를 통한 접근 제어를 알아보았다.   
이번에는 프록시를 활용해서 부가 기능을 추가해보자.  
이렇게 프록시로 부가 기능을 추가하는 것을 데코레이터 패턴이라 한다.  

데코레이터 패턴: 원래 서버가 제공하는 기능에 더해서 부가 기능을 수행한다.
- 예) 요청 값이나, 응답 값을 중간에 변형한다.
- 예) 실행 시간을 측정해서 추가 로그를 남긴다.

### 응답 값을 꾸며주는 데코레이터  
응답 값을 꾸며주는 데코레이터 프록시를 만들어보자.

![image](https://user-images.githubusercontent.com/28394879/141104449-762a4ef6-660c-45b3-923d-8c1496138747.png)

![image](https://user-images.githubusercontent.com/28394879/141104470-725c0acd-63f0-491e-8514-844de432df5b.png)


**MessageDecorator**  
주의: 테스트 패키지에 위치한다
```java
package hello.proxy.pureproxy.decorator.code;
import lombok.extern.slf4j.Slf4j;
@Slf4j
public class MessageDecorator implements Component {
 private Component component;
 public MessageDecorator(Component component) {
 this.component = component;
 }
 @Override
 public String operation() {
 log.info("MessageDecorator 실행");
 String result = component.operation();
 String decoResult = "*****" + result + "*****";
 log.info("MessageDecorator 꾸미기 적용 전={}, 적용 후={}", result,
decoResult);
 return decoResult;
 }
}
```
`MessageDecorator`는 `Component` 인터페이스를 구현한다.  
프록시가 호출해야 하는 대상을 `component`에 저장한다.  
`operation()`을 호출하면 프록시와 연결된 대상을 호출`(component.operation())`하고, 그 응답 값에 `*****`을 더해서 꾸며준 다음 반환한다.  
예를 들어서 응답 값이 `data`라면 다음과 같다.  
- 꾸미기 전: `data
- 꾸민 후: `*****data*****`

**DecoratorPatternTest - 추가**  
```java
@Test
void decorator1() {
 Component realComponent = new RealComponent();
 Component messageDecorator = new MessageDecorator(realComponent);
 DecoratorPatternClient client = new
DecoratorPatternClient(messageDecorator);
 client.execute();
}
```

`client -> messageDecorator -> realComponent`의 객체 의존 관계를 만들고 `client.execute()`를 호출한다.

**실행 결과**
```
MessageDecorator - MessageDecorator 실행
RealComponent - RealComponent 실행
MessageDecorator - MessageDecorator 꾸미기 적용 전=data, 적용 후=*****data*****
DecoratorPatternClient - result=*****data*****
```

실행 결과를 보면 `MessageDecorator`가 `RealComponent`를 호출하고 반환한 응답 메시지를 꾸며서 반환한 것을 확인할 수 있다. 




</details>

<details> <summary> 11. 데코레이터 패턴 - 예제 코드3 </summary>

## 11. 데코레이터 패턴 - 예제 코드3

### 실행 시간을 측정하는 데코레이터  
이번에는 기존 데코레이터에 더해서 실행 시간을 측정하는 기능까지 추가해보자.

![image](https://user-images.githubusercontent.com/28394879/141105409-3541cae7-9219-4a87-8f74-7731bafb7504.png)

**TimeDecorator**  
주의: 테스트 패키지에 위치한다.
```java
package hello.proxy.pureproxy.decorator.code;
import lombok.extern.slf4j.Slf4j;
@Slf4j
public class TimeDecorator implements Component {
 private Component component;
 public TimeDecorator(Component component) {
 this.component = component;
 }
 @Override
 public String operation() {
 log.info("TimeDecorator 실행");
 long startTime = System.currentTimeMillis();
 String result = component.operation();
 long endTime = System.currentTimeMillis();
 long resultTime = endTime - startTime;
 log.info("TimeDecorator 종료 resultTime={}ms", resultTime);
 return result;
 }
}
```
`TimeDecorator`는 실행 시간을 측정하는 부가 기능을 제공한다. 대상을 호출하기 전에 시간을 가지고 있다가, 대상의 호출이 끝나면 호출 시간을 로그로 남겨준다.

**DecoratorPatternTest - 추가** 
```java
@Test
void decorator2() {
 Component realComponent = new RealComponent();
 Component messageDecorator = new MessageDecorator(realComponent);
 Component timeDecorator = new TimeDecorator(messageDecorator);
 DecoratorPatternClient client = new DecoratorPatternClient(timeDecorator);
 client.execute();
}
```

`client -> timeDecorator -> messageDecorator -> realComponent`의 객체 의존관계를 설정하고, 실행한다.

**실행 결과**
```
TimeDecorator 실행
MessageDecorator 실행
RealComponent 실행
MessageDecorator 꾸미기 적용 전=data, 적용 후=*****data*****
TimeDecorator 종료 resultTime=7ms
result=*****data*****
```

실행 결과를 보면 `TimeDecorator`가 `MessageDecorator`를 실행하고 실행 시간을 측정해서 출력한 것을 확인할 수 있다.


</details>

<details> <summary> 12. 프록시 패턴과 데코레이터 패턴 정리 </summary>

## 12. 프록시 패턴과 데코레이터 패턴 정리

### GOF 데코레이터 패턴  
![image](https://user-images.githubusercontent.com/28394879/141106308-957f1cbf-cabd-4554-ada4-240047b0d6c5.png)

여기서 생각해보면 `Decorator` 기능에 일부 중복이 있다. 꾸며주는 역할을 하는 `Decorator`들은 스스로 존재할 수 없다.  
항상 꾸며줄 대상이 있어야 한다.  
따라서 내부에 호출 대상인 `Component`를 가지고 있어야 한다.  
그리고 `component`를 항상 호출 해야 한다. 이 부분이 중복이다.  
이런 중복을 제거하기 위해 `component`를 속성으로 가지고 있는 `Decorator`라는 추상 클래스를 만드는 방법도 고민할 수 있다.  
이렇게 하면 추가로 클래스 다이어그램에서 어떤 것이 실제 컴포넌트 인지, 데코레이터인지 명확하게 구분할 수 있다.  
여기까지 고민한 것이 바로 GOF에서 설명하는 데코레이터 패턴의 기본 예제이다.

### 프록시 패턴 vs 데코레이터 패턴  
여기까지 진행하면 몇가지 의문이 들 것이다.
- `Decorator`라는 추상 클래스를 만들어야 데코레이터 패턴일까?
- 프록시 패턴과 데코레이터 패턴은 그 모양이 거의 비슷한 것 같은데? 

**의도(intent)**  
사실 프록시 패턴과 데코레이터 패턴은 그 모양이 거의 같고, 상황에 따라 정말 똑같을 때도 있다.   
그러면 둘을 어떻게 구분하는 것일까?  
디자인 패턴에서 중요한 것은 해당 패턴의 겉모양이 아니라 그 패턴을 만든 의도가 더 중요하다. 따라서 의도에 따라 패턴을 구분한다.

- 프록시 패턴의 의도: 다른 개체에 대한 **접근을 제어**하기 위해 대리자를 제공
- 데코레이터 패턴의 의도: **객체에 추가 책임(기능)을 동적으로 추가**하고, 기능 확장을 위한 유연한 대안 제공

**정리**  
프록시를 사용하고 해당 프록시가 접근 제어가 목적이라면 프록시 패턴이고, 새로운 기능을 추가하는 것이 목적이라면 데코레이터 패턴이 된다. 


</details>

<details> <summary> 13. 인터페이스 기반 프록시 - 적용 </summary>

## 13. 인터페이스 기반 프록시 - 적용  
인터페이스와 구현체가 있는 V1 App에 지금까지 학습한 프록시를 도입해서 `LogTrace`를 사용해보자.  
**프록시를 사용하면 기존 코드를 전혀 수정하지 않고 로그 추적 기능을 도입할 수 있다.**

V1 App의 기본 클래스 의존 관계와 런타임시 객체 인스턴스 의존 관계는 다음과 같다.  

**V1 기본 클래스 의존 관계**  
![image](https://user-images.githubusercontent.com/28394879/141109598-8cc777ac-4d35-446a-9b9c-c3ac487a1125.png)

**V1 런타임 객체 의존 관계**  
![image](https://user-images.githubusercontent.com/28394879/141109655-654df878-9fb9-4e55-b2c2-2a49471a12e4.png)

### 여기에 로그 추적용 프록시를 추가하면 다음과 같다.**

**V1 프록시 의존 관계 추가**  
![image](https://user-images.githubusercontent.com/28394879/141109751-0d7eb665-127e-4b4d-ac44-04ca8c72b3b3.png)

`Controller`, `Service`, `Repository`각각 인터페이스에 맞는 프록시 구현체를 추가한다. (그림에서 리포지토리는 생략했다.)

**V1 프록시 런타임 객체 의존 관계**  
![image](https://user-images.githubusercontent.com/28394879/141109890-9cdff8f8-9855-4314-b2a8-0fd3eda86799.png)

그리고 애플리케이션 실행 시점에 프록시를 사용하도록 의존 관계를 설정해주어야 한다. 이 부분은 빈을 등록하는 설정 파일을 활용하면 된다. (그림에서 리포지토리는 생략했다.)

그럼 실제 프록시를 코드에 적용해보자.

**OrderRepositoryInterfaceProxy**
```java
package hello.proxy.config.v1_proxy.interface_proxy;
import hello.proxy.app.v1.OrderRepositoryV1;
import hello.proxy.trace.TraceStatus;
import hello.proxy.trace.logtrace.LogTrace;
import lombok.RequiredArgsConstructor;
@RequiredArgsConstructor
public class OrderRepositoryInterfaceProxy implements OrderRepositoryV1 {
 private final OrderRepositoryV1 target;
 private final LogTrace logTrace;
 @Override
 public void save(String itemId) {
 TraceStatus status = null;
 try {
 status = logTrace.begin("OrderRepository.request()");
 //target 호출
 target.save(itemId);
 logTrace.end(status);
 } catch (Exception e) {
 logTrace.exception(status, e);
 throw e;
 }
 }
}
```
- 프록시를 만들기 위해 인터페이스를 구현하고 구현한 메서드에 `LogTrace`를 사용하는 로직을 추가한다. 지금까지는 `OrderRepositoryImpl`에 이런 로직을 모두 추가해야했다. 프록시를 사용한 덕분에 이 부분을 프록시가 대신 처리해준다. 따라서 `OrderRepositoryImpl` 코드를 변경하지 않아도 된다.
- `OrderRepositoryV1 target`: 프록시가 실제 호출할 원본 리포지터리의 참조를 가지고 있어야 한다.

**OrderServiceInterfaceProxy**
```java
package hello.proxy.config.v1_proxy.interface_proxy;
import hello.proxy.app.v1.OrderServiceV1;
import hello.proxy.trace.TraceStatus;
import hello.proxy.trace.logtrace.LogTrace;
import lombok.RequiredArgsConstructor;
@RequiredArgsConstructor
public class OrderServiceInterfaceProxy implements OrderServiceV1 {
 private final OrderServiceV1 target;
 private final LogTrace logTrace;
 @Override
 public void orderItem(String itemId) {
 TraceStatus status = null;
 try {
 status = logTrace.begin("OrderService.orderItem()");
 //target 호출
 target.orderItem(itemId);
 logTrace.end(status);
 } catch (Exception e) {
 logTrace.exception(status, e);
 throw e;
 }
 }
}
```
앞과 같다.

**OrderControllerInterfaceProxy**
```java
package hello.proxy.config.v1_proxy.interface_proxy;
import hello.proxy.app.v1.OrderControllerV1;
import hello.proxy.trace.TraceStatus;
import hello.proxy.trace.logtrace.LogTrace;
import lombok.RequiredArgsConstructor;
@RequiredArgsConstructor
public class OrderControllerInterfaceProxy implements OrderControllerV1 {
 private final OrderControllerV1 target;
 private final LogTrace logTrace;
 @Override
 public String request(String itemId) {
 TraceStatus status = null;
 try {
 status = logTrace.begin("OrderController.request()");
 //target 호출
 String result = target.request(itemId);
 logTrace.end(status);
 return result;
 } catch (Exception e) {
 logTrace.exception(status, e);
 throw e;
 }
 }
 @Override
 public String noLog() {
 return target.noLog();
 }
}
```
- `noLog()` 메서드는 로그를 남기지 않아야 한다. 따라서 별도의 로직 없이 단순히 `target`을 호출하면 된다.

**InterfaceProxyConfig**
```java
package hello.proxy.config.v1_proxy;
import hello.proxy.app.v1.*;
import
hello.proxy.config.v1_proxy.interface_proxy.OrderControllerInterfaceProxy;
import
hello.proxy.config.v1_proxy.interface_proxy.OrderRepositoryInterfaceProxy;
import hello.proxy.config.v1_proxy.interface_proxy.OrderServiceInterfaceProxy;
import hello.proxy.trace.logtrace.LogTrace;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
@Configuration
public class InterfaceProxyConfig {
 @Bean
 public OrderControllerV1 orderController(LogTrace logTrace) {
 OrderControllerV1Impl controllerImpl = new
OrderControllerV1Impl(orderService(logTrace));
 return new OrderControllerInterfaceProxy(controllerImpl, logTrace);
 }
 @Bean
 public OrderServiceV1 orderService(LogTrace logTrace) {
 OrderServiceV1Impl serviceImpl = new
OrderServiceV1Impl(orderRepository(logTrace));
 return new OrderServiceInterfaceProxy(serviceImpl, logTrace);
 }
 @Bean
 public OrderRepositoryV1 orderRepository(LogTrace logTrace) {
 OrderRepositoryV1Impl repositoryImpl = new OrderRepositoryV1Impl();
 return new OrderRepositoryInterfaceProxy(repositoryImpl, logTrace);
 }
}
```

`LogTrace`가 아직 스프링 빈으로 등록되어 있지 않은데, 이 부분은 바로 다음에 등록할 것이다.

### V1 프록시 런타임 객체 의존 관계 설정 
- 이제 프록시의 런타임 객체 의존 관계를 설정하면 된다. 기존에는 스프링 빈이 `orderControllerV1Impl`, `orderServiceV1Impl`같은 실제 객체를 반환했다. 하지만 이제는 프록시를 사용해야 한다. 따라서 프록시를 생성하고 **프록시를 실제 스프링 빈 대신 등록한다. 실제 객체는 스프링 빈으로 등록하지 않는다.**
- 프록시는 내부에 실제 객체를 참조하고 있다. 예를 들어서 `OrderServiceInterfaceProxy`는 내부에 실제 대상 객체인 `OrderServiceV1Impl`을 가지고 있다.
- 정리하면 다음과 같은 의존 관계를 가지고 있다.
  - `proxy -> target`
  - `orderServiceInterfaceProxy -> orderServiceV1Impl`
- 스프링 빈으로 실제 객체 대신에 프록시 객체를 등록했기 때문에 앞으로 스프링 빈을 주입 받으면 **실제 객체 대신에 프록시 객체가 주입**된다.
- 실제 객체가 스프링 빈으로 등록되지 않는다고 해서 사라지는 것은 아니다. 프록시 객체가 실제 객체를 참조하기 때문에 프록시를 통해서 실제 객체를 호출할 수 있다. 쉽게 이야기해서 프록시 객체 안에 실제 객체가 있는 것이다. 

![image](https://user-images.githubusercontent.com/28394879/141110663-44ced687-4d5f-4c7e-bf46-f08e3ded887b.png)
`AppV1Config`를 통해 프록시를 적용하기 전
- 실제 객체가 스프링 빈으로 등록된다. 빈 객체의 마지막에 `@x0..`라고 해둔 것은 인스턴스라는 뜻이다.


![image](https://user-images.githubusercontent.com/28394879/141110767-7401a632-29d8-45d9-8b9a-12f285636e28.png)
`InterfaceProxyConfig`를 통해 프록시를 적용한 후
- 스프링 컨테이너에 프록시 객체가 등록된다. 스프링 컨테이너는 이제 실제 객체가 아니라 프록시 객체를 스프링 빈으로 관리한다.
- 이제 실제 객체는 스프링 컨테이너와 상관이 없다. 실제 객체는 프록시 객체를 통해서 참조될 뿐이다.
- 프록시 객체는 스프링 컨테이너가 관리하고 자바 힙 메모리에도 올라간다. 반면에 실제 객체는 자바 힙 메모리에는 올라가지만 스프링 컨테이너가 관리하지는 않는다. 

![image](https://user-images.githubusercontent.com/28394879/141110949-af0ea37a-3e90-4576-85cb-7fcb32aa5d65.png)  
최종적으로 이런 런타임 객체 의존관계가 발생한다. (리포지터리는 생략했다.)

**ProxyApplication**
```java
package hello.proxy;
import hello.proxy.config.AppV1Config;
import hello.proxy.config.AppV2Config;
import hello.proxy.config.v1_proxy.InterfaceProxyConfig;
import hello.proxy.trace.logtrace.LogTrace;
import hello.proxy.trace.logtrace.ThreadLocalLogTrace;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Import;
//@Import({AppV1Config.class, AppV2Config.class})
@Import(InterfaceProxyConfig.class)
@SpringBootApplication(scanBasePackages = "hello.proxy.app")
public class ProxyApplication {
 public static void main(String[] args) {
 SpringApplication.run(ProxyApplication.class, args);
 }
 @Bean
 public LogTrace logTrace() {
 return new ThreadLocalLogTrace();
 }
}
```
- `@Bean`: 먼저 `LogTrace` 스프링 빈 추가를 먼저 해주어야 한다. 이것을 여기에 등록한 이유는 앞으로 사용할 모든 예제에서 함께 사용하기 위해서다.
- `@Import(InterfaceProxyConfig.class)`: 프록시를 적용한 설정 파일을 사용하자
  - `//@Import({AppV1Config.class, AppV2Config.class}) 주석 처리하자.

**실행**
- http://localhost:8080/v1/request?itemId=hello

**실행 결과 - 로그**
```
[65b39db2] OrderController.request()
[65b39db2] |-->OrderService.orderItem()
[65b39db2] | |-->OrderRepository.request()
[65b39db2] | |<--OrderRepository.request() time=1002ms
[65b39db2] |<--OrderService.orderItem() time=1002ms
[65b39db2] OrderController.request() time=1003ms
```

실행 결과를 확인해보면 로그 추적 기능이 프록시를 통해 잘 동작하는 것을 확인할 수 있다.

**정리**  
추가된 요구사항들을 다시 확인해보자.

**추가된 요구사항**  
- ~~원본 코드를 전혀 수정하지 않고, 로그 추적기를 적용해라~~
- ~~특정 메서드는 로그를 출력하지 않는 기능~~
  - ~~보안상 일부는 로그를 출력하면 안된다.~~
- 다음과 같은 다양한 케이스에 적용할 수 있어야 한다.
  - ~~v1 - 인터페이스가 있는 구현 클래스에 적용~~
  - v2 - 인터페이스가 없는 구체 클래스에 적용
  - v3 - 컴포넌트 스캔 대상에 기능 적용  

프록시와 DI 덕분에 원본 코드를 전혀 수정하지 않고, 로그 추적기를 도입할 수 있었다. 물론 너무 많은 프록시 클래스를 만들어야 하는 단점이 있기는 하다.  
이 부분은 나중에 해결하기로 하고, 우선은 v2 - 인터페이스가 없는 구체 클래스에 프록시를 어떻게 적용할 수 있는지 알아보자.





</details>

<details> <summary> 14. 구체 클래스 기반 프록시 - 예제1 </summary>

## 14. 구체 클래스 기반 프록시 - 예제1

이번에는 구체 클래스에 프록시를 적용하는 방법을 학습해보자.

다음에 보이는 `ConcreteLogic`은 인터페이스가 없고 구체 클래스만 있다. 이렇게 인터페이스가 없어도 프록시를 적용할 수 있을까?  
먼저 프록시를 도입하기 전에 기본 코드를 작성해보자.

**ConcreteLogic**  
주의: 테스트 패키지에 위치한다.
```java
package hello.proxy.pureproxy.concreteproxy.code;
import lombok.extern.slf4j.Slf4j;
@Slf4j
public class ConcreteLogic {
 public String operation() {
 log.info("ConcreteLogic 실행");
 return "data";
 }
}
```
`ConcreteLogic`은 인터페이스가 없고, 구체 클래스만 있다. 여기에 프록시를 도입해야 한다.

![image](https://user-images.githubusercontent.com/28394879/141116223-ee6def4e-705d-4113-830a-98bf2a0321ea.png)

![image](https://user-images.githubusercontent.com/28394879/141116246-2978d016-5058-48f2-904b-8c49e08796fe.png)


**ConcreteClient**  
주의: 테스트 패키지에 위치한다.
```java
package hello.proxy.pureproxy.concreteproxy.code;
public class ConcreteClient {
 private ConcreteLogic concreteLogic;
 public ConcreteClient(ConcreteLogic concreteLogic) {
 this.concreteLogic = concreteLogic;
 }
 public void execute() {
 concreteLogic.operation();
 }
}
```

**ConcreteProxyTest**
```java
package hello.proxy.pureproxy.concreteproxy;
import hello.proxy.pureproxy.concreteproxy.code.ConcreteClient;
import hello.proxy.pureproxy.concreteproxy.code.ConcreteLogic;
import org.junit.jupiter.api.Test;
public class ConcreteProxyTest {
 @Test
 void noProxy() {
 ConcreteLogic concreteLogic = new ConcreteLogic();
 ConcreteClient client = new ConcreteClient(concreteLogic);
 client.execute();
 }
}
```
코드가 단순해서 이해하는데 어려움은 없을 것이다.




</details>

<details> <summary> 15. 구체 클래스 기반 프록시 - 예제2 </summary>

## 15. 구체 클래스 기반 프록시 - 예제2

### 클래스 기반 프록시 도입
지금까지 인터페이스를 기반으로 프록시를 도입했다. 그런데 자바의 다형성은 인터페이스를 구현하든, 아니면 클래스를 상속하든 상위 타입만 맞으면 다형성이 적용된다.  
쉽게 이야기해서 인터페이스가 없어도 프록시를 만들 수 있다는 뜻이다.   
그래서 이번에는 인터페이스가 아니라 클래스를 기반으로 상속을 받아서 프록시를 만들어보자.

![image](https://user-images.githubusercontent.com/28394879/141117171-d2f13140-8672-46d6-bf26-fa1c20f1b9b6.png)

**TimeProxy**  
주의: 테스트 패키지에 위치한다.
```java
package hello.proxy.pureproxy.concreteproxy.code;
import lombok.extern.slf4j.Slf4j;
@Slf4j
public class TimeProxy extends ConcreteLogic {
 private ConcreteLogic realLogic;
 public TimeProxy(ConcreteLogic realLogic) {
 this.realLogic = realLogic;
 }
 @Override
 public String operation() {
 log.info("TimeDecorator 실행");
 long startTime = System.currentTimeMillis();
 String result = realLogic.operation();
 long endTime = System.currentTimeMillis();
 long resultTime = endTime - startTime;
 log.info("TimeDecorator 종료 resultTime={}", resultTime);
 return result;
 }
}
```
`TimeProxy` 프록시는 시간을 측정하는 부가 기능을 제공한다. 그리고 인터페이스가 아니라 클래스인 `ConcreteLogic`를 상속 받아서 만든다.

**ConcreteProxyTest - addProxy() 추가**
```java
@Test
void addProxy() {
 ConcreteLogic concreteLogic = new ConcreteLogic();
 TimeProxy timeProxy = new TimeProxy(concreteLogic);
 ConcreteClient client = new ConcreteClient(timeProxy);
 client.execute();
}
```

여기서 핵심은 `ConcreteClient`의 생성자에 `concreteLogic`이 아니라 `timeProxy`를 주입하는 부분이다.  

`ConcreteClient`는 `ConcreteLogic`을 의존하는데, 다형성에 의해 `ConcreteLogic`에 `concreteLogic`도 들어갈 수 있고, `timeProxy`도 들어갈 수 있다.

**ConcreteLogic에 할당 할 수 있는 객체**
- `ConcreteLogic = concreteLogic` (본인과 같은 타입을 할당)
- `ConcreteLogic = timeProxy` (자식 타입을 할당)

**ConcreteClient 참고**
```java
public class ConcreteClient {
 private ConcreteLogic concreteLogic; //ConcreteLogic, TimeProxy 모두 주입 가능
 public ConcreteClient(ConcreteLogic concreteLogic) {
 this.concreteLogic = concreteLogic;
 }
 public void execute() {
 concreteLogic.operation();
 }
}
```

**실행 결과**
```
TimeDecorator 실행
ConcreteLogic 실행
TimeDecorator 종료 resultTime=1
```

실행 결과를 보면 인터페이스가 없어도 클래스 기반의 프록시가 잘 적용된 것을 확인할 수 있다.

> 참고: 자바 언어에서 다형성은 인터페이스나 클래스를 구분하지 않고 모두 적용한다. 해당 타입과 그 타입의  
> 하위 타입은 모두 다형성의 대상이 된다. 자바 언어의 너무 기본적인 내용을 이야기했지만, 인터페이스가  
> 없어도 프록시가 가능하다는 것을 확실하게 집고 넘어갈 필요가 있어서 자세히 설명했다.


</details>

<details> <summary> 16. 구체 클래스 기반 프록시 - 적용 </summary>

## 16. 구체 클래스 기반 프록시 - 적용

이번에는 앞서 학습한 내용을 기반으로 구체 클래스만 있는 V2 애플리케이션에 프록시 기능을 적용해보자 

**OrderRepositoryConcreteProxy**
```java
package hello.proxy.config.v1_proxy.concrete_proxy;
import hello.proxy.app.v2.OrderRepositoryV2;
import hello.proxy.trace.TraceStatus;
import hello.proxy.trace.logtrace.LogTrace;
public class OrderRepositoryConcreteProxy extends OrderRepositoryV2 {
 private final OrderRepositoryV2 target;
 private final LogTrace logTrace;
 public OrderRepositoryConcreteProxy(OrderRepositoryV2 target, LogTrace
logTrace) {
 this.target = target;
 this.logTrace = logTrace;
 }
 @Override
 public void save(String itemId) {
 TraceStatus status = null;
 try {
 status = logTrace.begin("OrderRepository.save()");
 //target 호출
 target.save(itemId);
 logTrace.end(status);
 } catch (Exception e) {
 logTrace.exception(status, e);
 throw e;
 }
 }
}
```
인터페이스가 아닌 `OrderRepositoryV2` 클래스를 상속 받아서 프록시를 만든다. 

**OrderServiceConcreteProxy**
```java
package hello.proxy.config.v1_proxy.concrete_proxy;
import hello.proxy.app.v2.OrderServiceV2;
import hello.proxy.trace.TraceStatus;
import hello.proxy.trace.logtrace.LogTrace;
public class OrderServiceConcreteProxy extends OrderServiceV2 {
 private final OrderServiceV2 target;
 private final LogTrace logTrace;
 public OrderServiceConcreteProxy(OrderServiceV2 target, LogTrace logTrace)
{
 super(null);
 this.target = target;
 this.logTrace = logTrace;
 }
 @Override
 public void orderItem(String itemId) {
 TraceStatus status = null;
 try {
 status = logTrace.begin("OrderService.orderItem()");
 //target 호출
 target.orderItem(itemId);
 logTrace.end(status);
 } catch (Exception e) {
 logTrace.exception(status, e);
 throw e;
 }
 }
}
```
- 인터페이스가 아닌 `OrderServiceV2` 클래스를 상속 받아서 프록시를 만든다.

**클래스 기반 프록시의 단점**
- `super(null)`: `OrderServiceV2`: 자바 기본 문법에 의해 자식 클래스를 생성할 때는 항상 `super()`로 부모 클래스의 생성자를 호출해야 한다. 이 부분을 생략하면 기본 생성자가 호출된다. 그런데 부모 클래스인 `OrderServiceV2`는 기본 생성자가 없고, 생성자에서 파라미터 1개를 필수로 받는다. 따라서 파라미터를 넣어서 `super(..)`를 호출해야 한다.
- 프록시는 부모 객체의 기능을 사용하지 않기 때문에 `super(null)`을 입력해도 된다.
- 인터페이스 기반 프록시는 이런 고민을 하지 않아도 된다.

**OrderServiceV2의 생성자 - 참고**
```java
public OrderServiceV2(OrderRepositoryV2 orderRepository) {
 this.orderRepository = orderRepository;
}
```

**OrderControllerConcreteProxy**
```java
package hello.proxy.config.v1_proxy.concrete_proxy;
import hello.proxy.app.v2.OrderControllerV2;
import hello.proxy.trace.TraceStatus;
import hello.proxy.trace.logtrace.LogTrace;
public class OrderControllerConcreteProxy extends OrderControllerV2 {
 private final OrderControllerV2 target;
 private final LogTrace logTrace;
 public OrderControllerConcreteProxy(OrderControllerV2 target, LogTrace
logTrace) {
 super(null);
 this.target = target;
 this.logTrace = logTrace;
 }
 @Override
 public String request(String itemId) {
 TraceStatus status = null;
 try {
 status = logTrace.begin("OrderController.request()");
 //target 호출
 String result = target.request(itemId);
 logTrace.end(status);
 return result;
 } catch (Exception e) {
 logTrace.exception(status, e);
 throw e;
 }
 }
}
```
앞과 같다.

**ConcreteProxyConfig**
```java
package hello.proxy.config.v1_proxy;
import hello.proxy.app.v2.OrderControllerV2;
import hello.proxy.app.v2.OrderRepositoryV2;
import hello.proxy.app.v2.OrderServiceV2;
import hello.proxy.config.v1_proxy.concrete_proxy.OrderControllerConcreteProxy;
import hello.proxy.config.v1_proxy.concrete_proxy.OrderRepositoryConcreteProxy;
import hello.proxy.config.v1_proxy.concrete_proxy.OrderServiceConcreteProxy;
import hello.proxy.trace.logtrace.LogTrace;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
@Configuration
public class ConcreteProxyConfig {
 @Bean
 public OrderControllerV2 orderControllerV2(LogTrace logTrace) {
 OrderControllerV2 controllerImpl = new
OrderControllerV2(orderServiceV2(logTrace));
 return new OrderControllerConcreteProxy(controllerImpl, logTrace);
 }
 @Bean
 public OrderServiceV2 orderServiceV2(LogTrace logTrace) {
 OrderServiceV2 serviceImpl = new
OrderServiceV2(orderRepositoryV2(logTrace));
 return new OrderServiceConcreteProxy(serviceImpl, logTrace);
 }
 @Bean
 public OrderRepositoryV2 orderRepositoryV2(LogTrace logTrace) {
 OrderRepositoryV2 repositoryImpl = new OrderRepositoryV2();
 return new OrderRepositoryConcreteProxy(repositoryImpl, logTrace);
 }
}
```
인터페이스 대신에 구체 클래스를 기반으로 프록시를 만든다는 것을 제외하고는 기존과 같다.

**ProxyApplication**
```java
package hello.proxy;
import hello.proxy.config.AppV1Config;
import hello.proxy.config.AppV2Config;
import hello.proxy.config.v1_proxy.ConcreteProxyConfig;
import hello.proxy.config.v1_proxy.InterfaceProxyConfig;
import hello.proxy.trace.logtrace.LogTrace;
import hello.proxy.trace.logtrace.ThreadLocalLogTrace;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Import;
//@Import({AppV1Config.class, AppV2Config.class})
//@Import(InterfaceProxyConfig.class)
@Import(ConcreteProxyConfig.class)
@SpringBootApplication(scanBasePackages = "hello.proxy.app")
public class ProxyApplication {
 public static void main(String[] args) {
 SpringApplication.run(ProxyApplication.class, args);
 }
 @Bean
 public LogTrace logTrace() {
 return new ThreadLocalLogTrace();
 }
}
```
- `@Import(ConcreteProxyConfig.class)`: 설정을 추가하자
  

**실행**  
- http://localhost:8080/v2/request?itemId=hello

실행해보면 클래스 기반 프록시도 잘 동작하는 것을 확인할 수 있다.



</details>

<details> <summary> 17. 인터페이스 기반 프록시와 클래스 기반 프록시 </summary>

## 17. 인터페이스 기반 프록시와 클래스 기반 프록시

**프록시**  
프록시를 사용한 덕분에 원본 코드를 전혀 변경하지 않고, V1, V2 애플리케이션에 `LogTrace`기능을 적용할 수 있었다.

**인터페이스 기반 프록시 vs 클래스 기반 프록시**
- 인터페이스가 없어도 클래스 기반으로 프록시를 생성할 수 있다.
- 클래스 기반 프록시는 해당 클레스에만 적용할 수 있다. 인터페이스 기반 프록시는 인터페이만 같으면 모든 곳에 적용할 수 있다.
- 클래스 기반 프록시는 상속을 사용하기 떄문에 몇가지 제약이 있다.
  - 부모 클래스의 생성자를 호출해야 한다(앞서 본 예제)
  - 클래스에 final 키워드가 붙으면 상속이 불가능하다.
  - 메서드에 final 키워드가 붙으면 해당 메서드를 오버라이딩 할 수 없다.

이렇게 보면 인터페이스 기반의 프록시가 더 좋아보인다. 맞다.  
인터페이스 기반의 프록시는 상속이라는 제약에서 자유롭다.  
프로그래밍 관점에서도 인터페이스를 사용하는 것이 역할과 구현을 명확하게 나누기 때문에 더 좋다.  
인터페이스 기반 프록시의 단점은 인터페이스가 필요하다는 그 자체이다. 인터페이스가 없으면 인터페이스 기반 프록시를 만들 수 없다.  

> 참고: 인터페이스 기반 프록시는 캐스팅 관련해서 단점이 있는데, 이 내용은 뒷부분에서 설명한다.

이론적으로는 모든 객체에 인터페이스를 도입해서 역할과 구현을 나누는 것이 좋다. 이렇게 하면 역하로가 구현을 나누어서 구현체를 매우 편리하게 변경할 수 있다.  
하지만 실제노느 구현을 거의 변경할 일이 없는 클래스도 많다.  
인터페이스를 도입하는 것은 구현을 변경할 가능성이 있을 때 효과적인데, 구현을 변경할 가능성이 거의 없는 코드에 무작정 인터페이스를 사용하는 것은 번거롭고 그렇게 실용적이지 않다.   
이런곳에는 실용적인 관점에서 인터페이스를 사용하지 않고 구체 클래스를 사용하는 것이 좋다 생각한다.   
(물론 인터페이스르 도입하는 다양한 이유가 있다. 여기서 핵심은 인터페이스가 항상 필요하지는 않다는 것이다.)

**결론**  
실무에서는 프록시를 적용할 떄 V1처럼 인터페이스도 있고, V2처럼 구체 클래스도 있다. 따라서 2가지 상황을 모두 대응할 수 있어야 한다. 

**너무 많은 프록시 클래스**  
지금까지 프록시를 사용해서 기존 코드를 변경하지 않고, 로그 추적기라는 부가 기능을 적용할 수 있었다.  
그런데 문제는 프록시 클래스를 너무 많이 만들어야 한다는 점이다.   
잘 보면 프록시 클래스가 하는 일은 `LogTrace`를 사용하는 것인데, 그 로직이 모두 똑같다.   
대상 클래스만 다를 뿐이다.  
만약 적용해야 하는 대상 클래스가 100개라면 프록시 클래스도 100개를 만들어야 한다.  
프록시 클래스를 하나만 만들어서 모든곳에 적용하는 방법은 없을까?  
**바로 다음에 설명할 동적 프록시 기술이 이 문제를 해결해준다.**


</details>

# [5. 동적 프록시 기술](./5.dynamic-proxy-technology)

<details> <summary> 1. 리플렉션 </summary>

## 1. 리플렉션

지금까지 프록시를 사용해서 기존 코드를 변경하지 않고, 로그 추적기라는 부가 기능을 적용할 수 있었다.  
그런데 문제는 대상 클래스 수 만큼 로그 추적을 위한 프록시 클래스를 만들어야 한다는 점이다.  
로그 추적을 위한 프록시 클래스들의 소스코드는 거의 같은 모양을 하고 있다.  

자바 기본으로 제공하는 JDK 동적 프록시 기술이나 CGLIB 같은 프록시 생성 오픈소스 기술을 활용하면   
프록시 객체를 동적으로 만들어낼 수 있다. 쉽개 이야기해서 프록시 클래스를 지금처럼 계속 만들지 않아도 된다는 것이다.  
프록시를 적용할 코드를 하나만 만들어두고 동적 프록시 기술을 사용해서 프록시 객체를 찍어내면 된다.  
자세한 내용은 조금 뒤에 코드로 확인해보자. 

JDK 동적 프록시를 이해하기 위해서는 먼저 자바의 리플렉션 기술을 이해해야 한다.  
리플렉션 기술을 사용하면 클래스나 메서드의 메타정보를 동적으로 획득하고, 코드도 동적으로 호출할 수 있다.  
여기서는 JDK 동적 프록시를 이해하기 위한 최소한의 리플랙션 기술을 알아보자.

**ReflectionTest**
```java
package hello.proxy.jdkdynamic;
import lombok.extern.slf4j.Slf4j;
import org.junit.jupiter.api.Test;
import java.lang.reflect.Method;
@Slf4j
public class ReflectionTest {
 @Test
 void reflection0() {
 Hello target = new Hello();
 //공통 로직1 시작
 log.info("start");
 String result1 = target.callA(); //호출하는 메서드가 다름
 log.info("result={}", result1);
 //공통 로직1 종료
 //공통 로직2 시작
 log.info("start");
 String result2 = target.callB(); //호출하는 메서드가 다름
 log.info("result={}", result2);
 //공통 로직2 종료
 }
 @Slf4j
 static class Hello {
 public String callA() {
 log.info("callA");
 return "A";
 }
 public String callB() {
 log.info("callB");
 return "B";
 }
 }
}
```
- 공통 로직1과 공통 로직2는 호출하는 메서드만 다르고 전체 코드 흐름이 완전히 같다.
  - 먼저 start 로그를 출력한다.
  - 어떤 메서드를 호출한다.
  - 메서드의 호출 결과를 로그로 출력한다.
- 여기서 공통 로직1과 공통 로직2를 하나의 메서드로 뽑아서 합칠 수 있을까?
- 쉬워 보이지만 메서드로 뽑아서 공통화하는 것이 생각보다 어렵다. 왜냐하면 중간에 호출하는 메서드가 다르기 떄문이다.
- 호출하는 메서드인 `target.callA()`,`target.callB()`이 부분만 동적으로 처리할 수 있다면 문제를 해결할 수 있을듯 하다.

```java
log.info("start");
String result = xxx(); //호출 대상이 다름, 동적 처리 필요
log.info("result={}", result);
```

이럴 떄 사용하는 기술이 바로 리플랙션이다. 리플랙션은 클래스나 메서드의 메타정보를 사용해서 동적으로 호출하는 메서드를 변경할 수 있따. 바로 리플랙션 사용해보자.

> 참고: 람다를 사용해서 공통화 하는 것도 가능하다. 여기서는 람다를 사용하기 어려운 상황이라 가정하자.  
> 그리고 리플랙션 학습이 목적이니 리플랙션에 집중하자.

**ReflectionTest - reflection1 추가**
```java
@Test
void reflection1() throws Exception {
 //클래스 정보
 Class classHello =
Class.forName("hello.proxy.jdkdynamic.ReflectionTest$Hello");
 Hello target = new Hello();
 //callA 메서드 정보
 Method methodCallA = classHello.getMethod("callA");
 Object result1 = methodCallA.invoke(target);
 log.info("result1={}", result1);
 //callB 메서드 정보
 Method methodCallB = classHello.getMethod("callB");
 Object result2 = methodCallB.invoke(target);
 log.info("result2={}", result2);
}
```
- `Class.forName("hello.proxy.jdkdynamic.ReflectionTest$Hello")`: 클래스 메타정보를 획득한다. 참고로 내부 클래스는 구분을 위해 `$`를 사용한다.
- `classHello.getMethod("call")`: 해당 클래스의 `call` 메서드 메타정보를 획득한다.
- `methodCallA.invoke(target)`: 획득한 메서드 메타정보로 실제 인스턴스의 메서드를 호출한다. 여기서 `methodCallA`는 `Hello`클래스의 `callA()`이라는 메서드 메타정보이다. 

`methodCallA.invoke(인스턴스)`를 호출하면서 인스턴스를 넘겨주면 해당 인스턴스의 `callA()`메서드를 찾아서 실행한다. 여기서는 `target`의 `callA()` 메서드를 호출한다.

그런데 `target.callA()`나 `target.callB()` 메서드를 직접 호출하면 되지 이렇게 메서드 정보를 획득해서 메서드를 호출하면 어떤 효과가 있을까? 여기서 중요한 핵심은 클래스나 메서드 정보를 동적으로 변경할 수 있다는 점이다.

기존의 `callA()`,`callB()` 메서드를 직접 호출하는 부분이 `Method`로 대체되었다. 덕분에 이제 공통로직을 만들 수 있게 되었따.

**ReflectionTest - reflection2 추가**
```java
@Test
void reflection2() throws Exception {
 Class classHello =
Class.forName("hello.proxy.jdkdynamic.ReflectionTest$Hello");
 Hello target = new Hello();
 Method methodCallA = classHello.getMethod("callA");
 dynamicCall(methodCallA, target);
 Method methodCallB = classHello.getMethod("callB");
 dynamicCall(methodCallB, target);
}
private void dynamicCall(Method method, Object target) throws Exception {
 log.info("start");
 Object result = method.invoke(target);
 log.info("result={}", result);
}
```
- `dynamicCall(Method method, Object target)`
  - 공통 로직1, 공통 로직2를 한번에 처리할 수 있는 통합된 공통 처리 로직이다.
  - `Method method`: 첫 번째 파라미터는 호출할 메서드 정보가 넘어온다. 이것이 핵심이다. 기존에는 메서드 이름을 직접 호출했지만, 이제는 `Method`라는 메타정보를 통해서 호출할 메서드 정보가 동적으로 제공된다.
  - `Object target`: 실제 실행할 인스턴스 정보가 넘어온다. 타입이 `Object`라는 것은 어떠한 인스턴스도 받을 수 있다는 뜻이다. 물론 `method.invoke(target)`를 사용할 때 호출할 클래스와 메서드 정보가 서로 다르면 예외가 발생한다.

**정리**   
정적인 `target.callA()`,`target.callB()` 코드를 리플랙션을 사용해서 `Method`라는 메타정보로 추상화했다. 덕분에 공통 로직을 만들 수 있게 되었다.

**주의**  
리플렉션을 사용하면 클래스와 메서드의 메타정보를 사용해서 애플리케이션을 동적으로 유연하게 만들 수 있다.   
하지만 리플랙션 기술은 런타임에 동작하기 떄문에, 컴파일 시점에 오류를 잡을 수 없다.  
예를 들어서 지금까지 살펴본 코드에서 `getMethod("callA")`안에 들어가는 문자를 실수로 `getMethod("callZ")`로 작성해도 컴파일 오류가 발생하지 않는다.  
그러나 해당 코드를 직접 실행하는 시점에 발생하는 오류인 런타임 오류가 발생한다.  
가장 좋은 오류는 개발자가 즉시 확인할 수 있는 컴파일 오류이고, 가장 무서운 오류는 사용자가 직접 실행할때 발생하는 런타임 오류다.  

따라서 리플렉션은 일반적으로 사용하면 안된다. 지금까지 프로그래밍 언어가 발달하면서 타입 정보를 기반으로 컴파일 시점에 오류를 잡아준 덕분에 개발자가 편하게 살았는데, 리플렉션은 그것에 역행하는 방식이다.

리플렉션은 프레임워크 개발이나 또는 매우 일반적인 공통 처리가 필요할 때 부분적으로 주의해서 사용해야한다.

</details>

<details> <summary> 2. JDK 동적 프록시 - 소개 </summary>

## 2. JDK 동적 프록시 - 소개

지금까지 프록시를 적용하기 위해 적용 대상의 숫자 만큼 많은 프록시 클래스를 만들었다.  
적용 대상이 100개면 프록시 클래스도 100개 만들었다.  
그런데 앞서 살펴본 것과 같이 프록시 클래스의 기본 코드와 흐름은 거의 같고, 프록시를 어떤 대상에 적용하는가 정도만 차이가 있었다.  
쉽게 이야기해서 프록시의 로직은 같은데, 적용 대상만 차이가 있는 것이다.

이 문제를 해결하는 것이 바로 동적 프록시 기술이다.  
동적 프록시 기술을 사용하면 개발자가 직접 프록시 클래스를 만들지 않아도 된다.  
이름 그대로 프록시 객체를 동적으로 런타임에 개발자 대신 만들어준다.  
그리고 동적 프록시에 원하는 실행 로직을 지정할 수 있다.  
사실 동적 프록시는 말로는 이해하기 쉽지 않다. 바로 예제 코드를 보자.

> 주의  
> JDK 동적 프록시는 인터페이스를 기반으로 프록시를 동적으로 만들어준다.  
> 따라서 인터페이스가 필수이다.

먼저 자바 언어가 기본으로 제공하는 JDK 동적 프록시를 알아보자.

### 기본 예제 코드  
JDK 동적 프록시르 이해하기 위해 아주 단순한 예제 코드를 만들어보자.  
간단히 `A`,`B` 클래스를 만드는데, JDK 동적 프록시는 인터페이스가 필수이다.  
따라서 인터페이스와 구현체로 구분했다.

**AInterface**  
주의: 테스트 패키지에 위치한다.
```java
package hello.proxy.jdkdynamic.code;
public interface AInterface {
 String call();
}
```

**AImpl**  
주의: 테스트 패키지에 위치한다.
```java
package hello.proxy.jdkdynamic.code;
import lombok.extern.slf4j.Slf4j;
@Slf4j
public class AImpl implements AInterface {
 @Override
 public String call() {
 log.info("A 호출");
 return "a";
 }
}
```

**BInterface**  
주의: 테스트 패키지에 위치한다.  
```java
package hello.proxy.jdkdynamic.code;
public interface BInterface {
 String call();
}
```

**BImpl**  
주의: 테스트 패키지에 위치한다. 
```java
package hello.proxy.jdkdynamic.code;
import lombok.extern.slf4j.Slf4j;
@Slf4j
public class BImpl implements BInterface {
 @Override
 public String call() {
 log.info("B 호출");
 return "b";
 }
}
```




</details>

<details> <summary> 3. JDK 동적 프록시 - 예제 코드 </summary>

## 3. JDK 동적 프록시 - 예제 코드

### JDK 동적 프록시 InvocationHandler

JDK 동적 프록시에 적용할 로직은 `InvocationHandler` 인터페이스를 구현해서 작성하면 된다.

**JDK 동적 프록시가 제공하는 InvocationHandler**
```java
package java.lang.reflect;
public interface InvocationHandler {
 public Object invoke(Object proxy, Method method, Object[] args)
 throws Throwable;
}
```

**제공되는 파라미터는 다음과 같다**
- `Object proxy`: 프록시 자신
- `Method method`: 호출한 메서드
- `Object[] args`: 메서드를 호출할 때 전달할 인수

이제 구현 코드를 보자. 

**TimeInvocationHandler**  
주의: 테스트 패키지에 위치한다.
```java
package hello.proxy.jdkdynamic.code;
import lombok.extern.slf4j.Slf4j;
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
@Slf4j
public class TimeInvocationHandler implements InvocationHandler {
 private final Object target;
 public TimeInvocationHandler(Object target) {
 this.target = target;
 }
 @Override
 public Object invoke(Object proxy, Method method, Object[] args) throws
Throwable {
 log.info("TimeProxy 실행");
 long startTime = System.currentTimeMillis();
 Object result = method.invoke(target, args);
 long endTime = System.currentTimeMillis();
 long resultTime = endTime - startTime;
 log.info("TimeProxy 종료 resultTime={}", resultTime);
 return result;
 }
}
```
- `TimeInvocationHandler`은 `InvocationHAndler` 인터페이스를 구현한다. 이렇게 해서 JDK 동적 프록시에 적용할 공통 로직을 개발할 수 있다.
- `Object target`: 동적 프록시가 호출할 대상
- `method.invoke(target, args)`: 리플렉션을 사용해서 `target` 인스턴스의 메서드를 실행한다. `args`는 메서드 호출시 넘겨줄 인수이다.

이제 테스트 코드로 JDK 동적 프록시를 사용해보자.

**JdkDynamicProxyTest**
```java
package hello.proxy.jdkdynamic;
import hello.proxy.jdkdynamic.code.*;
import lombok.extern.slf4j.Slf4j;
import org.junit.jupiter.api.Test;
import java.lang.reflect.Proxy;
@Slf4j
public class JdkDynamicProxyTest {
 @Test
 void dynamicA() {
 AInterface target = new AImpl();
 TimeInvocationHandler handler = new TimeInvocationHandler(target);
 AInterface proxy = (AInterface)
Proxy.newProxyInstance(AInterface.class.getClassLoader(), new Class[]
{AInterface.class}, handler);
 proxy.call();
 log.info("targetClass={}", target.getClass());
 log.info("proxyClass={}", proxy.getClass());
 }
 @Test
 void dynamicB() {
 BInterface target = new BImpl();
 TimeInvocationHandler handler = new TimeInvocationHandler(target);
 BInterface proxy = (BInterface)
Proxy.newProxyInstance(BInterface.class.getClassLoader(), new Class[]
{BInterface.class}, handler);
 proxy.call();
 log.info("targetClass={}", target.getClass());
 log.info("proxyClass={}", proxy.getClass());
 }
}
```
- `new TimeInvocationHandler(target)`: 동적 프록시에 적용할 핸들러 로직이다.
- `Proxy.newProxyInstance(AInterface.class.getClassLoader(), new Class[] {AInterface.class}, handler)
  - 동적 프록시는 `java.lang.reflect.Proxy`를 통해서 생성할 수 있다.
  - 클래스 로더 정보, 인터페이스, 그리고 핸들러 로직을 넣어주면 된다. 그러면 해당 인터페이스를 기반으로 동적 프록시를 생성하고 그 결과를 반환한다.


**dynamicA() 출력 결과**
```
TimeInvocationHandler - TimeProxy 실행
AImpl - A 호출
TimeInvocationHandler - TimeProxy 종료 resultTime=0
JdkDynamicProxyTest - targetClass=class hello.proxy.jdkdynamic.code.AImpl
JdkDynamicProxyTest - proxyClass=class com.sun.proxy.$Proxy1
```

출력 결과를 보면 프록시가 정상 수행된 것을 확인할 수 있다.

**생성된 JDK 동적 프록시**  
`proxyClass=class com.sun.proxy.$Proxy1`이 부분이 동적으로 생성된 프록시 클래스 정보이다.  
이것은 우리가 만든 클래스가 아니라 JDK 동적 프록시가 이름 그대로 동적으로 만들어준 프록시이다.  
이 프록시는 `TimeInvocationHandler` 로직을 실행한다.

**실행 순서**
1. 클라이언트는 JDK 동적 프록시의 `call()`을 실행한다.
2. JDK 동적 프록시는 `InvocationHandler.invoke()`를 호출한다. `TimeInvocationHandler`가 구현체로 있으므로 `TimeInvocationHandler.invoke()`가 호출된다.
3. `TimeInvocationHandler`가 내부 로직을 수행하고, `method.invoke(target,args)`를 호출해서 `target`인 실제 객체 (`AImpl`)를 호출한다.
4. `AImpl` 인스턴스의 `call()`이 실행된다.
5. `AImpl` 인스턴스의 `call()`의 실행이 끝나면 `TimeInvocationHandler`로 응답이 돌아온다. 시간 로그를 출력하고 결과를 반환한다.

**실행 순서 그림**  
![image](https://user-images.githubusercontent.com/28394879/141285898-f001f86a-fe13-44fe-931c-3dab2bc2034d.png)

**동적 프록시 클래스 정보**  
`dynamicA()`와 `dynamicB()` 둘을 동시에 함께 실행하면 JDK 동적 프록시가 각각 다른 동적 프록시 클래스를 만들어주는 것을 확인할 수 있다.
```
proxyClass=class com.sun.proxy.$Proxy1 //dynamicA
proxyClass=class com.sun.proxy.$Proxy2 //dynamicB
```

**정리**  
예제를 보면 `AImpl`, `BImpl` 각각 프록시를 만들지 않았다. 프록시는 JDK 동적 프록시를 사용해서 동적으로 만들고 `TimeInvocationHandler`는 공통으로 사용했다.  
JDK 동적 프록시 기술 덕분에 적용 대상 만큼 프록시 객체를 만들지 않아도 된다.   
그리고 같은 부가 기능 로직을 한번만 개발해서 공통으로 적용할 수 있다.  
만약 적용 대상이 100개여도 동적 프록시를 통해서 생성하고, 각가 필요한 `InvocationHandler`만 만들어서 넣어주면 된다.  
결과적으로 프록시 클래스를 수 없이 만들어야하는 문제도 해결하고, 부가 기능 로직도 하나의 클래스에 모아서 단일 책임 원칙(SRP)도 지킬 수 있게 되었다.  

JDK 동적 프록시 없이 직접 프록시를 만들어서 사용할 떄와 JDK 동적 프록시를 사용할 때의 차이를 그림으로 비교해보자.

**JDK 동적 프록시 도입 전 - 직접 프록시 생성**   
![image](https://user-images.githubusercontent.com/28394879/141286324-75a3866e-c2f8-41ed-8d12-e92112b43496.png)

**JDK 동적 프록시 도입 후**  
![image](https://user-images.githubusercontent.com/28394879/141286384-372a9d3d-80cc-43c9-88b0-7228b6f1dfc1.png)

- 점선은 개발자가 직접 만드는 클래스가 아니다.

**JDK 동적 프록시 도입 전**  
![image](https://user-images.githubusercontent.com/28394879/141286478-e57b5427-4789-489a-8395-f2aa4840ab15.png)


**JDK 동적 프록시 도입 후**  
![image](https://user-images.githubusercontent.com/28394879/141286527-911fa585-527d-4012-9c49-a913cda7bb5f.png)

지금까지 학습한 JDK 동적 프록시를 애플리케이션에 적용해보자. 


</details>

<details> <summary> 4. JDK 동적 프록시 - 적용1 </summary>

## 4. JDK 동적 프록시 - 적용1

JDK 동적 프록시는 인터페이스가 필수이기 때문에 V1 애플리케이션에만 적용할 수 있다.

먼저 `LogTrace`를 적용할 수 있는 `InvocationHandler`를 만들자.

**LogTraceBasicHandler**
```java
package hello.proxy.config.v2_dynamicproxy.handler;
import hello.proxy.trace.TraceStatus;
import hello.proxy.trace.logtrace.LogTrace;
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
public class LogTraceBasicHandler implements InvocationHandler {

 private final Object target;
 private final LogTrace logTrace;
 public LogTraceBasicHandler(Object target, LogTrace logTrace) {
 this.target = target;
 this.logTrace = logTrace;
 }
 @Override
 public Object invoke(Object proxy, Method method, Object[] args) throws
Throwable {
 TraceStatus status = null;
 try {
 String message = method.getDeclaringClass().getSimpleName() + "."
 + method.getName() + "()";
 status = logTrace.begin(message);
 //로직 호출
 Object result = method.invoke(target, args);
 logTrace.end(status);
 return result;
 } catch (Exception e) {
 logTrace.exception(status, e);
 throw e;
 }
 }
}
```
- `LogTraceBasicHandler`는 `InvocationHandler` 인터페이스를 구현해서 JDK 동적 프록시에서 사용된다.
- `private final Object target`: 프록시가 호출할 대상이다.
- `String message = method.getDeclaringClass().getSimpleName() + "." ...`
  - `LogTrace`에 사용할 메시지이다. 프록시를 직접 개발할 때는 `"OrderController.request()"`와 같이 프록시마다 호출되는 클래스와 메서드 이름을 직접 남겼다. 이제는 `Method`를 통해서 호출되는 메서드 정보와 클래스 정보를 동적으로 확인할 수 있기 때문에 이 정보를 사용하면 된다.

동적 프록시를 사용하도록 수동 빈 등록을 설정하자.

**DynamicProxyBasicConfig**
```java
package hello.proxy.config.v2_dynamicproxy;
import hello.proxy.app.v1.*;
import hello.proxy.config.v2_dynamicproxy.handler.LogTraceBasicHandler;
import hello.proxy.trace.logtrace.LogTrace;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import java.lang.reflect.Proxy;
@Configuration
public class DynamicProxyBasicConfig {
 @Bean
 public OrderControllerV1 orderControllerV1(LogTrace logTrace) {
 OrderControllerV1 orderController = new
OrderControllerV1Impl(orderServiceV1(logTrace));
 OrderControllerV1 proxy = (OrderControllerV1)
Proxy.newProxyInstance(OrderControllerV1.class.getClassLoader(),
 new Class[]{OrderControllerV1.class},
 new LogTraceBasicHandler(orderController, logTrace)
 );
 return proxy;
 }
 @Bean
 public OrderServiceV1 orderServiceV1(LogTrace logTrace) {
 OrderServiceV1 orderService = new
OrderServiceV1Impl(orderRepositoryV1(logTrace));
 OrderServiceV1 proxy = (OrderServiceV1)
Proxy.newProxyInstance(OrderServiceV1.class.getClassLoader(),
 new Class[]{OrderServiceV1.class},
 new LogTraceBasicHandler(orderService, logTrace)
 );
 return proxy;
 }
 @Bean
 public OrderRepositoryV1 orderRepositoryV1(LogTrace logTrace) {
 OrderRepositoryV1 orderRepository = new OrderRepositoryV1Impl();
 OrderRepositoryV1 proxy = (OrderRepositoryV1)
Proxy.newProxyInstance(OrderRepositoryV1.class.getClassLoader(),
 new Class[]{OrderRepositoryV1.class},
 new LogTraceBasicHandler(orderRepository, logTrace)
 );
 return proxy;
 }
}
```
- 이전에는 프록시 클래스를 직접 개발했지만, 이제는 JDK 동적 프록시 기술을 사용해서 각각의 `Controller`,`Service`, `Repository`에 맞는 동적 프록시를 생성해주면 된다.
- `LogTraceBasicHandler`: 동적 프록시를 만들더라도 `LogTrace`를 출력하는 로직은 모두 같기 때문에 프록시는 모두 `LogTraceBasicHandler`를 사용한다.

**ProxyApplication - 수정**
```java
import hello.proxy.config.v2_dynamicproxy.DynamicProxyBasicConfig;
//@Import({AppV1Config.class, AppV2Config.class})
//@Import(InterfaceProxyConfig.class)
//@Import(ConcreteProxyConfig.class)
@Import(DynamicProxyBasicConfig.class)
@SpringBootApplication(scanBasePackages = "hello.proxy.app")
public class ProxyApplication {
 public static void main(String[] args) {
 SpringApplication.run(ProxyApplication.class, args);
 }
 @Bean
 public LogTrace logTrace() {
 return new ThreadLocalLogTrace();
 }
}
```
`@Import(DynamicProxyBasicConfig.class)`: 이제 동적 프록시 설정을 `@Import`하고 실행해보자.

**실행** 
- http://localhost:8080/v1/request?itemId=hello

**그림으로 정리**  
![image](https://user-images.githubusercontent.com/28394879/141290624-4338e13c-8980-4b6b-8b97-76737dafed39.png)

![image](https://user-images.githubusercontent.com/28394879/141290652-0738829e-de07-47fb-bd3e-ea2b9b9de6ea.png)

**남은 문제**  
- http://localhost:8080/v1/no-log
- no-log를 실행해도 동적 프록시가 적용되고, `LogTraceBasicHandler`가 실행되기 떄문에 로그가 남는다. 이 부분을 로그가 남지 않도록 처리해야 한다. 



</details>

<details> <summary> 5. JDK 동적 프록시 - 적용2 </summary>

## 5. JDK 동적 프록시 - 적용2

**메서드 이름 필터 기능 추가** 

- http://localhost:8080/v1/no-log
  
요구사항에 의해 이것을 호출 했을 떄는 로그가 남으면 안된다.  
이런 문제를 해결하기 위해 메서드 이름을 기준으로 특정 조건을 만족할 떄만 로그를 남기는 기능을 개발해보자.

**LogTraceFilterHandler**
```java
package hello.proxy.config.v2_dynamicproxy.handler;
import hello.proxy.trace.TraceStatus;
import hello.proxy.trace.logtrace.LogTrace;
import org.springframework.util.PatternMatchUtils;
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
public class LogTraceFilterHandler implements InvocationHandler {

 private final Object target;
 private final LogTrace logTrace;
 private final String[] patterns;
 public LogTraceFilterHandler(Object target, LogTrace logTrace, String...
patterns) {
 this.target = target;
 this.logTrace = logTrace;
 this.patterns = patterns;
 }
 @Override
 public Object invoke(Object proxy, Method method, Object[] args) throws
Throwable {
 //메서드 이름 필터
 String methodName = method.getName();
 if (!PatternMatchUtils.simpleMatch(patterns, methodName)) {
 return method.invoke(target, args);
 }
 TraceStatus status = null;
 try {
 String message = method.getDeclaringClass().getSimpleName() + "."
 + method.getName() + "()";
 status = logTrace.begin(message);
 //로직 호출
 Object result = method.invoke(target, args);
 logTrace.end(status);
 return result;
 } catch (Exception e) {
 logTrace.exception(status, e);
 throw e;
 }
 }
}
```
- `LogTraceFilterHandler`는 기존 기능에 다음 기능이 추가되었다.
  - 특정 메서드 이름이 매칭 되는 경우에만 `LogTrace`로직을 실행한다. 이름이 매칭되지 않으면 실제 로직을 바로 호출한다.
- 스프링이 제공하는 `PatternMatchUtils.simpleMatch(..)`를 사용하면 단순한 매칭 로직을 쉽게 적용할 수 있다.
  - `xxx`: xxx가 정확히 매칭되면 참
  - `xxx*`: xxx로 시작하면 참
  - `*xxx`: xxx로 끝나면 참
  - `*xxx*`: xxx가 있으면 참
- `String[] patterns`: 적용할 패턴은 생성자를 통해서 외부에서 받는다.


**DynamicProxyFilterConfig**
```java
package hello.proxy.config.v2_dynamicproxy;
import hello.proxy.app.v1.*;
import hello.proxy.config.v2_dynamicproxy.handler.LogTraceFilterHandler;
import hello.proxy.trace.logtrace.LogTrace;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import java.lang.reflect.Proxy;
@Configuration
public class DynamicProxyFilterConfig {
 public static final String[] PATTERNS = {"request*", "order*", "save*"};
 @Bean
 public OrderControllerV1 orderControllerV1(LogTrace logTrace) {
 OrderControllerV1 orderController = new
OrderControllerV1Impl(orderServiceV1(logTrace));
 OrderControllerV1 proxy = (OrderControllerV1)
Proxy.newProxyInstance(DynamicProxyFilterConfig.class.getClassLoader(),
 new Class[]{OrderControllerV1.class},
 new LogTraceFilterHandler(orderController, logTrace, PATTERNS)
 );
 return proxy;
 }
 @Bean
 public OrderServiceV1 orderServiceV1(LogTrace logTrace) {
 OrderServiceV1 orderService = new
OrderServiceV1Impl(orderRepositoryV1(logTrace));
 OrderServiceV1 proxy = (OrderServiceV1)
Proxy.newProxyInstance(DynamicProxyFilterConfig.class.getClassLoader(),
 new Class[]{OrderServiceV1.class},
 new LogTraceFilterHandler(orderService, logTrace, PATTERNS)
 );
 return proxy;
 }
 @Bean
 public OrderRepositoryV1 orderRepositoryV1(LogTrace logTrace) {
 OrderRepositoryV1 orderRepository = new OrderRepositoryV1Impl();
 OrderRepositoryV1 proxy = (OrderRepositoryV1)
Proxy.newProxyInstance(DynamicProxyFilterConfig.class.getClassLoader(),
 new Class[]{OrderRepositoryV1.class},
 new LogTraceFilterHandler(orderRepository, logTrace, PATTERNS)
 );
 return proxy;
 }
}
```
- `public static final String[] PATTERNS = {"request*", "order*", "save*"};
  - 적용할 패턴이다. `request`, `order`, `save`로 시작하는 메서드에 로그가 남는다.
- `LogTraceFilterHandler`: 앞서 만든 필터 기능이 있는 핸들러를 사용한다. 그리고 핸들러에 적용 패턴도 넣어준다.

**ProxyApplication - 추가**
```java
import hello.proxy.config.v2_dynamicproxy.DynamicProxyFilterConfig;
@Import(DynamicProxyFilterConfig.class)
@SpringBootApplication(scanBasePackages = "hello.proxy.app")
public class ProxyApplication {
 public static void main(String[] args) {
 SpringApplication.run(ProxyApplication.class, args);
 }
 @Bean
 public LogTrace logTrace() {
 return new ThreadLocalLogTrace();
 }
}
```
`@Import(DynamicProxyFilterConfig.class)`으로 방금 만든 설정을 추가하자.

실행 
- http://localhost:8080/v1/request?itemId=hello
- http://localhost:8080/v1/no-log

실행해보면 `no-log`가 사용하는 `noLog()` 메서드에는 로그가 남지 않는 것을 확인할 수 있다.

**JDK 동적 프록시 - 한계**  
JDK 동적 프록시는 인터페이스가 필수이다.  
그렇다면 V2 애플리케이션 처럼 인터페이스 없이 클래스만 있는 경우에는 어떻게 동적 프록시를 적용할 수 있을까?  
이것은 일반적인 방법으로는 어렵고 `CGLIB`라는 바이트코드를 조작하는 특별한 라이브러리를 사용해야 한다.

</details>

<details> <summary> 6. CGLIB - 소개 </summary>

## 6. CGLIB - 소개

**CGLIB: Code Generator Library**
- CGLIB는 바이트 코드를 조작해서 동적으로 클래스를 생성하는 기술을 제공하는 라이브러리이다.
- CGLIB를 사용하면 인터페이스가 없어도 구체 클래스만 가지고 동적 프록시를 만들어낼 수 있다.
- CGLIB는 원래는 외부 라이브러리인데, 스프링 프레임워크가 스프링 내부 소스 코드에 포함했다. 따라서 스프링을 사용한다면 별도의 외부 라이브러리를 추가하지 않아도 사용할 수 있다. 

참고로 우리가 CGLIB를 직접 사용하는 경우는 거의 없다. 이후에 설명할 스프링의 `ProxyFactory`라는 것이 이 기술을 편리하게 사용하게 도와주기 때문에, 너무 깊이있게 파기 보다는 CGLIB가 무엇인지 대략 개념만 잡으면 된다.  
예제 코드로 CGLIB를 간단히 이해해보자.

### 공통 예제 코드  
앞으로 다양한 상황을 설명하기 위해서 먼저 공통으로 사용할 예제 코드를 만들어보자.

- 인터페이스와 구현이 있는 서비스 클래스 - `ServiceInterface`, `ServiceImpl`
- 구체 클래스만 있는 서비스 클래스 - `ConcreteService`

**ServiceInterface**  
주의: 테스트 패키지에 위치한다.
```java
package hello.proxy.common.service;
public interface ServiceInterface {
 void save();
 void find();
}
```

**ServiceImpl**  
주의: 테스트 패키지에 위치한다.
```java
package hello.proxy.common.service;
import lombok.extern.slf4j.Slf4j;
@Slf4j
public class ServiceImpl implements ServiceInterface {
 @Override
 public void save() {
 log.info("save 호출");
 }
 @Override
 public void find() {
 log.info("find 호출");
 }
}
```

**ConcreteService**  
주의: 테스트 패키지에 위치한다.
```java
package hello.proxy.common.service;
import lombok.extern.slf4j.Slf4j;
@Slf4j
public class ConcreteService {
 public void call() {
 log.info("ConcreteService 호출");
 }
}
```


</details>

<details> <summary> 7. CGLIB - 예제 코드 </summary>

## 7. CGLIB - 예제 코드

### CGLIB 코드

JDK 동적 프록시에서 실행 로직을 위해 `InvocationHandler`를 제공했듯이, CGLIB는 `MethodInterceptor`를 제공한다.

**MethodInterceptor - CGLIB 제공**
```java
package org.springframework.cglib.proxy;
public interface MethodInterceptor extends Callback {
 Object intercept(Object obj, Method method, Object[] args, MethodProxy
proxy) throws Throwable;
}
```
- `obj`: CGLIB가 적용된 객체
- `method`: 호출된 메서드 
- `args`: 메서드를 호출하면서 전달된 인수
- `proxy`: 메서드 호출에 사용


**TimeMethodInterceptor**  
주의: 테스트 패키지에 위치한다.
```java
package hello.proxy.cglib.code;
import lombok.extern.slf4j.Slf4j;
import org.springframework.cglib.proxy.MethodInterceptor;
import org.springframework.cglib.proxy.MethodProxy;
import java.lang.reflect.Method;
@Slf4j
public class TimeMethodInterceptor implements MethodInterceptor {
 private final Object target;
 public TimeMethodInterceptor(Object target) {
 this.target = target;
 }
 @Override
 public Object intercept(Object obj, Method method, Object[] args,
MethodProxy proxy) throws Throwable {
 log.info("TimeProxy 실행");
 long startTime = System.currentTimeMillis();
 Object result = proxy.invoke(target, args);
 long endTime = System.currentTimeMillis();
 long resultTime = endTime - startTime;
 log.info("TimeProxy 종료 resultTime={}", resultTime);
 return result;
 }
}
```
- `TimeMethodInterceptor`는 `MethodInterceptor`인터페이스를 구현해서 CGLIB 프록시의 실행 로직을 정의한다.
- JDK 동적 프록시를 설명할 떄 예제와 거의 같은 코드이다.
- `Object target`: 프록시가 호출할 실제 대상
- `proxy.invoke(target, args)`: 실제 대상을 동적으로 호출한다.
  - 참고로 `method`를 사용해도 되지만, CGLIB는 성능상 `MethodProxy proxy`를 사용하는 것을 권장한다.

이제 테스트코드로 CGLIB를 사용해보자.

**CglibTest**
```java
package hello.proxy.cglib;
import hello.proxy.cglib.code.TimeMethodInterceptor;
import hello.proxy.common.service.ConcreteService;
import hello.proxy.common.service.ServiceImpl;
import hello.proxy.common.service.ServiceInterface;
import lombok.extern.slf4j.Slf4j;
import org.junit.jupiter.api.Test;
import org.springframework.cglib.proxy.Enhancer;
@Slf4j
public class CglibTest {
 @Test
 void cglib() {
 ConcreteService target = new ConcreteService();
 Enhancer enhancer = new Enhancer();
 enhancer.setSuperclass(ConcreteService.class);
 enhancer.setCallback(new TimeMethodInterceptor(target));
 ConcreteService proxy = (ConcreteService)enhancer.create();
 log.info("targetClass={}", target.getClass());
 log.info("proxyClass={}", proxy.getClass());
 proxy.call();
 }
}
```
`ConcreteService`는 인터페이스가 없는 구체 클래스이다. 여기에 CGLIB를 사용해서 프록시를 생성해보자.

- `Enhancer`: CGLIB는 `Enhancer`를 사용해서 프록시를 생성한다.
- `enhancer.setSuperclass(ConcreteService.class)`: CGLIB는 구체 클래스를 상속 받아서 프록시를 생성할 수 있다. 어떤 구체 클래스를 상속 받을지 지정한다.
- `enhancer.setCallback(new TimeMethodInterceptor(target))`
  - 프록시에 적용할 실행 로직을 할당한다.
- `enhancer.create()`: 프록시를 생성한다. 앞서 설정한 `enhancer.setSuperclass(ConcreteService.class)`에서 지정한 클래스를 상속 받아서 프록시가 만들어진다.

JDK 동적 프록시는 인터페이스를 구현(implement)해서 프록시를 만든다. CGLIB는 구체 클래스를 상속(extends)해서 프록시를 만든다.

**실행 결과**
```
CglibTest - targetClass=class hello.proxy.common.service.ConcreteService
CglibTest - proxyClass=class hello.proxy.common.service.ConcreteService$
$EnhancerByCGLIB$$25d6b0e3
TimeMethodInterceptor - TimeProxy 실행
ConcreteService - ConcreteService 호출
TimeMethodInterceptor - TimeProxy 종료 resultTime=9
```
실행 결과를 보면 프록시가 정상 적용된 것을 확인할 수 있다.

**CGLIB가 생성한 프록시 클래스 이름**  
CGLIB를 통해서 생성된 클래스의 이름을 확인해보자.  
`ConcreteService$$EnhancerByCGLIB$$25d6b0e3`

CGLIB가 동적으로 생성하는 클래스 이름은 다음과 같은 규칙으로 생성된다.  
`대상클래스$$EnhancerByCGLIB$$임의코드`

참고로 다음은 JDK Proxy가 생성한 클래스 이름이다.  
`proxyClass=class com.sun.proxy.$Proxy1`

### 그림으로 정리 
![image](https://user-images.githubusercontent.com/28394879/141299471-a68a844a-42e3-453f-a374-bae6a0b1c927.png)

**CGLIB 제약** 
- 클래스 기반 프록시는 상속을 사용했기 떄문에 몇가지 제약이 있다.
  - 부모 클래스의 생성자를 체크해야 한다. -> CGLIB는 자식 클래스를 동적으로 생성하기 떄문에 기본 생성자가 필요하다.
  - 클래스에 `final` 키워드가 붙으면 상속이 불가능하다. -> CGLIB에서는 예외가 발생한다.
  - 메서드에 `final` 키워드가 붙으면 해당 메서드를 오버라이딩 할 수 없다. -> CGLIB에서는 프록시 로직이 동작하지 않는다.

> **참고**  
> CGLIB를 사용하면 인터페이스가 없는 V2 애플리케이션에 동적 프록시를 적용할 수 있다.  
> 그런데 지금 당장 적용하기에는 몇가지 제약이 있다.  
> V2 애플리케이션에 기본 생성자를 추가하고, 의존관계를 `setter`를 사용해서 주입하면 CGLIB를 적용할 수 있다.  
> 하지만 다음에 학습하는 `ProxyFactory`를 통해서 CGLIB를 적용하면 이런 단점을 해결하고 또 더 편리하기 떄문에, 애플리케이션에 CGLIB로 프록시를 적용하는 것은 조금 뒤에 알아보겠다. 


</details>

<details> <summary> 8. 정리 </summary>

## 8. 정리

**남은 문제**
- 인터페이스가 있는 경우에는 JDK 동적 프록시를 적용하고, 그렇지 않은 경우에는 CGLIB를 적용하려면 어떻게 해야 할까?
- 두 기술을 함께 사용할 때 부가 기능을 제공하기 위해서 JDK 동적 프록시가 제공하는 `InvocationHandler`와 CGLIB가 제공하는 `MethodInterceptor`를 각각 중복으로 만들어서 관리해야 할까? 
- 특정 조건에 맞을 때 프록시 로직을 적용하는 기능도 공통으로 제공되었으면? 

</details>

# [6. 스프링이 지원하는 프록시](./6.proxy-supported-by-spring)

<details> <summary> 1. 프록시 팩토리 - 소개 </summary>

## 1. 프록시 팩토리 - 소개

앞서 마지막에 설명했던 동적 프록시를 사용할 때 문제점을 다시 확인해보자.

**문제점**
- 인터페이스가 있는 경우에는 JDK 동적 프록시를 적용하고, 그렇지 않은 경우에는 CGLIB를 적용하려면 어떻게 해야할까?
- 두 기술을 함께 사용할 때 부가 기능을 제공하기 위해 JDK 동적 프록시가 제공하는 `InvocationHandler`와 CGLIB가 제공하는 `MethodInterceptor`를 각각 중복으로 만들어서 관리해야 할까?
- 특정 조건에 맞을 때 프록시 로직을 적용하는 기능도 공통으로 제공되었으면?


**Q: 인터페이스가 있는 경우에는 JDK 동적 프록시를 적용하고, 그렇지 않은 경우에는 CGLIB를 적용하려면 어떻게 해야할까?**  
스프링은 유사한 구체적인 기술들이 있을 때, 그것들을 통합해서 일관성 있게 접근할 수 있고, 더욱 편리하게 사용할 수 있는 추상화된 기술을 제공한다.  
스프링은 동적 프록시를 통합해서 편리하게 만들어주는 프록시 팩토리(`ProxyFactory`)라는 기능을 제공한다.  
이전에는 상황에 따라서 JDK 동적 프록시를 사용하거나 CGLIB를 사용해야 했다면, 이제는 이 프록시 팩토리 하나로 편리하게 동적 프록시를 생성할 수 있다.  
프록시 팩토리는 인터페이스가 있으면 JDK 동적 프록시를 사용하고, 구체 클래스만 있다면 CGLIB를 사용한다.  
그리고 이 설정을 변경할 수도 있다. 

**프록시 팩토리**  
![image](https://user-images.githubusercontent.com/28394879/141407992-a901db58-ebd3-478a-b083-2e3433980046.png)


**Q: 두 기술을 함께 사용할 때 부가 기능을 적용하기 위해 JDK 동적 프록시가 제공하는 InvocationHandler와 CGLIB가 제공하는 MethodInterceptor를 각각 중복으로 만들어야 할까?**  
스프링은 이 문제를 해결하기 위해 부가 기능을 적용할 때 `Advice`라는 새로운 개념을 도입했다.  
개발자는 `InvocationHandler`나 `MethodInterceptor`를 신경쓰지 않고, `Advice`만 만들면 된다.  
결과적으로 `InvocationHandler`나 `MethodInterceptor`는 `Advice`를 호출하게 된다.  
프록시 팩토리를 사용하면 `Advice`를 호출하는 전용 `InvocationHandler`, `MethodInterceptor`를 내부에서 사용한다.

![image](https://user-images.githubusercontent.com/28394879/141408183-9d4a17b6-64fd-4303-b727-bcc12a3aad91.png)


**Q: 특정 조건에 맞을 때 프록시 로직을 적용하는 기능도 공통으로 제공되었으면?**  
앞서 특정 메서드 이름의 조건에 맞을 때만 프록시 부가 기능이 적용되는 코드를 직접 만들었다.  
스프링은 `Pointcut`이라는 개념을 도입해서 이 문제를 일관성 있게 해결한다. 

</details>

<details> <summary> 2. 프록시 팩토리 - 예제 코드1 </summary>

## 2. 프록시 팩토리 - 예제 코드1

**Advice 만들기**  
`Advice`는 프록시에 적용하는 부가 기능 로직이다. 이것은 JDK 동적 프록시가 제공하는  
`InvocationHandler`와 CGLIB가 제공하는 `MethodInterceptor`의 개념과 유사하다.   
둘을 개념적으로 추상화 한 것이다. 프록시 팩토리를 사용하면 둘 대신에 `Advice`를 사용하면 된다.  

`Advice`를 만드는 방법은 여러가지가 있지만, 기본적으로 방법은 다음 인터페이스를 구현하면 된다.

**MethodInterceptor - 스프링에 제공하는 코드** 
```java
package org.aopalliance.intercept;
public interface MethodInterceptor extends Interceptor {
 Object invoke(MethodInvocation invocation) throws Throwable;
}
```
- `MethodInvocation invocation`
  - 내부에서 다음 메서드를 호출하는 방법, 현재 프록시 객체 인스턴스, `args`,  메서드 정보 등이 포함되어 있다. 기존에는 파라미터로 제공되는 부분들이 이 안으로 모두 들어갔다고 생각하면 된다.
- CGLIB의 `MethodInterceptor`와 이름이 같으므로 패키지 이름에 주의하자
  - 참고로 여기서 사용하는 `org.aopappliance.intercept` 패키지는 스프링의 AOP 모듈(`spring-top`) 안에 들어있따.
- `MethodInterceptor`는 `Interceptor`를 상속하고 `Interceptor`는 `Advice` 인터페이스를 상속한다.

이제 실제 `Advice`를 만들어보자.

**TimeAdvice**  
주의: 테스트 패키지에 위치한다.
```java
package hello.proxy.common.advice;
import lombok.extern.slf4j.Slf4j;
import org.aopalliance.intercept.MethodInterceptor;
import org.aopalliance.intercept.MethodInvocation;
@Slf4j
public class TimeAdvice implements MethodInterceptor {
 @Override
 public Object invoke(MethodInvocation invocation) throws Throwable {
 log.info("TimeProxy 실행");
 long startTime = System.currentTimeMillis();
 Object result = invocation.proceed();
 long endTime = System.currentTimeMillis();
 long resultTime = endTime - startTime;
 log.info("TimeProxy 종료 resultTime={}ms", resultTime);
 return result;
 }
}
```
- `TimeAdvice`는 앞서 설명한 `MethodInterceptor` 인터페이스를 구현한다. 패키지 이름에 주의하자.  
- `Object result = invocation.proceed()`
  - `invocation.proceed()`를 호출하면 `target` 클래스를 호출하고 그 결과를 받는다.
  - 그런데 기존에 보았던 코드들과 다르게 `target` 클래스의 정보가 보이지 않는다. `target`클래스의 정보는 `MethodInvocation invocation`안에 모두 포함되어 있다.
  - 그 이유는 바로 다음에 확인할 수 있는데, 프록시 팩토리로 프록시를 생성하는 단계에서 이미 `target`정보를 파라미터로 전달받기 떄문이다.

**ProxyFactoryTest**
```java
package hello.proxy.proxyfactory;
import hello.proxy.common.advice.TimeAdvice;
import hello.proxy.common.service.ConcreteService;
import hello.proxy.common.service.ServiceImpl;
import hello.proxy.common.service.ServiceInterface;
import lombok.extern.slf4j.Slf4j;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;
import org.springframework.aop.framework.ProxyFactory;
import org.springframework.aop.support.AopUtils;
import static org.assertj.core.api.Assertions.assertThat;
@Slf4j
public class ProxyFactoryTest {
 @Test
 @DisplayName("인터페이스가 있으면 JDK 동적 프록시 사용")
 void interfaceProxy() {
 ServiceInterface target = new ServiceImpl();
 ProxyFactory proxyFactory = new ProxyFactory(target);
 proxyFactory.addAdvice(new TimeAdvice());
 ServiceInterface proxy = (ServiceInterface) proxyFactory.getProxy();
 log.info("targetClass={}", target.getClass());
 log.info("proxyClass={}", proxy.getClass());
 proxy.save();
 assertThat(AopUtils.isAopProxy(proxy)).isTrue();
 assertThat(AopUtils.isJdkDynamicProxy(proxy)).isTrue();
 assertThat(AopUtils.isCglibProxy(proxy)).isFalse();
 }
}
```

- `new ProxyFactory(target)`: 프록시 팩토리를 생성할 때, 생성자에 프록시의 호출 대상을 함께 넘겨준다. 프록시 팩토리는 이 인스턴스 정보를 기반으로 프록시를 만들어낸다. 만약 이 인스턴스에 인터페이스가 있다면 JDK 동적 프록시를 기본으로 사용하고 인터페이스가 없고 구체 클래스만 있다면 CGLIB를 통해서 동적 프록시를 생성한다. 여기서는 `target`이 `new ServiceImpl()`의 인스턴스이기 떄문에 `ServiceInterface` 인터페이스가 있다. 따라서 이 인터페이스를 기반으로 JDK 동적 프록시를 생성한다.
- `proxyFactory.addAdvice(new TimeAdvice())`: 프록시 팩토리를 통해서 만든 프록시가 사용할 부가 기능 로직을 설정한다. JDK 동적 프록시가 제공하는 `InvocationHandler`와 CGLIB가 제공하는 `MethodInterceptor`의 개념과 유사하다. 이렇게 프록시가 제공하는 부가 기능 로직을 어드바이스 (`Advice`)라 한다. 번역하면 조언을 해준다고 생각하면 된다.
- `proxyFactory.getProxy()`: 프록시 객체를 생성하고 그 결과를 받는다.

**실행 결과**
```
ProxyFactoryTest - targetClass=class hello.proxy.common.service.ServiceImpl
ProxyFactoryTest - proxyClass=class com.sun.proxy.$Proxy13
TimeAdvice - TimeProxy 실행
ServiceImpl - save 호출
TimeAdvice - TimeProxy 종료 resultTime=1ms
```
실행 결과를 보면 프록시가 정상 적용된 것을 확인할 수 있다. `proxyClass=class com.sun.proxy.$Proxy13` 코드를 통해 JDK 동적 프록시가 적용된 것도 확인할 수 있다.

**프록시 팩토리를 통한 프록시 적용 확인**  
프록시 팩토리로 프록시가 잘 적용되었는지 확인하려면 다음 기능을 사용하면 된다.
- `AopUtils.isAopProxy(proxy)`: 프록시 팩토리를 통해서 프록시가 생성되면 JDK 동적 프록시나, CGLIB 모두 참이다.
- `AopUtils.isJdkDynamicProxy(proxy)`: 프록시 팩토리를 통해서 프록시가 생성되고, JDK 동적 프록시인 경우 참
- `AopUtils.isCglibProxy(proxy)`: 프록시 팩토리를 통해서 프록시가 생성되고, CGLIB 동적 프록시인 경우 참

물론 `proxy.getClass()`처럼 인스턴스의 클래스 정보를 직접 출력해서 확인할 수 있다. 



</details>


<details> <summary> 3. 프록시 팩토리 - 예제 코드2 </summary>

## 3. 프록시 팩토리 - 예제 코드2

**ProxyFactoryTest - concreteProxy 추가**
```java
@Test
@DisplayName("구체 클래스만 있으면 CGLIB 사용")
void concreteProxy() {
 ConcreteService target = new ConcreteService();
 ProxyFactory proxyFactory = new ProxyFactory(target);
 proxyFactory.addAdvice(new TimeAdvice());
 ConcreteService proxy = (ConcreteService) proxyFactory.getProxy();
 log.info("targetClass={}", target.getClass());
 log.info("proxyClass={}", proxy.getClass());
 proxy.call();
 assertThat(AopUtils.isAopProxy(proxy)).isTrue();
 assertThat(AopUtils.isJdkDynamicProxy(proxy)).isFalse();
 assertThat(AopUtils.isCglibProxy(proxy)).isTrue();
}
```
이번에는 구체 클래스만 있는 `ConcreteService`에 프록시를 적용해보자.  
프록시 팩토리는 인터페이스 없이 구체 클래스만 있으면 CGLIB를 사용해서 프록시를 적용한다.  
나머지 코드는 기존과 같다.

**실행 결과** 
```
ProxyFactoryTest - targetClass=class hello.proxy.common.service.ConcreteService
ProxyFactoryTest - proxyClass=class hello.proxy.common.service.ConcreteService$
$EnhancerBySpringCGLIB$$103821ba
TimeAdvice - TimeProxy 실행
ConcreteService - ConcreteService 호출
TimeAdvice - TimeProxy 종료 resultTime=1ms
```
실행 결과를 보면 프록시가 적용된 것을 확인할 수 있다.   
`proxyClass=class..ConcreteSErvice$EnhancerBySpringCGLIB$$103821ba` 코드를 통해 CGLIB프록시가 적용된 것도 확인할 수 있다.

**proxyTargetClass 옵션**

**ProxyFactoryTest - proxyTargetClass 추가**
```java
@Test
@DisplayName("ProxyTargetClass 옵션을 사용하면 인터페이스가 있어도 CGLIB를 사용하고, 클래스
기반 프록시 사용")
void proxyTargetClass() {
 ServiceInterface target = new ServiceImpl();
 ProxyFactory proxyFactory = new ProxyFactory(target);
 proxyFactory.setProxyTargetClass(true); //중요
 proxyFactory.addAdvice(new TimeAdvice());
 ServiceInterface proxy = (ServiceInterface) proxyFactory.getProxy();
 log.info("targetClass={}", target.getClass());
 log.info("proxyClass={}", proxy.getClass());
 proxy.save();
 assertThat(AopUtils.isAopProxy(proxy)).isTrue();
 assertThat(AopUtils.isJdkDynamicProxy(proxy)).isFalse();
 assertThat(AopUtils.isCglibProxy(proxy)).isTrue();
}
```
마지막으로 인터페이스가 있지만, CGLIB를 사용해서 인터페이스가 아닌 클래스 기반으로 동적 프록시를 만드는 방법을 알아보자.  
프록시 팩토리는 `proxyTargetClass`라는 옵션을 제공하는데, 이 옵션에 `true`값을 넣으면 인터페이스가 있어도 강제로 CGLIB를 사용한다.  
그리고 인터페이스가 아닌 클래스 기반으로 프록시를 만들어준다.

**실행 결과**
```
ProxyFactoryTest - targetClass=class hello.proxy.common.service.ServiceImpl
ProxyFactoryTest - proxyClass=class hello.proxy.common.service.ServiceImpl$
$EnhancerBySpringCGLIB$$2bbf51ab
TimeAdvice - TimeProxy 실행
ServiceImpl - save 호출
TimeAdvice - TimeProxy 종료 resultTime=1ms
```
`ServiceImpl$$EnhancerBySpringCGLIB...`를 보면 CGLIB 기반의 프록시가 생성된 것을 확인할 수 있다.  
인터페이스가 있지만 `proxyTargetClass`옵션에 의해 CGLIB가 사용된다.

**프록시 팩토리의 기술 선택 방법**
- 대상에 인터페이스가 있으면: JDK 동적 프록시, 인터페이스 기반 프록시
- 대상에 인터페이스가 없으면: CGLIB, 구체 클래스 기반 프록시
- `proxyTargetClass=true`: CGLIB, 구체 클래스 기반 프록시, 인터페이스 여부와 상관 없음

**정리** 
- 프록시 팩토리의 서비스 추상화 덕분에 구체적인 CGLIB, JDK 동적 프록시 기술에 의존하지 않고, 매우 편리하게 동적 프록시를 생성할 수 있따.
- 프록시의 부가 기능 로직도 특정 기술에 종속적이지 않게 `Advice` 하나로 편리하게 사용할 수 있었다. 이것은 프록시 팩토리가 내부에서 JDK 동적 프록시인 경우 `InvocationHandler`가 `Advice`를 호출하도록 개발해두고, CGLIB인 경우 `MethodInterceptor`가 `Advice`를 호출하도록 기능을 개발해두었기 때문이다.


> 참고  
> 스프링 부트는 AOP를 적용할 때 기본적으로 `proxyTargetClass=true`로 설정해서 사용한다.  
> 따라서 인터페이스가 있어도 항상 CGLIB를 사용해서 구체 클래스를 기반으로 프록시를 생성한다.   
> 자세한 이유는 뒷부분에서 설명한다.



</details>


<details> <summary> 4. 포인트컷, 어드바이스, 어드바이저 - 소개 </summary>

## 4. 포인트컷, 어드바이스, 어드바이저 - 소개

스프링 AOP를 공부했다면 다음과 같은 단어를 들어볼았을 것이다.   
항상 잘 정리가 안되는 단어들인데, 단순하지만 중요하니 이번에 확실히 정리하자.

- 포인트컷(`Pointcut`): 어디에 부가 기능을 적용할지, 어디에 부가 기능을 적용하지 않을지 판단하는 필터링 로직이다. 주로 클래스와 메서드 이름으로 필터링 한다. 이름 그대로 어떤 포인트(Point)에 기능을 적용할지 하지 않을지 잘라서(cut) 구분하는 것이다.
- 어드바이스(`Advice`): 이전에 본 것 처럼 프록시가 호출하는 부가 기능이다. 단순하게 프록시 로직이라 생각하면 된다.
- 어드바이저(`Advisor`): 단순하게 하나의 포인트컷과 하나의 어드바이스를 가지고 있는 것이다. 쉽게 이야기해서 **포인트컷1 + 어드바이스1**이다.

정리하면 부가 기능 로직을 적용해야 하는데, 포인트컷으로 어디에? 적용할지 선택하고, 어드바이스로 어떤 로직을 적용할지 선택하는 것이다.  
그리고 어디에? 어떤 로직을? 모두 알고 있는 것이 **어드바이저**이다.

**쉽게 기억하기**
- 조언(`Advice`)을 어디(`Pointcut`)에 할 것인가?
- 조언자(`Advisor`)는 어디(`Pointcut`)에 조언(`Advice`)을 해야할지 알고 있다.

**역할과 책임**  
이렇게 구분한 것은 역할과 책임을 명확하게 분리한 것이다.
- 포인트컷은 대상 여부를 확인하는 필터 역할만 담당한다.
- 어드바이스는 깔끔하게 부가 기능 로직만 담당한다.
- 둘을 합치면 어드바이저가 된다. 스프링의 어드바이저는 하나의 포인트컷 + 하나의 어드바이스로 구성 된다.


> 참고  
> 해당 단어들에 대한 정의는 지금은 문맥상 이해를 돕기 위해 프록시에 맞추어서 설명하지만, 이후에 AOP  
> 부분에서 다시 한번 AOP에 맞추어 정리하자. 그림은 이해를 돕기 위한 것이고, 실제 구현은 약간 다를 수 있다.

![image](https://user-images.githubusercontent.com/28394879/141463206-19a75ac3-6ad2-4c86-b6c2-f819e5408827.png)



</details>


<details> <summary> 5. 예제 코드1 - 어드바이저 </summary>

## 5. 예제 코드1 - 어드바이저

어드바이저는 하나의 포인트컷과 하나의 어드바이스를 가지고 있다.  
프록시 팩토리를 통해 프록시를 생성할 때 어드바이저를 제공하면 어디에 어떤 기능을 제공할 지 알 수 있다. 

**AdvisorTest**
```java
package hello.proxy.advisor;
import hello.proxy.common.service.ServiceImpl;
import hello.proxy.common.service.ServiceInterface;
import hello.proxy.common.advice.TimeAdvice;
import lombok.extern.slf4j.Slf4j;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;
import org.springframework.aop.ClassFilter;
import org.springframework.aop.MethodMatcher;
import org.springframework.aop.Pointcut;
import org.springframework.aop.framework.ProxyFactory;
import org.springframework.aop.support.DefaultPointcutAdvisor;
import org.springframework.aop.support.NameMatchMethodPointcut;
import java.lang.reflect.Method;
@Slf4j
public class AdvisorTest {
 @Test
 void advisorTest1() {
 ServiceInterface target = new ServiceImpl();
 ProxyFactory proxyFactory = new ProxyFactory(target);
 DefaultPointcutAdvisor advisor = new
DefaultPointcutAdvisor(Pointcut.TRUE, new TimeAdvice());
 proxyFactory.addAdvisor(advisor);
 ServiceInterface proxy = (ServiceInterface) proxyFactory.getProxy();
 proxy.save();
 proxy.find();
 }
}
```
- `new DefaultPointcutAdvisor`: `Advisor` 인터페이스의 가장 일반적인 구현체이다. 생서앚를 통해 하나의 포인트컷과 하나의 어드바이스를 넣어주면 된다. 어드바이저는 하나의 포인트컷과 하나의 어드바이스로 구성된다.
- `Pointcut.TRUE`: 항상 `true`를 반환하는 포인트컷이다. 이후에 직접 포인트컷을 구현해보자.
- `new TimeAdvice()`: 앞서 개발한 `TimeAdvice` 어드바이스를 제공한다.
- `proxyFactory.addAdvisor(advisor)`: 프록시 팩토리에 적용할 어드바이저를 지정한다. 어드바이저는 내부에 포인트컷과 어디바이스를 모두 가지고 있다. 따라서 어디에 어떤 부가 기능을 적용해야 할지 어드바이스 하나로 알 수 있따. 프록시 팩토리를 사용할 때 어드바이저는 필수이다.
- 그런데 생각해보면 이전에 분명히 `proxyFactory.addAdvice(new TimeAdvice())` 이렇게 어드바이저가 아니라 어드바이스를 바로 적용했다. 이것은 단순히 편의 메서드이고 결과적으로 해당 메서드 내부에서 지금 코드과 똑같은 다음 어드바이저가 생성된다.   `DefaultPointcutAdvisor(Pointcut.TRUE, new TimeAdvice())`

![image](https://user-images.githubusercontent.com/28394879/141466112-e094905a-462f-4135-94b6-a20e6bad6cae.png)

**실행 결과**
```
#save() 호출
TimeAdvice - TimeProxy 실행
ServiceImpl - save 호출
TimeAdvice - TimeProxy 종료 resultTime=0ms
#find() 호출
TimeAdvice - TimeProxy 실행
ServiceImpl - find 호출
TimeAdvice - TimeProxy 종료 resultTime=1ms
```
실행 결과를 보면 `save()`, `find()` 각각 모두 어드바이스가 적용된 것을 확인할 수 있다. 


</details>


<details> <summary> 6. 예제 코드2 - 직접 만든 포인트컷 </summary>

## 6. 예제 코드2 - 직접 만든 포인트컷

이번에는 `save()` 메서드에는 어드바이스 로직을 적용하지만, `find()` 메서드에는 어드바이스 로직을 적용하지 않도록 해보자.  
물론 과거에 했던 코드와 유사하게 어드바이저에 로직을 추가해서 메서드 이름을 보고 코드를 실행할지 말지 분기를 타도 된다.  
하지만 이런 기능에 특화되어서 제공되는 것이 바로 포인트컷이다.

이번에는 해당 요구사항을 만족하도록 포인트컷을 직접 구현해보자.

**Pointcut 관련 인터페이스 - 스프링 제공**  
```java
public interface Pointcut {
 ClassFilter getClassFilter();
 MethodMatcher getMethodMatcher();
}
public interface ClassFilter {
 boolean matches(Class<?> clazz);
}
public interface MethodMatcher {
 boolean matches(Method method, Class<?> targetClass);
 //..
}
```
포인트컷은 크게 `ClassFilter`와 `MethodMatcher` 둘로 이루어진다.  
이름 그대로 하나는 클래스가 맞는지, 하나는 메서드가 맞는지 확인할 때 사용한다.  
둘다 `true`로 반환해야 어드바이스를 적용할 수 있다.

일반적으로 스프링이 이미 만들어둔 구현체를 사용하지만 개념 학습 차원에서 간단히 직접 구현해보자.

**AdvisorTest - advisorTest2() 추가**
```java
@Test
@DisplayName("직접 만든 포인트컷")
void advisorTest2() {
 ServiceImpl target = new ServiceImpl();
 ProxyFactory proxyFactory = new ProxyFactory(target);
 DefaultPointcutAdvisor advisor = new DefaultPointcutAdvisor(new
MyPointcut(), new TimeAdvice());
 proxyFactory.addAdvisor(advisor);
 ServiceInterface proxy = (ServiceInterface) proxyFactory.getProxy();
 proxy.save();
 proxy.find();
}
static class MyPointcut implements Pointcut {
 @Override
 public ClassFilter getClassFilter() {
 return ClassFilter.TRUE;
 }
 @Override
 public MethodMatcher getMethodMatcher() {
 return new MyMethodMatcher();
 }
}
static class MyMethodMatcher implements MethodMatcher {
 private String matchName = "save";
 @Override
 public boolean matches(Method method, Class<?> targetClass) {
 boolean result = method.getName().equals(matchName);
 log.info("포인트컷 호출 method={} targetClass={}", method.getName(),
targetClass);
 log.info("포인트컷 결과 result={}", result);
 return result;
 }
 @Override
 public boolean isRuntime() {
 return false;
 }
 @Override
 public boolean matches(Method method, Class<?> targetClass, Object... args)
{
 throw new UnsupportedOperationException();
 }
}
```

**MyPointcut**
- 직접 구현한 포인트컷이다. `Pointcut` 인터페이스를 구현한다.
- 현재 메서드 기준으로 로직을 적용하면 된다. 클래스 필터는 항상 `true`를 반환하도록 했고, 메서드 비교 기능은 `MyMethodMatcher`를 사용한다.

**MyMethodMatcher**
- 직접 구현한 `MethodMatcher`이다. `MethodMatcher` 인터페이스를 구현한다.
- `matches()`: 이 메서드에 `method`, `targetClass` 정보가 넘어온다. 이 정보로 어드바이스를 적용할지 적용하지 않을지 판단할 수 있다.
- 여기서는 메서드 이름이 `"save"`인 경우에 `true`를 반환하도록 판단 로직을 적용했다.
- `isRuntime()`, `matches(... args)`: `isRuntime()`이 값이 참이면 `matches(... args)` 메서드가 대신 호출된다. 동적으로 넘어오는 매개변수를 판단 로직으로 사용할 수 있다.
  - `isRuntime()`이 `false`인 경우 클래스의 정적 정보만 사용하기 떄문에 스프링이 내부에서 캐싱을 통해 성능 향상이 가능하지만, `isRuntime()`이 `true`인 경우 매개변수가 동적으로 변경된다고 가정하기 떄문에 캐싱을 하지 않는다.
  - 크게 중요한 부분은 아니니 참고만 하고 넘어가자.

**new DefaultPointcutAdvisor(new MyPointcut(), new TimeAdvice())**
- 어드바이저에 직접 구현한 포인트컷을 사용한다.


**실행 결과**
```
#save() 호출
AdvisorTest - 포인트컷 호출 method=save targetClass=class
hello.proxy.common.service.ServiceImpl
AdvisorTest - 포인트컷 결과 result=true
TimeAdvice - TimeProxy 실행
ServiceImpl - save 호출
TimeAdvice - TimeProxy 종료 resultTime=1ms
#find() 호출
AdvisorTest - 포인트컷 호출 method=find targetClass=class
hello.proxy.common.service.ServiceImpl
AdvisorTest - 포인트컷 결과 result=false
ServiceImpl - find 호출
```
실행 결과를 보면 기대한 것과 같이 `save()`를 호출할 때는 어드바이스가 적용되지만, `find()`를 호출할 때는 어드바이스가 적용되지 않는다.

**그림으로 정리**  
**save() 호출**  
![image](https://user-images.githubusercontent.com/28394879/141468804-b79e716d-8613-4e2f-a701-e411fbdd9dab.png)
1. 클라이언트가 프록시의 `save()`를 호출한다.
2. 포인트컷에 `Service` 클래스의 `save()` 메서드에 어드바이스를 적용해도 될지 물어본다.
3. 포인트컷이 `true`를 반환한다. 따라서 어드바이스를 호출해서 부가 기능을 적용한다.
4. 이후 실제 인스턴스의 `save()`를 호출한다.

**find() 호출**  
![image](https://user-images.githubusercontent.com/28394879/141468934-0234a342-ebeb-4699-81ad-9d636703bcdf.png)  
1. 클라이언트가 프록시의 `find()`를 호출한다.
2. 포인트컷에 `Service` 클래스의 `find()`메서드에 어드바이스를 적용해도 될지 물어본다.
3. 포인트컷이 `false`를 반환한다. 따라서 어드바이스를 호출하지 않고, 부가 기능도 적용되지 않는다.
4. 실제 인스턴스를 호출한다.
  




</details>


<details> <summary> 7. 예제 코드3 - 스프링이 제공하는 포인트컷 </summary>

## 7. 예제 코드3 - 스프링이 제공하는 포인트컷

스프링은 우리가 필요한 포인트컷을 이미 대부분 제공한다.  
이번에는 스프링이 제공하는 `NameMatchMethodPointcut`를 사용해서 구현해보자.

**AdvisorTest - advisorTest3()추가**
```java
@Test
@DisplayName("스프링이 제공하는 포인트컷")
void advisorTest3() {
 ServiceImpl target = new ServiceImpl();
 ProxyFactory proxyFactory = new ProxyFactory(target);
 NameMatchMethodPointcut pointcut = new NameMatchMethodPointcut();
 pointcut.setMappedNames("save");
 DefaultPointcutAdvisor advisor = new DefaultPointcutAdvisor(pointcut, new
TimeAdvice());
 proxyFactory.addAdvisor(advisor);
 ServiceInterface proxy = (ServiceInterface) proxyFactory.getProxy();
 proxy.save();
 proxy.find();
}
```

**NameMatchMethodPointcut 사용 코드**
```java
NameMatchMethodPointcut pointcut = new NameMatchMethodPointcut();
pointcut.setMappedNames("save");
```
`NameMatchMethodPointcut`을 생성하고 `setMappedNames(...)`으로 메서드 이름을 지정하면 포인트컷이 완성된다.

**실행 결과**
```
#save() 호출
TimeAdvice - TimeProxy 실행
ServiceImpl - save 호출
TimeAdvice - TimeProxy 종료 resultTime=1ms
#find() 호출
ServiceImpl - find 호출
```

실행 결과를 보면 `save()`를 호출할 때는 어드바이스가 적용되지만, `find()`를 호출할 때는 어드바이스가 적용되지 않는다.

### 스프링이 제공하는 포인트컷
스프링은 무수히 많은 포인트컷을 제공한다.  
대표적인 몇가지만 알아보자.

- `NameMatchMethodPointcut`: 메서드 이름을 기반으로 매칭한다. 내부에서는 `PatternMatchUtils`를 사용한다.
  - 예) `*xxx*`허용
- `JdkRegexpMethodPointcut`: JDK 정규 표현식을 기반으로 포인트컷을 매칭한다.
- `TruePointcut`: 항상 참을 반환한다.
- `AnnotationMatchingPointcut`: 애노테이션으로 매칭한다.
- `AspectJExpressionPointcut`: aspectJ 표현식으로 매칭한다.

**가장 중요한 것은 aspectJ 표현식**  
여기에서 사실 다른 것은 중요하지 않다. 실무에서는 사용하기도 편리하고 기능도 가장 많은 aspectJ 표현식을 기반으로 사용하는 `AspectJExpressionPointcut`을 사용하게 된다.  
aspectJ 표현식과 사용방법은 중요해서 이후 AOP를 설명할 때 자세히 정리한다.  
지금은 `Pointcut`의 동작 방식과 전체 구조에 집중하자. 

</details>


<details> <summary> 8. 예제 코드4 - 여러 어드바이저 함께 적용 </summary>

## 8. 예제 코드4 - 여러 어드바이저 함께 적용 

어드바이저는 하나의 포인트컷과 하나의 어드바이스를 가지고 있다.  
만약 여러 어드바이저를 하나의 `target`에 적용하려면 어떻게 해야할까?  
쉽게 이야기해서 하나의 `target`에 여러 어드바이스를 적용하려면 어떻게 해야할까?

지금 떠오르는 방법은 프록시를 여러개 만들면 될 것 같다.

### 여러 프록시

**MultiAdvisorTest**
```java
package hello.proxy.advisor;
import hello.proxy.common.advice.TimeAdvice;
import hello.proxy.common.service.ServiceImpl;
import hello.proxy.common.service.ServiceInterface;
import lombok.extern.slf4j.Slf4j;
import org.aopalliance.intercept.MethodInterceptor;
import org.aopalliance.intercept.MethodInvocation;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;
import org.springframework.aop.Pointcut;
import org.springframework.aop.framework.ProxyFactory;
import org.springframework.aop.support.DefaultPointcutAdvisor;
public class MultiAdvisorTest {
 @Test
 @DisplayName("여러 프록시")
 void multiAdvisorTest1() {
 //client -> proxy2(advisor2) -> proxy1(advisor1) -> target
 //프록시1 생성
 ServiceInterface target = new ServiceImpl();
 ProxyFactory proxyFactory1 = new ProxyFactory(target);
 DefaultPointcutAdvisor advisor1 = new
DefaultPointcutAdvisor(Pointcut.TRUE, new Advice1());
 proxyFactory1.addAdvisor(advisor1);
 ServiceInterface proxy1 = (ServiceInterface) proxyFactory1.getProxy();
 //프록시2 생성, target -> proxy1 입력
 ProxyFactory proxyFactory2 = new ProxyFactory(proxy1);
 DefaultPointcutAdvisor advisor2 = new
DefaultPointcutAdvisor(Pointcut.TRUE, new Advice2());
 proxyFactory2.addAdvisor(advisor2);
 ServiceInterface proxy2 = (ServiceInterface) proxyFactory2.getProxy();
 //실행
 proxy2.save();
 }
 @Slf4j
 static class Advice1 implements MethodInterceptor {
 @Override
 public Object invoke(MethodInvocation invocation) throws Throwable {
 log.info("advice1 호출");
 return invocation.proceed();
 }
 }
 @Slf4j
 static class Advice2 implements MethodInterceptor {
 @Override
 public Object invoke(MethodInvocation invocation) throws Throwable {
 log.info("advice2 호출");
 return invocation.proceed();
 }
 }
}
```

이 코드는 런타임에 다음과 같이 동작한다.

![image](https://user-images.githubusercontent.com/28394879/141471330-5d8588ef-acc8-48aa-9999-59310e2a2dc3.png)

**실행 결과**
```
MultiAdvisorTest$Advice2 - advice2 호출
MultiAdvisorTest$Advice1 - advice1 호출
ServiceImpl - save 호출
```
포인트컷은 `advisor1`, `advisor2` 모두 항상 `true`를 반환하도록 설정했다. 따라서 둘다 어드바이스가 적용된다.

**여러 프록시의 문제**  
이 방법이 잘못된 것은 아니지만, 프록시를 2번 생성해야 한다는 문제가 있다. 만약 적용해야 하는 어드바이저가 10개라면 10개의 프록시를 생성해야 한다.

### 하나의 프록시, 여러 어드바이저  
스프링은 이 문제를 해결하기 위해 하나의 프록시에 여러 어드바이저를 적용할 수 있게 만들어두었다.

![image](https://user-images.githubusercontent.com/28394879/141471512-074edb33-51f6-47f8-be5b-5402f8aef8fe.png)

**MultiAdvisorTest - multiAdvisorTest2() 추가**
```java
@Test
@DisplayName("하나의 프록시, 여러 어드바이저")
void multiAdvisorTest2() {
 //proxy -> advisor2 -> advisor1 -> target
 DefaultPointcutAdvisor advisor2 = new DefaultPointcutAdvisor(Pointcut.TRUE,
new Advice2());
 DefaultPointcutAdvisor advisor1 = new DefaultPointcutAdvisor(Pointcut.TRUE,
new Advice1());
 ServiceInterface target = new ServiceImpl();
 ProxyFactory proxyFactory1 = new ProxyFactory(target);
 proxyFactory1.addAdvisor(advisor2);
 proxyFactory1.addAdvisor(advisor1);
 ServiceInterface proxy = (ServiceInterface) proxyFactory1.getProxy();
 //실행
 proxy.save();
}
```
- 프록시 팩토리에 원하는 만큼 `addAdvisor()`를 통해서 어드바이저를 등록하면 된다.
- 등록하는 순서대로 `advisor`가 호출된다. 여기서는 `advisor2`,`advisor1` 순서로 등록했다.

![image](https://user-images.githubusercontent.com/28394879/141471634-65b350ab-d833-4f72-8ef8-5f783ec50c4d.png)

**실행 결과**
```
MultiAdvisorTest$Advice2 - advice2 호출
MultiAdvisorTest$Advice1 - advice1 호출
ServiceImpl - save 호출
```

실행 결과를 보면 `advice2`, `advice1` 순서대로 호출된 것을 알 수 있다.

**정리**  
결과적으로 여러 프록시를 사용할 때와 비교해서 결과는 같고, 성능은 더 좋다.

> 중요  
> 사실 이번 장을 이렇게 풀어서 설명한 이유가 있다. 스프링의 AOP를 처음 공부하거나 사용하면, AOP 적용 수 만큼 프록시가 생성된다고 착각하게 된다.  
> 실제 많은 실무 개발자들도 이렇게 생각하는 경우가 많다.  
> 스프링은 AOP를 적용할 때, 최적화를 진행해서 지금처럼 프록시는 하나만 만들고, 하나의 프록시에 여러 어드바이저를 적용한다.  
> 정리하면 하나의 `target`에 여러 AOP가 동시에 적용되어도, 스프링의 AOP는 `target`마다 하나의 프록시만 생성한다.  
> 이부분을 꼭 기억해두자. 


</details>


<details> <summary> 9. 프록시 팩토리 - 적용1 </summary>

## 9. 프록시 팩토리 - 적용1

지금까지 학습한 프록시 팩토리를 사용해서 애플리케이션에 프록시를 만들어보자.  
먼저 인터페이스가 있는 v1 애플리케이션에 `LogTrace` 기능을 프록시 팩토리를 통해서 프록시를 만들어 적용해보자.

먼저 어드바이스를 만들자.

**LogTraceAdvice**
```java
package hello.proxy.config.v3_proxyfactory.advice;
import hello.proxy.trace.TraceStatus;
import hello.proxy.trace.logtrace.LogTrace;
import lombok.extern.slf4j.Slf4j;
import org.aopalliance.intercept.MethodInterceptor;
import org.aopalliance.intercept.MethodInvocation;
import java.lang.reflect.Method;
@Slf4j
public class LogTraceAdvice implements MethodInterceptor {
 private final LogTrace logTrace;
 public LogTraceAdvice(LogTrace logTrace) {
 this.logTrace = logTrace;
 }
 @Override
 public Object invoke(MethodInvocation invocation) throws Throwable {
 TraceStatus status = null;

 try {
 Method method = invocation.getMethod();
 String message = method.getDeclaringClass().getSimpleName() + "."
 + method.getName() + "()";
 status = logTrace.begin(message);
 //로직 호출
 Object result = invocation.proceed();
 logTrace.end(status);
 return result;
 } catch (Exception e) {
 logTrace.exception(status, e);
 throw e;
 }
 }
}
```
앞서 학습한 내용과 같아서 크게 어려운 부분은 없을 것이다.

**ProxyFactoryConfigV1**
```java
package hello.proxy.config.v3_proxyfactory;
import hello.proxy.app.v1.*;
import hello.proxy.config.v3_proxyfactory.advice.LogTraceAdvice;
import hello.proxy.trace.logtrace.LogTrace;
import lombok.extern.slf4j.Slf4j;
import org.springframework.aop.Advisor;
import org.springframework.aop.framework.ProxyFactory;
import org.springframework.aop.support.DefaultPointcutAdvisor;
import org.springframework.aop.support.NameMatchMethodPointcut;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
@Slf4j
@Configuration
public class ProxyFactoryConfigV1 {
 @Bean
 public OrderControllerV1 orderControllerV1(LogTrace logTrace) {
 OrderControllerV1 orderController = new
OrderControllerV1Impl(orderServiceV1(logTrace));
 ProxyFactory factory = new ProxyFactory(orderController);
 factory.addAdvisor(getAdvisor(logTrace));
 OrderControllerV1 proxy = (OrderControllerV1) factory.getProxy();
 log.info("ProxyFactory proxy={}, target={}", proxy.getClass(),
orderController.getClass());
 return proxy;
 }
 @Bean
 public OrderServiceV1 orderServiceV1(LogTrace logTrace) {
 OrderServiceV1 orderService = new
OrderServiceV1Impl(orderRepositoryV1(logTrace));
 ProxyFactory factory = new ProxyFactory(orderService);
 factory.addAdvisor(getAdvisor(logTrace));
 OrderServiceV1 proxy = (OrderServiceV1) factory.getProxy();
 log.info("ProxyFactory proxy={}, target={}", proxy.getClass(),
orderService.getClass());
 return proxy;
 }
 @Bean
 public OrderRepositoryV1 orderRepositoryV1(LogTrace logTrace) {
 OrderRepositoryV1 orderRepository = new OrderRepositoryV1Impl();
 ProxyFactory factory = new ProxyFactory(orderRepository);
 factory.addAdvisor(getAdvisor(logTrace));
 OrderRepositoryV1 proxy = (OrderRepositoryV1) factory.getProxy();
 log.info("ProxyFactory proxy={}, target={}", proxy.getClass(),
orderRepository.getClass());
 return proxy;
 }
 private Advisor getAdvisor(LogTrace logTrace) {
 //pointcut
 NameMatchMethodPointcut pointcut = new NameMatchMethodPointcut();
 pointcut.setMappedNames("request*", "order*", "save*");
 //advice
 LogTraceAdvice advice = new LogTraceAdvice(logTrace);
 //advisor = pointcut + advice
 return new DefaultPointcutAdvisor(pointcut, advice);
 }
}
```
- 포인트컷은 `NameMatchMethodPointcut`을 사용한다. 여기에는 심플 매칭 기능이 있어서 `*`을 매칭할 수 있다.
  - `request*`, `order*`, `save*`: `request`로 시작하는 메서드에 포인트컷은 `true`를 반환한다. 나머지도 같다.
  - 이렇게 설정한 이유는 `noLog()` 메서드에는 어드바이스를 적용하지 않기 위해서다.
- 어드바이저는 포인트컷(`NameMatchMethodPointcut`), 어드바이스(`LogTraceAdvice`)를 가지고 있다.
- 프록시 팩토리에 각각의 `target`과 `advisor`를 등록해서 프록시를 생성한다. 그리고 생성된 프록시를 스프링 빈으로 등록한다.

**ProxyApplication**
```java
//@Import({AppV1Config.class, AppV2Config.class})
//@Import(InterfaceProxyConfig.class)
//@Import(ConcreteProxyConfig.class)
//@Import(DynamicProxyBasicConfig.class)
//@Import(DynamicProxyFilterConfig.class)
@Import(ProxyFactoryConfigV1.class)
@SpringBootApplication(scanBasePackages = "hello.proxy.app")
public class ProxyApplication {
 public static void main(String[] args) {
 SpringApplication.run(ProxyApplication.class, args);
 }
 @Bean
 public LogTrace logTrace() {
 return new ThreadLocalLogTrace();
 }
}
```
프록시 팩토리를 통한 `ProxyFactoryConfigV1` 설정을 등록하고 실행해보자.

**애플리케이션 로딩 로그**
```
ProxyFactory proxy=class com.sun.proxy.$Proxy50,
target=class ...v1.OrderRepositoryV1Impl
ProxyFactory proxy=class com.sun.proxy.$Proxy52,
target=class ...v1.OrderServiceV1Impl
ProxyFactory proxy=class com.sun.proxy.$Proxy53,
target=class ...v1.OrderControllerV1Impl
```
V1 애플리케이션은 인터페이스가 있기 때문에 프록시 팩토리가 JDK동적 프록시를 적용한다.  
애플리케이션 로딩 로그를 통해서 JDK 동적 프록시가 적용된 것을 확인할 수 있다.

**실행 로그**  
http://localhost:8080/v1/request?itemId=hello
```
[aaaaaaaa] OrderControllerV1.request()
[aaaaaaaa] |-->OrderServiceV1.orderItem()
[aaaaaaaa] | |-->OrderRepositoryV1.save()
[aaaaaaaa] | |<--OrderRepositoryV1.save() time=1002ms
[aaaaaaaa] |<--OrderServiceV1.orderItem() time=1002ms
[aaaaaaaa] OrderControllerV1.request() time=1003ms
```



</details>


<details> <summary> 10. 프록시 팩토리 - 적용2 </summary>

## 10. 프록시 팩토리 - 적용2

이번에는 인터페이스가 없고, 구체 클래스만 있는 v2 애플리케이션에 `LogTrace` 기능을 프록시 팩토리를 통해서 프록시를 만들어 적용해보자.

```java
package hello.proxy.config.v3_proxyfactory;
import hello.proxy.app.v2.OrderControllerV2;
import hello.proxy.app.v2.OrderRepositoryV2;
import hello.proxy.app.v2.OrderServiceV2;
import hello.proxy.config.v3_proxyfactory.advice.LogTraceAdvice;
import hello.proxy.trace.logtrace.LogTrace;
import lombok.extern.slf4j.Slf4j;
import org.springframework.aop.Advisor;
import org.springframework.aop.framework.ProxyFactory;
import org.springframework.aop.support.DefaultPointcutAdvisor;
import org.springframework.aop.support.NameMatchMethodPointcut;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
@Slf4j
@Configuration
public class ProxyFactoryConfigV2 {
 @Bean
 public OrderControllerV2 orderControllerV2(LogTrace logTrace) {
 OrderControllerV2 orderController = new
OrderControllerV2(orderServiceV2(logTrace));
 ProxyFactory factory = new ProxyFactory(orderController);
 factory.addAdvisor(getAdvisor(logTrace));
 OrderControllerV2 proxy = (OrderControllerV2) factory.getProxy();
 log.info("ProxyFactory proxy={}, target={}", proxy.getClass(),
orderController.getClass());
 return proxy;
 }
 @Bean
 public OrderServiceV2 orderServiceV2(LogTrace logTrace) {
 OrderServiceV2 orderService = new
OrderServiceV2(orderRepositoryV2(logTrace));
 ProxyFactory factory = new ProxyFactory(orderService);
 factory.addAdvisor(getAdvisor(logTrace));
 OrderServiceV2 proxy = (OrderServiceV2) factory.getProxy();
 log.info("ProxyFactory proxy={}, target={}", proxy.getClass(),
orderService.getClass());
 return proxy;
 }
 @Bean
 public OrderRepositoryV2 orderRepositoryV2(LogTrace logTrace) {
 OrderRepositoryV2 orderRepository = new OrderRepositoryV2();
 ProxyFactory factory = new ProxyFactory(orderRepository);
 factory.addAdvisor(getAdvisor(logTrace));
 OrderRepositoryV2 proxy = (OrderRepositoryV2) factory.getProxy();
 log.info("ProxyFactory proxy={}, target={}", proxy.getClass(),
orderRepository.getClass());
 return proxy;
 }
 private Advisor getAdvisor(LogTrace logTrace) {
 //pointcut
 NameMatchMethodPointcut pointcut = new NameMatchMethodPointcut();
 pointcut.setMappedNames("request*", "order*", "save*");
 //advice
 LogTraceAdvice advice = new LogTraceAdvice(logTrace);
 //advisor = pointcut + advice
 return new DefaultPointcutAdvisor(pointcut, advice);
 }
}
```

```java
package hello.proxy;
//@Import({AppV1Config.class, AppV2Config.class})
//@Import(InterfaceProxyConfig.class)
//@Import(ConcreteProxyConfig.class)
//@Import(DynamicProxyBasicConfig.class)
//@Import(DynamicProxyFilterConfig.class)
//@Import(ProxyFactoryConfigV1.class)
@Import(ProxyFactoryConfigV2.class)
@SpringBootApplication(scanBasePackages = "hello.proxy.app")
public class ProxyApplication {
 public static void main(String[] args) {
 SpringApplication.run(ProxyApplication.class, args);
 }
 @Bean
 public LogTrace logTrace() {
 return new ThreadLocalLogTrace();
 }
}
```
프록시 팩토리를 통한 `ProxyFactoryConfigV2` 설정을 등록하고 실행하자.

**애플리케이션 로딩 로그**
```
ProxyFactory proxy=class hello.proxy.app.v2.OrderRepositoryV2$
$EnhancerBySpringCGLIB$$594e4e8, target=class
hello.proxy.app.v2.OrderRepositoryV2
ProxyFactory proxy=class hello.proxy.app.v2.OrderServiceV2$
$EnhancerBySpringCGLIB$$59e5130b, target=class
hello.proxy.app.v2.OrderServiceV2
ProxyFactory proxy=class hello.proxy.app.v2.OrderControllerV2$
$EnhancerBySpringCGLIB$$79c0b9e, target=class
hello.proxy.app.v2.OrderControllerV2
```

V2 애플리케이션은 인터페이스가 없고 구체 클래스만 있기 때문에 프록시 팩토리가 CGLIB을 적용한다.  
애플리케이션 로딩 로그를 통해서 CGLIB 프록시가 적용된 것을 확인할 수 있다.

**실행 로그**  
http://localhost:8080/v2/request?itemId=hello
```
[bbbbbbbb] OrderControllerV2.request()
[bbbbbbbb] |-->OrderServiceV2.orderItem()
[bbbbbbbb] | |-->OrderRepositoryV2.save()
[bbbbbbbb] | |<--OrderRepositoryV2.save() time=1001ms
[bbbbbbbb] |<--OrderServiceV2.orderItem() time=1003ms
[bbbbbbbb] OrderControllerV2.request() time=1005ms
```

</details>


<details> <summary> 11. 정리 </summary>

## 11. 정리

프록시 팩토리 덕분에 개발자는 매우 편리하게 프록시를 생성할 수 있게 되었다.  
추가로 어드바이저, 어드바이스, 포인트컷 이라는 개념 덕분에 **어떤 부가 기능**을 **어디에 적용**할지 명확하게 이해할 수 있었다.

**남은 문제**  
프록시 팩토리와 어드바이저 같은 개념 덕분에 지금까지 고민했던 문제들은 해결되었다.   
프록시도 깔끔하게 적용하고 포인트컷으로 어디에 부가 기능을 적용할지도 명확하게 정의할 수 있다.   
원본 코드를 전혀 손대지 않고 프록시를 통해 부가 기능도 적용할 수 있었다.  
그런데 아직 해결되지 않는 문제가 있다.  

**문제1 - 너무 많은 설정**  
바로 `ProxyFactoryConfigV1`, `ProxyFactoryConfigV2`와 같은 설정 파일이 지나치게 많다는 점이다.  
예를 들어서 애플리케이션에 스프링 빈이 100개가 있다면 여기에 프록시를 통해 부가 기능을 적용하려면 100개의 동적 프록시 생성 코드를 만들어야 한다!  
무수히 많은 설정 파일 때문에 설정 지옥을 경험하게 될 것이다.  
최근에는 스프링 빈을 등록하기 귀찮아서 컴포넌트 스캔까지 사용하는데, 이렇게 직접 등록하는 것도 모자라서, 프록시를 적용하는 코드까지 빈 생성 코드에 넣어야 한다.

**문제2 - 컴포넌트 스캔**  
애플리케이션 V3처럼 컴포넌트 스캔을 사용하는 경우 지금까지 학습한 방법으로는 프록시 적용이 불가능하다.  
왜냐하면 실제 객체를 컴포넌트 스캔으로 스프링 컨테이너에 스프링 빈으로 등록을 다 해버린 상태이기 때문이다.  
지금까지 학습한 프록시를 적용하려면, 실제 객체를 스프링 컨테이너에 빈으로 등록하는 것이 아니라 `ProxyFactoryConfigV1`에서 한 것 처럼, 부가 기능이 있는 프록시를 실제 객체 대신 스프링 컨테이너에 빈으로 등록해야 한다.

**두 가지 문제를 한번에 해결하는 방법이 바로 다음에 정리할 빈 후처리기이다.**

</details>


# [7. 빈 후처리기](./7.bean-postprocessor)

<details> <summary> 1. 빈 후처리기 - 소개 </summary>

## 1. 빈 후처리기 - 소개

![image](https://user-images.githubusercontent.com/28394879/141490079-38a6213f-f0b3-452c-9c6a-f397b314f3b1.png)  
`@Bean`이나 컴포넌트 스캔으로 스프링 빈을 등록하면, 스프링은 대상 객체를 생성하고 스프링 컨테이너 내부의 빈 저장소에 등록한다.  
그리고 이후에는 스프링 컨테이너를 통해 등록한 스프링 빈을 조회해서 사용하면 된다.

**빈 후처리기 - BeanPostProcessor**  
스프링이 빈 저장소에 등록할 목적으로 생성한 객체를 빈 저장소에 등록하기 직전에 조작하고 싶다면 빈 후처리기를 사용하면 된다.  
빈 포스트 프로세스(`BeanPostProcessor`)는 번역하면 빈 후처리기인데, 이름 그대로 빈을 생성한 후에 무언가를 처리하는 용도로 사용한다.

**빈 후처리기 기능**  
빈 후처리기의 기능은 막강하다.  
객체를 조작할 수도 있고, 완전히 다른 객체로 바꿔치기 하는 것도 가능하다.  

**빈 후처리기 과정**  
![image](https://user-images.githubusercontent.com/28394879/141490416-fa312f9f-a714-421d-a050-640603c0c008.png)

**빈 등록 과정을 빈 후처리기와 함께 살펴보자**  
1. 생성: 스프링 빈 대상이 되는 객체를 생성한다. (`@Bean`, 컴포넌트 스캔 모두 포함)
2. 전달: 생성된 객체를 빈 저장소에 등록하기 직전에 후 처리기에 전달한다.
3. 후 처리 작업: 빈 후처리기는 전달된 스프링 빈 객체를 조작하거나 다른 객체로 바꿔치기 할 수 있다.
4. 등록: 빈 후처리기는 빈을 반환한다. 전달 된 빈을 그대로 반환하면 해당 빈이 등록되고, 바꿔치기 하면 다른 객체가 빈 저장소에 등록된다.


**다른 객체로 바꿔치는 빈 후처리기**  
![image](https://user-images.githubusercontent.com/28394879/141490660-f5879e8d-2abe-4f4b-94f3-315f148cf4ba.png)

</details>

<details> <summary> 2. 빈 후처리기 - 예제 코드1 </summary>

## 2. 빈 후처리기 - 예제 코드1

### 일반적인 스프링 빈 등록 과정  
빈 후처리기를 학습하기 전에 먼저 일반적인 스프링 빈 등록 과정을 코드로 작성해보자.

![image](https://user-images.githubusercontent.com/28394879/141491501-fea5e59f-465e-4f70-9b9c-f027635ca5b0.png)

**BasicTest**
```java
package hello.proxy.postprocessor;
import lombok.extern.slf4j.Slf4j;
import org.junit.jupiter.api.Assertions;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.NoSuchBeanDefinitionException;
import org.springframework.context.ApplicationContext;
import
org.springframework.context.annotation.AnnotationConfigApplicationContext;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
public class BasicTest {
 @Test
 void basicConfig() {
 ApplicationContext applicationContext = new
AnnotationConfigApplicationContext(BasicConfig.class);
 //A는 빈으로 등록된다.
 A a = applicationContext.getBean("beanA", A.class);
 a.helloA();
 //B는 빈으로 등록되지 않는다.
 Assertions.assertThrows(NoSuchBeanDefinitionException.class,
 () -> applicationContext.getBean(B.class));
 }
 @Slf4j
 @Configuration
 static class BasicConfig {
 @Bean(name = "beanA")
 public A a() {
 return new A();
 }
 }
 @Slf4j
 static class A {
 public void helloA() {
 log.info("hello A");
 }
 }
 @Slf4j
 static class B {
 public void helloB() {
 log.info("hello B");
 }
 }
}
```
`new AnnotationConfigApplicationContext(BasicConfig.class)`  
스프링 컨테이너를 생성하면서 `BasicConfig.class`를 넘겨주었다.  
`BasicConfig.class`설정 파일은 스프링 빈으로 등록된다.

**등록**  
`BasicConfig.class`
```java
@Bean(name = "beanA")
public A a() {
 return new A();
}
```
`beanA`라는 이름으로 `A` 객체를 스프링 빈으로 등록했다.

**조회**  
`A a = applicationContext.getBean("beanA", A.class)`
- `beanA`라는 이름으로 `A`타입의 스프링 빈을 찾을 수 있따.

`applicationContext.getBean(B.class)`
- `B`타입의 객체는 스프링 빈으로 등록한 적이 없기 떄문에 스프링 컨테이너에서 찾을 수 없다.

</details>

<details> <summary> 3. 빈 후처리기 - 예제 코드2 </summary>

## 3. 빈 후처리기 - 예제 코드2

### 빈 후처리기 적용

이번에는 빈 후처리기를 통해서 A 객체를 B 객체로 바꿔치기 해보자.

![image](https://user-images.githubusercontent.com/28394879/141492976-86214fde-bdc1-4a91-9ee9-39cf517a8903.png)

**BeanPostProcessor 인터페이스 - 스프링 제공**
```java
public interface BeanPostProcessor {
 Object postProcessBeforeInitialization(Object bean, String beanName) throws
BeansException
 Object postProcessAfterInitialization(Object bean, String beanName) throws
BeansException
}
```
- 빈 후처리기를 사용하려면 `BeanPostProcessor` 인터페이스를 구현하고, 스프링 빈으로 등록하면 된다.
- `postProcessBeforeInitialization`: 객체 생성 이후에 `@PostConstruct` 같은 초기화가 발생하기 전에 호출되는 포스트 프로세서이다. 
- `postProcessAfterInitialization`: 객체 생성 이후에 `@PostConstruct` 같은 초기화가 발생한 다음에 호출되는 포스트 프로세서이다.

**BeanPostProcessorTest**
```java
package hello.proxy.postprocessor;
import lombok.extern.slf4j.Slf4j;
import org.junit.jupiter.api.Assertions;
import org.junit.jupiter.api.Test;
import org.springframework.beans.BeansException;
import org.springframework.beans.factory.NoSuchBeanDefinitionException;
import org.springframework.beans.factory.config.BeanPostProcessor;
import org.springframework.context.ApplicationContext;
import
org.springframework.context.annotation.AnnotationConfigApplicationContext;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
public class BeanPostProcessorTest {
 @Test
 void postProcessor() {
 ApplicationContext applicationContext = new
AnnotationConfigApplicationContext(BeanPostProcessorConfig.class);
 //beanA 이름으로 B 객체가 빈으로 등록된다.
 B b = applicationContext.getBean("beanA", B.class);
 b.helloB();
 //A는 빈으로 등록되지 않는다.
 Assertions.assertThrows(NoSuchBeanDefinitionException.class,
 () -> applicationContext.getBean(A.class));
 }
 @Slf4j
 @Configuration
 static class BeanPostProcessorConfig {
 @Bean(name = "beanA")
 public A a() {
 return new A();
 }
 @Bean
 public AToBPostProcessor helloPostProcessor() {
 return new AToBPostProcessor();
 }
 }
 @Slf4j
 static class A {
 public void helloA() {
 log.info("hello A");
 }
 }
 @Slf4j
 static class B {
 public void helloB() {
 log.info("hello B");
 }
 }
 @Slf4j
 static class AToBPostProcessor implements BeanPostProcessor {
 @Override
 public Object postProcessAfterInitialization(Object bean, String
beanName) throws BeansException {
 log.info("beanName={} bean={}", beanName, bean);
 if (bean instanceof A) {
 return new B();
 }
 return bean;
 }
 }
}
```
**AToBPostProcessor** 
- 빈 후처리기이다. 인터페이스인 `BeanPostProcessor`를 구현하고, 스프링 빈으로 등록하면 스프링 컨테이너가 빈 후처리기로 인식하고 동작한다.
- 이 빈 후처리기는 A 객체를 새로운 B 객체로 바꿔치기 한다. 파라미터로 넘어오는 빈(`bean`)객체가 `A`의 인스턴스이면 새로운 `B`객체를 생성해서 반환한다. 여기서 `A`대신에 반환된 값인 `B`가 스프링 컨테이너에 등록된다. 다음 실행 결과를 보면 `beanName=beanA`, `bean=A` 객체의 인스턴스가 빈 후처리기에 넘어온 것을 확인할 수 있다.

**실행 결과**  
```
..AToBPostProcessor - beanName=beanA
bean=hello.proxy.postprocessor...A@21362712
..B - hello B
```

`B b = applicationContext.getBean("beanA", B.class)`
- 실행 결과를 보면 최종적으로 `"beanA"`라는 스프링 빈 이름에 `A`객체 대신에 `B`객체가 등록된 것을 확인할 수 있다. `A`는 스프링 빈으로 등록조차 되지 않는다.

**정리**  
빈 후처리기는 빈을 조작하고 변경할 수 있는 후킹 포인트이다.  
이것은 빈 객체를 조작하거나 심지어 다른 객체로 바꾸어 버릴 수 있을 정도로 막강하다.  
여기서 조작이라는 것은 해당 객체의 특정 메서드를 호출하는 것을 뜻한다.  
일반적으로 스플이 컨테이너가 등록하는, 특정 컴포넌트 스캔의 대상이 되는 빈들은 중간에 조작할 방법이 없는데, 빈 후처리기를 사용하면 개발자가 등록하는 모든 빈을 중간에 조작할 수 있다.  
이 말은 **빈 객체를 프록시로 교체**하는 것도 가능하다는 뜻이다.

> 참고 - @PostConstruct의 비밀  
> `@PostConstruct`는 스프링 빈 생성 이후에 빈을 초기화 하는 역할을 한다.  
> 그런데 생각해보면 빈의 초기화 라는 것이 단순히 `@PostConstruct` 애노테이션이 붙은 초기화 메서드를 한번 호출만 하면 된다.  
> 쉽게 이야기해서 생성된 빈을 한번 조작하는 것이다.  
> 따라서 빈을 조작하는 행위를 하는 적절한 빈 후처리기가 있으면 될 것 같다.  
> 스프링은 `CommonAnnotationBeanPostProcessor`라는 빈 후처리기를 자동으로 등록하는데, 여기에서 `@PostConstruct`애노테이션이 붙은 메서드를 호출한다.  
> 따라서 스프링 스스로도 스프링 내부의 기능을 확장하기 위해 빈 후처리기를 사용한다. 


</details>

<details> <summary> 4. 빈 후처리기 - 적용 </summary>

## 4. 빈 후처리기 - 적용

빈 후처리기를 사용해서 실제 객체 대신 프록시를 스프링 빈으로 등록해보자.  
이렇게 하면 수동으로 등록하는 빈은 물론이고, 컴포넌트 스캔을 사용하는 빈까지 모두 프록시를 적용할수 있다.  
더 나아가서 설정 파일에 있는 수 많은 프록시 생성 코드도 한번에 제거할 수 있다.

![image](https://user-images.githubusercontent.com/28394879/141602864-a15041eb-87c2-4c52-8c39-8a694f199321.png)


**PackageLogTraceProxyPostProcessor**
```java
package hello.proxy.config.v4_postprocessor.postprocessor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.aop.Advisor;
import org.springframework.aop.framework.ProxyFactory;
import org.springframework.beans.BeansException;
import org.springframework.beans.factory.config.BeanPostProcessor;
@Slf4j
public class PackageLogTraceProxyPostProcessor implements BeanPostProcessor {
 private final String basePackage;
 private final Advisor advisor;
 public PackageLogTraceProxyPostProcessor(String basePackage, Advisor
advisor) {
 this.basePackage = basePackage;
 this.advisor = advisor;
 }
 @Override
 public Object postProcessAfterInitialization(Object bean, String beanName)
 throws
BeansException {
 log.info("param beanName={} bean={}", beanName, bean.getClass());
 //프록시 적용 대상 여부 체크
 //프록시 적용 대상이 아니면 원본을 그대로 반환
 String packageName = bean.getClass().getPackageName();
 if (!packageName.startsWith(basePackage)) {
 return bean;
 }
 //프록시 대상이면 프록시를 만들어서 반환
 ProxyFactory proxyFactory = new ProxyFactory(bean);
 proxyFactory.addAdvisor(advisor);
 Object proxy = proxyFactory.getProxy();
 log.info("create proxy: target={} proxy={}", bean.getClass(),
proxy.getClass());
 return proxy;
 }
}
```
- `PackageLogTraceProxyPostProcessor`는 원본 객체를 프록시 객체로 변환하는 역할을 한다. 이때 프록시 팩토리를 사용하는데, 프록시 팩토리는 `advisor`가 필요하기 때문에 이 부분은 외부에서 주입받도록 했다.
- 모든 스프링 빈들에 프록시를 적용할 필요는 없다. 여기서는 특정 패키지와 그 하위에 위치한 스프링 빈들만 프록시를 적용한다. 여기서는 `hello.proxy.app`과 관련된 부분에만 적용하면 된다. 다른 패키지의 객체들은 원본 객체를 그대로 반환한다.  
- 프록시 적용 대상의 반환 값을 보면 원본 객체 대신에 프록시 객체를 반환한다. 따라서 스프링 컨테이너에 원본 객체 대신에 프록시 객체가 스프링 빈으로 등록된다. 원본 객체는 스프링 빈으로 등록되지 않는다.  


**BeanPostProcessorConfig**
```java
package hello.proxy.config.v4_postprocessor;
import hello.proxy.config.AppV1Config;
import hello.proxy.config.AppV2Config;
import hello.proxy.config.v3_proxyfactory.advice.LogTraceAdvice;
import
hello.proxy.config.v4_postprocessor.postprocessor.PackageLogTraceProxyPostProce
ssor;
import hello.proxy.trace.logtrace.LogTrace;
import lombok.extern.slf4j.Slf4j;
import org.springframework.aop.Advisor;
import org.springframework.aop.support.DefaultPointcutAdvisor;
import org.springframework.aop.support.NameMatchMethodPointcut;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Import;
@Slf4j
@Configuration
@Import({AppV1Config.class, AppV2Config.class})
public class BeanPostProcessorConfig {
 @Bean
 public PackageLogTraceProxyPostProcessor
logTraceProxyPostProcessor(LogTrace logTrace) {
 return new PackageLogTraceProxyPostProcessor("hello.proxy.app",
getAdvisor(logTrace));
 }
 private Advisor getAdvisor(LogTrace logTrace) {
 //pointcut
 NameMatchMethodPointcut pointcut = new NameMatchMethodPointcut();
 pointcut.setMappedNames("request*", "order*", "save*");
 //advice
 LogTraceAdvice advice = new LogTraceAdvice(logTrace);
 //advisor = pointcut + advice
 return new DefaultPointcutAdvisor(pointcut, advice);
 }
}
```
- `@Import({AppV1Config.class, AppV2Config.class})`: V3는 컴포넌트 스캔으로 자동으로 스프링 빈으로 등록되지만, V1, V2 애플리케이션은 수동으로 스프링 빈으로 등록해야 동작한다. `ProxyApplication`에서 등록해도 되지만 편의상 여기에 등록하자.  
- `@Bean logTraceProxyPostProcessor()`: 특정 패키지를 기준으로 프록시를 생성하는 빈 후처리기를 스프링 빈으로 등록한다. 빈 후처리기는 스프링 빈으로만 등록함녀 자동으로 동작한다. 여기에 프록시는 적용할 패키지 정보 (`hello.proxy.app`)와 어드바이저(`getAdvisor(logTrace)`)를 넘겨준다.
- 이제 **프록시를 생성하는 코드가 설정 파일에는 필요 없다.** 순수한 빈 등록만 고민하면 된다. 프록시를 생성하고 프록시를 스프링 빈으로 등록하는 것은 빈 후처리기가 모두 처리해준다.


**ProxyApplication**
```java
//@Import({AppV1Config.class, AppV2Config.class})
//@Import(InterfaceProxyConfig.class)
//@Import(ConcreteProxyConfig.class)
//@Import(DynamicProxyBasicConfig.class)
//@Import(DynamicProxyFilterConfig.class)
//@Import(ProxyFactoryConfigV1.class)
//@Import(ProxyFactoryConfigV2.class)
@Import(BeanPostProcessorConfig.class)
@SpringBootApplication(scanBasePackages = "hello.proxy.app")
public class ProxyApplication {
 public static void main(String[] args) {
 SpringApplication.run(ProxyApplication.class, args);
 }
 @Bean
 public LogTrace logTrace() {
 return new ThreadLocalLogTrace();
 }
}
```
`BeanPostProcessorConfig.class`를 등록하자.

**애플리케이션 로딩 로그**  
중요 부분만 남기고 순서를 조정하고 축약했다.

```
#v1 애플리케이션 프록시 생성 - JDK 동적 프록시
create proxy: target=v1.OrderRepositoryV1Impl proxy=class com.sun.proxy.
$Proxy50
create proxy: target=v1.OrderServiceV1Impl proxy=class com.sun.proxy.$Proxy51
create proxy: target=v1.OrderControllerV1Impl proxy=class com.sun.proxy.
$Proxy52
#v2 애플리케이션 프록시 생성 - CGLIB
create proxy: target=v2.OrderRepositoryV2 proxy=v2.OrderRepositoryV2$
$EnhancerBySpringCGLIB$$x4
create proxy: target=v2.OrderServiceV2 proxy=v2.OrderServiceV2$
$EnhancerBySpringCGLIB$$x5
create proxy: target=v2.OrderControllerV2 proxy=v2.OrderControllerV2$
$EnhancerBySpringCGLIB$$x6
#v3 애플리케이션 프록시 생성 - CGLIB
create proxy: target=v3.OrderRepositoryV3 proxy=3.OrderRepositoryV3$
$EnhancerBySpringCGLIB$$x1
create proxy: target=v3.orderServiceV3 proxy=3.OrderServiceV3$
$EnhancerBySpringCGLIB$$x2
create proxy: target=v3.orderControllerV3 proxy=3.orderControllerV3$
$EnhancerBySpringCGLIB$$x3
```
- 여기서는 생략했지만, 실행해보면 스프링 부트가 기본으로 등록하는 수 많은 빈들이 빈 후처리기를 통과하는 것을 확인할 수 있다. 여기에 모두 프록시를 적용하는 것은 올바르지 않다. 꼭 필요한 곳에만 프록시를 적용해야 한다. 여기서는 `basePacke`를 사용해서 v1~v3애플리케이션 관련 빈들만 프록시 적용 대상이 되도록 했다.
- v1: 인터페이스가 있으므로 JDK 동적 프록시가 적용된다.
- v2: 구체 클래스만 있으므로 CGLIB 프록시가 적용된다.
- v3: 구체 클래스만 있으므로 CGLIB 프록시가 적용된다.

**컴포넌트 스캔에도 적용**  
여기서 중요한 포인트는 v1, v2와 같이 수동으로 등록한 빈 뿐만 아니라 컴포넌트 스캔을 통해 등록한 v3 빈들도 프록시를 적용할 수 있다는 점이다. 이것은 모두 빈 후처리기 덕분이다.

**실행**
- http://localhost:8080/v1/request?itemId=hello
- http://localhost:8080/v2/request?itemId=hello
- http://localhost:8080/v3/request?itemId=hello

실행해보면 모두 동일한 결과가 나오는 것을 확인할 수 있다.

**프록시 적용 대상 여부 체크**
- 애플리케이션을 실행해서 로그를 확인해보면 알겠지만, 우리가 직접 등록한 스프링 빈들 뿐만 아니라 스프링 부트가 기본으로 등록하는 수 많은 빈들이 빈 후처리기에 넘어온다. 그래서 어떤 빈을 프록시로 만들 것인지 기준이 필요하다. 여기서는 간단히 `basePackage`를 사용해서 특정 패키지를 기준으로 해당 패키지와 그 하위 패키지의 빈들을 프록시로 만든다.
- 스프링 부트가 기본으로 제공하는 빈 중에는 프록시 객체를 만들 수 없는 빈들도 있다. 따라서 모든 객체를 프록시로 만들 경우 오류가 발생한다. 


</details>

<details> <summary> 5. 빈 후처리기 - 정리 </summary>

## 5. 빈 후처리기 - 정리

이전에 보았던 문제들이 빈 후처리기를 통해서 어떻게 해결되었는지 정리해보자.

**문제1 - 너무 많은 설정**  
프록시를 직접 스프링 빈으로 등록하는 `ProxyFactoyConfig1`, `ProxyFactoryConfigV2`와 같은 설정 파일은 프록시 관련 설정이 지나치게 많다는 문제가 있다.  
예를 들어서 애플리케이션에 스프링 빈이 100개가 있다면 여기에 프록시를 통해 부가 기능을 적용하려면 100개의 프록시 설정 코드가 들어가야 한다.  
무수히 많은 설정 파일 때문에 설정 지옥을 경험하게 될 것이다.  
스프링 빈을 편리하게 등록하려고 컴포넌트 스캔까지 사용하는데, 이렇게 직접 등록하는 것도 모자라서, 프록시를 적용하는 코드까지 빈 생성 코드에 넣어야 했다. 

**문제2 - 컴포넌트 스캔**  
애플리케이션 V3처럼 컴포넌트 스캔을 사용하는 경우 지금까지 학습한 방법으로는 프록시 적용이 불가능했다.  
왜냐하면 컴포넌트 스캔으로 이미 스프링 컨테이너에 실제 객체를 스프링 빈으로 등록을 다 해버린 상태이기 때문이다.  
좀 더 풀어서 설명하자면, 지금까지 학습한 방식으로 프록시를 적용하려면, 원본 객체를 스프링 컨테이너에 빈으로 등록하는 것이 아니라 `ProxyFactoryConfigV1`에서 한 것 처럼, 프록시를 원본 객체 대신 스프링 컨테이너에 빈으로 등록해야 한다.  
그런데 컴포넌트 스캔은 원본 객체를 스프링 빈으로 자동으로 등록하기 떄문에 프록시 적용이 불가능하다.

**문제 해결**  
빈 후처리기 덕분에 프록시를 생성하는 부분을 하나로 집중할 수 있따.  
그리고 컴포넌트 스캔처럼 스프링이 직접 대상을 빈으로 등록하는 경우에도 중간에 빈 등록 과정을 가로채서 원본 대신에 프록시를 스프링 빈으로 등록할 수 있다.  
덕분에 애플리케이션에 수 많은 스프링 빈이 추가되어도 프록시와 관련된 코드는 전혀 변경하지 않아도 된다.  
그리고 컴포넌트 스캔을 사용해도 프록시가 모두 적용된다. 

**하지만 개발자의 욕심은 끝이 없다.**  
스프링은 프록시를 생성하기 위한 빈 후처리기를 이미 만들어서 제공한다.

> 중요  
> 프록시의 적용 대상 여부를 여기서는 간단히 패키지를 기준으로 설정했다.  
> 그런데 잘 생각해보면 포인트컷을 사용하면 더 깔끔할 것 같다.
> 포인트컷은 이미 클래스, 메서드 단위의 필터 기능을 가지고 있기 떄문에, 프록시 적용 대상 여부를 정밀하게 설정할 수 있다.  
> 참고로 어드바이저는 포인트컷을 가지고 있다.  
> 따라서 어드바이저를 통해 포인트컷을 확인할 수 있다.  
> 뒤에서 학습하겠지만 스프링 AOP는 포인트컷을 사용해서 프록시 적용 대상 여부를 체크한다.

> 결과적으로 포인트컷은 다음 두 곳에 사용된다.
> 1. 프록시 적용 대상 여부를 체크해서 꼭 필요한 곳에만 프록시를 적용한다. (빈 후처리기 - 자동 프록시 생성)
> 2. 프록시의 어떤 메서드가 호출 되었을 때 어드바이스를 적용할 지 판단한다. (프록시 내부)

</details>

<details> <summary> 6. 스프링이 제공하는 빈 후처리기1 </summary>

## 6. 스프링이 제공하는 빈 후처리기1

주의 - 다음을 꼭 추가해주어야 한다.  
build.gradle - 추가   
```
implementation 'org.springframework.boot:spring-boot-starter-aop'
```
이 라이브러리를 추가하면 `aspectjweaver`라는 `aspectJ`관련 라이브러리를 등록하고, 스프링 부트가 AOP 관련 클래스를 자동으로 스프링 빈에 등록한다.  
스프링 부트가 없던 시절에는 `@EnableAspectJAutoProxy`를 직접 사용해야 했는데, 이 부분을 스프링 부트가 자동으로 처리해준다.   
`aspectJ`는 뒤에서 설명한다.   
스프링 부트가 활성화하는 빈은 `AopAutoConfiguration`를 참고하자. 

**자동 프록시 생성기 - AutoProxyCreator**
- 앞서 이야기한 스프링 부트 자동 설정으로 `AnnotationAwareAspectJAutoProxyCreator`라는 빈 후처리기가 스프링 빈에 자동으로 등록한다.
- 이름 그대로 자동으로 프록시를 생성해주는 빈 후처리기이다.
- 이 빈 후처리기는 스프링 빈으로 등록된 `Advisor`들을 자동으로 찾아서 프록시가 필요한 곳에 자동으로 프록시를 적용해준다.
- `Advisor`안에는 `Pointcut`과 `Advice`가 이미 모두 포함되어 있다. 따라서 `Advisor`만 알고 있으면 그 안에 있는 `Pointcut`으로 어떤 스프링 빈에 프록시를 적용해야 할 지 알 수 있다. 그리고 `Advice`로 부가 기능을 적용하면 된다.

> 참고  
> `AnnotationAwareAspectJAutoProxyCreator`는 @AspectJ와 관련된 AOP 기능도 자동으로 찾아서 처리해준다.  
> `Advisor`는 물론이고, `@Aspect`도 자동으로 인식해서 프록시를 만들고 AOP를 적용해준다.  
> `@Aspect`에 대한 자세한 내용은 뒤에 설명한다. 


**자동 프록시 생성기의 작동 과정 - 그림**  
![image](https://user-images.githubusercontent.com/28394879/141670534-93dea9f3-0dcc-4d25-9802-363a6af2741a.png)

**자동 프록시 생성기의 작동 과정을 알아보자**  
1. 생성: 스프링이 스프링 빈 대상이 되는 객체를 생성한다. (`@Bean`, 컴포넌트 스캔 모두 포함)
2. 전달: 생성된 객체를 빈 저장소에 등록하기 직전에 빈 후처리기에 전달한다.
3. 모든 Advisor 빈 조회: 자동 프록시 생성기 - 빈 후처리기는 스프링 컨테이너에 모든 `Advisor`를 조회한다.
4. 프록시 적용 대상 체크: 앞서 조회한 `Advisor`에 포함되어 있는 포인트컷을 사용해서 해당 객체가 프록시를 적용할 대상인지 아닌지 판단한다. 이때 객체의 클래스 정보는 물론이고, 해당 객체의 모든 메서드를 포인트컷에 하나하나 모두 매칭해본다. 그래서 조건이 하나라도 만족하면 프록시 적용 대상이 된다. 예를 들어서 10개의 메서드 중에 하나만 포인트컷 조건에 만족해도 프록시 적용 대상이 된다.
5. 프록시 생성: 프록시 적용 대상이면 프록시를 생성하고 반환해서 프록시를 스프링 빈으로 등록한다. 만약 프록시 적용 대상이 아니라면 원본 객체를 반환해서 원본 객체를 스프링 빈으로 등록한다.
6. 빈 등록: 반환된 객체는 스프링 빈으로 등록된다.

**생성된 프록시**  
![image](https://user-images.githubusercontent.com/28394879/141670626-a4ccced2-5e5c-475f-a33c-03168e54c5e0.png)  
프록시는 내부에 어드바이저와 실제 호출해야할 대상 객체 (`target`)을 알고 있다. 

코드를 통해 바로 적용해보자.

**AutoProxyConfig**
```java
package hello.proxy.config.v5_autoproxy;
import hello.proxy.config.AppV1Config;
import hello.proxy.config.AppV2Config;
import hello.proxy.config.v3_proxyfactory.advice.LogTraceAdvice;
import hello.proxy.trace.logtrace.LogTrace;
import org.springframework.aop.Advisor;
import org.springframework.aop.aspectj.AspectJExpressionPointcut;
import org.springframework.aop.support.DefaultPointcutAdvisor;
import org.springframework.aop.support.NameMatchMethodPointcut;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Import;
@Configuration
@Import({AppV1Config.class, AppV2Config.class})
public class AutoProxyConfig {
  @Bean
  public Advisor advisor1(LogTrace logTrace) {
    NameMatchMethodPointcut pointcut = new NameMatchMethodPointcut();
    pointcut.setMappedNames("request*", "order*", "save*");
    LogTraceAdvice advice = new LogTraceAdvice(logTrace);
    //advisor = pointcut + advice
    return new DefaultPointcutAdvisor(pointcut, advice);
  }
}
```
- `AutoProxyConfig`코드를 보면 `advisor1`이라는 어드바이저 하나만 등록했다.
- 빈 후처리기는 이제 등록하지 않아도 된다. 스프링은 자동 프록시 생성기라는 (`AnnotationAwareAspectJAutoProxyCreator`)빈 후처리기르 자동으로 등록해준다.

```java
//@Import({AppV1Config.class, AppV2Config.class})
//@Import(InterfaceProxyConfig.class)
//@Import(ConcreteProxyConfig.class)
//@Import(DynamicProxyBasicConfig.class)
//@Import(DynamicProxyFilterConfig.class)
//@Import(ProxyFactoryConfigV1.class)
//@Import(ProxyFactoryConfigV2.class)
//@Import(BeanPostProcessorConfig.class)
@Import(AutoProxyConfig.class)
@SpringBootApplication(scanBasePackages = "hello.proxy.app")
public class ProxyApplication {
 public static void main(String[] args) {
 SpringApplication.run(ProxyApplication.class, args);
 }
 @Bean
 public LogTrace logTrace() {
 return new ThreadLocalLogTrace();
 }
}
```

**실행**  
- http://localhost:8080/v1/request?itemId=hello
- http://localhost:8080/v2/request?itemId=hello
- http://localhost:8080/v3/request?itemId=hello

실행하면 모두 프록시 적용된 결과가 나오는 것을 확인할 수 있다.

**실행하면 로그가 나오면 안됨** 
- http://localhost:8080/v1/no-log

로그가 출력되지 않는 것을 확인할 수 있다.

### 중요: 포인트컷은 2가지에 사용된다.

1. 프록시 적용 여부 판단 - 생성 단계
   - 자동 프록시 생성기는 포인트컷을 사용해서 해당 빈이 프록시를 생성할 필요가 있는지 없는지 체크한다.
   - 클래스 + 메서드 조건을 모두 비교한다. 이때 모든 메서드를 체크하는데, 퐁니트컷 조건에 하나하나 매칭해본다. 만약 조건에 맞는 것이 하나라도 있으면 프록시를 생성한다.
     - 예) `orderControllerV1`은 `request()`, `noLog()`가 있다. 여기에서 `request()`가 조건에 만족하므로 프록시를 생성한다.
   - 만약 조건에 맞는 것이 하나도 없으면 프록시를 생성할 필요가 없으므로 프록시를 생성하지 않는다.
2. 어드바이스 적용 여부 판단 - 사용단계
   - 프록시가 호출되었을 때 부가 기능인 어드바이스를 적용할지 말지 포인트컷을 보고 판단한다.
   - 앞서 설명한 예에서 `orderControllerV1`은 이미 프록시가 걸려있다.
   - `orderControllerV1`의 `request()`는 현재 포인트컷 조건에 만족하므로 프록시를 어드바이스를 먼저 호출하고, `target`을 호출한다.
   - `orderControllerV1`의 `noLog()`는 현재 포인트컷 조건에 만족하지 않으므로 어드바이스를 호출하지 않고 바로 `target`만 호출한다.

**참고**: 프록시를 모든 곳에 생성하는 것은 비용 낭비이다. 꼭 필요한 곳에 최소한의 프록시를 적용해야 한다.  
그래서 자동 프록시 생성기는 모든 스프링 빈에 프록시를 적용하는 것이 아니라 포인트컷으로 한번 필터링해서  
어드바이스가 사용될 가능성이 있는 곳에만 프록시를 생성한다.



</details>

<details> <summary> 7. 스프링이 제공하는 빈 후처리기2 </summary>

## 7. 스프링이 제공하는 빈 후처리기2

**애플리케이션 로딩 로그**
```
EnableWebMvcConfiguration.requestMappingHandlerAdapter()
EnableWebMvcConfiguration.requestMappingHandlerAdapter() time=63ms
```
애플리케이션 서버를 실행해보면, 스프링이 초기화 되면서 기대하지 않은 이러한 로그들이 올라온다.  
그 이유는 지금 사용한 포인트컷이 단순히 메서드 이름에 `"request*", "order*", "save*"`만 포함되어 있으면 매칭 된다고 판단하기 때문이다.  
결국 스프링이 내부에서 사용하는 빈에도 메서드 이름에 `request`라는 단어들만 들어가 있으면 프록시가 만들어지게 되고, 어드바이스도 적용되는 것이다.

결론적으로 패키지에 메서드 이름까지 함께 지정할 수 있는 매우 정밀한 포인트컷이 필요하다.

**AspectJExpressionPointcut**  
AspectJ라는 AOP에 특화된 포인트컷 표현식을 적용할 수 있다. AspectJ 포인트컷 표현식과 AOP는 조금 뒤에 자세히 정리하겠다.  
지금은 특별한 표현식으로 복잡한 포인트컷을 만들 수 있구나 라고 대략 이해하면 된다.  

**AutoProxyConfig - advisor2 추가**  
```java
@Bean
public Advisor advisor2(LogTrace logTrace) {
 AspectJExpressionPointcut pointcut = new AspectJExpressionPointcut();
 pointcut.setExpression("execution(* hello.proxy.app..*(..))");
 LogTraceAdvice advice = new LogTraceAdvice(logTrace);
 //advisor = pointcut + advice
 return new DefaultPointcutAdvisor(pointcut, advice);
}
```

**주의**  
`advisor1`에 있는 `@Bean`은 꼭 주석처리해주어야 한다. 그렇지 않으면 어드바이저가 중복 등록된다.

- `AspectJExpressionPointcut`: AspectJ 포인트컷 표현식을 적용할 수 있다.
- `execution(* hello.proxy.app..*(..))`: AspectJ가 제공하는 포인트컷 표현식이다. 이후 자세히 정리하고, 지금은 간단히 알아보자.
  - `*`: 모든 반환 타입
  - `hello.proxy.app..`: 해당 패키지와 그 하위 패키지
  - `*(..)`: `*` 모든 메서드 이름, `(..)` 파라미터는 상관 없음

쉽게 이야기해서 `hello.proxy.app` 패키지와 그 하위 패키지의 모든 메서드는 포인트컷의 매칭 대상이 된다.

**실행**  
- http://localhost:8080/v1/request?itemId=hello
- http://localhost:8080/v2/request?itemId=hello
- http://localhost:8080/v3/request?itemId=hello

실행하면 모두 동일한 결과가 나오는 것을 확인할 수 있다.

**실행하면 로그가 나오면 안됨**
- http://localhost:8080/v1/no-log

그런데 문제는 이 부분에 로그가 출력된다. `advisor2`에서는 단순히 `package`를 기준으로 포인트컷 매칭을 했기 떄문이다.

**AutoProxyConfig advisor3 추가**
```java
@Bean
public Advisor advisor3(LogTrace logTrace) {
 AspectJExpressionPointcut pointcut = new AspectJExpressionPointcut();
 pointcut.setExpression("execution(* hello.proxy.app..*(..)) && !execution(*
hello.proxy.app..noLog(..))");
 LogTraceAdvice advice = new LogTraceAdvice(logTrace);
 //advisor = pointcut + advice
 return new DefaultPointcutAdvisor(pointcut, advice);
}
```

**주의**  
- `advisor1`, `advisor2`에 있는 `@Bean`은 꼭 주석처리해주어야 한다. 그렇지 않으면 어드바이저가 중복 등록된다.

**표현식을 다음과 같이 수정하자.**
```
execution(* hello.proxy.app..*(..)) && !execution(* hello.proxy.app..noLog(..))
```
- `&&`: 두 조건을 모두 만족해야 함
- `!`: 반대

`hello.proxy.app` 패키지와 하위 패키지의 모든 메서드는 포인트컷의 매칭하되, `noLog()`메서드는 제외하라는 뜻이다.

**실행하면 로그가 나오면 안됨**  
- http://localhost:8080/v1/no-log 
이제 로그가 남지 않는 것을 확인할 수 있다.

> 참고   
> AspectJ, AOP는 이후에 자세히 정리한다. 지금은 이런 포인트컷도 있구나 정도 알고 넘어가면 된다.

</details>

<details> <summary> 8. 하나의 프록시, 여러 Advisor 적용 </summary>

## 8. 하나의 프록시, 여러 Advisor 적용

예를 들어서 어떤 스프링 빈이 `advisor1`, `advisor2`가 제공하는 포인트컷의 조건을 모두 만족하면  
프록시 자동 생성기는 프록시를 몇 개 생성할까?  
프록시 자동 생성기는 프록시를 하나만 생성한다. 왜냐하면 프록시 팩토리가 생성하는 프록시는 내부에 여러 `advisor`들을 포함할 수 있기 때문이다.  
따라서 프록시를 여러 개 생성해서 비용을 낭비할 이유가 없다.

**프록시 자동 생성기 상황별 정리**  
- `advisor1`의 포인트컷만 만족 -> 프록시 1개 생성, 프록시에 `advisor1`만 포함
- `advisor1`, `advisor2`의 포인트컷을 모두 만족 -> 프록시 1개 생성, 프록시에 `advisor1`, `advisor2` 모두 포함
- `advisor1`, `advisor2`의 포인트컷을 모두 만족하지 않음 -> 프록시가 생성되지 않음

### 이후에 설명할 스프링 AOP도 동일한 방식으로 동작한다.
![image](https://user-images.githubusercontent.com/28394879/141672473-4836901b-e1ba-433f-964c-41ccb76eefe1.png)




</details>

<details> <summary> 9. 정리 </summary>

## 9. 정리
자동 프록시 생성기인 `AnnotationAwareAspectJAutoProxyCreator` 덕분에 개발자는 매우 편리하게 프록시를 적용할 수 있다.  
이제 `Advisor`만 스프링 빈으로 등록하면 된다.

`Advisor` = `Pointcut` + `Advice`

다음은 `@Aspect` 애노테이션을 사용해서 더 편리하게 포인트컷과 어드바이스를 만들고 프록시를 적용해보자. 

</details>


# [8. @Aspect AOP](./8.aspect-aop)

<details> <summary> 1. @Aspect 프록시 - 적용 </summary>

## 1. @Aspect 프록시 - 적용

스프링 애플리케이션에 프록시를 적용하려면 포인트컷과 어드바이스로 구성되어 있는 어드바이저(`Advisor`)를 만들어서 스프링 빈으로 등록하면 된다.  
그러면 나머지는 앞서 배운 자동 프록시 생성기가 모두 자동으로 처리해준다.  
자동 프록시 생성기는 스프링 빈으로 등록된 어드바이저들을 찾고, 스프링 빈들에 자동으로 프록시를 적용해준다.  
(물론 포인트컷이 매칭되는 경우에 프록시를 생성한다.)

스프링은 `@Aspect` 애노테이션으로 매우 편리하게 포인트컷과 어드바이스로 구성되어 있는 어드바이저 생성 기능을 지원한다.

지금까지 어드바이저를 직접 만들었던 부분을 `@Aspect` 애노테이션을 사용해서 만들어보자.

> 참고: `@Aspect`는 관점 이향 프로그래밍(AOP)을 가능하게 하는 AspectJ 프로젝트에서 제공하는 애노테이션이다.  
> 스프링은 이것을 차용해서 프록시를 통한 AOP를 가능하게 한다.  
> AOP와 AspectJ 관련된 자세한 내용은 다음에 정리한다.  
> 지금은 프록시에 초점을 맞추자.  
> 우선 이 애노테이션을 사용해서 스프링이 편리하게 프록시를 만들어준다고 생각하면 된다.  


**LogTraceAspect**
```java
package hello.proxy.config.v6_aop.aspect;
import hello.proxy.trace.TraceStatus;
import hello.proxy.trace.logtrace.LogTrace;
import lombok.extern.slf4j.Slf4j;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
@Slf4j
@Aspect
public class LogTraceAspect {
 private final LogTrace logTrace;
 public LogTraceAspect(LogTrace logTrace) {
 this.logTrace = logTrace;
 }
 @Around("execution(* hello.proxy.app..*(..))")
 public Object execute(ProceedingJoinPoint joinPoint) throws Throwable {
 TraceStatus status = null;
// log.info("target={}", joinPoint.getTarget()); //실제 호출 대상
// log.info("getArgs={}", joinPoint.getArgs()); //전달인자
// log.info("getSignature={}", joinPoint.getSignature()); //join point
시그니처
 try {
 String message = joinPoint.getSignature().toShortString();
 status = logTrace.begin(message);
 //로직 호출
 Object result = joinPoint.proceed();
 logTrace.end(status);
 return result;
 } catch (Exception e) {
 logTrace.exception(status, e);
 throw e;
 }
 }
}
```
- `@Aspect`: 애노테이션 기반 프록시를 적용할 때 필요하다.
- `@Around("execution(* hello.proxy.app..*(..))")`
  - `@Around`의 값에 포인트컷 표현식을 넣는다. 표현식은 AspectJ 표현식을 사용한다.
  - `@Around`의 메서드는 어드바이스(`Advice`)가 된다.
  - `ProceedingJoinPoint joinPoint`: 어드바이스에서 살펴본 `MethodInvocation invocation`과 유사한 기능이다. 내부에 실제 호출 대상, 전달 인자, 그리고 어떤 객체와 어떤 메서드가 호출되었는지 정보가 포함되어 있다.  
  - `joinPoint.proceed()`: 실제 호출 대상(`target`)을 호출한다.

**AppConfig**
```java
package hello.proxy.config.v6_aop;
import hello.proxy.config.AppV1Config;
import hello.proxy.config.AppV2Config;
import hello.proxy.config.v6_aop.aspect.LogTraceAspect;
import hello.proxy.trace.logtrace.LogTrace;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Import;
@Configuration
@Import({AppV1Config.class, AppV2Config.class})
public class AopConfig {
 @Bean
 public LogTraceAspect logTraceAspect(LogTrace logTrace) {
 return new LogTraceAspect(logTrace);
 }
}
```
- `@Import({AppV1Config.class, AppV2Config.class})`: V1, V2 애플리케이션은 수동으로 스프링 빈으로 등록해야 동작한다.
- `@Bean logTraceAspect():` `@Aspect`가 있어도 스프링 빈으로 등록을 해줘얗 한다. 물론 `LogTraceAspect`에 `@Component` 애노테이션을 붙여서 컴포넌트 스캔을 상요해서 스프링 빈으로 등록해도 된다.

**ProxyApplication**
```java 
//@Import({AppV1Config.class, AppV2Config.class})
//@Import(InterfaceProxyConfig.class)
//@Import(ConcreteProxyConfig.class)
//@Import(DynamicProxyBasicConfig.class)
//@Import(DynamicProxyFilterConfig.class)
//@Import(ProxyFactoryConfigV1.class)
//@Import(ProxyFactoryConfigV2.class)
//@Import(BeanPostProcessorConfig.class)
//@Import(AutoProxyConfig.class)
//@Import(AutoProxyMultiAdvisorConfig.class)
@Import(AopConfig.class)
@SpringBootApplication(scanBasePackages = "hello.proxy.app")
public class ProxyApplication {
 public static void main(String[] args) {
 SpringApplication.run(ProxyApplication.class, args);
 }
 @Bean
 public LogTrace logTrace() {
 return new ThreadLocalLogTrace();
 }
}
```
- `AopConfig.class`를 등록하자.

**실행**
- http://localhost:8080/v1/request?itemId=hello
- http://localhost:8080/v2/request?itemId=hello
- http://localhost:8080/v3/request?itemId=hello

실행해보면 모두 프록시가 잘 적용된 것을 확인할 수 있다.

</details>

<details> <summary> 2. @Aspect 프록시 - 설명 </summary>

## 2. @Aspect 프록시 - 설명
앞서 자동 프록시 생성기를 학습할 때, 자동 프록시 생성기(`AnnotationAwareAspectJAutoProxyCreator`)는 `Advisor`를 자동으로 찾아와서 필요한 곳에 프록시를 생성하고 적용해준다고 했다.  
자동 프록시 생성기는 여기에 추가로 하나의 역할을 더 하는데, 바로 `@Aspect`를 찾아서 이것을 `Advisor`로 만들어준다.  
쉽게 이야기해서 지금까지 학습한 기능에 `@Aspect`를 `Advisor`로 변환해서 저장하는 기능도 한다.  
그래서 이름 앞에 `AnnotationAware`(애노테이션을 인식하는)가 붙어 있는 것이다.

![image](https://user-images.githubusercontent.com/28394879/141673848-ccfd272d-f907-4272-886c-3563f803c117.png)

**자동 프록시 생성기는 2가지 일을 한다**
1. `@Aspect`를 보고 어드바이저(`Advisor`)로 변환해서 저장한다.
2. 어드바이저를 기반으로 프록시를 생성한다.

### 1. `@Aspect`를 어드바이저로 변환해서 저장하는 과정 
![image](https://user-images.githubusercontent.com/28394879/141673876-b3bba163-501f-4f00-b355-c01152439283.png)

**@Aspect를 어드바이저로 변환해서 저장하는 과정을 알아보자**
1. 실행: 스프링 애플리케이션 로딩 시점에 자동 프록시 생성기를 호출한다.
2. 모든 @Aspect 빈 조회: 자동 프록시 생성기는 스프링 컨테이너에서 `@Aspect`에 애노테이션이 붙은 스프링 빈을 모두 조회한다.
3. 어드바이저 생성: `@Aspect` 어드바이저 빌더를 통해 `@Aspect` 애노테이션 정보를 기반으로 어드바이저를 생성한다.
4. @Aspect 기반 어드바이저 저장: 생성한 어드바이저를 `@Aspect` 어드바이저 빌더 내부에 저장한다.

**@Aspect 어드바이저 빌더**  
`BeanFactoryAspectJAdvisorsBuilder` 클래스이다.  
`@Aspect`의 정보를 기반으로 포인트컷, 어드바이스, 어드바이저를 생성하고 보관하는 것을 담당한다.  
`@Aspect`의 정보를 기반으로 어드바이저를 만들고, @Aspect 어드바이저 빌더 내부 저장소에 캐시한다.  
캐시에 어드바이저가 이미 만들어져 있는 경우 캐시에 저장된 어드바이저를 반환한다.


### 2. 어드바이저를 기반으로 프록시 생성 
![image](https://user-images.githubusercontent.com/28394879/141673967-5c77729f-1327-4b8c-86e6-d501de12cffd.png)

**자동 프록시 생성기의 작동 과정을 알아보자**
1. 생성: 스프링 빈 대상이 되는 객체를 생성한다. (`@Bean`, 컴포넌트 스캔 모두 포함)
2. 전달: 생성된 객체를 빈 저장소에 등록하기 직전에 빈 후처리기에 전달한다.  

3. Advisor 빈 조회 및  @Aspect Advisor 조회
   3-1. Advisor 빈 조회: 스프링 컨테이너에서 `Advisor`빈을 모두 조회한다.  
   3-2. @Aspect Advisor 조회: `@Aspect` 어드바이저 빌더 내부에 저장된 `Advisor`를 모두 조회한다.
4. 프록시 적용 대상 체크: 앞서 3-1, 3-2에서 조회한 `Advisor`에 포함되어 있는 포인트컷을 상요해서 해당 객체가 프록시를 적용할 대상인지 아닌지 판단한다. 이때 객체의 클래스 정보는 물론이고, 해당 객체의 모든 메서드를 포인트컷에 하나하나 모두 매칭해본다. 그래서 조건이 하나라도 만족하면 프록시 적용 대상이 된다. 예를 들어서 메서드 하나만 포인트컷 조건에 만족해도 프록시 적용 대상이 된다.
5. 프록시 생성: 프록시 적용 대상이면 프록시를 생성하고 프록시를 반환한다. 그래서 프록시를 스프링 빈으로 등록한다. 만약 프록시 적용 대상이 아니라면 원본 객체를 반환해서 원본 객체를 스프링 빈으로 등록한다.
6. 빈 등록: 반환된 객체는 스프링 빈으로 등록된다.

</details>

<details> <summary> 3. 정리 </summary>

## 3. 정리
`@Aspect`를 사용해서 애노테이션 기반 프록시를 매우 편리하게 적용해보았다. 실무에서 프록시를 적용할
때는 대부분이 이 방식을 사용한다.

지금까지 우리가 진행한 애플리케이션 전반에 로그를 남기는 기능은 특정 기능 하나에 관심이 있는 기능이
아니다. 애플리케이션의 여러 기능들 사이에 걸쳐서 들어가는 관심사이다.  
이것을 바로 횡단 **관심사(cross-cutting concerns)**라고 한다. 우리가 지금까지 진행한 방법이 이렇게
여러곳에 걸쳐 있는 횡단 관심사의 문제를 해결하는 방법이었다.  

![image](https://user-images.githubusercontent.com/28394879/141674332-b3f8c6f9-0e99-41a2-a72f-c14f8ac79b0e.png)

지금까지 프록시를 사용해서 이러한 횡단 관심사를 어떻게 해결하는지 점진적으로 매우 깊이있게 학습하고
기반을 다져두었다.  
이제 이 기반을 바탕으로 이러한 횡단 관심사를 전문으로 해결하는 스프링 AOP에 대해 본격적으로 알아보자

</details>


# [9. 스프링 AOP 개념](./9.spring-aop-concept)

<details> <summary> 1. AOP 소개 - 핵심 기능과 부가 기능 </summary>

## 1. AOP 소개 - 핵심 기능과 부가 기능

애플리케이션 로직은 크게 **핵심 기능**과 **부가 기능**으로 나눌 수 있다. 
- **핵심 기능**
  - 해당 객체가 제공하는 고유의 기능이다
  - 예) `OrderService`의 핵심 기능은 주문 로직이다.
- **부가 기능**
  - 핵심 기능을 보조하기 위해 제공되는 기능이다.
  - 예) 로그 추적 로직, 트랜잭션 기능이 있다.
  - 이러한 부가기능은 단독으로 사용되지 않고, 핵심 기능과 함꼐 사용된다.
  - 예) 로그 추적기 기능은 어떤 핵심 기능이 호출되었는지 로그를 남기기 위해서 사용된다.
  - 이름 그대로 핵심 기능을 보조하기 위해 존재한다.
  
![image](https://user-images.githubusercontent.com/28394879/141683706-c694c0f7-05b1-4c7d-be10-e2e34955d02e.png)

![image](https://user-images.githubusercontent.com/28394879/141683719-0f889b47-a3f9-4ad7-a6ae-e50ec592015c.png)

주문 로직을 실행하기 직전에 로그 추적 기능을 사용해야 하면, 핵심 기능인 주문 로직과 부가 기능인 로그 추적 로직이 하나의 객체 안에 섞여 들어가게 된다.  
부가 기능이 필요한 경우 이렇게 둘을 합해서 하나의 로직을 완성한다.  
이제 주문 서비스를 실행하면 핵심 기능인 주문 로직과 부가 기능인 로그 추적 로직이 함께 실행된다.

### 여러 곳에서 공통으로 사용하는 부가 기능
![image](https://user-images.githubusercontent.com/28394879/141683774-51effe62-baff-4fe3-a37e-d60e766025da.png)

보통 부가 기능은 여러 클래스에 걸쳐서 함께 사용된다. 예를 들어서 모든 애플리케이션 호출을 로깅해야 하는 요구사항을 생각해보자.  
이러한 부가 기능은 횡단 관심사(cross-cutting concerns)가 된다.  
쉽게 이야기해서 하나의 부가 기능이 여러 곳에 동일하게 사용된다는 뜻이다.

![image](https://user-images.githubusercontent.com/28394879/141683837-4d461eec-59d6-4dcc-8590-9ad60f695a57.png)

### 부가 기능 적용 문제
그런데 이런 부가 기능을 여러 곳에 적용하려면 너무 번거롭다.  
예를 들어서 부가 기능을 적용해야 하는 클래스가 100개면 100개 모두 동일한 코드를 추가해야 한다.  
부가 기능을 별도의 유틸리티 클래스로 만든다고 해도, 해당 유틸리티 클래스를 호출하는 코드가 결국 필요하다.  
그리고 부가 기능이 구조적으로 단순 호출이 아니라 `try-catch~finally` 같은 구조가 필요하다면 더욱 복잡해진다. (예: 실행 시간 측정)  
더 큰 문제는 수정이다.   
만약 부가 기능에 수정이 발생하면, 100개의 클래스 모두를 하나씩 찾아가면서 수정해야 한다.  
여기에 추가로 부가 기능이 적용되는 위치를 변경한다면 어떻게 될까?   
예를 들어서 부가 기능을 모든 컨트롤러, 서비스, 리포지토리에 적용했다가, 로그가 너무 많이 남아서 서비스 계층에만 적용한다고 수정해야하면 어떻게 될까? 또 수많은 코드를 고쳐야 할 것이다.

**부가 기능 적용 문제를 정리하면 다음과 같다.**  
- 부가 기능을 적용할 때 많은 반복이 필요하다.
- 부가 기능이 여러 곳에 퍼져서 중복 코드를 만들어낸다.
- 부가 기능을 변경할 때 중복 때문에 많은 수정이 필요하다.
- 부가 기능의 적용 대상을 변경할 때 많은 수정이 필요하다.

소프트웨어 개발에서 변경 지점은 하나가 될 수 있도록 잘 모듈화 되어야 한다.  
그런데 부가 기능처럼 특정 로직을 애플리케이션 전반에 적용하는 문제는 일반적인 OOP 방식으로는 해결이 어렵다. 


</details>

<details> <summary> 2. AOP 소개 - 애스팩트 </summary>

## 2. AOP 소개 - 애스팩트

### 핵심 기능과 부가 기능을 분리

누군가는 이러한 부가 기능 도입의 문제점을 해결하기 위해 오랜기간 고민해왔다.  
그 결과 부가 기능을 핵심 기능에서 분리하고 한 곳에서 관리하도록 했다.  
그리고 해당 부가 기능을 어디에 적용할지 선택하는 기능도 만들었다.  
이렇게 부가 기능과 부가 기능을 어디에 적용할지 선택하는 기능을 합해서 하나의 모듈로 만들었는데 이것이 바로 애스팩트(aspect)이다.  
애스팩트는 쉽게 이야기해서 부가 기능과, 해당 부가 기능을 어디에 적용할지 정의한 것이다.  
예를 들어서 로그 출력 기능을 모든 컨트롤러에 적용해라 라는 것이 정의되어 있다. 

그렇다 바로 우리가 이전에 알아본 `@Aspect`가 바로 그것이다.   
그리고 스프링이 제공하는 어드바이저도 어드바이스(부가 기능)과 포인트컷(적용 대상)을 가지고 있어서 개념상 하나의 애스펙트이다.

애스펙트는 우리말로 해석하면 관점이라는 뜻인데, 이름 그대로 애플리케이션을 바라보는 관점을 하나하나의 기능에서 횡단 관심사(cross-cutting concerns) 관점으로 달리 보는 것이다.  
이렇게 **애스펙트를 사용한 프로그래밍 방식을 관점 지향 프로그래밍 AOP(Aspect-Oriented Programming)**이라 한다.

**참고로 AOP는 OOP를 대체하기 위한 것이 아니라 횡단 관심사를 깔끔하게 처리하기 어려운 OOP의 부족한 부분을 보조하는 목적으로 개발되었다.**

![image](https://user-images.githubusercontent.com/28394879/141684734-a88b7d33-bd9c-4d97-ba4a-2130c30131c6.png)

**AspectJ 프레임워크**  
AOP의 대표적인 구현으로 AspectJ 프레임워크(https://www.eclipse.org/aspectj)가 있다.  
물론 스프링도 AOP를 지원하지만 대부분 AspectJ의 문법을 차용하고, AspectJ가 제공하는 기능의 일부만 제공한다.

AspectJ프레임워크는 스스로를 다음과 같이 설명한다.  
- 자바 프로그래밍 언어에 대한 완벽한 관점 지향 확장
- 횡단 관심사의 깔끔한 모듈화
  - 오류 검사 및 처리
  - 동기화
  - 성능 최적화(캐싱)
  - 모니터링 및 로깅 

</details>

<details> <summary> 3. AOP 적용 방식 </summary>

## 3. AOP 적용 방식

AOP를 사용하면 핵심 기능과 부가 기능이 코드상 완전히 분리되어서 관리된다.  
그렇다면 AOP를 사용할 때 부가 기능 로직은 어떤 방식으로 실제 롲기에 추가될 수 있을까?

**크게 3가지 방법이 있다.**
- 컴파일 시점
- 클래스 로딩 시점
- 런타임 시점(프록시)

**컴파일 시점**  
![image](https://user-images.githubusercontent.com/28394879/141685052-28c5798b-216a-46d8-ad80-73cbe3c4c49b.png)  
`.java` 소스 코드를 컴파일러를 사용해서 `.class`를 만드는 시점에 부가 기능 로직을 추가할 수 있다.  
이때는 AspectJ가 제공하는 특별한 컴파일러를 사용해야 한다. 컴파일 된 `.class`를 디컴파일 해보면 애스펙트 관련 호출 코드가 들어간다.  
이해하기 쉽게 풀어서 이야기하면 부가 기능 코드가 핵심 기능이 있는 컴파일된 코드 주변에 실제로 붙어 버린다고 생각하면 된다.  
AspectJ 컴파일러는 Aspect를 확인해서 해당 클래스가 적용 대상인지 먼저 확인하고, 적용 대상인 경우에 부가 기능 로직을 적용한다.  
참고로 이렇게 원본 로직에 부가 기능 로직이 추가되는 것을 위빙(Weaving)이라 한다.

- 위빙(Weaving): 옷감을 짜다. 직조하다. 애스펙트와 실제 코드를 연결해서 붙이는 것

**컴파일 시점 - 단점**  
컴파일 시점에 부가 기능을 적용하려면 특별한 컴파일러도 필요하고 복잡하다.

**클래스 로딩 시점**  
![image](https://user-images.githubusercontent.com/28394879/141685186-30b285a4-58df-42a8-9c21-fb80151f3fd6.png)

자바를 실행하면 자바 언어는 `.class` 파일을 JVM 내부의 클래스 로더에 보관하다.  
이때 중간에서 `.class`파일을 조작한 다음 JVM에 올릴 수 있다.  
자바 언어는 `.class`를 JVM에 저장하기 전에 조작할 수 있는 기능을 제공한다.  
궁금한 분은 java Instrumentation를 검색해보자.  
참고로 수 많은 모니터링 툴들이 이 방식을 사용한다.  
이 시점에 애스펙트를 적용하는 것을 로드 타임 위빙이라 한다.

**클래스 로딩 시점 - 단점**  
로드 타임 위빙은 자바를 실행할 때 특별한 옵션(`java -javaagent`)을 통해 클래스 로더 조작기를 지정해야 하는데, 이 부분이 번거롭고 운영하기 어렵다.

**런타임 시점**  
![image](https://user-images.githubusercontent.com/28394879/141685298-ae0aff4b-473d-429f-a1d3-ca874cb910f6.png)  
런타임 시점은 컴파일도 다 끝나고, 클래스 로더에 클래스도 다 올려가서 이미 자바가 실행되고 난 다음을 말한다.  
자바의 메인(`main`) 메서드가 이미 실행된 다음이다.  
따라서 자바 언어가 제공하는 범위 안에서 부가 기능을 적용해야 한다.  
스프링과 같은 컨테이너의 도움을 받고 프록시와 DI, 빈 포스트 프로세서 같은 개념들을 총 동원해야 한다.  
이렇게 하면 최종적으로 프록시를 통해 스프링 빈에 부가 기능을 적용할 수 있다.   
지금까지 우리가 학습한 것이 바로 프록시 방식의 AOP이다.  

프록시를 사용하기 때문에 AOP 기능에 일부 제약이 있다.  
하지만 특별한 컴파일러나, 자바를 실행할 떄 복잡한 옵션과 클래스 로더 조작기를 설정하지 않아도 된다.  
스프링만 있으면 얼마든지 AOP를 적용할 수 있다.

**부가 기능이 적용되는 차이를 정리하면 다음과 같다.**  
- 컴파일 시점: 실제 대상 코드에 애스팩트를 통한 부가 기능 호출 코드가 포함된다. AspectJ를 직접 사용해야 한다.
- 클래스 로딩 시점: 실제 대상 코드에 애스펙트를 통한 부가 기능 호출 코드가 포함된다. AspectJ를 직접 사용해야 한다.
- 런타임 시점: 실제 대상 코드는 그대로 유지된다. 대신에 프록시를 통해 부가 기능이 적용된다. 따라서 항상 프록시를 통해야 부가 기능을 사용할 수 있다. 스프링 AOP는 이 방식을 사용한다.

**AOP 적용 위치**  
AOP는 지금까지 학습한 메서드 실행 위치 뿐만 아니라 다음과 같은 다양한 위치에 적용할 수 있다.  
- 적용 가능 지점(조인 포인트): 생성자, 필드 값 접근, static 메서드 접근, 메서드 실행
  - 이렇게 AOP를 적용할 수 있는 지점을 조인 포인트(Join point)라 한다.
- AspectJ를 사용해서 컴파일 시점과 클래스 로딩 시점에 적용하는 AOP는 바이트코드를 실제 조작하기 떄문에 해당 기능을 모든 지점에 다 적용할 수 있다.
- 프록시 방식을 사용하는 스프링 AOP는 메서드 실행 시점에만 AOP를 적용할 수 있다.
  - 프록시는 메서드 오버라이딩 개념으로 동작한다. 따라서 생성자나 static 메서드, 필드 값 접근에는 프록시 개념이 적용될 수 없다.
  - 프록시를 사용하는 **스프링 AOP의 조인 포인트는 메서드 실행으로 제한**된다.
- 프록시 방식을 사용하는 스프링 AOP는 스프링 컨테이너가 관리할 수 있는 **스프링 빈에만 AOP를 적용**할 수 있다.

> 참고  
> 스프링은 AspectJ의 문법을 차용하고 프록시 방식의 AOP를 적용한다.  
> AspectJ를 직접 사용하느 것이 아니다.

> 중요  
> 스프링이 제공하는 AOP는 프록시를 사용한다.  
> 따라서 프록시를 통해 메서드를 실행하는 시점에만 AOP가 적용된다.  
> AspectJ를 사용하면 앞서 설명한 것 처럼 더 복잡하고 더 다양한 기능을 사용할 수 있다.  
> 그렇다면 스프링 AOP 보다는 더 기능이 많은 AspectJ를 직접 사용해서 AOP를 적용하는 것이 더 좋지 않을까?  
> AspectJ를 사용하려면 공부할 내용도 많고, 자바 관련 설정(특별한 컴파일러, AspectJ 전용 문법, 자바 실행 옵션)도 복잡하다.  
> 반면에 스프링 AOP는 별도의 추가 자바 설정 없이 스프링만 있으면 편리하게 AOP를 사용할 수 있다.  
> 실무에서는 스프링이 제공하는 AOP 기능만 사용해도 대부분의 문제를 해결할 수 있다.  
> 따라서 스프링 AOP가 제공하는 기능을 학습하는 것에 집중하자.

</details>



<details> <summary> 4. AOP 용어 정리 </summary>

## 4. AOP 용어 정리

![image](https://user-images.githubusercontent.com/28394879/141687046-c6fd917c-7d15-411b-9624-b6a1ac32c37c.png)

- 조인 포인트(Join Point)
  - 어드바이스가 적용될 수 있는 위치, 메소드 실행, 생성자 호출, 필드 값 접근, static 메서드 접근 같은 프로그램 실행 중 지점
  - 조인 포인트는 추상적인 개념이다. AOP를 적용할 수 있는 모든 지점이라 생각하면 된다.
  - 스프링 AOP는 프록시 방식을 사용하므로 조인 포인트는 항상 메소드 실행 지점으로 제한된다.
- 포인트컷(Pointcut)
  - 조인 포인트 중에서 어드바이스가 적용될 위치를 선별하는 기능
  - 주로 AspectJ 표현식을 사용해서 지정
  - 프록시를 사용하는 스프링 AOP는 메서드 실행 지점만 포인트컷으로 선별 가능
- 타겟(Target)
  - 어드바이스를 받는 객체, 포인트컷으로 결정
- 어드바이스(Advice)
  - 부가 기능
  - 특정 조인 포인트에서 Aspect에 의해 취해지는 조치
  - Around(주변), Before(전), After(후)와 같은 다양한 종류의 어드바이스가 있음
- 애스펙트(Aspect)
  - 어드바이스 + 포인트컷을 모듈화 한 것
  - `@Aspect`를 생각하면 됨
  - 여러 어드바이스와 포인트 컷이 함께 존재
- 어드바이저(Advisor)
  - 하나의 어드바이스와 하나의 포인트 컷으로 구성
  - 스프링 AOP에서만 사용되는 특별한 용어
- 위빙(Weaving)
  - 포인트컷으로 결정한 타겟의 조인 포인트에 어드바이스를 적용하는 것
  - 위빙을 통해 핵심 기능 코드에 영향을 주지 않고 부가 기능을 추가할 수 있음
  - AOP 적용을 위해 애스펙트를 객체에 연결한 상태
    - 컴파일 타임(AspectJ compiler)
    - 로드 타임
    - 런타임, 스프링 AOP는 런타임, 프록시 방식
- AOP 프록시
  - AOP 기능을 구현하기 위해 만든 프록시 객체, 스프링에서 AOP 프록시는 JDK 동적 프록시 또는 CGLIB 프록시이다.



</details>


# [10. 스프링 AOP 구현](./10.spring-aop-implementation)

<details> <summary> 1. 프로젝트 생성 </summary>

## 1. 프로젝트 생성

이번에는 스프링 웹 기술은 사용하지 않는다.   
Lombok만 추가하면 된다.  
참고로 스프링 프레임워크의 핵심 모듈들은 별도의 설정이 없어도 자동으로 추가 된다.  
추가로 AOP 기능을 사용하기 위해서 다음을 `build.gradle`에 직접 추가하자.

```
implementation 'org.springframework.boot:spring-boot-starter-aop'
```

`build.gradle`에 테스트 코드에서도 lombok을 사용할 수 있도록 다음 코드를 추가하자.
```
//테스트에서 lombok 사용
testCompileOnly 'org.projectlombok:lombok'
testAnnotationProcessor 'org.projectlombok:lombok'
```

**참고**  
`@Aspect`를 사용하려면 `@EnableAspectJAutoProxy`를 스프링 설정에 추가해야 하지만, 스프링 부트를 사용하면 자동으로 추가된다.

**build.gradle**
```
plugins {
id 'org.springframework.boot' version '2.5.5'
id 'io.spring.dependency-management' version '1.0.11.RELEASE'
id 'java'
}
group = 'hello'
version = '0.0.1-SNAPSHOT'
sourceCompatibility = '11'
configurations {
compileOnly {
extendsFrom annotationProcessor
}
}
repositories {
mavenCentral()
}
dependencies {
implementation 'org.springframework.boot:spring-boot-starter'
implementation 'org.springframework.boot:spring-boot-starter-aop' //직접 추가
compileOnly 'org.projectlombok:lombok'
annotationProcessor 'org.projectlombok:lombok'
testImplementation 'org.springframework.boot:spring-boot-starter-test'
 //테스트에서 lombok 사용
 testCompileOnly 'org.projectlombok:lombok'
 testAnnotationProcessor 'org.projectlombok:lombok'
}
test {
useJUnitPlatform()
}
```

- 동작 확인
  - 기본 메인 클래스 실행(`AopApplication.main()`)
  - 스프링 부트 실행 로그가 나오면 성공(스프링 웹 프로젝트를 추가하지 않아서 서버가 실행되지는 않는다.)

</details>

<details> <summary> 2. 예제 프로젝트 만들기 </summary>

## 2. 예제 프로젝트 만들기

AOP를 적용할 예제 프로젝트틀 만들어보자. 지금까지 학습 했던 내용과 비슷해서 큰 어려움은 없을 것이다.


**OrderRepository**  
```java
package hello.aop.order;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Repository;
@Slf4j
@Repository
public class OrderRepository {
 public String save(String itemId) {
 log.info("[orderRepository] 실행");
 //저장 로직
 if (itemId.equals("ex")) {
 throw new IllegalStateException("예외 발생!");
 }
 return "ok";
 }
}
```

**OrderService**  
```java
package hello.aop.order;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;
@Slf4j
@Service
public class OrderService {
 private final OrderRepository orderRepository;
 public OrderService(OrderRepository orderRepository) {
 this.orderRepository = orderRepository;
 }
 public void orderItem(String itemId) {
 log.info("[orderService] 실행");
 orderRepository.save(itemId);
 }
}
```

**AopTest**
```java
package hello.aop;
import hello.aop.order.OrderRepository;
import hello.aop.order.OrderService;
import lombok.extern.slf4j.Slf4j;
import org.junit.jupiter.api.Test;
import org.springframework.aop.support.AopUtils;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;
@Slf4j
@SpringBootTest
public class AopTest {
 @Autowired
 OrderService orderService;
 @Autowired
 OrderRepository orderRepository;
 @Test
 void aopInfo() {
 log.info("isAopProxy, orderService={}",
AopUtils.isAopProxy(orderService));
 log.info("isAopProxy, orderRepository={}",
AopUtils.isAopProxy(orderRepository));
 }
 @Test
 void success() {
 orderService.orderItem("itemA");
 }
 @Test
 void exception() {
 assertThatThrownBy(() -> orderService.orderItem("ex"))
 .isInstanceOf(IllegalStateException.class);
 }
}
```

`AopUtils.isAopProxy(...)`을 통해서 AOP 프록시가 적용 되었는지 확인할 수 있다.   
현재 AOP 관련 코드를 작성하지 않았으므로 프록시가 적용되지 않고, 결과도 `false`를 반환해야 정상이다.

여기서는 실제 결과를 검증하는 테스트가 아니라 학습 테스트를 진행한다.  
앞으로 로그를 직접 보면서 AOP가 잘 작동하는지 확인해볼 것디ㅏ.  
테스트를 실행해서 잘 동작하면 다음으로 넘어가자.

**실행 - success()**
```
[orderService] 실행
[orderRepository] 실행
```

![image](https://user-images.githubusercontent.com/28394879/141722818-c199401e-9805-4c8e-860d-27f37ac27c1c.png)


</details>

<details> <summary> 3. 스프링 AOP 구현1 - 시작 </summary>

## 3. 스프링 AOP 구현1 - 시작 

스프링 AOP를 구현하는 일반적인 방법은 앞서 학습한 `@Aspect`를 사용하는 방법이다.  
이번 시간에는 `@Aspect`를 사용해서 가장 단순한 AOP를 구현해보자.

**AspectV1**
```java
package hello.aop.order.aop;
import lombok.extern.slf4j.Slf4j;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
@Slf4j
@Aspect
public class AspectV1 {
 //hello.aop.order 패키지와 하위 패키지
 @Around("execution(* hello.aop.order..*(..))")
 public Object doLog(ProceedingJoinPoint joinPoint) throws Throwable {
 log.info("[log] {}", joinPoint.getSignature()); //join point 시그니처
 return joinPoint.proceed();
 }
}
```
- `@Around` 애노테이션의 값인 `execution(* hello.aop.order..*(..))`는 포인트컷이 된다.
- `@Around` 애노테이션의 메서드인 `doLog`는 어드바이스(`Advice`)가 된다.
- `execution(* hello.aop.order..*(..))`는 `hello.aop.order` 패키지와 그 하위 패키지(`..`)를 지정하는 AspectJ 포인트컷 표현식이다. 앞으로는 간단히 포인트컷 표현식이라 하겠다. 참고로 포인트컷 표현식은 뒤에서 자세히 정리하겠다.
- 이제 `OrderService`, `OrderRepository`의 모든 메서드는 AOP 적용의 대상이 된다. 참고로 스프링은 프록시 방식의 AOP를 사용하므로 프록시를 통하는 메서드만 적용 대상이 된다.

**참고**  
> 스프링 AOP는 AspectJ의 문법을 차용하고, 프록시 방식의 AOP를 제공한다. AspectJ를 직접 사용하는 것이 아니다.  
> 스프링 AOP를 사용할 때는 `@Aspect` 애노테이션을 주로 사용하는데, 이 애노테이션도 AspectJ가 제공하는 애노테이션이ㅏㄷ.

**참고**  
> `@Aspect`를 포함한 `org.aspectj` 패키지 관련 기능은 `aspectweaver.jar` 라이브러리가 제공하는 기능이다.  
> 앞서 `build.gradle`에 `spring-boot-starter-aop`를 포함했는데, 이렇게 하면 스프링의 AOP 관련 기능과 함께 `aspectjweaver.jar`도 함께 사용할 수 있게 의존 관계에 포함된다.  
> 그런데 스프링에서는 AspectJ가 제공하는 애노테이션이나 관련 인터페이스만 사용하는 것이고, 실제 AspectJ가 제공하는 컴파일, 로드타임 위버 등을 사용하는 것은 아니다.   
> 스프링은 지금까지 우리가 학습한 것 처럼 프록시 방식의 AOP를 사용한다.


**AopTest - 추가**
```java
package hello.aop;
import hello.aop.order.OrderRepository;
import hello.aop.order.OrderService;
import hello.aop.order.aop.AspectV1;
import org.junit.jupiter.api.Test;
import org.springframework.aop.support.AopUtils;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.context.annotation.Import;
import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;
@Import(AspectV1.class) //추가
@SpringBootTest
public class AopTest {
 @Autowired
 OrderService orderService;
 @Autowired
 OrderRepository orderRepository;
 @Test
 void aopInfo() {
 System.out.println("isAopProxy, orderService=" +
AopUtils.isAopProxy(orderService));
 System.out.println("isAopProxy, orderRepository=" +
AopUtils.isAopProxy(orderRepository));
 }
 @Test
 void success() {
 orderService.orderItem("itemA");
 }
 @Test
 void exception() {
 assertThatThrownBy(() -> orderService.orderItem("ex"))
 .isInstanceOf(IllegalStateException.class);
 }
}
```
`@Aspect`는 애스펙트라는 표식이지 컴포넌트 스캔이 되는 것은 아니다.   
따라서 `AspectV1`를 AOP로 사용하려면 스프링 빈으로 등록해야 한다.

스프링 빈으로 등록하는 방법은 다음과 같다.
- `@Bean`을 사용해서 직접 등록
- `@Component` 컴포넌트 스캔을 사용해서 자동 등록
- `@Import` 주로 설정 파일을 추가할 때 사용(`@Configuration`)

`@Import`는 주로 설정 파일을 추가할 때 사용하지만, 이 기능으로 스프링 빈도 등록할 수 있다.  
테스트에서는 버전을 올려가면서 변경할 예정이어서 간단하게 `@Import` 기능을 사용하자.

`AopTest`에 `@Import(AspectV1.class)`로 스프링 빈을 추가했다.

`AopUtils.isAopProxy(...)`도 프록시가 적용되었으므로 `true`를 반환한다.

**실행 - success()**  
테스트를 실행해보면 다음과 같이 로그가 잘 출력되는 것을 확인할 수 있다.
```
[log] void hello.aop.order.OrderService.orderItem(String)
[orderService] 실행
[log] String hello.aop.order.OrderRepository.save(String)
[orderRepository] 실행
```

![image](https://user-images.githubusercontent.com/28394879/141723846-9a3ee261-feb5-41dc-8503-479ecedfd19c.png)


</details>

<details> <summary> 4. 스프링 AOP 구현2 - 포인트컷 분리 </summary>

## 4. 스프링 AOP 구현2 - 포인트컷 분리

`@Around`에 포인트컷 표현식을 직접 넣을 수도 있지만, `@Pointcut` 애노테이션을 사용해서 별도로 분리할 수도 있따.

**AspectV2**
package hello.aop.order.aop;
import lombok.extern.slf4j.Slf4j;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Pointcut;
@Slf4j
@Aspect
public class AspectV2 {
 //hello.aop.order 패키지와 하위 패키지
 @Pointcut("execution(* hello.aop.order..*(..))") //pointcut expression
 private void allOrder(){} //pointcut signature
 @Around("allOrder()")
 public Object doLog(ProceedingJoinPoint joinPoint) throws Throwable {
 log.info("[log] {}", joinPoint.getSignature());
 return joinPoint.proceed();
 }
}

**@Pointcut**  
- `@Pointcut`에 포인트컷 표현식을 사용한다.
- 메서드 이름과 파라미터를 합쳐서 포인트컷 시그니처(signature)라 한다.
- 메서드의 반환 타입은 `void`여야 한다.
- 코드 내용은 비워둔다.
- 포인트컷 시그니처는 `allOrder()`이다. 이름 그대로 주문과 관련된 모든 기능을 대상으로 하는 포인트컷이다.
- `@Around` 어드바이스에서는 포인트컷을 직접 지정해도 되지만, 포인트컷 시그니처를 사용해도 된다. 여기서는 `@Around("allOrder()")`를 사용한다.
- `private`, `public` 같은 접근 제어자는 내부에서만 사용하면 `private`을 사용해도 되지만, 다른 애스펙트에서 참고하려면 `public`을 사용해야 한다.

결과적으로 `AspectV1`과 같은 기능을 수행한다. 이렇게 분리하면 하나의 포인트컷 표현식을 여러 어드바이스에서 함께 사용할 수 있다.  
그리고 뒤에 설명하겠지만 다른 클래스에 있는 외부 어드바이스에서도 포인트컷을 함께 사용할 수 있다.

**AopTest - 수정**
```java
//@Import(AspectV1.class)
@Import(AspectV2.class)
@SpringBootTest
public class AopTest {
}
```

`AspectV2`를 실행하기 위해서 다음 처리를 하자.
- `@Import(AspectV1.class)` 주석 처리
- `@Import(AspectV2.class)` 추가

실행해보면 이전과 동일하게 동작하는 것을 확인할 수 있다.

**실행 - success()**  
테스트를 실행해보면 다음과 같이 로그가 잘 출력되는 것을 확인할 수 있다.
```
[log] void hello.aop.order.OrderService.orderItem(String)
[orderService] 실행
[log] String hello.aop.order.OrderRepository.save(String)
[orderRepository] 실행
```


</details>

<details> <summary> 5. 스프링 AOP 구현3 - 어드바이스 추가 </summary>

## 5. 스프링 AOP 구현3 - 어드바이스 추가

이번에는 조금 복잡한 예제를 만들어보자.  
앞서 로그를 출력하는 기능에 추가로 트랜잭션을 적용하는 코드도 추가해보자.   
여기서는 진짜 트랜잭션을 실행하는 것은 아니다.  
기능이 동작한 것 처럼 로그만 남기겠다.

트랜잭션 기능은 보통 다음과 같이 동작한다.
- 핵심 로직 실행 직전에 트랜잭션을 시작
- 핵심 로직 실행
- 핵심 로직 실행에 문제가 없으면 커밋
- 핵심 로직 실행에 예외가 발생하면 롤백

**AspectV3**
```java
package hello.aop.order.aop;
import lombok.extern.slf4j.Slf4j;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Pointcut;
@Slf4j
@Aspect
public class AspectV3 {
 //hello.aop.order 패키지와 하위 패키지
 @Pointcut("execution(* hello.aop.order..*(..))")
 public void allOrder(){}
 //클래스 이름 패턴이 *Service
 @Pointcut("execution(* *..*Service.*(..))")
 private void allService(){}
 @Around("allOrder()")
 public Object doLog(ProceedingJoinPoint joinPoint) throws Throwable {
 log.info("[log] {}", joinPoint.getSignature());
 return joinPoint.proceed();
 }
 //hello.aop.order 패키지와 하위 패키지 이면서 클래스 이름 패턴이 *Service
 @Around("allOrder() && allService()")
 public Object doTransaction(ProceedingJoinPoint joinPoint) throws Throwable
{
 try {
 log.info("[트랜잭션 시작] {}", joinPoint.getSignature());
 Object result = joinPoint.proceed();
 log.info("[트랜잭션 커밋] {}", joinPoint.getSignature());
 return result;
 } catch (Exception e) {
 log.info("[트랜잭션 롤백] {}", joinPoint.getSignature());
 throw e;
 } finally {
 log.info("[리소스 릴리즈] {}", joinPoint.getSignature());
 }
 }
}
```
- `allOrder()` 포인트컷은 `hello.aop.order` 패키지와 하위 패키지를 대상으로 한다.
- `allService()` 포인트컷 타입 이름 패턴이 `*Service`를 대상으로 하는데 쉽게 이야기해서 `XxxService`처럼 `Service`로 끝나는 것을 대상으로 한다. `*Servi*`과 같은 패턴도 가능하다.
- 여기서 타입 이름 패턴이라고 한 이유는 클래스, 인터페이스에 모두 적용되기 때문이다.

`@Around("allOrder() && allService()")`
- 포인트컷은 이렇게 조합할 수 있다. `&&`(AND), `||`(OR), `!`(NOT) 3가지 조합이 가능하다.
- `hello.aop.order` 패키지와 하위 패키지 이면서 타입 이름 패턴이 `*Service`인 것을 대상으로 한다.
- 결과적으로 `doTransaction()` 어드바이스는 `OrderService`에만 적용된다.
- `doLog()` 어드바이스는 `OrderService`, `OrderRepository`에 모두 적용된다.

**포인트컷이 적용된 AOP 결과를 다음과 같다.**
- `orderService`: `doLog()`, `doTransaction()` 어드바이스 적용
- `orderRepository`: `doLog()` 어드바이스 적용

**AopTest - 수정**
```java
//@Import(AspectV1.class)
//@Import(AspectV2.class)
@Import(AspectV3.class)
@SpringBootTest
public class AopTest {
}
```

**실행 - success()**
```
[log] void hello.aop.order.OrderService.orderItem(String)
[트랜잭션 시작] void hello.aop.order.OrderService.orderItem(String)
[orderService] 실행
[log] String hello.aop.order.OrderRepository.save(String)
[orderRepository] 실행
[트랜잭션 커밋] void hello.aop.order.OrderService.orderItem(String)
[리소스 릴리즈] void hello.aop.order.OrderService.orderItem(String)
```

![image](https://user-images.githubusercontent.com/28394879/141802046-07d58ead-0001-4f59-bc48-7e2e61125b78.png)


전체 실행 순서를 분석해보자.

**AOP 적용 전**  
클라이언트 -> `orderService.orderItem()` -> `orderRepository.save()`

**AOP 적용 후**  
- 클라이언트 -> [`doLog()` -> `doTransaction()`] -> `orderService.orderItem()`
- -> [`doLog()`] -> `orderRepository.save()`

`orderService`에는 `doLog()`, `doTransaction()` 두가지 어드바이스가 적용되어 있고, `orderRepository`에는 `doLog()` 하나의 어드바이스만 적용된 것을 확인할 수 있다.

**실행 - exception()**
```
[log] void hello.aop.order.OrderService.orderItem(String)
[트랜잭션 시작] void hello.aop.order.OrderService.orderItem(String)
[orderService] 실행
[log] String hello.aop.order.OrderRepository.save(String)
[orderRepository] 실행
[트랜잭션 롤백] void hello.aop.order.OrderService.orderItem(String)
[리소스 릴리즈] void hello.aop.order.OrderService.orderItem(String)
```
예외 상황에서는 트랜잭션 커밋 대신에 트랜잭션 롤백이 호출되는 것을 확인할 수 있다.

그런데 여기에서 로그를 남기는 순서가 [`doLog()` -> `doTransaction()`] 순서로 작동한다.  
만약 어드바이스가 적용되는 순서를 변경하고 싶으면 어떻게 하면 될까?   
예를 들어서 실행 시간을 측정해야 하는데 트랜잭션과 관련된 시간을 제외하고 측정하고 싶다면 [`doTransaction()` -> `doLog()`] 이렇게 트랜잭션 이후에 로그를 남겨야 할 것이다.

그 전에 잠깐 포인트컷을 외부로 빼서 사용하는 방법을 먼저 알아보자.



</details>

<details> <summary> 6. 스프링 AOP 구현4 - 포인트컷 참조 </summary>

## 6. 스프링 AOP 구현4 - 포인트컷 참조

다음과 같이 포인트컷을 공용으로 사용하기 위해 별도의 외부 클래스에 모아두어도 된다.  
참고로 외부에서 호출할 때는 포인트컷의 접근 제어자를 `public`으로 열어두어야 한다.

**Pointcuts**
```java
package hello.aop.order.aop;
import org.aspectj.lang.annotation.Pointcut;
public class Pointcuts {
 //hello.springaop.app 패키지와 하위 패키지
 @Pointcut("execution(* hello.aop.order..*(..))")
 public void allOrder(){}
 //타입 패턴이 *Service
 @Pointcut("execution(* *..*Service.*(..))")
 public void allService(){}
 //allOrder && allService
 @Pointcut("allOrder() && allService()")
 public void orderAndService(){}
}
```

`orderAndService()`: `allOrder()` 포인트컷과 `allService()` 포인트컷을 조합해서 새로운 포인트컷을 만들었다.

**AspectV4Pointcut**
```java
package hello.aop.order.aop;
import lombok.extern.slf4j.Slf4j;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
@Slf4j
@Aspect
public class AspectV4Pointcut {
 @Around("hello.aop.order.aop.Pointcuts.allOrder()")
 public Object doLog(ProceedingJoinPoint joinPoint) throws Throwable {
 log.info("[log] {}", joinPoint.getSignature());
 return joinPoint.proceed();
 }
 @Around("hello.aop.order.aop.Pointcuts.orderAndService()")
 public Object doTransaction(ProceedingJoinPoint joinPoint) throws Throwable
{
 try {
 log.info("[트랜잭션 시작] {}", joinPoint.getSignature());
 Object result = joinPoint.proceed();
 log.info("[트랜잭션 커밋] {}", joinPoint.getSignature());
 return result;
 } catch (Exception e) {
 log.info("[트랜잭션 롤백] {}", joinPoint.getSignature());
 throw e;
 } finally {
 log.info("[리소스 릴리즈] {}", joinPoint.getSignature());
 }
 }
}
```

사용하는 방법은 패키지명을 포함한 클래스 이름과 포인트컷 시그니처를 모두 지정하면 된다.  
포인트컷을 여러 어드바이스에서 함께 사용할 때 이 방법을 사용하면 효과적이다.

**AopTest - 수정**
```java
//@Import(AspectV1.class)
//@Import(AspectV2.class)
//@Import(AspectV3.class)
@Import(AspectV4Pointcut.class)
@SpringBootTest
public class AopTest {
}
```

`AspectV4Pointcut.class`를 실행하기 위해서 다음 처리를 하자.
- `@Import(AspectV3.class)` 주석 처리
- `@Import(AspectV4Pointcut.class)` 추가

**실행**  
기존과 결과는 같다. 

</details>

<details> <summary> 7. 스프링 AOP 구현5 - 어드바이스 순서 </summary>

## 7. 스프링 AOP 구현5 - 어드바이스 순서

어드바이스는 기본적으로 순서를 보장하지 않는다.  
순서를 지정하고 싶으면 `@Aspect` 적용 단위로 `org.springframework.core.annotation.@Order` 애노테이션을 적용해야 한다.  
문제는 이것을 어드바이스 단위가 아니라 클래스 단위로 적용할 수 있다는 점이다.  
그래서 지금처럼 하나의 애스펙트에 여러 어드바이스가 있으면 순서를 보장 받을 수 없다.  
따라서 **애스펙트를 별도의 클래스로 분리**해야 한다.

현재 로그를 남기는 순서가 아마도 [`doLog()` -> `doTransaction()`] 이 순서로 남을 것이다.  
(참고로 이 순서로 실행되지 않는 사람도 있을 수 있다. JVM이나 실행 환경에 따라 달라질 수도 있다.)

로그를 남기는 순서를 바꾸어서 [`doTransaction()` -> `doLog()`] 트랜잭션이 먼저 처리되고, 이후에 로그가 남도록 변경해보자.

**AspectV5Order**
```java
package hello.aop.order.aop;
import lombok.extern.slf4j.Slf4j;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.springframework.core.annotation.Order;
@Slf4j
public class AspectV5Order {
 @Aspect
 @Order(2)
 public static class LogAspect {
 @Around("hello.aop.order.aop.Pointcuts.allOrder()")
 public Object doLog(ProceedingJoinPoint joinPoint) throws Throwable {
 log.info("[log] {}", joinPoint.getSignature());
 return joinPoint.proceed();
 }
 }
 @Aspect
 @Order(1)
 public static class TxAspect {
 @Around("hello.aop.order.aop.Pointcuts.orderAndService()")
 public Object doTransaction(ProceedingJoinPoint joinPoint) throws
Throwable {
 try {
 log.info("[트랜잭션 시작] {}", joinPoint.getSignature());
 Object result = joinPoint.proceed();
 log.info("[트랜잭션 커밋] {}", joinPoint.getSignature());
 return result;
 } catch (Exception e) {
 log.info("[트랜잭션 롤백] {}", joinPoint.getSignature());
 throw e;
 } finally {
 log.info("[리소스 릴리즈] {}", joinPoint.getSignature());
 }
 }
 }
}
```
하나의 애스펙트 안에 있던 어드바이스를 `LogAspect`, `TxAspect` 애스펙트로 각각 분리했다.  
그리고 각 애스펙트에 `@Order` 애노테이션을 통해 실행 순서를 적용했다.  
참고로 숫자가 작을 수록 먼저 실행된다.

**AopTest - 변경**
```java
//@Import(AspectV4Pointcut.class)
@Import({AspectV5Order.LogAspect.class, AspectV5Order.TxAspect.class})
@SpringBootTest
public class AopTest {
}
```
`AspectV5Order`를 실행하기 위해서 다음 처리를 하자.
- `@Import(AspectV4Pointcut.class)` 주석 처리
- `@Import({AspectV5Order.LogAspect.class, AspectV5Order.TxAspect.class})` 추가

**실행**   
실행 결과를 보면 트랜잭션 어드바이스가 먼저 실행되는 것을 확인할 수 있다.
```
[트랜잭션 시작] void hello.aop.order.OrderService.orderItem(String)
[log] void hello.aop.order.OrderService.orderItem(String)
[orderService] 실행
[log] String hello.aop.order.OrderRepository.save(String)
[orderRepository] 실행
[트랜잭션 커밋] void hello.aop.order.OrderService.orderItem(String)
[리소스 릴리즈] void hello.aop.order.OrderService.orderItem(String)
```

![image](https://user-images.githubusercontent.com/28394879/141806858-f696591d-efac-480f-afec-84b611853927.png)

</details>

<details> <summary> 8. 스프링 AOP 구현6 - 어드바이스 종류 </summary>

## 8. 스프링 AOP 구현6 - 어드바이스 종류

어드바이스는 앞서 살펴본 `@Around` 외에도 여러가지 종류가 있다.

**어드바이스 종류**
- `@Around`: 메서드 호출 전후에 수행, 가장 강력한 어드바이스, 조인 포인트 실행 여부 선택, 반환 값 변환, 예외 반환 등이 가능
- `@Before`: 조인 포인트 실행 이전에 실행
- `@AfterReturning`:조인 포인트가 정상 완료후 실행
- `AfterThrowing`: 메서드가 예외를 던지는 경우 실행
- `@After`: 조인 포인트가 정상 또는 예외에 관계없이 실행(finally)

예제를 만들면서 학습해보자.

**AspectV6Advice**
```java
package hello.aop.order.aop;
import lombok.extern.slf4j.Slf4j;
import org.aspectj.lang.JoinPoint;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.*;
@Slf4j
@Aspect
public class AspectV6Advice {
 @Around("hello.aop.order.aop.Pointcuts.orderAndService()")
 public Object doTransaction(ProceedingJoinPoint joinPoint) throws Throwable
{
 try {
 //@Before
 log.info("[around][트랜잭션 시작] {}", joinPoint.getSignature());
 Object result = joinPoint.proceed();
 //@AfterReturning
 log.info("[around][트랜잭션 커밋] {}", joinPoint.getSignature());
 return result;
 } catch (Exception e) {
 //@AfterThrowing
 log.info("[around][트랜잭션 롤백] {}", joinPoint.getSignature());
 throw e;
 } finally {
 //@After
 log.info("[around][리소스 릴리즈] {}", joinPoint.getSignature());
 }
 }
 @Before("hello.aop.order.aop.Pointcuts.orderAndService()")
 public void doBefore(JoinPoint joinPoint) {
 log.info("[before] {}", joinPoint.getSignature());
 }
 @AfterReturning(value = "hello.aop.order.aop.Pointcuts.orderAndService()",
returning = "result")
 public void doReturn(JoinPoint joinPoint, Object result) {
 log.info("[return] {} return={}", joinPoint.getSignature(), result);
 }
 @AfterThrowing(value = "hello.aop.order.aop.Pointcuts.orderAndService()",
throwing = "ex")
 public void doThrowing(JoinPoint joinPoint, Exception ex) {
 log.info("[ex] {} message={}", joinPoint.getSignature(),
ex.getMessage());
 }
 @After(value = "hello.aop.order.aop.Pointcuts.orderAndService()")
 public void doAfter(JoinPoint joinPoint) {
 log.info("[after] {}", joinPoint.getSignature());
 }
}
```

`doTransaction()` 메서드에 남겨둔 주석을 보자.  
복잡해 보이지만 사실 `@Around`를 제외한 나머지 어드바이스들은 `@Around`가 할 수 있는 일의 일부만 제공할 뿐이다.  
따라서 `@Around` 어드바이스만 사용해도 필요한 기능을 모두 수행할 수 있다.

### 참고 정보 획득
모든 어드바이스는 `org.aspectj.lang.JoinPoint`를 첫번째 파라미터에 사용할 수 있다. (생략해도 된다.)
단 `@Around`는 `ProceedingJoinPoint`를 사용해야 한다.

참고로 `ProceedingJoinPoint`는 `org.aspectj.lang.JoinPoint`의 하위 타입이다.

**JoinPoint 인터페이스의 주요 기능**
- `getArgs()`: 메서드 인수를 반환한다.
- `getThis()`: 프록시 객체를 반환한다.
- `getTarget()`: 대상 객체를 반환한다.
- `getSignature()`: 조언되는 메서드에 대한 설명을 반환한다.
- `toString()`: 조언되는 방법에 대한 유용한 설명을 인쇄한다.

**ProceedingJoinPoint 인터페이스의 주요 기능**
- `proceed()`: 다음 어드바이스나 타겟을 호출한다.

추가로 호출시 전달한 매개변수를 파라미터를 통해서도 전달 받을 수도 있는데, 이 부분은 뒤에서 정리하겠다.

### 어드바이스 종류
**@Before**  
조인 포인트 실행 전 
```java
@Before("hello.aop.order.aop.Pointcuts.orderAndService()")
public void doBefore(JoinPoint joinPoint) {
 log.info("[before] {}", joinPoint.getSignature());
}
```
`@Around`와 다르게 작업 흐름을 변경할 수는 없다.  
`@Around`는 `ProceedingJoinPoint.proceed()`를 호출해야 다음 대상이 호출된다.  
만약 호출하지 않으면 다음 대상이 호출되지 않는다.   
반면에 `@Before`는 `ProceedingJoinPoint.proceed()` 자체를 사용하지 않는다.  
메서드 종료시 자동으로 다음 타겟이 호출된다.    
물론 예외가 발생하면 다음 코드가 호출되지는 않는다.

**AfterReturning**  
메서드 실행이 정상적으로 반환될 때 실행
```java
@AfterReturning(value = "hello.aop.order.aop.Pointcuts.orderAndService()",
returning = "result")
public void doReturn(JoinPoint joinPoint, Object result) {
 log.info("[return] {} return={}", joinPoint.getSignature(), result);
}
```
- `returning`속성에 사용된 이름은 어드바이스 메서드의 매개변수 이름과 일치해야 한다.
- `returning`절에 지정된 타입의 값을 반환하는 메서드만 대상으로 실행한다. (부모 타입을 지정하면 모든 자식 타입은 인정된다.)
- `@Around`와 다르게 반환되는 객체를 변경할 수는 없다. 반환 객체를 변경하려면 `@Around`를 사용해야 한다. 참고로 반환 객체를 조작할 수는 있다.

**AfterThrowing**  
메서드 실행이 예외를 던져서 종료될 때 실행  
```java
@AfterThrowing(value = "hello.aop.order.aop.Pointcuts.orderAndService()",
throwing = "ex")
public void doThrowing(JoinPoint joinPoint, Exception ex) {
 log.info("[ex] {} message={}", joinPoint.getSignature(), ex.getMessage());
}
```
- `throwing`속성에 사용된 이름은 어드바이스 메서드의 매개변수 이름과 일치해야 한다.
- `throwing`절에 지정된 타입과 맞은 예외를 대상으로 실행한다. (부모 타입을 지정하면 모든 자식 타입은 인정된다.)

**@After**
- 메서드 실행이 종료되면 실행된다. (finally를 생각하면 된다.)
- 정상 및 예외 반환 조건을 모두 처리한다.
- 일반적으로 리소스를 해제하는데 사용한다.

**@Around**
- 메서드의 실행의 주변에서 실행된다. 메서드 실행 전후에 작업을 수행한다.
- 가장 강력한 어드바이스
  - 조인 포인트 실행 여부 선택 `joinPoint.proceed() 호출 여부 선택`
  - 전달 값 변환: `joinPoint.proceed(args[])`
  - 반환 값 변환
  - 예외 변환
  - 트랜잭션 처럼 `try ~ catch~ finally` 모두 들어가는 구문 처리 가능
- 어드바이스의 첫 번째 파라미터는 `ProceedingJoinPoint`를 사용해야 한다.
- `proceed()`를 통해 대상을 실행한다.
- `proceed()`를 여러번 실행할 수도 있음(재시도)

**AopTest - 변경**
```java
//@Import({AspectV5Order.LogAspect.class, AspectV5Order.TxAspect.class})
@Import(AspectV6Advice.class)
@SpringBootTest
public class AopTest {
}
```
`AspectV5Order`를 실행하기 위해서 다음 처리를 하자 
- `@Import({AspectV5Order.LogAspect.class, AspectV5Order.TxAspect.class})` 주석 처리
- `@Import(AspectV6Advice.class)` 입력

**실행**
```
[around][트랜잭션 시작] void hello.aop.order.OrderService.orderItem(String)
[before] void hello.aop.order.OrderService.orderItem(String)
[orderService] 실행
[orderRepository] 실행
[return] void hello.aop.order.OrderService.orderItem(String) return=null
[after] void hello.aop.order.OrderService.orderItem(String)
[around][트랜잭션 커밋] void hello.aop.order.OrderService.orderItem(String)
[around][리소스 릴리즈] void hello.aop.order.OrderService.orderItem(String)
```

![image](https://user-images.githubusercontent.com/28394879/141970960-3109dbc9-832c-4c3a-b353-8d42a6ee07a3.png)

**순서** 
- 스프링 5.2.7 버전부터 동일한 `@Aspect`안에서 동일한 조인포인트의 우선순위를 정했다.
- 실행 순서: `@Around`, `@Before`, `@After`, `@AfterReturning`, `@AfterThrowing`
- 어드바이스가 적용되는 순서는 이렇게 적용되지만, 호출 순서와 리턴 순서는 반대라는 점을 알아두자.
- 물론 `@Aspect`안에 동일한 종류의 어드바이스가 2개 있으면 순서가 보장되지 않는다. 이 경우 앞서 배운 것 처럼 `@Aspect`를 분리하고 `@Order`를 적용하자.

**@Around 외에 다른 어드바이스가 존재하는 이유**  
`@Around` 하나만 있어도 모든 기능을 수애할 수 있다.  
그런데 다른 어드바이스들이 존재하는 이유는 무엇일까?

다음 코드를 보자.
```java
@Around("hello.aop.order.aop.Pointcuts.orderAndService()")
public void doBefore(ProceedingJoinPoint joinPoint) {
 log.info("[before] {}", joinPoint.getSignature());
}
```

이 코드의 문제점을 찾을 수 있겠는가? 이 코드는 타겟을 호출하지 않는 문제가 있다.  
이 코드를 개발한 의도는 타겟 실행 전에 로그를 출력하는 것이다.  
그런데 `@Around`는 항상 `joinPoint.proceed()`를 호출해야 한다.  
만약 실수로 호출하지 않으면 타겟이 호출되지 않는 치명적인 버그가 발생한다.

다음 코드를 보자.
```java
@Before("hello.aop.order.aop.Pointcuts.orderAndService()")
public void doBefore(JoinPoint joinPoint) {
 log.info("[before] {}", joinPoint.getSignature());
}
```

`@Before`는 `joinPoint.proceed()`를 호출하는 고민을 하지 않아도 된다.

`@Around`가 가장 넓은 기능을 제공하는 것은 맞지만, 실수할 가능성이 있다.  
반면에 `@Before`, `@After` 같은 어드바이스는 기능은 적지만 실수할 가능성이 낮고, 코드도 단순하다.  
그리고 가장 중요한 점이 있는데, 바로 이 코드를 작성한 의도가 명확하게 들어난다는 점이다.   
`@Before`라는 애노테이션을 보는 순간 이 코드는 타겟 실행 전에 한정해서 어떤 일을 하는 코드구나 라는 것이 들어난다.

**좋은 설계는 제약이 있는것이다**  
좋은 설계는 제약이 있는 것이다.  
`@Around`만 있으면 되는데 왜? 이렇게 제약을 두는가?  
제약은 실수를 미연에 방지한다.  
일종에 가이드 역할을 한다.  
만약 `@Around`를 사용했는데, 중간에 다른 개발자가 해당 코드를 수정해서 호출하지 않았다면?  
큰 장애가 발생할 것이다.  
처음부터 `@Before`를 사용했다면 이런 문제 자체가 발생하지 않는다.  
제약 덕분에 역할이 명확해진다.  
다른 개발자도 이 코드를 보고 고민해야 하는 범위가 줄어들고 코드의 의도도 파악하기 쉽다. 

</details>


# [11. 스프링 AOP - 포인트컷](./11.spring-aop-pointcut)

<details> <summary> 1. 포인트컷 지시자 </summary>

## 1. 포인트컷 지시자

지금부터 포인트컷 표현식을 포함한 포인트컷에 대해서 자세히 알아보자.

애스펙트J는 포인트컷을 편리하게 표현하기 위한 특별한 표현식을 제공한다.  
예) `@Pointcut("execution(* hello.aop.order..*(..))")`  
포인트컷 표현식은 AspectJ pointcut expression 즉 애스펙터J가 제공하는 포인트컷 표현식을 줄여서 말하는 것이다.

**포인트컷 지시자**  
포인트컷 표현식은 `execution`같은 포인트컷 지시자(Pointcut Designator)로 시작한다. 줄여서 PCD라 한다.

- 포인트컷 지시자의 종류
  - `execution`: 메소드 실행 조인 포인트를 매칭한다. 스프링 AOP에서 가장 많이 사용하고, 기능도 복잡하다.
  - `within`: 특정 타입 내의 조인 포인트를 매칭한다.
  - `args`: 인자가 주어진 타입의 인스턴스인 조인 포인트
  - `this`: 스프링 빈 객체(스프링 AOP 프록시)를 대상으로 하는 조인 포인트
  - `target`: Target 객체(스프링 AOP 프록시)를 대상으로 하는 조인 포인트 
  - `@target`: 실행 객체의 클래스에 주어진 타입의 애노테이션이 있는 조인 포인트
  - `@within`: 주어진 애노테이션이 있는 타입 내 조인 포인트
  - `@annotation`: 메서드가 주어진 애노테이션을 가지고 있는 조인 포인트를 매칭
  - `@args`: 전달된 실제 인수의 런타임 타입이 주어진 타입의 애노테이션을 갖는 조인 포인트
  - `bean`: 스프링 전용 포인트컷 지시자, 빈의 이름으로 포인트컷을 지정한다.

포인트컷 지시자가 무엇을 뜻하는지, 사실 글로만 읽어보면 이해하기 쉽지 않다.  
예제를 통해서 하나씩 이해해보자.  
`execution`은 가장 많이 사용하고, 나머지는 자주 사용하지 않는다.  
따라서 `execution`을 중점적으로 이해하자.

</details>

<details> <summary> 2. 예제 만들기 </summary>

## 2. 예제 만들기

포인트컷 표현식을 이해하기 위해 예제 코드를 하나 추가하자.

**ClassAop**
```java
package hello.aop.member.annotation;
import java.lang.annotation.*;
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface ClassAop {
}
```

**MethodAop**
```java
package hello.aop.member.annotation;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface MethodAop {
 String value();
}
```

**MemberService**
```java
package hello.aop.member;
public interface MemberService {
 String hello(String param);
}
```

**MemberServiceImpl**
```java
package hello.aop.member;
import hello.aop.member.annotation.ClassAop;
import hello.aop.member.annotation.MethodAop;
import org.springframework.stereotype.Component;
@ClassAop
@Component
public class MemberServiceImpl implements MemberService {
 @Override
 @MethodAop("test value")
 public String hello(String param) {
 return "ok";
 }
 public String internal(String param) {
 return "ok";
 }
}
```


**ExecutionTest**
```java
package hello.aop.pointcut;
import hello.aop.member.MemberServiceImpl;
import lombok.extern.slf4j.Slf4j;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.aop.aspectj.AspectJExpressionPointcut;
import java.lang.reflect.Method;
import static org.assertj.core.api.Assertions.assertThat;
@Slf4j
public class ExecutionTest {
 AspectJExpressionPointcut pointcut = new AspectJExpressionPointcut();
 Method helloMethod;
 @BeforeEach
 public void init() throws NoSuchMethodException {
 helloMethod = MemberServiceImpl.class.getMethod("hello", String.class);
 }
 @Test
 void printMethod() {
 //public java.lang.String
hello.aop.member.MemberServiceImpl.hello(java.lang.String)
 log.info("helloMethod={}", helloMethod);
 }
}
```

`AspectJExpressionPointcut`이 바로 포인트컷 표현식을 처리해주는 클래스다.  
여기에 포인트컷 표현식을 지정하면 된다.  
`AspectJExpressionPointcut`는 상위에 `Pointcut`인터페이스를 가진다.

`printMethod()` 테스트는 `MemberServiceImpl.hello(String)` 메서드의 정보를 출력해준다.

**실행 결과**  
```
helloMethod = public java.lang.String
hello.aop.member.MemberServiceImpl.hello(java.lang.String)
```

이번에 알아볼 `execution`으로 시작하는 포인트컷 표현식은 이 메서드 정보를 매칭해서 포인트컷 대상을 찾아낸다.


</details>

<details> <summary> 3. execution - 1 </summary>

## 3. execution - 1

**execution 문법**
```
execution(modifiers-pattern? ret-type-pattern declaring-type-pattern?namepattern(param-pattern)
 throws-pattern?)
execution(접근제어자? 반환타입 선언타입?메서드이름(파라미터) 예외?
```
- 메소드 실행 조인 포인트를 매칭한다.
- ?는 생략할 수 있다.
- `*` 같은 패턴을 지정할 수 있다.

실제 코드를 하나씩 보면서 `execution`을 이해해보자.

**가장 정확한 포인트컷**  
먼저 `MemberServiceImpl.hello(String)`메서드와 가장 정확하게 모든 내용이 매칭되는 표현식이다.

**ExecutionTest - 추가**
```java
@Test
void exactMatch() {
 //public java.lang.String
hello.aop.member.MemberServiceImpl.hello(java.lang.String)
pointcut.setExpression("execution(public String 
hello.aop.member.MemberServiceImpl.hello(String))");
 assertThat(pointcut.matches(helloMethod,
MemberServiceImpl.class)).isTrue();
}
```
- `AspectJExpressionPointcut`에 `pointcut.setExpression`을 통해서 포인트컷 표현식을 적용할 수 있다.
- `pointcut.matches(메서드, 대상 클래스)`를 실행하면 지정한 포인트컷 표현식의 매칭 여부를 `true`, `false`로 변환한다.

**매칭 조건**
- 접근제어자?: `public`
- 반환타입: `String`
- 선언타입?: `hello.aop.member.MemberServiceImpl`
- 메서드이름: `hello`
- 파라미터: `(String)`
- 예외?: 생략

`MemberServiceImpl.hello(String)` 메서드와 포인트컷 표현식의 모든 내용이 정확하게 일치한다.  
따라서 `true`를 반환한다.


**가장 많이 생략한 포인트컷**
```java
@Test
void allMatch() {
 pointcut.setExpression("execution(* *(..))");
 assertThat(pointcut.matches(helloMethod,
MemberServiceImpl.class)).isTrue();
}
```
가장 많이 생략한 포인트컷이다.

**매칭 조건**
- 접근제어자?: 생략 
- 반환타입: `*`
- 선언타입?: 생략
- 메서드이름: `*`
- 파라미터: `(..)`
- 예외?: 없음

- `*`은 아무 값이 들어와도 된다는 뜻이다.
- 파라미터에서 `..`은 파라미터의 타입과 파라미터 수가 상관없다는 뜻이다. (`0..*`) 파라미터는 뒤에 자세히 정리하겠다.

**메서드 이름 매칭 관련 포인트컷**
```java
@Test
void nameMatch() {
 pointcut.setExpression("execution(* hello(..))");
 assertThat(pointcut.matches(helloMethod,
MemberServiceImpl.class)).isTrue();
}
@Test
void nameMatchStar1() {
 pointcut.setExpression("execution(* hel*(..))");
 assertThat(pointcut.matches(helloMethod,
MemberServiceImpl.class)).isTrue();
}
@Test
void nameMatchStar2() {
 pointcut.setExpression("execution(* *el*(..))");
 assertThat(pointcut.matches(helloMethod,
MemberServiceImpl.class)).isTrue();
}
@Test
void nameMatchFalse() {
 pointcut.setExpression("execution(* nono(..))");
 assertThat(pointcut.matches(helloMethod,
MemberServiceImpl.class)).isFalse();
}
```
메서드 이름 앞 뒤에 `*`을 사용해서 매칭할 수 있다.


**패키지 매칭 관련 포인트컷**
```java
@Test
void packageExactMatch1() {
 pointcut.setExpression("execution(*
hello.aop.member.MemberServiceImpl.hello(..))");
 assertThat(pointcut.matches(helloMethod,
MemberServiceImpl.class)).isTrue();
}
@Test
void packageExactMatch2() {
 pointcut.setExpression("execution(* hello.aop.member.*.*(..))");
 assertThat(pointcut.matches(helloMethod,
MemberServiceImpl.class)).isTrue();
}
@Test
void packageExactMatchFalse() {
 pointcut.setExpression("execution(* hello.aop.*.*(..))");
 assertThat(pointcut.matches(helloMethod,
MemberServiceImpl.class)).isFalse();
}
@Test
void packageMatchSubPackage1() {
 pointcut.setExpression("execution(* hello.aop.member..*.*(..))");
 assertThat(pointcut.matches(helloMethod,
MemberServiceImpl.class)).isTrue();
}
@Test
void packageMatchSubPackage2() {
 pointcut.setExpression("execution(* hello.aop..*.*(..))");
 assertThat(pointcut.matches(helloMethod,
MemberServiceImpl.class)).isTrue();
}
```


`hello.aop.member.*(1).*(2)`
- (1): 타입
- (2): 메서드 이름


패키지에서 `.`, `..`의 차이를 이해해야 한다.
- `.`: 정확하게 해당 위치의 패키지
- `..`: 해당 위치의 패키지와 그 하위 패키지도 포함

</details>

<details> <summary> 4. execution - 2 </summary>

## 4. execution - 2 

**타입 매칭 - 부모 타입 허용**

```java
@Test
void typeExactMatch() {
 pointcut.setExpression("execution(*
hello.aop.member.MemberServiceImpl.*(..))");
 assertThat(pointcut.matches(helloMethod,
MemberServiceImpl.class)).isTrue();
}
@Test
void typeMatchSuperType() {
 pointcut.setExpression("execution(*
hello.aop.member.MemberService.*(..))");
 assertThat(pointcut.matches(helloMethod,
MemberServiceImpl.class)).isTrue();
}
```

`typeExactMatch()`는 타입 정보가 정확하게 일치하기 떄문에 매칭된다.
`typeMatchSuperType()`을 주의해서 보아야 한다.  
`execution`에서는 `MemberService`처럼 부모 타입을 선언해도 그 자식 타입은 매칭된다.  
다형성에서 `부모타입 = 자식타입` 이 할당 가능하다는 점을 떠올려보면 된다.


**타입 매칭 - 부모 타입에 있는 메서드만 허용**
```java
@Test
void typeMatchInternal() throws NoSuchMethodException {
 pointcut.setExpression("execution(*
hello.aop.member.MemberServiceImpl.*(..))");
 Method internalMethod = MemberServiceImpl.class.getMethod("internal",
String.class);
 assertThat(pointcut.matches(internalMethod,
MemberServiceImpl.class)).isTrue();
}
//포인트컷으로 지정한 MemberService 는 internal 이라는 이름의 메서드가 없다.
@Test
void typeMatchNoSuperTypeMethodFalse() throws NoSuchMethodException {
 pointcut.setExpression("execution(*
hello.aop.member.MemberService.*(..))");
 Method internalMethod = MemberServiceImpl.class.getMethod("internal",
String.class);
 assertThat(pointcut.matches(internalMethod,
MemberServiceImpl.class)).isFalse();
}
```

`typeMatchInternal()`의 경우 `MemberServiceImpl`를 표현식에 선언했기 때문에 그 안에 있는 `internal(String)` 메서드도 매칭 대상이 된다.  
`typeMatchNoSuperTypeMethodFalse()`를 주의해서 보아야 한다.  
이 경우 표현식에 부모 타입인 `MemberService`를 선언했다.  
그런데 자식 타입인 `MemberServiceImpl`의 `internal(String)`메서드를 매칭하려 한다.  
이 경우 매칭에 실패한다. `MemberService`에는 `internal(String)`메서드가 없다!  
부모 타입을 표현식에 선언한 경우 부모 타입에서 선언한 메서드가 자식 타입에 있어야 매칭에 성공한다.  
그래서 부모 타입에 있는 `hello(String)` 메서드는 매칭에 성공하지만, 부모 타입에 없는 `internal(String)`는 매칭에 실패한다.

**파라미터 매칭**
```java
//String 타입의 파라미터 허용
//(String)
@Test
void argsMatch() {
 pointcut.setExpression("execution(* *(String))");
 assertThat(pointcut.matches(helloMethod,
MemberServiceImpl.class)).isTrue();
}
//파라미터가 없어야 함
//()
@Test
void argsMatchNoArgs() {
 pointcut.setExpression("execution(* *())");
 assertThat(pointcut.matches(helloMethod,
MemberServiceImpl.class)).isFalse();
}
//정확히 하나의 파라미터 허용, 모든 타입 허용
//(Xxx)
@Test
void argsMatchStar() {
 pointcut.setExpression("execution(* *(*))");
 assertThat(pointcut.matches(helloMethod,
MemberServiceImpl.class)).isTrue();
}
//숫자와 무관하게 모든 파라미터, 모든 타입 허용
//파라미터가 없어도 됨
//(), (Xxx), (Xxx, Xxx)
@Test
void argsMatchAll() {
 pointcut.setExpression("execution(* *(..))");
 assertThat(pointcut.matches(helloMethod,
MemberServiceImpl.class)).isTrue();
}
//String 타입으로 시작, 숫자와 무관하게 모든 파라미터, 모든 타입 허용
//(String), (String, Xxx), (String, Xxx, Xxx) 허용
@Test
void argsMatchComplex() {
 pointcut.setExpression("execution(* *(String, ..))");
 assertThat(pointcut.matches(helloMethod,
MemberServiceImpl.class)).isTrue();
}
```

**execution 파라미터 매칭 규칙은 다음과 같다.**
- `(String)`: 정확하게 String 타입 파라미터
- `()`: 파라미터가 없어야 한다.
- `(*)`: 정확히 하나의 파라미터, 단 모든 타입을 허용한다.
- `(*, *)`: 정확히 두개의 파라미터, 단 모든 타입을 허용한다.
- `(..)`: 숫자와 무관하게 모든 파라미터, 모든 타입을 허용한다. 참고로 파라미터가 없어도 된다. `0..*`로 이해하면 된다.
- `(String ..)`: String 타입으로 시작해야 한다. 숫자와 무관하게 모든 파라미터, 모든 타입을 허용한다.
  - 예) `(String)`, `(String, Xxx)`, `(String, Xxx, Xxx)` 허용

</details>

<details> <summary> 5. within </summary>

## 5. within

within 지시자는 특정 타입 내의 조인 포인트에 대한 매칭을 제한한다.  
쉽게 이야기해서 해당 타입이 매칭되면 그 안의 메서드(조인 포인트)들이 자동으로 매칭된다.  
문법은 단순한데 `execution`에서 타입 부분만 사용한다고 보면 된다.

**WithinTest**
```java
package hello.aop.pointcut;
import hello.aop.member.MemberServiceImpl;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;
import org.springframework.aop.aspectj.AspectJExpressionPointcut;
import java.lang.reflect.Method;
import static org.assertj.core.api.Assertions.assertThat;
public class WithinTest {
 AspectJExpressionPointcut pointcut = new AspectJExpressionPointcut();
 Method helloMethod;
 @BeforeEach
 public void init() throws NoSuchMethodException {
 helloMethod = MemberServiceImpl.class.getMethod("hello", String.class);
 }
 @Test
 void withinExact() {
 pointcut.setExpression("within(hello.aop.member.MemberServiceImpl)");
 assertThat(pointcut.matches(helloMethod,
MemberServiceImpl.class)).isTrue();
 }
 @Test
 void withinStar() {
 pointcut.setExpression("within(hello.aop.member.*Service*)");
 assertThat(pointcut.matches(helloMethod,
MemberServiceImpl.class)).isTrue();
 }
 @Test
 void withinSubPackage() {
 pointcut.setExpression("within(hello.aop..*)");
 assertThat(pointcut.matches(helloMethod,
MemberServiceImpl.class)).isTrue();
 }
}
```
코드를 보면 이해하는데 어려움은 없을 것이다.

**주의**  
그런데 `within`사용시 주의해야 할 점이 있다.  
표현식에 부모 타입을 지정하면 안된다는 점이다.  
정확하게 타입이 맞아야 한다.  
이부분에서 `execution`과 차이가 난다.  

**WithinTest - 추가**
```java
@Test
@DisplayName("타켓의 타입에만 직접 적용, 인터페이스를 선정하면 안된다.")
void withinSuperTypeFalse() {
 pointcut.setExpression("within(hello.aop.member.MemberService)");
 assertThat(pointcut.matches(helloMethod,
MemberServiceImpl.class)).isFalse();
}
@Test
@DisplayName("execution은 타입 기반, 인터페이스를 선정 가능.")
void executionSuperTypeTrue() {
 pointcut.setExpression("execution(*
hello.aop.member.MemberService.*(..))");
 assertThat(pointcut.matches(helloMethod,
MemberServiceImpl.class)).isTrue();
}
```
부모 타입(여기서는 `MemberService` 인터페이스) 지정시 `within`은 실패하고, `execution`은 성공하는 것을 확인할 수 있다.



</details>

<details> <summary> 6. args </summary>

## 6. args

- `args`: 인자가 주어진 타입의 인스턴스인 조인 포인트로 매칭
- 기본 문법은 `execution`의 `args` 부분과 같다.

**execution 과 args의 차이점**  
- `execution`은 파라미터 타입이 정확하게 매칭되어야 한다. `execution`은 클래스에 선언된 정보를 기반으로 판단한다.
- `args`는 부모 타입을 허용한다. `args`는 실제 넘어온 파라미터 객체 인스턴스를 보고 판단한다.

**ArgsTest**
```java
package hello.aop.pointcut;
import hello.aop.member.MemberServiceImpl;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.aop.aspectj.AspectJExpressionPointcut;
import java.lang.reflect.Method;
import static org.assertj.core.api.Assertions.assertThat;
public class ArgsTest {
 Method helloMethod;
 @BeforeEach
 public void init() throws NoSuchMethodException {
 helloMethod = MemberServiceImpl.class.getMethod("hello", String.class);
 }
 private AspectJExpressionPointcut pointcut(String expression) {
 AspectJExpressionPointcut pointcut = new AspectJExpressionPointcut();
 pointcut.setExpression(expression);
 return pointcut;
 }
 @Test
 void args() {
 //hello(String)과 매칭
 assertThat(pointcut("args(String)")
 .matches(helloMethod, MemberServiceImpl.class)).isTrue();
 assertThat(pointcut("args(Object)")
 .matches(helloMethod, MemberServiceImpl.class)).isTrue();
 assertThat(pointcut("args()")
 .matches(helloMethod, MemberServiceImpl.class)).isFalse();
 assertThat(pointcut("args(..)")
 .matches(helloMethod, MemberServiceImpl.class)).isTrue();
 assertThat(pointcut("args(*)")
 .matches(helloMethod, MemberServiceImpl.class)).isTrue();
 assertThat(pointcut("args(String,..)")
 .matches(helloMethod, MemberServiceImpl.class)).isTrue();
 }
 /**
 * execution(* *(java.io.Serializable)): 메서드의 시그니처로 판단 (정적)
 * args(java.io.Serializable): 런타임에 전달된 인수로 판단 (동적)
 */
 @Test
 void argsVsExecution() {
 //Args
 assertThat(pointcut("args(String)")
 .matches(helloMethod, MemberServiceImpl.class)).isTrue();
 assertThat(pointcut("args(java.io.Serializable)")
 .matches(helloMethod, MemberServiceImpl.class)).isTrue();
 assertThat(pointcut("args(Object)")
 .matches(helloMethod, MemberServiceImpl.class)).isTrue();
 //Execution
 assertThat(pointcut("execution(* *(String))")
 .matches(helloMethod, MemberServiceImpl.class)).isTrue();
 assertThat(pointcut("execution(* *(java.io.Serializable))") //매칭 실패
 .matches(helloMethod, MemberServiceImpl.class)).isFalse();
 assertThat(pointcut("execution(* *(Object))") //매칭 실패
 .matches(helloMethod, MemberServiceImpl.class)).isFalse();
 }
}
```
- `pointcut()`: `AspectJExpressionPointcut`에 포인트컷은 한번만 지정할 수 있다. 이번 테스트에서는 테스트를 편리하게 진행하기 위해 포인트컷을 여러번 지정하기 위해 포인트컷 자체를 생성하는 메서드를 만들었다.
- 자바가 기본으로 제공하는 `String`은 `Object`, `java.io.Serializable`의 하위 타입이다.
- 정적으로 클래스에 선언된 정보만 보고 판단하는 `execution(* *(Object))`는 매칭에 실패한다.
- 동적으로 실제 파라미터로 넘어온 객체 인스턴스로 판단하는 `args(Object)`는 매칭에 성공한다. (부모 타입 허용) 

> **참고**  
> `args`지시자는 단독으로 사용되기 보다는 뒤에서 설명할 파라미터 바인딩에서 주로 사용한다.

</details>

<details> <summary> 7. @target, @within </summary>

## 7. @target, @within 

**정의** 
- `@target`: 실행 객체의 클래스에 주어진 타입의 애노테이션이 있는 조인 포인트
- `@within`: 주어진 애노테이션이 있는 타입 내 포인트

**설명**  
`@target`, `@within`은 다음과 같이 타입에 있는 애노테이션으로 AOP 적용 여부를 판단한다.

- `@target(hello.aop.member.annotation.ClassAop)`
- `@within(hello.aop.member.annotation.ClassAop)`

```java
@ClassAop
class Target{}
```

**@target vs @within**
- `@target`은 인스턴스의 모든 메서드를 조인 포인트로 적용한다.
- `@within`은 해당 타입 내에 있는 메서드만 조인 포인트로 적용한다.

쉽게 이야기해서 `@target`은 부모 클래스의 메서드까지 어드바이스를 다 적용하고, `@within`은 자기 자신의 클래스에 정의된 메서드에만 어드바이스를 적용한다.

![image](https://user-images.githubusercontent.com/28394879/142205369-be90432c-c8a4-4d21-bf1c-614f036242a5.png)

**AtTargetAtWithinTest**
```java
package hello.aop.pointcut;
import hello.aop.member.annotation.ClassAop;
import lombok.extern.slf4j.Slf4j;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Import;
@Slf4j
@Import({AtTargetAtWithinTest.Config.class})
@SpringBootTest
public class AtTargetAtWithinTest {
 @Autowired
 Child child;
 @Test
 void success() {
 log.info("child Proxy={}", child.getClass());
 child.childMethod(); //부모, 자식 모두 있는 메서드
 child.parentMethod(); //부모 클래스만 있는 메서드
 }
 static class Config {
 @Bean
 public Parent parent() {
 return new Parent();
 }
 @Bean
 public Child child() {
 return new Child();
 }
 @Bean
 public AtTargetAtWithinAspect atTargetAtWithinAspect() {
 return new AtTargetAtWithinAspect();
 }
 }
 static class Parent {
 public void parentMethod(){} //부모에만 있는 메서드
 }
 @ClassAop
 static class Child extends Parent {
 public void childMethod(){}
 }
 @Slf4j
 @Aspect
 static class AtTargetAtWithinAspect {
 //@target: 인스턴스 기준으로 모든 메서드의 조인 포인트를 선정, 부모 타입의 메서드도 적용
 @Around("execution(* hello.aop..*(..)) &&
@target(hello.aop.member.annotation.ClassAop)")
 public Object atTarget(ProceedingJoinPoint joinPoint) throws Throwable
{
 log.info("[@target] {}", joinPoint.getSignature());
 return joinPoint.proceed();
 }
 //@within: 선택된 클래스 내부에 있는 메서드만 조인 포인트로 선정, 부모 타입의 메서드는
적용되지 않음
 @Around("execution(* hello.aop..*(..)) &&
@within(hello.aop.member.annotation.ClassAop)")
 public Object atWithin(ProceedingJoinPoint joinPoint) throws Throwable
{
 log.info("[@within] {}", joinPoint.getSignature());
 return joinPoint.proceed();
 }
 }
}
```

**실행 결과**
```
[@target] void hello.aop.pointcut.AtTargetAtWithinTest$Child.childMethod()
[@within] void hello.aop.pointcut.AtTargetAtWithinTest$Child.childMethod()
[@target] void hello.aop.pointcut.AtTargetAtWithinTest$Parent.parentMethod()
```

`parentMethod()`는 `Parent` 클래스에만 정의되어 있고, `Child` 클래스에 정의되어 있지 않기 때문에 `@within`에서 AOP 적용 대상이 되지 않는다.  
실행결과를 보면 `child.parentMethod()`를 호출 했을 때 `[@within]이 호출되지 않은 것을 확인할 수 있다.

> **참고**  
> `@target`, `@within` 지시자는 뒤에서 설명할 파라미터 바인딩에서 함께 사용된다.

> **주의**  
> 다음 포인트컷 지시자는 단독으로 사용하면 안된다. `args, @args, @target`  
> 이번 예제를 보면 `execution(* hello.aop..*(..))`를 통해 적용 대상을 줄여준 것을 확인할 수 있다.  
> `args`, `@args`, `@target`은 실제 객체 인스턴스가 생성되고 실행될 때 어드바이스 적용 여부를 확인할 수 있다.  
> 실행 시점에 일어나는 포인트컷 적용 여부도 결국 프록시가 있어야 실행 시점에 판단할 수 있다.  
> 프록시가 없다면 판단 자체가 불가능하다.  
> 그런데 스프링 컨테이너가 프록시를 생성하는 시점은 스프링 컨테이너가 만들어지는 애플리케이션 로딩 시점에 적용할 수 있다.  
> 따라서 `args`, `@args`, `@target` 같은 포인트컷 지시자가 있으면 스프링은 모든 스프링 빈에 AOP를 적용하려고 시도한다.  
> 앞서 설명한 것처럼 프록시가 없으면 싫애 시점에 판단 자체가 불가능하다.  
> 문제는 이렇게 모든 스프링 빈에 AOP 프록시를 적용하려고 하면 스프링이 내부에서 사용하는 빈 중에는 `final`로 지정된 빈들도 있기 떄문에 오류가 발생할 수 있다.  
> 따라서 이러한 표현식은 최대한 프록시 적용 대상을 축소하는 표현식과 함께 사용해야 한다.





</details>

<details> <summary> 8. @annotation, @args </summary>

## 8. @annotation, @args

**@annotation**  
**정의**  
`@annotation`: 메서드가 주어진 애노테이션을 가지고 있는 조인 포인트를 매칭

**설명**  
`@annotation(hello.aop.member.annotation.MethodAop)`

다음과 같이 메서드(조인 포인트)에 애노테이션이 있으면 매칭한다.  
```java
public class MemberServiceImpl {
 @MethodAop("test value")
 public String hello(String param) {
 return "ok";
 }
}
```

**AtAnnotationTest**
```java
package hello.aop.pointcut;
import hello.aop.member.MemberService;
import lombok.extern.slf4j.Slf4j;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.context.annotation.Import;
@Slf4j
@Import(AtAnnotationTest.AtAnnotationAspect.class)
@SpringBootTest
public class AtAnnotationTest {
 @Autowired
 MemberService memberService;
 @Test
 void success() {
 log.info("memberService Proxy={}", memberService.getClass());
 memberService.hello("helloA");
 }
 @Slf4j
 @Aspect
 static class AtAnnotationAspect {
 @Around("@annotation(hello.aop.member.annotation.MethodAop)")
 public Object doAtAnnotation(ProceedingJoinPoint joinPoint) throws
Throwable {
 log.info("[@annotation] {}", joinPoint.getSignature());
 return joinPoint.proceed();
 }
 }
}
```

**실행 결과**
```
[@annotation] String hello.aop.member.MemberService.hello(String)
```

**@args**  
**정의**
- `@args`: 전달된 실제 인수의 런타임 타입이 주어진 타입의 애노테이션을 갖는 조인 포인트

**설명**  
전달된 인수의 런타임 타입에 `@Check` 애노테이션이 있는 경우에 매칭한다.  
`@args(test.Check)`




</details>

<details> <summary> 9. bean </summary>

## 9. bean 

**정의** 
- `bean`: 스프링 전용 포인트컷 지시자, 빈의 이름으로 지정한다.

**설명**
- 스프링 빈의 이름으로 AOP 적용 여부를 지정한다. 이것은 스프링에서만 사용할 수 있는 특별한 지시자이다.
- `bean(orderService) || bean(*Repository)`
- `*`과 같은 패턴을 사용할 수 있다.

```java
package hello.aop.pointcut;
import hello.aop.order.OrderService;
import lombok.extern.slf4j.Slf4j;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.context.annotation.Import;
@Slf4j
@Import(BeanTest.BeanAspect.class)
@SpringBootTest
public class BeanTest {
 @Autowired
 OrderService orderService;
 @Test
 void success() {
 orderService.orderItem("itemA");
 }
 @Aspect
 static class BeanAspect {
 @Around("bean(orderService) || bean(*Repository)")
 public Object doLog(ProceedingJoinPoint joinPoint) throws Throwable {
 log.info("[bean] {}", joinPoint.getSignature());
 return joinPoint.proceed();
 }
 }
}
```

`OrderService`, `*Repository(OrderRepository)`의 메서드에 AOP가 적용된다.

**실행 결과**
```
[bean] void hello.aop.order.OrderService.orderItem(String)
[orderService] 실행
[bean] String hello.aop.order.OrderRepository.save(String)
[orderRepository] 실행
```


</details>

<details> <summary> 10. 매개변수 전달 </summary>

## 10. 매개변수 전달

다음은 포인트컷 표현식을 사용해서 어드바이스에 매개변수를 전달할 수 있다.  
**this, target, args, @target, @within, @annotation, @args**

다음과 같이 사용한다.  
```java
@Before("allMember() && args(arg,..)")
public void logArgs3(String arg) {
 log.info("[logArgs3] arg={}", arg);
}
```
- 포인트컷의 이름과 매개변수의 이름을 맞추어야 한다. 여기서는 `args`로 맞추었다.  
- 추가로 타입이 메서드에 지정한 타입으로 제한된다. 여기서는 메서드의 타입이 `String`으로 되어 있기 떄문에 다음과 같이 정의되는 것으로 이해하면 된다.  
  - `args(arg,...)` -> `args(String,...)`

다양한 매개변수 전달 예시를 확인해보자.  
**ParameterTest**
```java
package hello.aop.pointcut;
import hello.aop.member.MemberService;
import hello.aop.member.annotation.ClassAop;
import hello.aop.member.annotation.MethodAop;
import lombok.extern.slf4j.Slf4j;
import org.aspectj.lang.JoinPoint;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Before;
import org.aspectj.lang.annotation.Pointcut;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.context.annotation.Import;
@Slf4j
@Import(ParameterTest.ParameterAspect.class)
@SpringBootTest
public class ParameterTest {
 @Autowired
 MemberService memberService;
 @Test
 void success() {
 log.info("memberService Proxy={}", memberService.getClass());
 memberService.hello("helloA");
 }
 @Slf4j
 @Aspect
 static class ParameterAspect {
 @Pointcut("execution(* hello.aop.member..*.*(..))")
 private void allMember() {}
 @Around("allMember()")
 public Object logArgs1(ProceedingJoinPoint joinPoint) throws Throwable
{
 Object arg1 = joinPoint.getArgs()[0];
 log.info("[logArgs1]{}, arg={}", joinPoint.getSignature(), arg1);
 return joinPoint.proceed();
 }
 @Around("allMember() && args(arg,..)")
 public Object logArgs2(ProceedingJoinPoint joinPoint, Object arg)
throws Throwable {
 log.info("[logArgs2]{}, arg={}", joinPoint.getSignature(), arg);
 return joinPoint.proceed();
 }
 @Before("allMember() && args(arg,..)")
 public void logArgs3(String arg) {
 log.info("[logArgs3] arg={}", arg);
 }
 @Before("allMember() && this(obj)")
 public void thisArgs(JoinPoint joinPoint, MemberService obj) {
 log.info("[this]{}, obj={}", joinPoint.getSignature(),
obj.getClass());
 }
 @Before("allMember() && target(obj)")
 public void targetArgs(JoinPoint joinPoint, MemberService obj) {
 log.info("[target]{}, obj={}", joinPoint.getSignature(),
obj.getClass());
 }
 @Before("allMember() && @target(annotation)")
 public void atTarget(JoinPoint joinPoint, ClassAop annotation) {
 log.info("[@target]{}, obj={}", joinPoint.getSignature(),
annotation);
 }
 @Before("allMember() && @within(annotation)")
 public void atWithin(JoinPoint joinPoint, ClassAop annotation) {
 log.info("[@within]{}, obj={}", joinPoint.getSignature(),
annotation);
 }
 @Before("allMember() && @annotation(annotation)")
 public void atAnnotation(JoinPoint joinPoint, MethodAop annotation) {
 log.info("[@annotation]{}, annotationValue={}",
joinPoint.getSignature(), annotation.value());
 }
 }
}
```
- `logArgs1`: `joinPoint.getArgs()[0]`와 같이 매개변수를 전달 받는다.
- `logArgs2`: `args(arg, ...)`와 같이 매개변수를 전달 받는다.
- `logArgs3`: `@Before`를 사용한 축약 버전이다. 추가로 타입을 `String`으로 제한했다.
- `this`: 프록시 객체를 전달 받는다.
- `target`: 실제 대상 객체를 전달 받는다.
- `@target`, `@within`: 타입의 애노테이션을 전달 받는다.
- `@annotation`: 메서드의 애노테이션을 전달 받는다. 여기서는 `annotation.value()`로 해당 애노테이션의 값을 출력하는 모습을 확인할 수 있다.


**실행 결과**  
순서는 조정했음  
```
memberService Proxy=class hello.aop.member.MemberServiceImpl$
$EnhancerBySpringCGLIB$$82
[logArgs1]String hello.aop.member.MemberServiceImpl.hello(String), arg=helloA
[logArgs2]String hello.aop.member.MemberServiceImpl.hello(String), arg=helloA
[logArgs3] arg=helloA
[this]String hello.aop.member.MemberServiceImpl.hello(String), obj=class
hello.aop.member.MemberServiceImpl$$EnhancerBySpringCGLIB$$8
[target]String hello.aop.member.MemberServiceImpl.hello(String), obj=class
hello.aop.member.MemberServiceImpl
[@target]String hello.aop.member.MemberServiceImpl.hello(String),
obj=@hello.aop.member.annotation.ClassAop()
[@within]String hello.aop.member.MemberServiceImpl.hello(String),
obj=@hello.aop.member.annotation.ClassAop()
[@annotation]String hello.aop.member.MemberServiceImpl.hello(String),
annotationValue=test value
```

</details>

<details> <summary> 11. this, target </summary>

## 11. this, target

**정의**
- `this`: 스프링 빈 객체(스프링 AOP 프록시)를 대상으로 하는 조인 포인트
- `target`: Target 객체(스프링 AOP 프록시가 가르키는 실제 대상)를 대상으로 하는 조인 포인트

**설명**
- `this`, `target`은 다음과 같이 적용 타입 하나를 정확하게 지정해야 한다.
```java
this(hello.aop.member.MemberService)
target(hello.aop.member.MemberService)
```
- `*` 같은 패턴을 사용할 수 없다.
- 부모 타입을 허용한다.

**this vs target**  
단순히 타입 하나를 정하면 되는데, `this`와 `target`은 어떤 차이가 있을까?

스프링에서 AOP를 적용하면 실제 `target` 객체 대신에 프록시 객체가 스프링 빈으로 등록된다.
- `this`는 스프링 빈으로 등록되어 있는 **프록시 객체**를 대상으로 포인트컷을 매칭한다.
- `target`은 실제 **target 객체**를 대상으로 포인트컷을 매칭한다.

### 프록시 생성 방식에 따른 차이  
스프링은 프록시를 생성할 떄 JDK 동적 프록시와 CGLIB를 선택할 수 있다.  
둘의 프록시를 생성하는 방식이 다르기 떄문에 차이가 발생한다.  
- JDK 동적 프록시: 인터페이스가 필수이고, 인터페이스를 구현한 프록시 객체를 생성한다.
- CGLIB: 인터페이스가 있어도 구체 클래스를 상속 받아서 프록시 객체를 생성한다.

**JDK 동적 프록시**  
![image](https://user-images.githubusercontent.com/28394879/142222030-229daec3-e359-4794-9568-eaaaa589088e.png)
먼저 JDK 동적 프록시를 적용했을 때 `this`, `target`을 알아보자.

**MemberService 인터페이스 지정**
- `this(hello.aop.member.MemberService)`
  - proxy 객체를 보고 판단한다. `this`는 부모 타입을 허용하기 떄문에 AOP가 적용된다.
- `target(hello.aop.member.MemberService)`
  - target 객체를 보고 판단한다. `this`는 부모 타입을 허용하기 떄문에 AOP가 적용된다.

**MemberServiceImpl 구체 클래스 지정**
- `this(hello.aop.member.MemberServiceImpl)`: proxy 객체를 보고 판단한다. JDK 동적 프록시로 만들어진 proxy 객체는 `MemberService` 인터페이스를 기반으로 구현된 새로운 클래스다. 따라서 `MemberServiceImpl`를 전혀 알지 못하므로 **AOP 적용 대상이 아니다.**
- `target(hello.aop.member.MemberServiceImpl)`: target 객체를 보고 판단한다. target 객체가 `MemberServiceImpl` 타입이므로 AOP 적용 대상이다.

**CGLIB 프록시**  
![image](https://user-images.githubusercontent.com/28394879/142222571-fe5f9b96-f96b-4814-bc44-9bd568c7a322.png)  

**MemberService 인터페이스 지정**
- `this(hello.aop.member.MemberService)`: proxy 객체를 보고 판단한다. `this`는 부모 타입을 허용했기 때문에 AOP가 적용된다.
- `target(hello.aop.member.MemberService)`: target 객체를 보고 판단한다. `this`는 부모 타입을 허용했기 때문에 AOP가 적용된다.

**MemberServiceImpl 구체 클래스 지정**
- `this(hello.aop.member.MemberServiceImpl)`: proxy 객체를 보고 판단한다. CGLIB 동적 프록시로 만들어진 proxy 객체는 `MemberServiceImpl`를 상속 받아서 만들었기 때문에 AOP가 적용된다. `this`가 부모 타입을 허용하기 떄문에 포인트컷 대상이 된다. 
- `target(hello.aop.member.MemberServiceImpl)`: target 객체를 보고 판단한다. target 객체가 `MemberServiceImpl` 타입이므로 AOP 적용 대상이다.

**정리**  
프록시를 대상으로 하는 `this`의 경우 구체 클래스를 지정하면 프록시 생성 전략에 따라서 다른 결과가 나올 수 있다는 점을 알아두자.

**ThisTargetTest**
```java
package hello.aop.pointcut;
import hello.aop.member.MemberService;
import lombok.extern.slf4j.Slf4j;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.context.annotation.Import;
/**
 * application.properties
 * spring.aop.proxy-target-class=true CGLIB
 * spring.aop.proxy-target-class=false JDK 동적 프록시
 */
@Slf4j
@Import(ThisTargetTest.ThisTargetAspect.class)
@SpringBootTest(properties = "spring.aop.proxy-target-class=false") //JDK 동적
프록시
//@SpringBootTest(properties = "spring.aop.proxy-target-class=true") //CGLIB
public class ThisTargetTest {
 @Autowired
 MemberService memberService;
 @Test
 void success() {
 log.info("memberService Proxy={}", memberService.getClass());
 memberService.hello("helloA");
 }
 @Slf4j
 @Aspect
 static class ThisTargetAspect {
 //부모 타입 허용
 @Around("this(hello.aop.member.MemberService)")
 public Object doThisInterface(ProceedingJoinPoint joinPoint) throws
Throwable {
 log.info("[this-interface] {}", joinPoint.getSignature());
 return joinPoint.proceed();
 }
 //부모 타입 허용
 @Around("target(hello.aop.member.MemberService)")
 public Object doTargetInterface(ProceedingJoinPoint joinPoint) throws
Throwable {
 log.info("[target-interface] {}", joinPoint.getSignature());
 return joinPoint.proceed();
 }
 //this: 스프링 AOP 프록시 객체 대상
 //JDK 동적 프록시는 인터페이스를 기반으로 생성되므로 구현 클래스를 알 수 없음
 //CGLIB 프록시는 구현 클래스를 기반으로 생성되므로 구현 클래스를 알 수 있음
 @Around("this(hello.aop.member.MemberServiceImpl)")
 public Object doThis(ProceedingJoinPoint joinPoint) throws Throwable {
 log.info("[this-impl] {}", joinPoint.getSignature());
 return joinPoint.proceed();
 }
 //target: 실제 target 객체 대상
 @Around("target(hello.aop.member.MemberServiceImpl)")
 public Object doTarget(ProceedingJoinPoint joinPoint) throws Throwable
{
 log.info("[target-impl] {}", joinPoint.getSignature());
 return joinPoint.proceed();
 }
 }
}
```

`this`, `target`은 실제 객체를 만들어야 테스트 할 수 있따. 테스트에서 스프링 컨테이너를 사용해서 `target`, `proxy` 객체를 모두 만들어서 테스트해보자.

- `properties = {"spring.aop.proxy-target-class=false"}`: `application.properties`에 설정하는 대신에 해당 테스트에서만 설정을 임시로 적용한다. 이렇게 하면 각 테스트마다 다른 설정을 손쉽게 적용할 수 있다.
- `spring.aop.proxy-target-class=false`: 스프링이 AOP 프록시를 생성할 때 JDK 동적 프록시를 우선 생성한다. 물론 인터페이스가 없다면 CGLIB를 사용한다.
- `spring.aop.proxy-target-class=true`: 스프링 AOP 프록시를 생성할 때 CGLIB 프록시를 생성한다. 참고로 이 설정을 생략하면 스프링 부트에서 기본으로 CGLIB를 사용한다. 이 부분은 뒤에서 자세히 정리하겠다.

```java
@SpringBootTest(properties = "spring.aop.proxy-target-class=false") //JDK 동적
프록시
//@SpringBootTest(properties = "spring.aop.proxy-target-class=true") //CGLIB
```

**spring.aop.proxy-target-class=false**  
**JDK 동적 프록시 사용** 
```
memberService Proxy=class com.sun.proxy.$Proxy53
[target-impl] String hello.aop.member.MemberService.hello(String)
[target-interface] String hello.aop.member.MemberService.hello(String)
[this-interface] String hello.aop.member.MemberService.hello(String)
```
JDK 동적 프록시를 사용하면 `this(hello.aop.member.MemberServiceImpl)`로 지정한 `[this-impl]` 부분이 출력되지 않는 것을 확인할 수 있다.

```java
//@SpringBootTest(properties = "spring.aop.proxy-target-class=false") //JDK 동적
프록시
@SpringBootTest(properties = "spring.aop.proxy-target-class=true") //CGLIB
```

**spring.aop.proxy-target-class=true,** 또는 생략(스프링 부트 기본 옵션)  
**CGLIB 사용**  
```
memberService Proxy=class hello.aop.member.MemberServiceImpl$
$EnhancerBySpringCGLIB$$7df96bd3
[target-impl] String hello.aop.member.MemberServiceImpl.hello(String)
[target-interface] String hello.aop.member.MemberServiceImpl.hello(String)
[this-impl] String hello.aop.member.MemberServiceImpl.hello(String)
[this-interface] String hello.aop.member.MemberServiceImpl.hello(String)
```

> 참고  
> `this`, `target` 지시자는 단독으로 사용되기 보다는 파라미터 바인딩에서 주로 사용된다.


> 참고  
> 혹시 해당 내용이 잘 이해가 되지 않으면 스프링 AOP 실무 주의 사항에서 프록시 기술과 한계를 듣고 다시  
> 들어보면 더 이해가 쉬울 것이다.



</details>

# [12. 스프링 AOP - 실전 예제](./12.spring-aop-practical-example)

<details> <summary> 1. 예제 만들기 </summary>

## 1. 예제 만들기

지금까지 학습한 내용을 활용해서 유용한 스프링 AOP를 만들어보자.  
- `@Trace` 애노테이션으로 로그 출력하기
- `@Retry` 애노테이션으로 예외 발생시 재시도 하기

먼저 AOP를 적용할 예제를 만들자.

**ExamRepository**
```java
package hello.aop.exam;
import org.springframework.stereotype.Repository;
@Repository
public class ExamRepository {
 private static int seq = 0;
 /**
 * 5번에 1번 실패하는 요청
 */
 public String save(String itemId) {
 seq++;
 if (seq % 5 == 0) {
 throw new IllegalStateException("예외 발생");
 }
 return "ok";
 }
}
```
5번에 1번 정도는 실패하는 저장소이다. 이렇게 간헐적으로 실패할 경우 재시도 하는 AOP가 있으면 편리하다.

**ExamService**
```java
package hello.aop.exam;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;
@Service
@RequiredArgsConstructor
public class ExamService {
 private final ExamRepository examRepository;
 public void request(String itemId) {
 examRepository.save(itemId);
 }
}
```

**ExamTest**
```java
package hello.aop.exam;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
@SpringBootTest
public class ExamTest {
 @Autowired
 ExamService examService;
 @Test
 void test() {
 for (int i = 0; i < 5; i++) {
 examService.request("data" + i);
 }
 }
}
```
실행해보면 테스트가 5번째 루프를 실행할 때 리포지토리 위치에서 예외가 발생하면서 실패하는 것을 확인할 수 있다. 

</details>

<details> <summary> 2. 로그 출력 AOP </summary>

## 2. 로그 출력 AOP

먼저 로그 출력용 AOP를 만들어보자.  
`@Trace`가 메서드에 붙어 있으면 호출 정보가 출력되는 편리한 기능이다.

**@Trace**
```java
package hello.aop.exam.annotation;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Trace {
}
```

**TraceAspect**
```java
package hello.aop.exam.aop;
import lombok.extern.slf4j.Slf4j;
import org.aspectj.lang.JoinPoint;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Before;
@Slf4j
@Aspect
public class TraceAspect {
 @Before("@annotation(hello.aop.exam.annotation.Trace)")
 public void doTrace(JoinPoint joinPoint) {
 Object[] args = joinPoint.getArgs();
 log.info("[trace] {} args={}", joinPoint.getSignature(), args);
 }
}
```
`@annotation(hello.aop.exam.annotation.Trace)` 포인트컷을 사용해서 `@Trace`가 붙은 메서드에 어드바이스를 적용한다.

**ExamService - @Trace 추가**
```java
package hello.aop.exam;
import hello.aop.exam.annotation.Trace;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;
@Service
@RequiredArgsConstructor
public class ExamService {
 private final ExamRepository examRepository;
 @Trace
 public void request(String itemId) {
 examRepository.save(itemId);
 }
}
```

`request()`에 `@Trace`를 붙였다. 이제 메서드 호출 정보를 AOP를 사용해서 로그로 남길 수 있따.

**ExamRepository - @Trace 추가**
```java
package hello.aop.exam;
import hello.aop.exam.annotation.Trace;
import org.springframework.stereotype.Repository;
@Repository
public class ExamRepository {
 @Trace
 public String save(String itemId) {
 //...
 }
}
```

`save()`에 `@Trace`를 붙였다.

**ExamTest - 추가**
```java
@Import(TraceAspect.class)
@SpringBootTest
public class ExamTest {
}
```

`@Import(TraceAspect.class)`를 사용해서 `@TraceAspect`를 스프링 빈으로 추가하자. 이제 애스펙트가 적용된다.

**실행 결과**
```
[trace] void hello.aop.exam.ExamService.request(String) args=[data0]
[trace] String hello.aop.exam.ExamRepository.save(String) args=[data0]
[trace] void hello.aop.exam.ExamService.request(String) args=[data1]
[trace] String hello.aop.exam.ExamRepository.save(String) args=[data1]
...
```

실행해보면 `@Trace`가 붙은 `request()`, `save()` 호출 시 로그가 잘 남는 것을 확인할 수 있다. 

</details>

<details> <summary> 3. 재시도 AOP </summary>

## 3. 재시도 AOP

이번에는 좀 더 의미있는 재시도 AOP를 만들어보자.  
`@Retry` 애노테이션이 있으면 예외가 발생했을 때 다시 시도해서 문제를 복구한다.

**Retry**
```java
package hello.aop.exam.annotation;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Retry {
 int value() default 3;
}
```
이 애노테이션에는 재시도 횟수로 상요할 값이 있다. 기본적으로 `3`을 사용한다.

**RetryAspect**
```java
package hello.aop.exam.aop;
import hello.aop.exam.annotation.Retry;
import lombok.extern.slf4j.Slf4j;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
@Slf4j
@Aspect
public class RetryAspect {

 @Around("@annotation(retry)")
 public Object doRetry(ProceedingJoinPoint joinPoint, Retry retry) throws
Throwable {
 log.info("[retry] {} retry={}", joinPoint.getSignature(), retry);
 int maxRetry = retry.value();
 Exception exceptionHolder = null;
 for (int retryCount = 1; retryCount <= maxRetry; retryCount++) {
 try {
 log.info("[retry] try count={}/{}", retryCount, maxRetry);
 return joinPoint.proceed();
 } catch (Exception e) {
 exceptionHolder = e;
 }
 }
 throw exceptionHolder;
 }
}
```
- 재시도 하는 애스펙트이다.
- `@annotation(retry)`, `Retry retry`를 사용해서 어드바이스에 애노테이션을 파라미터로 전달한다.
- `retry.value()`를 통해서 애노테이션에 지정한 값을 가져올 수 있다.
- 예외가 발생해서 결과가 정상 반환되지 않으면 `retry.value()`만큼 재시도한다.

**ExamRepository - @Retry 추가**
```java
package hello.aop.exam;
import hello.aop.exam.annotation.Retry;
import hello.aop.exam.annotation.Trace;
import org.springframework.stereotype.Repository;
@Repository
public class ExamRepository {
 @Trace
 @Retry(value = 4)
 public String save(String itemId) {
 //...
 }
}
```
`ExamRepository.save()` 메서드에 `@Retry(value = 4)`를 적용했다. 이 메서드에서 문제가 발생하면 4번 재시도 한다.

**ExamTest - 추가**
```java
@SpringBootTest
//@Import(TraceAspect.class)
@Import({TraceAspect.class, RetryAspect.class})
public class ExamTest {
}
```
- `@Import(TraceAspect.class)`는 주석처리하고 
- `@Import(TraceAspect.class, RetryAspect.class})`를 스프링 빈으로 추가하자.

**실행 결과**
```
...
[retry] try count=1/5
[retry] try count=2/5
```

실행 결과를 보면 5번쨰 문제가 발생했을 때 재시도 덕분에 문제가 복구 되고, 정상 응답되는 것을 확인할 수 있다.

**참고**  
스프링이 제공하는 `@Transactional`은 가장 대표적인 AOP이다.


</details>


# [13. 스프링 AOP - 실무 주의사항](./13.spring-aop-practical-note)

<details> <summary> 1. 프록시와 내부 호출 - 문제 </summary>

## 1. 프록시와 내부 호출 - 문제

스프링은 프록시 방식의 AOP를 사용한다.  
따라서 AOP를 적용하려면 항상 프록시를 통해서 대상 객체(Target)을 호출해야 한다.  
이렇게 해야 프록시에서 먼저 어드바이스를 호출하고, 이후에 대상 객체를 호출한다.  
만약 프록시를 거치지 않고 대상 객체를 직접 호출하게 되면 AOP가 적용되지 않고, 어드바이스도 호출되지 않는다.

AOP를 적용하면 스프링은 대상 객체 대신에 프록시를 스프링 빈으로 등록한다.   
따라서 스프링은 의존관계 주입시에 항상 프록시 객체를 주입한다.  
프록시 객체가 주입되기 떄문에 대상 객체를 직접 호출하는 문제는 일반적으로 발생하지 않는다.  
하지만 대상 객체의 내부에서 메서드 호출이 발생하면 프록시를 거치지 않고 대상 객체를 직접 호출하는 문제가 발생한다.  
반드시 한번은 만나서 고생하는 문제이기 떄문에 꼭 이해하고 넘어가자.

예제를 통해서 내부 호출이 발생할 떄 어떤 문제가 발생하는지 알아보자.  
먼저 내부 호출이 발생하는 예제를 만들어보자.

**CallServiceV0**
```java
package hello.aop.internalcall;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Component;
@Slf4j
@Component
public class CallServiceV0 {
 public void external() {
 log.info("call external");
 internal(); //내부 메서드 호출(this.internal())
 }
 public void internal() {
 log.info("call internal");
 }
}
```
`CallServiceV0.external()`을 호출하면 내부에서 `internal()`이라는 자기 자신의 메서드를 호출한다.  
자바 언어에서 메서드를 호출할 때 대상을 지정하지 않으면 앞에 자기 자신의 인스턴스를 뜻하는 `this`가 붙게 된다.  
그러니까 여기서는 `this.internal()`이라고 이해하면 된다.

**CallLogAspect**
```java
package hello.aop.internalcall.aop;
import lombok.extern.slf4j.Slf4j;
import org.aspectj.lang.JoinPoint;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Before;
@Slf4j
@Aspect
public class CallLogAspect {
 @Before("execution(* hello.aop.internalcall..*.*(..))")
 public void doLog(JoinPoint joinPoint) {
 log.info("aop={}", joinPoint.getSignature());
 }
}
```
`CallServiceV0`에 AOP를 적용하기 위해서 간단한 `Aspect`를 하나 만들자.

**CalServiceV0Test**
```java
package hello.aop.internalcall;
import hello.aop.internalcall.aop.CallLogAspect;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.context.annotation.Import;
@Import(CallLogAspect.class)
@SpringBootTest
class CallServiceV0Test {
 @Autowired
 CallServiceV0 callServiceV0;
 @Test
 void external() {
 callServiceV0.external();
 }
 @Test
 void internal() {
 callServiceV0.internal();
 }
}
```
이제 앞서 만든 `CallServiceV0`을 실행할 수 있는 테스트 코드를 만들자.  
- `@Import(CallLogAspect.class)`: 앞서 만든 간단한 `Aspect`를 스프링 빈으로 등록한다. 이렇게 해서 `CallServiceV0`에 AOP 프록시를 적용한다.
- `@SpringBootTest`: 내부에 컴포넌트 스캔을 포함하고 있따. `CallServiceV0`에 `@Component`가 붙어 있으므로 스프링 빈 등록 대사잉 된다.

먼저 `callServiceV0.external()`을 실행해보자. 이 부분이 중요하다.

**실행 결과 - external()**
```
1. //프록시 호출
2. CallLogAspect : aop=void hello.aop.internalcall.CallServiceV0.external()
3. CallServiceV0 : call external
4. CallServiceV0 : call internal
```
![image](https://user-images.githubusercontent.com/28394879/142399420-915b6de3-c970-4fda-81b3-5ab74019672f.png)

실행 결과를 보면 `callServiceV0.external()`을 실행할 때는 프록시를 호출한다.  
따라서 `CallLogAspect` 어드바이스가 호출된 것을 확인할 수 있다.  
그리고 AOP Proxy는 `target.external()`을 호출한다.  
그런데 여기서 문제는 `callServiceV0.external()`안에서 `internal()`을 호출할 때 발생한다.  
이떄는 `CallLogAspect` 어드바이스가 호출되지 않는다.

자바 언어에서 메서드 앞에 별도의 참조가 없으면 `this`라는 뜻으로 자기 자신의 인스턴스를 가리킨다.  
결과적으로 자기 자신의 내부 메서드를 호출하는 `this.internal()`이 되는데, 여기서 `this`는 실제 대상 객체(target)의 인스턴스를 뜻한다.  
결과적으로 이러한 내부 호출은 프록시를 거치지 않는다.  
따라서 어드바이스도 적용할 수 없다.

이번에는 외부에서 `internal()`을 호출하는 테스트를 실행해보자.

**실행 결과 - internal()**
```
CallLogAspect : aop=void hello.aop.internalcall.CallServiceV0.internal()
CallServiceV0 : call internal
```

![image](https://user-images.githubusercontent.com/28394879/142399949-a68f3877-6b28-4c53-921a-d74c81a6fbd4.png)

외부에서 호출하는 경우 프록시를 거치기 때문에 `internal()`도 `CallLogAspect` 어드바이스가 적용된 것을 확인할 수 있다.

**프록시 방식의 AOP 한계**  
스프링은 프록시 방식의 AOP를 사용한다. 프록시 방식의 AOP는 메서드 내부 호출에 프록시를 적용할 수 없다.  
지금부터 이 문제를 해결하는 방법을 하나씩 알아보자.

> 참고  
> 실제 코드에 AOP를 직접 적용하는 AspectJ를 사용하면 이런 문제가 발생하지 않는다.  
> 프록시를 통하는 것이 아니라 해당 코드에 직접 AOP 적용 코드가 붙어 있기 때문에 내부 호출과 무관하게 AOP를 적용할 수 있다.  
> 하지만 로드 타임 위빙 등을 사용해야 하는데, 설정이 복잡하고 JVM 옵션을 주어야 하는 부담이 있다.  
> 그리고 지금부터 설명할 프록시 방식의 AOP에서 내부 호출에 대응할 수 있는 대안들도 있따.
> 
> 이런 이유로 AspectJ를 직접 사용하는 방법은 실무에서는 거의 사용하지 않는다.  
> 스프링 애플리케이션과 함께 직접 AspectJ 사용하는 방법은 스프링 공식 메뉴얼을 참고하자.
> https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#aop-using-aspectj

</details>

<details> <summary> 2. 프록시와 내부 호출 - 대안1 자기 자신 주입 </summary>

## 2. 프록시와 내부 호출 - 대안1 자기 자신 주입

내부 호출을 해결하는 가장 간단한 방법은 자기 자신을 의존관계 주입 받는 것이다.

**CallServiceV1**
```java
package hello.aop.internalcall;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;
/**
 * 참고: 생성자 주입은 순환 사이클을 만들기 때문에 실패한다.
 */
@Slf4j
@Component
public class CallServiceV1 {
 private CallServiceV1 callServiceV1;
 @Autowired
 public void setCallServiceV1(CallServiceV1 callServiceV1) {
 this.callServiceV1 = callServiceV1;
 }
 public void external() {
 log.info("call external");
 callServiceV1.internal(); //외부 메서드 호출
 }
 public void internal() {
 log.info("call internal");
 }
}
```
`callServiceV1`를 수정자를 통해서 주입 받는 것을 확인할 수 있다.  
스프링에서 AOP가 적용된 대상을 의존관계 주입 받으면 주입 받은 대상은 실제 자신이 아니라 프록시 객체이다.  
`external()`을 호출하면 `callServiceV1.internal()`를 호출하게 된다.  
주입 받은 `callServiceV1`은 프록시이다.  
따라서 프록시를 통해서 AOP를 적용할 수 있다.  

참고로 이 경우 생성자 주입시 오류가 발생한다. 본인을 생성하면서 주입해야 하기 떄문에 순환 사이클이 만들어진다.  
반면에 수정자 주입은 스프링이 생성된 이후에 주입할 수 있기 때문에 오류가 발생하지 않는다.

**CallServiceV1Test**
```java
package hello.aop.internalcall;
import hello.aop.internalcall.aop.CallLogAspect;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.context.annotation.Import;
@Import(CallLogAspect.class)
@SpringBootTest
class CallServiceV1Test {
 @Autowired
 CallServiceV1 callServiceV1;
 @Test
 void external() {
 callServiceV1.external();
 }
}
```

**실행 결과**
```
CallLogAspect : aop=void hello.aop.internalcall.CallServiceV1.external()
CallServiceV2 : call external
CallLogAspect : aop=void hello.aop.internalcall.CallServiceV1.internal()
CallServiceV2 : call internal
```
![image](https://user-images.githubusercontent.com/28394879/142402778-fafe4d84-f79f-4b40-a576-de39f3ec971f.png)

실행 결과를 보면 이제는 `internal()`을 호출할 때 자기 자신의 인스턴스를 호출하는 것이 아니라 프록시 인스턴스를 통해서 호출하는 것을 확인할 수 있다.  
당연히 AOP도 잘 적용된다.

</details>

<details> <summary> 3. 프록시와 내부 호출 - 대안2 지연 조회 </summary>

## 3. 프록시와 내부 호출 - 대안2 지연 조회

앞서 생성자 주입이 실패하는 이유는 자기 자신을 생성하면서 주입해야 하기 떄문이다.  
이 경우 수정자 주입을 사용하거나 지금부터 설명하는 지연 조회를 사용하면 된다.  
스프링 빈을 지연해서 조회하면 되는데, `ObjectProvider(Provider)`, `ApplicationContext`를 사용하면 된다.

**CallServiceV2**
```java
package hello.aop.internalcall;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.ObjectProvider;
import org.springframework.context.ApplicationContext;
import org.springframework.stereotype.Component;
/**
 * ObjectProvider(Provider), ApplicationContext를 사용해서 지연(LAZY) 조회
 */
@Slf4j
@Component
@RequiredArgsConstructor
public class CallServiceV2 {
// private final ApplicationContext applicationContext;
 private final ObjectProvider<CallServiceV2> callServiceProvider;
 public void external() {
 log.info("call external");
// CallServiceV2 callServiceV2 =
applicationContext.getBean(CallServiceV2.class);
 CallServiceV2 callServiceV2 = callServiceProvider.getObject();
 callServiceV2.internal(); //외부 메서드 호출
 }
 public void internal() {
 log.info("call internal");
 }
}
```
`ObjectProvider`는 기본편에서 학습한 내용이다.  
`ApplicationContext`는 너무 많은 기능을 제공한다.  
`ObjectProvider`는 객체를 스프링 컨테이너에서 조회하는 것을 스프링 빈 생성 시점이 아니라 실제 객체를 사용하는 시점으로 지연할 수 있다.  
`callServiceProvider.getObject()`를 호출하는 시점에 스프링 컨테이너에서 빈을 조회한다.  
여기서는 자기 자신을 주입 받는 것이 아니기 떄문에 순환 사이클이 발생하지 않는다.

**CallServiceV2Test**
```java
package hello.aop.internalcall;
import hello.aop.internalcall.aop.CallLogAspect;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.context.annotation.Import;
@Import(CallLogAspect.class)
@SpringBootTest
class CallServiceV2Test {
 @Autowired
 CallServiceV2 callServiceV2;
 @Test
 void external() {
 callServiceV2.external();
 }
}
```
**실행 결과**
```
CallLogAspect : aop=void hello.aop.internalcall.CallServiceV2.external()
CallServiceV2 : call external
CallLogAspect : aop=void hello.aop.internalcall.CallServiceV2.internal()
CallServiceV2 : call internal
```


</details>

<details> <summary> 4. 프록시와 내부 호출 - 대안3 구조 변경 </summary>

## 4. 프록시와 내부 호출 - 대안3 구조 변경

앞선 방법들은 자기 자신을 주입하거나 또는 `Provider`를 사용해야 하는 것 처럼 조금 어색한 모습을 만들었다.  
가장 나은 대안은 내부 호출이 발생하지 않도록 구조를 변경하느 것이다.  
실제 이 방법을 가장 권장한다.

**CallServiceV3**
```java
package hello.aop.internalcall;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Component;
/**
 * 구조를 변경(분리)
 */
@Slf4j
@Component
@RequiredArgsConstructor
public class CallServiceV3 {
 private final InternalService internalService;
 public void external() {
 log.info("call external");
 internalService.internal(); //외부 메서드 호출
 }
}
```
내부 호출을 `InternalService`라는 별도의 클래스로 분리했다.

**InternslService**
```java
package hello.aop.internalcall;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Component;
@Slf4j
@Component
public class InternalService {
 public void internal() {
 log.info("call internal");
 }
}
```

**CallServiceV3Test**
```java
package hello.aop.internalcall;
import hello.aop.internalcall.aop.CallLogAspect;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.context.annotation.Import;
@Import(CallLogAspect.class)
@SpringBootTest
class CallServiceV3Test {
 @Autowired
 CallServiceV3 callServiceV3;
 @Test
 void external() {
 callServiceV3.external();
 }
}
```

**실행 결과**
```
CallLogAspect : aop=void hello.aop.internalcall.CallServiceV3.external()
CallServiceV3 : call external
CallLogAspect : aop=void hello.aop.internalcall.InternalService.internal()
InternalService : call internal
```

![image](https://user-images.githubusercontent.com/28394879/142406171-dea7baf5-da26-4ea4-877e-01af5e1aa7d6.png)

내부 호출 자체가 사라지고, `callService` -> `internalService`를 호출하는 구조로 변경되었다.  
덕분에 자연스럽게 AOP가 적용된다.

여기서 구조를 변경한다는 것은 어떻게 단순하게 분리하는 것 뿐만 아니라 다양한 방법들이 있을 수 있따.

예를 들어서 다음과 같이 클라이언트에서 둘다 호출하는 것이다.
`클라이언트` -> `external()`
`클라이언트` -> `internal()`

물론 이 경우 `external()`에서 `internal()`을 내부 호출하지 않도록 코드를 변경해야 한다.  
그리고 클라이언트 `external()`, `internal()`을 모두 호출하도록 구조를 변경하면 된다.   
(물론 가능한 경우에 한해서)

> 참고  
> AOP는 주로 트랜잭션 적용이나 주요 컴포넌트의 로그 출력 기능에 사용된다.   
> 쉽게 이야기해서 인터페이스에 메스드가 나올 정도의 규모에 AOP를 적용하는 것이 적당하다.  
> 더 풀어서 이야기하면 AOP는 `public` 메서드에만 적용한다.  
> `private`메서드처럼 작은 단위에서 AOP를 적용하지 않는다.  
> AOP적용을 위해 `private`메서드를 외부 클래스로 변경하고 `public`으로 변경하는 일은 거의 없다.  
> 그러나 위 예제와 같이 `public`메서드에 `public`메서드를 내부 호출하는 경우에는 문제가 발생한다.  
> 실무에서 꼭 한번은 만나는 문제이기에 이번에 정리했다.  
> AOP가 잘 적용되지 않으면 내부 호출을 의심해보자. 

</details>

<details> <summary> 5. 프록시 기술과 한계 - 타입 캐스팅 </summary>

## 5. 프록시 기술과 한계 - 타입 캐스팅

JDK 동적 프록시와 CGLIB를 사용해서 AOP 프록시를 만드는 방법에는 각각 장단점이 있다.  
JDK 동적 프록시는 인터페이스가 필수이고, 인터페이스를 기반으로 프록시를 생성한다.  
CGLIB는 구체 클래스를 기반으로 프록시를 생성한다.

물론 인터페이스가 없고 구체 클래스만 있는 경우에는 CGLIB를 사용해야 한다.  
그런데 인터페이스가 있는 경우에는 JDK 동적 프록시나 CGLIB 둘중에 하나를 선택할 수 있다.

스프링이 프록시를 만들때 제공하는 `ProxyFactory`에 `proxyTargetClass` 옵션에 따라 둘중 하나를 선택해서 프록시를 만들 수 있다.

- `proxyTargetClass=false` JDK 동적 프록시를 사용해서 인터페이스 기반 프록시 생성
- `proxyTargetClass=true` CGLIB를 사용해서 구체 클래스 기반 프록시 생성
- 참고로 옵션과 무관하게 인터페이스가 없으면 JDK 동적 프록시를 적용할 수 없으므로 CGLIB를 사용한다.

**JDK 동적 프록시 한계**  
인터페이스 기반으로 프록시를 생성하는 JDK 동적 프록시는 구체 클래스로 타입 캐스팅이 불가능한 한계가 있다.  
어떤 한계인지 코드를 통해서 알아보자.

**ProxyCastingTest**
```java
package hello.aop.proxyvs;
import hello.aop.member.MemberService;
import hello.aop.member.MemberServiceImpl;
import lombok.extern.slf4j.Slf4j;
import org.junit.jupiter.api.Test;
import org.springframework.aop.framework.ProxyFactory;
import static org.junit.jupiter.api.Assertions.assertThrows;
@Slf4j
public class ProxyCastingTest {
 @Test
 void jdkProxy() {
 MemberServiceImpl target = new MemberServiceImpl();
 ProxyFactory proxyFactory = new ProxyFactory(target);
 proxyFactory.setProxyTargetClass(false);//JDK 동적 프록시
 //프록시를 인터페이스로 캐스팅 성공
 MemberService memberServiceProxy = (MemberService)
proxyFactory.getProxy();
 log.info("proxy class={}", memberServiceProxy.getClass());
 //JDK 동적 프록시를 구현 클래스로 캐스팅 시도 실패, ClassCastException 예외 발생
 assertThrows(ClassCastException.class, () -> {
 MemberServiceImpl castingMemberService =
(MemberServiceImpl) memberServiceProxy;
 }
 );
 }
}
```

**JDK 동적 프록시**  
![image](https://user-images.githubusercontent.com/28394879/142409405-0ed06041-c54f-400e-8545-8b2f94a8ca8c.png)

**jdkProxy() 테스트**  
여기서는 `MemberServiceImpl`타입을 기반으로 JDK 동적 프록시를 생성했다.  
`MemberServiceImpl`타입은 `MemberService` 인터페이스를 구현한다.  
따라서 JDK 동적 프록시는 `MemberService`인터페이스를 기반으로 프록시를 생성한다.  
이 프록시를 `JDK Proxy`라고 하자.  
여기서 `memberServiceProxy`가 바로 `JDK Proxy`이다.

**JDK 동적 프록시 캐스팅**  
![image](https://user-images.githubusercontent.com/28394879/142409587-daad1924-d103-41a5-8e4a-1db1bf727e2c.png)  

그런데 여기에서 JDK Proxy를 대상 클래스인 `MemberServiceImpl` 타입으로 캐스팅 하려고 하니 예외가 발생한다.  
왜냐하면 JDK 동적 프록시는 인터페이스를 기반으로 프록시를 생성하기 때문이다.  
JDK Proxy는 `MemberService` 인터페이스를 기반으로 생성된 프록시이다.  
따라서 JDK Proxy는 `MemberService`로 캐스팅은 가능하지만 `MemberServiceImpl`이 어떤 것인지 전혀 알지 못한다.  
따라서 `MemberServiceImpl`타입으로는 캐스팅이 불가능하다.  
캐스팅을 시도하면 `ClassCastException.class` 예외가 발생한다.

이번에는 CGLIB을 사용해보자.  

**ProxyCastingTest - 추가**
```java
@Test
void cglibProxy() {
 MemberServiceImpl target = new MemberServiceImpl();
 ProxyFactory proxyFactory = new ProxyFactory(target);
 proxyFactory.setProxyTargetClass(true);//CGLIB 프록시
 //프록시를 인터페이스로 캐스팅 성공
 MemberService memberServiceProxy = (MemberService) proxyFactory.getProxy();
 log.info("proxy class={}", memberServiceProxy.getClass());
 //CGLIB 프록시를 구현 클래스로 캐스팅 시도 성공
 MemberServiceImpl castingMemberService = (MemberServiceImpl)
memberServiceProxy;
}
```

**CGLIB 프록시**  
![image](https://user-images.githubusercontent.com/28394879/142409948-98a975fd-7134-4cdd-ab27-125c84f0487c.png)


**cglibProxy() 테스트**  
`MemberServiceImpl` 타입을 기반으로 CGLIB 프록시를 생성했다.  
`MemberServiceImpl` 타입은 `MemberService` 인터페이스를 구현햇다.  
CGLIB는 구체 클래스를 기반으로 프록시를 생성한다.  
따라서 CGLIB는 `MemberServiceImpl` 구체 클래스를 기반으로 프록시를 생성한다.  
이 프록시를 CGLIB Proxy라고 하자.  
여기서 `memberServiceProxy`가 바로 CGLIB Proxy이다. 

**CGLIB 프록시 캐스팅**  
![image](https://user-images.githubusercontent.com/28394879/142410210-da561b16-b701-4d86-9a63-5cb72c80d7bd.png)

여기에서 CGLIB Proxy를 대상 클래스인 `MemberServiceImpl` 타입으로 캐스팅하면 성공한다.  
왜냐하면 CGLIB는 구체 클래스를 기반으로 프록시를 생성하기 떄문이다.  
CGLIB Proxy는 `MemberServiceImpl` 구체 클래스를 기반으로 생성된 클래스이다.  
따라서 CGLIB Proxy는 `MemberServiceImpl`은 물론이고, `MemberServiceImpl`이 구현한 인터페이스인 `MemberService`로도 캐스팅 할 수 있다.

**정리**  
JDK 동적 프록시는 대상 객체인 `MemberServiceImpl`로 캐스팅 할 수 없다.  
CGLIB 동적 프록시는 대상 객체인 `MemberServieImpl`로 캐스팅 할 수 있다.

그런데 프록시를 캐스팅 할 일이 많지 않을 것 같은데 왜 이야기를 하는 것일까?   
진짜 문제는 의존관계 주입시에 발생한다.

</details>

<details> <summary> 6. 프록시 기술과 한계 - 의존관계 주입 </summary>

## 6. 프록시 기술과 한계 - 의존관계 주입

JDK 동적 프록시를 사용하면서 의존관계 주입을 할 때 어떤 문제가 발생하는지 코드로 알아보자.

**ProxyDIAspect**  
주의: 테스트 패키지에 위치한다.  
```java
package hello.aop.proxyvs.code;
import lombok.extern.slf4j.Slf4j;
import org.aspectj.lang.JoinPoint;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Before;
@Slf4j
@Aspect
public class ProxyDIAspect {
 @Before("execution(* hello.aop..*.*(..))")
 public void doTrace(JoinPoint joinPoint) {
 log.info("[proxyDIAdvice] {}", joinPoint.getSignature());
 }
}
```

AOP 프록시 생성을 위해 간단한 `Aspect`를 만들자.

**ProxyDITest**
```java
package hello.aop.proxyvs;
import hello.aop.member.MemberService;
import hello.aop.member.MemberServiceImpl;
import hello.aop.proxyvs.code.ProxyDIAspect;
import lombok.extern.slf4j.Slf4j;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.context.annotation.Import;
@Slf4j
@SpringBootTest(properties = {"spring.aop.proxy-target-class=false"}) //JDK 동적
프록시, DI 예외 발생
//@SpringBootTest(properties = {"spring.aop.proxy-target-class=true"}) //CGLIB
프록시, 성공
@Import(ProxyDIAspect.class)
public class ProxyDITest {
 @Autowired MemberService memberService; //JDK 동적 프록시 OK, CGLIB OK
 @Autowired MemberServiceImpl memberServiceImpl; //JDK 동적 프록시 X, CGLIB OK
 @Test
 void go() {
 log.info("memberService class={}", memberService.getClass());
 log.info("memberServiceImpl class={}", memberServiceImpl.getClass());
 memberServiceImpl.hello("hello");
 }
}
```

**코드 설명**  
- `@SpringBootTest`: 내부에 컴포넌트 스캔을 포함하고 있다. `MemberServiceImpl`에 `@Component`가 붙어있으므로 스프링 빈 등록 대상이 된다.  
- `properties = {"spring.aop.proxy-target-class=false"} : application.properties`에 설정하는 대신에 해당 테스트에서만 설정을 임시로 적용한다. 이렇게 하면 각 테스트마다 다른 설정을 손쉽게 적용할 수 있다.
- `spring.aop.proxy-target-class=false`: 스프링 AOP 프록시를 생성할 때 JDK 동적 프록시를 우선 생성한다. 물론 인터페이스가 없다면 CGLIB를 사용한다.
- `@Import(ProxyDIAspect.class)`: 앞서 만든 Aspect를 스프링 빈으로 등록한다.

### JDK 동적 프록시에 구체 클래스 타입 주입  
JDK 동적 프록시에 구체 클래스 타입을 주입할 때 어떤 문제가 발생하는지 지금부터 확인해보자.

**실행**  
먼저 `spring.aop.proxy-target-class=false`설정을 사용해서 스프링 AOP가 JDK동적 프록시를 사용하도록 했다.  
이렇게 실행하면 다음과 같이 오류가 발생한다.

**실행 결과**
```
BeanNotOfRequiredTypeException: Bean named 'memberServiceImpl' is expected to
be of type 'hello.aop.member.MemberServiceImpl' but was actually of type
'com.sun.proxy.$Proxy54'
```

타입과 관련된 예외가 발생한다. 자세히 읽어보면 `memberServiceImpl`에 주입되길 기대하는 타입은 `hello.aop.member.MemberServiceImpl`이지만 실제 넘어온 타입은 `com.sun.proxy.$Proxy54`이다.  
따라서 타입 예외가 발생한다고 한다.

![image](https://user-images.githubusercontent.com/28394879/142413004-92eace98-0484-4e49-9449-508b961c71fa.png)

- `@Autowired MemberService memberService`: 이 부분은 문제가 없다. JDK Proxy는 `MemberService`인터페이스를 기반으로 만들어진다. 따라서 해당 타입으로 캐스팅 할 수 있따. 
  - `MemberService = JDK Proxy`가 성립한다.
- `@Autowired MemberServiceImpl memberServiceImpl`: 문제는 여기다. JDK Proxy는 `MemberService` 인터페이스를 기반으로 만들어진다. 따라서 `MemberServiceImpl` 타입이 뭔지 전혀 모른다. 그래서 해당 타입에 주입할 수 없다.  
  - `MemberServiceImpl = JDK Proxy`가 성립하지 않는다.

### CGLIB 프록시에 구체 클래스 타입 주입

이번에는 JDK 동적 프록시 대신에 CGLIB를 사용해서 프록시를 적용해보자. 다음과 같이 주석을 반대로 걸어보자.

```java
//@SpringBootTest(properties = {"spring.aop.proxy-target-class=false"}) //JDK
동적 프록시, DI 예외 발생
@SpringBootTest(properties = {"spring.aop.proxy-target-class=true"}) //CGLIB
프록시, 성공
```

실행해보면 정상 동작하는 것을 확인할 수 있다.

![image](https://user-images.githubusercontent.com/28394879/142413390-641207d8-21de-4045-8c6b-574bf6ad005b.png)

- `@Autowired MemberService memberService`: CGLIB Proxy는 `MemberServiceImpl` 구체 클래스를 기반으로 만들어진다. `MemberServiceImpl`은 `MemberService` 인터페이스를 구현했기 떄문에 해당 타입으로 캐스팅할 수 있다.
  - `MemberService = CGLIB Proxy`가 성립한다.
- `@Autowired MemberServiceImpl memberServiceImpl`: CGLIB Proxy는 `MemberServiceImpl` 구체 클래스를 기반으로 만들어진다. 따라서 해당 타입으로 캐스팅 할 수 있따.
  - `MemberServiceImpl = CGLIB Proxy`가 성립한다.

**정리**  
JDK 동적 프록시는 대상 객체인 `MemberServiceImpl` 타입에 의존관계를 주입할 수 없다.  
CGLIB 프록시는 대상 객체인 `MemberServiceImpl` 타입에 의존관계 주입을 할 수 있다.

지금까지 JDK 동적 프록시가 가지는 한계점을 알아보았다.  
실제로 개발할 때는 인터페이스가 있으면 인터페이스를 기반으로 의존관계 주입을 받는 것이 맞다.  
DI의 장점이 무엇인가? DI 받는 클라이언트 코드의 변경 없이 구현 클래스를 변경할 수 있는 것이다.  
이렇게 하려면 인터페이스를 기반으로 의존관계를 주입 받아야 한다.  
`MemberServiceImpl` 타입으로 의존관계 주입을 받는 것 처럼 구현 클래스에 의존관계를 주입하면 향후 구현 클래스를 변경할 때 의존관계 주입을 받는 클라이언트의 코드도 함께 변경해야 한다.  
따라서 올바르게 잘 설계된 애플리케이션이라면 이런 문제가 자주 발생하지 않는다.  
그럼에도 불구하고 테스트, 또는 여러가지 이유로 AOP 프록시가 적용된 구체 클래스를 직접 의존관계 주입 받아야 하는 경우가 있을 수 있다.  
이떄는 CGLIB를 통해 구체 클래스 기반으로 AOP 프록시를 적용하면 된다.  

여기까지 듣고보면 CGLIB를 사용하는 것이 좋아보인다.  
CGLIB를 사용하면 사실 이런 고민 자체를 하지 않아도 된다.  
다음번에는 CGLIB의 단점을 알아보자.




</details>

<details> <summary> 7. 프록시 기술과 한계 - CGLIB </summary>

## 7. 프록시 기술과 한계 - CGLIB

스프링에서 CGLIB는 구체 클래스를 상속 받아서 AOP 프록시를 생성할 때 사용한다.  
CGLIB는 구체 클래스를 상속 받기 떄문에 다음과 같은 문제가 있다.

**CGLIB 구체 클래스 기반 프록시 문제점**
- 대상 클래스에 기본 생성자 필수
- 생성자 2번 호출 문제
- final 키워드 클래스, 메서드 사용 불가

하나씩 자세히 알아보자.

### 대상 클래스에 기본 생성자 필수  
CGLIB는 구체 클래스를 상속 받는다.   
자바 언어에서 상속을 받으면 자식 클래스의 생성자를 호출할 때 자식 클래스의 생성자에서 부모 클래스의 생성자도 호출해야 한다.  
(이 부분이 생략되어 있다면 자식 클래스의 생성자 첫줄에 부모 클래스의 기본 생성자를 호출하는 `super()`가 자동으로 들어간다.) 이 부분은 자바 문법 규약이다.  
CGLIB를 사용할 떄 CGLIB가 만드는 프록시의 생성자는 우리가 호출하는 것이 아니다.  
CGLIB 프록시는 대상 클래스를 상속 받고, 생성자에서 대상 클래스의 기본 생성자를 호출한다.  
따라서 대상 클래스에 기본 생성자를 만들어야 한다. (기본 생성자는 파라미터가 하나도 없는 생성자를 뜻한다. 생성자가 하나도 없으면 자동으로 만들어진다.)

### 생성자 2번 호출 문제  
CGLIB는 구체 클래스를 상속 받는다.  
자바 언어에서 상속을 받으면 자식 클래스의 생성자를 호출할 때 부모 클래스의 생성자도 호출해야 한다.  
그런데 왜 2번일까?  
1. 실제 target의 객체를 생성할 때 
2. 프록시 객체를 생성할 때 부모 클래스의 생성자 호출

![image](https://user-images.githubusercontent.com/28394879/142416215-417abc8b-9a2d-4949-bb04-b25299a85743.png)

### final 키워드 클래스, 메서드 사용 불가   
final 키워드가 클래스에 있으면 상속이 불가능하고, 메서드에 있으면 오버라이딩이 불가능하다.  
CGLIB는 상속을 기반으로 하기 떄문에 두 경우 프록시가 생성되지 않거나 정상 동작하지 않는다.

프레임워크 같은 개발이 아니라 일반적인 웹 애플리케이션을 개발할 때는 `final` 키워드를 잘 사용하지 않는다.   
따라서 이 부분이 특별히 문제가 되지는 않는다.

**정리**  
JDK 동적 프록시는 대상 클래스 타입으로 주입할 떄 문제가 있고, CGLIB는 대상 클래스에 기본 생성자 필수, 생성자 2번 호출 문제가 있다.

그렇다면 스프링은 어떤 방법을 권장할까? 



</details>

<details> <summary> 8. 프록시 기술과 한계 - 스프링의 해결책 </summary>

## 8. 프록시 기술과 한계 - 스프링의 해결책

스프링은 AOP 프록시 생성을 편리하게 제공하기 위해 오랜 시간 고민하고 문제들을 해결해왔다.

### 스프링의 기술 선택 변화 

**스프링은 3.2 CGLIB를 스프링 내부에 함께 패키징**  
 CGLIB를 사용하려면 CGLIB 라이브러리가 별도로 필요했다.  
스프링은 CGLIB 라이브러리를 스프링 내부에 함께 패키징해서 별도의 라이브러리 추가 없이 CGLIB를 사용할 수 있게 되었다.  
`CGLIB spring-core org.springframework`

**CGLIB 기본 생성자 필수 문제 해결**  
스프링 4.0부터 CGLIB의 기본 생성자가 필수인 문제가 해결되었다.  
`objenesis`라는 특별한 라이브러리를 사용해서 기본 생성자 없이 객체 생성이 가능하다.  
참고로 이 라이브러리는 생성자 호출 없이 객체를 생성할 수 있게 해준다.

**생성자 2번 호출 문제**  
스프링 4.0부터 CGLIB의 생성자 2번 호출 문제가 해결되었다.  
이것도 역시 `objenesis`라는 특별한 라이브러리 덕분에 가능해졌다.  
이제 생성자가 1번만 호출된다.

**스프링 부트 2.0 - CGLIB 기본 사용**  
스프링 부트 2.0 버전부터 CGLIB를 기본으로 사용하도록 했다.  
이렇게 해서 구체 클래스 타입으로 의존관계를 주입하는 문제를 해결했다.  
스프링 부트는 별도의 설정이 없다면 AOP를 적용할 때 기본적으로 `proxyTargetClass=true`로 설정해서 사용한다.  
따라서 인터페이스가 있어도 JDK 동적 프록시를 사용하는 것이 아니라 항상 CGLIB를 사용해서 구체클래스를 기반으로 프록시를 생성한다.  
물론 스프링은 우리에게 선택권을 열어주기 때문에 다음과 같이 설정하면 JDK 동적 프록시도 사용할 수 있다.

`application.properties`
```
spring.aop.proxy-target-class=false
```

**정리**  
스프링은 최종적으로 스프링 부트 2.0에서 CGLIB를 기본으로 사용하도록 결정했다.  
CGLIB를 사용하면 JDK 동적 프록시에서 동작하지 않는 구체 클래스 주입이 가능하다.  
여기에 추가로 CGLIB의 단점들이 이제는 많이 해결되었다.  
CGLIB의 남은 문제라면 `final`클래스나 `final`메서드가 있는데, AOP를 적용할 대상에는 `final`클래스나 `final`메서드를 잘 사용하지는 않으므로 이 부분은 크게 문제가 되지는 않는다.

개발자 입장에서 보면 사실 어떤 프록시 기술을 사용하든 상관이 없다.  
JDK 동적 프록시든 CGLIB든 또는 어떤 새로운 프록시 기술을 사용해도 된다.   
심지어 클라이언트 입장에서 어떤 프록시 기술을 사용하는지 모르고 잘 동작하는 것이 가장 좋다.  
단지 문제 업속, 개발하기에 편리하면 되는 것이다.

마지막으로 `ProxyDITest`를 다음과 같이 변경해서 아무런 설정 없이 싫애해보면 CGLIB가 기본으로 사용되는 것을 확인할 수 있다.

```java
@Slf4j
//@SpringBootTest(properties = {"spring.aop.proxy-target-class=false"}) //JDK
동적 프록시, DI 예외 발생
//@SpringBootTest(properties = {"spring.aop.proxy-target-class=true"}) //CGLIB
프록시, 성공
@SpringBootTest //추가
@Import(ProxyDIAspect.class)
public class ProxyDITest {...}
```

`@SpringBootTest`이 부분을 추가해야 한다. `application.properties`에 `spring.aop.proxy-target-class`관련 설정이 없어야 한다.

**실행 결과**
```
memberService class=class hello.aop.member.MemberServiceImpl$
$EnhancerBySpringCGLIB$$83e257b3
memberServiceImpl class=class hello.aop.member.MemberServiceImpl$
$EnhancerBySpringCGLIB$$83e257b3
```

</details>

