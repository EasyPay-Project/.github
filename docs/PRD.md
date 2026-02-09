# Product Requirements Document (PRD)

## 간편 송금/결제 시스템 (EasyPay)

**버전**: 1.0
**작성일**: 2026-02-08
**프로젝트 유형**: 개인 프로젝트 (대용량 트래픽 처리 역량 강화)

---

## 1. 프로젝트 개요 및 목표

### 1.1 개요 (Overview)

본 프로젝트는 **토스/카카오페이와 유사한 간편 송금 서비스**를 Java Spring Boot 기반으로 구현하는 개인 프로젝트이다. 핵심 목표는 **동시성 제어, 분산 트랜잭션, 대용량 배치 처리** 등 실무에서 요구되는 고난도 백엔드 기술을 직접 설계하고 구현하여, 대용량 트래픽 환경에서의 문제 해결 경험을 쌓는 것이다.

### 1.2 배경 및 목적 (Background & Purpose)

#### 배경

- 핀테크 서비스의 핵심 기술인 **잔액 정합성 보장**, **중복 결제 방지**, **실패 시 자동 보상**은 백엔드 개발자에게 가장 중요한 역량으로 평가된다
- 실제 면접에서 자주 출제되는 주제(동시성 제어, 분산 락, Saga 패턴)를 직접 구현하여 깊은 이해를 확보하고자 한다
- 부하 테스트를 통해 병목 지점을 식별하고 최적화하는 과정 자체가 핵심 학습 목표이다

#### 목적

| 목적 | 설명 | 측정 방법 |
|------|------|----------|
| **동시성 제어 역량** | 동시 1,000건 송금 요청에서 잔액 정합성 100% 보장 | 부하 테스트 후 잔액 검증 |
| **분산 트랜잭션 이해** | Saga 패턴 기반 보상 트랜잭션 구현 | 의도적 실패 시나리오 테스트 |
| **대용량 처리 경험** | 100만 건 이상 거래 내역 배치 정산 | Spring Batch 처리 시간 측정 |
| **부하 테스트 역량** | TPS 측정 및 병목 분석 리포트 작성 | k6/nGrinder 결과 분석 보고서 |
| **모니터링 체계 구축** | 장애 조기 감지 및 원인 추적 가능 | Grafana 대시보드 구성 |

### 1.3 타겟 사용자 (Target Users)

| 사용자 유형 | 설명 | 주요 시나리오 |
|------------|------|-------------|
| 송금자 | 타인에게 돈을 보내는 사용자 | 계좌 송금, 잔액 조회, 송금 내역 확인 |
| 수취인 | 돈을 받는 사용자 | 입금 알림 수신, 잔액 확인 |
| 가맹점 | 결제를 수신하는 사업자 | 결제 수신, 정산 내역 조회 |
| 시스템 관리자 | 서비스 운영/모니터링 담당자 | 거래 모니터링, 정산 관리, 이상 탐지 |

---

## 2. 핵심 기능 목록

### 2.1 필수 기능 (Essential / Must-Have)

#### 2.1.1 사용자 관리

| 기능 | 설명 | 기술 포인트 |
|------|------|-----------|
| 회원가입/로그인 | JWT 기반 인증, Spring Security | Access Token + Refresh Token |
| 프로필 관리 | 사용자 기본 정보 CRUD | 비밀번호 암호화 (BCrypt) |
| PIN 인증 | 6자리 PIN 기반 송금/결제 인증 | Redis 세션 기반 인증 상태 관리 |

#### 2.1.2 계좌 관리

| 기능 | 설명 | 기술 포인트 |
|------|------|-----------|
| 가상 계좌 생성 | 사용자별 가상 계좌 발급 (입금 전용) | 계좌번호 생성 알고리즘 (UUID 기반) |
| 잔액 조회 | 실시간 잔액 확인 | Redis 캐시 + DB 정합성 검증 |
| 충전 | 가상 계좌로 잔액 충전 | 실제 PG 연동 없이 Mock 처리 |
| 거래 내역 조회 | 송금/수신 내역 조회 (페이징) | Cursor-based Pagination |

#### 2.1.3 송금

| 기능 | 설명 | 기술 포인트 |
|------|------|-----------|
| 1:1 송금 | 사용자 간 송금 | **Saga 패턴 + 분산 락** |
| 멱등성 보장 | Idempotency Key 기반 중복 송금 방지 | **Redis 기반 멱등성 키 관리** |
| 송금 한도 관리 | 1회/1일 송금 한도 설정 | **Rate Limiting (Sliding Window)** |
| 송금 취소 | 일정 시간 내 송금 취소 | 보상 트랜잭션 (Compensating Transaction) |

#### 2.1.4 결제

| 기능 | 설명 | 기술 포인트 |
|------|------|-----------|
| 결제 요청 | 가맹점 결제 처리 | **멱등성 + 동시성 제어** |
| 결제 취소/환불 | 전액/부분 환불 처리 | 보상 트랜잭션 |
| 결제 상태 관리 | PENDING -> COMPLETED/FAILED/CANCELLED | 상태 머신 (State Machine) 패턴 |

#### 2.1.5 정산 배치

| 기능 | 설명 | 기술 포인트 |
|------|------|-----------|
| 일일 정산 | 가맹점별 일일 거래 집계 및 정산 | **Spring Batch (Chunk 기반)** |
| 정산 리포트 | 정산 결과 집계 및 조회 | Reader-Processor-Writer 패턴 |
| 실패 재처리 | 정산 실패 건 자동 재처리 | Skip/Retry 정책 |

#### 2.1.6 실시간 알림

| 기능 | 설명 | 기술 포인트 |
|------|------|-----------|
| 송금/수신 알림 | 거래 발생 시 실시간 알림 | **Kafka 이벤트 발행 + SSE/WebSocket** |
| 정산 완료 알림 | 가맹점 정산 완료 통지 | Kafka Consumer + 푸시 알림 |

### 2.2 선택 기능 (Nice-to-Have)

| 기능 | 설명 | 구현 시 학습 효과 |
|------|------|-----------------|
| 다건 송금 | 여러 명에게 동시에 송금 | 비동기 병렬 처리, CompletableFuture |
| 예약 송금 | 특정 시각에 자동 송금 | Spring Scheduler + Kafka Delayed Message |
| 거래 내역 검색 | 기간/금액/상대방 필터 검색 | Querydsl 동적 쿼리 |
| 이상 거래 탐지 | 비정상 패턴 실시간 감지 | Kafka Streams + 규칙 엔진 |
| 외부 은행 연동 Mock | 가상 외부 은행 API | Resilience4j Circuit Breaker |

### 2.3 제외 기능 (Out of Scope)

| 제외 기능 | 제외 사유 |
|----------|----------|
| 실제 PG/은행 연동 | 개인 프로젝트 범위 초과, 사업자 등록 필요 |
| KYC/AML 본인 인증 | 규제 준수 영역으로 개인 프로젝트에서 불필요 |
| 해외 송금/환전 | 핵심 학습 목표와 관련 없음 |
| 모바일 앱 개발 | 백엔드 역량 강화가 목적, API만 제공 |
| 실제 푸시 알림 (FCM) | SSE/WebSocket으로 알림 개념 증명에 충분 |

---

## 3. 시스템 아키텍처 개요

### 3.1 전체 아키텍처

```
                            ┌──────────────────────┐
                            │    Client (Postman    │
                            │    / Swagger UI)      │
                            └──────────┬───────────┘
                                       │ HTTPS
                                       ▼
                            ┌──────────────────────┐
                            │   API Gateway         │
                            │   (Spring Cloud       │
                            │    Gateway) [선택]     │
                            └──────────┬───────────┘
                                       │
              ┌────────────────────────┼────────────────────────┐
              │                        │                        │
              ▼                        ▼                        ▼
   ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
   │  Account Service │    │ Transfer Service│    │ Payment Service │
   │  (계좌 관리)      │    │ (송금 처리)      │    │ (결제 처리)      │
   └────────┬────────┘    └────────┬────────┘    └────────┬────────┘
            │                      │                      │
            └──────────────────────┼──────────────────────┘
                                   │
              ┌────────────────────┼────────────────────┐
              │                    │                    │
              ▼                    ▼                    ▼
   ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
   │   MySQL       │    │   Redis      │    │   Kafka      │
   │  (계좌/거래)   │    │ (분산 락,     │    │ (이벤트 버스) │
   │               │    │  캐시, Rate   │    │              │
   │               │    │  Limit)      │    │              │
   └──────────────┘    └──────────────┘    └──────────────┘
                                                  │
                                   ┌──────────────┼──────────────┐
                                   │              │              │
                                   ▼              ▼              ▼
                          ┌─────────────┐ ┌────────────┐ ┌─────────────┐
                          │ Notification │ │ Settlement │ │ Monitoring  │
                          │ Consumer    │ │ Batch      │ │ (Prometheus │
                          │ (알림 발송)  │ │ (정산 배치) │ │  + Grafana) │
                          └─────────────┘ └────────────┘ └─────────────┘
```

### 3.2 기술 스택 상세

| 카테고리 | 기술 | 버전 | 선택 사유 |
|---------|------|------|----------|
| **Language** | Java | 17+ | LTS 버전, 가상 스레드(Virtual Thread) 활용 가능 |
| **Framework** | Spring Boot | 3.x | 최신 LTS, Jakarta EE 전환 완료 |
| **Security** | Spring Security | 6.x | JWT 인증/인가, CSRF 방어 |
| **Batch** | Spring Batch | 5.x | 대용량 정산 배치 처리 |
| **ORM** | Spring Data JPA + Querydsl | - | 동적 쿼리, 타입 안전성 |
| **DB** | MySQL | 8.0+ | 트랜잭션 ACID 보장, InnoDB 락 활용 |
| **Cache/Lock** | Redis | 7.x | 분산 락 (Redisson), 캐시, Rate Limit, 멱등성 키 |
| **Message Broker** | Apache Kafka | 3.x | 이벤트 드리븐 아키텍처, 순서 보장 |
| **Container** | Docker Compose | - | 로컬 개발 환경 통합 |
| **Monitoring** | Prometheus + Grafana | - | 메트릭 수집 및 시각화 |
| **Load Test** | k6 / nGrinder | - | TPS 측정 및 병목 분석 |
| **API Docs** | Springdoc OpenAPI | 2.x | Swagger UI 자동 생성 |
| **Build** | Gradle | 8.x | 빌드 속도 및 의존성 관리 |

### 3.3 모노리스 vs 모듈러 모노리스 결정

**결론: 모듈러 모노리스 (Modular Monolith) 채택**

| 기준 | 마이크로서비스 | 모듈러 모노리스 | 결정 |
|------|-------------|---------------|------|
| 개발 속도 | 느림 (서비스 간 통신 구현) | 빠름 (단일 프로세스) | 모듈러 모노리스 |
| 운영 복잡도 | 높음 (서비스 디스커버리, 분산 추적) | 낮음 | 모듈러 모노리스 |
| 학습 가치 | 분산 시스템 경험 | 동시성/배치 집중 학습 | 모듈러 모노리스 |
| 개인 프로젝트 적합성 | 낮음 (인프라 부담) | 높음 | **모듈러 모노리스** |

**패키지 구조**:

```
com.easypay/
├── common/             # 공통 모듈 (예외, 유틸, 응답 포맷)
│   ├── exception/
│   ├── response/
│   └── config/
├── user/               # 사용자 모듈
│   ├── controller/
│   ├── service/
│   ├── domain/
│   └── repository/
├── account/            # 계좌 모듈
│   ├── controller/
│   ├── service/
│   ├── domain/
│   └── repository/
├── transfer/           # 송금 모듈
│   ├── controller/
│   ├── service/
│   ├── domain/
│   ├── repository/
│   └── saga/           # Saga 오케스트레이터
├── payment/            # 결제 모듈
│   ├── controller/
│   ├── service/
│   ├── domain/
│   └── repository/
├── settlement/         # 정산 배치 모듈
│   ├── batch/
│   ├── domain/
│   └── repository/
├── notification/       # 알림 모듈
│   ├── consumer/
│   ├── service/
│   └── dto/
└── infra/              # 인프라 모듈
    ├── redis/          # Redisson 분산 락, Rate Limiter
    ├── kafka/          # Producer/Consumer 설정
    └── monitoring/     # Prometheus 메트릭
```

---

## 4. API 설계

### 4.1 주요 엔드포인트

#### 4.1.1 인증 (Auth)

| Method | Endpoint | 설명 | 인증 |
|--------|----------|------|------|
| POST | `/api/v1/auth/signup` | 회원가입 | 불필요 |
| POST | `/api/v1/auth/login` | 로그인 (JWT 발급) | 불필요 |
| POST | `/api/v1/auth/refresh` | 토큰 갱신 | Refresh Token |
| POST | `/api/v1/auth/pin/verify` | PIN 인증 | JWT |

#### 4.1.2 계좌 (Account)

| Method | Endpoint | 설명 | 인증 |
|--------|----------|------|------|
| POST | `/api/v1/accounts` | 계좌 생성 | JWT |
| GET | `/api/v1/accounts/{accountId}` | 계좌 상세 조회 | JWT + 본인 확인 |
| GET | `/api/v1/accounts/{accountId}/balance` | 잔액 조회 | JWT + 본인 확인 |
| POST | `/api/v1/accounts/{accountId}/deposit` | 잔액 충전 | JWT + PIN |
| GET | `/api/v1/accounts/{accountId}/transactions` | 거래 내역 조회 | JWT + 본인 확인 |

#### 4.1.3 송금 (Transfer)

| Method | Endpoint | 설명 | 인증 | 특이사항 |
|--------|----------|------|------|---------|
| POST | `/api/v1/transfers` | 송금 요청 | JWT + PIN | **Idempotency-Key 헤더 필수** |
| GET | `/api/v1/transfers/{transferId}` | 송금 상태 조회 | JWT | 상태 추적 |
| POST | `/api/v1/transfers/{transferId}/cancel` | 송금 취소 | JWT + PIN | 보상 트랜잭션 실행 |

#### 4.1.4 결제 (Payment)

| Method | Endpoint | 설명 | 인증 | 특이사항 |
|--------|----------|------|------|---------|
| POST | `/api/v1/payments` | 결제 요청 | JWT + PIN | **Idempotency-Key 헤더 필수** |
| GET | `/api/v1/payments/{paymentId}` | 결제 상태 조회 | JWT | - |
| POST | `/api/v1/payments/{paymentId}/cancel` | 결제 취소 | JWT + PIN | 전액/부분 취소 |
| POST | `/api/v1/payments/{paymentId}/refund` | 환불 요청 | JWT (가맹점) | 가맹점 측 환불 |

#### 4.1.5 정산 (Settlement)

| Method | Endpoint | 설명 | 인증 |
|--------|----------|------|------|
| GET | `/api/v1/settlements` | 정산 내역 조회 | JWT (가맹점) |
| GET | `/api/v1/settlements/{date}` | 일별 정산 상세 | JWT (가맹점) |
| POST | `/api/v1/admin/settlements/run` | 수동 정산 실행 (관리자) | JWT (Admin) |

### 4.2 공통 응답 형식

```json
{
  "success": true,
  "data": { ... },
  "error": null,
  "timestamp": "2026-02-08T10:30:00+09:00"
}
```

```json
{
  "success": false,
  "data": null,
  "error": {
    "code": "INSUFFICIENT_BALANCE",
    "message": "잔액이 부족합니다. 현재 잔액: 5,000원, 요청 금액: 10,000원"
  },
  "timestamp": "2026-02-08T10:30:00+09:00"
}
```

### 4.3 송금 API 상세 (핵심 API)

**요청**:

```http
POST /api/v1/transfers
Content-Type: application/json
Authorization: Bearer {jwt_token}
Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000

{
  "senderAccountId": 1,
  "receiverAccountId": 2,
  "amount": 50000,
  "memo": "밥값"
}
```

**응답 (201 Created)**:

```json
{
  "success": true,
  "data": {
    "transferId": "TRF-20260208-000001",
    "status": "COMPLETED",
    "senderAccountId": 1,
    "receiverAccountId": 2,
    "amount": 50000,
    "fee": 0,
    "memo": "밥값",
    "senderBalanceAfter": 950000,
    "createdAt": "2026-02-08T10:30:00+09:00"
  },
  "error": null,
  "timestamp": "2026-02-08T10:30:00+09:00"
}
```

---

## 5. 데이터 모델 (ERD 개요)

### 5.1 핵심 엔티티

```
┌───────────────┐     ┌───────────────────┐     ┌──────────────────┐
│    User        │     │    Account         │     │   Transfer       │
├───────────────┤     ├───────────────────┤     ├──────────────────┤
│ id (PK)       │──┐  │ id (PK)           │  ┌──│ id (PK)          │
│ email         │  │  │ user_id (FK)      │──┘  │ transfer_code    │
│ password_hash │  └──│ account_number    │     │ sender_id (FK)   │
│ name          │     │ balance           │     │ receiver_id (FK) │
│ pin_hash      │     │ status            │     │ amount           │
│ role          │     │ daily_limit       │     │ fee              │
│ created_at    │     │ created_at        │     │ status           │
│ updated_at    │     │ updated_at        │     │ idempotency_key  │
└───────────────┘     │ version (낙관적 락)│     │ memo             │
                      └───────────────────┘     │ created_at       │
                                                │ completed_at     │
                                                └──────────────────┘

┌──────────────────┐     ┌────────────────────┐     ┌──────────────────┐
│  Transaction      │     │  Payment           │     │  Settlement      │
├──────────────────┤     ├────────────────────┤     ├──────────────────┤
│ id (PK)          │     │ id (PK)            │     │ id (PK)          │
│ account_id (FK)  │     │ payment_code       │     │ merchant_id (FK) │
│ transfer_id (FK) │     │ payer_id (FK)      │     │ settlement_date  │
│ type (DEBIT/     │     │ merchant_id (FK)   │     │ total_amount     │
│       CREDIT)    │     │ amount             │     │ fee_amount       │
│ amount           │     │ status             │     │ net_amount       │
│ balance_after    │     │ idempotency_key    │     │ transaction_count│
│ description      │     │ created_at         │     │ status           │
│ created_at       │     │ completed_at       │     │ created_at       │
└──────────────────┘     └────────────────────┘     └──────────────────┘

┌──────────────────┐     ┌──────────────────┐
│ IdempotencyKey    │     │  RateLimit        │
├──────────────────┤     ├──────────────────┤
│ key (PK)         │     │ user_id           │
│ request_hash     │     │ window_start      │
│ response_body    │     │ request_count     │
│ status_code      │     │ total_amount      │
│ created_at       │     │ (Redis 관리)       │
│ expires_at       │     └──────────────────┘
└──────────────────┘
```

### 5.2 테이블 DDL 핵심 (MySQL)

```sql
-- 계좌 테이블: 낙관적 락을 위한 version 컬럼 포함
CREATE TABLE account (
    id              BIGINT AUTO_INCREMENT PRIMARY KEY,
    user_id         BIGINT NOT NULL,
    account_number  VARCHAR(20) NOT NULL UNIQUE,
    balance         DECIMAL(15, 2) NOT NULL DEFAULT 0.00,
    status          ENUM('ACTIVE', 'FROZEN', 'CLOSED') NOT NULL DEFAULT 'ACTIVE',
    daily_limit     DECIMAL(15, 2) NOT NULL DEFAULT 5000000.00,
    version         BIGINT NOT NULL DEFAULT 0,  -- @Version (Optimistic Lock)
    created_at      DATETIME(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6),
    updated_at      DATETIME(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6) ON UPDATE CURRENT_TIMESTAMP(6),

    INDEX idx_account_user_id (user_id),
    CONSTRAINT fk_account_user FOREIGN KEY (user_id) REFERENCES user(id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- 송금 테이블: 멱등성 키 유니크 인덱스
CREATE TABLE transfer (
    id              BIGINT AUTO_INCREMENT PRIMARY KEY,
    transfer_code   VARCHAR(30) NOT NULL UNIQUE,
    sender_id       BIGINT NOT NULL,
    receiver_id     BIGINT NOT NULL,
    amount          DECIMAL(15, 2) NOT NULL,
    fee             DECIMAL(15, 2) NOT NULL DEFAULT 0.00,
    status          ENUM('PENDING', 'PROCESSING', 'COMPLETED', 'FAILED', 'CANCELLED', 'COMPENSATED') NOT NULL DEFAULT 'PENDING',
    idempotency_key VARCHAR(64) NOT NULL,
    memo            VARCHAR(100),
    failure_reason  VARCHAR(500),
    created_at      DATETIME(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6),
    completed_at    DATETIME(6),

    UNIQUE INDEX uk_transfer_idempotency (idempotency_key),
    INDEX idx_transfer_sender (sender_id),
    INDEX idx_transfer_receiver (receiver_id),
    INDEX idx_transfer_status_created (status, created_at),
    CONSTRAINT fk_transfer_sender FOREIGN KEY (sender_id) REFERENCES account(id),
    CONSTRAINT fk_transfer_receiver FOREIGN KEY (receiver_id) REFERENCES account(id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- 거래 내역 테이블: 계좌별 잔액 변동 이력
CREATE TABLE transaction (
    id              BIGINT AUTO_INCREMENT PRIMARY KEY,
    account_id      BIGINT NOT NULL,
    transfer_id     BIGINT,
    payment_id      BIGINT,
    type            ENUM('DEBIT', 'CREDIT') NOT NULL,
    amount          DECIMAL(15, 2) NOT NULL,
    balance_after   DECIMAL(15, 2) NOT NULL,
    description     VARCHAR(200),
    created_at      DATETIME(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6),

    INDEX idx_tx_account_created (account_id, created_at DESC),
    INDEX idx_tx_transfer (transfer_id),
    CONSTRAINT fk_tx_account FOREIGN KEY (account_id) REFERENCES account(id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

### 5.3 인덱스 전략

| 테이블 | 인덱스 | 용도 | 타입 |
|--------|--------|------|------|
| `transfer` | `uk_transfer_idempotency` | 멱등성 키 중복 체크 | UNIQUE |
| `transfer` | `idx_transfer_status_created` | 상태별 조회 + 배치 정산 대상 조회 | COMPOSITE |
| `transaction` | `idx_tx_account_created` | 계좌별 거래 내역 최신순 조회 | COMPOSITE |
| `account` | `idx_account_user_id` | 사용자별 계좌 조회 | SINGLE |

---

## 6. 동시성 제어 전략 상세

### 6.1 전략 개요

본 프로젝트에서는 **3단계 방어선** 전략을 채택한다.

```
[1단계] Redis 분산 락 (Redisson)
    → 동일 계좌에 대한 동시 요청을 직렬화

[2단계] DB 낙관적 락 (@Version)
    → 분산 락을 통과한 경우에도 DB 레벨에서 정합성 보장

[3단계] 멱등성 키 (Idempotency Key)
    → 네트워크 재시도 등으로 인한 중복 요청 방지
```

### 6.2 Redis 분산 락 (Redisson) 상세

**선택 사유**: 단순 `SETNX`가 아닌 Redisson을 사용하는 이유

| 비교 항목 | SETNX + Lua Script | Redisson (RLock) |
|----------|-------------------|-----------------|
| 락 재진입 (Reentrant) | 직접 구현 필요 | 기본 지원 |
| 락 만료 자동 연장 (Watchdog) | 미지원 | 기본 지원 (30초 기본, 자동 연장) |
| 페어 락 (Fair Lock) | 미지원 | 기본 지원 |
| Pub/Sub 기반 대기 | 폴링 방식 | Pub/Sub 기반 효율적 대기 |
| 구현 복잡도 | 높음 | 낮음 |

**구현 전략**:

```java
@Service
@RequiredArgsConstructor
public class TransferService {

    private final RedissonClient redissonClient;
    private final AccountRepository accountRepository;

    @Transactional
    public TransferResponse transfer(TransferRequest request, String idempotencyKey) {
        // [3단계] 멱등성 검증
        Optional<Transfer> existing = transferRepository.findByIdempotencyKey(idempotencyKey);
        if (existing.isPresent()) {
            return TransferResponse.from(existing.get());
        }

        // 락 키 순서 정렬 (데드락 방지)
        String[] lockKeys = sortLockKeys(
            "lock:account:" + request.getSenderAccountId(),
            "lock:account:" + request.getReceiverAccountId()
        );

        // [1단계] Redis 분산 락 획득 (Multi-Lock)
        RLock lock1 = redissonClient.getLock(lockKeys[0]);
        RLock lock2 = redissonClient.getLock(lockKeys[1]);
        RLock multiLock = redissonClient.getMultiLock(lock1, lock2);

        boolean locked = multiLock.tryLock(5, 10, TimeUnit.SECONDS);
        if (!locked) {
            throw new ConcurrencyException("송금 처리 중입니다. 잠시 후 다시 시도해주세요.");
        }

        try {
            // [2단계] 낙관적 락으로 잔액 차감
            Account sender = accountRepository.findById(request.getSenderAccountId())
                .orElseThrow(() -> new AccountNotFoundException("송금 계좌를 찾을 수 없습니다."));

            sender.withdraw(request.getAmount());  // 잔액 부족 시 예외 발생
            accountRepository.save(sender);        // @Version 충돌 시 OptimisticLockException

            Account receiver = accountRepository.findById(request.getReceiverAccountId())
                .orElseThrow(() -> new AccountNotFoundException("수취 계좌를 찾을 수 없습니다."));

            receiver.deposit(request.getAmount());
            accountRepository.save(receiver);

            // 송금 기록 생성
            Transfer transfer = Transfer.create(request, idempotencyKey);
            transfer.complete();
            transferRepository.save(transfer);

            // Kafka 이벤트 발행 (비동기)
            kafkaTemplate.send("transfer-events", TransferEvent.completed(transfer));

            return TransferResponse.from(transfer);
        } finally {
            multiLock.unlock();
        }
    }

    // 데드락 방지: 항상 작은 ID의 계좌부터 락 획득
    private String[] sortLockKeys(String key1, String key2) {
        return key1.compareTo(key2) < 0
            ? new String[]{key1, key2}
            : new String[]{key2, key1};
    }
}
```

### 6.3 데드락 방지 전략

| 시나리오 | 문제 | 해결 |
|---------|------|------|
| A->B 송금과 B->A 송금 동시 발생 | 교차 락 (Cross Lock) | **락 키 정렬**: 항상 작은 ID의 계좌부터 락 획득 |
| 락 보유 중 장애 발생 | 락 미해제 (Dead Lock) | **Redisson Watchdog**: TTL 자동 관리 (기본 30초) |
| 락 대기 시간 초과 | 무한 대기 | **tryLock 타임아웃**: 5초 대기 후 실패 반환 |

### 6.4 낙관적 락 vs 비관적 락 결정

**결론: 낙관적 락 (Optimistic Lock) 채택 (분산 락과 조합)**

| 기준 | 비관적 락 (`SELECT FOR UPDATE`) | 낙관적 락 (`@Version`) |
|------|-------------------------------|----------------------|
| DB 락 범위 | Row 단위 락 유지 | 락 없음 (커밋 시 버전 체크) |
| 동시성 높을 때 | 대기 시간 증가, 커넥션 풀 고갈 위험 | 충돌 시 재시도 필요 |
| 본 프로젝트 적합성 | 분산 락과 중복, 불필요한 DB 부하 | **분산 락이 1차 방어 → 낙관적 락은 안전장치** |

분산 락(Redisson)이 1차 동시성 제어를 담당하므로, DB 레벨에서는 낙관적 락으로 최종 안전망 역할만 수행하면 충분하다. 비관적 락은 커넥션 보유 시간이 길어져 TPS 저하 원인이 될 수 있다.

---

## 7. 분산 트랜잭션: Saga 패턴

### 7.1 Saga 오케스트레이션 패턴 채택

**Choreography vs Orchestration 비교**:

| 기준 | Choreography (이벤트 기반) | Orchestration (중앙 조율) |
|------|--------------------------|-------------------------|
| 복잡도 관리 | 이벤트 흐름 추적 어려움 | 중앙에서 흐름 관리 |
| 디버깅 | 분산 이벤트 추적 필요 | 오케스트레이터 로그로 추적 |
| 보상 트랜잭션 | 각 서비스가 독립 관리 | 오케스트레이터가 일괄 관리 |
| 개인 프로젝트 적합성 | 낮음 (이벤트 추적 복잡) | **높음 (명시적 흐름 관리)** |

### 7.2 송금 Saga 흐름

```
[TransferSagaOrchestrator]

성공 시나리오:
  Step 1. 멱등성 키 검증 (Redis)
      ↓
  Step 2. 송금자 잔액 차감 (Account DB - DEBIT)
      ↓
  Step 3. 수취인 잔액 증가 (Account DB - CREDIT)
      ↓
  Step 4. 거래 기록 생성 (Transaction DB)
      ↓
  Step 5. 이벤트 발행 (Kafka → 알림/정산)
      ↓
  Step 6. 송금 상태 COMPLETED로 변경

실패 시나리오 (Step 3에서 실패):
  Step 3. 수취인 잔액 증가 실패 (수취 계좌 비활성)
      ↓
  Compensate Step 2. 송금자 잔액 복원 (CREDIT 보상 트랜잭션)
      ↓
  Compensate Step 4. 보상 거래 기록 생성
      ↓
  Step 6. 송금 상태 COMPENSATED로 변경
      ↓
  이벤트 발행: 송금 실패 알림
```

### 7.3 Saga 상태 머신

```
                  ┌──────────┐
                  │ PENDING  │
                  └────┬─────┘
                       │ 송금 요청 접수
                       ▼
                  ┌──────────┐
                  │PROCESSING│
                  └──┬────┬──┘
          성공 /     │    │     \ 실패
               ▼     │    │     ▼
        ┌──────────┐ │    │ ┌──────────┐
        │COMPLETED │ │    │ │  FAILED  │
        └──────────┘ │    │ └──────────┘
                     │    │
              취소 요청/  \ 보상 완료
                   ▼      ▼
            ┌──────────┐ ┌──────────────┐
            │CANCELLED │ │ COMPENSATED  │
            └──────────┘ └──────────────┘
```

---

## 8. 멱등성 (Idempotency) 설계

### 8.1 멱등성 처리 흐름

```
Client                     Server                      Redis                   DB
  │                          │                           │                      │
  │── POST /transfers ──────>│                           │                      │
  │   Idempotency-Key: ABC   │                           │                      │
  │                          │── GET idempotency:ABC ───>│                      │
  │                          │<── null (최초 요청) ───────│                      │
  │                          │                           │                      │
  │                          │── SET idempotency:ABC ───>│  (상태: PROCESSING)  │
  │                          │   (TTL: 24h)              │                      │
  │                          │                           │                      │
  │                          │── 송금 로직 실행 ──────────────────────────────────>│
  │                          │<── 결과 반환 ──────────────────────────────────────│
  │                          │                           │                      │
  │                          │── UPDATE idempotency:ABC ─>│  (상태: COMPLETED,   │
  │                          │   (응답 본문 저장)          │   응답 캐싱)          │
  │                          │                           │                      │
  │<── 201 Created ─────────│                           │                      │
  │                          │                           │                      │
  │                          │                           │                      │
  │── POST /transfers ──────>│  (재시도: 동일 키)          │                      │
  │   Idempotency-Key: ABC   │                           │                      │
  │                          │── GET idempotency:ABC ───>│                      │
  │                          │<── 캐싱된 응답 반환 ────────│                      │
  │                          │                           │                      │
  │<── 201 Created ─────────│  (동일 응답 반환, DB 미접근) │                      │
```

### 8.2 멱등성 키 저장 구조 (Redis)

```
Key:    idempotency:{key_value}
Value:  {
            "status": "COMPLETED",
            "statusCode": 201,
            "responseBody": "{...}",
            "requestHash": "sha256_of_request_body",
            "createdAt": "2026-02-08T10:30:00"
        }
TTL:    24시간 (86400초)
```

### 8.3 주의사항

| 시나리오 | 처리 방식 |
|---------|----------|
| 동일 키 + 동일 요청 본문 | 캐싱된 응답 반환 (정상) |
| 동일 키 + 다른 요청 본문 | **409 Conflict** 반환 (키 재사용 방지) |
| 키 없이 요청 | **400 Bad Request** 반환 (필수 헤더 누락) |
| 처리 중 동일 키 재요청 | **202 Accepted** 반환 (처리 중 안내) |

---

## 9. Rate Limiting 설계

### 9.1 Sliding Window Counter 알고리즘

**선택 사유**: Fixed Window는 윈도우 경계에서 2배 트래픽 허용 가능성이 있으므로, Sliding Window 방식을 채택한다.

### 9.2 제한 정책

| 제한 유형 | 기준 | 한도 | 윈도우 |
|----------|------|------|--------|
| 송금 횟수 | 사용자별 | 30회 | 1시간 |
| 송금 금액 | 사용자별 | 500만원 | 1일 |
| 1회 송금 한도 | 사용자별 | 200만원 | 건당 |
| 결제 횟수 | 사용자별 | 50회 | 1시간 |
| API 호출 | IP별 | 1,000회 | 1분 |

### 9.3 Redis 기반 구현

```java
@Component
@RequiredArgsConstructor
public class RateLimiter {

    private final StringRedisTemplate redisTemplate;

    /**
     * Sliding Window Counter 기반 Rate Limiting
     * @return true: 허용, false: 제한 초과
     */
    public boolean isAllowed(String key, int limit, Duration window) {
        String redisKey = "ratelimit:" + key;
        long now = Instant.now().toEpochMilli();
        long windowStart = now - window.toMillis();

        // Lua Script로 원자적 실행
        // 1. 윈도우 밖의 오래된 요소 제거
        // 2. 현재 요청 추가
        // 3. 현재 윈도우 내 요소 수 반환
        Long count = redisTemplate.execute(rateLimitScript,
            List.of(redisKey),
            String.valueOf(windowStart),
            String.valueOf(now),
            String.valueOf(window.toSeconds())
        );

        return count != null && count <= limit;
    }
}
```

---

## 10. 정산 배치 시스템 (Spring Batch)

### 10.1 배치 아키텍처

```
┌─────────────────────────────────────────────────────────┐
│                  Settlement Batch Job                    │
│                                                         │
│  ┌────────────┐    ┌────────────┐    ┌────────────┐    │
│  │   Reader    │───>│ Processor  │───>│   Writer   │    │
│  │            │    │            │    │            │    │
│  │ 전일 완료   │    │ 가맹점별    │    │ Settlement │    │
│  │ 거래 조회   │    │ 집계 계산   │    │ 테이블 저장 │    │
│  │ (Cursor    │    │ (수수료     │    │ + Kafka    │    │
│  │  기반)     │    │  차감)     │    │  이벤트)    │    │
│  └────────────┘    └────────────┘    └────────────┘    │
│                                                         │
│  Chunk Size: 1,000                                      │
│  Skip Policy: 최대 10건 스킵 후 실패 처리                  │
│  Retry Policy: 최대 3회 재시도 (DB 연결 실패 시)            │
└─────────────────────────────────────────────────────────┘
```

### 10.2 배치 스케줄

| 작업 | 실행 시각 | 주기 | 예상 처리량 |
|------|----------|------|-----------|
| 일일 정산 | 매일 03:00 KST | 1일 | 10만~100만 건/일 |
| 정산 실패 재처리 | 매일 06:00 KST | 1일 | 실패 건만 |
| 월간 집계 | 매월 1일 02:00 KST | 1개월 | 전월 데이터 집계 |

### 10.3 성능 목표

| 지표 | 목표 | 측정 방법 |
|------|------|----------|
| 100만 건 처리 시간 | 10분 이내 | Spring Batch 메타데이터 테이블 조회 |
| 메모리 사용량 | 최대 512MB | JVM 힙 메모리 모니터링 |
| 실패율 | 0.01% 이하 | Skip Count / Total Count |

---

## 11. Kafka 이벤트 설계

### 11.1 토픽 설계

| 토픽 | 파티션 수 | 용도 | Key 전략 |
|------|---------|------|---------|
| `transfer-events` | 6 | 송금 이벤트 (완료/실패/취소) | `accountId` (순서 보장) |
| `payment-events` | 6 | 결제 이벤트 (완료/실패/환불) | `merchantId` |
| `notification-events` | 3 | 알림 발송 이벤트 | `userId` |
| `settlement-events` | 3 | 정산 이벤트 (완료/실패) | `merchantId` |

### 11.2 이벤트 스키마

```json
// transfer-events
{
  "eventId": "evt-20260208-000001",
  "eventType": "TRANSFER_COMPLETED",
  "timestamp": "2026-02-08T10:30:00+09:00",
  "data": {
    "transferId": "TRF-20260208-000001",
    "senderId": 1,
    "receiverId": 2,
    "amount": 50000,
    "senderName": "홍길동",
    "receiverName": "김철수"
  }
}
```

### 11.3 Consumer 그룹

| Consumer Group | 구독 토픽 | 역할 |
|---------------|----------|------|
| `notification-group` | `transfer-events`, `payment-events` | SSE/WebSocket 알림 발송 |
| `settlement-group` | `payment-events` | 정산 대상 데이터 적재 |
| `audit-group` | 전체 토픽 | 감사 로그 기록 |

---

## 12. 비기능 요구사항

### 12.1 성능 목표

| 지표 | 목표 | 근거 |
|------|------|------|
| 송금 API TPS | 1,000 TPS 이상 | 동시 1,000명 사용자 송금 시나리오 |
| 송금 API 응답 시간 (P99) | 200ms 이내 | 사용자 체감 성능 기준 |
| 송금 API 응답 시간 (P50) | 50ms 이내 | 평균 사용자 경험 |
| 잔액 정합성 | 100% | 금융 서비스 필수 요건 |
| 시스템 가용성 | 99.9% | 월간 다운타임 43분 이내 |
| 배치 정산 처리량 | 100만 건/10분 | 일일 최대 거래량 기준 |

### 12.2 보안 요구사항

| 항목 | 요구사항 | 구현 방법 |
|------|---------|----------|
| 인증 | JWT 기반 Stateless 인증 | Spring Security + jjwt |
| 민감 정보 암호화 | PIN, 비밀번호 암호화 | BCrypt (비밀번호), AES-256 (PIN) |
| API 접근 제어 | 역할 기반 접근 제어 (RBAC) | `@PreAuthorize`, Role enum |
| SQL Injection 방지 | 파라미터 바인딩 | JPA Named Parameter |
| XSS 방지 | 입력값 검증 | `@Valid`, Custom Validator |
| Rate Limiting | 과도한 요청 차단 | Redis Sliding Window |
| HTTPS | 전송 구간 암호화 | TLS 1.3 (운영 환경) |

### 12.3 관측성 (Observability)

| 영역 | 도구 | 수집 대상 |
|------|------|----------|
| Metrics | Prometheus + Grafana | JVM 메트릭, API 응답 시간, TPS, 에러율, 큐 길이 |
| Logging | Logback + MDC | 요청 ID 기반 추적 로그, 구조화 로깅 (JSON) |
| Tracing | Micrometer Tracing | 분산 추적 (선택: Zipkin/Jaeger 연동) |
| Alerting | Grafana Alert | 에러율 5% 초과, P99 > 500ms, 잔액 불일치 감지 |

### 12.4 Grafana 대시보드 구성 계획

| 대시보드 | 패널 구성 |
|---------|----------|
| **Application Overview** | TPS, 에러율, P50/P95/P99 응답 시간 |
| **Transfer Monitoring** | 송금 성공/실패 비율, 평균 처리 시간, 동시 락 대기 수 |
| **Kafka Monitoring** | Consumer Lag, 토픽별 처리량, 파티션 상태 |
| **Batch Monitoring** | 배치 잡 성공/실패, 처리 건수, 소요 시간 |
| **Infrastructure** | CPU/메모리/디스크, MySQL 커넥션 풀, Redis 메모리 |

---

## 13. 부하 테스트 계획

### 13.1 테스트 도구

**k6 채택 (1차), nGrinder (2차 보완)**

| 비교 항목 | k6 | nGrinder |
|----------|-----|----------|
| 스크립트 언어 | JavaScript | Groovy/Jython |
| 설치 용이성 | 바이너리 단일 파일 | Java 기반, 에이전트 설치 필요 |
| CI/CD 연동 | GitHub Actions 친화 | Jenkins 친화 |
| 분산 부하 | k6 Cloud (유료) 또는 k6-operator | 에이전트 기반 분산 |
| 개인 프로젝트 적합성 | **높음** | 보통 |

### 13.2 테스트 시나리오

#### 시나리오 1: 동시 송금 정합성 테스트

| 항목 | 값 |
|------|-----|
| 목적 | 동시 1,000명이 송금할 때 잔액 정합성 검증 |
| VUser | 1,000 |
| 기간 | 60초 |
| 검증 | 테스트 전후 전체 잔액 합계 일치 여부 |

```javascript
// k6 시나리오 개요
export const options = {
  scenarios: {
    concurrent_transfer: {
      executor: 'constant-vus',
      vus: 1000,
      duration: '60s',
    },
  },
  thresholds: {
    http_req_duration: ['p(99)<200'],
    http_req_failed: ['rate<0.01'],
  },
};
```

#### 시나리오 2: 점진적 부하 증가 (Ramp-up)

| 항목 | 값 |
|------|-----|
| 목적 | TPS 한계점 및 병목 지점 식별 |
| VUser | 10 -> 100 -> 500 -> 1000 -> 2000 (단계적 증가) |
| 기간 | 5분 |
| 측정 | 각 단계별 TPS, P99, 에러율 |

#### 시나리오 3: 동일 계좌 집중 테스트 (Hot Account)

| 항목 | 값 |
|------|-----|
| 목적 | 인기 계좌에 동시 송금 시 분산 락 성능 검증 |
| VUser | 500 (전부 동일 수취 계좌) |
| 기간 | 30초 |
| 측정 | 락 대기 시간, 락 실패율, TPS |

#### 시나리오 4: 멱등성 재시도 테스트

| 항목 | 값 |
|------|-----|
| 목적 | 동일 Idempotency Key 반복 요청 시 정상 동작 검증 |
| VUser | 100 (모두 동일 키) |
| 기간 | 30초 |
| 검증 | 실제 송금은 1건만 발생, 모든 응답 동일 |

### 13.3 병목 분석 체크리스트

| 계층 | 점검 항목 | 도구 |
|------|---------|------|
| Application | 스레드 풀 고갈, GC Pause | JVM Profiler, Grafana JVM 대시보드 |
| DB | Slow Query, Lock Wait, Connection Pool 고갈 | MySQL `SHOW PROCESSLIST`, `EXPLAIN`, HikariCP 메트릭 |
| Redis | 분산 락 경합, 메모리 사용량 | Redis `INFO`, `SLOWLOG` |
| Kafka | Consumer Lag, 파티션 불균형 | Kafka Manager, `kafka-consumer-groups.sh` |
| Network | TCP 커넥션 고갈, 타임아웃 | Prometheus `node_exporter` |

---

## 14. 개발 마일스톤

### 14.1 전체 일정 (8주)

```
Week 1-2: 기반 구축
Week 3-4: 핵심 기능 (송금 + 동시성 제어)
Week 5-6: 확장 기능 (결제 + 정산 + 알림)
Week 7:   부하 테스트 + 최적화
Week 8:   문서화 + 마무리
```

### 14.2 주차별 상세 계획

#### Week 1: 프로젝트 초기 설정

| 작업 | 산출물 |
|------|--------|
| Spring Boot 프로젝트 생성 (Gradle) | `build.gradle` |
| Docker Compose 구성 (MySQL, Redis, Kafka, Prometheus, Grafana) | `docker-compose.yml` |
| 공통 모듈 구현 (예외 처리, 응답 포맷, 로깅) | `common/` 패키지 |
| Spring Security + JWT 인증 | `user/` 모듈 |
| CI/CD 파이프라인 구성 (GitHub Actions) | `.github/workflows/` |

#### Week 2: 계좌 관리 + 기본 API

| 작업 | 산출물 |
|------|--------|
| Account 엔티티 + Repository | JPA 엔티티, 마이그레이션 |
| 계좌 CRUD API | REST API 5개 엔드포인트 |
| 잔액 충전 기능 | Mock PG 연동 |
| 거래 내역 조회 (Cursor Pagination) | Transaction 엔티티 |
| Swagger UI 설정 | Springdoc OpenAPI |

#### Week 3: 송금 핵심 로직

| 작업 | 산출물 | 학습 포인트 |
|------|--------|-----------|
| Redis 분산 락 구현 (Redisson) | `infra/redis/` | **Redisson RLock, MultiLock** |
| 낙관적 락 적용 (`@Version`) | Account 엔티티 | **JPA Optimistic Lock** |
| 멱등성 키 처리 | `transfer/service/` | **Redis + 인터셉터 패턴** |
| 기본 송금 API 구현 | Transfer API | 단위 테스트 필수 |
| 동시성 테스트 작성 | `TransferConcurrencyTest` | `CountDownLatch` 기반 |

#### Week 4: Saga 패턴 + Rate Limiting

| 작업 | 산출물 | 학습 포인트 |
|------|--------|-----------|
| Saga 오케스트레이터 구현 | `transfer/saga/` | **상태 머신, 보상 트랜잭션** |
| 보상 트랜잭션 구현 | 실패 시 자동 롤백 | 의도적 실패 시나리오 테스트 |
| Rate Limiting 구현 | `infra/redis/RateLimiter` | **Sliding Window Counter** |
| 송금 한도 관리 | 1회/1일 한도 체크 | Redis Lua Script |
| 통합 테스트 작성 | 송금 전체 흐름 테스트 | `@SpringBootTest` |

#### Week 5: 결제 + Kafka 이벤트

| 작업 | 산출물 | 학습 포인트 |
|------|--------|-----------|
| 결제 API 구현 | Payment 모듈 | 송금과 동일한 동시성 제어 적용 |
| 결제 취소/환불 | 보상 트랜잭션 재활용 | 상태 머신 확장 |
| Kafka Producer/Consumer 구현 | `infra/kafka/` | **이벤트 발행/구독 패턴** |
| 실시간 알림 (SSE) | `notification/` | **Server-Sent Events** |
| Kafka 이벤트 스키마 정의 | 이벤트 DTO | Avro/JSON Schema 검토 |

#### Week 6: 정산 배치 + 모니터링

| 작업 | 산출물 | 학습 포인트 |
|------|--------|-----------|
| Spring Batch Job 구현 | `settlement/batch/` | **Chunk 기반 배치, Tasklet** |
| 정산 리포트 API | Settlement API | 집계 쿼리 최적화 |
| Skip/Retry 정책 구현 | 배치 안정성 확보 | 실패 재처리 로직 |
| Prometheus 메트릭 설정 | `application.yml` | Micrometer 연동 |
| Grafana 대시보드 구성 | JSON 대시보드 파일 | 5개 대시보드 |

#### Week 7: 부하 테스트 + 최적화

| 작업 | 산출물 | 학습 포인트 |
|------|--------|-----------|
| k6 테스트 스크립트 작성 | `tests/load/` | 4개 시나리오 구현 |
| 부하 테스트 실행 + 결과 분석 | 부하 테스트 리포트 | **병목 식별 및 원인 분석** |
| 성능 최적화 | 쿼리 최적화, 커넥션 풀 튜닝 | `EXPLAIN ANALYZE`, HikariCP |
| 2차 부하 테스트 (최적화 후) | 개선 전후 비교 리포트 | 정량적 개선 효과 측정 |
| 잔액 정합성 최종 검증 | 검증 스크립트 | 전체 잔액 합계 일치 확인 |

#### Week 8: 문서화 + 마무리

| 작업 | 산출물 |
|------|--------|
| README.md 작성 | 프로젝트 소개, 실행 방법, 아키텍처 다이어그램 |
| 기술 의사결정 문서 (ADR) | 각 기술 선택의 근거 및 대안 분석 |
| 부하 테스트 결과 보고서 | 정량적 성능 지표, 병목 분석, 개선 과정 |
| 포트폴리오 정리 | 학습 포인트 및 기술적 도전 과제 정리 |
| 코드 리뷰 + 리팩토링 | 코드 품질 개선 |

---

## 15. 위험 요소 및 기술적 의사결정

### 15.1 위험 요소

| 위험 | 영향도 | 발생 확률 | 완화 전략 |
|------|--------|---------|----------|
| **Redis 단일 장애점 (SPOF)** | 높음 | 중간 | 개발 환경에서는 단일 Redis, 운영 관점에서 Sentinel/Cluster 구성 방안 문서화 |
| **Kafka Consumer 장애 시 이벤트 유실** | 중간 | 낮음 | `enable.auto.commit=false`, 수동 커밋, DLQ (Dead Letter Queue) 구성 |
| **MySQL 커넥션 풀 고갈** | 높음 | 중간 | HikariCP `maximumPoolSize` 튜닝, 커넥션 리크 모니터링 메트릭 추가 |
| **분산 락 장시간 보유** | 중간 | 낮음 | Redisson Watchdog 기본 30초 TTL, `tryLock` 타임아웃 5초 설정 |
| **배치 처리 시간 초과** | 중간 | 낮음 | Chunk Size 튜닝, 파티셔닝 (Step 병렬 실행) |
| **기능 범위 확대 (Scope Creep)** | 높음 | 높음 | Must-Have에만 집중, Nice-to-Have는 별도 브랜치 |

### 15.2 핵심 기술 의사결정 기록 (ADR)

#### ADR-001: 메시지 브로커 선택 (Kafka vs RabbitMQ)

| 기준 | Kafka | RabbitMQ |
|------|-------|----------|
| 메시지 순서 보장 | 파티션 단위 보장 | 큐 단위 보장 |
| 이벤트 재처리 (Replay) | Consumer Offset 조정으로 가능 | 기본적으로 불가 |
| 대용량 처리 | 높은 처리량에 최적화 | 낮은 지연에 최적화 |
| 정산 배치 연계 | 이벤트 소싱 + 배치 처리에 적합 | 가능하나 비효율적 |
| 학습 가치 | **높음 (면접 빈출 주제)** | 보통 |
| **결정** | **Kafka 채택** | - |

**결정 사유**: 금융 도메인에서 이벤트 재처리 가능성, 정산 배치와의 자연스러운 연계, 높은 면접 가치를 고려하여 Kafka를 채택한다.

#### ADR-002: 동시성 제어 방식 (분산 락 + 낙관적 락)

**결정**: Redis 분산 락을 1차 방어선으로, JPA 낙관적 락을 2차 안전장치로 사용한다.

**대안 비교**:

| 방식 | 장점 | 단점 | 채택 여부 |
|------|------|------|----------|
| DB 비관적 락만 사용 | 구현 단순 | 커넥션 보유 시간 증가, TPS 저하 | 미채택 |
| Redis 분산 락만 사용 | 빠른 응답 | Redis 장애 시 정합성 위험 | 미채택 |
| **분산 락 + 낙관적 락** | **다층 방어, 높은 TPS** | **구현 복잡도 증가** | **채택** |

#### ADR-003: Saga 패턴 방식 (Orchestration)

**결정**: 개인 프로젝트에서의 디버깅 용이성과 명시적 흐름 관리를 위해 Orchestration 방식을 채택한다. Choreography 방식은 이벤트 흐름 추적이 어려워 개인 프로젝트에서 학습 효율이 낮다.

#### ADR-004: 모듈러 모노리스 아키텍처

**결정**: 마이크로서비스 대비 운영 복잡도가 낮으면서도, 모듈 간 명확한 경계를 통해 향후 MSA 전환 가능성을 확보한다. 개인 프로젝트에서 인프라 관리 부담을 최소화하고 핵심 비즈니스 로직 구현에 집중한다.

---

## 16. 성공 지표 (Success Metrics)

### 16.1 정량적 지표

| 지표 | 목표 | 측정 시점 |
|------|------|----------|
| 동시 1,000건 송금 잔액 정합성 | **100%** | Week 7 부하 테스트 |
| 송금 API TPS (최적화 후) | **1,000 TPS 이상** | Week 7 부하 테스트 |
| 송금 API P99 응답 시간 | **200ms 이내** | Week 7 부하 테스트 |
| 멱등성 키 중복 방지 성공률 | **100%** | Week 4 통합 테스트 |
| 배치 정산 100만 건 처리 시간 | **10분 이내** | Week 6 배치 테스트 |
| Saga 보상 트랜잭션 성공률 | **100%** | Week 4 실패 시나리오 테스트 |

### 16.2 정성적 지표

| 지표 | 목표 |
|------|------|
| 기술 블로그 포스팅 | 동시성 제어, Saga 패턴, 부하 테스트 각 1편 이상 |
| 면접 대비 | 각 기술 선택의 근거를 Why-What-How로 설명 가능 |
| 코드 품질 | 테스트 커버리지 70% 이상, 핵심 로직 100% |
| 문서화 | README, ADR, 부하 테스트 리포트 완비 |

---

## 부록 A: Docker Compose 서비스 구성

```yaml
services:
  app:
    build: .
    ports:
      - "8080:8080"
    depends_on:
      - mysql
      - redis
      - kafka
    environment:
      SPRING_PROFILES_ACTIVE: local

  mysql:
    image: mysql:8.0
    ports:
      - "3306:3306"
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: easypay
    volumes:
      - mysql_data:/var/lib/mysql

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

  zookeeper:
    image: confluentinc/cp-zookeeper:7.5.0
    ports:
      - "2181:2181"
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181

  kafka:
    image: confluentinc/cp-kafka:7.5.0
    ports:
      - "9092:9092"
    depends_on:
      - zookeeper
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:29092,PLAINTEXT_HOST://localhost:9092
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1

  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./monitoring/prometheus.yml:/etc/prometheus/prometheus.yml

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    depends_on:
      - prometheus

volumes:
  mysql_data:
```

## 부록 B: 서비스 포트 요약

| 서비스 | 포트 | 용도 |
|--------|------|------|
| Spring Boot App | 8080 | REST API |
| MySQL | 3306 | 메인 데이터베이스 |
| Redis | 6379 | 분산 락, 캐시, Rate Limit |
| Kafka Broker | 9092 | 이벤트 메시지 브로커 |
| Zookeeper | 2181 | Kafka 클러스터 관리 |
| Prometheus | 9090 | 메트릭 수집 |
| Grafana | 3000 | 모니터링 대시보드 |
| Swagger UI | 8080/swagger-ui | API 문서 |

---

*본 문서는 개인 프로젝트 PRD로, 실제 서비스 운영을 목적으로 하지 않습니다. 핵심 학습 목표는 동시성 제어, 분산 트랜잭션, 대용량 배치 처리의 설계 및 구현 경험 확보입니다.*
