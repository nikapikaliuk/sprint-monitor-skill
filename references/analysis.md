# Sprint analysis

The job: given the reportee list from config, find their active-sprint work in Jira, classify each ticket, surface signals that warrant manager attention, and hand off to [output/canvas.md](output/canvas.md) for rendering.

This is the part where the skill earns its keep. The user is a busy manager — every minute of analysis time has to pay for itself in clearer signal.

**Process the reportees one at a time.** They're not necessarily on the same team, board, or sprint, so treat each as an independent mini-analysis. The output report aggregates across them at the end, but the data collection is per-person.

## Step 0 — Resolve Jira custom field IDs from config

Custom field IDs for Story Points, Remaining Story Points, Sprint, and the Flagged/Impediment marker **vary per Jira instance**. Read them from `jira_field_ids` in the inline config (see [../SKILL.md](../SKILL.md) "Config schema") — do **not** hardcode `customfield_10016` / `customfield_10026` / `customfield_10002` as defaults. Those are wrong for this instance and produced a "no estimates" false positive on the first run.

The expectation is that the routine template captured these IDs at setup time and inlines them on every run. If a specific field ID is `null` in the inline config, or the `jira_field_ids` block is missing entirely, the run can fall back to live discovery for this run only:

| Field | Discovery procedure |
|---|---|
| Story Points | JQL probe: `"Story Points" is not EMPTY` against a known ticket the user said has SP set. Then fetch that ticket with `fields=["*all"]` and find which `customfield_*` holds the matching numeric value. Cross-check via `"Story Points" = <expected value>` to confirm — the field alias resolves to one specific custom field ID, and any other `customfield_*` with the same numeric value is unrelated. |
| Remaining Story Points | Same probe with `"Remaining Story Points"` (also try `"Story Points Remaining"`, `"Story point estimate"` as common aliases). Two custom fields holding the *same* numeric value on a known ticket usually means one is SP and the other is rSP — distinguish by checking a ticket where the values must differ (a Done ticket where rSP has burned down, or a multi-sprint carry-over). |
| Sprint | Almost always `customfield_10020` or `customfield_10021` — the one whose value is an array of sprint objects (each with `id`/`name`/`state`/`startDate`/`endDate`). |
| Flagged | The Jira-built-in flag. Inspect a ticket the user says is flagged for the custom field with value `["Impediment"]`. If no flagged ticket is available, leave this as `null` — the Step 5 "Flagged" signal simply doesn't fire until the ID is discovered. |

Discovered IDs are **used for the current run only** — there's no file write-back. If the manager wants future runs to skip rediscovery, the routine's inlined config needs to be updated externally (e.g. the manager re-runs setup and re-commits the routine prompt with the discovered IDs filled in).

**Gotcha on rSP semantics.** On this team rSP is handled inconsistently — some reportees drain it to 0 on completion (burndown), others leave it at the carry-over allocation. There's no team-wide pattern to detect. Capacity math (Step 6) now picks the canonical value **per ticket**: rSP by default, fall back to SP only when Done + rSP=0. See [Step 6](#step-6--assess-per-reportee-capacity) "Story-point field to use — per ticket" for the full rule.

**Gotcha on result limits.** The Atlassian MCP's `searchJiraIssuesUsingJql` silently caps multi-assignee queries at ~10 results (`totalCount` matches the cap, no pagination signal). **Always query per reportee** with `assignee = "<accountId>" AND sprint in openSprints()` — never `assignee in (..., ..., ...)`. The per-reportee form returns the full set reliably.

## Step 1 — Load reportees

The reportee list lives under `reportees[]` in the inline config (each entry is `{name, email, jira_account_id, pto}`). The skill receives the full config block as input per [../SKILL.md](../SKILL.md) "Config" — by the time you're here, it's already parsed. Don't re-look-up names against Jira at run time — the `jira_account_id` is the key you'll use for every Jira query in the steps below, and it's already canonical.

If the inline config has no reportees (`reportees[]` is empty), emit a no-op report (*"No reportees configured for group `<slug>`."*) and stop. This is not an error — it's the natural result of a routine being set up for a manager with no direct reports yet — but the routine should still ship the message so the manager notices.

## Step 2 — For each reportee, find their active sprint

**Each reportee can be in multiple active sprints at once** — typical case is a frontend engineer who contributes to several squads, so they show up in ALPHA + BETA + IOTA boards simultaneously. The skill must catch all of them, not just one.

Fold sprint discovery into the ticket-pull JQL:

```
JQL: assignee = "<accountId>" AND sprint in openSprints() AND issuetype != Epic
```

One call per reportee, returns every sprint they're in plus every ticket in those sprints, in one round-trip. The sprint metadata for each ticket is in the response (`customfield_<sprint>` array), so the skill groups client-side after the fact.

There is no sprint cache — the skill is stateless across runs (see [../SKILL.md](../SKILL.md) "Config"). Each run pays one `openSprints()` call per reportee; this is fine.

### No active sprint

If `openSprints()` returns no tickets for **a particular reportee**, note it ("Bob has no active sprint") and keep going with the rest.

Only stop the whole skill if **no** reportee has an active sprint — in that case, tell the user plainly ("No active sprint for any of your reports.") and stop. Do not list backlog tickets; that would dilute the signal.

### Sprint length

The default assumption is a 10-working-day sprint. Step 6's capacity math is built on this assumption. (If a particular team's sprints are noticeably longer or shorter, the capacity threshold should be revisited there.)

**PTO and holidays.** Step 2 produces the active sprints; Step 6 applies the capacity scaling. Each reportee's `pto` array (inline-config + canvas-run calendar sync, merged in memory) plus the group's `country` field (drives the holiday lookup in [holidays.md](holidays.md)) feed into Step 6's `capacity_hours`. See [pto.md](pto.md) for the full spec.

## Step 3 — For each reportee, pull their tickets in their sprint

Still iterating per reportee:

```
JQL: assignee = "<accountId>" AND sprint in openSprints()
```

This is the same JQL Step 2 already runs — in practice Steps 2 and 3 are one round-trip per reportee. Request only the fields below — Jira's default response is huge and slow.

### Per-ticket fields to collect

| Field | Why we need it |
|---|---|
| `summary` | Display in the report. |
| `status`, `statusCategory` | Column the ticket is in right now. |
| `priority` | Drives risk ranking. Pulls the literal Jira priority string (`Critical`, `High`, `P0`, `Blocker`, etc. — varies by instance) and stores it verbatim in the per-ticket record. The report displays this string as-is; the abstract concept of "high priority" used for signal matching is defined in the "Priority taxonomy" subsection of Step 5. |
| Flag / `Flagged` field (use `jira_field_ids.flagged` from config; usually has value `["Impediment"]` when set) | The flag is the team's manual "this ticket is in trouble" signal. Always surface flagged tickets as high-attention items, regardless of status. If `jira_field_ids.flagged` is `null`, skip this signal entirely. |
| `issuetype` | Skip Epics; treat Bug/Story/Task uniformly otherwise. |
| `created`, `updated`, `duedate` | "Stale 6d" detection and due-date risk (see signals in Step 5 for the exact thresholds). |
| **Story points** (use `jira_field_ids.story_points` from config — discovered once per instance per Step 0, never hardcoded) | Original estimate. Used for sprint capacity context and scope-creep detection (estimate changing mid-sprint). **Valid values for this team are `{0, 1, 2, 4, 8}`.** Anything outside that set, treat as the unestimated case (null). A `0` is *not* the same as unset: 0 means "deliberately no work" (research, docs, etc.); null means "we haven't estimated yet" and is treated separately (see Step 5 "Missing estimate" signal). **`8` is an anomaly** — work that large should be split — and always gets surfaced as a per-ticket signal in Step 5 regardless of any other state. |
| **Remaining story points** (use `jira_field_ids.remaining_story_points` from config — discovered once per instance per Step 0) | *Intended semantics:* how much of the ticket's original work is allocated to **this** sprint, after any carry-over from previous sprints. *Actual semantics vary per ticket on this team:* some reportees drain rSP to 0 on completion (burndown), others leave it as a stable carry-over allocation. Step 6 picks the canonical SP value per ticket (rSP by default, fall back to SP for Done + rSP=0) — no team-wide decision needed here. If the team doesn't use either field, skip capacity analysis entirely. |
| `comment` — full list with timestamps for the **last 3 days only**, plus total count, plus the last 3 comments verbatim | Two purposes: (1) the last 3 verbatim drive "blocked on…" / "waiting on…" phrase detection — always pull these for tickets in the **Full** fetch tier (see "Tiered fetch by status" below); skipped for Minimal-tier tickets. (2) the timestamps for the last 3 days drive the high-velocity-comment signal in Step 5. **Skip the 3-day-window timestamp pull when `updated < (now − 3 calendar days)`** — the burst signal can't fire on a stale ticket, so the window pull is wasted. The last-3-verbatim is still cheap (it's bundled with the comment fetch anyway) and gets pulled whenever the ticket is in the Full tier. |
| `changelog` filtered to status transitions, story-points field, and sprint-membership field (use `jira_field_ids.sprint` from config) | Powers the time-in-column, scope-creep, reopen, and **late-sprint-addition** signals. This is the most important field after status itself. **Scope to the sprint window**: pass `since=<sprint.start_date>` (or the MCP equivalent — `expand=changelog&startAt=…` and filter client-side if no since-parameter exists) so we don't pull history from prior sprints. For carry-over tickets the pre-sprint history can be many KB and is irrelevant to every signal we derive — all signals look at transitions inside the current sprint window only. |
| Linked pull requests (via the development-info / `_expand=development` field, or the `dev-status` endpoint with `applicationType=GitHub&dataType=pullrequest`) | Lets us see whether an "In Progress" ticket actually has a PR open, how long it's been awaiting review, or whether a merged PR has a still-open ticket attached. |

Skip **subtasks** unless their parent isn't also in the sprint (otherwise you double-count work). Skip **Epics** unless the user explicitly asked for epic-level rollup — they distort counts.

### Derived per-ticket measurements

From the data above, compute these *before* moving to classification — they're inputs to multiple signals.

- **Time in current status (calendar days)** — `now - timestamp of the most recent status change into the current status`. If there's no status change in the changelog, the ticket has been in its current status since `created`, so use `created`.

  **Only compute this for tickets currently in one of these five "in-flight" statuses:**
  1. In Progress
  2. In Review
  3. Ready for Verification
  4. Verified
  5. Merged

  For any other status (To Do, Done, Blocked, Backlog, etc.) — **don't compute it at all**. No signal uses the value for those buckets, so the work is wasted. The threshold for raising the signal is **>3 calendar days** in one of the five in-flight statuses (see Step 5 / Step 4 for how it's used). Status names vary per Jira instance — if the team uses slightly different names ("Code Review" vs "In Review", "QA" vs "Verified"), match on the closest equivalent.

- **Reopened this sprint** — `true` if the changelog contains a transition from `statusCategory=Done` back to anything else, with a timestamp inside the sprint window.
- **Story points changed this sprint** — `true` if the story-points field has a changelog entry inside the sprint window. Capture old → new for the signal text.
- **Inter-sprint carry-over flag** — if `remaining_story_points < story_points`, the ticket was partially worked in a previous sprint and the current sprint only holds the remaining portion. Useful context for the manager ("PROJ-456 — 3 of 8 SP carried over from last sprint"). Don't treat it as a scope-creep signal — within-sprint burndown doesn't show up here. Only the rarer reverse case (`remaining > original`) is genuinely a scope-grew tell, and it's an inter-sprint event, not a within-sprint one.
- **Comments in the last 3 days** — count of comments with `created` ≥ `now − 72h`. Threshold for the high-velocity signal is **>5 comments in the last 3 days** (see Step 5).
- **PR status summary** — for each linked PR (after the canonical-link filter below): `state` (open/merged/closed), `last_review_at` if available, and time since opened. We don't need the diff or commits; just the lifecycle metadata.

- **Canonical-link filter for PRs.** A ticket often has more than one PR linked, and Jira's dev-info also surfaces PRs that someone else attached incidentally (a passing mention in a commit or PR body). For the Step 5 PR signals (PR open with no review, PR merged but ticket not closed), only count a PR as belonging to this ticket when **the ticket key appears in the PR's title or head branch name** (case-insensitive, full key like `THETA-1234` / `VULN-456`). PRs that match only via body text or manual link are dropped — they're cross-references, not the work for this ticket. Apply the filter once at parse time and use the filtered set everywhere; assume Jira's dev-info returns title and branch (they're standard fields), and if a particular MCP response lacks both for a given PR, drop that PR.

### Tiered fetch by status

Not every ticket needs every field. Most signals can only fire on actively-moving work, so for finished or untouched tickets we pull a minimal set and skip the expensive bits (changelog, PR fetch, comments). For a typical sprint where ~half the tickets are Done or in untouched To Do, this is the single biggest token saver in the analysis.

All tiers pull the **cheap scalar fields** — they're tiny and several Step 5 signals (Flagged, Due-date, Missing estimate, 8 SP anomaly) need to fire regardless of status. The tiering only governs the **three expensive fetches**: changelog, comments, and PR/dev-info.

| Tier | Status | Heavy fetches included | Heavy fetches skipped |
|---|---|---|---|
| **Minimal-Done** | `statusCategory == Done` AND `updated` is older than 2 working days | — | changelog, comments, PR/dev-info |
| **Minimal-untouched** | `statusCategory == To Do` OR status is `Development Ready`, AND `updated == created` (no edits since creation) | — | changelog (no transitions exist yet), comments, PR/dev-info |
| **Full** | Everything else — in-flight, blocked, edited To Do, edited Development Ready, recently-updated Done | all three (subject to the per-row rules above, e.g. comment-burst window skipped on stale tickets) | — |

The cheap fields pulled in *every* tier: `summary`, `status`, `statusCategory`, `priority`, `issuetype`, `story_points`, `remaining_story_points`, `created`, `updated`, `duedate`, `resolutiondate`, the Flagged custom field.

**Tier-decision rules:**

- **Recently-updated Done ticket** (Done but `updated` within last 2 working days): bump to **Full** tier. Could indicate a reopen-then-redo cycle we want to surface as a signal.
- **Edited To Do** (`updated > created`): bump to **Full** tier. The edit could be a comment, description change, or status flip worth knowing about.
- **8 SP tickets that are not Done**: always **Full** tier.

The tier decision is made from the cheap "first-pass" JQL response (summary + status + dates + SP) before issuing the expensive per-ticket calls. In practice this means a two-phase fetch per reportee: phase 1 pulls the cheap fields for all tickets, phase 2 fans out to the Full-tier tickets only with `expand=changelog` + dev-info. Minimal-tier tickets are done after phase 1.

These tiers apply equally in routine and local mode. The canvas's Risk deep-dive section adds per-ticket extras (full status timeline, last 3 comments, optional PR state) for the top 5 tickets only — see [output/canvas.md](output/canvas.md) Section 4.

### Practical batching

Pulling changelog and dev-info is the slow part. **Batch by reportee** when the MCP supports it: one phase-2 call that returns all of a reportee's Full-tier tickets with `expand=changelog` and the dev field included, rather than one call per ticket. In **routine mode**, treat this as a hard requirement — per-ticket calls multiplied across daily cadence add up fast. In **local mode** it's nice-to-have but not load-bearing.

If batching isn't available, parallelize across reportees but stay serial within a reportee to keep pagination consistent.

**Skip PR/dev-info entirely for Minimal-tier tickets** (Done and untouched-To-Do). A merged PR on a Done ticket is the expected state — there's no signal to derive. An untouched To Do can't have a PR yet. All three PR signals (`In Progress, no PR open`, `PR open, no review`, `PR merged, ticket not done`) require a Full-tier ticket; the fetch on Minimal-tier ones is pure waste. On a healthy team this drops the dev-info fetch from roughly ~50% of tickets.

## Step 4 — Classify each ticket

Five buckets. Apply in order — first match wins:

| Bucket | Rule |
|---|---|
| **done** | `statusCategory = Done` |
| **blocked** | Status name contains "blocked" / "waiting" / "on hold", OR has the `blocked` label/flag, OR most recent comment (within the sprint) contains "blocked on", "waiting on", "can't proceed", "need help" |
| **at-risk** | Not done, and any of: (a) `duedate` is within 7 calendar days or already passed; (b) sprint ends within 2 working days and status is still `To Do`; (c) reopened this sprint (from Step 3's derived data); (d) `updated` is older than 4 working days while status is `In Progress` (stale-in-flight); (e) time in current status > 3 calendar days **and** current status is one of the five in-flight statuses listed in Step 3 (In Progress / In Review / Ready for Verification / Verified / Merged); (f) ticket is **flagged** (Jira impediment flag set); (g) `remaining_story_points > story_points` (scope grew between sprints — rare but worth surfacing) |
| **not-started** | `statusCategory = To Do` AND `updated` < 1 day after sprint start (never touched) |
| **on-track** | Everything else not in the above |

A ticket can be "secretly at-risk" — looking fine but quietly drifting. The `updated` check and the time-in-column check on in-progress work catch this; don't skip them.

## Step 5 — Signal scan

These are the things the user actually needs to *act on*. For each reportee, collect:

### Stale tickets

`In Progress` with no `updated` activity in 4+ working days. Note the gap explicitly: "stale 6d".

### Stuck in an in-flight status

Ticket has been in its current status for **>3 calendar days** AND that status is one of the five in-flight statuses listed in Step 3 (In Progress, In Review, Ready for Verification, Verified, Merged).

The threshold is intentionally tight (3 days, not 5) because these are the statuses where work is supposed to be moving. A ticket sitting in "In Review" for 4 days means a PR isn't getting reviewed; a ticket sitting in "Ready for Verification" for 4 days means QA hasn't picked it up; a ticket in "Merged" for 4 days means the release/close step is being missed. Each of these is something a manager can unblock with a single message.

Other statuses (To Do, Done, Backlog, etc.) are exempt — being "stuck" in To Do isn't a workflow problem, it's a backlog choice. Distinct from "stale" — a ticket can have recent comment activity but still not have moved columns.

**Phrasing:**

- Standalone (no PR signal also firing): *"In Review for 5 days."* — status-centric.
- Combined with the "PR open, no review" signal (see below) on the **same ticket**: use the PR-centric form *"PR open 5d, no review activity"* — the status is implied by an open PR, and the combined phrasing avoids "review" appearing twice in two different senses on the same line. Status-centric phrasing is reserved for the no-PR case (where the implicit-status meaning of "PR open" doesn't apply).

### Reopens

Reopened this sprint (Done → not-Done transition inside the sprint window). Often a sign of incomplete acceptance criteria, regressions, or scope discovered late.

### Priority inversion

Reportee has a high-priority ticket sitting in `statusCategory == To Do`, AND they're actively working on lower-priority tickets. Suggests the urgent work hasn't been picked up yet — maybe an oversight, maybe a hidden blocker the team didn't formalize. Either way, the manager wants to know.

**Priority taxonomy.** Jira priority values vary by instance — common sets are `Critical/High/Medium/Low`, `Highest/High/Medium/Low/Lowest`, or `P0/P1/P2/P3`. We match the abstract concept of "high priority" using a **case-insensitive substring** match against this list: `Critical`, `Blocker`, `Highest`, `P0`, `P1`, `High`. Everything else is treated as non-urgent. We do **not** normalize the display string — the report uses whatever literal value Jira returns (e.g. *"PROJ-456 (Critical) ..."* on instances that use Critical, *"PROJ-456 (P0) ..."* on instances that use P0). The same rule applies everywhere priority is surfaced in the report.

**Trigger when:**
- At least one ticket has a high-priority label (per the list above) AND is in `statusCategory == To Do`.
- At least one of the reportee's in-flight tickets (any of the five in-flight statuses from Step 3) has a *lower* priority than that untouched high-priority ticket. To compare priorities across Jira's various labeling schemes, use this rank order (highest → lowest): `Critical/Blocker/Highest/P0` > `High/P1` > `Medium/P2` > `Low/P3` > `Lowest/P4`. Unmatched values rank at the bottom.

**Skip when:**
- The high-priority ticket itself is in-flight (the reportee is already on it).
- The high-priority ticket is blocked (a different signal — blocked has its own line).
- The high-priority ticket is done.
- Everything the reportee is working on is the same priority as the untouched ticket (no actual inversion — they may have just picked one of several same-priority tickets).

If there are multiple untouched high-priority tickets, surface the one with the highest priority. If multiple in-flight lower-priority tickets exist, name the one with the lowest priority (most extreme inversion) for the example.

Phrase as: *"PROJ-456 (Critical) in To Do while working on PROJ-471 (Medium)"*. The contrast — same line, both literal Jira priorities visible — is the whole point.

### Scope creep

Heuristics — any of these flag the ticket:

- Story point estimate changed mid-sprint (use the "story points changed this sprint" derived value, including old → new). A jump like 2→8 is a stronger signal than 2→4 (the valid SP values are `{0, 1, 2, 4, 8}` — any move *up* the scale tells you the work grew).
- **`remaining_story_points > story_points`** — the ticket's allocation to this sprint exceeds its original estimate. This is an inter-sprint scope change (not within-sprint burndown — that's not what remaining-SP tracks), and usually means the work was re-estimated upward when it got pulled into the current sprint.
- Multiple new subtasks added after sprint start.
- Summary edited substantially mid-sprint.

Don't flag every change — only those that look like work *expanded*, not clarified.

(Tickets *added to the sprint* after start are handled by their own signal — see "Late sprint addition" below — because the diagnosis is different from work expanding inside an existing ticket.)

### Late sprint addition

Ticket was added to the active sprint **more than 2 calendar days after sprint start**. Detect by comparing the **ticket's own sprint start date** (from the active sprint object inside its `customfield_<sprint>` array — pick the entry with `state == "active"` that matches the relevant board) against the first changelog entry where this ticket gained that sprint id.

- **Skip subtasks.** Sub-tickets often get created mid-sprint as breakdowns of in-progress parent work; that's expected, not a scope-discipline issue. The signal only fires on top-level tickets (issuetype != Sub-task).
- **2-day grace window** at the start covers normal sprint-setup churn — teams routinely tidy the sprint contents on day 1 (and sometimes spill into day 2). Anything added after that is an interruption to commitments already made.

Phrase as: *"PROJ-501 added to sprint on day 4 (after start + 2d grace)."*

If multiple tickets were added late for the same reportee, group them: *"3 tickets added after day 2: PROJ-501, PROJ-507, PROJ-512."*

### Missing estimate

Ticket has **no story-point value set** (null/unset, not `0` — `0` is intentional and not flagged here). Flag it as a missing-estimate signal — *unless* it's in one of the early-lifecycle or terminal statuses where being unestimated is normal:

**Exempt statuses (do not flag unestimated tickets in these):**
- To Do
- Blocked
- Development Ready
- Refinement Ready
- Won't Do

The reasoning: those statuses cover (a) tickets that haven't been refined yet, (b) tickets blocked from refinement, and (c) tickets that won't ship — none of which need an estimate to be useful. Anything *outside* this list (In Progress, In Review, Done, etc.) that lacks an estimate is a process miss: work is happening (or finished) without the team knowing how much it was sized at, which breaks capacity tracking for this sprint and velocity history for future planning.

Status names vary per Jira instance — if the team uses slightly different names (e.g. "Ready for Dev" instead of "Development Ready", "Backlog" alongside "To Do"), match on the closest equivalent. When in doubt, lean toward flagging — a false positive prompts a quick estimate and costs nothing; a missed unestimated ticket distorts the capacity verdict.

**Surfaced in the canvas Sprint health snapshot.** The "Unestimated" line in [output/canvas.md](output/canvas.md) Section 1 calls out in-flight unestimated tickets explicitly. The reason: in Step 6's capacity math we deliberately under-count in-flight unestimated tickets at 1.5h (the 1-SP equivalent) so they don't fake an overbooked verdict, which means the *only* place a manager learns about unsized in-flight work is the snapshot.

Phrase as: *"PROJ-456 — no estimate, currently In Progress."* Group these together at the bottom of the per-reportee section so they're easy to spot.

### Oversized ticket (8 SP)

Ticket has **story points = 8**. The valid SP scale for this team is `{0, 1, 2, 4, 8}` and `8` is intentionally outside the "normal" working range — work that large should be split into smaller tickets before sprint planning, not committed to as-is. Surface every 8 SP ticket regardless of any other state (it can be on-track, done, blocked — doesn't matter; the size itself is the issue).

This is distinct from the capacity verdict in Step 6 (which looks at the *aggregate* shape of a person's sprint). The 8 SP signal is a per-ticket process miss: it means refinement didn't break the work down.

Phrase as: *"✂️ PROJ-456 (8 SP) — should be split."*

### High comment velocity

Ticket has **more than 5 comments in the last 3 calendar days** (from the derived "comments in the last 3 days" count in Step 3). A burst of comment activity usually means people are arguing about scope, debugging a hard problem, or trying to unstick something live. Worth surfacing even if every other signal says "on track" — it's where the team's attention is actually going right now. Phrase as: *"7 comments in the last 3 days."*

### Due date approaching or passed

Surface any non-`Done` ticket whose `duedate` is within **7 calendar days** (coming up) or already in the past. Two phrasings:

- Coming up: *"Due in 3 days (Fri 22 May)."*
- Already passed: *"Due date passed 2 days ago (Fri 15 May)."*

Note that `duedate` is distinct from the sprint end date — it's a ticket-level deadline the team sets explicitly. Many tickets won't have one; only tickets that do are eligible for this signal.

### Flagged

Ticket has the Jira flag set (from the per-ticket `Flagged` / impediment field collected in Step 3). The flag is a manual "this is in trouble" marker from the team — always surface flagged tickets, regardless of any other signal status. Phrase as: *"Flagged by team."* If the flag value carries text (Jira lets you flag with a reason), include up to one line of it.

### Pull-request signals

Using the per-ticket PR status summary from Step 3:

- **PR open, no review** — PR has been open ≥3 working days with no review activity. Risk of getting forgotten before sprint end.
- **PR merged, ticket not closed** — linked PR is merged but the ticket is still in a status outside `{Done, Merged}`. Either a closing step is missing (QA, deploy) or the ticket got forgotten about. (A ticket in `Merged` status with a merged PR is the expected steady state, not a signal.)

These are common, fixable issues that managers catch in 1:1s — surfacing them automatically is high-value.

### "Blocked" / "waiting on" comments

Scan last 3 comments per ticket. If any contain phrases like:

- "blocked on", "waiting on", "blocked by", "waiting for"
- "need help with", "stuck on", "can't proceed without"
- "@<someone> can you", "ping <someone>"

Capture: ticket key, who's blocking, the verbatim phrase (max 1 line), comment date.

### Priority + at-risk overlap

A high-priority ticket (per the "Priority taxonomy" subsection above — `Critical`, `Blocker`, `Highest`, `P0`, `P1`, `High`, case-insensitive) that's also `at-risk` or `blocked` is the highest-attention item. Surface these first in the risks section. The display string is whatever Jira returned literally; don't normalize.

## Step 6 — Assess per-reportee capacity

For each reportee, compute a load verdict: **underworked / normal / borderline / overbooked / fragmented**. This is separate from per-ticket signals — it's about whether the *shape and volume* of their sprint commitment is healthy, not whether any individual ticket is in trouble.

This is a sprint-commitment question: did this person get loaded up correctly at planning? The verdict is mostly stable across runs within a sprint (commitments don't shift much day-to-day), which is fine — it's the kind of thing a manager wants flagged early and consistently so they can rebalance before sprint mid-point.

The math here treats story points as a proxy for **hours of expected work**, not a pure complexity score, because the manager-facing question is "does this fit in the time they have?" — not "is the abstract complexity reasonable?". The SP→hours mapping below is what makes the verdict legible.

### Story-point field to use — literal rSP

Use `remaining_story_points` (rSP) verbatim — it's the field that captures "this sprint's commitment" after any carry-over from previous sprints, and it's authored by the team to mean exactly that. Trust it.

```
canonical_sp = rSP            # this-sprint commitment (the answer in almost every case)
if rSP is null:
    canonical_sp = SP         # fallback when rSP isn't populated
if both null:
    canonical_sp = null       # unestimated — see "Unestimated handling"
```

**Don't second-guess rSP=0 on Done tickets.** A Done ticket with rSP=0 means the team marked it as "no remaining work this sprint" — usually because the work happened over multiple sprints and what landed in this sprint was just the close-out. Treating rSP=0 as 0.5h (the SP=0 mapping) is the team's recorded position. Earlier spec versions tried to "recover" the original SP for these tickets via a Done+rSP=0 → SP fallback; that overcounted carry-over work for reportees who actually use rSP as a stable allocation. The simpler literal rule is correct.

**rSP values outside `{0, 1, 2, 4, 8}`.** rSP can hold non-canonical values (3, 5, 6, etc.) when work was partially completed mid-sprint and re-estimated. Map to the nearest canonical bucket in `SP_HOURS` (e.g. `3 → 4 SP → 20h`, `5 → 4 SP → 20h`). Don't treat as unestimated — the value is real, just non-standard.

**Size-bucket counts (used by the Borderline/Overbooked verdict rules) still use the original `story_points`**, not rSP. The "5 × 4-SP tickets is overbooked" rule is about absolute ticket size (how many chunky tickets a person owns), not how much work remains in the current sprint. A carry-over 4-SP ticket with rSP=2 is still a 4-SP ticket on the person's plate. Use SP for `size_buckets`, rSP for `sprint_hours`. The 8-SP anomaly signal (Step 5) also keys off SP, not rSP.

If neither field is populated on any ticket, skip the capacity step entirely.

### SP → hours

Each valid SP value maps to an approximate hour budget:

| SP | Hours | Rough wall-clock |
|---|---|---|
| 0 | 0.5 | Under an hour |
| 1 | 1.5 | An hour or two |
| 2 | 8 | A workday |
| 4 | 20 | ~2.5 days |
| 8 | — | **Anomaly** — surfaced as a per-ticket signal in Step 5, not counted into hour totals like a normal ticket. Treat its hour contribution as 40 (double the 4 SP value) for the capacity math so it doesn't escape detection, but the primary handling is "this ticket shouldn't exist." |

These numbers are calibrated to **focused dev time**, not raw hours on the clock — they account for the work that actually moves a ticket, with the rest of the day going to meetings, reviewing others' code, helping teammates, etc. A "full sprint" lands at **~80h** of focused work over 10 working days (about a 50% productive ratio), which matches a "four 4-SP tickets is normal" team norm.

### Inputs (computed once per reportee)

Count **all** sprint tickets the reportee is assigned to, including ones already `Done`. The question is "did sprint planning load them right?" — Done tickets were part of what they signed up for, so they count.

- `sprint_hours` — sum of the per-ticket hour value across all the reportee's sprint tickets. This is the primary load measure. For *unestimated* tickets in an in-flight status, count them at **1.5h (the 1 SP value)** rather than excluding (see "Unestimated handling" below) — a deliberately low default so the math doesn't over-flag overbooking from missing estimates; the gap is surfaced loudly via the Step 5 signal instead. For unestimated tickets in an exempt status, exclude.
- `ticket_count` — count of all the reportee's sprint tickets.
- `size_buckets` — counts per SP value: `{0: N, 1: N, 2: N, 4: N, 8: N}`. Only tickets with a *real* SP value land here. Unestimated tickets — even the in-flight ones contributing to `sprint_hours` — are tracked separately (see below); they don't get put in the `1 SP` bucket just because we costed them at 1 SP.
- `small_hours` — sum of hours for 0 SP and 1 SP tickets (the "fragmentation pool"). Used by the Fragmented verdict.
- `unestimated_in_flight_count` / `unestimated_exempt_count` — counts of unestimated tickets, split by whether the ticket is in an in-flight status (counted at 1.5h in `sprint_hours`) or an exempt status (excluded from the math). The exempt list matches Step 5's "Missing estimate" exemption: To Do, Blocked, Development Ready, Refinement Ready, Won't Do.
- `capacity_hours` — base **80** (10 working days × ~8 productive hours per day, rounded for the focused-dev-time calibration), scaled down per reportee if they have PTO, sick leave, or public holidays during the sprint window. Formula: `capacity_hours = 80 × (10 − pto_days_in_sprint) / 10`. See [pto.md](pto.md) for the full rule and worked examples. Verdict thresholds scale proportionally with capacity (see Verdict rules below).

### Verdict rules

Apply in this order — first match wins. Thresholds scale with the reportee's `capacity_hours` (which is 80 for someone with no PTO, and lower if they have PTO/holidays during the sprint window — see [pto.md](pto.md)). The 4-SP / 5-SP bucket counts don't scale because they're about ticket-count shape, not hours.

| Verdict | Rule | Why it matters |
|---|---|---|
| **🔴 Overbooked** | `sprint_hours > 1.25 × capacity_hours` OR `size_buckets[4] ≥ 5` | Past the borderline — either over 125% of capacity, or five-plus 4-SP tickets. Too much; needs trimming. |
| **🟡 Borderline** | `sprint_hours ≥ capacity_hours` OR `size_buckets[4] ≥ 4` | Full to the brim. At or above capacity, or four+ 4-SP tickets. Will probably ship but any surprise pushes it over. Worth flagging. |
| **🟡 Fragmented** | `(size_buckets[0] + size_buckets[1]) ≥ 5` AND `small_hours / sprint_hours > 0.4` | Many tiny tickets dominate. Total hours might look fine, but the reportee is context-switching constantly. Distinct from overbooked — the problem is shape, not volume. |
| **🔵 Underworked** | `sprint_hours < 0.8125 × capacity_hours` | About 80% of capacity is the "fully loaded" line. Anything below that is light enough to flag. Could be sandbagging, blocked on getting work assigned, lots of carry-over, or the manager forgot to load them up. |
| **⚪ Normal** | Everything else | Reasonable load. Typically 80–100% of capacity. No action needed. |

At full capacity (80h), the thresholds resolve to: Underworked <65h, Normal 65–80h, Borderline 80–100h, Overbooked >100h. At a PTO-scaled capacity of 64h (2 days off): Underworked <52h, Normal 52–64h, Borderline 64–80h, Overbooked >80h.

**Note on 8 SP tickets:** they count as 40h (double the 4 SP value) in `sprint_hours` — they contribute heavily to the volume math, *and* they get flagged individually via the Step 5 "Oversized ticket (8 SP)" signal. The two are independent — even one 8 SP ticket gets close to half a full sprint's budget on its own, which is a deliberate consequence: an 8 SP ticket *should* feel disproportionate, because it shouldn't exist.

**Worked examples (`capacity_hours = 80` for everyone):**

| Scenario | sprint_hours | 4-SP count | Verdict |
|---|---|---|---|
| 4 × 4SP + 2 × 2SP | 96 | 4 | **Borderline** (≥80h and 4-SP count ≥4) |
| 4 × 4SP + 2 × 0SP | 81 | 4 | **Borderline** (≥80h and 4-SP count ≥4) |
| 2 × 4SP + 6 × 2SP | 88 | 2 | **Borderline** (≥80h, even with only 2 × 4SP) |
| 5 × 4SP | 100 | 5 | **Overbooked** (4-SP count ≥5) |
| 6 × 4SP | 120 | 6 | **Overbooked** (>100h and 4-SP count ≥5) |
| 10 × 0SP | 5 | 0 | **Underworked** (<65h) |
| 1 × 2SP + 2 × 4SP | 48 | 2 | **Underworked** (<65h — light load) |
| 12 × 1SP + 3 × 2SP | 42 | 0 | **Fragmented** (12 small tickets, 18/42 ≈ 43% small-hours) |

### Phrasing for the report

Each verdict gets a one-line explanation grounded in the actual numbers:

- **Underworked**: *"5 tickets, ~57h committed — has room for ~23h more (only ~1.4 weeks of work in a 2-week sprint)."*
- **Borderline**: *"5 tickets, 88h. Packed; no slack for surprises."*
- **Overbooked**: *"6 tickets, 120h. Trim something."*
- **Fragmented**: *"15 tickets, 12 at 0-1 SP — 42h total but most of it tiny. Context-switching risk."*
- **Normal**: *"4 tickets, ~60h, balanced mix."* (Or just omit the explanation in the report — normal is the boring case.)

The Capacity overview line in the canvas stays at verdict + percentage. When the manager needs more detail, the canvas's Capacity breakdown section at the bottom shows raw SP/rSP numbers per ticket.

### Edge cases

- **Tickets without SP at all** (neither `remaining_story_points` nor `story_points` populated, i.e. truly null) — split by status:
  - **In-flight unestimated** (status is one of the five in-flight statuses from Step 3 — In Progress, In Review, Ready for Verification, Verified, Merged — OR any other status outside the exempt list below): count each at **1.5h (the 1 SP value)** in `sprint_hours`. The low default is deliberate: we don't know how big the work actually is, so don't let a single unestimated ticket fake an overbooked verdict. The *real* response to unestimated in-flight work is the Step 5 signal, which always raises (see below). Track in `unestimated_in_flight_count`. Don't add them to `size_buckets` — the bucket is for *known* sizes, not assumed ones.
  - **Exempt-status unestimated** (To Do, Blocked, Development Ready, Refinement Ready, Won't Do — same list as Step 5's "Missing estimate" exemption): exclude from `sprint_hours` entirely. They're not yet expected to have an estimate, so they're not real load yet. Track in `unestimated_exempt_count` for context.
  - Tickets with `0` SP are *not* in either group — `0` is a valid SP value and counts as estimated (0.5h).
  - Surface in the phrasing: *"5 tickets (3 estimated = 36h, 1 unestimated in-flight ≈ 1.5h, 1 unestimated in To Do)."* The `≈` is intentional — make it visible that this number is a placeholder, not a measurement. Express the placeholder as "counted as 1 SP" in any per-ticket display (Step 5 signal + long-review capacity breakdown). If more than half of the *in-flight* tickets are unestimated, soften the verdict and note: *"Hard to tell — most of their in-flight work is unestimated."*
  - All in-flight unestimated tickets surface in the canvas Sprint health snapshot's "Unestimated" line — the manager never misses an unsized in-flight ticket. See [output/canvas.md](output/canvas.md) Section 1.
- **Team doesn't use story points at all** — skip this step entirely. Don't emit a capacity verdict; the data isn't there.
- **Reportee has zero sprint tickets** — verdict is "underworked", explanation: *"No sprint tickets assigned."*

### These are starting heuristics

Same caveat as sprint-health traffic-light: the SP→hours mapping (0→0.5, 1→1.5, 2→8, 4→20, 8→40) and the capacity (80h) are starting points calibrated for this team. Adjust if you can see something the rules miss — e.g. if the manager mentions "this is a hardening sprint with lots of small fixes" the fragmented threshold is too tight; if a reportee is doing exploratory research the unestimated count should weigh more. The manager cares about the verdict + the reason, not the rule it came from.

## Step 7 — Compute sprint health

One-line traffic light for the overall report. Rough rules:

- **🟢 Green** — ≥70% of sprint work is `on-track` or `done`, no blocked or at-risk high-priority items, no stale tickets older than 5 days.
- **🟡 Yellow** — anything not green or red.
- **🔴 Red** — any blocked high-priority ticket, OR ≥3 at-risk items across the team, OR a reportee with 0 done tickets past sprint midpoint, OR the sprint ends within 2 working days and ≥40% of work isn't done.

These are starting heuristics — adjust with judgment based on what's in the data. The traffic light is a vibe, not a contract.

## Step 8 — Hand off to output

Pass the structured data to [output/canvas.md](output/canvas.md). One run = one group = one rendered canvas.

```
{
  "group": {
    "slug": "alice",
    "manager": { "name": "Alice Johnson", "slack_id": "U001AAAAAA1" },
    "overall_health": "yellow",
    "verdict": "One-line summary for this group's sprint(s).",
    "reportees": [
      {
        "name": "Bob Smith",
        "sprints": [
          { "name": "ALPHA Sprint 12", "id": 13476, "start": "...", "end": "...", "days_remaining": 4 },
          { "name": "BETA Sprint 12", "id": 13150, "start": "...", "end": "...", "days_remaining": 4 }
        ],
        "counts": { "done": 2, "on_track": 3, "at_risk": 1, "blocked": 1, "not_started": 0 },
        "capacity": {
          "verdict": "overbooked",
          "sprint_hours": 117.5,
          "ticket_count": 8,
          "size_buckets": { "0": 0, "1": 1, "2": 2, "4": 5, "8": 0 },
          "small_hours": 1.5,
          "unestimated_in_flight_count": 0,
          "unestimated_exempt_count": 0,
          "sp_field_used": "remaining_story_points",
          "capacity_hours": 80,
          "explanation": "8 tickets, 117.5h. Trim something."
        },
        "tickets": [
          {
            "key": "PROJ-123",
            "summary": "...",
            "status": "In Review",
            "bucket": "at-risk",
            "priority": "High",
            "flagged": false,
            "story_points": 2,
            "remaining_story_points": 8,
            "hours": 36,
            "duedate": "2026-05-22",
            "duedate_days_away": 4,
            "time_in_status_days": 5,
            "status_is_in_flight": true,
            "comments_last_3d": 7,
            "pr_state": "open_no_review",
            "signal": "In Review 5d, 7 comments in last 3d — scope grew 2→8 SP (should split)",
            "updated_days_ago": 1
          }
        ]
      }
    ],
    "top_risks": [
      { "key": "PROJ-456", "owner": "Bob", "reason": "stale 6d, Critical, 2 working days left" }
    ],
    "follow_up_questions": [
      "Ask Bob what's needed to unblock PROJ-456 — has been stale since Monday."
    ]
  }
}
```

Notes on shape:

- `slug` is the group identifier (the `name` field from the inline config). Useful for logging.
- `manager` carries `slack_id` so the renderer can prepend `<@…>` to the message (and so it can search Slack for the prior canvas — see [output/canvas.md](output/canvas.md) "Sourcing the prior percentage").
- `sprints` is **per reportee, plural** — a reportee can be on multiple active boards/sprints simultaneously. The renderer aggregates the ticket data across them; each ticket carries its own sprint membership.
- `overall_health`, `verdict`, `top_risks`, and `follow_up_questions` summarize **this group only**. There is no cross-group aggregation — each routine entry runs independently.

The follow-up questions are 1:1 prompts the manager can paste into their notes — phrase them as questions or talking points, not status updates.

### Capacity diff against the prior canvas

Canvas mode's Section 2 (Capacity overview) renders a `prev_% → new_% · reason` line for any reportee whose percentage moved since the last canvas — see [output/canvas.md](output/canvas.md) Section 2 "Two line shapes" for the full rules. To support that, the analysis flow needs two extra hooks:

1. **At the start of a canvas run**, after parsing the inline config and before assembling output: find the prior canvas in Slack (see [../SKILL.md](../SKILL.md) "Finding the prior canvas" — search `output_channel` for the most recent bot message containing `<@<manager.slack_id>>` and a canvas URL). If found, call `slack_read_canvas` on that canvas id, parse the per-reportee percentages from the Capacity overview lines, key them by reportee name. This becomes the `prior_pct[<name>]` map the renderer uses. Capture the Slack message's `ts` as `prior_canvas_posted_at` for use in the reason-derivation JQLs below.
   - If the lookup finds nothing, the read fails, or no reportee's active sprint matches what the prior canvas covered: drop the map. Every line falls back to the absolute-percentage shape. Don't fail the run.
   - Parser scope: extract `<full name>` and the first `<N>%` token after it. Don't try to parse verdict tier, prior diff, or PTO suffix — those aren't needed.

2. **Per-reportee reason derivation** (only run for reportees whose percentage actually moved): query the four signals listed in [output/canvas.md](output/canvas.md) Section 2 "Deriving the reason text" — sprint adds, sprint removals, story-point changes, rSP drains — scoped to the window between `prior_canvas_posted_at` and now. Rank by absolute hour delta, take the top 1–2 contributors, render under 12 words. Most of the data is already in the per-reportee ticket fetch from Step 3; only the "tickets removed from sprint" branch needs an extra JQL (those tickets are gone from `openSprints()` and won't appear in Step 3's results).

3. **Tickets-removed list for Section 1b**: per reportee, per current active sprint id, run:
   ```
   assignee = "<accountId>" AND sprint CHANGED FROM <prior_sprint_id> AFTER "<prior_canvas_posted_at>"
   ```
   **Crucially do NOT add `sprint in openSprints()` to this query** — the removed ticket may no longer be in any open sprint. Per result, fetch `summary`, current `status`, `priority`, SP (`jira_field_ids.story_points`), and current `customfield_<sprint>` membership (used to decide "pushed to <next sprint>" vs "back to backlog"). Feeds [output/canvas.md](output/canvas.md) Section 1b and the delta's category 7b.

   **Same JQL pattern, different window for the delta**: when rendering delta mode, the window is `-24h` rather than `AFTER "<prior_canvas_posted_at>"`. Otherwise identical — the canvas and delta render formats share one detection mechanism, two scopes.

These hooks add at most three extra calls per canvas (one Slack search + `slack_read_canvas` to find/read the prior canvas, one "removed from sprint" JQL per reportee for the inter-canvas window, one extra "removed from sprint" JQL per reportee if Section 1b fires) — cheap, and only paid on canvas runs, not deltas (deltas pay their own one-call-per-reportee for category 7b).

### Step 9 — Post the channel message linking to the canvas

After the canvas is created (or updated in place via the Slack lookup), post a one-line channel message linking to it. See [../SKILL.md](../SKILL.md) Step 8 for the message format. **The `<@<manager.slack_id>>` mention on this message is what makes the canvas findable by the next canvas run** (the lookup keys on the bot's prior message containing that exact mention + a canvas link). Never omit the mention on channel posts.

There is no file write-back — the skill is stateless across runs. The canvas itself plus this channel message are the only durable artifacts; the next canvas run reconstructs everything else from Slack and Jira.

## Practical Jira-MCP tips

- **Pagination**: don't pull more than ~50 tickets without paginating; large sprints will time out.
- **Rate limits**: the Atlassian MCP is fine for a few dozen requests per run, but if you're doing per-ticket changelog pulls, batch by reportee rather than per ticket where possible.
- **Field selection**: always pass `fields=` to limit the response. Default Jira responses are huge.
- **`openSprints()` quirk**: only works for boards the authenticated user has visibility to. If a reportee's sprint isn't visible, you'll get zero results — fall back to `sprint in (...)` with explicit IDs or ask the user which board their team is on.
- **Time math**: "working days" = Monday-Friday, no holiday calendar. Close enough for these heuristics; don't overengineer.
- **Friday end ≈ following Monday end.** When a sprint's `endDate` falls on Friday and the next sprint would start the following Monday, treat the close-out as equivalent to a Monday-ending sprint for all "sprint ends within N working days" logic. The weekend collapses the gap — flagging *"sprint ends today (Fri)"* with extra urgency vs *"sprint ends Monday"* is noise. Concretely: a Fri-end report running on Fri and a Mon-end report running on Mon should produce the **same** traffic-light escalation, the **same** sprint-ends-soon suffix on the Capacity overview line, and the **same** threshold-crossings category content in the delta. If a reportee's Friday-ending sprint is the only one ending that close-out week and other reportees' sprints end the following Mon, don't single the Friday one out — they're the same week.
