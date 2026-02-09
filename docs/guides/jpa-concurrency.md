# JPA + 낙관적 락

## 핵심 개념

### JPA (Java Persistence API)
ORM(Object-Relational Mapping) 표준으로, Java 객체와 DB 테이블을 매핑한다. Hibernate가 구현체이며, `@Entity`, `@Table`, `@Column` 등의 어노테이션으로 매핑을 정의한다.

### 낙관적 락 (Optimistic Locking)
**"충돌은 드물다"**는 가정 하에, 락을 미리 잡지 않고 커밋 시점에 충돌을 감지하는 방식이다. `@Version` 필드를 사용하여 JPA가 UPDATE 쿼리에 `WHERE version = ?`을 자동으로 추가한다. 다른 트랜잭션이 먼저 수정했다면 version이 달라져 `OptimisticLockingFailureException`이 발생한다.

```sql
-- JPA가 자동 생성하는 UPDATE 쿼리
UPDATE account SET balance = ?, version = version + 1
WHERE id = ? AND version = ?
--                         ↑ 이전에 읽은 version과 비교
```

### EasyPay에서의 역할
`@Version`은 동시성 제어의 **최종 안전망**(5번째 계층)이다. 분산 락이 정상 동작하면 version 충돌이 발생하지 않지만, 락 누수나 예외 상황에서 데이터 정합성을 보장한다.

## 프로젝트 적용

### BaseTimeEntity

모든 엔티티의 공통 생성/수정 시간을 관리하는 추상 클래스.

```java
// BaseTimeEntity.java
@Getter
@MappedSuperclass
@EntityListeners(AuditingEntityListener.class)
public abstract class BaseTimeEntity {

    @CreatedDate
    @Column(nullable = false, updatable = false)
    private LocalDateTime createdAt;

    @LastModifiedDate
    @Column(nullable = false)
    private LocalDateTime updatedAt;
}
```

- `@MappedSuperclass`: 테이블을 생성하지 않고 자식 엔티티의 컬럼으로 포함
- `@EntityListeners(AuditingEntityListener.class)`: JPA Auditing 리스너 등록
- `@CreatedDate`, `@LastModifiedDate`: 자동 시간 주입
- `JpaAuditingConfig`에서 `@EnableJpaAuditing`으로 활성화

### 엔티티 설계 패턴

모든 엔티티는 다음 규칙을 따른다:

```java
@Entity
@Table(name = "account", indexes = @Index(...))
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)  // JPA용 기본 생성자 (외부 호출 차단)
public class Account extends BaseTimeEntity {

    @Builder                                          // 빌더 패턴으로 생성
    public Account(Long userId, String accountNumber, BigDecimal dailyLimit) {
        this.userId = userId;
        this.accountNumber = accountNumber;
        this.balance = BigDecimal.ZERO;               // 기본값 설정
        this.status = AccountStatus.ACTIVE;
    }

    // 비즈니스 로직은 엔티티 내부 메서드
    public void withdraw(BigDecimal amount) {
        validateActive();
        if (this.balance.compareTo(amount) < 0) {
            throw new BusinessException(ErrorCode.INSUFFICIENT_BALANCE);
        }
        this.balance = this.balance.subtract(amount);
    }
}
```

**핵심 원칙**:
1. **`@NoArgsConstructor(PROTECTED)`**: JPA는 기본 생성자가 필요하지만, 외부에서 호출하면 안 됨
2. **`@Builder`**: 생성자에 붙여서 필수 필드를 명시하고 기본값 설정
3. **비즈니스 메서드**: `withdraw()`, `deposit()` 등을 엔티티에 정의하여 도메인 로직 캡슐화
4. **Setter 없음**: `@Getter`만 사용하고, 상태 변경은 의미 있는 메서드로만

### 낙관적 락 적용 (Account)

```java
// Account.java:41-42
@Version
private Long version;
```

Account 엔티티에만 `@Version`이 적용된다. Transfer, Payment 등은 상태 전이 메서드로 관리하고, 동시 수정이 발생하는 것은 계좌 잔액뿐이기 때문이다.

### 상태 머신

각 도메인 엔티티는 상태 전이를 메서드로 캡슐화한다.

**Transfer (6개 상태)**:
```
PENDING → PROCESSING → COMPLETED
         (Saga 실행)   (정상 완료)
                     → FAILED      (Saga 실패)
                     → CANCELLED   (사용자 취소)
                     → COMPENSATED (보상 완료)
```

**Payment (5개 상태)**:
```
PENDING → COMPLETED
        → FAILED
        → CANCELLED
        → REFUNDED    (환불)
```

**Settlement (4개 상태)**:
```
PENDING → PROCESSING → COMPLETED
                     → FAILED
```

**Account (3개 상태)**:
```
ACTIVE → FROZEN
       → CLOSED
```

### BaseTimeEntity 상속 여부

| 엔티티 | BaseTimeEntity 상속 | 이유 |
|--------|-------------------|------|
| User | O | 생성/수정 시간 자동 관리 |
| Account | O | 생성/수정 시간 자동 관리 |
| Merchant | O | 생성/수정 시간 자동 관리 |
| Transfer | X | 커스텀 createdAt/completedAt 필요 |
| Payment | X | 커스텀 createdAt/completedAt 필요 |
| Settlement | X | 커스텀 createdAt 필요 |
| Transaction | X | 불변 레코드, createdAt만 필요 |

Transfer/Payment는 `completedAt`이라는 별도 타임스탬프가 필요하므로 BaseTimeEntity를 상속하지 않고 직접 관리한다.

## 코드 위치 및 구조

```
src/main/java/com/easypay/
├── common/
│   ├── domain/BaseTimeEntity.java        # 공통 시간 엔티티
│   └── config/JpaAuditingConfig.java     # @EnableJpaAuditing
├── user/domain/
│   ├── User.java                         # 사용자 (BaseTimeEntity 상속)
│   └── Role.java                         # USER, MERCHANT, ADMIN
├── account/domain/
│   ├── Account.java                      # 계좌 (@Version 낙관적 락)
│   ├── AccountStatus.java               # ACTIVE, FROZEN, CLOSED
│   ├── Transaction.java                  # 거래 내역 (불변)
│   └── TransactionType.java             # DEBIT, CREDIT
├── transfer/domain/
│   ├── Transfer.java                     # 송금 (6개 상태)
│   └── TransferStatus.java
├── payment/domain/
│   ├── Payment.java                      # 결제 (5개 상태)
│   ├── PaymentStatus.java
│   ├── Merchant.java                     # 가맹점
│   └── MerchantStatus.java              # ACTIVE, INACTIVE
└── settlement/domain/
    ├── Settlement.java                   # 정산 (4개 상태)
    └── SettlementStatus.java
```

## 설정

```groovy
// build.gradle - Querydsl 설정
implementation 'com.querydsl:querydsl-jpa:5.1.0:jakarta'
annotationProcessor 'com.querydsl:querydsl-apt:5.1.0:jakarta'
annotationProcessor 'jakarta.annotation:jakarta.annotation-api'
annotationProcessor 'jakarta.persistence:jakarta.persistence-api'

// Q클래스 생성 경로
def querydslDir = layout.buildDirectory.dir("generated/querydsl")
sourceSets { main.java.srcDirs += querydslDir.get().asFile }
```

Querydsl은 인프라로 준비되어 있으나, 현재 프로젝트에서는 실사용하지 않는다. 복잡한 동적 쿼리가 필요할 때 활용할 수 있다.

```yaml
# application-test.yml (H2 PostgreSQL 호환 모드)
spring:
  datasource:
    url: jdbc:h2:mem:easypay_test;DB_CLOSE_DELAY=-1;DB_CLOSE_ON_EXIT=FALSE;MODE=PostgreSQL
  jpa:
    hibernate:
      ddl-auto: create-drop
    properties:
      hibernate:
        dialect: org.hibernate.dialect.H2Dialect
```

## 동작 흐름 (낙관적 락)

```
Transaction A (송금 10,000원)          Transaction B (송금 5,000원)
  │                                     │
  │── SELECT * FROM account             │── SELECT * FROM account
  │   WHERE id = 1                      │   WHERE id = 1
  │   → balance=100,000, version=5      │   → balance=100,000, version=5
  │                                     │
  │── Account.withdraw(10,000)          │── Account.withdraw(5,000)
  │   balance = 90,000                  │   balance = 95,000
  │                                     │
  │── UPDATE account                    │
  │   SET balance=90,000, version=6     │
  │   WHERE id=1 AND version=5          │
  │   → 성공 (1 row affected)           │
  │                                     │── UPDATE account
  │                                     │   SET balance=95,000, version=6
  │                                     │   WHERE id=1 AND version=5
  │                                     │   → 실패! (0 rows, version 이미 6)
  │                                     │   → OptimisticLockingFailureException
```

## 트러블슈팅 & 주의사항

### transaction 예약어
PostgreSQL에서 `transaction`은 예약어이므로 테이블명으로 사용할 수 없다. `@Table(name = "transactions")`로 복수형을 사용한다. H2 PostgreSQL 호환 모드에서도 동일하게 적용된다.

### BigDecimal 사용
금액 필드는 반드시 `BigDecimal`을 사용한다. `double`이나 `float`는 부동소수점 오차가 발생하여 금융 계산에 부적합하다. JPA 매핑: `@Column(precision = 15, scale = 2)` → 최대 9,999,999,999,999.99원.

### @Version과 분산 락의 관계
분산 락이 정상 동작하면 `@Version` 충돌은 발생하지 않는다. 그러나 **방어적 프로그래밍** 관점에서 @Version을 유지한다:
1. 분산 락 없이 직접 계좌를 수정하는 관리 기능
2. 락 Lease Time(10초) 만료 후 다른 요청이 진입하는 극단적 상황
3. Redis 장애로 분산 락이 동작하지 않는 상황

### JPA Auditing과 테스트
`@EnableJpaAuditing`은 `JpaAuditingConfig`에 분리하여, `@WebMvcTest`에서 불필요한 JPA 설정 로딩을 방지한다. `@SpringBootTest`에서만 활성화된다.
