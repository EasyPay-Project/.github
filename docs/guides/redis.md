# Redis (Redisson + Spring Data Redis)

## 핵심 개념

Redis는 인메모리 key-value 저장소로, EasyPay에서는 **분산 락**, **멱등성 보장**, **Rate Limiting**, **Refresh Token 저장** 4가지 용도로 사용한다. Redisson은 Redis 기반의 Java 분산 객체 프레임워크로, 분산 락(RLock, MultiLock) 구현에 활용한다.

### 사용하는 Redis 자료구조

| 용도 | 자료구조 | 키 패턴 | TTL |
|------|----------|---------|-----|
| 분산 락 | Redisson RLock (내부적으로 Hash) | `lock:{accountId}` | 10초 (Lease) |
| 멱등성 | String (JSON) | `idempotency:{key}` | 24시간 |
| API Rate Limit | Sorted Set (ZSET) | `ratelimit:api:user:{userId}` | 1분 |
| 송금 Rate Limit | Sorted Set (ZSET) | `ratelimit:transfer:count:{userId}` | 1시간 |
| Refresh Token | String | `refresh_token:{userId}` | 7일 |

## 프로젝트 적용

### 1. 분산 락 (DistributedLockService)

송금 시 동일 계좌에 대한 동시 요청을 직렬화한다. Redisson의 `MultiLock`으로 송금자/수취자 계좌를 한 번에 잠근다.

**데드락 방지**: 락 키를 **정렬**하여 항상 같은 순서로 획득한다. 예를 들어 A→B 송금과 B→A 송금이 동시에 발생해도, 두 요청 모두 `[A, B]` 순서로 락을 획득하므로 데드락이 발생하지 않는다.

```java
// DistributedLockService.java:39-49
public <T> T executeWithMultiLock(String[] keys, Supplier<T> task) {
    String[] sortedKeys = Arrays.copyOf(keys, keys.length);
    Arrays.sort(sortedKeys);  // 데드락 방지를 위한 키 정렬

    RLock[] locks = Arrays.stream(sortedKeys)
            .map(key -> redissonClient.getLock(LOCK_PREFIX + key))
            .toArray(RLock[]::new);

    RLock multiLock = redissonClient.getMultiLock(locks);
    return executeWithRLock(multiLock, String.join(",", sortedKeys), task);
}
```

**락 파라미터**:
- `WAIT_TIME = 5초`: 락 대기 최대 시간. 초과 시 `LOCK_ACQUISITION_FAILED` 예외
- `LEASE_TIME = 10초`: 락 자동 해제 시간. 비정상 종료 시에도 10초 후 자동 해제

### 2. 멱등성 (IdempotencyService)

클라이언트가 보내는 `Idempotency-Key` 헤더를 기반으로 중복 요청을 방지한다. Redis에 요청의 처리 상태를 저장하여 3가지 시나리오를 처리한다.

```
요청 진입
  → Redis에서 idempotency:{key} 조회
  → 없음: PROCESSING 상태로 저장 → 비즈니스 로직 실행 → COMPLETED + 응답 캐싱
  → PROCESSING: 이미 처리 중 → IDEMPOTENCY_KEY_PROCESSING 예외
  → COMPLETED: 이전 응답 반환 (캐시 히트)
```

**요청 본문 해시**: 같은 멱등성 키로 다른 요청 본문을 보내면 `IDEMPOTENCY_KEY_CONFLICT` 예외를 발생시킨다. SHA-256으로 요청 본문을 해싱하여 비교한다.

```java
// IdempotencyService.java:96-106
public String hashRequest(Object request) {
    String json = objectMapper.writeValueAsString(request);
    MessageDigest digest = MessageDigest.getInstance("SHA-256");
    byte[] hash = digest.digest(json.getBytes(StandardCharsets.UTF_8));
    return HexFormat.of().formatHex(hash);
}
```

**실패 시 복구**: `fail()` 메서드로 Redis 엔트리를 삭제하여 동일 키로 재시도를 허용한다.

### 3. Rate Limiting (RateLimiter + ApiRateLimitFilter)

Redis Sorted Set + Lua 스크립트로 **Sliding Window Counter** 패턴을 구현한다.

```lua
-- RateLimiter.java Lua Script (원자적 실행)
local key = KEYS[1]
local window_start = tonumber(ARGV[1])
local now = tonumber(ARGV[2])
local ttl = tonumber(ARGV[3])

redis.call('ZREMRANGEBYSCORE', key, '-inf', window_start)  -- 윈도우 밖 제거
local count = redis.call('ZCARD', key)                       -- 현재 카운트
redis.call('ZADD', key, now, now .. '-' .. math.random(1000000))  -- 요청 추가
redis.call('EXPIRE', key, ttl)                               -- TTL 설정
return count + 1
```

**왜 Lua 스크립트인가**: ZREMRANGEBYSCORE → ZCARD → ZADD를 개별 명령으로 실행하면 race condition이 발생할 수 있다. Lua 스크립트는 Redis에서 원자적으로 실행되므로 TOCTTOU 문제를 방지한다.

**왜 Sorted Set인가**: 고정 윈도우(Fixed Window)는 윈도우 경계에서 2배의 요청이 통과할 수 있다. Sorted Set은 타임스탬프를 score로 사용하여 정확한 슬라이딩 윈도우를 구현한다.

**2계층 Rate Limiting**:

| 계층 | 서비스 | 기본값 | 윈도우 | 대상 |
|------|--------|--------|--------|------|
| API 전역 | ApiRateLimitFilter | 인증 100/미인증 20 | 1분 | 모든 API |
| 송금 전용 | TransferRateLimitService | 30회/시간, 1건 200만원 | 1시간 | 송금 API |

### 4. Refresh Token (RefreshTokenService)

JWT Refresh Token을 Redis에 key-value로 저장한다. 로그아웃 시 삭제하여 즉시 폐기한다.

```java
// RefreshTokenService.java:18-20
public void store(Long userId, String refreshToken, long expirationMs) {
    redisTemplate.opsForValue().set(KEY_PREFIX + userId, refreshToken,
            Duration.ofMillis(expirationMs));
}
```

## 코드 위치 및 구조

```
src/main/java/com/easypay/
├── infra/redis/
│   ├── DistributedLockService.java    # 분산 락 (Redisson MultiLock)
│   ├── IdempotencyService.java        # 멱등성 (SHA-256 + Redis String)
│   ├── RateLimiter.java               # Sliding Window Counter (Lua + ZSET)
│   ├── TransferRateLimitService.java  # 송금 전용 Rate Limit
│   └── RefreshTokenService.java       # JWT Refresh Token 저장
├── common/config/
│   └── ApiRateLimitFilter.java        # API 전역 Rate Limit 필터
```

## 설정

```yaml
# application-local.yml
spring:
  data:
    redis:
      host: localhost
      port: 6379

# application-docker.yml (부하 테스트용 오버라이드)
easypay:
  rate-limit:
    authenticated: 100000   # API Rate Limit 완화
    anonymous: 10000
  transfer:
    hourly-limit: 100000    # 송금 Rate Limit 완화
    single-limit: 2000000

# application-test.yml (외부 인프라 제외)
spring:
  autoconfigure:
    exclude:
      - org.redisson.spring.starter.RedissonAutoConfigurationV2
      - org.springframework.boot.autoconfigure.data.redis.RedisAutoConfiguration
```

## 동작 흐름

```
송금 요청 진입
  │
  ├─[1] TransferRateLimitService.checkTransferLimit()
  │     └─ RateLimiter.isAllowed("transfer:count:{userId}", 30, 1h)
  │         └─ Redis ZSET: ratelimit:transfer:count:{userId}
  │
  ├─[2] IdempotencyService.startOrGetExisting()
  │     └─ Redis GET: idempotency:{key}
  │     └─ 없으면 SET: {status: "PROCESSING", requestHash: "sha256...", responseBody: null}
  │
  ├─[3] DistributedLockService.executeWithMultiLock()
  │     └─ Redisson MultiLock: lock:{senderAccountId}, lock:{receiverAccountId}
  │     └─ 키 정렬 → tryLock(5s, 10s)
  │
  ├─[4] Saga 실행 (락 내부)
  │
  ├─[5] IdempotencyService.complete()
  │     └─ Redis SET: {status: "COMPLETED", requestHash: "...", responseBody: "{...}"}
  │
  └─[6] 락 해제: lock.unlock()
```

## 트러블슈팅 & 주의사항

### Redisson AutoConfiguration 클래스명
테스트 환경에서 Redisson을 제외할 때 `RedissonAutoConfigurationV2`를 사용해야 한다. `RedissonAutoConfiguration`(V1)은 Spring Boot의 AutoConfiguration 등록 목록에 없어서 exclude해도 효과가 없다.

### 테스트 Mock 설정
`TestRedisConfig`에서 `RedissonClient`와 `StringRedisTemplate`을 Mockito mock으로 제공한다. 테스트 클래스에 `@Import(TestRedisConfig.class)` 필수.

### Lua 스크립트 디버깅
Lua 스크립트 내부에서는 `print()` 등의 디버깅이 어렵다. Redis CLI에서 `MONITOR` 명령으로 실행되는 명령을 확인하거나, 단위 테스트에서 embedded Redis를 사용하여 검증한다.

### 분산 락 키 정렬의 중요성
정렬 없이 `[senderAccount, receiverAccount]` 순서로 락을 획득하면, A→B와 B→A가 동시에 실행될 때 데드락이 발생한다. `Arrays.sort()`로 항상 작은 ID부터 잠근다.
