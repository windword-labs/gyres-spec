# List conversations

The conversation list is one JSON array from `GET /v1/sessions`. There is no page size, offset, or cursor: replace the list you show with `sessions` on every successful read. Keep it current from the Core-wide event stream, then open one row with `GET /v1/sessions/{id}` (or the text transcript) and create a row with `POST /v1/sessions`.

`{id}` is a session id such as `2026-10-14T12-04-05_3f2a1b0c-4d5e-6789-abcd-ef0123456789`, or that id, `~`, and a lane name for a side lane. The list contains only main conversations. Bot conversations, and a child whose parent still exists, are omitted. Do not parse ids.

## 1. Load the live list

The only query parameter is `archived`. Omitted, or any value other than the exact string `true`, returns the live list. Rows are newest `modified` first. Pinned rows stay in that order; `pinned_at` is only a marker.

```bash
curl -sS -H "Authorization: Bearer $GYRES_KEY" \
  http://127.0.0.1:15100/v1/sessions
```

`200`:

```json
{
  "sessions": [
    {
      "path": "/data/sessions/2026-10-14T12-04-05_3f2a1b0c-4d5e-6789-abcd-ef0123456789.jsonl",
      "session_id": "2026-10-14T12-04-05_3f2a1b0c-4d5e-6789-abcd-ef0123456789",
      "created": "2026-10-14T12:04:05.000Z",
      "modified": "2026-10-14T12:10:00.000Z",
      "message_count": 4,
      "first_message": "Sketch the layout",
      "title": "Layout",
      "pinned_at": "2026-10-14T12:06:00.000Z",
      "running": false,
      "run_state": "idle",
      "unseen": ""
    }
  ]
}
```

Address the row by `session_id`. `title` is omitted when no custom title is stored; show `first_message` then. `pinned_at` is omitted when the row is not pinned. `running` is true while that lane has a live run, including a paused one. `run_state` is `idle`, `running`, `pausing`, `paused`, `waiting_children`, `interrupted`, or `blocked`. A row can have `running: true` and `run_state: "paused"`. `unseen` is `""`, `result`, or `failed`. `workspace`, `scheduled`, and `origin` appear only when set. An empty list is `"sessions": []`.

`archived=true` is the archived list, newest `archived_at` first. Those rows have `archived_at` and `bytes`, and omit `path`, `running`, `run_state`, `unseen`, `pinned_at`, `workspace`, and `origin`.

```bash
curl -sS -H "Authorization: Bearer $GYRES_KEY" \
  "http://127.0.0.1:15100/v1/sessions?archived=true"
```

## 2. Keep the list fresh

Open `GET /v1/events` and leave it open. The first frame is `ready`. Each later notification is one `data:` line of JSON and a blank line. There is no `id:` line and no resume cursor. A dropped connection starts again at `ready`. Ignore lines that start with `:` (a `: heartbeat` arrives about every 20 seconds). A `type` you do not know is safe to ignore.

```bash
curl -N -sS -H "Authorization: Bearer $GYRES_KEY" \
  http://127.0.0.1:15100/v1/events
```

```
data: {"type":"ready","version":"1.4.0","instance_id":"0123456789abcdef0123456789abcdef","publishes":["session.task","bot.changed","session.agent_usage","tls.changed"]}

data: {"type":"sessions.changed","session_id":"2026-10-14T12-04-05_3f2a1b0c-4d5e-6789-abcd-ef0123456789"}

data: {"type":"session.task","session_id":"2026-10-14T12-04-05_3f2a1b0c-4d5e-6789-abcd-ef0123456789"}
```

Refetch the whole list on:

- `sessions.changed` — created, deleted, archived, unarchived, pinned, unpinned, forked, renamed, a lane closed, a generated title, or an unseen run marked seen. `session_id` is the handle. Also refresh the title of that conversation if it is open.
- `session.run_started` and `session.run_finished` — the list's running state. `run_id`, when present, is `task_` plus 32 hex characters. On `session.run_finished`, also reload that conversation if it is open. The run stream closes on its own.
- `session.task` — run state, waiting messages, or child tasks changed. Refetch the list, and if that conversation is open also `GET /v1/sessions/{id}/run-state` and `GET /v1/sessions/{id}/pending-messages`. This event is optional: a Core sends it when `ready.publishes` lists it. Older Cores omit `publishes` and do not send `session.task`. `session.run_started` and `session.run_finished` still refresh running state.

`session.messages` means the transcript grew. It is frequent during a run. Reload `GET /v1/sessions/{id}` only when you are not already following that run's stream.

`resync` means this subscriber's queue overflowed and events were missed. Refetch the list and every open conversation. `503` with `Core is reloading` means connect again.

## 3. Open one conversation

```bash
curl -sS -H "Authorization: Bearer $GYRES_KEY" \
  http://127.0.0.1:15100/v1/sessions/2026-10-14T12-04-05_3f2a1b0c-4d5e-6789-abcd-ef0123456789
```

`200`:

```json
{
  "session_id": "2026-10-14T12-04-05_3f2a1b0c-4d5e-6789-abcd-ef0123456789",
  "records": [],
  "running": false,
  "run_state": "idle",
  "executor": "gyres",
  "history_version": "0-0"
}
```

`records` is the timeline, oldest first, including context records the list skips. Each record has `id`, `ts`, and `message` (`role` `user`, `assistant`, or `tool_result`, and `content`). `turn` is omitted when the record is not part of a turn; `status` is `running` until the turn finishes, then `completed`, `failed`, `cancelled`, or `interrupted`. Compare `history_version`; do not parse it. `history_cursor`, when present, is what a later reattach sends for an external-agent turn. `executor` is `gyres`, or `codex`, `grok`, `pi`, or `claude`. `404` means that id does not exist.

For a text-only view, `GET /v1/sessions/{id}/transcript` returns `records` of `{id, ts, role, text}` plus `transcript_size`, `truncated`, and `omitted`. It has no turns, metadata, or usage. The newest records that fit are kept; `truncated: true` means `omitted` records were left out. The response sends `Cache-Control: no-store`.

## 4. Create one

The body is ignored. Success is `200` and then `sessions.changed`:

```bash
curl -sS -X POST -H "Authorization: Bearer $GYRES_KEY" \
  http://127.0.0.1:15100/v1/sessions
```

```json
{ "session_id": "2026-10-14T12-04-05_3f2a1b0c-4d5e-6789-abcd-ef0123456789" }
```

Refetch the list, or insert this id and open it. A `500` means the conversation was not left in place.

## Edge cases

- A live row with `running: false` can still have `run_state` `interrupted`, `paused`, or `blocked`. Open it and read run-state before offering a new send.
- `title` missing is normal. `first_message` may be `""`.
- A side lane is not a list row. Open it as `{session_id}~{lane}` on `GET /v1/sessions/{id}`. The lane name is 1 to 48 characters, `[a-z0-9][a-z0-9_-]*`, and is never `main`.
- `500` on the list means the read failed. Retrying a `GET` is safe.
- While Core is reloading, a new `GET /v1/events` is `503`. An open stream ends; connect again. The same `503` can mean the data directory is unavailable, slow, or moving (`storage_unavailable`, `storage_maintenance`, `storage_slow`).

## See also

- [Conversations](../http/sessions.md) — `GET /v1/sessions`, `POST /v1/sessions`, `GET /v1/sessions/{id}`, `GET /v1/sessions/{id}/transcript`
- [GET /v1/events](../http/runs.md#get-v1events) and [Core-wide events](../events.md#core-wide-events-get-v1events) (`ready`, `resync`, `sessions.changed`, `session.task`, `session.run_started`, `session.run_finished`, `session.messages`)
- [Errors](../errors.md)
