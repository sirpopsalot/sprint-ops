# Workflow: Retro prep + retro → tickets

## Required setup

Before doing anything in this workflow, resolve config:

1. Read `~/Documents/.claude-plugin-config-root` to find the config root.
2. Read `<config-root>/plugins/sprint-ops.user-context.md` for plugin config.
3. Read `<config-root>/identity.md` for shared identity (optional — skip if missing).

If the pointer file or plugin config doesn't exist, stop and tell the user to run `/setup-sprint` first.

This file covers two related modes:

- **Retro seeding** — before retro, scan the just-finished sprint for concrete pain points, churn, and patterns worth discussing. Produces a list of candidate retro topics.
- **Retro → tickets** — after retro, take the action items captured in meeting notes and turn the actionable ones into tickets in the right place.

**Read first:** `sprint-context-check.md` (always — gather `config.context_sources.lookback_days` of meeting recaps + `config.context_sources.chat.channel` context before anything else), then `scope-check.md` (decide Personal vs. Product Management scope, soft-ask if mismatched), then `ticket-quality.md`.


## Scope: Product Management only

This workflow is PM-only. If the user is in Personal scope (default for non-PM users), don't run it. Soft-ask:

> This workflow is Product Management work — it produces team-facing output and writes to tickets across the sprint or backlog. You're in Personal scope by default. Want to switch to PM scope, or did you mean something narrower (e.g., "what should *I* personally bring up at retro" / "which of *my* tickets aren't ready")?

If the user says "switch to PM", proceed. If they describe a personal version, run a scoped-down read-only variant (filtered to their tickets, no team artifacts).

## Mode A — Seeding the retro

### Step 1 — Identify the just-finished sprint

The one whose end date is most recently in the past (or, if you are mid-week-2, the active sprint anticipating a retro at week-2 end). Use the sprint identification procedure in `sprint-context-check.md` — match against `config.tracker.sprint_naming_pattern`.

### Step 2 — Pull the data

Use the tracker MCP to fetch the full sprint: list all tasks in the resolved sprint project, including both completed and incomplete items (both are signal). Request name, ID, assignee, completion status, custom field values, and created/modified timestamps.

Also pull the sprint's standup and retro-relevant meeting notes if accessible via `config.context_sources.meetings` — they often capture qualitative themes (e.g., "log noise is making debugging hard") that don't show up in tracker fields.

### Step 3 — Scan for retro patterns

Look across the sprint for these patterns. For each, surface specific tickets/incidents — abstract themes don't drive useful retros.

| Pattern | What to look for |
|---|---|
| **Unplanned work pulled in mid-sprint** | Tickets created after sprint start. Why? Were the criteria for a mid-sprint pull clear? |
| **Tickets that rolled over** | Items in this sprint with creation date before the sprint started — i.e., carried from the previous sprint. Repeated rollovers are a planning signal. |
| **Reopened tickets** | Items that bounced back from QA, or items that were marked complete and then re-opened. |
| **Production incidents** | Hotfixes, urgent prod issues, broken releases. Cross-reference with release notes if available. |
| **QA bottlenecks** | Did tickets sit waiting on the QA lead for >2 days? |
| **Ambiguous tickets** | Tickets where developers asked clarifying questions in standup or chat. Those are signals AC was insufficient. |
| **Estimation misses** | Tickets estimated as 1–2 points that took the whole sprint. Or 5-pointers that landed in a day. Both are calibration data. |
| **Recurring themes from prior retros** | If the team identified a problem in a prior retro and it's still showing up in standups, that's a retro topic on its own. |
| **Process friction** | Mentions of "I didn't know what was assigned to me", "the notification was confusing", "duplicate ticket". |
| **Wins to keep doing** | Things that worked. Successful poker session. Clean deploy. Tight QA loop. Don't make retros only about pain. |

Scan, don't grade. The goal is candidate items for the team to discuss — not the user/Claude pre-deciding what the team's problems are.

### Step 4 — Output the retro board

Produce a structured list the user can drop into a retro board. Use these columns:

```
# Sprint {N} retro — candidate topics

## Went well
- {Specific item with <PREFIX>-XXXX ID or context}
- …

## Could go better
- {Specific item}
- …

## Patterns / themes
- {Cross-cutting observation, e.g., "Three tickets bounced from QA — worth checking AC quality before dev"}
- …

## Process / tooling
- {e.g., "Two duplicates created mid-sprint — create-from-feedback flow needs guardrails"}
- …
```

Each item is one bullet, ≤2 lines, and references specifics (ticket ID, person, day). Don't soften pain points but also don't sharpen them — the team will discuss.

Tell the user: "Here are candidate items. Want me to set up a retro board with these as cards, or do you want to bring them to retro as-is?" Wait for direction.

### When the source is a retro transcript

If the user pastes a retro meeting transcript (instead of asking for a pre-retro scan), look for:

- Action items the team agreed to ("we'll start writing emergency runbooks", "the tech lead to handle prod releases independently").
- Process changes the team committed to ("switching estimation to Fibonacci", "splitting refinement into product check-ins").
- Unanswered questions worth a follow-up ticket ("we need to figure out staging environment").

Convert those into the Mode B output (below) and skip the seeding work.

## Mode B — Retro action items → tickets

Once retro action items are agreed (in meeting notes or in a chat thread), turn the actionable ones into tickets.

### Step 1 — Classify each action item

For each action item, decide:

1. **Is this a ticket-shaped commitment?** Things like "implement Playwright config" are tickets. Things like "we'll experiment with later standup time" are not — those are calendar/process changes that don't need a tracker entry.
2. **Is it product, tech debt, or process?**
   - Product → backlog (product section) or sprint
   - Tech debt → backlog (tech section) or sprint
   - Process → could be a single tracking ticket "Retro N action items" with subtasks, OR posted in chat for accountability without a ticket. Ask the user.
3. **Who owns it?** The retro notes usually name an owner. Use that as the assignee. Match against `config.team.members`.
4. **What sprint?** Default to the *next* sprint if the item is high-priority, otherwise the backlog at `config.tracker.backlog_project_id`. You can promote at the next product check-in.

### Step 2 — Draft the tickets

Use the standard ticket-quality template. For retro items specifically:

- Title pattern: `[Retro N] [Action]` — e.g., `[Retro 14] Implement Playwright e2e testing config`. The `[Retro N]` prefix makes it obvious where the work originated.
- Acceptance criteria can be terse for retro items — sometimes just "Action item from Sprint N retro implemented to the team's satisfaction" with a link to the retro notes. Process items don't always have testable AC.
- Set `config.ticket_fields.ticket_type.field_name` to the tech-debt value for most retro items unless they're clearly product or admin features.
- Priority (`config.ticket_fields.priority.field_name`) defaults to high for retro action items (the team committed to them publicly), unless retro itself classified differently.

### Step 3 — Present and create

Present the draft tickets in the same table format as backlog triage. Wait for the user's approval. Use the tracker MCP to create after approval. Comment on each created ticket linking back to the retro meeting notes so the trace is preserved.

## Edge cases

- **Retro hasn't been scheduled yet** — produce the candidate topics anyway. You can use them to decide if a retro is needed.
- **The "retro" was actually a sprint planning meeting** — the line blurs. Distinguish: planning is forward-looking (what to build), retro is backward-looking (how we worked). If the meeting was mostly "what to build", run the planning-prep workflow instead.
- **No clear owner** — if the action item came up but nobody was named, ask the user to assign before creating. Don't default to the tech lead or product lead just because they're senior.
- **Action item is "talk about X with Y"** — that's a meeting, not a ticket. Don't create one. Note it for the user to add to their 1:1 agenda.
- **Lots of overlap with prior retro items** — if a topic is recurring, surface that as a meta-observation: "This is the third retro mentioning log noise. Worth elevating to a project rather than a ticket?"
