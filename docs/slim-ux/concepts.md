# Minions Concept Dictionary

> Plain-English definitions for every first-class noun in the Minions system. This is the team's
> shared language before we redesign the UI; it is intentionally factual — design opinions live in
> [`architecture-suggestions.md`](./architecture-suggestions.md).
>
> Every backing claim is anchored to a file path and line number. If you change the code, please
> update the line numbers here too.

---

## How the system fits together (one paragraph)

A **human** types something into the **Command Center** chat. CC turns that into a **Work Item**
(or a **Plan**, **Note**, **Schedule**, **Watch**, etc.) by emitting an `===ACTIONS===` JSON block.
The **Engine** wakes up on its 60-second tick, looks at the pending **Dispatch Queue**, picks an
**Agent** (a Minion / Team Member) by consulting **Routing**, spawns a CLI process for that agent
(Claude or Copilot), and waits. The agent reads its **Charter**, the **Playbook** for the work
type, the project's **CLAUDE.md**, the **Pinned Context**, the team's **Notes** and
**Knowledge Base**, then does the work — usually opening a **Pull Request**. When the agent exits,
it writes a **Completion** report; the engine syncs the PR back, marks the Work Item done, and
runs the **Inbox** through the **KB Sweep** so the team's memory grows. **Watches**, **Schedules**,
and **Pipelines** are the three ways things happen without a human typing.

---

## 1. Command Center (CC)

**What it is.** The chat box on the dashboard's home page. It's a Claude (or Copilot) instance
running with a beefy system prompt that knows about your projects, agents, work items, and the
JSON action grammar. You talk to CC; CC dispatches Work Items, creates Plans, files Notes, opens
Watches, etc. It is the primary human-to-fleet interface.

**Where it lives.**
- System prompt: `prompts/cc-system.md`
- Per-tab session state: `engine/cc-sessions.json` (also legacy `engine/cc-session.json`).
- Direct CLI invocation: `engine/llm.js` `callLLM({ direct: true })` — bypasses
  `spawn-agent.js` and runs the runtime CLI itself.

**Backing classes / functions.**
- Streaming endpoint handler: `dashboard.js` line 7194 (`/api/command-center/stream` → `handleCommandCenterStream`).
- Non-streaming handler: `dashboard.js` line 7193 (`/api/command-center` → `handleCommandCenter`).
- Action parser: `parseCCActions` / `executeCCActions` (search `dashboard.js` for `parseCCActions`).
- Tab session bookkeeping: dashboard.js around line 1185 (`/api/cc-sessions/:id`).

**How the existing dashboard surfaces it.** `dashboard/pages/home.html` — the Command Center is
the top-of-page input box. There is also a "past commands" / history modal.

**Endpoints (`dashboard.js` line refs from the `ROUTES` table starting at line 6807).**

| Method | Path | Line | Purpose |
|--------|------|------|---------|
| POST | `/api/command-center` | 7193 | Single-shot CC turn |
| POST | `/api/command-center/stream` | 7194 | SSE-streaming CC turn |
| POST | `/api/command-center/new-session` | 7191 | Clear active CC session |
| POST | `/api/command-center/abort` | 7192 | Cancel an in-flight stream |
| GET  | `/api/cc-sessions` | 7195 | List per-tab sessions |
| DELETE | `/api/cc-sessions/:id` | (see line 1185) | Close a tab |

**Relationships.** CC is the *creator* of almost every other concept on this page (Work Item,
Plan, Note, Schedule, Watch, Knowledge entry, Pinned note). Per-call cap is
`ENGINE_DEFAULTS.ccMaxTurns` (50, `engine/shared.js` line 860+); sessions themselves never expire.

---

## 2. Work Item

**What it is.** A unit of work the engine can dispatch to an agent. Has a title, type
(`implement`, `fix`, `review`, …), priority, optional project, and lifecycle state. Most things on
the home page eventually reduce to a Work Item.

**Where it lives.**
- Central queue: `work-items.json` (root).
- Per-project queue: `projects/<name>/work-items.json` (`projectWorkItemsPath`,
  `engine/shared.js` line ~1530).
- In-flight copy: `engine/dispatch.json` (`pending[] / active[] / completed[]`).

**Backing classes / functions.**
- Status constants: `engine/shared.js` line 1348 (`WI_STATUS`).
- Type constants: `engine/shared.js` line 1359 (`WORK_TYPE`).
- `addToDispatch`: `engine/dispatch.js` line 128.
- `updateWorkItemStatus`: `engine/lifecycle.js` line 484 (validates against `WI_STATUS`).
- `isItemCompleted`: `engine/lifecycle.js` line 477.
- Lifecycle: pending → dispatched → done | failed; failed auto-retries up to
  `ENGINE_DEFAULTS.maxRetries`.

**Existing dashboard.** `dashboard/pages/work.html` — the side-panel "Work Items" tab with
pending / active / completed columns.

**Endpoints (all under `/api/work-items*`, `dashboard.js` lines 6852–6888).**
- `POST /api/work-items` (6852) — create
- `POST /api/work-items/update` (6853) — edit
- `POST /api/work-items/retry` (6854) — back to pending
- `POST /api/work-items/delete` (6855) — kill + remove
- `POST /api/work-items/cancel` (6856) — cancel with reason
- `POST /api/work-items/archive` (6857) — move to archive
- `GET  /api/work-items/archive` (6858) — list archive
- `POST /api/work-items/reopen` (6859) — done/failed → pending
- `POST /api/work-items/feedback` (6860) — 👍 / 👎 → inbox note

**Relationships.** A Work Item is the *body* the **Dispatch Queue** carries. It points to a
**Project**, an **Agent** (post-dispatch), and a **Pull Request** (post-completion). Materialized
from a **PRD Item** via the plan-to-prd flow. **Pipelines** can declare Work Items as stages.

---

## 3. Plan

**What it is.** A markdown document in `plans/` describing what should be built and why. The plan
flow is: human (or agent) writes plan → human approves → `plan-to-prd` agent converts it to a
**PRD** JSON → materializer creates **Work Items** with `depends_on` edges → engine executes them
→ verify task auto-runs → human archives.

**Where it lives.**
- Drafts: `plans/*.md`.
- Archived: `plans/archive/*.md`.
- Status field is on the PRD JSON twin (see PRD).

**Backing classes / functions.**
- Status constants: `engine/shared.js` line 1365 (`PLAN_STATUS = { ACTIVE, AWAITING_APPROVAL,
  APPROVED, PAUSED, REJECTED, COMPLETED, REVISION_REQUESTED }`).
- Atomic dispatch: `queuePlanToPrd` (used by every plan-to-prd path).
- Plan completion / archival: `engine/lifecycle.js`
  - `checkPlanCompletion` line 22
  - `archivePlan` line 355
  - `cleanupPlanWorktrees` line 415.
- Plan dirs: `engine/pipeline.js` line 23 (`PLANS_DIR`).

**Existing dashboard.** `dashboard/pages/plans.html` — plan browser, approve / reject / archive
buttons.

**Endpoints (`dashboard.js` lines 6952–6963).**
- `GET  /api/plans` (6952) — list .md drafts + .json PRDs
- `POST /api/plans/trigger-verify` (6953)
- `POST /api/plans/approve` (6954) — kicks off plan-to-prd
- `POST /api/plans/pause` (6955)
- `POST /api/plans/execute` (6956)
- `POST /api/plans/reject` (6957)
- `POST /api/plans/regenerate` (6958)
- `POST /api/plans/delete` (6959)
- `POST /api/plans/archive` (6960) — manual archive
- `POST /api/plans/unarchive` (6961)
- `POST /api/plans/revise` (6962)
- `POST /api/plans/discuss` (6963)
- `POST /api/plans/create` (7055)

**Relationships.** Plan ↔ **PRD** ↔ **Work Items** is a one-to-one-to-many chain. **Pipelines**
can include `plan` stages. **Verification** is owned by the plan: when all work items finish, a
verify Work Item auto-spawns.

---

## 4. PRD (Product Requirements Document)

**What it is.** The structured JSON twin of a Plan. Where the Plan markdown is for humans, the
PRD JSON is what the engine reads to materialize work items. Each PRD has an `items` array; each
item has acceptance criteria, complexity, dependencies, and a status (`missing | updated | done`).

**Where it lives.** `prd/*.json`, archived to `prd/archive/*.json`. The materializer treats
`missing` and `updated` items as work to do (`PRD_MATERIALIZABLE`, `engine/shared.js` line 1370).

**Backing classes / functions.**
- Status constants: `engine/shared.js` line 1370 (`PRD_ITEM_STATUS`, `PRD_MATERIALIZABLE`).
- Sync from Work Items: `syncPrdItemStatus`, `reconcilePrdStatuses` (`engine/lifecycle.js`
  lines 564, 597).
- Plan-to-PRD agent: `playbooks/plan-to-prd.md`, dispatched via `queuePlanToPrd`.
- PRD dir: `engine/pipeline.js` line 24 (`PRD_DIR`).

**Existing dashboard.** `dashboard/pages/plans.html` — same page as Plan, with a graph view of
PRD items and their states. Recent feature: PRD graph view at parity with list view (commit
`b4767d2d`).

**Endpoints (`dashboard.js`).**
- `POST /api/prd-items` (6968) — create PRD item
- `POST /api/prd-items/update` (6969)
- `POST /api/prd-items/remove` (6970) — also cancels matching Work Item

**Relationships.** PRD is the bridge between **Plan** (human-readable) and the **Work Item**
queue (machine-executable). When a plan's PRD items are all `done`, the plan completes; archive
is still manual.

---

## 5. Note

**What it is.** A small markdown file written by an agent (after a task) or a human (via "+
Note") capturing what was learned. Notes are *raw*; the consolidation engine reads them and
either merges them into `notes.md` or promotes them into a **Knowledge Base** category.

**Where it lives.** `notes/inbox/*.md` while raw, `notes/archive/<category>/*.md` after KB sweep,
and merged content lives in `notes.md`.

**Backing classes / functions.**
- Inbox writer: `writeToInbox` — `engine/shared.js` line 648 (re-exported line 2959).
- ID parser: `parseNoteId` (same module).
- Consolidation orchestration: `engine/consolidation.js`
  - `consolidateInbox` line 24
  - `consolidateWithLLM` line 116
  - `classifyToKnowledgeBase` line 389
  - `archiveInboxFiles` line 444.

**Existing dashboard.** `dashboard/pages/inbox.html` — the inbox tab. A "+ Note" quick-action
exists in the top bar of `home.html`.

**Endpoints (`dashboard.js`).**
- `POST /api/notes` (6946) — write to inbox
- `GET  /api/notes-full` (6947) — read consolidated `notes.md`
- `POST /api/notes-save` (6948) — save edited `notes.md`

**Relationships.** Notes feed **Inbox** → **Consolidation** → **Knowledge Base** (or
`notes.md`). Agents are *required* to drop a note in the inbox at the end of every successful
task; failure cases must NOT write a note.

---

## 6. Knowledge Base (KB)

**What it is.** The team's long-term memory. After consolidation, notes get classified into one of
five categories and archived. Agents read the KB before researching outside.

**Where it lives.**
- `notes/archive/architecture/*.md`
- `notes/archive/conventions/*.md`
- `notes/archive/project-notes/*.md`
- `notes/archive/build-reports/*.md`
- `notes/archive/reviews/*.md`

**Backing classes / functions.**
- Categories: `engine/shared.js` line 840 (`KB_CATEGORIES`).
- Pin store: `engine/shared.js` line 15 (`PINNED_ITEMS_PATH = engine/kb-pins.json`).
- Sweep / classification: `engine/kb-sweep.js`, `engine/consolidation.js`
  `classifyToKnowledgeBase` line 389.

**Existing dashboard.** `dashboard/pages/inbox.html` — same tab as Notes. KB browser by category.

**Endpoints (`dashboard.js`).**
- `GET  /api/knowledge` (7146) — list KB grouped by category
- `POST /api/knowledge` (7147) — create entry directly
- `POST /api/knowledge/sweep` (7163) — async sweep
- `GET  /api/knowledge/sweep/status` (7164) — poll sweep

**Relationships.** Inbox → KB → agent prompt context. Pinned notes (next entry) are a *subset* of
the KB.

---

## 7. Pinned Context (`pinned.md`)

**What it is.** Critical context the human flagged as "READ FIRST." Prepended to every agent
prompt regardless of work type. Use sparingly — pinned notes pollute every dispatch.

**Where it lives.** `pinned.md` (root). Index file: `engine/kb-pins.json` (`PINNED_ITEMS_PATH`,
`engine/shared.js` line 15).

**Backing classes / functions.**
- Add / remove handlers: `dashboard.js` lines 6891, 6895, 6908.
- Parser: `parsePinnedEntries` (referenced at line 6893).
- Slow-state cache invalidation explicitly on edits (`includeSlow: true`, line 6905).

**Existing dashboard.** `dashboard/pages/home.html` — "Pinned Context" widget. Pin / unpin
buttons appear in the KB browser.

**Endpoints (`dashboard.js`).**
- `GET  /api/pinned` (6891)
- `POST /api/pinned` (6895) — add
- `POST /api/pinned/remove` (6908)

**Relationships.** Pinned ⊂ KB. Pinned content shows up in *every* agent prompt (even short
ones), so it's also coupled to **Charter** and **Routing** in terms of "what gets read first."

---

## 8. Notes Inbox

**What it is.** The waiting room for raw notes before consolidation. Files dropped into
`notes/inbox/` are the engine's signal that an agent succeeded; they get consolidated into
`notes.md` or promoted into the **KB** on a schedule.

**Where it lives.** `notes/inbox/*.md` (filename convention:
`<agent>-<work-item-id>-<date>-<time>.md`, with YAML frontmatter `id:`, `agent:`, `date:`).

**Backing classes / functions.**
- Inbox dir constant: `INBOX_DIR` in `engine/queries.js` (used at line 526).
- Consolidation tick path: `engine/consolidation.js` `consolidateInbox` line 24.
- Promote / delete API handlers: `handleInboxPersist`, `handleInboxPromoteKb`,
  `handleInboxOpen`, `handleInboxDelete`.

**Existing dashboard.** `dashboard/pages/inbox.html` — note list, promote-to-KB buttons,
consolidate-now button.

**Endpoints (`dashboard.js`).**
- `POST /api/inbox/persist` (7172) — promote to `notes.md`
- `POST /api/inbox/promote-kb` (7173) — promote to KB category
- `POST /api/inbox/open` (7174) — reveal in OS file manager
- `POST /api/inbox/delete` (7175)

**Relationships.** Inbox is the staging area between **Notes** (the act of writing) and the
**KB** / `notes.md` (the durable store). Failure inboxes are explicitly forbidden by team rules.

---

## 9. Schedule (cron)

**What it is.** A recurring trigger that creates a Work Item on a cron pattern. The engine ticks
every 60 s; if a schedule's last-run was ≥ its interval ago and the cron pattern matches, the
engine drops a Work Item into the queue.

**Where it lives.**
- Definitions: `config.json` `schedules: []` (3-field cron `min hour dow`).
- Last-run history: `engine/schedule-runs.json`.

**Backing classes / functions.**
- Cron parser: `engine/scheduler.js`
  - `parseCronField` line 63
  - `parseCronExpr` line 106
  - `shouldRunNow` line 145.
- Variable substitution: `resolveScheduleTemplateVars` line 46 (handles `{{date}}`, etc., before
  the work item lands in the queue).
- Work-item factory: `createScheduledWorkItem` line 168.
- Tick path: `discoverScheduledWork` line 209.

**Existing dashboard.** `dashboard/pages/schedule.html` — schedule list, natural-language → cron
helper.

**Endpoints (`dashboard.js`).**
- `GET  /api/schedules` (7200)
- `POST /api/schedules` (7201) — create
- `POST /api/schedules/update` (7202)
- `POST /api/schedules/delete` (7203)
- `POST /api/schedules/run-now` (7204)
- `POST /api/schedules/parse-natural` (7199) — natural-language → cron

**Relationships.** Schedule → emits Work Items into the **Dispatch Queue** on a cron. Closely
analogous to **Watch** (event-driven instead of time-driven).

---

## 10. Watch

**What it is.** A persistent monitor on a PR or Work Item that fires when a condition is met
(merged, build passes, build fails, status changes, new comments, vote changes). Watches are
event-driven, schedules are time-driven.

**Where it lives.** `engine/watches.json` (`engine/watches.js` line 17 `_watchesPath`).

**Backing classes / functions.**
- Status constants: `engine/shared.js` line 1380 (`WATCH_STATUS`).
- Condition constants: `WATCH_CONDITION` (same file, near 1380); fire-once set
  `WATCH_ABSOLUTE_CONDITIONS`.
- CRUD: `engine/watches.js` `createWatch` line 44, `updateWatch` line 88, `deleteWatch`
  line 116.
- Tick path: `evaluateWatch` line 138, `checkWatches` line 214 (runs every 3 ticks).
- State diff store: `_captureState` line 297 (`_lastState` per watch).

**Existing dashboard.** `dashboard/pages/watches.html` — watch list, condition picker.

**Endpoints (`dashboard.js`).**
- `GET  /api/watches` (7207)
- `POST /api/watches` (7208) — create
- `POST /api/watches/update` (7209) — pause / resume / modify
- `POST /api/watches/delete` (7210)

**Relationships.** Watch ⇒ optionally re-fires (notify, dispatch, etc.) when its condition flips.
A merged-PR watch is the canonical "tell me when X ships" primitive.

---

## 11. Pipeline

**What it is.** A multi-stage workflow. Stages can be a Work Item (`task`), a `meeting`, or a
`plan`. Stages can declare dependencies on previous stages. Pipelines run via triggers (manual,
schedule, watch) and persist their state in `pipeline-runs.json`.

**Where it lives.**
- Definitions: `pipelines/*.json` (one file per pipeline).
- Run state: `engine/pipeline-runs.json`.

**Backing classes / functions.**
- Dirs: `engine/pipeline.js` lines 19–24 (`PIPELINES_DIR`, `PIPELINE_RUNS_PATH`, `PLANS_DIR`,
  `PRD_DIR`).
- CRUD: `getPipelines` line 35, `getPipeline` line 53, `savePipeline` line 58, `deletePipeline`
  line 63.
- Run lifecycle: `getActiveRun` line 80, `startRun` line 86, `updateRunStage` line 116,
  `completeRun` line 127.

**Existing dashboard.** `dashboard/pages/pipelines.html` — pipeline editor, run history, stage
state.

**Endpoints (`dashboard.js`).**
- `GET  /api/pipelines` (7213)
- `POST /api/pipelines` (7220) — create
- `POST /api/pipelines/update` (7232)
- `POST /api/pipelines/delete` (7248)
- `POST /api/pipelines/trigger` (7256)
- `POST /api/pipelines/continue` (7268) — past wait stage
- `POST /api/pipelines/abort` (7280)
- `POST /api/pipelines/retrigger` (7307)

**Relationships.** Pipeline = composer of Work Items / Meetings / Plans. Confusion vector:
unrelated to ADO/GitHub Actions pipelines, even though the word collides.

---

## 12. Dispatch

**What it is.** The act of taking a pending Work Item and spawning an agent process for it. A
dispatch holds the agent's PID, prompt sidecar path, worktree path, and current status until the
process exits.

**Where it lives.** Inside `engine/dispatch.json`'s `active[]` array.

**Backing classes / functions.**
- `mutateDispatch`: `engine/dispatch.js` line 60 (the only safe RMW path).
- `addToDispatch`: line 128.
- `findActivePrOrBranchLock`: line 112 (dedupe).
- `completeDispatch`: line 327.
- `cancelPendingDispatchesForPr`: line 550.
- Result codes: `engine/shared.js` line 1433 (`DISPATCH_RESULT`).

**Existing dashboard.** `dashboard/pages/engine.html` — dispatch log; `home.html` — "Dispatch
Queue" widget.

**Endpoints.** No public CRUD — dispatch is a side effect of `/api/work-items` and the engine
tick. Read shape via `/api/status` (line 6838).

**Relationships.** Dispatch wraps a **Work Item** + an **Agent** + a **Project** worktree. A
**Completion** is dispatch's exit signal.

---

## 13. Dispatch Queue

**What it is.** The three-bucket structure inside `engine/dispatch.json`:

- `pending[]` — admitted Work Items waiting on agent / dependency / cooldown / budget.
- `active[]` — currently spawned.
- `completed[]` — finished (success or failure), kept for audit until cleaned.

The queue's invariants are: max concurrent (default 5), per-PR / per-branch lock to avoid two
agents fighting on the same branch, dependency-aware spawning (`depends_on` IDs must be done
first), retry on failure up to `ENGINE_DEFAULTS.maxRetries`.

**Where it lives.** `engine/dispatch.json`.

**Backing classes / functions.** All in `engine/dispatch.js`. Mutation only via `mutateDispatch`
line 60. Queue dedupe keys: `getDispatchProjectKey` line 80, `getPrDispatchTargetKey` line 85,
`getPrDispatchDedupeKey` line 95, `getBranchDispatchLockKey` line 103.

**Existing dashboard.** `dashboard/pages/home.html` ("Dispatch Queue" section) and
`engine.html` (full log).

**Endpoints.** Read-only via `/api/status` (line 6838).

**Relationships.** Queue ⊃ Dispatches ⊃ Work Items. The queue is the engine's state of mind.

---

## 14. Completion

**What it is.** The structured report an agent writes before exit. JSON shape lives at the path
the engine puts in `MINIONS_COMPLETION_REPORT`. Fields: `status`, `summary`, `verdict`, `pr`,
`failure_class`, `retryable`, `needs_rerun`, `artifacts[]`. Fenced ` ```completion ` blocks in
stdout are also accepted as a fallback.

**Where it lives.**
- File path: `engine/completions/<dispatchId>.json` (built by `dispatchCompletionReportPath`,
  `engine/shared.js` line 351).
- Field constants: `COMPLETION_FIELDS` (`engine/shared.js` line 1474–1475).

**Backing classes / functions.** Parsing happens during `completeDispatch` (`engine/dispatch.js`
line 327) and `engine/lifecycle.js`'s post-completion path (`syncPrsFromOutput` line 644).

**Existing dashboard.** `dashboard/pages/work.html` — completion details when an item is
expanded; `home.html` "Recent Completions" widget.

**Endpoints.** Read-only via `/api/status` and `/api/work-items/archive`.

**Relationships.** Completion sits between **Dispatch** (the run) and **Work Item** (the long-term
record). It also feeds **Pull Request** sync (`pr` field).

---

## 15. Pull Request (as tracked in minions)

**What it is.** A row in `projects/<name>/pull-requests.json`. Tracks status, vote tally, build
state, review verdict, and the work item that created it. *Not* the GitHub/ADO PR itself —
minions polls the host and writes a sanitized snapshot here.

**Where it lives.** `projects/<name>/pull-requests.json` (`projectPrPath`, `engine/shared.js`
line 1546). Cross-project links also tracked in `engine/pr-links.json`.

**Backing classes / functions.**
- Status / pollable constants: `engine/shared.js` line 1372 (`PR_STATUS`,
  `PR_POLLABLE_STATUSES`).
- GitHub poller: `engine/github.js`.
- ADO poller: `engine/ado.js`.
- Parallel comment poller for human comments only (filters bots).
- Sync from agent output: `syncPrsFromOutput` (`engine/lifecycle.js` line 644).
- PR-attachment requirement check: `isPrAttachmentRequired` (line 875).

**Existing dashboard.** `dashboard/pages/prs.html` — the "Pull Requests" tab.

**Endpoints (`dashboard.js`).**
- `POST /api/pull-requests/link` (6974) — manually link external PR
- `POST /api/pull-requests/delete` (7030)

**Relationships.** PR ↔ Work Item is one-to-one in normal flow (review / fix work items can
re-attach to the same PR). PR ↔ Watch is one-to-many (you can watch the same PR with multiple
conditions).

---

## 16. Agent / Minion / Team Member

**What it is.** A named role with a charter, a skill list, a default CLI runtime, and metrics.
Examples: Ripley (architect / explorer), Dallas (engineer), Lambert (analyst), Rebecca
(architect), Ralph (engineer). Sometimes called "Minions" (the project's name), "Team Members"
(in the dashboard), or just "Agents" (in the code).

**Where it lives.**
- Default roster: `engine/shared.js` line 1486 (`DEFAULT_AGENTS`).
- Charter: `agents/<id>/charter.md` — read by `getAgentCharter` (`engine/queries.js` line 505).
- Per-agent config (cli, model, budget, skills): `config.json` `agents.<id>`.
- Metrics: `engine/metrics.json` (per-agent token / cost / quality / runtime).

**Backing classes / functions.**
- Charter loader: `engine/queries.js` line 505.
- Roster builder: `getAgents` line 509.
- Spawn entry: `engine/spawn-agent.js`.
- Charter injection into prompt: `engine/playbook.js` line 517.

**Existing dashboard.** `dashboard/pages/home.html` — "Minions Members" / agent cards.

**Endpoints.**
- `POST /api/agent-kill/:id` (line 3847)
- `GET  /api/agent/:id/live-stream` (SSE — search for `/live-stream`)
- `GET  /api/agent-detail/:id` (line 6686)

**Relationships.** Agent ↔ Charter (1:1), Agent ↔ Skill (1:N), Agent ↔ runtime (Claude or
Copilot via `resolveAgentCli`), Agent ↔ Work Item (N:N over history). Routing decides which agent
gets which work type.

---

## 17. Project

**What it is.** A repository the engine knows about. Has a name, local path, repo host
(`github` or `ado`), repository ID, main branch, and per-source toggles (`workSources`).

**Where it lives.**
- Definitions: `config.json` `projects: []`.
- State: `projects/<name>/{work-items.json, pull-requests.json}`.
- Removal artifact: `projects/.archived/<name>-YYYYMMDD/`.

**Backing classes / functions.**
- Helpers: `engine/shared.js`
  - `getProjects` (~line 1502)
  - `projectStateDir`, `projectWorkItemsPath` (~line 1530)
  - `projectPrPath` (line 1546).
- Removal: `engine/projects.js` `removeProject` (canonical teardown — never edit `config.json`
  directly).

**Existing dashboard.** `dashboard/pages/settings.html` — Projects section.

**Endpoints (`dashboard.js`).**
- `POST /api/projects/browse` (7181)
- `POST /api/projects/scan` (7182)
- `POST /api/projects/confirm-token` (7183) — SEC-05 single-use token
- `POST /api/projects/add` (7184)
- `POST /api/projects/remove` (7185)

**Relationships.** Project ⊃ Work Items, Pull Requests, Schedules (via `project` field), and
Pipelines that target it. The minions repo is special: it's the home of the engine itself.

---

## 18. Skill

**What it is.** A reusable prompt / workflow snippet under `.claude/skills/<name>/SKILL.md`.
Skills can be `scope: minions` (installed personally to the runtime's user dir) or
`scope: project` (PR'd into the project repo). Agents auto-extract skills from their output via
` ```skill ` fenced blocks.

**Where it lives.**
- User skills: `~/.claude/skills/`, `~/.copilot/skills/`, etc. (runtime-specific).
- Project skills: `<project>/.claude/skills/`, `<project>/.github/skills/`.
- Frontmatter parser: `parseSkillFrontmatter` (`engine/shared.js`, exported line 14 of
  `engine/queries.js`).

**Backing classes / functions.**
- Discovery: `engine/queries.js` `collectSkillFiles` line 789, `getSkills` line 891,
  `getSkillIndex` line 916.
- Per-runtime roots: adapter `getSkillRoots()` / `getSkillWriteTargets()` (engine/runtimes/).

**Existing dashboard.** `dashboard/pages/tools.html` — skill browser.

**Endpoints.**
- `GET /api/skill` (line 7178) — read a single skill file.

**Relationships.** Skill ↔ Runtime adapter (storage paths differ). Skills ↔ Agent prompts
(skills can be invoked mid-task).

---

## 19. MCP Server

**What it is.** A Model Context Protocol server the runtime CLI can call. Examples: GitHub MCP,
Azure DevOps MCP, custom tool servers. Configured per-fleet and per-runtime; agents see them as
extra tools.

**Where it lives.**
- Configuration: `config.json` `mcpServers: { … }`.
- Aggregator: `getMcpServers` in `dashboard.js` line 759 (also re-reads runtime-specific
  configs at lines 750–787, e.g. `~/.copilot/mcp-config.json`, `~/.claude/mcp-config.json`).

**Backing classes / functions.**
- Loader: `dashboard.js` line 759 (`getMcpServers`).
- Engine status integration: `getStatusJson` includes `mcpServers` (line 905).
- Per-runtime adapter knobs: `engine.copilotDisableBuiltinMcps` (default true) strips the
  built-in `github-mcp-server` because it would mutate state behind the engine's back.

**Existing dashboard.** `dashboard/pages/tools.html` — MCP server list with status.

**Endpoints.** Read-only via `/api/status`. There is no dedicated MCP CRUD endpoint —
configuration is via `/api/settings` (config.json edits).

**Relationships.** MCP ↔ Runtime (Copilot has built-in MCPs that we explicitly disable). MCP ↔
Skill is unrelated; skills are prompt-level, MCPs are tool-level.

---

## 20. Engine

**What it is.** The orchestrator daemon. One Node process; runs a 60 s tick that drains the
**Dispatch Queue**, polls **PRs**, evaluates **Watches** every 3 ticks, runs **Schedules**,
consolidates the **Inbox**, and so on. It's also the gatekeeper for control state
(`paused | running | stopping`).

**Where it lives.**
- Process: `engine.js` (root).
- State: `engine/control.json` (mode), `engine/log.json` (audit ring buffer, max 2500 trimmed
  to 2000), `engine/metrics.json` (counters), `engine/dispatch.json` (queue).

**Backing classes / functions.**
- Tick: `engine.js` (top-level `tick()`).
- Defaults: `engine/shared.js` line 860 (`ENGINE_DEFAULTS`).
- Concurrency caps: `mutateJsonFileLocked` for all shared-JSON RMW.
- Restart re-attach: PID files in `engine/tmp/pid-<id>.pid` + `live-output.log` mtimes,
  20-min grace.

**Existing dashboard.** `dashboard/pages/engine.html` — engine timeline, log, controls.

**Endpoints.**
- `POST /api/engine/wakeup` (7406) — set `_wakeupAt` so the tick fires now.
- `POST /api/engine/restart` (7410) — graceful restart.
- `GET  /api/status` (6838), `/api/health` (6844), `/api/status-stream` (6839, SSE).

**Relationships.** Engine = the verb. Every other concept on this page is the engine's data.

---

## 21. Settings

**What it is.** `config.json`. Holds projects, agents, engine defaults, schedules, features, and
runtime knobs. Merged with `.claude/settings.local.json` for user-only overrides.

**Where it lives.** `config.json` (root); `.claude/settings.local.json` (gitignored).

**Backing classes / functions.**
- Loader: `engine/queries.js` `getConfig` (around line 86–166, includes migrations like
  `applyLegacyCcModelMigration`).
- Defaults: `engine/shared.js` line 860 (`ENGINE_DEFAULTS`).
- Routing knob: `routing.md` (separate file).

**Existing dashboard.** `dashboard/pages/settings.html` — multi-tab editor for engine, projects,
agents, runtimes.

**Endpoints (`dashboard.js`).**
- Settings reads / writes are via the `/api/settings*` family (search for them in the ROUTES
  table around 6249–6599 in earlier dashboard.js layouts; the agent-driven version uses the
  ROUTES registry).
- `POST /api/settings/reset` — restore defaults.
- `POST /api/settings/routing` — read `routing.md`.

**Relationships.** Settings is the *write target* for almost every config-changing action in CC
and the dashboard. Two locations to know about: per-fleet (`engine.*`) and per-agent
(`agents.<id>.*`). They don't fall through to each other for runtime selection.

---

## 22. Feature Flag

**What it is.** A temporary gate registered in `engine/features.js`. Resolved per request:
env var (`MINIONS_FEATURE_<UPPER_SNAKE>`) → `config.features[id]` → registry default. Throws if
asked for an unknown id. The slim UX itself is gated behind `slim-ux`.

**Where it lives.** `engine/features.js` lines 1–62 (registry + `isFeatureOn`).

**Backing classes / functions.**
- Registry: `FEATURES` (top of `engine/features.js`).
- Resolver: `isFeatureOn(id, config)`.
- Env override: any `MINIONS_FEATURE_*` translates id `slim-ux` ↔ `MINIONS_FEATURE_SLIM_UX`.

**Existing dashboard.** `dashboard/pages/settings.html` — "Show experimental flags" disclosure.
Slim UX also has its own minimal flags-only settings dialog (`dashboard/slim.html` line ~445).

**Endpoints (`dashboard.js`).**
- `GET  /api/features` (7457)
- `POST /api/features/toggle` (7458)

**Relationships.** Feature flag ↔ everything that gets ramped behind one. When the feature
ships, *delete the registry entry and the gate* — stale entries past `expires` surface in
preflight.

---

## 23. Routing (`routing.md`)

**What it is.** A markdown table mapping work types to agents (e.g. `implement → dallas`,
`review → ripley`, `fix → _author_`). The engine parses it on every tick.

**Where it lives.** `routing.md` (root).

**Backing classes / functions.**
- Parser: `engine/routing.js` `parseRoutingTable` line 31.
- Path constant: line 15.
- Cache: line 59 (`_routingCache`).
- Re-export: line 301.

**Existing dashboard.** `dashboard/pages/settings.html` — Routing panel (read-only with an edit
modal).

**Endpoints.**
- `POST /api/settings/routing` — read `routing.md`.

**Relationships.** Routing decides which **Agent** receives which **Work Type**. The CC action
parser pre-flights routing to surface "no agent available" warnings before enqueue. Per-project
overrides are possible via `projects/<name>/routing.md`.

---

## 24. Charter (Agent Charter)

**What it is.** A short markdown file describing an agent's role, voice, and rules of engagement.
Loaded into the agent's system prompt at every dispatch.

**Where it lives.** `agents/<id>/charter.md`. Examples: `agents/dallas/charter.md`,
`agents/ripley/charter.md`, …

**Backing classes / functions.**
- Loader: `engine/queries.js` `getAgentCharter` line 505.
- Injection into prompt: `engine/playbook.js` line 517 (`prompt += '## Your Charter\n\n' +
  charter`).

**Existing dashboard.** Agent detail modal on `home.html` shows the charter.

**Endpoints.** Read-only via `/api/agent-detail/:id` (line 6686).

**Relationships.** Charter ↔ Agent (1:1). Distinct from Skills (skills are reusable work
recipes; charter is identity).

---

## 25. Meeting

**What it is.** A multi-agent debate workflow. Three rounds: investigate → debate → conclude.
Participants are explicitly listed; each round writes a structured findings file before the
engine advances. The engine times out / advances rounds automatically.

**Where it lives.** `meetings/*.json` (definitions + transcripts). Per-agent artifact notes
land in `notes/inbox/`.

**Backing classes / functions.**
- Dirs / status: `engine/meeting.js`
  - lines 21–24 (`MEETINGS_DIR`, `MEETING_NOTE_ARTIFACT_ROOT`, `TERMINAL_MEETING_STATUSES`,
    `ROUND_STATUS_BY_NAME`).
  - line 29 (`ROUND_NUMBER_BY_NAME`).
- Round bookkeeping: `roundKeyFor` line 40, `getRoundFailures` line 46, `hasRoundFailure`
  line 59, `hasRoundSuccess` line 63, `hasRoundTerminalOutcome` line 69,
  `allParticipantsFinishedRound` line 73.
- Advance / fail handling: `advanceMeetingIfRoundComplete` line 93,
  `buildFailedMeetingConclusion` line 87.
- Empty-output detection: `EMPTY_OUTPUT_PATTERNS` line 15, `isEmptyMeetingContent` line 124.

**Existing dashboard.** `dashboard/pages/meetings.html` — meeting browser, round status, notes.

**Endpoints (`dashboard.js`).**
- `POST /api/meetings` (7322) — create
- `GET  /api/meetings` (7337)
- `POST /api/meetings/note` (7349)
- `POST /api/meetings/advance` (7359)
- `POST /api/meetings/end` (7369)
- `POST /api/meetings/archive` (7378)
- `POST /api/meetings/unarchive` (7387)
- `POST /api/meetings/delete` (7396)

**Relationships.** Meeting → emits Notes into the inbox. Each meeting round dispatches one Work
Item per participant. Pipelines may include `meeting` stages.

---

## Cross-cutting relationship cheatsheet

```
Human ──► Command Center ──► [Work Item | Plan | Note | Schedule | Watch | Pinned | Knowledge]
                              │
                              └─► Dispatch Queue ──► Dispatch ──► Agent (CLI)
                                                                   │
                                                                   ├─► Pull Request (synced from output)
                                                                   ├─► Completion (JSON report)
                                                                   └─► Notes Inbox  ──► Consolidation ──► KB / notes.md
                                                                                                            │
                                                                                                            └─► Pinned (manual subset)

Plan ──► PRD ──► Work Items (depends_on edges) ──► … ──► Verify Work Item ──► Manual Archive

Pipeline (stages: task / meeting / plan) ──► spawns Work Items / Meetings / Plans

Schedule (cron) ──► spawns Work Items
Watch (event)   ──► spawns notifications / Work Items / pipeline-continuations

Engine reads:  routing.md, config.json, all state JSON
Engine writes: control.json, log.json, metrics.json, dispatch.json, project state files
```
