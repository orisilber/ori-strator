# ori-strator

Iterative PRD solver for [Claude Code](https://claude.com/claude-code). Each iteration runs in a fresh git worktree with a cleared agent context, takes a bite-sized chunk of a planner-generated todo list, and runs an architect → coder → reviewer agent trio. Memories tagged by task carry lessons forward; per-iteration commits and branch chaining preserve progress.

The whole design is built to **avoid context drift**: each iteration starts from scratch, sees only what's relevant to its chunk, and produces a single atomic commit.

## What it does

Given a `prd.md`, the plugin:

1. Spawns a **planner** that decomposes the PRD into bite-sized tasks (`tasks.md`).
2. Loops:
   - Picks the next chunk of tasks (default 1).
   - Creates a new git worktree, branched from the last `READY` iteration (or `main` on the first run).
   - Spawns an **architect** with a fresh context. The architect spawns a **coder** (writes code + tests) and a **reviewer** (runs the tests, judges quality, gives a per-task verdict).
   - Optional validation gate (typecheck / targeted test).
   - The architect commits the iteration's work, writes tagged memories for the next iteration.
3. Stops when all tasks are `done`, max iterations is hit, or all remaining tasks are blocked / stuck.

## Install

```
/plugin marketplace add orisilber/ori-strator
/plugin install ori-strator@ori-strator
```

Then reload plugins (or restart Claude Code) and run:

```
/ori-strator:solve-prd path/to/prd.md
/ori-strator:solve-prd path/to/prd.md 15        # 15 max iterations (default 10)
/ori-strator:solve-prd path/to/prd.md 15 2      # ...with 2 tasks per iteration
```

> The repo is private — make sure your local git is authenticated to GitHub (`gh auth status`) so the plugin manager can clone it.

## Files written in your project

The plugin only writes inside the project you invoke it from.

```
.claude/prd-solver/
  state/<prd-slug>/
    tasks.md                  # planner's todo list, architect updates statuses
    run-log.md                # iteration history (used for stuck detection)
    last-ready-branch.txt     # branch-chaining pointer
  memories/<prd-slug>/
    iter-N-<topic>.md         # tagged memories (YAML frontmatter: tasks, scope)

.prd-solver-worktrees/<prd-slug>/iter-N/   # one worktree per iteration
```

You probably want to gitignore both `.prd-solver-worktrees/` and `.claude/prd-solver/state/`.

## Customizing prompts

The architect, coder, reviewer, and planner each have an editable prompt file. By default the plugin's bundled prompts are used.

To override per-project, drop a same-named file in your project's `.claude/prd-solver/` directory:

- `.claude/prd-solver/planner-prompt.md`
- `.claude/prd-solver/architect-prompt.md`
- `.claude/prd-solver/coder-prompt.md`
- `.claude/prd-solver/reviewer-prompt.md`
- `.claude/prd-solver/config.json`

The orchestrator prefers project-local files when present and falls back to the plugin defaults. It prints which source was loaded at startup.

To customize globally, edit the files in your plugin install dir (typically `~/.claude/plugins/ori-strator/prd-solver/`). Note that plugin updates will overwrite those edits; project-local overrides survive updates.

## Config

`config.json`:

| Field | Default | Description |
| --- | --- | --- |
| `chunk_size` | `1` | Tasks per iteration. CLI 3rd arg overrides. Keep small (1–3) to avoid context drift. |
| `validation_command` | `null` | Optional shell command run inside the iteration's worktree after a `READY` verdict. Non-zero exit downgrades the iteration to `NEEDS_WORK`. Example: `"npm run typecheck"`. |
| `validation_timeout_seconds` | `120` | Hard cap on the validation command. |

## How the loop avoids context drift

- **Each iteration is a fresh agent context.** Architect, coder, reviewer never see prior iterations' transcripts.
- **`tasks.md` is the persistent backbone.** The planner writes it once; the architect only flips chunk task statuses and appends discovered tasks at the end. Existing tasks are never rewritten.
- **Memories are tagged.** The architect reads only memories whose `tasks:` frontmatter intersects the current chunk, plus `scope: global` ones — not the whole memory dump.
- **Branch chaining.** Iteration N+1 branches from the last `READY` iteration's branch, so completed work accumulates without each iteration starting from `main`.
- **The architect ignores the coder's narrative.** It judges from the staged diff alone, then hands the diff to the reviewer.
- **The reviewer runs the tests and critiques their quality** — not just whether they pass, but whether they're meaningful (no constant-asserting, no mock-asserting, no tautologies).
- **Stuck detection.** If a task fails 3 consecutive iterations, it's auto-marked `blocked` and surfaced for human resolution.
- **One commit per iteration.** Iterations are atomic, inspectable records on dedicated branches.

## Stopping conditions

The loop ends on the first of:
- All non-blocked tasks are `done`.
- `max-iterations` reached (default 10, configurable per invocation).
- No runnable chunk remains (everything is blocked, in-progress, or done).

## Replanning

- To re-plan from scratch: delete `.claude/prd-solver/state/<prd-slug>/tasks.md`. Next run will re-invoke the planner.
- To wipe all progress: delete `.claude/prd-solver/state/<prd-slug>/`. Worktrees can be cleaned up with `git worktree prune` after removing `.prd-solver-worktrees/<prd-slug>/`.

## Layout of this repo

```
.claude-plugin/
  plugin.json              # plugin manifest
  marketplace.json         # marketplace catalog
commands/
  solve-prd.md             # the slash command (orchestrator)
prd-solver/
  planner-prompt.md
  architect-prompt.md
  coder-prompt.md
  reviewer-prompt.md
  config.json
```

## License

MIT
