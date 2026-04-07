# AGENTS.md

This file provides guidance to AI assistants (Claude Code, Copilot, Cursor, etc.) when working in this repository.

## Overview

This is an **Obsidian vault** — a personal knowledge base using Markdown files. It is not a software project; there are no build steps, tests, or dependencies to manage.

The owner uses this vault as a structured learning system, capturing study notes, progressive practice drills, and real-world project walkthroughs. Content is organized into self-contained topic folders.

---

## Vault Structure

```
brain/
  AGENTS.md                    ← this file (AI guidance)
  CLAUDE.md                    ← symlink to AGENTS.md
  KanbanWorks.md               ← kanban board for tracking work
  README.md
  Golang/                      ← Golang study course
    Golang Study.md            ← main reference note
    Golang Practice/           ← 12 topic drill files
    Golang Workflows/          ← 5 real-world project files
  Kubernetes/                  ← Kubernetes study course
    Kubernetes Study.md        ← main reference note
    Docker/                    ← Docker practice
    Core/                      ← K8s core practice (4 topics)
    Helm/                      ← Helm practice
    ArgoCD/                    ← ArgoCD GitOps practice
    GitHub Actions/            ← CI/CD practice
    Workflows/                 ← 5 real-world project files
  .obsidian/                   ← Obsidian config (do not edit manually)
```

---

## Installed Plugins

- **obsidian-git** — Automatically commits vault changes to git on a schedule. Commit messages follow the pattern `vault backup: YYYY-MM-DD HH:MM:SS`.
- **obsidian-kanban** — Powers Kanban board notes (identified by `kanban-plugin: board` frontmatter). Preserve the `%% kanban:settings %%` block exactly when editing.

---

## Working with Notes

- Notes use standard Markdown plus Obsidian extensions: `[[wikilinks]]`, `#tags`, YAML frontmatter, and callouts.
- Use `[[wikilinks]]` for internal vault links. Use `[text](url)` for external URLs only.
- Do not commit `.obsidian/workspace.json` or `.obsidian/workspace-mobile.json` — these track UI state and change frequently.

---

## Owner's Learning Style and Preferences

The vault owner learns through **structured, progressive repetition** ("Asian-style" drilling). When creating study content, always follow these patterns:

### Learning Philosophy
- **Muscle memory first** — problems are designed to be written from scratch repeatedly, not copy-pasted
- **Progressive difficulty** — each problem in a set adds exactly one new concept on top of the previous
- **Real-world context** — examples should reflect actual production scenarios, not toy examples
- **Hands-on local practice** — prefer tools that run locally (minikube for K8s, go run for Go, etc.)

### Note Structure Convention

Every **study course** follows this 3-layer structure:

```
Topic/
  Topic Study.md        ← layer 1: comprehensive reference + index
  Topic Practice/       ← layer 2: progressive drill files (12-15 problems each)
  Topic Workflows/      ← layer 3: real-world projects with full solutions
```

### Frontmatter Standard

```yaml
---
title: "Note Title"
date: YYYY-MM-DD
tags:
  - topic
  - subtopic
  - study        # for reference notes
  - practice     # for drill files
  - workflow     # for project files
aliases:
  - Alternative Name
status: in-progress   # or: complete, backlog
parent: "[[Parent Note]]"   # for drill/workflow files
---
```

### Callout Usage

The owner uses Obsidian callouts heavily for visual structure:

| Callout | Use case |
|---|---|
| `> [!tip]` | Best practices, shortcuts, pro advice |
| `> [!warning]` | Common gotchas, traps, mistakes |
| `> [!info]` | Neutral background info, comparisons |
| `> [!abstract]` | Course overview, structure summaries |
| `> [!example]` | Explaining a pattern section |
| `> [!success]` | Positive outcomes, completed steps |
| `> [!danger]` | Critical errors, data-loss risks |
| `> [!hint]- Hint` | Collapsible hint for practice problems |
| `> [!success]- Solution` | Collapsible full solution for practice problems |

### Practice Problem Format

Each practice problem in drill files must follow this template:

```markdown
### Problem N: Title

Brief description of what to implement. Include real-world context
(e.g., "You are building a user service that...").

**Expected output:**
\`\`\`
expected result here
\`\`\`

> [!hint]- Hint
> One-sentence nudge. Don't give away the answer.

> [!success]- Solution
> \`\`\`go
> // complete, runnable code
> \`\`\`
```

### Workflow Project Format

Workflow files are full real-world projects with:
1. **Architecture diagram** (ASCII art)
2. **Step-by-step incremental build** (6-8 steps, each building on the last)
3. **Full solution code** in a collapsible `> [!success]- Full Solution` callout
4. **Extensions / challenges** for extra practice

---

## Content Creation Rules

When the owner asks to create new study content:

1. **Always create the 3-layer structure**: reference note + practice folder + workflows folder
2. **12-15 problems per topic drill** — problems must be progressive (P1 is trivial, P12-15 is complex)
3. **5 workflow projects per course** — realistic end-to-end scenarios using multiple concepts
4. **Wikilinks between layers** — main reference links down to practice files; practice files link back to parent via frontmatter
5. **No shallow examples** — even simple problems should use realistic variable names, struct fields, and scenarios
6. **Local-first tooling** — prefer locally runnable solutions (minikube, go run, docker compose) over cloud-only

---

## File Naming Conventions

| Type | Pattern | Example |
|---|---|---|
| Main reference | `Topic Study.md` | `Golang Study.md` |
| Practice drill | `NN - Topic Name.md` | `01 - Variables and Types.md` |
| Practice (K8s style) | `Practice - Topic Name.md` | `Practice - Pods and Deployments.md` |
| Workflow project | `NN - Project Name.md` | `01 - CLI Task Manager.md` |
| Kanban board | `*Kanban*.md` or `KanbanWorks.md` | `KanbanWorks.md` |

---

## Completed Courses

| Course | Main Reference | Topics Covered | Status |
|---|---|---|---|
| **Golang** | [[Golang/Golang Study]] | Variables, Functions, Pointers, Structs, Interfaces, Slices/Maps, Errors, Defer/Panic, Goroutines, Concurrency, Generics, Testing | in-progress |
| **Kubernetes** | [[Kubernetes/Kubernetes Study]] | Docker, Pods, Deployments, Services, Networking, Config/Storage, Workloads, Helm, ArgoCD, GitHub Actions | in-progress |

---

## AI Assistant Notes

- **Do not modify `.obsidian/`** configuration files
- **Do not add emojis** unless explicitly requested
- **Preserve kanban `%% settings %%` blocks** exactly as-is
- **Always use wikilinks** `[[Note]]` for internal references, never markdown links for internal notes
- **Match the callout style** — use the callout types listed above consistently
- **Symlink awareness** — `CLAUDE.md` is a symlink to this file (`AGENTS.md`). Edit `AGENTS.md` directly; never overwrite the symlink target.
