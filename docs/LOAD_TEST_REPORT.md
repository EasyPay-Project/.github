# 부하 테스트 결과 보고서

## 테스트 환경

- **실행 방식**: Docker 컨테이너 기반 (docker-compose)
- **애플리케이션 스택**: Spring Boot 3.4.2 / Java 21 / PostgreSQL 16 / Redis / Kafka
- **부하 테스트 도구**: k6 v1.5.0 (grafana/k6:latest Docker 이미지)
- **테스트 일시**: 2026-02-09
- **테스트 데이터**:
  - 테스트 사용자: 10명 (user1~10@easypay.com)
  - 계좌: 각 사용자당 2개 (총 20개)
  - 초기 잔액: 계좌당 10,000,000원

## 4개 시나리오 결과 요약

### 시나리오 1: 동시 송금 정합성 (concurrent-transfer)

**목적**: 동일 계좌에 대해 동시 사용자가 송금 시 잔액 정합성(낙관적 락 + 분산 락) 검증

**설정**:
- Virtual Users: 50 VU
- 실행 시간: 1분 (원본: 300 VU, 5분)

**결과**: ✅ ALL PASS
- 총 요청: 23,196건 (송금 23,184건)
- 성공률: 100% (23,184/23,184)
- TPS: 381.59/s
- http_req_duration:
  - avg: 4.19ms
  - med: 3.24ms
  - p90: 6.89ms
  - p95: 8.95ms
  - p99: 16.79ms
  - max: 73.10ms
- transfer_duration: avg 4.17ms, p95 9ms, p99 16ms
- 에러율: 0%
- **임계값 충족**: p95 < 500ms ✓, p99 < 1500ms ✓, rate > 0.95 ✓

### 시나리오 2: Hot Account 집중 송금 (hot-account)

**목적**: 특정 인기 계좌(ID=1)로 다수 송금 집중 시 분산 락/낙관적 락 정상 동작 검증

**설정**:
- Virtual Users: 30 VU
- 실행 시간: 1분 (원본: 200 VU, 5분)

**결과**: ✅ ALL PASS
- 총 요청: 9,970건 (송금 9,958건)
- 성공률: 100% (9,958/9,958)
- TPS: 163.58/s
- http_req_duration:
  - avg: 4.52ms
  - med: 3.38ms
  - p90: 6.51ms
  - p95: 8.51ms
  - max: 189.29ms
- hot_duration: avg 4.53ms, p95 9ms, max 190ms
- 에러율: 0%
- **Hot Account 잔액 변동**: 28,586,940원 → 44,127,253원 (+15,540,313원)
- **임계값 충족**: p95 < 800ms ✓, rate > 0.85 ✓

### 시나리오 3: 멱등성 재시도 (idempotency-retry)

**목적**: 동일 멱등성 키로 5회 재시도 시 정확히 1회만 처리, 동일 결과 반환 검증

**설정**:
- Virtual Users: 20 VU
- 실행 시간: 1분 (원본: 100 VU, 5분)

**결과**: ✅ ALL PASS
- 총 요청: 10,150건 (첫 요청 2,028건 + 재시도 8,112건)
- 멱등성 일관성: 100% (8,112/8,112)
- TPS: 165.11/s
- http_req_duration:
  - avg: 2.47ms
  - med: 1.47ms
  - p90: 5.01ms
  - p95: 5.98ms
  - max: 152.10ms
- retry_duration: avg 2.43ms, p95 6ms
- http_req_failed: 0%
- **임계값 충족**: consistency_rate > 0.999 ✓

### 시나리오 4: 스트레스 테스트 (stress-test)

**목적**: 점진적으로 VU를 증가시키며 시스템 한계점과 성능 저하 시작 지점 파악

**설정**:
- Virtual Users: ramping 0→10→30→50(유지)→20→0
- 실행 시간: 3분 (원본: 0→50→200→500, 15분)

**결과**: ✅ ALL PASS
- 총 요청: 18,989건 (iterations: 18,979)
- 성공률: 99.75% (18,932/18,979)
- 실패: 47건 (0.25%) - 잔액 부족 등 비즈니스 예외
- TPS: 105.05/s (ramping 평균)
- http_req_duration:
  - avg: 3.01ms
  - med: 2.29ms
  - p90: 5.45ms
  - p95: 6.57ms
  - p99: 10.49ms
  - max: 67.55ms
- stress_tx_duration: avg 3.10ms, p95 7ms, p99 11ms
- 에러율: 0.24%
- 응답 시간 < 1s: 100% (18,979/18,979)
- **임계값 충족**: p95 < 500ms ✓, p99 < 1500ms ✓, rate > 0.95 ✓

## 종합 결과 테이블

| 시나리오 | VU | 시간 | 총 요청 | 성공률 | TPS | p50 | p90 | p95 | p99 | max |
|---------|-----|------|--------|--------|-----|-----|-----|-----|-----|-----|
| concurrent-transfer | 50 | 1m | 23,196 | 100% | 381/s | 3.24ms | 6.89ms | 8.95ms | 16.79ms | 73ms |
| hot-account | 30 | 1m | 9,970 | 100% | 164/s | 3.38ms | 6.51ms | 8.51ms | - | 189ms |
| idempotency-retry | 20 | 1m | 10,150 | 100% | 165/s | 1.47ms | 5.01ms | 5.98ms | - | 152ms |
| stress-test | 0→50 | 3m | 18,989 | 99.75% | 105/s | 2.29ms | 5.45ms | 6.57ms | 10.49ms | 68ms |

## 핵심 검증 항목 요약

### 1. 동시성 제어 (3단계 방어선)

- ✅ **Redis 분산 락 (Redisson)**: 정상 동작 확인. 데드락 미발생
- ✅ **JPA 낙관적 락 (@Version)**: Hot Account 집중 시에도 정합성 유지
- ✅ **멱등성 키 (Redis)**: 100% 일관성 검증 완료

### 2. 성능 SLA 달성 여부

- ✅ **p95 < 500ms**: 모든 시나리오에서 p95 < 10ms로 SLA 대비 50배 이상 여유
- ✅ **p99 < 1500ms**: 모든 시나리오에서 p99 < 20ms
- ✅ **에러율 < 1%**: 최대 0.25% (스트레스 테스트, 잔액 부족 등 비즈니스 예외)

### 3. 잔액 정합성

- ✅ Hot Account 테스트에서 잔액 변동 추적 가능 (28,586,940원 → 44,127,253원)
- ✅ 동시 송금 시 이중 차감/이중 입금 없음

## 발견된 이슈 및 수정 사항

### 이슈 1: API Rate Limit으로 부하 테스트 불가 (✅ 수정 완료)

**원인**: 기본 Rate Limit 100 req/min/user가 부하 테스트 규모에 부족

**수정**:
- Rate Limit 값을 `@Value`로 외부화
- application-docker.yml에서 100,000으로 설정
- 수정 파일: `ApiRateLimitFilter.java`, `TransferRateLimitService.java`, `application-docker.yml`

### 이슈 2: 계좌 소유권 불일치 (✅ 수정 완료)

**원인**: k6 스크립트가 VU→계좌 매핑 시 소유권을 고려하지 않아 403 FORBIDDEN 발생

**수정**:
- config.js `getAccountPair()` 및 각 시나리오 스크립트에서 userN→계좌(2N-1, 2N) 매핑 수정
- 수정 파일: `k6/common/config.js`, `concurrent-transfer.js`, `hot-account.js`, `stress-test.js`

### 이슈 3: 멱등성 테스트 memo 불일치 (✅ 수정 완료)

**원인**: `memo`에 `attempt=${attempt}` 포함 → 재시도마다 request hash 변경 → IDEMPOTENCY_KEY_CONFLICT 발생

**수정**:
- memo에서 attempt 제거, 동일 idempotency key에 동일 request body 보장
- 수정 파일: `k6/scenarios/idempotency-retry.js`

## 결론

EasyPay 백엔드 시스템은 로컬 축소 버전 부하 테스트(50 VU, 1~3분)에서 다음을 검증하였습니다.

1. **동시성 제어**: Redis 분산 락 + JPA 낙관적 락 조합이 동시 송금, Hot Account 집중, 스트레스 상황에서 모두 정상 동작
2. **멱등성 보장**: 동일 키로 5회 재시도 시 100% 일관성 유지
3. **성능 SLA**: 모든 시나리오에서 p95 < 10ms, p99 < 20ms로 목표 SLA(p95 < 500ms, p99 < 1500ms) 대비 50배 이상 여유
4. **안정성**: 에러율 0~0.25% (비즈니스 예외 제외 시 0%)

### 참고 사항

- **테스트 규모**: 로컬 축소 버전(local/*.js)으로 실행. 프로덕션 규모(300~500 VU, 5~15분)는 별도 인프라 필요
- **결과 JSON**: `k6/results/` 디렉토리에 시나리오별 summary 저장
- **k6 스크립트**: `k6/scenarios/` (원본 7개), `k6/scenarios/local/` (축소 4개)

### 향후 권장사항

1. 프로덕션 규모 부하 테스트 (300~500 VU, 5~15분) 별도 인프라에서 실행
2. Kafka 이벤트 처리, Batch 정산 작업에 대한 부하 테스트 추가
3. 장애 주입 테스트 (Chaos Engineering) 도입 검토
