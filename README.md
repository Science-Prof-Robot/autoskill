# autoskill

[![ClawHub](https://img.shields.io/badge/clawhub-autoskill%401.1.0-blue)](https://github.com/Science-Prof-Robot/autoskill)
[![License: MIT](https://img.shields.io/badge/license-MIT-green)](LICENSE)

**The skill you invoke when you don't know which skill to invoke.**

`autoskill` is a meta-skill for Claude Code that acts as an intelligent router. Instead of memorizing which specialized skill handles which kind of task, you describe your problem and autoskill figures out the right skills to apply — scoring every available skill, auto-applying the best matches, and letting you approve borderline ones.

---

## The problem it solves

Claude Code ships with a large and growing library of specialized skills: security reviewers, TDD guides, frontend pattern libraries, database reviewers, deployment helpers, and more. Knowing which one to reach for — and in what combination — adds cognitive overhead to every task.

`autoskill` removes that overhead. It reads your problem, scans the live skill list, scores each skill on four criteria, and invokes the top matches automatically. You get the right expert for the job without having to know who that is.

---

## How it works

When you run `/autoskill`, it executes a six-phase pipeline:

### Phase 1 — Problem Extraction
Reads your description (or infers it from the current conversation) and builds a structured context profile: problem statement, action intent, language/stack, domain tags, and keywords.

### Phase 2 — Skill Inventory Scan
Scans the live skill list already in Claude's context — no slow file reads. Groups all skills into domain buckets (testing, security, frontend, backend, database, deployment, etc.) and builds a candidate list relevant to your domains.

### Phase 3 — Relevance Scoring
Scores every candidate skill 0–100 using a weighted rubric:

| Criterion | Weight | What it checks |
|-----------|--------|----------------|
| Intent match | 35% | Does the skill's purpose match what you want to do? (fix / create / review / deploy / etc.) |
| Domain match | 30% | Does the skill apply to the relevant domain? (security, testing, frontend, database, etc.) |
| Keyword overlap | 20% | How many of your problem's keywords appear in the skill name or description? |
| Stack match | 15% | Does the skill target your detected language or framework? |

**Score thresholds:**
- **≥ 70** — auto-applied, no confirmation needed
- **40–69** — suggested, you decide
- **< 40** — skipped silently

### Phase 4 — User Confirmation
If any skills scored 40–69, autoskill pauses and asks which you want to add to the run. High-confidence matches (≥70) proceed without interruption, up to a max of 5.

### Phase 5 — Skill Execution
Runs approved skills one at a time in score order. Each skill may change project state that the next one depends on, so execution is always sequential. Blocked or context-starved skills are noted but don't abort the rest.

### Phase 6 — Decision Audit Report
Prints a full table showing every skill that was considered, its score, whether it was applied or skipped, and why. Nothing is hidden.

---

## Install

```bash
bash autoskill/install.sh
```

Restart Claude Code (or start a new session) to pick up `/autoskill`.

After install:
- Skill definition → `~/.claude/skills/autoskill/SKILL.md`
- Slash command → `~/.claude/commands/autoskill.md`

### Uninstall

```bash
rm -rf ~/.claude/skills/autoskill
rm -f  ~/.claude/commands/autoskill.md
```

---

## Usage

```
/autoskill fix the login crash on empty password
/autoskill add unit tests for the payment module
/autoskill review my PR before I merge
/autoskill refactor the auth module to use the repository pattern
/autoskill                    # infers from the current conversation
```

Pass a problem description as the argument, or omit it and autoskill synthesizes from recent messages, open files, and tool call context.

---

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

→ Applying `security-review` (score: 88) — fix intent + security domain

[security-review output...]

→ Applying `investigate` (score: 82) — fix intent + crash keyword

[investigate output...]

## autoskill Run Complete

| Skill              | Score | Applied   | Outcome   | Reason                          |
|--------------------|-------|-----------|-----------|----------------------------------|
| security-review    | 88    | ✅ Auto   | completed | fix intent + security domain     |
| investigate        | 82    | ✅ Auto   | completed | fix intent + keyword=crash       |
| typescript-reviewer| 75    | ✅ Auto   | completed | stack=typescript                 |
| code-review        | 72    | ✅ Auto   | completed | review intent match              |
| tdd-workflow       | 48    | ⏸ User   | skipped   | user declined                    |

Summary: 4 skills applied, 1 skipped, 0 blocked.
```

---

## Security and safe use

autoskill is designed to route to other skills automatically. Before installing, be aware of how it handles high-impact actions:

**High-risk skill gate.** Skills that deploy, send messages, modify data, or charge accounts (e.g. `ship`, `database-migrations`, `customer-billing-ops`, `github-ops`, `email-ops`) are **never auto-applied** — they always appear in the confirmation step regardless of their score, marked `[HIGH-RISK]`.

**Execution preview.** When more than one skill will run, autoskill shows you the full proposed execution plan and asks for approval before invoking anything. You can remove any skill from the queue at that point.

**Preamble transparency.** The Bash preamble that runs at startup only inspects local git state and project config files (`package.json`, `go.mod`, etc.). It does not modify files, run project code, or send data externally.

**Persistent install.** The installer adds a skill and slash command to `~/.claude/`. They remain available in future sessions until you uninstall them (see Uninstall above).

## Design principles

- **No hardcoded skill assumptions.** The candidate list is always derived from the live system-reminder, so new skills are automatically included without any changes to autoskill itself.
- **No bulk file reads.** The in-context skill list is sufficient for scoring. autoskill only reads a specific SKILL.md when it needs invocation details for an edge case.
- **Max 5 auto-applied skills.** Prevents runaway chaining on broad problem statements.
- **Sequential execution.** Skills run one at a time. Each may change project state that the next depends on.
- **Graceful degradation.** If the Skill tool is unavailable, autoskill prints the scored table and tells you which skills to invoke manually.

---

## Files

```
autoskill/
├── SKILL.md      # Full skill definition (the six-phase pipeline)
└── install.sh    # Copies skill + slash command into ~/.claude/

scripts/
└── publish_to_clawhub.sh   # Maintainer script for ClawHub releases
```
