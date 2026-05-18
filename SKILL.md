---
name: sprint-monitor
description: Generates a sprint health report for the user's direct reports — pulls tickets from Jira, classifies each by status (on-track / at-risk / blocked / done / not-started), flags stale or scope-creeping work, and posts a short manager-ready summary to Slack. Designed primarily for scheduled Claude routine tasks (recurring delivery to a configured channel), with a local interactive mode for on-demand long-review deep dives. Trigger whenever the user asks about their team's sprint, asks how their reports are doing, wants a sprint status / sprint health / sprint monitor / sprint check, or is prepping for a sprint review or 1:1. Use this even when the user doesn't say the word "sprint" — phrases like "how's my team", "anything blocking the team", "what should I ask in 1:1s this week" should trigger it.
---

# Sprint monitoring

A skill for engineering managers to get a fast, structured read on how their team's current sprint is going. Pulls from Jira via the Atlassian MCP and delivers the report to Slack.

## Config

The skill's group config is **inlined in the run prompt** — every routine fires with the full config block, and there is no file-based group config inside this skill repo. The routine template owns the config; the skill receives it as input on every invocation.

The skill is **stateless across runs**. There are no `cached_sprints`, no `last_canvas` block, no calendar-PTO write-back — anything the skill needs that wasn't in the prompt is re-derived from Jira / Slack / Google Calendar at run time.

### Config schema

The prompt carries a single JSON object with these fields:

- `name` — slug identifier for the group (e.g. `"alice"`). Used in log lines and in the structured hand-off to the renderer.
- `country` — ISO country code (e.g. `"PL"`) — the group's own country. Holidays from this country's list in [references/holidays.md](references/holidays.md) **reduce capacity** group-wide. Defaults to `"PL"` if missing.
- `info_countries` — array of ISO codes (e.g. `["IL"]`) — additional countries whose holidays surface as **info only** in the canvas/delta (cross-team awareness, e.g. when Israeli colleagues are off). These do NOT affect capacity math. Defaults to `["IL"]`.
- `jira_base_url` — the Jira instance URL (e.g. `https://acme.atlassian.net`).
- `jira_field_ids` — instance-discovered custom field IDs for `story_points`, `remaining_story_points`, `sprint`, and `flagged`. These vary per Jira instance — the routine template captures them once at setup and inlines them on every run.
- `manager` — `{ name, email, slack_id }` for the group's manager / chapter lead. The `slack_id` is also the identity `<@…>`-tagged on every channel post, and (critically) the disambiguator the skill uses to find this manager's prior canvas in `output_channel` — see "Finding the prior canvas" below.
- `output_channel` — Slack channel ID (`C…`) where every canvas link and every delta gets posted, with the manager `<@…>`-tagged in the message body. Set to `null` to fall back to a DM-to-manager (legacy behavior — channel post is the default). `"terminal_only"` suppresses Slack entirely (local-mode only).
- `reportees[]` — array of `{ name, email, jira_account_id, pto }`. `pto` is an array of vacation/sick entries the prompt author has chosen to include (manual overrides); the skill also fetches OOO live from each reportee's Google Calendar on canvas runs and merges in-memory for the duration of that run. See [references/pto.md](references/pto.md).

**Each run processes exactly one group.** If a manager wants reports for multiple groups (chapter leads), each group is a separate routine entry with its own inlined config.

### Finding the prior canvas

The skill renders a per-reportee `prev_% → new_%` diff line on the Capacity overview when a prior canvas exists. Because there's no file-based state, the prior canvas is recovered from Slack at run time:

1. Search `output_channel` for the most recent message authored by the bot that contains both:
   - the literal mention `<@<manager.slack_id>>` (the disambiguator — different managers in the same channel never share a canvas), AND
   - a canvas link (a Slack canvas URL — recognizable via the canvas URL shape).
2. If found, fetch that canvas via `slack_read_canvas`, parse the per-reportee percentages out of its Capacity overview section, and use them as `prior_pct[<name>]`. Then **update that canvas in place** via `slack_update_canvas` instead of creating a new one.
3. If no prior canvas message exists for this manager — or the read/parse fails — fall back silently: create a fresh canvas and render every Capacity overview line in absolute-% shape (no diff).

The manager's `slack_id` is the only identity that ties a canvas to a manager. **Never reuse another manager's canvas, even if it's the most recent one in the channel** — that's why the `<@<manager.slack_id>>` mention is the search predicate, not "most recent canvas".

## Modes of operation

The skill ships **two render formats** in v1, picked by the run's `--mode` flag (or natural-language intent in local mode):

| Mode | Output | Use when |
|---|---|---|
| **Canvas** ([references/output/canvas.md](references/output/canvas.md)) | Slack Canvas, full report (snapshot + capacity + concerns + risk deep-dive + capacity tables). Canvas link DM'd to the manager. | Mon and Thu (the deeper read days). Local mode "full report", "deep dive", "canvas". |
| **Delta** ([references/output/delta.md](references/output/delta.md)) | Short Slack message DM'd to the manager — *only what changed* since the last run. | Tue, Wed, Fri. The 30-second pulse-check days. |

Sun has no run. Neither mode writes any persisted-state file — the delta reconstructs "what changed since yesterday" live from Jira's changelog (`status CHANGED AFTER -24h`), so there is no snapshot lifecycle to maintain. See [references/output/delta.md](references/output/delta.md) "What makes this work" for details.

The third surface — **terminal** — applies in local mode for either render format: the skill prints the same content as markdown to stdout in addition to (or instead of) the Slack delivery.

**Detection:** treat invocations as routine-mode by default (non-interactive). Switch to local-mode behavior only when the user is clearly present — e.g. when the prompt is conversational. The cost of being wrong-toward-routine is "we don't ask a question we could have asked"; the cost of being wrong-toward-local is "the routine hangs forever waiting on input it can't get." The asymmetry favors routine-by-default.

## Routine setup

Every run processes the single group defined by the inline config in its prompt. **One routine entry = one group.** Running reports for multiple groups on the same cadence = multiple routine entries, each with its own inlined config block. This is deliberate: it keeps each Slack message scoped to its own audience and gives each manager their own independent canvas.

If the prompt has no recognizable config block, the run fails fast — emit an error to the report destination (*"Sprint Monitor cannot run: no config block in prompt."*) and stop. There's no fallback to "pick a group from somewhere else."

## PTO parsing (local mode)

In local mode, recognize PTO/sick-leave statements in the manager's prompt — even outside an explicit sprint-report request — and add the entry to the relevant reportee's `pto` array **in the in-memory config for this run**. Local-mode PTO parsing exists for ad-hoc reports the manager is composing live; the entry is not persisted anywhere. If the manager wants the entry to survive across runs, they update the routine's inlined config (or add the OOO to their reportee's Google Calendar — synced on every canvas run).

Triggers: *"X is on vacation / out / OOO / sick / on leave"* with a date or duration qualifier (*"today"*, *"this Friday"*, *"next week"*, *"for a week"*, *"from May 22 to May 26"*).

Rules:
- *"for a week"* = **5 working days** (not 7 calendar days) — per the manager's standing rule.
- *"7 days"* (or *"N days"* where the unit is ambiguous) — **ask**: "working days or calendar days?". If unable to ask (background context), default to N working days.
- Unambiguous parse → add the entry immediately, confirm what was added in the reply (key, dates, working-day count).
- Ambiguous reportee name → ask which person.

Full parsing rules, working-day math, public-holiday interaction, and edge cases live in [references/pto.md](references/pto.md). Routine mode does not parse natural-language PTO — it only uses entries that are already inlined in the prompt config (plus whatever the canvas-run Google Calendar sync pulls in live).

## Required connectors

- **Atlassian MCP** (Jira) — required for everything. In routine mode, must be authenticated up-front; the routine can't complete the auth flow itself. If unauthenticated, emit an error and stop.
- **Slack MCP** — required for routine delivery (unless `output_channel` is `"terminal_only"`). In local mode, also needed for `--post-slack`. Same rule on auth: if unauthenticated in routine mode, emit an error and stop.
- **Google Calendar MCP** — required for canvas runs. Powers the Step 1b PTO sync — pulls each reportee's all-day OOO events for the current sprint window and merges them into the in-memory `pto[]` arrays before capacity math runs. No write-back to anywhere — the calendar IS the source of truth. If unauthenticated when a canvas fires, emit a clear error to the report destination and stop — capacity math without the calendar produces misleading verdicts. See [references/permissions.md](references/permissions.md) "Required: Google Calendar" for the full rule.
Never hardcode tokens. All API access goes through the MCPs.

**Permissions are pre-granted for the duration of a session.** Once the user has invoked this skill (locally) or the routine has fired (with prior auth), treat the connector calls listed in [references/permissions.md](references/permissions.md) as authorized — don't pause to ask "can I call Jira?" or "is it OK to look up Slack users?" before each request. Read that file at the start of a session and proceed silently from then on.

## Sprint analysis

The flow is **per reportee within the group inlined in the prompt** — each reportee gets analyzed individually because they may be on different teams, boards, and sprints. Run every step below for every reportee in the prompt's `reportees[]` array. Detailed JQL queries, classification rules, signal definitions, and capacity math live in [references/analysis.md](references/analysis.md); the steps here describe *what* happens and *why*, not the exact queries.

### Step 1 — Load reportees

Parse the inline config block from the prompt — `jira_base_url`, `jira_field_ids`, `manager`, `output_channel`, and `reportees[]` all come from there. Iterate `reportees[]` — each one carries a `jira_account_id` and `email`; use those directly. Don't re-resolve names against Jira at runtime — the IDs are captured once during routine setup and treated as canonical.

### Step 1b — Sync PTO from Google Calendar *(canvas runs only)*

Before pulling Jira data, fetch each reportee's OOO from their Google Calendar and merge it into the in-memory `pto[]` arrays for this run. The capacity math in Step 6 reads from those arrays.

- **Canvas runs only.** Delta runs skip this — PTO is sprint-scope, not 24h-scope, and the delta is a short-cycle pulse-check that doesn't pay the calendar cost.
- **Hard requirement on the Google Calendar MCP.** If unauthenticated, emit a clear error to the report destination and stop — capacity math without the sync is misleading.
- **In-memory only.** The synced entries live for this run only. There is no write-back (the skill is stateless across runs). The next canvas run re-fetches from the calendar — the calendar is the source of truth.
- **Reconciliation with prompt-inlined entries.** Manual entries from the prompt config win in conflicts; otherwise merge.

Full rules — event filter, mapping to the `pto` schema, reconciliation, failure modes — live in [references/pto.md](references/pto.md) "Google Calendar OOO sync".

### Step 2 — Find each reportee's active sprint(s)

**A reportee can be in multiple active sprints at once** — typical case is an FE engineer contributing to 2–3 squads' boards. Fold sprint discovery into the ticket-pull JQL via `sprint in openSprints()` — one round-trip returns every active sprint plus its tickets. See [references/analysis.md](references/analysis.md) Step 2 for the full rule.

**No active sprint is a normal case, not an error** — note it for that reportee in the report and continue with the others.

### Step 3 — Pull tickets per reportee

For each reportee in their active sprint, collect per ticket:

- Status, story points, last 3 comments.
- Status-transition changelog scoped to the sprint window (powers time-in-column, scope-creep, and reopen detection).
- Any linked pull requests (via the development-info field).

Story points: resolve the SP and remaining-SP custom field IDs from `jira_field_ids` in the inline config — they vary per instance, captured once at routine-setup time. The procedure and instance gotchas (including how to detect whether the team uses rSP as a burndown counter, which flips which field is "canonical") live in [references/analysis.md](references/analysis.md) Step 0.

### Step 4 — Classify each ticket

Bucket each ticket into one of **on-track / at-risk / blocked / done / not-started**. Specific rules (which statuses map where, which signals push a ticket into "at-risk", etc.) are in [references/analysis.md](references/analysis.md).

### Step 5 — Scan for signals

Surface anomalies worth raising in the report. Detection rules and thresholds live in analysis.md; the categories are:

- **Stale** and **stuck-in-flight-status** (>3 days in In Progress / In Review / Ready for Verification / Verified / Merged).
- **Reopens** and **scope creep** (story-point increases mid-sprint).
- **Always-raise** — missing estimate, 8 SP anomaly.
- **High comment velocity** (>5 in 3 days), **due-date approaching/passed**, **flagged**.
- **Priority inversion** — high-priority ticket idle while lower-priority work is in flight.
- **PR-related** — no PR / no review / merged but ticket not done.
- **"Blocked" / "waiting on" phrases** in recent comments.

### Step 6 — Assess per-reportee capacity

Produce a verdict per person: **underworked / normal / borderline / overbooked / fragmented**. The math:

- Convert each ticket's story points to a focused-dev-hour estimate: 0 SP=0.5h, 1 SP=1.5h, 2 SP=8h, 4 SP=20h. 8 SP is an anomaly — counted at 40h *and* flagged separately as an "oversized ticket" signal in Step 5.
- Sum across tickets and compare to an 80h full-sprint capacity.
- Use literal `remaining_story_points` (rSP) for the per-ticket hour contribution — it's the field that captures "this sprint's commitment" after carry-over. Fall back to `story_points` only when rSP is null. Size-bucket counts and the 8-SP anomaly signal still use `story_points` (absolute ticket size, not remaining work). See [references/analysis.md](references/analysis.md) Step 6 for the full rule.

This catches sandbagging, over-commitment, and context-switching shapes that per-ticket signals miss. If a reportee has PTO entries (from the inline config or the canvas-run calendar sync) or the group's `country` has public holidays inside the sprint window, scale `capacity_hours` and verdict thresholds proportionally — full rules in [references/pto.md](references/pto.md).

### Step 7 — Build the report

Two render formats, branch on `--mode` (or natural-language intent in local mode):

**Canvas mode** — render per [references/output/canvas.md](references/output/canvas.md). Up to seven sections in order (sections 3 and 4 are conditional; 1b is conditional on a prior canvas being found in Slack):
1. Sprint health snapshot (blocked / unestimated / "need to be splitted")
2. Capacity overview (one line per reportee, sorted Overbooked → Underworked, status-colored icons; shows absolute % when no prior canvas was found in Slack, and a `prev_% → new_% · reason` diff when one was — see [references/output/canvas.md](references/output/canvas.md) Section 2)
3. Out this sprint — personal PTO only (vacation / sick), conditional
4. Group holidays this sprint — info block, country (PL) holidays first then `info_countries` (IL) holidays inside the sprint window, conditional
5. Concerns (per-reportee nested list, filtered to actionable signals)
6. Risk deep-dive (top 5)
7. Capacity breakdown (per-reportee tables)

The canvas title is `<traffic-light emoji> Sprint <N> — <DD Month>`.

**Delta mode** — render per [references/output/delta.md](references/output/delta.md). A short Slack message reconstructing the last 24h of Jira changelog activity (no stored baseline). Seven categories plus 7b: new blockers, reopens, shipped, status moves, threshold crossings, PTO updates, late sprint additions, sprint removals. **Empty categories are skipped entirely** — no `None.` placeholder lines. Plus a (conditional) "still open" persistent line, country-holiday-today lines (PL + IL), and a link back to the most recent canvas. Quiet days collapse to 2 lines acknowledging the routine ran.

In local mode, also print the chosen format as markdown to the terminal (canvas content: drop Slack-specific `![](@U…)` user-mention syntax, render as `@Name`).

**No persisted state.** Neither mode writes a snapshot, a config file, or any other "yesterday-state" artifact — the delta queries Jira's changelog directly for its 24h window, and the canvas's diff math finds the prior canvas in Slack. See [references/output/delta.md](references/output/delta.md) "No persisted state".

### Step 8 — Deliver

Each run delivers **one** output — for the single group whose config was inlined in the prompt.

**Canvas mode:**
1. Look up the prior canvas in `output_channel` (see "Finding the prior canvas" above). If found, call `slack_update_canvas` with `action=replace` on its canvas id to rewrite the whole canvas in one shot. If not, call `slack_create_canvas` for a fresh canvas. Pass the title via the `title` argument either way; do **not** repeat the title inside the content (it produces a duplicate heading at the top).
2. Post the canvas link to `output_channel` via `slack_send_message`. Lead the message body with `<@<manager.slack_id>>` so the manager gets pinged. Body: one-line title, the canvas URL, a 1–2 sentence TL;DR pointing at the most urgent item. **The `<@…>` tag on this message is what lets the next canvas run find this canvas again** — never omit it on channel posts.
3. **Channel post is the default.** Fall back to a DM-to-manager only when `output_channel` is `null`. If `output_channel` is `"terminal_only"`, suppress Slack entirely.

**Delta mode:**
1. Render the short Slack message per [delta.md](references/output/delta.md). Prepend the `<@<manager.slack_id>>` tag to the first line.
2. Send via `slack_send_message` to `output_channel`. Same fallback chain as canvas mode (DM-to-manager when `output_channel` is null; terminal-only suppresses Slack).

**Both modes:** the manager `<@…>` tag is what makes a channel post visible to the right person — without it, the report just sits in a busy channel. Always include it on channel posts; omit it on DMs (the DM target IS the audience).


## Behavioral guardrails

- **Be terse in the canvas.** Managers skim. The Capacity overview is one line per person; Concerns are filtered to actionable signals only (no duplicates of what's in Sprint health snapshot).
- **Routine mode never blocks.** Any branch that would require user input must have a graceful non-interactive fallback (treat-as-zero, skip-with-note, or hard-stop with a clear error). No silent hangs.
- **One delivery per run.** Routine mode posts to Slack exactly once; local mode prints exactly once and optionally posts exactly once with `--post-slack`. Re-runs are full re-runs, not append-to-previous.

## Files in this skill

- [SKILL.md](SKILL.md) — this file (entry point, modes, inline-config schema, sprint analysis flow).
- [references/permissions.md](references/permissions.md) — which MCPs/tools the skill treats as pre-granted.
- [references/analysis.md](references/analysis.md) — Jira queries, ticket classification, signal detection, capacity math.
- [references/pto.md](references/pto.md) — PTO/sick leave: parsing, calendar sync, capacity scaling, display in canvas + delta.
- [references/holidays.md](references/holidays.md) — public-holiday lookups per country/year (the group's `country` selects which list applies).
- [references/token-economy.md](references/token-economy.md) — single-page index of the token-saving strategies used across the skill.
- [references/output/canvas.md](references/output/canvas.md) — full canvas layout (Mon/Thu render format).
- [references/output/delta.md](references/output/delta.md) — short daily-delta Slack message (Tue/Wed/Fri render format).
- _(No `references/groups/` directory and no `references/snapshots/` directory.)_ Group config is inlined in the routine prompt, and the delta has no persisted state — see "Config" above and [references/output/delta.md](references/output/delta.md) "No persisted state". Delete any stale `references/groups/` or `references/snapshots/` files if found.
