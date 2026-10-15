# Events

Two Server-Sent Events streams. `GET /v1/events` is one Core-wide stream of small notifications: a type and ids, never message text, prompts, or paths. Apply it by refetching the ordinary route named on that type. `GET /v1/sessions/{id}/events` is the live progress of the conversation's current run. The same progress frames are what `POST /v1/sessions/{id}/turns` writes for the turn it accepted, from the start of that run.

Auth: any key

Both routes take `Authorization: Bearer <access key>`. Ordinary and administrator keys both work. A client ignores a `type` it does not know.

`{id}` is a conversation handle: a session id (a UTC timestamp, `_`, then a UUID, e.g. `2026-10-14T12-04-05_3f2a1b0c-4d5e-6789-abcd-ef0123456789`), or that id plus `~` and a lane name for a side lane. Events for a side lane name that handle, so a view of the main lane does not apply them.

## SSE framing

The body is `text/event-stream`, not compressed, with `Cache-Control: no-cache`. Each frame ends with a blank line. Neither stream sends an `event:` line. The name is the JSON `type` inside `data`.

A notification is one line:

```
data: {"type":"sessions.changed","session_id":"2026-10-14T12-04-05_3f2a1b0c-4d5e-6789-abcd-ef0123456789"}

```

A stored run event adds an `id:` line. The id is an opaque decimal. Keep the last one and send it back as `after` (below). Do not invent an id.

```
data: {"type":"text_delta","text":"Hello"}
id: 128

```

`run_resume`, `stream_error`, heartbeats, and every Core-wide frame have no `id`. A line that starts with `:` is a comment. Heartbeats are `: heartbeat` and carry nothing to apply.

`GET /v1/events` sends a heartbeat every 20 seconds. The run stream sends one every 10 seconds. Each subscriber of the Core-wide stream has a queue of 64 events. Publishing never waits. If that queue overflows, the missed events are gone and a `resync` follows the next event that was delivered, or comes just before the next heartbeat. There is no replay and no `Last-Event-ID` on either stream. A dropped Core-wide connection starts again at `ready`.

While Core is reloading, `GET /v1/events` answers `503` with `Core is reloading`. An open Core-wide stream ends when Core reloads; reconnect. A new connection can also be refused with `503` while the data directory is unavailable, slow, or moving.

`GET /v1/sessions/{id}/events` answers:

- `200` `text/event-stream` while a run is in progress. The response header `X-Gyres-Run-Stream` names that run's stream (16 hexadecimal characters). Browsers may read it: `Access-Control-Expose-Headers` lists it.
- `204` with an empty body when nothing is running. The header `X-Gyres-History-Version` is the current history version, the same string as `history_version` on `GET /v1/sessions/{id}`. It is two parts joined by `-`. Each part is a base-36 file size, `.`, and the modification time as base-36 Unix nanoseconds, or, once the conversation has a side lane, `l` plus that lane's end offset in base 36. `Access-Control-Expose-Headers` lists that header. If it matches the `history_version` just read, nothing ran in between.
- `409` `turn is preparing; retry observation` while a turn is starting and has no stream yet.
- `404` `session not found`.

Reconnect and resume use query parameters, checked in this order:

- `stream` plus `after`: `stream` is the `X-Gyres-Run-Stream` value and `after` is the last `id:` received. When both name this run and `after` is the end of one of its events, the first frame is `run_resume` and every later event follows, each exactly once. A different stream, a non-numeric `after`, or an `after` that is not the end of an event does not match.
- `history_cursor`: the `history_cursor` from `GET /v1/sessions/{id}` (the transcript version those records were read at: base-36 size, `.`, base-36 modification time; a lane uses `l` plus the end offset). An external-agent run marks the stream after the opening snapshot and after each transcript write it shows, and only when no streamed text or thinking is still unsaved. A matching cursor yields `run_resume`, then events after that mark. The client keeps the history it loaded. Approvals and questions that were not saved in the transcript are not resent; read `GET /v1/approvals` and `GET /v1/sessions/{id}/user-questions`. A long run keeps at most 8192 marks and then drops the older half, so an older cursor does not match.
- Anything else replays from the first event, which is `run_snapshot`. Gyres' own model runs place no marks, so `history_cursor` does not resume them. `stream` and `after` still do.

The stream ends when the run ends, after the remaining frames (the last stored one is `result` when the run published one). Close without `result` is a dropped connection: reconnect with `stream` and `after`. If the run has already finished, the next `GET` is `204`; reload `GET /v1/sessions/{id}` once. A broken progress log writes `stream_error` and then closes. The run itself continues; reload the conversation.

## Core-wide events (GET /v1/events)

The first frame is `ready`. Later frames are notifications. `session_id`, when present, is the conversation handle the change concerns.

### ready

Sent once when the stream opens, before any other frame.

```json
{
  "type": "ready",
  "version": "1.4.0",
  "instance_id": "0123456789abcdef0123456789abcdef",
  "storage_space": { "data": "ok", "home": "low" },
  "publishes": ["session.task", "bot.changed", "session.agent_usage", "tls.changed", "session.wake", "peers.changed"]
}
```

- `version` is this Core's build version. Omitted when empty. Compare it with the previous `ready` to notice an update.
- `instance_id` is the stable id of this Core, the same value as `GET /v1/identity`. 32 hexadecimal characters. Omitted when Core has none.
- `storage_space` is the current free-space level of the data volume and the home volume. Each of `data` and `home` is `ok`, `low`, or `critical`. Omitted before the first check.
- `publishes` lists optional types this Core sends. Stop polling for a type only when it is listed. Older Cores omit `publishes`.

Refetch: nothing yet. Use `version`, `instance_id`, and `storage_space` as the baseline.

### resync

Sent when this subscriber's queue overflowed. Events between the previous frame and this one may be missing.

```json
{ "type": "resync" }
```

No other fields.

Refetch: every view this client is showing (conversation list, the open conversation, approvals, schedules, and anything else those events would have refreshed).

### session.settings

A conversation's saved model choice changed: a settings update that stored a different choice, a turn that still sent a model and changed the saved choice, or a conversation that left an external agent and cleared that choice. An update that stores the same choice sends nothing. A title change does not send this type.

```json
{
  "type": "session.settings",
  "session_id": "2026-10-14T12-04-05_3f2a1b0c-4d5e-6789-abcd-ef0123456789"
}
```

- `session_id` is the conversation whose choice changed.

Refetch: `GET /v1/sessions/{id}/settings`.

### session.run_started

A run started. The id is the conversation the run belongs to, including a run started by a schedule, a channel, a goal continuation, or recovery.

```json
{
  "type": "session.run_started",
  "session_id": "2026-10-14T12-04-05_3f2a1b0c-4d5e-6789-abcd-ef0123456789",
  "run_id": "task_0123456789abcdef0123456789abcdef"
}
```

- `session_id` is the conversation handle.
- `run_id` is the task receipt id (`task_` plus 32 hexadecimal characters). Omitted when this Core is not keeping a receipt.

Refetch: the conversation list if it shows running state. Attach `GET /v1/sessions/{id}/events` to follow the run.

### session.run_finished

That run ended, including after stop or failure. `run_id` is the same value `session.run_started` carried for this run.

```json
{
  "type": "session.run_finished",
  "session_id": "2026-10-14T12-04-05_3f2a1b0c-4d5e-6789-abcd-ef0123456789",
  "run_id": "task_0123456789abcdef0123456789abcdef"
}
```

Refetch: `GET /v1/sessions/{id}` for the finished transcript. The run stream closes on its own.

### session.messages

The transcript grew: a message was appended, or an edit or delete was recorded. This is frequent during a run.

```json
{
  "type": "session.messages",
  "session_id": "2026-10-14T12-04-05_3f2a1b0c-4d5e-6789-abcd-ef0123456789"
}
```

Refetch: `GET /v1/sessions/{id}` when this client is not following that run. A client on the run stream already receives the new rows there and does not reload on each of these.

### sessions.changed

The conversation list changed for this handle: it was created, deleted, archived, unarchived, pinned, unpinned, forked, renamed, or a lane was closed. Also sent when a generated title arrives after creation, when a fork copies a title, and when an unseen finished run is marked seen. Clearing a seen mark that was already clear sends nothing. An unchanged title sends nothing.

```json
{
  "type": "sessions.changed",
  "session_id": "2026-10-14T12-04-05_3f2a1b0c-4d5e-6789-abcd-ef0123456789"
}
```

Refetch: `GET /v1/sessions`. If this handle is open, refetch that conversation's title and list fields too.

### approvals.changed

An approval request was saved, or approval settings or grants were saved.

```json
{ "type": "approvals.changed" }
```

No `session_id`.

Refetch: `GET /v1/approvals`.

### schedules.changed

A schedule plan or a schedule execution was saved or removed.

```json
{ "type": "schedules.changed" }
```

No `session_id`.

Refetch: `GET /v1/schedules`.

### chat_defaults.changed

The Settings default model changed (provider settings, the chat-defaults update, or a config write) and the new default is not the previous one.

```json
{ "type": "chat_defaults.changed" }
```

No `session_id`.

Refetch: `GET /v1/chat-defaults`.

### storage.space

The free-space level of the data volume or the home volume changed. The first observation sends nothing while the level is `ok`. A shared disk announces both volumes so neither side goes stale. The frame has no path and no byte counts.

```json
{ "type": "storage.space", "level": "critical", "volume": "data" }
```

- `level` is `ok`, `low`, or `critical`.
- `volume` is `data` or `home`.

Refetch: an administrator reads `GET /v1/storage` for sizes. The level in the event is enough for a badge.

### storage.unavailable

Sent once when the data directory latches unavailable, before in-flight work is cancelled. A short wait that then recovers sends nothing.

```json
{ "type": "storage.unavailable", "reason": "missing" }
```

- `reason` is the code, with no path: `identity_mismatch`, `timeout`, `full`, `read_only`, `not_writable`, `missing`, or `io_error`. It matches the unavailable reason on `GET /v1/storage`.

Refetch: show the unavailable state. An administrator reads `GET /v1/storage`. New requests fail until the directory is usable again.

### session.question

A run asked the owner a question (`ask_user_question`, or an external agent requesting input). The batch is the same id as `GET /v1/sessions/{id}/user-questions`.

```json
{
  "type": "session.question",
  "session_id": "2026-10-14T12-04-05_3f2a1b0c-4d5e-6789-abcd-ef0123456789",
  "question_id": "6f1c2a40-7b3e-4d1a-9c20-1a2b3c4d5e6f"
}
```

- `question_id` is the batch id.

Refetch: `GET /v1/sessions/{id}/user-questions`. A client already on the run stream also gets `user_question`.

### session.task

What the conversation's run state, pending messages, or child agents show has changed, apart from run start and finish. Sent when the task receipt's id, state, reason, non-internal queued messages, or children change; when pause or resume changes the live run (including the end of a pausing operation); when the run starts or stops waiting for children; after a successful queue, edit, delete, or hold of a pending message; and when an external-agent start is stopped before the turn is accepted. A child's change is sent for the child and again for its parent. Optional: listed in `ready.publishes`.

```json
{
  "type": "session.task",
  "session_id": "2026-10-14T12-04-05_3f2a1b0c-4d5e-6789-abcd-ef0123456789"
}
```

Refetch: `GET /v1/sessions/{id}/run-state` and `GET /v1/sessions/{id}/pending-messages`.

### bot.changed

The Bot's saved state changed (profile, models, chats, tasks), its memory text or memory change log changed, or a Bot chat started or finished a run. Optional: listed in `ready.publishes`.

```json
{ "type": "bot.changed" }
```

No `session_id`.

Refetch: `GET /v1/bot`. If the memory view is open, also `GET /v1/bot/memory`.

### peers.changed

Bot peering changed: a switch, a paired peer (paired, unpaired, online state, address), the Cores seen on the network, or a message in the Bot room. Optional: listed in `ready.publishes`.

```json
{ "type": "peers.changed" }
```

No `session_id`.

Refetch: `GET /v1/peering`, and the open room channel if one is shown.

### session.wake

A lane's pending [wake](http/runs.md#wakes) was set, replaced, cancelled, fired or dropped. Optional: sent only when `ready` lists it in `publishes`.

```json
{
  "type": "session.wake",
  "session_id": "2026-10-14T12-04-05_3f2a1b0c-4d5e-6789-abcd-ef0123456789"
}
```

`session_id` is the lane's handle.

Refetch: `GET /v1/wakes`.

### session.recap

A recap was stored for the conversation.

```json
{
  "type": "session.recap",
  "session_id": "2026-10-14T12-04-05_3f2a1b0c-4d5e-6789-abcd-ef0123456789"
}
```

Refetch: `GET /v1/sessions/{id}/recap`.

### session.agent_usage

Usage stored for an external agent on this conversation changed: new token totals, a finished agent turn, a rate-limit update during the run, or a background read of the Claude account the conversation asked for. Optional: listed in `ready.publishes`.

```json
{
  "type": "session.agent_usage",
  "session_id": "2026-10-14T12-04-05_3f2a1b0c-4d5e-6789-abcd-ef0123456789"
}
```

Refetch: `GET /v1/sessions/{id}/agent-usage`.

### plugins.changed

A plugin was installed, replaced, removed, enabled, disabled, pinned, had its settings saved, had an update check recorded, or a rescan ran.

```json
{ "type": "plugins.changed" }
```

No `session_id`.

Refetch: `GET /v1/plugins`.

### tls.changed

Core's HTTPS state or settings changed: anything `GET /v1/tls` reports except the handshake-error count. Optional: listed in `ready.publishes`.

```json
{ "type": "tls.changed" }
```

No `session_id`.

Refetch: an administrator reads `GET /v1/tls`.

## Run events (GET /v1/sessions/{id}/events)

Every stored frame's `data` is one JSON object with a `type`. The opening frame of a new observation is `run_snapshot`. A resumed observation starts with `run_resume` and does not repeat that snapshot.

`records` on `run_snapshot` are the same timeline rows as `GET /v1/sessions/{id}`. One row:

```json
{
  "id": "msg_6f1c2a40-7b3e-4d1a-9c20-1a2b3c4d5e6f",
  "ts": "2026-10-14T12:00:00.000Z",
  "message": {
    "role": "user",
    "content": [{ "type": "text", "text": "Hello" }]
  },
  "turn": {
    "id": "6f1c2a40-7b3e-4d1a-9c20-1a2b3c4d5e6f",
    "started_at": "2026-10-14T12:00:00.000Z",
    "status": "running"
  }
}
```

- `id` is the stored row id (`msg_` plus a UUID). A row written before ids existed is `legacy_<session>_<index>`.
- `ts` is UTC `YYYY-MM-DDTHH:MM:SS.mmmZ`.
- `message.role` is `user`, `assistant`, or `tool_result`.
- A user message is `role`, `content`, and optionally `provider_history` (`scope`, `output`, optional `memory_version`).
- An assistant message adds `stop_reason`, `usage`, and `error_message` (`null` when there is none). Optional: `failure`, `provider_history`, `duration_ms`, `kind` (`compaction_summary`, `stopped`, `truncated`, `failed_attempt`), `fallback`.
- A tool result message is `role`, `tool_call_id`, `tool_name`, `content`, `is_error`, and optional `details`.
- `content` items are tagged by `type`: `text` (`text`, optional `input_reference`), `image` (`data`, `mime_type`), `thinking` (`thinking`, `signature`), `redacted_thinking` (`data`), `tool_use` (`id`, `name`, `input` which is `null` when empty), `tool_result` (`tool_use_id`, `content`). Tool content is `text` plus `text`, or `image` plus `data` and `mime_type`.
- `usage` is `input_tokens`, `output_tokens`, `cache_read_tokens`, `cache_write_tokens`, and optional `cache_usage_reported`.
- `turn` is omitted when the row is not on a journal turn. `status` is `running` until the turn finishes, then `completed`, `failed`, `cancelled`, or `interrupted`. `finished_at` is omitted while running.
- `shared` is `true` on rows a side lane shares with the main conversation. Omitted otherwise.
- `metadata`, when present, may include `skill` (`name`), `authorId`, `authorParticipantId`, `targetParticipantIds`, `dispatchId`, `backgroundTrigger` (`kind`, `triggerId`, optional `label`), `internalMessage`, `contextEpoch`, `contextUpdateKey`, `importedFrom` (`terminal` when the row was copied from a terminal session), and `source` (`bot` when the Bot sent the message). `internalMessage` is `subAgentCompletion`, `turnAborted`, `dateStampReminder`, `goalContinuation`, `contextUpdate`, `coreMessage`, or `botTaskResult`.

`stop_reason` on an assistant message is `end_turn`, `tool_use`, `max_tokens`, `error`, or `aborted`. `failure`, when an error string is present, is `{ "code", "message", "http_status", "details" }`. `http_status` and `details` are omitted when empty. `code` is a safe classification such as `request_failed`, `rate_limited`, `timeout`, or `connection_failed`.

### run_snapshot

The first stored event of a run, and again after `output_discarded`. `records` is the timeline at that moment, including the user message that started the run once it has been stored.

```json
{
  "type": "run_snapshot",
  "session_id": "2026-10-14T12-04-05_3f2a1b0c-4d5e-6789-abcd-ef0123456789",
  "records": []
}
```

- `session_id` is the conversation handle.
- `records` is the timeline array above.

Refetch: do not merge this onto history already on screen. Replace the displayed transcript with `records`, then apply the events that follow.

### run_resume

Written only as the preamble of a resumed observation (`stream` plus `after`, or a matching `history_cursor`). It is not stored in the run and has no `id`.

```json
{
  "type": "run_resume",
  "session_id": "2026-10-14T12-04-05_3f2a1b0c-4d5e-6789-abcd-ef0123456789"
}
```

Refetch: nothing. Keep the transcript already shown, then apply only the events after this frame.

### agent_start

The agent loop started. An external-agent run sends the same type with no extra fields. A start that fails before the loop can send this and then `agent_end`.

```json
{
  "type": "agent_start",
  "turn_id": "6f1c2a40-7b3e-4d1a-9c20-1a2b3c4d5e6f",
  "started_at": "2026-10-14T12:00:00.000000000Z"
}
```

- `turn_id` is the journal turn id. Omitted when no journal turn was opened.
- `started_at` is UTC RFC3339 with nanoseconds. Omitted together with `turn_id`.

Refetch: nothing. Remember `turn_id` as the live turn.

### turn_start

A model step inside the run is starting. The first step is turn 1. Later steps, including a final explanation after repeated tool calls were blocked, send this before their output.

```json
{ "type": "turn_start", "turn": 1 }
```

- `turn` is the step number in this run, starting at 1.

Refetch: nothing. Open a new step in the live turn.

### text_delta

A piece of assistant text. Concatenate `text` in order until `message_complete` or `output_discarded`.

```json
{ "type": "text_delta", "text": "Hello" }
```

Refetch: nothing. Append `text`.

### thinking_delta

A piece of assistant thinking. Concatenate `thinking` in order. The signature arrives later on the thinking block inside `message_complete`.

```json
{ "type": "thinking_delta", "thinking": "Checking the file" }
```

Refetch: nothing. Append `thinking`.

### tool_input_delta

A fragment of a tool call. The first fragment of a call includes `name` and may have an empty `partial_json`. Later fragments omit `name`. While a tool runs, further fragments are progress text from that tool, not necessarily argument JSON. Concatenate `partial_json` for the same `id`.

```json
{
  "type": "tool_input_delta",
  "id": "call_1",
  "name": "read",
  "partial_json": "{\"path\":"
}
```

- `id` is the tool-call id.
- `name` is the tool name. Omitted when empty.
- `partial_json` is a text fragment. It may be `""`.

Refetch: nothing. Append the fragment to that call.

### message_complete

The assistant message for this step is complete. For a reply cut off at the output cap, `usage` is sent first and this event follows once the message is marked `truncated`. A discarded length stop does not send `message_complete` for the dropped reply.

```json
{
  "type": "message_complete",
  "message": {
    "content": [{ "type": "text", "text": "Hi" }],
    "stop_reason": "end_turn",
    "usage": {
      "input_tokens": 12,
      "output_tokens": 3,
      "cache_read_tokens": 0,
      "cache_write_tokens": 0
    },
    "error_message": null
  }
}
```

- `message` is an assistant message without `role`. Fields are the assistant fields listed above. `error_message` is always present (`null` or a string). `failure` is present when `error_message` is a string.

Refetch: nothing. Replace the streamed text of this step with `message`.

### usage

Token counts for the model step that just finished. Also sent when the reply is withheld for a length stop, before `output_discarded` or the truncated `message_complete`.

```json
{
  "type": "usage",
  "usage": {
    "input_tokens": 12,
    "output_tokens": 3,
    "cache_read_tokens": 0,
    "cache_write_tokens": 0
  }
}
```

Refetch: nothing. The conversation usage routes stay the source of stored totals.

### tool_start

A tool call is about to run. `input` is the full argument value.

```json
{
  "type": "tool_start",
  "call": { "id": "call_1", "name": "read", "input": { "path": "a.txt" } }
}
```

- `call.id` is the tool-call id.
- `call.name` is the tool name.
- `call.input` is the arguments, or `null` when there are none.

Refetch: nothing. Show the call as running. A call that is skipped sends `tool_blocked` instead, and a call cut off at the output cap can send `tool_end` without `tool_start`.

### tool_end

A tool call finished, or a call in a truncated reply was failed without running.

```json
{
  "type": "tool_end",
  "call": { "id": "call_1", "name": "read", "input": {} },
  "result": {
    "content": [{ "type": "text", "text": "ok" }],
    "is_error": false
  }
}
```

- `call` is the same shape as `tool_start`.
- `result.content` is tool content. An empty result is `[]`.
- `result.is_error` is `true` when the tool failed.
- `result.details` is an optional object whose fields depend on the tool. Omitted when empty.

Refetch: nothing. Show `result` on that call. This `result` has no `tool_call_id` or `tool_name`; those are on `call`.

### tool_blocked

The call did not run: the run was cancelled, a queued user message skipped the rest of the batch, or the same calls were repeated until they were stopped.

```json
{
  "type": "tool_blocked",
  "call": { "id": "call_2", "name": "write", "input": null },
  "reason": "Skipped because the agent run was cancelled.",
  "result": {
    "tool_call_id": "call_2",
    "tool_name": "write",
    "content": [{ "type": "text", "text": "Skipped because the agent run was cancelled." }],
    "is_error": true
  }
}
```

- `reason` is a short explanation. Omitted when empty. Repeated calls use `Repeated identical tool calls`.
- `result` is a tool-result message: `tool_call_id`, `tool_name`, `content`, `is_error`, optional `details`.

Refetch: nothing. Show the call as not run.

### steering_message

A user message is entering the run between model steps: the owner steered, a queued follow-up was taken, or Core inserted its own message. An ordinary message has no `metadata`. An internal one is sent with `metadata` instead of the plain form.

```json
{
  "type": "steering_message",
  "message": {
    "role": "user",
    "content": [{ "type": "text", "text": "Also check the tests" }]
  }
}
```

```json
{
  "type": "steering_message",
  "message": {
    "role": "user",
    "content": [{ "type": "text", "text": "[Child task outcome]" }]
  },
  "metadata": { "internalMessage": "subAgentCompletion" }
}
```

- `message` is a user message (`role`, `content`, optional `provider_history`).
- `metadata.internalMessage` is the stored kind. When the kind is empty it is sent as `subAgentCompletion`. Other kinds are the `internalMessage` values listed above.

Refetch: nothing. Show the message in the run in order. Internal rows are Core's own text, not a new owner prompt.

### approval

An approval request was opened, denied immediately by the unattended policy, or decided. The same shape is sent for Gyres tools and for an external agent's approval. The stored request can still change after the last frame (for example the policy changed before the call ran); `approvals.changed` on the Core-wide stream is the signal to read the list.

```json
{
  "type": "approval",
  "approval": {
    "id": "0123456789abcdef0123456789abcdef",
    "session_id": "2026-10-14T12-04-05_3f2a1b0c-4d5e-6789-abcd-ef0123456789",
    "run_id": "task_0123456789abcdef0123456789abcdef",
    "tool_call_id": "call_1",
    "tool": "exec_command",
    "resource": "exec_command ls",
    "input": { "command": "ls" },
    "cwd": "/work/app",
    "unattended": false,
    "status": "pending",
    "created_at": "2026-10-14T12:00:00.000000000Z"
  }
}
```

- `id` is 32 hexadecimal characters.
- `status` is `pending`, `approved`, `denied`, `rejected`, `expired`, or `cancelled`.
- `scope`, when the decision has one, is `once`, `session`, or `global`.
- `reason` is why it was denied, rejected, expired, or cancelled. Omitted while pending.
- `finished_at` is set once the request is no longer pending.
- `once_only` is `true` for a request that cannot be covered by a grant. Omitted otherwise.
- `tool_identity`, `tool_wide`, `policy_session`, `tool_label`, and `resolver` are present when set. `resolver` names who decided when it was not a person at the card.
- `unattended` is a boolean and is present even when `false`. `tool_wide` is present only when `true` (the request covers the whole tool).
- The match key used to compare requests is not included.

Refetch: `GET /v1/approvals` for the list. The card in the run can be updated from `approval` directly.

### user_question

The run is waiting for the owner. The native tool asks 1 to 3 questions, each with 2 or 3 options, and waits until `deadline` (60 seconds unless a longer wait was started). An external agent uses the same event.

```json
{
  "type": "user_question",
  "user_question": {
    "id": "6f1c2a40-7b3e-4d1a-9c20-1a2b3c4d5e6f",
    "session_id": "2026-10-14T12-04-05_3f2a1b0c-4d5e-6789-abcd-ef0123456789",
    "tool_call_id": "call_9",
    "questions": [
      {
        "id": "http",
        "header": "Client",
        "question": "Which HTTP client?",
        "options": [
          { "label": "fetch", "description": "Use the platform fetch." },
          { "label": "axios", "description": "Use the axios package." }
        ]
      }
    ],
    "created": "2026-10-14T12:00:00.000000000Z",
    "deadline": "2026-10-14T12:01:00.000000000Z"
  }
}
```

- `id` is the batch id (a UUID), the same as `question_id` on `session.question`.
- `questions[].id` is the id answers are keyed by.
- `created` and `deadline` are UTC timestamps.

Refetch: `GET /v1/sessions/{id}/user-questions` if this frame was missed. Answer with `POST /v1/sessions/{id}/user-questions/{qid}/answer`.

### user_question_resolved

The wait ended. On success, `source` is `user` (the owner answered) or `timeout` (each question took its first option). `answers` maps each question id to the chosen label.

```json
{
  "type": "user_question_resolved",
  "id": "6f1c2a40-7b3e-4d1a-9c20-1a2b3c4d5e6f",
  "session_id": "2026-10-14T12-04-05_3f2a1b0c-4d5e-6789-abcd-ef0123456789",
  "source": "user",
  "answers": { "http": "fetch" }
}
```

When the native wait is cancelled:

```json
{
  "type": "user_question_resolved",
  "id": "6f1c2a40-7b3e-4d1a-9c20-1a2b3c4d5e6f",
  "session_id": "2026-10-14T12-04-05_3f2a1b0c-4d5e-6789-abcd-ef0123456789",
  "source": "cancelled",
  "error": "context canceled",
  "failure": {
    "code": "request_failed",
    "message": "The request failed. Open details for more information.",
    "details": "context canceled"
  }
}
```

- `source` is `user`, `timeout`, or `cancelled`.
- `answers` is present on `user` and `timeout`. Omitted on the native `cancelled` frame.
- `error` is the wait error. A non-empty `error` is accompanied by `failure`.
- An external-agent question that is cancelled can instead send `source` as `""` and `answers` as `null`, with no `error`.

Refetch: drop the open question. The list from `GET /v1/sessions/{id}/user-questions` no longer includes a resolved batch.

### goal_updated

The conversation's goal was created, replaced, paused, completed, blocked, or stopped by the run (usage limit, error, or the continuation cap). Reading the goal does not send this.

```json
{
  "type": "goal_updated",
  "goal": {
    "id": "6f1c2a40-7b3e-4d1a-9c20-1a2b3c4d5e6f",
    "session_id": "2026-10-14T12-04-05_3f2a1b0c-4d5e-6789-abcd-ef0123456789",
    "objective": "Ship the export",
    "status": "active",
    "continuations": 0,
    "created": "2026-10-14T12:00:00.000000000Z",
    "updated": "2026-10-14T12:00:00.000000000Z"
  }
}
```

- `status` is `active`, `paused`, `blocked`, `complete`, `usage_limited`, or `capped`.
- `provider_id` and `model` are omitted when the goal has no model of its own.
- `reason` and `stopped_at` are set when the goal stopped continuing, and cleared when it is active again.
- `blocker`, `blocker_turns`, and `blocker_this_turn` describe a blocker report. Omitted when unset. `blocker_turns` is omitted at 0.
- `continuations` is how many automatic continuations have run. It is present even at 0.

Refetch: `GET /v1/sessions/{id}/goal` if this frame was missed. Otherwise apply `goal` (that route returns `{ "goal": <this object> }`).

### model_fallback

The model for this request failed before producing output, and the next model in the fallback chain is being tried. One event per model that is actually tried.

```json
{
  "type": "model_fallback",
  "from_provider_id": "prov_main",
  "from_model": "model-a",
  "provider_id": "prov_backup",
  "model": "model-b",
  "reason": "rate_limit",
  "status": 429
}
```

- `from_provider_id` and `from_model` are the model that failed.
- `provider_id` and `model` are the model now serving the request.
- `reason` is `rate_limit`, `server_error`, `timeout`, `network`, `model_not_found`, `auth`, or `balance`.
- `status` is the HTTP status of the failure. Omitted when there was none.

Refetch: nothing. The assistant message that succeeds carries the same switch on `fallback`. A pinned model does not emit this.

### retry

The model request failed and will be sent again after `delay_ms`. `attempt` counts failures so far, starting at 0. `max_retries` is the cap. `error` is the safe text of the failure.

```json
{
  "type": "retry",
  "attempt": 0,
  "max_retries": 3,
  "error": "request failed",
  "delay_ms": 2000
}
```

Refetch: nothing. Show a retry. Output already streamed is not discarded by this event.

### output_discarded

A recoverable length stop dropped the reply streamed since the last stored row. `reason` is `overflow`. The client drops the text and thinking it showed for that reply. A `run_snapshot` of the stored transcript follows when the transcript can be read; replace the view with that snapshot before the retried reply streams.

```json
{ "type": "output_discarded", "reason": "overflow" }
```

Refetch: if no `run_snapshot` follows, reload `GET /v1/sessions/{id}`.

### turn_end

The model step numbered `turn` has finished (tools included, or the step was interrupted and the run is carrying on).

```json
{ "type": "turn_end", "turn": 1 }
```

Refetch: nothing. Close that step. The run continues until `agent_end`.

### agent_end

The agent loop finished. An empty object means the run completed with no error string. External-agent runs send this before `result`.

```json
{ "type": "agent_end" }
```

```json
{
  "type": "agent_end",
  "error": "boom",
  "failure": {
    "code": "request_failed",
    "message": "The request failed. Open details for more information.",
    "details": "boom"
  },
  "provider_error_code": "429",
  "provider_error_type": "overloaded"
}
```

- `error` is the safe error text. Omitted when the run has none.
- `failure` is present when `error` is present.
- `provider_error_code` and `provider_error_type` are the provider's codes. Omitted when empty.

Refetch: nothing yet. Wait for `result`, then reload the conversation.

### result

The run's outcome, after the task receipt was updated. This is the last stored event of a normal run. `stop_reason` on this frame is `end_turn`, `error`, or `aborted` (a panic in the run is `error`).

```json
{
  "type": "result",
  "session_id": "2026-10-14T12-04-05_3f2a1b0c-4d5e-6789-abcd-ef0123456789",
  "cwd": "/work/app",
  "stop_reason": "end_turn"
}
```

```json
{
  "type": "result",
  "session_id": "2026-10-14T12-04-05_3f2a1b0c-4d5e-6789-abcd-ef0123456789",
  "stop_reason": "error",
  "error": "task interrupted by an internal error"
}
```

- `cwd` is the working directory. Omitted when empty.
- `error` is the failure text. Omitted when `stop_reason` is `end_turn` or `aborted`.

Refetch: when the stream closes, `GET /v1/sessions/{id}` is the authoritative transcript. `session.run_finished` on the Core-wide stream tells other clients.

### stream_error

The progress log could not be read. This frame is not stored and has no `id`. The stream then closes. The run is not cancelled by this frame.

```json
{
  "type": "stream_error",
  "error": "progress stream unavailable; reload conversation"
}
```

Refetch: `GET /v1/sessions/{id}`. Attach again with `GET /v1/sessions/{id}/events` if `running` is still set.
