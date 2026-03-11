# 🧑‍💻 MSA 기반 코딩 온라인 저지 플랫폼

> **단순한 채점 사이트가 아닌, 분산 시스템 설계를 직접 구현한 프로젝트입니다.**  
> 채점의 신뢰성, 실시간성, 확장성을 동시에 달성하기 위해 고민한 흔적을 담았습니다.


### 앵커 링크 | 빠르게 핵심만 보기

- [💡 기술적 설계 포인트로 이동하기!](#-기술적-설계-포인트)
- [🗺️ 아키텍처 사진으로 이동하기!](#️-시스템-아키텍처)
- [💻 프로젝트 코드 보기!](#-코드)
- [📀 시연 영상 보기!](#직접-만든-시스템이-어떻게-동작하는지-영상으로-확인해보세요)

<br />

-----

### 더 자세히 보기

> 기술적으로 더 자세한 디테일과 더 많은 자료 및 시연 영상을 보실 수 있습니다.

- <a href="https://github.com/dm-judge/.github/docs/infra.md" target="_blank">🔗 채점 서버의 기술적 원리</a>

- <a href="https://github.com/dm-judge/.github/docs/outbox.md" target="_blank">🔗 LLM 자동 문제 생성기의 원리와 SAGA 패턴이 어떻게 사용되었는가 </a>

- <a href="https://github.com/dm-judge/.github/docs/outbox.md" target="_blank">🔗 Outbox 패턴에 대한 기술적 분석 </a>

- <a href="https://github.com/dm-judge/.github/docs/demos.md" target="_blank">🔗 재채점, 부분 점수 등 다양한 시연 영상</a>

<br />

---


### 핵심 기술 키워드

`# MSA`, `# Kafka`, `# Outbox Pattern`, `# SAGA Pattern`, `# 보상 트랜잭션`, `# 비동기 이벤트 기반처리`, `# 아키텍쳐`, `# 멱등성(Idempotency)`, `# 메시징 큐`, `# 실시간 데이터 처리`


## 📌 프로젝트 소개

알고리즘 문제를 풀고 채점할 수 있는 **온라인 저지 플랫폼**입니다.  
BOJ, Codeforces 등의 서비스에서 영감을 받아, 직접 MSA 아키텍처로 설계·구현했습니다.

**실제 서비스 수준의 신뢰성과 일관성**을 목표로 메시지 큐, 분산 트랜잭션, 멱등성 설계까지 고려했습니다.

![Problem Example](./images/문제%201-2.png)

<br/>

## ✨ 주요 기능

| 기능 | 설명 |
|------|------|
| 🌐 **다국어 채점** | Python, C++, Java, Kotlin 등 다양한 언어 채점 지원 |
| 🎯 **서브태스크 & 부분점수** | 테스트케이스를 그룹화하여 부분점수 획득 가능 |
| ⚡ **실시간 채점 현황** | SSE 기반으로 채점 진행 상황을 실시간으로 확인 |
| 📊 **상세 채점 결과** | 테스트케이스별 결과 개별 확인 가능 |
| 🤖 **LLM 문제 자동 생성** | AI가 코딩 문제를 자동으로 생성, 실패 시 자동 롤백 |

<br/>

## 🏗️ 시스템 아키텍처

![Architectures](./images/architectures.png)

### 통신 방식

```
동기 내부 통신  → gRPC          (초저지연 서비스 간 직접 호출)
비동기 이벤트   → Kafka         (이벤트 드리븐, 서비스 간 결합도 최소화)
채점 큐        → RabbitMQ      (한 번에 하나의 채점 요청 보장)
실시간 푸시    → Redis Pub/Sub  (채점 상태 브로드캐스트)
```

<br/>

## 💡 기술적 설계 포인트

- ⚙️ 채점 큐 설계 — RabbitMQ + Redis
- 📨 서비스 간 통신 분리 — gRPC, Kafka
- 📦 Outbox 패턴 — Kafka 이벤트 유실 방지
- ✅ 멱등성 설계 — Exactly-Once 처리
- 🔄 분산 트랜잭션 — SAGA 패턴 (LLM 문제 생성), 보상 트랜잭션

---

### ⚙️ 채점 큐 설계 — RabbitMQ + Redis

> 채점 요청을 단순 HTTP API가 아닌 메시지 큐로 처리합니다.

- **RabbitMQ**로 Submit Server → Judge Server 간 채점 요청을 큐에 적재
- Judge Server는 **한 번에 하나의 채점만 처리** (동시 실행으로 인한 채점 중 리소스 부족 방지)
- 채점 진행 상황은 **Redis Pub/Sub**으로 발행 → Realtime Server가 구독 → SSE로 클라이언트에 스트리밍
- 채점 완료 이벤트는 **Kafka**로 발행하여 결과 저장을 다른 서비스에 위임 (채점 서버는 채점에만 집중)

---

### 📨 서비스 간 통신 분리 — gRPC + Kafka

> 동기/비동기 통신을 목적에 맞게 구분했습니다.

- **동기 통신 → gRPC**: 즉각적인 응답이 필요한 내부 호출에 사용. Protobuf 직렬화로 REST 대비 초저지연 달성
- **비동기 통신 → Kafka**: 결과 저장, 알림 등 즉시 응답이 불필요한 이벤트 처리에 사용. 각 서비스는 자신의 역할에만 집중

---

### 🔄 분산 트랜잭션 — SAGA 패턴 (LLM 문제 생성)

> LLM 문제 생성은 여러 서비스에 걸친 복잡한 작업입니다.

LLM 문제 생성 플로우는 여러 서비스를 거치는 복잡한 작업입니다.  
중간 단계에서 실패가 발생하면 **보상 트랜잭션(Compensating Transaction)** 이벤트를 발행하여 이전 단계의 작업을 자동으로 롤백합니다.

```
문제 생성 이벤트 발행 
  → LLM 호출
  → 문제 생성
  → 테스트케이스 생성
  → LLM 호출 성공 상태 저장
  → 문제 생성 완료 Kafka 이벤트 발행
  → 문제 저장 
  → 테스트케이스 저장 (실패!)
      ↓
  보상 트랜잭션 발행
  → 문제 삭제 (롤백)
  → LLM 호출 실패 상태 저장 (롤백)
```

---

### 📦 Outbox 패턴 — Kafka 이벤트 유실 방지

> DB 커밋과 Kafka 발행 사이의 크래시를 대비합니다.

- 이벤트를 Kafka에 직접 발행하지 않고, **DB의 Outbox 테이블에 먼저 저장**
- 별도 프로세스가 Outbox를 읽어 Kafka로 발행 후 삭제
- DB 트랜잭션과 이벤트 발행의 원자성을 보장하여 **이벤트 유실 Zero**

---

### ✅ 멱등성 설계 — Exactly-Once 처리

> 네트워크 재시도로 인한 중복 처리를 방지합니다.

- 모든 메시지는 **고유 메시지 ID**를 포함하도록 설계
- 메시지 유형에 따라 **Inbox 패턴** 적용 — 수신 측에서 처리 여부를 DB에 기록하여 중복 실행 차단
- 주로 Delta 기반 이벤트(예시: 포인트 100 추가)에 **Inbox 패턴**을 집중적으로 적용**


<br />
<br />

### ❓ 기술적으로 더 보고 싶으면..?

- LLM 문제 생성기 원리와 SAGA 패턴의 사용은 [여기](https://github.com/dm-judge/.github/docs/saga.md)를 클릭해주세요.
- Outbox 패턴의 사용과 기술적 분석은 [여기](https://github.com/dm-judge/.github/docs/outbox.md)를 클릭해주세요.
- 실시간 채점 서버의 기술적 원리와 인프라 구성은 [여기](https://github.com/dm-judge/.github/docs/infra.md)를 클릭해주세요.

## 📂 코드

> 모든 코드는 공개돼 있고, 각 서비스는 독립적인 레포지토리로 관리됩니다.

| 서비스 | 설명 | 링크 |
|--------|------|------|
| Auth | 인증 및 사용자 서비스 | [리포지토리 이동](https://github.com/dm-judge/dm-judge-backend-auth) |
| Problem | 문제 관리 서비스 | [리포지토리 이동](https://github.com/dm-judge/dm-judge-backend-problem) |
| Submit | 제출 데이터 관리 서비스 | [리포지토리 이동](https://github.com/dm-judge/dm-judge-backend-submit) |
| Realtime | SSE 스트리밍 서비스 | [리포지토리 이동](https://github.com/dm-judge/dm-judge-backend-realtime) |
| Judge | 채점 서버 | [리포지토리 이동](https://github.com/dm-judge/dm-judge-backend-judge) |
| Inference | LLM 문제 생성 파이프라인 | [리포지토리 이동](https://github.com/dm-judge/dm-judge-inference) |
| Gateway | Spring Cloud Gateway | [리포지토리 이동](https://github.com/dm-judge/dm-judge-backend-gateway) |
| Flyway | 데이터베이스 스키마 중앙 관리 | [리포지토리 이동](https://github.com/dm-judge/dm-judge-database-flyway) |
| Infra | 기타 인프라 Docker Compose | [리포지토리 이동](https://github.com/dm-judge/dm-judge-database) |

<br/>

---

<div align="center">


#### 직접 만든 시스템이 어떻게 동작하는지 영상으로 확인해보세요.

#### 더 다양한 시연 영상은 <a href="https://github.com/dm-judge/.github/docs/demos.md" target="_blank">여기</a>를 클릭해주세요.


![Judge Demo](./images/default.gif)




</div>
