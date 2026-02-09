# Docker Compose + 멀티스테이지 빌드

## 핵심 개념

### Docker Compose
여러 컨테이너를 하나의 `docker-compose.yml`로 정의하고 관리한다. 서비스 간 의존관계, 네트워크, 볼륨, Health Check를 선언적으로 구성한다.

### 멀티스테이지 빌드
Dockerfile에서 빌드 단계와 실행 단계를 분리하여 최종 이미지 크기를 줄인다. JDK(빌드용) → JRE(실행용)로 전환하면 이미지 크기가 약 1/3로 줄어든다.

## 프로젝트 적용

### Docker Compose 서비스 구성 (7개)

```
┌─────────────────────────────────────────────────────┐
│                  Docker Network (backend_default)     │
│                                                       │
│  ┌─────┐  ┌───────┐  ┌───────┐  ┌───────────┐       │
│  │ app │──│postgres│  │ redis │  │ zookeeper │       │
│  │:8080│  │ :5432  │  │ :6379 │  │   :2181   │       │
│  └──┬──┘  └───────┘  └───────┘  └─────┬─────┘       │
│     │                                   │             │
│     │     ┌───────┐  ┌──────────┐  ┌───┴───┐        │
│     └─────│kafka  │  │prometheus│  │ kafka  │        │
│           │:9092  │  │  :9090   │  │ :29092 │        │
│           │:29092 │  └────┬─────┘  └───────┘        │
│           └───────┘       │                          │
│                      ┌────┴────┐                     │
│                      │ grafana │                     │
│                      │  :3000  │                     │
│                      └─────────┘                     │
└─────────────────────────────────────────────────────┘
```

### 서비스별 상세

| 서비스 | 이미지 | 포트 | Health Check | 의존 |
|--------|--------|------|-------------|------|
| app | 멀티스테이지 빌드 | 8080 | /actuator/health | postgres, redis, kafka |
| postgres | postgres:16-alpine | 5432 | pg_isready | - |
| redis | redis:7-alpine | 6379 | redis-cli ping | - |
| zookeeper | confluentinc/cp-zookeeper:7.5.0 | 2181 | nc -z | - |
| kafka | confluentinc/cp-kafka:7.5.0 | 9092, 29092 | kafka-topics --list | zookeeper |
| prometheus | prom/prometheus:latest | 9090 | - | app |
| grafana | grafana/grafana:latest | 3000 | - | prometheus |

### 의존 관계와 Health Check

```yaml
# docker-compose.yml (app 서비스)
app:
  depends_on:
    postgres:
      condition: service_healthy     # PostgreSQL 준비 대기
    redis:
      condition: service_healthy     # Redis 준비 대기
    kafka:
      condition: service_healthy     # Kafka 준비 대기
  environment:
    SPRING_PROFILES_ACTIVE: docker   # docker 프로파일 활성화
```

`condition: service_healthy`는 단순 시작이 아닌 **Health Check 통과**를 기다린다. PostgreSQL이 쿼리를 받을 준비가 되어야 app이 시작한다.

### 멀티스테이지 빌드 (Dockerfile)

```dockerfile
# Stage 1: 빌드
FROM eclipse-temurin:21-jdk AS build
WORKDIR /app
COPY . .
RUN ./gradlew bootJar --no-daemon

# Stage 2: 실행
FROM eclipse-temurin:21-jre
WORKDIR /app
COPY --from=build /app/build/libs/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

- **Stage 1 (JDK)**: Gradle 빌드 실행. 소스 코드, Gradle Wrapper, 의존성이 포함된 큰 이미지
- **Stage 2 (JRE)**: JAR만 복사. JRE만 포함하여 경량 이미지 (JDK 대비 약 1/3 크기)
- `--no-daemon`: CI/Docker 빌드에서 Gradle Daemon 미사용 (리소스 절약)

### 3개 프로파일

| 프로파일 | 용도 | DB | Redis | Kafka |
|---------|------|-----|-------|-------|
| `local` | 로컬 개발 | localhost:5432 | localhost:6379 | localhost:9092 |
| `docker` | Docker 컨테이너 | postgres:5432 | redis:6379 | kafka:29092 |
| `test` | 테스트 | H2 인메모리 | Mock (TestRedisConfig) | Mock |

### Kafka 듀얼 리스너

Kafka는 Docker 내부와 외부에서 각각 다른 포트로 접근해야 한다.

```yaml
# docker-compose.yml (kafka 서비스)
kafka:
  environment:
    KAFKA_LISTENERS: INTERNAL://0.0.0.0:29092,EXTERNAL://0.0.0.0:9092
    KAFKA_ADVERTISED_LISTENERS: INTERNAL://kafka:29092,EXTERNAL://localhost:9092
    KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: INTERNAL:PLAINTEXT,EXTERNAL:PLAINTEXT
    KAFKA_INTER_BROKER_LISTENER_NAME: INTERNAL
```

- **INTERNAL (29092)**: `kafka:29092` — Docker 네트워크 내부에서 `app` 컨테이너가 사용
- **EXTERNAL (9092)**: `localhost:9092` — 호스트에서 로컬 개발 또는 k6 테스트 시 사용

### 볼륨 영속화

```yaml
volumes:
  postgres_data:     # PostgreSQL 데이터 영속화

services:
  postgres:
    volumes:
      - postgres_data:/var/lib/postgresql/data
```

`docker-compose down`해도 데이터가 유지된다. `docker-compose down -v`로 볼륨까지 삭제 가능.

## 코드 위치 및 구조

```
backend/
├── docker-compose.yml              # 7개 서비스 정의
├── Dockerfile                       # 멀티스테이지 빌드
├── src/main/resources/
│   ├── application.yml              # 공통 설정
│   ├── application-local.yml        # 로컬 개발 설정
│   ├── application-docker.yml       # Docker 컨테이너 설정
│   └── application-test.yml         # 테스트 설정 (H2 + Mock)
├── monitoring/
│   ├── prometheus.yml               # Prometheus 스크래핑 설정
│   └── grafana/                     # Grafana 대시보드/알림
```

## 설정

### 주요 환경 변수 (docker 프로파일)

```yaml
# application-docker.yml
spring:
  datasource:
    url: jdbc:postgresql://postgres:5432/easypay    # Docker 내부 DNS
  data:
    redis:
      host: redis                                    # Docker 내부 DNS
  kafka:
    bootstrap-servers: kafka:29092                   # 내부 리스너

# Rate Limit 완화 (부하 테스트용)
easypay:
  rate-limit:
    authenticated: 100000
    anonymous: 10000
  transfer:
    hourly-limit: 100000
```

## 동작 흐름

### 전체 실행
```bash
docker-compose up -d
# 1. postgres, redis, zookeeper 시작
# 2. kafka 시작 (zookeeper healthy 대기)
# 3. app 시작 (postgres, redis, kafka healthy 대기)
# 4. prometheus 시작 (app:8080 스크래핑)
# 5. grafana 시작 (prometheus 데이터소스)
```

### 인프라만 실행 + 앱 로컬
```bash
docker-compose up -d postgres redis zookeeper kafka prometheus grafana
./gradlew bootRun   # local 프로파일 (기본값)
```

### 앱 리빌드
```bash
docker-compose up -d --build app   # 코드 변경 후 이미지 재빌드
```

## 트러블슈팅 & 주의사항

### 앱 컨테이너 재빌드 필수
`docker-compose up -d`는 기존 이미지가 있으면 재사용한다. 코드를 변경했으면 반드시 `--build` 플래그를 추가해야 한다.

### PostgreSQL Health Check 타이밍
PostgreSQL이 시작되더라도 초기화 스크립트가 완료되지 않을 수 있다. `pg_isready` 명령은 연결 수락 가능 여부만 확인하므로, JPA DDL 자동 생성이 실패할 수 있다. `interval: 10s`, `retries: 5`로 충분한 대기 시간을 확보한다.

### Kafka 듀얼 리스너 혼동
Docker 내부에서 `localhost:9092`로 접근하면 실패한다. `application-docker.yml`에서 반드시 `kafka:29092` (INTERNAL 리스너)를 사용해야 한다.

### 포트 충돌
Grafana(3000)와 Next.js(기본 3000)가 충돌할 수 있다. Next.js Dev 서버를 3001 포트로 변경하여 해결했다.
