# 백엔드 개발 진행사항

**최종 갱신**: 2026-02-09 (4차)
**기술 스택**: Spring Boot 3.4.2 / Java 21 / Gradle
**최신 커밋**: `048eb94` SSE 알림에 계좌 ID 대신 사용자 ID 전달

---

## 커밋 이력

| 커밋 | 날짜 | 내용 |
|------|------|------|
| `048eb94` | 2026-02-09 | fix(transfer): SSE 알림에 계좌 ID 대신 사용자 ID 전달 |
| `b02a3ed` | 2026-02-09 | feat(common): CORS에 localhost:3001 추가하여 Grafana 포트 충돌 해소 |
| `c1b1330` | 2026-02-09 | test(k6): 부하 테스트 실행 및 멱등성 스크립트 수정 |
| `88cba9b` | 2026-02-09 | feat(infra,k6): Rate Limit 설정 외부화 및 k6 계좌 소유자 매칭 수정 |
| `97aeba3` | 2026-02-09 | docs: CLAUDE.md 코드베이스 분석 기반 전면 개선 |
| `fee6042` | 2026-02-09 | refactor: Settlement 서비스 추출, Notification DTO 정식화, 테스트 보강 |
| `e103275` | 2026-02-09 | feat: Spring Boot 앱 Docker 컨테이너화 |
| `b4341ba` | 2026-02-09 | docs: CLAUDE.md 전면 개선 및 README 테스트 수 업데이트 |
| `8cc1a85` | 2026-02-09 | 대용량 부하 테스트 리라이트 + k6 README |
| `03dfb64` | 2026-02-09 | SSE EventSource 쿼리 파라미터 토큰 인증 |
| `6605f6c` | 2026-02-09 | JWT accessToken에 name 클레임 추가 |
| `91399ff` | 2026-02-09 | 기능 고도화, 보안 강화, 인프라/테스트 구성 |
| `543857d` | 2026-02-09 | feat(common): CORS 설정 추가 |
| `6cb65aa` | 2026-02-09 | feat(monitoring): 커스텀 메트릭, k6 부하 테스트, Grafana 대시보드 구현 |
| `80f4258` | 2026-02-09 | feat(payment, kafka, settlement): 결제, 이벤트, 정산 배치 구현 |
| `97e199d` | 2026-02-09 | feat(transfer): 송금 핵심 로직 구현 (분산 락, 멱등성, Saga, Rate Limiting) |
| `0c45e1c` | 2026-02-09 | feat(auth, account): JWT 인증 및 계좌 관리 API 구현 |
| `f17bd87` | 2026-02-08 | chore: 프로젝트 초기 설정 (Spring Boot 3.4.2 / Java 21) |

---

## 모듈별 구현 현황

### Account (계좌) — ✅ 완료

| 레이어 | 파일 | 상태 |
|--------|------|:---:|
| Domain | Account, AccountStatus, Transaction, TransactionType | ✅ |
| Repository | AccountRepository, TransactionRepository (커서 페이지네이션) | ✅ |
| Service | AccountService (생성, 잔액 조회, 충전, 거래 내역) | ✅ |
| DTO | AccountCreateRequest, AccountResponse, BalanceResponse, DepositRequest, TransactionPageResponse, TransactionResponse | ✅ |
| Controller | 6개 엔드포인트 (생성, 목록, 상세, 잔액, 충전, 거래 내역) | ✅ |
| **테스트** | AccountServiceTest (6개), AccountControllerTest | ✅ |

---

### Transfer (송금) — ✅ 완료

| 레이어 | 파일 | 상태 |
|--------|------|:---:|
| Domain | Transfer, TransferStatus | ✅ |
| Repository | TransferRepository | ✅ |
| Service | TransferService (Rate Limiting, 멱등성, Saga 호출) | ✅ |
| Saga | TransferSagaOrchestrator, TransferSagaContext, SagaStep\<T\> | ✅ |
| DTO | TransferRequest, TransferResponse | ✅ |
| Controller | 3개 엔드포인트 (송금, 상태 조회, 취소) | ✅ |
| **테스트** | TransferConcurrencyTest (2개: 동시성, 잔액 부족) | ✅ |

---

### Payment (결제) — ✅ 완료

| 레이어 | 파일 | 상태 |
|--------|------|:---:|
| Domain | Payment, PaymentStatus | ✅ |
| Repository | PaymentRepository | ✅ |
| Service | PaymentService (멱등성, 분산 락, 이벤트 발행) | ✅ |
| DTO | PaymentRequest, PaymentResponse | ✅ |
| Controller | 4개 엔드포인트 (결제, 상태 조회, 취소, 환불) | ✅ |
| **테스트** | PaymentServiceTest (4개: 성공, 미인증, 취소, 잔액 부족) | ✅ |

---

### Settlement (정산) — ✅ 완료

| 레이어 | 파일 | 상태 |
|--------|------|:---:|
| Domain | Settlement, SettlementStatus | ✅ |
| Repository | SettlementRepository | ✅ |
| Batch | DailySettlementJobConfig (Reader-Processor-Writer, 청크 1,000건) | ✅ |
| Service | SettlementService (가맹점별 조회, 날짜별 조회, 수동 실행) | ✅ |
| DTO | SettlementResponse | ✅ |
| Controller | 3개 엔드포인트 (목록, 날짜별, 수동 실행) | ✅ |
| **테스트** | SettlementServiceTest (4개) | ✅ |

---

### Auth/User (인증/사용자) — ✅ 완료

| 레이어 | 파일 | 상태 |
|--------|------|:---:|
| Domain | User, Role (USER/MERCHANT/ADMIN) | ✅ |
| Repository | UserRepository | ✅ |
| Service | AuthService (JWT 발급/갱신, BCrypt, PIN 검증) | ✅ |
| DTO | SignupRequest, LoginRequest, TokenRefreshRequest, PinVerifyRequest, TokenResponse, UserResponse | ✅ |
| Controller | 4개 엔드포인트 | ✅ |
| **테스트** | AuthServiceTest, AuthControllerTest | ✅ |

- JWT에 name 클레임 추가 (프론트엔드 디코드 호환)

---

### Notification (알림) — ✅ 완료

| 레이어 | 파일 | 상태 |
|--------|------|:---:|
| Controller | SSE 구독 (`/notifications/subscribe?token=`) | ✅ |
| Service | NotificationService (SSE Emitter 관리) | ✅ |
| Consumer | NotificationConsumer (Kafka: 송금/결제 이벤트) | ✅ |
| DTO | NotificationEvent, NotificationResponse | ✅ |
| **테스트** | NotificationServiceTest (4개), NotificationConsumerTest (4개) | ✅ |

- SSE 인증 방식: 쿼리 파라미터 토큰 (`?token=`)으로 변경됨
- Named events: `transfer`, `payment`, `connect`

---

### Infra (인프라) — ✅ 완료

| 모듈 | 구현 내용 | 상태 |
|------|----------|:---:|
| **Kafka** | KafkaConfig (4개 토픽), EventPublisher, EventMessage, KafkaTopics | ✅ |
| **Redis** | DistributedLockService, IdempotencyService, RateLimiter, TransferRateLimitService | ✅ |
| **Monitoring** | MetricsConfig (11개 빈), Prometheus 연동 | ✅ |

- TestRedisConfig 추가 (테스트 환경 Redis 모킹)

---

### Common (공통) — ✅ 완료

| 구현 내용 | 상태 |
|----------|:---:|
| SecurityConfig, JwtTokenProvider, JwtAuthenticationFilter | ✅ |
| GlobalExceptionHandler + ErrorCode (40+) + BusinessException | ✅ |
| ApiResponse\<T\>, BaseTimeEntity, RequestIdFilter, SwaggerConfig, CORS, JpaAuditingConfig | ✅ |

---

## 테스트 현황

| 테스트 클래스 | 메서드 수 | 영역 |
|--------------|:---:|------|
| AccountServiceTest | 6 | 계좌 생성, 인증, 충전, 조회, 거래 내역 |
| AccountControllerTest | — | 컨트롤러 통합 |
| AuthServiceTest | — | 인증 서비스 |
| AuthControllerTest | — | 인증 컨트롤러 |
| PaymentServiceTest | 4 | 결제 성공, 미인증, 취소, 잔액 부족 |
| TransferConcurrencyTest | 2 | 동시 송금, 잔액 부족 |
| SettlementServiceTest | 4 | 가맹점별 조회, 날짜별 조회, 빈 결과, 전체 필드 검증 |
| NotificationServiceTest | 4 | SSE 구독/교체, 미구독 전송, 정상 전송 |
| NotificationConsumerTest | 4 | 송금 양측 알림, 결제 알림, 잘못된 JSON 처리 |
| TestRedisConfig | — | (설정 클래스) |
| EasyPayApplicationTests | 1 | 컨텍스트 로드 |
| **합계** | **~48** | |

---

## 미완료 항목

~~모든 항목 완료~~ ✅

- ~~부하 테스트 실행~~ ✅ 2026-02-09 (4개 시나리오 ALL PASS, p95 < 10ms, 성공률 99.75~100%)

---

## 인프라 (docker-compose.yml)

| 서비스 | 이미지 | 포트 | 상태 |
|--------|--------|------|:---:|
| PostgreSQL 16 | postgres:16-alpine | 5432 | ✅ |
| Redis 7 | redis:7-alpine | 6379 | ✅ |
| Kafka | confluentinc/cp-kafka:7.5.0 | 9092 | ✅ |
| Zookeeper | confluentinc/cp-zookeeper:7.5.0 | 2181 | ✅ |
| Prometheus | prom/prometheus:latest | 9090 | ✅ |
| Grafana | grafana/grafana:latest | 3000 | ✅ |

### Docker 컨테이너화 — ✅ 완료 (`e103275`)

- `Dockerfile`: 멀티스테이지 빌드 (Gradle → JRE 21)
- `application-docker.yml`: Docker 환경 전용 설정
- `docker-compose.yml`에 `app` 서비스 추가
- `docker-compose up -d`로 전체 환경(인프라+앱) 일괄 기동 가능

---

## 부하 테스트 실행 결과 (2026-02-09)

### 주요 변경사항

1. **Rate Limit 외부화** — ApiRateLimitFilter, TransferRateLimitService 설정값을 `@Value`로 외부화 (`application.yml`에서 관리)
2. **k6 스크립트 계좌 소유권 매핑 수정** — `concurrent-transfer.js`, `hot-account.js`에서 계좌 ID와 사용자 매핑 정확성 개선
3. **멱등성 테스트 memo 수정** — `idempotency-retry.js`에서 동일한 멱등성 키에 대해 memo 필드 일치하도록 수정

### 테스트 시나리오 및 결과

| 시나리오 | 가상 사용자 | 기간 | 결과 | TPS | p95 | 성공률 |
|---------|:---:|:---:|:---:|:---:|:---:|:---:|
| 1. 동시 송금 정합성 | 50 | 1분 | ✅ PASS | 381/s | 8.95ms | 100% |
| 2. Hot Account 집중 | 30 | 1분 | ✅ PASS | 164/s | 8.51ms | 100% |
| 3. 멱등성 재시도 | 20 | 1분 | ✅ PASS | 165/s | 5.98ms | 100% |
| 4. 스트레스 (Ramping) | 0→50 | 3분 | ✅ PASS | 105/s | 6.57ms | 99.75% |

**종합 결과**: 4/4 시나리오 ALL PASS. p95 레이턴시 < 10ms, 성공률 99.75~100%. 분산 락 데드락 없음, 잔액 정합성 유지, 멱등성 키 중복 방어 정상 동작.

상세 리포트: `docs/LOAD_TEST_REPORT.md`
