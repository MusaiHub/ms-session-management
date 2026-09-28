# ms-session-management

Hermes Agent skill for listing, inspecting, exporting, hiding, archiving, and deleting Hermes sessions across profiles and platforms.

## Install

```bash
# Clone into your Hermes skills directory
git clone https://github.com/mushfiqsaikat/ms-session-management.git \
  "%LOCALAPPDATA%\hermes\skills\ms-session-management"
```

Or copy the `ms-session-management/` folder directly to:
```
%LOCALAPPDATA%\hermes\skills\ms-session-management\
```

## Usage

Load the skill in Hermes:
```
skill_view(name="ms-session-management")
```

Then follow the workflow in `SKILL.md` for:
- Machine-wide session inventory (fast listing via `hermes sessions list` + `stats`)
- Mandatory table output format
- Changing visibility (hidden/archived) via `SessionDB.set_session_hidden()`
- Full session export for debugging
- Safe deletion

## Commands Reference

The skill wraps these Hermes CLI subcommands:
- `hermes sessions list` — list recent sessions
- `hermes sessions stats` — show store statistics
- `hermes sessions export` — export session content (JSONL/Markdown/QMD)
- `hermes sessions delete` — delete a session
- `hermes sessions archive` / `prune` — bulk operations
- `hermes sessions pin` / `unpin` / `pinned` — durable keep flags
- `hermes sessions repair` / `recover` — database repair

See `hermes sessions --help` for all 19 subcommands.

## License

MIT — see [LICENSE](LICENSE).