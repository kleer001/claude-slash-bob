---
name: bob
description: >
  Build Our Breadcrumbs — session handoff for Claude Code. Compresses conversation
  context, todos, and state into a structured BREADCRUMB.md so a future session can
  resume without re-explanation. Use this skill whenever the user types /bob with any
  combination of flags (--stay, --force, --dry-run, --clear, --go, --wait). Also
  trigger when the user mentions "breadcrumbs", "session handoff", "pick up where I
  left off", "save my progress", or "resume previous work". /bob with no flags always
  writes and exits. /bob --go or /bob --wait always loads.
allowed-tools: Read, Write, Edit, Glob, Grep, Bash, Task
argument-hint: "[--stay] [--force] [--dry-run] [--clear] [--go] [--wait]"
---

# `/bob` — Build Our Breadcrumbs

Session handoff for Claude Code. Compresses context into `BREADCRUMB.md` at the
project root so a future session can pick up exactly where this one left off.

---

## Core Command Model

There are exactly two modes and they never overlap:

| Command | Mode | What happens |
|---|---|---|
| `/bob` (no load flags) | **Write** | Compress session → write/update `BREADCRUMB.md` → exit |
| `/bob --go` | **Load + autostart** | Read `BREADCRUMB.md` → begin working through all todos |
| `/bob --wait` | **Load + pause** | Read `BREADCRUMB.md` → summarize → wait for user direction |

The presence of `--go` or `--wait` means load. Their absence means write.
There is no ambiguity, no heuristic, no guessing.

---

## BREADCRUMB.md Structure

Always located at the **project root**: `./BREADCRUMB.md`

```
fresh                ← Line 1: session state flag (fresh | stale)

## Summary
[Compressed description of the current task or chain of thought]

## Todos

### Parallel
- [ ] #1 Update README

### Sequential
- [ ] #4 (needs: #2) Deploy to staging

## Context
[Key decisions, file paths, research web links, constraints, gotchas, relevant snippets.
Include the *why* behind dependencies here — reasoning that doesn't fit in a marker.]

## Next Step
[The single most important thing to do when resuming]

## Done
- [x] #2 Add unit tests for auth module   ← kept only because live #4 still (needs: #2)

/absolute/path/to/project   ← Last line: absolute project path
```

### Structural Rules

- **Line 1** is always exactly `fresh` or `stale`. Nothing else. No blank lines before it.
- **Last line** is always the absolute project path. Readable via `tail -1 BREADCRUMB.md`.
- **Section headers** (`## Summary`, `## Todos`, etc.) are recommended but flexible. Use
  judgment on what's worth capturing. The goal is a useful handoff, not bureaucratic compliance.
- **Todo numbers** (`#1`, `#2`, etc.) always increment. Never reuse a number, even if the
  original item was completed or erased — so any surviving `(needs: #N)` reference stays
  unambiguous.
- **Completed todos are erased, not archived.** The breadcrumb is working memory, not a
  history log. When a todo completes, erase it — *unless* an incomplete todo still
  references it via `(needs: #N)`. Keep a completed item in `## Done` only while a live
  dependent still points at it; when the last live dependent finishes, erase it too. A file
  whose work is all finished and unreferenced drains to empty todo sections. `## Done` is a
  small holding area for still-needed anchors, never a growing archive.

---

## Write Mode

Triggered when the user calls `/bob` without `--go` or `--wait`.

### Decision tree:

```
/bob called
  │
  ├─ No BREADCRUMB.md exists → CREATE (write new file)
  │
  ├─ BREADCRUMB.md exists, line 1 = fresh → ADD (merge into existing)
  │
  └─ BREADCRUMB.md exists, line 1 = stale
       │
       ├─ --force flag → overwrite without prompting
       └─ no --force → warn: "A stale breadcrumb exists from a previous session.
                        Overwriting will replace it. Proceed?" → wait for confirmation
```

### Write behavior (CREATE — no file exists):

1. Compress the conversation into `## Summary`.
2. Survey all open work from the session — write each item as a todo in `## Todos`,
   grouped under `### Parallel` and `### Sequential` with `(needs: #N)` markers and
   incrementing `#N` numbers.
3. Survey completed work from the session — record an item in `## Done` (with its original
   number) *only* if an incomplete todo references it via `(needs: #N)`. Erase every other
   completed item; do not record it.
4. Capture key decisions, file paths, research web links, constraints, and dependency
   reasoning in `## Context`, and the single most critical next action in `## Next Step`.
   Capture only what's still relevant — leave out anything already resolved or obsolete.
5. Identify the single most critical next action for `## Next Step`. This is a pointer
   into the todo list — the "start here" item, not the only item.
6. If the session has minimal context (e.g., user called `/bob` immediately), write a
   status-oriented breadcrumb: repo overview, branch state, recent git activity, and an
   empty `## Todos` section. There is always something useful to capture.
7. Set line 1 to `fresh`.
8. Write the absolute current project path (`pwd`) as the last line.
9. **Do not ask the user any questions.** Compress and write silently. Then output exactly one line: `Type /exit to close the session.` and stop.

### Write behavior (ADD — fresh file exists):

1. Read the existing `BREADCRUMB.md`.
2. Identify new todos from the conversation not already in the file — add them with
   new incremented `#N` numbers. Place them under the appropriate `### Parallel` or
   `### Sequential` sub-header based on their dependency characteristics.
3. Identify newly completed todos. Erase each one — *unless* an incomplete todo still
   references it via `(needs: #N)`, in which case move it to `## Done` (preserving its
   original `#N`) as an anchor for that reference.
4. Sweep `## Done` and erase any entry whose last live dependent has now completed —
   once nothing incomplete references it, the anchor is dead weight.
5. Merge new context into `## Context` — new key decisions, file paths, research web links,
   constraints, and dependency reasoning. Compress and deduplicate; do not blindly append.
   **Prune** anything now resolved or obsolete — don't just accumulate. Re-summarize if the
   section is getting long.
6. Update `## Next Step` if a clearer next action has emerged; drop a step that's already done.
7. Update `## Summary` if the scope has meaningfully changed.
8. Keep line 1 as `fresh`. Do not change it.
9. **Do not ask the user any questions.** Then output exactly one line: `Type /exit to close the session.` and stop.

### After writing (both CREATE and ADD) and before exiting:

- If this is the first time writing `BREADCRUMB.md` in this project and a `.git/`
  directory exists, check whether `BREADCRUMB.md` appears in `.gitignore`. If not,
  inform the user and offer to add it:
  ```
  # Claude Code session breadcrumbs
  BREADCRUMB.md
  ```
  This check is stateless: just grep `.gitignore` for the filename. No tracking needed.
- Unless `--stay` is set, output exactly one line: `Type /exit to close the session.`
  then stop responding entirely. Do not add commentary, summaries, or follow-up messages.
  `/bob` is the command a user runs when they're done — closing the laptop, stepping away.
- Unless `--dry-run` is set, actually write the file. See the flags section below.

---

## Load Mode

Triggered when the user calls `/bob --go` or `/bob --wait`.

### Decision tree:

```
/bob --go or /bob --wait called
  │
  ├─ No BREADCRUMB.md exists → inform user: "No breadcrumbs found. Nothing to load."
  │
  ├─ BREADCRUMB.md exists, line 1 = fresh
  │    → mark line 1 as stale
  │    → load and summarize
  │    → proceed per flag (--go: autostart, --wait: pause)
  │
  ├─ BREADCRUMB.md exists, line 1 = stale
  │    ├─ --force → load without prompting, do NOT change line 1 (stays stale)
  │    └─ no --force → warn: "These breadcrumbs were already loaded in a previous
  │         session and may reflect outdated state. Some todos may already be done.
  │         Proceed?" → wait for confirmation → on confirm, load
  │
  └─ BREADCRUMB.md exists but malformed (line 1 is not fresh/stale, structure
       is unrecognizable, file is empty, etc.)
       → alert user: "The breadcrumbs file seems to be malformed. I can do my best
         to interpret it, but I can't autostart reliably. Here's what I understand:
         [summary of parseable content]. Want me to proceed with what I have, or
         can you provide more context?"
       → wait for direction regardless of --go/--wait
```

### After loading:

1. Summarize what was loaded in plain language. Use friendly phrasing like "Picking up
   where you left off" — never surface raw `fresh`/`stale` terminology to the user.
2. Check the last line against the current working directory (`pwd`). If they differ:
   ```
   ⚠️ These breadcrumbs were written in a different project directory
   ([stored path]). Loaded anyway — please verify the context still applies.
   ```
   Do not block. The user may have intentionally moved the file.
3. Survey todos — if loading stale breadcrumbs and any items appear already completed
   (based on file state, git history, or other signals), notify the user.

### Autostart behavior (`--go`):

When autostarting, read all three layers of todo information before beginning any work:

1. **Sub-headers**: `### Parallel` vs `### Sequential` for the big picture.
2. **Inline markers**: `(needs: #N)` for specific dependency chains.
3. **Context prose**: `## Context` for the reasoning behind dependencies.

Execution strategy:
- **Parallel todos**: Dispatch these as concurrent sub-agents using the Task tool. They
  have no dependencies on each other and can run simultaneously.
- **Sequential todos**: Execute in dependency order. A todo with `(needs: #3)` waits
  until `#3` is complete. As todos finish, apply the prune rule: erase each completed todo
  unless an incomplete one still references it, so `## Done` only ever holds live anchors.
- **Blocked todos**: If a todo cannot proceed (missing information, requires a decision,
  external dependency), surface the blocker to the user immediately. Do not silently skip it.
  Do not continue past it if downstream todos depend on it.
- **All todos complete**: When every todo from the breadcrumb is finished, notify the
  user with a clear completion summary and then delete `BREADCRUMB.md`. Do not delete
  silently — the user must see what was accomplished and know the file is gone:
  ```
  ✅ All breadcrumb todos complete (3/3 finished). Cleaning up — deleting BREADCRUMB.md.
  ```
  The task chain is finished. Leaving a stale breadcrumb would create noise on the next session.

### Wait behavior (`--wait`):

Summarize everything that was loaded, then stop and wait for the user to direct next steps.

---

## Flags Reference

| Flag | Mode | Effect |
|---|---|---|
| *(none)* | Write | Write/add breadcrumbs, then exit |
| `--stay` | Write | Write/add breadcrumbs, do **not** exit |
| `--force` | Either | Skip all confirmation prompts (stale warnings, overwrite warnings) |
| `--dry-run` | Write | Show a plain-language summary of what would be written (or what would change in add mode). No file changes. No exit. |
| `--clear` | N/A | Prompt for confirmation, then delete `BREADCRUMB.md` |
| `--clear --force` | N/A | Delete `BREADCRUMB.md` immediately, no confirmation |
| `--go` | Load | Load breadcrumbs and autostart all todos |
| `--wait` | Load | Load breadcrumbs and wait for user direction |

### Flag Combination Rules

- `--force` combines with any flag. It suppresses confirmation prompts, nothing else.
- `--stay` only applies to write mode. It suppresses the exit, nothing else.
- `--dry-run` only applies to write mode. It suppresses file writes and exit. It
  combines cleanly with `--force` (dry-run still shows output, just no prompts).
- `--go` and `--wait` are mutually exclusive. Do not combine them.
- `--go` and `--wait` define load mode. Do not combine them with `--stay` or `--dry-run`
  (which are write-mode flags).
- `--clear` is standalone. The only flag it combines with is `--force`. Ignore any other
  flags if `--clear` is present.

### Invalid combinations

If the user provides an invalid combination (e.g., `--go --stay`), explain the conflict
briefly and ask them to clarify. Do not guess intent.

---

## Edge Cases

### Race condition (two sessions, one project)
Session A writes fresh breadcrumbs. Session B loads them (marks stale). Session A calls
`/bob` again expecting add mode but finds the file is stale. This is an inherent limitation
of file-based state with no locking mechanism. The stale warning in this case is the correct
behavior — it alerts Session A that something changed. This is documented, not a bug.

### Path mismatch
The last line of `BREADCRUMB.md` records where the file was written. On load, compare
against `pwd`. Warn on mismatch but do not block. Users may intentionally copy breadcrumb
files between projects.

### Todo completion during autostart
When running `--go`, if all todos complete successfully, always notify the user with an
explicit completion summary before deleting `BREADCRUMB.md`:
```
✅ All breadcrumb todos complete (N/N finished). Cleaning up — deleting BREADCRUMB.md.
```
Never delete silently. The user should see what was accomplished and know the file is gone.
The task chain is finished — leaving a stale breadcrumb would just create noise on the
next session.

### Empty or trivial session on write
Never refuse to write. Even with minimal conversation, capture: the current branch, recent
commits, repo structure notes, working directory. An empty `## Todos` section is fine.
The breadcrumb still serves as a snapshot of project state.

---

## UX Principles

- **Never ask questions during write mode.** `/bob` is the "I'm done for now" command.
  Do your best with what you have, write the file, get out.
- **Use friendly language on load.** "Picking up where you left off" — not "loading fresh
  breadcrumb file with autostart flag."
- **Stop after writing.** When write mode completes (without `--stay`), output
  `Type /exit to close the session.` and stop. Do not continue with follow-up messages.
- **Compression over completeness.** The breadcrumb should be dense and useful, not a
  transcript. Future Claude needs the *decisions*, not the deliberation.
- **Working memory, not an archive.** The breadcrumb is a scratch pad — put things in
  temporarily, erase what's no longer needed. Every write pass prunes: erase completed
  todos (keeping only those a live `(needs: #N)` still anchors), and drop resolved or
  obsolete `## Context` / `## Next Step` content. Don't let `## Done` grow into a history log.
- **Deduplicate aggressively in add mode.** Don't let the file bloat. Re-summarize
  context on each add pass. Merge, don't append.
- **Clean up when done.** When all todos from a `--go` session are complete, notify the
  user with a completion summary and delete `BREADCRUMB.md`. Never delete silently.
