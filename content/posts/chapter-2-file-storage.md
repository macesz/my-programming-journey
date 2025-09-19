+++
date = '2025-09-19T13:35:54+02:00'
draft = false
title = 'Chapter 2: From In-Memory to File Storage - Persistence Without Database Overhead 💾'

tags = ["golang", "learning", "architecture", "clean-architecture", "web-development", "file-storage", "backend", "persistence", "atomic-operations", "concurrency"]
categories = ["Go Journey"]
+++

Keeping the momentum going! Yesterday, I split my tiny Go Todo app into a layered structure. Today I added a file‑based storage adapter so todos survive restarts—without jumping to a full database (yet).

**Repo/branch**: [https://github.com/macesz/todo-go/tree/create-filestorage](https://github.com/macesz/todo-go/tree/create-filestorage)

## Why Start In-Memory? 🧠

Starting with in-memory storage wasn't just about being lazy—it was strategic:

- **Fast feedback**: No I/O, fewer moving parts to debug
- **Focus on the domain**: Behavior first, storage later
- **Simple wiring**: Great for sketching out architecture patterns

But of course, it came with trade-offs:

- **Volatile**: Restart = data gone 💨
- **Single process**: No sharing across instances
- **Not very realistic**: Hides I/O errors, locking, and durability concerns

## Why a File Store Next? 📁

A file-based approach gives us the best of both worlds for learning:

- **Persistence without DB overhead**: Perfect for understanding storage patterns
- **Good habits**: Forces you to think about serialization, error handling, and concurrency
- **"Right enough"** for small tools, CLIs, and early prototypes

## When Files Are Fine vs When to Skip to Database

### ✅ **Files Work Great For:**
- Solo/single‑user applications
- Modest data volumes with simple query patterns
- Offline or resource-constrained environments
- Early iterations where schemas change frequently
- Configuration storage, logs, or simple data dumps

### ❌ **Skip Files, Go Database When:**
- Multi‑user apps or multiple application instances
- Need transactions, complex queries, or schema migrations
- Larger datasets requiring indexing or performance optimization
- Strong durability, replication, or backup requirements
- Security/audit guarantees and user permissions

## What Changed in the Code 🔧

The beauty of clean architecture really showed here—most of my code stayed exactly the same:

### New File Storage Implementation

```go
// dal/filestore/store.go
type FileStore struct {
    mu       sync.RWMutex
    filePath string
    data     map[int]domain.Todo
    nextID   int
}

func (fs *FileStore) Create(ctx context.Context, title string) (domain.Todo, error) {
    fs.mu.Lock()
    defer fs.mu.Unlock()

    todo := domain.Todo{
        ID:        fs.nextID,
        Title:     title,
        Done:      false,
        CreatedAt: time.Now(),
    }

    fs.data[fs.nextID] = todo
    fs.nextID++

    if err := fs.persist(); err != nil {
        // Rollback on persist failure
        delete(fs.data, fs.nextID-1)
        fs.nextID--
        return domain.Todo{}, fmt.Errorf("failed to persist: %w", err)
    }

    return todo, nil
}
```

## Atomic Writes for Safety

```go
func (fs *FileStore) persist() error {
    // Create temporary file
    tempFile := fs.filePath + ".tmp"

    file, err := os.Create(tempFile)
    if err != nil {
        return fmt.Errorf("failed to create temp file: %w", err)
    }
    defer file.Close()

    // Write JSON data
    encoder := json.NewEncoder(file)
    encoder.SetIndent("", "  ")

    if err := encoder.Encode(fs.data); err != nil {
        os.Remove(tempFile) // cleanup
        return fmt.Errorf("failed to encode data: %w", err)
    }

    // Force write to disk
    if err := file.Sync(); err != nil {
        os.Remove(tempFile)
        return fmt.Errorf("failed to sync: %w", err)
    }

    file.Close()

    // Atomic rename (the magic!)
    if err := os.Rename(tempFile, fs.filePath); err != nil {
        os.Remove(tempFile)
        return fmt.Errorf("failed to rename: %w", err)
    }

    return nil
}
```

### Key Implementation Details

**JSON serialization** with proper error handling
**sync.RWMutex** for safe concurrent reads/writes
**Atomic writes**: Write to temp file → fsync → os.Rename (prevents corruption)
**Configurable path**: Easy to change storage location
**Load on start**, flush on shutdown: Data persistence across restarts
**Domain and service** layers stayed identical—just swapped the adapter!

### Swapping Storage Adapters
This is where the ports & adapters pattern really shines:
```go
// cmd/main.go - Before
func main() {
    // todoRepo := inmemorytodo.NewInMemoryStore()

    // After - just one line change!
    todoRepo, err := filestore.NewFileStore("data/todos.json")
    if err != nil {
        log.Fatal("Failed to initialize file store:", err)
    }
    defer todoRepo.Close() // Graceful shutdown

    // Everything else stays the same
    todoService := todo.NewTodoService(todoRepo)
    // ... rest of wiring
}
```

No changes needed in:

- ✅ **Domain layer**
- ✅ **Service layer**
- ✅ **HTTP handlers**
- ✅ **DTOs or interfaces**

## Lessons Learned 📚

1. **Atomic operations matter**: Never write directly to your main file
2. **Error handling gets complex fast**: Rollback strategies are crucial
3. **Concurrency is tricky**: Even file I/O needs proper locking
4. **JSON is great for humans**: Easy to debug and inspect
5. **Clean architecture pays off**: Swapping storage was trivial

## What's Next 🚀

First of all, **lots of TESTS** 🤓 I need to verify:

- Crash recovery scenarios
- Concurrent read/write operations
- File corruption handling
- Load performance with larger datasets

Then:

- [ ] **Add Users**: Introduce ownership and authentication concepts
- [ ] **Move to PostgreSQL**: Real database with proper ACID properties
- [ ] **Migrations**: Schema versioning and data migration patterns
- [ ] **Indexes and optimization**: Query performance tuning
- [ ] **More comprehensive testing**: Integration tests, benchmarks, chaos engineering

## Performance Notes 📊

Early observations with file storage:

- **Read-heavy workloads**: Excellent (in-memory cache)
- **Write performance**: Limited by disk I/O and JSON marshaling
- **Concurrent reads**: Great with RWMutex
- **File size**: JSON isn't the most compact, but it's debuggable

For a todo app with hundreds of items, this approach is perfectly fine. For thousands or complex queries, database time!

## Why Share This Journey? 🌟

I'm learning in public because:

1. **Accountability**: Keeps me consistent and honest about progress
2. **Community**: Others can learn from my mistakes and successes
3. **Feedback**: Experienced developers catch things I miss
4. **Documentation**: Future me will thank present me

**Suggestions, corrections, and resource tips are very welcome!** If you're on a similar Go learning path, let's compare notes and learn together.

---

*This is Chapter 2 of my Go learning series. Follow along as I build from simple concepts to production-ready patterns, one layer at a time.*

**Next up**: Chapter 3 will tackle comprehensive testing strategies and introduce user management with proper authentication.

![Go Gopher with Floppy Disk](/my-programming-journey/images/go-gopher-floppy.png)
