---
layout: post
title: "Redis 기반 Global Rate Limiter로 업스트림 서비스 보호하기"
summary: "복잡한 Aggregator API에서 업스트림 서비스들을 보호하기 위해 Redis 기반 분산 Rate Limiter를 도입한 과정을 공유합니다."
author: woodypark
date: '2026-02-01 16:13:12 +0900'
category: redis
thumbnail: /assets/img/posts/redisthumb.png
keywords: redis, rate limiter, distributed systems, lua script, aggregator api, fail-open, microservices, golang
permalink: /blog/redis-rate-limiter/
---

---

### 배경: 업스트림 서비스 보호의 필요성

최근 프로젝트에서 사용자 신뢰도 점수를 계산하는 복잡한 Aggregator API를 운영하면서 겪었던 문제와 해결 과정을 공유합니다.

이 API는 단순히 데이터를 읽어오는 API가 아니라, 여러 소스에서 수집한 정보를 바탕으로 최적의 결과를 계산하기 위해 내부적으로 **5개 이상의 마이크로서비스**를 호출하는 'Aggregator' 역할을 수행합니다.

#### 🚨 문제의 발단

**문제의 시작점**은 특정 업스트림 서비스(Transaction Service)의 가용 용량이 다른 서비스들보다 작아서, 트래픽이 몰릴 때마다 해당 서비스가 먼저 과부하가 걸리는 상황이었습니다. 

하지만 근본적으로는 **모든 업스트림 서비스들을 안정적으로 보호**해야 한다는 필요성을 느꼈습니다.

#### 🔍 왜 Rate Limiter가 필요했을까?

아래는 실제 비즈니스 로직을 추상화해서 예시로 작성한 `GetUserTrustScore` API의 구조입니다. 하나의 요청이 얼마나 많은 업스트림 서비스에 부하를 주는지 한눈에 알 수 있습니다.

```go
func GetUserTrustScore(ctx context.Context, req *TrustScoreRequest) (*TrustScoreResponse, error) {
    // [Stage 1] 기본 사용자 정보 조회
    userId, err := authService.GetUserId(ctx, req.Token)
    if err != nil {
        return nil, err
    }

    // [Stage 2] 병렬로 여러 신뢰도 계산 방법 수행
    scoreCandidates := make(chan *ScoreCandidate, 5)
    
    go func() {
        // 우선순위 1: 거래 히스토리 기반 점수 (Transaction Service) -> [처리 용량이 상대적으로 작음]
        if score := transactionService.GetTrustScoreFromHistory(ctx, userId); score != nil {
            scoreCandidates <- &ScoreCandidate{Source: "transaction", Score: score, Priority: 1}
        }
    }()
    
    go func() {
        // 우선순위 2: 리뷰/평점 기반 점수 (Review Service)
        if score := reviewService.GetTrustScoreFromReviews(ctx, userId); score != nil {
            scoreCandidates <- &ScoreCandidate{Source: "review", Score: score, Priority: 2}
        }
    }()
    
    go func() {
        // 우선순위 3: 소셜 연결 기반 점수 (Social Service)
        if score := socialService.GetTrustScoreFromConnections(ctx, userId); score != nil {
            scoreCandidates <- &ScoreCandidate{Source: "social", Score: score, Priority: 3}
        }
    }()
    
    go func() {
        // 우선순위 4: 행동 패턴 기반 점수 (Behavior Service)
        if score := behaviorService.GetTrustScoreFromBehavior(ctx, userId); score != nil {
            scoreCandidates <- &ScoreCandidate{Source: "behavior", Score: score, Priority: 4}
        }
    }()
    
    go func() {
        // 우선순위 5: 기본 점수 (Default Service)
        if score := defaultService.GetDefaultTrustScore(ctx); score != nil {
            scoreCandidates <- &ScoreCandidate{Source: "default", Score: score, Priority: 5}
        }
    }()

    // [Stage 3] 우선순위에 따라 최적의 신뢰도 점수 선택
    return selectOptimalTrustScore(ctx, scoreCandidates)
}
```

> **핵심 문제:** Aggregator 패턴은 '팬아웃(Fan-out)' 구조를 가집니다. 즉, 클라이언트의 요청 1개가 우리 서버 내부에서 5개 이상의 요청으로 증폭됩니다. 업스트림 서비스들 입장에선 우리가 가장 무서운 **'트래픽 빌런'**이 될 수 있는 거죠. 각 업스트림 서비스마다 처리 용량과 안정성이 다르기 때문에, 트래픽이 몰리면 어느 한 곳에서 병목이 발생하고 이는 결국 전체 API의 장애(Cascading Failure)로 이어집니다.

실제로 운영하면서 이런 상황을 겪었습니다. 평소에는 문제없이 동작하던 API가, 트래픽이 조금만 몰리면 업스트림 서비스들에서 타임아웃이 발생하면서 전체 API가 불안정해졌습니다. 처음에는 Transaction Service에서 문제가 시작되었지만, **모든 업스트림 서비스가 잠재적 위험 요소**였습니다.

---

### 해결책: Redis 기반 Rate Limiter로 업스트림 서비스 보호

분산 환경에서 **업스트림 서비스들을 안정적으로 보호**하기 위해 **Redis 공유 저장소**와 **Lua 스크립트**를 활용한 Rate Limiter를 구현했습니다. 여러 서버 인스턴스가 하나의 서버처럼 정합성 있게 트래픽을 제한할 수 있도록 설계했습니다.

#### 🛠️ Core Utility: `RateLimiter`

단순한 제한을 넘어, Redis 장애 시 시스템 전체가 마비되는 것을 막기 위해 **Fail-Open** 전략을 선택할 수 있도록 설계했습니다.

```go
package utils

import (
    "context"
    "time"
    
    "github.com/ulule/limiter/v3"
    limiterRedis "github.com/ulule/limiter/v3/drivers/store/redis"
)

type RateLimiter struct {
    limiter  *limiter.Limiter
    key      string
    failOpen bool // true: Redis 장애 시 통과(가용성 우선), false: 차단(보호 우선)
}

func NewRateLimiterWithFailOpen(redisClient limiterRedis.Client, key string, rps int, failOpen bool) (*RateLimiter, error) {
    // 1초 단위로 요청 제한(RPS) 설정
    rate := limiter.Rate{
        Period: 1 * time.Second,
        Limit:  int64(rps),
    }

    // Redis를 스토어로 사용하며 'ratelimit' 접두사로 키 관리
    store, err := limiterRedis.NewStoreWithOptions(redisClient, limiter.StoreOptions{
        Prefix:          "ratelimit",
        CleanUpInterval: 0, // Redis TTL에 의존하여 클린업 비활성화
    })
    if err != nil {
        return nil, err
    }

    return &RateLimiter{
        limiter:  limiter.New(store, rate),
        key:      key,
        failOpen: failOpen,
    }, nil
}

func (l *RateLimiter) Allow(ctx context.Context) (bool, error) {
    // Redis Lua 스크립트를 호출하여 원자적(Atomic)으로 토큰 차감
    limiterCtx, err := l.limiter.Get(ctx, l.key)
    if err != nil {
        // [Fail-Open 전략] Redis 연결 오류 시 서비스 지속 여부 결정
        return l.failOpen, err
    }
    return !limiterCtx.Reached, nil
}
```

여기서 중요한 건 **`failOpen` 옵션**입니다. Redis가 장애 났을 때 어떻게 동작할지를 결정하는데:

<table style="width: 100%; border-collapse: collapse; margin: 20px 0;">
<thead>
<tr style="background-color: #2d2d2d; color: #fff;">
<th style="padding: 12px; border: 1px solid #444; text-align: left;">전략</th>
<th style="padding: 12px; border: 1px solid #444; text-align: left;">Redis 장애 시</th>
<th style="padding: 12px; border: 1px solid #444; text-align: left;">장점</th>
<th style="padding: 12px; border: 1px solid #444; text-align: left;">단점</th>
<th style="padding: 12px; border: 1px solid #444; text-align: left;">적용 케이스</th>
</tr>
</thead>
<tbody>
<tr style="background-color: #1e1e1e;">
<td style="padding: 12px; border: 1px solid #444;"><strong>🔓 Fail-Open<br><code>failOpen = true</code></strong></td>
<td style="padding: 12px; border: 1px solid #444;">요청 통과</td>
<td style="padding: 12px; border: 1px solid #444;"><strong>서비스 가용성 유지</strong></td>
<td style="padding: 12px; border: 1px solid #444;">보호 기능 상실</td>
<td style="padding: 12px; border: 1px solid #444;">안정적인 업스트림 서비스</td>
</tr>
<tr style="background-color: #252525;">
<td style="padding: 12px; border: 1px solid #444;"><strong>🔒 Fail-Closed<br><code>failOpen = false</code></strong></td>
<td style="padding: 12px; border: 1px solid #444;">요청 차단</td>
<td style="padding: 12px; border: 1px solid #444;"><strong>강력한 보호 유지</strong></td>
<td style="padding: 12px; border: 1px solid #444;">서비스 가용성 저하</td>
<td style="padding: 12px; border: 1px solid #444;">취약한 업스트림 서비스</td>
</tr>
</tbody>
</table>

---

### 실무 적용: 클라이언트 Decorator 패턴

Rate Limiter를 비즈니스 로직에 직접 넣으면 코드가 지저분해집니다. 저희는 기존 클라이언트를 래핑하는 **Decorator 패턴**을 사용하여 **캐싱(Cache)과 처리율 제한(Rate Limit)**을 깔끔하게 결합했습니다.

#### 📦 `cachedTransactionServiceClient` 구현체

업스트림 서비스를 보호하기 위해, 실제 네트워크 호출이 발생하는 **Cache Miss** 시점에만 Rate Limit 쿼터를 소모합니다.

```go
package client

import (
    "context"
    "encoding/json"
    "time"
    
    "your-project/metrics"
    "your-project/ratelimiter"
)

type cachedTransactionServiceClient struct {
    client      TransactionServiceClient
    redis       *redis.Client
    rateLimiter *ratelimiter.RateLimiter
}

func NewCachedTransactionServiceClient(
    client TransactionServiceClient,
    redis *redis.Client,
    redisForRateLimit *redis.Client,
) *cachedTransactionServiceClient {
    // failOpen=false로 설정하여, Redis 문제 시 업스트림 서버를 엄격히 보호
    rateLimiter, _ := ratelimiter.NewRateLimiterWithFailOpen(
        redisForRateLimit,
        "transaction_service",
        50, // 초당 50개 요청으로 제한
        false, // Redis 장애 시 차단 (보호 우선)
    )
    
    return &cachedTransactionServiceClient{
        client:      client,
        redis:       redis,
        rateLimiter: rateLimiter,
    }
}

func (c *cachedTransactionServiceClient) GetTrustScoreFromHistory(ctx context.Context, userId int64) (*TrustScore, error) {
    cacheKey := fmt.Sprintf("transaction_trust_score:%d", userId)
    
    // 1. 캐시 우선 조회 (캐시 히트 시 Rate Limit 소모 없음 - 업스트림 서비스 보호)
    val, err := c.redis.Get(ctx, cacheKey).Result()
    if err == nil {
        var trustScore TrustScore
        if json.Unmarshal([]byte(val), &trustScore) == nil {
            metrics.RecordCacheHit("transaction_service_client")
            return &trustScore, nil
        }
    }

    // 2. 캐시 미스 시 Rate Limiter 작동
    allowed, err := c.rateLimiter.Allow(ctx)
    if !allowed {
        // 차단된 요청은 메트릭으로 기록하여 모니터링
        metrics.RecordRateLimitExceeded("transaction_service_client")
        return nil, status.Error(codes.ResourceExhausted, "rate limit exceeded")
    }

    // 3. 실제 업스트림 서비스 호출
    result, err := c.client.GetTrustScoreFromHistory(ctx, userId)
    if err != nil {
        return nil, err
    }
    
    // 4. 호출 성공 시 결과를 비동기로 캐시에 저장
    go func() {
        if data, err := json.Marshal(result); err == nil {
            c.redis.Set(context.Background(), cacheKey, data, 5*time.Minute)
        }
    }()
    
    return result, nil
}
```

이렇게 구현하면 다음과 같은 이점이 있습니다:

- **캐시 히트 시**: Rate Limit 쿼터를 소모하지 않아 효율적
- **캐시 미스 시**: Rate Limiter가 업스트림 서비스를 보호
- **장애 시**: Fail-Open 전략에 따라 적절히 대응

---

### Fail-Open vs Fail-Closed 전략 선택

Rate Limiter 설계에서 가장 고민했던 부분은 **Redis 장애 시 어떻게 동작할 것인가**였습니다.

#### 🤔 상황별 전략 선택

**1. 보호가 중요한 업스트림 서비스 (Fail-Closed 선택)**

```go
// Transaction Service: 장애 시 차단 (보호 우선)
rateLimiter := ratelimiter.NewRateLimiterWithFailOpen(
    redis,
    "transaction_service", 
    50,    // RPS
    false, // Redis 장애 시 차단
)
```

Transaction Service처럼 처리 용량이 상대적으로 작거나 장애 시 복구가 어려운 서비스는, Redis가 죽어도 **보호를 우선**하도록 했습니다.

**2. 상대적으로 안정적인 업스트림 서비스 (Fail-Open 선택)**

```go
// Network Service: 장애 시 통과 (가용성 우선)  
rateLimiter := ratelimiter.NewRateLimiterWithFailOpen(
    redis,
    "network_service",
    200,  // RPS
    true, // Redis 장애 시 통과
)
```

Network Service는 상대적으로 안정적이고 처리 용량이 커서, Redis 장애 시 **가용성을 우선**하도록 했습니다.

#### 💡 실제 운영에서의 교훈

처음에는 모든 서비스를 Fail-Closed로 설정했었는데, Redis 클러스터에 일시적인 문제가 생겼을 때 **전체 API가 먹통**이 되는 경험을 했습니다.

그 이후로 서비스별 특성과 안정성을 고려해서 전략을 다르게 가져가기로 했습니다. 결과적으로 Redis 장애 시에도 **핵심 기능은 유지**하면서 **필요한 업스트림 서비스들을 적절히 보호**할 수 있게 되었습니다.

---

### 성과 및 모니터링 (Observability)

Rate Limiter는 도입하는 것보다 **운영**하는 것이 더 중요합니다. 저희는 다음과 같은 메트릭을 달아 적정 RPS를 지속적으로 튜닝하고 있습니다.

#### 📊 핵심 메트릭들

```go
package metrics

import (
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promauto"
)

var (
    rateLimitExceededCounter = promauto.NewCounterVec(
        prometheus.CounterOpts{
            Name: "rate_limit_exceeded_total",
            Help: "Total number of requests that were rate limited",
        },
        []string{"service"},
    )
    
    failOpenCounter = promauto.NewCounterVec(
        prometheus.CounterOpts{
            Name: "rate_limit_fail_open_total", 
            Help: "Total number of requests that passed due to Redis failure",
        },
        []string{"service"},
    )
)

func RecordRateLimitExceeded(service string) {
    rateLimitExceededCounter.WithLabelValues(service).Inc()
}

func RecordFailOpen(service string) {
    failOpenCounter.WithLabelValues(service).Inc()
}
```

#### 📈 운영하면서 배운 것들

**🔍 모니터링 관점**
- **Rate Limit Exceeded 메트릭**: 언제 얼마나 차단되는지 보면서 RPS 수준이 적절한지 판단
- **Fail-Open 발생 횟수**: Redis 문제로 예상치 못하게 통과된 요청들을 추적해서 시스템 안정성 체크

**⚖️ 튜닝 관점**  
- **서비스별 패턴 분석**: 각 업스트림 서비스마다 트래픽 패턴이 다르다는 걸 알게 됨
- **개별 조정의 필요성**: 획일적인 RPS보다는 서비스별 맞춤 설정이 중요

> 💭 **현재 상태**  
> 이런 메트릭들을 보면서 **점진적으로 더 적절한 RPS 수준을 찾아나가고 있는 중**입니다.  
> 아직 정답을 찾은 건 아니고, 계속 튜닝해가면서 각 서비스에 맞는 적정선을 찾아가는 과정이라고 생각해요.

---

### 트레이드오프와 현실적 선택

이번에 Rate Limiter를 구현하면서 **완벽한 솔루션은 없다**는 걸 다시 한번 느꼈습니다.

#### 🤔 알게 된 문제점들

**멱등성 문제**: 같은 요청이라도 우선순위 1번 서비스가 Rate Limit에 걸리면 2번으로 넘어가서, 결과가 달라질 수 있습니다. 이론적으로는 같은 Input에 같은 Output이 나와야 하는데, 시스템 상태에 따라 다른 결과가 나오는 셈이죠.

**복잡도 증가**: 각 서비스별로 다른 Fail-Open/Fail-Closed 전략, 모니터링, 튜닝... 관리해야 할 포인트가 확실히 많아졌습니다.

#### ✅ 그럼에도 선택한 이유

하지만 우리 시스템 특성상 **여러 우선순위의 폴백**이 있어서, 완전히 실패하기보다는 "차선의 결과"라도 제공할 수 있다고 판단했습니다.

**시스템 전체가 죽는 것 vs 일부 요청의 멱등성이 깨지는 것** - 이 트레이드오프에서는 후자를 선택하는 게 낫다고 생각했어요.

#### 🔄 앞으로 시도해볼 것들

- **동적 Rate Limiting**: 시간대나 서비스 상태에 따른 자동 조정
- **Circuit Breaker와의 조합**: Rate Limit + 장애 감지로 더 스마트한 보호
- **멱등성 개선**: 캐시 레이어에서 동일 요청 처리 방안

---

### 마무리

처음에는 단순히 "트래픽이 몰리니까 속도 제한을 걸어야겠다"는 생각으로 시작했는데, 막상 구현하고 운영해보니 생각보다 고려할 것들이 많았습니다.

특히 **Fail-Open vs Fail-Closed 선택**에서 꽤 고민했었는데, 결국 서비스별로 다르게 가져가는 게 답이었다는 걸 깨달았습니다. 처음엔 "무조건 보호해야지"라고 생각했지만, 실제로는 상황에 맞게 유연하게 대응하는 게 더 중요하더라고요.

그리고 이번에 Rate Limiter를 도입하면서 **멱등성이 보장되지 않는 문제**가 있다는 걸 알게 되었습니다. 같은 요청이라도 우선순위 1번(Transaction Service)이 제한에 걸리면 우선순위 2번으로 넘어가서, 결과가 달라질 수 있거든요.

하지만 우리 시스템의 특성상 **여러 우선순위의 폴백이 있어서**, 안정성을 지키면서 완전히 실패하지 않고 "차선의 결과"를 제공할 수 있다는 걸 깨달았습니다. 완벽한 솔루션은 아니지만, **트레이드오프에서는 시스템 전체가 죽는 것보다 이쪽이 훨씬 낫다**고 판단했습니다.

앞으로는 동적 Rate Limiting도 시도해볼 생각인데, 혹시 비슷한 경험 있으시거나 다른 접근법 써보신 분들 계시면 궁금합니다!