# Sprint context check (run before every workflow)

## Required setup

Before doing anything in this workflow, resolve config:

1. Read `~/Documents/.claude-plugin-config-root` to find the config root.
2. Read `<config-root>/plugins/sprint-ops.user-context.md` for plugin config.
3. Read `<config-root>/identity.md` for shared identity (optional — skip if missing).

If the pointer file or plugin config doesn't exist, stop and tell the user to run `/setup-sprint` first.

**Goal:** before any sprint-ops workflow runs, gather recent sprint context so drafts and recommendations are grounded in what the team actually said and did — not just what's in the tracker. This catches decisions made in standup that never made it to a ticket, retro action items that haven't been formalized, and chat discussions that change the right answer.

This check is required for every workflow in this skill: backlog triage, user-story / AC drafting, sprint planning prep, mid-sprint progress check, retro prep, and release notes. Skip it only if the user explicitly says "skip context" or "I just need this fast — don't pull meetings."

## What to gather

Scope is controlled by config:

- **Meetings** — only if `config.context_sources.meetings` is set (`m365`, `google_workspace`). If `none`, skip the meetings step entirely.
- **Chat** — only if `config.context_sources.chat.platform` is set (`slack`, `teams`, `discord`). If `none`, skip the chat step. Use the channel at `config.context_sources.chat.channel`.
- **Lookback window** — `config.context_sources.lookback_days` (default 14).

Typical haul for a two-week sprint with both sources configured: 1 planning, 2–3 standups, possibly 1 retro, possibly 1 product check-in, plus the configured chat channel.

Don't broaden scope without asking. If the user says "look further back" or "include another channel too", that's a one-off override — confirm and proceed.

## Connector preflight

Before fetching, verify the configured connectors are available. If a configured source is missing its connector, stop and prompt for setup (see "Missing connector" below). Don't silently skip. Probe only the sources your config asks for.

| Configured source | What to probe |
|---|---|
| `meetings: m365` | Run a small SharePoint search. If it returns without an auth error, you're connected. |
| `meetings: google_workspace` | Run a small Google Drive search. If it returns without an auth error, you're connected. |
| `chat: slack` | Resolve the configured channel by name. If it resolves, you're connected. |
| `chat: teams` | List Teams channels and confirm the configured channel is accessible. |
| `chat: discord` | Resolve the configured channel/guild. If it resolves, you're connected. |

If a probe returns an auth/permission error, treat the connector as missing.

## Step 1 — Pull meeting context

Skip entirely if `config.context_sources.meetings == "none"`.

Two-pass approach regardless of platform: first search the document store directly for recap/transcript artifacts, then fall back to the calendar to enumerate meetings and resolve each to its recap.

### If `meetings == "m365"`

Recordings, transcripts, and Copilot recaps for Teams meetings land in SharePoint/OneDrive (the recording owner's "Recordings" folder, surfaced to attendees).

**Pass A — SharePoint search (preferred, faster).** Use the M365 MCP to search SharePoint with queries like `sprint planning OR standup OR retrospective OR retro`, plus targeted queries by likely meeting title (`"Sprint planning"`, `"Standup"`, `"Retrospective"`, `"Refinement"`). Narrow to the lookback window if the tool supports a date filter; otherwise filter results post-hoc by modified date.

Look for files matching recap/transcript patterns: `*Recap*`, `*Transcript*`, `*Meeting Notes*`, `.vtt`, `.docx`, `.txt`. Recordings (`.mp4`) usually have a sibling `.vtt` or `.docx` — prefer the text artifact. Read each promising hit.

**Pass B — Outlook calendar fallback.** If SharePoint comes back thin, search the Outlook calendar for the same meeting titles within the lookback window. For each calendar hit, the meeting body often contains a "Meeting recap" link pointing into SharePoint — follow it.

### If `meetings == "google_workspace"`

Same two-pass shape, different tools.

**Pass A — Google Drive search.** Use the Google Drive MCP to search for meeting recap / transcript files with the same title patterns above. Google Meet recordings land in the organizer's Drive; Gemini-generated recaps usually sit alongside them as Docs. Filter to the lookback window via Drive's `modifiedTime`.

**Pass B — Google Calendar fallback.** Search Google Calendar for the same meeting titles. Calendar events with recordings/recaps will link to the Drive artifact in the description or attachments — follow the link.

### What to extract (either platform)

For each meeting, capture in working notes (not in chat):

- Meeting title + date
- Decisions made (especially scope changes, prioritization calls, deferrals)
- Action items assigned (owner + what)
- Blockers or risks raised
- Tickets explicitly named (e.g., `<PREFIX>-XXXX` using `config.tracker.ticket_id_prefix`)
- Member or stakeholder names mentioned (for ticket attribution downstream)

If you find meetings on the calendar but no recap exists, that's information itself — note it and move on. Don't fabricate notes from a meeting title alone.

## Step 2 — Pull chat context

Skip entirely if `config.context_sources.chat.platform == "none"`.

Resolve `config.context_sources.chat.channel` on the configured platform, then read the last `config.context_sources.lookback_days` of messages, including thread replies.

Read messages in chronological order. Capture:

- Decisions or "let's just do X" moments
- Bugs flagged that may not have a ticket yet
- Ticket references (cross-reference back to the tracker)
- Discussions that change the shape of in-flight work
- Member- or stakeholder-reported issues being relayed by the product lead or QA

If the channel is high-volume, focus on threads with reactions, replies, or `@mentions` of the people in `config.team.members` (especially the tech lead, product lead, and QA).

## Step 3 — Use the context, don't dump it

Hold the gathered context in working memory for the rest of the workflow. When you produce drafts later, cite specific signals — e.g., "AC reflects the tech lead's call in Tuesday's standup that the empty-state should match the existing pattern" or "noting the log-noise concern raised in the team channel last Thursday."

Don't dump the raw context into chat. You know what was said in your own meetings — your job is to *use* the context, not recite it.

If the gathered context contradicts something the user just said, surface that explicitly: "Heads up — in Wednesday's standup the tech lead said this would be deferred to the next sprint. Want me to reflect that, or is the call different now?"

## Missing connector — prompt and instructions

If a configured connector is unavailable, stop the workflow and show this prompt before doing anything else.

### Template message

> I want to ground this in the last `<lookback_days>` days of sprint context, but I'm missing a connector:
>
> - **`<meetings platform>`**: [✅ connected | ❌ not connected]
> - **`<chat platform>`**: [✅ connected | ❌ not connected]
>
> Connect the missing one and I'll pick this up. Steps below.

Show rows only for sources you have configured. Fill in the checkmarks from the preflight results. Then include setup steps only for the connector(s) that are missing.

### Generic setup template

For each missing connector, give the user this shape (substitute the platform-specific tokens):

1. In Cowork, open **Settings → Connectors**.
2. Find **`<connector name>`** in the list and click **Connect**.
3. A browser window will open — sign in with the account that has access to `<the relevant workspace / org / tenant>`.
4. Approve the requested permissions: `<scope list for this platform>`. All scopes are needed — declining any will leave gaps.
5. Return to this chat and tell me "connected" — I'll re-check and continue.

If the connector is already listed but a probe returns an auth error, the token has likely expired. Click **Reconnect** on the same row.

### Platform-specific tokens for the template

- **Microsoft 365 (Teams + SharePoint + Outlook)** — connector name: `Microsoft 365`. Scopes: Teams chat, calendar, SharePoint, OneDrive.
- **Google Workspace** — connector name: `Google Workspace` (or separate `Google Drive` + `Google Calendar` if listed individually). Scopes: Drive read, Calendar read.
- **Slack** — connector name: `Slack`. Scopes: read public channels you're a member of. Confirm the user is a member of `config.context_sources.chat.channel` — the connector can't read channels they're not in.
- **Microsoft Teams (chat)** — connector name: `Microsoft Teams` (often bundled under Microsoft 365). Scopes: channel messages read. Confirm membership in the configured channel.
- **Discord** — connector name: `Discord`. Scopes: read messages in the configured guild/channel.

### When the user declines to connect

If the user says "skip it, just proceed", proceed without the context — but warn once at the top of your output:

> Proceeding without the sprint context check. Drafts may miss decisions made in meetings or chat discussions that haven't reached the tracker yet. Re-run later if you'd like a re-grounded version.

Don't repeat the warning every turn. Once is enough.

## Edge cases

- **Sprint just started, less than the lookback window of context exists** — pull what's there. If the previous sprint ended within the window, include its retro and final standups.
- **Holiday or off-week with no meetings** — say so. "No standups in the last `<lookback_days>` days; sprint planning is the only meeting in scope." Don't pad.
- **Recap exists but is empty / auto-notes didn't generate** — note the meeting happened, mark recap as missing, move on.
- **Same workflow run twice in one session** — cache the gathered context in working memory; don't re-pull unless the user says "refresh" or more than ~30 minutes have passed.
- **User explicitly disables this check** — phrases like "skip context", "don't pull meetings", or "just use the tracker" — honor it. No warning needed beyond a one-line acknowledgement.
- **Only one source configured** — run the check against whatever is configured. Don't prompt the user to add the missing source; that's a `/setup-sprint` decision, not a per-run one.
