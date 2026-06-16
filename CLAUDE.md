# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Purpose

This repo implements the `/bob` Claude Code skill — "Build Our Breadcrumbs" — a session handoff mechanism. The deliverable is **`SKILL.md`** (a Claude Code skill definition), not the Python code in `src/`. The Python package is scaffolding for tests and future utilities.

`SKILL.md` is the authoritative spec. All behavior for `/bob` is defined there — do not deviate from it when implementing features.

## Commands

```bash
# Run all tests
pytest

# Run a single test
pytest tests/test_main.py::test_example

# Run with coverage
pytest --cov=src
```

Dependencies are in `requirements.txt`. A `.venv/` is present in the repo root.

## Architecture

```
SKILL.md          # The skill definition — this IS the product
src/main.py       # Python entry point (currently a stub)
tests/test_main.py
```

**Skill format** (`SKILL.md`): YAML frontmatter block followed by Markdown. The frontmatter defines `name`, `description`, `allowed-tools`, and `argument-hint`. Claude Code reads this file and activates the skill when triggered.

**Key behavioral contract in SKILL.md:**
- Two non-overlapping modes: **Write** (`/bob` with no load flags) and **Load** (`/bob --go` or `--wait`)
- `BREADCRUMB.md` always has `fresh`/`stale` as line 1, and absolute project path as the last line
- Todo numbering never reuses `#N` identifiers — completed todos are erased unless an incomplete todo still references them via `(needs: #N)`, in which case they wait in `## Done` until the last live dependent finishes
- Write mode never asks questions; it compresses and exits silently

## CI

GitHub Actions (`.github/workflows/test.yml`) runs `pytest` on push and PR.
