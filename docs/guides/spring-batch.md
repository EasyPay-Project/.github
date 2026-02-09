# Spring Batch 정산 배치

## 핵심 개념

### Spring Batch 구조
```
Job (작업 단위)
  └─ Step (단계)
       └─ Chunk (청크, N건 단위 처리)
            ├─ ItemReader   : 데이터 읽기
            ├─ ItemProcessor: 데이터 변환/필터링
            └─ ItemWriter   : 데이터 저장
```

- **Job**: 배치 작업의 최상위 단위. `JobParameters`로 구분하여 중복 실행을 방지한다.
- **Step**: Job 내의 실행 단계. 하나의 Step은 Reader → Processor → Writer 파이프라인을 구성한다.
- **Chunk**: N건 단위로 트랜잭션을 묶어 처리한다. 1,000건을 읽어서 한 번에 커밋하면 DB 부하가 줄어든다.

### EasyPay에서의 역할
전일 **COMPLETED 결제**를 가맹점별로 집계하여 정산(Settlement) 레코드를 생성한다. 수수료 3%를 적용하고 순매출액을 계산한다.

## 프로젝트 적용

### 일일 정산 배치 (DailySettlementJobConfig)

```java
// DailySettlementJobConfig.java 주요 상수
private static final int CHUNK_SIZE = 1000;           // 청크 크기
private static final BigDecimal FEE_RATE = new BigDecimal("0.03");  // 수수료 3%
```

### Reader: 전일 완료 결제 조회

JpaPagingItemReader로 전일 00:00:00 ~ 23:59:59 사이에 완료된 결제를 조회한다.

```java
// DailySettlementJobConfig.java:69-88
@Bean
@StepScope
public JpaPagingItemReader<Payment> completedPaymentReader() {
    LocalDate yesterday = LocalDate.now().minusDays(1);
    LocalDateTime start = yesterday.atStartOfDay();
    LocalDateTime end = yesterday.atTime(23, 59, 59);

    return new JpaPagingItemReaderBuilder<Payment>()
            .name("completedPaymentReader")
            .entityManagerFactory(entityManagerFactory)
            .queryString("SELECT p FROM Payment p WHERE p.status = :status " +
                    "AND p.completedAt BETWEEN :start AND :end ORDER BY p.id")
            .parameterValues(Map.of(
                    "status", PaymentStatus.COMPLETED,
                    "start", start,
                    "end", end))
            .pageSize(CHUNK_SIZE)
            .build();
}
```

**왜 JpaPagingItemReader인가**: 전체 데이터를 메모리에 로드하지 않고, 페이지 단위로 읽어 OOM을 방지한다.

### Processor: Pass-through

현재 필터링이나 변환 로직이 없어 pass-through로 구현한다.

```java
@Bean
public ItemProcessor<Payment, Payment> paymentProcessor() {
    return payment -> payment;
}
```

### Writer: 가맹점별 집계 + 정산 생성

```java
// DailySettlementJobConfig.java:95-142 (핵심 로직)
@Bean
public ItemWriter<Payment> settlementWriter() {
    return chunk -> {
        long start = System.nanoTime();

        // 가맹점별 집계
        Map<Long, SettlementAccumulator> merchantTotals = new HashMap<>();
        for (Payment payment : chunk) {
            merchantTotals.computeIfAbsent(payment.getMerchantId(),
                    k -> new SettlementAccumulator())
                    .add(payment.getAmount());
        }

        LocalDate yesterday = LocalDate.now().minusDays(1);

        // 정산 레코드 생성
        for (Map.Entry<Long, SettlementAccumulator> entry : merchantTotals.entrySet()) {
            Long merchantId = entry.getKey();
            SettlementAccumulator acc = entry.getValue();

            // 중복 체크
            Optional<Settlement> existing = settlementRepository
                    .findByMerchantIdAndSettlementDate(merchantId, yesterday);
            if (existing.isPresent()) {
                log.debug("이미 정산 완료: merchantId={}", merchantId);
                continue;
            }

            BigDecimal feeAmount = acc.totalAmount.multiply(FEE_RATE)
                    .setScale(2, RoundingMode.HALF_UP);
            BigDecimal netAmount = acc.totalAmount.subtract(feeAmount);

            Settlement settlement = Settlement.builder()
                    .merchantId(merchantId)
                    .settlementDate(yesterday)
                    .totalAmount(acc.totalAmount)
                    .feeAmount(feeAmount)
                    .netAmount(netAmount)
                    .transactionCount(acc.count)
                    .build();
            settlement.complete();  // 즉시 COMPLETED
            settlementRepository.save(settlement);
        }

        settlementBatchCounter.increment();
        settlementBatchTimer.record(System.nanoTime() - start, TimeUnit.NANOSECONDS);
    };
}
```

**SettlementAccumulator**: 가맹점별 총액과 건수를 메모리에서 집계하는 내부 클래스.

```java
private static class SettlementAccumulator {
    BigDecimal totalAmount = BigDecimal.ZERO;
    long count = 0;

    void add(BigDecimal amount) {
        totalAmount = totalAmount.add(amount);
        count++;
    }
}
```

### Fault Tolerance

```java
// DailySettlementJobConfig.java:61-66
.<Payment, Payment>chunk(CHUNK_SIZE)
.faultTolerant()
.skipLimit(10)          // 최대 10건 스킵
.skip(Exception.class)  // 모든 예외 스킵 가능
.retryLimit(3)          // 최대 3회 재시도
.retry(Exception.class)  // 모든 예외 재시도 가능
```

- **skipLimit(10)**: 개별 레코드 처리 실패 시 최대 10건까지 건너뛰고 계속 진행
- **retryLimit(3)**: 일시적 오류(DB 연결 등)에 대해 3회까지 재시도
- 결제 누락보다 배치 중단이 더 심각하므로, 일부 실패는 스킵하고 로그로 추적

### 수동 실행

배치는 자동 스케줄링 없이 **관리자 API**로만 실행한다.

```java
// SettlementService.java:43-48
public void runSettlement() throws Exception {
    jobLauncher.run(dailySettlementJob,
            new JobParametersBuilder()
                    .addLong("timestamp", System.currentTimeMillis())  // 고유 파라미터
                    .toJobParameters());
}

// SettlementController.java:39-44
@PreAuthorize("hasRole('ADMIN')")
@PostMapping("/api/v1/admin/settlements/run")
public ApiResponse<String> runSettlement() throws Exception {
    settlementService.runSettlement();
    return ApiResponse.success("정산 배치 실행 시작");
}
```

**`timestamp` 파라미터**: Spring Batch는 동일 `JobParameters`로 Job을 재실행하지 않는다. `System.currentTimeMillis()`를 파라미터로 전달하여 매번 고유한 실행으로 인식시킨다.

## 코드 위치 및 구조

```
src/main/java/com/easypay/settlement/
├── batch/
│   └── DailySettlementJobConfig.java   # Job/Step/Reader/Processor/Writer 정의
├── domain/
│   ├── Settlement.java                  # 정산 엔티티
│   └── SettlementStatus.java           # PENDING/PROCESSING/COMPLETED/FAILED
├── repository/
│   └── SettlementRepository.java       # 조회 메서드 (merchantId, date)
├── service/
│   └── SettlementService.java          # 조회 + 수동 실행
├── controller/
│   └── SettlementController.java       # API 엔드포인트 (3개)
└── dto/
    └── SettlementResponse.java         # 응답 DTO
```

## 설정

```yaml
# application-local.yml / application-docker.yml
spring:
  batch:
    jdbc:
      initialize-schema: always   # Batch 메타 테이블 자동 생성
    job:
      enabled: false              # 자동 실행 비활성화 (수동만)
```

`initialize-schema: always`는 Spring Batch가 사용하는 메타 테이블(`BATCH_JOB_INSTANCE`, `BATCH_JOB_EXECUTION`, `BATCH_STEP_EXECUTION` 등)을 자동으로 생성한다.

## 동작 흐름

```
관리자 API 호출
  │
  ├─ POST /api/v1/admin/settlements/run
  │  └─ @PreAuthorize("hasRole('ADMIN')")
  │
  ├─ SettlementService.runSettlement()
  │  └─ JobLauncher.run(dailySettlementJob, {timestamp: now})
  │
  ├─ dailySettlementJob
  │  └─ settlementStep (Chunk = 1,000)
  │     │
  │     ├─ [Reader] completedPaymentReader
  │     │  └─ SELECT p FROM Payment p
  │     │     WHERE status = 'COMPLETED'
  │     │     AND completedAt BETWEEN 어제00:00 AND 어제23:59
  │     │     ORDER BY p.id
  │     │     (1,000건씩 페이징)
  │     │
  │     ├─ [Processor] paymentProcessor
  │     │  └─ pass-through (변환 없음)
  │     │
  │     └─ [Writer] settlementWriter
  │        ├─ 가맹점별 HashMap 집계
  │        │  └─ merchantId → {totalAmount, count}
  │        │
  │        ├─ 각 가맹점에 대해:
  │        │  ├─ 중복 체크 (merchantId + settlementDate)
  │        │  ├─ feeAmount = totalAmount × 0.03
  │        │  ├─ netAmount = totalAmount - feeAmount
  │        │  └─ Settlement 생성 → COMPLETED
  │        │
  │        └─ 메트릭 기록
  │           ├─ settlementBatchCounter++
  │           └─ settlementBatchTimer.record()
  │
  └─ Job 완료 → BATCH_JOB_EXECUTION에 기록
```

## 트러블슈팅 & 주의사항

### 중복 정산 방지
`findByMerchantIdAndSettlementDate()`로 이미 정산된 가맹점은 건너뛴다. 단, 동일 날짜에 배치를 여러 번 실행해도 안전하다(멱등성).

### 배치 메타 테이블
Spring Batch는 실행 이력을 `BATCH_JOB_INSTANCE`, `BATCH_JOB_EXECUTION` 등의 테이블에 저장한다. 테스트 환경에서는 `initialize-schema: always`로 H2에 자동 생성된다.

### 상태 전이: PENDING → COMPLETED (PROCESSING 생략)
현재 구현에서는 Settlement를 생성하고 즉시 `complete()`를 호출한다. PROCESSING 상태를 거치지 않는데, 이는 단일 Step 내에서 동기적으로 처리되기 때문이다. 외부 정산 시스템과 연동한다면 PENDING → PROCESSING → COMPLETED 흐름이 필요하다.

### 수수료 계산 정밀도
`BigDecimal.multiply(FEE_RATE).setScale(2, RoundingMode.HALF_UP)`: 소수점 3자리 이하를 반올림하여 2자리로 맞춘다. 은행 표준 반올림 방식이다.
