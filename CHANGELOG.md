# Changelog

All notable changes to `sprint-ops` will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.1.0] — 2026-05-19

Initial release. Tracker-agnostic Program Lead workflow plugin, generalized from a working internal plugin originally built for ForFounder.

### Added

- `/setup-sprint` command — three-phase interview (resolve config root, read shared identity, capture sprint-ops settings)
- `sprint-ops` skill — routes user requests to one of six workflows
- Six workflow reference files: backlog triage, user story / AC drafting, sprint planning prep, mid-sprint progress check, retro seeding, production release notes
- Four shared reference files: team context loader, ticket quality rubric, sprint context check (meeting + chat grounding), scope check (PM vs. Personal)
- Compatible with BrightWay's nucleus config-root pattern (`~/Documents/.claude-plugin-config-root` → `<config-root>/plugins/sprint-ops.user-context.md`)
