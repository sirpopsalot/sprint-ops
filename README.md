# sprint-ops

A Program Lead workflow for any team running sprints. Turns raw inputs (feedback, bug reports, meeting transcripts) into well-formed tickets, preps the sprint for kickoff and poker estimation, runs a mid-sprint progress check, seeds the retrospective, and compiles production release notes.

Tracker-agnostic — works with Asana, Jira, Linear, Shortcut, GitHub Issues, or anything else you have an MCP for. Configurable per user via `/setup-sprint`.

---

## What this is

`sprint-ops` is a generic, tracker-agnostic version of an internal Claude plugin originally built for a single team running two-week sprints. The workflow architecture (six routines below + shared discipline) is unchanged from that battle-tested original; everything team-specific has been lifted into user-supplied config.

**Not a marketplace** — this is a single-plugin repo. Install it directly from this repo, or fork it into your own org and add it to a marketplace manifest.

## Six workflows

The skill routes the user's request to one of these based on phrasing cues. All workflows share the same discipline (context grounding, scope check, never-overwrite, draft-then-confirm, "drafted" vs. "ready" gating).

| Workflow | Trigger phrases | What it does |
|---|---|---|
| **Backlog triage** | "triage this feedback", "turn this into tickets", "clean up triage" | Turns pasted feedback / bug reports / transcripts into well-formed tickets in the backlog project, drafted and waiting for human review |
| **User story / AC drafting** | "draft AC for X", "fill in the user story", "write acceptance criteria" | Drafts Context / User Story / Acceptance Criteria on an existing ticket, appending below existing content (never overwriting) |
| **Sprint planning prep** | "is the sprint ready?", "which tickets aren't ready for poker?" | Audits the current sprint against the ticket-quality rubric, surfaces what's missing |
| **Mid-sprint progress check** | "sprint health check", "are we on track?", "draft a status update" | Assesses tickets in flight, blocked, and at risk; produces a status-ready summary for chat |
| **Retro seeding** | "scan the sprint for retro items", "prep retro topics" | Surfaces patterns, blockers, and wins from the sprint as concrete retro discussion items. Also: turns a retro transcript into actionable tickets. |
| **Release notes** | "compile release notes", "what shipped this sprint" | Compiles a sprint's production releases into a single document and publishes to the configured destination |

## Shared discipline across every workflow

These are the patterns that make the skill trustworthy in practice — preserved verbatim from the original:

- **Two-week context grounding.** Before any workflow, pull recent meeting recaps + chat channel context. Catches decisions made in standup or chat that never made it back into the tracker. Configurable lookback window; configurable sources (M365, Google Workspace, Slack, Teams, Discord, or none).
- **PM vs. Personal scope.** Identity-based default: emails in your configured PM list get write-capable behavior; everyone else defaults to read-only Personal scope. Soft-ask before crossing the boundary.
- **Drafted, not ready.** When the skill drafts content into a ticket, it sets the "ready-for-sprint" gate field to its *drafted* value, never the *ready* value. The user reviews and flips to ready themselves. This prevents silent "this ticket is poker-ready" signals.
- **Never overwrite.** Existing ticket content is preserved verbatim. New draft content is appended below the existing content with a clear marker.
- **Confirmation before any tracker write.** Every workflow produces drafts in chat. Silence is not approval.
- **Read-only escape hatch.** Phrases like "no draft" / "just tell me" / "info only" suppress all draft generation. Useful when you want to think about a ticket without triggering the write machinery.
- **Don't pad.** Empty sections say "None this sprint" — the skill doesn't invent filler.

## Install

### Direct (for evaluation)

```
/plugin marketplace add sirpopsalot/sprint-ops
/plugin install sprint-ops
```

### Via a marketplace (recommended for ongoing use)

Add this entry to your marketplace's `marketplace.json`:

```json
{
  "name": "sprint-ops",
  "source": { "source": "github", "repo": "<your-org>/sprint-ops" },
  "description": "Program Lead workflow for any team running sprints. Tracker-agnostic.",
  "author": { "name": "<your-org>" }
}
```

## First-time setup

Run `/setup-sprint`. The interview runs in three phases, matching the pattern used by [BrightWayAI/core-ops](https://github.com/BrightWayAI/core-ops):

1. **Resolve config root** — picks the folder where all your Claude plugin configs live (one-time bootstrap if this is your first plugin in the family). Default suggestion: `~/Documents/Claude/`. The choice is recorded in `~/Documents/.claude-plugin-config-root`.
2. **Read shared identity** — if you have `claude-cortex` installed and have run `/setup-identity`, name/email/company/role are pre-filled from `<config-root>/identity.md`. Otherwise, captured inline.
3. **The interview** — ~9 questions if Phase 2 pre-filled identity, ~12 if not. Covers: tracker (which one, workspace, backlog project, sprint naming, ticket prefix), team roster + PM-scope emails, context sources (meetings + chat), ticket field schema (the ready-for-sprint gate, planning poker, priority, ticket type), optional member-feedback attribution, optional release-notes destination, optional sprint-length and release-cadence preferences.

Config is written to `<config-root>/plugins/sprint-ops.user-context.md` as human-readable markdown sections. Re-run `/setup-sprint` anytime to update individual sections.

## Composes with `claude-cortex`

If you install [BrightWayAI/claude-cortex](https://github.com/BrightWayAI/claude-cortex) and run `/setup-identity`, sprint-ops will pre-fill identity questions from the shared `identity.md` instead of re-asking. The same shared file is used by other nucleus plugins (`core-ops`, `news-curator`, `weekly-outreach`, etc.) — set it once, every plugin reads it.

Cortex is not a hard dependency. Without it, `/setup-sprint` captures identity inline.

## What gets installed

```
sprint-ops/
├── .claude-plugin/plugin.json
├── commands/
│   └── setup-sprint.md
├── skills/
│   └── sprint-ops/
│       ├── SKILL.md                # routing + shared behaviors
│       └── references/
│           ├── team-context.md     # how to read team config
│           ├── ticket-quality.md   # ready-for-poker rubric, body template
│           ├── sprint-context-check.md   # meeting + chat grounding
│           ├── scope-check.md      # PM vs. Personal logic
│           └── workflow-*.md       # six workflow files
├── docs/specs/
│   └── 2026-05-19-sprint-ops-design.md
├── README.md
└── CHANGELOG.md
```

The skill itself is ~1,250 lines of prompt content. The `setup-sprint` command is ~250 lines. No code — pure prompt-driven plugin.

## Opinionated choices (called out so you can decide whether to keep them)

- **Two-week sprints are the documented default.** Other cadences work, but some phrasing ("end of week 1 progress check") assumes two-week timing. The user is expected to adapt the phrasing or override via the optional cadence config.
- **The skill always sets the ready-for-sprint gate to "drafted", never "ready".** Reviewing AC and flipping the gate is the user's job. If your tracker doesn't have a distinct "drafted" value in its workflow, the skill leaves the field unset and adds a comment instead.
- **Six workflows, no more.** The skill is deliberately scoped. It does not facilitate standups, write PRDs, or run product strategy. If you want it to do something else, write another skill — don't expand this one.
- **Tracker writes always require explicit user confirmation.** This is non-negotiable in the source plugin's design; preserved here. If you want auto-write behavior, that's a fork, not a config flag.
- **Context grounding (meetings + chat) is on by default.** Workflows pull the last 14 days of context before drafting. If your team genuinely doesn't have meetings or a primary channel, configure `context_sources.meetings: none` and `context_sources.chat.platform: none` and the skill skips the sweep.

## Known limitations

- **No automated tracker schema discovery.** Users provide their tracker's field names and values during setup. For trackers that require enum option GIDs when writing (Asana), users may need to paste GIDs into config — the skill doesn't currently auto-resolve them.
- **Release notes marker convention.** The release-notes workflow looks for tasks in the sprint project matching a "release marker" naming pattern (default suggestion: `Production Release - YYYY-MM-DD`). Teams without this convention need to either adopt one or configure their own pattern.
- **No tests.** This is a prompt-driven plugin; behavior is validated by running the workflows against a real tracker. There's no automated test suite.
- **English only.** Prompts are written in English. They will likely work for other languages with config values translated, but it's untested.

## Origin

This plugin was generalized from `product-owner-forfounder`, an internal plugin originally built for the ForFounder team. The source plugin lives at [sirpopsalot/forfounder-claude-plugins](https://github.com/sirpopsalot/forfounder-claude-plugins) — it has additional team-specific context and a marketplace manifest that aren't useful in a generic context.

The generalization preserves every workflow pattern from the source. What was lifted into config:

- All team member names → `config.team.members`
- Specific chat channel → `config.context_sources.chat.channel`
- Ticket ID prefix → `config.tracker.ticket_id_prefix`
- Asana workspace and project GIDs → `config.tracker.*`
- Asana custom field GIDs and enum option GIDs → `config.ticket_fields.*` (human-readable names; users supply GIDs if their tracker needs them)
- Outline as the release-notes destination → `config.release_notes.destination`
- Microsoft 365 + Slack as the fixed context sources → configurable across M365 / Google Workspace / Slack / Teams / Discord / none

See [docs/specs/2026-05-19-sprint-ops-design.md](docs/specs/2026-05-19-sprint-ops-design.md) for the full design rationale.

## License

MIT.
