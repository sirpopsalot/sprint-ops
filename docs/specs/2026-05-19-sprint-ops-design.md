# sprint-ops — Design Spec

**Date:** 2026-05-19
**Status:** Draft for review
**Owner:** Erica Hruby (sirpopsalot)
**Target audience for the deliverable:** Zach @ BrightWay AI, for possible inclusion in [BrightWayAI/nucleus](https://github.com/BrightWayAI/nucleus)

## Why this exists

`sprint-ops` is a generic, tracker-agnostic Claude Code plugin that applies a Program Lead workflow to sprint-management tasks: turning raw inputs (feedback, bug reports, meeting transcripts) into well-formed tickets, preparing the sprint for kickoff and poker estimation, running a mid-sprint progress check, seeding the retrospective, and compiling production release notes.

It is the generic, configurable descendant of the `product-owner-forfounder` plugin in [sirpopsalot/forfounder-claude-plugins](https://github.com/sirpopsalot/forfounder-claude-plugins) — same workflow architecture, with every team-specific reference (names, channels, tracker IDs, field schemas, ticket prefixes) lifted into user-supplied config.

It exists so Zach can evaluate it as a candidate for inclusion in `BrightWayAI/nucleus` — slotting in as a sibling to `weekly-alignment`, `core-ops`, etc., filling a gap in nucleus's coverage of engineering team sprint operations.

## Non-goals

- A polished marketplace of its own — nucleus is the marketplace; this is a single-plugin repo.
- Tracker-specific MCP integrations. The plugin describes *what* it needs in tracker-neutral language and lets Claude select the right MCP tools at runtime based on what the user has installed.
- A universal field-schema mapper. The plugin asks the user to provide their tracker's field names during setup and uses those values directly. No translation layer.
- Standup facilitation, PRD authoring, or product strategy work — those are out of scope by design, same as the source plugin.

## Architecture

### Repo layout

```
sprint-ops/
├── .claude-plugin/
│   └── plugin.json              # plugin manifest
├── commands/
│   └── setup-sprint.md          # /setup-sprint — captures user config
├── skills/
│   └── sprint-ops/
│       ├── SKILL.md
│       └── references/
│           ├── team-context.md
│           ├── ticket-quality.md
│           ├── sprint-context-check.md
│           ├── scope-check.md
│           ├── workflow-backlog-triage.md
│           ├── workflow-user-story-ac-drafting.md
│           ├── workflow-sprint-planning-prep.md
│           ├── workflow-progress-check.md
│           ├── workflow-retro-prep.md
│           └── workflow-release-notes.md
├── docs/
│   └── specs/
│       └── 2026-05-19-sprint-ops-design.md   # this file
├── README.md
├── CHANGELOG.md
└── .gitignore
```

**Differences from the source repo:**

- No top-level `.claude-plugin/marketplace.json`. Nucleus is the marketplace; this is a single-plugin repo that nucleus's manifest will reference by `source: { source: github, repo: BrightWayAI/sprint-ops }` (or `sirpopsalot/sprint-ops` for direct eval).
- A `commands/` directory with `setup-sprint.md`, matching the BrightWay pattern (`/setup-core`, `/setup-news`, `/setup-time`, `/setup-projects`).
- One skill (`sprint-ops`), not two. The source repo has a separate `production-release-notes` skill duplicating content already routed through `workflow-release-notes.md` in `product-owner`. Consolidating to one skill drops duplication and reduces the eval surface for Zach.

### Component responsibilities

| Component | Responsibility | Boundary |
|---|---|---|
| `plugin.json` | Plugin manifest. Static. | No logic. |
| `setup-sprint.md` (command) | Interview user, write config to `.claude/sprint-ops.local.md`. | Writes config only. Does not read from tracker. Does not invoke the skill. |
| `SKILL.md` | Route the user's request to the right workflow. Enforce shared behaviors (context check, scope confirmation, never-overwrite, draft-then-confirm). | Owns routing + cross-workflow invariants. Does not contain workflow logic. |
| `references/team-context.md` | How to load team roster, ceremony cadence, tool stack from config. | Read-only. Describes what to read from config, not what to do with it. |
| `references/ticket-quality.md` | Generic ticket rubric (Context / User Story / AC). Naming rules. How to read field mappings from config. | No tracker-specific field GIDs. Tracker-neutral language only. |
| `references/sprint-context-check.md` | Procedure for pulling recent meeting recaps + chat context. | Reads source from config (M365 / Google Workspace / Slack / none). Skips gracefully if config says "none". |
| `references/scope-check.md` | Personal vs. Product Management scope rules. Identity-driven defaults. | Matches `# userEmail` from system context against config's PM-scope user list. |
| `references/workflow-*.md` | One file per workflow. Concrete instructions, output formats, write-confirmation rules. | Each workflow is independently invokable. Workflows do not call each other. |

### Data flow (typical: backlog triage)

1. User pastes feedback + says "turn this into tickets"
2. `SKILL.md` routes to `workflow-backlog-triage.md`
3. Before the workflow runs, `SKILL.md` triggers two shared behaviors:
   - Run `sprint-context-check.md` to pull recent meeting + chat context (configurable sources)
   - Run `scope-check.md` to confirm PM vs. Personal scope
4. `workflow-backlog-triage.md` reads config to learn the tracker, the backlog project identifier, the field schema
5. Claude drafts tickets in chat using `ticket-quality.md`'s Context / User Story / AC template
6. User reviews, edits, says "yes" or "edit and yes"
7. Claude calls whatever tracker MCP the user has installed (e.g. `mcp__asana__*`, `mcp__linear__*`) to create the tickets with the configured field values
8. Claude reports back: created tickets with their URLs

No workflow ever calls `mcp__asana__*` or `mcp__linear__*` by hardcoded name. Workflows describe the action and Claude selects the tool. Concretely, where the source plugin says:

> Use `asana_get_projects_for_workspace` with workspace `1209885295218312` and filter to names matching `^Sprint \d+\s*\(.+\)`

the generalized version says:

> List active projects in the tracker workspace (`config.tracker.workspace_id`) using whichever tracker MCP the user has installed. Filter to projects whose names match `config.tracker.sprint_project_pattern`.

Asana is documented in setup as a known-working example, but not assumed.

## Configurability — the `/setup-sprint` pattern

Matches BrightWay's existing pattern (`/setup-core`, `/setup-news`, etc.): a one-time interview command that writes config to `.claude/sprint-ops.local.md` using YAML frontmatter.

### Config schema

```yaml
---
tracker:
  name: "Asana"               # display name; informs setup walkthrough only
  workspace_id: "1209885295218312"  # whatever the user's tracker calls it
  backlog_project_id: "1210533877425463"
  sprint_project_pattern: "^Sprint \\d+\\s*\\(.+\\)"  # regex for active sprint projects

team:
  members:
    - name: "Diego"
      email: "diego@example.com"
      role: "tech_lead"
    - name: "Olivia"
      email: "olivia@example.com"
      role: "product_lead"
    - name: "Anne"
      email: "anne@example.com"
      role: "qa"
  pm_scope_emails:
    - "olivia@example.com"
    - "your-email@example.com"

context_sources:
  meetings: "m365"            # m365 | google_workspace | none
  chat:
    platform: "slack"         # slack | none
    channel: "#development-team"
  lookback_days: 14

ticket_fields:
  user_story_status:
    field_name: "Sufficient User Story and AC?"
    values:
      drafted: "Drafted"
      ready: "Yes"
      not_ready: "No"
      needs_revert: "Needs Revert"
  planning_poker:
    field_name: "Planning Poker"
    values: [1, 2, 3, 5, 8]
  priority:
    field_name: "Priority"
    values: ["Critical", "High", "Med", "Low"]
  ticket_type:
    field_name: "Ticket Type"
    values:
      - "Tech"
      - "New Feature"
      - "Improvement"
      - "Bug"
  ticket_id:
    field_name: "TICKET_ID"   # the human-readable prefix field, e.g. "FF" for FF-1234
    prefix: "FF"              # optional — leave blank if your tracker auto-IDs

member_feedback:               # optional — set to null/omit to disable
  enabled: true
  related_field: "Related Member"
  flag_field: "Member Submitted Idea?"
  flag_yes_value: "Yes"

release_notes:
  destination: "outline"      # outline | confluence | notion | markdown_file
  collection_or_path: "Release Notes"
---
```

### Setup walkthrough

`/setup-sprint` walks the user through ~10 questions in order:

1. What tracker do you use? (free text — informs walkthrough wording)
2. What's your tracker's workspace identifier? (with help text per tracker)
3. What's your backlog project identifier?
4. How are active sprints named in your tracker? (regex or example)
5. Who's on the team? (collect name / email / role for each)
6. Which team members default to Product Management scope?
7. Where do meeting transcripts live? (M365 / Google Workspace / none)
8. Which chat channel grounds your context? (or "none")
9. What's your ticket-readiness field called, and what values does it have? (with FF example shown)
10. Do you track member-submitted ideas? (yes → field names; no → omit)
11. Where do release notes go? (Outline / Confluence / Notion / markdown file)

Each step is a single question with a recommended default and an "Other" escape hatch. On completion, write `.claude/sprint-ops.local.md` and show the user a one-line summary of what was captured.

### How references read config

Every reference file begins with:

> Before doing anything in this workflow, read `.claude/sprint-ops.local.md`. If it doesn't exist, stop and tell the user to run `/setup-sprint` first.

References then refer to config values by their YAML path (e.g. `config.tracker.backlog_project_id`) when describing actions. This is documentation shorthand for Claude — there's no literal template engine. Claude reads the config file at the start of a workflow and substitutes the values mentally when it acts. No build step, no preprocessing.

## What's stripped vs. kept

### Stripped (FF-specific → config-driven)

- All team member names (Diego, Olivia, Anne, Erica) → `config.team.members`
- `#development-team` Slack channel → `config.context_sources.chat.channel`
- `FF-XXXX` ticket IDs → `config.ticket_fields.ticket_id.prefix`
- Hardcoded Asana workspace `1209885295218312`, BACKLOG project `1210533877425463`, sprint project GIDs → `config.tracker.*`
- Asana custom field GIDs and enum option GIDs throughout `ticket-quality.md` → config field-name strings, used as the tracker MCP expects (most accept human-readable names; if user's tracker requires GIDs, they paste those values in config)
- `Related Member` / `Member Submitted Idea?` workflow → optional `config.member_feedback` block
- Outline-specific release notes destination → `config.release_notes.destination`
- `forfounder-context.md` file → `team-context.md`, which describes how to *read* team context from config
- `ehruby@forfounder.com` / `olivia*@forfounder.com` → `config.team.pm_scope_emails`

### Kept (the differentiating workflow value)

- The six workflows: backlog triage, user story / AC drafting, sprint planning prep, mid-sprint progress check, retro seeding, release notes
- Two-week context grounding (meetings + chat) before every workflow
- PM vs. Personal scope split with soft-ask
- "Drafted" vs. "Ready" gating to prevent silent poker-ready signals
- "Never overwrite existing ticket content — append a clearly-labeled draft section below" guarantee
- "Don't pad, don't editorialize" voice in skill prompts
- Read-only escape hatch ("no draft", "just tell me", "info only")
- Confirmation required before any tracker write
- Six-workflow routing logic in `SKILL.md`

## Error handling

- **No config file present:** every reference checks for `.claude/sprint-ops.local.md` at the top and stops with a clear setup prompt if missing.
- **Incomplete config:** references check the specific keys they need (e.g. `workflow-release-notes.md` checks `config.release_notes.destination`). If a needed key is missing, stop and tell the user which `/setup-sprint` question to re-answer.
- **No tracker MCP installed:** Claude will not find a tool to call; the workflow stops at the "create tickets" step with a clear message ("I don't see a tracker MCP installed — install one for Asana / Jira / Linear / etc. and re-run").
- **Context sources unavailable** (e.g. M365 not connected): `sprint-context-check.md` already handles this in the source plugin — keep that behavior, generalized to whatever source the user configured.
- **Tracker write fails:** report the failure verbatim in chat. Do not retry silently. Do not partially write (if creating 5 tickets and ticket 3 fails, report and stop — the user decides whether to retry from ticket 3).

## Testing

Before declaring the plugin shippable:

1. **Install locally and run `/setup-sprint`** with mock answers. Confirm `.claude/sprint-ops.local.md` is written correctly.
2. **Run each of the six workflows** against a test Asana workspace with a real sprint project. Confirm:
   - Workflow reads config correctly
   - Tickets are drafted in chat with the right structure
   - Write only happens on explicit confirmation
   - Existing ticket content is preserved (verify by editing a ticket with existing notes)
   - `user_story_status` is set to the "drafted" value, not "ready"
3. **Run a workflow without config present.** Confirm clean error + setup prompt.
4. **Run a workflow with a non-PM-scope user simulated** (override `# userEmail`). Confirm read-only escape hatch and scope soft-ask.
5. **README readability check:** ask someone unfamiliar with the source plugin (could be Claude) to read the README and explain what it does and how to install it. If they can't, iterate.

No automated test suite — this is a prompt-driven plugin, behavioral testing is the right granularity.

## README / handoff framing

Written for Zach's evaluation, not end users:

- **One-line description** matching nucleus marketplace tone
- **What this is**: origin (genericized from a working ForFounder team plugin), what's been lifted into config, what's still opinionated
- **How it fits nucleus**: sibling to `weekly-alignment` / `core-ops`; fills the engineering-team sprint-ops gap
- **Install**: `/plugin marketplace add sirpopsalot/sprint-ops` for direct eval, or fork to `BrightWayAI/sprint-ops` for nucleus inclusion
- **Setup**: run `/setup-sprint`, answer ~10 questions
- **Six workflows section**: what each does, what it requires from config
- **Opinionated choices flagged honestly**: e.g. "this plugin assumes two-week sprints; one-week and three-week sprints will mostly work but the 'end of week 1' progress check phrasing is hardcoded for two-week cadence"
- **Known limitations**: no automated tracker schema discovery; Jira/Linear users may need to paste field IDs into config manually depending on their MCP

## Open questions (to resolve during implementation)

1. **Config storage location.** BrightWay's `plugin-settings` skill documents `.claude/<plugin-name>.local.md` for per-project config. For a per-user plugin like this, `~/.claude/sprint-ops.local.md` may be more appropriate. Confirm during implementation by checking how `core-ops` and `news-curator` store their settings.
2. **Default tracker example in README.** Use Asana (matches the source plugin and is well-tested) vs. Linear (cleaner public API, possibly more common in BrightWay's client base). Defaulting to Asana for now; revisit if Zach has a preference.
3. **Whether to ship a sample config.** A `examples/sprint-ops.local.md` showing a filled-in config could speed up evaluation. Lean yes; flag if it creates maintenance burden.

## Out-of-scope follow-ups

- A nucleus PR adding `sprint-ops` to `BrightWayAI/nucleus/.claude-plugin/marketplace.json` — that's Zach's call after evaluating.
- Linear / Jira / Shortcut-specific reference files. If demand exists, those could be added as `references/tracker-linear.md` etc., but the plugin should work without them.
- A `setup-sprint` test mode that validates config against the actual tracker (e.g. confirms `backlog_project_id` resolves to a real project). Useful but adds tracker-specific logic; defer.
