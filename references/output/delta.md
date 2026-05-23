# Daily delta

The short daily companion to the [canvas.md](canvas.md) full report. 

The delta is a single Slack message DM'd to the manager — not a canvas. ~10–15 lines, scannable in 30 seconds.

## What makes this work: a 24h Jira changelog window

The delta is reconstructed live from Jira's changelog, not from a stored baseline. **No local state persists between runs.** Each category translates to a JQL query bounded by `... AFTER -24h` (or the equivalent changelog filter), and the answer is whatever Jira reports right now.

Why this works:

- Jira is already authoritative.
- The Atlassian MCP supports `status CHANGED AFTER -24h`, `status CHANGED TO Done AFTER -24h`, `status CHANGED FROM ("Done", "Closed", ...) AFTER -24h`, `sprint CHANGED AFTER -24h`, etc. — directly. One JQL per reportee per category.

### Window convention

- The default window is **-24h**: tickets whose status / sprint / scope changed in the last 24 calendar hours from the moment the delta runs.
- The window is hardcoded at 24h.

## Empty sections are skipped entirely

Every category below is rendered **only when it has ≥1 item**. If a delta run has zero new blockers, zero reopens, etc., those headings do not appear in the message at all — no `## 🔄 Reopens — None.` filler. The delta is a signal-only doc; placeholder lines for empty categories defeat its 30-second-scan purpose.

If *every* category is empty (genuinely quiet day), fall back to the collapsed two-line form in "Quiet days" below.

## Don't label sprint length

The title is `<traffic-light> <SprintName> — <@managerID>` — traffic light, sprint name (number included, exactly as Jira labels it, e.g. `ZETA Sprint 185`), em-dash, manager mention. No "daily delta" label, no date in parens, no "for Alice" — minify aggressively, the channel + cadence already imply what this post is. The skill's capacity math assumes a 2-week (10 working day) sprint; the actual length comes from Jira's `sprint.startDate → sprint.endDate` and should never be re-labelled in prose. If you need to reference time remaining, say "ends Friday" or "3 working days left" — not "this 1-week sprint" (almost always wrong; most teams here run 2-week sprints).

## Per-line formatting rules

These apply to every category. Render the message as Slack `mrkdwn` (the format `slack_send_message` accepts as the message body), so the standard markdown link syntax works.

- **Ticket keys are always links.** Render `[KEY](<jira_base_url>/browse/<KEY>)` — never bare text. The base URL comes from the inline config's `jira_base_url`. Example: `[ZETA-2700](https://acme.atlassian.net/browse/ZETA-2700)`.
- **Status names go in backticks, in their human form.** Every status reference is monospaced — e.g. `In Review`, `In Progress`, `Blocked`, `To Do`, `Ready for Verification`. **Normalize Jira's machine form to the human form** before rendering: `IN_REVIEW` → `In Review`, `READY_FOR_VERIFICATION` → `Ready for Verification`, etc. The Jira API sometimes returns the UPPER_SNAKE shape on changelog entries; always re-shape to Title Case With Spaces. Keeps status names readable and visually distinct from priority labels. Applies everywhere statuses surface — status moves, threshold-crossings, "was `<status>`" parentheticals, and the Stuck section.
- **Priorities are Slack emoji, never text.** Use the workspace's custom priority emoji inline next to the ticket key — `:priority-critical:`, `:priority-high:`, `:priority-medium:`, `:priority-low:`, `:priority-lowest:`. Position: **immediately after the ticket link, before the owner parens.** Example: `[ZETA-2700](…) :priority-high: (Frank) — ...`. **Never render the priority label as bracketed text** like `(High)` or `(Critical)`. If the workspace doesn't have a `:priority-<name>:` emoji set up, fall back to the bare priority word — but no brackets either way. If the ticket has no priority set, omit (no emoji, no text).
- **Owners are first names, in parens.** Drop the surname in delta lines (`(Bob)` not `(Bob Smith)`) — the canvas carries full names; delta is scan-speed. The owner parens contain only the owner; priority does not go inside them.
- **Sprint name in title is bold.** Title line is `<traffic-light>  *<SprintName>*  —  <@managerID>` (extra spaces around the em-dash to breathe visually). The asterisks make the sprint name pop against the rest of the surface.
- **Category subtitle line is bold with a `·` separator and count.** Format: `<emoji>  *<Category name>*  ·  <count>`. Examples: `🚧  *New blockers*  ·  2`, `🔁  *Status moves*  ·  4`, `✅  *Shipped*  ·  3`. Two spaces around the emoji and around the `·` — gives the line presence on its own.
- **Group by reportee with 3/8-space layered indent — collapse single-ticket reportees to inline.** Every delta category renders as a per-reportee group, but the shape depends on the per-person ticket count:
  - **Single-ticket reportee → inline form.** When a reportee has exactly one ticket in this category, put the ticket on the same line as the name: `   *Frank*: [KEY](…) :priority-…: — …`. 3-space indent before the bold name, `:` immediately after the closing `*`, single space, then the ticket payload. No count, no numbered prefix, no separate ticket line. Keeps a single-item entry to one line instead of three.
  - **Multi-ticket reportee → stacked, numbered tickets.** When a reportee has 2+ tickets, the name takes its own line — **no count appended**, just `   *Frank*`. Then each ticket on its own line, indented **8 spaces** with a `1.` / `2.` / `3.` numbered prefix: `        1. [KEY](…) :priority-…: — …`. The numbering gives an immediate at-a-glance sense of how many items the person has without needing a parenthetical count, and the indent depth still shows the hierarchy.
  - **Blank line between reportees** in the same category — applies in both forms — otherwise the bolded names run together and the per-person groups disappear.
  - **Literal spaces only — never tab characters.** Slack mrkdwn does not reliably render tabs as indent; a tab-indented message collapses flush-left and the layering disappears.
  - Owner parens `(Bob)` are dropped from individual ticket lines inside the group — the bolded name already carries the owner. Skip a reportee entirely if they have no items in that section.
  - **Sort reportees by ticket count descending** (the person with the most items in this category appears first), ties broken alphabetically. This applies to **every per-reportee section, including Stuck** — a person with 5 stuck tickets matters more at-a-glance than a person with 1, so they get the top slot. Single-ticket reportees fall below multi-ticket ones; among single-ticket reportees, ties broken alphabetically.
- **Horizontal rule between categories.** Insert a line of `━` characters between every two adjacent categories (and between the Today block and the first category). **Keep the rule short — ~12 characters of `━`** (e.g. `━━━━━━━━━━━━`). Longer rules wrap on mobile, which defeats the purpose. The rule sits on its own line with a blank line above and below.

## "Today" block — render at the very top

Before any delta category, render a small block of **today's situational context**: overall sprint health, then people-availability. These are the lines a manager wants to see *first* on opening the message — they reframe everything below ("oh, Eve is sick today, that explains the dip in Shipped"). All lines are conditional; skip any line that doesn't apply, and skip the whole block if every line is empty.

Order inside the block:

1. **Sprint health** — one-line headline numbers, **no leading emoji and no `Sprint health:` label** — the title's traffic-light already carries the icon, and repeating it on the next line is doubled-status noise. Phrase as bare numbers, e.g. `11 blocked (1 :priority-high:), 2 unestimated in-flight, 2 days to sprint end (Grace).` or `14 not done, sprint ends Sun May 24.` or `1 blocked :priority-low:` when there's a single blocker. **Priorities are rendered as Slack emoji** (`:priority-low:`, `:priority-high:`, …), never as bracketed text — write `(1 :priority-high:)`, not `(1 High)`; write `1 blocked :priority-low:`, not `1 blocked (Low)`. Same rule as the per-line formatting rules above; applies everywhere a priority surfaces in this line. Render only when the traffic light is non-green or any sprint is closing within 2 working days — on a steady 🟢 mid-sprint day, omit.
2. **Empty line** before the OOO / holiday lines that follow. Gives the at-a-glance scan a visual break between "what's tense" (sprint health) and "who's around" (PTO / holidays). Always insert the blank line when both blocks are present; if sprint health is omitted, no separator is needed.
3. **🌴 Out today** — anyone on PTO or sick *today*, regardless of when their absence started. One line, leading `🌴` (or `🤒` if everyone listed is sick), then `<Name> (<date1>–<date2>)` per person, comma-separated. Drop the `Out today:` label — the leaf emoji is the label. The parens carry the **PTO window**, not the absence type: e.g. `(May 22–26)` for a multi-day vacation, `(May 22)` for a single day. Use en-dashes for ranges, single date for one-day absences, and `(May 22 – Jun 1)` (spaces around the en-dash) only when the range crosses a month. Example: `🌴 Anna (May 22–26), Bob (May 22)`. If vacation and sick coexist on the same day, use per-person inline emoji: `🌴 Anna (May 22–26), 🤒 Bob (May 22)` — the leading emoji becomes whichever applies to the first listed person.
4. **🌴 PTO starts today / ends today** — window-edge transitions. Phrase as `🌴 Bob — vacation starts today (May 22–26, 3d)` or `🤒 Eve — sick today (1d)`. One line per person. Skip if already covered by "Out today" — no duplication.
5. **🇵🇱 Country holiday today** — one line per country (the group's `country` first, then each `info_countries` entry). Phrase as `🇵🇱 PL holiday today: Labour Day (May 1)`.
6. **🇮🇱 Country holiday tomorrow** — heads-up line, especially for IL erev half-days. Phrase as `🇮🇱 IL holiday tomorrow: Shavuot (Fri 22 May)`.

```
11 blocked (1 :priority-high:), 2 unestimated in-flight, 2 days to sprint end (Grace).

🌴 Bob (May 20–24), 🤒 Eve (May 22)
🇮🇱 IL holiday today: Shavuot (Fri 22 May)
```

### Where the data comes from

- **Holidays** — from [../holidays.md](../holidays.md) keyed on the inline config's `country` and `info_countries`.
- **PTO** — from two sources, merged in-memory for this run:
  1. The inline config's `reportees[].pto[]` arrays (manual entries the prompt author included).
  2. A **delta-time Google Calendar fetch**: for each reportee, query their primary calendar for events overlapping *today only* (a one-call-per-reportee fetch, much smaller than the canvas's full-sprint sync). Same event filter as the canvas sync — see [../pto.md](../pto.md) "Google Calendar OOO sync → Event filter".
  This is what makes ongoing mid-vacation PTO show up. If a reportee started vacation 3 days ago and is still out today, the inlined `pto[]` may not have the entry — but their calendar does. Without this fetch, the delta only sees PTO edges (today equals `start` or `end`), so a person mid-vacation goes silent. Bug, not feature — and the calendar fetch is cheap enough to pay daily.
  **Partial-day OOO is ignored.** A timed OOO event must have **≥6 hours of overlap with the 9:00–19:00 working window** to count toward the Today block. Short blocks (doctor's appointments, half-day errands) don't surface — they aren't real absences for availability purposes, and surfacing them creates phantom "out today" lines. Full rule in [../pto.md](../pto.md) "Event filter".
  If the Google Calendar MCP isn't authenticated, fall back to inlined-only and add a one-line note: *"⚠️ Calendar not authenticated — PTO only from inline config."* Don't hard-fail the delta over it (canvas does hard-fail because capacity math is wrong without PTO; delta tolerates the degradation because it's already an at-a-glance pulse).

## Delta categories

Six categories (plus 4b "removed from sprint"), listed in the order they should appear in the message when present. The ordering follows **attention first → informational → wins**: blockers / reopens / threshold-crossings up top so the manager hits the things that need their attention immediately, sprint-membership changes in the middle, then status moves and Shipped at the end so the message closes on movement and wins rather than red flags.

### 1. 🚧 New blockers

Tickets that transitioned **to** a Blocked status in the last 24h.

JQL: `assignee = "<accountId>" AND sprint in openSprints() AND status CHANGED TO "Blocked" AFTER -24h`

Group by reportee. One numbered line per ticket: key, priority emoji, time since the transition, and whether a comment was added at the time of the transition (pull last comment timestamp; if it's within ±10 minutes of the changelog entry, include the first line).

```
🚧  *New blockers*  ·  2

   *Frank*: [ZETA-2700](https://acme.atlassian.net/browse/ZETA-2700) :priority-medium: — `Blocked` 2h ago, no comment yet

   *Grace*: [EPSILON-3900](https://acme.atlassian.net/browse/EPSILON-3900) :priority-high: — `Blocked` 6h ago, _"waiting on infra team"_
```

### 2. 🔄 Reopens

Tickets whose status moved **from** a Done-bucket status to an indeterminate/new status in the last 24h.

JQL: `assignee = "<accountId>" AND sprint in openSprints() AND status CHANGED FROM ("Done", "Closed", "Resolved", "Verified", "Merged", "Won't Do", "Duplicate") AFTER -24h`

Reopened tickets are high-attention — surface the from/to and any explanatory comment.

```
🔄  *Reopened*  ·  1

   *Bob*: [ALPHA-1009](https://acme.atlassian.net/browse/ALPHA-1009) :priority-medium: — `Done` → `In Review`, _"Netlify failure"_
```

### 3. ⚠️ Threshold crossings (due-now / sprint-ending)

Surface tickets that **just crossed** a threshold the canvas would also flag — but only if the crossing happened in the last 24h.

- **Due date now ≤2d.** Ticket has `duedate >= today AND duedate <= today + 2 days AND duedate > today - 1 days` from yesterday's reference frame. In practice: a ticket whose `duedate` is today, tomorrow, or the day after, and was *not* in that window yesterday. Approximate by: `duedate <= today + 2d AND duedate >= today + 2d` (i.e., the duedate is exactly 2 days out — that's the day the threshold first fires). Acceptable to also include `duedate = today + 1d` if the spec is being strict about same-day clarity.
- **Sprint is about to end and there are concerns — fires from Wed of the second week onward, surfaces every reportee with not-done work.** This sub-block does **not** wait for "≤2 working days remaining" the way the other threshold-crossings do — it fires once the active sprint has reached **the Wednesday of its second working week** (i.e. a 2-week sprint's second-week Wed; on a 1-week sprint it never fires via this rule). From that day onward, every daily delta carries the sub-block until the sprint closes. **Fri-end ≈ Mon-end of the same close-out week** — see [../analysis.md](../analysis.md) "Practical Jira-MCP tips → Time math"; the Wed-of-week-2 trigger naturally absorbs both Fri-ending and Mon-following-weekend sprint shapes without singling either out.
  - **Population: every reportee with not-done work in the closing sprint, not just those with changes today.** Unlike every other delta category — which lists only reportees who had Jira activity in the last 24h — this sub-block lists **all reportees who still have non-Done tickets in the closing sprint**, even if they did nothing yesterday. That silent-but-loaded state is the exact thing the manager needs to see this late in the sprint. The rest of the delta (blockers, reopens, status moves, shipped, sprint adds/removes) continues to filter to "people who changed something" — only this sub-block ignores activity.
  - **Priority filter:** within this sub-block, render **Critical / Highest / P0 / Blocker / High / P1 / Medium / P2 only**. Drop Low, Lowest, Lowest-equivalent, and unprioritized tickets — a sprint closing with un-prioritized low work isn't urgent enough for the delta's top-attention block (it'll still appear in the canvas's risk breakdown). The rationale: this block is meant to drive end-of-sprint triage, and low-priority unfinished work is generally fine to push to the next sprint.
  - **Owner grouping** uses the same per-reportee shape as every other category (see Per-line formatting rules above). Within each person, sort tickets descending by priority then descending by SP.
  - **One ticket per line.** Never comma-group multiple keys on a line.

The Due-now sub-block follows the same per-reportee layered shape as the rest of the delta. The **Sprint-ends** sub-block uses a **per-sprint** header (sprint name + end date + count), then the same per-reportee body shape — including when only one reportee has items.

```
⚠️  *Due now ≤2d*  ·  1

   *Bob*: [ALPHA-1009](https://acme.atlassian.net/browse/ALPHA-1009) :priority-high: — due 23 May

━━━━━━━━━━━━

⚠️  *ZETA Sprint 185 ends Sun May 24*  ·  14 not done

   *Olga*
        1. [ZETA-3707](https://acme.atlassian.net/browse/ZETA-3707) :priority-medium: — `Development Ready` (4 SP)
        2. [ZETA-3625](https://acme.atlassian.net/browse/ZETA-3625) :priority-medium: — `Development Ready` (4 SP)
        3. [ZETA-3554](https://acme.atlassian.net/browse/ZETA-3554) :priority-medium: — `In Progress` (4 SP)
        4. [ZETA-3317](https://acme.atlassian.net/browse/ZETA-3317) :priority-medium: — `Merged` (4 SP)
        5. [ZETA-3546](https://acme.atlassian.net/browse/ZETA-3546) :priority-medium: — `In Review` (2 SP)
        6. [ZETA-3547](https://acme.atlassian.net/browse/ZETA-3547) :priority-medium: — `Backlog` (2 SP)

   *Illia*
        1. [ZETA-3557](https://acme.atlassian.net/browse/ZETA-3557) :priority-medium: — `Merged` (8 SP)
        2. [ZETA-3556](https://acme.atlassian.net/browse/ZETA-3556) :priority-medium: — `In Review` (8 SP)
        3. [ZETA-3772](https://acme.atlassian.net/browse/ZETA-3772) :priority-medium: — `To Do` (4 SP)
        4. [ZETA-3577](https://acme.atlassian.net/browse/ZETA-3577) :priority-medium: — `Merged` (2 SP)
        5. [ZETA-3771](https://acme.atlassian.net/browse/ZETA-3771) :priority-medium: — `Merged` (1 SP)

   *Eugene*
        1. [ZETA-3696](https://acme.atlassian.net/browse/ZETA-3696) :priority-medium: — `Development Ready` (8 SP)
        2. [ZETA-3700](https://acme.atlassian.net/browse/ZETA-3700) :priority-medium: — `Backlog` (4 SP)
        3. [ZETA-3695](https://acme.atlassian.net/browse/ZETA-3695) :priority-medium: — `In Progress` (4 SP)
```

### 4. 🆕 Late sprint additions

Tickets whose `sprint` field gained the active-sprint ID in the last 24h.

JQL: `assignee = "<accountId>" AND sprint in openSprints() AND sprint CHANGED AFTER -24h` — then filter to entries where the *added* sprint id matches one of the active sprint ids.

```
🆕  *Added to sprint*  ·  2

   *Bob*: [ALPHA-1112](https://acme.atlassian.net/browse/ALPHA-1112) :priority-medium: — added today, 4 SP, +20h to his load

   *Frank*: [ZETA-2710](https://acme.atlassian.net/browse/ZETA-2710) :priority-low: — 1 SP
```

Skip if the additions are sub-tasks of an in-progress parent — those are normal mid-sprint breakdown, not real additions.

### 4b. 🚮 Removed from sprint

Tickets whose `sprint` field lost an active-sprint ID in the last 24h. The mirror image of category 4 — the manager wants to see what walked out of the sprint, not just what came in.

JQL (per reportee, per current active sprint id): `assignee = "<accountId>" AND sprint CHANGED FROM <sprint_id> AFTER -24h`.

**Don't add `sprint in openSprints()`** to this query — the whole point is the ticket *left* the sprint, so it may no longer be in any open sprint. The `assignee` + `sprint CHANGED FROM <id>` pair is the correct selector.

Per result line: ticket key, owner, prior SP, status when removed, and where the ticket went (current sprint or backlog).

```
🚮  *Removed from sprint*  ·  2

   *Dave*: [ETA-5404](https://acme.atlassian.net/browse/ETA-5404) :priority-medium: — 4 SP, was `Blocked`, pushed to ETA Sprint 13

   *Bob*: [ALPHA-1010](https://acme.atlassian.net/browse/ALPHA-1010) :priority-medium: — 4 SP, was `Backlog`, back to backlog
```

Destination phrasing matches the canvas's Section 1b ([canvas.md](canvas.md) "Removed since last canvas"): "pushed to " / "back to backlog" / "moved to ".

Skip sub-task removals — same rationale as the additions category.

### 5. 🔁 Status moves (in-flight)

Tickets whose status changed in the last 24h **without** crossing into Done or Blocked (those have their own categories — 6 Shipped and 1 New blockers respectively). Captures the "PR pushed for review" / "QA picked it up" / "carry-over resumed" beats that fill out a healthy day.

JQL: `assignee = "<accountId>" AND sprint in openSprints() AND status CHANGED AFTER -24h` — then drop the rows already accounted for in categories 1 (Blocked) and 6 (Shipped). Reopens (category 2) are NOT dropped — show the reopen → in-flight chain so the manager sees where the ticket landed after being reopened.

**Show every transition in the window, not just the latest.** If a ticket bounced `To Do` → `In Progress` → `In Review` in 24h, render both jumps. The intermediate hop is signal (someone picked it up, then handed it off — both worth seeing). Use one bullet per transition, in chronological order.

**Human-readable status names.** Jira's changelog field sometimes returns the machine form (`IN_REVIEW`, `READY_FOR_VERIFICATION`). Always normalize to the human Title-Case form before rendering: `IN_REVIEW` → `In Review`, `READY_FOR_VERIFICATION` → `Ready for Verification`. The user shouldn't see snake-case.

**Add inline prose for context-rich transitions.** Short trailing phrase in parens when the move carries meaning beyond the status itself:

- `→ \`In Review` (no extra prose for the routine PR-up-for-review case — the status carries it)
- `→ \`In Review (PR opened)` when a linked PR was just opened
- `→ \`In Progress (carry-over resumed)` when a previously-active ticket re-entered In Progress mid-sprint
- `→ \`Verification (QA can pick up)` for the dev→QA handoff
- `→ \`To Do (deprioritized?)` when an in-flight ticket regressed to To Do — worth flagging as a question

The prose is *optional* — drop it if the move is unremarkable. The fact that it's in the Status moves list already implies "something happened"; only add prose when there's extra signal.

```
🔁  *Status moves*  ·  4

   *Bob*: [ALPHA-1009](https://acme.atlassian.net/browse/ALPHA-1009) :priority-medium: → `In Review` (PR opened)

   *Carol*: [GAMMA-551](https://acme.atlassian.net/browse/GAMMA-551) :priority-medium: → `In Progress` (carry-over resumed)

   *Dave*: [ETA-5592](https://acme.atlassian.net/browse/ETA-5592) :priority-medium: → `In Review`

   *Eve*: [DELTA-2810](https://acme.atlassian.net/browse/DELTA-2810) :priority-medium: → `Verification` (QA can pick up)
```

### 6. ✅ Shipped

Tickets that moved **to** a Done-bucket status in the last 24h.

JQL: `assignee = "<accountId>" AND sprint in openSprints() AND statusCategory CHANGED TO done AFTER -24h`

The wins. Closes the message on movement rather than red flags.

```
✅  *Shipped*  ·  4

   *Carol*
        1. [GAMMA-663](https://acme.atlassian.net/browse/GAMMA-663)
        2. [GAMMA-649](https://acme.atlassian.net/browse/GAMMA-649)

   *Eve*: [DELTA-2722](https://acme.atlassian.net/browse/DELTA-2722)

   *Grace*: [EPSILON-3848](https://acme.atlassian.net/browse/EPSILON-3848)
```

Priority emoji is **optional in Shipped** — once a ticket's done, its priority no longer matters for action and the line reads cleaner without it. Include only if the workspace convention is to keep emojis everywhere.

## Useful info — the "Stuck" section

A single persistent block: every ticket that's been **stuck in an in-flight or blocked status for more than 5 days**, regardless of priority and regardless of which reportee owns it. At that point it doesn't matter whether the ticket is High, Medium, or Low; nothing has moved, and the manager should see it.

- **Threshold:** `time_in_status > 5 calendar days` (no exception for priority, owner, sprint, or assignee absence).
- **Eligible statuses:** `In Progress`, `In Review`, `Blocked`, `Verification`, `Verified`, `Ready for Verification`. Skip `To Do`, `Backlog`, `Development Ready`, `Done`, `Merged`, `Won't Do`.
- **Source of truth:** the live Jira changelog.
- **No artificial cap.** Show all qualifying tickets. If a team has 12 tickets stuck >5d, the manager should see all 12 — the volume itself is signal.
- **Sort reportees by ticket count descending, ties alphabetical** — the person with the most stuck items appears first. Within each person, sort tickets by **days-stuck descending** (longest-stuck on top).
- **Drop the per-ticket owner parens** — the bolded name above carries the owner.

```
📌  *Stuck*  ·  4

   *Bob*
        1. [ZETA-2624](https://acme.atlassian.net/browse/ZETA-2624) :priority-high: — `Blocked` 18d
        2. [ZETA-2700](https://acme.atlassian.net/browse/ZETA-2700) :priority-medium: — `In Review` 15d

   *Carol*: [ALPHA-1102](https://acme.atlassian.net/browse/ALPHA-1102) :priority-low: — `In Review` 15d

   *Eve*: [DELTA-2810](https://acme.atlassian.net/browse/DELTA-2810) :priority-medium: — `In Progress` 17d
```

## Format — full skeleton

```
🔴  *ZETA Sprint 12*  —  <@U001AAAAAA1>

*11 blocked* (1 :priority-high:)  ·  2 unestimated in-flight  ·  2 days to sprint end (Grace)

🌴 Bob (May 20–24), 🤒 Eve (May 22)
🇮🇱 IL holiday today: Shavuot (Fri 22 May)

━━━━━━━━━━━━

🚧  *New blockers*  ·  2

   *Frank*: [ZETA-2700](https://acme.atlassian.net/browse/ZETA-2700) :priority-medium: — `Blocked` 2h ago

   *Grace*: [EPSILON-3900](https://acme.atlassian.net/browse/EPSILON-3900) :priority-high: — `Blocked` 6h ago

━━━━━━━━━━━━

🔄  *Reopened*  ·  1

   *Bob*: [ALPHA-1009](https://acme.atlassian.net/browse/ALPHA-1009) :priority-medium: — `Done` → `In Review`

━━━━━━━━━━━━

⚠️  *EPSILON Sprint 12 ends today*  ·  3 not done

   *Grace*
        1. [EPSILON-3792](https://acme.atlassian.net/browse/EPSILON-3792) :priority-high: — `In Review` (3 SP)
        2. [EPSILON-3848](https://acme.atlassian.net/browse/EPSILON-3848) :priority-high: — `In Review` (5 SP)
        3. [EPSILON-3837](https://acme.atlassian.net/browse/EPSILON-3837) :priority-medium: — `In Progress` (8 SP)

━━━━━━━━━━━━

🆕  *Added to sprint*  ·  1

   *Bob*: [ALPHA-1112](https://acme.atlassian.net/browse/ALPHA-1112) :priority-medium: — added today, 4 SP

━━━━━━━━━━━━

🔁  *Status moves*  ·  3

   *Carol*: [GAMMA-551](https://acme.atlassian.net/browse/GAMMA-551) :priority-medium: → `In Progress` (carry-over resumed)

   *Dave*: [ETA-5592](https://acme.atlassian.net/browse/ETA-5592) :priority-medium: → `In Review` (PR opened)

   *Eve*: [DELTA-2810](https://acme.atlassian.net/browse/DELTA-2810) :priority-medium: → `Verification` (QA can pick up)

━━━━━━━━━━━━

✅  *Shipped*  ·  3

   *Carol*
        1. [GAMMA-663](https://acme.atlassian.net/browse/GAMMA-663)
        2. [GAMMA-649](https://acme.atlassian.net/browse/GAMMA-649)

   *Eve*: [DELTA-2722](https://acme.atlassian.net/browse/DELTA-2722)

━━━━━━━━━━━━

📌  *Stuck*  ·  2

   *Bob*: [ZETA-2624](https://acme.atlassian.net/browse/ZETA-2624) :priority-high: — `Blocked` 18d

   *Eve*: [DELTA-2810](https://acme.atlassian.net/browse/DELTA-2810) :priority-medium: — `In Progress` 16d
```

This is the *maximum* shape. A real run will usually have several of these sections missing — skip them, don't render `None.` Section order:

1. **Today block at the top** — sprint health, Out today, holidays.
2. **Delta categories**, attention-first: blockers, reopens, threshold crossings, sprint additions/removals, status moves, Shipped.
3. **Stuck** — long-tail forgotten work (>5d in in-flight/blocked), at the bottom.

## Quiet days

If every category is empty (no new blockers, no reopens, no shipped tickets, no status moves, no threshold crossings, no late additions/removals) — still ship the message, but collapse to 2 lines:

```
🟢  *ZETA Sprint 12*  —  <@U001AAAAAA1>
No material changes in the last 24h.
```

Reasoning: a silent ping confirms the routine ran. A missing ping makes the manager wonder if the skill broke.

## What the delta does NOT include

- Per-reportee capacity overview (canvas)
- Full concerns list per reportee (canvas)
- Risk deep-dive (canvas)
- Capacity breakdown tables (canvas)
- Per-reportee capacity verdict changes (canvas)
- 1:1 follow-up questions (canvas)

The delta is a **diff**, not a re-render. Anything that wasn't a change in the last 24h stays in the canvas.

## Delivery

Post to `output_channel` with the manager `<@…>`-mentioned on the first line. Use `manager.slack_id` from the inline config. Channel post is the default — DM-to-manager is the fallback that fires only when `output_channel` is `null`. `"terminal_only"` suppresses Slack entirely.

**Slack mention rendering.** The `<@USERID>` syntax must resolve to the manager's display name (`@Alice Johnson`), not show the raw ID. Two requirements:

- **Send the message with `mrkdwn` parsing enabled** — pass `mrkdwn: true` (the default) to `slack_send_message`. Without it, Slack shows the literal `<@U001AAAAAA1>` text.
- The `<@USERID>` mention is the **only** name reference on the title line — no `for Alice` prefix, no `(Alice Johnson)` suffix. If a Slack surface ever fails to resolve the mention (notification preview, email digest), the raw `<@U…>` shows; that's acceptable since the channel + cadence already imply ownership. The manager's full display name lives in the canvas.

On a channel post the leading line is the minified title: traffic-light, sprint name, em-dash, manager mention. Nothing else:

```
🔴  *ZETA Sprint 12*  —  <@U001AAAAAA1>

*11 blocked* (1 :priority-high:)  ·  2 days to sprint end (Grace)

🤒 Eve (May 22)
🇮🇱 IL holiday today: Shavuot (Fri 22 May)

━━━━━━━━━━━━

🚧  *New blockers*  ·  1

   ...
```

The `<@USERID>` both pings and identifies — no second name reference in body text. Keep the title to two information units (sprint, manager).

The **Today** block sits between the title and the first delta category — sprint health + anything urgent about people-availability shows immediately under the title.

On a DM (fallback when `output_channel` is `null`) the tag is dropped — the DM target is already the audience and a self-mention is noise.

The delta is always a Slack message (not a canvas) — it's meant to be ephemeral. Don't create a new canvas per day.