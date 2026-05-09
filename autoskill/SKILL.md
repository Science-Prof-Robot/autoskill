---
name: autoskill
preamble-tier: 2
version: 1.0.0
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

**Constraint: max 5 auto-apply skills per invocation.** If more than 5 score ≥70, take the top 5 by score.

Print the scoring table (show only skills scoring ≥ 30):

```
SKILL SCORING
─────────────────────────────────────────────────────────────────
Skill                  Score  Tier        Reason
─────────────────────── ─────  ──────────  ─────────────────────
tdd-workflow           88     AUTO-APPLY  intent=create, domain=testing, keyword=test
security-review        82     AUTO-APPLY  intent=fix, domain=security, keyword=auth
typescript-reviewer    75     AUTO-APPLY  stack=typescript, domain=code-quality
code-review            72     AUTO-APPLY  intent=review match
database-reviewer      55     SUGGEST     domain=database, weak intent match
...
seo                    8      SKIP        no frontend/content signals
─────────────────────────────────────────────────────────────────
AUTO-APPLY: N skills | SUGGEST: M skills | SKIP: K skills
```

## Phase 4 — User Confirmation (for SUGGEST tier only)

If there are any SUGGEST-tier skills (score 40–69):

Use **AskUserQuestion** with this format:

> **autoskill found [N] skills to auto-apply and [M] to suggest.**
>
> Auto-applying (score ≥70): [list]
>
> Also applicable (score 40–69) — want any of these?
> - `skill-name` — [one-line reason it might apply]
> - `skill-name` — [one-line reason it might apply]
>
> Which would you like to add to the run? (Enter names, "all", or "none")

If there are NO suggest-tier skills, skip this step and proceed directly to Phase 5.

Add any user-selected skills to the execution queue before continuing.

## Phase 5 — Skill Execution

**Goal:** Apply each queued skill in order.

**Execution order:**
1. Sort auto-apply skills by score descending
2. User-selected suggest-tier skills append at the end

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

| Skill | Score | Applied | Outcome | Reason |
|-------|-------|---------|---------|--------|
| tdd-workflow | 88 | ✅ Auto | completed | create intent + testing domain |
| security-review | 82 | ✅ Auto | completed | fix intent + security domain |
| database-reviewer | 55 | ⏸ User | completed | user added from suggest list |
| seo | 8 | ❌ Skip | — | no frontend signals |

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
- **Graceful degradation.** If the Skill tool is unavailable, print the scored table and explain which skills the user should invoke manually.
- **Max 5 auto-applied skills.** Prevents runaway chaining on broad problem statements.
- **Sequential execution.** Skills run one at a time, in score order. Never parallel — each skill may change project state that the next skill depends on.
