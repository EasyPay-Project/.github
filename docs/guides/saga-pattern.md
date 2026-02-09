# Saga 패턴 (오케스트레이션)

## 핵심 개념

Saga 패턴은 분산 환경에서 **긴 트랜잭션을 여러 로컬 트랜잭션으로 분리**하고, 중간에 실패하면 이전 단계를 **보상 트랜잭션(Compensating Transaction)**으로 롤백하는 패턴이다.

### 오케스트레이션 vs 코레오그래피

| 구분 | 오케스트레이션 | 코레오그래피 |
|------|-------------|-------------|
| 중앙 제어 | 오케스트레이터가 전체 흐름 관리 | 각 서비스가 이벤트로 소통 |
| 흐름 파악 | 한 곳에서 전체 보임 | 이벤트 추적 필요 |
| 결합도 | 오케스트레이터에 의존 | 느슨한 결합 |
| 보상 처리 | 중앙에서 역순 실행 | 각 서비스가 자체 처리 |
| 적합한 경우 | 모놀리식/적은 서비스 | 마이크로서비스 |

EasyPay는 **모듈러 모놀리스**이므로, 흐름 파악이 쉽고 구현이 단순한 **오케스트레이션** 방식을 채택했다.

## 프로젝트 적용

### 송금 Saga 4단계

```
[Step 1] CreateTransfer    → Transfer 엔티티 생성 (PENDING → PROCESSING)
[Step 2] DebitSender       → 송금자 계좌에서 출금 (Account.withdraw)
[Step 3] CreditReceiver    → 수취자 계좌에 입금 (Account.deposit)
[Step 4] RecordTransactions → 거래 내역 2건 기록 (DEBIT + CREDIT)
```

**실패 시 역순 보상**:
```
Step 3 실패 시:
  → Step 2 보상: 송금자 계좌에 재입금 (Account.deposit)
  → Step 1 보상: Transfer 상태를 FAILED로 변경

Step 4 실패 시:
  → Step 3 보상: 수취자 계좌에서 출금 (Account.withdraw)
  → Step 2 보상: 송금자 계좌에 재입금
  → Step 1 보상: Transfer 상태를 FAILED로 변경
```

### SagaStep 인터페이스

각 단계는 `execute()`와 `compensate()`를 구현한다.

```java
// TransferSagaOrchestrator 내부 인터페이스
interface SagaStep {
    void execute(TransferSagaContext context);
    void compensate(TransferSagaContext context);
}
```

### TransferSagaContext

Saga 실행 중 단계 간 데이터를 공유하는 컨텍스트 객체.

```java
// TransferSagaContext.java
@Getter @Setter
public class TransferSagaContext {
    private final TransferRequest request;
    private final String idempotencyKey;
    private final Long userId;

    private Transfer transfer;
    private Account senderAccount;
    private Account receiverAccount;
    private BigDecimal senderBalanceAfter;
}
```

### 오케스트레이터 실행 로직

```java
// TransferSagaOrchestrator.java (핵심 로직)
@Transactional
public TransferResponse execute(TransferSagaContext context) {
    List<SagaStep> completedSteps = new ArrayList<>();

    try {
        for (SagaStep step : steps) {
            step.execute(context);
            completedSteps.add(step);
        }
        // 모든 단계 성공
        context.getTransfer().complete();
        publishTransferEvent(context);  // Kafka 이벤트 발행
        return TransferResponse.from(context.getTransfer(), context.getSenderBalanceAfter());

    } catch (Exception e) {
        // 역순 보상
        Collections.reverse(completedSteps);
        for (SagaStep step : completedSteps) {
            try {
                step.compensate(context);
            } catch (Exception compensationError) {
                log.error("보상 트랜잭션 실패", compensationError);
            }
        }
        context.getTransfer().fail(e.getMessage());
        throw e;
    }
}
```

## 5계층 동시성 제어 전체 흐름

Saga는 5계층 방어선의 4번째 계층이다. 송금 요청이 컨트롤러에서 서비스까지 거치는 전체 흐름:

```
클라이언트 → POST /api/v1/transfers (Idempotency-Key 헤더 필수)
  │
  ├─[1층] Rate Limiting (TransferRateLimitService)
  │     ├─ 1건 한도: amount ≤ 2,000,000원
  │     └─ 시간당 횟수: ≤ 30회/시간 (Redis ZSET)
  │     └─ 위반 시: 429 RATE_LIMIT_EXCEEDED
  │
  ├─[2층] 멱등성 (IdempotencyService)
  │     ├─ 요청 본문 SHA-256 해시
  │     ├─ Redis에서 기존 처리 확인
  │     │   ├─ COMPLETED → 캐시된 응답 즉시 반환
  │     │   └─ PROCESSING → 409 IDEMPOTENCY_KEY_PROCESSING
  │     └─ 신규: PROCESSING 상태로 저장
  │
  ├─[3층] 분산 락 (DistributedLockService)
  │     ├─ 락 키: [senderAccountId, receiverAccountId] → 정렬
  │     ├─ Redisson MultiLock: tryLock(WAIT=5s, LEASE=10s)
  │     └─ 획득 실패: 409 LOCK_ACQUISITION_FAILED
  │
  ├─[4층] Saga 오케스트레이션 (TransferSagaOrchestrator)
  │     ├─ @Transactional 범위 내 실행
  │     ├─ Step 1: Transfer 생성 (PENDING → PROCESSING)
  │     ├─ Step 2: 송금자 출금 (Account.withdraw)
  │     ├─ Step 3: 수취자 입금 (Account.deposit)
  │     ├─ Step 4: 거래 내역 기록 (Transaction × 2)
  │     ├─ 성공: Transfer.complete() + Kafka 이벤트 발행
  │     └─ 실패: 역순 보상 → Transfer.fail()
  │
  └─[5층] 낙관적 락 (@Version on Account)
        ├─ JPA가 UPDATE 쿼리에 WHERE version=? 자동 추가
        └─ 버전 충돌 시: OptimisticLockingFailureException
```

## 코드 위치 및 구조

```
src/main/java/com/easypay/transfer/
├── domain/
│   ├── Transfer.java              # 엔티티 (상태 전이 메서드)
│   └── TransferStatus.java        # PENDING/PROCESSING/COMPLETED/FAILED/CANCELLED/COMPENSATED
├── service/
│   ├── TransferService.java       # 5계층 제어 진입점
│   ├── TransferSagaOrchestrator.java  # Saga 실행/보상 로직 + 4개 SagaStep 내부 클래스
│   └── TransferSagaContext.java   # Saga 컨텍스트
├── dto/
│   ├── TransferRequest.java       # 요청 DTO (senderAccountId, receiverAccountId, amount, memo)
│   └── TransferResponse.java      # 응답 DTO (from 팩토리 메서드)
├── repository/
│   └── TransferRepository.java
└── controller/
    └── TransferController.java    # POST /api/v1/transfers
```

## 동작 흐름 (시퀀스)

```
Client          Controller       TransferService      RateLimit   Idempotency   Lock        SagaOrchestrator    Account DB
  │                │                  │                   │           │           │              │                  │
  │──POST /transfers──→│              │                   │           │           │              │                  │
  │                │──transfer()──→│                      │           │           │              │                  │
  │                │              │──checkLimit()────→│   │           │           │              │                  │
  │                │              │←────── ok ────────│   │           │           │              │                  │
  │                │              │──startOrGet()─────────→│          │           │              │                  │
  │                │              │←───── empty ──────────│           │           │              │                  │
  │                │              │──executeWithMultiLock()───────────→│          │              │                  │
  │                │              │                       │           │──tryLock()→│              │                  │
  │                │              │                       │           │←── ok ────│              │                  │
  │                │              │                       │           │           │──execute()──→│                  │
  │                │              │                       │           │           │              │──withdraw()──────→│
  │                │              │                       │           │           │              │←──── ok ─────────│
  │                │              │                       │           │           │              │──deposit()───────→│
  │                │              │                       │           │           │              │←──── ok ─────────│
  │                │              │                       │           │           │←── response ─│                  │
  │                │              │                       │           │──unlock()─→│              │                  │
  │                │              │──complete()───────────→│          │           │              │                  │
  │                │←── response ──│                      │           │           │              │                  │
  │←── 201 Created ──│              │                   │           │           │              │                  │
```

## 트러블슈팅 & 주의사항

### @Transactional과 Saga 보상의 관계
Saga 오케스트레이터에 `@Transactional`이 걸려 있으므로, 예외 발생 시 전체 트랜잭션이 롤백된다. 그러나 **보상 트랜잭션은 별도 트랜잭션으로 실행해야 하는 경우**도 있다. 현재 구현에서는 모놀리스 내에서 단일 DB 트랜잭션으로 처리하므로, 보상 로직은 JPA의 `@Transactional` 롤백에 의존한다.

### 보상 트랜잭션 실패 처리
보상 중에도 예외가 발생할 수 있다. `catch (Exception compensationError)` 블록에서 로그만 남기고 계속 진행하여, 가능한 많은 보상을 실행한다. 실무에서는 보상 실패 시 Dead Letter Queue나 수동 개입 메커니즘이 필요하다.

### Transfer 상태 머신
```
PENDING → PROCESSING → COMPLETED   (정상 완료)
                     → FAILED      (Saga 실패)
                     → CANCELLED   (사용자 취소, 완료 후 30분 이내)
                     → COMPENSATED (보상 트랜잭션 완료)
```

### senderId/receiverId는 사용자 ID
Transfer 엔티티의 `senderId`, `receiverId`는 **Account ID가 아닌 User ID**이다. Kafka 이벤트에서 SSE 알림을 보낼 때 User ID로 라우팅하기 위함이다. 분산 락 키는 Account ID를 사용한다.
