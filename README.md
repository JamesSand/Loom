# claudeloop

A lightweight task console for Claude Code. You give each task a goal,
the deep-interview Claude pane refines a `PLAN.md`, you spin up one or
more git worktrees on `zhongzhu/<slug>` branches, and `/goal` your way
through the work in the same Claude session — or any resumed one.

There is no agent loop, no autonomous worker, and no evaluator: you
drive Claude yourself, the console just removes the bookkeeping.

```
┌─ Project root (e.g. ~/work/xorl) ──────────────────────────────────┐
│ .RUD/                                                              │
│ ├── NOTES.md            ← project-scoped scratchpad (Notes button) │
│ ├── task-order.json                                                 │
│ └── <slug>/                                                         │
│     ├── task.json       ← title, goal, skills_path, worktrees,…    │
│     ├── PLAN.md         ← deep-interview writes this, you edit it  │
│     ├── INTERVIEW.md    ← transcript log of the deep-interview     │
│     └── work/                                                       │
│         ├── xorl-internal/   ← git worktree, branch zhongzhu/<slug>│
│         └── xorl-sglang/     ← second worktree, same branch name   │
└────────────────────────────────────────────────────────────────────┘
```

## Install

```bash
cd /path/to/claudeloop
pip install -e .

# Authenticate the Claude Code CLI used by the interview pane.
claude
```

Optional but needed for the Claude pane:

```bash
tmux -V       # tmux must be on PATH
git --version
```

## Run

From any directory:

```bash
claudeloop web --project /path/to/your/project --port 8765
# open http://127.0.0.1:8765/
```

Useful flags:

| Flag | Purpose |
|------|---------|
| `--project PATH` | Project root. Defaults to `$PWD`. Can be a git repo OR a container directory with several git repos inside. |
| `--skills PATH` | Default skills markdown injected into every Claude session. Defaults to `claudeloop/skills/AK_skills.md`. |
| `--projects` | Multi-project workspace: launch dir is a container; prune the parent registry entry once children are registered. |
| `--auth-token TOKEN` | Require HTTP basic / bearer auth. Username can be anything; password = token. Also reads `CLAUDELOOP_WEB_AUTH_TOKEN`. |
| `--daemon` / `--nohup` | Re-spawn in the background and exit. Logs land in `<project>/.RUD/web.log`. |
| `--openclaw …` | Optional OpenClaw gateway events; full flag list in `claudeloop/cli.py`. |

```bash
claudeloop init   # writes minimal PLAN.md + NOTES.md in $PWD
claudeloop --help # all commands
```

## Web UI flow

### 1. Register a project

Use **+ Add repo** in the top bar. Anything you register is just stored
in `~/.claudeloop/web-projects.json` (no files are written outside the
registered path). Pick from the subfolder chips if the launch directory
is a container.

### 2. Create a task

**Create Task** asks only for two things:

- **Title** — becomes the slug (lowercased, dash-separated).
- **General goal** — a short description; the deep-interview will turn
  it into PLAN.md.

A `.RUD/<slug>/` directory is created with an empty `PLAN.md`. If the
project root is itself a git repo, **one worktree is auto-created** at
`.RUD/<slug>/work/<repo-name>/` on branch `zhongzhu/<slug>`. Container
projects (no `.git` of their own) skip auto-creation — you'll pick the
source repo from the Claude tab.

You can rename the title or rewrite the goal at any time by clicking on
them in the task header.

### 3. The Claude tab

Each task has a single Claude tab with:

- **Worktrees** card — list of every git worktree owned by this task,
  with branch, live `git status` (clean / N modified / ↑3 ↓2), and a
  **Push** button per row. **+ Add worktree** opens a picker showing
  candidate git repos (project root if it's a repo, else direct
  children); already-added ones are dimmed. **Push all** pushes every
  branch in one go. The × removes a worktree (calls
  `git worktree remove`).
- **Sessions** card — every Claude Code session UUID the pane has ever
  spawned (collected by scanning `~/.claude/projects/<encoded-cwd>/`).
  Click **Resume** on any session to launch a fresh tmux pane with
  `claude --resume <uuid>` — works even if the original tmux was
  killed.
- **Live tmux pane** preview that polls every 4s.
- INTERVIEW.md raw + rendered view underneath.

Starting the pane runs `tmux new-session` + `claude` in the **primary**
worktree (the first entry in the worktrees list) and pastes the
deep-interview prompt 5 seconds later. Special keys (↑ ↓ ← → Enter Esc
Ctrl-C) live in a keycap row next to the textarea.

### 4. PLAN.md tab

A markdown editor + live preview for `.RUD/<slug>/PLAN.md`. The Save
button has a `•` dot when there are unsaved changes; **Cmd / Ctrl + S**
saves. The interview pane writes here when you click *Start Claude* the
first time; afterwards you edit and `/goal` against it inside the
Claude pane.

### 5. Notes (project-scoped)

The **📓 Notes** button in the top bar opens a fullscreen modal editor
for `<project>/.RUD/NOTES.md`. One file per project, persistent, not
tied to a task. Use it for future ideas, open questions, links to PRs,
etc. **Cmd / Ctrl + S** saves.

## CLI reference

There are only two CLI commands now (everything else lives in the web
UI):

```bash
claudeloop init                          # PLAN.md + NOTES.md in $PWD
claudeloop web --project DIR --port N    # the web UI
claudeloop web --daemon                  # background; logs to .RUD/web.log
```

## Storage layout

```
<project>/.RUD/
├── NOTES.md              # project-scoped scratch (📓 Notes button)
├── task-order.json
└── <slug>/
    ├── task.json
    ├── PLAN.md
    ├── INTERVIEW.md
    └── work/
        └── <repo>/...    # git worktree, branch zhongzhu/<slug>
```

```
~/.claude/projects/<encoded-cwd>/
└── <session-uuid>.jsonl  # native Claude Code session transcripts
```

```
~/.claudeloop/
└── web-projects.json     # registered project paths
```

## HTTP API

Everything is plain JSON; pass `?project=<id>` (or the
`X-ClaudeLoop-Project` header) to scope.

| Method | URL | Purpose |
|--------|-----|---------|
| `GET`  | `/api/project` | Active project root + skills path |
| `GET`  | `/api/projects` | List registered projects, default, launch root |
| `POST` | `/api/projects` `{path}` | Register a project root |
| `POST` | `/api/projects/<id>/activate` | Set the default project |
| `POST` | `/api/projects/<id>/move` `{direction}` | Reorder |
| `POST` | `/api/projects/reorder` `{ids}` | Persist arbitrary order |
| `DELETE` | `/api/projects/<id>` | Drop from registry (files untouched) |
| `GET`  | `/api/notes` | Project NOTES.md |
| `PUT`  | `/api/notes` `{content}` | Save project NOTES.md |
| `GET`  | `/api/tasks` | All tasks for the active project |
| `POST` | `/api/tasks` `{title, general_goal}` | Create task (auto-worktree if project root is a git repo) |
| `POST` | `/api/tasks/reorder` `{slugs}` | Persist order |
| `GET`  | `/api/tasks/<slug>` | Meta + PLAN.md + INTERVIEW.md + Claude summary + worktree statuses |
| `PUT`  | `/api/tasks/<slug>/meta` `{title?, general_goal?}` | Rename / re-goal |
| `PUT`  | `/api/tasks/<slug>/template` `{name, content}` | Write PLAN.md |
| `DELETE` | `/api/tasks/<slug>` | Delete task tree (also unregisters worktrees) |
| `GET`  | `/api/tasks/<slug>/worktree-candidates` | Repos you could base a worktree on |
| `POST` | `/api/tasks/<slug>/worktree` `{source_repo}` | Create a worktree |
| `DELETE` | `/api/tasks/<slug>/worktree?path=…` | `git worktree remove` + prune meta |
| `POST` | `/api/tasks/<slug>/worktree/push` `{path}` | `git push -u origin <branch>` |
| `POST` | `/api/tasks/<slug>/worktrees/push-all` | Push every task worktree branch |
| `POST` | `/api/tasks/<slug>/claude/start` | Launch Claude pane in primary worktree |
| `POST` | `/api/tasks/<slug>/claude/stop` | Kill tmux pane (sessions on disk stay resumable) |
| `POST` | `/api/tasks/<slug>/claude/resume` `{session_id}` | New tmux, `claude --resume <id>` |
| `GET`  | `/api/tasks/<slug>/claude-sessions` | Tracked UUIDs + on-disk transcripts |
| `GET`  | `/api/tmux/sessions?project=<id>` | claudeloop-related tmux sessions for that project |
| `GET`  | `/api/tmux/capture?target=…&lines=N` | Pane scrollback |
| `POST` | `/api/tmux/send-text` `{target, text, submit}` | Type into a pane |
| `POST` | `/api/tmux/send-key` `{target, key}` | Send a tmux key |

(For backwards compatibility, `/api/tasks/<slug>/interview/{start,stop}`
still resolves to the Claude pane endpoints.)

## What this used to be

Earlier versions had a full `claudeloop run` / `claudeloop tmux` agent
loop with runner / evaluator panes, plus an `Ask` pane, plus a
TASK_PROMPT.md + SUCCESS_CONDITION.md + auto-commit + multi-repo
worktree picker, plus integration with an external "OpenClaw" gateway
for wake-up signals. The agent loop was deleted on purpose — the new
flow is "human drives Claude Code, console keeps the worktrees and
notes tidy".

If you need the old behaviour, check `git log` for the `Remove worker
tab and append K8S notes…` commit and earlier.
