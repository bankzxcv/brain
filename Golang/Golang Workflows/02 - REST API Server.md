---
title: "Go Workflow: REST API Server"
date: 2026-04-07
tags:
  - golang
  - workflow
  - project
  - rest-api
  - net-http
  - json
  - middleware
  - context
parent: "[[Golang Study]]"
---

# Go Workflow: REST API Server

## Overview

Build a RESTful API for a bookstore using only the standard library. You will implement full CRUD operations, JSON request/response handling, middleware (logging, CORS, authentication), and graceful shutdown.

**Concepts practiced:** `net/http`, JSON encoding/decoding, structs, maps, error handling, middleware pattern, `context`, `sync.Mutex`, `os/signal`.

## Features

- CRUD operations for books
- In-memory store with thread-safe access
- JSON request/response with proper HTTP status codes
- Middleware: logging, request ID, CORS, basic auth
- Search and filtering via query parameters
- Graceful shutdown

---

## Step-by-Step Implementation

### Step 1: Define the Book Struct and In-Memory Store

```go
package main

import (
	"sync"
	"time"
)

type Book struct {
	ID          string    `json:"id"`
	Title       string    `json:"title"`
	Author      string    `json:"author"`
	ISBN        string    `json:"isbn"`
	PublishedAt string    `json:"published_at"`
	Price       float64   `json:"price"`
	Genre       string    `json:"genre"`
	CreatedAt   time.Time `json:"created_at"`
	UpdatedAt   time.Time `json:"updated_at"`
}

type BookStore struct {
	mu     sync.RWMutex
	books  map[string]Book
	nextID int
}

func NewBookStore() *BookStore {
	return &BookStore{
		books:  make(map[string]Book),
		nextID: 1,
	}
}
```

> [!tip] RWMutex vs Mutex
> Use `sync.RWMutex` when reads are far more frequent than writes. Multiple goroutines can hold a read lock simultaneously, but a write lock is exclusive. HTTP servers handle requests concurrently, so this matters.

---

### Step 2: Implement Store Methods

```go
import (
	"fmt"
	"strings"
)

func (s *BookStore) Create(b Book) Book {
	s.mu.Lock()
	defer s.mu.Unlock()

	b.ID = fmt.Sprintf("%d", s.nextID)
	s.nextID++
	b.CreatedAt = time.Now()
	b.UpdatedAt = b.CreatedAt
	s.books[b.ID] = b
	return b
}

func (s *BookStore) Get(id string) (Book, bool) {
	s.mu.RLock()
	defer s.mu.RUnlock()

	book, ok := s.books[id]
	return book, ok
}

func (s *BookStore) GetAll() []Book {
	s.mu.RLock()
	defer s.mu.RUnlock()

	books := make([]Book, 0, len(s.books))
	for _, b := range s.books {
		books = append(books, b)
	}
	return books
}

func (s *BookStore) Update(id string, updated Book) (Book, error) {
	s.mu.Lock()
	defer s.mu.Unlock()

	existing, ok := s.books[id]
	if !ok {
		return Book{}, fmt.Errorf("book %s not found", id)
	}

	updated.ID = existing.ID
	updated.CreatedAt = existing.CreatedAt
	updated.UpdatedAt = time.Now()
	s.books[id] = updated
	return updated, nil
}

func (s *BookStore) Delete(id string) error {
	s.mu.Lock()
	defer s.mu.Unlock()

	if _, ok := s.books[id]; !ok {
		return fmt.Errorf("book %s not found", id)
	}
	delete(s.books, id)
	return nil
}

func (s *BookStore) Search(query string) []Book {
	s.mu.RLock()
	defer s.mu.RUnlock()

	query = strings.ToLower(query)
	var results []Book
	for _, b := range s.books {
		if strings.Contains(strings.ToLower(b.Title), query) ||
			strings.Contains(strings.ToLower(b.Author), query) ||
			strings.Contains(strings.ToLower(b.ISBN), query) {
			results = append(results, b)
		}
	}
	return results
}
```

---

### Step 3: Build HTTP Handlers

Create a handler struct that holds a reference to the store.

```go
import (
	"encoding/json"
	"net/http"
)

type APIHandler struct {
	store *BookStore
}

// Helper: write JSON response
func writeJSON(w http.ResponseWriter, status int, data any) {
	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(status)
	json.NewEncoder(w).Encode(data)
}

// Helper: write error response
type ErrorResponse struct {
	Error   string `json:"error"`
	Message string `json:"message"`
}

func writeError(w http.ResponseWriter, status int, msg string) {
	writeJSON(w, status, ErrorResponse{
		Error:   http.StatusText(status),
		Message: msg,
	})
}

// Helper: extract path parameter (e.g., /books/123 -> "123")
func extractID(path, prefix string) string {
	return strings.TrimPrefix(path, prefix)
}
```

> [!info] Go 1.22+ Routing
> Go 1.22 added method and path parameter support to `http.ServeMux`: patterns like `GET /books/{id}`. If using Go 1.22+, you can use `r.PathValue("id")` instead of manual path parsing. This solution works with both old and new approaches.

---

### Step 4: Implement Each Handler

```go
func (h *APIHandler) handleBooks(w http.ResponseWriter, r *http.Request) {
	switch r.Method {
	case http.MethodGet:
		// Check for search query
		query := r.URL.Query().Get("q")
		genre := r.URL.Query().Get("genre")

		var books []Book
		if query != "" {
			books = h.store.Search(query)
		} else {
			books = h.store.GetAll()
		}

		// Filter by genre
		if genre != "" {
			filtered := make([]Book, 0)
			for _, b := range books {
				if strings.EqualFold(b.Genre, genre) {
					filtered = append(filtered, b)
				}
			}
			books = filtered
		}

		writeJSON(w, http.StatusOK, books)

	case http.MethodPost:
		var book Book
		if err := json.NewDecoder(r.Body).Decode(&book); err != nil {
			writeError(w, http.StatusBadRequest, "Invalid JSON body")
			return
		}
		if book.Title == "" || book.Author == "" {
			writeError(w, http.StatusBadRequest, "Title and author are required")
			return
		}
		created := h.store.Create(book)
		writeJSON(w, http.StatusCreated, created)

	default:
		writeError(w, http.StatusMethodNotAllowed, "Method not allowed")
	}
}

func (h *APIHandler) handleBook(w http.ResponseWriter, r *http.Request) {
	id := extractID(r.URL.Path, "/books/")
	if id == "" {
		writeError(w, http.StatusBadRequest, "Book ID is required")
		return
	}

	switch r.Method {
	case http.MethodGet:
		book, ok := h.store.Get(id)
		if !ok {
			writeError(w, http.StatusNotFound, "Book not found")
			return
		}
		writeJSON(w, http.StatusOK, book)

	case http.MethodPut:
		var book Book
		if err := json.NewDecoder(r.Body).Decode(&book); err != nil {
			writeError(w, http.StatusBadRequest, "Invalid JSON body")
			return
		}
		updated, err := h.store.Update(id, book)
		if err != nil {
			writeError(w, http.StatusNotFound, err.Error())
			return
		}
		writeJSON(w, http.StatusOK, updated)

	case http.MethodDelete:
		if err := h.store.Delete(id); err != nil {
			writeError(w, http.StatusNotFound, err.Error())
			return
		}
		w.WriteHeader(http.StatusNoContent)

	default:
		writeError(w, http.StatusMethodNotAllowed, "Method not allowed")
	}
}
```

---

### Step 5: Middleware

Middleware in Go follows a simple pattern: a function that takes an `http.Handler` and returns an `http.Handler`.

```go
import (
	"context"
	"crypto/rand"
	"encoding/hex"
	"log"
)

type contextKey string

const requestIDKey contextKey = "requestID"

// Logging middleware
func loggingMiddleware(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		start := time.Now()
		reqID, _ := r.Context().Value(requestIDKey).(string)

		// Wrap ResponseWriter to capture status code
		wrapped := &statusRecorder{ResponseWriter: w, status: http.StatusOK}
		next.ServeHTTP(wrapped, r)

		log.Printf("[%s] %s %s %s %d %s",
			reqID, r.RemoteAddr, r.Method, r.URL.Path,
			wrapped.status, time.Since(start))
	})
}

type statusRecorder struct {
	http.ResponseWriter
	status int
}

func (sr *statusRecorder) WriteHeader(code int) {
	sr.status = code
	sr.ResponseWriter.WriteHeader(code)
}

// Request ID middleware
func requestIDMiddleware(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		id := generateID()
		ctx := context.WithValue(r.Context(), requestIDKey, id)
		w.Header().Set("X-Request-ID", id)
		next.ServeHTTP(w, r.WithContext(ctx))
	})
}

func generateID() string {
	bytes := make([]byte, 8)
	rand.Read(bytes)
	return hex.EncodeToString(bytes)
}

// CORS middleware
func corsMiddleware(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		w.Header().Set("Access-Control-Allow-Origin", "*")
		w.Header().Set("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE, OPTIONS")
		w.Header().Set("Access-Control-Allow-Headers", "Content-Type, Authorization")

		if r.Method == http.MethodOptions {
			w.WriteHeader(http.StatusOK)
			return
		}
		next.ServeHTTP(w, r)
	})
}

// Basic auth middleware
func basicAuthMiddleware(username, password string) func(http.Handler) http.Handler {
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			// Skip auth for GET requests (public read access)
			if r.Method == http.MethodGet {
				next.ServeHTTP(w, r)
				return
			}

			u, p, ok := r.BasicAuth()
			if !ok || u != username || p != password {
				w.Header().Set("WWW-Authenticate", `Basic realm="bookstore"`)
				writeError(w, http.StatusUnauthorized, "Unauthorized")
				return
			}
			next.ServeHTTP(w, r)
		})
	}
}
```

> [!warning] Middleware Order Matters
> Middleware executes in the order you wrap it. The outermost middleware runs first on the request and last on the response. A typical order: Request ID -> CORS -> Logging -> Auth -> Handler.

---

### Step 6: Router Setup

```go
func setupRouter(handler *APIHandler) http.Handler {
	mux := http.NewServeMux()

	mux.HandleFunc("/books", handler.handleBooks)
	mux.HandleFunc("/books/", handler.handleBook)
	mux.HandleFunc("/health", func(w http.ResponseWriter, r *http.Request) {
		writeJSON(w, http.StatusOK, map[string]string{"status": "ok"})
	})

	// Apply middleware chain (innermost to outermost)
	var h http.Handler = mux
	h = basicAuthMiddleware("admin", "secret")(h)
	h = loggingMiddleware(h)
	h = corsMiddleware(h)
	h = requestIDMiddleware(h)

	return h
}
```

---

### Step 7: Graceful Shutdown

```go
import (
	"os/signal"
	"syscall"
)

func main() {
	store := NewBookStore()

	// Seed some data
	store.Create(Book{Title: "The Go Programming Language", Author: "Donovan & Kernighan", ISBN: "978-0134190440", Price: 34.99, Genre: "programming"})
	store.Create(Book{Title: "Concurrency in Go", Author: "Katherine Cox-Buday", ISBN: "978-1491941195", Price: 29.99, Genre: "programming"})

	handler := &APIHandler{store: store}
	router := setupRouter(handler)

	server := &http.Server{
		Addr:         ":8080",
		Handler:      router,
		ReadTimeout:  10 * time.Second,
		WriteTimeout: 10 * time.Second,
		IdleTimeout:  60 * time.Second,
	}

	// Start server in a goroutine
	go func() {
		log.Printf("Server starting on %s", server.Addr)
		if err := server.ListenAndServe(); err != nil && err != http.ErrServerClosed {
			log.Fatalf("Server failed: %v", err)
		}
	}()

	// Wait for interrupt signal
	quit := make(chan os.Signal, 1)
	signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
	<-quit

	log.Println("Shutting down server...")

	ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
	defer cancel()

	if err := server.Shutdown(ctx); err != nil {
		log.Fatalf("Server forced to shutdown: %v", err)
	}

	log.Println("Server stopped gracefully")
}
```

> [!tip] Graceful Shutdown Pattern
> `server.Shutdown(ctx)` stops accepting new connections and waits for in-flight requests to complete (up to the context deadline). This prevents dropped requests during deployments.

---

## Full Solution

> [!success]- Full Solution
> ```go
> package main
> 
> import (
> 	"context"
> 	"crypto/rand"
> 	"encoding/hex"
> 	"encoding/json"
> 	"fmt"
> 	"log"
> 	"net/http"
> 	"os"
> 	"os/signal"
> 	"strings"
> 	"sync"
> 	"syscall"
> 	"time"
> )
> 
> // --- Types ---
> 
> type Book struct {
> 	ID          string    `json:"id"`
> 	Title       string    `json:"title"`
> 	Author      string    `json:"author"`
> 	ISBN        string    `json:"isbn"`
> 	PublishedAt string    `json:"published_at,omitempty"`
> 	Price       float64   `json:"price"`
> 	Genre       string    `json:"genre"`
> 	CreatedAt   time.Time `json:"created_at"`
> 	UpdatedAt   time.Time `json:"updated_at"`
> }
> 
> type BookStore struct {
> 	mu     sync.RWMutex
> 	books  map[string]Book
> 	nextID int
> }
> 
> type ErrorResponse struct {
> 	Error   string `json:"error"`
> 	Message string `json:"message"`
> }
> 
> type APIHandler struct {
> 	store *BookStore
> }
> 
> type contextKey string
> 
> const requestIDKey contextKey = "requestID"
> 
> // --- BookStore ---
> 
> func NewBookStore() *BookStore {
> 	return &BookStore{
> 		books:  make(map[string]Book),
> 		nextID: 1,
> 	}
> }
> 
> func (s *BookStore) Create(b Book) Book {
> 	s.mu.Lock()
> 	defer s.mu.Unlock()
> 
> 	b.ID = fmt.Sprintf("%d", s.nextID)
> 	s.nextID++
> 	b.CreatedAt = time.Now()
> 	b.UpdatedAt = b.CreatedAt
> 	s.books[b.ID] = b
> 	return b
> }
> 
> func (s *BookStore) Get(id string) (Book, bool) {
> 	s.mu.RLock()
> 	defer s.mu.RUnlock()
> 	book, ok := s.books[id]
> 	return book, ok
> }
> 
> func (s *BookStore) GetAll() []Book {
> 	s.mu.RLock()
> 	defer s.mu.RUnlock()
> 	books := make([]Book, 0, len(s.books))
> 	for _, b := range s.books {
> 		books = append(books, b)
> 	}
> 	return books
> }
> 
> func (s *BookStore) Update(id string, updated Book) (Book, error) {
> 	s.mu.Lock()
> 	defer s.mu.Unlock()
> 
> 	existing, ok := s.books[id]
> 	if !ok {
> 		return Book{}, fmt.Errorf("book %s not found", id)
> 	}
> 	updated.ID = existing.ID
> 	updated.CreatedAt = existing.CreatedAt
> 	updated.UpdatedAt = time.Now()
> 	s.books[id] = updated
> 	return updated, nil
> }
> 
> func (s *BookStore) Delete(id string) error {
> 	s.mu.Lock()
> 	defer s.mu.Unlock()
> 	if _, ok := s.books[id]; !ok {
> 		return fmt.Errorf("book %s not found", id)
> 	}
> 	delete(s.books, id)
> 	return nil
> }
> 
> func (s *BookStore) Search(query string) []Book {
> 	s.mu.RLock()
> 	defer s.mu.RUnlock()
> 
> 	query = strings.ToLower(query)
> 	var results []Book
> 	for _, b := range s.books {
> 		if strings.Contains(strings.ToLower(b.Title), query) ||
> 			strings.Contains(strings.ToLower(b.Author), query) ||
> 			strings.Contains(strings.ToLower(b.ISBN), query) {
> 			results = append(results, b)
> 		}
> 	}
> 	return results
> }
> 
> // --- JSON Helpers ---
> 
> func writeJSON(w http.ResponseWriter, status int, data any) {
> 	w.Header().Set("Content-Type", "application/json")
> 	w.WriteHeader(status)
> 	json.NewEncoder(w).Encode(data)
> }
> 
> func writeError(w http.ResponseWriter, status int, msg string) {
> 	writeJSON(w, status, ErrorResponse{
> 		Error:   http.StatusText(status),
> 		Message: msg,
> 	})
> }
> 
> func extractID(path, prefix string) string {
> 	return strings.TrimPrefix(path, prefix)
> }
> 
> // --- Handlers ---
> 
> func (h *APIHandler) handleBooks(w http.ResponseWriter, r *http.Request) {
> 	switch r.Method {
> 	case http.MethodGet:
> 		query := r.URL.Query().Get("q")
> 		genre := r.URL.Query().Get("genre")
> 
> 		var books []Book
> 		if query != "" {
> 			books = h.store.Search(query)
> 		} else {
> 			books = h.store.GetAll()
> 		}
> 
> 		if genre != "" {
> 			filtered := make([]Book, 0)
> 			for _, b := range books {
> 				if strings.EqualFold(b.Genre, genre) {
> 					filtered = append(filtered, b)
> 				}
> 			}
> 			books = filtered
> 		}
> 
> 		if books == nil {
> 			books = []Book{}
> 		}
> 		writeJSON(w, http.StatusOK, books)
> 
> 	case http.MethodPost:
> 		var book Book
> 		if err := json.NewDecoder(r.Body).Decode(&book); err != nil {
> 			writeError(w, http.StatusBadRequest, "Invalid JSON body")
> 			return
> 		}
> 		if book.Title == "" || book.Author == "" {
> 			writeError(w, http.StatusBadRequest, "Title and author are required")
> 			return
> 		}
> 		created := h.store.Create(book)
> 		writeJSON(w, http.StatusCreated, created)
> 
> 	default:
> 		writeError(w, http.StatusMethodNotAllowed, "Method not allowed")
> 	}
> }
> 
> func (h *APIHandler) handleBook(w http.ResponseWriter, r *http.Request) {
> 	id := extractID(r.URL.Path, "/books/")
> 	if id == "" {
> 		writeError(w, http.StatusBadRequest, "Book ID is required")
> 		return
> 	}
> 
> 	switch r.Method {
> 	case http.MethodGet:
> 		book, ok := h.store.Get(id)
> 		if !ok {
> 			writeError(w, http.StatusNotFound, "Book not found")
> 			return
> 		}
> 		writeJSON(w, http.StatusOK, book)
> 
> 	case http.MethodPut:
> 		var book Book
> 		if err := json.NewDecoder(r.Body).Decode(&book); err != nil {
> 			writeError(w, http.StatusBadRequest, "Invalid JSON body")
> 			return
> 		}
> 		updated, err := h.store.Update(id, book)
> 		if err != nil {
> 			writeError(w, http.StatusNotFound, err.Error())
> 			return
> 		}
> 		writeJSON(w, http.StatusOK, updated)
> 
> 	case http.MethodDelete:
> 		if err := h.store.Delete(id); err != nil {
> 			writeError(w, http.StatusNotFound, err.Error())
> 			return
> 		}
> 		w.WriteHeader(http.StatusNoContent)
> 
> 	default:
> 		writeError(w, http.StatusMethodNotAllowed, "Method not allowed")
> 	}
> }
> 
> // --- Middleware ---
> 
> type statusRecorder struct {
> 	http.ResponseWriter
> 	status int
> }
> 
> func (sr *statusRecorder) WriteHeader(code int) {
> 	sr.status = code
> 	sr.ResponseWriter.WriteHeader(code)
> }
> 
> func generateID() string {
> 	bytes := make([]byte, 8)
> 	rand.Read(bytes)
> 	return hex.EncodeToString(bytes)
> }
> 
> func requestIDMiddleware(next http.Handler) http.Handler {
> 	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
> 		id := generateID()
> 		ctx := context.WithValue(r.Context(), requestIDKey, id)
> 		w.Header().Set("X-Request-ID", id)
> 		next.ServeHTTP(w, r.WithContext(ctx))
> 	})
> }
> 
> func loggingMiddleware(next http.Handler) http.Handler {
> 	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
> 		start := time.Now()
> 		reqID, _ := r.Context().Value(requestIDKey).(string)
> 
> 		wrapped := &statusRecorder{ResponseWriter: w, status: http.StatusOK}
> 		next.ServeHTTP(wrapped, r)
> 
> 		log.Printf("[%s] %s %s %s %d %s",
> 			reqID, r.RemoteAddr, r.Method, r.URL.Path,
> 			wrapped.status, time.Since(start))
> 	})
> }
> 
> func corsMiddleware(next http.Handler) http.Handler {
> 	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
> 		w.Header().Set("Access-Control-Allow-Origin", "*")
> 		w.Header().Set("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE, OPTIONS")
> 		w.Header().Set("Access-Control-Allow-Headers", "Content-Type, Authorization")
> 
> 		if r.Method == http.MethodOptions {
> 			w.WriteHeader(http.StatusOK)
> 			return
> 		}
> 		next.ServeHTTP(w, r)
> 	})
> }
> 
> func basicAuthMiddleware(username, password string) func(http.Handler) http.Handler {
> 	return func(next http.Handler) http.Handler {
> 		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
> 			if r.Method == http.MethodGet || r.Method == http.MethodOptions {
> 				next.ServeHTTP(w, r)
> 				return
> 			}
> 			u, p, ok := r.BasicAuth()
> 			if !ok || u != username || p != password {
> 				w.Header().Set("WWW-Authenticate", `Basic realm="bookstore"`)
> 				writeError(w, http.StatusUnauthorized, "Unauthorized")
> 				return
> 			}
> 			next.ServeHTTP(w, r)
> 		})
> 	}
> }
> 
> // --- Router ---
> 
> func setupRouter(handler *APIHandler) http.Handler {
> 	mux := http.NewServeMux()
> 
> 	mux.HandleFunc("/books", handler.handleBooks)
> 	mux.HandleFunc("/books/", handler.handleBook)
> 	mux.HandleFunc("/health", func(w http.ResponseWriter, r *http.Request) {
> 		writeJSON(w, http.StatusOK, map[string]string{"status": "ok"})
> 	})
> 
> 	var h http.Handler = mux
> 	h = basicAuthMiddleware("admin", "secret")(h)
> 	h = loggingMiddleware(h)
> 	h = corsMiddleware(h)
> 	h = requestIDMiddleware(h)
> 
> 	return h
> }
> 
> // --- Main ---
> 
> func main() {
> 	store := NewBookStore()
> 
> 	// Seed data
> 	store.Create(Book{
> 		Title: "The Go Programming Language", Author: "Donovan & Kernighan",
> 		ISBN: "978-0134190440", Price: 34.99, Genre: "programming",
> 	})
> 	store.Create(Book{
> 		Title: "Concurrency in Go", Author: "Katherine Cox-Buday",
> 		ISBN: "978-1491941195", Price: 29.99, Genre: "programming",
> 	})
> 	store.Create(Book{
> 		Title: "Dune", Author: "Frank Herbert",
> 		ISBN: "978-0441172719", Price: 9.99, Genre: "sci-fi",
> 	})
> 
> 	handler := &APIHandler{store: store}
> 	router := setupRouter(handler)
> 
> 	server := &http.Server{
> 		Addr:         ":8080",
> 		Handler:      router,
> 		ReadTimeout:  10 * time.Second,
> 		WriteTimeout: 10 * time.Second,
> 		IdleTimeout:  60 * time.Second,
> 	}
> 
> 	go func() {
> 		log.Printf("Bookstore API running on http://localhost%s", server.Addr)
> 		log.Println("Endpoints:")
> 		log.Println("  GET    /books          - List all books")
> 		log.Println("  GET    /books?q=go     - Search books")
> 		log.Println("  GET    /books/{id}     - Get a book")
> 		log.Println("  POST   /books          - Create a book (auth required)")
> 		log.Println("  PUT    /books/{id}     - Update a book (auth required)")
> 		log.Println("  DELETE /books/{id}     - Delete a book (auth required)")
> 		log.Println("  GET    /health         - Health check")
> 		if err := server.ListenAndServe(); err != nil && err != http.ErrServerClosed {
> 			log.Fatalf("Server failed: %v", err)
> 		}
> 	}()
> 
> 	quit := make(chan os.Signal, 1)
> 	signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
> 	<-quit
> 
> 	log.Println("Shutting down server...")
> 	ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
> 	defer cancel()
> 
> 	if err := server.Shutdown(ctx); err != nil {
> 		log.Fatalf("Server forced to shutdown: %v", err)
> 	}
> 	log.Println("Server stopped gracefully")
> }
> ```
>
> **Test with curl:**
> ```bash
> # List all books
> curl http://localhost:8080/books | jq
>
> # Search
> curl "http://localhost:8080/books?q=go" | jq
>
> # Filter by genre
> curl "http://localhost:8080/books?genre=sci-fi" | jq
>
> # Get single book
> curl http://localhost:8080/books/1 | jq
>
> # Create (requires auth)
> curl -X POST http://localhost:8080/books \
>   -u admin:secret \
>   -H "Content-Type: application/json" \
>   -d '{"title":"Clean Code","author":"Robert Martin","price":29.99,"genre":"programming"}' | jq
>
> # Update
> curl -X PUT http://localhost:8080/books/1 \
>   -u admin:secret \
>   -H "Content-Type: application/json" \
>   -d '{"title":"The Go Programming Language","author":"Donovan & Kernighan","price":39.99,"genre":"programming"}' | jq
>
> # Delete
> curl -X DELETE http://localhost:8080/books/3 -u admin:secret -v
> ```

---

## Extensions / Challenges

- **Pagination:** Add `?page=1&limit=10` query parameters and return pagination metadata.
- **Sorting:** Add `?sort=price&order=asc` query parameters.
- **Database:** Replace the in-memory store with SQLite using `database/sql`.
- **Rate limiting:** Add a middleware that limits requests per IP using `sync.Map` and a token bucket.
- **Testing:** Write table-driven tests for each handler using `httptest.NewRecorder`.
- **OpenAPI:** Define the API spec and validate requests against it.

> [!abstract] Key Takeaways
> - The standard library `net/http` is surprisingly capable for building APIs.
> - Middleware is just function composition: `func(http.Handler) http.Handler`.
> - Always protect shared state with `sync.RWMutex` in concurrent servers.
> - Graceful shutdown prevents dropped connections during deploys.
> - Set server timeouts to prevent slow clients from exhausting resources.
