---
name: imsg
description: Use for local iMessage/SMS archive reads, iMessage contact lookup, visible Messages.app contact lookup, chat history, watch, and explicitly requested sends.
---

# imsg

Use this for Messages.app history, chat lookup, streaming, visible UI contact lookup, and sends. Reading is local DB access; sending uses Messages automation and must be explicitly requested.

## cmux Default Wrapper

In cmux/Codex/Claude terminals, do not probe with bare `imsg` first. The terminal process tree usually lacks Full Disk Access and bare `imsg` often fails with `authorization denied (code: 23)`.

Use the cmux iMessage helper by default for reads and explicit sends:

```bash
tools/cmux-imsg/build/cmux-imsg status
tools/cmux-imsg/build/cmux-imsg run -- <imsg subcommand> ...
```

If status returns `ok: true`, the helper can pass through upstream `imsg` subcommands, including `send`. If it cannot connect to `/tmp/cmux-imsg.sock`, launch `tools/cmux-imsg/build/cmux iMessage Helper.app` or `/Applications/cmux iMessage Helper.app`, then retry.

If the helper returns `authorization denied (code: 23)`, grant Full Disk Access to `cmux iMessage Helper.app`, restart the helper, and retry the same helper command. Do not fall back to bare `imsg` unless the user has confirmed the current shell has Full Disk Access and Messages Automation.

## Sources

- DB: `~/Library/Messages/chat.db`
- Repo: `~/Projects/imsg`
- CLI: `imsg`
- JSON output is NDJSON; pipe to `jq -s` for arrays.

## Read Workflow

Check DB access through the helper:

```bash
tools/cmux-imsg/build/cmux-imsg status
```

For a visible Messages.app person/name, start with chats. The UI-resolved name usually appears as `contact_name`; it may not appear in `imsg search`, raw `message.text`, or the `handle` table.

```bash
tools/cmux-imsg/build/cmux-imsg run -- chats --limit 200 --json \
  | jq -r '.commands[0].stdout' \
  | jq -s '.[] | select((.contact_name // .display_name // .name // .identifier // "" | ascii_downcase) | contains("beatrix"))'
```

Then read the chat by id:

```bash
tools/cmux-imsg/build/cmux-imsg run -- history --chat-id ID --json \
  | jq -r '.commands[0].stdout' | jq -s
```

Use `imsg search --query ... --json` for message-body search only; do not treat no search hits as proof that a visible UI contact does not exist. Use `--attachments` when attachment metadata matters. Use `--start`/`--end` with absolute timestamps for date-scoped questions.

Direct DB checks are only a fallback. The `handle` table is keyed by phone/email and often lacks the contact display name that `imsg chats` resolves.

## Sends

Only send, react, mark read, or show typing when the user explicitly asks. Prefer dry wording in the final confirmation: recipient, service, and what was sent.

Common send command:

```bash
tools/cmux-imsg/build/cmux-imsg run -- send --to "+15551234567" --text "message" --service auto
```

The helper returns a JSON envelope. Treat `commands[0].exitCode == 0` and `commands[0].stdout` containing `sent` as success. Use bare `imsg send` only when the user has confirmed the current shell has Full Disk Access and Messages Automation permission.

## Verification

For repo edits:

```bash
make test
make build
```

For live read proof:

```bash
tools/cmux-imsg/build/cmux-imsg run -- chats --limit 3 --json | jq -r '.commands[0].stdout' | jq -s
```
