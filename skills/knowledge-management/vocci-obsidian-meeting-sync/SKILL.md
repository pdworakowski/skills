---
name: vocci-obsidian-meeting-sync
description: Turn Vocci ring-captured meetings/conversations into a linked Obsidian vault of atomic Meeting, Task, and Project notes, with auto-refreshed Upcoming Tasks / Project Status / Meeting Log views. Use when the user wants Vocci meetings synced to Obsidian, an Obsidian "second brain" fed from Vocci recordings, action items pulled out of meetings and tracked in Obsidian, or asks to sync/import/export Vocci sessions into a notes vault. Captures land in an Inbox as drafts (ai_access: review-only) and only become durable Meeting/Task/Project notes after an explicit promote step — nothing is auto-filed sight-unseen. Supports separate private and work vaults (scope-locked, so a session can't land in the wrong one) and chaining downstream automation off each promotion (n8n/Zapier/Make webhooks — e.g. push high-priority tasks to Todoist, post meeting digests to Slack, mirror tasks into Notion/Airtable). Local filesystem access is the default and recommended path; remote (Local REST API) access is opt-in and covered separately.
---

# Vocci → Obsidian meeting sync

Pulls meeting/conversation data from Vocci MCP, has the agent (you) extract
action items, deadlines, project context, and a sensitivity classification
with judgment, then hands a structured payload to `scripts/vault_sync.py`.
The script never talks to Vocci and never decides project/priority/
classification itself — that's your job. Its job is to stay mechanically
correct: consistent frontmatter, no duplicate sessions, no notes promoted
without a human having seen them first.

**Two-phase workflow, always** — `capture` then `promote`. Nothing becomes
a real Meeting/Task/Project note, and nothing is sent to any downstream
webhook, until a human has looked at what was captured and said to proceed:

```
Vocci session
  → capture   → 00_Inbox/Vocci/<draft>.md   (status: to-process, ai_access: review-only)
  → [ you review it, in chat or in Obsidian ]
  → promote   → Meetings/ + Tasks/ + Projects/ (ai_access: approved) + webhook fires
       (or)
  → reject    → draft marked rejected, nothing else happens
```

Never call `promote` in the same turn as `capture` without the user
explicitly confirming what's in the draft first — showing them the
captured summary/tasks/classification in chat and getting a go-ahead
counts; silently chaining capture → promote does not.

## Vault layout this produces

```
<vault>/
  00_Inbox/Vocci/2026-09-27 - Klient ABC — Discovery.md   # draft, pre-approval
  Meetings/2026-09-27 - Klient ABC — Discovery.md          # after promote
  Tasks/2026-09-27 - Prepare process workshop agenda.md    # after promote
  Projects/Klient ABC.md                                   # after promote
  Views/Upcoming Tasks.md      # regenerated on every promote; Inbox drafts never appear here
  Views/Project Status.md      # regenerated on every promote
  Views/Meeting Log.md         # regenerated on every promote
  .vocci-sync/state.json       # session_id -> status/files, for dedup
  .vocci-sync/vault_scope.json # this vault's locked scope (private|work)
  .vocci-sync/pending/*.json   # cached payloads for captured-but-not-yet-promoted sessions
```

Every note carries plain-scalar YAML frontmatter (`project`, `priority`,
`status`, `due`, `scope`, `ai_access`, `vocci_session_id`, ...) and the note
*body* carries the actual `[[wikilinks]]` — frontmatter stays plain scalars
so the views script can parse it back out reliably. `Views/*.md` are plain
markdown tables the script computes from that frontmatter — no community
plugin required. See "Optional: live queries instead" below for Dataview/
Bases as an upgrade.

## Step -1 — two vaults, not one

Private and work content go in **physically separate vaults** —
`Private-Life-OS` and (for example) `HANBUD-Sales-OS` — never one vault
with just a tag distinguishing them. A tag is a label on content that got
read either way; a separate vault is content a query pointed at the other
vault cannot see at all.

Each vault gets locked to one scope the first time you run `setup` against
it:

```bash
python3 scripts/vault_sync.py setup --vault "/path/to/Private-Life-OS" --scope private
python3 scripts/vault_sync.py setup --vault "/path/to/HANBUD-Sales-OS"  --scope work
```

`setup` is idempotent for a matching `--scope`, but **refuses** to
re-declare a vault under a different scope — that's the guardrail against a
copy-paste mistake putting work content in the private vault. Every later
`capture` into that vault must pass the matching `--scope`; a mismatch is
refused too, before anything is written. If the user hasn't told you which
vault is which, ask — don't guess which path is "the work one."

If the user says everything going through this skill so far is one kind of
content (all personal, or all work), a single vault is fine — just run
`setup` once with that one scope and don't bother standing up a second
vault until it's actually needed.

## Step 0 — pick a backend: local by default

- **`--backend fs`** (default, recommended) — direct filesystem writes.
  Requires `--vault /path/to/Vault`. Use this unless the user has
  specifically set up and asked for remote access.
- **`--backend rest`** — writes over the Obsidian **Local REST API**
  community plugin's HTTP API, so the script can run on a different
  machine than the vault (e.g. this session, running remotely). **Treat
  this as administrative access to a knowledge base, not a convenience
  feature.** Only use it once the user has explicitly decided to accept
  that exposure — don't default to it, and don't suggest a bare public
  tunnel as a first option. If they want it:
  - No public tunnel by default. Bind the plugin to `127.0.0.1` only.
  - If remote access is genuinely needed, put an identity-aware proxy
    (e.g. Cloudflare Access) in front of the tunnel rather than exposing
    the bare URL — a leaked tunnel URL + API key alone should not be
    enough to reach it.
  - Start read-only if the plugin/setup supports scoping that; this
    script's own `capture`/`promote` split already limits *what* gets
    written even with write access, but the transport-level exposure is a
    separate decision from the workflow-level approval gate.
  - Needs `OBSIDIAN_REST_URL` and `OBSIDIAN_REST_API_KEY` (env vars — don't
    pass the key as a bare CLI flag if avoidable, it lands in shell
    history). Verify before anything else:
    ```bash
    OBSIDIAN_REST_URL="https://<url>" OBSIDIAN_REST_API_KEY="<key>" \
      python3 scripts/vault_sync.py check --backend rest
    ```
    A 401 means a bad key; "could not reach" means the tunnel isn't up —
    report the specific failure, don't retry blindly, they'll need to fix
    it on their end. (The plugin's exact response shapes are assumed from
    its long-stable core endpoints — GET/PUT/DELETE on `/vault/{path}`,
    GET on `/vault/{dir}/` for a listing — `check` is what actually proves
    it against their real install.)
  - Never log or echo `OBSIDIAN_REST_API_KEY` anywhere outside the command
    that needs it.

## Step 1 — find which Vocci sessions to handle

```bash
python3 scripts/vault_sync.py status --vault "/path/to/Vault"          # everything: captured/promoted/rejected
python3 scripts/vault_sync.py list-pending --vault "/path/to/Vault"    # only what's still awaiting your review
```

Then list candidate sessions from Vocci:
- "Sync my recent meetings" → `mcp__Vocci__sessions_list` (`sessionType: "recording"`).
- "Sync meetings about X" → `mcp__Vocci__search` (`sourceTypes: ["session"]`), fast depth first, retry `standard`/`deep` only if it doesn't find it.
- A single named meeting → resolve its session id via `sessions_list` or `search`, then go straight to Step 2.

Skip any session id already present in `status` (any status: captured,
promoted, or rejected) unless the user explicitly asks to redo it (then use
`--force` in the relevant step below).

## Step 2 — read the transcript, extract with judgment

For each session:

```
mcp__Vocci__session_get(sessionId, includeEvents: true, projection: "transcript_text")
```

Page with `eventsCursor` until `eventsNextCursor` is absent if the
transcript is long. Check the `truncated` flags — don't extract from a
transcript you know is incomplete without telling the user.

From the transcript + the session's own summary/notes, extract:

- **Title** — short, descriptive; don't just reuse a raw timestamp title.
- **Date** — `YYYY-MM-DD`, from session metadata.
- **Attendees** — only if the conversation makes identity clear. Vocci's speaker labels ("Speaker 1") are **not** stable identities — don't turn a label into a name unless the transcript itself names that speaker.
- **Project** — match against existing `Projects/*.md` (or names visible in `status`) by name/topic; if nothing matches, propose a clear new one. Every session must get a project — use `"Unsorted"` rather than skipping categorization if genuinely ambiguous.
- **Classification** — always one of `public` / `internal` / `confidential` / `restricted`. This is a *proposal*, checked by the human during the Inbox review, not a final judgment — but propose it honestly:
  - `confidential`: pricing, negotiation detail, client-identifying specifics, anything the org wouldn't want outside the deal team.
  - `restricted`: legal, HR, health, or anything with a "don't even keep this longer than necessary" quality — pair with a `delete-after-Nd` retention suggestion.
  - `internal`: normal work content, not sensitive outside the org.
  - `public`: fine either way.
  - For the private vault, use your judgment the same way — health/family/finances lean `restricted` or `confidential`, general notes/ideas lean `internal`/`public`.
- **Retention** — `"review"` (default), `"keep"`, or `"delete-after-Nd"` (e.g. `delete-after-90d`) if the raw content is time-sensitive and shouldn't be kept indefinitely once summarized.
- **Action items** — concrete, owned follow-up work, not general discussion. For each: `description` (imperative, one line), `owner` (or `"UNKNOWN"` — same rule as attendees, don't infer identity from a bare speaker label), `due` (resolve relative dates like "by Friday" to `YYYY-MM-DD`, or `""` if none stated — don't invent one), `priority` (`high`/`medium`/`low` — `high` for a near-term deadline or urgency language, `low` for no deadline + explicitly optional framing, `medium` otherwise), `context` (a short verbatim quote backing it up).
- **Summary** — a few sentences for the note body.

## Step 3 — capture (draft, not durable yet)

```json
{
  "vocci_session_id": "sess_123",
  "date": "2026-09-19",
  "title": "Q3 Roadmap Sync",
  "project": "Website Relaunch",
  "attendees": ["Alice"],
  "summary": "Discussed the homepage redesign timeline and CMS migration blockers.",
  "classification": "internal",
  "retention": "review",
  "tasks": [
    {"description": "Send revised wireframes to design team", "owner": "Alice", "due": "2026-09-22", "priority": "high", "context": "We need those wireframes by Tuesday or we slip the sprint."}
  ]
}
```

`tasks` may be `[]` — every session still gets a draft, even with zero
action items, so the eventual Meeting Log stays complete once promoted.

```bash
python3 scripts/vault_sync.py capture --vault "/path/to/Vault" --scope work --payload /tmp/entry.json
```

Validates required fields, `classification`, `retention`, and each task's
`priority`/`status`, and refuses if `--scope` doesn't match the vault's
locked scope. Writes **one** draft note to `00_Inbox/Vocci/` and caches the
payload — it does **not** touch `Meetings/`/`Tasks/`/`Projects/`/`Views/`,
and it never fires a webhook. Already-captured session ids are skipped by
default (reported as `"skipped": true`, with the existing status); pass
`--force` to re-capture (e.g. correcting a mis-extraction) — safe, it only
touches the draft, never anything already promoted.

**Now show the user what was captured** — project, classification,
priority breakdown of the tasks — before doing anything else with it.

## Step 4 — promote (only after review) or reject

Once the user has confirmed the draft looks right:

```bash
python3 scripts/vault_sync.py promote --vault "/path/to/Vault" --session-id sess_123
python3 scripts/vault_sync.py promote --vault "/path/to/Vault" --all             # everything still pending
```

Turns the draft into real `Meeting`/`Task`/`Project` notes (`ai_access:
approved`), marks the Inbox draft `status: processed` with a link to what
it became, rebuilds `Views/*.md`, and — only now — fires the webhook if one
is configured (see below). `--force` re-promotes an already-promoted
session (deletes and rewrites its meeting + task notes; never touches the
project note, so user edits to project descriptions survive).

If instead the draft shouldn't become anything durable:

```bash
python3 scripts/vault_sync.py reject --vault "/path/to/Vault" --session-id sess_123 --reason "duplicate of sess_100"
```

Marks it rejected in place (audit trail kept, nothing deleted). A rejected
session can still be promoted later with `--force` if the user changes
their mind.

## Step 5 — report back

Per session: what got promoted (meeting/task paths, priorities, which
project — flag `"Unsorted"` clearly) or rejected, and point the user at
`Views/Upcoming Tasks.md` / `Views/Project Status.md` for the roll-up.
Anything still sitting in `list-pending` is unfinished business — mention
it rather than letting it go stale silently.

## Downstream automation (n8n, Zapier, Make, ...)

`promote` can POST a JSON summary to an automation tool's webhook right
after each promotion — never on capture, since nothing should leave the
system until a human approved it. It's push-based: the automation tool
gets exact structured, *already-approved* data the moment it exists,
instead of polling and re-parsing Markdown (or worse, seeing unreviewed
drafts).

```bash
python3 scripts/vault_sync.py promote --vault "/path/to/Vault" --session-id sess_123 \
  --webhook-url "https://<n8n-host>/webhook/vocci-sync"
# or: export N8N_WEBHOOK_URL="https://<n8n-host>/webhook/vocci-sync"
```

Optional — omit it and nothing changes. Delivery is best-effort: a dead or
misconfigured webhook is reported in the result
(`"webhook": {"delivered": false, "error": ...}`) but never fails the
promotion, since the notes are already safely written by the time it
fires. Relay `webhook.delivered: false` to the user rather than silently
swallowing it.

Payload POSTed (`Content-Type: application/json`):

```json
{
  "event": "vocci_meeting_promoted",
  "promoted_at": "2026-09-19T23:45:51Z",
  "vocci_session_id": "sess_123",
  "scope": "work",
  "classification": "internal",
  "meeting": {"path": "Meetings/....md", "title": "...", "date": "2026-09-19", "attendees": ["Alice"], "summary": "..."},
  "project": {"name": "Website Relaunch", "path": "Projects/Website Relaunch.md", "is_new": true},
  "tasks": [
    {"path": "Tasks/....md", "title": "...", "description": "...", "priority": "high", "status": "open", "due": "2026-09-22", "owner": "Alice", "context": "..."}
  ]
}
```

Build the n8n (or Zapier/Make) workflow against this contract: a
**Webhook** trigger receiving this JSON, then branch on `tasks[].priority`
(e.g. `high` → Todoist/Asana/Linear task with `due`), on `meeting` (→ post
`summary` to Slack/Discord/email), and mirror each task into a
Notion/Airtable/Sheets row keyed by `project.name`. If `classification` is
`confidential`/`restricted`, treat that as a signal in the workflow too —
e.g. skip the Slack post, route only to internal tools. Don't hand-build
the n8n workflow JSON blind — node parameter schemas drift across n8n
versions and a wrong one fails to import silently; once the user has a
real n8n instance and has picked specific apps, build/import against that
instance instead of guessing its JSON export format here.

## Optional: live queries instead of static views

`Views/*.md` are plain markdown tables refreshed on every promote — zero
plugins required, always in sync as of the last run. If the user has the
**Dataview** community plugin (`.obsidian/plugins/dataview/`) and wants
live queries instead, a minimal starting block:

````markdown
```dataview
TABLE due AS "Due", priority AS "Priority", project AS "Project", owner AS "Owner"
FROM "Tasks"
WHERE status != "done"
SORT due ASC
```
````

Obsidian's built-in **Bases** core plugin can do the same as a `.base`
file — if the user wants that, check Obsidian's current Bases
documentation for exact syntax first (its schema has been evolving; a
wrong one silently produces an empty view). Either way, keep this skill's
frontmatter schema (plain scalars: `type`, `status`, `priority`, `project`,
`due`, `owner`, `scope`, `classification`, `ai_access`,
`source_meeting`, `vocci_session_id`) as the contract underneath.

## Notes / failure modes

- **Vault path wrong or not a vault** (`fs`) → fails fast with a clear error; don't guess a path, ask the user.
- **Tunnel down or bad API key** (`rest`) → `check` fails fast with which one it is; report it, don't retry blindly.
- **Scope mismatch** (`--scope` doesn't match a vault's locked scope) → refused before anything is written; this usually means the wrong `--vault`/`--api-url` was passed — double check with the user which vault is which rather than forcing past it.
- **Duplicate capture/promote** → handled by `.vocci-sync/state.json`; don't dedupe by scanning filenames yourself.
- **Long transcripts** → page with `eventsCursor`; don't extract from a page you haven't confirmed is complete.
- **Ambiguous owner/attendee** → `"UNKNOWN"` / empty list rather than guessing from a speaker label; mention it in your report if it affects several tasks.
- **No action items in a meeting** → still capture it (`tasks: []`) so the eventual Meeting Log stays complete once promoted.
- **Never promote without the user having seen the draft first** — that review is the actual safety mechanism; the Inbox staging is the backstop if it's ever skipped, not a substitute for it.
- **Never log or echo `OBSIDIAN_REST_API_KEY`** anywhere outside the command that needs it.
