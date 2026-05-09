# autoskill

A meta-skill for Claude Code that automatically identifies and applies the right skills for any problem — without you having to know which skill to pick.

## What it does

When you invoke `/autoskill`, it:

1. **Reads your problem** — from your description or infers it from the current conversation
2. **Scans all available skills** — using the live skill list already in Claude's context (no slow file reads)
3. **Scores each skill** on four criteria: intent match, domain match, keyword overlap, and stack signals
4. **Auto-applies** the top matches (score ≥ 70, max 5 at a time)
5. **Suggests** borderline matches (score 40–69) for you to approve
6. **Reports** a decision audit table showing every skill with its score, tier, and reason

## Install

**Option 1 — ClaWHub (recommended)**

Search for `autoskill` on [clawhub.io](https://clawhub.io) and click Install. ClaWHub handles copying the skill files into `~/.claude/` automatically.

**Option 2 — Manual**

```bash
bash autoskill/install.sh
```

Then restart Claude Code (or start a new session).

## Adding skills in Claude Code and Cursor

**Claude Code (CLI / desktop app)**

Drop a `SKILL.md` file into `~/.claude/skills/<skill-name>/` and a corresponding `.md` command file into `~/.claude/commands/`. Restart the session — the skill appears in `/` autocomplete automatically.

**Cursor**

Cursor does not have a native skill system. To use autoskill inside Cursor, add the contents of `SKILL.md` as a custom rule in **Cursor Settings → Rules for AI**, or paste it into a `.cursorrules` file at your project root. Invoke it by asking Claude: "follow the autoskill workflow for: [your problem]".

## Usage

```
/autoskill fix the login crash on empty password
/autoskill add unit tests for the payment module
/autoskill review my PR before I merge
/autoskill                    # infers from current conversation
```

## How scoring works

Each candidate skill is scored 0–100:

| Criterion | Weight | What it checks |
|-----------|--------|----------------|
| Intent match | 35% | Does the skill's purpose match what you want to DO? (fix / create / review / deploy / etc.) |
| Domain match | 30% | Does the skill apply to the relevant domain? (security, testing, frontend, database, etc.) |
| Keyword overlap | 20% | How many of your problem's keywords appear in the skill name or description? |
| Stack match | 15% | Does the skill target your detected language or framework? |

**Thresholds:**
- **≥ 70** — applied automatically
- **40–69** — suggested, you decide
- **< 40** — skipped silently

## Example output

```
CONTEXT PROFILE
───────────────
Problem:  Fix login crash when password field is empty
Intent:   fix
Stack:    typescript
Domains:  security, backend, api
Keywords: login, crash, password, empty, authentication

Found 12 candidate skills in relevant buckets.

SKILL SCORING
──────────────────────────────────────────────────────────────
Skill               Score  Tier        Reason
─────────────────── ─────  ──────────  ──────────────────────
security-review     88     AUTO-APPLY  fix intent + security domain + keyword=auth
investigate         82     AUTO-APPLY  fix intent + keyword=crash
typescript-reviewer 75     AUTO-APPLY  stack=typescript, code-quality domain
code-review         72     AUTO-APPLY  review intent match
tdd-workflow        48     SUGGEST     testing domain, weak intent match
──────────────────────────────────────────────────────────────
AUTO-APPLY: 4 skills | SUGGEST: 1 skill | SKIP: 7 skills
```

## Files

```
autoskill/
├── SKILL.md      # Full skill definition read by Claude Code
└── install.sh    # Copies skill + command alias into ~/.claude/
```

After install, the skill lives at `~/.claude/skills/autoskill/SKILL.md` and the slash command at `~/.claude/commands/autoskill.md`.
