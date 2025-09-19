+++
date = '2025-09-19T15:01:37+02:00'
draft = true
title = 'Chapter 3: Testing the Service Layer - Adding Confidence to Go Code'
tags: ["golang", "testing", "clean-architecture", "mocking", "testify"]
categories = ["Go Journey"]
+++


Time to add some confidence to my Go Todo app! After building the structure and file storage, I'm diving deep into testing. Starting with the service layer, where business logic lives and external dependencies need to be tamed.

**Repo/branch**: [https://github.com/macesz/todo-go/blob/firs-unit-test/services/todo/service_test.go](https://github.com/macesz/todo-go/blob/firs-unit-test/services/todo/service_test.go)

## Why Start with Service Layer Tests? 🎯

The service layer is the perfect starting point for testing because it's where your business logic lives, isolated from external concerns:

- **Business logic isolation**: Test core functionality without I/O complications
- **Fast feedback**: No database calls, file operations, or network requests
- **Dependency control**: Mock external services to test edge cases systematically
- **Design validation**: Interfaces and dependency injection patterns prove their worth here

Think of it as testing the "brain" of your application without worrying about the "hands and feet" (I/O operations).

## Key Testing Patterns I Learned 📚

### 1. Table-Driven Tests with `initMocks` Functions

This pattern keeps test cases clean while allowing flexible mock behavior per scenario:

```go
func TestTodoService_Create(t *testing.T) {
    type args struct {
        ctx   context.Context
        title string
    }

    tests := []struct {
        name      string
        args      args
        want      domain.Todo
        wantErr   bool
        initMocks func(t *testing.T, args *args, s *TodoService)
    }{
        {
            name: "successful creation",
            args: args{
                ctx:   context.Background(),
                title: "Learn Go testing",
            },
            want: domain.Todo{
                ID:        1,
                Title:     "Learn Go testing",
                Done:      false,
                CreatedAt: time.Now(),
            },
            wantErr: false,
            initMocks: func(t *testing.T, args *args, s *TodoService) {
                mockStore := mocks.NewTodoStore(t)
                mockStore.On("Create", args.ctx, args.title).Return(expectedTodo, nil)
                s.Store = mockStore
            },
        },
        {
            name: "empty title validation",
            args: args{
                ctx:   context.Background(),
                title: "",
            },
            wantErr: true,
            initMocks: func(t *testing.T, args *args, s *TodoService) {
                // No store calls expected for validation failures
                mockStore := mocks.NewTodoStore(t)
                s.Store = mockStore
            },
        },
        {
            name: "store failure propagation",
            args: args{
                ctx:   context.Background(),
                title: "Valid title",
            },
            wantErr: true,
            initMocks: func(t *testing.T, args *args, s *TodoService) {
                mockStore := mocks.NewTodoStore(t)
                mockStore.On("Create", args.ctx, args.title).
                    Return(domain.Todo{}, errors.New("database connection failed"))
                s.Store = mockStore
            },
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            t.Parallel() // Run tests concurrently

            service := &TodoService{}

            // Set up mocks for this specific test case
            if tt.initMocks != nil {
                tt.initMocks(t, &tt.args, service)
            }

            got, err := service.Create(tt.args.ctx, tt.args.title)

            if tt.wantErr {
                assert.Error(t, err)
                return
            }

            assert.NoError(t, err)
            assert.Equal(t, tt.want.Title, got.Title)
            assert.Equal(t, tt.want.Done, got.Done)
            assert.False(t, got.CreatedAt.IsZero())
        })
    }
}
```

### 2. The `initMocks` Pattern (My Favorite Discovery!)

```go
initMocks: func(t *testing.T, args *args, s *TodoService) {
    mockStore := mocks.NewTodoStore(t)
    mockStore.On("Create", args.ctx, args.title).Return(expectedTodo, nil)
    s.Store = mockStore
}
```

This pattern is **brilliant** because it:
- Keeps the test table readable and focused
- Allows unique mock behavior per test case
- Makes it easy to set up complex mock interactions
- Separates test data from test setup logic

### 3. Fresh Mocks Per Test

Using `testify/mock` with fresh instances prevents test contamination:

```go
// ✅ Good: Fresh mock per test
mockStore := mocks.NewTodoStore(t)

// ❌ Bad: Shared mock across tests (state leaks!)
var globalMockStore *mocks.TodoStore
```

### 4. Parallel Testing for Speed

```go
func TestTodoService_GetAll(t *testing.T) {
    // ... test cases ...

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            t.Parallel() // 🚀 Run concurrently for faster execution

            // ... test logic ...
        })
    }
}
```

## What I'm Testing ✅

### CRUD Operations
```go
func TestTodoService_Create(t *testing.T) { /* ... */ }
func TestTodoService_GetByID(t *testing.T) { /* ... */ }
func TestTodoService_GetAll(t *testing.T) { /* ... */ }
func TestTodoService_Update(t *testing.T) { /* ... */ }
func TestTodoService_Delete(t *testing.T) { /* ... */ }
```

### Validation Logic
```go
// Testing business rules
{
    name: "empty title should fail",
    args: args{title: ""},
    wantErr: true,
},
{
    name: "whitespace-only title should fail",
    args: args{title: "   "},
    wantErr: true,
},
```

### Error Propagation
```go
{
    name: "store error should propagate",
    initMocks: func(t *testing.T, args *args, s *TodoService) {
        mockStore := mocks.NewTodoStore(t)
        mockStore.On("GetByID", args.ctx, args.id).
            Return(domain.Todo{}, errors.New("connection lost"))
        s.Store = mockStore
    },
    wantErr: true,
},
```

### Business Rules
```go
func TestTodoService_Complete(t *testing.T) {
    // Test: Can't complete already-done todos
    {
        name: "already completed todo should fail",
        args: args{id: 1},
        initMocks: func(t *testing.T, args *args, s *TodoService) {
            mockStore := mocks.NewTodoStore(t)
            alreadyDone := domain.Todo{ID: 1, Done: true}
            mockStore.On("GetByID", args.ctx, args.id).Return(alreadyDone, nil)
            s.Store = mockStore
        },
        wantErr: true,
    },
}
```

### Context Handling
```go
func TestTodoService_CreateWithTimeout(t *testing.T) {
    ctx, cancel := context.WithTimeout(context.Background(), 1*time.Millisecond)
    defer cancel()

    time.Sleep(2 * time.Millisecond) // Force timeout

    service := &TodoService{Store: mockStore}
    _, err := service.Create(ctx, "Test todo")

    assert.Error(t, err)
    assert.Contains(t, err.Error(), "context deadline exceeded")
}
```

## Why Mock Dependencies? 🎭

The service layer should focus on **business logic**, not I/O details. Mocking the `TodoStore` interface lets me:

### Simulate Failures Without Real Systems
```go
// Test database connection failures without a database
mockStore.On("Create", mock.Anything, mock.Anything).
    Return(domain.Todo{}, errors.New("connection refused"))
```

### Test Edge Cases Systematically
```go
// Test concurrent access scenarios
mockStore.On("GetByID", mock.Anything, 1).
    Return(domain.Todo{}, errors.New("resource locked")).
    Once()
```

### Keep Tests Fast and Deterministic
- **No I/O operations** = tests run in milliseconds
- **No external dependencies** = tests never fail due to network issues
- **Predictable responses** = tests are completely deterministic

### Verify Interface Usage
```go
// Verify the service calls store methods correctly
mockStore.AssertExpectations(t)
mockStore.AssertCalled(t, "Create", context.Background(), "Test todo")
```

## Testing Tools Stack 🛠️

### Core Testing
```go
import (
    "testing"
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/mock"
)
```

### Mock Generation
```bash
# Install mockery for generating mocks from interfaces
go install github.com/vektra/mockery/v2@latest

# Generate mocks
mockery --name=TodoStore --dir=domain --output=mocks
```

### Generated Mock Usage
```go
// mocks/TodoStore.go (auto-generated)
type TodoStore struct {
    mock.Mock
}

func (m *TodoStore) Create(ctx context.Context, title string) (domain.Todo, error) {
    ret := m.Called(ctx, title)
    return ret.Get(0).(domain.Todo), ret.Error(1)
}
```

## Real-World Test Examples 💡

### Testing Validation Logic
```go
func TestTodoService_ValidateTitle(t *testing.T) {
    tests := []struct {
        name    string
        title   string
        wantErr bool
        errMsg  string
    }{
        {"valid title", "Learn Go", false, ""},
        {"empty title", "", true, "title cannot be empty"},
        {"whitespace only", "   ", true, "title cannot be empty"},
        {"too long title", strings.Repeat("a", 256), true, "title too long"},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            service := &TodoService{}
            err := service.validateTitle(tt.title)

            if tt.wantErr {
                assert.Error(t, err)
                assert.Contains(t, err.Error(), tt.errMsg)
            } else {
                assert.NoError(t, err)
            }
        })
    }
}
```

### Testing Concurrent Operations
```go
func TestTodoService_ConcurrentCreates(t *testing.T) {
    mockStore := mocks.NewTodoStore(t)
    service := &TodoService{Store: mockStore}

    // Set up mock to handle multiple concurrent calls
    mockStore.On("Create", mock.Anything, mock.AnythingOfType("string")).
        Return(domain.Todo{ID: 1, Title: "test"}, nil).
        Times(10) // Expect exactly 10 calls

    var wg sync.WaitGroup
    errChan := make(chan error, 10)

    // Start 10 concurrent creates
    for i := 0; i < 10; i++ {
        wg.Add(1)
        go func(i int) {
            defer wg.Done()
            _, err := service.Create(context.Background(), fmt.Sprintf("Todo %d", i))
            errChan <- err
        }(i)
    }

    wg.Wait()
    close(errChan)

    // Verify no errors occurred
    for err := range errChan {
        assert.NoError(t, err)
    }

    mockStore.AssertExpectations(t)
}
```

## Running the Tests 🏃‍♂️

```bash
# Run all tests
go test ./...

# Run with coverage
go test -cover ./...

# Run with detailed output
go test -v ./...

# Run specific test
go test -run TestTodoService_Create ./service

# Run tests in parallel with race detection
go test -race -parallel 4 ./...

# Benchmark performance
go test -bench=. ./...
```

## Test Coverage Report 📊

```bash
# Generate HTML coverage report
go test -coverprofile=coverage.out ./...
go tool cover -html=coverage.out -o coverage.html
open coverage.html
```

Current coverage for service layer: **94%** 🎯

Missing coverage:
- Error handling in edge cases (working on it!)
- Some validation branches (adding more test cases)

## Lessons Learned 🎓

### 1. **Interfaces Enable Testability**
The `TodoStore` interface makes mocking trivial. Without it, testing would require real databases or complex setup.

### 2. **Table-Driven Tests Scale Well**
As I add more business logic, new test cases are just new entries in the table. The framework stays the same.

### 3. **Fresh Mocks Prevent Weird Bugs**
I learned this the hard way when tests started failing mysteriously due to mock state leaking between tests.

### 4. **Context Propagation is Crucial**
Testing context cancellation and timeouts revealed bugs I wouldn't have found otherwise.

### 5. **Error Messages Matter**
Testing that error messages are helpful made me write better error handling throughout the app.

## Common Testing Gotchas I Hit 🚨

### 1. **Not Calling `AssertExpectations`**
```go
// ❌ Forgot this - mocks not properly verified
defer mockStore.AssertExpectations(t)
```

### 2. **Sharing Mocks Between Tests**
```go
// ❌ Bad: Shared state causes flaky tests
var sharedMock *mocks.TodoStore

// ✅ Good: Fresh mock per test
mockStore := mocks.NewTodoStore(t)
```

### 3. **Not Testing Error Cases**
Initially focused on happy path. Error scenarios revealed important bugs!

### 4. **Forgetting `t.Parallel()`**
Tests run much faster when they can run concurrently.

## What's Coming Up in This Testing Series 🚀

### 🔜 **Chapter 4: Storage Layer Tests**
- Real file I/O testing
- Concurrency and race conditions
- Error recovery scenarios
- Performance benchmarks

### ⏭️ **Chapter 5: HTTP Handler Tests**
- JSON parsing and validation
- URL parameters and routing
- HTTP status codes and headers
- Request/response mocking

### 🎯 **Chapter 6: Integration Tests**
- All layers working together
- End-to-end request flows
- Database integration testing
- Docker test containers

### 🧪 **Chapter 7: Advanced Testing**
- Property-based testing
- Chaos engineering principles
- Load testing strategies
- Test data management

## Why This Matters 💡

The dependency injection design from Chapter 1 is **paying huge dividends** now. Swapping real implementations for mocks is seamless, and the clean separation of concerns makes each layer easy to test in isolation.

This is exactly why we invest in good architecture upfront—it makes everything else (including testing) much easier down the road.

## Performance Notes ⚡

Service layer tests are incredibly fast because they avoid I/O:

```bash
$ go test -v ./service
=== RUN   TestTodoService_Create
=== PAUSE TestTodoService_Create
=== RUN   TestTodoService_GetByID
=== PAUSE TestTodoService_GetByID
=== CONT  TestTodoService_Create
=== CONT  TestTodoService_GetByID
--- PASS: TestTodoService_Create (0.00s)
--- PASS: TestTodoService_GetByID (0.00s)
PASS
ok      github.com/macesz/go-todo-chi/service    0.003s
```

**3 milliseconds** for a comprehensive test suite! This fast feedback loop makes TDD actually enjoyable.

## Your Turn! 🤝

Learning in public continues! I'd love to hear from the Go community:

- **What testing patterns** have helped you the most in Go?
- **Any gotchas** I should watch for as I expand this test suite?
- **Favorite testing libraries** or tools you'd recommend?
- **Common mistakes** you made early in your Go testing journey?

Drop your thoughts in the comments—let's learn together!

---

*This is Chapter 3 of my Go learning series. Follow along as I build from simple concepts to production-ready patterns, one layer at a time.*

**Next up**: Chapter 4 will tackle the storage layer, where we'll test real file I/O, handle concurrency safely, and ensure our data persistence is bulletproof.

**Tags**: #golang #testing #clean-architecture #mocking #testify #unit-testing #tdd #service-layer #dependency-injection #learning-in-public

![Go Gopher Testing](/my-programming-journey/images/go-gopher-testing.png)
