---
name: autoskill
preamble-tier: 2
version: 1.2.0
author: Science-Prof-Robot
homepage: https://github.com/Science-Prof-Robot/autoskill
license: MIT
description: |
  Intelligent skill and MCP-tool router. Analyzes the current problem statement
  and context, scores all available Skills **and** MCP tools for applicability,
  and recommends the most relevant ones in priority order. **Nothing is ever
  invoked without your explicit approval.**

  Also scores and routes MCP tools attached to the session (Signoz, Slack,
  Gmail, Calendar, Drive, Jenkins, context7, etc.) using the same rubric and
  the same mandatory confirmation gate.

  Use when you want Claude to automatically identify and recommend the right
  skills and MCP tools without manually choosing them. Great for complex tasks
  where the right set of capabilities is non-obvious.

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
  - ToolSearch
  - mcp__*
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

The following skills perform irreversible, externally-visible, or broadly-scoped
actions. They are **always treated as SUGGEST-tier** — they will never be
recommended for automatic inclusion and always require individual user
confirmation before running.

```
HIGH_RISK_SKILLS = [
  # Deployment / release / CI
  ship, land-and-deploy, canary, deploy, setup-deploy, prp-pr, deployment-patterns,

  # Payment / billing / money
  customer-billing-ops, finance-billing-ops, agent-payment-x402,

  # Database mutations
  database-migrations,

  # External communications / public posts
  github-ops, x-api, email-ops, messages-ops, unified-notifications-ops,
  crosspost, content-writer,

  # Account / enterprise / credential operations
  enterprise-agent-ops, investor-outreach, cso, security-bounty-hunter,

  # Broad shell / file system access
  careful, guard, safety-guard,
]
```

**Additional heuristic:** Any skill whose description contains keywords such as
"deploy", "payment", "billing", "money", "purchase", "credential", "account",
"external message", "post to", "send email", "database migration",
"DROP TABLE", "rm -rf", "force-push", or "broad shell" is also treated as
high-risk even if it is not in the fixed list above.

When scoring (Phase 3), check each candidate against this registry and the
heuristic. If it matches, force its tier to SUGGEST and add a `[HIGH-RISK]`
label in the scoring table, regardless of its numeric score.

## High-Risk MCP Tool Registry

MCP tools can have external side effects — sending Slack messages, drafting
emails, creating calendar events, modifying Drive files, authenticating
against third-party providers, or mutating remote state. The following name
patterns are **always treated as SUGGEST-tier** — they will never be
recommended for automatic inclusion and always require individual user
confirmation before running.

```
HIGH_RISK_MCP_PATTERNS = [
  # External communications
  *__slack_send_*, *__slack_schedule_*, *__slack_update_*, *__slack_create_*,
  *__send_message*, *__send_email*, *__send_*_to_slack,
  *__create_draft, *__label_*, *__unlabel_*,
  *__create_label, *__delete_label, *__update_label,

  # Calendar / scheduling
  *__create_event, *__update_event, *__delete_event, *__respond_to_event,

  # File system / drive writes
  *__create_file, *__copy_file, *__download_file_content,
  *__update_canvas, *__create_canvas,

  # Auth / credential flows
  *__authenticate, *__complete_authentication,

  # Mutating ops on external systems
  *__reset_*, *__delete_*, *__update_*, *__create_*,
  *__charge_*, *__capture_*, *__refund_*, *__export_*,
]
```

**Heuristic:** any MCP tool whose suffix verb is one of `send`, `create`,
`update`, `delete`, `write`, `post`, `publish`, `authenticate`, `charge`,
`refund`, `reset`, `export`, `schedule`, `draft`, or `upload` is forced to
SUGGEST tier and tagged `[HIGH-RISK]`, regardless of numeric score.

Read-verbs (`get`, `list`, `search`, `query`, `read`, `aggregate`, `fetch`,
`resolve`, `suggest`) are **not** gated — they may be auto-recommended on
their own merits.

When scoring (Phase 3), check each MCP candidate against this registry and
the heuristic alongside the skill registry. The same `[HIGH-RISK]` label and
tier override apply.

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

## Phase 2 — Inventory Scan (Skills + MCP Tools)

**Goal:** Build two candidate lists — one for available Skills, one for
available MCP tools.

Both inventories come from in-context system-reminders. You do NOT need to
read files or probe with `ToolSearch` during this phase — schemas only load
on demand in Phase 5.

### 2a. Skill inventory

The full skill list is already loaded in your context (from the
system-reminder's "The following skills are available" section). Use the
in-context list directly.

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

### 2b. MCP tool inventory

The deferred-tools list is also already loaded in your context (from the
system-reminder that says "The following deferred tools are now available
via ToolSearch"). Extract every name starting with `mcp__`.

**Steps:**

1. For each name `mcp__<server>__<tool>`, parse the **server** segment
   (between the first and second `__`) and the **tool** segment (everything
   after). E.g. `mcp__claude_ai_Signoz__search_logs` → server
   `claude_ai_Signoz`, tool `search_logs`.

2. Group MCP tools by server, then map each server to one or more domain
   buckets using this table:

| MCP server pattern | Bucket(s) |
|--------------------|-----------|
| `*Slack*` | communication |
| `*Gmail*`, `*Email*` | communication |
| `*Calendar*` | scheduling |
| `*Drive*`, `*Sheets*`, `*Docs*` | documentation, data |
| `*Signoz*`, `*Grafana*`, `*Datadog*` | observability, infrastructure |
| `*Jenkins*`, `*argocd*` | deployment, infrastructure |
| `*Atlassian*`, `*Jira*`, `*Linear*` | planning |
| `*HubSpot*`, `*Salesforce*` | crm |
| `*context7*` | documentation, planning |
| `*Supabase*`, `*Postgres*`, `*Mongo*` | database |
| `*Vercel*`, `*AWS*`, `*Azure*` | deployment, infrastructure |
| `*Meta_ADS*`, `*Google_Ads*` | marketing |
| `*Zapier*` | automation |

   A tool's effective domain set = its server's bucket(s) ∪ any keywords
   inferred from the tool-name suffix (e.g. `*_logs*` → observability,
   `*_alert*` → observability, `*_pr*` → deployment, `*_file*` →
   documentation).

3. Build a flat MCP candidate list: every tool whose effective domain set
   intersects the context profile's `Domain tags`.

### Inventory output

Print one combined line summarizing both counts:

```
Found N candidate skills and M candidate MCP tools in relevant buckets.
```

## Phase 3 — Relevance Scoring

**Goal:** Score every candidate Skill and every candidate MCP tool, then
decide what to recommend, suggest, or skip.

For each candidate, score 0–100 using this rubric:

| Criterion | Weight | How to evaluate |
|-----------|--------|-----------------|
| **Intent match** | 35% | Does the candidate's purpose directly match the context profile's `action intent`? Exact match = 35, close match = 20, weak match = 10, no match = 0 |
| **Domain match** | 30% | How many of the context profile's `domain tags` appear in this candidate's description or bucket? Each match adds ~10 points up to 30 |
| **Keyword overlap** | 20% | How many of the context profile's `keywords` appear (roughly) in the candidate's name or description? Each match adds ~4 points up to 20 |
| **Stack match** | 15% | Does the candidate explicitly target the detected language/framework? Match = 15, stack-agnostic = 10, mismatch = 0 |

### MCP-specific scoring notes

The same rubric applies to MCP tools with three small adjustments:

- **Intent match for MCP tools** is decided by the tool-name verb prefix.
  Map intents to verb sets:
  - `analyze` ↔ `get`, `list`, `search`, `query`, `read`, `aggregate`
  - `create` ↔ `create`, `send`, `draft`, `schedule`
  - `fix` / `investigate` ↔ `get_logs`, `get_*_status`, `trace`, `alert`
  - `document` ↔ `search_files`, `read_file_content`, `query-docs`
  - `deploy` ↔ `*_build_*`, `*_app_*` (Jenkins / argocd patterns)
- **Stack match** is mostly N/A for MCP tools — treat as stack-agnostic (10)
  unless the tool name explicitly references a stack.
- **High-risk MCP override** runs after numeric scoring (see below).

**Thresholds (same for both kinds):**
- **≥ 70** → RECOMMENDED (high confidence — included in the proposed plan)
- **40–69** → SUGGEST (borderline — presented to user for optional inclusion)
- **< 40** → Skip silently

**High-risk override:** If a Skill appears in the HIGH_RISK_SKILLS registry,
or an MCP tool matches the HIGH_RISK_MCP_PATTERNS registry, or either
matches its respective heuristic, force the tier to SUGGEST and mark it
`[HIGH-RISK]` in the table, regardless of numeric score.

**Constraints (independent caps):**
- **Max 5 RECOMMENDED skills** per invocation.
- **Max 5 RECOMMENDED MCP tools** per invocation.

If more than 5 of either kind score ≥70, take the top 5 by score within
that kind. The caps are independent so an observability-heavy investigation
(many high-scoring MCP tools) does not crowd out skill recommendations and
vice versa.

Print the scoring table (show only candidates scoring ≥ 30). The `Kind`
column distinguishes skills from MCP tools:

```
SKILL & MCP SCORING
─────────────────────────────────────────────────────────────────────────────
Kind   Name                                  Score  Tier                Reason
────── ──────────────────────────────────── ─────  ──────────────────  ─────────────────────
skill  security-review                       88     RECOMMENDED          intent=fix, domain=security, keyword=auth
skill  investigate                           82     RECOMMENDED          intent=fix, keyword=crash
mcp    Signoz__search_logs                   84     RECOMMENDED          observability + keyword=logs
mcp    Signoz__get_firing_alerts             76     RECOMMENDED          observability + fix intent
skill  typescript-reviewer                   75     RECOMMENDED          stack=typescript
mcp    Slack__slack_search_public            58     SUGGEST              weak domain match
skill  ship                                  71     SUGGEST [HIGH-RISK]  score≥70 but forced — deployment skill
mcp    Slack__slack_send_message             71     SUGGEST [HIGH-RISK]  write tool — forced to SUGGEST
mcp    Gmail__create_draft                   64     SUGGEST [HIGH-RISK]  write tool — forced to SUGGEST
seo                                          8      SKIP                 no frontend/content signals
─────────────────────────────────────────────────────────────────────────────
RECOMMENDED: N (X skill, Y mcp) | SUGGEST: M (K high-risk) | SKIP: K
```

## Phase 4 — Execution Preview and Mandatory Confirmation

**This phase ALWAYS runs before any skill or MCP tool is invoked.** There
are no exceptions. Even a single RECOMMENDED item requires explicit user
confirmation.

### Step 4a — Show the execution plan

Print the full proposed run as a preview. Do not invoke anything yet. The
`Kind` column distinguishes skills (`skill`) from MCP tools (`mcp`):

```
EXECUTION PLAN
─────────────────────────────────────────────────────────────────────
 #  Kind   Name                            Tier         Score  Why
──  ────── ──────────────────────────────  ───────────  ─────  ────────────────────────
 1  skill  security-review                 RECOMMENDED  88     fix intent + security domain
 2  skill  investigate                     RECOMMENDED  82     fix intent + keyword=crash
 3  mcp    Signoz__search_logs             RECOMMENDED  84     observability + keyword=logs
 4  mcp    Signoz__get_firing_alerts       RECOMMENDED  76     observability + fix intent
 5  skill  typescript-reviewer             RECOMMENDED  75     stack=typescript
 6  mcp    Slack__slack_send_message       HIGH-RISK    71     write tool — requires confirmation
─────────────────────────────────────────────────────────────────────
```

### Step 4b — Mandatory confirmation gate

Use **AskUserQuestion** for EVERY run. Do not skip this step.

Format the question as follows. Recommendations are grouped into two
sub-sections (Skills / MCP Tools) under each header so the user can scan
each kind separately:

> **autoskill recommends [N] items for: "[problem statement]"**
>
> These will run **only after you confirm** below:
>
> **Recommended skills (score ≥70):**
> - `skill-name` — [reason it applies]
> - `skill-name` — [reason it applies]
>
> **Recommended MCP tools (score ≥70):**
> - `mcp__server__tool` — [reason it applies]
> - `mcp__server__tool` — [reason it applies]
>
> **Also applicable — want any of these?**
> - `skill-name` [SUGGEST] — [reason it might apply]
> - `mcp__server__tool` [SUGGEST] — [reason it might apply]
> - `skill-name` [HIGH-RISK] — [why it needs confirmation]
> - `mcp__server__tool` [HIGH-RISK] — [why it needs confirmation]
>
> **Actions:**
> - Type the names of any suggested/high-risk items you want to add
> - Type "none" to run only the recommended items
> - Type "cancel" to stop and do nothing

Omit any sub-section that has no entries (e.g. if there are no MCP
recommendations, skip the "Recommended MCP tools" header entirely).

If the user replies "cancel" or selects nothing and there are no recommended
items, abort and report BLOCKED.

Add any user-selected items to the execution queue before continuing.

## Phase 5 — Execution (Skills + MCP Tools)

**Goal:** Apply each queued item in order.

**Execution order:**
1. RECOMMENDED **skills** sorted by score descending
2. RECOMMENDED **MCP tools** sorted by score descending
3. User-approved SUGGEST / HIGH-RISK items (skills and MCP tools) appended in selection order

Skills come before MCP tools because their multi-step output often gives
subsequent MCP calls better context (e.g. `investigate` may surface the
exact service name to pass to `Signoz__search_logs`).

**For each item in the queue:**

If `kind == skill`:

1. Print: `→ Applying \`[skill-name]\` (score: [N]) — [one-line reason]`
2. Invoke: `Skill(skill="[skill-name]", args="[relevant portion of the original problem statement]")`
3. Wait for the skill to complete before starting the next one.
4. Note the outcome (completed / blocked / needs-context).

If `kind == mcp`:

1. Print: `→ Loading schema for \`[mcp-tool-name]\` …`
2. Materialize the schema:
   `ToolSearch(query="select:[mcp-tool-name]", max_results=1)`
3. Print: `→ Invoking \`[mcp-tool-name]\` (score: [N]) — [one-line reason]`
4. Synthesize arguments from the context profile:
   - keywords → search terms / query fields
   - problem statement → query body / message body
   - intent + time tone (recent / today / this week) → time-window fields
5. If the schema requires fields autoskill cannot safely infer (e.g. a
   specific channel ID, a recipient email, an alert rule ID), fall back to
   **AskUserQuestion** to collect the missing parameter rather than
   guessing. Never invent IDs.
6. Invoke the MCP tool with the synthesized arguments. Wait for completion.
7. Note the outcome (completed / blocked / needs-context).

Execution is always **sequential** — no parallel MCP calls. Earlier output
(Signoz logs, Jenkins build status, Drive search results) may inform the
next call's arguments.

**If a queued item returns BLOCKED or NEEDS_CONTEXT:** note it in the audit
table and continue to the next item. Do not abort the entire queue for one
blocked item.

**If NO skills AND no MCP tools score ≥40:**

Do not silently do nothing. Instead use AskUserQuestion:

> No skills or MCP tools scored above the applicability threshold for: "[problem statement]"
>
> This usually means the request is best handled directly (not via a specialized skill or MCP tool),
> or the problem description needs more context.
>
> Options:
> A) Let me handle this directly without a skill or MCP tool
> B) Tell me more about what you need (I'll re-score)
> C) Show me all available skills and MCP tools so I can pick manually

## Phase 6 — Decision Audit Report

After all items have run (or been skipped), print the final report. The
`Kind` column distinguishes skills (`skill`) from MCP tools (`mcp`):

```markdown
## autoskill Run Complete

**Problem:** [problem statement]
**Intent:** [action intent] | **Stack:** [stack] | **Domains:** [domains]

| Kind  | Name                        | Score | Tier        | Applied | Outcome   | Reason |
|-------|-----------------------------|-------|-------------|---------|-----------|--------|
| skill | security-review             | 88    | RECOMMENDED | ✅      | completed | fix intent + security domain |
| skill | investigate                 | 82    | RECOMMENDED | ✅      | completed | fix intent + keyword=crash |
| mcp   | Signoz__search_logs         | 84    | RECOMMENDED | ✅      | completed | observability + keyword=logs |
| mcp   | Signoz__get_firing_alerts   | 76    | RECOMMENDED | ✅      | completed | observability + fix intent |
| mcp   | Slack__slack_send_message   | 71    | HIGH-RISK   | ⏸ User | skipped   | user declined |
| skill | database-reviewer           | 55    | SUGGEST     | ⏸ User | completed | user approved |
| skill | seo                         | 8     | SKIP        | ❌      | —         | no frontend signals |

**Summary:** [N] items applied ([X] skill, [Y] mcp), [M] skipped, [K] blocked.
```

## Completion Status Protocol

Report final status as one of:
- **DONE** — All queued skills completed successfully
- **DONE_WITH_CONCERNS** — Completed, but one or more skills returned BLOCKED or NEEDS_CONTEXT
- **BLOCKED** — Could not determine applicable skills, all skills failed, or user cancelled
- **NEEDS_CONTEXT** — Problem statement too vague to score skills reliably

## Design Constraints

- **Never bulk-read skill files.** The system-reminder list is sufficient for scoring. Only read a specific SKILL.md file if you need to understand invocation details for an edge case.
- **Never hardcode skill assumptions.** Always derive the candidate list from the live system-reminder. New skills added to the system are automatically included.
- **Mandatory confirmation for every run.** No skill or MCP tool is ever invoked without the user first seeing the execution plan and explicitly confirming or selecting which items to run. There are no single-skip exceptions.
- **High-risk items always require individual confirmation.** Skills and MCP tools that deploy, send messages, modify data, charge accounts, authenticate against external providers, or access broad shell/file scope are never automatically included — they are always presented as optional and marked `[HIGH-RISK]`.
- **Heuristic + registry for high-risk detection.** The fixed HIGH_RISK_SKILLS and HIGH_RISK_MCP_PATTERNS lists are each supplemented by a verb heuristic so newly added skills or tools with dangerous behavior are caught even if the registries have not been updated.
- **MCP tools are inventoried from the deferred-tools system-reminder only.** No discovery probes via `ToolSearch` during inventory or scoring — MCP schemas load on demand in Phase 5, just before invocation.
- **High-risk MCP gate.** Any MCP tool whose verb implies a write / send / auth side effect is forced to SUGGEST tier regardless of score, mirroring the skill registry behavior.
- **Show the plan before executing.** The execution preview in Phase 4 ensures the user always sees what will run before any skill or MCP tool is invoked.
- **Graceful degradation.** If the Skill tool or `ToolSearch` is unavailable, print the scored table and explain which items the user should invoke manually.
- **Independent caps.** Max 5 recommended skills AND max 5 recommended MCP tools per invocation. Prevents runaway chaining on broad problem statements and prevents one kind from crowding out the other.
- **Sequential execution.** Skills and MCP tools run one at a time, in queue order. Never parallel — earlier output may change project or remote state that later items depend on.

## Safety Notice

`autoskill` is a meta-skill that recommends and routes to other skills **and
MCP tools**. It does not itself perform file modifications, deployments, or
external communications. However, the items it recommends may do so.
Because of this:

1. **Always review the execution plan** before confirming.
2. **Remove any item you do not want** by selecting only the ones you trust.
3. **Be especially cautious with [HIGH-RISK] items** — they are marked for a reason.
4. **If you are unsure, choose "cancel"** — autoskill will stop and you can proceed manually.

**MCP tools deserve extra care.** Many MCP tools have external side effects:
sending Slack messages, creating Gmail drafts or labels, creating or
modifying calendar events, uploading to Drive, resetting rate limits,
authenticating against third-party providers, or mutating remote state in
Supabase / Vercel / HubSpot / Atlassian / etc. The same mandatory
confirmation gate applies — these tools are never auto-included and always
appear as `[HIGH-RISK]` suggestions, regardless of their numeric score.
