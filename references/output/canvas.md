# Canvas output

The richest output surface — a Slack Canvas document holding the full sprint report (snapshot + capacity + concerns + risk deep-dive + capacity tables). Used when the manager wants a single browsable artifact instead of (or in addition to) a Slack message and terminal print.

When to use:
- Local-mode runs where the manager asks for a "full report", "deep dive", "long review", or explicitly says "canvas".
- Routine mode if `output_channel` is set to a canvas-capable destination (not v1 default; v1 routine still uses Slack messages).

Delivery: create the canvas via `slack_create_canvas`, then post the link to the group's `output_channel` (default) with the manager `<@…>`-tagged. DM-to-manager is the fallback when `output_channel` is `null`. The channel post / DM is a short message with the link — the canvas itself carries the content.

## Canvas-flavored markdown notes

Slack canvas markdown is close to standard CommonMark but with quirks worth knowing:

- Links use standard markdown `[text](url)`. Not the `<url|text>` mrkdwn form.
- User mentions: `![](@U12345)`. Channel mentions: `![](#C12345)`.
- Nested lists work with 2-space indentation under the parent bullet.
- Tables render. Hours/notes in extra columns are fine.
- Bullet markers are `-` or `*` (both render the same).
- Status names inside backticks render as monospace — keeps them visually distinct from priority labels.
- Do **not** include the title inside the content; pass it via the `title` argument when creating the canvas.

## Title

`<traffic-light emoji> Sprint <N> — <DD Month>` — e.g. `🟡 Sprint 12 — 20 May`. The traffic-light emoji reflects overall sprint health (🟢/🟡/🔴) per the Step 7 rule in [analysis.md](../analysis.md). Date format is day-month, no year. Don't include "full report" or other qualifiers — the canvas is always the full thing.

## Section list (in order)

1. **Sprint health snapshot** (3 lines max)
1b. **Removed since last canvas** — *conditional*, only when a prior canvas was found in Slack (see [../../SKILL.md](../../SKILL.md) "Finding the prior canvas") and tickets were removed from the active sprint(s) in the window between the prior canvas's posting time and now.
2. **Capacity overview** (one line per reportee, sorted, colored)
3. **Out this sprint** — *conditional*, personal PTO only (vacation / sick). See [../pto.md](../pto.md) "Display" for the format.
4. **Group holidays this sprint** — *conditional*, info block listing country holidays in the sprint window (group's `country` first, then `info_countries`).
5. **Concerns** (nested per-reportee list, filtered by signal type)
6. **Risk deep-dive** (top 5 risk tickets)
7. **Capacity breakdown** (detailed per-ticket tables)

No "Active right now" section. No "What I'd ask in 1:1s" section. No "Generated for X" preamble. No "Notes on this run" trailer. Keep the canvas tight — the manager skims top-to-bottom and the most signal-dense parts are first.

## 1. Sprint health snapshot

Three lines (no leading bullet marker), each starting with a category emoji + bold label + count:

```
🚧 **Blocked: <N> tickets** — <K> High (<links>), <M> Medium, <L> Low.
❓ **Unestimated: <N> tickets** — <K> in-flight `Blocked` (<links>).
✂️ **Need to be splitted: <N> tickets** — <link1> (<owner>, `<status>`) and <link2> (<owner>, `<status>`, half-done).
```

Rules:
- **Blocked line.** Always lead with High-priority blocked tickets by ticket key + link. Drop the distribution (per-reportee counts) — the Capacity overview already shows who has load issues. If 0 High blocked, just say "Blocked: N tickets — all Medium/Low" or similar.
- **Unestimated line.** Surface only the **acute** subset: in-flight unestimated tickets (in any of the five in-flight statuses + Blocked). These can't be capacity-sized, which is why they matter. Don't dump the long list of pre-work unestimated (Backlog/To Do) — that's refinement debt, not actionable here.
- **"Need to be splitted" line.** All 8-SP tickets, regardless of status. The label uses "splitted" (manager preference; do not "correct" to "split"). For each: ticket link, owner, current status, brief context if half-done.

If a category has 0 tickets, drop the entire line (don't write "Blocked: 0 tickets").

## 1b. Removed since last canvas *(conditional)*

Renders only when **both** conditions hold:

- A prior canvas was found in Slack for this manager (see [../../SKILL.md](../../SKILL.md) "Finding the prior canvas"), AND
- At least one ticket assigned to a current reportee was removed from one of the active sprint(s) between the prior canvas's posting time (`prior_canvas_posted_at`) and now.

Otherwise the entire section is omitted — no "Nothing removed" placeholder.

The window bound — `prior_canvas_posted_at` — is the `ts` of the Slack message that linked the prior canvas (or, equivalently, the canvas's `created`/`updated` timestamp). It replaces the file-based `last_canvas.run_at` from earlier spec revisions.

### Detection

Per reportee, per current active sprint id:

```
JQL: assignee = "<accountId>" AND sprint CHANGED FROM <prior_sprint_id> AFTER "<prior_canvas_posted_at>"
```

**Don't filter by `sprint in openSprints()`** — the whole point is the ticket *left* the sprint, so it may not be in any open sprint at all anymore. The `assignee` + `sprint CHANGED FROM` pair is what we need.

For each result, also pull (in the same fetch):
- `summary`, current `status`, current `priority`, the story-points custom field (use `jira_field_ids.story_points` from the inline config — SP at the time of fetch, which equals "SP when removed" for our purposes — Jira doesn't undo SP when a sprint changes).
- The current sprint membership (use `jira_field_ids.sprint`): tells you *where* the ticket went — back to backlog (no active sprint), into a different team's sprint, or pushed to the next sprint of the same board.

### Format

Section heading + a nested list, grouped by reportee, sorted reportees by count of removals descending:

```
## Removed since last canvas

- **Dave Wilson** (3 removed)
  - [ETA-5404](https://acme.atlassian.net/browse/ETA-5404) (4 SP, was `Blocked`) — pushed to ETA Sprint 13
  - [ETA-5366](https://acme.atlassian.net/browse/ETA-5366) (2 SP, was `Blocked`) — pushed to ETA Sprint 13
  - [ETA-5362](https://acme.atlassian.net/browse/ETA-5362) (2 SP, was `Blocked`) — back to backlog
- **Bob Smith** (1 removed)
  - [ALPHA-1010](https://acme.atlassian.net/browse/ALPHA-1010) (4 SP, was `Backlog`) — pushed to ALPHA Sprint 13
```

Per-line shape: `[KEY] (<SP> SP, was <status>) — <destination>`.

**Destination phrasing:**
- "pushed to <Sprint Name>" — if the ticket now has a different sprint (next sprint of same board or another board).
- "back to backlog" — if the ticket has no active sprint membership now.
- "moved to <Sprint Name>" — if the ticket landed in a sprint that closed since (rare, but if Jira shows it).

If a reportee has 6+ removals, collapse to a one-line summary instead of listing every key: `- **Bob** (8 removed) — see Jira; cluster pattern suggests sprint over-loaded at planning.`

If only one reportee has removals, drop the per-reportee grouping and just list the tickets flat.

### Why this section exists

Sprint removals are a discipline signal worth seeing explicitly — they're the most likely cause when the Capacity overview's `prev_% → new_%` line drops without an obvious "lots of stuff shipped" story. Surfacing the keys lets the manager (a) sanity-check that the removals were intentional, and (b) chase any that quietly walked out of the sprint without a decision.

Pairs with the delta's "🚮 Removed from sprint" category ([../output/delta.md](../output/delta.md) category 7b) — same signal, 24h scope instead of inter-canvas scope.

## 2. Capacity overview

One paragraph line per reportee. **Sort by load: Overbooked → Borderline → Normal → Underworked.** Within each tier, sort by percentage descending.

### Two line shapes — first canvas of the sprint vs. subsequent canvases

The Capacity overview line has two shapes depending on whether a prior canvas exists for the same sprint(s):

**Shape A — first canvas of the sprint (no prior to compare against):** absolute percentage only.

```
<icon> **<Full Name>** · <sprints> · <Verdict> (<%>)
```

Example: *🔴 **Dave Wilson** · ETA · Overbooked (263%)*.

Use Shape A when:
- No prior canvas was found for this manager in `output_channel` (see "Sourcing the prior percentage" below), OR
- The prior canvas was found but its sprint window doesn't overlap the current active sprint(s) — sprint roll-over since last canvas — start fresh, OR
- The prior canvas can't be read for any reason (deleted, permission error, parse failure) — fall back to Shape A silently.

**Shape B — subsequent canvases:** percentage diff + short reason, but only render the diff when the percentage actually moved.

```
<icon> **<Full Name>** · <sprints> · <prev_%> → <new_%> · <reason>
```

Example: *🔴 **Dave Wilson** · ETA · 263% → 115% · 3 tickets moved out of sprint; SP changed on 2 tickets*.

Use Shape B when a prior canvas for the same sprint exists and is readable. **If the percentage is unchanged**, drop the diff and reason entirely — collapse back to Shape A (`<Verdict> (<%>)`). No `(no change)` filler.

The verdict label (Overbooked / Borderline / Normal / Underworked) is implied by the icon and dropped from the line text under Shape B — the diff carries the load story. If a manager wants the verdict label spelled out, it's still in the Capacity breakdown section.

### Sourcing the prior percentage

The skill needs to know what the previous canvas reported. There's no file-based state — the prior canvas is recovered from Slack at run time:

1. **Search `output_channel` for the prior canvas message.** Look for the most recent message authored by the bot that contains BOTH `<@<manager.slack_id>>` (the disambiguator — see [../../SKILL.md](../../SKILL.md) "Finding the prior canvas") AND a canvas URL. The `<@…>` mention is what pins the canvas to **this manager** — never reuse another manager's canvas even if it sits more recently in the same channel.
2. **Read the canvas** via `slack_read_canvas` on the canvas id extracted from the message body.
3. **Parse the per-reportee percentages** out of the rendered Capacity overview section and key them by reportee name. This becomes the `prior_pct[<name>]` map.
4. **Capture the message's `ts`** (the Slack message timestamp) — this is `prior_canvas_posted_at`, used as the lower bound on the reason-derivation JQLs (see "Deriving the reason text" below) and on the Section 1b "Removed since last canvas" detection JQL.
5. **Capture the active-sprint context** the prior canvas covered. The simplest source is the prior canvas's Capacity overview lines, which name each reportee's sprint(s) (e.g. "ALPHA + BETA"). If a current reportee's active sprint set has no overlap with what the prior canvas showed, that reportee gets Shape A (sprint rolled over for them) — the percentage diff would be meaningless.

The reportee-name → percentage map (plus `prior_canvas_posted_at` and the canvas id for the in-place update) is everything the skill extracts. Don't try to reconstruct prior hour totals or bucket counts from canvas text — they aren't needed for the diff, and the text isn't guaranteed to be parseable past v1.

This is intentionally a lightweight "two canvases ago doesn't matter" memory — the diff is always to the most recent canvas, never to a baseline. If the lookup fails for any reason, fall back to Shape A silently and create a fresh canvas.

### Deriving the reason text

Run these per-reportee queries to explain why the percentage moved, scoped to the window between `prior_canvas_posted_at` and now:

| Signal | JQL / source | Hour-delta interpretation |
|---|---|---|
| Tickets removed from sprint | `assignee = "<id>" AND sprint CHANGED FROM <active_id> AFTER "<prior_canvas_posted_at>"` | Each removed ticket subtracts its prior per-ticket hours. |
| Tickets added to sprint | `assignee = "<id>" AND sprint CHANGED TO <active_id> AFTER "<prior_canvas_posted_at>"` | Each added ticket adds its current per-ticket hours. |
| Story-point changes | `assignee = "<id>" AND sprint in openSprints() AND "Story Points" CHANGED AFTER "<prior_canvas_posted_at>"` (use `jira_field_ids.story_points` if Jira aliases don't match) | Map old → new SP via SP_HOURS, subtract delta. |
| rSP drained on Done | from the per-ticket data already pulled — tickets where `statusCategory == Done` AND `remaining_story_points == 0` AND the rSP changelog entry's `created` is after `prior_canvas_posted_at` | Per ticket: prior contribution (SP_HOURS[prior_rSP]) becomes 0.5h. Often the dominant cause of large drops, especially mid-to-late sprint. |
| Status moved to/from in-flight | already covered by the per-ticket fetch — no extra call | Doesn't change `sprint_hours` directly (the hour mapping is SP-based, not status-based), but worth mentioning when a big chunk of work shipped. |

Compose the reason from the top 1–2 contributors by absolute hour impact. **Keep it under ~12 words.** Example phrasings:

- *3 tickets moved out of sprint; SP changed on 2 tickets*
- *rSP drained on 5 completed tickets (-78h)*
- *2 tickets added late; 8-SP ticket re-estimated to 4*
- *Nothing material moved — verdict tier shifted on threshold rounding*

If the percentage moved but no signal explains it (rare — usually means a non-SP field changed), write *unattributed shift*. Don't fabricate causes.

### Status icon mapping

- 🔴 Overbooked
- 🟡 Borderline
- 🟢 Normal
- 🔵 Underworked

The icon **always reflects the current verdict**, even under Shape B. A reportee who crossed from Overbooked to Borderline shows the 🟡 icon plus the `263% → 115%` diff — icon and prev_% don't need to agree.

### Sprint name format

Drop the sprint-number suffix when sprints are consistent across the group. E.g. show "ALPHA + BETA" not "ALPHA 12 + BETA 12" if all are in sprint 12. If a reportee has multiple boards on different sprint numbers, restore the suffix to disambiguate.

Don't include ticket count, status breakdown, or hours commitment in this section — they live in Capacity breakdown at the bottom. The percentage (or diff) is the legibility signal.

### Sprint-ends-soon note

If a reportee's sprint ends very soon (≤2 working days) and they're not Normal/Underworked, append `· sprint ends <Day> (<Nd>)` to their line. Drop the note once the cluster of reportees with the same shortcut is documented in Concerns.

**Fri-end ≈ Mon-end of the same close-out week.** Don't flag a Friday-ending sprint with extra urgency relative to the same-week Monday-ending sprints — see [../analysis.md](../analysis.md) "Practical Jira-MCP tips → Time math" for the rule. If every other reportee's sprint ends the following Mon and one reportee's ends Fri, either all of them get the sprint-ends-soon note or none do — the weekend collapses the gap.

**PTO suffix** — if a reportee has PTO, sick leave, or public holidays inside the current sprint window, append a PTO fragment to their line:

```
🟢 **Bob Smith** · ALPHA + BETA · Normal (94%) · 🌴 PTO May 22–26 (3d)
🤒 **Eve Martinez** · DELTA + ZETA · Underworked (40%) · sick today
```

Emoji: 🌴 vacation, 🤒 sick. Comma-separate multiple entries. Only show entries with at least one day inside the sprint window. The verdict percentage is computed against the **scaled** `capacity_hours` (see [../pto.md](../pto.md) "Capacity math"). See [../pto.md](../pto.md) "Display" for full rules.

## 3. Out this sprint *(conditional)*

Only rendered when at least one reportee has **personal** PTO (vacation or sick) inside the current sprint window. Public holidays do NOT appear here — they have a separate info block (section 4). If no personal PTO this sprint, omit this section.

Format:

```
## Out this sprint

🌴 Bob — May 22, 25, 26 (3d vacation)
🤒 Eve — May 20 (sick today)
```

Order: anyone out *today* first, then upcoming dates, then past dates inside the current sprint.

See [../pto.md](../pto.md) "Display" for the full rules and edge cases.

## 4. Group holidays this sprint *(conditional)*

Info block listing country holidays whose dates fall inside the current sprint window. Renders only when at least one holiday from the group's `country` or any `info_countries` is in range; otherwise omit the section entirely.

Format:

```
## Group holidays this sprint

🇵🇱 Poland — May 24 (Pentecost Sunday, weekend), Jun 4 (Corpus Christi, Thu)
🇮🇱 Israel — May 22 (Shavuot, Fri)
```

Rules:
- One line per country. The group's `country` line goes **first**, then each `info_countries` entry in array order.
- List every holiday from that country's list in [../holidays.md](../holidays.md) whose date is inside the sprint window — past, today, and upcoming. Per holiday: `<MMM DD> (<name>, <weekday>)`. Comma-separate multiple.
- This is **info only**. The capacity math already applied the `country` reductions silently in Step 6 of [../analysis.md](../analysis.md) — this section doesn't restate the math.
- `info_countries` holidays do **not** affect capacity for anyone in this group — they're for cross-team awareness (e.g. the Polish team knowing Israeli colleagues are off).
- Skip a country's line if it has no holidays in the sprint window. Skip the whole section if all countries are empty.

## 5. Concerns

Renamed from "Concerns per reportee" — kept short. Format is a nested bulleted list:

```
- **<Reportee Name>**
  - <concern 1>
  - <concern 2>
- **<Reportee Name>**
  - <concern>
```

Names are first-level bold list items. Concerns are nested second-level items. Reportees with no qualifying concerns are **omitted entirely** — no empty name line.

### Which concerns to include

Only the following signal categories surface here. Everything else lives in Sprint health snapshot or Capacity breakdown:

- **Upcoming due dates.** Ticket has `duedate` within 7 calendar days. Always show.
- **Not-started, High-priority, close to sprint end.** Ticket is in `To Do` or `Backlog`, priority matches the High-priority taxonomy (Critical/Blocker/Highest/P0/P1/High), and the sprint ends within 2 working days. Not-started Medium/Low tickets don't qualify even if the sprint is ending — capacity overview already flags the load.
- **Reopened.** `Done` → not-Done transition inside the sprint window.
- **8 SP tickets** (the "should be split" anomaly). Always show. Lead with ✂️.
- **Stuck in status >3 days** — but **exclude** `To Do`, `Development Ready`, `Merged`, and `Done`. So In Progress, In Review, Blocked, Verification, Verified are all eligible for the stuck signal.

### What NOT to include

- **"Just Blocked" tickets** without any other signal. The Sprint health snapshot already lists the blocked count and surfaces the High-priority one. Don't duplicate them here. A blocked ticket only surfaces here if it *also* hits one of the categories above (e.g. stuck in Blocked >3d — which it usually does if it's been blocked a while).
- Scope creep, comment-velocity, PR signals, etc. — not in this section in v1 (move to risk deep-dive if they apply to a top-5 risk ticket).

### Within-reportee ordering

Sort concerns by urgency:
1. Upcoming due date (most urgent first)
2. Not-started High close to sprint end
3. Reopened
4. Stuck in status (longest stuck first)
5. ✂️ 8 SP last (it's a process flag, not time-critical)

### Phrasing rules

- One concern per line. No multi-line bullets.
- Lead with the ticket key as a markdown link.
- Use compact phrasing: `[ALPHA-1009] due 25 May (4d), stuck in In Progress 5.5d`. Status names in backticks.
- High priority: surface `(High)` inline. Don't surface Medium/Low — they're the default.
- For 8-SP concerns: lead with ✂️ emoji, format as `✂️ [TICKET] (8 SP, status) — should be split`.
- For the highest-attention item across the team (typically the High + blocked + no estimate one): lead with ❗ emoji.

## 6. Risk deep-dive (top 5)

For each ticket classified as **at-risk** or **blocked** in [../analysis.md](../analysis.md) Step 4, give the full context needed to act on it. Lives between Concerns and Capacity breakdown.

### Selection and ordering

- Pull all at-risk and blocked tickets from across the group's reportees.
- **Order by:** (1) priority (highest first, per the rank order in [../analysis.md](../analysis.md) Step 5 "Priority taxonomy" — Critical/Blocker/Highest/P0 > High/P1 > Medium/P2 > Low/P3 > Lowest/P4 > unmatched), (2) bucket (blocked before at-risk within same priority — blocked tickets have an explicit dependency to chase), (3) days-in-current-status (longest first).
- **Cap at 5 tickets total** across the whole group. If there are >5 candidates, take the top 5 and add a trailing line: *"+ N more at-risk/blocked tickets — see Concerns section above for the list."*

The cap matters: this section is heavy per entry, and a deep-dive on 12 tickets is unreadable. If the manager has more than 5 risks they can act on, they have a different problem.

### Structure per ticket

```
### [TICKET-KEY] — Summary (Owner, Priority, bucket)

**Status timeline (this sprint):**

- MMM DD  HH:MM  from-status → to-status
- MMM DD  HH:MM  from-status → to-status

PR: <state and signal line — or "no PR linked">

Last comments:

- MMM DD  author: "verbatim, truncated to 1 line max"
- MMM DD  author: "verbatim"
- MMM DD  author: "verbatim"

**Recommended:** <one-sentence suggested action>
```

### Per-field rules

**Status timeline:**
- Only transitions inside the current sprint window (`sprint.startDate` → now).
- One line per transition. Time format: `MMM DD HH:MM` (local time of the Jira instance; don't normalize).
- If there are no transitions this sprint (ticket has sat in the same status the entire sprint), write: *"In <status> since sprint start (<N>d)."*
- Cap at 6 transitions. If more, show first 2 + *"… <N> more transitions …"* + last 2.

**PR line:**
- If linked PR(s) exist: `PR: #<number> <state>, <age> open, <last review summary>`. Example: *"PR: #4421 open, 5d open, no review activity."*
- Multiple PRs on one ticket: list them on separate lines under `PRs:`.
- No linked PR: *"PR: no PR linked."* — surface even when not actionable, because absence often IS the signal for In-Progress tickets.
- Skip the PR line entirely if PR/dev-info wasn't fetched for this run (not all routine runs pay the dev-info cost).

**Last comments:**
- Pull the last 3 comments from the data already collected in [../analysis.md](../analysis.md) Step 3.
- Truncate each to ~120 chars on a single line — append `…` if cut. No multi-line quotes.
- Comment format: date + author + verbatim text. Don't paraphrase.
- If the ticket has fewer than 3 comments, show what exists. If zero comments this sprint, write: *"No comments this sprint."*

**Recommended:**
- One sentence. The manager-facing action: who to ping, what to ask, what to consider trimming. Tied to the *specific* signals on the ticket, not generic ("check with the assignee").
- Phrase actively: *"Bob needs a 30-min pairing with someone on the auth team."* not *"Could be worth pairing."*
- If the signals don't suggest a clear action, write: *"Watch for changes; nothing obvious to do yet."*

## 7. Capacity breakdown — at the bottom of the canvas

The detailed per-ticket math. Lives at the **end** of the canvas because most managers won't drill into it during a standup-style read; the snapshot + capacity overview already give them the verdict.

Intro paragraph above the per-reportee blocks:
```
Hours use literal `remaining_story_points` (rSP) — that's what's actually committed to *this* sprint. `SP` column shows the ticket's absolute size for context.

**Verdict thresholds:** Underworked <65h · Normal 65–80h · Borderline 80–100h (or 4+ × 4-SP tickets) · Overbooked >100h (or 5+ × 4-SP).
```

Per reportee:
- `### <Full Name> — <Verdict> (<total_h>h / <budget>h budget)` — budget is 80 by default, scaled down for PTO/holidays per [../pto.md](../pto.md). Append a PTO suffix when applicable: `### Bob Smith — Normal (60h / 64h budget) · 🌴 2d PTO`.
- A markdown table with columns: `Ticket | Sprint | SP | rSP | Status | Notes`
  - `Notes` column is **optional** — only include it if there's at least one ticket needing a per-ticket note (carry-over, ✂️ 8 SP, stuck, etc.). If no notes, drop the column.
  - **Do not include an `Hours` column.** Hours are computed under the hood (from SP/rSP via the Step 6 mapping) but managers don't need them in the table; the total is enough.
- One-line summary after the table: `Total: <Xh> (<Y%>). <interpretation>`.

**For reportees with too many tickets to table** (e.g. 20+ tickets, common on triage-heavy teams): replace the table with a status-breakdown summary + bucket-size summary, then the total. Format:

```
Sprint: <Sprint Name>. <N> tickets — too many to table here.

- Breakdown: <N> Done (<X SP>), <M> In Progress (<Y SP>), <K> Blocked (<Z SP>), ...
- Bucket counts (by SP): N × 4-SP, M × 2-SP, K × 1-SP, ... (+ unestimated if any)

Total: <Xh> (<Y%>). <interpretation>
```

The cutoff is judgmental — if rendering the table would push the canvas past ~200 lines, switch to the summary form for that reportee.

## Surfaces in v1

**Canvas is currently the only rendered output** — both local-mode and routine-mode runs produce a canvas and post the link to `output_channel` (or DM the manager when `output_channel` is `null`).

For the canvas + channel-post combo: after creating the canvas, send a one-line message to `output_channel` with `<@manager.slack_id>` leading the body, then the canvas URL, then a TL;DR (1–2 sentence summary of the most urgent thing).

## Example channel post accompanying a canvas

```
<@U001AAAAAA1>
🟡 *Sprint Monitor — Sprint 12 (20 May)* — full report
<canvas URL>

TL;DR: 1 High blocked (ZETA-2624 — 12d, no estimate); Grace's sprint ends in 2d with 6/7 not done. Canvas has the full breakdown.
```

The `<@…>` tag at the top is the manager — without it the post sits in the channel without pinging anyone. The post is one short paragraph + the link. Manager clicks through to the canvas for everything else. (When falling back to DM-to-manager because `output_channel` is null, drop the `<@…>` tag — the DM target is already the audience.)

## Updating an existing canvas

The canonical source for "which canvas to update" is the canvas id discovered via the Slack lookup at the start of the run (see [../../SKILL.md](../../SKILL.md) "Finding the prior canvas" and "Sourcing the prior percentage" above). If the lookup found a canvas and the prior sprint context overlaps the current active set, call `slack_update_canvas` with `action=replace` (no `section_id`) on that id to rewrite the whole canvas in one shot. If the lookup found nothing, or the sprint rolled over, create a fresh canvas via `slack_create_canvas` instead.

Section-level replaces via `section_id` have proven unreliable for nested-list edits (they sometimes insert rather than replace) — prefer full canvas replaces for any structural change. Single-line text tweaks via `section_id` are fine.

## Don't bake in stale section content

The canvas content the skill generates should be **derived from the data at run time**, not copied from prior canvases. If a prior canvas had something the current data doesn't support (e.g. a section was manually edited), regeneration drops it. The manager can re-add manual content after a refresh if they want it preserved — there's no auto-merge.
