# Bot and chat channels

The Bot is one persistent assistant: a profile, a memory file, a main chat, optional side chats, one chat per connected channel kind, and background tasks. These routes are how a frontend reads and changes that state. Ordinary conversations are a different list; a Bot chat (the main chat, a side chat, a channel chat, or a hidden task conversation) is omitted from it. Read chats and tasks from `GET /v1/bot`.

A successful body is `application/json`. An error body is `{"error":"<message>","failure":{"code":"core_request_failed","message":"<message>","http_status":<status>,"details":"<message>"}}`. Channel routes send `Cache-Control: no-store`.

`bot.changed` on `GET /v1/events` has no session id. Refetch `GET /v1/bot` after it, and `GET /v1/bot/memory` when the memory may have changed. It is sent when the profile, chat list, task list, or memory changes, not when a read changes nothing.

## Profile

`GET /v1/bot` and `PUT /v1/bot` return the same object.

- `configured` (bool) is false until the first successful `PUT /v1/bot`, then stays true.
- `name` (string) is `Gyres Bot` until a save. A save stores the trimmed name.
- `description` (string) is built-in delegation guidance until the owner saves one. A stored description that is empty, or exactly `Your persistent local assistant.`, is filled with that guidance once, before any request. A save stores the trimmed description, including `""`, and a later empty save stays empty.
- `main_session_id` (string) is the current main chat. The first read or write that needs it creates the chat.
- `bot_provider_id` and `bot_model` (strings, always present) are the Bot model: the one model of every Bot chat (main chat, side chats, each channel kind's chat, including chats `/new` creates). Both `""` means the Settings default, then the first model Core can use. A Bot chat's own saved model is not used; `PUT /v1/sessions/{id}/settings` in any Bot chat sets this pair. An external-agent pair is ignored as if it were unset. The first Core start after Bot models were merged fills an empty pair once from the main chat's saved model, else from the old `channel_provider_id`/`channel_model`, which are then dropped.
- `bot_reasoning_effort` and `task_reasoning_effort` (strings, always present) are the reasoning effort of the Bot model and of the task model. `""` is the model's default (no effort is sent). A task without a task model takes the Bot model with its effort. The first Core start after efforts were added copies the main chat's saved effort to an empty `bot_reasoning_effort` once, when that saved setting names the Bot model.
- `task_provider_id` and `task_model` (strings, always present) choose the model for a new background task. Both `""` means the Bot model, then Core's default model resolution. A non-empty pair is pinned for every new task: the fallback chain is not used. A pair that names an external agent is ignored at task creation as if it were unset.
- `system_prompt` (string) is the prompt Bot conversations receive: a fixed introduction, then ` Your name is <name>.` when the name is non-empty, then a space and the description when that is non-empty, then a fixed style section. Memory is not part of this string.
- `chats` (array) is every Bot chat, oldest first. See Chats.
- `channel_chats` (object) maps a kind (`feishu`, `discord`) to the session id of the chat that kind talks to now. Kinds with no chat yet are absent. `{}` when none are bound.
- `chat_rotation` and `side_chat_branches` are always true: this Core accepts `POST /v1/bot/chats/{id}/new` and a `from` on `POST /v1/bot/chats`.
- `tasks` (array) is newest first. See Tasks.

A session id is UTC time `YYYY-MM-DDTHH-MM-SS`, then `_`, then a UUID. Example: `2026-10-14T12-02-00_3f1c2a90-7b6e-4d11-8a55-0c1e2d3f4a5b`.

External-agent provider ids, refused on `PUT /v1/bot`, are `local-codex`, `local-grok`, `local-pi`, `local-claude`, and `local-cursor`.

## Chats

Each chat:

- `session_id` (string). For a branched side chat this is `<main session id>~sc_<12 lowercase hex>`, a lane of that main chat. It shares the main chat's history through `from_entry` and then continues alone.
- `title` (string). The main chat's stored title is `Main chat`. A channel chat's title is `Feishu` or `Discord`.
- `created` (string, UTC `YYYY-MM-DDTHH:MM:SSZ`).
- `lane` (bool) is present only when true (a branched side chat).
- `from_entry` (string) is present for a branched side chat: the main-chat message id it branched after.
- `ready` (bool) is present only when true. A branched chat is ready once its model snapshot has been copied. A successful branch returns it true.
- `channel` (string) is present for a chat created for a kind (`feishu` or `discord`). It stays after `/new` points the kind at a newer chat.
- `ended` (string, UTC `YYYY-MM-DDTHH:MM:SSZ`) is present after `/new` moved the main-chat pointer or the kind's binding off this chat. The chat stays listed and usable.
- `running` (bool) is true while a live turn holds the chat, including while that turn is `paused` or `pausing`.
- `run_state` (string) while a live turn holds the chat is `running`, `pausing`, `paused`, or `waiting_children` (`running` that is waiting on child runs). When nothing is live but a task is still pending, it is the pending task's state as [`GET /v1/sessions/{id}/run-state`](runs.md#get-v1sessionsidrun-state) derives it: `paused`, `interrupted`, or `blocked`. Otherwise `idle`.

`settings` is never included. The profile's `chats` always include `running` and `run_state`. `POST /v1/bot/chats` without `from` returns the chat without those two fields.

## Tasks

`POST /v1/bot/tasks` returns a task as stored at creation. `GET /v1/bot` returns the same shape later, with status and result fields filled in.

- `id` (string) is `bot_` plus 32 lowercase hex characters.
- `session_id` (string) is the task's own conversation. A hidden task's conversation is a Bot chat and is omitted from the ordinary conversation list.
- `title` (string). An empty title is stored as `Background task`.
- `prompt` (string).
- `source_session_id` (string) is present when the request named one.
- `provider_id` and `model` (strings) are present when a model was pinned for this task.
- `model_pinned` (bool) is present and true when this task must not use the fallback chain.
- `status` (string) is `queued` in the create response, then `running`. Pause sets `paused`, resume sets `running`, stop sets `cancelled`. A finished run sets `completed` or `failed`.
- `result` (string) is present when `completed`: the last assistant text, or `Task completed. Open the task session for the full transcript.` when that text is empty.
- `error` (string) is present when `failed`. It is the run's error, or `task stopped` when the run was aborted and reported no error. A task already set to `cancelled` is left as `cancelled`.
- `created` and `updated` (strings, UTC `YYYY-MM-DDTHH:MM:SSZ`).
- `visible` (bool) is present and true when the task runs in an ordinary conversation the owner can open. `POST /v1/bot/tasks` does not set it. Those tasks are not created by this route.
- `adopted` (bool) is present and true when the Bot sent into an existing conversation it did not open. `prompt` is then `""`.
- `executor` (string) is present when a visible task runs on an external agent: `codex`, `grok`, `pi`, `claude`, or `cursor`.
- `run_id` (string) is present once the run whose outcome is reported back exists.
- `delivered` (bool) is present and true once that outcome was handed to the requesting chat.
- `report` (bool) is present and true on a hidden task whose outcome is reported back to the requesting chat (every hidden task created since #1507).
- `reply` (object) is present when the request arrived from a channel: `channel` (kind), `chat_id`, and `message_id`. Absent means the request came from the app.

## Memory

`content` is the whole memory file, raw. A missing file is `""`.

A line `<!-- gyres:bot-rules -->` that starts the text or follows a newline, and ends the text or is followed by a newline, splits the file. Text before it, trailing whitespace removed, is the owner's notes. Text after it, trimmed, is the Bot-written rules. With no such line, the whole file is the owner's notes.

Rules are entries joined by a blank line, `---`, and a blank line. An entry the Bot writes is the rule, then a newline, then `— `, then the Core host's local date `YYYY-MM-DD`, then ` · `, then `Web`, `Feishu`, or `Discord`.

`version` is the SHA-256 of the raw file bytes as 64 lowercase hex characters, or `""` when `content` is empty. Send it back on `PUT` to avoid overwriting a newer file.

`changes` is the Bot's own writes, newest first, at most the last 200. An owner save does not add one. Each change:

- `id` (string), 16 lowercase hex characters.
- `at` (string, UTC RFC3339).
- `op` (string): `add`, `replace`, or `remove`.
- `source` (string): `Web`, `Feishu`, or `Discord`.
- `session_id` (string), the conversation the write came from, omitted when empty.
- `old` (string), the full entry before a `replace` or `remove`, omitted otherwise.
- `new` (string), the full entry after an `add` or `replace`, omitted otherwise.
- `reason` (string), omitted when empty.
- `undone_at` (string, UTC RFC3339), omitted until this change is undone. Undone changes stay in the list.

`changes` is always an array. `[]` when there are none.

## Channels

Channel routes are administrator only. Kinds, in list order, are `feishu` and `discord`. At most one connection of each kind. Credentials are never returned: no Feishu app secret and no Discord token.

`GET /v1/bot/channels` and `PUT /v1/bot/channels` return:

- `display.settings`, the stored global display object. Unset fields are omitted, so `{}` means nothing is stored. A field that is present is set, including `false` and `0`.
- `display.builtin`, what an unset field resolves to when the kind has no built-in of its own. See the values below.
- `channels`, one object per kind.
- `feishu`, the same object as the `feishu` element of `channels`.

Each kind object:

- `kind` (`feishu` or `discord`).
- `configured` (bool). False deletes nothing about display: a display override outlives the credentials.
- `enabled` (bool). False when not configured. When configured, false means the credentials are kept and the connection is down.
- `owner_bound` (bool). True after an owner is bound.
- `claim_code` (string, 6 digits, leading zeros kept) is present after a manual setup until the owner claims it. The owner sends a direct message whose text, with surrounding whitespace removed, is exactly this code. The first such sender is the owner. Other messages before that are ignored. After a claim, `owner_bound` is true and `claim_code` is omitted. Scan-to-create binds the scanning user and has no claim code.
- `state` (string) is present while a connection is up: `connecting`, `connected`, `reconnecting`, or `error`. Omitted when the kind is not configured or is switched off. A new connection starts at `connecting`.
- `error` (string) is present when the connection reports one.
- `display` is that kind's display view: `override` (stored kind override, `{}` when none), `inherited` (resolved with no override), and `effective` (resolved with the override). `inherited` and `effective` always include every display field.
- Feishu, once configured: `brand` (`feishu` or `lark`), `app_id`, `bot_name`.
- Feishu, while a scan is in progress or has failed and not been cleared: `registration`. `state` is `waiting` or `failed`. While waiting, `url` is the link to show as a QR code and `expires_at` is when it expires (RFC3339, Core host offset), both omitted until the link is known. On failure, `error` is set and `url` is omitted.
- Discord, once configured: `bot_name`, `application_id`, and `invite_url`. `invite_url` is the link that adds the bot to a server the owner is in, which is what lets them open a direct message. Its query has `client_id` set to the application id, `scope` `bot`, `permissions` `67584`, and `integration_type` `0`. Omitted when there is no application id.

Display fields. A value resolves, first one set wins, from the kind override, the global setting, the kind's built-in, then the global built-in. `tool_progress_command` and `followup_grace_seconds` skip the kind override and the kind built-in.

| Field | JSON type | Allowed | Global built-in | Kind built-in |
| --- | --- | --- | --- | --- |
| `tool_progress` | string | `off`, `new`, `all`, `verbose`, `log` | `all` | Feishu `new`, Discord `all` |
| `tool_progress_grouping` | string | `accumulate`, `separate` | `accumulate` | none |
| `show_reasoning` | bool | | false | false |
| `reasoning_style` | string | `code`, `blockquote`, `subtext` | `code` | Discord `subtext` |
| `interim_assistant_messages` | bool | | true | true |
| `long_running_notifications` | bool | | true | true |
| `cleanup_progress` | bool | | false | none |
| `tool_preview_length` | int | 0 through 100000 | 0 | Feishu and Discord 40 |
| `tool_progress_command` | bool | global only | false | none |
| `busy_input_mode` | string | `interrupt`, `steer`, `queue` | `interrupt` | none |
| `followup_grace_seconds` | number | 0 through 60 | 3 | none |

`tool_progress` is how much tool activity a channel shows: `off` none, `new` when the tool changes, `all` every call, `verbose` every call with its arguments, `log` nothing in the chat. `accumulate` edits one progress message; `separate` sends one message per line. `tool_progress_command` lets the owner send `/verbose` in the channel, which moves that kind's `tool_progress` one step along `off`, `new`, `all`, `verbose`, `log`, then back to `off`, and saves the step as the kind's override. `busy_input_mode` is what a direct message does while that kind's chat is busy: `interrupt` stops the running turn (it waits instead when that chat has subagents or is compacting), `steer` adds the message to the running turn or waits if that is refused, `queue` waits. `followup_grace_seconds` is how long after a turn starts a new message waits instead of interrupting; `0` turns that off.

`display.builtin` is the global built-in column: `tool_progress` `all`, `tool_progress_grouping` `accumulate`, `show_reasoning` false, `reasoning_style` `code`, `interim_assistant_messages` true, `long_running_notifications` true, `cleanup_progress` false, `tool_preview_length` 0, `tool_progress_command` false, `busy_input_mode` `interrupt`, `followup_grace_seconds` 3.

A `PUT` of `display` replaces the whole stored object. A field left out is cleared, not kept. `{"display":{}}` clears it. Unknown fields are rejected.

### GET /v1/bot

Auth: any key
Effect: Returns the Bot profile, creating the main chat the first time it is missing. The first time this process ensures the main chat, it also finishes or drops side chats left half-created or half-deleted.

Request: no body.

Response: `200`

```json
{
  "configured": true,
  "name": "Research Bot",
  "description": "Keeps long-running work moving.",
  "main_session_id": "2026-10-14T12-02-00_3f1c2a90-7b6e-4d11-8a55-0c1e2d3f4a5b",
  "task_provider_id": "",
  "task_model": "",
  "bot_provider_id": "",
  "bot_model": "",
  "bot_reasoning_effort": "",
  "task_reasoning_effort": "",
  "chats": [
    {
      "session_id": "2026-10-14T12-02-00_3f1c2a90-7b6e-4d11-8a55-0c1e2d3f4a5b",
      "title": "Main chat",
      "created": "2026-10-14T12:02:00Z",
      "running": false,
      "run_state": "idle"
    }
  ],
  "channel_chats": {},
  "chat_rotation": true,
  "side_chat_branches": true,
  "tasks": [],
  "system_prompt": "You are the user's persistent local Gyres Bot."
}
```

`system_prompt` above is only the opening sentence. The field is the full assembled prompt: that introduction, then ` Your name is Research Bot.`, a space, the description, then the fixed style section. Before any save, `configured` is false, `name` is `Gyres Bot`, and `description` is the built-in guidance.

Errors:

- `500` the main chat could not be created, or a half-finished side chat could not be repaired

Events: `sessions.changed` and `bot.changed` when this read creates the main chat. `bot.changed` when a repair changes the chat list. A read that changes nothing emits nothing.

### PUT /v1/bot

Auth: any key
Effect: Saves the name and description, and optionally the task model or the Bot model. The description sent here is kept as sent, including empty. Also creates the main chat if it does not exist yet.

Request:

```json
{
  "name": "Research Bot",
  "description": "Keeps long-running work moving.",
  "task_provider_id": "provider-b",
  "task_model": "model-b"
}
```

- `name` (string, required). Trimmed. Empty or whitespace-only is rejected. The body is one JSON object of at most 16384 bytes.
- `description` (string). Trimmed. Omitted is `""` and clears a description already saved. Send the current description back when a save should keep it.
- `task_provider_id` and `task_model` (strings). Omit both to leave the task model unchanged. Send both as `""` to clear it. Send both non-empty to pin it. Any other combination is rejected, including one empty and one non-empty. The same rule applies to `bot_provider_id` and `bot_model`.
- `bot_reasoning_effort` and `task_reasoning_effort` (strings, optional). Omitted keeps the effort while the model stays the same and clears it when the same body changes that model. `""` is the model's default. A non-empty value must be one of the model's `reasoning_efforts` (`GET /v1/models`) and needs that model set.
- A non-empty provider id of `local-codex`, `local-grok`, `local-pi`, `local-claude`, or `local-cursor` is rejected. Other fields are ignored.

Response: `200` the profile, same shape as `GET /v1/bot`, after the save and after the main chat exists.

Errors:

- `400` bot name is required. Also for a body that is not JSON or is larger than 16384 bytes.
- `400` task_provider_id and task_model must be set together
- `400` bot_provider_id and bot_model must be set together
- `400` unknown or unavailable model (a Bot model that is not available now)
- `400` unsupported reasoning_effort for this model
- `400` reasoning_effort needs a model
- `400` Bot conversations run on a configured provider, not an external agent
- `500` the profile could not be saved, or the main chat could not be created. A failure while creating the main chat happens after a successful save.

Events: `bot.changed`. Also `sessions.changed` when this call creates the main chat.

### GET /v1/bot/memory

Auth: any key
Effect: Returns the memory file, its version, and the Bot's change log. Changes nothing.

Request: no body.

Response: `200`

```json
{
  "content": "Prefer concise research notes.",
  "version": "ab3dde22904e89dda9261e2d905489478b3d23c55d3bb8612fcd3200fe0cbd1b",
  "changes": []
}
```

`version` above is the SHA-256 of that exact `content`. An empty file uses `""`. See Memory for `content` and `changes`.

Errors:

- `500` the change log could not be read, including when it is corrupt (`change log is corrupt: …`)

Events: none.

### PUT /v1/bot/memory

Auth: any key
Effect: Replaces the whole memory file with `content`. Does not add a change-log entry. Bot conversations pick the new text up on their next turn.

Request:

```json
{
  "content": "Prefer concise research notes.",
  "version": "ab3dde22904e89dda9261e2d905489478b3d23c55d3bb8612fcd3200fe0cbd1b"
}
```

- `content` (string). Stored exactly, including an empty string. The body is one JSON object of at most 262144 bytes. This save is not limited to 3000 characters; that limit applies to the Bot-written section when the Bot writes it and when an undo would grow it.
- `version` (string, optional). When present, it must equal the current version or the save is refused and the file is left unchanged. Omit it, or send JSON `null`, to save unconditionally. `""` matches only an empty file.
- A client that edits the owner's notes and wants to keep the Bot-written rules must send the marker line and the rules after it as part of `content`.

Response: `200` the memory view, same shape as `GET /v1/bot/memory`, with the new `version`.

Errors:

- `400` invalid json. Also when the body is larger than 262144 bytes.
- `409` Bot memory changed; reload before saving
- `500` the file could not be written, or it was written but the view could not be read back (a corrupt change log). In the second case the new content is already stored and `bot.changed` was already sent.

Events: `bot.changed` after a successful write, including when the following read of the change log fails.

### POST /v1/bot/memory/changes/{id}/undo

Auth: any key
Effect: Reverts one Bot write in the rules section and marks that change undone. The owner's notes are left as they are. Undoing an add removes that entry. Undoing a replace puts the old entry back. Undoing a remove appends the old entry.

Path: `id` change id from `changes`.

Request: no body.

Response: `200` the memory view, same shape as `GET /v1/bot/memory`. `content` and `version` match the reverted file. The undone change stays in `changes` with `undone_at` set.

Errors:

- `404` change not found
- `409` this change was already undone
- `409` cannot undo: a later change touched this rule; undo that one first. A later change touches this one when either entry text (`old` or `new`, when non-empty) equals the other's.
- `409` cannot undo: the rule was changed or removed since. An add or replace whose `new` entry is no longer the only exact copy, and a replace whose `old` entry is already present.
- `409` cannot undo: the rule is already there. A remove whose `old` entry is already present.
- `409` cannot undo: unknown change op
- `409` cannot undo: the Bot-written section would exceed 3000 characters; merge or remove old rules first (replace/remove), then retry
- `500` the file or the change log could not be read or written, including a corrupt change log

Events: `bot.changed` on success.

### POST /v1/bot/chats

Auth: any key
Effect: Opens a side chat. With `from`, the chat is a lane of the current main chat, sharing history through that message. Without `from`, the chat is a new empty conversation of its own.

Request:

```json
{ "title": "Research thread" }
```

```json
{
  "title": "Research thread",
  "from": {
    "session_id": "2026-10-14T12-02-00_3f1c2a90-7b6e-4d11-8a55-0c1e2d3f4a5b",
    "entry_id": "msg_3f1c2a90-7b6e-4d11-8a55-0c1e2d3f4a5b"
  }
}
```

- `title` (string, required). Trimmed. Empty is rejected. The body is one JSON object of at most 4096 bytes.
- `from` (object, optional). Omit it, or send JSON `null`, for an empty side chat. `session_id` must be the current main chat (`main_session_id`), not an earlier main chat. `entry_id` must be non-empty and must be the assistant message that ends a finished turn on the main chat's current path: not inside a turn that is still running, not a turn whose tool call has no result yet, and not a message that is not that final assistant reply.
- Other fields are ignored.

Response: `200`

An empty side chat:

```json
{
  "session_id": "2026-10-14T12-05-00_3f1c2a90-7b6e-4d11-8a55-0c1e2d3f4a5b",
  "title": "Research thread",
  "created": "2026-10-14T12:05:00Z"
}
```

A branched side chat:

```json
{
  "session_id": "2026-10-14T12-02-00_3f1c2a90-7b6e-4d11-8a55-0c1e2d3f4a5b~sc_6f1a2b3c4d5e",
  "title": "Research thread",
  "created": "2026-10-14T12:05:00Z",
  "lane": true,
  "from_entry": "msg_3f1c2a90-7b6e-4d11-8a55-0c1e2d3f4a5b",
  "ready": true,
  "running": false,
  "run_state": "idle"
}
```

`session_id` of a branch is the main session id, `~sc_`, and 12 lowercase hex characters.

Errors:

- `400` chat title is required. Also for a body that is not JSON or is larger than 4096 bytes.
- `400` a side chat branches from an entry of the Bot main chat
- `400` message `<entry_id>` is not in the main conversation
- `409` the main chat is busy; try again
- `409` this turn is still running; branch from an earlier turn
- `409` this turn has not finished; branch from a finished turn
- `409` branch from the end of a turn
- `503` storage unavailable; reconnect and retry
- `503` Core is moving its data directory; try again when it finishes
- `500` the chat could not be created. If copying the branched chat's model setting fails after the chat was recorded, the record stays and the next `GET /v1/bot` finishes it.

Events: an empty side chat emits `sessions.changed` for the new conversation and `bot.changed`. It does not create the main chat. A branch emits `session.messages` for the new lane and `bot.changed`. A branch that has to create the main chat also emits `sessions.changed` and `bot.changed` for that chat.

### POST /v1/bot/chats/{id}/new

Auth: any key
Effect: Sets aside the current main chat, or the chat a channel kind is bound to now, and binds a fresh chat with the same model setting and no history. The old chat stays listed, with `ended` set, and remains usable. Its memory review is queued, immediately if it is idle, otherwise once its current turn finishes. This route does not stop that turn and does not drop messages waiting on a channel.

Path: `id` session id of the current main chat, or of the chat a kind is bound to now (`channel_chats`).

Request: no body.

Response: `200` the new chat, with `running` false and `run_state` `idle`.

```json
{
  "session_id": "2026-10-14T12-06-00_3f1c2a90-7b6e-4d11-8a55-0c1e2d3f4a5b",
  "title": "Main chat",
  "created": "2026-10-14T12:06:00Z",
  "running": false,
  "run_state": "idle"
}
```

A new main chat has title `Main chat` and `main_session_id` moves to it. A new channel chat has title `Feishu` or `Discord`, `channel` `feishu` or `discord`, and `channel_chats` for that kind moves to it. Refetch `GET /v1/bot` for the updated pointers and the old chat's `ended` time.

Errors:

- `409` only the main chat or a channel's current chat can start a new chat. Also when this chat was already rotated by a concurrent `/new`.
- `500` the new chat could not be created, including `the chat changed meanwhile; try again` if the binding moved during the write

Events: `sessions.changed` for the new conversation, and `bot.changed`. Also `sessions.changed` for the main chat when this call creates it.

### DELETE /v1/bot/chats/{id}

Auth: any key
Effect: Deletes a Bot chat other than the current main chat, and drops it from the profile. A branched side chat closes its lane and leaves the main chat's history in place. Deleting the chat a kind is bound to clears that binding; the kind's next message opens a new chat. Undelivered task reports that were aimed at this chat are aimed at the current main chat instead, with no channel reply.

Path: `id` session id of a chat in `chats`, other than the current main chat.

Request: no body.

Response: `200`

```json
{ "session_id": "2026-10-14T12-05-00_3f1c2a90-7b6e-4d11-8a55-0c1e2d3f4a5b" }
```

Errors:

- `404` side chat not found. The id is not a Bot chat, or it is the current main chat. Set the main chat aside with `POST /v1/bot/chats/{id}/new` before deleting the earlier one.
- `404` session not found. The profile still lists the chat but the conversation is already gone. The next `GET /v1/bot` drops that record.
- `409` turn already running
- `409` delete the side chats branched from this chat first: `<titles>`. Titles are comma-separated. This applies to a former main chat that still has branched side chats.
- `500` the conversation or the chat record could not be removed, including `Cannot remove conversation images` after the conversation itself was removed

Events: `sessions.changed` and `bot.changed` on success.

### POST /v1/bot/tasks

Auth: any key
Effect: Creates a hidden background task, records it, and starts it. The task uses the Bot's task model when one is pinned, otherwise the Bot model, otherwise Core's default model resolution. The response is sent while `status` is still `queued`; the run then moves it to `running`. Creates the main chat first when it does not exist yet. When it settles, its outcome is reported once as a Bot turn in `source_session_id` (or the current main chat when that chat is gone); `report: true` on the task marks this, and `delivered` is set once the report turn is admitted.

Request:

```json
{
  "title": "Quick research",
  "prompt": "Return a short result.",
  "SourceSessionID": "2026-10-14T12-02-00_3f1c2a90-7b6e-4d11-8a55-0c1e2d3f4a5b"
}
```

- `title` (string, optional). Trimmed. Empty becomes `Background task`.
- `prompt` (string, required). Trimmed. Empty is rejected. The body is one JSON object of at most 131072 bytes.
- `SourceSessionID` (string, optional). Stored as sent and returned as `source_session_id`. Note the request key is `SourceSessionID` (any letter case), not `source_session_id`.
- Other fields are ignored. This route cannot open a visible conversation.

Response: `202`

```json
{
  "id": "bot_0123456789abcdef0123456789abcdef",
  "session_id": "2026-10-14T12-07-00_3f1c2a90-7b6e-4d11-8a55-0c1e2d3f4a5b",
  "title": "Quick research",
  "prompt": "Return a short result.",
  "source_session_id": "2026-10-14T12-02-00_3f1c2a90-7b6e-4d11-8a55-0c1e2d3f4a5b",
  "provider_id": "provider-b",
  "model": "model-b",
  "model_pinned": true,
  "status": "queued",
  "created": "2026-10-14T12:07:00Z",
  "updated": "2026-10-14T12:07:00Z"
}
```

`provider_id`, `model`, and `model_pinned` are omitted when no task model is pinned. `source_session_id` is omitted when the request did not send one. Refetch `GET /v1/bot` for `running`, `completed`, `failed`, `paused`, and `cancelled`.

Errors:

- `400` task prompt is required. Also for a body that is not JSON or is larger than 131072 bytes.
- `500` the task or its conversation could not be created

Events: `bot.changed` when the task is recorded and again each time its status changes. `sessions.changed` for the new task conversation, and for the main chat when this call creates it. The run then emits `session.run_started`, `session.task`, `session.messages`, and `session.run_finished` on the task conversation.

### POST /v1/bot/tasks/{id}/{action}

Auth: any key
Effect: Pauses, resumes, or stops one background task. Pause and resume require a live run on the task's conversation. Stop cancels a live run, and also cancels a task that is still pending after the live run is gone.

Path: `id` task id. `action` is `pause`, `resume`, or `stop`.

Request: no body.

Response: `200`

```json
{ "ok": true }
```

Pause sets the task's `status` to `paused`. Resume sets it to `running`. Stop sets it to `cancelled`. The chat's `run_state` follows the live run (`pausing` while pause is still finishing in-flight work, then `paused`).

Errors:

- `404` unknown bot task action
- `404` bot task not found
- `409` task is no longer running. Pause or resume without a live run, or stop when nothing is live and no pending task remains. A task still `queued` that has not started its run returns this.
- `500` the run state or the task record could not be saved. The message `bot task not found` here means the task disappeared after the `404` check. If saving the task record fails, pause, resume, or stop has already been applied to the live run.

Events: `bot.changed`. Pause, resume, and stop also emit `session.task` for the task conversation. Stop of a live run also leads to `session.run_finished`.

### GET /v1/bot/channels

Auth: admin
Effect: Lists every channel kind, configured or not, plus the global display settings and each kind's display view. Changes nothing. While a Feishu scan is running, poll this route: the scan does not emit an event.

Request: no body.

Response: `200`

```json
{
  "display": {
    "settings": {},
    "builtin": {
      "tool_progress": "all",
      "tool_progress_grouping": "accumulate",
      "show_reasoning": false,
      "reasoning_style": "code",
      "interim_assistant_messages": true,
      "long_running_notifications": true,
      "cleanup_progress": false,
      "tool_preview_length": 0,
      "tool_progress_command": false,
      "busy_input_mode": "interrupt",
      "followup_grace_seconds": 3
    }
  },
  "channels": [
    {
      "kind": "feishu",
      "configured": false,
      "enabled": false,
      "owner_bound": false,
      "display": {
        "override": {},
        "inherited": {
          "tool_progress": "new",
          "tool_progress_grouping": "accumulate",
          "show_reasoning": false,
          "reasoning_style": "code",
          "interim_assistant_messages": true,
          "long_running_notifications": true,
          "cleanup_progress": false,
          "tool_preview_length": 40,
          "tool_progress_command": false,
          "busy_input_mode": "interrupt",
          "followup_grace_seconds": 3
        },
        "effective": {
          "tool_progress": "new",
          "tool_progress_grouping": "accumulate",
          "show_reasoning": false,
          "reasoning_style": "code",
          "interim_assistant_messages": true,
          "long_running_notifications": true,
          "cleanup_progress": false,
          "tool_preview_length": 40,
          "tool_progress_command": false,
          "busy_input_mode": "interrupt",
          "followup_grace_seconds": 3
        }
      }
    },
    {
      "kind": "discord",
      "configured": false,
      "enabled": false,
      "owner_bound": false,
      "display": {
        "override": {},
        "inherited": {
          "tool_progress": "all",
          "tool_progress_grouping": "accumulate",
          "show_reasoning": false,
          "reasoning_style": "subtext",
          "interim_assistant_messages": true,
          "long_running_notifications": true,
          "cleanup_progress": false,
          "tool_preview_length": 40,
          "tool_progress_command": false,
          "busy_input_mode": "interrupt",
          "followup_grace_seconds": 3
        },
        "effective": {
          "tool_progress": "all",
          "tool_progress_grouping": "accumulate",
          "show_reasoning": false,
          "reasoning_style": "subtext",
          "interim_assistant_messages": true,
          "long_running_notifications": true,
          "cleanup_progress": false,
          "tool_preview_length": 40,
          "tool_progress_command": false,
          "busy_input_mode": "interrupt",
          "followup_grace_seconds": 3
        }
      }
    }
  ],
  "feishu": {
    "kind": "feishu",
    "configured": false,
    "enabled": false,
    "owner_bound": false
  }
}
```

`feishu` is the same object as `channels` for `feishu`, including `display` and, when set, `registration`, `brand`, `app_id`, and `bot_name`. The example omits those repeated fields. A configured Discord object adds `bot_name`, `application_id`, `invite_url`, and, until claimed, `claim_code`. `state` and `error` appear while the connection is up.

Errors: none.

Events: none.

### PUT /v1/bot/channels

Auth: admin
Effect: Replaces the global display settings. Fields left out of `display` are cleared. Kind overrides are unchanged.

Request:

```json
{ "display": { "tool_progress": "verbose", "cleanup_progress": true } }
```

- The body is one JSON object of at most 65536 bytes, and its only field is `display`.
- `display` is one object of the display fields. Unknown fields are rejected. See Channels for types, allowed values, and which fields are global-only. A kind-only restriction does not apply here: `tool_progress_command` and `followup_grace_seconds` may be set.
- `{"display":{}}` clears every global setting.

Response: `200` the same object as `GET /v1/bot/channels`, after the save.

Errors:

- `400` expected {"display": {...}}
- `400` invalid display settings
- `400` tool_progress must be one of off, new, all, verbose, log
- `400` tool_progress_grouping must be one of accumulate, separate
- `400` reasoning_style must be one of code, blockquote, subtext
- `400` tool_preview_length must be between 0 and 100000
- `400` busy_input_mode must be one of interrupt, steer, queue
- `400` followup_grace_seconds must be between 0 and 60
- `500` the settings could not be saved

Events: none.

### PUT /v1/bot/channels/{kind}

Auth: admin
Effect: Sets up a kind that is not configured yet, switches a configured kind on or off, or replaces that kind's display override. A setup and a display change cannot be in the same body. The display override applies whether or not the kind is configured, and it survives deleting the credentials.

Path: `kind` is `feishu` or `discord`.

Request:

Feishu setup, only while Feishu is not configured. Unknown fields are rejected. The body is at most 131072 bytes.

```json
{
  "brand": "feishu",
  "app_id": "cli_example",
  "app_secret": "...",
  "enabled": true
}
```

- `brand` (string, optional). `feishu` or `lark`. Omitted is stored as `feishu`. Lark makes Core's own messages in this channel English; Feishu uses Chinese.
- `app_id` and `app_secret` (strings, required). Trimmed. The secret is stored and never returned.
- `enabled` (bool, optional). Omitted or true connects immediately. False stores the credentials disconnected.

Discord setup, only while Discord is not configured. Unknown fields are rejected.

```json
{ "token": "...", "enabled": true }
```

- `token` (string, required). Surrounding whitespace is removed, and a leading `Bot ` prefix is removed regardless of letter case. The token is stored and never returned. Discord is asked who the bot is; the response then carries `bot_name`, `application_id`, and `invite_url`.
- `enabled` (bool, optional). Same meaning as for Feishu.

Switch, only when that kind is configured. The body is exactly this one field:

```json
{ "enabled": false }
```

`enabled` must be a JSON boolean. Sending the current value is a success and changes nothing.

Display, configured or not. This replaces the kind's whole override. `{}` deletes it. `tool_progress_command` and `followup_grace_seconds` are rejected.

```json
{ "display": { "tool_progress": "off" } }
```

Display beside the switch, only when the kind is configured. The display is saved first, then the switch. The body has exactly these two fields:

```json
{ "enabled": false, "display": { "tool_progress": "new" } }
```

The credential check waits up to 15 seconds. A manual setup returns a new `claim_code`.

Response: `200` that kind's object, the same shape as one element of `channels` from `GET /v1/bot/channels`.

Errors:

- `404` unknown channel
- `404` this channel is not configured. `{"enabled": true}` or `{"enabled": false}` alone, before setup.
- `400` invalid settings request. Not a JSON object, a body larger than 131072 bytes, or a setup body with an unknown field or a wrong JSON type.
- `400` "display" goes alone or beside "enabled" of a configured channel
- `400` enabled must be true or false
- `400` invalid display settings
- `400` tool_progress must be one of off, new, all, verbose, log
- `400` tool_progress_grouping must be one of accumulate, separate
- `400` reasoning_style must be one of code, blockquote, subtext
- `400` tool_preview_length must be between 0 and 100000
- `400` busy_input_mode must be one of interrupt, steer, queue
- `400` followup_grace_seconds must be between 0 and 60
- `400` tool_progress_command is a global setting
- `400` followup_grace_seconds is a global setting
- `400` app_id, app_secret and brand (feishu or lark) are required
- `400` Feishu rejected these credentials: `<reason>`. Credentials and URL query strings are removed from the reason.
- `400` token is required
- `400` Discord rejected this token: `<reason>`
- `409` a Feishu bot is already connected; disconnect it first
- `409` a Discord bot is already connected; disconnect it first
- `500` a claim code could not be created, the credentials or the display could not be saved, or the switch failed. If the display was saved and the switch then failed, the new display is already in place.

Events: none.

### DELETE /v1/bot/channels/{kind}

Auth: admin
Effect: Disconnects that kind and deletes its stored credentials. The app or bot on Feishu, Lark, or Discord is left in place. A Feishu scan in progress is cancelled. The kind's display override is kept.

Path: `kind` is `feishu` or `discord`.

Request: no body.

Response: `200` that kind's object, with `configured` false and `enabled` false. Deleting a kind that was not configured is the same success.

Errors:

- `404` unknown channel
- `500` the credentials could not be deleted

Events: none.

### POST /v1/bot/channels/{kind}/registration

Auth: admin
Effect: Starts Feishu or Lark scan-to-create. Any scan already in progress is cancelled and replaced. The new app uses the Bot's saved name, or `Gyres Bot` when the name is empty, and the description `Your persistent local assistant.` (not the Bot profile description). The owner can edit that name and description on the scan page. On success the scanning user is the owner, `configured` becomes true, and `registration` disappears. Poll `GET /v1/bot/channels` for the link and the result. Discord has no scan.

Path: `kind` is `feishu` for this flow. `discord` is rejected.

Request: no body.

Response: `202` the Feishu channel object, as soon as the QR link is known, the attempt has failed, the attempt has finished, or 10 seconds have passed. The scan itself is abandoned 11 minutes after it starts.

```json
{
  "kind": "feishu",
  "configured": false,
  "enabled": false,
  "owner_bound": false,
  "registration": {
    "state": "waiting",
    "url": "...",
    "expires_at": "2026-10-14T12:12:00+08:00"
  }
}
```

`url` is the platform's link; the example shows it as `...`. Before the link arrives, `registration` is `{"state":"waiting"}`. On failure within the wait, `state` is `failed`, `error` is the reason, and `url` is omitted. If the scan finishes successfully within the wait, the body is the configured channel with no `registration`, `owner_bound` true, and `brand`, `app_id`, and `bot_name` set. `brand` is `lark` when the scanning tenant is Lark, otherwise `feishu`.

Errors:

- `404` unknown channel
- `404` no registration flow for this channel
- `409` a Feishu bot is already connected; disconnect it first
- `503` channels are not running

Events: none.

### DELETE /v1/bot/channels/{kind}/registration

Auth: admin
Effect: Cancels Feishu scan-to-create, if one is in progress, and returns the Feishu channel. Cancelling when nothing is in progress is a success. Discord has no scan.

Path: `kind` is `feishu` for this flow. `discord` is rejected.

Request: no body.

Response: `200` the Feishu channel object, with no `registration`.

Errors:

- `404` unknown channel
- `404` no registration flow for this channel

Events: none.
