# Workflow: Backlog triage → tickets

## Required setup

Before doing anything in this workflow, resolve config:

1. Read `~/Documents/.claude-plugin-config-root` to find the config root.
2. Read `<config-root>/plugins/sprint-ops.user-context.md` for plugin config.
3. Read `<config-root>/identity.md` for shared identity (optional — skip if missing).

If the pointer file or plugin config doesn't exist, stop and tell the user to run `/setup-sprint` first.

**Goal:** take a raw input (bug report, feedback email, meeting transcript, feedback form submission, or a conversation) and produce well-formed tickets in the right place.

**Read first:** `sprint-context-check.md` (always — gather `config.context_sources.lookback_days` of meeting recaps + the configured chat channel context before anything else), then `scope-check.md` (decide Personal vs. Product Management scope, soft-ask if mismatched), then `ticket-quality.md` (Context / User Story / AC template, naming rules, field references).

## Scope: Product Management only

This workflow is PM-only. If the user is in Personal scope (default for non-PM users), don't run it. Soft-ask:

> This workflow is Product Management work — it produces team-facing output and writes to tickets across the sprint or backlog. You're in Personal scope by default. Want to switch to PM scope, or did you mean something narrower (e.g., "what should *I* personally bring up at retro" / "which of *my* tickets aren't ready")?

If the user says "switch to PM", proceed. If they describe a personal version, run a scoped-down read-only variant (filtered to their tickets, no team artifacts).

## Source map

Where triage inputs come from, and what that tells you about how to handle them:

| Source | What it usually contains | Handling note |
|---|---|---|
| User feedback form (in-product) | Bug reports and feature requests from users | If `config.member_feedback.enabled`, attribute to the submitting user — set `config.member_feedback.related_field` + `config.member_feedback.flag_field` to its yes value. |
| Support email inbox | Forwarded user messages, sometimes from the product lead | Same attribution. Often has more context than the form. |
| Bug report API (recent ticket) | Structured error reports with user/device info | Usually goes straight to Bugs section; technical detail is already sharp. |
| Configured chat channel | Developers flagging issues or ideas | Check who raised it — if the tech lead, likely tech debt; if an engineer, likely something they hit mid-implementation. |
| Meeting transcripts | Mixed — decisions, observations, half-formed ideas | Most selective triage here. See "Meeting transcripts" below. |
| Stakeholder conversations | New feature ideas, requests | Route non-engineering items separately (not sprint tickets). |
| Your own "invisible work" | Things you are doing that should be visible to the team | Worth surfacing as tickets so the team sees capacity allocation. |

## Step 1 — Classify before you write

For each candidate item, decide:

1. **Is this a ticket at all?** Some feedback is "noise" (complimentary, duplicate, a question answerable by docs). Say so and skip. Don't manufacture a ticket just because a sentence has a noun in it.
2. **What Ticket Type?** Map to `config.ticket_fields.ticket_type.values`. If it's clearly a feature vs. bug vs. tech-debt vs. spike, pick one. If it's ambiguous, say so and ask.
3. **Is this a duplicate?** Search before creating. Use the tracker MCP to search task titles and bodies with keywords from the input, across active sprint projects AND the backlog. If there's an existing ticket, add a comment with the new context instead of creating a new one — and surface that to the user for confirmation.
4. **Where should it go?** Default: the backlog project (`config.tracker.backlog_project_id`), in the appropriate section:
   - `Triage` for things that haven't been categorized yet
   - `Bugs` for confirmed bugs
   - A product-area section for feature work in a specific surface
   - A tech-area section for tech debt in a specific area
   - A "Future / Ideas" section for things that aren't ready to do soon

If the user explicitly said "pull this into the current sprint", put it in the sprint project directly (resolve via the sprint identification procedure). Otherwise, default to the backlog — you will promote to sprint at the product check-in.

## Step 2 — Draft the ticket

Use the template from `ticket-quality.md`. Make specific, non-generic drafts:

- Pull verbatim language from the source only when the source used a specific term (e.g., a user said "the connect button doesn't show up"). Otherwise paraphrase — don't quote an entire feedback email.
- Translate internal debugging language into user-visible behavior for the AC. A developer's "schema divergence in the extract node" becomes "search returns empty results for questions about specific entities".
- If you don't have enough to write 2–6 AC bullets, don't pad. Write 2 bullets plus a "More info needed" note — it's fine to ship an under-specified ticket if the next step is a conversation with the tech lead.

### Field defaults when drafting

- **Ticket Type** — set based on classification, using a value from `config.ticket_fields.ticket_type.values`.
- **Priority** — default the middle value from `config.ticket_fields.priority.values` unless the source says "critical" / "blocking" / "paying user affected" (then the high tier) or "nice to have" / "someday" (then low or future).
- **User Story readiness** — set `config.ticket_fields.user_story_status.field_name` to its `drafted` value until you review. Do not mark it `ready` on your own.
- **Ticket ID** — if `config.tracker.ticket_id_prefix` is set, leave blank in the draft and assign at creation by finding the highest existing `<PREFIX>-XXXX` in the workspace and incrementing. If no prefix is configured, the tracker auto-assigns.
- **User attribution fields** — fill whenever the source is a specific user, per `config.member_feedback`.
- **Assignee** — leave blank in the draft. Assignment happens at sprint kickoff with the tech lead's input.
- **Planning Poker, Shirt Size, or equivalent estimation field** — leave blank (filled at poker).

## Step 3 — Present drafts for review

Show the drafts in a table first for a scan, then the full body of each for read-through. Never create in the tracker without explicit approval.

**Draft summary format (table first):**

```
| # | Title | Type | Priority | Section | Notes |
|---|---|---|---|---|---|
| 1 | BUG- User: Connect button missing on iPhone | Bug | High | Bugs | From feedback form (user identifier) |
| 2 | Onboarding: Flow skips verification step | Bug | Critical | Product- Onboarding | Hit by real paying user — urgent |
| 3 | Tech: Reduce log noise in auth package | Tech | Med | Tech- Infrastructure | Dup of existing <TICKET-ID>, add comment instead? |
```

Then full-body drafts below the table, numbered to match.

Ask: "Which of these should I create? Any edits to titles, AC, or placement?"

## Step 4 — Create after approval

For each approved draft:

1. **Check duplicates one more time** with a tracker title-keyword search across the workspace. If a match surfaces that wasn't in the original scan, stop and surface it.
2. **Assign the next ticket ID** if `config.tracker.ticket_id_prefix` is set: search the workspace for the prefix, sort by creation, take the highest numeric suffix, add 1, and write the chosen ID into the corresponding field on creation.
3. **Create the task** via the tracker MCP:
   - Target project: the backlog or a resolved sprint project.
   - Target section: the appropriate section in that project.
   - Title: as drafted.
   - Body: the three sections **Context**, **User Story**, **Acceptance Criteria** (in that order, per `ticket-quality.md`). Wrap the new content in a "Claude draft — for developer review" header so it's clearly the skill's output.
   - Custom fields: set per the field map in `ticket-quality.md`, using `config.ticket_fields.*.field_name` and values from config — never hardcode field IDs.
4. **Confirm back.** "Created `<TICKET-ID>` in BACKLOG → Bugs." Include the tracker URL so it can be opened.

### When the source is a meeting transcript

Meeting transcripts are noisier. Rules:

- **Don't turn every discussion point into a ticket.** The team explicitly talks about many things that are just updates, decisions, or context — not actionable work.
- **Look for explicit commitments.** "PL to follow up with X", "the tech lead to implement Y" → these are candidate tickets (though they may already live in action items in the meeting notes).
- **Look for product-shaped complaints.** "This keeps breaking", "users keep asking for", "we should really" — those often become tickets.
- **Ignore meta-discussion.** Discussions about process itself (how standups should run, whether to split refinements) belong in a retro ticket or a doc, not the backlog.

### When the source is a feedback form

The feedback form is an area the user typically owns. The correct destination is the `Triage` section of the backlog, not straight to `Bugs` or a sprint. Re-classify at product check-in.

## Edge cases

- **Input is vague** — "the site is slow sometimes". Don't create a ticket. Respond with clarifying questions for the user to send back to the reporter, or create a spike ticket ("Spike: Investigate reported slowness") only if the user confirms the spike is warranted.
- **Input names a specific user who left the platform or is sensitive** — still attribute, but flag it for the user before creating publicly.
- **Input is actually a retro item** — if the feedback is about "how the team works" (process, meetings, roles), it belongs in retro prep, not a product ticket. Redirect to the retro workflow.
- **Input is out-of-scope for the engineering tracker** — content ideas, marketing requests, etc. don't go in the tracker as engineering tickets. Tell the user so they can route appropriately.
- **Input explicitly references an existing ticket ID** — open that ticket, don't create a new one. Add the new context as a comment.
