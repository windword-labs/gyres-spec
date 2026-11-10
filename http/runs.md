# Runs: sending, streaming, stopping

These routes start a turn, watch it, stop it, and change what is waiting to be sent. A turn belongs to Core: closing the HTTP response does not stop it. `POST /v1/sessions/{id}/turns` is the stream for the client that sent the turn. Anyone else, and that client after a dropped connection, attaches with `GET /v1/sessions/{id}/events`. Stop only with `POST /v1/sessions/{id}/cancel`. Revoking the access key that started a live turn stops it. Replacing that key does not.

`{id}` is a conversation handle. A session id is its main lane: a UTC timestamp `YYYY-MM-DDTHH-MM-SS`, then `_`, then a UUID, for example `2026-10-14T12-04-05_018f3a2b-7c4d-7e5f-a6b7-8c9d0e1f2a3b`. Another lane is that id, `~`, and a lane name (`draft` gives `…a3b~draft`). A lane name is 1 to 48 characters, starts with a letter or digit, then letters, digits, `_` or `-`, and is never `main`. Lanes of one session run at the same time. Each has its own turn, queue, goal, questions and run stream. Codex, Grok, pi and Claude run only on the main lane.

JSON responses are `application/json`. A usual error is

```json
{"error":"session not found","failure":{"code":"core_request_failed","message":"session not found","http_status":404,"details":"session not found"}}
```

`error` is the message to show. The three routes that answer `409` with a different body say so below. Timestamps are UTC.

Core-wide notifications (`GET /v1/events`) and run-stream frames (`GET /v1/sessions/{id}/events` and the body of `POST /v1/sessions/{id}/turns`) are specified in [Events](../events.md). This file says how to open, resume and close those streams.

## Run state

`state` on `GET /v1/sessions/{id}/run-state`, and the `state` of a pause or resume:

- `idle` — nothing is running, and no task is waiting to be continued.
- `running` — a turn is in progress.
- `pausing` — pause was requested; work already started is finishing. New work waits.
- `paused` — paused. Continue with `resume` or stop with `cancel`.
- `waiting_children` — running, and the turn is waiting for child tasks.
- `interrupted` — the task stopped and can be continued. A turn that finished with messages still queued is `interrupted`, with `reason` `追加消息已保存，请继续处理`.
- `blocked` — the task cannot continue until the owner acts. `reason` says why.

`reason` is that explanation, or `""`. `reason_code` (Gyres 0.98+) is its stable
machine code; show text localised from the code and fall back to `reason` for
a code you do not know.

The state is derived from the conversation's operation log (what recovery
would do now) plus the live run and the owner's pause; it reads the same after
a reconnect or a Core restart as the stream showed before.

### Reason codes

One code per decision; a code never changes meaning. `state` is where the
code appears; `next` is the recovery action (`GET …/run-state`).

| Code | State | Meaning |
|---|---|---|
| `tool_unknown` | blocked | A call that must not run twice was started and its result was not saved. Check what it did, then continue. |
| `prompt_missing` | blocked | The task's prompt was edited or deleted. |
| `step_budget` | blocked | The step used its attempts across retries and restarts. |
| `local_agent` | blocked | An external agent's dispatch outcome is unknown: after a crash a Cursor task, a dsh task, or one whose history was unreadable or whose follow-up was in flight, waits for the owner's Continue. |
| `agent_approval` | blocked | An external agent was waiting for an approval when Core stopped. |
| `op_mismatch` | blocked | The log's open operation is not this task's. |
| `log_corrupt` | blocked | The operation log is invalid or unreadable. |
| `hook_unknown` | blocked | A hook's outcome is unknown. |
| `key_revoked` | blocked | The key that created the task was revoked. |
| `peer_question` | blocked | A peer's question is not continued after a restart. |
| `start_failed` | blocked | The run could not start (configuration, storage). |
| `resume_failed` | blocked | The recovered run could not start. |
| `internal_error` | blocked | The run stopped on an internal error. |
| `child_failed` | blocked | A child task could not be resumed. |
| `schedule_failed` | blocked | A scheduled run could not be resumed. |
| `external_blocked` | blocked | An external agent's run stopped for review. |
| `storage_failed` | blocked | Task storage failed a write; nothing advances. |
| `tool_changed` | blocked | After a restart, a tool with an unfinished call is gone, or its replay policy or input schema differs from the interrupted run's (`context.jsonl`). Other context changes resume. Continue proceeds. |
| `unspecified` | blocked | A task blocked before reason codes existed. |
| `paused` | paused | The owner paused the run. |
| `storage_recovered` | paused | Storage failed and recovered; continue by hand. |
| `auto_resume_off` | interrupted | Automatic recovery is off; continue by hand. |
| `queued_pending` | interrupted | The reply is saved and queued messages wait. |
| `resume_step` | interrupted | Recovery continues at the saved step. |
| `resume_calls` | interrupted | Recovery continues; started calls without a result are run again or marked interrupted by their tool's policy. |
| `replay_safe` | interrupted | Recovery runs again calls that are safe to repeat. |
| `interrupted_calls` | interrupted | Started calls that must not repeat get an interrupted result. |
| `queued_delivered` | interrupted | A saved reply is followed by queued messages, which run. |
| `child_redelivered` | interrupted | A saved reply is followed by child results not yet delivered. |
| `planned_restart` | interrupted | After a planned restart or update; no attempt is charged. |
| `manual_continue` | (record only) | The owner pressed Continue. |
| `parent_resume` | (record only) | A child resumed by its parent after a restart. |
| `settled`, `abort_settled`, `output_truncated`, `log_missing` | interrupted | The log already settles the task; nothing runs. |
| `stopped`, `parent_finished`, `conversation_gone`, `lane_closed`, `schedule_cancelled` | (settled receipts) | Why a task was cancelled. |

`operation_resumed` records carry the resume code (`code`); claim settle
records (`<data>/tasks/claims.jsonl`) carry `claim_started`,
`claim_finished`, `claim_accepted`, `claim_released`, `claim_uncertain` or
`claim_superseded` in `reason`.

## Goal

`status` is `active`, `paused`, `blocked`, `complete`, `usage_limited`, `capped` or `budget_limited`.

An `active` goal keeps starting another turn after a turn that finishes normally, up to the goal continuation cap (`goal_max_continuations` of `GET /v1/task-limits`, 100 by default; `continuations` counts those). The owner stopping a turn leaves the goal as it is. A turn that fails becomes `blocked`, or `usage_limited` when the failure is a provider usage limit. Reaching the cap becomes `capped` with `reason` `<cap> automatic continuations`. `resume`, and an edit that starts the goal again, set `continuations` back to 0. They do not spend one.

`pause` does not stop a turn that is already running. That turn finishes, and no further goal turn starts until `resume`. `clear` removes the goal and also leaves a running turn alone.

A goal may have a token budget (`token_budget`, none by default; set with `set` or `budget` of `POST …/goal`, or `PUT /v1/tasks/{id}/budget`). `tokens_used` is the input plus output tokens of the conversation lane's model requests since the goal was set (cache reads and writes are not counted), as in the [usage records](usage.md); subagent conversations are not counted. Core measures it before every goal turn starts, at each step of a running turn and when a turn ends. An `active` or `paused` goal that has used its budget becomes `budget_limited` with `reason` `used <tokens_used> of its token budget of <token_budget>`: no further goal turn starts, and a turn still running is not stopped but is told to wrap up (its next step gets a wrap-up note once; a turn that just gave its final answer gets none, nor does a goal whose budget was raised again meanwhile). `budget_limited` takes precedence over `paused`: `pause` leaves it as it is. `resume` and an edit do not start it again while the budget is used up (`resume` answers `409`); raising the budget above `tokens_used`, or clearing it, makes it `active` with the same reset as `resume` and starts a turn when the conversation is idle. Lowering the budget to `tokens_used` or below makes an `active` goal `budget_limited` at once.

The first turn of a newly set goal shows `objective` as the owner's message. Later goal turns are hidden continuation turns.

A goal turn may wait (the model's `wait_goal`): the turn ends with no continuation after it, and the lane's [wake](#wakes) starts the next goal turn later, counted as one continuation. Any `POST …/goal` action cancels the wait.

## Wakes

A wake is one deferred turn of a lane: when it is due, Core starts a turn whose first message is hidden (`internalMessage` `wake`). A lane has at most one pending wake; setting another replaces it. Wakes survive a Core restart. A wake waits while its lane runs and fires when the lane is free. `source` is `goal` (a goal waiting), `plugin:<name>` (a plugin, such as the loop plugin's next check) or `api`. Deleting or archiving the conversation drops its wakes.

A wake object:

```json
{
  "id": "wk_0c1d2e3f405162738495a6b7",
  "session_id": "2026-10-14T12-04-05_018f3a2b-7c4d-7e5f-a6b7-8c9d0e1f2a3b",
  "source": "plugin:loop",
  "due_at": "2026-10-14T12:34:05Z",
  "note": "build 41 running",
  "created_at": "2026-10-14T12:04:05Z"
}
```

`note` is absent when empty. `task_id` is present on a goal's wake: the id of the goal's [task](#get-v1sessionsidtask). Times are UTC.

## Message input

`POST /v1/sessions/{id}/turns` and `POST /v1/sessions/{id}/messages` take the same input fields.

- `prompt` (string). Required unless `attachments` is non-empty. Codex, Grok, pi and Claude also reject a prompt that is only whitespace.
- `attachments` (array of strings, optional). Ids of images already stored for this conversation. Each id is 64 lowercase hex characters. At most 20 images, at most 40 MiB together. A model that does not accept images (`images_supported: false`) is rejected. A model whose support is not confirmed accepts them, but its requests carry a text placeholder instead of each image.
- `references` (array, optional). At most 64. Each object has `type`, `label` (at most 200 Unicode characters; may be `""`), and the fields for its type:
  - `file` or `directory`: `path`, an absolute path on the Core host inside this workspace. A file is a regular file. A directory is a directory.
  - `host_path`: `path`, an absolute path of a file or directory anywhere on the Core host. Optional `kind` is `file` or `directory` and must match. Core sets `outside_workspace`.
  - `skill`: `path`, the absolute path of a skill available in this workspace. Core replaces `label` with the skill's name.
  - `session`: `session_id` of a conversation that exists.
- `input_placements` (array, optional). At most 84. Each object is `marker` (string, 1 to 256 characters, appearing exactly once in `prompt`), `type` (`reference` or `image`) and `index` (which reference or attachment). The marker is not sent as text. It is where that reference or image sits. Anything not listed is appended after the prompt.

## Pending list

`GET`, `PATCH` and `POST …/pending-messages/…` return:

```json
{
  "messages": [
    {
      "id": "queued_0123456789abcdef0123456789abcdef",
      "prompt": "Also check the tests",
      "mode": "steer",
      "editable": true
    }
  ],
  "executor": "native",
  "actions": {"edit": true, "front": true, "insert": true, "send_now": true},
  "insert_note": ""
}
```

- `messages` is an array, empty when nothing is waiting. Order is delivery order.
- `id` is the id sent with the message, or `queued_` plus 32 hex characters when the client omitted one.
- `prompt` is the message text.
- `mode` is `steer` or `queue`. `steer` can enter at the next step. `queue` waits until the current work takes a follow-up. A Grok turn always stores follow-ups as `queue`.
- `editable` is true only when the message is a single text block. Attachments or references make it false.
- `image_count` (number) is present only when the message has images.
- `delivering` (true) is present only while this message is being handed to the live turn. It cannot be deleted or reordered then.
- `held` (true) and `held_until` (UTC, to the second) are present only while an edit hold is active. See `hold`.
- `not_sent` (true) is present when the task this message was queued into failed or was stopped before sending it. Nothing sends it by itself: the next turn started in the conversation (`POST …/turns`, or `send-now` on it) sends the not-sent messages first, in this order, before its own prompt, as one user message. Goal turns and loop wakes do not take them and are refused with `409` while they wait. Only messages a person sent wait like this; input from the Bot or a plugin ends with its failed task. The list survives a restart.
- `executor` is `native` (Gyres), `codex`, `grok`, `pi`, `claude` or `cursor`.
- `actions.edit` and `actions.front` are true. `actions.insert` and `actions.send_now` are true only while a turn is live (not while it is still preparing). For Grok, `insert` is true only when that turn can take text immediately. For `not_sent` messages `send_now` is true and `insert` false.
- `insert_note` is `""`, or `grok_timing_not_guaranteed` for Grok: Grok decides when inserted text is used.

A hold lasts 2 minutes and is kept only in memory. Another `hold` refreshes it. Delivery does not pass a held message at the front of the queue. `release`, a successful edit, or expiry clears it. A restart drops every hold.

### GET /v1/events

Auth: any key
Effect: Opens the Core-wide notification stream. The first frame is `ready`. The stream then forwards notifications until the client disconnects or Core closes it (Core is reloading). There is no resume cursor: connect again. The server does not read a last-event id. If this client falls behind, a later frame is `resync`; refetch anything you were watching. Frames and fields are in [Events](../events.md).

Request: no body.

Response: `200` `text/event-stream`

`Cache-Control: no-cache` and `X-Accel-Buffering: no` are set. Each notification is one `data:` line of JSON and a blank line. There is no `id:` line. About every 20 seconds the stream sends a comment line `: heartbeat`. Ignore lines that start with `:`.

Errors:

- `503` Core is reloading. Connect again.

Events: none.

### GET /v1/sessions/{id}/events

Auth: any key
Effect: Attaches to the turn already running in this conversation, without starting one. A dropped connection continues from the last frame it handled. Closing this response does not stop the turn.

Path: `id` conversation handle.

Query: `stream` and `after`, or `history_cursor`. All optional.

- `stream` is the `X-Gyres-Run-Stream` value from the stream you were reading (16 lowercase hex characters). `after` is the SSE `id` of the last frame you handled, a decimal byte offset. Both are required together. They are accepted only for this turn's own stream and only when `after` is the end of a frame. The server does not read a last-event id header.
- `history_cursor` is the `history_cursor` returned with this conversation's transcript. It resumes an external-agent turn (Codex, Grok, pi, Claude) after that transcript. A Gyres turn does not resume this way.
- `stream` and `after` are tried first. If they do not match, `history_cursor` is tried. Anything else replays the turn from the beginning: a missing or unknown cursor, a cursor from an earlier turn, a turn with no resume marks, or a position that is not on this stream.

Request: no body.

Response: `200` `text/event-stream` while a turn is running.

`Content-Type: text/event-stream`, `Cache-Control: no-cache`, `X-Accel-Buffering: no`. `X-Gyres-Run-Stream` names this turn's stream. `Access-Control-Expose-Headers: X-Gyres-Run-Stream`.

Each frame is a `data:` line of JSON, then an `id:` line (the offset to pass back as `after`), then a blank line. About every 10 seconds the stream sends `: heartbeat`, which has no `id`. If the progress stream breaks, one frame is sent and the response ends:

```json
{"type":"stream_error","error":"progress stream unavailable; reload conversation"}
```

That frame has no `id`. Reload the transcript.

When `stream`/`after` or `history_cursor` resumes, the first frame is `{"type":"run_resume","session_id":"<id>"}` with no `id`. Keep the transcript you already show, then apply only the frames after it. A full replay starts at the beginning of the turn; replace what you show with that baseline before applying later frames. Frame fields are in [Events](../events.md).

The response stays open until the turn finishes or you disconnect. After it ends, read the transcript for the result.

Response: `204` no body, when nothing is running (including a task that is paused, blocked or interrupted but not live).

`X-Gyres-History-Version` is an opaque version of the stored history. `Access-Control-Expose-Headers: X-Gyres-History-Version`. Compare it with `history_version` from the transcript you just loaded. The same value means nothing ran in between.

Response: `409` while the turn is preparing and cannot be observed yet. Retry.

Errors:

- `404` session not found
- `409` turn is preparing; retry observation

Events: none.

### GET /v1/sessions/{id}/run-state

Auth: any key
Effect: Reports whether this conversation is idle, running, paused, or waiting to be continued. Read this after a dropped stream, or when `session.task` says the task changed.

Path: `id` conversation handle.

Request: no body.

Response: `200`

```json
{"state":"blocked","reason":"write 的执行结果未保存……","reason_code":"tool_unknown",
 "phase":"tools","op":"task_…","step":1,"next":"block",
 "calls":[{"id":"call_1","name":"write","status":"started"}]}
```

`state` and `reason` are [Run state](#run-state). `reason` is always present.
Since Gyres 0.98 (additions, all optional):

- `reason_code` — a [reason code](#reason-codes).
- `phase`, `op`, `step` — where the open operation is (`accepted`,
  `between`, `preparing`, `step_open`, `tools`, `final`, `aborting`,
  `external`), its task id and step; only while a task is pending.
- `next` — what recovery would do now when no run is live: `resume`,
  `block`, `wait`, `finish` or `close`.
- `waiting` — what a live run waits for: `approval`, `question` or `children`.
- `calls` — the open step's calls without a saved result: `status`
  `unstarted` (never started) or `started` (started, result not saved), and
  `next` (`execute`, `cancel`, `keep`) when no run is live and it is known before the run resumes.
- `queued`, `children` — queued messages not yet in the transcript, child
  results not yet delivered.

Errors:

- `404` session not found

Events: none.

### GET /v1/sessions/{id}/pending-messages

Auth: any key
Effect: Lists messages waiting to enter the running turn, still waiting on a task that can be continued, or not sent because their task failed or was stopped (`not_sent`). While a turn is preparing, the list is what is already queued and `insert` and `send_now` are false.

Path: `id` conversation handle.

Request: no body.

Response: `200` the [pending list](#pending-list).

Errors:

- `404` session not found
- `500` the waiting list could not be read

Events: none.

### GET /v1/sessions/{id}/goal

Auth: any key
Effect: Returns this conversation's goal, or `null` when it has none. Does not start or change a turn.

Path: `id` conversation handle.

Request: no body.

Response: `200`

```json
{
  "goal": {
    "id": "018f3a2b-7c4d-7e5f-a6b7-8c9d0e1f2a3b",
    "session_id": "2026-10-14T12-04-05_018f3a2b-7c4d-7e5f-a6b7-8c9d0e1f2a3b",
    "objective": "Ship the parser fix",
    "status": "active",
    "provider_id": "openai",
    "model": "gpt-4.1",
    "continuations": 0,
    "created": "2026-10-14T12:04:05Z",
    "updated": "2026-10-14T12:04:05Z"
  }
}
```

```json
{"goal": null, "wake": null}
```

`wake` is the goal's pending wait (a [wake](#wakes) with source `goal`), or `null`.

- `id` is a UUID. `session_id` is the conversation handle.
- `objective` is the text the owner set.
- `status` and `continuations` are [Goal](#goal).
- `provider_id` and `model` are present when the goal was given a model. Its turns use that model.
- `reason` and `stopped_at` are present when the goal stopped continuing (`blocked`, `usage_limited`, `capped`, `budget_limited`). Both are cleared when it is `active` again.
- `token_budget` is present when the goal has a [token budget](#goal). `tokens_used` (always present) is its tokens as Core last measured them; Core 0.119+.
- `blocker` and `blocker_turns` are present while a blocker is being counted. `blocker_this_turn` is true when the turn still running has reported one.
- `created` and `updated` are UTC.

Errors:

- `404` session not found
- `503` goals are unavailable
- `500` the goal could not be read

Events: none.

### GET /v1/sessions/{id}/task

Auth: any key
Effect: Returns the task behind this conversation's goal, or `null` when it has none. A goal is stored as a task: a durable work ticket whose first attempt is this conversation. `GET …/goal` shows the same goal in its own shape. Does not start or change a turn.

Path: `id` conversation handle.

Request: no body.

Response: `200`

```json
{
  "task": {
    "id": "018f3a2b-7c4d-7e5f-a6b7-8c9d0e1f2a3b",
    "objective": "Ship the parser fix",
    "has_completion": true,
    "project_id": "",
    "executor": "gyres",
    "provider_id": "openai",
    "model": "gpt-4.1",
    "cwd": "/Users/me/work/parser",
    "status": "active",
    "budget": {},
    "attempts": [
      {"session": "2026-10-14T12-04-05_018f3a2b-7c4d-7e5f-a6b7-8c9d0e1f2a3b", "lane": "main", "started": "2026-10-14T12:04:05Z"}
    ],
    "initiator": "key_0c1d2e3f",
    "goal": {"continuations": 0},
    "created": "2026-10-14T12:04:05Z",
    "updated": "2026-10-14T12:04:05Z"
  },
  "wake": null
}
```

```json
{"task": null}
```

- `id` is the goal's `id`. `objective` is the completion criterion; `has_completion` is true for a goal.
- `project_id` is the conversation's project, `""` for none. `cwd` is its working directory when the goal was set.
- `executor` is `gyres` (goals run on Core's own loop). `provider_id` and `model` are the goal's model, when it has one.
- `status` is the goal's [status](#goal), or `cancelled` for a cleared goal that had not finished (no longer returned here).
- `budget` has `max_attempts`, `max_iterations`, `max_days` and `daily_tokens` when set; a goal sets only `token_budget`, its [token budget](#goal), when it has one.
- `attempts` lists the conversations (handle split into `session` and `lane`) that worked on the task. An attempt that stopped has `ended` and `outcome` (`replaced`, `cleared`). For a goal, an attempt is a lane; tasks of other kinds (Core 0.114+: loops, Bot tasks) record each run as an attempt with `run`, `tokens` and an `outcome` of `completed`, `failed`, `cancelled` or `interrupted`, and an attempt whose conversation was deleted has `conversation_deleted: true`.
- `initiator` is the id of the access key that set the goal, absent when unknown.
- `goal` holds the goal's `continuations`, `tokens_used` (omitted at 0), `reason`, `stopped_at`, `blocker`, `blocker_turns` and `blocker_this_turn`, as in `GET …/goal`.
- `wake` is the goal's pending wait, as in `GET …/goal`.

Errors:

- `404` session not found
- `500` the task could not be read

Events: none.

### GET /v1/tasks

Auth: any key
Effect: Lists every task (goals, loops, Bot tasks and standalone tasks), newest first. Core 0.114+.

Query: `project_id` (optional) keeps the tasks of that project.

Response: `200` `{"tasks": [ … ]}`, each task as in [`GET …/task`](#get-v1sessionsidtask), with these fields when set:

- `notice`: the needs-attention condition Core last posted to the Bot for this task (`blocked`, `capped`, `usage_limited`, or `approval:` plus the attempt). Core posts each condition once (an approval wait once until it is answered).
- `skips`: triggers that arrived while an attempt was running and were skipped, the newest 20: `{at, source, trigger}`, `source` being `schedule:<plan id>`, `schedule:<plan id> (run now)`, `bot_task` or `owner (advance now)`.
- `reason`: why a standalone task is `capped`, `paused` or `cancelled` (for example `reached its budget of 3 attempt(s)`). `capped_until`: when a standalone task capped by its `daily_tokens` takes attempts again (the next local midnight).
- `tokens_today` (always present): the tokens of the task's attempts started since local midnight, what `daily_tokens` measures. `next_wake`: the task's next wake, when it has one: its lane's goal or loop wake, else the next run of an active plan that advances it.
- `project_name`: the Project's name (absent for global work or a Project that no longer exists). `initiator_name`: the initiator's readable name, the access key's name or, for `schedule:<plan id>`, the plan's. `plans`: the plans whose `task_id` is this task, `{id, name, state}` (standalone tasks).
- A standalone task's attempt also has `source` (what started it: `schedule:<plan id>`, `owner` for a plan's "run now" or the board's advance, `bot_task`) and, for a scheduled attempt, `report` (`notify`, `quiet` or `complete`, what the run reported). Attempts recorded before Core 0.114 have neither.

A plugin page may read these through its hook token: see [Tasks for plugin pages](#tasks-for-plugin-pages). A Bot task's `bot.result` is left out there, and the answer adds `projects`: the registered Projects, `[{id, name}]` (a page cannot read `GET /v1/projects`).

Errors: `403` a hook token outside that grant; `503` tasks are not available; `500` the tasks could not be read.

Events: none.

### GET /v1/tasks/{id}

Auth: any key
Effect: Returns one task, as in `GET /v1/tasks`.

Errors: `404` task not found; `503` tasks are not available.

Events: none.

### POST /v1/tasks

Auth: any key; the Tasks board's [plugin page](#tasks-for-plugin-pages) hook token
Effect: Creates a standalone task: a work ticket advanced only by attempts that a schedule with `task_id` ([Scheduled tasks](schedules.md)) or the Bot's `bot_task send` starts, each in a new conversation. Starts nothing by itself. Core 0.114+.

Request:

```json
{
  "title": "Weekly dependency check",
  "objective": "Keep the parser's dependencies current; open a note when one has a security fix.",
  "has_completion": false,
  "project_id": "parser",
  "cwd": "",
  "provider_id": "",
  "model": "",
  "budget": {"max_attempts": 10}
}
```

- `objective` (required, at most 64000 bytes): what the task is for; with `has_completion` true it is the completion criterion.
- `title` (optional, at most 200 characters): defaults to the objective's first line, cut to 80 characters.
- `project_id` (project id or name) and `cwd` (absolute directory on the Core host) bind the attempts' conversations; both empty uses a scratch directory.
- `provider_id`/`model` as for schedules; an external agent (`local-claude`, …) needs `project_id` or `cwd` and its unattended use on. Empty uses the plan's model (a schedule) or the Bot's task model (`bot_task send`).
- `budget` (each 0 = none): `max_attempts` (0 to 100000): after that many attempts the task becomes `capped`; `max_days` (0 to 365): `capped` once that many days have passed since it was created; `daily_tokens`: `capped` until the next local midnight once its attempts started today used that many tokens (`max_days` and `daily_tokens` Core 0.114+, from Phase 4). `max_iterations` is a loop's and is refused.

Response: `200` the task: `id` (`tk_` plus 16 hex characters), `status` `waiting`, `attempts` `[]`, `initiator` the access key id.

Statuses of a standalone task: `waiting` (between attempts), `active` (an attempt runs), `complete` (a scheduled attempt reported `complete` and `has_completion` is true; without a criterion the report only completes the plan), `capped` (a budget reached), `paused` and `cancelled` (the owner's [controls](#post-v1tasksidaction)). An attempt is `{session, lane, run, started, ended, outcome, tokens}` as for loops; one attempt runs at a time.

When a task becomes `blocked`, `capped` or `usage_limited`, or one of its attempts waits on an approval, Core posts one notice into the Bot main chat and forwards it to the chat channel the owner wrote from most recently (none configured: the main chat only). A `failed` task gets no notice.

Errors: `400` invalid body, missing objective, unknown project, invalid `cwd`, invalid budget, unavailable model or agent; `503` tasks are not available.

Events: none.

### POST /v1/tasks/{id}/{action}

Auth: any key; a [plugin page](#tasks-for-plugin-pages)'s hook token
Effect: The owner's control of a task, `action` being `pause`, `resume`, `advance` (start its next step now) or `cancel`, carried out by the task kind's own driver. Core 0.114+.

- **Goal**: `pause`, `resume` and `cancel` are the goal API's `pause`, `resume` and `clear` in the lane where the goal is current (`409` when it is no longer current anywhere; `resume` also `409` for a `budget_limited` goal whose budget is still used up). `advance` fires a pending `wait_goal` wake now, else starts the next goal turn when the lane is idle (`409` while a run is going or the goal is not active).
- **Loop**: the loop plugin's own `set` (`status` `paused`/`active`) and `stop` tools in the loop's conversation, approved as the owner's control (resolver `owner:tasks`); `cancel` stops the loop (`complete`, reason `stopped by the owner from Tasks`). `advance` moves the pending check to now (a pre-check still runs first); `409` when the loop is not active or has no check pending, or the loop plugin refuses (its text is the error).
- **Bot task**: `pause`, `resume` and `cancel` are `/v1/bot/tasks/{id}/pause|resume|stop` (only while it runs); `advance` is `409`.
- **Standalone**: `pause` sets `paused` (a running attempt finishes) and pauses the active plans that advance it, with reason `its Task is paused`; `resume` makes it `waiting` again (`active` while an attempt still runs), `409` while a budget is still used up, and resumes the plans Core paused for it (reasons starting `its Task is paused` or `its Task is capped: `); `cancel` sets `cancelled`, completes its plans and stops a running attempt; `advance` runs its first active plan now ("run now"), `409` when the task cannot take an attempt, an attempt is running (recorded in `skips` as `owner (advance now)`) or no plan advances it.

Request: no body.

Response: `200` the task as in `GET /v1/tasks/{id}`.

Errors: `400` not applicable to the kind; `403` a hook token outside the grant; `404` task not found or unknown action; `409` refused (the message says why); `503` tasks are not available.

Events: those of the driver (`goal_updated`, `session.wake`, `schedules.changed`, …).

### PUT /v1/tasks/{id}/budget

Auth: any key; a [plugin page](#tasks-for-plugin-pages)'s hook token
Effect: Replaces a task's budget. Core 0.114+.

Request: `{"max_attempts": 0, "max_iterations": 0, "max_days": 0, "daily_tokens": 0, "token_budget": 0}` (unknown fields refused).

- **Standalone**: `max_attempts`, `max_days` and `daily_tokens` as in `POST /v1/tasks` (`max_iterations` and `token_budget` refused). A `waiting` or `capped` task is measured against the new budget at once: a raised budget makes a capped task `waiting` and resumes the plans Core paused for it.
- **Loop**: `max_iterations`, `max_days` and `daily_tokens` through the loop plugin's `set` tool (its own rules: `max_days` 1 to 365); `max_attempts` and `token_budget` refused.
- **Goal**: `token_budget` only (0 or omitted clears it), as the goal API's `budget` action (Core 0.119+; earlier `400`). Any other field is `400`: its continuation cap is the task limits' `goal_max_continuations`.
- **Bot task** (`400`: one run) has none.

Response: `200` the task as in `GET /v1/tasks/{id}`.

Errors: as for `POST /v1/tasks/{id}/{action}`.

Events: as for `POST /v1/tasks/{id}/{action}`.

### Tasks for plugin pages

A hook token reaches `GET /v1/tasks`, `GET /v1/tasks/{id}`, `POST /v1/tasks/{id}/{pause|resume|advance|cancel}` and `PUT /v1/tasks/{id}/budget` only when its invocation is a tool call from the page of a **bundled** plugin (`POST /v1/plugins/{name}/tools/{tool}` without `session_id`, run in the Bot main chat, the owner's click or page load). Any other hook token, including one of the same plugin from a model's call or a panel in another conversation, gets `403`. Only the bundled `tasks` plugin's page (the Tasks board) may also `POST /v1/tasks`; its task's `initiator` is the access key that made the page call. Core 0.114+.

### GET /v1/wakes

Auth: any key
Effect: Lists every pending wake, soonest first. Changes nothing.

Request: no body.

Response: `200`

```json
{"wakes": [{"id": "wk_0c1d2e3f405162738495a6b7", "session_id": "2026-10-14T12-04-05_018f3a2b-7c4d-7e5f-a6b7-8c9d0e1f2a3b", "source": "plugin:loop", "due_at": "2026-10-14T12:34:05Z", "created_at": "2026-10-14T12:04:05Z"}]}
```

Each entry is a [wake](#wakes).

Events: `session.wake` on `GET /v1/events` whenever a lane's wake is set, replaced, cancelled, fired or dropped.

### GET /v1/sessions/{id}/wake

Auth: any key
Effect: Returns the lane's pending wake, or `null`. Changes nothing.

Path: `id` conversation handle.

Request: no body.

Response: `200`

```json
{"wake": null, "running": false}
```

- `wake` is a [wake](#wakes) or `null`.
- `running` is true while a turn of the lane runs.

Errors:

- `404` session not found
- `503` wakes are unavailable

Events: none.

### GET /v1/sessions/{id}/subagents

Auth: any key
Effect: Lists child tasks this conversation dispatched, and names the parent when this conversation itself is a child. Children are separate conversations. Their turns, queues and streams are their own.

Path: `id` conversation handle.

Request: no body.

Response: `200`

```json
{
  "children": [
    {
      "session_id": "child_0123456789abcdef0123456789abcdef",
      "task_id": "task_0123456789abcdef0123456789abcdef",
      "name": "parser",
      "call_id": "call_1",
      "delivered": false,
      "cwd": "/work/parser",
      "state": "running",
      "reason": ""
    }
  ],
  "parent_session": ""
}
```

- `children` is an array, empty when there are none. A child id is `child_` plus 32 hex characters. `task_id` is `task_` plus 32 hex characters.
- `name` is the short name given when it was dispatched. `call_id` is the dispatch id, or `""` when the child was found only from its own task.
- `delivered` is true after the parent has been given the outcome.
- `outcome` is present once delivered, and is the recorded result state (`completed`, `failed`, `cancelled`, or another task state).
- `cwd` is present only when the child's working directory differs from the parent's.
- `state` while the child runs is `running`, `pausing`, `paused` or `waiting_children`; while its task is pending, the child conversation's [run state](#get-v1sessionsidrun-state) (`interrupted`, `paused`, `blocked`), with `reason_code`; otherwise the task state (`completed`, `failed`, `cancelled`). A child whose start was never recorded is `failed` with `reason` `子任务启动记录不完整`. After delivery, `state` is `outcome`, or `completed` when no outcome was stored, and `reason` may be `已交付；后续执行见子会话`.
- `reason` is present when the task has one.
- `result` is present once the child has finished and is not still running. It is the child's assistant text, cut at 16000 characters with a trailing `\n[Result truncated; use session_search with child session ID for full history.]`, plus the task error when there is one. After delivery it is the summary stored at delivery.
- `parent_session` is the parent's conversation id, or `""` when this conversation is not a child.

Errors:

- `404` session not found
- `500` a child's result could not be read

Events: none.

### GET /v1/sessions/{id}/user-questions

Auth: any key
Effect: Lists questions a live turn is waiting on. A new question is also announced as `session.question` on `GET /v1/events`. The list is empty when nothing is waiting. Order is not significant. Questions live only in memory; a restart drops them.

Path: `id` conversation handle.

Request: no body.

Response: `200`

```json
{
  "questions": [
    {
      "id": "018f3a2b-7c4d-7e5f-a6b7-8c9d0e1f2a3c",
      "session_id": "2026-10-14T12-04-05_018f3a2b-7c4d-7e5f-a6b7-8c9d0e1f2a3b",
      "tool_call_id": "call_1",
      "questions": [
        {
          "id": "library",
          "header": "Library",
          "question": "Which HTTP client?",
          "options": [
            {"label": "net/http", "description": "Use the standard library"},
            {"label": "req", "description": "Use a third-party client"}
          ]
        }
      ],
      "created": "2026-10-14T12:04:05Z",
      "deadline": "2026-10-14T12:05:05Z"
    }
  ]
}
```

- `id` is the batch id (a UUID). Pass it as `{qid}` when answering.
- `session_id` is the conversation. `tool_call_id` is the tool call that asked.
- `questions` holds 1 to 3 questions. Each `id` is unique in the batch. `header` and `question` are short text. `options` holds 2 or 3 objects with `label` and `description`. The first option is the one a timeout selects.
- `created` and `deadline` are UTC. The deadline is 60 seconds after `created`. After it, the turn continues with the first option of each question and this batch disappears.

Errors:

- `404` session not found
- `503` ask_user_question is unavailable

Events: none.

### POST /v1/sessions/{id}/turns

Auth: any key (administrator key when this turn uses Codex, Grok, pi or Claude and that agent does not allow non-administrators)
Effect: Starts one turn and returns its run stream. The turn is accepted before the stream is attached. Closing the response does not stop it. Reattach with `GET /v1/sessions/{id}/events`. Without `idempotency_key` the request is not idempotent: after an ambiguous failure, read the conversation before sending again. With one, a repeat of an accepted send starts nothing (below). Sending `provider_id` and `model` stores them as this conversation's model when that choice differs from the saved one.

Path: `id` conversation handle.

Request:

```json
{
  "prompt": "Fix the parser and add a test",
  "provider_id": "openai",
  "model": "gpt-4.1",
  "cwd": "/work/app",
  "attachments": ["0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef"],
  "references": [{"type": "file", "path": "/work/app/parser.go", "label": "parser"}],
  "input_placements": [{"marker": "@parser", "type": "reference", "index": 0}],
  "idempotency_key": "send-0f3c9a2e"
}
```

- Message fields are [Message input](#message-input). `prompt` may be `""` when `attachments` is non-empty.
- `provider_id` and `model` (strings, optional). Omit both to use this conversation's model, then the settings default, then the first available model. If either is sent, that pair is used as given. A conversation whose saved model is no longer available fails until another model is chosen.
- `reasoning_effort` and `service_tier` (strings, optional). Empty means the model's default. `service_tier` applies only to Codex: it is rejected for Grok, pi, Claude and provider models, and must be one Codex offers (`default` is accepted). For a provider model, a `reasoning_effort` not among its `reasoning_efforts` is dropped and the turn runs on the model's default.
- `cwd` (string, optional) is the working directory for this turn. Empty uses the conversation's workspace.
- `system_prompt` (string, optional) replaces the default style for a provider model. A blank value keeps the default. A Bot conversation ignores it. Codex, Grok, pi and Claude reject a non-empty value.
- `context_settings` (object, optional) is rejected for Codex, Grok, pi and Claude. For a provider model, `context_window` is an integer from 4096 to 10000000. `output_reserve`, `keep_recent_tokens`, `summary_max_tokens` and `threshold_tokens` are optional integers; omitted or 0 uses 4096, 20000, 2048, and `context_window - output_reserve`. The summary must fit in the output reserve, recent messages must fit under the threshold, and the threshold must leave the output reserve inside the window.
- `max_turns` (integer, optional) caps steps of this task. When a global cap is enabled, a missing or higher value is lowered to that cap. A lower value is kept.
- `idempotency_key` (string, optional) is chosen by the client, one per send, at most 128 bytes; Core keeps it in the turn's task record. When the conversation's latest task record carries the same key and was updated within the last 10 minutes, nothing new starts (the body is still decoded, so invalid JSON is `400`, but only the key is compared, not the prompt or other fields): while that run is live the response is its run stream (`200`, from the snapshot, as for the first POST); otherwise `409` `duplicate_request` (below). A different key, or none, is a new send. The key is ignored when task records are not kept. Retry a POST whose connection failed before any answer with the same key.

Response: `200` `text/event-stream`

Same headers, heartbeats and frames as `GET /v1/sessions/{id}/events`. This response always starts at the beginning of the turn. Keep `X-Gyres-Run-Stream` and the last `id` if you need to reconnect. The stream stays open until the turn finishes.

Errors:

- `400` invalid json
- `400` `idempotency_key is longer than 128 bytes`
- `400` missing prompt
- `400` the prompt, references, attachments or placements were rejected. Stable messages include `unknown reference type`, `at most 64 references`, `reference label too long`, `too many input placements`, `invalid or duplicated input marker`, `invalid reference placement`, `invalid image placement`, `invalid input placement type`, `A message can contain at most 20 images`, `Images in one message exceed the 40 MiB limit`, `Invalid attachment ID`, `Image attachment not found`, `The selected model does not support images; remove the images or choose another model`
- `400` unknown or unconfigured model, or `missing endpoint or model`
- `400` `This conversation's model setting (<model>) is no longer available. Choose another model.`
- `400` `Response speed currently requires an external agent` (`service_tier` on a provider model)
- `400` `context_window must be between 4096 and 10000000`, or `invalid context budgets: reserve output and leave room for summary plus recent messages`
- `400` invalid project cwd, when `cwd` cannot be used
- `400` `Bot conversations run on a configured provider, not an external agent`
- `400` `<Label> cannot run in a branch of a conversation` (Codex, Grok, pi or Claude on a lane)
- `400` `<Label> is disabled in Settings → External agents`
- `400` the agent is not installed or not signed in. Codex: `Codex is unavailable; check External agents settings and run codex login on the Core host`. Grok: `Grok is unavailable; check External agents settings and run grok login on the Core host`. pi: `pi is unavailable; check External agents settings and configure a model in pi on the Core host`. Claude: `Claude is unavailable; check External agents settings and sign in to Claude Code in a terminal on the Core host`
- `400` `unknown Codex model`, `Unsupported Codex reasoning effort`, `Unsupported Codex service tier`, and the same kind of message for Grok (`unknown Grok model`, `Unsupported Grok reasoning effort`, `Grok does not support service tiers`), pi (`unknown pi model`, `Unsupported pi thinking level`, `pi does not support service tiers`) and Claude (`unknown Claude model`, `Unsupported Claude effort level`, `Claude does not support service tiers`)
- `400` `<Label> manages its own instructions and context` when `system_prompt` or `context_settings` is sent to Codex, Grok, pi or Claude
- `403` `Ask an administrator to turn on "Allow non-administrators" for <Label> in Settings → External agents.`
- `404` session not found
- `409` `This conversation uses <Label>; start a new conversation to switch agents`
- `409` `<Label> conversation workspace cannot change`
- `409` `Start a new conversation to use Codex` (or Grok, pi, Claude) when that agent is asked to take a conversation that already has history
- `409` `子会话请通过父任务追加工作` — this conversation is a child; send further work from the parent
- `409` `任务尚未完成，请先继续或停止` — a task is still open
- `409` `turn already running`, or for an external agent `turn already running or awaiting recovery`
- `409` the task is waiting and nothing is live. The body is not the usual error envelope:

```json
{"error":"Gyres is waiting to be continued; continue or stop it first","code":"awaiting_continuation","agent":"Gyres","state":"interrupted"}
```

`agent` is `Gyres`, `Codex`, `Grok`, `pi` or `Claude`. `state` is the [run state](#run-state).

- `409` stop won before the turn was accepted. Nothing was recorded. The body is not the usual error envelope:

```json
{"error":"stopped before the turn started","code":"turn_stopped"}
```

- `409` the same `idempotency_key` was already accepted and its run is no longer live. Nothing was started. The body is not the usual error envelope:

```json
{"error":"this request was already accepted","code":"duplicate_request","task_id":"task_0123456789abcdef0123456789abcdef","state":"completed"}
```

`task_id` is the task that request started and `state` its task state. Read the conversation instead of sending again.

- `500` the turn could not be saved or the progress stream could not be created (`cannot create progress stream`, `cannot read task limits: …`)
- `503` `storage unavailable; reconnect and retry`, or `Core is moving its data directory; try again when it finishes`

Events: `session.run_started`, `session.run_finished`, `session.messages`, `session.task`. Naming a model that changes the conversation's setting also emits `session.settings`.

### POST /v1/sessions/{id}/cancel

Auth: any key
Effect: Stops the live turn, or marks a task that is waiting (paused, interrupted, blocked, or still running with no live turn) as cancelled. Child tasks of this turn are stopped too. When the turn belongs to a scheduled task, that occurrence is cancelled. The response can arrive before the stream has closed. A second stop, when nothing is left, conflicts.

Path: `id` conversation handle.

Request: no body.

Response: `200`

```json
{"session_id":"2026-10-14T12-04-05_018f3a2b-7c4d-7e5f-a6b7-8c9d0e1f2a3b","canceled":true}
```

`canceled` is always true on success. Follow `run-state` or the run stream until the live turn has left; then `state` is `idle`.

Errors:

- `404` session not found
- `409` no active turn
- `500` `task stopped, but cancellation could not be saved: …` — the turn was still stopped

Events: `session.task`, `session.run_finished`. A child that was stopped emits `session.task` for the child and for this conversation.

### POST /v1/sessions/{id}/{action}

Auth: any key (administrator key to resume a Codex, Grok, pi or Claude conversation when that agent does not allow non-administrators)
Effect: Pauses or resumes this conversation's turn. `pause` stops new work and lets work already started finish (`pausing`, then `paused`). It does not cancel the turn. `resume` continues a paused live turn, or starts a task that is waiting and has no live turn. Children follow the parent's pause. Codex, Grok, pi and Claude have no pause; stop them with `POST …/cancel`.

Path: `id` conversation handle. `action` is `pause` or `resume`. Any other value is `404`.

Request: no body.

Response: `200`

```json
{"state":"paused"}
```

`state` is the [run state](#run-state) after the change: `pausing` or `paused` for pause, `running` when resume opens a live turn. Resuming a task that the log has already finished does not start a turn; `state` is then whatever `GET …/run-state` would report.

Errors:

- `404` session not found
- `404` unknown action
- `403` `Ask an administrator to turn on "Allow non-administrators" for <Label> in Settings → External agents.`
- `409` `<Label> supports Stop, not Gyres safe pause` — `pause` on a Codex, Grok, pi or Claude conversation
- `409` no active task — `pause` or `resume` when nothing is live and there is no task to continue
- `409` task already finished — the live turn ended during the request
- `409` resume refused. Stable messages: `没有可恢复的任务`, `等待手动继续`, `创建任务的访问密钥已撤销，不能恢复`, `scheduler is not running`, `父任务已结束，不能恢复旧子任务`, `请先继续父任务`. A task that cannot be continued safely is left `blocked` and `error` is the reason.
- `500` the new state could not be saved

Events: `session.task`. When this conversation is a child, the parent emits `session.task` too. A resume that starts a turn also emits `session.run_started` and, when it ends, `session.run_finished`.

### POST /v1/sessions/{id}/messages

Auth: any key (administrator key when the conversation uses Codex, Grok, pi or Claude and that agent does not allow non-administrators)
Effect: Queues one message for the turn that is running. It does not start a turn. The same `id` with the same content is accepted again, including after the message was delivered or the turn ended, and does not create a second message. The mode is the Core running-input mode at this moment (`steer` or `queue`), except a Grok turn always queues.

Path: `id` conversation handle.

Request:

```json
{
  "id": "queued_0123456789abcdef0123456789abcdef",
  "prompt": "Also check the tests",
  "attachments": [],
  "references": [],
  "input_placements": []
}
```

- `id` (string, optional). At most 128 characters. Empty is replaced with `queued_` plus 32 hex characters. A retry must send the same id.
- `prompt`, `attachments`, `references` and `input_placements` follow [Message input](#message-input). The body is at most 1 MiB.
- A prompt that is empty or only whitespace is rejected unless `attachments` is non-empty.

Response: `200`

```json
{
  "session_id": "2026-10-14T12-04-05_018f3a2b-7c4d-7e5f-a6b7-8c9d0e1f2a3b",
  "queued": true,
  "id": "queued_0123456789abcdef0123456789abcdef"
}
```

`queued` is true when the message is waiting and when this `id` was already accepted. `id` is the id to use on the pending-message routes. Read `GET …/pending-messages` for `mode`.

Errors:

- `400` invalid queued message — the body is not JSON, is larger than 1 MiB, `id` is longer than 128 characters, or there is no prompt and no attachments
- `400` the prompt, references, attachments, placements or workspace were rejected (same messages as `POST …/turns`)
- `403` `Ask an administrator to turn on "Allow non-administrators" for <Label> in Settings → External agents.`
- `404` session not found
- `409` no active turn ready for input — nothing is running, the turn is still preparing, or it has already been stopped
- `409` message id already used — this `id` was accepted with different content
- `409` task already finished

Events: `session.task`

### PATCH /v1/sessions/{id}/pending-messages/{message}

Auth: any key
Effect: Replaces the text of one waiting message. The message must be editable (a single text block). Saving clears an edit hold. A message that is being delivered, or that is no longer waiting, is left unchanged.

Path: `id` conversation handle. `message` the pending message `id`.

Request:

```json
{"prompt": "Also check the parser tests"}
```

- `prompt` (string) is the new text. Required, not only whitespace. The body is at most 1 MiB. Attachments and references cannot be added here.

Response: `200` the [pending list](#pending-list), including this message with the new text.

Errors:

- `400` invalid message text — the body is not JSON, is larger than 1 MiB, or `prompt` is missing or only whitespace
- `400` messages with attachments or references cannot be edited
- `400` the new text or the workspace was rejected
- `404` session not found
- `409` turn is preparing
- `409` message is no longer pending
- `409` message is being delivered
- `500` the new text could not be saved

Events: `session.task`

### DELETE /v1/sessions/{id}/pending-messages/{message}

Auth: any key
Effect: Removes one waiting message so it will not be sent. A message that is being delivered, or that is no longer waiting, is left as it is. An edit hold on it is cleared.

Path: `id` conversation handle. `message` the pending message `id`.

Request: no body.

Response: `200`

```json
{"deleted": true}
```

Errors:

- `404` session not found
- `409` turn is preparing
- `409` message is no longer pending
- `409` message is being delivered
- `500` the removal could not be saved

Events: `session.task`

### POST /v1/sessions/{id}/pending-messages/{message}/{op}

Auth: any key
Effect: Holds, reorders, or sends one waiting message. `{op}` is `hold`, `release`, `front`, `insert` or `send-now`.

- `hold` starts or refreshes a 2-minute edit hold so the message is not delivered. With no live turn, this is a success and changes nothing.
- `release` clears the hold. It succeeds even when the message is already gone.
- `front` moves the message to the front and keeps its `mode`.
- `insert` requires a live turn. On Gyres, Codex, pi and Claude it moves the message to the front and sets `mode` to `steer`, so it enters at the next step without stopping the turn. On Grok it hands the text to the running turn; Grok decides when to use it, and the message leaves the list only once Grok accepts it. Images, or anything that is not text, cannot be inserted into Grok.
- `send-now` on a `not_sent` message, with nothing running, moves it to the front and starts a turn on the conversation's saved model that sends every not-sent message, this one first, and nothing else. The response is the list as it was before that turn took them; follow the turn on `GET …/run` or `GET /v1/events`. A refusal to start (for example `409 turn already running`) leaves the messages listed.
- Otherwise `send-now` requires a live turn. The message moves to the front. On Gyres the current step is asked to stop, without ending the turn, and the message is delivered next. On Codex, Grok, pi and Claude the current turn is interrupted and the message waits at the front as `queue` so it is sent when that interruption lands.

`front`, `insert` and `send-now` are refused while this message is held, or while another message is being delivered.

Path: `id` conversation handle. `message` the pending message `id`. `op` one of `hold`, `release`, `front`, `insert`, `send-now`.

Request: no body.

Response: `200` the [pending list](#pending-list).

Errors:

- `400` Grok can only take text into a running task; send it after the task instead
- `404` session not found
- `404` unknown action
- `409` turn is preparing
- `409` message is no longer pending — not used for `release`, which returns the list
- `409` message is being delivered
- `409` message is being edited — `front`, `insert` or `send-now` on a held message
- `409` another message is being delivered
- `409` no active turn to insert into
- `409` no active turn to stop — `send-now` when nothing is live and the message is not `not_sent`
- `send-now` on a `not_sent` message also returns the errors of `POST …/turns` when the turn cannot start
- `500` the change could not be saved
- `502` Grok did not accept the inserted text. The message stays pending.

Events: `session.task`. A Grok `insert` that Grok accepts also emits `session.messages`.

### POST /v1/sessions/{id}/goal

Auth: any key
Effect: Sets, edits, pauses, resumes or clears this conversation's goal. `set` steers a running turn, or starts one when the conversation is idle. `edit` does the same only when the objective changed and the goal is `active` afterwards, which includes reactivating a finished or stalled goal. `resume` starts a turn when the conversation is idle. The response returns when that turn has been started or refused, not when it finishes. Watch the run stream or `GET /v1/events`. A busy conversation or unavailable storage leaves the goal `active` and starts no turn. Any other refusal to start marks the goal `blocked` with `reason` beginning `the goal turn could not start:`.

Path: `id` conversation handle.

Request:

```json
{
  "action": "set",
  "objective": "Ship the parser fix",
  "replace": false,
  "provider_id": "openai",
  "model": "gpt-4.1",
  "token_budget": 500000
}
```

- `action` (string) is `set`, `edit`, `budget`, `pause`, `resume` or `clear`. Letter case and surrounding whitespace do not matter.
- `objective` (string) is required for `set` and `edit`, after surrounding whitespace is removed. Ignored for the other actions.
- `replace` (boolean) applies to `set`. When an unfinished goal exists, `true` completes it and stores the new one. `false` or omitted answers `409` and leaves the current goal.
- `provider_id` and `model` (strings, optional) apply to `set`. Omit both to use the conversation's model. They are stored on the goal and used for its turns. An external-agent model is rejected.
- `token_budget` (integer, 0 or more) is the goal's [token budget](#goal): optional for `set` (0 or omitted: none), required for `budget`, where 0 clears it. Core 0.119+.

`budget` sets or clears the token budget of the current goal; see [Goal](#goal) for how it starts or stops the goal. `pause` sets `paused` (a `budget_limited` goal stays `budget_limited`). `resume` sets `active`, clears `continuations`, `reason`, `stopped_at` and the blocker fields, and starts a turn when the conversation is idle. `clear` removes the goal. `edit` changes `objective`. A `complete`, `blocked`, `usage_limited` or `capped` goal becomes `active` again, with the same reset as `resume`. A `paused` goal stays `paused` and does not start a turn. An `active` goal whose objective did not change does not start or steer a turn.

Response: `200`

```json
{
  "goal": {
    "id": "018f3a2b-7c4d-7e5f-a6b7-8c9d0e1f2a3b",
    "session_id": "2026-10-14T12-04-05_018f3a2b-7c4d-7e5f-a6b7-8c9d0e1f2a3b",
    "objective": "Ship the parser fix",
    "status": "active",
    "provider_id": "openai",
    "model": "gpt-4.1",
    "continuations": 0,
    "created": "2026-10-14T12:04:05Z",
    "updated": "2026-10-14T12:04:05Z"
  }
}
```

`clear` returns `{"goal": null}`. Every other success returns the goal object. Fields are the same as `GET …/goal`. The object is the goal after this call. A turn that later stops the goal on its own shows up on a later read.

Errors:

- `400` invalid json
- `400` missing objective
- `400` missing token_budget — `budget` without it; `token_budget must be 0 or more`
- `400` unknown action
- `400` `<Label> does not use Gyres goals` when `provider_id` is Codex, Grok, pi or Claude
- `404` session not found
- `404` no goal — `edit`, `budget`, `pause` or `resume` when there is none
- `409` the goal used its token budget — `resume` of a `budget_limited` goal whose budget is still used up
- `409` `<Label> does not use Gyres goals` when this conversation already runs on Codex, Grok, pi or Claude
- `409` an unfinished goal already exists. The body is not the usual error envelope. `goal` is the current goal:

```json
{
  "error": "an unfinished goal already exists",
  "goal": {
    "id": "018f3a2b-7c4d-7e5f-a6b7-8c9d0e1f2a3b",
    "session_id": "2026-10-14T12-04-05_018f3a2b-7c4d-7e5f-a6b7-8c9d0e1f2a3b",
    "objective": "Ship the parser fix",
    "status": "active",
    "continuations": 2,
    "created": "2026-10-14T12:04:05Z",
    "updated": "2026-10-14T12:06:05Z"
  }
}
```

- `500` the goal could not be saved
- `503` goals are unavailable

Events: `session.run_started`, `session.run_finished`, `session.messages`, `session.task` when this starts a turn. Otherwise none.

### PUT /v1/sessions/{id}/wake

Auth: admin
Effect: Sets the lane's wake, replacing a pending one. When it is due Core starts a turn whose first message is `prompt`, hidden. Plugins reach this with their hook token for their own conversation (source `plugin:<name>`); with an access key the source is `api`.

Path: `id` conversation handle.

Request:

```json
{"delay_seconds": 1800, "prompt": "Check whether build 41 finished.", "note": "build 41 running"}
```

- `delay_seconds` (number) or `due_at` (RFC 3339), not both. From now to 8 days ahead.
- `prompt` (string, required, at most 8,000 bytes): the hidden first message of the woken turn.
- `note` (string, optional, at most 1,000 bytes): shown with the wake.

Response: `200`

```json
{"wake": {"id": "wk_0c1d2e3f405162738495a6b7", "session_id": "2026-10-14T12-04-05_018f3a2b-7c4d-7e5f-a6b7-8c9d0e1f2a3b", "source": "api", "due_at": "2026-10-14T12:34:05Z", "note": "build 41 running", "created_at": "2026-10-14T12:04:05Z"}, "replaced": null}
```

`replaced` is the wake it replaced, or `null`.

Errors:

- `400` invalid body, time out of range, missing or too long prompt
- `403` administrator key required
- `404` session not found
- `409` an external agent's conversation cannot be woken
- `503` wakes are unavailable

Events: `session.wake`.

### GET /v1/sessions/{id}/loop

Auth: any key, hook token (own lane)
Effect: Returns the lane's loop (the loop plugin's state, docs/loop.md "State"), kept on the lane's loop task, or `null`. A loop file an older Core kept in the conversation's work directory becomes the task on this first read. Core 0.114+.

Path: `id` conversation handle.

Request: no body.

Response: `200`

```json
{"loop": {"version": 1, "id": "lp_0c1d2e3f40516273", "goal": "CI on main is green", "status": "active", "iteration": 2, "cap": 200, "max_days": 14, "daily_tokens": 5000000, "started_at": "2026-10-14T12:04:05Z", "updated_at": "2026-10-14T12:09:05Z", "next_wake_at": "2026-10-14T12:14:05Z", "wake_id": "wk_0c1d2e3f405162738495a6b7", "last_note": "build 41 running", "last_delay_seconds": 300, "last_run": "task_0c1d2e3f"}, "task_id": "lp_0c1d2e3f40516273"}
```

```json
{"loop": null}
```

- `loop.status` is `active`, `paused`, `stopped`, `capped` or `expired`.
- `task_id` is the loop's task (`GET …/task` shows the lane's goal task only).

Errors:

- `404` session not found
- `500` the loop could not be read
- `503` tasks are unavailable

Events: none.

### PUT /v1/sessions/{id}/loop

Auth: administrator key, hook token (own lane)
Effect: Stores the lane's loop (the loop plugin's write). A loop with a new `id` is a new loop task whose first attempt is the hook token's run; an unfinished loop it replaces ends cancelled. Core 0.114+.

Path: `id` conversation handle.

Request:

```json
{"loop": {"version": 1, "id": "lp_0c1d2e3f40516273", "goal": "CI on main is green", "status": "active", "iteration": 1, "cap": 200, "max_days": 14, "daily_tokens": 5000000, "started_at": "2026-10-14T12:04:05Z", "updated_at": "2026-10-14T12:04:05Z"}}
```

Response: `200` `{"loop": {…}, "task_id": "lp_0c1d2e3f40516273"}` with the stored loop.

Errors:

- `400` no loop, an invalid id, or an id that is another lane's loop
- `403` administrator key required
- `404` session not found
- `500` the loop could not be read
- `503` tasks are unavailable

Events: none.

### DELETE /v1/sessions/{id}/wake

Auth: any key
Effect: Cancels the lane's pending wake.

Path: `id` conversation handle.

Request: no body.

Response: `200`

```json
{"removed": true, "wake": {"id": "wk_0c1d2e3f405162738495a6b7", "session_id": "2026-10-14T12-04-05_018f3a2b-7c4d-7e5f-a6b7-8c9d0e1f2a3b", "source": "plugin:loop", "due_at": "2026-10-14T12:34:05Z", "created_at": "2026-10-14T12:04:05Z"}}
```

`removed` is false and `wake` `null` when there was none.

Errors:

- `404` session not found
- `503` wakes are unavailable

Events: `session.wake` when one was removed.

### POST /v1/sessions/{id}/deliver

Auth: administrator key, hook token (own lane)
Effect: Delivers the conversation's last answer, headed with `heading`, to the Bot's chats in the background (the loop plugin's delivery, docs/loop.md "Delivery to the Bot"): the Bot main chat, or the channel kind of the Bot channel chat it runs in.

Path: `id` conversation handle.

Request:

```json
{"heading": "check 3 — CI on main is green"}
```

Response: `202`

```json
{"target": "main"}
```

Errors:

- `400` heading missing or over 2,000 bytes
- `403` administrator key required
- `404` session not found
- `409` the Bot is not set up

### POST /v1/sessions/{id}/user-questions/{qid}/answer

Auth: any key
Effect: Answers one waiting question batch. The live turn continues with these answers. Answer before `deadline`. After the timeout, or after another client answers, the batch is gone. The value does not have to be one of the option labels. Every question in the batch needs an answer.

Path: `id` conversation handle. `qid` the batch `id` from `GET …/user-questions`.

Request:

```json
{"answers": {"library": "net/http"}}
```

- `answers` (object) maps each question `id` to a non-empty string. Extra ids are rejected. An empty object is rejected.

Response: `200`

```json
{"ok": true}
```

Errors:

- `400` invalid json
- `400` missing answers
- `400` `unknown question "<id>"`, `empty answer for "<id>"`, or `missing answer for "<id>"`
- `404` session not found
- `404` question does not belong to this session
- `409` question not found
- `409` question already resolved
- `503` ask_user_question is unavailable

Events: none.
