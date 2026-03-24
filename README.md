<img src="docs/images/banner.svg" alt="/bob — Close the session. Keep the thread." width="900"/>

A [Claude Code](https://claude.ai/code) skill for session handoff. Compresses conversation context, todos, and state into a `BREADCRUMB.md` file so a future session can resume without re-explanation.

## Install

```bash
claude plugins marketplace add kleer001/claude-slash-bob && claude plugins install bob
```

## Usage

| Command | What it does |
|---|---|
| `/bob` | Compress session → write `BREADCRUMB.md` → exit |
| `/bob --stay` | Same, but don't exit |
| `/bob --dry-run` | Preview what would be written |
| `/bob --go` | Load breadcrumbs → autostart all todos |
| `/bob --wait` | Load breadcrumbs → wait for your direction |
| `/bob --force` | Skip stale/overwrite confirmation prompts |
| `/bob --clear` | Delete `BREADCRUMB.md` |

Also triggers on phrases like "save my progress", "resume previous work", or "pick up where I left off".

## How it works

`/bob` (write mode) captures: a compressed summary, open todos (parallel vs. sequential with dependency markers), key decisions and constraints, and the single most important next step.

`/bob --go` / `--wait` (load mode) reads `BREADCRUMB.md`, marks it consumed, and either autostarts work or waits for direction.

See [`SKILL.md`](SKILL.md) for the full behavioral spec.
