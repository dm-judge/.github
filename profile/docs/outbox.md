# 📦 Outbox 패턴

> Kafka 이벤트 발행과 데이터베이스 트랜잭션 사이의 정합성을 보장하기 위해 Outbox 패턴을 도입했다.

<br/>

## 목차

1. [💡 도입 배경](#1-도입-배경)
2. [⚙️ 동작 원리](#2-동작-원리)
3. [🔍 장애 케이스 분석](#3-장애-케이스-분석)
4. [💻 코드 예시](#4-코드-예시)

---

## 1. 도입 배경

Kafka 이벤트 발행은 데이터베이스 트랜잭션의 범위 밖에서 일어난다.

즉, 아래와 같은 상황이 발생할 수 있다.

```
DB 트랜잭션 커밋 성공
        ↓
Kafka 이벤트 발행 실패 ← 여기서 크래시
        ↓
다른 서비스들은 이벤트를 받지 못해 후속 처리 누락
```

이를 방지하기 위해 **Outbox 테이블**을 도입하여, Kafka 이벤트 발행을 DB 트랜잭션 범위 안으로 끌어들였다.

---

## 2. 동작 원리

```
[서비스 트랜잭션]
  비즈니스 로직 실행
  └── Outbox 테이블에 이벤트 삽입 (status: PENDING)
  └── 트랜잭션 커밋

[별도 Scheduler / Polling]
  PENDING 이벤트 조회
  └── Kafka 발행 성공 → status: PUBLISHED
  └── Kafka 발행 실패 → status: FAILED (이후 수동 처리 또는 자동 재처리)
```

핵심은 **이벤트 발행 의도 자체를 DB 트랜잭션 안에 기록**하는 것이다.  
서비스 로직이 롤백되면 Outbox 삽입도 함께 롤백되고, 커밋이 성공해야만 Scheduler가 이벤트를 Kafka로 전달한다.

---

## 3. 장애 케이스 분석

### Case 1 — 서비스 트랜잭션 자체에서 오류 발생

- 트랜잭션이 롤백되므로 비즈니스 로직 변경 사항과 Outbox 삽입 모두 취소된다.
- Kafka에는 아무것도 발행되지 않는다.
- 사용자는 API 오류 응답을 받는다.

✅ **정합성 유지됨**

---

### Case 2 — Outbox 삽입 시 오류 발생

- 같은 트랜잭션 내에서 오류가 발생하므로 전체 롤백된다.
- Case 1과 동일하게 처리된다.

✅ **정합성 유지됨**

---

### Case 3 — Scheduler의 Kafka 발행 실패

- Outbox 이벤트를 처리하는 트랜잭션 안에서 예외를 잡아 Outbox 데이터를 `FAILED`로 변경 후 커밋한다.
- Kafka에는 이벤트가 전달되지 않고, Outbox 테이블에 `FAILED` 상태로 남는다.
- 이후 수동 또는 자동 재처리를 통해 복구할 수 있다.

✅ **이벤트 유실 없이 추적 가능**

---

### Case 4 — Kafka 발행은 성공했지만, Outbox 상태 업데이트 실패

이 케이스가 가장 까다롭다.

- Kafka 발행은 성공했으나, 이후 DB 트랜잭션이 롤백되어 Outbox 데이터가 여전히 `PENDING` 상태로 남는다.
- Scheduler가 이 이벤트를 다시 조회하여 Kafka에 **중복 발행**될 수 있다.
- 이는 Kafka의 **at-least-once** 특성상 자연스러운 상황이다.

⚠️ **중복 발행 가능 → 수신 측에서 멱등성(idempotency) 보장 필요**

> 이를 해결하기 위해, 이벤트를 소비하는 서비스에서는 **Inbox 패턴** 또는 고유 메시지 ID 기반의 중복 처리 방어 로직을 적용하여 exactly-once 처리를 보장한다.

---

## 4. 코드 예시

### 서비스 레이어 — Outbox 삽입

아래는 문제를 생성하는 서비스 함수다.  
`@Transactional` 범위 안에서 `OutboxService`를 호출하여, 이벤트 발행 의도를 DB 트랜잭션과 함께 묶는다.

```kotlin
@Transactional
fun create(
    userId: String,
    request: CreateProblemRequest
): ProblemResponse {
    problemPolicy.checkCanCreate()

    val tags = tagQueryRepository.findByIds(request.tagIds)

    val command = ProblemCreateCommand.of(
        request = request,
        tags = tags,
        userId = userId
    )
    val problem = Problem.create(command)

    val saved = problemRepository.save(problem)
    problemRepository.flush()

    outboxService.publish(
        aggregateId = userId,
        eventType = KafkaTopics.PROBLEM_CREATED,
        payload = ProblemCreatedEvent(
            problemId = saved.id,
            problemNumber = saved.problemNumber,
            userId = userId,
        )
    )

    return ProblemResponse.from(saved)
}
```

---

### Scheduler — PENDING 이벤트 폴링 및 Kafka 발행

서비스 로직과 별개로, Scheduler가 1초마다 Outbox 테이블을 폴링하여 `PENDING` 이벤트를 Kafka로 발행한다.

```kotlin
@Component
class OutboxPollingScheduler(
    private val eventPublisher: OutboxEventPublisher,
) {
    private val log = LoggerFactory.getLogger(javaClass)

    @Scheduled(fixedDelay = 1000)
    fun publishPendingEvents() {
        try {
            eventPublisher.publishPendingEvents(100)
        } catch (e: Exception) {
            log.error("Outbox polling failed", e)
        }
    }
}
```

---

### 동시성 제어 — `FOR UPDATE SKIP LOCKED`

여러 서버 인스턴스가 동시에 Scheduler를 실행할 경우, 같은 Outbox 이벤트를 중복으로 처리할 수 있다.  
이를 방지하기 위해 `FOR UPDATE SKIP LOCKED`를 적용했다.

```kotlin
interface OutboxRepository : JpaRepository<OutboxEvent, Long> {
    @Query(
        value = """
            SELECT *
            FROM outbox_events
            WHERE status = 'PENDING'
            ORDER BY created_at
            LIMIT :batchSize
            FOR UPDATE SKIP LOCKED
        """,
        nativeQuery = true
    )
    fun findPendingForUpdate(
        @Param("batchSize") batchSize: Int
    ): List<OutboxEvent>
}
```

| 옵션 | 역할 |
|------|------|
| `FOR UPDATE` | 조회된 row에 row-level lock을 걸어 다른 트랜잭션의 수정을 차단한다 |
| `SKIP LOCKED` | 이미 다른 Worker가 lock을 잡은 row는 대기 없이 건너뛰어, 여러 인스턴스가 효율적으로 분산 처리한다 |