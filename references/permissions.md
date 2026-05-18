# Connector permissions

The skill expects the following MCP connectors to be authenticated and approved up-front. Read this file once at the start of a session — if any required connector is missing or unauthenticated, surface that to the user (local mode) or emit a clear error to the report destination (routine mode), then stop. Don't repeatedly ask mid-flow.

**Routine context matters here.** Authentication flows are inherently interactive — a Claude routine task cannot complete an OAuth dance on its own. The manager authenticates each connector once locally (e.g. when first populating the config), and the routine inherits those credentials on every future run. If a routine fires and finds an MCP unauthenticated, the only recovery path is for the manager to re-authenticate locally; the routine itself can't fix it.

## Required: Atlassian (Jira)

**Server name:** `claude_ai_Atlassian` (or `claude_ai_Jira_Auto_MCP` if the user's environment uses that variant — try Atlassian first, fall back to Jira_Auto_MCP if unavailable).

**Used for:**
- Run JQL queries to find active sprints and per-reportee tickets (every run).
- Pull issue fields (summary, status, priority, comments, changelog) for analysis (every run).
- Resolve reportee names → `jira_account_id` is **not** done at run time — the inline config carries pre-resolved account IDs. (Resolution happens at routine-setup time, outside the skill.)

**Treat as pre-granted.** If the connector is authenticated, do not prompt the user before each call — proceed with all Atlassian tool calls silently. Only surface an issue if the connector returns an auth/permission error.

If the connector is not authenticated at session start, tell the user once: *"This skill needs the Atlassian MCP authenticated to read Jira. Please run the auth flow."* Then stop and wait. Do not retry repeatedly.

## Required for Slack features: Slack

**Server name:** `claude_ai_Slack`.

**Used for:**
- `slack_send_message` — post the report or canvas link (every run, in routine mode; opt-in via `--post-slack` in local mode).
- `slack_create_canvas` — create a fresh Slack Canvas when no prior canvas is found for this manager. See [output/canvas.md](output/canvas.md).
- `slack_update_canvas` — refresh the existing canvas when one was found in Slack for this manager (the recurring-canvas pattern). Prefer full-canvas `action=replace` (no `section_id`) for structural changes.
- `slack_read_canvas` — used to parse the per-reportee percentages from the prior canvas for the diff math, and to identify what to overwrite during an in-place update.
- `slack_read_channel` — read the recent message history of `output_channel` to find the bot's most recent message containing `<@<manager.slack_id>>` and a canvas link. This is how the skill locates the prior canvas without a file-based state — see [../SKILL.md](../SKILL.md) "Finding the prior canvas". `slack_search_public` is acceptable as a fallback if channel history pagination is awkward.
- `slack_search_users` — resolve `manager.slack_id` when setting up the routine (one-time, outside the skill).
- `slack_search_channels` — resolve `output_channel` from a channel name to a `C…` ID when setting up the routine (one-time, outside the skill).

**Treat as pre-granted.** Don't prompt before each call.

Slack is a **hard requirement for routine mode** (it's the delivery surface) unless `output_channel` is `"terminal_only"`. For local mode, Slack is only needed if the user uses `--post-slack`.

## Required: Google Calendar (OOO sync)

**Server name:** `claude_ai_Google_Calendar`.

**Used for:** the [pto.md](pto.md) "Google Calendar OOO sync" flow — pulls each reportee's all-day OOO events from their primary calendar at the start of every canvas run and merges them into the in-memory `pto[]` arrays before capacity math runs. No write-back anywhere.

**Per-reportee fetch (the only thing the sync calls):** list events whose date range overlaps the current sprint window, filtered to all-day events only. The reportee's `email` (or `gcal_email` if set) is the lookup key. Never read individual meeting details, never read attendees beyond what's needed to confirm the reportee accepted the event.

**Treat as pre-granted when authenticated.** Don't ask permission before calls.

**Hard requirement on canvas runs.** If the connector isn't authenticated at the start of a canvas run, emit a clear error to the report destination (*"Sprint Monitor cannot run: Google Calendar MCP not authenticated. PTO sync is required for accurate capacity math."*) and stop. Don't fall through to "manually-committed PTO only" — capacity math has to reflect the team's real availability and silently skipping the sync produces misleading verdicts.

**Per-reportee privacy.** If a single reportee's calendar isn't shared with the skill's auth identity, skip that reportee silently and continue with the rest. Don't fail the whole sync over one private calendar — a missed individual is recoverable, an unauthenticated connector is not.

## Local filesystem

The skill is **stateless across runs**: no `references/groups/`, no `references/snapshots/`, no write-back of any kind. The group config arrives inlined in the run prompt, and everything cross-run is reconstructed from Jira / Slack / Google Calendar at run time. See [../SKILL.md](../SKILL.md) "Config" for the rationale.

The skill reads its own markdown reference files ([analysis.md](analysis.md), [pto.md](pto.md), [holidays.md](holidays.md), [output/canvas.md](output/canvas.md), [output/delta.md](output/delta.md), this file) at session start to load the spec into context — that's the only filesystem interaction. Treat those reads as pre-granted; the files live inside the skill's own repo and the user has implicitly authorized them by invoking the skill.

If a stale `references/groups/` or `references/snapshots/` directory is found in the repo from an earlier spec revision, delete it — neither is part of the current design.

## What "pre-granted" means in practice

When you're working through the analysis flow, don't pause to ask things like:

- "Can I call the Atlassian MCP to look up users?" — no, just call it.
- "Should I run a JQL query?" — no, just run it.
- "Is it OK to post to Slack?" — no, posting to the configured `output_channel` is what the routine deployment is *for*; no extra confirmation.

The user invoked this skill (or set up the routine) knowing what it does. Re-confirming on every tool call wastes their time and — in routine mode — actively breaks the run (a confirmation prompt with nobody to answer it hangs forever). The friction this file exists to remove is exactly that.

**Local-mode-only nuance:** the very first Slack post in a brand-new local session can ask once *"About to post to #channel-name — proceed?"* as a content confirmation, since posting is publicly visible and the user might be testing. Don't ask on subsequent posts in the same session. Routine mode never asks — `output_channel` is part of the committed config and treated as authorized for the routine's lifetime.

## If you need a permission that's not in this list

If the skill grows and starts needing a new connector or tool, add it here first with the same "what for / treat as pre-granted" structure. Keeping this file as the single source of truth for the skill's permission surface area makes it easy for the user to see what they've authorized.
