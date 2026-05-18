# Daily delta

The short daily companion to the [canvas.md](canvas.md) full report. Posted on days when the manager doesn't need the whole picture — just *what changed* since yesterday.

Designed for routine cadences like:
- **Mon, Thu** → full canvas ([canvas.md](canvas.md))
- **Tue, Wed, Fri** → daily delta (this file)

Sat and Sun have no run. The 5-day workweek rhythm matches a typical sprint cadence: Mon/Thu carry the deeper read, the in-between weekdays get a 30-second pulse, weekends are off.

The delta is a single Slack message DM'd to the manager — not a canvas. ~10–15 lines, scannable in 30 seconds.

## What makes this work: a 24h Jira changelog window

The delta is reconstructed live from Jira's changelog, not from a stored baseline. **No local state persists between runs.** Each category translates to a JQL query bounded by `... AFTER -24h` (or the equivalent changelog filter), and the answer is whatever Jira reports right now.

Why this works:
- Jira is already authoritative. A local snapshot file would just be a copy that drifts out of sync if the manager (or anyone else) edited a ticket mid-window.
- The Atlassian MCP supports `status CHANGED AFTER -24h`, `status CHANGED TO Done AFTER -24h`, `status CHANGED FROM ("Done", "Closed", ...) AFTER -24h`, `sprint CHANGED AFTER -24h`, etc. — directly. One JQL per reportee per category.
- First-run scenarios disappear. There is no "first" run because there is no baseline to establish; the delta just looks back 24h every time.

**The skill does not write `references/snapshots/<slug>.json`** (or any other persisted-yesterday file). Earlier spec revisions used a snapshot; that mechanism has been removed.

### Window convention

- The default window is **`-24h`**: tickets whose status / sprint / scope changed in the last 24 calendar hours from the moment the delta runs.
- The Tue/Wed/Fri cadence means a 24h window covers Mon→Tue, Tue→Wed, Thu→Fri respectively — clean alignment with the daily cycle.
- The Mon delta (if a Mon-delta day were added) would naturally miss Fri→Mon changes that landed over the weekend; the spec's cadence puts the Mon slot on the *canvas* anyway, which renders full state and sidesteps the issue.
- The window is hardcoded at 24h. Don't try to widen it to "since the last canvas" — that introduces stateful timing logic again, which is exactly what removing the snapshot was supposed to avoid.

## Empty sections are skipped entirely

Every category below is rendered **only when it has ≥1 item**. If a delta run has zero new blockers, zero reopens, etc., those headings do not appear in the message at all — no `## 🔄 Reopens — None.` filler. The delta is a signal-only doc; placeholder lines for empty categories defeat its 30-second-scan purpose.

If *every* category is empty (genuinely quiet day), fall back to the collapsed two-line form in "Quiet days" below.

## Delta categories

Seven categories (plus 7b "removed from sprint"), listed in the order they should appear in the message when present. An earlier spec revision carried a "⚖️ Capacity verdict changes" category, dropped when the snapshot mechanism was removed; the canvas's Capacity overview still shows current verdicts.

### 1. 🚧 New blockers

Tickets that transitioned **to** a Blocked status in the last 24h.

JQL: `assignee = "<accountId>" AND sprint in openSprints() AND status CHANGED TO "Blocked" AFTER -24h`

One line per ticket: key, owner, priority, time since the transition, and whether a comment was added at the time of the transition (pull last comment timestamp; if it's within ±10 minutes of the changelog entry, include the first line).

```
🚧 New blockers (2):
  ZETA-2700 (Frank, Medium) — Blocked 2h ago, no comment yet
  EPSILON-3900 (Grace, High) — Blocked 6h ago, "waiting on infra team"
```

### 2. 🔄 Reopens

Tickets whose status moved **from** a Done-bucket status to an indeterminate/new status in the last 24h.

JQL: `assignee = "<accountId>" AND sprint in openSprints() AND status CHANGED FROM ("Done", "Closed", "Resolved", "Verified", "Merged", "Won't Do", "Duplicate") AFTER -24h`

Reopened tickets are high-attention — surface the from/to and any explanatory comment.

```
🔄 Reopened (1):
  ALPHA-1009 (Bob) — Done → In Review, "Netlify failure"
```

### 3. 📈 Shipped

Tickets that moved **to** a Done-bucket status in the last 24h.

JQL: `assignee = "<accountId>" AND sprint in openSprints() AND statusCategory CHANGED TO done AFTER -24h`

The wins. Anchors the report even on busy days.

```
📈 Shipped (4):
  GAMMA-663, GAMMA-649 (Carol)
  DELTA-2722 (Eve)
  EPSILON-3848 (Grace)
```

Group by owner if more than 1 per person.

### 4. 🔁 Status moves (in-flight)

Tickets whose status changed in the last 24h **without** crossing into Done or Blocked (those have their own categories). Captures the "PR pushed for review" / "QA picked it up" / "carry-over resumed" beats that fill out a healthy day.

JQL: `assignee = "<accountId>" AND sprint in openSprints() AND status CHANGED AFTER -24h` — then drop the rows already accounted for in categories 1–3.

```
🔁 Status moves (3):
  ALPHA-1009 (Bob) → IN_REVIEW
  ETA-5592 (Dave) → IN_REVIEW
  GAMMA-551 (Carol) → In Progress (carry-over resumed)
```

### 5. ⚠️ Threshold crossings (newly stuck / now at-risk)

Surface tickets that **just crossed** a threshold the canvas would also flag — but only if the crossing happened in the last 24h.

- **Stuck >3d in an in-flight status.** A ticket is "newly stuck" when its current `time_in_status` is between 72h and 96h AND there was no status change in the last 24h. The 72–96h band means the threshold was crossed between yesterday and today. Pull `time_in_status` from the changelog (most-recent status transition timestamp). Eligible statuses: In Progress, In Review, Blocked, Verification, Verified. Exclude To Do, Dev Ready, Merged, Done.
- **Due date now ≤2d.** Ticket has `duedate >= today AND duedate <= today + 2 days AND duedate > today - 1 days` from yesterday's reference frame. In practice: a ticket whose `duedate` is today, tomorrow, or the day after, and was *not* in that window yesterday. Approximate by: `duedate <= today + 2d AND duedate >= today + 2d` (i.e., the duedate is exactly 2 days out — that's the day the threshold first fires). Acceptable to also include `duedate = today + 1d` if the spec is being strict about same-day clarity.
- **Sprint ends ≤2d with this ticket still not done.** Same logic against `sprint.endDate` rather than `duedate`. **Fri-end ≈ Mon-end of the same close-out week** — see [../analysis.md](../analysis.md) "Practical Jira-MCP tips → Time math". Don't fire this on Friday for a Fri-ending sprint when you wouldn't also fire it on the prior Thursday for a Monday-ending sprint — the weekend collapses the gap and singling out Friday-ending sprints is noise.

These can each be computed without a snapshot — they're all functions of the current changelog and current dates.

```
⚠️ Newly stuck >3d:
  ETA-5404 (Dave) — In Review 3d (sprint ends Fri)

⚠️ Due now ≤2d:
  ALPHA-1009 (Bob, due 23 May)
```

### 6. 🌴 PTO updates

Someone's PTO starts today or ends today. Detect by scanning the inlined `pto[]` array on each reportee for entries whose `start` or `end` equals today. The skill has no across-run memory, so it can't surface "added since yesterday" — only today-edge transitions fire. Public holidays do NOT appear here — they have their own dedicated lines in the persistent info section below.

```
🌴 PTO updates (2):
  Bob — out today (vacation, May 20–22, 3d, added yesterday)
  Eve — sick today (added 2h ago)
```

When a PTO range crosses sprint boundaries, only mention the dates that fall inside the current sprint window. See [../pto.md](../pto.md) "Display in delta" for full rules.

### 7. 🆕 Late sprint additions

Tickets whose `sprint` field gained the active-sprint ID in the last 24h.

JQL: `assignee = "<accountId>" AND sprint in openSprints() AND sprint CHANGED AFTER -24h` — then filter to entries where the *added* sprint id matches one of the active sprint ids.

```
🆕 Added to sprint (2):
  ALPHA-1112 (Bob, 4 SP) — added today, +20h to his load
  ZETA-2710 (Frank, 1 SP)
```

Skip if the additions are sub-tasks of an in-progress parent — those are normal mid-sprint breakdown, not real additions.

### 7b. 🚮 Removed from sprint

Tickets whose `sprint` field lost an active-sprint ID in the last 24h. The mirror image of category 7 — the manager wants to see what walked out of the sprint, not just what came in.

JQL (per reportee, per current active sprint id): `assignee = "<accountId>" AND sprint CHANGED FROM <sprint_id> AFTER -24h`.

**Don't add `sprint in openSprints()`** to this query — the whole point is the ticket *left* the sprint, so it may no longer be in any open sprint. The `assignee` + `sprint CHANGED FROM <id>` pair is the correct selector.

Per result line: ticket key, owner, prior SP, status when removed, and where the ticket went (current sprint or backlog).

```
🚮 Removed from sprint (2):
  ETA-5404 (Dave, 4 SP, was Blocked) — pushed to ETA Sprint 13
  ALPHA-1010 (Bob, 4 SP, was Backlog) — back to backlog
```

Destination phrasing matches the canvas's Section 1b ([canvas.md](canvas.md) "Removed since last canvas"): "pushed to <Sprint Name>" / "back to backlog" / "moved to <Sprint Name>".

Skip sub-task removals — same rationale as the additions category.

## Useful info — the "still standing" section

Beyond pure deltas, the manager wants 1–3 lines of persistent context they shouldn't have to remember. **Each line is conditional**: render only if the underlying signal applies. Skip the whole section if all three are empty.

- The **highest-attention item that hasn't moved**. E.g. "ZETA-2624 still blocked 12d, still no estimate" — surfaces every day it doesn't change, since stagnation IS the signal. Pull from the canvas's most recent at-risk/blocked top-5; render only if at least one of those items shows no changelog activity in the last 24h.
- **Headline numbers**: total in-flight, days left in sprint, 1-line traffic light. Render only on days where the traffic light is non-green or sprint is closing within 2 working days; otherwise it's clutter.
- **Country holidays today + tomorrow** — one line per country whose list has *today* or *the next working day* as a holiday. `country` line first (capacity-affecting), then each `info_countries` entry in array order. The "tomorrow" half is a heads-up so the delta isn't blindsided by an eve-of-holiday day where colleagues may be on a half-day (especially the IL erev-of-holiday pattern — see [../holidays.md](../holidays.md) for half-day flags). Phrase tomorrow's entry as `🇮🇱 IL holiday tomorrow: Shavuot (Fri 22 May)`. Skip the line for any country with no holiday today or tomorrow; skip the whole subsection if both days are normal for every relevant country.
- **Link to most recent canvas** — render only when a canvas is available to link to (which is essentially always after the first Mon/Thu of the sprint). Drop if missing rather than emitting a broken link.

```
📌 Still open:
  ZETA-2624 (High) — Blocked 12d (no movement since Mon)

🇵🇱 PL holiday today: Labour Day (May 1)

🟡 Sprint health: 11 blocked (1 High), 2 unestimated in-flight, 2 days to sprint end (Grace).
📄 Full report: <link to last canvas>
```

The "still open" line is judgment-driven: surface 1 ticket (max 2) that's been stuck the longest with the highest priority. Don't list every standing blocker — that's what the canvas is for.

The country-holiday lines are bare — just the flag, country code, holiday name, and date. The `country` line's capacity reduction was already applied silently in the math; the `info_countries` line is decorative. Neither needs an inline explanation.

## Format — full skeleton

```
🟡 Sprint 12 — daily delta (Tue 21 May)
Since yesterday. 4 days left in sprint.

🚧 New blockers (2):
  ZETA-2700 (Frank, Medium) — Blocked 2h ago
  EPSILON-3900 (Grace, High) — Blocked 6h ago

🔄 Reopened (1):
  ALPHA-1009 (Bob) — Done → In Review

📈 Shipped (3):
  GAMMA-663, GAMMA-649 (Carol)
  DELTA-2722 (Eve)

🔁 Status moves (2):
  ETA-5592 (Dave) → IN_REVIEW
  GAMMA-551 (Carol) → In Progress

⚠️ Newly stuck >3d:
  ETA-5404 (Dave) — In Review 3d

📌 Still open:
  ZETA-2624 (High) — Blocked 12d (no movement)

📄 Full report: https://acme.slack.com/docs/.../<canvas-id>
```

This is the *maximum* shape. A real run will usually have several of these sections missing — skip them, don't render `None.`

## Quiet days

If every category is empty (no new blockers, no reopens, no shipped tickets, no status moves, no threshold crossings, no PTO updates, no late additions/removals) — still ship the message, but collapse to 2 lines:

```
🟢 Sprint 12 — daily delta (Wed 22 May)
No material changes in the last 24h. 3 days left. 📄 Full report: <link>
```

Reasoning: a silent ping confirms the routine ran. A missing ping makes the manager wonder if the skill broke.

## What the delta does NOT include

- Per-reportee capacity overview (that's in the canvas)
- Full concerns list per reportee (canvas)
- Risk deep-dive (canvas)
- Capacity breakdown tables (canvas)
- Per-reportee capacity verdict changes (would require persisted yesterday-state; removed when the snapshot was removed — the canvas still shows current verdicts)
- The 1:1 follow-up questions (canvas — also dropped per canvas.md)

The delta is a **diff**, not a re-render. Anything that wasn't a change in the last 24h stays in the canvas.

## Delivery

Post to `output_channel` with the manager `<@…>`-tagged on the first line. Use `manager.slack_id` from the inline config. Channel post is the default — DM-to-manager is the fallback that fires only when `output_channel` is `null`. `"terminal_only"` suppresses Slack entirely.

On a channel post the leading line is the `<@…>` tag plus the daily-delta title:

```
<@U001AAAAAA1> 🔴 Sprint 12 — daily delta (Fri 22 May)
Since yesterday. Most boards close Mon; Grace's EPSILON closes today.

🚧 New blockers (1):
  ...
```

The `<@…>` tag goes inline with the title — not on its own line — so the Slack notification preview includes "Sprint 12 — daily delta" rather than just a bare mention.

On a DM (legacy fallback when `output_channel` is `null`) the tag is dropped — the DM target is already the audience and a self-mention is noise.

The delta is always a Slack message (not a canvas) — it's meant to be ephemeral. Don't create a new canvas per day.

## No persisted state

This file used to describe a `references/snapshots/<slug>.json` lifecycle (read-at-start, write-at-end). That mechanism is **removed**. The skill must not create that file or any equivalent — the 24h Jira changelog window is the only delta input. If you find a leftover snapshot file in an old repo, delete it.

The Mon/Thu canvas runs also do not write any snapshot; their state is the rendered canvas itself, which Jira's data can always reconstruct.
