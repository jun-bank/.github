# Jun Bank

MSA 환경에서 은행 도메인의 트랜잭션 관리와 데이터 일관성을 학습하기 위한 프로젝트입니다.

---

## 왜 이 프로젝트를 시작했나?

결제 시스템을 개발할 때마다 실제 은행 API나 PG사 연동이 필요합니다.  
테스트를 위해 매번 복잡한 계약 절차를 거치거나, 샌드박스 환경의 제약에 부딪히게 됩니다.

이 프로젝트는 **가상의 은행 시스템**을 직접 구축하여:

- 결제 시스템 개발 시 자유롭게 테스트할 수 있는 환경 제공
- 은행 도메인의 인프라 구조 이해
- **MSA 환경에서 트랜잭션 관리** 방법 학습
- **데이터 일관성**과 **동시성 제어** 문제 해결 경험

---

## 기술 스택

| 구분 | 기술 | 버전 |
|------|------|------|
| Language | Java | 21 |
| Framework | Spring Boot | 4.0.0 |
| Cloud | Spring Cloud | 2025.1.0 |
| Database | PostgreSQL + pgvector | 17+ |
| Messaging | Apache Kafka (KRaft) | 3.8.0 |
| Query | QueryDSL | 5.1.0 jakarta |
| Container | Docker, Docker Compose | - |
| Monitoring | Prometheus, Grafana, Zipkin | - |

---

## 아키텍처

### 전체 구조

```
                           ┌─────────────────┐
                           │     Nginx       │
                           │    (L7 LB)      │
                           └────────┬────────┘
                                    │
                    ┌───────────────┴───────────────┐
                    ▼                               ▼
            ┌───────────────┐               ┌───────────────┐
            │ Gateway Server│               │ Gateway Server│
            │   -1 :8080    │               │   -2 :8089    │
            └───────┬───────┘               └───────┬───────┘
                    │                               │
                    └───────────────┬───────────────┘
                                    │
        ┌───────────────────────────┼───────────────────────────┐
        │                           │                           │
        ▼                           ▼                           ▼
┌───────────────┐           ┌───────────────┐           ┌───────────────┐
│ User Service  │           │ Auth Server   │           │Account Service│
│    :8087      │           │    :8086      │           │    :8081      │
└───────────────┘           └───────────────┘           └───────────────┘
        │                           │                           │
        ▼                           ▼                           ▼
┌───────────────┐           ┌───────────────┐           ┌───────────────┐
│Transaction Svc│           │Transfer Service│          │ Card Service  │
│    :8082      │           │    :8083      │           │    :8084      │
└───────────────┘           └───────────────┘           └───────────────┘
        │                           │                           │
        └───────────────────────────┼───────────────────────────┘
                                    │
                                    ▼
                            ┌───────────────┐
                            │Ledger Service │
                            │    :8085      │
                            └───────────────┘
```

### 인프라 서버

```
┌─────────────────┐     ┌─────────────────┐
│ Eureka Server-1 │◄───►│ Eureka Server-2 │
│     :8761       │     │     :8762       │
└─────────────────┘     └─────────────────┘
         ▲
         │
┌─────────────────┐
│  Config Server  │
│     :8888       │
└─────────────────┘
```

---

## 서비스 목록

### 인프라 서버 (3개)

| 서비스 | 포트 | 설명 |
|--------|------|------|
| [eureka-server](./eureka-server) | 8761, 8762 | 서비스 디스커버리 (이중화) |
| [config-server](./config-server) | 8888 | 중앙 설정 관리 |
| [gateway-server](./gateway-server) | 8080, 8089 | API Gateway (이중화) |

### 비즈니스 서비스 (7개)

| 서비스 | 포트 | 설명 | 핵심 학습 |
|--------|------|------|----------|
| [user-service](./user-service) | 8087 | 사용자 관리 | 기본 CRUD, 이벤트 발행 |
| [auth-server](./auth-server) | 8086 | 인증/인가 | JWT, Spring Security |
| [account-service](./account-service) | 8081 | 계좌 관리 | 낙관적/비관적 락, 동시성 |
| [transaction-service](./transaction-service) | 8082 | 입출금 처리 | 멱등성 (Idempotency Key) |
| [transfer-service](./transfer-service) | 8083 | 계좌 이체 | SAGA 패턴, Outbox 패턴 |
| [card-service](./card-service) | 8084 | 카드 결제 | Resilience4j, 한도 관리 |
| [ledger-service](./ledger-service) | 8085 | 원장 기록 | 불변 데이터, 복식부기 |

---

## 핵심 학습 주제

### 1. 분산 트랜잭션 (Transfer Service)

단일 DB에서는 ACID가 보장되지만, MSA에서는 서비스마다 DB가 분리됩니다.

- **SAGA 패턴**: 보상 트랜잭션을 통한 Eventually Consistent
- **Outbox 패턴**: DB 트랜잭션과 이벤트 발행의 원자성 보장

### 2. 동시성 제어 (Account Service)

같은 계좌에 동시에 여러 거래가 발생하면 잔액이 꼬일 수 있습니다.

- **낙관적 락**: `@Version` 필드를 통한 충돌 감지
- **비관적 락**: `@Lock(PESSIMISTIC_WRITE)` DB 레벨 잠금

### 3. 멱등성 (Transaction Service)

네트워크 장애로 같은 요청이 중복 전송될 수 있습니다.

- **Idempotency Key**: `X-Idempotency-Key` 헤더로 중복 요청 방지

### 4. 장애 대응 (Card Service)

일부 서비스가 죽어도 전체 시스템은 동작해야 합니다.

- **Circuit Breaker**: 장애 전파 차단
- **Retry + Backoff**: 일시적 장애 복구
- **Rate Limiter**: 과부하 방지

---

## 프로젝트 구조

### 공통 라이브러리

```
common-lib/                      # 공통 모듈 (Maven Central 배포)
├── event/                       # IntegrationEvent (멱등성/순서보장/재시도/TTL)
├── util/                        # UuidUtils (도메인별 ID 생성)
└── exception/                   # 공통 예외
```

**의존성 추가**
```groovy
implementation 'io.github.jun-bank:common-lib:0.0.1'
```

### 서비스별 아키텍처 (헥사고날 + 도메인 중심 하이브리드)

```
service/
├── global/                      # 전역 설정 레이어
│   ├── config/                  # JPA, Kafka, Security, Feign, Swagger, Async
│   ├── infrastructure/          # BaseEntity, AuditorAware
│   ├── security/                # UserPrincipal, HeaderAuthenticationFilter
│   ├── feign/                   # FeignErrorDecoder, FeignRequestInterceptor
│   └── aop/                     # LoggingAspect
└── domain/
    └── {domain}/                # 도메인 단위
        ├── domain/              # 순수 도메인 (Entity, VO, Enum)
        ├── application/         # 유스케이스 + Port + DTO
        ├── infrastructure/      # Adapter (Out) - Repository, Kafka
        └── presentation/        # Adapter (In) - Controller
```

### Global 레이어 공통 구성 (110개 파일)

| 패키지 | 파일 | 설명 |
|--------|------|------|
| `config/` | JpaConfig | JPA Auditing 활성화 |
| | QueryDslConfig | JPAQueryFactory 빈 등록 |
| | KafkaProducerConfig | 멱등성 Producer (Spring Kafka 4.0) |
| | KafkaConsumerConfig | 수동 ACK Consumer |
| | SecurityConfig | 헤더 기반 인증, Stateless |
| | FeignConfig | 로깅, 에러 디코더 |
| | SwaggerConfig | OpenAPI 문서화 |
| | AsyncConfig | ThreadPoolTaskExecutor |
| `infrastructure/` | BaseEntity | Audit + Soft Delete |
| | AuditorAwareImpl | JPA Auditing 사용자 정보 |
| `security/` | UserPrincipal | UserDetails 구현체 |
| | HeaderAuthenticationFilter | Gateway 헤더 → SecurityContext |
| | SecurityContextUtil | 현재 사용자 조회 유틸리티 |
| `feign/` | FeignErrorDecoder | HTTP 에러 → BusinessException |
| | FeignRequestInterceptor | 인증 헤더 전파 |
| `aop/` | LoggingAspect | 요청/응답 로깅 |

---

## 인증 아키텍처

```
Client → Gateway → JWT 검증 → 헤더 주입 → 내부 서비스
                                │
                                ▼
                    X-User-Id, X-User-Role, X-User-Email
```

- **Gateway**: JWT 토큰 검증 후 사용자 정보를 헤더로 전달
- **내부 서비스**: 헤더만 신뢰, JWT 검증 없음
- **HeaderAuthenticationFilter**: 헤더 → SecurityContext 변환

---

## 이벤트 기반 통신

### Kafka 토픽

| 토픽 | Producer | Consumer | 설명 |
|------|----------|----------|------|
| `user.created` | User | Auth | 회원가입 → 계정 생성 |
| `user.deleted` | User | All | 회원탈퇴 → 연관 데이터 정리 |
| `account.balance.changed` | Account | Ledger | 잔액 변경 기록 |
| `transfer.debit.requested` | Transfer | Account | SAGA 출금 요청 |
| `transfer.credit.requested` | Transfer | Account | SAGA 입금 요청 |
| `transfer.completed` | Transfer | Ledger | 이체 완료 기록 |

### IntegrationEvent 구조

```java
public record IntegrationEvent(
    String eventId,           // 멱등성 보장
    String eventType,         // 이벤트 타입
    int sequenceNumber,       // 순서 보장
    LocalDateTime timestamp,  // 발생 시각
    LocalDateTime expiresAt,  // TTL
    int retryCount,           // 재시도 횟수
    Object payload            // 실제 데이터
) {}
```

---

## 빠른 시작

### 사전 요구사항

- Java 21
- Docker & Docker Compose
- Gradle 8.x

### 실행 순서

```bash
# 1. 인프라 실행 (Kafka, PostgreSQL, Zipkin)
cd infrastructure
docker-compose up -d

# 2. Eureka Server 실행 (서비스 디스커버리)
cd eureka-server
./gradlew bootRun

# 3. Config Server 실행 (설정 서버)
cd config-server
./gradlew bootRun

# 4. Gateway Server 실행
cd gateway-server
./gradlew bootRun

# 5. 비즈니스 서비스 실행
cd user-service && ./gradlew bootRun &
cd auth-server && ./gradlew bootRun &
cd account-service && ./gradlew bootRun &
# ... 나머지 서비스
```

---

## 구현 진행 상황

### ✅ 완료

- [x] 프로젝트 초기 구조 설정 (10개 서비스)
- [x] common-lib 구현 (IntegrationEvent, UuidUtils, 113개 테스트)
- [x] 인프라 서버 기본 설정 (Eureka, Config, Gateway)
- [x] 비즈니스 서비스 Global 레이어 (110개 파일)
    - JPA, Kafka, Security, Feign, Swagger, Async 설정
    - BaseEntity (Soft Delete), JPA Auditing
    - 헤더 기반 인증 필터
    - 로깅 AOP

### 🔜 예정

- [ ] Auth Server JWT 구현
- [ ] Gateway JWT 검증 필터
- [ ] 각 서비스 도메인 로직 구현
- [ ] SAGA 패턴 구현 (Transfer Service)
- [ ] Resilience4j 적용 (Card Service)
- [ ] 통합 테스트
- [ ] Docker Compose 전체 구성

---

## 관련 저장소

| 저장소 | 설명 |
|--------|------|
| [config-repo](https://github.com/jun-bank/config-repo) | 설정 파일 저장소 |
| [infrastructure](https://github.com/jun-bank/infrastructure) | Docker Compose, 인프라 설정 |

---

## 라이선스

MIT License