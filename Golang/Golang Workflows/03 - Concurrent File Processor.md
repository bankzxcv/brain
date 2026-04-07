---
title: "Go Workflow: Concurrent File Processor"
date: 2026-04-07
tags:
  - golang
  - workflow
  - project
  - concurrency
  - goroutines
  - channels
  - worker-pool
  - file-io
parent: "[[Golang Study]]"
---

# Go Workflow: Concurrent File Processor

## Overview

Build a tool that walks a directory, processes every file concurrently using a worker pool, and aggregates the results. The "processing" is computing statistics (word count, line count, byte size, etc.) for each file, but the architecture generalizes to any file-processing task.

**Concepts practiced:** goroutines, channels, `sync.WaitGroup`, worker pool pattern, `filepath.Walk`, file I/O, error handling, `context` cancellation, semaphore pattern.

## Features

- Recursively walk a directory tree
- Process files in parallel with a configurable worker pool
- Collect per-file statistics: line count, word count, byte count
- Aggregate totals across all files
- Progress reporting during processing
- Context-based cancellation and timeout
- Configurable concurrency limit

---

## Step-by-Step Implementation

### Step 1: Walk a Directory and Collect File Paths

Start with the simplest version: collect all regular file paths from a directory tree.

```go
package main

import (
	"fmt"
	"os"
	"path/filepath"
)

func collectFiles(root string) ([]string, error) {
	var files []string

	err := filepath.Walk(root, func(path string, info os.FileInfo, err error) error {
		if err != nil {
			return err // propagate permission errors, etc.
		}
		if !info.IsDir() && info.Mode().IsRegular() {
			files = append(files, path)
		}
		return nil
	})

	return files, err
}
```

> [!tip] filepath.Walk vs filepath.WalkDir
> Go 1.16 introduced `filepath.WalkDir`, which is more efficient because it avoids calling `os.Stat` on every entry. Prefer `WalkDir` in new code. We use `Walk` here for clarity, but the change is trivial.

---

### Step 2: Process Files Sequentially

Define a result type and a function that processes a single file.

```go
import (
	"bufio"
	"strings"
)

type FileResult struct {
	Path      string
	Lines     int
	Words     int
	Bytes     int64
	Error     error
}

func processFile(path string) FileResult {
	result := FileResult{Path: path}

	info, err := os.Stat(path)
	if err != nil {
		result.Error = fmt.Errorf("stat %s: %w", path, err)
		return result
	}
	result.Bytes = info.Size()

	file, err := os.Open(path)
	if err != nil {
		result.Error = fmt.Errorf("open %s: %w", path, err)
		return result
	}
	defer file.Close()

	scanner := bufio.NewScanner(file)
	for scanner.Scan() {
		result.Lines++
		result.Words += len(strings.Fields(scanner.Text()))
	}
	if err := scanner.Err(); err != nil {
		result.Error = fmt.Errorf("scan %s: %w", path, err)
	}

	return result
}
```

Sequential processing for reference:

```go
func processSequential(files []string) []FileResult {
	results := make([]FileResult, 0, len(files))
	for _, f := range files {
		results = append(results, processFile(f))
	}
	return results
}
```

---

### Step 3: Convert to Concurrent Processing with a Worker Pool

The worker pool pattern uses a fixed number of goroutines (workers) that pull jobs from a shared channel.

```go
import "sync"

func processWithWorkerPool(files []string, numWorkers int) []FileResult {
	jobs := make(chan string, len(files))
	resultsCh := make(chan FileResult, len(files))

	// Start workers
	var wg sync.WaitGroup
	for i := 0; i < numWorkers; i++ {
		wg.Add(1)
		go func(workerID int) {
			defer wg.Done()
			for path := range jobs {
				result := processFile(path)
				resultsCh <- result
			}
		}(i)
	}

	// Send jobs
	for _, f := range files {
		jobs <- f
	}
	close(jobs)

	// Wait for all workers then close results channel
	go func() {
		wg.Wait()
		close(resultsCh)
	}()

	// Collect results
	var results []FileResult
	for r := range resultsCh {
		results = append(results, r)
	}

	return results
}
```

> [!info] Why Buffer the Channels?
> Buffering `jobs` with `len(files)` means the sender never blocks. Buffering `resultsCh` similarly prevents workers from blocking if the collector is slow. For very large file sets, you might use a smaller buffer and accept backpressure.

---

### Step 4: Aggregate Results

```go
type Summary struct {
	TotalFiles  int
	TotalLines  int
	TotalWords  int
	TotalBytes  int64
	Errors      int
	ErrorList   []string
}

func aggregate(results []FileResult) Summary {
	var s Summary
	s.TotalFiles = len(results)

	for _, r := range results {
		if r.Error != nil {
			s.Errors++
			s.ErrorList = append(s.ErrorList, r.Error.Error())
			continue
		}
		s.TotalLines += r.Lines
		s.TotalWords += r.Words
		s.TotalBytes += r.Bytes
	}

	return s
}

func (s Summary) String() string {
	return fmt.Sprintf(`--- Summary ---
Files processed: %d
Total lines:     %d
Total words:     %d
Total bytes:     %d (%.2f MB)
Errors:          %d`,
		s.TotalFiles, s.TotalLines, s.TotalWords,
		s.TotalBytes, float64(s.TotalBytes)/(1024*1024),
		s.Errors)
}
```

---

### Step 5: Add Progress Reporting

Report progress as files are processed, without blocking the workers.

```go
func processWithProgress(files []string, numWorkers int) []FileResult {
	jobs := make(chan string, numWorkers*2)
	resultsCh := make(chan FileResult, numWorkers*2)

	var wg sync.WaitGroup
	for i := 0; i < numWorkers; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			for path := range jobs {
				resultsCh <- processFile(path)
			}
		}()
	}

	// Send jobs in a goroutine so we don't block
	go func() {
		for _, f := range files {
			jobs <- f
		}
		close(jobs)
	}()

	// Close results when all workers are done
	go func() {
		wg.Wait()
		close(resultsCh)
	}()

	// Collect with progress
	total := len(files)
	var results []FileResult
	for r := range resultsCh {
		results = append(results, r)
		done := len(results)
		if done%100 == 0 || done == total {
			fmt.Fprintf(os.Stderr, "\rProcessed %d/%d files (%.1f%%)",
				done, total, float64(done)/float64(total)*100)
		}
	}
	fmt.Fprintln(os.Stderr) // newline after progress

	return results
}
```

> [!warning] Progress to Stderr
> Always write progress output to stderr so it doesn't pollute stdout. This lets users pipe the actual results elsewhere: `processor /tmp > results.txt`.

---

### Step 6: Add Context Cancellation and Timeout

```go
import "context"

func processWithContext(ctx context.Context, files []string, numWorkers int) ([]FileResult, error) {
	jobs := make(chan string, numWorkers*2)
	resultsCh := make(chan FileResult, numWorkers*2)

	var wg sync.WaitGroup
	for i := 0; i < numWorkers; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			for path := range jobs {
				select {
				case <-ctx.Done():
					return
				default:
					resultsCh <- processFile(path)
				}
			}
		}()
	}

	// Send jobs, respecting cancellation
	go func() {
		defer close(jobs)
		for _, f := range files {
			select {
			case <-ctx.Done():
				return
			case jobs <- f:
			}
		}
	}()

	go func() {
		wg.Wait()
		close(resultsCh)
	}()

	var results []FileResult
	for r := range resultsCh {
		results = append(results, r)
	}

	if ctx.Err() != nil {
		return results, fmt.Errorf("processing cancelled: %w", ctx.Err())
	}
	return results, nil
}
```

> [!tip] Context Check Placement
> Check `ctx.Done()` at two points: before sending a job (to stop feeding the pipeline) and before processing (to stop workers promptly). Checking only in one place leaves a lag.

---

### Step 7: Configurable Concurrency with Semaphore

An alternative to a fixed worker pool is a semaphore that limits how many goroutines run at once.

```go
func processWithSemaphore(ctx context.Context, files []string, maxConcurrency int) ([]FileResult, error) {
	sem := make(chan struct{}, maxConcurrency)
	resultsCh := make(chan FileResult, len(files))

	var wg sync.WaitGroup

	for _, f := range files {
		select {
		case <-ctx.Done():
			break
		default:
		}

		wg.Add(1)
		go func(path string) {
			defer wg.Done()

			// Acquire semaphore slot
			sem <- struct{}{}
			defer func() { <-sem }()

			select {
			case <-ctx.Done():
				return
			default:
				resultsCh <- processFile(path)
			}
		}(f)
	}

	go func() {
		wg.Wait()
		close(resultsCh)
	}()

	var results []FileResult
	for r := range resultsCh {
		results = append(results, r)
	}

	if ctx.Err() != nil {
		return results, fmt.Errorf("processing cancelled: %w", ctx.Err())
	}
	return results, nil
}
```

> [!info] Worker Pool vs Semaphore
> **Worker pool:** Fixed goroutines, jobs come through a channel. Lower goroutine overhead, predictable memory usage. Best when the number of tasks is large.
> **Semaphore:** One goroutine per task, but only N run at a time. Simpler code for small task counts. Can create many goroutines (they just block on the semaphore).

---

## Full Solution

> [!success]- Full Solution
> ```go
> package main
> 
> import (
> 	"bufio"
> 	"context"
> 	"flag"
> 	"fmt"
> 	"os"
> 	"os/signal"
> 	"path/filepath"
> 	"runtime"
> 	"strings"
> 	"sync"
> 	"syscall"
> 	"time"
> )
> 
> // --- Types ---
> 
> type FileResult struct {
> 	Path  string `json:"path"`
> 	Lines int    `json:"lines"`
> 	Words int    `json:"words"`
> 	Bytes int64  `json:"bytes"`
> 	Error error  `json:"-"`
> }
> 
> type Summary struct {
> 	TotalFiles int
> 	TotalLines int
> 	TotalWords int
> 	TotalBytes int64
> 	Errors     int
> 	ErrorList  []string
> 	Duration   time.Duration
> }
> 
> func (s Summary) String() string {
> 	return fmt.Sprintf(`
> --- Summary ---
> Files processed: %d
> Total lines:     %d
> Total words:     %d
> Total bytes:     %d (%.2f MB)
> Errors:          %d
> Duration:        %s`,
> 		s.TotalFiles, s.TotalLines, s.TotalWords,
> 		s.TotalBytes, float64(s.TotalBytes)/(1024*1024),
> 		s.Errors, s.Duration.Round(time.Millisecond))
> }
> 
> // --- File Collection ---
> 
> func collectFiles(root string) ([]string, error) {
> 	var files []string
> 	err := filepath.WalkDir(root, func(path string, d os.DirEntry, err error) error {
> 		if err != nil {
> 			// Log and skip directories we can't read
> 			fmt.Fprintf(os.Stderr, "Warning: %v\n", err)
> 			return nil
> 		}
> 		if !d.IsDir() && d.Type().IsRegular() {
> 			files = append(files, path)
> 		}
> 		return nil
> 	})
> 	return files, err
> }
> 
> // --- File Processing ---
> 
> func processFile(path string) FileResult {
> 	result := FileResult{Path: path}
> 
> 	info, err := os.Stat(path)
> 	if err != nil {
> 		result.Error = fmt.Errorf("stat %s: %w", path, err)
> 		return result
> 	}
> 	result.Bytes = info.Size()
> 
> 	file, err := os.Open(path)
> 	if err != nil {
> 		result.Error = fmt.Errorf("open %s: %w", path, err)
> 		return result
> 	}
> 	defer file.Close()
> 
> 	scanner := bufio.NewScanner(file)
> 	// Handle long lines
> 	buf := make([]byte, 0, 64*1024)
> 	scanner.Buffer(buf, 1024*1024)
> 
> 	for scanner.Scan() {
> 		result.Lines++
> 		result.Words += len(strings.Fields(scanner.Text()))
> 	}
> 	if err := scanner.Err(); err != nil {
> 		result.Error = fmt.Errorf("scan %s: %w", path, err)
> 	}
> 
> 	return result
> }
> 
> // --- Worker Pool ---
> 
> func processFiles(ctx context.Context, files []string, numWorkers int) ([]FileResult, error) {
> 	jobs := make(chan string, numWorkers*2)
> 	resultsCh := make(chan FileResult, numWorkers*2)
> 
> 	// Start workers
> 	var wg sync.WaitGroup
> 	for i := 0; i < numWorkers; i++ {
> 		wg.Add(1)
> 		go func(workerID int) {
> 			defer wg.Done()
> 			for path := range jobs {
> 				select {
> 				case <-ctx.Done():
> 					return
> 				default:
> 					resultsCh <- processFile(path)
> 				}
> 			}
> 		}(i)
> 	}
> 
> 	// Feed jobs
> 	go func() {
> 		defer close(jobs)
> 		for _, f := range files {
> 			select {
> 			case <-ctx.Done():
> 				return
> 			case jobs <- f:
> 			}
> 		}
> 	}()
> 
> 	// Close results when done
> 	go func() {
> 		wg.Wait()
> 		close(resultsCh)
> 	}()
> 
> 	// Collect with progress reporting
> 	total := len(files)
> 	results := make([]FileResult, 0, total)
> 	for r := range resultsCh {
> 		results = append(results, r)
> 		done := len(results)
> 		if done%100 == 0 || done == total {
> 			fmt.Fprintf(os.Stderr, "\rProcessed %d/%d files (%.1f%%)",
> 				done, total, float64(done)/float64(total)*100)
> 		}
> 	}
> 	fmt.Fprintln(os.Stderr)
> 
> 	if ctx.Err() != nil {
> 		return results, fmt.Errorf("processing interrupted: %w", ctx.Err())
> 	}
> 	return results, nil
> }
> 
> // --- Aggregation ---
> 
> func aggregate(results []FileResult, duration time.Duration) Summary {
> 	s := Summary{
> 		TotalFiles: len(results),
> 		Duration:   duration,
> 	}
> 	for _, r := range results {
> 		if r.Error != nil {
> 			s.Errors++
> 			s.ErrorList = append(s.ErrorList, r.Error.Error())
> 			continue
> 		}
> 		s.TotalLines += r.Lines
> 		s.TotalWords += r.Words
> 		s.TotalBytes += r.Bytes
> 	}
> 	return s
> }
> 
> // --- Main ---
> 
> func main() {
> 	dir := flag.String("dir", ".", "Directory to process")
> 	workers := flag.Int("workers", runtime.NumCPU(), "Number of worker goroutines")
> 	timeout := flag.Duration("timeout", 5*time.Minute, "Processing timeout")
> 	verbose := flag.Bool("verbose", false, "Print per-file results")
> 	flag.Parse()
> 
> 	// Set up cancellation: both timeout and interrupt signal
> 	ctx, cancel := context.WithTimeout(context.Background(), *timeout)
> 	defer cancel()
> 
> 	sigCh := make(chan os.Signal, 1)
> 	signal.Notify(sigCh, syscall.SIGINT, syscall.SIGTERM)
> 	go func() {
> 		<-sigCh
> 		fmt.Fprintln(os.Stderr, "\nInterrupted, finishing current files...")
> 		cancel()
> 	}()
> 
> 	// Collect files
> 	fmt.Fprintf(os.Stderr, "Scanning %s...\n", *dir)
> 	files, err := collectFiles(*dir)
> 	if err != nil {
> 		fmt.Fprintf(os.Stderr, "Error scanning directory: %v\n", err)
> 		os.Exit(1)
> 	}
> 	fmt.Fprintf(os.Stderr, "Found %d files. Processing with %d workers...\n", len(files), *workers)
> 
> 	if len(files) == 0 {
> 		fmt.Println("No files found.")
> 		return
> 	}
> 
> 	// Process
> 	start := time.Now()
> 	results, err := processFiles(ctx, files, *workers)
> 	duration := time.Since(start)
> 
> 	if err != nil {
> 		fmt.Fprintf(os.Stderr, "Warning: %v\n", err)
> 	}
> 
> 	// Print per-file results if verbose
> 	if *verbose {
> 		fmt.Printf("%-8s %-8s %-12s %s\n", "LINES", "WORDS", "BYTES", "FILE")
> 		fmt.Printf("%-8s %-8s %-12s %s\n", "-----", "-----", "-----", "----")
> 		for _, r := range results {
> 			if r.Error != nil {
> 				fmt.Printf("%-8s %-8s %-12s %s [ERROR: %v]\n",
> 					"-", "-", "-", r.Path, r.Error)
> 			} else {
> 				fmt.Printf("%-8d %-8d %-12d %s\n",
> 					r.Lines, r.Words, r.Bytes, r.Path)
> 			}
> 		}
> 		fmt.Println()
> 	}
> 
> 	// Summary
> 	summary := aggregate(results, duration)
> 	fmt.Println(summary)
> 
> 	if summary.Errors > 0 {
> 		fmt.Printf("\nErrors:\n")
> 		for _, e := range summary.ErrorList {
> 			fmt.Printf("  - %s\n", e)
> 		}
> 	}
> }
> ```
>
> **Usage:**
> ```bash
> # Process current directory with default workers (number of CPUs)
> go run main.go -dir .
>
> # Process with 8 workers, verbose output, 2 minute timeout
> go run main.go -dir /path/to/project -workers 8 -verbose -timeout 2m
>
> # Process and redirect summary to file
> go run main.go -dir ~/Documents > summary.txt
> ```

---

## Extensions / Challenges

- **File type filtering:** Add `-ext .go,.md` to only process certain file types.
- **Top N:** Display the top 10 files by line count or word count.
- **Parallel directory walk:** Use a concurrent walker instead of `filepath.WalkDir` for faster scanning on SSDs.
- **Checksum:** Compute SHA256 hashes concurrently for file deduplication.
- **Output formats:** Add JSON or CSV output in addition to the table format.
- **Benchmarking:** Add `-bench` mode that compares sequential vs concurrent performance and reports the speedup factor.

> [!abstract] Key Takeaways
> - The worker pool pattern (N goroutines reading from a jobs channel) is the most common concurrency pattern in Go.
> - `sync.WaitGroup` coordinates "wait for all workers to finish" cleanly.
> - Always close channels from the sender side, never the receiver.
> - Context cancellation propagates "stop" signals through the entire pipeline.
> - Buffered channels prevent goroutine leaks when a stage stops consuming.
