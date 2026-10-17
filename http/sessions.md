# Conversations

A conversation is the unit a frontend lists, opens, and edits. These routes create and address one, return its timeline, and change its title, archive, pin, model, workspace, context, and recap. Sending a message and following a live run are separate routes.

Every JSON response is `application/json`. An error body is `{"error":"<message>","failure":{"code":"core_request_failed","message":"<message>","http_status":<status>,"details":"<message>"}}`. The `GET` transcript route also sends `Cache-Control: no-store`. Success is `200`. Empty arrays are `[]`, not `null`.

Many routes take either a session id or a lane handle in `{id}`. A lane is another path through the same conversation, branched from one message. The list contains only main conversations. Settings, the recap, and the seen mark belong to the lane named in the path. The workspace, the context epoch, and the context mode belong to the whole conversation: a lane handle reads and writes the same values as its session id. Archive, unarchive, pin, and unpin reject a lane handle before they check that the conversation exists.

## Identifiers

A new session id looks like `2026-10-14T12-04-05_3f2a1b0c-4d5e-6789-abcd-ef0123456789`: a UTC timestamp with `:` and `.` rewritten as `-` and cut to 19 characters, then `_`, then a UUID. A message id looks like `msg_3f2a1b0c-4d5e-6789-abcd-ef0123456789`. A turn id looks like `turn_3f2a1b0c4d5e6789abcdef0123456789`. Do not parse these.

A lane handle is `<session id>~<lane id>`, with exactly one `~`. The session id is 1 to 160 of `A–Z`, `a–z`, `0–9`, `-`, and `_`. The lane id matches `^[a-z0-9][a-z0-9_-]{0,47}$` and is not `main`. `~main` is not a handle. The bare session id is the main lane.

An id that was stored without one is reported as `legacy_`, the session id with any other character replaced by `_`, `_`, and the 0-based index. Clients should keep the `id` they were given.

Timestamps written by these routes (`created`, `modified`, `pinned_at`, `archived_at`, record `ts`, turn times, `updated_at`, recap `created_at`) are UTC `2026-10-14T12:04:05.000Z`. A terminal import's `imported_at` is UTC `2026-10-14T12:04:05Z`.

## Run state

`running` is true while this lane has a live run slot, including while that run is paused. `run_state` is separate:

- `running` while the slot is active and not paused.
- `pausing` while a pause was asked and work is still inside the run.
- `paused` while the slot is paused and that work has left.
- `waiting_children` while the slot says running and is waiting on children.
- Otherwise, if a task receipt is still pending: `blocked` when the receipt store is blocked, else the receipt's `running`, `paused`, `interrupted`, or `blocked`.
- `idle` when none of that applies.

A list row can have `running: true` and `run_state: "paused"`. A leftover receipt can have `running: false` and `run_state` `interrupted`, `paused`, or `blocked`.

`unseen` is `""`, `result` (a completed run ended after the conversation was last marked seen), or `failed`. A cancelled run does not mark the conversation, and a child conversation (one with a parent) does not either. A scheduled task's run marks it only as its plan's `notify` level allows (see the scheduled tasks reference). On a live list row `unseen` is always present; `""` means nothing unseen. `unseen_kind` is who started that run: `user`, `scheduled`, `bot_task`, `external`, `wake`, `goal`, or `recovery`. It is omitted when `unseen` is `""` or the mark was written before this field existed.

## List rows

`GET /v1/sessions` without `archived=true` returns live rows, newest `modified` first (string order, which is chronological for the timestamp above). Pinned conversations stay in that order; `pinned_at` is only a marker. Bot conversations are omitted, and so is a child conversation whose parent still exists. A side lane is not its own row.

```json
{
  "path": "/data/sessions/2026-10-14T12-04-05_3f2a1b0c-4d5e-6789-abcd-ef0123456789.jsonl",
  "session_id": "2026-10-14T12-04-05_3f2a1b0c-4d5e-6789-abcd-ef0123456789",
  "created": "2026-10-14T12:04:05.000Z",
  "modified": "2026-10-14T12:10:00.000Z",
  "message_count": 4,
  "first_message": "Sketch the layout",
  "title": "Layout",
  "pinned_at": "2026-10-14T12:06:00.000Z",
  "model": {"provider_id": "example-provider", "model": "example-model"},
  "running": false,
  "run_state": "idle",
  "unseen": ""
}
```

- `path` is the transcript file on the Core host. Address the conversation by `session_id`.
- `created` is the `ts` of the first record on the main path, including a context record, or `modified` when there is none. `modified` is the transcript file's modification time.
- `message_count` counts records on the main path except context records. A tool result counts. Context records still appear in `GET /v1/sessions/{id}`.
- `first_message` is the text of the first user or assistant record on that path that is not a context record. Text blocks are joined with one space and trimmed. On a user message, a second text block that is a `<session_references>` wrapper is skipped. Tool results contribute no text. `""` when nothing qualifies. Always present.
- A context record has `metadata.contextUpdateKey` of `workspace_environment`, `current_date`, `bot_memory`, or `workspace_roots_status`.
- `title` is omitted when no custom title is stored. Clients then show `first_message`.
- `pinned_at` is omitted when the conversation is not pinned. Pinning again keeps the original time.
- `model` is `{"provider_id","model"}` of the conversation's saved `settings` (`GET /v1/sessions/{id}/settings`), without reasoning effort or speed. It is omitted when no model is saved; a turn then uses the Settings default. An external-agent conversation carries the agent's provider id (`local-codex`, `local-grok`, `local-pi`, `local-claude`).
- `workspace`, when present, is the workspace view below. It is omitted when the conversation has no workspace, and also when that binding cannot be read.
- `scheduled`, when present, is `{"job_id","name","run_id"}` for a conversation a standalone scheduled run created.
- `origin`, when present on a list row, is only a Bot origin (below). A terminal import's origin is not on the list.
- `unseen_kind` is omitted when `unseen` is `""`. When present it is `user`, `scheduled`, `bot_task`, `external`, `wake`, `goal`, or `recovery`.

`archived=true` (exactly that string; `1`, `True`, and a missing parameter are the live list) returns archived rows, newest `archived_at` first. Bot conversations are omitted. Side lanes are not rows.

```json
{
  "session_id": "2026-10-14T12-04-05_3f2a1b0c-4d5e-6789-abcd-ef0123456789",
  "created": "2026-10-14T12:04:05.000Z",
  "modified": "2026-10-14T12:10:00.000Z",
  "message_count": 4,
  "first_message": "Sketch the layout",
  "title": "Layout",
  "archived_at": "2026-10-14T13:00:00.000Z",
  "bytes": 18432
}
```

`bytes` is roughly what deleting the conversation frees, including files kept with it beyond the transcript. There is no `path`, `running`, `run_state`, `unseen`, `pinned_at`, `workspace`, or `origin`.

## Timeline

`GET /v1/sessions/{id}` returns the lane's current path. Each record:

```json
{
  "id": "msg_3f2a1b0c-4d5e-6789-abcd-ef0123456789",
  "ts": "2026-10-14T12:04:05.000Z",
  "message": { "role": "user", "content": [{ "type": "text", "text": "Sketch the layout" }] },
  "turn": {
    "id": "turn_3f2a1b0c4d5e6789abcdef0123456789",
    "started_at": "2026-10-14T12:04:05.100Z",
    "finished_at": "2026-10-14T12:04:20.000Z",
    "status": "completed"
  },
  "shared": true
}
```

- `ts` is always present and may be `""`.
- `metadata` is omitted when empty. See below.
- `turn` is omitted on a record that belongs to no turn. A record belongs to at most one turn. `status` is `running` until the turn finishes. A finished turn is `completed`, `failed`, `cancelled`, or `interrupted`. A finish with no outcome is `interrupted`. `finished_at` is omitted while the turn is still running. The same turn object is shared by the records it covers.
- `shared` is `true` on the leading records of a side lane, the prefix it shares with the conversation it branched from. Those indexes cannot be edited or deleted. Omitted otherwise.

`history_version` is an opaque string. Compare it; do not parse it. It changes when the transcript or the turn placement changes. It is two parts joined by `-`. Before any side lane exists, each part is the file size and modification time in base 36, or `0` when that file is missing. Once another lane exists, each part is `l` plus an end offset in base 36, and each lane has its own value.

`history_cursor`, when present, is the transcript half of the version these records were read at. It is omitted when the read could not pin one version. A live run stream accepts it as the `history_cursor` query.

`branch`, only for a side lane:

```json
{
  "session_id": "2026-10-14T12-04-05_3f2a1b0c-4d5e-6789-abcd-ef0123456789",
  "lane": "side1",
  "from_message_id": "msg_aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee",
  "shared_count": 2
}
```

`from_message_id` is the message the lane branched from, or `""` when it branched at the start or that id cannot be read. `shared_count` is how many leading records have `shared: true`.

`parent_session` is included when task records are kept: the parent conversation id, or `""` when there is none. It is omitted when task records are not kept.

`executor` is `gyres` unless an external agent owns the conversation, then `codex`, `grok`, `pi`, or `claude`. If more than one binding is present, the last one applied wins, in that order, so Claude wins over pi, Grok, then Codex. `executor_model` is that binding's model. `executor_options` for Codex is `{"reasoning_effort","service_tier"}`, either of which may be `""`. For Grok, pi, and Claude it is `{"reasoning_effort"}` only.

`origin` is a Bot origin, unless a Codex, Grok or Claude binding carries a terminal origin, which replaces it. pi does not set `origin`.

```json
{ "source": "bot", "bot_session_id": "2026-10-14T11-00-00_aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee", "task_id": "bot_0123456789abcdef0123456789abcdef" }
```

A Bot origin is present when a visible task that has not been adopted opened the conversation. `source` is `bot`.

```json
{
  "agent": "codex",
  "source": "terminal",
  "conversation_id": "2026-10-14T12-04-05_3f2a1b0c-4d5e-6789-abcd-ef0123456789",
  "title": "Imported chat",
  "model": "example-model",
  "reasoning_effort": "medium",
  "sandbox": "workspace-write",
  "sandbox_changed_to": "workspace-write",
  "imported_at": "2026-10-14T12:04:05Z",
  "dropped": { "thinking": 1, "images": 2, "compaction": 1, "other": 1 },
  "complete": true
}
```

`agent` is `codex` or `grok`. `source` is `terminal`. `conversation_id` is this Gyres conversation. `complete` is true once the import finished. `title`, `model`, `reasoning_effort`, `sandbox`, `sandbox_changed_to`, and `dropped` are omitted when empty. `dropped` counts records the import did not copy; keys that appear are `thinking`, `images`, `other`, and for Codex also `compaction`.

`cwd`, `cwd_kind` (`project` or `scratch`), and `workspace` are present when a workspace is bound. A failure reading the workspace or an agent binding is `500`.

## Metadata

Omitted as a whole when empty. Field names are camelCase.

- `skill`: `{"name"}`.
- `authorId`, `authorParticipantId`, `dispatchId`: strings.
- `targetParticipantIds`: array of strings.
- `backgroundTrigger`: `{"kind","triggerId","label"}`. `label` is omitted when unset.
- `internalMessage`: `subAgentCompletion`, `turnAborted`, `dateStampReminder`, `goalContinuation`, `contextUpdate`, `coreMessage`, `botTaskResult`, or `wake` (the hidden first message of a turn a [wake](runs.md#wakes) started).
- `contextEpoch`: string.
- `contextUpdateKey`: string. The four values listed under list rows mark a context record.
- `importedFrom`: `terminal` when the record was copied from a terminal session.
- `source`: `bot` when the Bot sent the message into a conversation it opened.

## Messages

`message.role` is `user`, `assistant`, or `tool_result`. On input, `toolResult` is accepted and stored as `tool_result`. `content` is never `null`; an empty message has `[]`.

User:

```json
{ "role": "user", "content": [{ "type": "text", "text": "Sketch the layout" }] }
```

Only `role` and `content` are on the wire.

Assistant:

```json
{
  "role": "assistant",
  "content": [{ "type": "text", "text": "Done." }],
  "stop_reason": "end_turn",
  "usage": {
    "input_tokens": 12,
    "output_tokens": 4,
    "cache_read_tokens": 0,
    "cache_write_tokens": 0
  },
  "error_message": null
}
```

- `stop_reason` is `end_turn`, `tool_use`, `max_tokens`, `error`, or `aborted`. On input, a missing `stop_reason` is stored as `end_turn`.
- `usage` always includes the four token counts (0 included). `cache_usage_reported` is added when true.
- `error_message` is always present: a string, or `null`. Credentials are stripped.
- `failure`, when `error_message` was set, is `{code, message, details, http_status}`. Display `failure.message`. `error_message` is the safe original. `details` is that same safe text. `http_status` is omitted when the failure did not carry one. `code` is `request_failed`, `invalid_request`, `authentication_failed`, `permission_denied`, `not_found`, `timeout`, `request_too_large`, `unsupported_media`, `rate_limited`, `service_unavailable`, or `connection_failed`. This is not the route error's `core_request_failed`.
- `duration_ms` is the model call that produced the reply, from sending the request to the end of its stream, in milliseconds. It is omitted when unset (older messages, replies Core built itself).
- `request_id` and `step` name that model call: the `request_id` and `step` of its usage rows and of the `usage` event. Each is omitted when unset (older messages, replies Core built itself, `step` outside a run's steps).
- `provider_id` and `model` name the model that answered that call (the fallback model when one answered). `started_at` (RFC 3339) is when the request went to the provider, the start of `duration_ms`, and `first_token_at` when its stream delivered the first output; with `duration_ms` they give the call's output speed and first-token latency. Each is omitted when unset (older messages, replies Core built itself); `first_token_at` is also omitted when nothing streamed.
- `kind`, when set, is `compaction_summary`, `stopped`, `truncated`, or `failed_attempt`.
- `fallback`, when a fallback model answered: `provider_id`, `model`, `from_provider_id`, `from_model`, `reason`, and `status` (omitted when there was no HTTP status). `reason` is `rate_limit`, `server_error`, `timeout`, `network`, `model_not_found`, `auth`, or `balance`.
- `provider_history`, when present: `memory_version` (omitted when empty), `scope`, and `output` (an array of JSON values). It is opaque replay data. A client can ignore it for display.

Tool result:

```json
{
  "role": "tool_result",
  "tool_call_id": "call_1",
  "tool_name": "present_file",
  "content": [{ "type": "text", "text": "saved" }],
  "is_error": false
}
```

`is_error` is always present. On input, `toolCallId`, `toolName`, and `isError` are accepted and stored under the snake_case names. `details` is omitted when empty; its shape depends on the tool.

Content blocks carry only the fields of their `type`:

- `text`: `type`, `text`, and `input_reference` when true.
- `image`: `type`, `data` (the image payload), `mime_type`.
- `thinking`: `type`, `thinking`, `signature` (present even when `""`).
- `redacted_thinking`: `type`, `data`.
- `tool_use`: `type`, `id`, `name`, `input`. `input` is JSON, `null` when empty.
- `tool_result`: `type`, `tool_use_id`, `content`. Each item is `{"type":"text","text"}` or `{"type":"image","data","mime_type"}`.

## Workspace view

`GET` returns JSON `null` when the conversation has no workspace. Otherwise:

```json
{
  "project_id": "demo",
  "roots": ["/projects/demo", "/projects/demo/library"],
  "follow_primary": true,
  "cwd": "/projects/demo",
  "kind": "project",
  "library": "/projects/demo/library",
  "chat_dir": "/projects/demo/chats/layout",
  "status": {
    "available": true,
    "directories": [{ "path": "/projects/demo", "available": true }]
  }
}
```

- `kind` is `project` or `scratch`. `cwd` and `kind` are always present.
- `project_id`, `roots`, `follow_primary`, `library`, `chat_dir`, and `child_root` are omitted when empty. `id` (a UUID) is set on a scratch workspace and omitted otherwise. `follow_primary` means `cwd` tracks the project's primary directory.
- `child_root` is the directory a child conversation was given when that directory is not one of the parent's.
- `status.available` is false when `cwd` or, for a project, its primary directory cannot be used. An extra root is listed with `available: false` but does not by itself clear `status.available`. A project library or conversation directory that has not been created yet still counts as available.
- `directories[].reason` is omitted when the directory is usable. When `cwd` is no longer inside the project it is `current directory no longer belongs to this project; choose another location`. A path that fails the directory check carries that check's text, such as `use an absolute directory path on the Core host` or `invalid project cwd "..."`.

## External agents

Codex, Grok, pi, and Claude can own a conversation. Their provider ids are `local-codex`, `local-grok`, `local-pi`, and `local-claude`. Routes that refuse a history or workspace change answer `409` `{Label} owns this conversation history and workspace; start a new conversation to change them`. `{Label}` is `Codex`, `Grok`, `pi`, or `Claude`. A failure while reading the binding is refused the same way; when no name is known the label is `An external agent`. The context routes use a different sentence, and only when a binding is found: `{Label} owns its conversation context`.

### GET /v1/sessions

Auth: any key
Effect: Lists conversations. The live list is the default. `archived=true` lists archived conversations instead.

Query: `archived` (string). Exactly `true` selects the archived list. Any other value, including omitted, selects the live list.

Request: no body.

Response: `200`

```json
{ "sessions": [] }
```

`sessions` is the live rows or the archived rows above. An empty list is `[]`.

Errors:

- `500` the list could not be read

Events: none.

### POST /v1/sessions

Auth: any key
Effect: Creates an empty main conversation and stores the Core default context mode (`window` only when that default is exactly `window`, otherwise `summary`). The body is not read.

Request: no body.

A body, if sent, is ignored.

Response: `200`

```json
{ "session_id": "2026-10-14T12-04-05_3f2a1b0c-4d5e-6789-abcd-ef0123456789" }
```

Errors:

- `500` the conversation could not be created. If the context mode cannot be stored, the new conversation is removed.

Events: `sessions.changed`. If storing the context mode fails, the removed conversation is announced the same way.

### GET /v1/sessions/{id}

Auth: any key
Effect: Returns one lane's timeline and the conversation fields a client needs to open it. A lane handle selects that lane; the bare session id selects main.

Path: `id` session id or lane handle.

Request: no body.

Response: `200`

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

`records` is the timeline above, including context records the list skips. `running` is true if a run slot existed before the read or still exists after it. Optional fields, when present, are `history_cursor`, `title`, `branch`, `origin`, `parent_session`, `executor_model`, `executor_options`, `cwd`, `cwd_kind`, and `workspace`, as specified above.

Errors:

- `404` session not found
- `500` the timeline, an agent binding, or the workspace could not be read

Events: none.

### GET /v1/sessions/{id}/transcript

Auth: any key
Effect: Returns a text-only, size-capped view of the same lane path. It has no turns, metadata, or usage. The response sends `Cache-Control: no-store`.

Path: `id` session id or lane handle.

Request: no body.

Response: `200`

```json
{
  "session_id": "2026-10-14T12-04-05_3f2a1b0c-4d5e-6789-abcd-ef0123456789",
  "transcript_size": 4096,
  "truncated": false,
  "omitted": 0,
  "records": [
    {
      "id": "msg_3f2a1b0c-4d5e-6789-abcd-ef0123456789",
      "ts": "2026-10-14T12:04:05.000Z",
      "role": "user",
      "text": "Sketch the layout"
    }
  ]
}
```

- `transcript_size` is the transcript file's size in bytes, or, once the conversation has another lane, this lane's end offset. A missing transcript is `0`. `PUT` recap sends this number back as `if_transcript_size`.
- Each text field is clipped in Unicode code points: message text at 16000, a tool call's `input` at 2000, a tool result's text at 8000. A clipped value ends with `…`.
- The view keeps the newest records that fit in 2097152 bytes of JSON, plus one byte counted per record. The first user record that is not internal is always kept, even when it falls outside that tail.
- `omitted` is how many records were left out. `truncated` is true when `omitted` is greater than 0.
- `internal` is true when `metadata.internalMessage` is set, and is omitted otherwise. `source: "bot"` alone does not set it.
- `tool_calls` is present for an assistant record that used tools: `{id, name, input}`. `input` is the tool's JSON as clipped text, not an object.
- A tool result adds `tool_call_id`, `tool_name`, and `is_error`. `is_error` is omitted when false.
- `ts` and `text` are omitted when empty.

Errors:

- `404` session not found
- `500` the transcript could not be read

Events: none.

### PATCH /v1/sessions/{id}

Auth: any key
Effect: Sets or clears the lane's title. Whitespace is trimmed and each run of whitespace, including newlines, becomes one space. A title longer than 100 Unicode code points is rejected. An empty result clears the title, and the list falls back to `first_message`. A lane handle names that lane's title. A Bot chat is renamed in the Bot's chat list instead, and its title cannot be cleared.

Path: `id` session id or lane handle.

Request:

```json
{ "title": "Layout" }
```

- `title` (string) is required. The body is at most 4096 bytes. Other fields are ignored.

Response: `200`

```json
{
  "session_id": "2026-10-14T12-04-05_3f2a1b0c-4d5e-6789-abcd-ef0123456789",
  "title": "Layout"
}
```

`title` is the normalized value, `""` when the title was cleared.

Errors:

- `404` session not found
- `400` title is required
- `400` title is too long (100 characters at most)
- `400` a Bot chat needs a title
- `400` this Bot conversation cannot be renamed
- `500` the title could not be saved

Events: `sessions.changed` when a non-Bot title actually changes. The same title again publishes nothing. A Bot rename always publishes `sessions.changed` and `bot.changed` (`bot.changed` has no `session_id`).

### DELETE /v1/sessions/{id}

Auth: any key
Effect: Deletes a main conversation and the files kept for it. A lane handle closes that lane only: its rows leave the open path, and the conversation and its other lanes stay. Closing a lane is refused while that lane's run slot is held. Deleting the whole conversation is refused while a lane is running or still has a task, and while Bot side chats branched from it still exist.

Path: `id` session id or lane handle.

Request: no body.

Response: `200`

```json
{ "session_id": "2026-10-14T12-04-05_3f2a1b0c-4d5e-6789-abcd-ef0123456789~side1" }
```

`session_id` is the id that was given.

Errors:

- `404` session not found (unknown id, or a lane that is not open)
- `409` turn already running
- `409` `delete the side chats branched from this chat first: ` followed by those chats' titles
- `500` the conversation could not be removed. `Cannot remove conversation images` means the conversation itself is already gone and only its images could not be removed. A failure before removal leaves the conversation in place.

Events: deleting a main conversation publishes `sessions.changed`, and also `bot.changed` when the id is a Bot chat. Closing a lane publishes `session.messages` and `sessions.changed`, and also `bot.changed` when this Core keeps Bot state, even if the lane was not a Bot chat. `bot.changed` has no `session_id`.

### POST /v1/sessions/{id}/archive

Auth: any key
Effect: Archives the conversation so it leaves the live list and appears on the archived list. Archiving removes a pin. Archiving a conversation that is already archived changes nothing and publishes no event. A lane handle is rejected even when the conversation does not exist.

Path: `id` session id.

Request: no body.

Response: `200`

```json
{
  "session_id": "2026-10-14T12-04-05_3f2a1b0c-4d5e-6789-abcd-ef0123456789",
  "archived": "true"
}
```

`archived` is the string `"true"`, not a boolean.

Errors:

- `400` this action applies to the whole conversation; use its session id
- `404` session not found
- `409` turn already running (a run slot exists, including a paused one)
- `500` the conversation could not be archived

Events: `sessions.changed` when the conversation was not already archived.

### POST /v1/sessions/{id}/unarchive

Auth: any key
Effect: Returns an archived conversation to the live list. It does not restore a pin. A conversation that was not archived still answers `200` and still publishes `sessions.changed`. A lane handle is rejected even when the conversation does not exist. A running conversation can be unarchived.

Path: `id` session id.

Request: no body.

Response: `200`

```json
{
  "session_id": "2026-10-14T12-04-05_3f2a1b0c-4d5e-6789-abcd-ef0123456789",
  "archived": "false"
}
```

`archived` is the string `"false"`.

Errors:

- `400` this action applies to the whole conversation; use its session id
- `404` session not found
- `500` the conversation could not be restored

Events: `sessions.changed`

### POST /v1/sessions/archived/delete

Auth: any key
Effect: Deletes archived conversations. `older_than_days` of `0`, or an empty body, deletes every archived conversation that can be removed. A positive value deletes those whose `modified` is strictly before now minus that many 24-hour days (`modified` is parsed as a timestamp with a fraction). Each conversation removed publishes `sessions.changed`. Bot conversations are not offered and are not counted. A conversation whose `modified` cannot be parsed, or that is not old enough, is not counted. One that is no longer archived, or whose modification time changed after it was listed, is left and not counted. A running conversation, a pinned one, or a delete that fails increments `skipped`. If storage becomes unavailable the scan stops and the response is still `200` with the counts so far.

Request:

```json
{ "older_than_days": 30 }
```

- `older_than_days` (integer, optional) is `0` to `36500`. Absent or an empty body means `0`. The body is at most 4096 bytes.

Response: `200`

```json
{ "deleted": 2, "skipped": 1, "freed_bytes": 36864 }
```

`freed_bytes` sums the same byte accounting as the archived list's `bytes`.

Errors:

- `400` invalid request
- `400` older_than_days must be between 0 and 36500
- `500` the archived list could not be read

Events: `sessions.changed` once for each conversation removed.

### POST /v1/sessions/{id}/pin

Auth: any key
Effect: Pins the conversation. The list row then includes `pinned_at`. Pinning an already pinned conversation keeps the original time and publishes nothing. A lane handle is rejected even when the conversation does not exist. A running conversation can be pinned. An archived one cannot.

Path: `id` session id.

Request: no body.

Response: `200`

```json
{
  "session_id": "2026-10-14T12-04-05_3f2a1b0c-4d5e-6789-abcd-ef0123456789",
  "pinned": true
}
```

`pinned` is a boolean.

Errors:

- `400` this action applies to the whole conversation; use its session id
- `404` session not found
- `409` archived conversations cannot be pinned
- `500` the pin could not be saved

Events: `sessions.changed` when the conversation was not already pinned.

### POST /v1/sessions/{id}/unpin

Auth: any key
Effect: Removes the pin. Unpinning a conversation that is not pinned publishes nothing. A lane handle is rejected even when the conversation does not exist.

Path: `id` session id.

Request: no body.

Response: `200`

```json
{
  "session_id": "2026-10-14T12-04-05_3f2a1b0c-4d5e-6789-abcd-ef0123456789",
  "pinned": false
}
```

Errors:

- `400` this action applies to the whole conversation; use its session id
- `404` session not found
- `500` the pin could not be removed

Events: `sessions.changed` when the conversation was pinned.

### POST /v1/sessions/{id}/seen

Auth: any key
Effect: Marks this lane seen. It writes only while `unseen` is not `""`, and it is safe to call again.

Path: `id` session id or lane handle.

Request: no body.

Response: `200`

```json
{
  "session_id": "2026-10-14T12-04-05_3f2a1b0c-4d5e-6789-abcd-ef0123456789",
  "changed": true
}
```

`changed` is false when nothing was unseen.

Errors:

- `404` session not found
- `500` the seen mark could not be saved

Events: `sessions.changed` when `changed` is true.

### POST /v1/sessions/seen

Auth: any key
Effect: Marks every main conversation seen, archived ones included, and every finished scheduled-task result read. After it, no list row has `unseen` set and the scheduled tasks' `unread` count is `0`, so a badge built from either reaches zero. Side lanes keep their own marks. Safe to call again. When the scheduler is not running or its storage is unavailable, the conversations are still marked and `runs_read` is `0`.

Request: no body.

Response: `200`

```json
{
  "sessions": ["2026-10-14T12-04-05_3f2a1b0c-4d5e-6789-abcd-ef0123456789"],
  "runs_read": 2
}
```

`sessions` lists the conversations that were unseen and are now seen (`[]` when none). `runs_read` counts the results it marked read.

Errors:

- `500` a seen mark or a result could not be saved; marks written before the failure stay

Events: `sessions.changed` for each conversation in `sessions`; `schedules.changed` when a result was marked read.

### POST /v1/sessions/{id}/fork

Auth: any key
Effect: Copies a prefix of this lane's path into a new main conversation. Message ids are new. Timestamps and metadata are copied. The workspace is not copied. The context mode is copied; if that cannot be stored, the new conversation is removed. The model and title are copied when that copy succeeds; a failure there still returns the new conversation. Snapshots named by kept successful `present_file` results are copied, and missing snapshots are skipped. An external agent that owns the source refuses the fork before the missing-conversation check.

Path: `id` session id or lane handle of the source.

Request:

```json
{ "keep_count": 4 }
```

- `keep_count` (integer, optional) is how many leading records to keep. Omitted or `null` keeps all of them. `0` keeps none. A value past the end keeps all. A negative value is rejected. An empty body is accepted. Other JSON is rejected. Other fields are ignored.

Response: `200`

```json
{
  "session_id": "2026-10-14T12-11-00_aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee",
  "source_session_id": "2026-10-14T12-04-05_3f2a1b0c-4d5e-6789-abcd-ef0123456789~side1"
}
```

`source_session_id` is the id that was given, which may be a lane handle. `session_id` is the new main conversation.

Errors:

- `409` `{Label} owns this conversation history and workspace; start a new conversation to change them`
- `404` session not found
- `400` invalid json
- `400` keep_count must be >= 0
- `500` the fork could not be written, or the context mode could not be stored (the new conversation is then removed)

Events: `sessions.changed` for the new conversation, and a second `sessions.changed` for it when the model and title were copied or there was nothing to copy.

### DELETE /v1/sessions/{id}/messages/{index}

Auth: any key
Effect: Drops this lane's record at `index` and every record after it. Index 0 rewinds the lane to empty. The index in the response is the index that was given, not a new length. An external agent that owns the conversation refuses this before the missing-conversation check. The run slot is taken before the index is parsed, so a running lane is `409` even when the index is bad.

Path: `id` session id or lane handle. `index` is the 0-based index on that lane's current path, the `records` array from `GET /v1/sessions/{id}`.

Request: no body.

Response: `200`

```json
{
  "session_id": "2026-10-14T12-04-05_3f2a1b0c-4d5e-6789-abcd-ef0123456789",
  "index": 2
}
```

Errors:

- `409` `{Label} owns this conversation history and workspace; start a new conversation to change them`
- `404` session not found
- `409` turn already running
- `400` invalid index (not an integer)
- `400` `index N out of bounds (session length M)` (also a negative integer)
- `409` `message N is in the shared history (first M messages)`
- `500` the path could not be updated

Events: `session.messages`

### PUT /v1/sessions/{id}/messages/{index}

Auth: any key
Effect: Replaces the record at `index` and drops every record after it. The replacement is a new record with a new id. The same external-agent, running, bounds, and shared-prefix rules as delete apply. The run slot is taken before the body is read.

Path: `id` session id or lane handle. `index` as on delete.

Request:

```json
{
  "message": {
    "role": "user",
    "content": [{ "type": "text", "text": "Sketch the layout again" }]
  }
}
```

- `message` is a user, assistant, or tool-result message, in the shapes above. `role` is required. User and assistant messages need `content`. A tool result needs `tool_call_id`, `tool_name`, and `content`. A missing or empty-role message is `missing message`. A body that is not that object is `invalid json`.

Response: `200`

```json
{
  "session_id": "2026-10-14T12-04-05_3f2a1b0c-4d5e-6789-abcd-ef0123456789",
  "index": 2
}
```

Errors:

- `409` `{Label} owns this conversation history and workspace; start a new conversation to change them`
- `404` session not found
- `409` turn already running
- `400` invalid index
- `400` invalid json
- `400` missing message
- `400` `index N out of bounds (session length M)`
- `409` `message N is in the shared history (first M messages)`
- `500` the path could not be updated

Events: `session.messages`

### GET /v1/sessions/{id}/settings

Auth: any key
Effect: Returns the model saved for this lane and the model a later turn uses when the turn does not name one. A lane handle reads that lane's settings.

Path: `id` session id or lane handle.

Request: no body.

Response: `200`

```json
{
  "settings": {
    "provider_id": "example-provider",
    "model": "example-model",
    "reasoning_effort": "",
    "speed": "",
    "updated_at": "2026-10-14T12:08:00.000Z",
    "updated_by": "0123456789abcdef01234567",
    "title": "Layout"
  },
  "effective": {
    "provider_id": "example-provider",
    "model": "example-model",
    "reasoning_effort": "",
    "speed": ""
  }
}
```

- `settings` is `null` when no model is saved. A title with no model still returns `null`. When present, `provider_id`, `model`, `reasoning_effort`, `speed`, and `updated_at` are always included. Empty `reasoning_effort` or `speed` means the model's default. `updated_by` is the id of the access key that saved the choice, never the secret, and is omitted when unset. `title` is omitted when unset. `PUT` settings does not change `title`.
- `effective` is the saved choice, else the Settings default, else the first available model. A saved external-agent model on a Bot conversation counts as unset. The four fields are always present; `reasoning_effort` and `speed` may be `""`.
- `channel_model`, when this conversation is a Bot channel chat and a non-external channel model is set, has the same four fields as `effective`. Channel turns use it instead of `settings`. It is omitted otherwise.

Errors:

- `404` session not found
- `500` the settings could not be read

Events: none.

### PUT /v1/sessions/{id}/settings

Auth: any key
Effect: Saves this lane's model. The choice is what a later turn uses when that turn does not name a model. A turn already running is left as it is. Saving the choice that is already stored publishes nothing. The title is kept.

Path: `id` session id or lane handle.

Request:

```json
{
  "provider_id": "example-provider",
  "model": "example-model",
  "reasoning_effort": "",
  "speed": ""
}
```

- `provider_id` and `model` (strings) are required and must name a model that is available now.
- `reasoning_effort` and `speed` (strings, optional) must be empty for a provider model. For an external agent, a non-empty `reasoning_effort` must be one that model offers. `speed` of `""` or `default` is always accepted; any other value must be one of that model's service tier ids.
- The body is at most 4096 bytes. Other fields are ignored.

A conversation an external agent owns can stay on that agent. Pointing it at a provider, or at a different external agent, is refused. A Bot conversation cannot be pointed at an external agent.

Response: `200`, the same object as `GET /v1/sessions/{id}/settings`, including the saved choice.

Errors:

- `404` session not found
- `400` invalid json
- `400` provider_id and model are required
- `400` unknown or unavailable model
- `400` reasoning_effort and speed currently require an external agent model
- `400` unsupported reasoning_effort for this model
- `400` unsupported speed for this model
- `400` Bot conversations run on a configured provider, not an external agent
- `409` `This conversation uses {Label}; start a new conversation to switch agents`
- `500` the agent binding or the settings could not be read or saved

Events: `session.settings` when the saved choice changes.

### GET /v1/sessions/{id}/usage

Auth: any key
Effect: Returns one conversation's token totals and output speed, broken down by the model that answered, computed from its usage records (the same rows as `GET /v1/usage`). A lane handle covers only that lane's rows; a conversation id covers only the main conversation.

Path: `id` session id or lane handle. An unknown id or a conversation that does not exist is `404`.

Request: no body.

Response: `200`

```json
{
  "session_id": "2026-10-17T10-00-00_…",
  "totals": {
    "input_tokens": 41000, "output_tokens": 6900, "cache_read_tokens": 32000, "cache_write_tokens": 0,
    "cost_usd": null, "calls": 19, "timed_calls": 17, "speed_tps": 66.1, "first_token_median_ms": 1100
  },
  "models": [
    {"provider_id": "volc", "model": "glm-5", "purpose": "chat", "fallback": false, "calls": 14, "timed_calls": 13, "output_tokens": 5600, "speed_tps": 81.3, "first_token_median_ms": 1100},
    {"provider_id": "deepseek", "model": "deepseek-v4", "purpose": "chat", "fallback": true, "calls": 3, "timed_calls": 3, "output_tokens": 900, "speed_tps": 40.2, "first_token_median_ms": 2300},
    {"provider_id": "volc", "model": "glm-5-flash", "purpose": "compaction", "fallback": false, "calls": 2, "timed_calls": 0, "output_tokens": 400, "speed_tps": null, "first_token_median_ms": null}
  ],
  "agent": null
}
```

An external-agent conversation (Codex, Grok, Claude):

```json
{
  "session_id": "…",
  "totals": {"input_tokens": 52000, "output_tokens": 6900, "cache_read_tokens": 40000, "cache_write_tokens": 0, "cost_usd": 0.09, "calls": 0, "timed_calls": 0, "speed_tps": null, "first_token_median_ms": null},
  "models": [],
  "agent": {"executor": "claude", "turns": 7, "timed_turns": 7, "output_tokens": 6900, "speed_tps": 22.4, "speed_basis": "api_duration"}
}
```

- A model call is **timed** when it has a first-token time, finished after it, and its usage was reported with output tokens. `speed_tps` is Σ output tokens of timed calls ÷ Σ (stream end − first token) of those calls in seconds, to one decimal; never an average of per-call speeds. `first_token_median_ms` is the median of (first token − request sent) over timed calls. Both are `null` when `timed_calls` is 0, never 0. Untimed calls count in `calls` only.
- `totals` token counts and `cost_usd` cover every row, external-agent turns included. `calls`, `timed_calls`, `speed_tps` and `first_token_median_ms` cover Core's own model calls only. `cost_usd` is `null` when no row carried a cost (Core does not price its own requests; external agents report their own estimate).
- `models` has one row per provider, model, purpose and `fallback` (true when a model of the fallback chain served the call in place of the conversation's model). `purpose` is `chat` for the conversation's own turns; anything else (`compaction`, `memory_review`, a plugin's purpose) is background work on the conversation's behalf. `chat` rows come first, then by `output_tokens` descending. External-agent turns are not listed here.
- `agent` is `null` unless the conversation is bound to an external agent or has turns from one; a bound conversation before its first turn has zero counts. External agents only report per-turn durations, so a turn counts toward `speed_tps` (`timed_turns`) when its usage was reported and it has a duration. `speed_basis` is `api_duration` when every such turn carried the agent's own time in model requests (Claude), else `turn_wall_clock`, the turn's wall-clock time including tools. `speed_tps` is `null` when no turn counts.

Errors:

- `404` session not found
- `500` cannot read usage records
- `500` a binding cannot be read

Events: none. Refetch after a run's `usage` frames or `session.agent_usage`.

### GET /v1/sessions/{id}/workspace

Auth: any key
Effect: Returns the conversation's workspace. A lane handle returns the same binding as the session id. No binding is JSON `null`, not an empty object.

Path: `id` session id or lane handle.

Request: no body.

Response: `200`

```json
null
```

When a workspace is bound, the body is the workspace view above.

Errors:

- `404` session not found
- `500` the workspace could not be read

Events: none.

### PUT /v1/sessions/{id}/workspace

Auth: any key
Effect: Binds or replaces the conversation's workspace, shared by every lane. An external agent refuses the change before the missing-conversation check. The change is refused while a lane of the conversation is running or still has a task.

Path: `id` session id or lane handle.

Request:

```json
{ "project_id": "demo", "cwd": "/projects/demo", "name": "Layout" }
```

The body is at most 65536 bytes. Other fields are ignored.

- `project_id` and `cwd` both empty keeps the existing scratch workspace, or allocates a new one named from `name`.
- `project_id` empty and `cwd` set binds that absolute directory. It must exist, be a directory, and be listable. `kind` becomes `project`, `roots` is that directory, and there is no `project_id`.
- `project_id` set must name a project. `cwd` empty follows the project's primary (`follow_primary` true). A `cwd` must be inside a project directory. A project with no folders uses its library. A directory that cannot be used is `400`.

Response: `200`, the workspace view.

Errors:

- `409` `{Label} owns this conversation history and workspace; start a new conversation to change them`
- `404` session not found
- `409` stop the current turn before changing workspace
- `400` invalid workspace (body is not the object above, or is larger than 65536 bytes)
- `400` the directory or project cannot be bound (`project not found`, `current directory must be inside a project directory`, `invalid project cwd "..."`, `use an absolute directory path on the Core host`, or the library or conversation directory could not be created)

Events: none.

### GET /v1/sessions/{id}/context

Auth: any key
Effect: Returns the epoch of the conversation's built context, shared by every lane. An empty string means no context has been built. An external agent that owns the conversation refuses the read. A failure reading the binding does not.

Path: `id` session id or lane handle.

Request: no body.

Response: `200`

```json
{ "epoch": "" }
```

After a build, `epoch` is a UUID.

Errors:

- `404` session not found
- `409` `{Label} owns its conversation context`

Events: none.

### POST /v1/sessions/{id}/context/reload

Auth: any key
Effect: Rebuilds the conversation's context when `expected_epoch` is still current, and returns the new epoch. The check and the build cover every lane. An empty `expected_epoch` matches a conversation whose context has not been built. An external agent that owns the conversation is refused before the body is read. A failure reading the binding does not refuse the route.

Path: `id` session id or lane handle.

Request:

```json
{ "expected_epoch": "" }
```

- `expected_epoch` (string) is required, including `""`. Missing, `null`, or a non-string is rejected. The body is at most 4096 bytes.

Response: `200`

```json
{ "epoch": "3f2a1b0c-4d5e-6789-abcd-ef0123456789" }
```

Errors:

- `404` session not found
- `409` `{Label} owns its conversation context`
- `400` expected_epoch is required
- `409` stop or finish the current task before reloading context
- `400` the workspace could not be resolved (the error text)
- `409` context changed; reload its state and try again
- `500` the context could not be built

Events: none.

### GET /v1/sessions/{id}/context-mode

Auth: any key
Effect: Returns the conversation's context mode, one value for every lane: the latest stored selection, or `summary` when none has been stored. A turn already running keeps the mode it started with.

Path: `id` session id or lane handle.

Request: no body.

Response: `200`

```json
{ "mode": "summary" }
```

`mode` is `summary` or `window`.

Errors:

- `404` session not found
- `500` invalid stored context mode

Events: none.

### PUT /v1/sessions/{id}/context-mode

Auth: any key
Effect: Stores `summary` or `window` for the whole conversation. The missing-conversation check comes before the external-agent check. A turn already running keeps the mode it started with; the new mode applies to a later turn.

Path: `id` session id or lane handle.

Request:

```json
{ "mode": "window" }
```

- `mode` (string) must be exactly `summary` or `window`. The body is at most 1024 bytes. Other fields are ignored.

Response: `200`

```json
{ "mode": "window" }
```

Errors:

- `404` session not found
- `409` `{Label} owns this conversation history and workspace; start a new conversation to change them`
- `400` mode must be summary or window
- `500` the mode could not be stored or read back

Events: none.

### GET /v1/sessions/{id}/recap

Auth: any key
Effect: Returns this lane's recap while the recap plugin is on and the recap's transcript size still equals `transcript_size` from the transcript route. Otherwise `recap` is `null`. A lane handle reads that lane's recap.

Path: `id` session id or lane handle.

Request: no body.

Response: `200`

```json
{
  "recap": {
    "text": "The layout is sketched.",
    "next": "Add the empty state.",
    "created_at": "2026-10-14T12:20:00.000Z"
  }
}
```

```json
{ "recap": null }
```

`text` is the recap. `next` is the suggested next message and may be `""`. `created_at` is when it was stored. `transcript_size` is not included.

Errors:

- `404` session not found
- `500` the recap could not be read

Events: none.

### PUT /v1/sessions/{id}/recap

Auth: admin
Effect: Stores this lane's recap. An ordinary key is refused before the conversation is looked up. `text` and `next` are trimmed, then clipped to 400 and 200 Unicode code points; a clipped value ends with `…`. `if_transcript_size` must equal the transcript route's current `transcript_size`, including `0` for an empty transcript. The recap can be stored while the recap plugin is off; `GET` then still returns `null` until the plugin is on. `created_at` is set when the recap is stored.

Path: `id` session id or lane handle.

Request:

```json
{
  "text": "The layout is sketched.",
  "next": "Add the empty state.",
  "if_transcript_size": 4096
}
```

- `text` (string) is required and must not be blank after trimming.
- `next` (string, optional) is the suggested next message.
- `if_transcript_size` (integer) is required.
- The body is one object of at most 65536 bytes and may contain only these fields. An unknown field is rejected with the same status as a missing `text`.

Response: `200`

```json
{ "ok": true }
```

Errors:

- `403` administrator key required
- `400` text and if_transcript_size are required
- `404` session not found
- `409` transcript changed
- `500` the recap could not be saved

Events: `session.recap`
