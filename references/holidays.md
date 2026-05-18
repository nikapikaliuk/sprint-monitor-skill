# Public holidays

Static lookup of public holidays per country and year. Used by [pto.md](pto.md) for capacity math and for the "Group holidays this sprint" canvas section.

Each group's config carries two country-related fields:
- `country` (default `"PL"`) — the group's own country. Holidays from this list **reduce working-day capacity** group-wide and surface in the canvas/delta info block.
- `info_countries` (default `["IL"]`) — additional countries whose holidays surface as **info only**. Useful for cross-team coordination (e.g. Polish team needs to know when Israeli colleagues are off, even though it doesn't change the Polish team's own capacity).

## Format

Each country's list is one fenced `json` code block per year. The skill reads the block matching the group's `country` and current year. Dates are ISO `YYYY-MM-DD`. Weekend holidays (Sat/Sun) are still listed for human reference, but the capacity math skips them (only Mon–Fri holidays reduce working days).

```json
{
  "country": "PL",
  "year": 2026,
  "holidays": [
    { "date": "2026-01-01", "name": "New Year's Day", "weekday": "Thu" },
    { "date": "2026-01-06", "name": "Epiphany", "weekday": "Tue" },
    { "date": "2026-04-05", "name": "Easter Sunday", "weekday": "Sun" },
    { "date": "2026-04-06", "name": "Easter Monday", "weekday": "Mon" },
    { "date": "2026-05-01", "name": "Labour Day", "weekday": "Fri" },
    { "date": "2026-05-03", "name": "Constitution Day", "weekday": "Sun" },
    { "date": "2026-05-24", "name": "Pentecost Sunday", "weekday": "Sun" },
    { "date": "2026-06-04", "name": "Corpus Christi", "weekday": "Thu" },
    { "date": "2026-08-15", "name": "Assumption of Mary", "weekday": "Sat" },
    { "date": "2026-11-01", "name": "All Saints' Day", "weekday": "Sun" },
    { "date": "2026-11-11", "name": "Independence Day", "weekday": "Wed" },
    { "date": "2026-12-25", "name": "Christmas Day (Boże Narodzenie, day 1)", "weekday": "Fri" },
    { "date": "2026-12-26", "name": "Second Day of Christmas (Boże Narodzenie, day 2)", "weekday": "Sat" }
  ]
}
```

## Polish 2026 (PL) — verification notes

Easter Sunday 2026 = April 5, so Easter Monday = April 6. Pentecost = 49 days after Easter = May 24 (Sun). Corpus Christi = 60 days after Easter = June 4 (Thu).

**Effective working-day reductions in 2026** (Mon–Fri only):
- Jan 1 Thu, Jan 6 Tue, Apr 6 Mon, May 1 Fri, Jun 4 Thu, Nov 11 Wed, Dec 25 Fri = **7 working days off**

The skill should always look up holidays for the **current calendar year** at run time. When the calendar rolls over to 2027, add a new code block for 2027 PL (and any other countries the team adopts).

## Israel 2026 (IL)

Used as an `info_countries` entry by default — surfaces Israeli holidays in the canvas/delta info block so the Polish team knows when Israeli colleagues are off, without affecting Polish-side capacity. Hebrew-calendar dates shift each year; **verify before relying on them for any cross-team planning**. Half-day holidays where most offices close early are flagged with `"half_day": true`.

```json
{
  "country": "IL",
  "year": 2026,
  "holidays": [
    { "date": "2026-03-03", "name": "Purim", "weekday": "Tue", "half_day": true },
    { "date": "2026-04-01", "name": "Passover Eve (Erev Pesach)", "weekday": "Wed", "half_day": true },
    { "date": "2026-04-02", "name": "Passover (Pesach, day 1)", "weekday": "Thu" },
    { "date": "2026-04-08", "name": "Passover (Pesach, 7th day)", "weekday": "Wed" },
    { "date": "2026-04-14", "name": "Holocaust Remembrance Day (Yom HaShoah)", "weekday": "Tue" },
    { "date": "2026-04-21", "name": "Memorial Day (Yom HaZikaron)", "weekday": "Tue", "half_day": true },
    { "date": "2026-04-22", "name": "Independence Day (Yom Ha'atzmaut)", "weekday": "Wed" },
    { "date": "2026-05-21", "name": "Shavuot Eve (Erev Shavuot)", "weekday": "Thu", "half_day": true },
    { "date": "2026-05-22", "name": "Shavuot", "weekday": "Fri" },
    { "date": "2026-07-23", "name": "Tisha B'Av", "weekday": "Thu", "half_day": true },
    { "date": "2026-09-12", "name": "Rosh Hashanah (day 1)", "weekday": "Sat" },
    { "date": "2026-09-13", "name": "Rosh Hashanah (day 2)", "weekday": "Sun" },
    { "date": "2026-09-21", "name": "Yom Kippur", "weekday": "Mon" },
    { "date": "2026-09-26", "name": "Sukkot (day 1)", "weekday": "Sat" },
    { "date": "2026-10-03", "name": "Simchat Torah", "weekday": "Sat" }
  ]
}
```

**Verification needed:** the dates above are best-effort conversions of the Hebrew calendar; the manager should sanity-check against an authoritative IL calendar before the next sprint that includes one of these dates and edit this file as needed.

## Other countries

Add a section per country as the team's footprint grows. Common ones to keep handy:

- BY (Belarus — engineering team)
- US (US-based observers)

These aren't filled in for v1 — add when a group's `country` or `info_countries` is set to a new code. For unknown codes, the skill falls back to Mon–Fri only (no holidays) and surfaces a warning: *"⚠️ Group `<slug>` country is `<code>` but no holiday list exists. Add a `<code> <year>` block to references/holidays.md."*

## When the list is wrong

If a holiday is missing, wrong, or a year's list isn't filled in yet, the capacity math just under-counts holiday PTO (treats those days as working days). Not catastrophic — the manager can correct by adding explicit PTO entries to specific reportees, or by editing this file.

This file is meant to be edited by hand. The skill reads but never writes it.
