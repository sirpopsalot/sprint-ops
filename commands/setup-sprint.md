---
description: Configure sprint-ops for your team, tracker, and ceremony cadence via a short interview. Writes results to `<config-root>/plugins/sprint-ops.user-context.md` (where `<config-root>` is the folder you choose during first-time setup, stored at `~/Documents/.claude-plugin-config-root`). Re-run anytime to update.
---

# /setup-sprint

Short interview that captures the context the sprint-ops skill needs to be useful for *your* team — your tracker, your ceremony cadence, who's on the team, where context lives.

---

## Step 0 — Resolve plugin config root

Per-plugin config in this marketplace lives under a user-chosen folder, recorded at `~/Documents/.claude-plugin-config-root` (a single-line text file in the user's home directory). Resolve it before doing anything else.

### A — Try the pointer

Ensure access to `~/Documents`. In Cowork, call `request_cowork_directory(~/Documents)` once if not already granted. In Claude Code (or any environment with direct filesystem access), no mount is needed. Then read `~/Documents/.claude-plugin-config-root`.

- **Pointer exists**: read line 1 → that's the config root path. Ensure access to `<config-root>`. If running in Cowork and the folder isn't already mounted in this session, call `request_cowork_directory(<config-root>)`. If running in Claude Code or another environment with direct filesystem access, no mount call is needed. Skip to section C.
- **Pointer missing**: continue to section B.

### B — First-time bootstrap

This is the user's first plugin setup of any kind. Prompt:

> "First-time plugin setup. Where should I store your plugin config — identity, voice, and per-plugin settings? Pick a folder you control. Examples: `~/Documents/Claude/` (a common pick — and where cortex memory already writes if you have it installed) or `~/Documents/PluginConfig/` or any other path you prefer. The folder will hold one `identity.md`, one `voice.md`, and a `plugins/` subdirectory with one file per plugin you set up."

Once the user provides the path:

1. Ensure access to `<path>`. If running in Cowork and the folder isn't already mounted in this session, call `request_cowork_directory(<path>)`. If running in Claude Code or another environment with direct filesystem access, no mount call is needed — proceed to read or write the file.
2. Create `<path>/plugins/` if it doesn't exist.
3. Write the absolute path to `~/Documents/.claude-plugin-config-root`.
4. Confirm: "Saved. All marketplace plugin configs will live under `<path>` from now on. You can change this later by editing `~/Documents/.claude-plugin-config-root` directly."

### C — Read shared identity

Read `<config-root>/identity.md` (the canonical identity file populated by cortex's `/setup-identity`).

- **Exists and populated** → pre-fill the Identity section of this interview from those values (name, email, company, role). Skip those questions in Section 1; just confirm what you read.
- **Missing** → offer: "Want to capture name/company/role once via `/setup-identity` (in cortex) so all marketplace plugins can read it? Or capture identity inline here only?" Route to `/setup-identity` if user prefers, then resume. Otherwise proceed inline (the inline answers go to this plugin's config file only — not to the shared `identity.md`).

For the rest of this document, **`<config-root>`** refers to the resolved path. This plugin's config file lives at **`<config-root>/plugins/sprint-ops.user-context.md`**.

---

## Step 1 — Check for existing config

Read `<config-root>/plugins/sprint-ops.user-context.md` if it exists.

- If it exists and is populated → ask: "You've already configured sprint-ops. Want to update specific sections, or start over?"
  - "Update [section]" → jump to that section, ask only those questions, write back. Sections are: Identity, Tracker, Team, Context Sources, Ticket Fields, Member Feedback, Release Notes.
  - "Start over" → continue full interview.
- If it doesn't exist → start fresh.

---

## Step 2 — The interview

Ask one section at a time. After each section, summarize what you heard and ask the user to confirm or correct before moving on. Don't bombard with all questions at once.

### Section 1 — Identity

Only ask the questions whose answers weren't pre-filled from `<config-root>/identity.md` in Step 0C.

- Your name
- Your work email (used to match against PM-scope defaults later)
- Your team or company name
- Your role on this team (e.g., Program Lead, Engineering Manager, Product Manager, Scrum Master, IC who runs ceremonies)

### Section 2 — Tracker

- What task tracker does your team use? (Asana / Jira / Linear / Shortcut / GitHub Issues / Trello / other)
- What's your tracker workspace identifier?
  - **Asana**: the workspace GID (a long number; find it in the URL when viewing your team's workspace home).
  - **Jira / Linear / Shortcut**: usually a slug or workspace name. Paste what your tracker's MCP expects.
  - **GitHub Issues**: `owner/repo` (the repo where issues live).
  - **No tracker yet**: skip this section; the skill will tell you to set one up before running workflows.
- What's the project identifier for your backlog (where bugs and features get triaged before being scheduled)? Paste the ID, slug, or URL — whatever the tracker uses.
- How are active sprints named in your tracker? Give an example name (e.g., "Sprint 15 (April 20 – May 1)") or a regex if your naming is consistent. The skill uses this to find the current sprint.
- What's your ticket ID prefix? (e.g., "BW" if your tickets are `BW-1234`, "ENG" if they're `ENG-1234`, or "none" if your tracker auto-IDs without a custom prefix.)

### Section 3 — Team

For each person on the sprint team, capture:

- Name
- Email (used for scope-check identity matching)
- Role — pick from: `tech_lead`, `product_lead`, `engineer`, `qa`, `designer`, `program_lead`, `other`. Multiple people can share a role (e.g., three engineers).

After capturing the roster, ask:

- Which team members default to **Product Management scope** in this skill? (Usually the Program Lead and Product Lead. Their email matches will give them write-capable behavior; everyone else defaults to read-only Personal scope.)

The user's own email from Section 1 is auto-suggested for PM scope; they can confirm or override.

### Section 4 — Context sources

The skill grounds every workflow in the last ~14 days of context — meeting recaps + a chat channel — before producing drafts. This catches decisions that happened in standup or chat but never made it into the tracker.

- Where do your meeting transcripts or recaps live? (Microsoft 365 Teams / Google Workspace Meet / none)
- Which chat channel grounds your context? Format depends on platform:
  - **Slack**: channel name (e.g., `#engineering` or `#sprint-team`)
  - **Microsoft Teams chat**: channel/team name
  - **Discord**: server + channel
  - **None**: chat grounding will be skipped
- How many days of context should the skill pull on each invocation? (Default: 14. Most teams want 7–21.)

### Section 5 — Ticket fields

This section captures your tracker's custom field schema. The skill needs to know which field gates whether a ticket is "ready for sprint planning poker" — and what values that field can hold.

- What's the name of your "ticket is ready for the sprint" field? (Examples: `Ready for Poker?`, `Sufficient User Story and AC?`, `Refined?`, `Definition of Ready`.)
- What are the values of that field? Identify:
  - The "ready" value (e.g., `Yes`, `Ready`, `Refined`).
  - The "drafted but not yet reviewed" value the skill should set when *it* drafts content. **This must be different from the "ready" value** — the skill should never silently mark a ticket as poker-ready. If your tracker doesn't have this distinction, the skill will use a generic comment instead. (Examples: `Drafted`, `Pending Review`, `AI-Draft`.)
  - The "not ready" value (e.g., `No`, `Not Yet`).
- What's your planning poker / estimate field called and what values does it accept? (Fibonacci `1, 2, 3, 5, 8` is the common default. T-shirt sizes also work.)
- What's your priority field called and what values does it accept? (`Critical / High / Med / Low` is common.)
- What's your ticket type field called and what values does it accept? (Examples: `Bug`, `Feature`, `Tech Debt`, `Spike`. List the values your team uses.)

For trackers that require option/enum GIDs rather than human-readable names when writing (Asana is one): if the user knows their GIDs, capture them here. If they don't, capture the human-readable names and note that the first write attempt may need a GID-lookup step.

### Section 6 — Member / customer feedback (optional)

If your team attributes feedback or feature requests back to specific members, customers, or users, the skill can populate attribution fields automatically when it sees a name or email in the source material.

- Do you track member-submitted ideas or customer-attributed feedback? (Yes / No)
- If yes:
  - What field holds the related member's name or email? (e.g., `Related Member`, `Customer`, `Requested By`.)
  - What field flags that a ticket came from member feedback? (e.g., `Member Submitted Idea?`, `From Customer`.)
  - What value indicates "yes, this came from a member"? (e.g., `Yes`, `True`.)
- If no: skip the rest of this section.

### Section 7 — Release notes (optional)

The release notes workflow compiles a sprint's production releases into a single document.

- Where do you publish release notes? (Outline / Confluence / Notion / a markdown file in a repo / Slack post / "we don't publish them")
- If you publish to a wiki: what's the collection or parent page name? (e.g., `Release Notes` in Outline; `Engineering / Releases` in Confluence.)
- If you publish to a file: what's the file path or repo destination?
- If you don't publish release notes: the release notes workflow will draft a markdown summary in chat without writing anywhere.

### Section 8 — Optional preferences

- **Sprint length.** Two-week sprints are the default. One-week or three-week sprints work too, but some phrasing in the skill (e.g., "end of week 1 progress check") may not match your cadence.
- **Production release cadence.** Some teams release once per sprint; some release weekly within a sprint; some release on demand. How does your team release?
- **Anything else** the skill should know about how your team works? (Free text. Optional.)

---

## Step 3 — Write the config

Populate `<config-root>/plugins/sprint-ops.user-context.md` with the answers. Format clearly under section headings — the skill reads this file every invocation, so structure it for fast scanning, not as raw YAML:

```markdown
# sprint-ops user context

_Last updated: [date]_

## Identity
- **Name:** ...
- **Email:** ...
- **Team / company:** ...
- **Your role:** ...

## Tracker
- **Tool:** ...
- **Workspace identifier:** ...
- **Backlog project identifier:** ...
- **Sprint naming pattern:** ...
- **Ticket ID prefix:** ...

## Team
| Name | Email | Role |
|---|---|---|
| ... | ... | ... |

**PM-scope members:**
- email-1
- email-2

## Context sources
- **Meetings:** m365 | google_workspace | none
- **Chat platform:** slack | teams | discord | none
- **Chat channel:** ...
- **Lookback days:** 14

## Ticket fields
### Ready-for-sprint gate
- **Field name:** ...
- **"Ready" value:** ...
- **"Drafted" value:** ...
- **"Not ready" value:** ...

### Planning poker
- **Field name:** ...
- **Values:** ...

### Priority
- **Field name:** ...
- **Values:** ...

### Ticket type
- **Field name:** ...
- **Values:** ...

## Member feedback
- **Enabled:** yes | no
- **Related-member field:** ...
- **Flag field:** ...
- **"Yes" value:** ...

## Release notes
- **Destination:** ...
- **Collection / path:** ...

## Preferences
- **Sprint length:** ...
- **Release cadence:** ...
- **Other:** ...
```

---

## Step 4 — Confirm and offer next step

After writing, summarize what was saved (one short paragraph) and offer concrete next steps:

- "Try `triage this feedback` (paste a bug report or user message and the skill will draft a ticket)."
- "Or ask `is the sprint ready for poker?` to do a readiness check."
- "Or just describe what you need — the skill routes between six workflows."

If member feedback was enabled, mention: "The skill will auto-populate your member-attribution fields when it spots a name or email in source material."

If meeting / chat context sources are set to `none`, mention: "Context grounding is disabled — workflows will skip the meeting / chat sweep. You can re-enable later via `/setup-sprint`."

---

## Behavior rules

- **One section at a time.** Don't paste all questions at once.
- **Skip what doesn't apply.** "No tracker yet" or "no release notes" are valid answers — capture as such.
- **Don't pad answers.** If the user says "skip Section 6," skip it. Note in the config that it was skipped so the skill knows to use defaults.
- **Idempotent.** Running `/setup-sprint` again should let the user update sections without re-doing everything.
- **Privacy-respecting.** Everything written goes to `<config-root>/plugins/sprint-ops.user-context.md`, which lives outside the plugin source tree and should be gitignored — never committed to a fork.
- **No tracker writes.** This command writes config only. It does not call the tracker MCP. It does not invoke the skill.
