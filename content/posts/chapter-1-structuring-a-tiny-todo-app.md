+++
date = '2025-09-19T10:57:00+02:00'
draft = true
title = 'Chapter 1: Building a Go Todo App - Architecture and Structure 🐹✨'
tags = ["golang", "learning", "architecture", "clean-architecture", "ddd", "testing", "web-development"]
categories = ["Go Journey"]
+++


I'm jumping into Go and documenting the journey so others can learn with me (and kindly correct me when I go off track). I'm starting simple, then growing step by step.

Today's focus: **project structure**

## The Evolution: From Prototype to Structure

I moved from a single-file-style prototype to a small layered layout inspired by hexagonal/DDD ideas. The goal: keep the core logic clean and make storage and delivery swappable.

Here's what my project structure looks like now:

**📁 go-todo-chi/**
- **📁 cmd/**
  - `main.go` — app entry + wiring
- **📁 domain/**
  - `todo.go` — entities + invariants
- **📁 services/todo/**
  - `interfaces.go` — ports (use cases)
  - `service.go` — business logic
  - `factory.go` — lightweight DI/composition
- **📁 delivery/web/**
  - `interfaces.go` — handler contracts
  - `dto.go` — request/response DTOs
  - `todohandler.go` — HTTP handlers (chi)
  - `server.go` — router/bootstrap
- **📁 dal/inmemorytodo/**
  - `store.go` — in-memory repo (first adapter)
- `go.mod`, `go.sum` — Go module files

## Core Principles Behind the Structure

### 🎯 **Start Simple**
I began with an in-memory store first to focus on domain behavior without getting caught up in database complexities. This let me nail down the business logic before worrying about persistence.

### 🔌 **Ports & Adapters**
Interfaces live in the service layer, concrete implementations live in their respective packages (`dal/`, `delivery/`). This keeps dependencies pointing inward toward the domain.

### 🧩 **Separation of Concerns**
- **Domain**: Pure business entities and rules
- **Services**: Use cases and business logic
- **Delivery**: HTTP transport layer (handlers, routing)
- **DAL**: Data access layer (repositories, storage)

### 🔄 **Swappable Storage**
Same service layer, different repository implementations. Today it's in-memory, tomorrow it's file-based, next week it's PostgreSQL.

### 📦 **Small, Testable Units**
Each package has one clear reason to change, making testing straightforward and focused.

## The Domain Layer

Starting with the core entity:

```go
// domain/todo.go
package domain

import "time"

type Todo struct {
    ID        int       `json:"id"`
    Title     string    `json:"title"`
    Done      bool      `json:"done"`
    CreatedAt time.Time `json:"createdAt"`
}

// Business rules live here
func (t *Todo) MarkComplete() {
    t.Done = true
}

func (t *Todo) Validate() error {
    if t.Title == "" {
        return errors.New("title cannot be empty")
    }
    if len(t.Title) > 255 {
        return errors.New("title too long")
    }
    return nil
}

```
No external dependencies. Pure business logic.

## The Service Layer

This is where use cases live:

```go

// services/todo/interfaces.go
package todo

import (
    "context"
    "github.com/macesz/todo-go/domain"
)

type Repository interface {
    Create(ctx context.Context, title string) (domain.Todo, error)
    List(ctx context.Context) ([]domain.Todo, error)
    GetByID(ctx context.Context, id int) (domain.Todo, error)
    Update(ctx context.Context, id int, title string, done bool) (domain.Todo, error)
    Delete(ctx context.Context, id int) error
}

type Service interface {
    CreateTodo(ctx context.Context, title string) (domain.Todo, error)
    ListTodos(ctx context.Context) ([]domain.Todo, error)
    GetTodo(ctx context.Context, id int) (domain.Todo, error)
    UpdateTodo(ctx context.Context, id int, title string, done bool) (domain.Todo, error)
    DeleteTodo(ctx context.Context, id int) error
}
```

The service implementation adds business logic on top of repository operations:

// services/todo/service.go
```go
func (s *TodoService) CreateTodo(ctx context.Context, title string) (domain.Todo, error) {
    // Validation happens here
    if strings.TrimSpace(title) == "" {
        return domain.Todo{}, errors.New("title cannot be empty")
    }

    // Delegate to repository
    return s.repo.Create(ctx, strings.TrimSpace(title))
}
```

## The HTTP Layer

Chi router with clean handler separation:

// delivery/web/todohandler.go
```go
type TodoHandlers struct {
    Service todo.Service
}

func (h *TodoHandlers) CreateTodo(w http.ResponseWriter, r *http.Request) {
    var req CreateTodoRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }

    todo, err := h.Service.CreateTodo(r.Context(), req.Title)
    if err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }

    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusCreated)
    json.NewEncoder(w).Encode(todo)
}
```

## The Storage Layer

Starting simple with in-memory:

```go
// dal/inmemorytodo/store.go

type InMemoryStore struct {
    mu     sync.RWMutex
    data   map[int]domain.Todo
    nextID int
}

func (s *InMemoryStore) Create(ctx context.Context, title string) (domain.Todo, error) {
    s.mu.Lock()
    defer s.mu.Unlock()

    todo := domain.Todo{
        ID:        s.nextID,
        Title:     title,
        Done:      false,
        CreatedAt: time.Now(),
    }

    s.data[s.nextID] = todo
    s.nextID++

    return todo, nil
}
```

## Dependency Injection & Wiring

Clean composition in main:

```go
// cmd/main.go
func main() {
    // Storage layer
    todoRepo := inmemorytodo.NewInMemoryStore()

    // Service layer
    todoService := todo.NewTodoService(todoRepo)

    // HTTP layer
    todoHandlers := &web.TodoHandlers{Service: todoService}

    // Router
    r := chi.NewRouter()
    r.Post("/todos", todoHandlers.CreateTodo)
    r.Get("/todos", todoHandlers.ListTodos)
    // ... other routes

    log.Println("Server starting on :8080")
    log.Fatal(http.ListenAndServe(":8080", r))
}
```

### What's Next in This Journey

#### Here's my roadmap as I continue learning Go:

✅ Completed
 In-memory repository + unit tests
 Service layer with business logic
 HTTP handlers with Chi router
 Comprehensive test coverage (service, storage, handlers)

🔜 Coming Next
 File storage: JSON persistence for I/O + error handling
 Integration tests: All layers working together
 Middleware: CORS, logging, error handling

⏭️ Future Chapters
 Database adapter: SQLite → PostgreSQL with database/sql
 Performance: Benchmarks and profiling
 Observability: Structured logging, metrics, tracing
 Security & Auth: JWT, rate limiting, secure headers
 Deployment: Docker, CI/CD, graceful shutdown

#### Key Learnings So Far:

Go's interface system is brilliant for dependency inversion
Explicit dependency injection keeps things clear and testable
Small interfaces are easier to implement and test
Package structure in Go encourages good separation of concerns
Testing each layer independently builds confidence

#### Why I'm Sharing This Journey

I'm excited to learn a new language and "learn in public." If you see better ways to structure things in Go—or resources I should read—please share your thoughts!

The code is growing organically from simple to more sophisticated. Each chapter will tackle new challenges while maintaining clean architecture principles.

What would you do differently? I'm always open to feedback and better approaches!

This is Chapter 1 of my Go learning series. Follow along as I build from a simple Todo app to a production-ready service, making mistakes and learning along the way!

Next up: Chapter 2 will dive into file-based storage and error handling patterns in Go.
