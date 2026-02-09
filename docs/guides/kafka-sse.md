# Kafka 이벤트 + SSE 실시간 알림

## 핵심 개념

### Kafka
Apache Kafka는 분산 메시지 큐로, **Producer → Topic(Partition) → Consumer** 구조로 동작한다. 메시지는 토픽의 파티션에 저장되며, 같은 파티션 키를 가진 메시지는 동일 파티션에 순서대로 저장되어 **순서 보장**이 된다.

### SSE (Server-Sent Events)
SSE는 HTTP 기반의 단방향 실시간 통신 프로토콜이다. 서버에서 클라이언트로 이벤트를 push할 수 있으며, WebSocket과 달리 HTTP/1.1만으로 동작하고 자동 재연결을 지원한다.

### EasyPay에서의 역할
송금/결제 완료 시 Kafka 이벤트를 발행하고, Consumer가 이를 소비하여 SSE로 프론트엔드에 실시간 알림을 전달한다.

```
TransferService → EventPublisher → Kafka Topic → NotificationConsumer → NotificationService → SSE → Frontend
```

## 프로젝트 적용

### Kafka 토픽 구성

| 토픽 | 파티션 수 | 용도 |
|------|----------|------|
| `transfer-events` | 6 | 송금 이벤트 (TRANSFER_COMPLETED) |
| `payment-events` | 6 | 결제 이벤트 (PAYMENT_COMPLETED) |
| `notification-events` | 3 | 알림 이벤트 (예약) |
| `settlement-events` | 3 | 정산 이벤트 (예약) |

파티션 수 결정 기준: 송금/결제는 트래픽이 높아 6개, 알림/정산은 상대적으로 적어 3개로 설정했다.

### 이벤트 발행 (EventPublisher)

비동기 발행으로 메인 트랜잭션을 블로킹하지 않는다.

```java
// EventPublisher.java:24-41
public void publish(String topic, String key, EventMessage message) {
    String payload = objectMapper.writeValueAsString(message);
    kafkaTemplate.send(topic, key, payload)
            .whenComplete((result, ex) -> {
                if (ex != null) {
                    log.error("Kafka 이벤트 발행 실패: topic={}, eventType={}",
                            topic, message.getEventType());
                } else {
                    log.debug("Kafka 이벤트 발행 성공: partition={}",
                            result.getRecordMetadata().partition());
                }
            });
}
```

**파티션 키 전략**: `senderId`(사용자 ID)를 key로 사용하여 같은 사용자의 이벤트가 동일 파티션에 순서대로 저장된다.

### 이벤트 메시지 구조 (EventMessage)

```java
// EventMessage.java
{
    "eventId": "uuid",           // 이벤트 고유 ID
    "eventType": "TRANSFER_COMPLETED",  // 이벤트 유형
    "timestamp": "2025-02-10T...",      // 발생 시각
    "data": {                           // 도메인 데이터
        "senderId": 123,         // 사용자 ID (계좌 ID 아님!)
        "receiverId": 456,
        "amount": 50000,
        "transferCode": "TRF20250210001"
    }
}
```

### 이벤트 소비 (NotificationConsumer)

두 토픽을 소비하여 SSE 알림으로 전달한다.

```java
// NotificationConsumer.java
@KafkaListener(topics = KafkaTopics.TRANSFER_EVENTS, groupId = "notification-group")
public void handleTransferEvent(String payload) {
    EventMessage event = objectMapper.readValue(payload, EventMessage.class);
    NotificationEvent notifEvent = NotificationEvent.ofTransfer(...);

    // 송금자에게: "송금이 완료되었습니다."
    notificationService.sendToUser(notifEvent.getSenderId(), "transfer",
            NotificationResponse.transferSent(notifEvent));

    // 수취자에게: "입금이 되었습니다."
    notificationService.sendToUser(notifEvent.getReceiverId(), "transfer",
            NotificationResponse.transferReceived(notifEvent));
}

@KafkaListener(topics = KafkaTopics.PAYMENT_EVENTS, groupId = "notification-group")
public void handlePaymentEvent(String payload) {
    // 결제자에게: "결제가 완료되었습니다."
    notificationService.sendToUser(notifEvent.getSenderId(), "payment",
            NotificationResponse.paymentCompleted(notifEvent));
}
```

### SSE 구독 관리 (NotificationService)

`ConcurrentHashMap`으로 사용자별 SSE 연결을 관리한다.

```java
// NotificationService.java
private static final long SSE_TIMEOUT = 30 * 60 * 1000L; // 30분
private final Map<Long, SseEmitter> emitters = new ConcurrentHashMap<>();

public SseEmitter subscribe(Long userId) {
    SseEmitter emitter = new SseEmitter(SSE_TIMEOUT);
    emitters.put(userId, emitter);

    emitter.onCompletion(() -> emitters.remove(userId));
    emitter.onTimeout(() -> emitters.remove(userId));
    emitter.onError(e -> emitters.remove(userId));

    sendToUser(userId, "connect", "SSE 연결 성공");
    return emitter;
}

public void sendToUser(Long userId, String eventName, Object data) {
    SseEmitter emitter = emitters.get(userId);
    if (emitter != null) {
        emitter.send(SseEmitter.event().name(eventName).data(data));
    }
    // 미연결 사용자는 무시 (at-most-once)
}
```

## 코드 위치 및 구조

```
src/main/java/com/easypay/
├── infra/kafka/
│   ├── KafkaConfig.java          # 토픽 자동 생성 (4개 토픽)
│   ├── KafkaTopics.java          # 토픽명 상수
│   ├── EventPublisher.java       # 비동기 이벤트 발행
│   └── EventMessage.java         # 이벤트 메시지 래퍼
├── notification/
│   ├── consumer/
│   │   └── NotificationConsumer.java  # Kafka 소비자 (2개 토픽 리스닝)
│   ├── service/
│   │   └── NotificationService.java   # SSE 이미터 관리
│   ├── controller/
│   │   └── NotificationController.java  # GET /subscribe (SSE 엔드포인트)
│   └── dto/
│       ├── NotificationEvent.java     # 소비된 이벤트 DTO
│       └── NotificationResponse.java  # SSE 전송용 응답 DTO
```

## 설정

```yaml
# application.yml (Spring Kafka 기본 설정)
spring:
  kafka:
    bootstrap-servers: localhost:9092
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.apache.kafka.common.serialization.StringSerializer
      acks: all                    # 모든 복제본에 저장 확인
    consumer:
      group-id: notification-group
      auto-offset-reset: latest    # 최신 메시지부터 소비
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.apache.kafka.common.serialization.StringDeserializer

# application-docker.yml (Docker 내부 통신)
spring:
  kafka:
    bootstrap-servers: kafka:29092   # 내부 리스너 포트
```

### Kafka 듀얼 리스너 (docker-compose.yml)

```yaml
kafka:
  environment:
    KAFKA_LISTENERS: INTERNAL://0.0.0.0:29092,EXTERNAL://0.0.0.0:9092
    KAFKA_ADVERTISED_LISTENERS: INTERNAL://kafka:29092,EXTERNAL://localhost:9092
```
- **INTERNAL (29092)**: Docker 컨테이너 간 통신 (app → kafka)
- **EXTERNAL (9092)**: 호스트에서 접근 (로컬 개발, k6 테스트)

## 동작 흐름

```
[1] 송금 완료
    TransferSagaOrchestrator
    └─ eventPublisher.publish(
         KafkaTopics.TRANSFER_EVENTS,     // topic
         String.valueOf(senderId),         // partition key
         EventMessage.of("TRANSFER_COMPLETED", Map.of(
             "senderId", userId,           // USER ID (계좌 ID 아님!)
             "receiverId", receiverUserId,
             "amount", amount,
             "transferCode", code
         ))
       )

[2] Kafka 브로커
    transfer-events (6 partitions)
    └─ senderId 기반 파티션 할당
       └─ partition 0~5 중 hash(senderId) % 6

[3] Consumer 소비
    NotificationConsumer.handleTransferEvent()
    └─ JSON 역직렬화 → NotificationEvent 변환
    └─ 타입 변환: Number → Long/BigDecimal (toLong, toBigDecimal 헬퍼)

[4] SSE 전송
    NotificationService.sendToUser(userId, "transfer", response)
    └─ ConcurrentHashMap에서 SseEmitter 조회
    └─ emitter.send(SseEmitter.event().name("transfer").data(response))

[5] 프론트엔드 수신
    EventSource('/api/v1/notifications/subscribe?token=<jwt>')
    └─ event: transfer
       data: {"type":"TRANSFER_COMPLETED","message":"송금이 완료되었습니다.","amount":50000,"code":"TRF..."}
    └─ Sonner 토스트로 표시
```

## 트러블슈팅 & 주의사항

### SSE와 JWT 인증
브라우저의 `EventSource` API는 커스텀 헤더를 지원하지 않는다. 따라서 JWT를 쿼리 파라미터로 전달한다: `?token=<jwt>`. `JwtAuthenticationFilter`에서 `Authorization` 헤더를 먼저 확인하고, 없으면 `?token` 파라미터로 폴백한다.

### senderId/receiverId는 사용자 ID
Kafka 이벤트의 `senderId`/`receiverId`는 **Account ID가 아니라 User ID**이다. SSE 알림은 사용자 단위로 전송하기 때문이다. 초기 구현에서 Account ID를 전달하여 알림이 누락되는 버그가 있었다.

### at-most-once 전달 보장
현재 구현은 사용자가 SSE 미연결 상태이면 알림이 유실된다. 중요한 알림은 DB에 별도 저장하고, SSE 연결 시 미수신 알림을 조회하는 방식으로 개선할 수 있다.

### Consumer Group과 파티션
`notification-group`이라는 단일 Consumer Group을 사용한다. 서버를 스케일아웃하면 각 인스턴스가 파티션을 나눠 소비하게 되는데, SSE 이미터가 인메모리 `ConcurrentHashMap`에 저장되므로 다른 인스턴스의 사용자에게 알림을 보낼 수 없다. 스케일아웃 시 Redis Pub/Sub 등으로 SSE 이미터를 공유해야 한다.

### 30분 SSE 타임아웃
`SseEmitter` 타임아웃이 30분으로 설정되어 있다. 프론트엔드에서 `EventSource`의 `onerror` 콜백에서 자동 재연결 로직을 구현해야 한다. `EventSource`는 기본적으로 자동 재연결을 지원하지만, 서버에서 명시적으로 연결을 끊는 경우 재연결이 필요하다.
