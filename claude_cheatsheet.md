# Claude Code — 30-Minute Cheatsheet (for experienced engineers)

Claude Code is an **agentic coding tool that runs in your terminal** (also in VS Code/JetBrains and a desktop app). It reads your codebase, edits files, runs commands, and drives git — all via natural language, gated by a permission system. Think of it as a coworker with a bash shell, not an autocomplete.

---

## 1. The mental model (read this first)

There are exactly **six primitives**. Everything else is UI on top of these:

| Primitive | Lives in | What it's for |
|---|---|---|
| **CLAUDE.md** | repo root / `~/.claude/` / subfolders | Persistent project memory. Read automatically every session. |
| **Skills** | `.claude/skills/<name>/SKILL.md` | Reusable prompt+scripts. Invoked with `/name` **or** auto-triggered when relevant. |
| **Subagents** | `.claude/agents/<name>.md` | Isolated Claude instances with their own context window for focused subtasks. |
| **Slash commands** | `.claude/commands/<name>.md` (legacy) | Saved prompt templates. Being merged into Skills. |
| **Hooks** | `.claude/settings.json` | Shell commands fired on lifecycle events (auto-format, lint, block unsafe ops). |
| **MCP servers** | `claude mcp add …` / `.mcp.json` | Plug external systems (GitHub, DBs, Jira, browsers) in as tools. |

**Plugins** just bundle several of these into one installable unit.

Two facts that save you grief:
- **Session** = one `claude` run in a dir. History is per-session; **CLAUDE.md persists across sessions.**
- The **context window** fills up as Claude reads files. You manage it with `/compact` and `/clear`. Degraded output usually means a bloated context, not a dumb model.

---

## 2. Install & launch

```bash
# Native binary (recommended)
curl -fsSL https://claude.ai/install.sh | bash
# or Homebrew (macOS):  brew install --cask claude-code
# npm install is deprecated; migrate with `claude install`

cd your-project
claude                      # start interactive session
claude "fix the failing auth test"   # start with an initial prompt
claude -p "run lint and summarize"   # headless: print result and exit
claude -c   / claude --continue      # resume most recent session
claude --model <alias>               # pin a model for the session
```

First thing in any new repo: run **`/init`** — it scans the project and generates a starter `CLAUDE.md`.

---

## 3. Essential slash commands

Type `/` on an empty prompt to see everything available in *your* setup (built-ins + your skills + MCP + plugins). The ones you'll actually reach for:

**Context & session**
- `/init` — generate starter CLAUDE.md
- `/clear` — wipe conversation, keep project memory (aliases `/reset`, `/new`)
- `/compact` — summarize history to reclaim context (takes optional focus hint)
- `/context` — visualize what's eating your context window
- `/rewind` — roll conversation **and/or code** back to a checkpoint
- `/resume` (`/continue`) — reopen a past session
- `/btw` — ask a side question without polluting the main thread
- `/export`, `/copy` — save/copy the conversation or last block

**Working**
- `/plan` — enter Plan Mode (read-only: explore + propose, no edits)
- `/diff` — interactive viewer of uncommitted + per-turn changes
- `/memory` — view/edit CLAUDE.md files in scope
- `/add-dir` — grant access to another directory this session

**Config & extension**
- `/model` — view/switch model · `/permissions` — tool approval rules
- `/config` (`/settings`) · `/agents` — subagents · `/hooks` · `/skills` · `/mcp`

**Meta**
- `/cost`, `/usage` — spend & rate-limit status
- `/status`, `/doctor` (diagnose install), `/feedback`

---

## 4. Keyboard shortcuts & permission modes

| Key | Action |
|---|---|
| **Shift+Tab** | Cycle permission mode: `default → acceptEdits → plan` (+ `auto`/`bypass` if enabled) |
| **Esc** | Interrupt Claude mid-response |
| **Esc, Esc** | Open rewind/checkpoint menu |
| **Ctrl+R** | Reverse-search prompt history |
| **Ctrl+O** | Expand full verbose transcript |
| **`@path`** | Reference a file/dir in your prompt |
| **`#text`** | Quick-add a line to CLAUDE.md from chat |
| **`?`** | Show shortcuts for your terminal/IDE |

**Permission modes** are the single most important lever for speed vs. safety:
- **default** — asks before anything touching your machine
- **acceptEdits** — auto-approves file edits, still prompts for shell commands
- **plan** — read-only; nothing gets changed
- `--dangerously-skip-permissions` — full auto (only in throwaway/sandboxed envs)

---

## 5. CLAUDE.md — your highest-leverage file

Read every session. Keep it **short** (a lean 30-line file beats an ignored 300-line one). A good skeleton:

```markdown
# Project
One line on what this is.

## Stack
- language/framework + versions

## Commands
- dev:   <how to run>
- test:  <how to run tests>
- build: <build cmd>

## Architecture
- services/  business logic, throws DomainError
- queries/   read-only, no side effects
- (a map of where things live — makes Plan Mode dramatically more accurate)

## Conventions
- one or two non-obvious rules

## Don't
- traps that look inviting but break things
```

Scopes stack: `~/.claude/CLAUDE.md` (personal, all projects) → repo root (team, commit it) → subfolder (local overrides).

---

## 6. Plan Mode (the habit that pays off)

For anything multi-step or multi-file: **Shift+Tab into Plan Mode** (or `/plan`, or just say "plan first"). Claude explores read-only, proposes a plan, and waits. You approve, tweak ("change step 3 to…"), or discard. This routinely saves 2–3 correction cycles versus letting it charge ahead. Pair it with an architecture map in CLAUDE.md.

---

## 7. Subagents & skills (when to use which)

- **Slash command / skill** → "insert this prompt template." Use a **skill** when there's real domain logic, structured input, or helper files; a plain command when it's just a canned prompt.
- **Subagent** → isolated, parallel work in a *separate context window* (up to ~10 in parallel). Great for "go read the whole codebase and tell me how X works" without polluting your main thread. Best practice: give subagents **read-only** tools and let the parent handle edits/approvals.

Custom subagent (`.claude/agents/code-reviewer.md`):
```markdown
---
name: code-reviewer
description: Reviews code for quality, security, maintainability. Use after edits.
tools: Read, Grep, Glob, Bash
model: inherit
---
You are a senior reviewer. On invoke: run git diff, then flag issues by priority.
```

Custom skill (`.claude/skills/security-scan/SKILL.md`):
```markdown
---
name: security-scan
description: Run a security review of the diff. Use before opening a PR.
allowed-tools: Read, Grep, Glob, Bash
model: claude-opus-4-8
---
Audit the diff between main and HEAD for injection, XSS, exposed secrets, insecure config.
```

Custom commands support `$ARGUMENTS` / `$1 $2` templating and inline shell with `` !`git status` `` (output injected into the prompt before Claude reasons).

---

## 8. Hooks (enforce rules with code, not vibes)

Shell commands fired on lifecycle events, configured in `.claude/settings.json`. Common events: `PreToolUse`, `PostToolUse`, `UserPromptSubmit`, `SessionStart`, `Stop`, `PreCompact`. Env vars passed in include `CLAUDE_FILE_PATH`, `CLAUDE_TOOL_NAME`, `CLAUDE_PROJECT_DIR`.

```json
{
  "hooks": {
    "PostToolUse": [
      { "matcher": "Edit|Write",
        "hooks": [{ "type": "command", "command": "npx prettier --write $CLAUDE_FILE_PATH || true" }] }
    ]
  }
}
```

Classic uses: auto-format on write, run lint/tests, block `rm -rf`, prepend team rules on every prompt, ping a sound on completion.

---

## 9. MCP — external tools

```bash
# Remote HTTP server
claude mcp add --transport http github https://api.githubcopilot.com/mcp/
# Local stdio server with env
claude mcp add --transport stdio postgres --env "DATABASE_URL=…" -- npx -y <server>
claude mcp list        # see connected servers
```
Scopes: `--scope project` (shared via `.mcp.json`) vs `--scope user` (personal). Manage live in-session with `/mcp`.

---

## 10. Git & CI workflow

- Claude drives git directly — ask it to branch, commit (it writes Conventional Commit messages from the diff), and open PRs. For PRs and CI-log fixing, install the GitHub CLI (`gh auth login`) so it can call `gh`.
- **Headless / CI:** `claude -p "…"` runs without a TTY — drop it into a GitHub Action or nightly job. It reuses the same settings, hooks, and permission rules. Add `--output-format stream-json` for parseable events.
- **Checkpoints:** every turn is a checkpoint. **Esc, Esc** (or `/rewind`) rolls code *and* conversation back — cheaper than fighting a bad direction.

---

## 11. The 30-minute path

1. **(5 min)** Install, `cd` into a real repo, run `claude`, then `/init`. Skim the generated CLAUDE.md and trim it.
2. **(5 min)** Ask a plain question ("how does auth work here?"). Watch it read files. Try `@path` to point it at one.
3. **(5 min)** Give a small real task in **Plan Mode** (Shift+Tab). Approve the plan, watch it edit, `/diff` the result.
4. **(5 min)** Learn the context loop: `/context`, `/compact`, `/clear`. Notice how output quality tracks context health.
5. **(5 min)** Cycle permission modes with Shift+Tab. Do a task in `acceptEdits`. Break something and `/rewind`.
6. **(5 min)** Add one hook (auto-format) and one custom skill or subagent. That's the whole extensibility model.

## Senior-engineer habits that actually matter
- **Invest in CLAUDE.md and Plan Mode** before anything fancy — 80% of the value.
- **Compact early, compact often.** Treat context like RAM.
- **One task per session.** Start fresh (`/clear` or new session) between unrelated tasks instead of `/model`-swapping mid-thread.
- **Read the diff.** `/diff` before you commit; you're the reviewer.
- **Codify repeated prompts** into skills/commands — executable team documentation, reviewed in PRs like code.
- `/help` and `/` are the source of truth for your exact version; it ships weekly, so menus drift.

---
*Verified against Anthropic's official Claude Code docs & help center, July 2026. Model lineup you'll see in `/model`: Opus 4.8, Sonnet, Haiku, Fable 5 — switch per task with `/model` and tune reasoning depth with `/effort`.*
