# Ticket quality rubric

## Required setup

Before doing anything in this workflow, resolve config:

1. Read `~/Documents/.claude-plugin-config-root` to find the config root.
2. Read `<config-root>/plugins/sprint-ops.user-context.md` for plugin config.
3. Read `<config-root>/identity.md` for shared identity (optional — skip if missing).

If the pointer file or plugin config doesn't exist, stop and tell the user to run `/setup-sprint` first.

---

A ticket is "ready" when a developer can read it at sprint kickoff, poker-estimate it without asking questions, and build the right thing without pinging the user. This page is the definition of that bar.

## The ready-for-poker checklist

Every ticket going into a sprint must have all of:

1. **A clear title.** Bug name first, then audience. `BUG- Member: Connect button missing on iPhone`, not `iPhone bug`. Drop the sprint codename from the title.
2. **Body with three sections in this order: Context, User Story, Acceptance Criteria.** That's the entire template — no Definition of Done section, no Notes section by default. (See "Body template" below.)
3. **Ticket Type set.** Use the values from `config.ticket_fields.ticket_type.values`. Multi-enum trackers allow more than one value, but a single ticket almost always wants one — if it genuinely spans two (rare), pick the most visible one to users.
4. **Priority set.** Use the values from `config.ticket_fields.priority.values`. Severity values (e.g., Critical / High / Med / Low) answer "how urgent?". Audience-scoped values (e.g., tier names) answer "who is this for?" and can coexist with a severity if the tracker allows.
5. **Ticket ID.** Every ticket gets a `<PREFIX>-XXXX` using `config.tracker.ticket_id_prefix`. If you're creating a new ticket and the tracker doesn't auto-assign, take the next unused number (search the backlog for the highest existing prefix-number and increment).
6. **Assignee.** A real developer, not "TBD". If you don't know who, ask the tech lead at the next check-in — don't punt on this field.
7. **Member feedback fields populated** if `config.member_feedback.enabled` is true and the source was a specific user's feedback. Set `config.member_feedback.related_field` to the user's record and `config.member_feedback.flag_field` to `config.member_feedback.flag_yes_value`. This is how downstream comms credit users.
8. **The configured ready-for-sprint gate field set to its `ready` value.** The field is named in `config.ticket_fields.user_story_status.field_name`; the values are in `config.ticket_fields.user_story_status.values.ready` / `.drafted` / `.not_ready`. This is the gate. The skill only ever sets this field to its `drafted` value — you flip it to `ready` yourself after you review.

### One deliverable per ticket

If a ticket's acceptance criteria mix two teams' work (e.g., "backend workflow updates" + "frontend form change"), split it. Use subtasks, each with its own assignee. Mixed-deliverable tickets are a common pain point — accountability blurs when one ticket has one assignee but three collaborators.

**Rule of thumb:** if the AC would naturally be reviewed by two different people in two different PRs, those are two subtasks at minimum, possibly two separate tickets. Split mixed-deliverable tickets along PR-review seams.

### Epics and oversized tickets

The planning-poker scale has a top end (the highest value in `config.ticket_fields.planning_poker.values`). If the honest estimate exceeds that top value, the rule is break it up. Ways to break up:

- By user flow (e.g., "Onboarding — new user path" / "Onboarding — returning user path" / "Onboarding — referral path").
- By layer (backend API + frontend form + migration + notification email).
- By phase (spike to learn the shape, then implementation ticket, then polish ticket).

If you can't break it up and it's genuinely one atomic piece of work, flag it for the next check-in — the tech lead may want to shape it differently or split along a different seam.

## Body template

The ticket body has three sections in this order: **Context**, **User Story**, **Acceptance Criteria**. That's it. No Definition of Done section, no Notes section, no Subtasks-as-checklist-in-the-body. Subtasks live in the tracker's subtask field; DoD is implied by the team's standard process and doesn't need to repeat in every ticket.

Aim for ~5–12 lines total. Tickets that are too terse confuse developers, and tickets that are too essayistic bury the actual ask.

```
**Context**
[1–3 sentences. What's the current behavior? Why are we doing this now? Link any related tickets, chat threads, or user feedback. Keep it short — context is for situating the ask, not retelling the discussion.]

**User Story**
As a [user role], I want to [action] so that [outcome].

**Acceptance Criteria**
- [Checkable statement 1]
- [Checkable statement 2]
- …
```

**Good acceptance criteria:**
- When a user clicks the Connect button on the mobile homepage, they are routed to the destination workspace without an error page.
- If the integration is disconnected, the button renders disabled with a tooltip explaining why.
- Error state is logged with a tag suitable for search in the logging system.

**Bad acceptance criteria:**
- Connect works. _(not checkable)_
- Fix the iPhone bug. _(not specific — which behavior?)_
- Also, while we're in there, clean up the logs. _(scope creep — separate ticket)_

User stories that span two audiences (e.g., "As a user or admin…") are usually two tickets. If it really is one piece of work that affects both, use the audience whose experience is most prominent and call out the other in Context.

## Naming patterns

- Bugs: `BUG- [Audience]: [Symptom]` → `BUG- Member: Connect button missing on iPhone`
- Improvements: `[Area]: [What changes]` → `SEARCH: Faster results for name-based queries`
- New features: `[Audience] can [action]` → `Admins can edit references directly`
- Tech debt: `Tech: [Change]` → `Tech: Cache user and industry data on call`
- Spikes: `Spike: [Question to answer]` → `Spike: How should we automate feedback triage?`

Drop `[Dev]`, `[Prod]`, `Sprint-N`, emoji prefixes, or owner names from titles. The sprint project membership already tells you the sprint; custom fields tell you the type; assignee tells you the owner.

## Writing field values

The skill reads field-name strings and value strings from `config.ticket_fields.*` at runtime. When updating a ticket, use the configured field name (e.g., `config.ticket_fields.priority.field_name`) and the configured value string (e.g., `config.ticket_fields.priority.values.high`) — never hardcode either in workflow logic.

**Trackers that require IDs instead of names.** Some trackers (notably Asana) require enum option IDs rather than human-readable names when writing custom field values. In those cases:

- During `/setup-sprint`, capture the option IDs in the config alongside each value string. The config can store both the display name and the write-time ID.
- If IDs weren't captured at setup, the skill does a one-time lookup on first write — fetch the field's option list from the tracker, match each configured value string to its ID, and either write the value or prompt the user to record the IDs in config to avoid repeating the lookup.

Other trackers (Jira, Linear, GitHub Projects) accept the value string directly — no ID translation needed. The skill should detect this from `config.tracker.tool` and skip the ID-resolution step.

**Two principles that always apply, regardless of tracker:**

1. **The skill ALWAYS sets the ready-for-sprint gate field to its `drafted` value when *it* drafts content.** Never the `ready` value. The user owns the `ready` flip after review — that's the human gate. If the skill ever writes `ready`, the gate is meaningless.
2. **Don't set planning poker programmatically.** That field is filled at the poker session by the team estimating together. The skill leaves `config.ticket_fields.planning_poker.field_name` untouched on draft.

Text-only fields (ticket ID, related-user references, free-form notes) accept string values directly across all trackers — no ID resolution needed.
