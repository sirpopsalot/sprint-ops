# Workflow: User Story / Acceptance Criteria drafting

## Required setup

Before doing anything in this workflow, resolve config:

1. Read `~/Documents/.claude-plugin-config-root` to find the config root.
2. Read `<config-root>/plugins/sprint-ops.user-context.md` for plugin config.
3. Read `<config-root>/identity.md` for shared identity (optional — skip if missing).

If the pointer file or plugin config doesn't exist, stop and tell the user to run `/setup-sprint` first.

---

**Goal:** draft the **Context**, **User Story**, and **Acceptance Criteria** for a specific ticket in your tracker — without overwriting anything that's already on the ticket. The output is appended below the existing notes as a clearly-labeled "Claude draft — for developer review" block, and the ticket's user-story-status field is set to its `drafted` value. You review, edit, keep what you like, and flip the field to `ready` yourself.

**Read first:** `sprint-context-check.md` (always — gather recent meeting recaps + chat context from `config.context_sources` before anything else), then `scope-check.md` (decide Personal vs. Product Management scope, soft-ask if mismatched), then `ticket-quality.md` (the body template + field references).


## Scope: works in either scope

This workflow runs in both Personal and Product Management scope, with different behaviors:

- **PM scope (users in `config.team.pm_scope_emails`):** standard behavior as described below — sprint-wide reads, drafts to tickets after confirmation, sets the user-story-status field to its `drafted` value, may produce team-facing artifacts.
- **Personal scope (default for everyone else):** read-only by default. Filter tracker queries to tickets the user is assigned to or following. Drafts are shown in chat only — no tracker writes, no field updates, no team-facing artifacts. If the user explicitly says "write this to <TICKET-ID>", confirm once more, then write, but never set the user-story-status field (that's a PM signal).

Announce the active scope at the top of your response in one line.

## When this triggers

- "Draft AC for <PREFIX>-1124"
- "Write a user story for the iPhone chat bug"
- "Fill in the user story on this ticket: <tracker URL>"
- "Add acceptance criteria to <PREFIX>-1198"
- "Can you draft a body for this ticket I just made?"
- "Help me write up <description of work>" — when the user describes work they want captured as a ticket and clearly wants the body drafted

If the user is pasting *new* feedback to be triaged, that's the backlog-triage workflow instead. The line: AC drafting works on a ticket the user has already identified or created. Backlog triage starts from raw inputs.

## Step 1 — Identify the target ticket

Get the ticket reference from the user's message. Accepted forms:

- A ticket ID like `<PREFIX>-1124` (using `config.tracker.ticket_id_prefix`) → use the tracker MCP to search for the matching task, scoped to `config.tracker.workspace_identifier`.
- A tracker URL → extract the task identifier from the URL.
- A task title or description with no ID — ask: "I see you want me to draft AC. Which ticket — paste the ID or URL?" Don't guess.

Once you have the task identifier, use the tracker MCP to fetch the full task. Pull enough to see: name, full notes/description, comments (request at least 20), assignee, all custom fields with their current values, and which projects/sections it belongs to. Include subtasks.

You need the full notes (existing body), comments (often where context lives), and current custom field values so you don't accidentally overwrite them.

## Step 2 — Read everything that's already there

Before drafting:

- **Notes / description.** What's already in the body? Treat this as the source of truth — never replace it.
- **Comments.** Often the richest context. Pull explicitly anything that looks like a constraint, a decision, or a quote from a user/customer.
- **Custom field values.** Note current Ticket Type, Priority, the member-feedback fields (if `config.member_feedback.enabled`), the prefix-tag/area field, and the user-story-status value. Don't change these unless explicitly asked.
- **Subtasks.** A ticket may already have subtasks describing the work. Use them to inform AC, but don't restate them as AC bullets.
- **Linked chat threads, PRs, or wiki pages mentioned in notes.** Read them if linked and accessible — they often contain the context that makes the ticket make sense.

If the existing body already has clear Context / User Story / AC sections, don't draft a parallel version. Tell the user: "This ticket already has a Context, User Story, and AC. Do you want me to refine what's there, or are you adding to a specific section?" Wait for direction.

## Step 3 — Draft Context, User Story, and AC

Use the template from `ticket-quality.md`:

```
**Context**
[1–3 sentences situating the work. Pull from existing notes, comments, linked threads, or what the user told you. Don't repeat the User Story or AC — Context is for the *why* and the current state.]

**User Story**
As a [role], I want to [action] so that [outcome].

**Acceptance Criteria**
- [Checkable statement 1]
- [Checkable statement 2]
- …
```

### Drafting principles

- **One audience per ticket.** If two roles seem to need it, pick the one whose experience is most prominent and mention the other in Context. You can split if you want.
- **AC bullets are checkable.** "X happens when Y" or "Z no longer appears" or "field is required and rejects empty values". No "the experience is improved" without a measurable threshold.
- **Translate internal language.** A developer's "schema divergence in the extractor" becomes "search returns empty results when querying by name". The audience for the body is the team, but the audience for AC is the person verifying — write so verification is unambiguous.
- **Ask if you'd be guessing.** If the user's ask doesn't include enough to write 2+ AC bullets and the ticket itself doesn't either, surface that: "I can draft Context and the User Story, but I need 2–3 more details to write checkable AC: [specific questions]." Don't pad.
- **Match a neutral, factual voice.** Brief, neutral, factual. Avoid marketing tone, exclamation points, or "Imagine if…" framing. The team reads tickets to ship work, not to be persuaded.

### What "Context" should and shouldn't contain

| Goes in Context | Doesn't go in Context |
|---|---|
| Current behavior ("users currently see X when Y") | Restating the User Story |
| Why this is in scope now (user feedback, recent prod incident) | Marketing copy ("this will delight users") |
| Constraints inherited from prior tickets (link them) | Step-by-step implementation plan (that's the developer's call) |
| User attribution if applicable ("reported via feedback form") | DoD or process steps |
| The specific surface affected (mobile only / admin panel only) | Subtask breakdown (that lives in the tracker's subtasks) |

## Step 4 — Present the draft for review

Show the *full* proposed task body — meaning: everything that's already in the notes, **unchanged**, followed by the new "Claude draft" block. This makes it visually obvious what's preserving and what's new.

```
**<TICKET-ID> — <ticket title>**

### Existing notes (preserved)

[the existing notes, copied verbatim]

### Claude draft — for developer review

**Context**
…

**User Story**
…

**Acceptance Criteria**
- …
- …
```

Then ask:

> "Want me to add this to the ticket? I'll preserve the existing notes and append this draft below them, then mark the user-story-status field as `drafted`. You'll review in the tracker and flip to `ready` when it's good."

Wait for explicit approval. If the user edits the draft inline in chat, capture the edits before writing.

## Step 5 — Write to the tracker after approval

When the user approves:

1. **Construct the new body.** Take the existing notes verbatim, append a horizontal rule, then a heading (`## Claude draft — for developer review`), then the three sections. The horizontal rule + heading are deliberate — you scan for them when reviewing in the tracker.
2. **Write to the task.** Use the tracker MCP to update the task. Set the notes/description field to the new combined body, and set the `config.ticket_fields.user_story_status.field_name` field to its `config.ticket_fields.user_story_status.values.drafted` value. Never set it to `ready` — that's the user's call after review.
3. **Don't change other custom fields** unless the user asked. Specifically, don't touch Priority, Ticket Type, Planning Poker, the area/prefix-tag field, or Assignee — those are the user's decisions.

If the existing ticket uses rich-text notes (HTML or similar), the tracker may require the entire rich-text field to be replaced, not appended to. In that case, fetch the rich text, append the new section in matching markup (`<hr><h2>Claude draft — for developer review</h2>…`), and write the full result back. If the ticket uses plain markdown notes, plain markdown is fine.

4. **Confirm back.** "Drafted on <TICKET-ID> — user-story-status is now `drafted`. The original notes are preserved at the top." Include the tracker URL.

## Edge cases

- **Existing notes already have a "Claude draft" block from a prior session** — overwrite that block (and only that block), not the whole notes. If you can't tell where the prior block ends, ask the user before writing.
- **The ticket already has user-story-status = `ready`** — that means it's already been approved. Don't draft over a ready ticket. Tell the user: "<TICKET-ID> is already marked ready. Are you sure you want to revise it? If so, I'll set it back to `drafted` after I append my new draft."
- **Draft is for a ticket that doesn't exist yet** — the user is describing work, not pointing at an existing ticket. Two paths: (a) create the ticket from scratch via the backlog-triage workflow if it's a feedback/bug-shaped input, or (b) ask the user to create the empty ticket first so the draft attaches to a real ticket ID. Don't guess which path.
- **The "ticket" the user points at is actually a comment or subtask** — confirm before drafting on a subtask (subtasks rarely need their own AC; usually they're checklist items under a parent).
- **The notes are very long and the user asked for a "rewrite"** — that's an edit request, not an AC drafting request. Show a rewrite proposal alongside the original; never silently shorten the existing notes.
