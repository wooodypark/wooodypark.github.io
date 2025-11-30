---
layout: post
title: "레이어드 아키텍처에서 DB 커넥션 관리"
summary: "Service와 Repository 레이어에서 DB 커넥션을 어떻게 관리할지 고민하다가 시도한 Handler 패턴에 대해 공유합니다."
author: woodypark
date: '2025-11-30 19:45:31 +0900'
category: go
thumbnail: /assets/img/posts/pxfuel.jpg
keywords: architecture, layered architecture, transaction, database handler, repository pattern, service layer, golang, bun, go
permalink: /blog/db-handler-pattern/
---

---

### 배경: 레이어드 아키텍처에서의 DB 관리

최근 프로젝트에서 레이어드 아키텍처를 사용하면서, Service 레이어와 Repository 레이어 간에 DB 커넥션을 어떻게 관리할지 고민이 되었습니다. 

특히 **여러 Repository 작업을 하나의 트랜잭션으로 묶어야 하는 상황**이 생기면서, 기존 구조로는 해결이 어려워졌습니다.

이번 글에서는 실제로 겪었던 문제와 해결 과정을 공유합니다.

---

### 초기 구조: Repository가 DB를 직접 관리

처음에는 Repository가 Service와 동일하게 `readDB`와 `writeDB`를 필드로 가지고, 각 메서드에서 이를 직접 사용하는 구조로 시작했습니다.

```go
// Repository가 readDB, writeDB를 필드로 가지고 있음
type ARepository struct {
    readDB  *bun.DB
    writeDB *bun.DB
}

func NewARepository(readDB, writeDB *bun.DB) ARepository {
    return &ARepository{
        readDB:  readDB,
        writeDB: writeDB,
    }
}

// 조회는 내부 readDB 사용
func (r *ARepository) GetByID(ctx context.Context, id int64) (*AModel, error) {
    var model AModel
    err := r.readDB.NewSelect().
        Model(&model).
        Where("id = ?", id).
        Scan(ctx)
    return &model, err
}

// 저장은 내부 writeDB 사용
func (r *ARepository) Create(ctx context.Context, model *AModel) (int64, error) {
    result, err := r.writeDB.NewInsert().
        Model(model).
        Exec(ctx)
    return result.LastInsertId()
}
```

#### Service에서의 사용

```go
// Service도 readDB, writeDB를 필드로 가지고 있음
type AService struct {
    readDB      *bun.DB
    writeDB     *bun.DB
    aRepo       repository.ARepository
    bRepo       repository.BRepository
}

func NewAService(readDB, writeDB *bun.DB) *AService {
    return &AService{
        readDB:  readDB,
        writeDB: writeDB,
        aRepo:   repository.NewARepository(readDB, writeDB),
        bRepo:   repository.NewBRepository(readDB, writeDB),
    }
}
```

이 구조는 간단하고 직관적이었습니다. 각 Repository가 자신이 필요한 DB를 직접 관리하므로 코드가 명확했고, 처음에는 이대로 충분할 것 같았습니다.

---

### 문제 발생: 여러 Repository를 트랜잭션으로 묶고 싶다

하지만 비즈니스 로직이 복잡해지면서, **여러 Repository 작업을 하나의 트랜잭션으로 묶어야 하는 요구사항**이 생겼습니다.

예를 들어, A 엔티티를 생성하고 동시에 B 엔티티도 함께 생성해야 하는 경우가 있었는데, 두 작업이 모두 성공하거나 모두 실패해야 했습니다.

```go
// 예: A와 B를 함께 생성해야 함
func (s *AService) CreateAWithB(ctx context.Context, req *CreateRequest) error {
    // 문제: 각 Repository가 내부 DB를 사용하므로 같은 트랜잭션으로 묶을 수 없음
    id, err := s.aRepo.Create(ctx, aModel)  // writeDB 사용
    if err != nil {
        return err
    }
    
    // 이건 다른 트랜잭션이 되어버림
    err = s.bRepo.Create(ctx, bModel)  // writeDB 사용
    // ❌ 두 작업이 하나의 트랜잭션으로 묶이지 않음
    // 만약 bRepo.Create가 실패하면 aRepo.Create는 이미 커밋된 상태
}
```

이 경우, `aRepo.Create`는 성공했지만 `bRepo.Create`가 실패하면 데이터 일관성이 깨지는 문제가 있었습니다. 실제로 이런 상황을 겪으면서 구조를 바꿔야겠다고 생각했습니다.

---

### 해결 시도 1: Service에서 DB를 관리하고 Repository에 전달

이 문제를 해결하기 위해, **Service에서 DB를 관리하고 Repository에 전달**하는 방식으로 변경해봤습니다.

#### 변경된 Repository 구조

```go
// Repository는 DB를 필드로 가지지 않음
type ARepository struct{}

func NewARepository() ARepository {
    return &ARepository{}
}

// DB를 파라미터로 받음
func (r *ARepository) Create(
    ctx context.Context, 
    db *bun.DB,  // 또는 tx *bun.Tx
    model *AModel,
) (int64, error) {
    result, err := db.NewInsert().Model(model).Exec(ctx)
    return result.LastInsertId()
}
```

#### Service에서 트랜잭션으로 묶기

```go
func (s *AService) CreateAWithB(ctx context.Context, req *CreateRequest) error {
    // Service에서 트랜잭션 시작
    err := s.writeDB.RunInTx(ctx, nil, func(ctx context.Context, tx bun.Tx) error {
        // 같은 tx를 여러 Repository에 전달
        id, err := s.aRepo.Create(ctx, &tx, aModel)
        if err != nil {
            return err
        }
        
        bModel.ParentID = id
        err = s.bRepo.Create(ctx, &tx, bModel)
        return err
    })
    return err
}
```

이제 두 Repository 작업이 하나의 트랜잭션으로 묶여서, 하나라도 실패하면 전체가 롤백되도록 만들 수 있었습니다.

---

### 새로운 문제: 휴먼 에러 발생 가능성

하지만 이 방식으로 바꾸고 나니 새로운 문제를 발견했습니다. Service에서 매번 **어떤 DB를 전달할지 결정**해야 하는데, 이 과정에서 실수가 발생하기 쉬웠습니다.

실제로 코드 리뷰를 하다 보니 이런 실수들이 종종 발견되었습니다.

#### 문제 상황 1: 조회 시 readDB vs writeDB 선택 실수

```go
func (s *AService) GetA(ctx context.Context, id int64) (*AModel, error) {
    // 실수로 writeDB를 전달할 수도 있음
    return s.aRepo.GetByID(ctx, s.writeDB, id)  // 😰 readDB를 써야 하는데
}
```

읽기 작업인데 writeDB를 사용하면 불필요하게 마스터 DB에 부하를 주게 됩니다.

#### 문제 상황 2: 트랜잭션 내부에서 일반 DB를 전달하는 실수

```go
err := s.writeDB.RunInTx(ctx, nil, func(ctx context.Context, tx bun.Tx) error {
    // 실수로 writeDB를 전달 (tx를 전달해야 함)
    id, err := s.aRepo.Create(ctx, s.writeDB, aModel)  // 😰 tx를 써야 하는데
    // 이렇게 하면 트랜잭션 밖에서 실행되어 롤백되지 않음
    // ...
})
```

트랜잭션 내부에서 일반 DB를 전달하면 트랜잭션이 제대로 작동하지 않습니다.

#### 문제 상황 3: 메서드 시그니처 복잡도

```go
// Repository 메서드가 *bun.DB와 *bun.Tx 둘 다 받아야 함?
func (r *repo) Create(
    ctx context.Context,
    db *bun.DB,  // 일반 DB
    model *AModel,
) (int64, error)

// 또는 별도 메서드?
func (r *repo) CreateTx(
    ctx context.Context,
    tx *bun.Tx,  // 트랜잭션
    model *AModel,
) (int64, error)  // 😰 메서드 중복
```

`*bun.DB`와 `*bun.Tx`를 모두 지원하려면 메서드가 중복되거나, 타입 체크가 필요해져서 코드가 복잡해졌습니다.

---

### 해결책: Handler 패턴 도입

이러한 문제들을 해결하기 위해 **Handler 패턴**을 도입하기로 했습니다. 이 패턴을 통해 DB 선택 로직을 한 곳에 모으고, 실수할 가능성을 줄이고 싶었습니다.

#### 핵심 아이디어

1. **`*bun.DB`와 `*bun.Tx`를 하나의 인터페이스로 통일**

2. **readDB/writeDB 선택 로직을 Handler에 캡슐화**

3. **Service는 옵션만 지정하면 Handler가 적절한 DB 선택**

```go
// Handler 인터페이스: DB 작업을 추상화
type Handler interface {
    NewSelect() *bun.SelectQuery
    NewInsert() *bun.InsertQuery
    NewUpdate() *bun.UpdateQuery
    NewDelete() *bun.DeleteQuery
}

// bun.DB와 bun.Tx 모두 Handler 인터페이스를 만족
var (
    _ Handler = (*bun.DB)(nil)   // ✅
    _ Handler = (*bun.Tx)(nil)   // ✅
    _ Handler = (*dbHandler)(nil) // ✅
)
```

#### Handler의 선택 로직

```go
type dbHandler struct {
    readDB  *bun.DB
    writeDB *bun.DB
    opts    *options
}

func (h *dbHandler) NewSelect() *bun.SelectQuery {
    if h.opts.tx != nil {
        return h.opts.tx.NewSelect()  // 트랜잭션이 있으면 tx 사용
    }

    if h.opts.useWriteDB {
        return h.writeDB.NewSelect()  // 강제로 writeDB 사용
    }

    return h.readDB.NewSelect()  // 기본적으로 readDB 사용
}

func (h *dbHandler) NewInsert() *bun.InsertQuery {
    if h.opts.tx != nil {
        return h.opts.tx.NewInsert()  // 트랜잭션이 있으면 tx 사용
    }

    return h.writeDB.NewInsert()  // INSERT는 항상 writeDB
}
```

Handler는 다음 우선순위로 DB를 선택하도록 구현했습니다:

1. **트랜잭션이 있으면 → tx 사용**

2. **`useWriteDB` 옵션이 있으면 → writeDB 사용**

3. **그 외 → readDB 사용 (SELECT의 경우)**

이렇게 하면 Service 레이어에서는 옵션만 지정하면 되고, 실제 DB 선택은 Handler가 알아서 처리합니다.

---

### 개선된 사용 방식

#### Repository는 Handler만 받음

```go
// Repository는 DB를 필드로 가지지 않음
type ARepository struct{}

func NewARepository() ARepository {
    return &ARepository{}
}

// Handler 인터페이스를 받음
func (r *ARepository) GetByID(ctx context.Context, dbHandler Handler, id int64) (*AModel, error) {
    var model AModel
    err := dbHandler.NewSelect().
        Model(&model).
        Where("id = ?", id).
        Scan(ctx)
    return &model, err
}

func (r *ARepository) Create(ctx context.Context, dbHandler Handler, model *AModel) (int64, error) {
    result, err := dbHandler.NewInsert().Model(model).Exec(ctx)
    return result.LastInsertId()
}
```

이렇게 하면 Repository는 `*bun.DB`인지 `*bun.Tx`인지 신경 쓸 필요가 없어졌습니다. Handler 인터페이스만 알면 되니까요.

#### Service에서 간단하게 사용

```go
// Service는 여전히 readDB, writeDB를 필드로 가짐
type AService struct {
    readDB  *bun.DB
    writeDB *bun.DB
    aRepo   repository.ARepository
    bRepo   repository.BRepository
}

// 일반 조회: readDB 자동 사용
func (s *AService) GetA(ctx context.Context, id int64) (*AModel, error) {
    dbHandler := dbexec.NewHandler(s.readDB, s.writeDB)
    return s.aRepo.GetByID(ctx, dbHandler, id)
}

// 트랜잭션으로 여러 Repository 묶기
func (s *AService) CreateAWithB(ctx context.Context, req *CreateRequest) error {
    err := s.writeDB.RunInTx(ctx, nil, func(ctx context.Context, tx bun.Tx) error {
        // WithTx 옵션만 추가하면 자동으로 tx 사용
        dbHandler := dbexec.NewHandler(s.readDB, s.writeDB, dbexec.WithTx(&tx))

        // 같은 dbHandler를 여러 Repository에 전달
        id, err := s.aRepo.Create(ctx, dbHandler, aModel)
        if err != nil {
            return err
        }

        bModel.ParentID = id
        err = s.bRepo.Create(ctx, dbHandler, bModel)
        return err
    })
    return err
}

// 강제로 writeDB 사용 (최신 데이터 필요)
func (s *AService) GetAFromWrite(ctx context.Context, id int64) (*AModel, error) {
    dbHandler := dbexec.NewHandler(s.readDB, s.writeDB, dbexec.WithWriteDB())
    return s.aRepo.GetByID(ctx, dbHandler, id)
}
```

---

### 해결된 문제들

#### 문제 1 해결: 트랜잭션으로 여러 Repository 묶기

✅ **Service에서 트랜잭션을 시작하고, 같은 `dbHandler`를 여러 Repository에 전달**하면 모든 Repository 작업이 하나의 트랜잭션으로 묶이게 되었습니다.

```go
err := s.writeDB.RunInTx(ctx, nil, func(ctx context.Context, tx bun.Tx) error {
    dbHandler := dbexec.NewHandler(s.readDB, s.writeDB, dbexec.WithTx(&tx))
    
    // 여러 Repository 호출이 모두 같은 tx 사용
    id, err := s.aRepo.Create(ctx, dbHandler, aModel)
    err = s.bRepo.Create(ctx, dbHandler, bModel)
    return err
})
```

#### 문제 2 해결: 휴먼 에러 감소

✅ **옵션만 지정하면 Handler가 적절한 DB를 자동으로 선택**하므로, 실수할 가능성이 크게 줄어들었습니다.

- 트랜잭션 내부에서는 `WithTx` 옵션만 추가하면 자동으로 tx 사용

- 일반 조회는 기본적으로 readDB 사용

- Repository는 Handler 인터페이스만 알면 됨

이제 코드 리뷰에서도 이런 실수를 발견할 일이 거의 없어졌습니다.

#### 문제 3 해결: 코드 단순화

✅ **Repository 메서드 중복이 제거**되고, Service 코드가 훨씬 간결해졌습니다.

- 일반 메서드와 Tx 메서드를 따로 만들 필요 없음

- DB 선택 로직이 Handler 한 곳에 집중되어 유지보수가 쉬워짐

- 타입 체크나 조건문이 필요 없어서 코드가 깔끔해짐

---

### 성능 및 유지보수성 개선

이 패턴을 도입한 후 다음과 같은 개선 효과를 얻었습니다:

1. **트랜잭션 관리 일관성**: 모든 트랜잭션 처리가 동일한 패턴으로 통일되어, 코드를 이해하기 쉬워졌습니다

2. **코드 가독성 향상**: Service 코드가 더 명확하고 이해하기 쉬워져서, 코드 리뷰 시간도 단축되었습니다

3. **실수 방지**: 옵션 기반으로 실수할 가능성이 크게 줄어들었습니다

---

### 추가 이점: sqlmock을 사용한 테스트 용이성 향상

추가적으로 테스트에서의 이점도 있었습니다. Handler 패턴을 도입하면서 **sqlmock을 사용한 테스트 작성이 훨씬 쉬워졌습니다**.

#### 기존 방식의 테스트 문제점

Handler 패턴이 없을 때, sqlmock을 사용해서 Service 메서드를 테스트하는 것은 어려웠습니다. 핵심 문제는 Repository가 `*bun.DB`와 `*bun.Tx`를 구분해야 한다는 점입니다.

트랜잭션을 사용하는 Service 메서드를 테스트할 때, `RunInTx` 내부에서 생성된 `tx` 객체를 Repository에 전달해야 하는데, sqlmock으로는 이 `tx` 객체를 직접 제어하기 어려웠습니다.

```go
// Service 메서드가 트랜잭션 사용
func (s *AService) CreateAWithB(ctx context.Context, req *CreateRequest) error {
    err := s.writeDB.RunInTx(ctx, nil, func(ctx context.Context, tx bun.Tx) error {
        // Repository가 *bun.Tx를 직접 받아야 함
        // 하지만 sqlmock으로는 tx 객체를 직접 제어하기 어려움
        id, err := s.aRepo.Create(ctx, &tx, aModel)  // ❌ tx 객체를 어떻게 모킹?
        // ...
    })
    return err
}
```

sqlmock으로 테스트하려면, executor를 만들어서 mock으로 처리해야 했습니다. 트랜잭션 시작, 커밋, 롤백 등을 모두 모킹해야 했고, 일반 DB 사용 케이스와 트랜잭션 사용 케이스를 분리해서 처리해야 했습니다.
- *sqlmock: Go에서 DB 쿼리를 모킹하는 라이브러리*
- *executor: `*bun.DB`와 `*bun.Tx`를 모두 처리할 수 있도록 만든 추상화 객체*

#### Handler 패턴으로 해결된 테스트

Handler 패턴의 핵심 장점은 Repository가 `*bun.DB`와 `*bun.Tx`를 구분할 필요가 없다는 점입니다. Repository는 Handler 인터페이스만 받기 때문에, sqlmock으로 테스트할 때 `bunDB`를 그대로 사용해도 됩니다.

```go
// sqlmock 테스트가 간단해짐
func TestAService_CreateAWithB(t *testing.T) {
    sqlDB, sqlMock, _ := sqlmock.New()
    bunDB := bun.NewDB(sqlDB, mysqldialect.New())
    
    // bunDB를 그대로 사용
    // RunInTx()가 호출되면 sqlmock이 기대값과 매칭
    sqlMock.ExpectBegin()
    sqlMock.ExpectExec("INSERT INTO A...").WillReturnResult(sqlmock.NewResult(1, 1))
    sqlMock.ExpectExec("INSERT INTO B...").WillReturnResult(sqlmock.NewResult(1, 1))
    sqlMock.ExpectCommit()
    
    service := &AService{
        writeDB: bunDB,  // bunDB를 그대로 사용
        // ...
    }
    
    // Handler가 내부적으로 tx를 처리하므로,
    // Repository는 신경 쓸 필요 없고, sqlmock의 기대값과 매칭됨
    err := service.CreateAWithB(ctx, req)
    assert.NoError(t, err)
}
```

기존 방식과 비교하면:

- **기존**: Repository가 `*bun.Tx`를 직접 받아야 해서, sqlmock으로 `tx` 객체를 제어하기 어려웠고, **executor를 만들어서 mock으로 처리**해야 했음

- **Handler 패턴**: Repository는 Handler 인터페이스만 받으므로, `bunDB`를 그대로 사용해도 `RunInTx` 내부의 쿼리가 sqlmock의 기대값과 매칭됨. **executor를 만들 필요 없이** 간단하게 테스트 가능

sqlmock 테스트 코드가 훨씬 간단해지고, 일반 DB 사용 케이스와 트랜잭션 사용 케이스를 동일한 방식으로 처리할 수 있게 되었습니다.

---

### 정리: Handler 패턴의 핵심

이 과정을 통해 배운 점은 다음과 같습니다:

1. **Repository는 DB 선택 책임을 가져서는 안 된다**

   - Repository는 데이터 접근 로직에만 집중해야 함

   - DB 선택은 Service 레이어에서 관리해야 함

2. **트랜잭션 관리는 Service 레이어에서 해야 한다**

   - 여러 Repository를 하나의 트랜잭션으로 묶으려면 Service에서 제어해야 함

   - Repository가 트랜잭션을 관리하면 레이어 간 결합도가 높아짐

3. **인터페이스로 추상화하면 유연성이 높아진다**

   - `*bun.DB`와 `*bun.Tx`를 하나의 인터페이스로 통일

   - Repository는 구체적인 타입을 알 필요가 없음

4. **옵션 패턴으로 복잡도를 관리할 수 있다**

   - 다양한 시나리오를 옵션으로 표현

   - 코드 중복 없이 유연한 제어 가능

5. **인터페이스 기반 설계는 테스트를 쉽게 만든다**

   - Handler 인터페이스를 모킹하기 쉬움

   - sqlmock을 사용할 때 `bunDB`를 그대로 사용해도 기대값과 매칭

   - executor를 만들어서 mock 처리할 필요 없이 간단하게 테스트 가능

---

### 마무리

레이어드 아키텍처에서 DB 커넥션을 관리하는 것은 단순해 보였지만, 실제로는 여러 고려사항이 있었습니다.

특히 **트랜잭션으로 여러 Repository를 묶어야 하는 요구사항**과 **read/write DB 분리**가 필요했던 제 상황에서는, 기존의 단순한 구조로는 해결이 어려웠습니다.

Handler 패턴을 도입하면서 제 경우에는 다음과 같은 이점을 얻을 수 있었습니다:

- Service에서 트랜잭션 제어가 가능해졌습니다

- Repository는 DB 선택 책임에서 해방되어 데이터 접근 로직에만 집중할 수 있게 되었습니다

- 휴먼 에러 가능성이 크게 줄어들었습니다

- 코드가 단순해지고 유지보수성이 향상되었습니다

- 테스트 작성이 훨씬 쉬워졌습니다

물론 각 프로젝트의 요구사항과 상황이 다르기 때문에, 모든 경우에 이 패턴이 적합한 것은 아닐 수 있습니다. 단순한 경우라면 Repository가 DB를 직접 관리하는 것이 충분히 좋은 선택일 수 있습니다.

이 글은 제가 겪었던 문제와 해결 과정을 공유한 것이고, 비슷한 고민을 하셨던 분들은 어떻게 해결하셨는지도 알려주시면 좋을 것 같습니다!
