---
name: vocci-obsidian-meeting-sync
description: Turn Vocci ring-captured meetings/conversations into a linked Obsidian vault of atomic Meeting, Task, and Project notes, with auto-refreshed Upcoming Tasks / Project Status / Meeting Log views. Use when the user wants Vocci meetings synced to Obsidian, an Obsidian "second brain" fed from Vocci recordings, action items pulled out of meetings and tracked in Obsidian, or asks to sync/import/export Vocci sessions into a notes vault — including from a remote/cloud Claude session with no local filesystem access to the vault (via the Obsidian Local REST API backend). Also covers chaining downstream automation off each sync (n8n/Zapier/Make webhooks — e.g. push high-priority tasks to Todoist, post meeting digests to Slack, mirror tasks into Notion/Airtable). Requires no Obsidian plugins for the local path (Dataview/Bases optional upgrades) — every conversation gets a project and a priority, deterministically.
---

# Vocci → Obsidian meeting sync

Pulls meeting/conversation data from Vocci MCP, has the agent (you) extract
action items, deadlines, and project context with judgment, then hands a
structured payload to `scripts/vault_sync.py`, which deterministically writes
the vault files, dedupes against already-synced sessions, and regenerates the
three view notes. Fetching + extraction is your job (judgment); writing files
is the script's job (must stay consistent, must not corrupt existing vault
content).

## Vault layout this produces

```
<vault>/
  Meetings/2026-09-19 - Q3 Roadmap Sync.md
  Tasks/2026-09-19 - Send revised wireframes to design team.md
  Projects/Website Relaunch.md
  Views/Upcoming Tasks.md      # regenerated every sync
  Views/Project Status.md      # regenerated every sync
  Views/Meeting Log.md         # regenerated every sync
  .vocci-sync/state.json       # session_id -> files, for dedup
```

Every Task and Meeting note carries plain-scalar YAML frontmatter (`project`,
`priority`, `status`, `due`, `vocci_session_id`, ...) and the note *body*
carries the actual `[[wikilinks]]` (Obsidian frontmatter doesn't need to hold
links — keeping frontmatter to plain scalars is what makes the views script
able to parse it back out reliably). The three `Views/*.md` files are plain
markdown tables computed by the script from that frontmatter, so they render
correctly in a stock Obsidian install — no community plugin required. See
"Optional: live queries instead" below if the user has Dataview or Bases and
wants live queries instead of the regenerated snapshot.

## Step -1 — pick a backend

`scripts/vault_sync.py` has two backends behind the same commands and
payload shape. Pick based on whether *this* Claude session can see the
vault on disk:

- **`--backend fs`** (default) — direct filesystem writes. Use when Claude
  is running on the same machine as the vault (a local Claude Code
  session). Needs only `--vault /path/to/Vault`.
- **`--backend rest`** — writes over the Obsidian **Local REST API**
  community plugin's HTTP API instead. Use whenever this session is remote
  (a cloud/web Claude session, or any machine other than the one holding
  the vault) — the whole point is it doesn't need local disk access.
  Needs `OBSIDIAN_REST_URL` and `OBSIDIAN_REST_API_KEY` (env vars — don't
  pass the key as a bare CLI flag if you can avoid it, it'll land in shell
  history). If neither is set and the user hasn't said "local", ask which
  they want rather than assuming.

**One-time setup for `rest`, done by the user on their own machine** (you
can't do this part remotely — walk them through it, don't attempt it
yourself):
1. Install and enable the **Local REST API** community plugin in Obsidian; copy the API key it generates.
2. Expose it off the local machine with a tunnel that terminates real TLS — e.g. `cloudflared tunnel --url http://127.0.0.1:27123` or `ngrok http 27123` — pointed at the plugin's **plain HTTP** port (27123), not its self-signed HTTPS port (27124). Point the tunnel at the plugin's port, not at any other local service.
3. They give you the resulting `https://...` tunnel URL and the API key.

Before anything else, verify the connection:

```bash
OBSIDIAN_REST_URL="https://<tunnel-url>" OBSIDIAN_REST_API_KEY="<key>" \
  python3 scripts/vault_sync.py check --backend rest
```

`{"ok": false, ...}` with a 401 means a bad key; a "could not reach" error
means the tunnel isn't up — report the specific failure back to the user
rather than retrying blindly, they'll need to fix it on their end.

(The plugin's exact response shapes are assumed from its long-stable core
endpoints — GET/PUT/DELETE on `/vault/{path}`, GET on `/vault/{dir}/` for a
listing — but `check` is what actually proves it against their install
before you write anything.)

## Step 0 — run setup once

```bash
python3 scripts/vault_sync.py setup --vault "/path/to/Vault"          # fs
python3 scripts/vault_sync.py setup --backend rest                    # rest (env vars set)
```

Idempotent — safe to run again later. For `fs`, creates `Meetings/`,
`Tasks/`, `Projects/`, `Views/`, and the sync-state file if any are
missing. For `rest`, folders appear automatically as notes get written
into them, so this mainly re-verifies the connection and seeds the
sync-state file. Never touches existing notes either way.

## Step 1 — find which Vocci sessions to sync

Check what's already synced so you don't reprocess it:

```bash
python3 scripts/vault_sync.py status --vault "/path/to/Vault"    # or --backend rest
```

Then list candidate sessions:
- "Sync my recent meetings" → `mcp__Vocci__sessions_list` (`sessionType: "recording"`, optionally `lastSyncTime` set to just after the newest `synced_at` you saw in `status`, to skip old ones server-side).
- "Sync meetings about X" / a specific topic → `mcp__Vocci__search` (`sourceTypes: ["session"]`), fast depth first, retry `standard`/`deep` only if it doesn't find it.
- A single named meeting → resolve its session id via `sessions_list` or `search`, then go straight to Step 2.

Skip any session id already present in the `status` output unless the user
explicitly asks to re-sync (then use `--force` in Step 3).

## Step 2 — read the full transcript, extract with judgment

For each session to sync:

```
mcp__Vocci__session_get(sessionId, includeEvents: true, projection: "transcript_text")
```

Page with `eventsCursor` until `eventsNextCursor` is absent if the transcript
is long. Check the `truncated` flags — don't extract action items from a
transcript you know is incomplete without telling the user.

From the transcript + the session's own summary/notes, extract:

- **Title** — short, descriptive; don't just reuse the raw session title if it's a timestamp.
- **Date** — the session's date (from `session_get`/`sessions_list` metadata), `YYYY-MM-DD`.
- **Attendees** — only if the conversation makes identity clear. Vocci's speaker labels ("Speaker 1") are **not** stable identities — don't turn a label into a name unless the transcript itself names that speaker. Leave the list empty rather than guessing.
- **Project** — match against existing `Projects/*.md` in the vault (list the folder, or reuse the names already visible in the `status`/prior sync output) by name/topic. If nothing matches, pick a clear new project name — a new project note is created automatically. **Every meeting and every task must get a project; never leave it blank.** If the conversation genuinely doesn't belong to any project, use `"Unsorted"` as the project so it's still tracked and easy for the user to reclassify later, rather than skipping categorization.
- **Action items** — concrete, owned pieces of follow-up work, not general discussion topics. For each one:
  - `description` — imperative, one line.
  - `owner` — the person who owns it, if the transcript makes it clear; otherwise `"UNKNOWN"`. Same rule as attendees: don't infer identity from a bare speaker label.
  - `due` — an explicit date if stated; resolve relative dates ("by Friday", "next week") to an absolute `YYYY-MM-DD` using the session's date as the anchor. Leave `""` if no deadline was stated — don't invent one.
  - `priority` — always one of `high` / `medium` / `low`, using this rule of thumb:
    - `high`: due within ~3 days of the session date, or the speakers used urgency language ("blocker", "ASAP", "critical", "before the launch").
    - `low`: no stated deadline and explicitly optional/exploratory framing ("nice to have", "someday", "if we get to it").
    - `medium`: everything else (the default when unsure).
  - `context` — a short verbatim quote or paraphrase from the transcript backing the item up, so the user can verify it later.
- **Summary** — a few sentences of what the meeting covered, for the meeting note body (reuse the Vocci session summary if it's good; tighten it if not).

## Step 3 — write the payload, run the sync

One JSON payload per session:

```json
{
  "vocci_session_id": "sess_123",
  "date": "2026-09-19",
  "title": "Q3 Roadmap Sync",
  "project": "Website Relaunch",
  "attendees": ["Alice"],
  "summary": "Discussed the homepage redesign timeline and CMS migration blockers.",
  "tasks": [
    {
      "description": "Send revised wireframes to design team",
      "owner": "Alice",
      "due": "2026-09-22",
      "priority": "high",
      "context": "We need those wireframes by Tuesday or we slip the sprint."
    }
  ]
}
```

`tasks` may be an empty list — every session still gets a Meeting note even
with zero action items (that's what makes the Meeting Log complete, per
"every captured conversation entry is categorized").

```bash
python3 scripts/vault_sync.py sync-entry --vault "/path/to/Vault" --payload /tmp/entry.json   # fs
python3 scripts/vault_sync.py sync-entry --backend rest --payload /tmp/entry.json             # rest (env vars set)
```

The script validates `priority`/`status`/required fields itself and exits
non-zero with a clear message if something's missing — fix the payload and
re-run rather than working around it. Already-synced session ids are skipped
by default (reported as `"skipped": true"`); pass `--force` only when the
user explicitly wants a re-sync (e.g. they corrected something in Vocci) —
`--force` deletes and rewrites that session's meeting + task notes, but never
touches the project note (so user edits to project descriptions survive).

The command also regenerates `Views/*.md` automatically — no separate step
needed. Run `rebuild-views` by hand only if the user manually edited
frontmatter in Tasks/Projects/Meetings outside this workflow.

## Step 4 — report back

Tell the user, per session: the meeting note path, how many tasks were
created and at what priorities, and which project it landed under (flag it
clearly if you used `"Unsorted"` so they know to reclassify). Point them at
`Views/Upcoming Tasks.md` and `Views/Project Status.md` for the roll-up.

## Downstream automation (n8n, Zapier, Make, ...)

`sync-entry` can POST a JSON summary to an automation tool's webhook right
after a successful sync, so the vault isn't a dead end — the same sync that
writes the notes can also kick off "create a Todoist task," "post a Slack
digest," "mirror this row into Notion," etc. It's push-based (fires once,
with structured data, the moment real action items exist) rather than
making the automation tool poll and re-parse Markdown.

```bash
python3 scripts/vault_sync.py sync-entry --vault "/path/to/Vault" \
  --payload /tmp/entry.json --webhook-url "https://<n8n-host>/webhook/vocci-sync"
# or: export N8N_WEBHOOK_URL="https://<n8n-host>/webhook/vocci-sync"
```

Optional — omit it and nothing changes. Delivery is best-effort: a dead or
misconfigured webhook is reported in the result (`"webhook": {"delivered":
false, "error": ...}`) but never fails the sync or blocks the vault write,
since the notes are already safely written by the time it fires. Relay that
`webhook.delivered: false` to the user rather than silently swallowing it —
their vault is fine, but the downstream automation didn't run. It never
fires on a skipped (already-synced) session.

Payload POSTed (`Content-Type: application/json`):

```json
{
  "event": "vocci_meeting_synced",
  "synced_at": "2026-09-19T23:45:51Z",
  "vocci_session_id": "sess_123",
  "meeting": {"path": "Meetings/....md", "title": "...", "date": "2026-09-19", "attendees": ["Alice"], "summary": "..."},
  "project": {"name": "Website Relaunch", "path": "Projects/Website Relaunch.md", "is_new": true},
  "tasks": [
    {"path": "Tasks/....md", "title": "...", "description": "...", "priority": "high", "status": "open", "due": "2026-09-22", "owner": "Alice", "context": "..."}
  ]
}
```

That's the whole contract — build the n8n (or Zapier/Make) workflow against
it: a **Webhook** trigger node receiving this JSON, then branch on
`tasks[].priority` (e.g. `high` → create a Todoist/Asana/Linear task with
`due`) and on the `event`/`meeting` fields (→ post `meeting.summary` to
Slack/Discord/email; mirror each task into a Notion/Airtable/Sheets row
keyed by `project.name`). Don't hand-build the n8n workflow JSON blind —
node parameter schemas drift across n8n versions and a wrong one fails to
import silently; once the user actually has an n8n instance and has picked
which apps (Todoist vs. Asana, Slack vs. email, Notion vs. Airtable, ...),
build/import the workflow against that real instance instead of guessing
its JSON export format here.

If the automation side needs to *read* the vault too (not just react to
this webhook) — e.g. a nightly digest, or reacting to notes edited by hand
in Obsidian rather than through this skill — point it at the same `rest`
backend's Local REST API instead of re-implementing vault access: same
tunnel, same API key, it can `GET /vault/Tasks/` on a schedule the same way
this script's `RestBackend` does.

## Optional: live queries instead of static views

The generated `Views/*.md` are plain markdown tables refreshed on every
sync — they work with zero plugins and are always in sync with the vault at
the time of the last run. If the user has the **Dataview** community plugin
installed (check for `.obsidian/plugins/dataview/`) and wants live,
always-current queries instead, a minimal starting block for upcoming tasks:

````markdown
```dataview
TABLE due AS "Due", priority AS "Priority", project AS "Project", owner AS "Owner"
FROM "Tasks"
WHERE status != "done"
SORT due ASC
```
````

Obsidian's newer built-in **Bases** core plugin can do the same as a
`.base` file with filterable/sortable table and board views — if the user
wants that instead, check Obsidian's current Bases documentation for exact
syntax before writing one (its YAML schema has been evolving and getting it
wrong silently produces an empty view). Either way, keep the frontmatter
schema this skill writes (plain scalars: `type`, `status`, `priority`,
`project`, `due`, `owner`, `source_meeting`, `vocci_session_id`) as the
contract — swapping the view layer doesn't require changing how notes are
written.

## Notes / failure modes

- **Vault path wrong or not a vault** (`fs`) → `setup`/`sync-entry` fail fast with a clear error; don't guess a path, ask the user.
- **Tunnel down or bad API key** (`rest`) → `check` fails fast with which one it is; report it, don't retry blindly — it needs the user to fix something on their machine.
- **Duplicate sync** → handled by `.vocci-sync/state.json` (read/written through whichever backend you picked); don't try to dedupe by scanning filenames yourself.
- **Long transcripts** → page with `eventsCursor`; don't extract from a page you haven't confirmed is complete (`truncated: false` and no `eventsNextCursor`).
- **Ambiguous owner/attendee** → use `"UNKNOWN"` / leave attendees empty rather than guessing from a speaker label; say so in your report if it affects several tasks.
- **No action items in a meeting** → still sync it (`tasks: []`) so the Meeting Log stays complete.
- **Never log or echo `OBSIDIAN_REST_API_KEY`** in output, commit messages, or anywhere outside the command that needs it.
