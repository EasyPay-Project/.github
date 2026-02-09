# 다음 할 일 (TODO)

**작성일**: 2026-02-09
**기준**: 백엔드 `048eb94`, 프론트엔드 `d33ef04` 커밋 기준

---

## 현재 상태 요약

| 영역 | 완성도 | 테스트 | 핵심 미비 사항 |
|------|:---:|:---:|------------|
| **백엔드** | 100% | 48개 메서드, 부하 4개 시나리오 | — |
| **프론트엔드** | 100% | 단위 29개, E2E 4개 | — |
| **연동** | ~99% | — | SSE 브라우저 수동 검증 (optional) |

---

## ~~백엔드 다음 단계~~ ✅ BE-1~3 완료

### ~~BE-1. SettlementService 추출~~ ✅ `fee6042`

- `settlement/service/SettlementService.java` 생성
- Controller → Service → Repository 구조로 분리
- getSettlements, getSettlementsByDate, runSettlement 메서드 분리

### ~~BE-2. Notification DTO 정식화~~ ✅ `fee6042`

- `notification/dto/NotificationEvent.java` — Kafka 수신 이벤트 타입 안전 매핑
- `notification/dto/NotificationResponse.java` — SSE 클라이언트 전송 데이터
- NotificationConsumer의 Map 기반 → DTO 기반으로 전면 교체

### ~~BE-3. Settlement / Notification 테스트 작성~~ ✅ `fee6042`

- `SettlementServiceTest` (4개) — 가맹점별 조회, 날짜별 조회, 빈 결과, 전체 필드 검증
- `NotificationServiceTest` (4개) — SSE 구독/교체, 미구독 전송, 정상 전송
- `NotificationConsumerTest` (4개) — 송금 양측 알림, 결제 알림, 잘못된 JSON 처리

---

### ~~BE-4. 부하 테스트 실행 및 결과 분석~~ ✅ 2026-02-09

k6 스크립트 작성 완료(`8cc1a85`). Docker 컨테이너화 완료(`e103275`).

**실행 결과:**
- 4개 시나리오 ALL PASS (동시 송금, 점진적 부하, Hot Account, 멱등성)
- p95 레이턴시 < 10ms (7~9.5ms 범위)
- 성공률 99.75~100%
- Rate Limit 외부화, k6 계좌 소유권 매핑 수정, 멱등성 memo 수정 완료
- `docs/LOAD_TEST_REPORT.md` 작성 완료

---

## ~~프론트엔드 다음 단계~~ ✅ 완료

### ~~FE-1. SSE 알림 앱 연결~~ ✅ `deba32e`

- `NotificationListener` 클라이언트 컴포넌트 생성 → `(main)/layout.tsx`에 삽입
- accessToken 존재 시 자동 SSE 연결, 송금/결제 알림 토스트 + Header 배지 unreadCount 반영

### ~~FE-2. 에러 바운더리 추가~~ ✅ `deba32e`

- `src/app/(main)/error.tsx` — 전역 에러 폴백 (에러 메시지 + 재시도 버튼)
- `src/app/(main)/loading.tsx` — 페이지 전환 스피너 (Loader2 아이콘)

---

## 연동 검증 (양쪽 작업 완료 후)

### INT-1. End-to-End 통합 검증 [🟡 중간]

#### 사전 조건

1. Docker 엔진 실행 중
2. ~~포트 충돌 해결~~ ✅ Next.js 기본 포트가 3001로 변경되어 Grafana(3000)와 충돌 없음

#### 환경 기동

```bash
# 1. 인프라 + 백엔드 일괄 기동 (Docker 컨테이너화 완료)
cd backend && docker-compose up -d

# 2. 프론트엔드 개발 서버 (기본 포트 3001)
cd frontend && pnpm dev
```

#### 자동화된 테스트 (Playwright E2E)

기존 E2E 테스트가 시나리오 1~3을 부분 커버합니다:

| E2E 스펙 | 커버 시나리오 | 실행 |
|----------|------------|------|
| `e2e/auth.spec.ts` | 시나리오 1 (회원가입→로그인→대시보드) | `pnpm test:e2e` |
| `e2e/account.spec.ts` | 시나리오 2 (계좌 생성→충전→잔액) | `pnpm test:e2e` |
| `e2e/transfer.spec.ts` | 시나리오 3 일부 (미인증 리다이렉트) | `pnpm test:e2e` |

#### 수동 검증 시나리오

| # | 시나리오 | 검증 절차 | 기대 결과 |
|:-:|---------|----------|----------|
| 1 | 회원가입→로그인 | POST `/api/v1/auth/signup` → POST `/api/v1/auth/login` → 프론트엔드 대시보드 접근 | JWT 발급, 사용자 이름 표시 |
| 2 | 계좌 생성→충전 | POST `/api/v1/accounts` → POST `/api/v1/accounts/{id}/deposit` → GET `/api/v1/accounts/{id}/balance` | 잔액 = 충전 금액 |
| 3 | 송금 | POST `/api/v1/transfers` (Idempotency-Key, PIN) → GET `/api/v1/transfers/{id}` | 상태 COMPLETED, 양쪽 잔액 정확 |
| 4 | 결제→환불 | POST `/api/v1/payments` → POST `/api/v1/payments/{id}/refund` → 잔액 확인 | 잔액 복원 |
| 5 | SSE 알림 | 브라우저 2개 열어 송금 수행 → 수신자 토스트 확인 | 송금 수신 알림 토스트 표시 |
| 6 | 수동 정산 | POST `/api/v1/settlements/run` → GET `/api/v1/settlements` → 프론트엔드 정산 대시보드 | 정산 내역 반영 |

#### 정리

```bash
cd backend && docker-compose down -v
```

#### 검증 결과 (2026-02-09)

| # | 시나리오 | API 검증 | E2E 검증 | 비고 |
|:-:|---------|:---:|:---:|------|
| 1 | 회원가입→로그인 | ✅ | ✅ auth.spec.ts | JWT 발급, 사용자 이름 정상 |
| 2 | 계좌 생성→충전→잔액 | ✅ | ✅ account.spec.ts | API 정상, E2E waitForFunction 수정으로 PASS |
| 3 | 송금 + 멱등성 | ✅ | ✅ transfer.spec.ts | COMPLETED, 양쪽 잔액 정확, 멱등성 캐시 동작 |
| 4 | 결제→환불→잔액 복원 | ✅ | — | 결제 30,000원 → 환불 → 잔액 950,000원 복원 |
| 5 | SSE 알림 | ✅ | ✅ sse-notification.spec.ts | 연결OK, Kafka발행OK, 토스트 표시 확인 (userId 버그 수정 `048eb94`) |
| 6 | 수동 정산 | ✅ | — | 배치 실행 성공, 당일 결제는 전일 기준으로 정산 미대상 (정상) |

**단위 테스트**: 백엔드 48개 PASS, 프론트엔드 29개 PASS
**E2E 테스트**: 4/4 PASS

**잔여 이슈**: 없음 ✅

---

## 작업 순서 권장

~~모든 핵심 작업 완료~~ ✅

```
✅ [백엔드] BE-1~4 완료 (SettlementService, Notification DTO, 테스트, 부하 테스트)
✅ [프론트엔드] FE-1~2 완료 (SSE 알림, 에러 바운더리)
✅ [연동] INT-1 완료 (API 6개 시나리오, E2E 4개, SSE 브라우저 검증 포함)

모든 작업 완료. 잔여 이슈 없음.
```
