# Slim UX — Architectural Suggestions

> Written after Round 1 (Phase A concept mapping + Phase B layout rebuild). These are **proposals**
> for engineering changes that would make the slim UX cheaper to maintain and more honest about
> what's happening under the hood. **None of these are implemented in Round 1.** They're for
> human discussion before any of them get prioritized.
>
> Format per section: **Problem → Proposal → Cost → Risk**.
>
> Cost is rough person-days. Risk is what could go wrong.

---

## 1. A `/api/cockpit` aggregate endpoint

**Problem.** Right now the slim cockpit has to hit `/api/status` every 5 seconds and pluck six
fields out of a ~60 KB blob (`dispatch.active`, `dispatch.pending`, `pullRequests`, `watches`,
`engine`, `dispatch.completed`). The full status payload includes `agents`, `inbox`, `notes`,
`metrics`, `prdProgress`, `verifyGuides`, `archivedPrds`, `skills`, `mcpServers`, `schedules`,
`pipelines`, `pinned`, `projects`, `autoMode`, `version` — all of which the cockpit ignores.
Polling that whole thing every 5 s on every open tab is wasteful.

**Proposal.** A new `/api/cockpit` that returns *only* the cockpit-relevant aggregates:

```jsonc
{
  "engine":   { "running": true, "mode": "running", "startedAt": "…" },
  "dispatches": { "active": 2, "pending": 5, "workingAgents": ["dallas", "ralph"] },
  "prs":      { "active": 3, "failingBuilds": 1 },
  "watches":  { "active": 4, "triggeredRecently": 1 },
  "schedules":{ "dueWithinHour": 2 },
  "lastEvents": [ { "kind": "completion", "ts": "…", "title": "…" }, … ]
}
```

Compute it from the existing fast-state cache in `dashboard.js` (`_fastState`) so the cost is
near-zero — just a few field projections. Slim polls this; full dashboard keeps `/api/status`.

**Cost.** ~0.5 day. New handler + a couple of unit tests covering aggregate math.

**Risk.** Low. Pure read-side; doesn't touch any state. The only failure mode is the slim
showing stale numbers if the cache pruner gets confused, but that's the same risk `/api/status`
already has.

---

## 2. SSE for cockpit updates instead of polling

**Problem.** 5-second polling is fine for Round 1 but feels laggy when an agent finishes — you
see "Sending…" disappear in the chat 30 s before "Active dispatches" updates from 1 to 0. The
full dashboard already has `/api/status-stream` (SSE); slim just doesn't use it.

**Proposal.** Slim subscribes to `/api/status-stream` (or a new `/api/cockpit-stream` that emits
deltas only). Polling becomes a fallback for browsers without SSE. Visibility-change pause
already exists in slim (round-1 code), so background tabs cost nothing.

**Cost.** ~0.5 day if reusing `/api/status-stream`; ~1.5 days if writing a delta-encoded
`/api/cockpit-stream`.

**Risk.** Medium. SSE connections occasionally drop and need exponential-backoff reconnect.
Multi-tab leaks are a real concern (each tab opens its own EventSource). Need a small connection
manager on the client.

---

## 3. A unified event stream for the History panel

**Problem.** The History panel currently merges three sources client-side: `dispatch.active`,
`dispatch.completed`, and `pullRequests`. Each has its own timestamp field (`startedAt`,
`completedAt`, `updatedAt`), its own shape, and its own truthiness rules. The merge logic is
fragile — adding a fourth source (e.g. consolidation runs, watch fires, schedule executions,
pipeline state changes) means more glue.

**Proposal.** A canonical `engine/events.jsonl` (append-only, rotated weekly) with a uniform
shape:

```jsonc
{ "ts": "2026-05-08T15:23:11Z", "kind": "completion", "agent": "dallas",
  "title": "fix slim button", "workItemId": "W-…", "pr": "#42",
  "summary": "…", "level": "info" }
```

Plus `/api/events?since=…&limit=…` for paged reads. The engine already writes most of these
moments to `engine/log.json` (the audit ring buffer); we just need to enrich them and expose
them. The History panel becomes one fetch.

**Cost.** ~1.5 days. The hard part is catching every event-emit site — dispatch start, dispatch
complete, PR sync, watch fire, schedule run, pipeline transition, meeting round advance,
consolidation run. Mostly mechanical.

**Risk.** Medium-low. Events are reads; the risk is missing one. Add a unit test that asserts
each lifecycle path emits an event.

---

## 4. Should Work Items + Pipelines merge in the data model?

**Problem.** Carlos called out "Work Items" and "Pipelines" as conceptually overlapping in the
existing UI. Looking at the data: a Pipeline is a sequence of stages where each stage *is*
either a Work Item, a Meeting, or a Plan. So a Pipeline isn't really a separate concept — it's
a Work Item with a stage list. Yet they have entirely separate stores
(`engine/dispatch.json` vs `engine/pipeline-runs.json`), separate API surfaces
(`/api/work-items*` vs `/api/pipelines*`), and separate UIs.

**Proposal.** Don't merge the *storage* — that's a deep migration. Instead, introduce a
"workspace" view at the API layer that returns Work Items and Pipeline runs in one paginated
list, tagged with `kind: "work-item" | "pipeline-run" | "pipeline-stage"`, sorted by recency.
Slim's "Work" button opens this unified view; it can drill down into stages when the user
expands a pipeline.

**Cost.** ~2 days. Read-only aggregator + a UI that knows how to render the three flavors. No
storage changes.

**Risk.** Low at the engine layer (read-only); medium at the UX layer (pipelines have richer
state — wait conditions, retriggers — that don't map cleanly to a flat list).

---

## 5. Plan + PRD: collapse to one document?

**Problem.** Carlos asked "What is a PRD? Should I generate one to get started?" The answer is
"no, PRDs are auto-generated from approved plans by the `plan-to-prd` agent" — but that's
opaque. The two-stage flow (plan → PRD → work items) exists because the plan is a human
discussion artifact and the PRD is the machine-readable execution contract.

**Proposal.** Don't drop the distinction; *unify the surface*. Treat the plan as the primary
document and embed the PRD JSON as a fenced block inside the plan markdown:

```markdown
# Plan: Refresh the Settings sidebar

## Goal
Three small commits, no big-bang refactor.

## Acceptance criteria
- …

```prd
{ "items": [ … ] }
```
```

The materializer reads the fenced block instead of a separate `prd/*.json` file. Plan and PRD
have the same lifecycle, the same `source_plan`, and the same archive path. The slim shows one
"Plans" button instead of two concepts.

**Cost.** ~3 days. Migration script for existing PRDs + materializer changes + verify flow.
Risky because plan resume logic depends on reading the PRD JSON twin.

**Risk.** Medium-high. The plan-to-prd diff-aware update logic is non-trivial and would need to
be re-verified. **Recommendation: defer until at least one round of slim UX shows the unified
"Plans" button is the right primitive.**

---

## 6. Notes + KB + Pinned: which survives?

**Problem.** Carlos couldn't tell Notes vs KB vs Pinned Context apart. The reality:

- **Notes inbox** = raw, unsorted (one file per agent per task).
- **KB** = consolidated, classified into 5 categories.
- **Pinned** = hand-picked subset that gets prepended to every agent prompt.
- **`notes.md`** = a *separate* consolidated team-decisions blob, also injected into agent
  prompts.

Four distinct concepts for "the team's memory" is at least one too many.

**Proposal.** Collapse to two concepts the human sees: **"Knowledge"** (everything durable)
and **"Pinned"** (the always-prepended subset). Hide the inbox/`notes.md` distinction from the
UI — they're consolidation pipeline internals.

The data layer keeps four stores under the hood; the UI presents two:

| UI label  | Data sources                                                  |
|-----------|---------------------------------------------------------------|
| Knowledge | `notes/inbox/*.md` + `notes/archive/**/*.md` + `notes.md`     |
| Pinned    | `pinned.md`                                                   |

Pinning a Knowledge entry promotes it into `pinned.md`; un-pinning returns it. Inbox vs archived
becomes a "freshness" badge on the Knowledge entry rather than a separate tab.

**Cost.** ~1 day for the API aggregator + new UI tab. No engine changes.

**Risk.** Low. Pure presentation.

---

## 7. Schedule vs Watch: is one a special case of the other?

**Problem.** Schedule = "fire on a cron pattern." Watch = "fire when a condition flips." They
share most of the lifecycle (definition, fire history, pause/resume, expire/stopAfter). The
two state files (`schedule-runs.json` / `watches.json`) and parallel CRUD APIs add maintenance
without adding real concepts.

**Proposal.** Generalize to **Trigger**, with two `kind`s: `cron` and `event`. Backend stays
in two files for now (avoid the migration), but the API and UI present them as one.

```jsonc
{
  "id": "nightly-tests",
  "kind": "cron",
  "cron": "0 2 *",
  "action": { "type": "work-item", "title": "Nightly tests", "agent": "dallas" }
}
```

```jsonc
{
  "id": "watch-pr-42",
  "kind": "event",
  "target": "PR-42",
  "condition": "merged",
  "action": { "type": "notify", "channel": "inbox" }
}
```

Slim's "Trigger" button opens one creator that branches on `kind`.

**Cost.** ~2 days. New aggregator endpoint, one creator UI, normalized response shape.

**Risk.** Medium. Watches have a richer condition vocabulary
(`new-comments`, `status-change`, `vote-change`, `build-fail`, …) than Schedules. The unified
surface can't paper over that — the creator UI still needs branch-per-kind rendering. So this
is mostly a mental-model win, not a surface-area-savings win.

---

## 8. "Two unrelated tasks side-by-side on the same project"

**Problem.** Carlos asked how to run two unrelated work streams in parallel on the same project.
Today the answer is: just queue two work items; the dispatcher will pick two agents (up to
`engine.maxConcurrent = 5`) and they'll work in separate worktrees. **But** there's no UI
affordance for separating them — both show up in the same flat dispatch queue. There's no
"context A vs context B" boundary.

**Proposal.** Add a **"thread" / "stream" tag** to Work Items. Optional string, free-form. The
slim's Status and History panels group by thread when present. Default thread = `default`.
Threads aren't in the engine at all — they're a presentation grouping. CC accepts them on
dispatch.

```jsonc
{ "title": "fix login bug", "thread": "auth-rewrite" }
{ "title": "rename buttons", "thread": "ux-cleanup" }
```

This costs almost nothing technically, and gives the human a way to keep two storylines
mentally distinct.

**Cost.** ~1 day. Schema field on work items, group-by in queries, slim UX wiring.

**Risk.** Low. Backward compat by treating missing thread as `default`.

---

## 9. Settings sprawl: split fast vs advanced

**Problem.** Carlos said Settings has too many options. The full dashboard's Settings page
mixes:

- Agents (per-agent CLI/model/skill/budget)
- Projects (paths, repo hosts, work sources)
- Engine knobs (timeouts, retries, concurrency)
- Runtime defaults (default CLI, default model)
- Feature flags
- ADO/GitHub auth
- MCP servers
- Cache controls / reset

Most users only ever touch feature flags + the slim-ux toggle. Most settings are either
*infra* (set once at install) or *fleet* (rarely changed).

**Proposal.** Two-tier settings:

- **Slim "Quick settings"**: just feature flags + a button to open the full settings.
- **Full "Advanced settings"**: everything else. Stays as is in the existing dashboard. The
  slim deep-links to it.

Round 1 already does this. The next step is to *split* the full dashboard's Settings page into
"Quick" (the things `/api/settings/reset` fixes) and "Advanced" (the things you tune once).

**Cost.** ~1 day on the full dashboard side. Zero on the slim side (already done).

**Risk.** None.

---

## 10. Recent Completions: less metadata, more context

**Problem.** Carlos said the completions widget shows too much (ID column, agent column, no
click-through to source). His mental model is: "I want to know what shipped, when, and what
prompted it."

**Proposal.** Restructure each completion entry as a **prompt → result** pair:

```
12:04  Carlos: "Fix the dashboard typo on the Plans page"
       → Dallas: ✅ shipped PR #43 (2 files, 4 lines) · 1m ago
                 view diff · view PR · view chat thread
```

The "what prompted it" piece requires the engine to remember the originating CC message — which
it doesn't today. Either:

1. **Cheap:** When CC dispatches, stash the originating message text in the work item's
   `description` field (it already does this loosely).
2. **Right:** Add `_originPrompt` (originating CC turn) and `_originSession` (CC session id) to
   the work item. Slim's history panel can then render the prompt above the result and link
   back to the chat.

**Cost.** ~1 day for the cheap version, ~2 days for the right version.

**Risk.** Low. Read-side change to the dispatch creation path.

---

## 11. "Skills + MCPs probably belong off the main screen"

**Problem.** Carlos called these out as "off the main screen." Skills and MCPs are
power-user/runtime concerns; they pollute the slim's mental model.

**Proposal.** Drop the "Tools" page entirely from slim. Keep them in the full dashboard.
Surface a single "X skills, Y MCPs available" line in slim's Settings dialog with a deep link.

**Cost.** Zero (already done in Round 1 — slim has no Tools page).

**Risk.** None.

---

## 12. Dashboard build pipeline: extract slim assets

**Problem.** The full dashboard SPA is built once at startup from `dashboard/layout.html` +
fragments and gzipped. The slim deliberately bypasses this so iteration is hot — read fresh on
every request. As slim grows past the 1000-line mark, that single-file model will hurt:

- CSS, action-prompt JS, cockpit JS, history JS all in one file.
- No component reuse between slim panels.
- Hot-reload works for the file but not for individual sections.

**Proposal.** Extract slim assets into `dashboard/slim/` once it crosses a complexity threshold
(round 3 or 4):

```
dashboard/slim/
  index.html        ← references the assets below
  cockpit.js
  history.js
  actions.js
  chat.js
  styles.css
```

`serveSlimUx` reads `index.html` and inlines or links the others. Hot-reload still works. Each
file < 300 lines. **Don't do this yet** — the seam is wrong below 1500 lines.

**Cost.** ~1 day.

**Risk.** Low. CSP relaxation already in place; just needs to extend to include
`/dashboard/slim/*` if assets become cross-origin-y. (They won't for same-origin assets.)

---

## 13. CC actions: the "create a watch from chat" path needs a visual confirm

**Problem.** Round 1 wires the four Action buttons through CC by typing
"Create a work item: …" into the chat. CC parses, emits `===ACTIONS===`, and the action
executes. But the user has no easy way to confirm "yes, that's the work item I meant" before
it's queued. CC's free-text understanding is good but not perfect.

**Proposal.** Add an optional `dryRun: true` flag on CC actions. When set, CC echoes the parsed
action as a structured preview and *does not* enqueue. The slim modal sets `dryRun: true` by
default; the user clicks "Confirm" to submit a second turn that drops the flag.

Or, simpler: skip CC entirely for the four buttons, and call `/api/work-items` /
`/api/plans/create` / `/api/notes` / `/api/schedules` directly with form fields. Round 2 task.

**Cost.** ~1 day for direct API path. ~1.5 days for the dryRun preview flow.

**Risk.** Low for the direct path; medium for dryRun (needs CC system prompt edits + tests).
**Recommendation: do the direct API path; skip dryRun.**

---

## 14. Charter + Routing should be one page in slim

**Problem.** Charter is "who the agent is." Routing is "what work goes to whom." They're tightly
coupled (you can't route review to an agent whose charter says "doesn't review") but live in
totally separate places.

**Proposal.** Add a slim sub-page (off the gear menu) called **"Team"** that shows each agent
side-by-side with: charter excerpt, routing rules they own, recent completions, current
dispatch (if any). One screen for "what is the team doing?"

**Cost.** ~1.5 days. Read-only aggregator + simple grid UI.

**Risk.** Low.

---

## 15. The slim-ux feature flag itself: ramp plan

**Problem.** `slim-ux` is currently the only entry point; if it breaks, the user can't get back
to the full dashboard except via the `?fullDashboard=1` deep link (round-1 added this) or the
settings dialog. That's fine for early dogfooding, but at GA we'll want a smoother fallback.

**Proposal.** Two feature flags instead of one:

- `slim-ux` — show the slim *option* (renders a "Try slim UX" link from the full dashboard).
- `slim-ux-default` — slim becomes the root route; full moves to `/full`.

Round 1's behavior corresponds to `slim-ux: true && slim-ux-default: true`. Once slim is GA,
default flips on; `slim-ux` registry entry gets deleted as redundant.

**Cost.** ~0.5 day.

**Risk.** None. Simple gating.

---

## 16. Surprises found during Phase A

A few things were genuinely surprising while building the concept dictionary. Capturing them so
the team can decide whether they're features or footguns:

**a. `notes.md` and the KB are *separate* memory stores.** Both feed agent prompts; they don't
share content. Consolidation either writes to `notes.md` (decisions / pinned-style) or
classifies into a KB category. There's no "this fact lives in both" path. **Probably fine; just
worth a docstring somewhere.**

**b. CC sessions are *non-expiring*.** The CC system intentionally never prunes sessions
(per `CLAUDE.md` line ~XYZ). So a long-lived tab keeps growing the prompt forever. The
prompt-hash-mismatch invalidator is the only safety net. **Consider a soft "session age" warning
in the UI when context approaches the cache limit.**

**c. Build-failure cache can go stale.** `_buildStatusStale` is honored by the auto-fix
dispatcher but not by the cockpit display. Slim's PR tile could show "1 failing build" when the
build is actually green-after-rerun. **Add `_buildStatusStale` consideration to the cockpit
display.**

**d. `dispatchCompletionReportPath` writes to `engine/completions/<dispatchId>.json` but the
oversized-prompt sidecar writes to `engine/contexts/<dispatchId>.json`.** Two directories with
similar purposes. The dashboard widget for "open completion report" sometimes hits the wrong
one if you copy-paste a path. **Either merge the directories or make the API explicit about
which kind of artifact you want.**

**e. `WORK_TYPE` has 13 entries but `routing.md` only routes ~7 of them.** The unrouted types
(`MEETING`, `EXPLORE`, `ASK`, …) are routed by other code paths. **Either consolidate routing
or document the gap clearly.**

---

## Suggested order of operations

If we were prioritizing for a Round 2 sprint, this is roughly the order I'd go:

1. **§1** `/api/cockpit` (½ day, immediate slim perf win)
2. **§13** Direct API calls for the four Action buttons (1 day, removes the TEMP layer)
3. **§3** `/api/events` + history feed (1.5 days, replaces three client-side merges)
4. **§6** Knowledge / Pinned UI consolidation (1 day, big mental-model simplification)
5. **§2** SSE for cockpit (½–1.5 days, removes the polling loop)
6. **§10** Origin prompt linkage on completions (1 day, makes "what prompted this?" answerable)

Total: ~6 days for a meaningful Round 2 that earns real shipping confidence.

The bigger model-merger questions (§4 Work Items + Pipelines, §5 Plan + PRD, §7 Schedule +
Watch) should wait until at least Round 3 — we should let the slim UX expose where the
*model* friction actually is, not assume it from the current UX friction.
