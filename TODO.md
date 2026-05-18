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

