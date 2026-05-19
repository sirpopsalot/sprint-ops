# Team context

## Required setup

Before doing anything in this workflow, resolve config:

1. Read `~/Documents/.claude-plugin-config-root` to find the config root.
2. Read `<config-root>/plugins/sprint-ops.user-context.md` for plugin config.
3. Read `<config-root>/identity.md` for shared identity (optional — skip if missing).

If the pointer file or plugin config doesn't exist, stop and tell the user to run `/setup-sprint` first.

## Where team context lives

Team-specific context — roster, ceremony schedule, tooling, sprint cadence, ticket field names — lives in `<config-root>/plugins/sprint-ops.user-context.md`, captured during `/setup-sprint`. This file does **not** describe a specific team; it documents the *shape* of the config the skill reads and how that config drives workflow behavior.

When drafting status posts, ticket assignments, or DMs, default to first names from the roster.

## Roster schema

The `config.team.members` list describes the people the skill needs to reason about. Expected shape:

| Field | Purpose |
|---|---|
| `name` | Display name. Skill uses first-name form in chat output. |
| `email` | Used to match against meeting attendees, ticket assignees, chat handles. |
| `role` | One of the role tags below. Drives default assignment + filtering logic. |
| `notes` | Free-form. Surfaced when the skill needs nuance (e.g., "reduced availability", "QA throughout week 2"). |

`config.team.pm_scope_emails` is a separate list of emails treated as "in the product-management scope" — used to filter meetings, chat, and decisions to what the user should actually see.

## Roles the skill cares about

| Role tag | What the skill assumes |
|---|---|
| `tech_lead` | Owns technical scoping, release execution, and assignment of tech tickets. Default reviewer for technical tickets. Co-decides sprint shape with the user. |
| `product_lead` | Final call on feature prioritization. Default reviewer for product/business decisions. |
| `qa` | Reviews tickets for testability. May not attend standups. Tickets need sufficient QA info before they're ready. |
| `engineer` | Picks up tickets during sprint execution. Subject to planning-poker estimation. |
| `program_lead` | The user. See "Assumed user role" below. |
| `inactive` | Person is on the roster historically but should not be assigned forward-looking work. |
| `external` | Consumes outputs (release notes, content) but isn't part of the sprint team. |

Roles outside this list are tolerated but get no special handling.

## Sprint cadence patterns

The skill handles several common cadence shapes. Relevant config fields: `config.tracker.sprint_naming_pattern`, `config.cadence.sprint_length_weeks`, `config.cadence.releases_per_sprint`, `config.cadence.has_retro`, `config.cadence.standup_time`.

- **Sprint length.** 1-week, 2-week, or 3-week sprints. The skill resolves the *current* sprint at runtime by querying the tracker for projects matching the naming pattern — it never hardcodes a sprint number or date range.
- **Releases per sprint.** Some teams ship once per sprint; others ship weekly (so a 2-week sprint has two releases). The skill expects each release to have a marker task in the sprint's tracker project. Default naming convention is `Production Release - YYYY-MM-DD` — the date is parsed out of the task name to label release-notes sections. Override the pattern in `config.release_notes.marker_pattern` if your team uses a different naming convention.
- **Retro frequency.** Some teams retro every sprint; others retro on a cluster-of-pain basis. `config.cadence.has_retro` is a boolean default; the skill still surfaces retro topics opportunistically.
- **Refinement / product check-in.** A small-group meeting (typically tech lead + product lead + user) to review ticket status and reprioritize. Developers should not see refinement churn — tickets are expected to be fully detailed before sprint kickoff so they can be poker-estimated cold.
- **Standups.** Frequency varies (often fewer in week 1, more in week 2). The skill doesn't schedule them, but it consumes standup transcripts/notes for the activity-log workflows.
- **Planning poker.** Fibonacci: `1` (≤1 hour) · `2` (1–2 days) · `3` (3–4 days) · `5` (4–8 days) · `8+` (break into smaller tickets). When votes diverge, highest explains what lower voters might be missing; lowest explains the simpler path.

## Tooling

`config.tracker.tool` is the source of truth for work. The skill is tracker-agnostic — it describes actions ("list active projects", "set the ready-for-sprint field to its drafted value") and lets Claude pick the MCP tool based on what's installed.

Other configurable sources:

- `config.context_sources.meetings` — `m365` | `google_workspace` | `none`. Used to pull meeting attendees, transcripts, and calendar context.
- `config.context_sources.chat.platform` and `config.context_sources.chat.channel` — primary channel for team comms and async status posts.
- `config.release_notes.destination` — `outline` | `confluence` | `notion` | `markdown_file` | `slack` | `none`. Where the release-notes workflow publishes its output.

## Assumed user role

The skill assumes the user owns the scrum-process aspects of the sprint: cadence, ticket readiness, ceremony prep, release notes. It does **not** assume the user facilitates standups or runs retros — those remain team ceremonies. The user pairs with the tech lead on technical shape and the product lead on business prioritization.
