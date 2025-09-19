+++
date = '2025-09-19T15:57:21+02:00'
draft = false
title = 'Chapter 4: Testing the Storage Layer - From Mocks to Reality 💾🧪'
tags = ["golang", "testing", "file-io", "concurrency", "storage", "race-conditions", "testify", "temp-directories", "learning-in-public", "clean-architecture"]
categories = ["Go Journey"]

+++

From mocking to reality! After testing the service layer with mocks, I'm now tackling the storage layer—where data actually hits the disk. This is where things get interesting (and messy) with real file I/O, concurrency, and error conditions you can't easily simulate.

**Repo/branch**: [https://github.com/macesz/todo-go/tree/handlers-web-unit-tests/dal/infiletodo](https://github.com/macesz/todo-go/tree/handlers-web-unit-tests/dal/infiletodo)

*Quick confession: I somehow forgot to save storage tests to a separate branch as I was in flow and made storage tests and handlers tests one after the other. I know it's not good practice, and I really hate it when this happens! 🤦‍♂️ But hey, that's real development—sometimes you get so focused you forget the process. Learn from my mistake!*

## Why Storage Layer Tests Are Different 🎯

Testing the storage layer is a completely different beast compared to service layer testing:

### **Real I/O Operations**
- **No mocks**: We're testing actual file reading, writing, and JSON marshaling
- **Filesystem interactions**: Directory creation, file permissions, disk space
- **Performance considerations**: Real I/O is slower and more unpredictable

### **Concurrency Challenges**
- **Race conditions**: Multiple goroutines accessing the same file
- **File locking**: Atomic operations to prevent corruption
- **Resource contention**: File handles, memory, CPU

### **Error Scenarios**
- **Corrupted files**: Invalid JSON, partial writes
- **Permission issues**: Read-only files, protected directories
- **Resource exhaustion**: No disk space, too many open files

### **Data Integrity**
- **Serialization correctness**: CSV format, timestamp encoding
- **Round-trip integrity**: Save then load should be identical
- **Format validation**: Proper structure and field types

## Key Testing Patterns I Discovered 🔍

### 1. **`t.TempDir()` for Perfect Isolation**

This built-in Go testing function is a game-changer:

```go
func TestFileStore_Create(t *testing.T) {
    // Each test gets its own temporary directory
    tempDir := t.TempDir()

    store := NewFileStore(filepath.Join(tempDir, "todos.csv"))

    todo, err := store.Create(context.Background(), "Test todo")

    require.NoError(t, err)
    assert.Equal(t, "Test todo", todo.Title)

    // Directory automatically cleaned up when test ends
}
```

**Benefits:**
- **No manual cleanup**: Go handles directory removal automatically
- **Perfect isolation**: Tests can't interfere with each other
- **Parallel-safe**: Each test has its own filesystem space
- **No leftover files**: Clean slate every time

### 2. **Concurrent Testing with Goroutines**

Testing real concurrency reveals issues that mocks simply can't catch:

```go
func TestFileStore_ConcurrentWrites(t *testing.T) {
    tempDir := t.TempDir()
    store := NewFileStore(filepath.Join(tempDir, "todos.csv"))

    const numGoroutines = 200
    const todosPerGoroutine = 5

    var wg sync.WaitGroup
    results := make(chan domain.Todo, numGoroutines*todosPerGoroutine)
    errors := make(chan error, numGoroutines*todosPerGoroutine)

    // Spawn 200 goroutines creating todos simultaneously
    for i := 0; i < numGoroutines; i++ {
        wg.Add(1)
        go func(goroutineID int) {
            defer wg.Done()

            for j := 0; j < todosPerGoroutine; j++ {
                title := fmt.Sprintf("Todo %d-%d", goroutineID, j)

                todo, err := store.Create(context.Background(), title)
                if err != nil {
                    errors <- err
                    return
                }

                results <- todo
            }
        }(i)
    }

    wg.Wait()
    close(results)
    close(errors)

    // Verify no errors occurred
    var errorCount int
    for err := range errors {
        t.Errorf("Concurrent write error: %v", err)
        errorCount++
    }
    require.Zero(t, errorCount, "Expected no concurrent write errors")

    // Verify all todos were created with unique IDs
    seenIDs := make(map[int]bool)
    todoCount := 0

    for todo := range results {
        require.False(t, seenIDs[todo.ID], "Duplicate ID found: %d", todo.ID)
        seenIDs[todo.ID] = true
        todoCount++
    }

    assert.Equal(t, numGoroutines*todosPerGoroutine, todoCount)

    // Verify file integrity after concurrent operations
    todos, err := store.GetAll(context.Background())
    require.NoError(t, err)
    assert.Equal(t, numGoroutines*todosPerGoroutines, len(todos))
}
```

**This test caught a race condition I never would have found with simple unit tests!**

### 3. **Error Condition Testing**

With real files, you can create actual error scenarios:

```go
func TestFileStore_CorruptedFile(t *testing.T) {
    tempDir := t.TempDir()
    filePath := filepath.Join(tempDir, "todos.csv")

    // Create a corrupted CSV file
    corruptedContent := `1,"Valid todo",false,2024-01-18T10:00:00Z
2,"Another todo",invalid_boolean,2024-01-18T11:00:00Z
this is not even CSV format
3,"Third todo"`

    err := os.WriteFile(filePath, []byte(corruptedContent), 0644)
    require.NoError(t, err)

    store := NewFileStore(filePath)

    // Should handle corruption gracefully
    todos, err := store.GetAll(context.Background())

    // Depending on your error handling strategy:
    if err != nil {
        // Option 1: Fail fast on corruption
        assert.Error(t, err)
        assert.Contains(t, err.Error(), "corrupted")
    } else {
        // Option 2: Skip corrupted records, load valid ones
        assert.Equal(t, 1, len(todos)) // Only valid record loaded
        assert.Equal(t, "Valid todo", todos[0].Title)
    }
}

func TestFileStore_PermissionDenied(t *testing.T) {
    if runtime.GOOS == "windows" {
        t.Skip("Permission testing on Windows requires different approach")
    }

    tempDir := t.TempDir()
    filePath := filepath.Join(tempDir, "readonly.csv")

    // Create file and make it read-only
    err := os.WriteFile(filePath, []byte(""), 0444) // Read-only
    require.NoError(t, err)

    store := NewFileStore(filePath)

    _, err = store.Create(context.Background(), "Should fail")

    assert.Error(t, err)
    assert.Contains(t, err.Error(), "permission denied")
}
```

### 4. **File Content Verification**

Testing not just that files are created, but that they contain exactly what we expect:

```go
func TestFileStore_FileFormat(t *testing.T) {
    tempDir := t.TempDir()
    filePath := filepath.Join(tempDir, "todos.csv")

    store := NewFileStore(filePath)

    // Create a todo
    created := time.Now().UTC().Truncate(time.Second)
    todo, err := store.Create(context.Background(), "Test todo")
    require.NoError(t, err)

    // Read raw file content
    content, err := os.ReadFile(filePath)
    require.NoError(t, err)

    lines := strings.Split(strings.TrimSpace(string(content)), "\n")
    assert.Equal(t, 2, len(lines)) // Header + 1 data row

    // Verify header
    assert.Equal(t, "id,title,done,created_at", lines[0])

    // Verify data format
    expectedLine := fmt.Sprintf(`%d,"Test todo",false,%s`,
        todo.ID,
        created.Format(time.RFC3339))

    assert.Equal(t, expectedLine, lines[1])
}

func TestFileStore_RoundTripIntegrity(t *testing.T) {
    tempDir := t.TempDir()
    store := NewFileStore(filepath.Join(tempDir, "todos.csv"))

    // Original data
    originalTodos := []string{
        "Learn Go testing",
        "Write blog post",
        "Deploy to production",
    }

    // Save todos
    var savedTodos []domain.Todo
    for _, title := range originalTodos {
        todo, err := store.Create(context.Background(), title)
        require.NoError(t, err)
        savedTodos = append(savedTodos, todo)
    }

    // Load todos
    loadedTodos, err := store.GetAll(context.Background())
    require.NoError(t, err)

    // Verify round-trip integrity
    require.Equal(t, len(savedTodos), len(loadedTodos))

    for i, saved := range savedTodos {
        loaded := loadedTodos[i]
        assert.Equal(t, saved.ID, loaded.ID)
        assert.Equal(t, saved.Title, loaded.Title)
        assert.Equal(t, saved.Done, loaded.Done)
        assert.True(t, saved.CreatedAt.Equal(loaded.CreatedAt))
    }
}
```

## What I'm Testing ✅

### **File Serialization**
- **CSV format correctness**: Headers, quoting, escaping
- **Timestamp encoding**: RFC3339 format, UTC handling
- **Data type preservation**: Boolean values, integers, strings

### **Load/Save Operations**
- **Create operations**: File creation, data appending
- **Read operations**: File parsing, data reconstruction
- **Update operations**: In-place editing, data consistency
- **Delete operations**: Record removal, file compaction

### **Concurrent Operations**
- **200 goroutines**: Simultaneous creates with unique IDs
- **Race condition detection**: Using `go test -race`
- **File locking**: Atomic operations preventing corruption
- **Resource management**: File handle lifecycle

### **Error Recovery**
- **Corrupted files**: Invalid JSON, malformed CSV
- **Missing permissions**: Read-only files, protected directories
- **Resource exhaustion**: Disk full, too many open files
- **Network issues**: NFS/remote filesystem problems

### **Resource Management**
- **File handles**: Proper opening/closing
- **Temporary directories**: Cleanup after tests
- **Memory usage**: Large file handling
- **Concurrent access**: Multiple readers/writers

## The Concurrency Test Was Eye-Opening! 👀

Spawning 200 goroutines to create todos simultaneously and verifying every ID is unique caught a race condition I never would have found with simple unit tests. Here's what happened:

```go
// The race condition bug (now fixed):
func (s *FileStore) Create(ctx context.Context, title string) (domain.Todo, error) {
    // ❌ Race condition: Multiple goroutines could read the same last ID
    todos, err := s.GetAll(ctx)
    if err != nil {
        return domain.Todo{}, err
    }

    // ❌ Two goroutines might calculate the same nextID
    nextID := s.getNextID(todos)

    // ❌ Both goroutines write with duplicate IDs
    todo := domain.Todo{
        ID:        nextID,
        Title:     title,
        Done:      false,
        CreatedAt: time.Now().UTC(),
    }

    return s.saveTodo(todo)
}
```

**The fix required proper synchronization:**

```go
// ✅ Fixed version with proper locking
func (s *FileStore) Create(ctx context.Context, title string) (domain.Todo, error) {
    s.mu.Lock()  // Protect the entire create operation
    defer s.mu.Unlock()

    todos, err := s.GetAll(ctx)
    if err != nil {
        return domain.Todo{}, err
    }

    nextID := s.getNextID(todos)

    todo := domain.Todo{
        ID:        nextID,
        Title:     title,
        Done:      false,
        CreatedAt: time.Now().UTC(),
    }

    return s.saveTodo(todo)
}
```

**The power of Go's concurrency model really shines here!** This test gave me confidence that my file storage could handle real-world concurrent usage.

## Testing Tools and Setup 🛠️

### **Core Testing Libraries**
```go
import (
    "context"
    "os"
    "path/filepath"
    "sync"
    "testing"
    "time"

    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require" // Perfect for this layer - fail fast
)
```

### **Why `require` Over `assert`?**

For storage layer tests, I prefer `require` for critical operations:

```go
// ✅ Good: Fail fast on I/O errors
tempDir := t.TempDir()
store := NewFileStore(filepath.Join(tempDir, "todos.csv"))

todo, err := store.Create(context.Background(), "Test")
require.NoError(t, err) // Stop here if create fails

// Only continue if create succeeded
assert.Equal(t, "Test", todo.Title)
assert.False(t, todo.Done)

// ❌ Bad: Could panic if create failed
todo, err := store.Create(context.Background(), "Test")
assert.NoError(t, err)  // Test continues even on failure
assert.Equal(t, "Test", todo.Title) // Could panic if todo is empty
```

### **Race Condition Detection**
```bash
# Run tests with race detector
go test -race ./...

# Run specific storage tests with race detection
go test -race -v ./dal/infiletodo

# Run concurrent tests multiple times to catch intermittent issues
for i in {1..10}; do go test -race ./dal/infiletodo; done
```

### **Benchmarking Storage Performance**
```go
func BenchmarkFileStore_Create(b *testing.B) {
    tempDir := b.TempDir()
    store := NewFileStore(filepath.Join(tempDir, "bench.csv"))

    b.ResetTimer()

    for i := 0; i < b.N; i++ {
        title := fmt.Sprintf("Benchmark todo %d", i)
        _, err := store.Create(context.Background(), title)
        if err != nil {
            b.Fatal(err)
        }
    }
}

func BenchmarkFileStore_GetAll(b *testing.B) {
    tempDir := b.TempDir()
    store := NewFileStore(filepath.Join(tempDir, "bench.csv"))

    // Create test data
    for i := 0; i < 1000; i++ {
        store.Create(context.Background(), fmt.Sprintf("Todo %d", i))
    }

    b.ResetTimer()

    for i := 0; i < b.N; i++ {
        _, err := store.GetAll(context.Background())
        if err != nil {
            b.Fatal(err)
        }
    }
}
```

## Key Learnings 🎓

### 1. **File I/O is Inherently Complex**
Unlike in-memory operations, file I/O involves:
- **OS-level operations**: Syscalls, kernel interaction
- **Filesystem behavior**: Different filesystems, network mounts
- **Resource management**: File descriptors, disk space, permissions
- **Concurrent access**: Multiple processes/threads accessing files

### 2. **`testify/require` is Perfect for This Layer**
When testing I/O operations, failing fast prevents cascading errors:
- **File creation fails**: Don't try to read from it
- **Permission denied**: Don't attempt further operations
- **Corrupted data**: Stop before making assumptions

### 3. **Temp Directories Make Tests Reliable**
Using `t.TempDir()` solved many problems:
- **No test interference**: Each test has isolated filesystem
- **Automatic cleanup**: No manual file deletion needed
- **Parallel execution**: Tests can run concurrently safely
- **Consistent state**: Fresh environment every time

### 4. **Real Concurrency Testing Reveals Hidden Issues**
Mocks can't simulate:
- **Race conditions**: Actual timing-dependent bugs
- **Resource contention**: File locks, memory pressure
- **OS scheduler behavior**: Context switching effects
- **Hardware limitations**: Disk I/O bottlenecks

### 5. **Error Conditions are Easier to Test with Real Files**
Creating actual error scenarios is straightforward:
- **Corrupted files**: Write invalid content
- **Permission errors**: Change file modes
- **Disk full**: Fill up temp filesystem
- **Concurrent access**: Multiple processes accessing same file

## Common Storage Layer Testing Patterns 📋

### **Test Structure Template**
```go
func TestFileStore_Operation(t *testing.T) {
    // Setup: Create isolated environment
    tempDir := t.TempDir()
    store := NewFileStore(filepath.Join(tempDir, "test.csv"))

    // Optional: Pre-populate data
    setupData(t, store)

    // Action: Perform operation under test
    result, err := store.Operation(context.Background(), params...)

    // Assertions: Verify results
    require.NoError(t, err)
    assert.Equal(t, expected, result)

    // Optional: Verify side effects (file content, state changes)
    verifyFileContent(t, store)
}
```

### **Concurrent Testing Template**
```go
func TestFileStore_ConcurrentOperation(t *testing.T) {
    tempDir := t.TempDir()
    store := NewFileStore(filepath.Join(tempDir, "concurrent.csv"))

    const numWorkers = 100
    var wg sync.WaitGroup
    results := make(chan Result, numWorkers)
    errors := make(chan error, numWorkers)

    // Launch concurrent workers
    for i := 0; i < numWorkers; i++ {
        wg.Add(1)
        go func(workerID int) {
            defer wg.Done()

            result, err := store.Operation(context.Background(), params...)
            if err != nil {
                errors <- err
            } else {
                results <- result
            }
        }(i)
    }

    wg.Wait()
    close(results)
    close(errors)

    // Verify no errors
    assert.Empty(t, errors)

    // Verify results
    assert.Len(t, results, numWorkers)

    // Verify final state consistency
    verifyDataIntegrity(t, store)
}
```

## Performance Observations 📊

Storage layer tests are naturally slower than service layer tests due to I/O:

```bash
$ go test -v ./dal/infiletodo
=== RUN   TestFileStore_Create
--- PASS: TestFileStore_Create (0.01s)
=== RUN   TestFileStore_ConcurrentWrites
--- PASS: TestFileStore_ConcurrentWrites (0.15s)
=== RUN   TestFileStore_GetAll
--- PASS: TestFileStore_GetAll (0.00s)
PASS
ok      github.com/macesz/todo-go/dal/infiletodo    0.168s
```

**168ms vs 3ms for service layer** - that's the cost of real I/O! But the confidence gained is worth it.

### **Optimization Strategies**
- **Parallel tests**: Use `t.Parallel()` where safe
- **Smaller test datasets**: Don't create 10,000 records unless testing scale
- **Focused assertions**: Test specific behavior, not everything
- **Benchmark separately**: Use `go test -bench` for performance testing

## The Branching Mistake (Learn From My Pain!) 🤦‍♂️

I made a classic mistake: got so into flow that I implemented storage tests AND handler tests in the same branch. This makes the commit history messy and review harder.

**What I should have done:**
```bash
git checkout -b storage-layer-tests
# Implement storage tests
git add . && git commit -m "Add comprehensive storage layer tests"

git checkout main
git checkout -b http-handler-tests
# Implement handler tests
git add . && git commit -m "Add HTTP handler tests"
```

**What I actually did:**
```bash
# Stayed on main branch (oops)
# Implemented storage tests
# Got excited and immediately implemented handler tests
# One messy commit with everything mixed together
```

**Lesson learned**: When you're in flow, it's tempting to keep going, but good git hygiene pays off during code review and debugging. Set a timer or reminder to commit/branch regularly!

## Coming Up in This Testing Series 🚀

### 🔜 **Chapter 5: HTTP Handler Tests**
- JSON marshaling and unmarshaling
- HTTP status codes and headers
- Request validation and error handling
- Middleware testing

### ⏭️ **Chapter 6: Integration Tests**
- All layers working together
- End-to-end request flows
- Database integration testing
- Docker test containers

### 🎯 **Chapter 7: Performance and Load Testing**
- Benchmarking critical paths
- Memory profiling and optimization
- Load testing strategies
- Chaos engineering principles

## Architecture Benefits Continue 🏗️

The layered architecture from Chapter 1 continues to pay dividends:

- **Storage layer isolation**: Can test file I/O without HTTP concerns
- **Clear interfaces**: `TodoStore` interface makes testing straightforward
- **Dependency injection**: Easy to swap implementations for different test scenarios
- **Separation of concerns**: Each layer has focused, testable responsibilities

This separation makes it easy to test storage logic in isolation while building confidence in the overall system integrity.

## Community Question 💬

I'd love to hear from the community:

- **Have you found concurrency bugs through testing** that would never show up in single-threaded production usage?
- **What's your approach to testing file I/O operations?** Temp files, mocking the filesystem, or something else?
- **How do you balance test speed vs realism** when testing storage layers?
- **Any war stories** about race conditions that testing caught (or missed)?

Share your experiences—let's learn from each other's wins and fails!

---

*This is Chapter 4 of my Go learning series. Follow along as I build from simple concepts to production-ready patterns, one layer at a time.*

**Next up**: Chapter 5 will tackle HTTP handler testing, where we'll deal with JSON parsing, status codes, and request validation.

![Go Gopher with Floppy Disk](/my-programming-journey/images/go-gopher-storage-testing.png)
