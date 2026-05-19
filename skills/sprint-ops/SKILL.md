---
name: sprint-ops
description: Program Lead workflow for any team running sprints. Use whenever the user asks to "triage the backlog", "turn this feedback into tickets", "draft a user story", "draft acceptance criteria", "fill in the AC", "write a user story for X", "prep for sprint planning", "check if the sprint is ready", "see which tickets aren't ready for poker", "do a mid-sprint progress check", "write a sprint status update", "scan the sprint for retro items", "prep retro topics", "summarize sprint issues for retro", "compile release notes", "write up sprint N", "what did we ship this sprint", or any variation on sprint planning prep, backlog triage, ticket drafting, mid-sprint progress check, retro seeding, or production release notes. Also trigger on "program lead work", "PL work", "sprint prep", "refine backlog", "sprint health check", or when the user pastes feedback, bug reports, meeting notes, or transcripts that need to become tickets.
---

# sprint-ops

This skill applies a Program Lead workflow to sprint-management work: turning raw inputs (feedback, bug reports, meeting transcripts, conversations) into well-formed tickets, preparing the sprint for kickoff and poker estimation, running a mid-sprint progress check, seeding the retrospective with concrete discussion items, and compiling production release notes.

It's tracker-agnostic — your tracker, team, and field schema come from config written by `/setup-sprint`. The workflow patterns (six routines below) are the value the skill adds on top.

## Required setup

Before any workflow runs, this skill needs config to exist. Resolve it in this order:

1. Read `~/Documents/.claude-plugin-config-root` to find the config root.
2. Read `<config-root>/plugins/sprint-ops.user-context.md` for plugin config.
3. Read `<config-root>/identity.md` for shared identity (optional — skip if missing).

If the pointer file or plugin config doesn't exist, stop and tell the user:

> I need configuration before I can do sprint work. Run `/setup-sprint` — it'll walk you through ~10 questions (tracker, team, ceremony cadence) and write the config. Then come back.

Don't try to substitute guesses for the config. The whole skill is built on it.

## How to route

The skill covers six workflows. Figure out which one the user is asking for based on these cues, then read the matching reference file before acting:

- **Backlog triage → tickets** — user pasted feedback, a bug report, a meeting transcript, an email, or notes and wants them turned into tickets; or asked to "triage the backlog", "clean up triage", "look at the feedback form". → read `references/workflow-backlog-triage.md`
- **User story / AC drafting** — user pointed at an existing ticket (by ID or URL) and asked to "draft AC", "write a user story", "fill in the user story", "add acceptance criteria", or asked to draft a ticket body for something specific they have in mind. → read `references/workflow-user-story-ac-drafting.md`
- **Sprint planning prep** — user asked "is the sprint ready?", "which tickets aren't ready for poker?", "prep for planning", "do a readiness check on the sprint", "are there any tickets missing AC?". → read `references/workflow-sprint-planning-prep.md`
- **Progress check (mid-sprint)** — user asked for a "mid-sprint status", "sprint health check", "are we on track?", "draft a status update for the sprint". → read `references/workflow-progress-check.md`
- **Retro seeding** — user asked to "scan the sprint for retro items", "prep retro topics", "summarize sprint issues for retro", "what should we talk about at retro", or pasted a retro transcript to turn into tickets. → read `references/workflow-retro-prep.md`
- **Release notes** — user asked to "compile release notes", "write up the sprint", "what shipped this sprint", "post release notes", or any variation on production release notes. → read `references/workflow-release-notes.md`

If the request straddles two workflows (e.g., "do a progress check and flag retro items"), do them as separate passes and tell the user which you're doing first. Don't interleave outputs — it makes review harder.

Every workflow relies on the ticket-quality rubric in `references/ticket-quality.md`. Read it once per session before writing or evaluating any ticket.

## Shared behaviors across all workflows

### Always run the sprint context check first

Before any of the six workflows, run the procedure in `references/sprint-context-check.md`. It pulls the configured lookback window (default 14 days) of meeting recaps + the configured chat channel, and uses that context to ground the draft you produce. Teams often make decisions in standup or in chat that don't make it back into the tracker — without this step, drafts can contradict the most recent call.

Before fetching, verify the configured context sources are connected (the connector your team uses for meetings — Microsoft 365 / Google Workspace — and your chat platform). If either is missing, stop and prompt the user with the setup steps in `sprint-context-check.md` before proceeding. Don't silently skip — silently skipping is how you produce a draft that misses a Tuesday-standup call to defer something.

If the user explicitly says "skip context" or "don't pull meetings", honor it and proceed. Warn once at the top of the output.

If the user's config sets context sources to `none`, skip the check silently — they've opted out at setup time.

### Confirm the working scope

After the context check and before any workflow-specific logic, decide whether the user is operating in **Product Management scope** or **Personal scope**, and announce it in one line. Full procedure in `references/scope-check.md`.

Default scope by user (matched against `# userEmail` in system context against the `PM-scope members` list in config):

- **User's email is in the PM-scope list** → Product Management scope
- **User's email is not in the PM-scope list** → Personal scope

Personal scope is read-only by default, filters tracker queries to the user's own assignments, and suppresses team-facing artifacts (status posts, wiki writes, retro topic lists). Some workflows are PM-only — when a non-PM-default user asks for one, soft-ask before proceeding ("This is PM work — want to switch, or did you mean a personal version?"). Don't gatekeep beyond the soft-ask.

Override the default when phrasing makes scope explicit: "what's on my plate" → Personal even for a PM-default user; "the sprint" / "the team" → PM even for non-PM users (with the soft-ask).

### Read-only escape hatch

If the user includes any of "no draft", "don't draft", "just tell me", "read-only", "don't write anything", "no writes", or "info only" in their request, suppress draft generation entirely. Pull and analyze data normally, produce findings in chat, but do not offer to write anything to the tracker and do not produce paste-ready ticket drafts. Works in either scope. Useful when the user wants to think about a ticket without triggering the write machinery.

### Never write to the tracker without confirming first

Every workflow produces drafts in chat for you to review before anything is created, updated, or moved in the tracker. This is deliberate — drafts going straight to writes are how duplicate tickets get created and how irrelevant notifications get sent. Silence is not approval. Present the draft, wait for "yes" or edits, then act.

When presenting drafts, show the entire ticket (title, type, priority, user story / AC, assignee suggestion) in one block. Use a markdown table when presenting multiple tickets at once so you can scan quickly.

### Always preserve existing ticket content

When editing or augmenting an existing ticket — for example, drafting AC on a ticket that already has notes — never overwrite what's there. Read the existing notes, keep them verbatim, and append a clearly-labeled "Claude draft — for review" section with the new content below the original. The user reviews and chooses what to keep. The existing content is the source of truth until the user edits it themselves.

### Default the ready-for-sprint field to "drafted", never to "ready"

When the skill adds a draft user story or acceptance criteria to a ticket — whether on a brand-new ticket or by augmenting an existing one — set the configured ready-for-sprint field (config's `Ready-for-sprint gate`) to its **"Drafted" value**, never to its **"Ready" value**. The "ready" value is reserved for the human's review pass: they read the draft, edit, and flip the field themselves.

Setting "ready" from the skill silently signals "this ticket is ready for poker" when it isn't — and the team will start estimating against content nobody has reviewed yet.

If the user's tracker doesn't have a distinct "drafted" value in their field, the skill leaves the field unset and adds a comment instead: "AI draft awaiting review — flip the gate field when you've signed off."

### Identify the target sprint

If the user names a sprint number or identifier, use it. Otherwise:

1. List active projects in the tracker workspace (`config.tracker.workspace_identifier`) using whichever tracker MCP the user has installed.
2. Filter to projects whose names match `config.tracker.sprint_naming_pattern`.
3. Parse start/end dates from each project name if present.
4. Pick the one whose date range contains today, or if none does, the one whose start is nearest in the future (for planning prep) or whose end is nearest in the past (for progress check / retro seeding).
5. Confirm out loud: "I'm working against **<sprint name>**. Proceed?" — then continue.

If your tracker doesn't expose sprints as projects (e.g., Jira's sprint object is different): adapt the lookup to whatever your tracker MCP provides for active sprints. The skill stays at the level of "find the current sprint" — Claude picks the tool.

### Don't pad, don't editorialize

Users read a lot of AI-generated content every day and notice when output is stretching. Keep drafts short, specific, and structured. If a section is empty, write "None this sprint" — don't invent filler. If you're uncertain about a call (e.g., is this a bug or a feature?), say so and ask, rather than guessing and hoping.

### Attribute member feedback when you see it

If `config.member_feedback.enabled` is `yes` and the source you're triaging mentions a specific member, customer, or user — by name, email, or quoted message — populate the configured `Related-member field` on the ticket and set the configured `Flag field` to its "Yes" value. This is how feedback gets credited back to the people who reported it.

## What this skill does NOT do

- **Standup facilitation**: not in scope. If you ask for a standup recap or prep, say so and offer to add it.
- **Writing PRDs or product strategy docs**: this is backlog / ceremony work, not product discovery.
- **Moving tickets or changing status without confirmation**: always present drafts in chat first and wait for approval. Duplicate tickets and notification noise erode team trust fast.
- **Setting the ready-for-sprint gate field to its "Ready" value**: only the user does that, after reviewing.
- **Overwriting existing ticket content**: drafts are appended, originals preserved.
- **Configuring itself**: setup is `/setup-sprint`. The skill reads config but does not change it.

## Reference files

- `references/sprint-context-check.md` — meeting + chat context grounding (run before every workflow)
- `references/scope-check.md` — Personal vs. Product Management scope rules, identity-based defaults, soft-ask phrasing, read-only escape hatch
- `references/team-context.md` — how to load team roster, ceremony cadence, and tool stack from config
- `references/ticket-quality.md` — what a "ready" ticket looks like, the Context / User Story / AC template, naming rules, plus field-write conventions
- `references/workflow-backlog-triage.md` — turn feedback / bug reports / transcripts into tickets
- `references/workflow-user-story-ac-drafting.md` — draft Context / User Story / AC on a specific ticket while preserving existing content
- `references/workflow-sprint-planning-prep.md` — check which tickets are ready for poker, flag the rest
- `references/workflow-progress-check.md` — mid-sprint status assessment + status-ready update
- `references/workflow-retro-prep.md` — scan sprint for retro topics + turn retro action items into tickets
- `references/workflow-release-notes.md` — compile production release notes for a sprint and publish to the configured destination
