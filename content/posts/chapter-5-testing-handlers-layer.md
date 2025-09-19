+++
date = '2025-09-19T16:20:40+02:00'
draft = false
title = 'Chapter 5: Testing HTTP Handlers - From HTTP Requests to JSON Responses 🌐🧪'
categories = ["Go Journey"]
tags = ["golang", "testing", "learning-in-public", "clean-architecture", "web-development", "http-handlers", "unit-testing","mocking", "httptest", "chi-router", "testify", "rest-api", ]
+++

Time to test the web layer! After conquering service logic and file storage, I'm now tackling HTTP handlers—where HTTP requests meet business logic. This is where JSON parsing, status codes, URL parameters, and all the web-specific concerns come into play.

**Repo/branch**: [https://github.com/macesz/todo-go/tree/handlers-web-unit-tests/delivery/web](https://github.com/macesz/todo-go/tree/handlers-web-unit-tests/delivery/web)


## Why HTTP Handler Testing Is Unique 🎯

Testing HTTP handlers brings a whole new set of challenges compared to service or storage layers:

### **HTTP Protocol Concerns**
- **Request/Response cycle**: Simulating full HTTP interactions
- **Status codes**: Ensuring correct HTTP semantics (200, 201, 400, 404, 500)
- **Headers**: Content-Type, Accept, custom headers
- **HTTP methods**: GET, POST, PUT, DELETE behavior differences

### **JSON Serialization**
- **Request parsing**: JSON unmarshaling from request body
- **Response formatting**: JSON marshaling to response body
- **Validation**: Ensuring data integrity at the HTTP boundary
- **Error formatting**: Consistent error response structure

### **Routing and Parameters**
- **URL parameters**: Path variables like `/todos/{id}`
- **Query parameters**: Optional filters and pagination
- **Route matching**: Ensuring handlers respond to correct routes
- **Context propagation**: Router context, request context

### **Integration Concerns**
- **Service layer interaction**: Mocking dependencies properly
- **Middleware**: Authentication, logging, CORS
- **Content negotiation**: Accept headers, response formats

## Testing Tools and Setup 🛠️

The Go standard library provides excellent HTTP testing tools:

```go
import (
    "context"
    "errors"
    "net/http"
    "net/http/httptest"  // 🌟 The MVP for HTTP testing
    "strings"
    "testing"
    "time"

    chi "github.com/go-chi/chi/v5"
    "github.com/macesz/todo-go/delivery/web/mocks"
    "github.com/macesz/todo-go/domain"
    "github.com/stretchr/testify/mock"
)
```

### **Key Testing Components**

- **`httptest.NewRecorder()`**: Records HTTP responses for testing
- **`httptest.NewRequest()`**: Creates HTTP requests for testing
- **`chi.NewRouteContext()`**: Simulates URL parameters for chi router
- **Mock services**: Isolate handler logic from business logic

## Key Testing Patterns I Discovered 🔍

### 1. **Simple Handler Testing - Health Check**

Starting with the simplest case - a static response handler:

```go
func TestHealthCheckHandler(t *testing.T) {
    // Create HTTP request
    req, err := http.NewRequest("GET", "/health", nil)
    if err != nil {
        t.Fatal(err)
    }

    // Create response recorder to capture handler output
    rr := httptest.NewRecorder()
    handler := http.HandlerFunc(HealthCheckHandler)

    // Execute the handler
    handler.ServeHTTP(rr, req)

    // Verify status code
    if status := rr.Code; status != http.StatusOK {
        t.Errorf("handler returned wrong status code: got %v want %v",
            status, http.StatusOK)
    }

    // Verify response body
    expected := `{"alive": true}`
    if rr.Body.String() != expected {
        t.Errorf("handler returned unexpected body: got %v want %v",
            rr.Body.String(), expected)
    }
}
```

**Why start here?**
- **No dependencies**: Tests handler logic in isolation
- **Simple assertions**: Status code and response body
- **Foundation pattern**: Template for more complex tests

### 2. **Table-Driven Handler Tests with Mocks**

Testing different scenarios with the same handler structure:

```go
func TestListTodos(t *testing.T) {
    fixedTime := time.Date(2024, time.January, 1, 12, 0, 0, 0, time.UTC)

    tests := []struct {
        name           string
        mockReturn     []domain.Todo
        mockError      error
        expectedStatus int
        expectedBody   string
    }{
        {
            name:           "No todos",
            mockReturn:     []domain.Todo{},
            mockError:      nil,
            expectedStatus: http.StatusOK,
            expectedBody:   "[]\n",
        },
        {
            name: "One todo",
            mockReturn: []domain.Todo{
                {ID: 1, Title: "Test Todo 1", Done: false, CreatedAt: fixedTime},
            },
            mockError:      nil,
            expectedStatus: http.StatusOK,
            expectedBody:   `[{"ID":1,"Title":"Test Todo 1","Done":false,"CreatedAt":"2024-01-01T12:00:00Z"}]` + "\n",
        },
        {
            name:           "Service error",
            mockReturn:     nil,
            mockError:      http.ErrServerClosed,
            expectedStatus: http.StatusInternalServerError,
            expectedBody:   `{"error":"http: Server closed"}` + "\n",
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            // Create fresh mock for each test case
            mockService := mocks.NewTodoService(t)
            mockService.On("ListTodos", mock.Anything).Return(tt.mockReturn, tt.mockError)

            // Set up handler with mock
            handlers := &TodoHandlers{Service: mockService}
            handler := http.HandlerFunc(handlers.ListTodos)

            // Execute HTTP request
            rr := httptest.NewRecorder()
            req, err := http.NewRequest("GET", "/todos", nil)
            if err != nil {
                t.Fatal(err)
            }

            handler.ServeHTTP(rr, req)

            // Verify results
            if status := rr.Code; status != tt.expectedStatus {
                t.Errorf("handler returned wrong status code: got %v want %v",
                    status, tt.expectedStatus)
            }

            if rr.Body.String() != tt.expectedBody {
                t.Errorf("handler returned unexpected body: got %v want %v",
                    rr.Body.String(), tt.expectedBody)
            }

            // Ensure mock was called as expected
            mockService.AssertExpectations(t)
            rr.Body.Reset()
        })
    }
}
```

**Key patterns here:**
- **Fixed timestamps**: Predictable JSON output for assertions
- **Fresh mocks per test**: Prevents state contamination
- **Error scenario coverage**: Service errors mapped to HTTP status codes
- **AssertExpectations**: Verifies service layer was called correctly

### 3. **Testing JSON Request Bodies - POST/PUT Handlers**

Testing handlers that parse JSON from request bodies:

```go
func TestCreateTodo(t *testing.T) {
    fixedTime := time.Date(2024, time.January, 1, 12, 0, 0, 0, time.UTC)

    tests := []struct {
        name           string
        inputBody      string
        mockReturn     domain.Todo
        mockError      error
        expectedStatus int
        expectedBody   string
    }{
        {
            name:           "Valid input",
            inputBody:      `{"Title": "New Todo"}`,
            mockReturn:     domain.Todo{ID: 1, Title: "New Todo", Done: false, CreatedAt: fixedTime},
            mockError:      nil,
            expectedStatus: http.StatusCreated,
            expectedBody:   `{"id":1,"title":"New Todo","done":false,"createdAt":"2024-01-01T12:00:00Z"}`,
        },
        {
            name:           "Invalid JSON",
            inputBody:      `{"Title": "New Todo"`, // Malformed JSON
            mockReturn:     domain.Todo{},
            mockError:      nil,
            expectedStatus: http.StatusBadRequest,
            expectedBody:   `"unexpected EOF"`,
        },
        {
            name:           "Service error",
            inputBody:      `{"Title": "New Todo"}`,
            mockReturn:     domain.Todo{},
            mockError:      errors.New("error creating todo"),
            expectedStatus: http.StatusBadRequest,
            expectedBody:   `"error creating todo"`,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            mockService := mocks.NewTodoService(t)

            // Only set up mock expectation for valid cases
            if tt.mockError != nil || tt.mockReturn.ID != 0 {
                mockService.On("CreateTodo", mock.Anything, "New Todo").
                    Return(tt.mockReturn, tt.mockError)
            }

            handlers := &TodoHandlers{Service: mockService}
            handler := http.HandlerFunc(handlers.CreateTodo)

            // Create request with JSON body
            rr := httptest.NewRecorder()
            rbody := strings.NewReader(tt.inputBody)
            req, err := http.NewRequest("POST", "/todos", rbody)
            if err != nil {
                t.Fatal(err)
            }

            handler.ServeHTTP(rr, req)

            // Verify HTTP response
            if status := rr.Code; status != tt.expectedStatus {
                t.Errorf("handler returned wrong status code: got %v want %v",
                    status, tt.expectedStatus)
            }

            if strings.TrimSpace(rr.Body.String()) != tt.expectedBody {
                t.Errorf("handler returned unexpected body: got %v want %v",
                    rr.Body.String(), tt.expectedBody)
            }

            mockService.AssertExpectations(t)
        })
    }
}
```

**Critical testing points:**
- **JSON parsing errors**: Malformed JSON should return 400 Bad Request
- **Request body handling**: Using `strings.NewReader()` to simulate request body
- **Different status codes**: 201 Created vs 400 Bad Request based on scenario
- **Service layer isolation**: Mock prevents dependency on actual business logic

### 4. **URL Parameter Testing with Chi Router**

The trickiest part—testing handlers that extract URL parameters:

```go
func TestGetTodo(t *testing.T) {
    fixedTime := time.Date(2024, time.January, 1, 12, 0, 0, 0, time.UTC)

    tests := []struct {
        name           string
        url            string
        mockReturn     domain.Todo
        mockError      error
        expectedStatus int
        expectedBody   string
    }{
        {
            name:           "Valid ID",
            url:            "/todos/1",
            mockReturn:     domain.Todo{ID: 1, Title: "Test Todo", Done: false, CreatedAt: fixedTime},
            mockError:      nil,
            expectedStatus: http.StatusOK,
            expectedBody:   `{"id":1,"title":"Test Todo","done":false,"createdAt":"2024-01-01T12:00:00Z"}`,
        },
        {
            name:           "Non-integer ID",
            url:            "/todos/abc",
            mockReturn:     domain.Todo{},
            mockError:      nil,
            expectedStatus: http.StatusBadRequest,
            expectedBody:   `{"error":"id must be an integer"}`,
        },
        {
            name:           "Todo not found",
            url:            "/todos/999",
            mockReturn:     domain.Todo{},
            mockError:      errors.New("not found"),
            expectedStatus: http.StatusNotFound,
            expectedBody:   `{"error":"not found"}`,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            mockService := mocks.NewTodoService(t)

            if tt.mockError != nil || tt.mockReturn.ID != 0 {
                mockService.On("GetTodo", mock.Anything, mock.AnythingOfType("int")).
                    Return(tt.mockReturn, tt.mockError)
            }

            handler := &TodoHandlers{Service: mockService}
            rr := httptest.NewRecorder()
            req := httptest.NewRequest("GET", tt.url, nil)

            // 🌟 The magic: Simulate chi router URL parameters
            rctx := chi.NewRouteContext()
            // Extract ID from URL path and set as URL parameter
            rctx.URLParams.Add("id", strings.TrimPrefix(tt.url, "/todos/"))
            // Add routing context to request context
            req = req.WithContext(context.WithValue(req.Context(), chi.RouteCtxKey, rctx))

            // Call handler method directly (not through ServeHTTP)
            handler.GetTodo(rr, req)

            // Verify response
            if status := rr.Code; status != tt.expectedStatus {
                t.Errorf("handler returned wrong status code: got %v want %v",
                    status, tt.expectedStatus)
            }

            if strings.TrimSpace(rr.Body.String()) != tt.expectedBody {
                t.Errorf("handler returned unexpected body: got %v want %v",
                    rr.Body.String(), tt.expectedBody)
            }

            mockService.AssertExpectations(t)
        })
    }
}
```

**The URL parameter testing breakthrough:**
This was the trickiest part! Chi router normally extracts URL parameters automatically, but in tests, we need to manually simulate this:

1. **Create route context**: `chi.NewRouteContext()`
2. **Add URL parameters**: `rctx.URLParams.Add("id", value)`
3. **Attach to request**: `req.WithContext(context.WithValue(...))`
4. **Call handler directly**: `handler.GetTodo(rr, req)` instead of `ServeHTTP`

### 5. **Complex Handler Testing - Update with Body and Parameters**

Testing handlers that need both URL parameters AND JSON body:

```go
func TestUpdateTodo(t *testing.T) {
    fixedTime := time.Date(2024, time.January, 1, 12, 0, 0, 0, time.UTC)

    tests := []struct {
        name           string
        urlParam       string
        inputBody      string
        mockID         int    // Expected ID for mock
        mockTitle      string // Expected title for mock
        mockDone       bool   // Expected done for mock
        shouldCallMock bool   // Whether to expect service call
        mockReturn     domain.Todo
        mockError      error
        expectedStatus int
        expectedBody   string
    }{
        {
            name:      "Valid input",
            urlParam:  "1",
            inputBody: `{"title": "Updated Todo", "done": true}`,
            mockID:    1,
            mockTitle: "Updated Todo",
            mockDone:  true,
            mockReturn: domain.Todo{
                ID: 1, Title: "Updated Todo", Done: true, CreatedAt: fixedTime,
            },
            mockError:      nil,
            shouldCallMock: true,
            expectedStatus: http.StatusOK,
            expectedBody:   `{"id":1,"title":"Updated Todo","done":true,"createdAt":"2024-01-01T12:00:00Z"}`,
        },
        {
            name:           "Invalid JSON",
            urlParam:       "1",
            inputBody:      `{"title": "Updated Todo", "done": true`, // Missing closing brace
            shouldCallMock: false,
            expectedStatus: http.StatusBadRequest,
            expectedBody:   `"unexpected EOF"`,
        },
        {
            name:           "Invalid data (empty title)",
            urlParam:       "1",
            inputBody:      `{"title": "", "done": true}`,
            shouldCallMock: false,
            expectedStatus: http.StatusBadRequest,
            expectedBody:   `{"error":"title is required and must be between 1 and 255 characters; done is required"}`,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            mockService := mocks.NewTodoService(t)

            // Only set up mock expectations for cases that should call the service
            if tt.shouldCallMock {
                mockService.On("UpdateTodo", mock.Anything, tt.mockID, tt.mockTitle, tt.mockDone).
                    Return(tt.mockReturn, tt.mockError)
            }

            handlers := &TodoHandlers{Service: mockService}

            rr := httptest.NewRecorder()
            rbody := strings.NewReader(tt.inputBody)

            req, err := http.NewRequest("PUT", "/todos/"+tt.urlParam, rbody)
            if err != nil {
                t.Fatal(err)
            }

            // Set Content-Type header (important for JSON parsing!)
            req.Header.Set("Content-Type", "application/json")

            // Set up chi router context for URL parameters
            rctx := chi.NewRouteContext()
            rctx.URLParams.Add("id", tt.urlParam)
            req = req.WithContext(context.WithValue(req.Context(), chi.RouteCtxKey, rctx))

            handlers.UpdateTodo(rr, req)

            if status := rr.Code; status != tt.expectedStatus {
                t.Errorf("handler returned wrong status code: got %v want %v",
                    status, tt.expectedStatus)
            }

            gotBody := strings.TrimSpace(rr.Body.String())
            if gotBody != tt.expectedBody {
                t.Errorf("handler returned unexpected body:\ngot:  %q\nwant: %q",
                    gotBody, tt.expectedBody)
            }

            mockService.AssertExpectations(t)
        })
    }
}
```

**Key complexity here:**
- **`shouldCallMock` pattern**: Only mock service calls for valid input cases
- **Content-Type header**: Critical for JSON parsing—forgot this initially!
- **Both URL and body validation**: Test parameter parsing AND JSON parsing
- **Precise mock expectations**: Match exact parameters passed to service

## What I'm Testing ✅

### **HTTP Status Codes**
- **200 OK**: Successful GET requests
- **201 Created**: Successful POST requests
- **204 No Content**: Successful DELETE requests
- **400 Bad Request**: Invalid input, malformed JSON
- **404 Not Found**: Resource doesn't exist
- **500 Internal Server Error**: Service layer errors

### **JSON Serialization**
- **Request unmarshaling**: Parse JSON from request body
- **Response marshaling**: Convert structs to JSON response
- **Field mapping**: Ensure proper JSON tag handling
- **Error formatting**: Consistent error response structure

### **URL Parameter Handling**
- **Path variables**: `/todos/{id}` parameter extraction
- **Type conversion**: String to integer conversion
- **Validation**: Non-integer IDs, missing IDs
- **Router integration**: Chi router context simulation

### **Request Validation**
- **Required fields**: Title must not be empty
- **Field constraints**: Title length, boolean values
- **JSON structure**: Proper JSON syntax
- **Content-Type headers**: Proper request formatting

### **Error Scenarios**
- **Malformed JSON**: Syntax errors in request body
- **Service layer errors**: Business logic failures
- **Not found cases**: Requesting non-existent resources
- **Validation failures**: Invalid input data

## Key Learnings and Gotchas 🎓

### 1. **Fresh Mocks Are Critical**

Each test needs its own mock instance to prevent state contamination:

```go
// ✅ Good: Fresh mock per test
for _, tt := range tests {
    t.Run(tt.name, func(t *testing.T) {
        mockService := mocks.NewTodoService(t) // Fresh mock
        // ... test logic
    })
}

// ❌ Bad: Shared mock across tests
mockService := mocks.NewTodoService(t)
for _, tt := range tests {
    // Mock state leaks between tests!
}
```

### 2. **Content-Type Headers Matter**

Forgot this initially and spent time debugging why JSON wasn't parsing:

```go
req, err := http.NewRequest("PUT", "/todos/1", rbody)
req.Header.Set("Content-Type", "application/json") // 🌟 Don't forget!
```

### 3. **URL Parameter Testing is Router-Specific**

Each router framework has different testing approaches:
- **Chi**: Requires manual route context setup
- **Gorilla/mux**: Different context handling
- **Gin**: Test context creation

Understanding your router's testing patterns is crucial!

### 4. **Status Code Mapping is Business Logic**

Deciding which errors map to which HTTP status codes is important design:

```go
// Business decision: What errors are 400 vs 500?
if err == ErrNotFound {
    return http.StatusNotFound  // 404
}
if err == ErrValidation {
    return http.StatusBadRequest // 400
}
// Default: Internal server error
return http.StatusInternalServerError // 500
```

### 5. **`strings.TrimSpace()` Saves Debugging Time**

JSON encoding often adds trailing newlines:

```go
// ✅ Good: Handle whitespace differences
if strings.TrimSpace(rr.Body.String()) != tt.expectedBody {
    // Compare without whitespace issues
}

// ❌ Bad: Brittle due to whitespace
if rr.Body.String() != tt.expectedBody {
    // Fails due to trailing newlines
}
```

## Common HTTP Testing Patterns 📋

### **Basic Handler Test Template**
```go
func TestHandler(t *testing.T) {
    // Arrange: Set up mock, request, recorder
    mockService := mocks.NewService(t)
    mockService.On("Method", args...).Return(result, error)

    handler := &Handlers{Service: mockService}
    rr := httptest.NewRecorder()
    req := httptest.NewRequest("GET", "/path", nil)

    // Act: Execute handler
    handler.Method(rr, req)

    // Assert: Verify response
    assert.Equal(t, expectedStatus, rr.Code)
    assert.Equal(t, expectedBody, rr.Body.String())
    mockService.AssertExpectations(t)
}
```

### **JSON Body Test Template**
```go
func TestJSONHandler(t *testing.T) {
    jsonBody := `{"field": "value"}`
    req := httptest.NewRequest("POST", "/path", strings.NewReader(jsonBody))
    req.Header.Set("Content-Type", "application/json")

    rr := httptest.NewRecorder()
    handler.ServeHTTP(rr, req)

    // Verify JSON response
    var response ResponseStruct
    err := json.Unmarshal(rr.Body.Bytes(), &response)
    assert.NoError(t, err)
    assert.Equal(t, expected, response)
}
```

### **URL Parameter Test Template**
```go
func TestURLParamHandler(t *testing.T) {
    req := httptest.NewRequest("GET", "/path/123", nil)

    // Set up router context (chi-specific)
    rctx := chi.NewRouteContext()
    rctx.URLParams.Add("id", "123")
    req = req.WithContext(context.WithValue(req.Context(), chi.RouteCtxKey, rctx))

    rr := httptest.NewRecorder()
    handler.Method(rr, req)

    assert.Equal(t, http.StatusOK, rr.Code)
}
```

## Testing Strategy Evolution 📈

My HTTP handler testing approach evolved through this chapter:

### **Phase 1: Basic Status Code Testing**
```go
// Started simple: Just verify it doesn't crash
if rr.Code != http.StatusOK {
    t.Error("Expected 200")
}
```

### **Phase 2: Response Body Verification**
```go
// Added content verification
expected := `{"alive": true}`
if rr.Body.String() != expected {
    t.Errorf("Wrong body: got %v want %v", rr.Body.String(), expected)
}
```

### **Phase 3: Table-Driven with Mocks**
```go
// Comprehensive scenarios with service mocking
tests := []struct {
    name           string
    mockReturn     interface{}
    mockError      error
    expectedStatus int
    expectedBody   string
}{
    // Multiple test cases...
}
```

### **Phase 4: Complex Integration Testing**
```go
// URL params + JSON body + headers + context
req.Header.Set("Content-Type", "application/json")
rctx := chi.NewRouteContext()
rctx.URLParams.Add("id", "123")
req = req.WithContext(context.WithValue(req.Context(), chi.RouteCtxKey, rctx))
```

## Performance and Coverage 📊

HTTP handler tests are slower than service tests due to HTTP overhead:

```bash
$ go test -v ./delivery/web
=== RUN   TestHealthCheckHandler
--- PASS: TestHealthCheckHandler (0.00s)
=== RUN   TestListTodos
--- PASS: TestListTodos (0.01s)
=== RUN   TestCreateTodo
--- PASS: TestCreateTodo (0.01s)
=== RUN   TestGetTodo
--- PASS: TestGetTodo (0.01s)
=== RUN   TestUpdateTodo
--- PASS: TestUpdateTodo (0.01s)
=== RUN   TestDeleteTodo
--- PASS: TestDeleteTodo (0.01s)
PASS
ok      github.com/macesz/todo-go/delivery/web    0.058s
```

**58ms** - slower than service layer (3ms) but still very fast for HTTP testing!

### **Current Coverage: 91%** 🎯

Missing coverage areas:
- Some error handling edge cases
- Middleware integration (i dont have any yet)
- Headers and content negotiation

## Architecture Benefits Shine Through 🏗️

The clean architecture from Chapter 1 makes HTTP testing straightforward:

- **Service layer mocking**: Handler logic isolated from business logic
- **Dependency injection**: Easy to swap real services for mocks
- **Clear interfaces**: TodoService interface makes mocking simple
- **Separation of concerns**: HTTP concerns separate from business logic

Testing each layer in isolation builds confidence in the overall system!


## Coming Up in This Testing Series 🚀

### 🔜 **Chapter 6: Integration Tests**
- All layers working together
- End-to-end HTTP request flows
- Database integration with testcontainers
- Real file I/O + HTTP + service integration

## Community Questions 💬

I'm curious about the community's experience:

- **How do you handle router-specific testing patterns** in your HTTP tests?
- **What's your approach to mocking vs integration** for web layer tests?
- **Any tools or libraries** that make HTTP testing easier in Go?
- **Common gotchas** you've encountered testing JSON APIs?
- **How do you balance test speed vs realism** in HTTP handler tests?

Share your experiences and war stories! Let's learn from each other's wins and failures.

---

*This is Chapter 5 of my Go learning series. Follow along as I build from simple concepts to production-ready patterns, one layer at a time.*

**Next up**: Chapter 6 will bring all layers together in comprehensive integration tests, showing how service + storage + HTTP layers work as a cohesive system.
