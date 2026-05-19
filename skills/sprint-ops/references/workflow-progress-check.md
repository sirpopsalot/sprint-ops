# Workflow: Mid-sprint progress check

## Required setup

Before doing anything in this workflow, resolve config:

1. Read `~/Documents/.claude-plugin-config-root` to find the config root.
2. Read `<config-root>/plugins/sprint-ops.user-context.md` for plugin config.
3. Read `<config-root>/identity.md` for shared identity (optional — skip if missing).

If the pointer file or plugin config doesn't exist, stop and tell the user to run `/setup-sprint` first.

**Goal:** at the configured mid-sprint check moment, give the user a quick sense of whether the sprint is on track, and draft a chat-ready three-line update for the tech lead + the user + the product lead.

**Read first:** `sprint-context-check.md` (always — gather meeting recaps from `config.context_sources.meetings` and chat context from `config.context_sources.chat.channel` over `config.context_sources.lookback_days` before anything else), then `scope-check.md` (decide Personal vs. Product Management scope, soft-ask if mismatched), then `team-context.md` (sprint cadence and ceremony schedule from config).


## Scope: works in either scope

This workflow runs in both Personal and Product Management scope, with different behaviors:

- **PM scope (emails in `config.team.pm_scope_emails`):** standard behavior as described below — sprint-wide reads, drafts to tickets after confirmation, sets the `config.ticket_fields.user_story_status.field_name` field to its `drafted` value when authoring, may produce team-facing artifacts.
- **Personal scope (default for everyone else):** read-only by default. Filter tracker queries to tickets the user is assigned to or following. Drafts are shown in chat only — no tracker writes, no field updates, no team-facing artifacts. If the user explicitly says "write this to `<TICKET-ID>`", confirm once more, then write, but never set the `user_story_status` field (that's a PM signal).

Announce the active scope at the top of your response in one line.

## When this runs

The default cadence assumes a two-week sprint with a mid-sprint informal chat check-in among the tech lead, the user, and the product lead. The usual question: "Are we mostly on target? Need course corrections?" This workflow produces the assessment + the draft message.

Other cadences work (one-week, three-week, continuous flow) but you may need to use judgment about *when* in the sprint to run this — the value is highest with enough sprint left to course-correct, and enough already done to read pace. For a two-week sprint, end of week 1 is the sweet spot. For shorter or longer cycles, ask the user when they want it.

It can also run later in the sprint if the user wants a "where are we?" snapshot before the deployment day.

## Step 1 — Identify the active sprint

The sprint whose date range contains today, identified by matching `config.tracker.sprint_naming_pattern` against current projects in `config.tracker.workspace_identifier`. Confirm with the user.

## Step 2 — Pull the snapshot

Use the tracker MCP to list all tasks in the active sprint project — both open and completed. Request enough field detail to read each ticket's name, assignee, the custom fields named in `config.ticket_fields` (status, planning poker, priority, ticket type), and modification timestamp. Combine open and completed lists; they're both "in scope" for the sprint.

## Step 3 — Compute the buckets

Bucket every ticket by status. The exact status names live in `config.ticket_fields.user_story_status.values` and adjacent fields, but the meaningful groupings are:

| Bucket | Interpretation |
|---|---|
| **Done** | Shipped + verified. |
| **Ready for prod / in prod testing** | Effectively done — will close when verified. |
| **In QA** | In the QA queue. |
| **In review / progress** | Active work, including code review. |
| **Not started** | Hasn't begun, including "needs design". |
| **Blocked** | Surface explicitly. |
| **Out of scope** | Canceled or no-QA-needed; don't double-count. |

If the user's tracker uses status names not obviously mapping to these buckets, ask once which statuses map where, then proceed.

Also compute:

- **Planning Poker total** of all in-scope tickets (sum of points where set, using `config.ticket_fields.planning_poker.field_name`).
- **Points done** (sum of points on tickets in Done / Ready for prod / In prod testing buckets).
- **% complete by points** (points done / total points). This is the most defensible pace metric when points are set.
- **Days remaining** in the sprint (today vs. the sprint end date).

If planning poker isn't set on most tickets (early-sprint case, or team doesn't estimate), fall back to **% by ticket count** and call that out.

## Step 4 — Identify risks

Surface specific tickets, not abstract worries. Look for:

- **Blocked tickets** — list them with the ticket ID, title, and assignee. Suggest who can unblock if the blocker is obvious from the task notes.
- **Stuck tickets** — `In Progress` for >3 days without a status change (use the modification timestamp). Worth a chat ping.
- **Tickets nobody's started late in the sprint** — `Not started` past the rough midpoint of week 2 (or the analogous point for non-two-week sprints). May not ship this sprint.
- **QA backlog** — count of tickets sitting in the QA-ready bucket. If there's a single QA owner and the queue is deep (>5), that's a risk.
- **Critical bugs late in sprint** — anything at the `critical` value of `config.ticket_fields.priority.field_name` not yet at a "ready for prod" status late in the sprint.
- **Pulled-in tickets** — tickets added mid-sprint (heuristic: `created_at` after sprint start). Flag because they push other work later.

Don't write "we might be behind" without naming what specifically is behind.

## Step 5 — Output

Two parts: the analysis (for the user) and the chat draft (for the team).

### Part A — Analysis

```
# {Sprint name} — Mid-sprint check ({today})

**Sprint window:** {start} – {end}. **Days remaining:** {n}.

## Progress
- {points_done}/{points_total} points done ({pct}%)
- {ticket_done}/{ticket_total} tickets in done/prod buckets
- Pace: {on_track | slightly behind | behind | ahead} given {days_remaining} days left

## Status distribution
| Bucket | Count | Tickets |
|---|---|---|
| Done | {n} | {<PREFIX>-XXXX, …} |
| Ready/in prod | {n} | … |
| In QA | {n} | … |
| In review/progress | {n} | … |
| Not started | {n} | … |
| Blocked | {n} | … |

## Risks
- **{<PREFIX>-XXXX}** — {title}. {Specific risk}. {Suggested action}.
- …

## Wins worth naming
- {<PREFIX>-XXXX shipped, e.g., the tech lead merged the login refactor}
- …
```

The "wins worth naming" section is short and specific. If the sprint has been hard, surface what *did* land — the team's morale benefits from that.

### Part B — Chat draft

Concise, chat-friendly. ~3–5 lines, no emoji unless the user usually uses them in this thread. Address the tech lead and the product lead.

```
**{Sprint name} mid-sprint check**

We're at {pct}% of points with {days_remaining} days left — {one-line read on pace}.

Wins this week: {1–2 specific items}.

Watching: {1–2 specific risks with ticket IDs}. {Optional: any course correction worth suggesting}.

Anything I'm missing?
```

If there's an existing voice/cadence on these threads (search prior check-ins on `config.context_sources.chat.channel`), match that style — don't impose a new template.

## Step 6 — Wait for approval before posting

Show both parts. Ask: "Want me to post the chat draft to your usual thread, or do you want to send it yourself?" If the user wants the skill to post, use the chat MCP appropriate to `config.context_sources.chat.platform` — but always present the message text first.

## Edge cases

- **Sprint is too early** (e.g., day 2 of a two-week sprint) — most tickets will be in the not-started bucket. Skip the progress check; this is too early for it to be meaningful. Suggest a later day.
- **Sprint is over** — run the retro-prep workflow instead.
- **Most tickets have no Planning Poker value** — fall back to ticket-count percentage and call out that points aren't set.
- **User wants only the chat message** — skip the analysis section; produce the message and the underlying snapshot table only.
- **Pace looks bad** — don't editorialize ("the sprint is in trouble"). Lay out the numbers and the specific risks; the user decides how to frame it.
