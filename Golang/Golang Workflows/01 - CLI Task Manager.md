---
title: "Go Workflow: CLI Task Manager"
date: 2026-04-07
tags:
  - golang
  - workflow
  - project
  - cli
  - structs
  - json
  - file-io
  - error-handling
parent: "[[Golang Study]]"
---

# Go Workflow: CLI Task Manager

## Overview

Build a command-line task manager similar to a simplified Todoist. You will manage tasks from the terminal -- adding, listing, completing, and deleting them -- with persistent storage in a JSON file.

**Concepts practiced:** structs, slices, JSON marshaling/unmarshaling, file I/O, error handling, `flag` package, `time` package.

## Features

- Add a new task with a title
- List all tasks (with filtering: all, pending, completed)
- Mark a task as completed
- Delete a task by ID
- Persistent storage using a JSON file
- Due dates and priority levels

---

## Step-by-Step Implementation

### Step 1: Define the Task Struct

Start by modeling a single task. Every task needs an identifier, a title, a completion status, and a creation timestamp.

```go
package main

import "time"

type Priority int

const (
	PriorityLow Priority = iota + 1
	PriorityMedium
	PriorityHigh
)

func (p Priority) String() string {
	switch p {
	case PriorityLow:
		return "low"
	case PriorityMedium:
		return "medium"
	case PriorityHigh:
		return "high"
	default:
		return "unknown"
	}
}

type Task struct {
	ID        int       `json:"id"`
	Title     string    `json:"title"`
	Done      bool      `json:"done"`
	Priority  Priority  `json:"priority"`
	CreatedAt time.Time `json:"created_at"`
	DueDate   string    `json:"due_date,omitempty"` // format: YYYY-MM-DD
}
```

> [!tip] JSON Tags
> The `json:"..."` struct tags control how fields are serialized. Use `omitempty` for optional fields so they don't clutter the JSON file when empty.

---

### Step 2: Implement In-Memory CRUD Operations

Create a `TaskStore` that holds all tasks in a slice and provides methods to manipulate them.

```go
type TaskStore struct {
	Tasks  []Task `json:"tasks"`
	NextID int    `json:"next_id"`
}

func NewTaskStore() *TaskStore {
	return &TaskStore{
		Tasks:  []Task{},
		NextID: 1,
	}
}

func (s *TaskStore) Add(title string, priority Priority, dueDate string) Task {
	task := Task{
		ID:        s.NextID,
		Title:     title,
		Done:      false,
		Priority:  priority,
		CreatedAt: time.Now(),
		DueDate:   dueDate,
	}
	s.Tasks = append(s.Tasks, task)
	s.NextID++
	return task
}

func (s *TaskStore) Complete(id int) error {
	for i := range s.Tasks {
		if s.Tasks[i].ID == id {
			s.Tasks[i].Done = true
			return nil
		}
	}
	return fmt.Errorf("task with ID %d not found", id)
}

func (s *TaskStore) Delete(id int) error {
	for i, task := range s.Tasks {
		if task.ID == id {
			s.Tasks = append(s.Tasks[:i], s.Tasks[i+1:]...)
			return nil
		}
	}
	return fmt.Errorf("task with ID %d not found", id)
}

func (s *TaskStore) List(filter string) []Task {
	if filter == "all" || filter == "" {
		return s.Tasks
	}

	var result []Task
	for _, task := range s.Tasks {
		switch filter {
		case "pending":
			if !task.Done {
				result = append(result, task)
			}
		case "done":
			if task.Done {
				result = append(result, task)
			}
		}
	}
	return result
}
```

> [!warning] Slice Deletion Gotcha
> When deleting from a slice with `append(s[:i], s[i+1:]...)`, the underlying array is modified. This is fine for our use case, but be careful in concurrent code -- you would need a mutex.

---

### Step 3: Add JSON File Persistence

Persist the store to disk so tasks survive between runs.

```go
import (
	"encoding/json"
	"os"
)

const defaultFilePath = "tasks.json"

func (s *TaskStore) Save(filepath string) error {
	data, err := json.MarshalIndent(s, "", "  ")
	if err != nil {
		return fmt.Errorf("marshaling tasks: %w", err)
	}

	err = os.WriteFile(filepath, data, 0644)
	if err != nil {
		return fmt.Errorf("writing file %s: %w", filepath, err)
	}
	return nil
}

func LoadTaskStore(filepath string) (*TaskStore, error) {
	data, err := os.ReadFile(filepath)
	if err != nil {
		if os.IsNotExist(err) {
			return NewTaskStore(), nil // fresh start
		}
		return nil, fmt.Errorf("reading file %s: %w", filepath, err)
	}

	var store TaskStore
	if err := json.Unmarshal(data, &store); err != nil {
		return nil, fmt.Errorf("unmarshaling tasks: %w", err)
	}
	return &store, nil
}
```

> [!info] Error Wrapping
> Using `fmt.Errorf("context: %w", err)` wraps the original error so callers can unwrap it with `errors.Is()` or `errors.As()`. Always add context describing what operation failed.

---

### Step 4: Add CLI with Flag Parsing

Wire up the subcommands using `os.Args` for the command and `flag` for options within each command.

```go
import (
	"flag"
	"fmt"
	"os"
	"strconv"
	"text/tabwriter"
)

func printUsage() {
	fmt.Println(`Usage: tasks <command> [options]

Commands:
  add      Add a new task
  list     List tasks
  done     Mark a task as completed
  delete   Delete a task

Run 'tasks <command> -h' for command-specific help.`)
}

func main() {
	if len(os.Args) < 2 {
		printUsage()
		os.Exit(1)
	}

	store, err := LoadTaskStore(defaultFilePath)
	if err != nil {
		fmt.Fprintf(os.Stderr, "Error loading tasks: %v\n", err)
		os.Exit(1)
	}

	command := os.Args[1]

	switch command {
	case "add":
		cmdAdd(store)
	case "list":
		cmdList(store)
	case "done":
		cmdDone(store)
	case "delete":
		cmdDelete(store)
	default:
		fmt.Fprintf(os.Stderr, "Unknown command: %s\n", command)
		printUsage()
		os.Exit(1)
	}

	if err := store.Save(defaultFilePath); err != nil {
		fmt.Fprintf(os.Stderr, "Error saving tasks: %v\n", err)
		os.Exit(1)
	}
}
```

---

### Step 5: Implement Each Subcommand

```go
func cmdAdd(store *TaskStore) {
	fs := flag.NewFlagSet("add", flag.ExitOnError)
	title := fs.String("title", "", "Task title (required)")
	prio := fs.Int("priority", 2, "Priority: 1=low, 2=medium, 3=high")
	due := fs.String("due", "", "Due date (YYYY-MM-DD)")
	fs.Parse(os.Args[2:])

	if *title == "" {
		// Allow positional argument for convenience
		if fs.NArg() > 0 {
			*title = fs.Arg(0)
		} else {
			fmt.Fprintln(os.Stderr, "Error: title is required")
			fs.Usage()
			os.Exit(1)
		}
	}

	if *due != "" {
		if _, err := time.Parse("2006-01-02", *due); err != nil {
			fmt.Fprintf(os.Stderr, "Error: invalid due date format, use YYYY-MM-DD\n")
			os.Exit(1)
		}
	}

	task := store.Add(*title, Priority(*prio), *due)
	fmt.Printf("Added task #%d: %s\n", task.ID, task.Title)
}

func cmdList(store *TaskStore) {
	fs := flag.NewFlagSet("list", flag.ExitOnError)
	filter := fs.String("filter", "all", "Filter: all, pending, done")
	fs.Parse(os.Args[2:])

	tasks := store.List(*filter)
	if len(tasks) == 0 {
		fmt.Println("No tasks found.")
		return
	}

	w := tabwriter.NewWriter(os.Stdout, 0, 4, 2, ' ', 0)
	fmt.Fprintln(w, "ID\tSTATUS\tPRIORITY\tDUE\tTITLE")
	fmt.Fprintln(w, "--\t------\t--------\t---\t-----")

	for _, t := range tasks {
		status := "[ ]"
		if t.Done {
			status = "[x]"
		}
		due := t.DueDate
		if due == "" {
			due = "-"
		}
		fmt.Fprintf(w, "%d\t%s\t%s\t%s\t%s\n",
			t.ID, status, t.Priority, due, t.Title)
	}
	w.Flush()
}

func cmdDone(store *TaskStore) {
	if len(os.Args) < 3 {
		fmt.Fprintln(os.Stderr, "Usage: tasks done <id>")
		os.Exit(1)
	}

	id, err := strconv.Atoi(os.Args[2])
	if err != nil {
		fmt.Fprintf(os.Stderr, "Error: invalid task ID %q\n", os.Args[2])
		os.Exit(1)
	}

	if err := store.Complete(id); err != nil {
		fmt.Fprintf(os.Stderr, "Error: %v\n", err)
		os.Exit(1)
	}
	fmt.Printf("Task #%d marked as done.\n", id)
}

func cmdDelete(store *TaskStore) {
	if len(os.Args) < 3 {
		fmt.Fprintln(os.Stderr, "Usage: tasks delete <id>")
		os.Exit(1)
	}

	id, err := strconv.Atoi(os.Args[2])
	if err != nil {
		fmt.Fprintf(os.Stderr, "Error: invalid task ID %q\n", os.Args[2])
		os.Exit(1)
	}

	if err := store.Delete(id); err != nil {
		fmt.Fprintf(os.Stderr, "Error: %v\n", err)
		os.Exit(1)
	}
	fmt.Printf("Task #%d deleted.\n", id)
}
```

---

### Step 6: Add Due Date Overdue Detection

Enhance the list display to highlight overdue tasks.

```go
func isOverdue(dueDate string) bool {
	if dueDate == "" {
		return false
	}
	due, err := time.Parse("2006-01-02", dueDate)
	if err != nil {
		return false
	}
	return time.Now().After(due)
}
```

Use this in `cmdList` to append `OVERDUE` next to tasks past their due date.

---

## Full Solution

> [!success]- Full Solution
> ```go
> package main
> 
> import (
> 	"encoding/json"
> 	"flag"
> 	"fmt"
> 	"os"
> 	"strconv"
> 	"text/tabwriter"
> 	"time"
> )
> 
> // --- Types ---
> 
> type Priority int
> 
> const (
> 	PriorityLow    Priority = 1
> 	PriorityMedium Priority = 2
> 	PriorityHigh   Priority = 3
> )
> 
> func (p Priority) String() string {
> 	switch p {
> 	case PriorityLow:
> 		return "low"
> 	case PriorityMedium:
> 		return "medium"
> 	case PriorityHigh:
> 		return "high"
> 	default:
> 		return "unknown"
> 	}
> }
> 
> type Task struct {
> 	ID        int       `json:"id"`
> 	Title     string    `json:"title"`
> 	Done      bool      `json:"done"`
> 	Priority  Priority  `json:"priority"`
> 	CreatedAt time.Time `json:"created_at"`
> 	DueDate   string    `json:"due_date,omitempty"`
> }
> 
> type TaskStore struct {
> 	Tasks  []Task `json:"tasks"`
> 	NextID int    `json:"next_id"`
> }
> 
> const defaultFilePath = "tasks.json"
> 
> // --- TaskStore Methods ---
> 
> func NewTaskStore() *TaskStore {
> 	return &TaskStore{
> 		Tasks:  []Task{},
> 		NextID: 1,
> 	}
> }
> 
> func (s *TaskStore) Add(title string, priority Priority, dueDate string) Task {
> 	task := Task{
> 		ID:        s.NextID,
> 		Title:     title,
> 		Done:      false,
> 		Priority:  priority,
> 		CreatedAt: time.Now(),
> 		DueDate:   dueDate,
> 	}
> 	s.Tasks = append(s.Tasks, task)
> 	s.NextID++
> 	return task
> }
> 
> func (s *TaskStore) Complete(id int) error {
> 	for i := range s.Tasks {
> 		if s.Tasks[i].ID == id {
> 			if s.Tasks[i].Done {
> 				return fmt.Errorf("task #%d is already completed", id)
> 			}
> 			s.Tasks[i].Done = true
> 			return nil
> 		}
> 	}
> 	return fmt.Errorf("task with ID %d not found", id)
> }
> 
> func (s *TaskStore) Delete(id int) error {
> 	for i, task := range s.Tasks {
> 		if task.ID == id {
> 			s.Tasks = append(s.Tasks[:i], s.Tasks[i+1:]...)
> 			return nil
> 		}
> 	}
> 	return fmt.Errorf("task with ID %d not found", id)
> }
> 
> func (s *TaskStore) List(filter string) []Task {
> 	if filter == "all" || filter == "" {
> 		return s.Tasks
> 	}
> 	var result []Task
> 	for _, task := range s.Tasks {
> 		switch filter {
> 		case "pending":
> 			if !task.Done {
> 				result = append(result, task)
> 			}
> 		case "done":
> 			if task.Done {
> 				result = append(result, task)
> 			}
> 		}
> 	}
> 	return result
> }
> 
> // --- Persistence ---
> 
> func (s *TaskStore) Save(filepath string) error {
> 	data, err := json.MarshalIndent(s, "", "  ")
> 	if err != nil {
> 		return fmt.Errorf("marshaling tasks: %w", err)
> 	}
> 	if err := os.WriteFile(filepath, data, 0644); err != nil {
> 		return fmt.Errorf("writing file %s: %w", filepath, err)
> 	}
> 	return nil
> }
> 
> func LoadTaskStore(filepath string) (*TaskStore, error) {
> 	data, err := os.ReadFile(filepath)
> 	if err != nil {
> 		if os.IsNotExist(err) {
> 			return NewTaskStore(), nil
> 		}
> 		return nil, fmt.Errorf("reading file %s: %w", filepath, err)
> 	}
> 	var store TaskStore
> 	if err := json.Unmarshal(data, &store); err != nil {
> 		return nil, fmt.Errorf("unmarshaling tasks: %w", err)
> 	}
> 	return &store, nil
> }
> 
> // --- Helpers ---
> 
> func isOverdue(dueDate string) bool {
> 	if dueDate == "" {
> 		return false
> 	}
> 	due, err := time.Parse("2006-01-02", dueDate)
> 	if err != nil {
> 		return false
> 	}
> 	return time.Now().After(due)
> }
> 
> // --- CLI Commands ---
> 
> func printUsage() {
> 	fmt.Println(`Usage: tasks <command> [options]
> 
> Commands:
>   add      Add a new task
>   list     List tasks (--filter=all|pending|done)
>   done     Mark a task as completed
>   delete   Delete a task
> 
> Examples:
>   tasks add -title "Buy groceries" -priority 3 -due 2026-04-10
>   tasks add "Quick task title"
>   tasks list -filter pending
>   tasks done 1
>   tasks delete 2`)
> }
> 
> func cmdAdd(store *TaskStore) {
> 	fs := flag.NewFlagSet("add", flag.ExitOnError)
> 	title := fs.String("title", "", "Task title (required)")
> 	prio := fs.Int("priority", 2, "Priority: 1=low, 2=medium, 3=high")
> 	due := fs.String("due", "", "Due date (YYYY-MM-DD)")
> 	fs.Parse(os.Args[2:])
> 
> 	if *title == "" {
> 		if fs.NArg() > 0 {
> 			*title = fs.Arg(0)
> 		} else {
> 			fmt.Fprintln(os.Stderr, "Error: title is required")
> 			fs.Usage()
> 			os.Exit(1)
> 		}
> 	}
> 
> 	if *prio < 1 || *prio > 3 {
> 		fmt.Fprintln(os.Stderr, "Error: priority must be 1, 2, or 3")
> 		os.Exit(1)
> 	}
> 
> 	if *due != "" {
> 		if _, err := time.Parse("2006-01-02", *due); err != nil {
> 			fmt.Fprintln(os.Stderr, "Error: invalid due date format, use YYYY-MM-DD")
> 			os.Exit(1)
> 		}
> 	}
> 
> 	task := store.Add(*title, Priority(*prio), *due)
> 	fmt.Printf("Added task #%d: %s (priority: %s)\n", task.ID, task.Title, task.Priority)
> }
> 
> func cmdList(store *TaskStore) {
> 	fs := flag.NewFlagSet("list", flag.ExitOnError)
> 	filter := fs.String("filter", "all", "Filter: all, pending, done")
> 	fs.Parse(os.Args[2:])
> 
> 	tasks := store.List(*filter)
> 	if len(tasks) == 0 {
> 		fmt.Println("No tasks found.")
> 		return
> 	}
> 
> 	w := tabwriter.NewWriter(os.Stdout, 0, 4, 2, ' ', 0)
> 	fmt.Fprintln(w, "ID\tSTATUS\tPRIORITY\tDUE\tTITLE")
> 	fmt.Fprintln(w, "--\t------\t--------\t---\t-----")
> 
> 	for _, t := range tasks {
> 		status := "[ ]"
> 		if t.Done {
> 			status = "[x]"
> 		}
> 
> 		due := t.DueDate
> 		if due == "" {
> 			due = "-"
> 		} else if !t.Done && isOverdue(t.DueDate) {
> 			due = t.DueDate + " OVERDUE"
> 		}
> 
> 		fmt.Fprintf(w, "%d\t%s\t%s\t%s\t%s\n",
> 			t.ID, status, t.Priority, due, t.Title)
> 	}
> 	w.Flush()
> 
> 	fmt.Printf("\nTotal: %d tasks\n", len(tasks))
> }
> 
> func cmdDone(store *TaskStore) {
> 	if len(os.Args) < 3 {
> 		fmt.Fprintln(os.Stderr, "Usage: tasks done <id>")
> 		os.Exit(1)
> 	}
> 	id, err := strconv.Atoi(os.Args[2])
> 	if err != nil {
> 		fmt.Fprintf(os.Stderr, "Error: invalid task ID %q\n", os.Args[2])
> 		os.Exit(1)
> 	}
> 	if err := store.Complete(id); err != nil {
> 		fmt.Fprintf(os.Stderr, "Error: %v\n", err)
> 		os.Exit(1)
> 	}
> 	fmt.Printf("Task #%d marked as done.\n", id)
> }
> 
> func cmdDelete(store *TaskStore) {
> 	if len(os.Args) < 3 {
> 		fmt.Fprintln(os.Stderr, "Usage: tasks delete <id>")
> 		os.Exit(1)
> 	}
> 	id, err := strconv.Atoi(os.Args[2])
> 	if err != nil {
> 		fmt.Fprintf(os.Stderr, "Error: invalid task ID %q\n", os.Args[2])
> 		os.Exit(1)
> 	}
> 	if err := store.Delete(id); err != nil {
> 		fmt.Fprintf(os.Stderr, "Error: %v\n", err)
> 		os.Exit(1)
> 	}
> 	fmt.Printf("Task #%d deleted.\n", id)
> }
> 
> // --- Main ---
> 
> func main() {
> 	if len(os.Args) < 2 {
> 		printUsage()
> 		os.Exit(1)
> 	}
> 
> 	store, err := LoadTaskStore(defaultFilePath)
> 	if err != nil {
> 		fmt.Fprintf(os.Stderr, "Error loading tasks: %v\n", err)
> 		os.Exit(1)
> 	}
> 
> 	command := os.Args[1]
> 
> 	switch command {
> 	case "add":
> 		cmdAdd(store)
> 	case "list":
> 		cmdList(store)
> 	case "done":
> 		cmdDone(store)
> 	case "delete":
> 		cmdDelete(store)
> 	case "help", "-h", "--help":
> 		printUsage()
> 		return
> 	default:
> 		fmt.Fprintf(os.Stderr, "Unknown command: %s\n\n", command)
> 		printUsage()
> 		os.Exit(1)
> 	}
> 
> 	if err := store.Save(defaultFilePath); err != nil {
> 		fmt.Fprintf(os.Stderr, "Error saving tasks: %v\n", err)
> 		os.Exit(1)
> 	}
> }
> ```

---

## Extensions / Challenges

- **Search:** Add a `search` command that finds tasks by keyword in the title.
- **Sort:** Sort tasks by priority, due date, or creation date.
- **Categories:** Add a `Category` field and allow filtering by category.
- **Undo:** Keep a history of actions and implement an `undo` command.
- **Colors:** Use ANSI escape codes or a library like `fatih/color` to colorize output (red for overdue, green for completed).
- **Export:** Add CSV or Markdown export alongside JSON.

> [!abstract] Key Takeaways
> - Structs with JSON tags give you free serialization.
> - `flag.NewFlagSet` lets you create per-subcommand flags cleanly.
> - Always handle the "file doesn't exist yet" case gracefully.
> - Wrap errors with `%w` to preserve the error chain.
