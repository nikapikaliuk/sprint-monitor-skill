# Token economy

Every token spent in the routine multiplies across daily runs and across reportees, so the skill is deliberately stingy. This file is a single-page index of the optimizations baked into the spec — useful for review, for explaining the design, and for spotting where to push further if cost becomes a concern. The authoritative definitions stay in their owning spec files; the entries here link out.

## What each optimization buys

Rough orders of magnitude, assuming a manager with 5 reportees, ~10 sprint tickets each, ~50 total tickets per run.

| # | Strategy | Lives in | Roughly saves |
|---|---|---|---|
| 1 | Tiered fetch by status (Minimal vs Full) | [analysis.md](analysis.md) Step 3 "Tiered fetch by status" | ~50% of expensive per-ticket fetches on a healthy sprint |
| 2 | Skip PR/dev-info on Minimal-tier tickets | [analysis.md](analysis.md) Step 3 "Practical batching" | The Minimal-tier portion of every dev-info call (typically half) |
| 3 | Skip the comment-burst window pull on stale tickets | [analysis.md](analysis.md) Step 3 comment field | The burst-window pull on every ticket with `updated < now − 3d` |
| 4 | Scope changelog fetches to the sprint window | [analysis.md](analysis.md) Step 3 changelog field | The pre-sprint history for carry-over tickets (often KBs) |
| 5 | Field-selection on every Jira call | [analysis.md](analysis.md) "Practical Jira-MCP tips" | The default Jira response is huge; field selection drops it by ~70% |
| 6 | Batch by reportee instead of per ticket | [analysis.md](analysis.md) Step 3 "Practical batching" | One round-trip per reportee instead of one per ticket on phase-2 fetches |
| 7 | Per-reportee parallelism only between people, not within | [analysis.md](analysis.md) Step 3 "Practical batching" | No multiplication, but avoids pagination breakage that costs retries |
| 8 | Phase-2 enrichment scoped to top-5 risk tickets only | [output/canvas.md](output/canvas.md) Section 4 / [analysis.md](analysis.md) Step 3 | Full changelog+comments fetch on every in-flight ticket (often 20–30 per run) |
| 9 | Each routine entry is independent at delivery time | [SKILL.md](../SKILL.md) Step 8 / [analysis.md](analysis.md) Step 8 hand-off | No chapter-wide rollup work; each routine entry emits one self-contained message |

The earlier "cache-aware sprint discovery" optimization (caching `openSprints()` results in a `cached_sprints` field on the config file) has been **removed** with the move to stateless runs — see [../SKILL.md](../SKILL.md) "Config". One `openSprints()` call per reportee per run is cheap enough to not need caching, and the cache invalidation logic was a real source of complexity.

## Detail

### 1. Tiered fetch by status

Most signals can only fire on actively-moving work. The skill tiers each ticket into one of three buckets after a cheap first-pass JQL response, and only the **Full** tier gets the expensive three fetches (changelog, comments, PR/dev-info):

- **Minimal-Done** — `Done` for >2 working days. No expensive fetches.
- **Minimal-untouched** — `To Do` or `Development Ready` with no edits since `created`. No expensive fetches.
- **Full** — everything else.

On a typical sprint where ~half the tickets are Done or untouched, this is the single biggest token saver.

→ See [analysis.md](analysis.md) Step 3 "Tiered fetch by status".

### 2. Skip PR/dev-info on Minimal-tier tickets

PR/dev-info is the slowest Jira call. None of the three PR signals can fire on a Minimal-tier ticket (no PR can exist on an untouched To Do; a merged PR on a Done ticket is the expected steady state). The fetch is pure waste there — skipped entirely.

→ See [analysis.md](analysis.md) Step 3 "Practical batching".

### 3. Skip the comment-burst window pull on stale tickets

The high-comment-velocity signal fires on >5 comments in the last 3 calendar days. If the ticket's `updated` timestamp is older than 3 days, no burst can possibly be in that window — the timestamp pull is wasted. The skill skips it. The "last 3 comments verbatim" (used for "blocked on…" phrase detection) is still pulled because it's bundled with the comment fetch anyway.

→ See [analysis.md](analysis.md) Step 3 `comment` field row.

### 4. Scope changelog fetches to the sprint window

Pass `since=<sprint.start_date>` (or the MCP equivalent) on changelog pulls. Every signal that uses the changelog (time-in-column, scope creep, reopens, late sprint addition) only cares about transitions inside the current sprint window. For carry-over tickets, pre-sprint history can be many KB and is irrelevant.

→ See [analysis.md](analysis.md) Step 3 `changelog` field row.

### 5. Field-selection on every Jira call

Jira's default response is huge. The skill always passes `fields=` to limit the response to exactly what's needed. The cheap-fields list (every tier) is `summary, status, statusCategory, priority, issuetype, story_points, remaining_story_points, created, updated, duedate, resolutiondate, Flagged`. The Full-tier adds `comment, changelog, development-info` selectively.

→ See [analysis.md](analysis.md) "Practical Jira-MCP tips".

### 6. Batch by reportee

When the MCP supports it, fetch all of a reportee's Full-tier tickets in one phase-2 call (with `expand=changelog` + dev-info) instead of one call per ticket. In routine mode this is a hard requirement — per-ticket calls multiplied by daily cadence add up fast.

→ See [analysis.md](analysis.md) Step 3 "Practical batching".

### 7. Per-reportee parallelism

Parallelize across reportees but stay serial within a reportee. Inter-reportee parallelism doesn't reduce token count but cuts wall-clock time. Intra-reportee parallelism breaks pagination consistency and ends up costing retries.

→ See [analysis.md](analysis.md) Step 3 "Practical batching".

### 8. Phase-2 enrichment scoped to top-5 risk tickets only

The canvas's Risk deep-dive needs detailed per-ticket data (full status timeline, last 3 comments, PR state) — but only for the top 5 tickets across the group. Don't pull that for every in-flight ticket. Run Step 3's cheap phase-1 fetch first to identify candidates by signal (stuck, reopened, high-pri at-risk, due-soon), rank them per the [canvas.md](output/canvas.md) Section 4 ordering, then pay the heavy fetch on the 5 winners only.

On a team like the test data this is the difference between 5 detailed fetches vs ~30 (one per in-flight ticket across all reportees) — roughly an order of magnitude.

→ See [output/canvas.md](output/canvas.md) Section 4 "Selection and ordering" and [analysis.md](analysis.md) Step 3 "Tiered fetch by status".

### 9. Each routine entry is independent at delivery time

Each routine entry processes one group's inlined config and emits one self-contained Slack message; there is no chapter-wide rollup — no aggregated health verdict across groups, no cross-group sorting, no shared follow-up-question pool. This keeps the per-message render cost flat regardless of how many routines a manager has.

→ See [analysis.md](analysis.md) Step 8 hand-off and [SKILL.md](../SKILL.md) Step 8 "Deliver".

## What we deliberately don't optimize

Some plausible savings the spec leaves on the table on purpose:

- **Don't try to memoize Jira tickets across runs.** Tickets change day-to-day; cache invalidation is the cost-multiplier you're trying to avoid in the first place. Refetch every run.
- **Don't share state across reportees beyond what's strictly equal** (e.g. the sprint object, only if they share a sprint). The risk of a wrong cross-reference dwarfs the saving.
- **Don't compress changelog into "interesting transitions only" pre-fetch.** The filter is already at the field level (only status / SP / sprint-field rows); below that, we need the full row for the timestamp anyway.
