# PTO and sick leave

How the skill tracks time-off, scales capacity, and surfaces it in the canvas + delta. Covers vacation, sick leave, and public holidays (which behave like group-wide PTO for everyone in a given country).

## Storage

PTO entries live in each reportee's block in the **inline config in the run prompt** (see [../SKILL.md](../SKILL.md) "Config schema"). There is no file-based PTO store inside the skill repo — every routine fires with whatever PTO the prompt author has inlined, and the canvas run also fetches OOO live from Google Calendar and merges in-memory for that run only.

```json
{
  "name": "Bob Smith",
  "email": "bob.smith@example.com",
  "jira_account_id": "557058:00000000-0000-0000-0000-000000000001",
  "pto": [
    {
      "type": "vacation",
      "start": "2026-05-22",
      "end": "2026-05-26",
      "working_days": 3,
      "source": "manual: 'Bob is on vacation next week, Mon to Wed'"
    },
    {
      "type": "sick",
      "start": "2026-05-20",
      "end": "2026-05-20",
      "working_days": 1,
      "source": "manual: 'Bob called in sick today'"
    }
  ]
}
```

Per entry:

| Field | Description |
|---|---|
| `type` | `vacation` or `sick`. Public holidays don't go here — they live in the holiday lookup ([holidays.md](holidays.md)) and apply group-wide. |
| `start`, `end` | Inclusive date range. Same date for a one-day absence. |
| `working_days` | Pre-computed working-days count for this range (excludes weekends but NOT public holidays — that double-counts if the team observes the holiday anyway). |
| `source` | Verbatim user input or `"google_calendar:<event_id>"` for synced entries. Useful for audit when the parse is ambiguous later. |

Calendar-synced entries are added to the array **in memory** during Step 1b of a canvas run; they're not written back anywhere. The canvas run sees the merged set; the next canvas run re-fetches from the calendar fresh.

## Group-level country settings

The inline config carries two country-related fields at the top level (see [../SKILL.md](../SKILL.md) "Config schema"):

```json
{
  "name": "alice",
  "country": "PL",
  "info_countries": ["IL"],
  "jira_base_url": "...",
  ...
}
```

- `country` — the group's own country. Holidays from this list **reduce working-day capacity** group-wide (Mon–Fri holidays subtract from the 10-day sprint base). Default is `"PL"`.
- `info_countries` — array of additional country codes whose holidays surface in the canvas/delta as **info only**. Capacity math ignores them. Useful for "the Polish team should know when Israeli colleagues are off, but it doesn't change the Polish team's own capacity." Default is `["IL"]`.

Both fields look up their holiday list from [holidays.md](holidays.md).

If `country` is missing or unknown, treat as `"PL"` and surface a one-line note in the next report: *"⚠️ No country set for this group — assuming Polish holidays. Add `\"country\": \"<code>\"` to the inline config to suppress this note."* `info_countries` missing is silent (defaults to `["IL"]`).

## Google Calendar OOO sync

The skill auto-discovers PTO from each reportee's Google Calendar at the start of every canvas run. This is the **primary and required** PTO source — the manager doesn't have to enter time-off manually as long as reportees keep their OOO on a calendar. Natural-language parsing (local mode only) and any pre-existing prompt-inlined entries still work and take precedence in conflicts, but they're supplements, not substitutes.

### When it runs

- **Canvas runs** (Mon / Thu) — **sprint-window fetch.** Pull every all-day OOO event overlapping `sprint.startDate → sprint.endDate` for each reportee. This is the load-bearing fetch — capacity math reads from it.
- **Delta runs** (Tue / Wed / Fri) — **today-only fetch.** Pull every all-day OOO event overlapping just today, per reportee. Much smaller than the canvas fetch (no historical events, no future events past today), and only feeds the **Today** block in [output/delta.md](output/delta.md) — capacity math does not run on deltas. Added so ongoing mid-vacation PTO ("Eve is out today, day 3 of 5") surfaces in the daily delta; without it, the delta only sees PTO whose `start` or `end` equals today.
- Runs after the inline config is parsed, before any output assembly — the synced entries need to be in the in-memory `pto[]` arrays for downstream sections to read.
- **Auth handling differs by mode:**
  - **Canvas:** hard-error if the Google Calendar MCP isn't authenticated. Capacity math without the calendar produces misleading verdicts; silently skipping the sync would let a Borderline call sit on top of unreflected PTO. Emit *"Sprint Monitor cannot run: Google Calendar MCP not authenticated. PTO sync is required for accurate capacity math."* and stop.
  - **Delta:** soft-degrade. If the calendar MCP isn't authenticated, fall back to inlined-only PTO and add a one-line note at the bottom of the delta: *"⚠️ Calendar not authenticated — PTO only from inline config."* Don't hard-fail — the delta is still useful without it.

### What it queries

For each reportee in the inlined config, query their primary calendar via the Google Calendar MCP for events whose date range overlaps the current sprint window (`sprint.startDate` → `sprint.endDate`). The reportee's `email` field is the lookup key; if a reportee has a separate calendar address, the prompt config can carry an optional `gcal_email` field on their block and the skill uses that instead.

**Event filter:**
- **All-day events** — always counted (the canonical OOO shape).
- **Timed events with a PTO-pattern title** — counted only when they meaningfully consume the workday. Compute the event's overlap with the **working window** (9:00–19:00 in the reportee's local timezone). Apply the **6-hour threshold**:
  - **≥6 hours of working-window overlap** in a single day → count the day as OOO.
  - **<6 hours** → ignore. A 2-hour doctor's appointment or a half-day errand isn't a real absence for capacity / availability purposes; treating it as OOO produces false "Eve is out today" lines that make the manager chase phantoms.
  - **Multiple timed OOO events on the same day:** sum their overlaps with the working window, then apply the threshold to the sum. (Two 3-hour blocks → 6h total → counted. A 2h block + a 3h block → 5h → ignored.)
  - **Multi-day timed events** (rare, e.g. a single calendar event spanning Mon 14:00 → Wed 11:00): split per local-calendar day, apply the threshold to each day independently. Days where the in-window overlap is <6h are dropped.
- Title matches the case-insensitive PTO pattern: substring match on any of `vacation`, `pto`, `ooo`, `out of office`, `out of the office`, `sick`, `leave`, `holiday`, `away`. Override per group via an optional `pto_calendar_patterns` field on the inline config.
- Working window: 9:00–19:00 local time. Override per group via an optional `working_window: { "start": "HH:MM", "end": "HH:MM" }` field on the inline config — out of scope for v1 in most cases; default works for typical knowledge-worker teams.
- Status `accepted` or unspecified — skip events the reportee has `declined`.

### Mapping events → `pto[]` entries

For each matching event that passed the filter (all-day, or timed with ≥6h working-window overlap), build an in-memory entry:

| Source field | → Target field |
|---|---|
| Title (matched keyword) | `type` — `sick` if title contains `sick`; otherwise `vacation`. |
| `start.date` (all-day) **or** the local-calendar date of `start.dateTime` (timed event passing the threshold) | `start`. |
| `end.date` minus 1 day (all-day; Google's `end.date` is exclusive) **or** the local-calendar date of `end.dateTime` (timed event) | `end`. |
| Computed per [working-days math](#working-days-math); for timed events that pass the threshold, the day still counts as 1 working day | `working_days`. |
| `"google_calendar:<event_id>"` | `source` — preserves the event id for traceability if a manager asks where an entry came from. |

The `source` prefix `google_calendar:` is how the skill distinguishes synced entries from prompt-inlined ones during reconciliation in the same run.

### Reconciliation

Conflicts between a synced entry and a prompt-inlined entry are resolved per reportee, in order:

1. **Prompt-inlined entry that overlaps a synced event:** keep the inlined entry, drop the synced one. The manager has the more authoritative read; calendars can be stale.
2. **No matching prompt-inlined entry:** insert the synced entry into the in-memory `pto[]` array for this run.

Prompt-inlined entries are never modified or dropped by the sync — they only get retained.

### No persistence

The sync result lives **in memory only**, scoped to the current canvas run. The skill does NOT write back to any file (there is no file). The next canvas run re-fetches the calendar fresh, which means:

- If a reportee cancels a vacation in their calendar, the next canvas run won't see it — correct behavior.
- If the manager wants an entry to survive across runs *and* it's not in the calendar, they need to either add it to the calendar or update the routine's inlined config. The skill cannot "remember" a parsed PTO entry across firings.

### Failure modes

- **One reportee's calendar is private / not shared with the skill's auth identity:** log the skip per-reportee and continue. Sync still applies for everyone else.
- **Calendar MCP returns an error mid-loop:** stop the sync, keep whatever was synced before the error, render canvas with that partial state, surface a note.
- **Rate limit:** retry once with backoff; if it fails again, treat like the MCP error case.

The sync is **best-effort context**, not a load-bearing dependency. The canvas always renders even when sync fails entirely — the PTO suffix on the Capacity overview just shows manually-committed entries only.

## Parsing user input

The skill detects PTO statements in natural-language prompts during local-mode runs. The manager talks about it conversationally; the skill parses, adds the entry, and surfaces what it did.

### Recognition triggers

The skill should recognize any of these phrasings (case-insensitive, fuzzy on the action verb):

- *"<name> is on vacation/PTO/holiday/leave"*
- *"<name> is taking time off"*
- *"<name> is out"* (treat as vacation unless context suggests sick)
- *"<name> is sick / out sick / called in sick / on sick leave"*
- *"<name> is OOO"*

Plus duration / date qualifiers:
- *"today"* / *"tomorrow"* / *"this Friday"* — resolve against today's date
- *"next week"* — Mon–Fri of next week (5 working days)
- *"for a week"* — **5 working days** starting from the resolved start date (per manager rule, not 7 calendar days)
- *"for N days"* — N working days starting from the start date
- *"from X to Y"* — explicit date range, count working days in it
- *"Mon to Wed"* / *"22 to 26 May"* — explicit date range, same rule

### Disambiguation rules

When the parse is unambiguous (clear name, clear dates, clear type), **add the entry immediately and confirm what was added in the reply**. Don't ask permission for confident parses.

Ask when:
- The reportee name matches multiple people in the group (e.g. "Bob" with two Bobs on the team) — list candidates, ask which.
- The duration is `"7 days"` or `"14 days"` exactly — this is genuinely ambiguous (could be calendar week or 7 working days). Ask: *"Is that 7 working days or 7 calendar days (which would be 5 working days)?"*
- The reportee name doesn't match anyone in the group — ask whether it's a typo or someone outside the group (and skip if outside).
- The type is unclear (e.g. *"Bob is out tomorrow"* — vacation or sick? Default to vacation, mention in the reply that the assumption can be corrected).

If running in **routine mode** (no interactive user), parsing only applies to whatever PTO entries are already inlined in the prompt config (plus whatever the Google Calendar sync pulls in live). The skill doesn't receive new natural-language input in routine mode. Skip the parse path entirely.

### Working-days math

A "working day" is Mon–Fri AND not a public holiday in the group's country.

Examples (group country = PL, holidays per [holidays.md](holidays.md)):

- *"Bob is out tomorrow"* (today is Wed 2026-05-20) → `start=2026-05-21`, `end=2026-05-21`, `working_days=1`. Thu is not a Polish holiday in May, so 1 day.
- *"Bob is on vacation May 1 to May 5"* → `start=2026-05-01`, `end=2026-05-05`. That range is Fri–Tue: May 1 is a Polish holiday (Labour Day), May 2–3 are Sat–Sun, May 4 is Mon, May 5 is Tue. Effective `working_days=2`.
- *"Bob is taking a week off starting Mon 25 May"* → `start=2026-05-25`, `end=2026-05-29`. 5 working days, no holidays in that range.
- *"Bob is out for 7 days"* → ask: working days or calendar days? If unable to ask, default to 7 working days (per manager rule).

## Capacity math (scaling)

The base capacity is **80h per 10 working days** in a 2-week sprint. If a reportee has PTO during the sprint window, scale their capacity proportionally:

```
working_days_in_sprint = 10 − (PTO + sick + holidays falling inside sprint window)
capacity_hours = 80 × working_days_in_sprint / 10
```

Then scale the verdict thresholds the same way:

| Verdict | Threshold (full 80h) | Threshold formula |
|---|---|---|
| Underworked | <65h | `< 0.8125 × capacity_hours` |
| Normal | 65–80h | `0.8125 × capacity_hours` to `1.0 × capacity_hours` |
| Borderline | 80–100h (or 4+ × 4-SP) | `1.0 × capacity_hours` to `1.25 × capacity_hours`, or 4+ × 4-SP unchanged |
| Overbooked | >100h (or 5+ × 4-SP) | `> 1.25 × capacity_hours`, or 5+ × 4-SP unchanged |

The 4-SP / 5-SP bucket counts don't scale — those are about ticket-count shape, not hours. Someone with 5 chunky tickets is overbooked regardless of how many days they're at work.

### Worked example

Bob has 2 days of vacation this sprint:
- `working_days_in_sprint = 10 − 2 = 8`
- `capacity_hours = 80 × 8/10 = 64h`
- Verdict thresholds:
  - Underworked: <52h
  - Normal: 52–64h
  - Borderline: 64–80h
  - Overbooked: >80h

If Bob's `sprint_hours = 70h`, he's Borderline (where on full capacity he'd have been Normal). Manager should consider trimming.

### Holidays inside the sprint window

If May 1 (Polish Labour Day) falls inside the current sprint window, treat it as 1 working day deducted from **every reportee in the group**. The capacity math runs the same — only the count of working days changes.

When a reportee already has a PTO entry covering a public holiday, **don't double-count it**. Compute the union of PTO days + holiday days, not the sum. Bob taking vacation Apr 30 – May 5 in a sprint that contains May 1 (holiday) = 5 days off, not 6.

## Display

### In the canvas ([canvas.md](output/canvas.md))

**1. Capacity overview line:** append a PTO suffix when a reportee has time off this sprint.

```
🟢 **Bob Smith** · ALPHA + BETA · Normal (94%) · 🌴 PTO May 22–26 (3 days)
🤒 **Eve Martinez** · DELTA + ZETA · Underworked (40%) · sick today
```

Emoji map:
- 🌴 vacation
- 🤒 sick
- 🎉 public holiday (when the whole group is off — Group-level note, not per reportee, see below)

If the reportee has multiple PTO entries, comma-separate: `🌴 PTO May 22 (1d), 🤒 sick May 14 (1d)`. Only show entries with at least 1 day inside the current sprint window — past entries outside the sprint don't clutter the line.

**2. New section: "Out this sprint"** — between Capacity overview and Concerns, only when at least one reportee has personal PTO (vacation or sick) inside the sprint window:

```
## Out this sprint

🌴 Bob — May 22, 25, 26 (3d vacation)
🤒 Eve — May 20 (sick today)
```

Order: today first (anyone out *today* leads), then upcoming dates, then past dates inside the current sprint. Personal PTO only — public holidays do NOT appear here (they have a separate info block, see below).

If no personal PTO at all this sprint, omit this section.

**3. New info block: "Group holidays this sprint"** — between "Out this sprint" and Concerns, only when at least one country holiday (group's `country` or any `info_countries`) falls inside the sprint window:

```
## Group holidays this sprint

🇵🇱 Poland — May 24 (Pentecost Sunday, weekend), Jun 4 (Corpus Christi, Thu)
🇮🇱 Israel — May 22 (Shavuot, Fri)
```

Rules:
- One line per country. `country` first, then each `info_countries` entry in array order.
- Each line lists every holiday from that country's list whose date falls inside the current sprint window — past, today, and upcoming.
- Format per holiday: `<date> (<name>, <weekday>)`. Group multiple on one line separated by commas.
- This block is **info-only display**. The capacity math has already applied the `country` reductions silently in Step 6; this section doesn't re-explain the math.
- Skip a country's line entirely if it has no holidays in the sprint window. Skip the whole section if all countries are empty.
- Public holidays are **never** listed under personal PTO ("Out this sprint") or in any reportee's `pto` array. They live only in [holidays.md](holidays.md) and surface only here.

**3. Capacity breakdown** — when a reportee has PTO, show the scaled budget in the section header:

```
### Bob Smith — Normal (60h / 64h budget) · 🌴 2 days PTO
```

The total budget is 64 instead of 80, and the verdict reflects the scaled thresholds.

### In the delta ([delta.md](output/delta.md))

All PTO and holiday surfaces live in the delta's **Today** block at the very top of the message — see [output/delta.md](output/delta.md) "Today block" for the full spec. Summary:

**1. 🌴 Out today** — every reportee who is on PTO or sick *today*, sourced from the union of inlined `pto[]` arrays + the delta-time Google Calendar fetch. One collective line:

```
🌴 Out today: Bob (vacation), Eve (sick)
```

This catches both window-edge PTO ("Eve's vacation starts today") and ongoing mid-vacation PTO ("Bob is day 3 of 5"). Earlier spec revisions only surfaced edges (a "PTO updates" category for `start == today` or `end == today`); that missed people sitting in the middle of a vacation, which is the most common case. The "Out today" line replaces the edge-only category — it covers both, more legibly.

**2. 🌴 PTO starts today / ends today** — *separately,* call out window-edge transitions if you want extra detail on what changed at midnight. Optional and only shown when it adds info beyond "Out today" (e.g. "Bob's sick leave ended today, back tomorrow"). Skip otherwise.

**3. 🇵🇱 Country holiday today / tomorrow** — if today is a public holiday for the group's `country` or any `info_countries`, render a line per country. `country` first, then `info_countries` in array order. The "tomorrow" half is a heads-up for IL erev half-days (see [holidays.md](holidays.md) `half_day` flags).

```
🇵🇱 PL holiday today: Labour Day (May 1)
🇮🇱 IL holiday today: Shavuot (May 22)
🇮🇱 IL holiday tomorrow: Shavuot (Fri 22 May)
```

Bare lines — just flag, country code, name, date. The `country` line's capacity reduction was already applied silently on the most recent canvas; the `info_countries` line never affects capacity. Neither needs an inline explanation.

## Edge cases

- **Manager corrects a recent entry (local mode)**: the manager might say *"actually Bob is only out Mon-Tue, not the whole week"*. Detect the correction in the prompt and update the in-memory entry for this run. The correction does NOT survive to future runs — the next routine fires with whatever PTO is inlined in its prompt + the live calendar fetch. For durable corrections, the manager updates either the calendar event or the routine's inlined config.
- **PTO covers the whole sprint**: capacity_hours = 0, no verdict computed for that reportee. Display: `🌴 Bob — out the whole sprint (PTO). Not counted in capacity math.`. Skip them from per-reportee blocks in Concerns and Capacity breakdown.
- **Past PTO entries inlined in the prompt**: filter out at display time. They don't clutter the report.
- **Sprint boundaries**: if a PTO range crosses sprint boundaries (e.g. vacation Apr 28 – May 2 over a sprint that ends Apr 30), count only the days inside the current sprint window for capacity math.
- **Weekend-only PTO**: if a manager somehow logs PTO for Sat-Sun only, `working_days = 0`. Don't error; just keep the entry and ignore for capacity.
- **Auto-apply Polish holidays**: holidays are NOT stored per-reportee. They're loaded from [holidays.md](holidays.md) based on the group's `country` field and applied to everyone. If a reportee is in a different country than the group (rare, but possible), add a per-reportee `country_override` field — out of scope for v1.
