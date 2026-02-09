# EasyPay 프론트엔드 개발 계획서

**버전**: 1.0
**작성일**: 2026-02-08
**기술 스택**: Next.js (App Router)

---

## 1. 기술 스택

| 카테고리 | 기술 | 선택 사유 |
|---------|------|----------|
| **Framework** | Next.js 15 (App Router) | SSR/RSC 지원, 파일 기반 라우팅 |
| **Language** | TypeScript | 타입 안전성, API 응답 타입 공유 |
| **상태 관리** | Zustand | 경량, 보일러플레이트 최소화 |
| **서버 상태** | TanStack Query (React Query) | 캐싱, 자동 재요청, 낙관적 업데이트 |
| **스타일링** | Tailwind CSS + shadcn/ui | 빠른 UI 구성, 일관된 디자인 시스템 |
| **폼 관리** | React Hook Form + Zod | 유효성 검증, 타입 추론 |
| **HTTP 클라이언트** | Axios | 인터셉터 기반 토큰 관리 |
| **실시간 통신** | EventSource (SSE) | 서버 알림 수신 |
| **차트** | Recharts | 정산 대시보드 시각화 |
| **패키지 매니저** | pnpm | 디스크 효율, 빠른 설치 |

---

## 2. 디렉토리 구조

```
frontend/
├── public/
├── src/
│   ├── app/                        # App Router 페이지
│   │   ├── (auth)/                 # 비인증 레이아웃 그룹
│   │   │   ├── login/
│   │   │   │   └── page.tsx
│   │   │   ├── signup/
│   │   │   │   └── page.tsx
│   │   │   └── layout.tsx
│   │   ├── (main)/                 # 인증 필요 레이아웃 그룹
│   │   │   ├── dashboard/
│   │   │   │   └── page.tsx
│   │   │   ├── accounts/
│   │   │   │   ├── [accountId]/
│   │   │   │   │   └── page.tsx
│   │   │   │   └── page.tsx
│   │   │   ├── transfer/
│   │   │   │   ├── page.tsx        # 송금 폼
│   │   │   │   └── [transferId]/
│   │   │   │       └── page.tsx    # 송금 상태 조회
│   │   │   ├── payment/
│   │   │   │   ├── page.tsx
│   │   │   │   └── [paymentId]/
│   │   │   │       └── page.tsx
│   │   │   ├── transactions/
│   │   │   │   └── page.tsx        # 거래 내역
│   │   │   ├── settlement/         # 가맹점 정산
│   │   │   │   └── page.tsx
│   │   │   ├── admin/              # 관리자
│   │   │   │   └── page.tsx
│   │   │   └── layout.tsx
│   │   ├── layout.tsx              # 루트 레이아웃
│   │   └── not-found.tsx
│   ├── components/
│   │   ├── ui/                     # shadcn/ui 컴포넌트
│   │   ├── layout/
│   │   │   ├── header.tsx
│   │   │   ├── sidebar.tsx
│   │   │   └── bottom-nav.tsx
│   │   ├── account/
│   │   │   ├── balance-card.tsx
│   │   │   └── account-list.tsx
│   │   ├── transfer/
│   │   │   ├── transfer-form.tsx
│   │   │   ├── pin-modal.tsx
│   │   │   └── transfer-result.tsx
│   │   ├── payment/
│   │   │   ├── payment-form.tsx
│   │   │   └── payment-status.tsx
│   │   └── transaction/
│   │       ├── transaction-list.tsx
│   │       └── transaction-item.tsx
│   ├── lib/
│   │   ├── api/
│   │   │   ├── client.ts           # Axios 인스턴스 + 인터셉터
│   │   │   ├── auth.ts
│   │   │   ├── account.ts
│   │   │   ├── transfer.ts
│   │   │   ├── payment.ts
│   │   │   └── settlement.ts
│   │   ├── hooks/
│   │   │   ├── use-auth.ts
│   │   │   ├── use-accounts.ts
│   │   │   ├── use-transfer.ts
│   │   │   ├── use-notifications.ts  # SSE 연결
│   │   │   └── use-pin.ts
│   │   ├── store/
│   │   │   ├── auth-store.ts
│   │   │   └── notification-store.ts
│   │   ├── types/
│   │   │   ├── api.ts              # 공통 응답 타입
│   │   │   ├── account.ts
│   │   │   ├── transfer.ts
│   │   │   ├── payment.ts
│   │   │   └── settlement.ts
│   │   └── utils/
│   │       ├── format.ts           # 금액 포맷팅
│   │       └── idempotency.ts      # 멱등성 키 생성
│   └── middleware.ts               # 인증 미들웨어
├── .env.local
├── next.config.ts
├── tailwind.config.ts
├── tsconfig.json
└── package.json
```

---

## 3. 페이지별 기능 명세

### 3.1 인증

| 페이지 | 경로 | 기능 |
|--------|------|------|
| 로그인 | `/login` | 이메일/비밀번호 로그인, JWT 저장 |
| 회원가입 | `/signup` | 이메일, 비밀번호, 이름, PIN 설정 |

### 3.2 메인 (인증 필요)

| 페이지 | 경로 | 기능 |
|--------|------|------|
| 대시보드 | `/dashboard` | 잔액 요약, 최근 거래 내역, 빠른 송금 |
| 계좌 목록 | `/accounts` | 보유 계좌 목록, 계좌 생성 |
| 계좌 상세 | `/accounts/[id]` | 잔액, 충전, 거래 내역 (커서 페이징) |
| 송금 | `/transfer` | 송금 폼 (수취인, 금액, 메모) → PIN 인증 → 결과 |
| 송금 상태 | `/transfer/[id]` | 송금 상태 추적 (PENDING → COMPLETED) |
| 결제 | `/payment` | 가맹점 결제 요청 |
| 결제 상태 | `/payment/[id]` | 결제 상태, 취소/환불 |
| 거래 내역 | `/transactions` | 전체 거래 내역, 필터링 |
| 정산 | `/settlement` | 가맹점 정산 내역, 차트 |
| 관리자 | `/admin` | 수동 정산 실행, 시스템 모니터링 |

---

## 4. 핵심 구현 사항

### 4.1 JWT 인증 흐름

```
1. 로그인 → Access Token (메모리) + Refresh Token (httpOnly cookie)
2. API 요청 → Axios interceptor가 Authorization 헤더 자동 첨부
3. 401 응답 → Refresh Token으로 자동 갱신 → 원래 요청 재시도
4. Refresh 실패 → 로그인 페이지 리다이렉트
```

- `middleware.ts`에서 쿠키 기반으로 인증 여부 확인, 미인증 시 `/login` 리다이렉트

### 4.2 PIN 인증

```
송금/결제 요청 → PIN 모달 표시 → 6자리 입력 → POST /api/v1/auth/pin/verify
→ 성공 시 Redis 세션에 인증 상태 저장 (서버) → 송금/결제 API 호출
```

- `pin-modal.tsx`: 숫자 6자리 입력 UI, 보안 키패드 (선택)
- PIN 인증 상태는 서버 측 Redis 세션으로 관리 (일정 시간 유효)

### 4.3 멱등성 키 관리

```typescript
// lib/utils/idempotency.ts
export function generateIdempotencyKey(): string {
  return crypto.randomUUID();
}
```

- 송금/결제 요청 시 `Idempotency-Key` 헤더에 UUID 첨부
- 폼 제출 후 버튼 비활성화로 중복 클릭 방지
- 네트워크 오류 시 동일 키로 재시도

### 4.4 실시간 알림 (SSE)

```typescript
// lib/hooks/use-notifications.ts
// EventSource로 서버의 SSE 엔드포인트에 연결
// 송금 수신, 결제 완료, 정산 완료 알림을 실시간으로 수신
// Zustand 스토어에 알림 저장 → 토스트 UI로 표시
```

### 4.5 커서 기반 페이지네이션

```
GET /api/v1/accounts/{id}/transactions?cursor={lastId}&size=20
```

- TanStack Query의 `useInfiniteQuery` 활용
- 스크롤 기반 무한 로딩 또는 "더보기" 버튼

---

## 5. API 클라이언트 설정

```typescript
// lib/api/client.ts
// Axios 인스턴스 생성
// - baseURL: process.env.NEXT_PUBLIC_API_URL
// - request interceptor: Authorization 헤더 첨부
// - response interceptor: 401 시 토큰 갱신 → 재시도
// - 공통 에러 핸들링: ApiResponse<T> 타입으로 래핑
```

### 공통 타입

```typescript
// lib/types/api.ts
interface ApiResponse<T> {
  success: boolean;
  data: T | null;
  error: { code: string; message: string } | null;
  timestamp: string;
}
```

---

## 6. 주요 화면 흐름

### 6.1 송금 플로우

```
대시보드 → [송금하기] 클릭
→ /transfer 페이지
→ 수취 계좌 입력 + 금액 입력 + 메모 입력
→ [송금] 버튼 클릭
→ PIN 모달 표시 → 6자리 입력
→ PIN 검증 API 호출
→ 멱등성 키 생성 + 송금 API 호출
→ 결과 화면 표시 (성공/실패)
→ [확인] → 대시보드 복귀
```

### 6.2 거래 내역 조회 플로우

```
계좌 상세 페이지
→ 최근 거래 20건 표시 (useInfiniteQuery)
→ 스크롤 하단 → 다음 20건 자동 로드 (cursor 기반)
→ 거래 항목 클릭 → 상세 정보 바텀시트/모달
```

---

## 7. 환경 설정

### 7.1 환경 변수

```env
NEXT_PUBLIC_API_URL=http://localhost:8080/api/v1
```

### 7.2 프록시 설정

```typescript
// next.config.ts
// 개발 환경: rewrites()로 /api/v1/** → localhost:8080 프록시
// CORS 이슈 방지
```

---

## 8. 개발 마일스톤

백엔드 개발 일정과 병행하여 진행한다.

| 주차 | 작업 | 연동 백엔드 (PRD 기준) |
|------|------|----------------------|
| **Week 1** | 프로젝트 셋업 (Next.js, Tailwind, shadcn/ui), 레이아웃, 라우팅 구성 | Week 1: Spring Boot 초기 설정 |
| **Week 2** | 로그인/회원가입 페이지, JWT 인증 흐름, 계좌 목록/상세/충전 페이지 | Week 2: Auth + Account API |
| **Week 3** | 송금 폼, PIN 모달, 멱등성 키 처리, 송금 결과 화면 | Week 3: Transfer API |
| **Week 4** | 거래 내역 (커서 페이징), 송금 취소, 에러 핸들링 고도화 | Week 4: Saga + Rate Limiting |
| **Week 5** | 결제 페이지, SSE 실시간 알림, 토스트 알림 UI | Week 5: Payment + Kafka |
| **Week 6** | 정산 대시보드 (Recharts), 관리자 페이지 | Week 6: Batch + Monitoring |
| **Week 7** | 전체 통합 테스트, UI/UX 개선 | Week 7: 부하 테스트 |
| **Week 8** | 반응형 대응, 최종 점검 | Week 8: 문서화 |

---

## 9. 품질 기준

| 항목 | 기준 |
|------|------|
| TypeScript strict mode | 활성화 |
| ESLint + Prettier | 코드 일관성 유지 |
| 에러 바운더리 | 페이지 단위 `error.tsx` 처리 |
| 로딩 상태 | 페이지 단위 `loading.tsx` + Skeleton UI |
| 반응형 | 모바일 우선 (min-width: 375px) |

---

*본 문서는 PRD(백엔드)와 병행하는 프론트엔드 개발 계획으로, 백엔드 API 완성 시점에 맞춰 페이지별 연동을 진행한다.*
