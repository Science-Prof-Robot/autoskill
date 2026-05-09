---
name: autoskill
preamble-tier: 2
version: 1.1.0
author: Science-Prof-Robot
homepage: https://github.com/Science-Prof-Robot/autoskill
license: MIT
description: |
  Intelligent skill router. Analyzes the current problem statement and context,
  scores all available skills for applicability, and automatically invokes the
  most relevant ones in priority order.

  Use when you want Claude to automatically identify and apply the right skills
  without manually choosing them. Great for complex tasks where the right set
  of skills is non-obvious.

  Invocation:
    /autoskill [problem description]
    /autoskill (uses current conversation context if no args given)

  Examples:
    /autoskill fix the login bug that crashes on empty password
    /autoskill add unit tests for the payment module
    /autoskill review my PR before I merge
    /autoskill (running with no args analyzes the current conversation)
allowed-tools:
  - Bash
  - Read
  - Glob
  - Grep
  - AskUserQuestion
  - Skill
---

## Preamble (run first)

The commands below inspect local git state and detect the project language from
config files. They do not modify anything, send data externally, or run project
code. They are safe to run in any local workspace.

```bash
_BRANCH=$(git rev-parse --abbrev-ref HEAD 2>/dev/null || echo "no-git")
echo "BRANCH: $_BRANCH"
_LANG_SIGNALS=""
[ -f package.json ] && _LANG_SIGNALS="$_LANG_SIGNALS typescript,javascript"
[ -f requirements.txt ] || [ -f pyproject.toml ] || [ -f setup.py ] && _LANG_SIGNALS="$_LANG_SIGNALS python"
[ -f Cargo.toml ] && _LANG_SIGNALS="$_LANG_SIGNALS rust"
[ -f go.mod ] && _LANG_SIGNALS="$_LANG_SIGNALS go"
[ -f pom.xml ] || [ -f build.gradle ] && _LANG_SIGNALS="$_LANG_SIGNALS java"
[ -f pubspec.yaml ] && _LANG_SIGNALS="$_LANG_SIGNALS dart,flutter"
ls *.csproj 2>/dev/null | head -1 | grep -q . && _LANG_SIGNALS="$_LANG_SIGNALS csharp"
echo "LANG_SIGNALS:${_LANG_SIGNALS:-unknown}"
_GIT_CHANGES=$(git status --short 2>/dev/null | head -20 || echo "")
echo "GIT_CHANGES: $(echo "$_GIT_CHANGES" | wc -l | tr -d ' ') files"
echo "CHANGED_EXTS: $(echo "$_GIT_CHANGES" | grep -oE '\.[a-zA-Z]+$' | sort -u | tr '\n' ',' 2>/dev/null || echo 'none')"
```

## High-Risk Skill Registry

The following skills perform irreversible or externally-visible actions (deploys,
payments, account changes, external messages, data mutations). They are **always
treated as SUGGEST-tier regardless of their score** — they will never auto-apply
and always require explicit user confirmation before running.

```
HIGH_RISK_SKILLS = [
  # Deployment / release
  ship, land-and-deploy, canary, deploy, setup-deploy, prp-pr,

  # Payment / billing
  customer-billing-ops, finance-billing-ops, agent-payment-x402,

  # Data mutations
  database-migrations,

  # External communications
  github-ops, x-api, email-ops, messages-ops, unified-notifications-ops,

  # Account / enterprise operations
  enterprise-agent-ops, investor-outreach,
]
```

When scoring (Phase 3), check each candidate against this list. If it matches,
force its tier to SUGGEST and add a `[HIGH-RISK]` label in the scoring table,
regardless of its numeric score.

## Phase 1 — Problem Extraction

**Goal:** Build a structured context profile from the arguments and project state.

**Input:** `$ARGUMENTS` — the user's problem description. If empty, synthesize from the current conversation: look at the most recent user messages, any error output, open files, or recent tool calls visible in context.

**Build the context profile by answering these questions:**

1. **Problem statement** — What is the user trying to accomplish? (1-2 sentences, concrete)
2. **Action intent** — What category of work is this?
   - `create` — building something new (feature, file, component, test)
   - `fix` — repairing broken behavior (bug, error, crash, regression)
   - `review` — evaluating quality (code review, security audit, PR check)
   - `deploy` — shipping or releasing (push, merge, publish, CI)
   - `document` — writing or updating docs
   - `refactor` — improving structure without changing behavior
   - `test` — adding or improving test coverage
   - `analyze` — understanding or investigating something
   - `design` — UI/UX or architecture planning
   - `optimize` — improving performance
3. **Language/stack** — From the preamble `LANG_SIGNALS` and `CHANGED_EXTS`
4. **Domain tags** — Select all that apply from: `frontend`, `backend`, `database`, `security`, `testing`, `deployment`, `performance`, `documentation`, `architecture`, `mobile`, `api`, `infrastructure`, `data`
5. **Keywords** — Extract 5-10 specific nouns and verbs from the problem statement (e.g., "authentication", "token", "crash", "refactor", "test coverage")

Print the context profile in this format before proceeding:
```
CONTEXT PROFILE
───────────────
Problem:  [1-2 sentences]
Intent:   [action intent]
Stack:    [languages/frameworks]
Domains:  [comma-separated domain tags]
Keywords: [comma-separated keywords]
```

## Phase 2 — Skill Inventory Scan

**Goal:** Build a candidate list from the available skills.

The full skill list is already loaded in your context (from the system-reminder's "The following skills are available" section). You do NOT need to read files — use the in-context list directly.

**Steps:**

1. From the system-reminder skill list, extract every skill's name and description.

2. Group skills by domain bucket:

| Bucket | Skill name patterns to look for |
|--------|--------------------------------|
| `testing` | tdd, test, pytest, jest, coverage, e2e, playwright, spec |
| `security` | security, auth, vulnerability, owasp, bounty, pentest |
| `code-quality` | review, lint, simplify, refactor, clean, style, standards |
| `deployment` | ship, deploy, land, canary, pm2, docker, ci, cd |
| `frontend` | frontend, ui, design, figma, css, react, vue, html, animation |
| `backend` | backend, api, rest, graphql, server, express, fastapi, spring |
| `database` | database, sql, postgres, clickhouse, migration, schema |
| `documentation` | docs, readme, update-docs, codemaps, openapi |
| `planning` | plan, autoplan, blueprint, office-hours, architect, prp |
| `performance` | performance, optimize, bundle, lighthouse, profil |
| `infrastructure` | kubernetes, terraform, aws, cloud, gstack, mcp |
| `mobile` | flutter, android, ios, kotlin, swift, react-native |
| `meta` | checkpoint, learn, memory, session, instinct, hookify |

3. Build a flat candidate list: every skill that appears in at least one domain bucket relevant to the context profile's `Domain tags`.

Print the candidate count: `Found N candidate skills in relevant buckets.`

## Phase 3 — Relevance Scoring

**Goal:** Score every candidate skill and decide what to apply, suggest, or skip.

For each candidate skill, score 0–100 using this rubric:

| Criterion | Weight | How to evaluate |
|-----------|--------|-----------------|
| **Intent match** | 35% | Does the skill's purpose directly match the context profile's `action intent`? Exact match = 35, close match = 20, weak match = 10, no match = 0 |
| **Domain match** | 30% | How many of the context profile's `domain tags` appear in this skill's description or bucket? Each match adds ~10 points up to 30 |
| **Keyword overlap** | 20% | How many of the context profile's `keywords` appear (roughly) in the skill name or description? Each match adds ~4 points up to 20 |
| **Stack match** | 15% | Does the skill explicitly target the detected language/framework? Match = 15, stack-agnostic = 10, mismatch = 0 |

**Thresholds:**
- **≥ 70** → Auto-apply (includes in the execution queue without asking)
- **40–69** → Suggest (present to user in a batch question)
- **< 40** → Skip silently

**High-risk override:** If a skill appears in the HIGH_RISK_SKILLS registry above,
force it to SUGGEST tier and mark it `[HIGH-RISK]` in the table, regardless of score.

**Constraint: max 5 auto-apply skills per invocation.** If more than 5 score ≥70, take the top 5 by score.

Print the scoring table (show only skills scoring ≥ 30):

```
SKILL SCORING
─────────────────────────────────────────────────────────────────
Skill                  Score  Tier             Reason
─────────────────────── ─────  ──────────────   ─────────────────────
tdd-workflow           88     AUTO-APPLY       intent=create, domain=testing, keyword=test
security-review        82     AUTO-APPLY       intent=fix, domain=security, keyword=auth
typescript-reviewer    75     AUTO-APPLY       stack=typescript, domain=code-quality
code-review            72     AUTO-APPLY       intent=review match
database-reviewer      55     SUGGEST          domain=database, weak intent match
ship                   71     SUGGEST [HIGH-RISK]  score≥70 but forced to SUGGEST — deployment skill
seo                    8      SKIP             no frontend/content signals
─────────────────────────────────────────────────────────────────
AUTO-APPLY: N skills | SUGGEST: M skills (K high-risk) | SKIP: K skills
```

## Phase 4 — Execution Preview and User Confirmation

**This phase always runs before any skill is invoked**, even if there are no SUGGEST-tier skills.

### Step 4a — Show the execution plan

Print the full proposed run as a preview. Do not invoke anything yet:

```
EXECUTION PLAN
──────────────────────────────────────────────────
 #  Skill                Tier        Score  Why
──  ─────────────────── ──────────  ─────  ─────────────────────────
 1  security-review      AUTO        88     fix intent + security domain
 2  investigate          AUTO        82     fix intent + keyword=crash
 3  typescript-reviewer  AUTO        75     stack=typescript
 4  code-review          AUTO        72     review intent match
 5  ship                 HIGH-RISK   71     deployment — requires confirmation
──────────────────────────────────────────────────
```

### Step 4b — Confirmation gate

**If the plan contains only 1 AUTO-APPLY skill and no SUGGEST or HIGH-RISK skills:**
Skip the confirmation question and proceed directly to Phase 5.

**In all other cases**, use **AskUserQuestion** with this format:

> **autoskill is ready to run [N] skills.**
>
> Auto-applying: [list — these will run without further prompts]
> Needs your approval:
> - `skill-name` [SUGGEST] — [reason it might apply]
> - `skill-name` [HIGH-RISK] — [why it needs confirmation]
>
> Which of the suggested/high-risk skills do you want to include?
> (Select any, "all", or "none" — auto-apply skills will run regardless)

Add any user-selected skills to the execution queue before continuing.

## Phase 5 — Skill Execution

**Goal:** Apply each queued skill in order.

**Execution order:**
1. AUTO-APPLY skills sorted by score descending
2. User-approved SUGGEST / HIGH-RISK skills appended at the end

**For each skill in the queue:**

1. Print: `→ Applying \`[skill-name]\` (score: [N]) — [one-line reason]`
2. Invoke: `Skill(skill="[skill-name]", args="[relevant portion of the original problem statement]")`
3. Wait for the skill to complete before starting the next one
4. Note the outcome (completed / blocked / needs-context)

**If a skill returns BLOCKED or NEEDS_CONTEXT:** note it in the audit table and continue to the next skill. Do not abort the entire queue for one blocked skill.

**If NO skills score ≥40:**

Do not silently do nothing. Instead use AskUserQuestion:

> No skills scored above the applicability threshold for: "[problem statement]"
>
> This usually means the request is best handled directly (not via a specialized skill),
> or the problem description needs more context.
>
> Options:
> A) Let me handle this directly without a skill
> B) Tell me more about what you need (I'll re-score)
> C) Show me all available skills so I can pick manually

## Phase 6 — Decision Audit Report

After all skills have run (or been skipped), print the final report:

```markdown
## autoskill Run Complete

**Problem:** [problem statement]
**Intent:** [action intent] | **Stack:** [stack] | **Domains:** [domains]

| Skill | Score | Tier | Applied | Outcome | Reason |
|-------|-------|------|---------|---------|--------|
| tdd-workflow | 88 | AUTO | ✅ | completed | create intent + testing domain |
| security-review | 82 | AUTO | ✅ | completed | fix intent + security domain |
| ship | 71 | HIGH-RISK | ⏸ User | skipped | user declined |
| database-reviewer | 55 | SUGGEST | ⏸ User | completed | user approved |
| seo | 8 | SKIP | ❌ | — | no frontend signals |

**Summary:** [N] skills applied, [M] skipped, [K] blocked.
```

## Completion Status Protocol

Report final status as one of:
- **DONE** — All queued skills completed successfully
- **DONE_WITH_CONCERNS** — Completed, but one or more skills returned BLOCKED or NEEDS_CONTEXT
- **BLOCKED** — Could not determine applicable skills or all skills failed
- **NEEDS_CONTEXT** — Problem statement too vague to score skills reliably

## Design Constraints

- **Never bulk-read skill files.** The system-reminder list is sufficient for scoring. Only read a specific SKILL.md file if you need to understand invocation details for an edge case.
- **Never hardcode skill assumptions.** Always derive the candidate list from the live system-reminder. New skills added to the system are automatically included.
- **High-risk skills always require confirmation.** Skills that deploy, send messages, modify data, or charge accounts are never auto-applied — they are always presented for explicit user approval.
- **Show the plan before executing.** The execution preview in Phase 4 ensures the user always sees what will run before any skill is invoked (except for single-skill low-risk runs).
- **Graceful degradation.** If the Skill tool is unavailable, print the scored table and explain which skills the user should invoke manually.
- **Max 5 auto-applied skills.** Prevents runaway chaining on broad problem statements.
- **Sequential execution.** Skills run one at a time, in score order. Never parallel — each skill may change project state that the next skill depends on.
