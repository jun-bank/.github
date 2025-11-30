# Jun-Bank 도메인 설계 정리

## 📋 공통 설계 원칙

### 1. ID 생성 전략
- **Entity 레이어에서 ID 생성** (fromDomain 시점)
- **UuidUtils 사용**: `PREFIX-xxxxxxxx` 형식
- **도메인 모델**: ID가 null이면 신규, 있으면 기존

### 2. ID 프리픽스 (common-lib UuidUtils)
| 도메인 | 프리픽스 | 예시 |
|--------|----------|------|
| User | USR | USR-a1b2c3d4 |
| Account | ACC | ACC-e5f6g7h8 |
| Transaction | TXN | TXN-20250115143052-a1b2c3 (타임스탬프) |
| Transfer | TRF | TRF-20250115143052-a1b2c3 (타임스탬프) |
| Card | CRD | CRD-i9j0k1l2 |
| Ledger | LDG | LDG-20250115143052-a1b2c3 (타임스탬프) |
| Event | EVT | EVT-m3n4o5p6 |
| RefreshToken | RTK | RTK-xxxxxxxx (추가 필요) |
| Payment | PAY | PAY-xxxxxxxx (추가 필요) |

### 3. Soft Delete
- **도메인 모델**: `status` 필드로 표현 (DELETED 상태)
- **Entity (BaseEntity)**: `isDeleted`, `deletedAt`, `deletedBy`
- **매핑**: 도메인 status가 DELETED → Entity isDeleted = true

### 4. 감사 필드 (BaseEntity)
| 필드 | 설명 | 도메인에서 |
|------|------|-----------|
| createdAt | 생성일시 | O (조회용) |
| updatedAt | 수정일시 | O (조회용) |
| createdBy | 생성자 | O (조회용) |
| updatedBy | 수정자 | O (조회용) |
| deletedAt | 삭제일시 | X (Entity에서 status 보고 설정) |
| deletedBy | 삭제자 | X (Entity에서 설정) |
| isDeleted | 삭제여부 | X (Entity에서 status 보고 설정) |

---

## 🏛️ 서비스별 도메인 모델

### 1. User Service (8087)

#### User
```
┌─────────────────────────────────────────────┐
│                    User                      │
├─────────────────────────────────────────────┤
│ userId: UserId (USR-xxxxxxxx)               │
│ email: Email (VO)                           │
│ password: Password (VO, 암호화)             │
│ name: String                                │
│ phoneNumber: PhoneNumber (VO)               │
│ birthDate: LocalDate                        │
│ status: UserStatus                          │
│ createdAt: LocalDateTime                    │
│ updatedAt: LocalDateTime                    │
│ createdBy: String                           │
│ updatedBy: String                           │
└─────────────────────────────────────────────┘

비즈니스 메서드:
- create(): 신규 생성 (userId는 null)
- restore(): DB 복원
- updateProfile(name, phoneNumber)
- changePassword(newPassword)
- withdraw(): status → DELETED
- suspend(): status → SUSPENDED
- activate(): status → ACTIVE
- isNew(): userId == null

상태 전이:
- ACTIVE ↔ INACTIVE (휴면)
- ACTIVE ↔ SUSPENDED (정지)
- ACTIVE/INACTIVE/SUSPENDED → DELETED (탈퇴, 복구 불가)
```

#### UserStatus
```java
ACTIVE,     // 정상
INACTIVE,   // 휴면
SUSPENDED,  // 정지
DELETED     // 탈퇴 (Soft Delete)
```

#### Value Objects
- UserId: USR-xxxxxxxx 패턴 검증
- Email: 이메일 형식 검증
- Password: 암호화된 값 보관
- PhoneNumber: 전화번호 형식 검증, 마스킹

---

### 2. Auth Server (8086)

#### RefreshToken
```
┌─────────────────────────────────────────────┐
│               RefreshToken                   │
├─────────────────────────────────────────────┤
│ refreshTokenId: RefreshTokenId (RTK-xxx)    │
│ userId: UserId (USR-xxx, 참조)              │
│ token: String (Unique)                      │
│ expiresAt: LocalDateTime                    │
│ isRevoked: Boolean                          │
│ deviceInfo: String                          │
│ createdAt: LocalDateTime                    │
└─────────────────────────────────────────────┘

비즈니스 메서드:
- create(): 신규 토큰 생성
- revoke(): 토큰 무효화
- isExpired(): 만료 여부
- isValid(): !isRevoked && !isExpired
```

#### LoginHistory
```
┌─────────────────────────────────────────────┐
│               LoginHistory                   │
├─────────────────────────────────────────────┤
│ loginHistoryId: LoginHistoryId              │
│ userId: UserId                              │
│ email: String                               │
│ loginAt: LocalDateTime                      │
│ ipAddress: String                           │
│ userAgent: String                           │
│ success: Boolean                            │
│ failReason: String                          │
└─────────────────────────────────────────────┘

특성: Append-only (INSERT만)
```

---

### 3. Account Service (8081)

#### Account
```
┌─────────────────────────────────────────────┐
│                  Account                     │
├─────────────────────────────────────────────┤
│ accountId: AccountId (ACC-xxxxxxxx)         │
│ accountNumber: AccountNumber (VO, 14자리)   │
│ userId: UserId (소유자)                     │
│ accountType: AccountType                    │
│ balance: Money (VO)                         │
│ dailyWithdrawalAmount: Money                │
│ status: AccountStatus                       │
│ createdAt: LocalDateTime                    │
│ updatedAt: LocalDateTime                    │
│ createdBy: String                           │
│ updatedBy: String                           │
└─────────────────────────────────────────────┘

비즈니스 메서드:
- create(): 신규 계좌 개설
- deposit(amount): 입금
- withdraw(amount): 출금 (잔액 검증)
- freeze(): 동결
- close(): 해지
- isWithdrawable(amount): 출금 가능 여부

상태 전이:
- ACTIVE ↔ DORMANT (휴면)
- ACTIVE → FROZEN (동결)
- FROZEN → ACTIVE (해제)
- ACTIVE/DORMANT → CLOSED (해지)
```

#### AccountType
```java
CHECKING,   // 입출금 통장
SAVINGS,    // 저축 통장
DEPOSIT     // 정기 예금
```

#### AccountStatus
```java
ACTIVE,     // 정상
DORMANT,    // 휴면
FROZEN,     // 동결
CLOSED      // 해지
```

#### Value Objects
- AccountId: ACC-xxxxxxxx 패턴
- AccountNumber: 14자리 계좌번호, 체크섬(Luhn) 검증
- Money: BigDecimal 래핑, 금액 연산

---

### 4. Transaction Service (8082)

#### Transaction
```
┌─────────────────────────────────────────────┐
│               Transaction                    │
├─────────────────────────────────────────────┤
│ transactionId: TransactionId (TXN-ts-xxx)   │
│ accountId: AccountId                        │
│ type: TransactionType                       │
│ amount: Money                               │
│ balanceAfter: Money                         │
│ status: TransactionStatus                   │
│ description: String                         │
│ idempotencyKey: String (Nullable)           │
│ createdAt: LocalDateTime                    │
│ processedAt: LocalDateTime                  │
└─────────────────────────────────────────────┘

비즈니스 메서드:
- createDeposit(): 입금 트랜잭션 생성
- createWithdrawal(): 출금 트랜잭션 생성
- complete(): 완료 처리
- fail(reason): 실패 처리
- cancel(): 취소 처리
```

#### IdempotencyRecord
```
┌─────────────────────────────────────────────┐
│            IdempotencyRecord                 │
├─────────────────────────────────────────────┤
│ id: Long (Auto)                             │
│ idempotencyKey: String (Unique)             │
│ requestHash: String                         │
│ responseBody: String (JSON)                 │
│ httpStatus: Integer                         │
│ createdAt: LocalDateTime                    │
│ expiresAt: LocalDateTime                    │
└─────────────────────────────────────────────┘

특성: TTL 기반 자동 삭제
```

#### TransactionType
```java
DEPOSIT,      // 입금
WITHDRAWAL,   // 출금
TRANSFER_IN,  // 이체 입금
TRANSFER_OUT, // 이체 출금
PAYMENT,      // 결제
REFUND        // 환불
```

#### TransactionStatus
```java
PENDING,    // 처리 중
SUCCESS,    // 성공
FAILED,     // 실패
CANCELLED   // 취소
```

---

### 5. Transfer Service (8083) - SAGA 오케스트레이터

#### Transfer
```
┌─────────────────────────────────────────────┐
│                 Transfer                     │
├─────────────────────────────────────────────┤
│ transferId: TransferId (TRF-ts-xxx)         │
│ fromAccountNumber: AccountNumber            │
│ toAccountNumber: AccountNumber              │
│ amount: Money                               │
│ fee: Money                                  │
│ status: TransferStatus                      │
│ sagaStatus: SagaStatus                      │
│ failReason: String                          │
│ memo: String                                │
│ requestedAt: LocalDateTime                  │
│ completedAt: LocalDateTime                  │
└─────────────────────────────────────────────┘

비즈니스 메서드:
- create(): 이체 요청 생성
- startDebit(): 출금 시작
- completeDebit(): 출금 완료
- failDebit(reason): 출금 실패
- startCredit(): 입금 시작
- completeCredit(): 입금 완료 → 전체 완료
- failCredit(reason): 입금 실패 → 보상 시작
- compensate(): 보상 트랜잭션 시작
- completeCompensation(): 보상 완료
```

#### OutboxEvent (Outbox 패턴)
```
┌─────────────────────────────────────────────┐
│               OutboxEvent                    │
├─────────────────────────────────────────────┤
│ outboxId: Long (Auto)                       │
│ aggregateType: String                       │
│ aggregateId: String                         │
│ eventType: String                           │
│ payload: String (JSON)                      │
│ status: OutboxStatus                        │
│ retryCount: Integer                         │
│ createdAt: LocalDateTime                    │
│ sentAt: LocalDateTime                       │
└─────────────────────────────────────────────┘
```

#### SagaStatus
```java
STARTED,           // SAGA 시작
DEBIT_PENDING,     // 출금 요청 중
DEBIT_COMPLETED,   // 출금 완료
DEBIT_FAILED,      // 출금 실패
CREDIT_PENDING,    // 입금 요청 중
CREDIT_COMPLETED,  // 입금 완료
CREDIT_FAILED,     // 입금 실패
COMPENSATING,      // 보상 진행 중
COMPENSATED,       // 보상 완료
COMPLETED,         // 성공 완료
FAILED             // 최종 실패
```

---

### 6. Card Service (8084)

#### Card
```
┌─────────────────────────────────────────────┐
│                   Card                       │
├─────────────────────────────────────────────┤
│ cardId: CardId (CRD-xxxxxxxx)               │
│ cardNumber: CardNumber (VO, 암호화)         │
│ userId: UserId                              │
│ accountId: AccountId                        │
│ cardType: CardType                          │
│ status: CardStatus                          │
│ expiryDate: YearMonth                       │
│ cvv: String (암호화)                        │
│ dailyLimit: Money                           │
│ monthlyLimit: Money                         │
│ dailyUsed: Money                            │
│ monthlyUsed: Money                          │
│ createdAt: LocalDateTime                    │
│ updatedAt: LocalDateTime                    │
└─────────────────────────────────────────────┘

비즈니스 메서드:
- create(): 카드 발급
- block(): 분실/도난 신고
- unblock(): 차단 해제
- terminate(): 해지
- canPay(amount): 결제 가능 여부 (한도 체크)
- useLimit(amount): 사용 금액 반영
- resetDailyLimit(): 일일 한도 초기화
- resetMonthlyLimit(): 월간 한도 초기화
```

#### Payment
```
┌─────────────────────────────────────────────┐
│                  Payment                     │
├─────────────────────────────────────────────┤
│ paymentId: PaymentId (PAY-xxxxxxxx)         │
│ cardId: CardId                              │
│ merchantName: String                        │
│ merchantId: String                          │
│ amount: Money                               │
│ status: PaymentStatus                       │
│ approvalNumber: String                      │
│ failReason: String                          │
│ requestedAt: LocalDateTime                  │
│ approvedAt: LocalDateTime                   │
│ cancelledAt: LocalDateTime                  │
└─────────────────────────────────────────────┘

비즈니스 메서드:
- create(): 결제 요청 생성
- approve(approvalNumber): 승인
- decline(reason): 거절
- cancel(): 취소
- refund(): 환불
```

#### CardType
```java
DEBIT,    // 체크카드
CREDIT    // 신용카드
```

#### CardStatus
```java
ACTIVE,      // 정상
INACTIVE,    // 비활성화
BLOCKED,     // 분실/도난 신고
EXPIRED,     // 만료
TERMINATED   // 해지
```

#### PaymentStatus
```java
PENDING,    // 처리 중
APPROVED,   // 승인
DECLINED,   // 거절
CANCELLED,  // 취소
REFUNDED    // 환불
```

---

### 7. Ledger Service (8085)

#### LedgerEntry (Append-only)
```
┌─────────────────────────────────────────────┐
│               LedgerEntry                    │
├─────────────────────────────────────────────┤
│ ledgerEntryId: LedgerEntryId (LDG-ts-xxx)   │
│ transactionId: String (원본 거래 ID)        │
│ accountNumber: AccountNumber                │
│ entryType: EntryType (DEBIT/CREDIT)         │
│ amount: Money                               │
│ balanceAfter: Money                         │
│ description: String                         │
│ category: TransactionCategory               │
│ referenceType: String                       │
│ referenceId: String                         │
│ createdAt: LocalDateTime                    │
└─────────────────────────────────────────────┘

특성:
- INSERT만 허용 (UPDATE/DELETE 금지)
- 복식부기: 차변 합 = 대변 합

비즈니스 메서드:
- create(): 원장 기록 생성 (팩토리 메서드만)
- 수정/삭제 메서드 없음
```

#### AuditLog (Append-only)
```
┌─────────────────────────────────────────────┐
│                AuditLog                      │
├─────────────────────────────────────────────┤
│ auditLogId: AuditLogId                      │
│ eventId: String (UUID)                      │
│ eventType: String                           │
│ serviceName: String                         │
│ userId: UserId                              │
│ resourceType: String                        │
│ resourceId: String                          │
│ action: String                              │
│ previousValue: String (JSON)                │
│ newValue: String (JSON)                     │
│ ipAddress: String                           │
│ userAgent: String                           │
│ timestamp: LocalDateTime                    │
└─────────────────────────────────────────────┘

특성: INSERT만 허용
```

#### EntryType
```java
DEBIT,   // 차변
CREDIT   // 대변
```

#### TransactionCategory
```java
DEPOSIT,       // 입금
WITHDRAWAL,    // 출금
TRANSFER_IN,   // 이체 입금
TRANSFER_OUT,  // 이체 출금
PAYMENT,       // 결제
REFUND,        // 환불
FEE,           // 수수료
INTEREST       // 이자
```

---

## 🔗 서비스 간 ID 참조

```
User Service
└── User (USR-xxx)
    │
    ├── Auth Server
    │   ├── RefreshToken (RTK-xxx) → userId
    │   └── LoginHistory → userId
    │
    ├── Account Service
    │   └── Account (ACC-xxx) → userId
    │       │
    │       ├── Transaction Service
    │       │   └── Transaction (TXN-ts-xxx) → accountId
    │       │
    │       ├── Transfer Service
    │       │   └── Transfer (TRF-ts-xxx) → fromAccountNumber, toAccountNumber
    │       │
    │       └── Card Service
    │           ├── Card (CRD-xxx) → userId, accountId
    │           └── Payment (PAY-xxx) → cardId
    │
    └── Ledger Service
        ├── LedgerEntry (LDG-ts-xxx) → accountNumber, transactionId
        └── AuditLog → userId, resourceId
```

---

## 📌 추가 필요 사항

### common-lib UuidUtils에 추가할 프리픽스
```java
public static final String PREFIX_REFRESH_TOKEN = "RTK";
public static final String PREFIX_LOGIN_HISTORY = "LGH";
public static final String PREFIX_PAYMENT = "PAY";
public static final String PREFIX_OUTBOX = "OBX";
public static final String PREFIX_AUDIT = "AUD";
```

### 공통 Value Objects (common-lib 또는 각 서비스)
- Money: BigDecimal 래핑, 금액 연산, 통화
- 각 서비스별 ID VO: 패턴 검증만 담당

---

## 🚀 구현 순서

1. **User Service** - 기본 CRUD, 이벤트 발행
2. **Auth Server** - JWT, 리프레시 토큰
3. **Account Service** - 계좌 관리, 낙관적/비관적 락
4. **Transaction Service** - 입출금, 멱등성
5. **Transfer Service** - SAGA, Outbox 패턴
6. **Card Service** - 결제, Circuit Breaker
7. **Ledger Service** - Append-only, 이벤트 소싱