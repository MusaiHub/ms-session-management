---
name: ms-session-management
description: "Use when listing, inspecting, exporting, hiding, archiving, or deleting Hermes sessions across profiles and platforms. For simple listing within the active profile, prefer the `session_search` tool or `hermes sessions list`."
version: 1.1.2
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

### 1. Inventory profiles before counting sessions

```bash
hermes profile list
```

Treat every profile as an independent session store. Never report the active profile's count as the machine total.

### 2. Count each profile separately

```bash
hermes sessions stats
hermes --profile <name> sessions stats
```

Sum the reported totals programmatically. Distinguish stored sessions from active `Hermes.exe` processes: application, backend, and gateway processes are not chat sessions.

Treat the total session count as useful orientation, but do not assume the CLI's per-source subtotals are exhaustive. Some versions omit sources such as `desktop` from the subtotal breakdown even when those rows are included in the total. For complete inventories, enumerate the live databases directly and use those rows as the authority.

### 3. Compare totals with visible listings

```bash
hermes sessions list
hermes --profile <name> sessions list
```

A smaller listing does not mean records are missing. The ordinary list excludes rows whose `hidden` flag is set. Profile isolation also prevents one profile's sessions from appearing in another profile's Desktop sidebar.

### 4. Inspect visibility state when counts disagree

Open each profile's `state.db` read-only with Python's `sqlite3` and select only the fields needed for diagnosis:

```sql
SELECT id, title, source, model, message_count, archived, hidden,
       started_at, last_activity_at
FROM sessions
ORDER BY COALESCE(last_activity_at, started_at) DESC;
```

Default store: `$HERMES_HOME/state.db`, normally the main Hermes home. Named profiles use `profiles/<name>/state.db`.

Report `hidden` and `archived` separately. Hidden removes a row from the normal list but retains its transcript and resumability. Archived is a separate lifecycle flag.

For a machine-wide "all sessions" request, include the default profile and every named profile returned by `hermes profile list`. Exclude backup, snapshot, archive, cache, and exported databases. Enumerate rows programmatically, then verify that the number of displayed rows equals the declared machine-wide total before answering.

Use this standard output schema when the user does not request another format:

- Profile
- Title
- Session ID
- Source
- Model
- Message count
- Hidden
- Archived
- Last activity

Convert Unix timestamps to the machine's local timezone for display. Preserve the original stored value internally if conversion fails, and label it rather than guessing.

### 5. Explain cause conservatively

Hermes intentionally hides canonical Bot chats, group-chat plumbing, agent-owned sessions, and sessions created or patched with `hidden: true`. The database records the current flag, not the actor or historical reason that set it. If provenance is absent, say the cause is likely rather than proven.

Do not infer that every hidden row was hidden by Bot Mode from its current title. Titles may change, and only recorded creation or mutation context can establish the trigger.

Report the database `source` exactly. Never infer a more specific source subtype from a title or ID prefix. In particular, a `bg_*` ID does not by itself prove that the row is a background task. If useful, describe the prefix separately as an observation, not as verified provenance.

### 6. Change visibility through Hermes APIs

Prefer the supported session setter over direct SQL because it updates the session's compression lineage consistently:

```python
from hermes_state import SessionDB

db = SessionDB()
try:
    changed = db.set_session_hidden("SESSION_ID", False)  # True hides
    print(changed)
finally:
    db.close()
```

Bind `HERMES_HOME` to the intended named profile before constructing `SessionDB`. Refresh the Desktop session list or restart Desktop after the write.

The gateway equivalents are `session.set_hidden` and `PATCH /api/sessions/<id>` with an explicit `hidden` boolean. Set fields explicitly rather than relying on endpoint defaults.

### 7. Verify after mutation

Re-read the exact target row and confirm its `hidden` value, then run the profile's normal session listing to verify the user-visible effect. Do not claim success from the setter response alone.

## Complete Inventory Pattern

Use a short Python `sqlite3` script for complete machine-wide listings. Build the profile roots from the default Hermes home plus the names returned by `hermes profile list`. The default profile's database may live at the top-level `state.db` rather than `profiles/default/state.db`; probe both locations when building per-profile roots, and do not assume every named profile subdirectory contains a `state.db`. Open each found `state.db` read-only, and select only the required columns. Sort each profile by `COALESCE(last_activity_at, started_at) DESC`.

The script must calculate and print:

1. Number of live profiles inspected.
2. Row count for each profile.
3. Machine-wide row count.
4. Number of rows actually rendered.

Fail or investigate when the machine-wide count and rendered count differ. Also flag, but do not silently "correct," any mismatch between database rows and CLI statistics.

## Pitfalls

- Enumerate `state.db` only from active profile homes, excluding snapshot and backup directories, because historical snapshots inflate the live machine total.
- Never edit `hidden` with raw SQL when the supported setter is available, because raw SQL can leave compression-lineage rows inconsistent.
- Expect intentional Bot Mode sessions to be hidden again during reconciliation; unhide ordinary user sessions, but do not fight canonical plumbing visibility without changing the owning feature's policy.
- Pinning normally clears `hidden`, but canonical Bot Chats are intentionally exempt; do not present pinning as a universal unhide control.
- Separate three numbers in reports: profile count, stored session count, and running process count. Combining them creates a misleading machine total.

## Inspecting Session Content

### Find and export a session

```bash
hermes sessions list --limit 20
hermes sessions export --session-id <ID> --format jsonl -
```

Current Hermes versions may return one JSON object containing a `messages` array rather than one message per physical line. Inspect the top-level structure before filtering. Never use `tail` to identify the latest messages because one line may contain the complete transcript.

For the common object shape:

```bash
hermes sessions export --session-id <ID> --format jsonl - \
  | jq '{id, message_count: (.messages | length), timings}'

hermes sessions export --session-id <ID> --format jsonl - \
  | jq '.messages | map(select(.role == "user" or .role == "assistant")) | .[-4:] | map({id, role, content, finish_reason})'
```

When the user asks for the message before another message, report the immediately preceding chronological row. If useful, identify the previous assistant response separately instead of silently skipping an intervening user message.

For large sessions, query the appropriate profile's `state.db` read-only and select only the required columns, such as `id`, `role`, `content`, `finish_reason`, and `tool_calls`. The SQLite database is authoritative; raw JSON snapshots may not exist.

Interpret the final assistant message's `finish_reason` carefully:

- `stop`: normal completion.
- `tool_calls`: interruption during tool dispatch is possible.
- `length`: context limit reached.

Resume an interrupted task with `hermes --resume <ID>`. Inspect without resuming when only historical context or evidence is needed.

## Telegram-Originated Sessions

### List and verify

```bash
hermes sessions list --source telegram
```

If source filtering is unavailable or ambiguous, inspect session metadata and require `source: telegram` before classifying or deleting a session.

### Delete local Telegram session records

The CLI accepts one session ID per invocation:

```bash
for id in <ID1> <ID2> <ID3>; do
  hermes sessions delete --yes "$id"
done
```

Never pass multiple IDs to one `hermes sessions delete` command. Deleting a Hermes session removes the local Hermes record only; it does not delete Telegram chat history. A deleted active Telegram session can be recreated as a new session by the next incoming message.

After deletion, verify with:

```bash
hermes sessions list --source telegram
```

Do not claim that Telegram-side messages were deleted unless a separate Telegram API operation was explicitly performed and verified.
