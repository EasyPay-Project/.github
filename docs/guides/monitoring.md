# Prometheus + Grafana 모니터링

## 핵심 개념

### Prometheus
시계열 데이터베이스 기반 모니터링 시스템. **Pull 방식**으로 타겟 애플리케이션의 `/actuator/prometheus` 엔드포인트를 주기적으로 스크래핑(scraping)하여 메트릭을 수집한다.

### 메트릭 유형
- **Counter**: 단조 증가하는 누적값 (예: 총 요청 수, 총 에러 수). 절대값보다 `rate()`로 초당 증가율을 계산하여 사용.
- **Timer**: 소요 시간을 측정. 내부적으로 count + sum + histogram 버킷을 저장하여 p50/p95/p99 계산 가능.
- **Gauge**: 현재 상태를 나타내는 값 (예: 활성 스레드 수, 커넥션 풀 크기). 오르내림 가능.

### Grafana
Prometheus를 데이터소스로 사용하는 시각화/알림 도구. PromQL 쿼리로 대시보드 패널을 구성하고, 임계값 기반 알림을 설정한다.

## 프로젝트 적용

### 커스텀 메트릭 (MetricsConfig)

Micrometer를 통해 비즈니스 메트릭 11개를 정의한다.

| # | 메트릭명 | 유형 | 태그 | 설명 |
|---|---------|------|------|------|
| 1 | `easypay_transfer_total` | Counter | status=success | 송금 성공 횟수 |
| 2 | `easypay_transfer_total` | Counter | status=fail | 송금 실패 횟수 |
| 3 | `easypay_transfer_duration_seconds` | Timer | - | 송금 처리 시간 |
| 4 | `easypay_transfer_lock_duration_seconds` | Timer | - | 분산 락 획득 시간 |
| 5 | `easypay_transfer_rate_limited_total` | Counter | - | Rate Limit 거부 횟수 |
| 6 | `easypay_payment_total` | Counter | status=success | 결제 성공 횟수 |
| 7 | `easypay_payment_total` | Counter | status=fail | 결제 실패 횟수 |
| 8 | `easypay_payment_duration_seconds` | Timer | - | 결제 처리 시간 |
| 9 | `easypay_settlement_batch_total` | Counter | - | 정산 배치 처리 횟수 |
| 10 | `easypay_settlement_batch_duration_seconds` | Timer | - | 정산 배치 처리 시간 |

Spring Boot Actuator가 자동으로 수집하는 메트릭 (JVM, HTTP, HikariCP 등)도 Prometheus에 노출된다.

### Timer.record() 패턴

비즈니스 로직의 소요 시간을 측정하는 패턴:

```java
// TransferService.java (개념)
long start = System.nanoTime();
try {
    // 송금 비즈니스 로직
    TransferResponse response = sagaOrchestrator.execute(context);
    transferSuccessCounter.increment();
    return response;
} catch (Exception e) {
    transferFailCounter.increment();
    throw e;
} finally {
    transferTimer.record(System.nanoTime() - start, TimeUnit.NANOSECONDS);
}

// DistributedLockService.java:55-56
long lockStart = System.nanoTime();
locked = lock.tryLock(WAIT_TIME, LEASE_TIME, TimeUnit.SECONDS);
lockTimer.record(System.nanoTime() - lockStart, TimeUnit.NANOSECONDS);
```

### Grafana 대시보드 (2개)

**1. Application Overview** (`application-overview.json`)

전체 애플리케이션 상태를 한눈에 파악하는 대시보드.

| 패널 | PromQL (핵심) | 설명 |
|------|-------------|------|
| Uptime | `process_uptime_seconds` | 서버 가동 시간 |
| RPS | `rate(http_server_requests_seconds_count[5m])` | 초당 요청 수 |
| 5xx Error Rate | `sum(rate(...{status=~"5.."}[5m])) / sum(rate(...[5m]))` | 서버 에러율 |
| JVM Heap | `jvm_memory_used_bytes{area="heap"}` | 힙 메모리 사용량 |
| CPU Usage | `process_cpu_usage` | CPU 사용률 |
| HTTP p50/p95/p99 | `histogram_quantile(0.95, ...)` | 응답 시간 분포 |
| HikariCP | `hikaricp_connections_active` | DB 커넥션 풀 상태 |

**2. Transfer Monitoring** (`transfer-monitoring.json`)

송금/결제/정산 비즈니스 메트릭에 특화된 대시보드.

| 패널 | PromQL (핵심) | 설명 |
|------|-------------|------|
| Transfer TPS | `rate(easypay_transfer_total[5m])` | 초당 송금 처리량 |
| Success Rate | `success / (success + fail) × 100` | 송금 성공률 |
| Response Time | `easypay_transfer_duration_seconds` p50/p95/p99 | 송금 응답 시간 |
| Lock Duration | `easypay_transfer_lock_duration_seconds` p95/p99 | 락 획득 시간 |
| Rate Limit | `rate(easypay_transfer_rate_limited_total[5m])` | Rate Limit 거부율 |
| Payment TPS | `rate(easypay_payment_total[5m])` | 결제 처리량 |
| Settlement | `easypay_settlement_batch_duration_seconds` | 배치 소요 시간 |

### Grafana 알림 (4개)

```yaml
# monitoring/grafana/provisioning/alerting/alerts.yml
```

| # | 알림 | 조건 | 지속 시간 | 심각도 |
|---|------|------|----------|--------|
| 1 | 5xx 에러율 경고 | 5xx 비율 > 5% | 5분 | critical |
| 2 | HTTP p95 경고 | p95 응답 시간 > 1초 | 5분 | warning |
| 3 | JVM Heap 경고 | Heap 사용률 > 90% | 5분 | critical |
| 4 | 송금 실패율 경고 | 송금 실패율 > 5% | 5분 | critical |

각 알림은 `for: 5m` 설정으로 일시적 스파이크가 아닌 **지속적 문제**에만 발생한다.

## 코드 위치 및 구조

```
backend/
├── src/main/java/com/easypay/infra/monitoring/
│   └── MetricsConfig.java                    # 커스텀 메트릭 11개 Bean 정의
├── monitoring/
│   ├── prometheus.yml                         # Prometheus 스크래핑 설정
│   └── grafana/
│       ├── dashboards/
│       │   ├── application-overview.json      # 애플리케이션 개요 대시보드
│       │   └── transfer-monitoring.json       # 송금/결제/정산 대시보드
│       └── provisioning/
│           ├── datasources/
│           │   └── datasource.yml             # Prometheus 데이터소스 설정
│           ├── dashboards/
│           │   └── dashboard.yml              # 대시보드 자동 로드 설정
│           └── alerting/
│               └── alerts.yml                 # 알림 규칙 4개
```

## 설정

### Prometheus 스크래핑

```yaml
# monitoring/prometheus.yml
global:
  scrape_interval: 15s           # 15초마다 메트릭 수집
  evaluation_interval: 15s       # 15초마다 알림 규칙 평가

scrape_configs:
  - job_name: 'easypay'
    metrics_path: '/actuator/prometheus'
    static_configs:
      - targets: ['app:8080']    # Docker 내부 DNS
        labels:
          application: 'easypay'
```

### Spring Boot Actuator

```yaml
# application.yml
management:
  endpoints:
    web:
      exposure:
        include: health, prometheus   # 노출 엔드포인트
  metrics:
    tags:
      application: easypay            # 전역 태그
```

### Grafana 데이터소스 프로비저닝

```yaml
# monitoring/grafana/provisioning/datasources/datasource.yml
datasources:
  - name: Prometheus
    type: prometheus
    url: http://prometheus:9090      # Docker 내부 DNS
    isDefault: true
    jsonData:
      timeInterval: "15s"            # 스크래핑 간격과 동일
```

## 동작 흐름

```
Spring Boot App (:8080)
  │
  ├─ /actuator/prometheus          ← Prometheus가 15초마다 Pull
  │  ├─ JVM 메트릭 (자동)
  │  ├─ HTTP 메트릭 (자동)
  │  ├─ HikariCP 메트릭 (자동)
  │  └─ easypay_* 메트릭 (커스텀)
  │
  ▼
Prometheus (:9090)
  │
  ├─ 시계열 데이터 저장
  ├─ PromQL 쿼리 엔진
  │
  ▼
Grafana (:3000)
  │
  ├─ 대시보드 2개 시각화
  ├─ 알림 규칙 4개 평가 (1분 간격)
  └─ 임계값 초과 시 알림 발생
```

## 트러블슈팅 & 주의사항

### Counter vs Gauge 선택 기준
- 요청 수, 에러 수처럼 **누적되는 값**은 Counter. `rate()`로 초당 증가율을 계산.
- 커넥션 풀 크기, 메모리 사용량처럼 **현재 상태**는 Gauge. 그대로 표시.
- Counter를 그래프에 그대로 그리면 단조 증가 곡선만 보인다. 반드시 `rate()` 또는 `increase()`와 함께 사용.

### PromQL 기본 패턴
```promql
# 초당 증가율 (최근 5분 기준)
rate(easypay_transfer_total{status="success"}[5m])

# 백분위수 (p95)
histogram_quantile(0.95,
  sum(rate(easypay_transfer_duration_seconds_bucket[5m])) by (le))

# 비율 계산
sum(rate(http_server_requests_seconds_count{status=~"5.."}[5m]))
/
sum(rate(http_server_requests_seconds_count[5m])) * 100
```

### Grafana 대시보드 프로비저닝
`monitoring/grafana/provisioning/dashboards/dashboard.yml`에서 `updateIntervalSeconds: 30`으로 설정하여, JSON 파일 변경 시 30초 후 자동 반영된다. `allowUiUpdates: true`로 UI에서도 수정 가능.

### 태그 기반 Counter 분리
`easypay_transfer_total`은 `status` 태그로 success/fail을 구분한다. 동일 메트릭명에 태그만 다른 Bean을 2개 정의하여, PromQL에서 `{status="success"}`로 필터링한다.
