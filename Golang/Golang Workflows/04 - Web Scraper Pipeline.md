---
title: "Go Workflow: Web Scraper Pipeline"
date: 2026-04-07
tags:
  - golang
  - workflow
  - project
  - concurrency
  - pipeline
  - http-client
  - channels
  - rate-limiting
parent: "[[Golang Study]]"
---

# Go Workflow: Web Scraper Pipeline

## Overview

Build a concurrent web scraper that uses the pipeline pattern. Each stage of the pipeline (URL generation, fetching, parsing, output) runs concurrently and communicates through channels. This is one of the most powerful concurrency patterns in Go.

**Concepts practiced:** goroutines, channels, pipeline pattern, fan-out/fan-in, `net/http` client, `context`, `sync`, error handling, rate limiting, retry logic.

## Features

- Pipeline architecture with distinct stages
- Concurrent HTTP fetching with configurable parallelism
- HTML parsing to extract page titles and links
- Rate limiting to avoid overwhelming target servers
- Context cancellation and timeout
- Retry logic with exponential backoff
- Results written to stdout or a file

---

## Step-by-Step Implementation

### Step 1: Simple HTTP GET and Parse Response

Start with the basics: make an HTTP request and read the body.

```go
package main

import (
	"fmt"
	"io"
	"net/http"
	"time"
)

var httpClient = &http.Client{
	Timeout: 10 * time.Second,
}

func fetch(url string) (string, error) {
	resp, err := httpClient.Get(url)
	if err != nil {
		return "", fmt.Errorf("GET %s: %w", url, err)
	}
	defer resp.Body.Close()

	if resp.StatusCode != http.StatusOK {
		return "", fmt.Errorf("GET %s: status %d", url, resp.StatusCode)
	}

	body, err := io.ReadAll(resp.Body)
	if err != nil {
		return "", fmt.Errorf("reading body from %s: %w", url, err)
	}

	return string(body), nil
}
```

> [!warning] Always Set a Client Timeout
> The default `http.Client` has **no timeout**. A hung server will block your goroutine forever. Always create a client with an explicit `Timeout`.

---

### Step 2: Define Pipeline Types

Define the data structures that flow between pipeline stages.

```go
type FetchResult struct {
	URL        string
	StatusCode int
	Body       string
	Error      error
}

type PageData struct {
	URL    string
	Title  string
	Links  []string
	Error  error
}
```

---

### Step 3: Build the URL Generator Stage

The first stage produces URLs to scrape. In a real scraper this might read from a file, a database, or generate URLs from a pattern.

```go
import "context"

func generateURLs(ctx context.Context, urls []string) <-chan string {
	out := make(chan string)
	go func() {
		defer close(out)
		for _, u := range urls {
			select {
			case <-ctx.Done():
				return
			case out <- u:
			}
		}
	}()
	return out
}
```

> [!info] Pipeline Convention
> Each stage follows the same pattern:
> 1. Takes a context and an input channel (or initial data)
> 2. Returns an output channel
> 3. Launches a goroutine that reads from input, processes, and writes to output
> 4. Closes the output channel when done
>
> This convention makes stages composable.

---

### Step 4: Build the Fetcher Stage (Concurrent, Fan-Out/Fan-In)

This stage fans out to multiple goroutines for parallel fetching, then fans results back into a single channel.

```go
import "sync"

func fetchURLs(ctx context.Context, urls <-chan string, numWorkers int) <-chan FetchResult {
	out := make(chan FetchResult)

	var wg sync.WaitGroup
	for i := 0; i < numWorkers; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			for url := range urls {
				select {
				case <-ctx.Done():
					return
				default:
				}

				result := FetchResult{URL: url}
				resp, err := httpClient.Get(url)
				if err != nil {
					result.Error = err
					out <- result
					continue
				}

				result.StatusCode = resp.StatusCode
				body, err := io.ReadAll(resp.Body)
				resp.Body.Close()
				if err != nil {
					result.Error = fmt.Errorf("reading body: %w", err)
					out <- result
					continue
				}

				result.Body = string(body)
				out <- result
			}
		}()
	}

	go func() {
		wg.Wait()
		close(out)
	}()

	return out
}
```

> [!tip] Fan-Out/Fan-In
> **Fan-out:** Multiple goroutines read from the same input channel. Go channels are safe for concurrent reads -- each value is received by exactly one goroutine.
> **Fan-in:** Multiple goroutines write to the same output channel. Use a `WaitGroup` to close the channel only after all writers are done.

---

### Step 5: Build the Parser/Extractor Stage

Extract useful data from the HTML body. We keep it simple with string-based parsing (no external dependencies).

```go
import (
	"regexp"
	"strings"
)

var (
	titleRegex = regexp.MustCompile(`(?i)<title[^>]*>(.*?)</title>`)
	linkRegex  = regexp.MustCompile(`(?i)href=["']([^"']+)["']`)
)

func parsePages(ctx context.Context, results <-chan FetchResult) <-chan PageData {
	out := make(chan PageData)
	go func() {
		defer close(out)
		for result := range results {
			select {
			case <-ctx.Done():
				return
			default:
			}

			page := PageData{URL: result.URL}

			if result.Error != nil {
				page.Error = result.Error
				out <- page
				continue
			}

			// Extract title
			if match := titleRegex.FindStringSubmatch(result.Body); len(match) > 1 {
				page.Title = strings.TrimSpace(match[1])
			}

			// Extract links
			matches := linkRegex.FindAllStringSubmatch(result.Body, -1)
			for _, m := range matches {
				if len(m) > 1 {
					link := m[1]
					// Only keep http(s) links
					if strings.HasPrefix(link, "http") {
						page.Links = append(page.Links, link)
					}
				}
			}

			out <- page
		}
	}()
	return out
}
```

> [!warning] HTML Parsing in Production
> Regex-based HTML parsing is fragile. For production scrapers, use a proper parser like `golang.org/x/net/html` or `github.com/PuerkitoBio/goquery`. We use regex here to avoid external dependencies.

---

### Step 6: Build the Output/Writer Stage

The final stage consumes processed pages and writes results.

```go
func writeResults(ctx context.Context, pages <-chan PageData) (successes, failures int) {
	for page := range pages {
		select {
		case <-ctx.Done():
			return
		default:
		}

		if page.Error != nil {
			fmt.Fprintf(os.Stderr, "[FAIL] %s: %v\n", page.URL, page.Error)
			failures++
			continue
		}

		successes++
		fmt.Printf("[OK] %s\n", page.URL)
		fmt.Printf("     Title: %s\n", page.Title)
		fmt.Printf("     Links: %d found\n", len(page.Links))
		fmt.Println("---")
	}
	return
}
```

---

### Step 7: Connect the Pipeline and Add Rate Limiting

Rate limiting prevents overwhelming the target server. A simple approach uses a ticker-based rate limiter.

```go
func rateLimitedFetchURLs(ctx context.Context, urls <-chan string, numWorkers int, rps int) <-chan FetchResult {
	out := make(chan FetchResult)
	limiter := time.NewTicker(time.Second / time.Duration(rps))

	var wg sync.WaitGroup
	for i := 0; i < numWorkers; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			for url := range urls {
				select {
				case <-ctx.Done():
					return
				case <-limiter.C: // wait for rate limit token
				}

				result := FetchResult{URL: url}
				resp, err := httpClient.Get(url)
				if err != nil {
					result.Error = err
					out <- result
					continue
				}

				result.StatusCode = resp.StatusCode
				body, err := io.ReadAll(resp.Body)
				resp.Body.Close()
				if err != nil {
					result.Error = fmt.Errorf("reading body: %w", err)
					out <- result
					continue
				}

				result.Body = string(body)
				out <- result
			}
		}()
	}

	go func() {
		wg.Wait()
		limiter.Stop()
		close(out)
	}()

	return out
}
```

Wire up the full pipeline:

```go
func runPipeline(ctx context.Context, urls []string, workers int, rps int) {
	// Stage 1: Generate URLs
	urlCh := generateURLs(ctx, urls)

	// Stage 2: Fetch pages (rate-limited, concurrent)
	fetchCh := rateLimitedFetchURLs(ctx, urlCh, workers, rps)

	// Stage 3: Parse HTML
	pageCh := parsePages(ctx, fetchCh)

	// Stage 4: Write output and collect results
	successes, failures := writeResults(ctx, pageCh)

	fmt.Printf("\nDone: %d succeeded, %d failed out of %d URLs\n",
		successes, failures, len(urls))
}
```

---

### Step 8: Add Retry Logic

Wrap the HTTP fetch in a retry function with exponential backoff.

```go
import "math"

func fetchWithRetry(ctx context.Context, url string, maxRetries int) FetchResult {
	result := FetchResult{URL: url}

	for attempt := 0; attempt <= maxRetries; attempt++ {
		if attempt > 0 {
			// Exponential backoff: 1s, 2s, 4s, 8s...
			backoff := time.Duration(math.Pow(2, float64(attempt-1))) * time.Second
			select {
			case <-ctx.Done():
				result.Error = ctx.Err()
				return result
			case <-time.After(backoff):
			}
		}

		req, err := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
		if err != nil {
			result.Error = err
			return result // don't retry malformed URLs
		}
		req.Header.Set("User-Agent", "GoScraper/1.0 (learning project)")

		resp, err := httpClient.Do(req)
		if err != nil {
			result.Error = err
			if ctx.Err() != nil {
				return result
			}
			continue
		}

		result.StatusCode = resp.StatusCode
		body, err := io.ReadAll(resp.Body)
		resp.Body.Close()

		if err != nil {
			result.Error = fmt.Errorf("reading body: %w", err)
			continue
		}

		// Don't retry client errors (4xx)
		if resp.StatusCode >= 400 && resp.StatusCode < 500 {
			result.Error = fmt.Errorf("client error: %d", resp.StatusCode)
			return result
		}

		// Retry server errors (5xx)
		if resp.StatusCode >= 500 {
			result.Error = fmt.Errorf("server error: %d", resp.StatusCode)
			continue
		}

		result.Body = string(body)
		result.Error = nil
		return result
	}

	result.Error = fmt.Errorf("failed after %d retries: %w", maxRetries, result.Error)
	return result
}
```

> [!tip] Retry Only Transient Errors
> Retry on 5xx (server errors) and network timeouts. Do **not** retry on 4xx (client errors like 404) -- those will not succeed on retry. Add jitter to backoff in production to avoid thundering herds.

---

## Full Solution

> [!success]- Full Solution
> ```go
> package main
> 
> import (
> 	"context"
> 	"flag"
> 	"fmt"
> 	"io"
> 	"math"
> 	"net/http"
> 	"os"
> 	"os/signal"
> 	"regexp"
> 	"strings"
> 	"sync"
> 	"syscall"
> 	"time"
> )
> 
> // --- Types ---
> 
> type FetchResult struct {
> 	URL        string
> 	StatusCode int
> 	Body       string
> 	Error      error
> }
> 
> type PageData struct {
> 	URL   string   `json:"url"`
> 	Title string   `json:"title"`
> 	Links []string `json:"links"`
> 	Error error    `json:"-"`
> }
> 
> // --- HTTP Client ---
> 
> var httpClient = &http.Client{
> 	Timeout: 15 * time.Second,
> 	Transport: &http.Transport{
> 		MaxIdleConns:        100,
> 		MaxIdleConnsPerHost: 10,
> 		IdleConnTimeout:     30 * time.Second,
> 	},
> }
> 
> // --- Regex Patterns ---
> 
> var (
> 	titleRegex = regexp.MustCompile(`(?i)<title[^>]*>(.*?)</title>`)
> 	linkRegex  = regexp.MustCompile(`(?i)href=["']([^"']+)["']`)
> )
> 
> // --- Pipeline Stages ---
> 
> // Stage 1: URL Generator
> func generateURLs(ctx context.Context, urls []string) <-chan string {
> 	out := make(chan string)
> 	go func() {
> 		defer close(out)
> 		for _, u := range urls {
> 			select {
> 			case <-ctx.Done():
> 				return
> 			case out <- u:
> 			}
> 		}
> 	}()
> 	return out
> }
> 
> // Stage 2: Fetcher with rate limiting and retry
> func fetchStage(ctx context.Context, urls <-chan string, numWorkers, rps, maxRetries int) <-chan FetchResult {
> 	out := make(chan FetchResult)
> 	limiter := time.NewTicker(time.Second / time.Duration(rps))
> 
> 	var wg sync.WaitGroup
> 	for i := 0; i < numWorkers; i++ {
> 		wg.Add(1)
> 		go func(workerID int) {
> 			defer wg.Done()
> 			for url := range urls {
> 				// Rate limit
> 				select {
> 				case <-ctx.Done():
> 					return
> 				case <-limiter.C:
> 				}
> 
> 				result := fetchWithRetry(ctx, url, maxRetries)
> 				select {
> 				case <-ctx.Done():
> 					return
> 				case out <- result:
> 				}
> 			}
> 		}(i)
> 	}
> 
> 	go func() {
> 		wg.Wait()
> 		limiter.Stop()
> 		close(out)
> 	}()
> 
> 	return out
> }
> 
> func fetchWithRetry(ctx context.Context, url string, maxRetries int) FetchResult {
> 	result := FetchResult{URL: url}
> 
> 	for attempt := 0; attempt <= maxRetries; attempt++ {
> 		if attempt > 0 {
> 			backoff := time.Duration(math.Pow(2, float64(attempt-1))) * time.Second
> 			select {
> 			case <-ctx.Done():
> 				result.Error = ctx.Err()
> 				return result
> 			case <-time.After(backoff):
> 			}
> 		}
> 
> 		req, err := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
> 		if err != nil {
> 			result.Error = err
> 			return result // don't retry malformed URLs
> 		}
> 		req.Header.Set("User-Agent", "GoScraper/1.0 (learning project)")
> 
> 		resp, err := httpClient.Do(req)
> 		if err != nil {
> 			result.Error = err
> 			if ctx.Err() != nil {
> 				return result
> 			}
> 			continue
> 		}
> 
> 		result.StatusCode = resp.StatusCode
> 		body, err := io.ReadAll(resp.Body)
> 		resp.Body.Close()
> 
> 		if err != nil {
> 			result.Error = fmt.Errorf("reading body: %w", err)
> 			continue
> 		}
> 
> 		// Don't retry client errors (4xx)
> 		if resp.StatusCode >= 400 && resp.StatusCode < 500 {
> 			result.Error = fmt.Errorf("client error: %d", resp.StatusCode)
> 			return result
> 		}
> 
> 		// Retry server errors (5xx)
> 		if resp.StatusCode >= 500 {
> 			result.Error = fmt.Errorf("server error: %d", resp.StatusCode)
> 			continue
> 		}
> 
> 		result.Body = string(body)
> 		result.Error = nil
> 		return result
> 	}
> 
> 	if result.Error != nil {
> 		result.Error = fmt.Errorf("failed after %d retries: %w", maxRetries, result.Error)
> 	}
> 	return result
> }
> 
> // Stage 3: Parser
> func parseStage(ctx context.Context, results <-chan FetchResult) <-chan PageData {
> 	out := make(chan PageData)
> 	go func() {
> 		defer close(out)
> 		for result := range results {
> 			select {
> 			case <-ctx.Done():
> 				return
> 			default:
> 			}
> 
> 			page := PageData{URL: result.URL}
> 
> 			if result.Error != nil {
> 				page.Error = result.Error
> 				out <- page
> 				continue
> 			}
> 
> 			// Extract title
> 			if match := titleRegex.FindStringSubmatch(result.Body); len(match) > 1 {
> 				page.Title = strings.TrimSpace(match[1])
> 			} else {
> 				page.Title = "(no title)"
> 			}
> 
> 			// Extract links (deduplicated)
> 			seen := make(map[string]bool)
> 			matches := linkRegex.FindAllStringSubmatch(result.Body, -1)
> 			for _, m := range matches {
> 				if len(m) > 1 {
> 					link := m[1]
> 					if strings.HasPrefix(link, "http") && !seen[link] {
> 						page.Links = append(page.Links, link)
> 						seen[link] = true
> 					}
> 				}
> 			}
> 
> 			out <- page
> 		}
> 	}()
> 	return out
> }
> 
> // Stage 4: Output
> func outputStage(ctx context.Context, pages <-chan PageData) (successes int, failures int) {
> 	for page := range pages {
> 		select {
> 		case <-ctx.Done():
> 			return
> 		default:
> 		}
> 
> 		if page.Error != nil {
> 			fmt.Fprintf(os.Stderr, "[FAIL] %s: %v\n", page.URL, page.Error)
> 			failures++
> 			continue
> 		}
> 
> 		successes++
> 		fmt.Printf("[OK] %s\n", page.URL)
> 		fmt.Printf("     Title: %s\n", page.Title)
> 		fmt.Printf("     Links: %d found\n", len(page.Links))
> 
> 		// Print first 5 links
> 		limit := 5
> 		if len(page.Links) < limit {
> 			limit = len(page.Links)
> 		}
> 		for _, link := range page.Links[:limit] {
> 			fmt.Printf("       -> %s\n", link)
> 		}
> 		if len(page.Links) > 5 {
> 			fmt.Printf("       ... and %d more\n", len(page.Links)-5)
> 		}
> 		fmt.Println()
> 	}
> 	return
> }
> 
> // --- Pipeline Runner ---
> 
> func runPipeline(ctx context.Context, urls []string, workers, rps, retries int) {
> 	start := time.Now()
> 
> 	// Connect pipeline stages
> 	urlCh := generateURLs(ctx, urls)
> 	fetchCh := fetchStage(ctx, urlCh, workers, rps, retries)
> 	pageCh := parseStage(ctx, fetchCh)
> 	successes, failures := outputStage(ctx, pageCh)
> 
> 	duration := time.Since(start)
> 
> 	fmt.Println("========================================")
> 	fmt.Printf("Pipeline complete in %s\n", duration.Round(time.Millisecond))
> 	fmt.Printf("  Total URLs:  %d\n", len(urls))
> 	fmt.Printf("  Succeeded:   %d\n", successes)
> 	fmt.Printf("  Failed:      %d\n", failures)
> 	fmt.Printf("  Workers:     %d\n", workers)
> 	fmt.Printf("  Rate limit:  %d req/s\n", rps)
> }
> 
> // --- Main ---
> 
> func main() {
> 	workers := flag.Int("workers", 3, "Number of concurrent fetchers")
> 	rps := flag.Int("rps", 5, "Max requests per second")
> 	timeout := flag.Duration("timeout", 2*time.Minute, "Overall timeout")
> 	retries := flag.Int("retries", 2, "Max retries per URL")
> 	flag.Parse()
> 
> 	// URLs can come from arguments or defaults
> 	urls := flag.Args()
> 	if len(urls) == 0 {
> 		urls = []string{
> 			"https://go.dev",
> 			"https://pkg.go.dev",
> 			"https://blog.golang.org",
> 			"https://github.com/golang/go",
> 			"https://gobyexample.com",
> 			"https://tour.golang.org",
> 			"https://play.golang.org",
> 			"https://www.rust-lang.org",
> 			"https://httpbin.org/status/500",
> 			"https://httpbin.org/delay/30",
> 		}
> 	}
> 
> 	// Context with timeout and signal handling
> 	ctx, cancel := context.WithTimeout(context.Background(), *timeout)
> 	defer cancel()
> 
> 	sigCh := make(chan os.Signal, 1)
> 	signal.Notify(sigCh, syscall.SIGINT, syscall.SIGTERM)
> 	go func() {
> 		<-sigCh
> 		fmt.Fprintln(os.Stderr, "\nInterrupted, draining pipeline...")
> 		cancel()
> 	}()
> 
> 	fmt.Printf("Starting scraper pipeline: %d URLs, %d workers, %d req/s\n\n",
> 		len(urls), *workers, *rps)
> 
> 	runPipeline(ctx, urls, *workers, *rps, *retries)
> }
> ```
>
> **Usage:**
> ```bash
> # Run with default URLs
> go run main.go
>
> # Custom URLs with options
> go run main.go -workers 5 -rps 10 -retries 3 \
>   https://example.com https://go.dev https://github.com
>
> # With tight timeout
> go run main.go -timeout 30s -rps 2 https://go.dev
> ```

---

## Extensions / Challenges

- **Depth crawling:** Follow extracted links up to N levels deep (breadth-first). Track visited URLs in a `sync.Map` to avoid loops.
- **Proper HTML parsing:** Replace regex with `golang.org/x/net/html` for robust extraction.
- **JSON output:** Add a `-json` flag that outputs structured `PageData` as JSON lines.
- **Persistent queue:** Write unprocessed URLs to a file so you can resume after interruption.
- **robots.txt:** Parse and respect `robots.txt` before fetching.
- **Concurrent pipelines:** Run multiple independent pipelines for different domains, each with its own rate limiter.

> [!abstract] Key Takeaways
> - The pipeline pattern decomposes complex processing into composable, concurrent stages.
> - Each stage owns its output channel and closes it when done -- this propagates completion through the pipeline.
> - Fan-out/fan-in at the fetch stage gives concurrent I/O while the rest stays single-goroutine.
> - Rate limiting with `time.Ticker` is simple but effective; for production use `golang.org/x/time/rate`.
> - `http.NewRequestWithContext` ensures fetches respect cancellation immediately.
