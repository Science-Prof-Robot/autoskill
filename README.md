# autoskill

[![ClawHub](https://img.shields.io/badge/clawhub-autoskill%401.1.1-blue)](https://github.com/Science-Prof-Robot/autoskill)
[![License: MIT](https://img.shields.io/badge/license-MIT-green)](LICENSE)

**The skill you invoke when you don't know which skill — or which MCP tool — to invoke.**

`autoskill` is a meta-skill for Claude Code that acts as an intelligent router. Instead of memorizing which specialized skill handles which kind of task, or which MCP tool to reach for, you describe your problem and autoskill figures out the right items to apply — scoring every available skill **and every attached MCP tool**, auto-applying the best matches, and letting you approve borderline ones.

---

## The problem it solves

Claude Code ships with a large and growing library of specialized skills: security reviewers, TDD guides, frontend pattern libraries, database reviewers, deployment helpers, and more. Knowing which one to reach for — and in what combination — adds cognitive overhead to every task.

`autoskill` removes that overhead. It reads your problem, scans the live skill list, scores each skill on four criteria, and invokes the top matches automatically. You get the right expert for the job without having to know who that is.

---

## How it works

When you run `/autoskill`, it executes a six-phase pipeline:

### Phase 1 — Problem Extraction
Reads your description (or infers it from the current conversation) and builds a structured context profile: problem statement, action intent, language/stack, domain tags, and keywords.

### Phase 2 — Inventory Scan (Skills + MCP Tools)
Scans the live skill list **and the deferred MCP tool list** already in Claude's context — no slow file reads, no discovery probes. Groups skills into domain buckets (testing, security, frontend, backend, database, deployment, etc.) and groups MCP tools by server (Signoz, Slack, Gmail, Calendar, Drive, Jenkins, context7, …) mapped to the same buckets. Builds two candidate lists relevant to your domains.

### Phase 3 — Relevance Scoring
Scores every candidate skill **and every candidate MCP tool** 0–100 using the same weighted rubric:

| Criterion | Weight | What it checks |
|-----------|--------|----------------|
| Intent match | 35% | Does the item's purpose match what you want to do? (fix / create / review / deploy / etc.) For MCP tools, derived from the tool's verb prefix (`get` / `search` / `create` / `send` / …). |
| Domain match | 30% | Does the item apply to the relevant domain? (security, testing, frontend, database, observability, communication, …) |
| Keyword overlap | 20% | How many of your problem's keywords appear in the item's name or description? |
| Stack match | 15% | Does the item target your detected language or framework? (MCP tools are usually stack-agnostic.) |

**Score thresholds:**
- **≥ 70** — recommended, shown in the execution plan
- **40–69** — suggested, you decide whether to include
- **< 40** — skipped silently

### Phase 4 — Execution Preview and Mandatory Confirmation
**Every run** shows the full proposed execution plan — skills and MCP tools together, grouped by kind — and asks for your explicit approval before invoking anything. There are no exceptions; even a single recommended item requires confirmation. You can remove anything from the queue before proceeding.

### Phase 5 — Execution
Runs approved skills first, then approved MCP tools, one at a time in score order. MCP tool schemas are loaded via `ToolSearch` just before invocation. If an MCP tool's schema needs a field autoskill can't safely infer (a channel ID, a recipient, an alert rule ID), it asks you rather than guessing. Each item may change project or remote state that the next one depends on, so execution is always sequential. Blocked or context-starved items are noted but don't abort the rest.

### Phase 6 — Decision Audit Report
Prints a full table showing every skill that was considered, its score, whether it was applied or skipped, and why. Nothing is hidden.

---

## Install

### Quick install (natural language)

**Claude Code**
```
/install autoskill
```

**Cursor**
```
@autoskill
```
Or open Cursor Settings → MCP / Skills → Add Skill → search `autoskill`.

**OpenClaw**
```bash
npx clawhub@latest install autoskill
```

### Manual install (any environment)

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

SKILL & MCP SCORING
─────────────────────────────────────────────────────────────────────────
Kind   Name                            Score  Tier              Reason
────── ───────────────────────────── ─────  ───────────────   ──────────
skill  security-review                 88     RECOMMENDED       fix intent + security domain + keyword=auth
skill  investigate                     82     RECOMMENDED       fix intent + keyword=crash
mcp    Signoz__search_logs             84     RECOMMENDED       observability + keyword=crash
skill  typescript-reviewer             75     RECOMMENDED       stack=typescript, code-quality
skill  code-review                     72     RECOMMENDED       review intent match
skill  tdd-workflow                    48     SUGGEST           testing domain, weak intent match
mcp    Slack__slack_send_message       64     SUGGEST [HIGH-RISK]  write tool — forced to SUGGEST
─────────────────────────────────────────────────────────────────────────
RECOMMENDED: 5 (4 skill, 1 mcp) | SUGGEST: 2 (1 high-risk) | SKIP: 7

** autoskill recommends 5 items for: "Fix login crash when password field is empty" **

These will run only after you confirm below:

Recommended skills (score ≥70):
- `security-review` — fix intent + security domain
- `investigate` — fix intent + keyword=crash
- `typescript-reviewer` — stack=typescript
- `code-review` — review intent match

Recommended MCP tools (score ≥70):
- `mcp__claude_ai_Signoz__search_logs` — observability + keyword=crash

Also applicable — want any of these?
- `tdd-workflow` [SUGGEST] — testing domain, weak intent match
- `mcp__claude_ai_Slack__slack_send_message` [HIGH-RISK] — write tool

Actions:
- Type the names of any suggested/high-risk items you want to add
- Type "none" to run only the recommended items
- Type "cancel" to stop and do nothing

→ Applying `security-review` (score: 88) — fix intent + security domain
[security-review output...]

→ Applying `investigate` (score: 82) — fix intent + crash keyword
[investigate output...]

→ Loading schema for `mcp__claude_ai_Signoz__search_logs` …
→ Invoking `mcp__claude_ai_Signoz__search_logs` (score: 84) — observability + keyword=crash
[Signoz log results...]

## autoskill Run Complete

| Kind  | Name                            | Score | Tier        | Applied | Outcome   | Reason |
|-------|---------------------------------|-------|-------------|---------|-----------|--------|
| skill | security-review                 | 88    | RECOMMENDED | ✅      | completed | fix intent + security domain |
| skill | investigate                     | 82    | RECOMMENDED | ✅      | completed | fix intent + keyword=crash |
| mcp   | Signoz__search_logs             | 84    | RECOMMENDED | ✅      | completed | observability + keyword=crash |
| skill | typescript-reviewer             | 75    | RECOMMENDED | ✅      | completed | stack=typescript |
| skill | code-review                     | 72    | RECOMMENDED | ✅      | completed | review intent match |
| skill | tdd-workflow                    | 48    | SUGGEST     | ⏸       | skipped   | user declined |
| mcp   | Slack__slack_send_message       | 64    | HIGH-RISK   | ⏸       | skipped   | user declined |

Summary: 5 items applied (4 skill, 1 mcp), 2 skipped, 0 blocked.
```

---

## Security and safe use

autoskill is designed to route to other skills automatically. Before installing, be aware of how it handles high-impact actions:

**Mandatory execution preview.** Every single run shows the full proposed execution plan and asks for your explicit confirmation before invoking anything. There are no exceptions — even one recommended skill requires approval. You can remove any skill from the queue at that point, or type `cancel` to abort entirely.

**High-risk skill gate.** Skills that deploy, send messages, modify data, charge accounts, or access broad shell/file scope (e.g. `ship`, `database-migrations`, `customer-billing-ops`, `github-ops`, `email-ops`) are **never automatically included** — they always appear as optional suggestions marked `[HIGH-RISK]`, regardless of their score. A keyword heuristic also catches newly added skills with dangerous descriptions even if they are not in the fixed registry.

**High-risk MCP gate.** MCP tools that send messages, create drafts or labels, modify calendars, upload to Drive, reset rate limits, authenticate against third-party providers, or otherwise mutate remote state are also **never auto-included** — they appear as `[HIGH-RISK]` suggestions just like deployment/billing skills. The gate is driven by both a name-pattern registry (`*__slack_send_*`, `*__create_draft`, `*__create_event`, `*__authenticate`, `*__update_*`, …) and a verb heuristic (`send`, `create`, `update`, `delete`, `authenticate`, `reset`, `export`, …). Read-verbs (`get`, `list`, `search`, `query`, `read`) are not gated.

**Preamble transparency.** The Bash preamble that runs at startup only inspects local git state and project config files (`package.json`, `go.mod`, etc.). It does not modify files, run project code, or send data externally.

**Persistent install.** The installer adds a skill and slash command to `~/.claude/`. They remain available in future sessions until you uninstall them (see Uninstall above).

## Design principles

- **No hardcoded skill or MCP assumptions.** Both candidate lists are derived from the live system-reminders, so newly added skills and newly attached MCP servers are automatically included without any changes to autoskill itself.
- **No bulk file reads, no discovery probes.** The in-context skill list and deferred-tools list are sufficient for scoring. autoskill only reads a specific SKILL.md when it needs invocation details for an edge case, and only loads MCP schemas via `ToolSearch` immediately before invoking them in Phase 5.
- **Independent caps: max 5 auto-applied skills + max 5 auto-applied MCP tools.** Prevents runaway chaining on broad problem statements and prevents one kind from crowding out the other.
- **Sequential execution.** Skills first, then MCP tools, one at a time. Each may change project or remote state that the next one depends on.
- **Graceful degradation.** If the Skill tool or `ToolSearch` is unavailable, autoskill prints the scored table and tells you which items to invoke manually.

---

## Files

```
autoskill/
├── SKILL.md      # Full skill definition (the six-phase pipeline)
└── install.sh    # Copies skill + slash command into ~/.claude/

scripts/
└── publish_to_clawhub.sh   # Maintainer script for ClawHub releases
```
