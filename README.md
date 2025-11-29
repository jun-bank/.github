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

## 핵심 학습 주제

### 1. 분산 트랜잭션

단일 DB에서는 ACID가 보장되지만, MSA에서는 서비스마다 DB가 분리됩니다.  
계좌 이체 시 출금과 입금이 서로 다른 서비스에서 일어날 때, 어떻게 일관성을 보장할까요?

- **SAGA 패턴**: 보상 트랜잭션을 통한 Eventually Consistent
- **Outbox 패턴**: 이벤트 발행의 원자성 보장

### 2. 동시성 제어

같은 계좌에 동시에 여러 거래가 발생하면 잔액이 꼬일 수 있습니다.

- **낙관적 락**: Version 필드를 통한 충돌 감지
- **비관적 락**: DB 레벨 잠금
- **분산 락**: Redis를 활용한 서비스 레벨 잠금

### 3. 멱등성

네트워크 장애로 같은 요청이 중복 전송될 수 있습니다.  
같은 이체 요청이 두 번 처리되면 안 됩니다.

- **Idempotency Key**: 요청별 고유 키로 중복 방지

### 4. 장애 대응

일부 서비스가 죽어도 전체 시스템은 동작해야 합니다.

- **Circuit Breaker**: 장애 전파 차단
- **Retry + Backoff**: 일시적 장애 복구
- **Fallback**: 대체 로직 실행

---

## 도메인 구성

```
Jun Bank
├── Account Service      # 계좌 관리
├── Transaction Service  # 입출금 처리  
├── Transfer Service     # 이체 (SAGA 오케스트레이터)
├── Card Service         # 카드 결제
└── Ledger Service       # 원장 기록 (Append-only)
```

---

## 기술 스택

| 구분 | 기술 |
|------|------|
| Language | Java 21 |
| Framework | Spring Boot 4.0, Spring Cloud |
| Database | PostgreSQL, MySQL |
| Messaging | Apache Kafka (KRaft) |
| Container | Docker, Docker Compose |
| Auth | Spring Authorization Server |
| Monitoring | Prometheus, Grafana |
| Logging | ELK Stack |

---

## 저장소 목록

### 인프라

| 저장소 | 설명 |
|--------|------|
| [infrastructure](https://github.com/jun-bank/infrastructure) | Docker Compose, 인프라 설정 |
| [api-gateway](https://github.com/jun-bank/api-gateway) | API 라우팅, 인증 필터 |
| [config-server](https://github.com/jun-bank/config-server) | 설정 중앙화 |
| [eureka-server](https://github.com/jun-bank/eureka-server) | 서비스 디스커버리 |
| [auth-server](https://github.com/jun-bank/auth-server) | OAuth2 인증 서버 |

### 비즈니스 서비스

| 저장소 | 설명 | 핵심 학습 |
|--------|------|----------|
| [account-service](https://github.com/jun-bank/account-service) | 계좌 관리 | 낙관적 락, 동시성 |
| [transaction-service](https://github.com/jun-bank/transaction-service) | 입출금 처리 | 멱등성, Outbox |
| [transfer-service](https://github.com/jun-bank/transfer-service) | 계좌 이체 | SAGA 패턴 |
| [card-service](https://github.com/jun-bank/card-service) | 카드 결제 | 한도 관리 |
| [ledger-service](https://github.com/jun-bank/ledger-service) | 원장 기록 | 불변 데이터, 감사 추적 |