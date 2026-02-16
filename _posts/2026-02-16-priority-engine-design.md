---
layout: post
title: "복잡한 로직일수록 설계가 먼저다."
summary: "아키텍처를 신경써 복잡한 로직을 구현한 덕분에, 코드 몇 줄 수정만으로 다운스트림 부하 98%를 '쉽게' 차단할 수 있었던 경험을 공유합니다."
author: woodypark
date: '2026-02-16 12:01:13 +0900'
category: architecture
thumbnail: /assets/img/posts/architecture.png
keywords: architecture, design, golang, goroutine, downstream protection, priority engine, microservices, aggregator, latency, context cancel
permalink: /blog/priority-engine-design/
---

---

복잡한 마이크로서비스 환경에서 여러 소스의 데이터를 합쳐 하나의 신뢰할 수 있는 지표(Fused Data)를 만드는 일은 흔한 과제입니다. 이때 엔지니어는 보통 **"Latency를 낮추기 위해 병렬로 다 찌르느냐"**와 **"다운스트림 부하를 위해 순차적으로 호출하느냐"**의 기로에 서게 됩니다.

저도 최근 유저 신뢰도 점수 API를 개발하면서 이런 고민을 했었는데요. 대부분의 최적화는 로직이 복잡해진 뒤에 거대한 리팩토링 리소스를 투입하며 고통스럽게 진행되잖아요. 그런데 이번에는 달랐습니다. 초기 설계 단계에서 **'우선순위 엔진'**이라는 구조적 유연성을 미리 확보해 둔 덕분에, 운영 지표를 확인한 뒤 단 몇 줄의 코드 수정만으로 다운스트림에 전파되던 불필요한 부하를 98%나 차단할 수 있었습니다.

오늘은 '무엇을 개선했는가'보다, **'어떤 설계를 했기에 개선이 그토록 쉬웠는가'**에 대한 경험을 공유하고 싶습니다.

> **참고**: 본문에 등장하는 코드와 기능명, 메서드명 등은 실제 구현과 다르게 임의로 변경하여 작성했습니다. 설명의 흐름을 위해 추상화된 예시임을 참고해 주세요.

---

### 1. 트레이드 오프의 시작: 정확도를 위한 5가지 데이터 소스와 Latency의 충돌

저희에게는 유저의 신뢰도를 판단하기 위한 5가지 데이터 소스가 있었고, 각 소스는 비즈니스 중요도에 따라 엄격한 순위가 정해져 있었습니다.

* **Priority 1**: 가장 신뢰도 높은 실시간 트랜잭션 데이터 (Transaction Service)
* **Priority 2**: 시스템 알고리즘이 분석한 유저 신뢰 패턴 (Review Service)
* **Priority 3**: 최근 활동 이력을 기반으로 한 추정치 (Social Service)
* **Priority 4**: 가공되지 않은 유저의 생(Raw) 행동 로그 (Behavior Service)
* **Priority 5**: 유저의 가입 정보 기반 기본 점수 (Default Service)

가장 직관적인 구현은 **순차적 체이닝(Sequential Chaining)**이죠. 1번이 없으면 2번을 부르고, 2번도 없으면 3번을 부르는 방식입니다. 그런데 이 방식은 데이터가 없는 유저의 경우 5개 서비스의 RPC 응답 시간을 고스란히 다 합쳐서 기다려야 하는 **Worst-case Latency** 문제를 야기합니다. 하위 순위로 갈수록 연산이 무겁기 때문에 이는 서비스 전체의 사용자 경험을 해치는 요인이었어요.

---

### 2. 초기 설계: 병렬 Fan-out과 조기 취소 엔진 (Arbitrator)

Latency를 최소화하기 위해 저희는 모든 소스를 동시에 호출하는 **병렬 Fan-out** 구조를 선택했습니다. 하지만 무분별한 병렬 호출은 하부 서비스에 큰 부담을 주죠. 그래서 **상위 우선순위 결과가 나오는 즉시 나머지 불필요한 요청을 중단**하는 지능형 엔진을 설계했습니다.

#### ⚙️ 우선순위 중재 엔진: `runPriorityPipeline`

비즈니스 로직에서 고루틴 관리 코드를 완전히 분리해서, 어떤 데이터 소스가 추가되더라도 일관된 방식으로 처리할 수 있는 범용 엔진을 구축했습니다.

```go
// [Engine] 채널로 들어오는 결과들을 우선순위에 따라 선별하고 불필요한 요청을 취소합니다.
func runPriorityPipeline(
    ctx context.Context,
    cancel context.CancelFunc,
    results chan scoreResult,
    wg *sync.WaitGroup,
    maxPriority int,
) (*Score, error) {
    // 모든 고루틴 완료 시 채널을 닫아줍니다.
    go func() {
        wg.Wait()
        close(results)
    }()

    received := make(map[int]scoreResult)

    for p := 1; p <= maxPriority; p++ {
        // 1. 이미 상위 우선순위 결과가 도착해 있다면 즉시 반환하고 다른 요청 취소
        if r, ok := received[p]; ok {
            if r.score != nil || r.err != nil {
                cancel() // 나머지 불필요한 Downstream 요청들 취소
                return r.score, r.err
            }
        }

        // 2. 현재 우선순위(p) 결과가 올 때까지 채널 대기
        for {
            select {
            case <-ctx.Done():
                return nil, ctx.Err()
            case r, ok := <-results:
                if !ok { break }
                received[r.priority] = r
                
                // 지금 기다리는 우선순위(p) 결과가 도착했다면 판단
                if r.priority == p {
                    if r.score != nil || r.err != nil {
                        cancel() // 하위 순위 고루틴들 중단
                        return r.score, r.err
                    }
                    break // 데이터가 nil이면 다음 순위(p+1)를 기다리러 나감
                }
            }
        }
    }
    return nil, nil
}
```

#### 초기 정책 구현: `GetUserTrustScore`

이 엔진을 활용해 초기에는 5가지 소스를 전면 병렬로 실행했습니다. `send` 함수 내부에 `select` 문을 두어, 이미 상위 순위에서 결과가 확정되었을 경우 채널 전송을 포기하고 빠르게 종료되도록 했어요.

```go
func GetUserTrustScore(ctx context.Context, req *TrustScoreRequest) (*TrustScoreResponse, error) {
    results := make(chan scoreResult, 5)
    wg := &sync.WaitGroup{}
    pCtx, cancel := context.WithCancel(ctx)
    defer cancel()

    // 결과를 채널로 안전하게 보내는 클로저 (Early Exit 지원)
    send := func(priority int, score *Score, err error) {
        select {
        case results <- scoreResult{priority: priority, score: score, err: err}:
        case <-pCtx.Done(): // 이미 높은 순위에서 결과가 나와 컨텍스트가 종료된 경우
            return
        }
    }

    // Priority 1 ~ 5 병렬 실행
    wg.Add(5)
    go func() { defer wg.Done(); score := transactionService.GetScore(pCtx); send(1, score, nil) }()
    go func() { defer wg.Done(); score := reviewService.GetScore(pCtx); send(2, score, nil) }()
    // ... P3, P4, P5 고루틴 생략 ...

    return runPriorityPipeline(pCtx, cancel, results, wg, 5)
}
```

---

### 3. 모니터링이 알려준 진실: 98%의 불필요한 전파

시스템 배포 후, 각 우선순위의 실제 채택률(Hit Rate)을 모니터링한 결과 흥미로운 지표가 확인됐습니다.

> **"전체 요청의 98%가 Priority 1, 2, 3 단계에서 이미 확정된다."**

Latency 관점에서는 성공적이었지만, **다운스트림 보호** 관점에서는 큰 낭비가 있더라고요. 비록 엔진이 조기에 결과를 반환하더라도, 고루틴이 생성되어 RPC 요청을 날린 시점에서는 이미 하부 서비스(특히 무거운 P4, P5)에 부하가 가해진 후였기 때문입니다. 즉, 98%의 상황에서 4~5번 서비스는 쓰지도 않을 응답을 만드느라 불필요한 리소스를 태우고 있었던 거죠.

---

### 4. 최적화: 유연한 구조가 준 선물, 'Phase 기반 레이어링'

이 문제를 해결하기 위해 **1-2-3순위(Phase 1)**와 **4-5순위(Phase 2)**를 분리하기로 했습니다. 98%의 요청은 Phase 1에서 병렬로 끝내고, 거기서 답을 못 찾은 극소수의 요청만 비로소 Phase 2를 깨우는 방식이에요.

여기서 초기 설계의 힘이 발휘됐습니다. **엔진(Arbitrator)**과 **정책(Policy)**을 엄격히 분리해 두었기에, 엔진 코드는 한 줄도 건드리지 않고 정책 로직만 살짝 수정하는 것으로 충분했습니다.

#### 🛠️ 수정된 정책: 정책 분리만으로 달성한 최적화

```go
func GetUserTrustScore(ctx context.Context, req *TrustScoreRequest) (*TrustScoreResponse, error) {
    // [Phase 1] 98%가 해결되는 상위 3개 소스만 먼저 병렬 실행 (Fast Path)
    result, err := runPhase(ctx, req.UserId, 1, 3) 
    if err != nil || result != nil {
        return result, err // 98%는 여기서 다운스트림 부하 없이 종료
    }

    // [Phase 2] 앞선 결과가 없을 때만(2%) 비로소 무거운 4, 5순위를 깨움 (Slow Path)
    // 이 수정만으로 P4, P5 서비스의 인입 트래픽을 98% 차단했습니다.
    return runPhase(ctx, req.UserId, 4, 5)
}

// runPhase는 내부적으로 기존 runPriorityPipeline 엔진을 그대로 재사용합니다.
func runPhase(ctx context.Context, userId int64, start, end int) (*Score, error) {
    // ... 기존 병렬 실행 및 runPriorityPipeline 호출 로직 ...
}
```

---

### 5. 결론: 좋은 구조는 최적화의 비용을 낮춘다

이번 최적화 작업이 단 하루 만에 성공적으로 끝날 수 있었던 이유는 명확합니다.

1. **엔진과 정책의 분리**: 우선순위 판단 알고리즘을 범용 도구로 만들어두었기에 로직 변경이 함수 호출 순서를 바꾸는 수준으로 간단했어요.
2. **다운스트림 보호 장치**: 초기부터 컨텍스트 캔슬(`cancel()`)과 `select`를 활용한 전송 제어를 설계에 반영했기에 병렬 그룹을 나누는 것이 자연스러웠습니다.
3. **데이터 기반 확신**: 세밀하게 심어둔 매트릭 덕분에 "어디서 레이어를 끊어야 효율적인지"를 정량적으로 판단할 수 있었어요.
4. **국가별 정책 분리**: 이 API는 글로벌로 4개국을 지원하는데, 각 국가마다 우선순위 정책이 달랐어요. `handleKR`처럼 국가별 메서드를 분리해두어서 한 나라의 정책을 볼 때 해당 메서드만 보면 됐습니다. Phase 수정할 때도 KR 정책만 바꾸면 되니까 그 부분만 집중해서 볼 수 있었죠.

여기서 한 가지 더 고민했던 점이 있습니다. 지나친 추상화로 모든 국가를 하나의 모듈에 묶어버렸다면, 국가별 정책이 섞여 있어서 수정할 때마다 "이 로직이 어느 나라 건가?"를 찾느라 시간이 오래 걸렸을 거예요. 그래서 **적당한 선에서의 구조**를 설계하는 게 관건이었습니다. 각국 정책을 독립적으로 볼 수 있으면서도, 공통 엔진은 재사용하는 균형이요.

구현 난이도가 높은 로직일수록 구조를 정교하게 잡아야 한다는 걸 다시 한번 체감했습니다. 엔지니어링의 정수는 단순히 기능을 구현하는 것이 아니라, **나중에 올 최적화의 순간에 내가 쏟아야 할 리소스를 미리 아껴두는 설계**에 있는 것 같아요. 유연한 아키텍처는 결국 가장 효율적인 방식으로 다운스트림을 보호하고 서비스의 지속 가능성을 보장합니다.

---

### 마무리

이 기능은 복잡한 로직을 구현하느라 고민을 굉장히 많이 했던 기능이에요. 어떻게 해도 깨끗하게 코드가 짜여지지 않는 탓에 애를 많이 먹었거든요. 처음에는 "병렬로 다 호출해서 Latency만 잡으면 되지"라고 생각했는데, 막상 운영해보니 다운스트림 부하가 생각보다 컸습니다.

그런데 다행히 초기에 설계와 구조 고민에 힘을 많이 쓴 덕분에, 이 이슈가 터졌을 때 방어하는 데 드는 리소스가 매우 적었어요. 리팩토링 없이 정책만 바꾸는 것으로 98%의 불필요한 트래픽을 막을 수 있었거든요. 그 결과를 얻었을 때 굉장히 뿌듯했습니다.

이런 생각을 하면 안 되지만, 이 이슈가 한번 터져서 오히려 좋은 경험이 됐던 것 같습니다. "좋은 구조가 중요하다"는 말은 많이 들었는데, 직접 겪어보니 진짜 그렇게 되더라고요. 들어서 아는 것과 해봤을 때 느끼는 건 다르잖아요.

**한 줄 요약**: 병렬 처리 엔진으로 Latency를 잡고, 유연한 구조 덕분에 코드 몇 줄 수정으로 하부 서비스 트래픽 98%를 방어한 설계 사례였습니다.

비슷한 Aggregator API를 설계하시거나, 다운스트림 보호에 고민이 있으신 분들께 도움이 되었으면 좋겠습니다. 다른 접근법을 써보신 분들 계시면 댓글로 달아주시면 너무 좋을 것 같아요!
