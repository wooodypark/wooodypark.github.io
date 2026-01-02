---
layout: post
title: "SQL 쿼리 튜닝: 실무에서 인덱스 하나 추가하기까지"
summary: "인덱스가 있어도 느린 쿼리를 해결하기 위해 실행계획 분석부터 대안 검토, 인덱스 추가까지 거친 실무 경험을 공유합니다."
author: woodypark
date: '2026-01-01 20:10:34 +0900'
category: sql
thumbnail: /assets/img/posts/sql1.jpg
keywords: sql, query tuning, index, explain, database optimization, filter, random access, production, real-world, datadog, performance tuning, execution plan
permalink: /blog/sql-query-tuning2/
---

---

### 문제 상황: 느린 RPC 응답

최근 특정 RPC(API)를 호출하는 클라이언트에서 해당 RPC의 latency가 느리다는 문제를 발견했습니다. 

단순히 느리다고 느끼는 것이 아니라, 실제로 응답 시간이 예상보다 훨씬 오래 걸리고 있었습니다.

---

### 원인 파악: Datadog으로 성능 분석

왜 느린지 확인하기 위해 해당 요청을 **Datadog(트레이싱 및 모니터링 시스템)**을 통해 확인했습니다.

분석 결과, **DB에 쿼리하는 시간이 전체 응답 시간 중 가장 많은 부분을 차지**하고 있음을 확인했습니다. 즉, 이 RPC의 병목은 데이터베이스 쿼리였습니다.

---

### 문제의 쿼리 확인

해당 RPC에서 사용하는 쿼리를 확인해보니, 만료되지 않은 로우를 가져오는 쿼리였습니다:

```sql
SELECT id, user_id, status, created_at, expired_at
FROM users
WHERE user_id = 12345
AND status = 'ACTIVE'
AND expired_at IS NULL
```

이 쿼리는 특정 사용자의 활성 상태이며 만료되지 않은 위치 데이터를 조회하는 쿼리입니다.

---

### 실행계획 확인: EXPLAIN 분석

쿼리가 느린 이유를 파악하기 위해 `EXPLAIN`을 실행해봤습니다.

```sql
EXPLAIN
SELECT id, user_id, status, created_at, expired_at
FROM users
WHERE user_id = 12345
AND status = 'ACTIVE'
AND expired_at IS NULL;
```

**EXPLAIN 결과:**

```
# id  select_type  table  type  possible_keys              key                          key_len  ref       rows  Extra
# 1   SIMPLE       users  ref   idx_user_status_createdat  idx_user_status_createdat   403      const,const  701  10  Using index condition; Using where
```

**EXPLAIN ANALYZE 결과:**

```
-> Filter: (users.expired_at is null)  (cost=182.26 rows=70) (actual time=12.153..12.155 rows=1 loops=1)
   -> Index lookup on users using idx_user_status_createdat (user_id=12345, status='ACTIVE'), with index condition: (users.status = 'ACTIVE')  (cost=182.26 rows=701) (actual time=0.658..12.116 rows=701 loops=1)
```

실행계획을 확인해보니 **인덱스를 사용하긴 하는데**, WHERE 절에서 **필터링(Filter)이 발생**하고 있었고, 이 필터링으로 인해 접근한 로우 수가 결과보다 훨씬 많았습니다.

- **인덱스 스캔**: `idx_user_status_createdat` 인덱스를 사용하여 **701개의 로우 ID**를 찾았습니다.
- **필터링**: 이 701개의 로우 각각에 대해 테이블에 접근하여 `expired_at IS NULL` 조건을 확인했습니다.
- **최종 결과**: 701개 중 **단 1개만** 조건에 만족하여 반환되었습니다.

즉, **701번의 테이블 랜덤 액세스**가 발생했지만, 실제로는 **1개의 데이터만 필요**했던 것입니다.

---

### 인덱스 구조 확인

인덱스 구조를 확인해보니 다음과 같았습니다:

```sql
SHOW INDEX FROM users WHERE Key_name = 'idx_user_status_createdat';
```

**인덱스 구조:**

<table style="width: 100%; border-collapse: collapse; margin: 20px 0;">
<thead>
<tr style="background-color: #2d2d2d; color: #fff;">
<th style="padding: 12px; border: 1px solid #444; text-align: left;">Table</th>
<th style="padding: 12px; border: 1px solid #444; text-align: left;">Key_name</th>
<th style="padding: 12px; border: 1px solid #444; text-align: left;">Seq_in_index</th>
<th style="padding: 12px; border: 1px solid #444; text-align: left;">Column_name</th>
</tr>
</thead>
<tbody>
<tr style="background-color: #1e1e1e;">
<td style="padding: 12px; border: 1px solid #444;">users</td>
<td style="padding: 12px; border: 1px solid #444;">idx_user_status_createdat</td>
<td style="padding: 12px; border: 1px solid #444;">1</td>
<td style="padding: 12px; border: 1px solid #444;">user_id</td>
</tr>
<tr style="background-color: #252525;">
<td style="padding: 12px; border: 1px solid #444;">users</td>
<td style="padding: 12px; border: 1px solid #444;">idx_user_status_createdat</td>
<td style="padding: 12px; border: 1px solid #444;">2</td>
<td style="padding: 12px; border: 1px solid #444;">status</td>
</tr>
<tr style="background-color: #1e1e1e;">
<td style="padding: 12px; border: 1px solid #444;">users</td>
<td style="padding: 12px; border: 1px solid #444;">idx_user_status_createdat</td>
<td style="padding: 12px; border: 1px solid #444;">3</td>
<td style="padding: 12px; border: 1px solid #444;">created_at</td>
</tr>
</tbody>
</table>

인덱스는 **`user_id + status + created_at`** 순서로 구성되어 있었습니다.

여기서 중요한 점은 **`expired_at`이 인덱스에 포함되어 있지 않다**는 것입니다.

---

### 문제의 핵심: 인덱스에 없는 조건

앞의 두 조건(`user_id`, `status`)은 인덱스를 타고 있으니 빠르지 않을까? 라고 생각할 수 있습니다.

하지만 실행계획을 보면 **701개의 로우를 스캔했지만, 결과는 단 1개만 반환**되었습니다. 왜 이런 일이 발생했을까요?

**문제는 `expired_at`이 인덱스에 들어가 있지 않기 때문**입니다.

1. `user_id + status`까지는 인덱스로 접근 가능
2. 하지만 `expired_at IS NULL` 조건은 인덱스에 없음
3. 따라서 **인덱스로 찾은 모든 로우에 대해 테이블에 접근하여 `expired_at` 값을 확인**해야 함
4. 이 과정에서 필터링이 발생하여, 결과보다 접근한 로우 수가 훨씬 많아짐

만약 `expired_at`이 null인 로우가 없다면(모두 만료가 됐다면), 인덱스 조건에 만족하는 **모든 로우를 테이블에 랜덤 액세스**해서 값을 확인해야 했기 때문에, 이 로우가 많으면 많을수록 쿼리가 느려지는 것입니다.

---

> #### 💡 테이블 랜덤 액세스가 느린 이유
> 
> **테이블 랜덤 액세스(Table Random Access)**는 인덱스에서 찾은 각 로우의 실제 데이터를 테이블에서 읽어오는 과정입니다.
> 
> #### 왜 느린가?
> 
> 1. **디스크 I/O 발생**: 인덱스는 메모리에 있을 수 있지만, 테이블 데이터는 디스크에 있을 가능성이 높습니다. 디스크 I/O는 메모리 접근보다 수백 배 느립니다.
> 
> 2. **비순차 접근**: 인덱스로 찾은 로우들이 테이블에서 물리적으로 멀리 떨어져 있을 수 있습니다. 순차적으로 읽는 것과 달리, 디스크 헤드를 여러 번 이동시켜야 합니다.
> 
> 3. **페이지 단위 읽기**: 데이터베이스는 페이지(보통 4KB~16KB) 단위로 데이터를 읽습니다. 필요한 로우가 여러 페이지에 흩어져 있으면, 각 페이지마다 I/O가 발생합니다.
> 
> ```
> 인덱스 스캔: 인덱스에서 조건에 맞는 로우 ID 찾기 (빠름, 메모리)
>     ↓
> 테이블 랜덤 액세스: 각 로우 ID에 해당하는 실제 데이터 읽기 (느림, 디스크)
>     ↓
> 필터링: expired_at IS NULL 확인
> ```
> 
> 인덱스 조건에 만족하는 로우가 많을수록, 테이블 랜덤 액세스 횟수가 증가하고, 이로 인해 쿼리 성능이 크게 저하됩니다.

---

### 해결 방법: 인덱스 추가

해결 방법은 간단합니다. **`user_id + status + expired_at` 인덱스를 추가**하면 됩니다.

이렇게 하면 `expired_at IS NULL` 조건도 인덱스로 처리할 수 있어, 테이블 랜덤 액세스 없이 인덱스만으로 필요한 데이터를 찾을 수 있습니다.

#### 구체적인 예시로 이해하기

예를 들어, 과거에 활동이 많았던 헤비 유저가 있다고 가정해봅시다. 이 유저는 `user_id = 12345`이고 `status = 'ACTIVE'`인 상태입니다.

하지만 이 유저는 과거에 매우 활발하게 활동했기 때문에, 해당 조건에 맞는 **백만 개의 로우**가 쌓여있습니다. 다만 지금은 활동을 하지 않아서, 그 중 **만료되지 않은 로우는 단 1개**뿐입니다.

##### 인덱스가 없을 때 (`user_id + status + created_at`만 있는 경우)

1. 인덱스에서 `user_id = 12345` AND `status = 'ACTIVE'` 조건으로 **백만 개의 로우 ID**를 찾습니다.
2. 이 백만 개의 로우 ID 각각에 대해 **테이블에 랜덤 액세스**를 수행합니다.
3. 각 로우의 `expired_at` 값을 확인하여 NULL인지 체크합니다.
4. 결국 **백만 번의 디스크 접근**이 발생하고, 그 중 단 1개만 결과로 반환됩니다.

```
인덱스 스캔: 백만 개의 로우 ID 찾기
    ↓
테이블 랜덤 액세스: 백만 번의 디스크 접근
    ↓
필터링: 각 로우의 expired_at 확인
    ↓
결과: 1개만 반환 (99.9999%는 불필요한 접근)
```

##### 인덱스가 있을 때 (`user_id + status + expired_at` 인덱스 추가)

1. 인덱스에서 `user_id = 12345` AND `status = 'ACTIVE'` AND `expired_at IS NULL` 조건으로 **직접 1개의 로우 ID**만 찾습니다.
2. 이 1개의 로우 ID에 대해서만 **테이블에 랜덤 액세스**를 수행합니다.
3. **단 1번의 디스크 접근**만으로 결과를 반환합니다.

```
인덱스 스캔: 1개의 로우 ID만 찾기 (expired_at IS NULL 조건 포함)
    ↓
테이블 랜덤 액세스: 1번의 디스크 접근
    ↓
결과: 1개 반환 (필요한 데이터만 접근)
```

이렇게 인덱스에 `expired_at`을 포함시키면, **후보군 자체가 이미 필터링된 상태**이기 때문에 불필요한 테이블 접근을 크게 줄일 수 있습니다.

‼️ 하지만 여기서 중요한 고려사항이 있습니다.

---

### 인덱스 추가의 부담: 샤딩과 대용량 데이터

만약 DB가 **샤딩이 많이 되어있고, 데이터가 굉장히 많다면** 인덱스를 추가하는 것만으로도 큰 부담이 됩니다.

#### 왜 부담이 되는가?

1. **기존 프로세스 실행계획 영향**: 새로운 인덱스가 추가되면 옵티마이저가 다른 쿼리의 실행계획을 변경할 수 있습니다. 기존에 잘 작동하던 쿼리가 갑자기 느려지거나, 예상치 못한 인덱스를 사용하게 될 위험이 있습니다. 특히 복잡한 쿼리나 여러 인덱스가 있는 테이블에서는 더욱 주의가 필요합니다.

2. **동시성 영향**: 인덱스 생성 중에는 테이블에 락이 걸릴 수 있어, 서비스 중단 없이 진행하기 어려울 수 있습니다.

3. **인덱스 생성 시간 및 DDL 작업 부담**: 대용량 테이블에 인덱스를 추가하면, 기존 데이터를 모두 스캔하여 인덱스를 구축해야 합니다. 테이블이 크면 클수록 이 과정에 시간이 오래 걸리며(수 시간에서 수일), DB 리소스를 많이 사용하므로 다른 작업에 영향을 줄 수 있습니다. 또한 작업 실패 시 롤백이나 재시도에 대한 부담도 있습니다.

4. **샤딩 환경**: 샤딩된 환경에서는 각 샤드마다 인덱스를 생성해야 하므로, 전체 작업 시간과 리소스가 샤드 수만큼 곱해집니다.

5. **유지보수 비용**: 인덱스가 많아질수록 INSERT/UPDATE/DELETE 작업 시 인덱스도 함께 업데이트해야 하므로, 쓰기 성능에 영향을 줄 수 있습니다.

6. **디스크 공간**: 인덱스도 디스크 공간을 차지합니다. 특히 복합 인덱스는 여러 컬럼을 포함하므로 상당한 공간이 필요합니다.

따라서 인덱스를 추가하기 전에, **다른 대안이 없는지 먼저 검토**해야 합니다.

---

### 대안 검토

인덱스를 추가하기 전에, 다른 해결 방법이 있는지 확인해야 합니다. 인덱스 추가는 비용이 드는 작업이므로, 먼저 더 간단한 대안이 있는지 검토하는 것이 중요합니다.

#### 대안 1: 데이터 구조 변경 고려

**만료된 데이터를 물리적으로 삭제하면 쿼리가 훨씬 빨라지고 데이터도 복잡하지 않을 텐데?**

이 방법을 고려할 때는 다음 사항들을 검토해야 합니다:

- **비즈니스 요구사항**: 만료된 데이터를 보관해야 하는가? (예: 분석, 감사, 복구 등)
- **데이터 보관 정책**: 법적 요구사항이나 회사 정책상 데이터 보관 기간이 있는가?
- **쿼리 패턴**: 만료된 데이터를 조회하는 쿼리가 있는가?
- **트레이드오프**: 쿼리 성능 vs 데이터 보관의 우선순위는?

만약 만료된 데이터가 필요 없다면, 물리적 삭제가 가장 간단한 해결책입니다. 하지만 데이터 보관이 필요한 경우에는 이 방법을 사용할 수 없습니다.

#### 대안 2: 기존 인덱스 활용 방법 검토

**`expired_at`이 null인 것은 가장 최신 데이터가 아닌가? 그럼 `created_at`으로 인덱스가 되어있으니, ORDER BY created_at DESC LIMIT 1로 가장 최신 것 한 개만 가져오면 되지 않나?**

이 방법을 고려할 때는 다음을 확인해야 합니다:

- **데이터 일관성**: `expired_at IS NULL`인 데이터가 항상 `created_at`이 가장 최신인가?
- **로직 보장**: 이 가정이 항상 성립하는지 수학적으로 보장할 수 있는가?
- **엣지 케이스**: 예외 상황이 발생할 가능성은 없는가?

이론상으로는 맞을 수 있지만, **데이터가 많을수록 이 가정이 깨질 위험이 있습니다**. 특히 다음과 같은 경우에는 위험합니다:

- 데이터가 비동기적으로 업데이트되는 경우
- 여러 프로세스가 동시에 데이터를 생성/만료 처리하는 경우
- `created_at`과 `expired_at`의 관계가 명확하지 않은 경우

**서비스 로직상 리스크가 있어 채택하지 않기로 했습니다.**

---

### 최종 결정: 인덱스 추가

다른 대안들을 검토한 결과, 더 나은 대안이 없었습니다.

이제 **DB팀에 히스토리와 대안들, 그리고 선택한 결과를 함께 보내** 최종 결정을 요청했습니다. DB팀의 승인을 받아 프로덕션 환경에 인덱스를 추가하게 되었습니다.

---

### 최종 결과

인덱스를 추가한 후, 성능 개선 결과를 확인했습니다.

#### 인덱스 추가 전 (`user_id + status + created_at` 인덱스 사용)

```sql
EXPLAIN ANALYZE
SELECT id, user_id, status, created_at, expired_at
FROM users USE INDEX (idx_user_status_createdat)
WHERE user_id = 12345
AND status = 'ACTIVE'
AND expired_at IS NULL;
```

**실행계획:**

```
-> Filter: (users.expired_at is null)  (cost=309 rows=73.7) (actual time=286..286 rows=1 loops=1)
   -> Index lookup on users using idx_user_status_createdat (user_id=12345, status='ACTIVE'), with index condition: (users.status = 'ACTIVE')  (cost=309 rows=737) (actual time=26.9..286 rows=737 loops=1)
```

**결과**: `actual time=26.9..286 rows=737 loops=1`로 **737건의 만료된 레코드를 불필요하게 읽었습니다**.

#### 인덱스 추가 후 (`user_id + status + expired_at` 인덱스 사용)

```sql
EXPLAIN ANALYZE
SELECT id, user_id, status, created_at, expired_at
FROM users
WHERE user_id = 12345
AND status = 'ACTIVE'
AND expired_at IS NULL;
```

**실행계획:**

```
-> Index lookup on users using idx_user_status_expiredat (user_id=12345, status='ACTIVE', expired_at=NULL), with index condition: ((users.status = 'ACTIVE') and (users.expired_at is null))  (cost=0.519 rows=1) (actual time=1.06..1.06 rows=1 loops=1)
```

**결과**: `actual time=1.06..1.06 rows=1 loops=1`로 **정확히 1건만 읽고 종료**하는 실행계획을 확인했습니다.

#### 성능 비교

<table style="width: 100%; border-collapse: collapse; margin: 20px 0;">
<thead>
<tr style="background-color: #2d2d2d; color: #fff;">
<th style="padding: 12px; border: 1px solid #444; text-align: left;">항목</th>
<th style="padding: 12px; border: 1px solid #444; text-align: left;">인덱스 추가 전</th>
<th style="padding: 12px; border: 1px solid #444; text-align: left;">인덱스 추가 후</th>
<th style="padding: 12px; border: 1px solid #444; text-align: left;">개선율</th>
</tr>
</thead>
<tbody>
<tr style="background-color: #1e1e1e;">
<td style="padding: 12px; border: 1px solid #444;"><strong>접근한 로우 수</strong></td>
<td style="padding: 12px; border: 1px solid #444;">737개</td>
<td style="padding: 12px; border: 1px solid #444;">1개</td>
<td style="padding: 12px; border: 1px solid #444;"><strong>99.86% 감소</strong></td>
</tr>
<tr style="background-color: #252525;">
<td style="padding: 12px; border: 1px solid #444;"><strong>실행 시간</strong></td>
<td style="padding: 12px; border: 1px solid #444;"><strong>286ms</strong></td>
<td style="padding: 12px; border: 1px solid #444;"><strong>1.06ms</strong></td>
<td style="padding: 12px; border: 1px solid #444;"><strong>약 270배 빠름<br>99.63% 감소</strong></td>
</tr>
<tr style="background-color: #1e1e1e;">
<td style="padding: 12px; border: 1px solid #444;"><strong>테이블 랜덤 액세스</strong></td>
<td style="padding: 12px; border: 1px solid #444;">737번</td>
<td style="padding: 12px; border: 1px solid #444;">1번</td>
<td style="padding: 12px; border: 1px solid #444;"><strong>99.86% 감소</strong></td>
</tr>
</tbody>
</table>

인덱스 추가 전에는 **737번의 테이블 랜덤 액세스**가 발생했지만, 인덱스 추가 후에는 **단 1번의 접근**만으로 필요한 데이터를 찾을 수 있게 되었습니다.

이런 일련의 과정을 거쳐 latency가 줄어들면서 문제가 해결되었습니다.

---

### 정리: 인덱스 하나를 추가하기까지

결국 인덱스를 추가했다는 얘기지만, 단순히 "인덱스가 없어서 느렸다"고 판단하고 바로 추가한 것은 아닙니다.

**하나의 인덱스를 추가하기 위해, 그리고 어떤 latency 성능적인 문제를 해결하기 위해 어떤 과정을 밟았는지** 정리해보면:

1. **문제 인식**: RPC latency가 느리다는 것을 확인
2. **원인 파악**: Datadog으로 DB 쿼리 시간이 병목임을 확인
3. **쿼리 분석**: 실제 사용하는 쿼리 확인
4. **실행계획 분석**: EXPLAIN으로 왜 느린지 확인
5. **인덱스 구조 확인**: 현재 인덱스가 어떻게 되어있는지 확인
6. **문제 원인 파악**: 인덱스에 없는 조건으로 인한 테이블 랜덤 액세스 발생
7. **대안 검토**: 인덱스 추가 외의 다른 방법 검토
8. **테스트**: 알파 환경에서 인덱스 추가 후 성능 확인
9. **문서화 및 승인**: DB팀에 히스토리와 대안, 결과를 정리하여 승인 요청
10. **프로덕션 적용**: 최종 승인 후 프로덕션에 적용

이렇게 **체계적인 과정을 거쳐 문제를 해결**했습니다.

---

### 마무리

저는 이번에 쿼리 튜닝을 하면서 단순히 "인덱스를 추가하면 된다"고 생각하지 않고, 체계적으로 접근했습니다.

실행계획을 분석하고, 왜 느린지 정확히 파악한 후, 여러 대안을 검토하고, 테스트를 통해 검증한 다음, 적절한 승인 절차를 거쳐 적용했습니다.

특히 **대용량 데이터베이스나 샤딩 환경**에서는 인덱스 추가가 단순한 작업이 아니었기 때문에, 더욱 신중하게 접근했습니다.

이러한 과정을 거쳐 문제를 해결할 수 있었고, 비슷한 상황을 겪는 분들에게 도움이 되었으면 좋겠습니다.
