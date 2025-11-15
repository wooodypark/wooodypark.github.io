---
layout: post
title: "SQL 쿼리 튜닝: 인덱스가 있어도 느린 쿼리"
summary: "인덱스를 추가했는데도 쿼리가 느린 이유를 찾아보고\n실행계획 분석의 중요성에 대해 알아봅시다. (feat.자동 형변환)"
author: woodypark
date: '2025-11-15 20:45:31 +0900'
category: sql
thumbnail: /assets/img/posts/sql1.jpg
keywords: sql, query tuning, index, explain, database optimization, postgresql, gist index
permalink: /blog/sql-query-tuning-index-type-conversion/
---

## 문제 상황

최근 작업을 하다가 공간 데이터를 다루는 쿼리를 작성했는데, 인덱스를 추가했음에도 불구하고 쿼리 실행 시간이 예상보다 훨씬 오래 걸리는 문제가 발생했습니다. 

인덱스를 추가했는데도 느리다면, **실제로 인덱스를 타고 있는지 확인**해야 합니다.

## 실행계획 확인: EXPLAIN의 중요성

쿼리가 느리다고 느껴질 때는 무작정 인덱스를 추가하기보다, 먼저 **실행계획(Execution Plan)을 확인**해야 합니다. PostgreSQL에서는 `EXPLAIN` 또는 `EXPLAIN ANALYZE`를 사용하여 쿼리의 실행 계획을 확인할 수 있습니다.

> **참고**: 아래 쿼리는 실제 사용 쿼리가 아니라 설명을 위해 수정한 예시입니다.

```sql
EXPLAIN ANALYZE
SELECT * FROM locations AS lo
WHERE ST_DWithin(
    lo.geom::geography,
    ST_GeomFromText('POINT(127.0 37.5)', 4326)::geography,
    100
);
```

위의 쿼리를 아주 간단히 설명하면 특정 좌표(경도 127.0, 위도 37.5)로부터 100m 거리 안에 있는 모든 locations 값들을 조회하는 쿼리입니다.

실행계획을 확인해보니 다음과 같은 결과가 나왔습니다:

```
Gather  (cost=1000.00..864201.10 rows=16 width=304) (actual time=248.512..3684.564 rows=4 loops=1)
  Workers Planned: 2
  Workers Launched: 2
  ->  Parallel Seq Scan on locations lo  (cost=0.00..863199.50 rows=7 width=304) (actual time=1454.043..3521.002 rows=1 loops=3)
        Filter: st_dwithin((geom)::geography, '...'::geography, '100'::double precision, true)
        Rows Removed by Filter: 9567
Planning Time: 0.111 ms
Execution Time: 3684.599 ms
```

실행계획을 보면 **Parallel Seq Scan** (병렬 순차 스캔)이 발생하고 있습니다. 즉, 인덱스를 전혀 사용하지 않고 **테이블 풀스캔(Full Table Scan)**을 하고 있었습니다. 인덱스를 추가했는데도 왜 인덱스를 타지 않을까요?

## 원인 분석: 형변환으로 인한 인덱스 미사용

우선 인덱스가 잘 사용이 되고 있지 않으니 인덱스가 잘 추가되어 있는지부터 확인해야 합니다.
```sql
CREATE INDEX idx_location_geometry ON locations USING GIST (geom);
```

문제의 원인을 찾았습니다. 

**GIST 인덱스가 `geometry` 타입으로 생성되어 있었는데**, 공간 쿼리를 위해 `geography` 타입으로 변환한 후 쿼리를 실행하고 있었습니다. 

```sql
-- 인덱스는 geometry 타입으로 생성됨
CREATE INDEX idx_location_geometry ON locations USING GIST (geom);

-- 하지만 쿼리에서는 geography로 변환
WHERE ST_DWithin(
    lo.geom::geography,  -- 형변환 발생!
    ST_GeomFromText('POINT(127.0 37.5)', 4326)::geography,
    100
)
```

> ‼️ **형변환이 발생하면 인덱스를 사용할 수 없습니다.** 데이터베이스는 칼럼의 원본 타입에 맞춰 인덱스를 생성하기 때문에, 쿼리에서 타입을 변환하면 인덱스와 매칭되지 않아 인덱스를 사용하지 못하게 됩니다.

## 이해하기 쉬운 예시

이를 더 쉽게 이해하기 위해 간단한 예시를 들어보겠습니다.

### 예시: 문자열 칼럼에 정수로 검색하기

```sql
-- users 테이블의 id 칼럼이 VARCHAR(문자열) 타입으로 저장되어 있다고 가정
CREATE TABLE users (
    id VARCHAR(50) PRIMARY KEY,
    name VARCHAR(100)
);

-- id 칼럼에 인덱스가 자동으로 생성됨 (PRIMARY KEY)
CREATE INDEX idx_users_id ON users(id);

-- 문제가 되는 쿼리: 정수값으로 검색
SELECT * FROM users WHERE id = 12345;  -- ❌ 인덱스 미사용
```

위 쿼리에서 `id`는 문자열 타입인데, 정수값 `12345`로 검색하고 있습니다. 이 경우 데이터베이스는 **자동 형변환**을 수행합니다.

실제로 데이터베이스 내부에서는 다음과 같이 변환됩니다:

```sql
-- 우리가 작성한 쿼리
SELECT * FROM users WHERE id = 12345;

-- 데이터베이스가 내부적으로 변환하는 쿼리 (의사 코드)
SELECT * FROM users WHERE TO_NUMBER(id) = 12345;
-- 또는
SELECT * FROM users WHERE CAST(id AS INTEGER) = 12345;
```

즉, **칼럼 자체에 형변환이 발생**하게 됩니다:

```
VARCHAR 칼럼(id) = INTEGER 값(12345)
→ TO_NUMBER(id) = 12345  (칼럼에 함수 적용)
→ 칼럼에 함수가 적용되면 인덱스 사용 불가
→ 테이블 풀스캔 발생
```

**핵심은 칼럼에 함수나 형변환이 적용되면 인덱스를 사용할 수 없다는 점**입니다. 인덱스는 원본 칼럼 값에 대해 생성되기 때문에, 칼럼을 변환한 값으로는 인덱스를 찾을 수 없습니다.

### 해결 방법

간단합니다! **입력값을 칼럼의 타입에 맞게 변환**하면 됩니다.

```sql
-- 올바른 쿼리: 문자열로 검색
SELECT * FROM users WHERE id = '12345';  -- ✅ 인덱스 사용
```

입력값을 문자열로 바꿔주면, 데이터베이스는 형변환 없이 인덱스를 사용할 수 있어 쿼리 성능이 크게 향상됩니다.

## 실제 해결 과정

제가 겪었던 공간 쿼리 문제도 동일한 원리였습니다.

### 문제가 된 쿼리
```sql
-- geometry 인덱스가 있는데 geography로 변환하여 쿼리
WHERE ST_DWithin(
    lo.geom::geography,  -- 형변환 발생
    ST_MakePoint(127.0, 37.5)::geography,
    100
);
```

### 해결 방법
```sql
-- 방법 1: geometry 타입으로 쿼리 수정 (권장)
SELECT *
FROM locations AS lo
WHERE ST_DWithin(
    lo.geom,  -- 형변환 제거, geometry 타입 그대로 사용
    ST_GeomFromText('POINT(127.0 37.5)', 4326),  -- geometry 타입 그대로 사용
    0.0009  -- 단위: 도(degree), 약 100m에 해당
);

-- 방법 2: geography 타입으로 인덱스 재생성
CREATE INDEX idx_locations_geography ON locations USING GIST (geom::geography);
```

내용이 너무 길어지므로 권장 방법인 1만 검증을 해보겠습니다.

geometry 타입으로 쿼리를 수정한 후 실행계획을 다시 확인해보니 다음과 같은 결과가 나왔습니다:

```
Index Scan using index_locations_on_geom on locations lo  (cost=0.41..268.69 rows=16 width=304) (actual time=0.042..0.138 rows=4 loops=1)
  Index Cond: (geom && st_expand('...'::geometry, '0.0009'::double precision))
  Filter: st_dwithin(geom, '...'::geometry, '0.0009'::double precision)
  Rows Removed by Filter: 2
Planning Time: 0.098 ms
Execution Time: 0.161 ms
```

### 성능 비교

수정 전후의 실행계획을 비교해보면:

<table style="width: 100%; border-collapse: collapse; margin: 20px 0;">
<thead>
<tr style="background-color: #2d2d2d; color: #fff;">
<th style="padding: 12px; border: 1px solid #444; text-align: left;">항목</th>
<th style="padding: 12px; border: 1px solid #444; text-align: left;">수정 전 (geography 형변환)</th>
<th style="padding: 12px; border: 1px solid #444; text-align: left;">수정 후 (geometry)</th>
<th style="padding: 12px; border: 1px solid #444; text-align: left;">개선율</th>
</tr>
</thead>
<tbody>
<tr style="background-color: #1e1e1e;">
<td style="padding: 12px; border: 1px solid #444;"><strong>스캔 방식</strong></td>
<td style="padding: 12px; border: 1px solid #444;">Parallel Seq Scan<br>(병렬 순차 스캔)</td>
<td style="padding: 12px; border: 1px solid #444;">Index Scan<br>(인덱스 스캔)</td>
<td style="padding: 12px; border: 1px solid #444;">✅ 인덱스 사용</td>
</tr>
<tr style="background-color: #252525;">
<td style="padding: 12px; border: 1px solid #444;"><strong>실행 시간</strong></td>
<td style="padding: 12px; border: 1px solid #444;"><strong>3,684.599 ms</strong></td>
<td style="padding: 12px; border: 1px solid #444;"><strong>0.161 ms</strong></td>
<td style="padding: 12px; border: 1px solid #444;"><strong>약 22,900배 빠름</strong><br><strong>99.996% 감소</strong></td>
</tr>
</tbody>
</table>

수정 전에는 **3.68초**가 걸렸던 쿼리가, 수정 후에는 **0.16ms**로 단축되었습니다. 이는 **약 22,900배 빠른 성능**이며, 실행 시간이 **99.996% 감소**한 것입니다.

실행계획을 보면 이제 **Index Scan**을 사용하고 있어, 인덱스를 통해 효율적으로 데이터를 조회하고 있음을 확인할 수 있습니다.

## 정리: 실행계획 확인의 중요성

이 경험을 통해 배운 점은 다음과 같습니다:

1. **인덱스를 추가했다고 해서 항상 인덱스를 사용하는 것은 아니다**
2. **쿼리가 느리다면 반드시 실행계획을 확인해야 한다**
3. **형변환이 발생하면 인덱스를 사용할 수 없다**
4. **칼럼의 타입과 쿼리의 타입이 일치해야 인덱스가 제대로 작동한다**

쿼리 성능 문제가 발생했을 때, 실행계획을 확인하고 왜 느린지 정확히 판단할 수 있는 능력이 중요합니다. 단순히 인덱스를 추가하는 것만으로는 해결되지 않는 경우가 많기 때문입니다.

## 마무리

데이터베이스 쿼리 튜닝은 단순히 인덱스를 추가하는 것이 전부가 아닙니다. 실행계획을 분석하고, 왜 인덱스를 사용하지 못하는지 원인을 파악하는 것이 더 중요합니다. 

특히 **형변환으로 인한 인덱스 미사용**은 흔히 발생하는 문제이지만, 해결 방법은 간단합니다. 칼럼의 타입에 맞게 쿼리를 작성하면 됩니다.

다음번에 쿼리가 느리다고 느껴질 때는, 먼저 `EXPLAIN`으로 실행계획을 확인해봅시다!

> **⚠️ 주의사항**
> 
> 이 글에서는 인덱스를 사용하지 못해 발생한 성능 문제를 다뤘지만, **인덱스만 사용한다고 해서 항상 성능이 좋아지는 것은 아닙니다.** 테이블의 대부분의 데이터를 조회해야 하는 경우나, 데이터가 매우 적은 경우에는 오히려 **풀스캔(Full Table Scan)이 더 효율적**일 수 있습니다.
> 
> 다만, **OLTP(Online Transaction Processing) 환경**에서는 대부분의 쿼리가 소량의 데이터를 조회하는 경우가 많기 때문에, 인덱스를 통한 성능 향상이 주로 발생합니다. 따라서 이 글에서는 인덱스 활용에 초점을 맞춰 작성했습니다.
> 
> 추후에는 **풀스캔이 인덱싱보다 더 효율적인 경우**에 대해서도 다뤄보겠습니다.
