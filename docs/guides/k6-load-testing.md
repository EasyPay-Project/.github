# k6 부하 테스트

## 핵심 개념

### k6
Grafana에서 개발한 오픈소스 부하 테스트 도구. JavaScript로 테스트 시나리오를 작성하고, VU(Virtual User) 단위로 동시 사용자를 시뮬레이션한다. Go로 작성되어 높은 성능을 제공한다.

### 주요 용어
- **VU (Virtual User)**: 동시 접속 사용자 한 명을 시뮬레이션하는 단위
- **Iteration**: 한 VU가 default function을 한 번 실행하는 것
- **Threshold**: SLA 기준. 미달 시 테스트 실패로 판정
- **Executor**: VU 배분 전략 (constant-vus, ramping-vus 등)

### EasyPay에서의 역할
송금 시스템의 **동시성 제어**, **멱등성**, **성능 SLA**를 검증하는 7가지 시나리오를 제공한다.

## 프로젝트 적용

### 테스트 데이터 구성

```
USER_POOL: 10명 (user1~user10@easypay.com)
ACCOUNT_POOL: 20개 (사용자당 2개 계좌)
계좌당 잔액: 10,000,000원
```

**계좌 소유자 매칭**: VU별로 올바른 사용자의 계좌를 사용하도록 보장한다.

```javascript
// config.js:102-116
// userN → 계좌 2N-1, 2N (소유자 일치 보장)
export function getAccountPair() {
    const userIdx = (__VU - 1) % USER_POOL.length; // 0-9
    const myAccounts = [ACCOUNT_POOL[userIdx * 2], ACCOUNT_POOL[userIdx * 2 + 1]];
    const sender = myAccounts[Math.floor(Math.random() * 2)];
    let receiver;
    do {
        receiver = randomElement(ACCOUNT_POOL);
    } while (receiver === sender || myAccounts.includes(receiver));
    return { sender, receiver };
}
```

### SLA 기준

```javascript
// config.js:26-32
export const SLA_THRESHOLDS = {
    http_req_duration: ['p(95)<300', 'p(99)<1000'],  // p95 < 300ms, p99 < 1s
    http_req_failed: ['rate<0.001'],                   // 에러율 < 0.1%
    http_reqs: ['rate>100'],                           // 초당 100건 이상
};
```

### 7가지 테스트 시나리오

#### 1. Concurrent Transfer (동시 송금)
**목적**: 분산 락 + 낙관적 락이 동시 송금을 정확히 처리하는지 검증

```
설정: 300 VU, 5분 (로컬: 50 VU, 1분)
동작: 각 VU가 자신의 계좌에서 랜덤 계좌로 1,000~50,000원 송금
커스텀 메트릭: transfer_success_rate, transfer_duration
기대: 성공률 > 90%, p95 < 500ms
```

#### 2. Stress Test (점진적 부하)
**목적**: 시스템이 점진적으로 증가하는 부하를 견디는지 검증

```
설정: ramping-vus (50→200→500→500→200→50), 15분
      (로컬: 0→10→30→50→50→20→0, 3분)
동작: 혼합 워크로드 (송금 40%, 조회 40%, 입금 20%)
커스텀 메트릭: stress_success_rate, stress_tx_duration
기대: 성공률 > 95%, p95 < 500ms
```

#### 3. Idempotency Retry (멱등성 재시도)
**목적**: 동일 Idempotency-Key로 5회 재시도 시 동일 응답 반환 확인

```
설정: 100 VU, 5분 (로컬: 20 VU, 1분)
동작: 한 번 송금 → 같은 키로 4회 더 재시도
      첫 응답과 이후 응답의 transferId 비교
커스텀 메트릭: idempotency_consistency_rate
기대: 일관성 > 99.9%
```

#### 4. Hot Account (핫 계좌)
**목적**: 특정 인기 계좌에 트래픽이 집중될 때의 성능 검증

```
설정: 200 VU, 5분 (로컬: 30 VU, 1분)
동작: 모든 VU가 하나의 수취 계좌로 송금 (락 경합 극대화)
커스텀 메트릭: hot_success_rate, hot_duration
기대: 성공률 > 85%, p95 < 800ms
```

#### 5. Mixed Workload (혼합 워크로드)
**목적**: 실제 트래픽 패턴을 시뮬레이션

```
설정: 300 VU, 10분
동작: 조회 40%, 거래내역 20%, 송금 20%, 결제 10%, 입출금 10%
기대: 전체 에러율 < 1%
```

#### 6. Soak Test (내구성)
**목적**: 장시간 부하 시 메모리 누수, 성능 저하 감지

```
설정: 200 VU, 30분
동작: 지속적인 송금 요청
기대: 30분간 성능 유지, GC 이상 없음
```

#### 7. Spike Test (급증)
**목적**: 갑작스러운 트래픽 급증 시 시스템 회복력 검증

```
설정: 50 → 1000 VU (10초 내 급증), 3분 유지
동작: 순간 20배 부하 증가
기대: 시스템 크래시 없이 점진적 회복
```

### 로컬 시나리오

본 시나리오(CI/CD용)의 축소 버전. `k6/scenarios/local/` 디렉토리에 위치.

```javascript
// concurrent-transfer-local.js
export { setup, default as default, teardown } from '../concurrent-transfer.js';

export const options = {
    scenarios: {
        concurrent_transfer: {
            executor: 'constant-vus',
            vus: 50,              // 300 → 50 축소
            duration: '1m',       // 5min → 1min 축소
        },
    },
    thresholds: {
        http_req_duration: ['p(95)<500', 'p(99)<1500'],  // SLA 완화
        transfer_success_rate: ['rate>0.90'],
    },
};
```

## 코드 위치 및 구조

```
backend/k6/
├── common/
│   └── config.js                  # 공통 설정 (USER_POOL, SLA, 헬퍼 함수)
├── setup-test-data.sh             # 테스트 데이터 자동 프로비저닝
├── scenarios/
│   ├── concurrent-transfer.js     # 동시 송금 (본 시나리오)
│   ├── stress-test.js             # 점진적 부하
│   ├── idempotency-retry.js       # 멱등성 재시도
│   ├── hot-account.js             # 핫 계좌
│   ├── mixed-workload.js          # 혼합 워크로드
│   ├── soak-test.js               # 내구성
│   ├── spike-test.js              # 급증
│   └── local/                     # 로컬 축소 버전
│       ├── concurrent-transfer-local.js
│       ├── stress-test-local.js
│       ├── hot-account-local.js
│       ├── idempotency-local.js
│       └── idempotency-retry-local.js
```

## 설정

### 테스트 데이터 셋업

```bash
# 1. 인프라 + 앱 실행
docker-compose up -d

# 2. 테스트 데이터 생성 (10명 사용자, 20개 계좌, 각 1천만원)
bash k6/setup-test-data.sh
```

`setup-test-data.sh`의 3단계:
1. **사용자 생성/로그인**: user1~10 회원가입 → 토큰 획득
2. **계좌 생성**: 사용자당 2개 계좌 생성 (dailyLimit: 50,000,000원)
3. **잔액 충전**: 계좌당 10,000,000원 입금

### k6 실행 방법

```bash
# Docker 네트워크를 통한 실행 (Docker 환경)
docker run --rm --network backend_default \
  -v "$(pwd)/k6:/scripts" grafana/k6:latest \
  run -e BASE_URL=http://easypay-app:8080 \
  /scripts/scenarios/local/concurrent-transfer-local.js

# 로컬 k6 설치 시 (로컬 환경)
k6 run -e BASE_URL=http://localhost:8080 \
  k6/scenarios/local/concurrent-transfer-local.js
```

## 동작 흐름

```
[1] setup() 함수 (1회 실행)
    └─ loginMultipleUsers()
       └─ 10명 사용자 로그인 → 토큰 배열 반환

[2] default() 함수 (각 VU가 반복 실행)
    └─ VU 1:  token = tokens[0] → getAccountPair() → 송금 요청
    └─ VU 2:  token = tokens[1] → getAccountPair() → 송금 요청
    └─ ...
    └─ VU 50: token = tokens[9] → getAccountPair() → 송금 요청

[3] 각 Iteration
    ├─ UUID 멱등성 키 생성
    ├─ POST /api/v1/transfers
    │   Headers: Authorization, Idempotency-Key
    │   Body: {senderAccountId, receiverAccountId, amount}
    ├─ 응답 검증 (status 201 or 200)
    ├─ 커스텀 메트릭 기록
    │   ├─ transfer_success_rate.add(1)
    │   └─ transfer_duration.add(duration)
    └─ sleep(0.1 ~ 0.5초)

[4] teardown() 함수 (1회 실행)
    └─ 결과 요약 출력

[5] 결과 출력
    ├─ http_req_duration .... p(95)=245ms  ✓ < 300ms
    ├─ http_req_failed ...... 0.03%        ✓ < 0.1%
    ├─ transfer_success_rate  97.5%        ✓ > 90%
    └─ http_reqs ............ 523/s        ✓ > 100/s
```

## 트러블슈팅 & 주의사항

### 계좌 소유자 매칭
초기 구현에서 VU가 다른 사용자의 계좌로 송금을 시도하여 403 에러가 발생했다. `getAccountPair()` 함수에서 `__VU` (VU 인덱스)를 기반으로 올바른 소유자의 계좌만 sender로 사용하도록 수정했다.

### Docker 네트워크 접근
k6를 Docker로 실행할 때 `--network backend_default`를 지정해야 앱 컨테이너(`easypay-app:8080`)에 접근할 수 있다. `localhost:8080`은 Docker 컨테이너 내부에서는 동작하지 않는다.

### Rate Limit 오버라이드
부하 테스트 시 `docker` 프로파일에서 Rate Limit을 100,000으로 완화한다. 기본값(30회/시간)으로는 VU 50개가 1분 만에 한도에 도달한다.

### 멱등성 테스트 시 주의
`idempotency-retry.js`에서 같은 키로 재시도할 때, 첫 요청의 응답과 이후 응답의 `transferId`가 동일한지 비교한다. 불일치 시 `idempotency_consistency_rate` 메트릭이 0으로 기록된다.

### Graceful Ramp-down
`stress-test.js`에서 `gracefulRampDown: '30s'` 설정으로, VU 수를 줄일 때 진행 중인 요청이 완료될 시간을 확보한다. 설정하지 않으면 진행 중인 요청이 강제 취소될 수 있다.
