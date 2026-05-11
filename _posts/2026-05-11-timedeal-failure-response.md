---
layout: post
title: "타임딜 시스템 설계 — 장애를 어떻게 대응할까?"
summary: "선착순 한정 판매 시스템의 전체 설계를 살펴보면서 어떤 장애를 막아야 하고 어떤 정합성을 보장해야 할지 짚어본 글입니다."
author: woodypark
date: '2026-05-11 21:46:12 +0900'
category: architecture
thumbnail: /assets/img/posts/server-err.jpg
keywords: msa, distributed system, eventual consistency, outbox pattern, idempotency, redis failover, saga orchestration, reconciliation, dlq, 분산 시스템, 정합성, 장애 복구, 타임딜, 선착순
permalink: /blog/timedeal-failure-response/
---

---

[이전 글](/blog/pre-design-failure-scenarios/)에서는 본격 설계에 들어가기 전에 "뭐가 터질 수 있는지부터 상상해보자"라는 이야기를 적었습니다. 그 사이 설계가 끝났고, 이제 그 위에서 **데이터 정합성을 보장하고 장애에 대응하는 부분**을 풀어보려 합니다.

---

### 어떤 시스템인가

- 100만 명이 정오에 몰리는 **선착순 한정 판매** 커머스
- 재고 1,000족, 주문서 진입 후 5분 안에 결제 안 되면 재고 환원
- 사용자는 10분 안에 성공/실패 피드백을 받아야 함
- MSA 기반 (User, Order, Inventory, Payment 분리, 각자 독립 DB)
- 인프라: Kubernetes, Kafka(메시지 브로커), Redis(인메모리 스토어), 외부 PG API

### 전체 설계 요약

본격적인 설계를 거치며 시스템은 이렇게 잡혔습니다.

**대기열**: Redis Master-Replica + Sorted Set 기반으로 100만 명의 진입을 받아냅니다. 클러스터를 분할하지 않은 이유는 선착순 정합성 때문이고요. 사용자에게는 가변 Polling(뒤쪽 대기자는 10초, 입장 직전은 1초)으로 순번을 알려주고, 통과한 사용자에게는 JWT 입장 토큰을 발급해 주문 API로 흘려보냅니다.

**재고 동시성 제어**: Redisson Pub/Sub 기반 분산 락으로 동시 차감을 막습니다. 재고는 즉시 차감하지 않고 5분 동안 `RESERVED` 상태로 가점유한 뒤, 결제가 성공하면 `SOLD`로 확정하고, 실패하거나 5분 안에 결제가 끝나지 않으면 `AVAILABLE`로 복구하는 흐름이고요. 결제 Timeout은 즉시 실패로 처리하지 않고 `PENDING_PAYMENT_CONFIRM` 상태로 두면서 Exponential Backoff로 PG에 결제 결과를 재조회합니다.

**주문-결제 분산 트랜잭션**: Saga Orchestration을 채택했습니다. `OrderSagaOrchestrator`가 주문별 진행 상태를 관리하고, 각 단계는 로컬 트랜잭션 + Outbox 발행으로 묶입니다. 외부 PG 호출은 절대 내부 DB 트랜잭션 안에서 하지 않고, 결제 API는 `202 Accepted`로 즉시 반환한 뒤 사용자는 polling/SSE로 최종 결과를 확인합니다. PG Timeout은 `FAILED`가 아니라 `UNKNOWN`으로 두고 상태 조회로 확정하는 보수적 정책을 채택했고요.

> **📎 상세 설계는 다음 글들에서 확인할 수 있습니다.**
> - 대기열 설계 — [Part 1-1](https://velog.io/@suhyun1257/MSA-환경에서-대기열-시스템-설계하기-part-1-1) · [Part 1-2](https://velog.io/@suhyun1257/MSA-환경에서-대기열-시스템-설계하기-part-1-2)
> - 재고 동시성 제어 — [시스템 설계 - 타임딜 커머스에서 재고 동시성 제어](https://velog.io/@lilloo04/시스템-설계-타임딜-커머스에서-재고-동시성-제어)
> - 주문-결제 Saga — [커머스 도메인 주문-결제 분산 트랜잭션 설계](https://root-2707.tistory.com/10)

이제 이 시스템을 살펴보면서, 어떤 장애를 막아야 하고 어떤 정합성을 보장해야 할지 하나씩 짚어보겠습니다.

---

### 1. 메시지를 발행하는 곳이 곳곳에 있습니다

분산 시스템 정합성을 짚을 때 가장 먼저 봐야 할 게 **"서비스들 사이에 무엇이 오가는가"** 입니다. 우리 시스템은 MSA니까 주문/결제/재고가 분리돼 있고, 동기 호출이 아니라 Kafka 메시지로 협력하기로 설계됐습니다. 메시지가 시스템의 혈관 역할인 셈인데, 혈관 하나가 막히면(메시지 한 건이 유실되면) 시스템이 마비됩니다. 그래서 첫 분석은 "어디서 어떻게 메시지가 발행되는지 짚는 일"이 됩니다.

시스템 전체를 펼쳐놓고 보면, **메시지 발행 지점이 한두 군데가 아니라는 게** 가장 먼저 눈에 들어옵니다. 떠올려보면 이런 곳들이 있습니다.

- 대기열을 통과한 사용자에게 토큰을 발급하고 주문 서비스로 흘려보낼 때
- Saga Orchestrator가 `ReserveInventory`, `PaymentApproveRequested`, `ConfirmReservation` 같은 명령을 내릴 때
- 결제 서비스가 PG 응답을 받고 `PaymentApproved` 또는 `PaymentFailed` 이벤트를 발행할 때
- 재고 서비스가 `InventoryReserved`, `InventoryConfirmed` 같은 후속 이벤트를 발행할 때
- 5분 만료나 결제 실패로 인한 `ReleaseReservation` 보상 이벤트를 발행할 때

거의 모든 상태 전이가 메시지 발행과 함께 일어나는 구조입니다. 한 주문이 끝까지 가는 동안 거치는 메시지가 십수 건이 되는 셈입니다. 그래서 결론은 단순합니다: **모든 메시지 발행 지점이 반드시 "발행 보장"을 해야 합니다.**

#### 여기서 일어날 수 있는 일 — Dual-Write 문제

가장 단단해 보이는 이 코드가 사실 위험합니다.

```java
orderRepository.save(order);                // ① DB 커밋
kafkaTemplate.send("order-created", ...);   // ② Kafka 발행
```

이유는 ①과 ②가 **서로 다른 시스템**이라는 점입니다.

- ①은 **DB의 트랜잭션**
- ②는 **Kafka의 트랜잭션**
- 애플리케이션 코드가 이 둘을 *순서대로* 호출하긴 하지만, **하나의 원자적 트랜잭션으로 묶지 못합니다**

그래서 이런 일이 벌어집니다.

<table style="width: 100%; border-collapse: collapse; margin: 20px 0;">
<thead>
<tr style="background-color: #2d2d2d; color: #fff;">
<th style="padding: 12px; border: 1px solid #444; text-align: left;">시나리오</th>
<th style="padding: 12px; border: 1px solid #444; text-align: left;">결과</th>
</tr>
</thead>
<tbody>
<tr style="background-color: #1e1e1e;">
<td style="padding: 12px; border: 1px solid #444;">① 성공 → 서버 다운 → ② 실행 못 함</td>
<td style="padding: 12px; border: 1px solid #444;">DB엔 주문 있는데 결제 서비스는 모름 → <strong>유령 주문</strong></td>
</tr>
<tr style="background-color: #252525;">
<td style="padding: 12px; border: 1px solid #444;">② 성공 → 서버 다운 → ① 실행 못 함 (순서 뒤집어도)</td>
<td style="padding: 12px; border: 1px solid #444;">Kafka엔 이벤트인데 DB엔 주문 없음</td>
</tr>
<tr style="background-color: #1e1e1e;">
<td style="padding: 12px; border: 1px solid #444;">② 호출했지만 ack 못 받음 → Producer 재시도</td>
<td style="padding: 12px; border: 1px solid #444;">같은 메시지 두 번 발행 → <strong>재고 두 번 차감 위험</strong></td>
</tr>
</tbody>
</table>

이걸 **Dual-Write 문제**라고 부릅니다. 분산 시스템에서 가장 흔하지만, 정통으로 맞으면 가장 무서운 정합성 사고의 원천입니다. 특히 한정 판매 도메인에서는 메시지 한 건의 유실이 곧 **유령 주문 또는 초과 판매**라는 비즈니스 사고로 직결됩니다.

#### 어떻게 대응할 수 있을까

핵심 발상은 한 줄로 표현하면:

> **"메시지를 발행한다"를 "메시지를 발행하겠다는 의도를 DB에 기록한다"로 바꾼다.**

즉, Kafka 호출을 비즈니스 트랜잭션 *밖*으로 빼고, 그 안에 **Outbox 테이블 Insert를 같이 묶습니다**.

```sql
BEGIN;
  -- 비즈니스 변경
  UPDATE orders SET status = 'PAYMENT_IN_PROGRESS' WHERE id = :orderId;
  -- 보낼 메시지 기록
  INSERT INTO outbox (event_id, aggregate_id, topic, payload, occurred_at, status)
  VALUES (:eventId, :orderId, 'payment-approve-requested', :payload, NOW(), 'PENDING');
COMMIT;
```

이렇게 하면:

- ✅ **비즈니스 변경 + 메시지 발행 의도가 같은 트랜잭션** → 원자성 보장
- ✅ **서버가 죽어도 DB에 기록이 남음** → 복구 가능
- ✅ **별도 Relay가 Outbox를 읽어 Kafka로 발행** → 비즈니스 로직과 분리

이때 의미 자체가 바뀝니다. **"Kafka 발행 성공 = 메시지 보낸 것"** 이 아니라 **"Outbox 커밋 성공 = 메시지를 보내겠다는 약속이 영구 기록된 것"** 입니다. 발행은 Relay가 책임지고 끝까지 재시도하는 셈입니다.

**Relay 방식 — 폴링 vs CDC**

<table style="width: 100%; border-collapse: collapse; margin: 20px 0;">
<thead>
<tr style="background-color: #2d2d2d; color: #fff;">
<th style="padding: 12px; border: 1px solid #444; text-align: left;"></th>
<th style="padding: 12px; border: 1px solid #444; text-align: left;">폴링</th>
<th style="padding: 12px; border: 1px solid #444; text-align: left;">CDC</th>
</tr>
</thead>
<tbody>
<tr style="background-color: #1e1e1e;">
<td style="padding: 12px; border: 1px solid #444;"><strong>동작</strong></td>
<td style="padding: 12px; border: 1px solid #444;">1초마다 <code>SELECT ... WHERE status = 'PENDING'</code></td>
<td style="padding: 12px; border: 1px solid #444;">Debezium이 DB binlog를 직접 스트리밍</td>
</tr>
<tr style="background-color: #252525;">
<td style="padding: 12px; border: 1px solid #444;"><strong>지연</strong></td>
<td style="padding: 12px; border: 1px solid #444;">초 단위</td>
<td style="padding: 12px; border: 1px solid #444;">밀리초 단위</td>
</tr>
<tr style="background-color: #1e1e1e;">
<td style="padding: 12px; border: 1px solid #444;"><strong>운영 부담</strong></td>
<td style="padding: 12px; border: 1px solid #444;">추가 스택 없음</td>
<td style="padding: 12px; border: 1px solid #444;">Kafka Connect 등 별도 운영 필요</td>
</tr>
</tbody>
</table>

타임딜처럼 *지연이 사용자 경험에 직접 영향을 주는* 도메인이면 **CDC 쪽이 유리해 보입니다.** 결제 응답이 1초 늦으면 사용자가 답답해하고, 재고 점유 시간이 1초 늘어나면 다음 대기자가 그만큼 늦어지니까요.

**시스템 전역 멱등성 키 규약**

Outbox는 발행 *유실*은 막아주지만, **중복 발행은 막지 못합니다**. Relay 재시도 등으로 같은 메시지가 두 번 나갈 수 있어서, 모든 메시지에 **고유 식별자(`event_id`)**를 붙여야 합니다.

Saga 설계에서 정의한 `payment:{orderId}:{attemptNo}` 형식을 시스템의 다른 메시지에도 일관되게 적용해두면 좋을 것 같아요.

- 대기열 통과 신호: `queue-pass:{token-id}`
- 재고 환원 이벤트: `release:{orderId}:{reason}`
- PG 결제 명령: `payment:{orderId}:{attemptNo}` (Saga 설계 그대로)

수신 측은 **Inbox 패턴**으로 처음 받은 `event_id`만 처리합니다. 두 번째 같은 이벤트가 도착하면 무시합니다.

```sql
INSERT INTO inbox_events (event_id, consumer, processed_at)
VALUES (:eventId, 'inventory-service', NOW())
ON CONFLICT (event_id) DO NOTHING;
-- 영향받은 row가 0이면 중복, 1이면 처음 도착
```

발신 측 Outbox + 수신 측 Inbox + 일관된 `event_id` = **"정확히 한 번 처리(effectively once)"** 효과를 얻습니다. Kafka의 at-least-once 배달이 기본 보장하지 못하는 걸 애플리케이션 레벨에서 만드는 셈입니다.

---

### 2. Redis는 비동기 복제로 굴러갑니다

시스템 인프라를 살펴보면 **Redis가 Master-Replica 구성**으로 굴러갑니다. 대기열의 Sorted Set과 분산 락(Redisson Pub/Sub)이 모두 이 위에서 동작합니다.

빠른 응답을 위한 표준 구성이지만, 한 가지 짚어야 할 특성이 있습니다. **Redis의 복제는 기본적으로 비동기**입니다.

- 마스터에 쓰자마자 클라이언트에 ack 회신
- *그 다음에* 리플리카로 데이터를 흘려보냄 (수십~100ms 지연)
- 마스터가 죽으면 리플리카가 마스터로 승격 = **페일오버**

평상시엔 이 100ms 구간이 거의 보이지 않습니다. 그런데 타임딜처럼 **초당 수만 건의 쓰기**가 마스터로 들어오는 상황에선, 그 짧은 구간이 *수천 건의 유실*로 확대될 수 있습니다.

#### 여기서 일어날 수 있는 일

**대기열 측 — 사라진 순번**

타임딜 오픈 직후 초당 수만 건의 `ZADD`가 마스터로 들어옵니다. 그중 일부가 아직 리플리카로 복제되지 못한 시점에 마스터가 갑자기 다운된다면 어떻게 될까요.

리플리카가 승격돼서 서비스는 계속되는데, **복제되지 못한 수천 명의 진입 기록은 영영 사라집니다**. 이 사용자들은 시스템에서 "원래 없던 사람"이 돼버려서, 토큰만 손에 든 채 순번 정보가 없어진 상태가 됩니다.

**분산 락 측 — 락이 두 명에게 동시에**

분산 락은 더 까다롭습니다. 락 정보 자체가 휘발됩니다. 새 마스터로 승격된 리플리카는 락 없는 상태로 받기 시작해서, *과거 마스터에서 락을 보유했던 워커가 작업을 계속하는 동시에 새로 락을 획득한 워커도 같은 재고에 접근*하는 상황이 가능합니다. Redlock 알고리즘이 단일 Redis 페일오버 상황에서 안전성을 보장하지 못한다는 [유명한 지적](https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html)이 정확히 이 지점입니다. 분산 락이 깨지면 결과적으로 **두 사용자가 같은 재고에 동시 차감**해서 초과 판매로 이어집니다.

#### 어떻게 대응할 수 있을까

**대기열 측 — 토큰 기반 재진입**

1편에서 다뤘던 토큰 기반 재진입이 메인 대응입니다. 핵심 발상은 *"Redis가 잃어버린 정보를 클라이언트가 들고 있게 한다"* 입니다.

```
[진입]
  서버: ZADD queue 1745200000123 u-12345
  서버: 토큰 발급 { user_id, original_score, signature }
  클라이언트: 토큰 보관

──── Redis 페일오버 (진입 기록 일부 유실) ────

[페일오버 후 폴링]
  클라이언트 → 토큰 첨부해서 폴링
  서버: ZRANK → null
  서버: 토큰 서명 검증 → 유효
  서버: ZADD queue original_score u-12345   ← 원래 score로 복원
  응답: "4987번째"  (사용자 입장: 아무 일 없었던 것처럼)
```

`original_score`가 토큰에 노출되니 위조 방지를 위해 **서버 측 서명 검증(HMAC, JWT 등)이 필수**입니다. 같은 토큰으로 폴링이 여러 번 와도 ZRANK가 이미 있으면 skip이라 **멱등성도 자연스럽게 보장**됩니다.

**경계 케이스 — 토큰까지 잃어버린다면?**

이 방법은 *클라이언트가 토큰을 들고 있을 때만* 작동합니다. 앱 강제 종료나 캐시 삭제로 토큰까지 사라지면 복구가 안 됩니다. 이 케이스까지 잡아야 할지는 *도메인 정책*의 문제고, 잡고 싶다면 `ZADD`와 동시에 Kafka에 `QueueEntered` 이벤트를 함께 발행해두고 페일오버 시 거기서 재적재하는 **영속 로그 병행 방식**도 있습니다. 다만 쓰기 부하와 Dual-Write 문제 재귀를 감수해야 해서, *정말 필요한 경우*에만 도입을 고려할 만합니다.

**분산 락 측 — Fencing Token**

[Fencing Token](https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html)을 같이 쓰는 방법이 자주 언급됩니다. 핵심 발상은 *"작업 받는 쪽이 마지막에 한 번 더 검증한다"* 입니다.

작동 원리는 단순합니다.

- 락 발급마다 1씩 증가하는 토큰을 같이 줍니다 (`42, 43, 44, ...`)
- 작업할 때(예: 재고 차감) 그 토큰을 항상 첨부합니다
- **재고 DB는 "마지막으로 본 토큰 번호"를 기억**해두고, *자기보다 작은 토큰이 들어오면 거부*합니다

시나리오로 보면 (Worker A의 작업이 *in-flight*인 사이에 페일오버가 일어나는 케이스):

```
[Worker A 락 획득]
  토큰 = 42
  Worker A: 재고 차감 요청 송신...
       (네트워크 지연/GC 멈춤 등으로 DB에 늦게 도착 중)

──── Redis 페일오버 (Worker A의 락 정보 휘발) ────

[Worker B 같은 락 새로 획득]
  토큰 = 43   ← 새로 발급된 거라 항상 더 큼
  Worker B → 재고 DB에 차감 (토큰 43)
  DB: 적용됨. last_token = 43

[Worker A의 늦은 요청이 드디어 DB 도착]
  Worker A → 재고 DB에 차감 (토큰 42)
  DB: A의 토큰(42) < 이미 본 last_token(43)
  → "이미 더 최신 작업이 다녀갔어" → 거부 ✓
```

SQL로 표현하면 UPDATE 문에 토큰 비교를 같이 두는 형태입니다.

```sql
UPDATE inventory
SET ..., last_fencing_token = :myToken
WHERE last_fencing_token < :myToken;
```

이렇게 두면 페일오버로 같은 락을 두 명이 잡았더라도, **항상 더 큰 토큰을 받은 쪽만 작업이 인정**돼서 동시 차감이 막힙니다.

**그럼 거부된 Worker A 측 사용자는?**

A의 차감이 거부되면 시스템이 실패를 감지해 *내부에서 자동 재시도*해야 합니다. 재시도 시점에 재고가 남아 있으면 정상 진행되고, 이미 매진이라면 "매진" 안내가 사용자에게 갑니다. *페일오버 직후의 일부 사용자가 손해를 볼 수 있는* 트레이드오프인데, 이 도메인에선 **초과 판매가 훨씬 큰 사고**라 이 쪽의 안전을 우선해야한다고 판단을 했습니다.

---

### 3. 진행 중 상태가 사라지면 어떻게 될까요

시스템에 "주문 한 건이 끝까지 가도록 추적하는" 컴포넌트들이 몇 개 있습니다. 살펴보면:

- **OrderSagaOrchestrator** — 주문별 Saga 진행 상태 관리
- **PG 재조회 Scheduler** — 5분 동안 Exponential Backoff로 PG에 결제 상태 묻기
- **타임아웃 만료 Worker** — 5분 결제 미완료 시 재고 자동 해제
- **좀비 유저 정제 Scheduler** — 대기열의 만료된 유저 정제

이들이 죽으면 어떻게 될까요? 만약 진행 중 상태를 메모리에만 두고 있었다면, 인스턴스가 다운되는 순간 그 정보가 그대로 사라집니다. 100만 명이 몰리는 한정 판매에서 "이 주문이 지금 어느 단계에 있는지" 모르면 자동 복구가 불가능해집니다.

#### 여기서 일어날 수 있는 일

- Orchestrator 인스턴스 OOM/eviction → 진행 중 Saga 정보가 휘발 → 해당 주문이 미아가 됨
- PG 재조회 Scheduler 다운 → 5분 재조회 작업 누락 → PG 응답 영영 확정 못 함
- Worker 재시작 후 어디서부터 다시 시작할지 모름 → 일부 보상 트랜잭션 누락
- 다중 인스턴스로 띄웠을 때 같은 작업 동시 수행 → 중복 처리

#### 어떻게 대응할 수 있을까

핵심 발상은 한 줄로 표현하면:

> **"진행 중 상태는 메모리에 두지 않는다."**

모든 진행 상태를 DB에 영속화하고, 다음 행동은 항상 *현재 DB 상태*에서 도출하는 방식입니다.

**Saga 상태 영속화**

모든 단계 진행 시 DB에 Saga 상태를 update합니다. 각 단계는 *로컬 트랜잭션 + Outbox 발행 + saga state update*가 하나로 묶이는 셈이고요.

```sql
-- saga_executions 테이블 예시
CREATE TABLE saga_executions (
  order_id      VARCHAR PRIMARY KEY,
  current_step  VARCHAR NOT NULL,  -- ex: 'INVENTORY_RESERVE_REQUESTED'
  status        VARCHAR NOT NULL,  -- ex: 'IN_PROGRESS', 'COMPLETED', 'COMPENSATING'
  last_event_at TIMESTAMP NOT NULL,
  retry_count   INT NOT NULL DEFAULT 0
);
```

**복구 Job**

인스턴스가 시작될 때 unfinished Saga를 자동으로 픽업합니다.

```sql
SELECT * FROM saga_executions
WHERE status = 'IN_PROGRESS'
  AND last_event_at < NOW() - INTERVAL '30 seconds'
FOR UPDATE SKIP LOCKED;
```

`FOR UPDATE SKIP LOCKED`를 쓰면 *다중 인스턴스에서 같은 Saga를 동시에 픽업하지 않게* 됩니다. 죽었던 인스턴스의 작업을 다른 인스턴스가 이어받는 효과를 얻을 수 있습니다.

전체 흐름은 이렇습니다.

```
[정상 처리 중]
  Orchestrator A: Saga 단계마다
    saga_executions 테이블 update (current_step, last_event_at)
       ↓
   ↓ 인스턴스 A 다운 ↓
       ↓
[복구 Job 동작 — 다른 인스턴스 B]
  주기적으로 SELECT WHERE status = 'IN_PROGRESS'
                    AND last_event_at < 30초 전
                    FOR UPDATE SKIP LOCKED
       ↓
  B가 픽업 → saga state의 current_step부터 이어받음
       ↓
  계속 진행 → COMPLETED
```

**Stateless Orchestrator로 한 발 더**

더 나아가서 Orchestrator 자체를 *stateless*로 설계하면, 다음 행동을 매번 *현재 DB 상태*에서 도출하게 됩니다. 인스턴스가 언제 죽었다 살아나든 *같은 입력(DB 상태) → 같은 출력*이 보장돼서 **멱등성이 자연스럽게 따라옵니다**.

**PG 재조회 Scheduler에도 같은 원리**

Part 2에서 정의한 `PENDING_PAYMENT_CONFIRM` 상태를 DB에 두고, *다음 재조회 시각*도 DB에 기록해두면 워커가 다운돼도 다른 인스턴스가 같은 큐를 픽업할 수 있습니다. Scheduler 단일 인스턴스의 SPOF가 자연스럽게 사라지는 셈입니다.

---

### 4. 그래도 새는 부분이 있을 겁니다

지금까지 짚어본 대응들 — Outbox 통합, Redis 페일오버 보강, 진행 중 상태 영속화 — 은 모두 *실시간 보장*에 초점이 맞춰져 있습니다. 그런데 분산 시스템에서 실시간 보장만으로 정합성을 100% 잡는 건 사실상 불가능합니다. 어디선가 0.1%는 새기 마련입니다.

#### 여기서 일어날 수 있는 일

- PG Webhook 자체가 유실되고 (PG 측 재발송이 없거나 우리 서버가 못 받음)
- 보상 트랜잭션(환불)을 시도했는데 환불 호출도 실패하고
- 알 수 없는 인프라 이상으로 메시지가 영영 도착 안 하고
- 멀티 인스턴스 race condition으로 중복 처리되거나 누락되고

그래서 안전망이 한 겹 더 필요합니다. 실시간으로 못 잡은 것은 *사후*에 잡고, 사후로도 못 잡은 것은 *운영자가 개입*하는 3중 구조가 분산 시스템 정합성의 마지막 보루더라고요.

#### 어떻게 대응할 수 있을까

**정합성 검증 배치 (Reconciliation)**

타임딜 종료 후 일정 시간(예: 30분 정도?) 뒤에 전체 데이터를 대조하는 식을 떠올려볼 수 있습니다.

- Order DB의 주문 상태
- Payment DB의 결제 상태
- Inventory DB의 재고 상태
- PG 시스템의 결제 진실 (PG 상태 조회 API)

이 네 가지 데이터를 주문 ID 단위로 join해서 비교한다고 했을 때, 발견되는 불일치는 보통 두 갈래로 나뉘지 않을까 싶습니다.

<table style="width: 100%; border-collapse: collapse; margin: 20px 0;">
<thead>
<tr style="background-color: #2d2d2d; color: #fff;">
<th style="padding: 12px; border: 1px solid #444; text-align: left;">불일치 케이스</th>
<th style="padding: 12px; border: 1px solid #444; text-align: left;">보정 방향</th>
</tr>
</thead>
<tbody>
<tr style="background-color: #1e1e1e;">
<td style="padding: 12px; border: 1px solid #444;">결제는 <code>APPROVED</code>인데 주문은 <code>PAYMENT_IN_PROGRESS</code></td>
<td style="padding: 12px; border: 1px solid #444;">자동: 주문 완료 처리</td>
</tr>
<tr style="background-color: #252525;">
<td style="padding: 12px; border: 1px solid #444;">주문은 <code>EXPIRED</code>인데 결제는 <code>APPROVED</code></td>
<td style="padding: 12px; border: 1px solid #444;">자동: 환불 보상 트리거</td>
</tr>
<tr style="background-color: #1e1e1e;">
<td style="padding: 12px; border: 1px solid #444;">위 패턴으로 분류 안 되는 케이스</td>
<td style="padding: 12px; border: 1px solid #444;">수동: 운영 알람 + 수동 개입 큐로 이관</td>
</tr>
</tbody>
</table>

배치 주기는 종료 직후 1회 + 매일 새벽 1회 정도면 어떨까 싶습니다. 종료 직후 배치는 즉시 보정 용도로, 새벽 배치는 누락 검출 용도로 두는 식이고요.

**DLQ (Dead Letter Queue)**

메시지가 N회(예: 5회) 재시도해도 실패하면 DLQ로 이관합니다. DLQ로 들어간 메시지는 자동 재처리하지 않고 *운영 알람 발송 + 수동 처리* 큐가 됩니다.

```
[정상 처리]                  [재시도 N회 실패]
   ↓                              ↓
 처리 완료                    DLQ 이관 + 알람
                                  ↓
                             운영자 원인 파악
                                  ↓
                          수동 재처리 or 데이터 보정
```

DLQ 자체의 메시지 수가 *모니터링 지표*가 됩니다. DLQ에 평소보다 메시지가 빨리 쌓이면 시스템 어딘가가 이상하다는 신호입니다.

**모니터링 지표**

운영 안전망을 작동시키려면 평소에 시스템을 어떻게 보고 있어야 할지가 중요합니다. 한정 판매 도메인에서는 다음 지표들이 의미 있어 보입니다.

<table style="width: 100%; border-collapse: collapse; margin: 20px 0;">
<thead>
<tr style="background-color: #2d2d2d; color: #fff;">
<th style="padding: 12px; border: 1px solid #444; text-align: left;">지표</th>
<th style="padding: 12px; border: 1px solid #444; text-align: left;">이상 신호</th>
</tr>
</thead>
<tbody>
<tr style="background-color: #1e1e1e;">
<td style="padding: 12px; border: 1px solid #444;">PG <code>UNKNOWN</code> 상태 카운트</td>
<td style="padding: 12px; border: 1px solid #444;">평소보다 많으면 PG 또는 네트워크 이상</td>
</tr>
<tr style="background-color: #252525;">
<td style="padding: 12px; border: 1px solid #444;">Saga 평균 완료 시간 (p99)</td>
<td style="padding: 12px; border: 1px solid #444;">정상 범위 벗어나면 어딘가 막힘</td>
</tr>
<tr style="background-color: #1e1e1e;">
<td style="padding: 12px; border: 1px solid #444;">가점유 → SOLD 전환율</td>
<td style="padding: 12px; border: 1px solid #444;">결제 실패가 많아지면 PG/결제 서비스 이상</td>
</tr>
<tr style="background-color: #252525;">
<td style="padding: 12px; border: 1px solid #444;">DLQ 메시지 적재 속도</td>
<td style="padding: 12px; border: 1px solid #444;">0이 정상, 0이 아니면 즉시 알람</td>
</tr>
<tr style="background-color: #1e1e1e;">
<td style="padding: 12px; border: 1px solid #444;">정합성 검증 배치 불일치 건수</td>
<td style="padding: 12px; border: 1px solid #444;">0이 정상</td>
</tr>
</tbody>
</table>

이 지표들이 모두 안정 범위 안에 있을 때만 *"시스템이 잘 돌고 있다"*고 말할 수 있을 것 같습니다.

---

### 마무리

여기까지 시스템을 살펴보면서 짚어본 내용입니다. 정리하면서 몇 가지 깨달은 게 있습니다.

첫째로, 분산 시스템에서 정합성을 다루는 일은 결국 **"못 믿기"가 출발점**입니다. 메시지도, 노드도, 외부 API도 못 믿는다는 전제 위에서 설계해야 안전망이 두꺼워지더라고요.

둘째로, 1편에서 추측으로 짚었던 시나리오들이 실제 설계 위에서 구체화됐습니다. *유령 주문은 Outbox 통합으로*, *사라진 순번은 토큰 재진입으로*, *광클 중복 결제는 멱등성 키 규약*으로 어느 지점에서 어떻게 막을지가 분명해졌습니다. 거기에 1편엔 없었던 **Saga Orchestrator 영속화, Fencing Token, 정합성 검증 배치** 같은 영역도 새로 보였습니다.

셋째로, **실시간 보장으로 100%를 만들겠다는 욕심을 버리는 게 오히려 안전합니다**. 실시간으로만 잡으려다 시스템이 무거워지면 새 장애 면을 만드니까요. 사후 정합성 검증 + 운영 개입이라는 마지막 방어선을 같이 두는 편이 *결과적으로 더 견고*하더라고요.

**한 줄 요약**: 분산 시스템 정합성은 결국 *못 믿는 전제 위에 안전망을 여러 겹 쌓아 최종 일관성을 보장하는 일*이었습니다.

분산 시스템 정합성·장애 대응에 고민이 있으신 분들께 조금이라도 도움이 됐으면 좋겠습니다. 다른 접근이나 "이 부분은 저희 팀에서는 이렇게 풀었어요" 같은 경험 있으시면 댓글로 공유해주시면 너무 좋을 것 같아요!
