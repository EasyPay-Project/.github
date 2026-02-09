# 프론트엔드 개발 진행사항

**최종 갱신**: 2026-02-09 (4차)
**기술 스택**: Next.js 16 / React 19 / TypeScript / pnpm
**최신 커밋**: `d33ef04` SSE 실시간 알림 E2E 테스트 추가

---

## 커밋 이력

| 커밋 | 날짜 | 내용 |
|------|------|------|
| `d33ef04` | 2026-02-09 | test(e2e): SSE 실시간 알림 E2E 테스트 추가 |
| `8013d79` | 2026-02-09 | fix: 프론트엔드 기본 포트를 3001로 변경하여 Grafana 충돌 해소 |
| `01eddb5` | 2026-02-09 | fix: auth-store persist 미들웨어 추가 및 E2E 테스트 안정성 개선 |
| `deba32e` | 2026-02-09 | feat: SSE 알림 레이아웃 연결 및 에러 바운더리/로딩 UI 추가 |
| `94545e0` | 2026-02-09 | docs: CLAUDE.md 최신 프로젝트 상태 반영 |
| `12e752d` | 2026-02-09 | docs: README.md 프로젝트 문서 재작성 및 CLAUDE.md 작업 범위 가이드 추가 |
| `e6934f1` | 2026-02-09 | 에러 핸들링, SSE 알림, E2E/단위 테스트 추가 |
| `e444df0` | 2026-02-09 | 백엔드 API 정합성 맞춤 (타입/API/훅) |
| `c6d6c8a` | 2026-02-09 | chore: .gitignore 업데이트 |
| `ede2ee2` | 2026-02-09 | feat: 대시보드 페이지 구현 |
| `5ad02b4` | 2026-02-09 | feat: 정산 대시보드 및 관리자 페이지 구현 |
| `5d17367` | 2026-02-09 | feat: 거래 내역 페이지 및 TransactionList 구현 |
| `80a2524` | 2026-02-09 | feat: 결제 페이지 및 컴포넌트 구현 |
| `6136342` | 2026-02-09 | feat: 송금 및 송금 상태 페이지 구현 |
| `fb36a35` | 2026-02-09 | feat: 계좌 목록 및 상세 페이지 구현 |
| `c472ee3` | 2026-02-09 | fix: 기존 코드 lint 에러 수정 |
| `20c372e` | 2026-02-08 | chore: 프로젝트 초기 설정 및 기반 구조 구성 |
| `2a247db` | 2026-02-08 | Initial commit from Create Next App |

---

## 페이지 구현 현황

### (auth) 그룹 — 비인증 레이아웃

| 경로 | 페이지 | 상태 |
|------|--------|:---:|
| `/login` | 로그인 | ✅ |
| `/signup` | 회원가입 | ✅ |

### (main) 그룹 — 인증 필요 레이아웃

| 경로 | 페이지 | 상태 |
|------|--------|:---:|
| `/dashboard` | 대시보드 | ✅ |
| `/accounts` | 계좌 목록 | ✅ |
| `/accounts/[accountId]` | 계좌 상세 | ✅ |
| `/transfer` | 송금 | ✅ |
| `/transfer/[transferId]` | 송금 상태 | ✅ |
| `/payment` | 결제 | ✅ |
| `/payment/[paymentId]` | 결제 상태 | ✅ |
| `/transactions` | 거래 내역 | ✅ |
| `/settlement` | 정산 | ✅ |
| `/admin` | 관리자 | ✅ |

---

## 컴포넌트 — 전체 ✅ 완료

Layout: header, sidebar, bottom-nav. Account: account-list, balance-card. Transfer: transfer-form, transfer-result, pin-modal. Payment: payment-form, payment-status. Transaction: transaction-list, transaction-item. shadcn/ui 10개.

---

## API 레이어 — 전체 ✅ 완료

client.ts (인터셉터, 401 토큰 갱신, 전역 에러 토스트), auth.ts, account.ts, transfer.ts, payment.ts, settlement.ts.

---

## Custom Hooks — 전체 ✅ 완료

use-auth, use-accounts, use-transfer, use-pin, use-payment, use-settlement, use-notifications (SSE).

---

## 상태 관리 — 전체 ✅ 완료

auth-store (accessToken, refreshToken, user), notification-store (notifications, unreadCount).

---

## 유틸리티

| 파일 | 상태 | 구현 내용 |
|------|:---:|----------|
| `utils.ts` | ✅ | cn() — clsx + tailwind-merge |
| `format.ts` | ✅ | formatCurrency, formatDate, maskAccountNumber |
| `idempotency.ts` | ✅ | generateIdempotencyKey |
| `jwt.ts` | ✅ | **신규** — JWT 디코드 (sub, email, name, role 추출) |
| `error.ts` | ✅ | **신규** — 백엔드 ErrorCode → 한국어 메시지 매핑 (38+ 코드), extractErrorMessage/extractErrorCode |

---

## 테스트

### 단위 테스트 (Vitest + React Testing Library + jsdom) — ✅ 구축 완료

| 파일 | 테스트 수 | 영역 |
|------|:---:|------|
| `jwt.test.ts` | 1 | JWT 디코드 검증 |
| `error.test.ts` | 8 | 에러 메시지 추출, 코드 매핑 |
| `format.test.ts` | 4 | 금액 포맷, 날짜 포맷, 계좌번호 마스킹 |
| `auth-store.test.ts` | 4 | 로그인, 로그아웃, 토큰 설정 |
| `notification-store.test.ts` | 6+ | 알림 추가, 읽음 처리, 전체 삭제 |
| **합계** | **~29** | |

```bash
pnpm test             # 전체 실행
pnpm test:watch       # watch 모드
```

### E2E 테스트 (Playwright + Chromium) — ✅ 구축 완료

| 파일 | 영역 |
|------|------|
| `e2e/auth.spec.ts` | 회원가입 → 로그인 → 대시보드 |
| `e2e/account.spec.ts` | 계좌 생성 → 충전 → 잔액 확인 |
| `e2e/sse-notification.spec.ts` | SSE 실시간 알림 (송금 토스트 확인) |
| `e2e/transfer.spec.ts` | 미인증 리다이렉트 |

```bash
pnpm test:e2e         # 전체 실행
pnpm test:e2e:ui      # UI 모드
```

---

## 인증 흐름 (업데이트)

1. 로그인 → 백엔드가 `{ accessToken, refreshToken }` 반환
2. `jwt.ts`로 accessToken에서 사용자 정보 디코드 (sub, email, name, role)
3. Zustand에 accessToken + refreshToken + user 저장
4. refreshToken은 브라우저 쿠키에도 저장 (미들웨어 호환)
5. 401 시 refreshToken body 전송으로 갱신 → 쿠키 동기화 → 재시도

---

## SSE 알림

- `use-notifications.ts`: EventSource `/notifications/subscribe?token=` 연결
- Named events: `transfer`, `payment`, `connect`
- Sonner 토스트로 알림 표시
- ✅ `NotificationListener` 클라이언트 컴포넌트가 `(main)/layout.tsx`에 삽입 완료 (`deba32e`)
- accessToken 존재 시 자동 SSE 연결, 송금/결제 알림 토스트 + Header 배지 unreadCount 반영

---

## ~~미완료 항목~~ ✅ 전체 완료

모든 프론트엔드 기능 구현 완료. (`deba32e`)

---

## 백엔드 연동 정합성

| 항목 | 상태 | 비고 |
|------|:---:|------|
| API 엔드포인트 매핑 | ✅ | e444df0 커밋에서 정합성 맞춤 |
| ApiResponse\<T\> 래퍼 | ✅ | 양쪽 동일 구조 |
| 멱등성 키 헤더 | ✅ | 프론트 자동 생성 → 백엔드 검증 |
| 상태 enum | ✅ | Transfer/Payment/Settlement 일치 |
| 커서 페이지네이션 | ✅ | TransactionPage (nextCursor: number) |
| CORS | ✅ | 백엔드 설정 완료 |
| JWT name 클레임 | ✅ | 백엔드 추가 → 프론트 디코드 연동 |
| SSE 토큰 인증 | ✅ | 쿼리 파라미터 방식 양쪽 일치 |
| 에러 코드 매핑 | ✅ | 백엔드 ErrorCode 38+ → 프론트 한국어 매핑 |
| SSE 알림 | ✅ | 양쪽 구현 완료, NotificationListener로 앱 연결 완료 |
