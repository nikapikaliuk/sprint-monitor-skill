# TODO

Captured ideas and follow-ups. Not commitments — pick up when relevant.

Each item is independent; tackle in any order. Status tag at the top of each says whether it's a restoration (existed in earlier versions) or a new build.

## Day-2-of-sprint carry-over report

*Status: to add. New feature, never built.*

A **separate routine cron entry** that fires on day 2 of each sprint (e.g. Tuesday morning if sprints start Monday) and emits a focused carry-over report. Intentionally a separate run, not a branch inside the daily routine — different content, cadence, and framing. The daily routine continues unchanged.


|         | Daily routine               | This new day-2 routine |
| ------- | --------------------------- | ---------------------- |
| Content | Concerns + active-right-now | Carry-over centric     |
| Cadence | Daily                       | Once per sprint        |
| Framing | Progress check              | Planning hygiene       |


**Content** — per reportee, count and list tickets rolled over from prior sprints. Detect via:

- `remaining_story_points < story_points` (the canonical inter-sprint carry-over indicator), OR
- checking each ticket's sprint-history changelog for prior-sprint entries.

Show per reportee: count of carry-over tickets, total SP carried, current statuses, and call out any still in To Do / Blocked (those carried over *without* progress, which is worse than the ones already In Progress).

**Aggregate view** — team-wide rollup at the top: *"6 of 22 sprint tickets are carry-overs (27%), 18 SP rolled in. Bob and Dan account for 5 of the 6."* — so the manager can see the trend without counting.

**Open design questions:**

1. Also surface as a banner in the daily routine output for the rest of the sprint, or is day-2 the only place it shows?
2. Timezone for "day 2 morning" — manager's, or team's majority TZ?
3. What's "day 2" if the sprint actually started on a Friday — calendar day 2 or working day 2? Probably working day 2 to match the existing PTO-check semantics.

## Per-person sprint summary (end-of-sprint retrospective)

*Status: idea. Not yet designed.*

A **separate routine** that fires once at sprint end (e.g. the Mon morning after sprint close) and produces a **per-reportee summary of what they actually did this sprint**. Distinct from the daily delta (24h scope) and the canvas (point-in-time snapshot) — this one is the "what got done over the last 10 working days, person by person" view a manager would want for 1:1s, perf reviews, sprint retros, and "what should I praise / what should I follow up on" conversations.

**Content per reportee — rough sketch:**

- **Shipped this sprint** — list of tickets moved to a Done-bucket status during the sprint window, with their SP totals. The headline number ("Bob shipped 4 tickets, 14 SP").
- **In-flight at sprint end** — tickets still open with status, time-in-status, and SP. Distinguishes "almost done" (In Review for 1d) from "stuck and carried over" (In Progress for 9d).
- **Carry-overs in / out** — tickets that came in from the previous sprint (rSP<SP at start) AND tickets pushed to the next sprint (sprint-changed events at the end).
- **Reopens** — tickets that bounced Done → not-Done at any point in the sprint window. Worth flagging — possibly a process or QA signal.
- **Mid-sprint scope changes** — SP edits (scope creep) + rSP edits (bookkeeping pattern, see [analysis.md → rSP modified mid-sprint](references/analysis.md#rsp-modified-mid-sprint)).
- **Capacity vs delivery** — capacity verdict from the start of sprint (Underworked / Normal / Borderline / Overbooked) vs what actually got delivered. "Was committed at 95h; shipped 40h of 95h committed; 3 carry-overs."
- **Long-stuck items** — tickets that sat in any in-flight status >7d (lower threshold than the daily delta's >14d since the window is wider).
- **1:1 talking points** — auto-generated questions the manager could open with. "Why did ALPHA-1234 sit in In Review for 8d?" "What did the rSP edits on ETA-5678 reflect?" "How was the Borderline load — sustainable next sprint?" Optional, judgment-driven.

**Aggregate view (team-wide rollup):**

- Total tickets shipped + total SP shipped, by reportee, sorted descending.
- Reopens count by reportee.
- Carry-over rate (% of sprint tickets that didn't ship).
- Capacity-vs-delivery deltas — who consistently over-commits vs under-commits across sprints (would require historical data, so probably a v2 feature).

**Output surface:** Slack Canvas (richer than a delta message — this is intended for 1:1 prep, so persistence + browseability matter). Same `output_channel` as the daily routine; same `<@manager.slack_id>` mention discipline.

**When to fire:** Mon morning after the sprint closes (so the data is settled — sprint just ended Sunday/Friday, all sprint-close Jira edits have happened). Configurable per group via cron in the routine setup.

**Open design questions:**

1. One canvas per reportee, or one canvas with sections per reportee? Per-reportee is easier to share individually (e.g. with the reportee themselves before a 1:1); single-canvas is easier to scan as a manager.
2. Compare against the prior sprint (delta-vs-delta) or just absolute numbers for this sprint? Comparison adds signal but requires historical data.
3. Include the 1:1 talking points by default, or behind a flag? They're judgment calls; some managers want them, others find them prescriptive.
4. How does this interact with reportees who are also chapter leads? They have a manager role too; do they get their own summary as a manager (chapter view) AND as an IC (reportee view)?

