# EasyPay

토스/카카오페이 스타일의 간편 송금/결제 시스템. 동시성 제어, 분산 트랜잭션(Saga), 대용량 트래픽 처리를 핵심 기술 과제로 구현한 풀스택 프로젝트.

## 주요 기술 과제

- **3단계 동시성 제어**: Redis 분산 락(Redisson) + JPA 낙관적 락(@Version) + 멱등성 키(Redis TTL 24h)
- **Saga 패턴**: 출금 → 입금 순서 실행, 실패 시 보상 트랜잭션 자동 롤백
- **부하 테스트 검증**: k6 4개 시나리오 ALL PASS — p95 < 10ms, 에러율 0.25% 미만, 최대 381 TPS

## 기술 스택

### Backend

| 분류 | 기술 |
|------|------|
| Language | Java 21 |
| Framework | Spring Boot 3.4.2, Spring Security, Spring Batch |
| Database | PostgreSQL 16 |
| Cache / Lock | Redis 7 (Redisson 분산 락) |
| Message Queue | Apache Kafka 7.5 |
| Monitoring | Prometheus + Grafana, Micrometer |
| API Docs | Springdoc OpenAPI (Swagger UI) |
| Test | JUnit 5 (48개), k6 부하 테스트 (7 시나리오) |

### Frontend

| 분류 | 기술 |
|------|------|
| Framework | Next.js 16 (App Router) + TypeScript |
| UI | Tailwind CSS 4 + shadcn/ui |
| Server State | TanStack React Query |
| Client State | Zustand |
| Form / Validation | React Hook Form + Zod 4 |
| HTTP | Axios (인터셉터 기반 토큰 관리) |
| Test | Vitest + RTL (29개), Playwright E2E (4개) |

## 아키텍처

```
┌─────────────────────────────────────────────────────────────┐
│                    Frontend (Next.js 16)                     │
│  App Router · React Query · Zustand · SSE 실시간 알림        │
└────────────────────────┬────────────────────────────────────┘
                         │ /api/v1/* (rewrite → :8080)
┌────────────────────────▼────────────────────────────────────┐
│                  Backend (Spring Boot 3.4.2)                 │
│                     모듈러 모노리스                           │
│                                                              │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌────────────────┐  │
│  │   User   │ │ Account  │ │ Transfer │ │    Payment     │  │
│  │ JWT Auth │ │ @Version │ │ Saga+Lock│ │  Idempotency   │  │
│  └──────────┘ └──────────┘ └──────────┘ └────────────────┘  │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌────────────────┐  │
│  │Settlement│ │Notifica- │ │  Common  │ │     Infra      │  │
│  │  Batch   │ │  tion    │ │ Security │ │ Redis·Kafka·   │  │
│  │          │ │ SSE+Kafka│ │ ErrorCode│ │ Lock·Metrics   │  │
│  └──────────┘ └──────────┘ └──────────┘ └────────────────┘  │
└──┬──────────────┬──────────────┬──────────────┬─────────────┘
   │              │              │              │
┌──▼───┐   ┌─────▼────┐   ┌────▼───┐   ┌─────▼──────┐
│Postgr│   │  Redis   │   │ Kafka  │   │ Prometheus │
│ SQL  │   │ Lock/TTL │   │ Events │   │ + Grafana  │
└──────┘   └──────────┘   └────────┘   └────────────┘
```

### 송금 처리 흐름

```
요청 → Rate Limit → 멱등성 검증 → 분산 락 획득 → Saga 실행 → 캐시 저장
                                                     │
                                    ┌────────────────┼────────────────┐
                                    ▼                ▼                ▼
                              송금 생성          출금 처리          입금 처리
                            (PENDING)        (@Version 락)     (@Version 락)
                                    │                │                │
                                    │    실패 시 역순 보상 트랜잭션     │
                                    └────────────────┴────────────────┘
```

## 빠른 시작

### 사전 요구사항

- Docker & Docker Compose
- Java 21 (로컬 실행 시)
- Node.js 20+ & pnpm (프론트엔드)

### Docker로 전체 실행

```bash
# 백엔드 + 인프라 전체 실행
cd backend && docker-compose up -d

# 프론트엔드 실행
cd frontend && pnpm install && pnpm dev
```

### 로컬 개발

```bash
# 1. 인프라만 Docker로 실행
cd backend && docker-compose up -d postgres redis zookeeper kafka prometheus grafana

# 2. 백엔드 로컬 실행
cd backend && ./gradlew bootRun

# 3. 프론트엔드 실행
cd frontend && pnpm dev
```

### 접속 URL

| 서비스 | URL |
|--------|-----|
| 프론트엔드 | http://localhost:3001 |
| 백엔드 API | http://localhost:8080 |
| Swagger UI | http://localhost:8080/swagger-ui |
| Grafana | http://localhost:3000 (인프라) |
| Prometheus | http://localhost:9090 |

## 프로젝트 구조

```
EasyPay/
├── backend/                    # Spring Boot 3.4.2 / Java 21
│   ├── src/main/java/com/easypay/
│   │   ├── user/               # 인증 (JWT Access/Refresh, BCrypt, PIN)
│   │   ├── account/            # 계좌, 입출금, 거래 내역
│   │   ├── transfer/           # 송금 (분산 락 + 멱등성 + Saga)
│   │   ├── payment/            # 결제, 가맹점 관리
│   │   ├── settlement/         # 일별 정산 (Spring Batch)
│   │   ├── notification/       # 실시간 알림 (Kafka → SSE)
│   │   ├── common/             # Security, 예외 처리, ApiResponse
│   │   └── infra/              # Redis, Kafka, 분산 락, 메트릭
│   ├── k6/                     # 부하 테스트 스크립트 (7 시나리오)
│   └── docker-compose.yml
├── frontend/                   # Next.js 16 / TypeScript / pnpm
│   └── src/
│       ├── app/(auth)/         # 로그인, 회원가입
│       ├── app/(main)/         # 대시보드, 계좌, 송금, 결제, 정산
│       ├── components/         # UI 컴포넌트 (shadcn/ui + 커스텀)
│       └── lib/                # API, hooks, store, types, utils
└── docs/                       # PRD, 진행사항, 부하 테스트 보고서
```

## 핵심 기능

### 송금 시스템

5계층 동시성 제어로 잔액 정합성 보장:

1. **API Rate Limit** — Redis Sorted Set 슬라이딩 윈도우 (시간당 30회, 1건 200만원)
2. **멱등성 검증** — `Idempotency-Key` 헤더, Redis 캐시 24시간 TTL
3. **분산 락** — Redisson MultiLock, 데드락 방지를 위해 계좌 ID 정렬
4. **Saga 오케스트레이터** — 4단계 실행, 실패 시 역순 보상 트랜잭션
5. **낙관적 락** — `@Version` 어노테이션, DB 레벨 최종 안전망

### 결제 시스템

- 멱등성 키 기반 결제/취소
- Kafka 이벤트 발행으로 비동기 알림

### 정산 배치

- Spring Batch 일별 정산
- 전일 완료 결제 → 가맹점별 수수료 3% 차감 → Settlement 생성
- Chunk 1,000건 단위 대량 처리

### 실시간 알림

- Kafka Consumer → SSE(Server-Sent Events)
- 프론트엔드 토스트 알림 자동 표시

### 보안

- JWT Access/Refresh Token (Refresh는 Redis 저장, 로그아웃 시 폐기)
- BCrypt 비밀번호/PIN 해싱
- API Rate Limiting (인증 100req/min, 미인증 20req/min)
- Spring Security 필터 체인

## API 엔드포인트

| 메서드 | 경로 | 설명 |
|--------|------|------|
| POST | `/api/v1/auth/signup` | 회원가입 |
| POST | `/api/v1/auth/login` | 로그인 |
| POST | `/api/v1/auth/refresh` | 토큰 갱신 |
| POST | `/api/v1/auth/logout` | 로그아웃 |
| POST | `/api/v1/accounts` | 계좌 생성 |
| GET | `/api/v1/accounts` | 내 계좌 목록 |
| GET | `/api/v1/accounts/{id}/balance` | 잔액 조회 |
| POST | `/api/v1/accounts/{id}/deposit` | 입금 |
| POST | `/api/v1/accounts/{id}/withdraw` | 출금 |
| GET | `/api/v1/accounts/{id}/transactions` | 거래 내역 |
| POST | `/api/v1/transfers` | 송금 (`Idempotency-Key` 필수) |
| GET | `/api/v1/transfers/{id}` | 송금 상세 |
| POST | `/api/v1/transfers/{id}/cancel` | 송금 취소 (30분 이내) |
| POST | `/api/v1/payments` | 결제 (`Idempotency-Key` 필수) |
| POST | `/api/v1/payments/{id}/cancel` | 결제 취소 |
| POST | `/api/v1/merchants` | 가맹점 등록 |

## 부하 테스트 결과

k6로 4개 핵심 시나리오 실행, **전체 ALL PASS**:

| 시나리오 | VU | 총 요청 | 성공률 | TPS | p95 |
|---------|-----|--------|--------|-----|-----|
| 동시 송금 정합성 | 50 | 23,196 | 100% | 381/s | 8.95ms |
| Hot Account 집중 | 30 | 9,970 | 100% | 164/s | 8.51ms |
| 멱등성 재시도 | 20 | 10,150 | 100% | 165/s | 5.98ms |
| 스트레스 (Ramping) | 0→50 | 18,989 | 99.75% | 105/s | 6.57ms |

- SLA 목표 p95 < 500ms 대비 **50배 이상 여유** (실측 p95 < 10ms)
- 상세 보고서: [`docs/LOAD_TEST_REPORT.md`](docs/LOAD_TEST_REPORT.md)

## 테스트

```bash
# 백엔드 단위/통합 테스트 (48개, H2 인메모리)
cd backend && ./gradlew test

# 프론트엔드 단위 테스트 (29개, Vitest)
cd frontend && pnpm test

# 프론트엔드 E2E 테스트 (4개, Playwright)
cd frontend && pnpm test:e2e

# 부하 테스트 (k6, Docker 환경 권장)
cd backend && docker run --rm --network backend_default \
  -v "$(pwd)/k6:/scripts" grafana/k6:latest \
  run -e BASE_URL=http://easypay-app:8080 \
  /scripts/scenarios/concurrent-transfer.js
```

## 모니터링

- **Prometheus** (`localhost:9090`) — 메트릭 수집
- **Grafana** (`localhost:3000`) — Application Overview, Transfer Monitoring 대시보드
- **Actuator** (`localhost:8080/actuator/prometheus`) — JVM, HTTP, 커스텀 메트릭
- **Alert 규칙**: p95 > 1s, 에러율 > 5%, 송금 실패율 > 5%, JVM Heap > 90%

## 문서

| 문서 | 설명 |
|------|------|
| [`docs/PRD.md`](docs/PRD.md) | 제품 요구사항 정의서 |
| [`docs/BACKEND_PROGRESS.md`](docs/BACKEND_PROGRESS.md) | 백엔드 구현 현황 |
| [`docs/FRONTEND_PROGRESS.md`](docs/FRONTEND_PROGRESS.md) | 프론트엔드 구현 현황 |
| [`docs/LOAD_TEST_REPORT.md`](docs/LOAD_TEST_REPORT.md) | 부하 테스트 결과 보고서 |
| [`docs/TODO.md`](docs/TODO.md) | 작업 체크리스트 |
| [`backend/README.md`](backend/README.md) | 백엔드 상세 문서 |
| [`frontend/README.md`](frontend/README.md) | 프론트엔드 상세 문서 |
