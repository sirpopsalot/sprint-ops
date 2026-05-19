# Workflow: Production release notes

## Required setup

Before doing anything in this workflow, resolve config:

1. Read `~/Documents/.claude-plugin-config-root` to find the config root.
2. Read `<config-root>/plugins/sprint-ops.user-context.md` for plugin config.
3. Read `<config-root>/identity.md` for shared identity (optional — skip if missing).

If the pointer file or plugin config doesn't exist, stop and tell the user to run `/setup-sprint` first.

---

**Goal:** produce one release-notes document **per production release** (a sprint may have several), in the destination configured at `config.release_notes.destination`, summarizing what shipped to production in that release. After publishing, draft a plain-language announcement to the chat channel at `config.context_sources.chat.channel`. Source of truth is the tracker: each sprint project contains one or more parent tasks identifying a deploy (the "release marker task"); the marker's notes or subtasks enumerate the shipped items.

**Read first:** `sprint-context-check.md` (always — gather the configured lookback window of meeting recaps + chat context first), then `scope-check.md` (decide Personal vs. Team scope, soft-ask if mismatched), then `team-context.md` so you can route questions if needed.


## Scope: Team-facing work only

This workflow produces team-facing artifacts and writes to the wiki/chat. If the user is in Personal scope, don't run it. Soft-ask:

> This workflow produces team-facing output and writes to a shared destination. You're in Personal scope by default. Want to switch to Team scope, or did you mean something narrower (e.g., "what did *I* ship this sprint")?

If the user says "switch to team", proceed. If they describe a personal version, run a scoped-down read-only variant (filtered to their tickets, no published artifacts).

## When this triggers

- "Compile release notes for today's production release"
- "Write up the release notes for the sprint that just ended"
- "Log this release to the wiki"
- "What did we ship in the last sprint?" (use this skill to produce a structured summary, not a bullet list off the top)

If the user references "today's release" or a specific date, target the marker task matching that date. If they name a sprint with multiple releases, ask which one. If no hint at all, default to the most recent release marker (highest `created_at` across active sprint projects).

## Conventions this workflow assumes

- **Sprint project naming**: matches `config.tracker.sprint_naming_pattern`. There is one tracker project per sprint.
- **Release marker task**: a parent task inside the sprint project whose name follows a recognizable convention. Default suggestion: `Production Release - YYYY-MM-DD` (one per deploy). If the team uses a different convention (e.g., a ship emoji prefix, a "Release vX.Y" pattern, etc.), this skill adapts — the user should mention the pattern they use the first time, and you can remember it for next time. The marker's notes or subtasks enumerate shipped items.
- **Destination**: `config.release_notes.destination` is one of `outline`, `confluence`, `notion`, `markdown_file`, `slack`, or `none`. Publishing branches on this value (see Step 7).
- **Collection / path**: `config.release_notes.collection_or_path` is the wiki collection ID, Notion parent, file path, or channel where the doc goes.

## Workflow

### 1. Identify the target release

If the user named a specific sprint or date, use it. Otherwise:

1. Use the tracker MCP to list projects in the configured workspace, filtered to names matching `config.tracker.sprint_naming_pattern`.
2. For each candidate sprint project, search its tasks for the release marker convention (the user's pattern, or the default "Production Release" naming).
3. Pick the marker with the most recent `created_at`, or the one matching the user's date hint.
4. Confirm in one sentence: "Compiling notes for the [sprint name] production release dated [date]. Proceed?" — then continue.

### 2. Fetch the marker task and parse the shipped-items manifest

Use the tracker MCP to fetch the marker task with its name, notes/HTML notes, completion state, custom fields, and subtasks.

The marker's notes typically contain a structured list or table of shipped items — ticket IDs (e.g., `<PREFIX>-XXXX`), task titles, and (if the tracker integrates with a code host) PR links to GitHub / GitLab / Bitbucket. There may also be an appended section listing changes that aren't represented as tickets (e.g., LLM workflow updates, config changes, direct commits).

Parse the manifest liberally. Look for: ticket IDs (regex `<PREFIX>-\d+` using `config.tracker.ticket_id_prefix`), task titles, code-host PR URLs, and tracker task URLs (which usually contain the task ID). Build the shipped-items list from:

1. Every row in the manifest table or list.
2. Every entry in appended sections (e.g., "Also shipped:", "Workflow changes:").
3. Every subtask of the marker (excluding empty subtasks).

Deduplicate by ticket ID or task name. Items in the manifest may not be marked `completed=true` in the tracker yet — a common workflow is ship → QA in prod → flip to completed. Don't filter by `completed`. A task's status field at or past the team's "ready for prod" / "in prod" / "feature complete" stages confirms it's in the release.

### 2b. Look up each shipped item's full detail

For each item identified, fetch its task from the tracker in parallel, requesting name, notes, and custom fields.

If the manifest references an item with no ticket (e.g., "ad-hoc / direct commit / no PR"), include it by name and description from the manifest. Mark "(direct commit)" in the bullet.

If PR links are present in the manifest or the ticket, surface them. If the team's tracker doesn't link to a code host, just list tickets and short descriptions.

### 3. Categorize each item

Use the `config.ticket_fields.ticket_type.field_name` value first, then fall back to keywords in the task name/notes. Put each item in exactly one section.

- **New features** → Ticket type value indicates "new" (e.g., "New Feature"). Or the type field is empty AND the task name or user story clearly introduces a new user-facing capability. Or items that are new capabilities added to existing features — prefer New features for clearly new user-visible capabilities even if the type field says "Improvement".
- **Improvements & bug fixes** → Type indicates "Improvement" or "Bug Fix". Also falls here if the task name starts with `BUG -`, `[Prod]`, `[Dev]`, or `Fix` and is clearly a fix.
- **Infrastructure / internal tooling** (optional section, only render if non-empty) → Type is `Tech` OR the task is clearly internal team enablement (wiki, CI/CD, terraform, monitoring, internal APIs not exposed to end users, dependency swaps, perf optimizations that don't change the UX). Prefix bullets with "Internal:" so readers know it's not a product change.
- **Breaking changes / migrations** → Task name or notes contain any of: "breaking change", "migration", "schema change", "deprecat", "requires re-login", "requires redeploy", "config change required". If nothing matches, write "None this release." — don't fabricate.
- **Known issues / coming next** → Don't derive from completed items. Query the next sprint's project for user-facing high-priority items. Prioritize items whose type is "New", "Improvement", or "Bug Fix" with priority Critical/High/Med (per `config.ticket_fields.priority.values`). Keep this to 3–6 items max.

If a task's type is genuinely ambiguous (both fix and new behavior), place under Improvements & bug fixes and note the new behavior in the description.

### 4. Write each entry

Each item gets a single markdown bullet:

```
- **[Short title]** — [1–2 sentence user-facing description]. (<TICKET-ID>)
```

**Short title:** Make it user-readable. Drop internal prefixes (`[Dev]`, `BUG -`, `Spike-`), drop sprint codenames, re-title for at-a-glance comprehension.

**Description rules:**
- Address the user ("You can now...", "Members will see...", "Admins can..."). No first-person ("we shipped").
- Derive content from the User Story and Acceptance Criteria in task notes. Don't copy AC verbatim — synthesize the outcome.
- 1–2 sentences. If the impact is nuanced, link the ticket for full context rather than writing a paragraph.
- No internal-only debugging context (e.g., "schema-divergence bug in the extractor code node"). Translate into plain language.
- Don't name individual team members or contributors.

**Ticket ID:** Append the `<PREFIX>-XXXX` ID from the tracker at the end in parentheses. If the ID is missing, omit it — don't invent one.

**Example transformations:**

Input:
> Name: "BUG - Error when accessing connect button on iPhone"
> Type: "Bug Fix"
> Ticket: "<PREFIX>-1067"
> Notes: "when tapping the connect button on the homepage, users get an error"

Output:
```
- **Connect button from iPhone** — Fixed an error users hit when tapping the connect button on the mobile homepage. (<PREFIX>-1067)
```

### 5. Draft the document

Assemble the markdown body (no H1 if the destination renders the title separately — Outline/Notion/Confluence do; a raw markdown file does not):

```markdown
**Sprint window:** {start_date} – {end_date}
**Released to production:** {release_date} ({"second deploy of the sprint" if not the first})

## New features

{bullets — or "None this release." if empty}

## Improvements & bug fixes

{bullets — or "None this release."}

## Infrastructure & internal tooling

{bullets with "Internal:" prefix — OMIT this section entirely if empty}

## Breaking changes / migrations

{bullets — or "None this release."}

## Known issues / coming next

{3–6 bullets previewing the next sprint's priority work}

---

*Compiled from the [sprint project]({tracker_project_url}). Questions or corrections, reach out.*
```

Use real em-dashes (—). Dates in "Month D" format ("April 6"); year only when the range crosses a year boundary.

### 6. Review before publishing

Always show the full draft markdown before publishing. Adapt the question to the destination:

> "Here's the draft for the {YYYY-MM-DD} release. Anything you want to change before I post it to [destination]?"

Wait for explicit approval. If changes are requested, apply them and re-present.

### 7. Publish to the configured destination

Once approved, branch on `config.release_notes.destination`:

- **outline / confluence / notion**: use the configured wiki MCP to create a page under `config.release_notes.collection_or_path`. Set the title per the convention below and pass the approved markdown body. If the platform supports page icons and a convention has been established, apply it. Capture the returned URL.
- **markdown_file**: write the markdown to a file under `config.release_notes.collection_or_path`, named after the title convention with a `.md` extension. If the file exists, never overwrite — append a numeric suffix and tell the user.
- **slack**: post the markdown (converted to Slack mrkdwn — single asterisks for bold, `•` bullets, `<url|label>` links) to the channel at `config.release_notes.collection_or_path`. Capture the message permalink.
- **none**: skip publishing; return the markdown body to the user so they can paste it wherever they want.

**Title format**: `YYYY-MM-DD [Sprint identifier] Production Release Notes` — date prefix using the marker's release date (ISO format), then the sprint identifier. Examples (genericized):
- `2026-05-06 Sprint 15 Production Release Notes`
- `2026-04-29 Sprint 15 Production Release Notes`

If the user has established a different title convention previously, use it.

After publishing, capture the returned URL (or file path) and include it in your reply.

### 8. Draft the chat announcement

After the doc is published, draft a plain-language summary to `config.context_sources.chat.channel`. **Always ask the user who to tag** before drafting — the people tagged vary by release.

Use the chat MCP's draft action (saves a draft, does NOT send). If the platform doesn't have drafts, ask the user whether to post directly or return the text for them to paste. If a draft already exists in the channel and the platform allows only one, tell the user — they need to delete the existing draft before this skill can recreate it.

**Chat draft template** (adapt formatting to the platform — Slack mrkdwn uses single asterisks, Teams uses standard markdown):

```
*Production Release Notes*
Here are the updates from today's release. Cc <@user1> <@user2> ...

*New things you can do*
• *<Short title>* — <plain-language description>.
...

*Smaller fixes you'll feel*
• <plain-language description>.
...

{If breaking changes:}
*Heads up*
• <description>.

{else:}
No breaking changes this release.

Full notes: <{doc_url}|{YYYY-MM-DD} Production Release Notes>
```

**Tone rules:**
- The opening must be exactly: `*Production Release Notes*\nHere are the updates from today's release. Cc <@u1> <@u2> ...` (do not vary).
- Plain language, no engineering jargon (no "JOIN optimizations", no internal system names, no PR numbers). Translate everything to user-facing impact.
- Tag the people the user named, plus any contributor whose bug report became a shipped fix (brief "thanks for surfacing this" mention next to the relevant bullet).
- Don't use emojis unless the user's prior message used them.

After creating the draft, give the user the channel link so they can review and send.

## Output to the user

After publishing both the doc and the chat draft, return a short message:

> Release notes posted: {doc_url}
>
> - {count} items shipped
> - {count_features} new features, {count_improvements} improvements/fixes, {count_infra} internal, {count_breaking} breaking
> - Next sprint preview: {count_preview} items
> - Chat draft saved to {channel} — open it to review and send

## Edge cases

- **Multiple production releases in one sprint**: produce **one doc per marker task**, dated by that marker's release date. Do not combine releases into one doc.
- **Sprint still in progress**: if asked for notes on a sprint that hasn't ended yet, confirm — releases inside the active sprint are valid (they happen mid-sprint), but a "to-date" sprint-wide summary is a different ask.
- **No release manifest in the marker's notes**: fall back to the marker's subtasks + all sprint-project tasks whose status is at or past the team's "ready for prod" stage. Surface this fallback: "The marker doesn't have a manifest yet. Using status-based fallback — review carefully."
- **A task's `completed` flag is still false**: don't exclude. Common workflow is ship → QA in prod → mark completed. Tasks in late-stage status values are in scope.
- **No type field anywhere**: keyword categorization. Names starting with `BUG`, `[Dev]`, `[Prod]`, `Fix` → Improvements & bug fixes. `Spike-`, `Test:`, `Setup`, `Tech` → Infrastructure & internal tooling.
- **Tasks referenced by name only (no ticket)**: include them by name + description from the manifest. Don't fabricate a ticket ID. Mark "(direct commit)" if that's what the manifest indicated.
- **Chat draft already exists in the channel and the platform only allows one**: ask the user to delete the existing draft, then recreate.
- **No release marker task found**: tell the user. Suggest either creating one in the tracker following the recognized convention, or pointing this skill at a different sprint project.
- **User requests a custom section order, template, or chat opening**: honor it for the current release, and remember to update this file if the user says "make this the new default."
