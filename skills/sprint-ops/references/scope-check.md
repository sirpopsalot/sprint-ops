# Scope check (run after the context check, before drafting)

## Required setup

Before doing anything in this workflow, resolve config:

1. Read `~/Documents/.claude-plugin-config-root` to find the config root.
2. Read `<config-root>/plugins/sprint-ops.user-context.md` for plugin config.
3. Read `<config-root>/identity.md` for shared identity (optional — skip if missing).

If the pointer file or plugin config doesn't exist, stop and tell the user to run `/setup-sprint` first.

**Goal:** before doing any work, confirm whether the user is operating in **Product Management scope** (sprint-wide, team-facing, can write to the tracker, produces team artifacts) or **Personal scope** (this user's own tickets only, read-only by default, no team-facing artifacts). Skipping this is the most common way the skill misfires — a developer asking "what's on my plate?" should not get a sprint readiness report.

This check runs after `sprint-context-check.md` and before any workflow-specific logic.

## The two scopes

### Product Management scope

The user is acting in a product/program lead capacity. The skill behaves as designed:

- Reads sprint-wide and backlog-wide data.
- Drafts Context / User Story / AC and writes to the tracker after confirmation.
- Sets the `<user_story_status.field_name>` field to its `<user_story_status.values.drafted>` value.
- Produces team-facing artifacts: chat updates, retro topic lists, release notes.
- All six workflows are available.

### Personal scope

The user is organizing their own work. The skill is conservative:

- **Read-only by default.** No tracker writes, no field updates, no comments, no ticket creation. Drafts are produced in chat and shown to the user, but nothing is pushed to the tracker unless the user explicitly says "write this to the ticket" — and even then, only on tickets they own (assignee or follower).
- **Scoped to the user's tickets.** Tracker queries filter to tasks assigned to the user (or followed by the user when relevant). Sprint-wide and backlog-wide pulls are not allowed.
- **No team-facing artifacts.** No chat-ready update generation, no wiki writes, no retro topic lists, no release notes.
- **Only some workflows are available.** See "Workflow availability by scope" below.

## Default scope by user

Decide the default by matching the user's email (provided in system context as `# userEmail`) against `config.team.pm_scope_emails`:

- If the user's email appears in `config.team.pm_scope_emails` → default to **Product Management** scope.
- Otherwise → default to **Personal** scope.
- If `userEmail` is not present in context → default to **Personal** and confirm scope explicitly before proceeding.

Matching should tolerate aliases when the configured value uses a wildcard (e.g., `noor*@example.com`).

## How to decide and announce scope

Three signals, in order of strength:

1. **Identity match.** Use the rule above as the default.
2. **Workflow type.** Some workflows are PM-only (see below). If the user requested a PM-only workflow but the default is Personal, soft-ask before proceeding.
3. **Phrasing.** Override the default when the request itself signals scope:
   - **→ Personal:** "what's on my plate", "my tickets", "for me", "my workflow", "tickets I'm assigned to", "I want to know if I'm on track"
   - **→ PM:** "the sprint", "the team", "for kickoff", "is the sprint ready", "compile release notes", "scan the sprint for retro items", "for the whole team"

Once the scope is decided, announce it in one line and proceed. Don't make this a multi-question interview.

### Announcement formats

For a PM-scope user with a clearly PM request:

> Working in **Product Management scope** — sprint-wide, can write to tickets after your confirmation.

For a non-PM-default user asking something personal:

> Working in **Personal scope** — read-only, scoped to your assignments. Tell me "switch to PM" if you mean team-wide.

For a non-PM-default user asking something that sounds like PM work (the soft-ask):

> I'm reading this as Product Management work — sprint-wide, with the ability to draft into tickets. You're not in your team's configured PM-scope list. Want to proceed in **PM scope**, or stay in **Personal scope** (read-only, your tickets only)?

Wait for explicit answer before continuing. If the user picks PM, proceed but don't change defaults — next session reverts to Personal.

For a PM-scope user asking something clearly personal ("what's on my plate this sprint?"):

> Working in **Personal scope** for this — your tickets only, read-only. Say "PM scope" if you actually want a sprint-wide view.

## Workflow availability by scope

| Workflow | Personal scope | Product Management scope |
|---|---|---|
| Backlog triage → tickets | Soft-ask: "Triaging the backlog into tickets is PM work. Want to switch to PM scope?" | Standard behavior |
| User story / AC drafting | Allowed only on tickets the user is assigned to or following. Drafts shown in chat; no writes to tracker, no field updates. | Standard behavior |
| Sprint planning prep | Soft-ask: "Sprint readiness is PM work. Want to switch to PM scope, or did you want to know which of *your* tickets aren't ready?" — if the latter, run a personal version that lists only the user's tickets and their AC-readiness state. | Standard behavior |
| Mid-sprint progress check | Personal version: "Where am I against my assigned tickets?" — shows the user's tickets, their statuses, anything stuck. No chat draft. | Standard behavior with chat-ready 3-line draft |
| Retro prep | Soft-ask: "Retro prep is team-level. Want PM scope, or did you mean 'what should *I* bring up at retro about my own work'?" — if the latter, scope to user's own tickets and stuck items. | Standard behavior |
| Release notes | Soft-ask: "Release notes are a PM artifact published to the team's release-notes destination. Want PM scope?" | Standard behavior |

## Personal scope behavioral rules

When operating in Personal scope:

1. **Tracker queries filter to the user.** Use the tracker MCP to filter tasks by assignee (or include a follower filter when relevant). Don't pull sprint-wide.
2. **No writes.** Task updates, task creation, comment creation, and field updates are all suppressed by default. If the user explicitly says "write this draft to `<TICKET-ID>`", confirm once more ("you want me to append this draft to `<TICKET-ID>`'s notes?"), then write — but never set the `<user_story_status.field_name>` field, even to its `<user_story_status.values.drafted>` value. That's a PM signal.
3. **No team-facing artifacts.** Skip chat-update drafting, wiki/release-notes writes, and retro topic lists.
4. **Outputs are framed personally.** "Your tickets / your work / what you have in flight" rather than "the sprint / the team / what's on track".
5. **Surface PM-mode opportunities gently.** If during personal-scope work the skill notices something that should be raised to a PM ("`<TICKET-ID>` is blocked and assigned to you, but the blocker is unowned"), flag it but don't act on it.

## Read-only escape hatch (works in either scope)

If the user includes any of these phrases anywhere in the request, suppress draft generation entirely and produce information-only output:

- "no draft", "don't draft", "just tell me", "read-only", "don't write anything", "no writes", "info only"

In this mode:

- Pull and analyze data normally (with the active scope's filters).
- Produce findings/analysis in chat.
- Do **not** offer to write anything to the tracker.
- Do **not** produce drafts that look ready to paste into a ticket.

This is for users who want to think out loud about a ticket or a sprint without the skill setting up a write action.

## Edge cases

- **User is in the PM-scope list but explicitly says "personal scope"** — honor it. Run in Personal scope for this turn. Don't argue.
- **User is non-PM but explicitly says "PM scope"** — honor it. Don't gatekeep beyond the soft-ask. The soft-ask is information, not permission.
- **No `userEmail` in context** — default to Personal scope and confirm out loud: "I don't see your email in context — defaulting to Personal scope. Tell me your role if you need PM behaviors."
- **Mid-conversation scope flip** — the user starts personal and then says "actually, do this for the whole team" — switch scopes, announce the switch, and re-pull data with the new filters. Don't carry over personal-filtered results.
- **Workflow runs in Personal scope but needs a single team-level fact** (e.g., "is my ticket the only one stuck?") — fetch the minimum needed, frame the answer in the user's context, don't expand into a sprint-wide report.
