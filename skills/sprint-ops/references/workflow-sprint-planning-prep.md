# Workflow: Sprint planning prep

## Required setup

Before doing anything in this workflow, resolve config:

1. Read `~/Documents/.claude-plugin-config-root` to find the config root.
2. Read `<config-root>/plugins/sprint-ops.user-context.md` for plugin config.
3. Read `<config-root>/identity.md` for shared identity (optional — skip if missing).

If the pointer file or plugin config doesn't exist, stop and tell the user to run `/setup-sprint` first.

---

**Goal:** before sprint kickoff, give the user a readiness report so every ticket in the sprint is poker-ready. Developers should be able to estimate without asking what a ticket means.

**Read first:** `sprint-context-check.md` (always — gather recent meeting recaps and chat context from `config.context_sources` before anything else), then `scope-check.md` (decide Personal vs. Product Management scope, soft-ask if mismatched), then `ticket-quality.md` (what "ready" means + field references).

## Scope: Product Management only

This workflow is PM-only. If the user is in Personal scope (default for non-PM users), don't run it. Soft-ask:

> This workflow is Product Management work — it produces team-facing output and writes to tickets across the sprint or backlog. You're in Personal scope by default. Want to switch to PM scope, or did you mean something narrower (e.g., "what should *I* personally bring up at retro" / "which of *my* tickets aren't ready")?

If the user says "switch to PM", proceed. If they describe a personal version, run a scoped-down read-only variant (filtered to their tickets, no team artifacts).

## Why this matters

Tickets must be fully detailed before developers see them at sprint kickoff, so the kickoff is an estimation meeting, not a refinement discussion. Refinement churn moves to an async product check-in between the tech lead, product lead, and you. If tickets arrive at kickoff under-specified, the meeting overruns and developers context-switch.

Duplicate tickets are another common trap: the same work shows up in both the active sprint and the backlog, and gets treated as two separate items. This workflow catches that before kickoff.

## Step 1 — Identify the target sprint

Use the tracker MCP to list active projects in `config.tracker.workspace_identifier` matching `config.tracker.sprint_naming_pattern`. Default to the sprint whose start date is nearest in the future (the one about to kick off). Confirm with the user in one sentence before proceeding.

## Step 2 — Pull the sprint's tickets

Use the tracker MCP to search tasks in the resolved sprint project. Filter to incomplete tasks. Include fields needed for the readiness checks: name, ticket ID, assignee, notes/description, and the custom fields named in `config.ticket_fields` (user story status, planning poker, priority, ticket type), plus section/swimlane membership.

Expect ~15–25 tickets for a typical two-week sprint. If the count is materially off, flag it — might be a stale sprint or incomplete pull.

## Step 3 — Run the readiness checks

For each ticket, check these conditions:

### Hard blockers (ticket cannot go to poker as-is)

1. **`config.ticket_fields.user_story_status.field_name` is not at its `ready` value** — either `drafted`, `not_ready`, or unset.
2. **No acceptance criteria in notes** — open the task notes; if there's no AC section, it's under-specified even if the status field says ready.
3. **No `config.ticket_fields.ticket_type.field_name` set.**
4. **No `config.ticket_fields.priority.field_name` set.** (Priority can be refined in the product check-in, but "no priority at all" means the ticket hasn't been triaged.)
5. **No ticket ID** — if `config.tracker.ticket_id_prefix` is set, every ticket should carry a `<PREFIX>-XXXX` identifier.
6. **Mixed-deliverable body** — AC spans two teams' work without subtasks assigned to different people.
7. **Looks like an 8+ pointer** — the AC describes a multi-phase effort (migration + frontend + email + tests, etc.). Candidate for break-up.
8. **Duplicate** — the same ticket (by title or ID) appears in the backlog AND the sprint. Or appears in two sprints.

### Soft flags (worth surfacing but not blocking)

- **No assignee yet** — OK to go into poker unassigned, but the tech lead should pre-assign before kickoff so the team knows who's estimating for themselves.
- **Outdated AC** — if the task notes reference a behavior that has since shipped or changed, the AC may be stale.
- **Vague "improvements" without a threshold** — "make X faster" without a target.
- **Priority mismatch** — Critical-priority item in sprint but nobody in standup or chat has mentioned it recently; possibly over-prioritized.

## Step 4 — Produce the readiness report

Format the output as a two-section report you can use directly in the product check-in.

```
# Sprint {N} — Planning readiness (as of {today})

**Scope:** {total} tickets in Sprint {N}. Target is ~20.

## Not ready for poker ({count})

| ID | Title | Blocker | Action |
|---|---|---|---|
| <PREFIX>-1124 | BUG: connect button missing on mobile | No AC in notes | Add AC — 2–4 bullets |
| <PREFIX>-1101 | Onboarding — full flow rework | Likely 8+ pointer | Split along flow type |
| <PREFIX>-1089 | Tech: Reduce log noise | Duplicate of <PREFIX>-1044 in backlog | Remove one; add comment on survivor |
| <PREFIX>-1132 | Search: faster results | User story status = Drafted | Mark ready after final pass |

## Soft flags ({count})

- **<PREFIX>-1118 — Admin: feedback form triage** — no assignee. Suggest the tech lead pre-assign before kickoff.
- **<PREFIX>-1127 — Cache user data** — AC mentions "faster" without a target. Consider adding a concrete threshold (e.g., "p50 < 1.5s on staging").

## Ready for poker ({count})

{compact list of <PREFIX>-XXXX: title — not blocking, no action needed}

## Capacity check

- Tickets: {count} (vs ~20 target)
- Carry-over from prior sprint: {count}
- Bug mix: {bugs} / {features} / {tech}
- Known unavailability: {e.g., team member on PTO Week 2}
```

## Step 5 — Offer to fix the fixable

After presenting the report, ask the user if they want to:

1. **Add drafted AC** — for tickets flagged "No AC" or with user story status at `drafted`. If yes, draft AC one ticket at a time and present for the user to approve before writing back to the tracker. Use the ticket-quality template.
2. **Flip the user story status to `ready`** — only after the AC actually exists and the user has read it. Use the tracker MCP to update the task: set the `config.ticket_fields.user_story_status.field_name` field to its `ready` value. Never flip this field based on your own judgment.
3. **Propose a split** — for suspected 8+ pointers. Draft the 2–3 child tickets, present, wait for approval, then create them via the tracker MCP. Split along PR-review seams where possible.
4. **Resolve duplicates** — present the pair, propose which to keep, wait for approval, then delete or archive the other.

Don't batch-update tickets. Each write gets its own confirmation. This is deliberate — silent ticket churn erodes trust fast.

## What "ready" looks like at the end

At sprint kickoff, every ticket in the sprint project should have:

- User story status field at its `ready` value
- Ticket type set
- Priority set
- AC written out in notes (2–6 bullets)
- Ticket ID populated (if the tracker uses a prefix)
- Assignee set (or an explicit "team-wide" note if it's truly unassigned for poker)
- Planning poker / size estimate fields **blank** — those get filled at the poker session itself

Developers will estimate from that state and won't ask "what does this mean?" which is the whole point.

## Edge cases

- **Team capacity is uncertain** — e.g., someone transitioning out, availability unclear. Don't try to compute point-capacity; just flag the uncertainty in the report so the user can factor it in.
- **A ticket is at a "Blocked" status** — surface it but don't block on it. A blocked ticket can still be estimated; the user and tech lead decide whether it stays in the sprint.
- **Tickets at a "Needs Design" status** — these probably shouldn't poker yet. Flag separately under "Not ready — design first".
- **The sprint has already kicked off** — say so and offer to do a mid-sprint readiness pass instead (progress check workflow).
