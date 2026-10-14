# Scheduled tasks

Scheduled tasks are plans Core runs on its own clock. A client lists and edits plans, starts or stops an execution, and reads each execution's result. Closing the client does not stop a plan. Core has to be running for a due plan to start.

A plan id and a result id are 32 lowercase hex characters. A conversation id looks like `2026-10-14T12-00-00_01234567-89ab-cdef-0123-456789abcdef`. A message id looks like `msg_01234567-89ab-cdef-0123-456789abcdef`. A project id is a UUID. Timestamps are UTC, with fractional seconds when Core has them.

Every route below needs an access key. Every route answers `503` before doing anything else when the scheduler is not running (`scheduler is not running`) or its storage has failed (`scheduler storage unavailable:` plus the cause). Successful bodies are JSON. A failure uses the usual JSON error object; match on the HTTP status.

A plan is one JSON object. Create and update send the fields below; a saved plan adds the server fields.

- `name` (string, required): 1 to 200 bytes.
- `prompt` (string, required): 1 to 64000 bytes. Core sends this when the plan runs.
- `mode` (string, required): `standalone` or `followup`.
  - `standalone` starts a new conversation for each execution. Set `project_id` to bind a project, or omit it and the executions share one scratch directory. `session_id` must be omitted. The saved plan includes `workspace_session_id` (`schedule-` plus 32 hex characters), the id of that binding. Replacing the plan with the same `project_id` keeps the binding; a different `project_id` allocates a new one.
  - `followup` continues one conversation. `session_id` is required and must name a conversation that exists and whose workspace is available. `project_id` must be omitted; the conversation's project is used. Once that conversation has a saved model, a run uses that model. Until then it uses `provider_id` and `model`. While that conversation is busy the result stays `queued` and the conversation is not interrupted.
- `provider_id` and `model` (strings, optional) are stored as sent. Both empty: each run uses the current default model. A `provider_id` with an empty `model` uses that provider's first active model. A `model` with an empty `provider_id` searches enabled providers. Both set must name an enabled provider's active model. Saving is rejected when that resolution fails. `local-claude` and `local-grok` are accepted only while that agent's unattended use is on. `local-codex` and `local-pi` are rejected.
- `timing` (object, required). `kind` is `once`, `interval`, `daily` or `weekly`.
  - `once`: `at` (timestamp) is required and must be in the future.
  - `interval`: `seconds` (int) is required, from 60 through 31536000. The gap is measured from the time an execution finishes, so a run that overruns does not produce extra runs.
  - `daily` and `weekly`: `time` (`HH:MM`) and `timezone` (IANA name, such as `America/Los_Angeles`) are required. `weekly` also requires `weekdays`, integers `0` (Sunday) through `6`.
  - The timing must have a future occurrence. A non-existent civil time in that timezone is skipped; a repeated civil time runs once.
- `until` (timestamp, optional): a deadline in the future. The first occurrence must not fall after it. Omit it to clear the deadline on update.
- `max_runs` (int, optional): `0` means no limit (the field is then omitted on output). Otherwise `1` to `100000`. Finished executions count, including failed and cancelled ones. A queued execution that pause, complete or delete cancels before it has been picked up does not count.
- `monitor` (bool): always present on output. When true, an execution may record a result as quiet (kept, and not unread). Errors are never quiet.
- `deliver` (string): where a finished result is also sent. `local` means the scheduled list only. `main` is the Bot main chat. Any other token must be one of `deliver_kinds` from the list route (`feishu` and `discord` when that channel is configured). Several targets are comma-separated; Core trims, lowercases and de-duplicates them, and drops `local` when another target remains. The stored value is either `local` or a comma-separated list with no spaces, such as `main,feishu`. `origin` is rejected on these routes. On create, an empty or omitted `deliver` is stored as `local`. On update, an empty or omitted `deliver` keeps the stored value. A plan saved with no `deliver` is returned as `local`.
- `id` (string): plan id. Set by Core.
- `state` (string): `active` (the clock will start it), `paused` or `completed`. A deleted plan is not returned.
- `reason` (string, omitted when empty): why a plan was paused because an execution could not start (the conversation or its workspace is gone, or the access key that last saved the plan was revoked). Update and resume clear it.
- `next_run_at` (timestamp): when the next execution may start. `0001-01-01T00:00:00Z` means there is none. A manual run does not move it. After any other execution finishes, it is recalculated from that finish time.
- `created_at`, `updated_at` (timestamps).
- `completed_runs` (int): executions counted toward `max_runs`.
- `active_run_id` (string, omitted when empty): the result id of the current execution. A plan has at most one. While it is set, update is rejected. At most four executions across all plans are started at once; further due plans wait.

The access key of a successful create or update is the key later executions run as. It is not in the JSON. Revoking it pauses the plan.

A result is one execution. The list route adds `standalone` and `archived` to each result.

- `id`, `job_id`, `name`.
- `mode`: `standalone` or `followup`. Omitted on receipts saved before the field existed.
- `session_id`: the conversation this execution wrote to. Omitted on a standalone result until the conversation exists. `message_id` is the user message that opened the execution (`msg_` plus a UUID), omitted until that message exists.
- `due_at`, `created_at`, `started_at`, `finished_at`. The last two are omitted until they happen.
- `state`: `queued`, `running`, `succeeded`, `failed`, `cancelled`, `interrupted` or `recoverable`. `interrupted` will not be replayed. `recoverable` is waiting to continue the same execution.
- `summary`: the execution's answer, omitted when empty. Longer than 4000 bytes, it is cut to 1000 Unicode code points. The conversation holds the full answer.
- `error`, `warning`: omitted when empty. `warning` means a tool step failed or was blocked.
- `quiet` (bool): kept, and excluded from the unread count. `read` (bool): someone opened it. `manual` (bool): started by the `run` action rather than the clock.
- `recovery`, `cancel_requested`, `complete_plan`, `pause_plan` (bools, omitted when false). `cancel_requested` means a running execution has been told to stop. `complete_plan` means the execution met its stopping condition and, when the plan was `active`, the plan is now `completed`. `pause_plan` means this execution paused the plan; the plan's `reason` is the result's `error`.
- `deliver_to` (string, omitted when nothing is delivered): the plan's targets at the time a `succeeded` non-quiet run, or a `failed` run, finished. Quiet, cancelled and interrupted results omit it. `delivered_at` is when Core claimed that delivery (at most once). `delivery_error` is one or more `target: reason` entries joined by `; `.
- `standalone` (bool): this result's conversation was created for a standalone plan. A receipt with `mode` `standalone` and a `session_id` is standalone. A receipt with no `mode` is standalone only when its plan is standalone, the conversation is not one that plan follows, and no other result shares the conversation. An unknown plan counts as not standalone.
- `archived` (bool): `standalone` is true, the conversation still exists, and it is archived.

`standalone` is what archive uses. Delete removes the conversation only when that is true and the receipt shows the conversation was created for this result: `mode` is `standalone`, or, when `mode` is absent, a user message in the conversation contains `run:` followed by this result id. A followup conversation is left in place.

### GET /v1/schedules

Auth: any key
Effect: Returns every plan that has not been deleted, plus one page of results, the unread count, and the channel kinds a plan may deliver to. Nothing is changed.

Query: `job_id` (string, optional) limits `runs` to that plan. `offset` (int, default 0) skips that many results after the filter; negative and non-numeric values are read as 0, and an offset past the end is clamped to the filtered count. The page size is 100 and is not a parameter.

Request: no body.

Response: `200`

```json
{
  "jobs": [
    {
      "name": "Daily digest",
      "prompt": "Summarise overnight mail.",
      "mode": "standalone",
      "project_id": "3f1a2b4c-5d6e-7890-abcd-ef1234567890",
      "timing": { "kind": "interval", "seconds": 3600 },
      "monitor": false,
      "deliver": "local",
      "id": "a1b2c3d4e5f60718293a4b5c6d7e8f90",
      "state": "active",
      "next_run_at": "2026-10-14T13:00:00Z",
      "created_at": "2026-10-14T12:00:00Z",
      "updated_at": "2026-10-14T12:00:00Z",
      "completed_runs": 0,
      "workspace_session_id": "schedule-b7c6d5e4f3029180736455a4b3c2d1e0"
    }
  ],
  "runs": [
    {
      "id": "0f1e2d3c4b5a69788796a5b4c3d2e1f0",
      "job_id": "a1b2c3d4e5f60718293a4b5c6d7e8f90",
      "name": "Daily digest",
      "mode": "standalone",
      "session_id": "2026-10-14T12-00-00_01234567-89ab-cdef-0123-456789abcdef",
      "message_id": "msg_01234567-89ab-cdef-0123-456789abcdef",
      "due_at": "2026-10-14T12:00:00Z",
      "created_at": "2026-10-14T12:00:00Z",
      "started_at": "2026-10-14T12:00:01Z",
      "finished_at": "2026-10-14T12:00:20Z",
      "state": "succeeded",
      "summary": "Three new papers.",
      "quiet": false,
      "read": false,
      "manual": false,
      "standalone": true,
      "archived": false
    }
  ],
  "total_runs": 1,
  "unread": 1,
  "next_offset": 1,
  "deliver_kinds": ["feishu"]
}
```

- `jobs` is every plan that is not deleted, newest `created_at` first. `job_id` does not filter it. An empty list is `[]`.
- `runs` is the page of results, newest `created_at` first, after the `job_id` filter. Results of a deleted plan stay here. An empty page is `[]`.
- `total_runs` is the filtered count before paging. `next_offset` is the offset of the next page; it equals `total_runs` on the last page, including a request whose offset was already past the end.
- `unread` counts finished results that are not `read` and not `quiet`, across every plan. The filter and the page do not change it.
- `deliver_kinds` is the configured channel kinds in display order, or `[]`. Send one of these, `local`, or `main` in `deliver`.

Errors:

- `503` scheduler is not running, or scheduler storage is unavailable

Events: none.

### POST /v1/schedules

Auth: any key
Effect: Creates one plan in state `active` and returns it. The first `next_run_at` is the next occurrence after the request. No execution is started by this request.

Request:

```json
{
  "name": "Daily digest",
  "prompt": "Summarise overnight mail.",
  "mode": "standalone",
  "project_id": "3f1a2b4c-5d6e-7890-abcd-ef1234567890",
  "timing": { "kind": "interval", "seconds": 3600 },
  "max_runs": 30,
  "until": "2026-12-01T00:00:00Z",
  "deliver": "main,feishu"
}
```

- The body is one JSON object of at most 131072 bytes, the plan fields in the introduction. Unknown fields are ignored.
- `monitor` omitted is false. `deliver` omitted or blank is stored as `local`.
- A `followup` body sends `session_id` and omits `project_id`.

Response: `200`

```json
{
  "name": "Daily digest",
  "prompt": "Summarise overnight mail.",
  "mode": "standalone",
  "project_id": "3f1a2b4c-5d6e-7890-abcd-ef1234567890",
  "timing": { "kind": "interval", "seconds": 3600 },
  "until": "2026-12-01T00:00:00Z",
  "max_runs": 30,
  "monitor": false,
  "deliver": "main,feishu",
  "id": "a1b2c3d4e5f60718293a4b5c6d7e8f90",
  "state": "active",
  "next_run_at": "2026-10-14T13:00:00Z",
  "created_at": "2026-10-14T12:00:00Z",
  "updated_at": "2026-10-14T12:00:00Z",
  "completed_runs": 0,
  "workspace_session_id": "schedule-b7c6d5e4f3029180736455a4b3c2d1e0"
}
```

The body is the saved plan. `max_runs` is omitted when it is 0. `until` is omitted when unset.

Errors:

- `400` invalid schedule (the body is not JSON, or is larger than 131072 bytes)
- `400` name and prompt are required (max 200 / 64000 bytes)
- `400` mode must be standalone or followup
- `400` followup requires a session and inherits its project
- `400` standalone cannot target a session
- `400` max_runs must be 0..100000
- `400` deadline must be in the future
- `400` at is required
- `400` interval must be between 60 seconds and one year
- `400` time must be HH:MM
- `400` timezone is required, or the timezone name is unknown
- `400` weekdays are required
- `400` weekday must be 0..6
- `400` unknown schedule kind
- `400` schedule has no future occurrence
- `400` first run is after deadline
- `400` conversation not found, the project does not exist, or the workspace is unavailable
- `400` unknown or unconfigured model, or the provider cannot be used
- `400` Codex cannot run scheduled tasks, or pi cannot run scheduled tasks
- `400` Claude or Grok unattended use is off (`Claude is not allowed to run unattended. Turn on "Allow unattended use" for Claude in Settings → External agents to let scheduled tasks use it.`; the same sentence with Grok)
- `400` deliver `origin` is only available when the plan is saved from a Bot chat
- `400` unknown delivery target (the message lists `local`, `main` and the configured kinds, or says no channels are configured)
- `503` scheduler is not running, or scheduler storage is unavailable

Events: `schedules.changed`

### PUT /v1/schedules/{job}

Auth: any key
Effect: Replaces one plan's specification and returns the saved plan. `completed_runs` is kept. A `completed` plan becomes `paused`. An `active` or `paused` plan keeps that state. `reason` is cleared and `next_run_at` is recomputed. The access key of this request replaces the key later executions use.

Path: `job` plan id.

Request:

```json
{
  "name": "Daily digest",
  "prompt": "Summarise overnight mail.",
  "mode": "standalone",
  "project_id": "3f1a2b4c-5d6e-7890-abcd-ef1234567890",
  "timing": { "kind": "daily", "time": "09:00", "timezone": "America/Los_Angeles" }
}
```

- The same object as create, at most 131072 bytes. Fields you omit are cleared: `max_runs` becomes unlimited, `until` is removed, `monitor` becomes false, and `provider_id` and `model` become empty. `deliver` omitted or blank keeps the stored value.
- Rejected while `active_run_id` is set. Cancel or wait for that execution first.

Response: `200`

```json
{
  "name": "Daily digest",
  "prompt": "Summarise overnight mail.",
  "mode": "standalone",
  "project_id": "3f1a2b4c-5d6e-7890-abcd-ef1234567890",
  "timing": { "kind": "daily", "time": "09:00", "timezone": "America/Los_Angeles" },
  "monitor": false,
  "deliver": "local",
  "id": "a1b2c3d4e5f60718293a4b5c6d7e8f90",
  "state": "active",
  "next_run_at": "2026-10-15T16:00:00Z",
  "created_at": "2026-10-14T12:00:00Z",
  "updated_at": "2026-10-14T18:00:00Z",
  "completed_runs": 2,
  "workspace_session_id": "schedule-b7c6d5e4f3029180736455a4b3c2d1e0"
}
```

The body is the saved plan.

Errors:

- `400` invalid schedule (the body is not JSON, or is larger than 131072 bytes)
- `400` the same validation failures as create
- `400` schedule not found (`file does not exist`), including a plan already deleted
- `400` a current execution exists (`conversation is busy`)
- `503` scheduler is not running, or scheduler storage is unavailable

Events: `schedules.changed`

### POST /v1/schedules/{job}/{action}

Auth: any key
Effect: Applies one action to a plan. The response is a small JSON object, not the plan. An execution already `running` keeps running after `pause`, `complete` or `delete`.

Path: `job` plan id. `action` is `pause`, `resume`, `run`, `cancel`, `complete` or `delete`. Any other word is an error. `archive-results` is a different route.

Request:

```json
{ "delete_results": true }
```

- `pause` sets `paused`, `complete` sets `completed`, and `delete` hides the plan (`deleted`). A `running` execution continues. A `queued` result with `recovery` true stays queued. A `queued` result that has not been picked up is saved as `cancelled` and `quiet`, and `completed_runs` does not increase. A `queued` result that has been picked up is cancelled; it ends `cancelled`, not quiet, and then counts.
- `resume` sets `active` and clears `reason`. When `next_run_at` is `0001-01-01T00:00:00Z`, it computes a new one.
- `run` queues one execution now. `state` and `next_run_at` stay as they are, including when the plan is `paused` or `completed`. The result's `manual` is true.
- `cancel` stops only the current execution. When that execution has been picked up, the result gains `cancel_requested` and this response returns before `state` is `cancelled`. When it is still queued and not picked up, it is saved as `cancelled` immediately, not quiet, and it counts.
- Only `delete` reads a body, and only `delete_results` (bool, default false). False removes the plan and keeps results. True also removes finished results of this plan, including a queued result this request cancelled before it was picked up. A result already in progress stays. A followup conversation stays. A standalone conversation created for a removed result is removed with it.
- The body is optional and at most 4096 bytes. A missing body, a JSON value that does not decode, or a body over that size leaves `delete_results` false, and the plan is still deleted. Other actions ignore a body.

Response: `200`

```json
{ "ok": true }
```

When `delete` is called with `delete_results` true, the body is counts instead. The plan is already deleted; a result that could not be removed is counted in `failed` and does not undo the delete.

```json
{ "ok": true, "deleted": 2, "conversations_deleted": 1, "failed": 0 }
```

- `deleted` is results removed. `conversations_deleted` is conversations removed with them. `failed` is results that could not be removed.

Errors:

- `409` schedule not found (`file does not exist`), including a plan already deleted
- `409` unknown action
- `409` resume: maximum runs reached; edit the limit first
- `409` resume: deadline has passed
- `409` resume: one-time task already ran; edit its schedule
- `409` run: conversation is busy (an execution is already current)
- `409` run: maximum runs reached
- `409` run: deadline has passed
- `409` cancel: no current execution
- `503` scheduler is not running, or scheduler storage is unavailable

Events: `schedules.changed`, `sessions.changed`

`sessions.changed` is sent when `delete_results` removes a conversation.

### POST /v1/schedules/{job}/archive-results

Auth: any key
Effect: Archives the conversations of one plan's finished standalone results. Results stay on the scheduled list. A conversation that is already archived is skipped. A followup plan archives nothing. One failure is counted and the rest continue.

Path: `job` plan id.

Request: no body.

Response: `200`

```json
{ "archived": 1, "failed": 0 }
```

- `archived` is conversations newly archived. `failed` is conversations that could not be archived.

Errors:

- `404` schedule not found, including a plan already deleted
- `503` scheduler is not running, or scheduler storage is unavailable

Events: `sessions.changed`

Sent when at least one conversation is newly archived.

### POST /v1/schedule-runs/{run}/read

Auth: any key
Effect: Marks one result read, finished or not. The unread count drops when that result was finished and not quiet. Marking it again is a success.

Path: `run` result id.

Request: no body.

Response: `200`

```json
{ "ok": true }
```

Errors:

- `400` result not found (`file does not exist`), or the result could not be saved
- `503` scheduler is not running, or scheduler storage is unavailable

Events: `schedules.changed`

### POST /v1/schedule-runs/{run}/archive

Auth: any key
Effect: Archives the conversation of one standalone result. The result stays. Archiving a conversation that is already archived is a success and changes nothing.

Path: `run` result id.

Request: no body.

Response: `200`

```json
{ "ok": true }
```

Errors:

- `404` result not found
- `400` this result lives in a conversation of its own; only standalone results can be archived
- `404` session not found (the conversation is already gone)
- `409` turn already running
- `500` the conversation could not be archived
- `503` scheduler is not running, or scheduler storage is unavailable

Events: `sessions.changed`

Sent when the conversation was not already archived.

### DELETE /v1/schedule-runs/{run}

Auth: any key
Effect: Removes one finished result. When the result is standalone and the conversation was created for it, the conversation is removed too and `conversation_deleted` is true. A followup result, a standalone conversation Core did not create for this result, or a conversation that is already gone removes only the result, and `conversation_deleted` is false. A result that has not finished is left in place.

Path: `run` result id.

Request: no body.

Response: `200`

```json
{ "ok": true, "conversation_deleted": true }
```

Errors:

- `404` result not found
- `409` execution has not finished
- `409` turn already running
- `409` side chats still branch from this conversation (`delete the side chats branched from this chat first`)
- `500` the conversation could not be removed; the result stays
- `409` the result could not be removed after its conversation was removed
- `503` scheduler is not running, or scheduler storage is unavailable

Events: `schedules.changed`, `sessions.changed`

`sessions.changed` is sent when the conversation is removed.

### POST /v1/schedule-runs/clear-read

Auth: any key
Effect: Removes finished results that are read or quiet, and removes a standalone conversation created for such a result. Unread results that are not quiet stay. Unfinished results stay. One failure is counted and the rest continue. The response is always success when the scheduler is up.

Request:

```json
{ "job_id": "a1b2c3d4e5f60718293a4b5c6d7e8f90" }
```

- `job_id` (string, optional) limits the removal to one plan. Omit it, or send `{}`, to include every plan. An unknown id removes nothing.
- The body is optional and at most 4096 bytes. A missing body, a body that does not decode, or a body over that size is treated as no `job_id`, so every plan is included.

Response: `200`

```json
{ "deleted": 2, "conversations_deleted": 1, "failed": 0 }
```

- `deleted` is results removed. `conversations_deleted` is conversations removed with them. `failed` is results that could not be removed. All three are present when they are zero.

Errors:

- `503` scheduler is not running, or scheduler storage is unavailable

Events: `schedules.changed`, `sessions.changed`

`sessions.changed` is sent when a conversation is removed. When nothing is removed, no event is sent.
