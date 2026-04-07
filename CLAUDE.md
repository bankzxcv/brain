# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is an Obsidian vault — a personal knowledge base using Markdown files. It is not a software project; there are no build steps, tests, or dependencies to manage.

## Structure

- `.obsidian/` — Obsidian configuration (plugins, workspace layout). Do not edit manually.
- `*.md` — Notes in Markdown format, including Obsidian-flavored extensions.

## Installed Plugins

- **obsidian-git** — Automatically commits vault changes to git on a schedule. Commit messages follow the pattern `vault backup: YYYY-MM-DD HH:MM:SS`.
- **obsidian-kanban** — Powers Kanban board notes (identified by `kanban-plugin: board` frontmatter).

## Working with Notes

- Notes use standard Markdown plus Obsidian extensions: `[[wikilinks]]`, `#tags`, YAML frontmatter, and Dataview queries.
- Kanban board files contain a `%% kanban:settings %%` block at the bottom — preserve it exactly when editing.
- Do not commit `.obsidian/workspace.json` or `.obsidian/workspace-mobile.json` unless intentional — these track UI state and change frequently.
