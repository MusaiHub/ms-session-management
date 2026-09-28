---
name: ms-session-management
description: "Use when listing, inspecting, exporting, hiding, archiving, or deleting Hermes sessions across profiles and platforms. For simple listing within the active profile, prefer the `session_search` tool or `hermes sessions list`."
version: 1.2.0
author: Mushfiq Saikat
license: MIT
platforms: [windows, linux, macos]
metadata:
  hermes:
    tags: [hermes, sessions, profiles, hidden, archived, desktop]
---

# Hermes Session Management

## When to Use

Use this workflow to count sessions machine-wide, explain why Desktop shows only a subset, distinguish profile isolation from hidden or archived state, or safely change a stored session's visibility.

Audit and repair stored Hermes sessions across isolated profiles, especially when Desktop shows fewer sessions than `hermes sessions stats` reports.

## Procedure

### 1. Machine-wide inventory (use Hermes CLI, not raw SQL)

**Fast listing (metadata only — no exports):**
```bash
hermes profile list
hermes sessions stats
hermes --profile <name> sessions stats  # repeat for each profile
hermes sessions list --limit 100
hermes --profile <name> sessions list --limit 100
```

**Full inspection (when you need message payloads, models, tool calls):**
```bash
hermes sessions export --session-id <ID> --format jsonl -
```

The CLI commands above are authoritative. They already:
- Enumerate all profiles and their databases
- Exclude backup/snapshot/cache directories
- Report hidden/archived flags correctly
- Sort by last activity

**Do not write custom SQLite scripts** — the CLI is the single source of truth.

**Use fast listing for the mandatory table output.** Export only when debugging message content, verifying model config, or auditing tool calls.

### 2. Mandatory table output

Always render session listings as a table with columns:
`Profile | Title | Source | Model | Messages | Hidden | Archived | Last Activity`

Never use bullet points, prose, or any other format. Convert Unix timestamps to local timezone.

### 3. Change visibility (hidden/archived) via supported API only

```python
from hermes_state import SessionDB

db = SessionDB()
try:
    changed = db.set_session_hidden("SESSION_ID", True)   # True hides, False unhides
    print(changed)
finally:
    db.close()
```

Bind `HERMES_HOME` to the intended profile before constructing `SessionDB`. Refresh Desktop after the write.

Gateway equivalents: `session.set_hidden` and `PATCH /api/sessions/<id>` with explicit `hidden` boolean.

### 4. Verify after mutation

Re-read the target row via `hermes sessions list` or `hermes sessions export` and confirm the `hidden` value. Do not claim success from the setter response alone.

### 5. Export & inspect session content

```bash
hermes sessions export --session-id <ID> --format jsonl -
```

The output is a JSON object with a `messages` array. Use `jq` to filter:
- Last 4 user/assistant messages: `.messages | map(select(.role == "user" or .role == "assistant")) | .[-4:] | map({id, role, content, finish_reason})`
- Message count: `.messages | length`

For large sessions, the CLI is still the correct path — do not query `state.db` directly.

### 6. Delete sessions

```bash
hermes sessions delete --yes "<SESSION_ID>"
```

One ID per invocation. Deletes local Hermes record only; does not delete Telegram chat history. Verify with `hermes sessions list --source telegram` afterward.

## Pitfalls

- Never edit `hidden` or `archived` with raw SQL — breaks compression lineage.
- Expect Bot Mode sessions to be re-hidden during reconciliation; unhide user sessions only.
- Pinning clears `hidden` but canonical Bot Chats are exempt.
- Separate three numbers in reports: profile count, stored session count, running process count.
- `hermes sessions list` excludes hidden rows by default; use `--all` if available or rely on stats for total count.
