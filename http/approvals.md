# Tool approvals

A frontend uses these routes to show tool calls that are waiting for the owner, to approve or reject them, and to read or change the policy and the remembered grants. Core keeps the decision. Closing the page does not approve, reject or cancel a waiting request, and any client with a key can decide it. A call that is read-only, the `subagent` tool, or one of these actions never appears here: `memory` `read`, `schedule` `list` or `report`, `schedule_manage` `list`, `skill_manage` `list` or `read`, `computer_use` `capture`, `list_apps`, `list_windows` or `wait`. The managed browser does not ask. A call that a grant or the policy already allows runs without a new request. The same operation rejected or denied earlier in that run does not ask again.

A failed request uses the JSON error body: `error` is the message named below.

Request `id` and grant `id` are 32 lowercase hex characters. Timestamps are UTC.

`config.mode` is the attended policy: `ask`, `edit` or `allow`. The initial value is `ask`. `config.unattended` is the policy for a call with `unattended` true (a scheduled run), used instead of `mode`: `deny`, `wait` or `allow`. The initial value is `deny`.

- `ask` asks, unless a grant covers the call.
- `edit` does the same, and also allows `write` and `edit` when the target file is inside the conversation's working directory or a workspace root. It does not by itself allow other tools.
- `allow` allows the attended call without asking.
- Unattended `deny` stores the request as `denied` and does not run the call. `reason` is `unattended policy denies operations requiring approval`.
- Unattended `wait` leaves the request `pending` until a client decides it.
- Unattended `allow` runs the unattended call without a request.

A request with `once_only` true is never covered by a mode, a grant or unattended `allow`. An unattended one is denied under every unattended policy, and `reason` is `unattended runs refuse operations only a person may approve; it would have hit:` followed by a newline and `resource`. This is `exec_command` when it would signal a process Core did not start. Clipboard and recording ask unless a grant covers them; `mode` and unattended `allow` do not. An ordinary app asks unless the owner's app mode already allows it. An app reserved to the Core host never creates a request. A blocked app, or an app outside an allow list, fails without a request. An external agent, a plugin tool, and an MCP tool other than the managed browser follow `ask`, `edit` and `allow` like any other call.

`session_modes` overrides `mode` per conversation. A missing entry inherits `config.mode`. A request with `policy_session` uses the mode of that parent conversation instead of its own. A child conversation's own entry cannot be set.

A grant with `scope` `global` omits `session_id` and applies in every conversation. A grant with `scope` `session` includes `session_id` and applies to that conversation and to a request whose `policy_session` is that same id. `tool_wide` true matches the same `tool` and the same `tool_identity` (both empty still matches) and does not depend on `resource`. Otherwise the grant matches the same `tool` and the same `key` only. Grants are also created outside a card, for example a global tool grant made on the Core host; they appear in this list and are revoked here. `key` is an opaque match string, not a credential. Request objects do not include it.

`status` on a request is `pending`, `approved`, `rejected`, `denied`, `expired` or `cancelled`. `scope` on an approved request is `once`, `session` or `global`. `finished_at` is set once the request leaves `pending`. A restart stores a still-waiting request as `expired` with `reason` `Core restarted; this operation was not replayed`.

Changes publish `approvals.changed` on `GET /v1/events`, with no conversation id. One request can publish it more than once. Refetch `GET /v1/approvals`. Submitting a policy or a mode that is already stored publishes nothing.

### GET /v1/approvals

Auth: any key
Effect: Returns the policy, the per-conversation modes, every grant, the waiting requests, and one page of decision history.

Query: `session_id` (string, optional) keeps only requests whose `session_id` equals this conversation. It does not filter `pending`, `pending_requests`, grants or `session_modes`. An empty value does not filter. `offset` (int, optional) is the start of the history page. A missing value, or a value that is not an integer, counts as 0. A negative value counts as 0. A value past the filtered history counts as the end of it. The page holds at most 100 requests. There is no page-size parameter.

Request: no body.

Response: `200`

```json
{
  "next_offset": 1,
  "pending": 1,
  "pending_requests": [
    {
      "tool_wide": true,
      "id": "a1b2c3d4e5f60718293a4b5c6d7e8f90",
      "session_id": "2026-10-14T12-00-00_01234567-89ab-cdef-0123-456789abcdef",
      "run_id": "0123456789abcdef0123456789abcdef",
      "tool_call_id": "call_01",
      "tool": "exec_command",
      "resource": "{\"command\":\"printf hello\"}",
      "input": {"command": "printf hello"},
      "cwd": "/work/demo",
      "unattended": false,
      "status": "pending",
      "created_at": "2026-10-14T12:00:00Z"
    }
  ],
  "requests": [
    {
      "tool_wide": true,
      "id": "a1b2c3d4e5f60718293a4b5c6d7e8f90",
      "session_id": "2026-10-14T12-00-00_01234567-89ab-cdef-0123-456789abcdef",
      "run_id": "0123456789abcdef0123456789abcdef",
      "tool_call_id": "call_01",
      "tool": "exec_command",
      "resource": "{\"command\":\"printf hello\"}",
      "input": {"command": "printf hello"},
      "cwd": "/work/demo",
      "unattended": false,
      "status": "pending",
      "created_at": "2026-10-14T12:00:00Z"
    }
  ],
  "state": {
    "config": {"mode": "ask", "unattended": "deny"},
    "grants": [
      {
        "tool_wide": true,
        "cwd": "/work/demo",
        "id": "fedcba9876543210fedcba9876543210",
        "key": "0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef",
        "tool": "exec_command",
        "resource": "{\"command\":\"printf hello\"}",
        "scope": "session",
        "session_id": "2026-10-14T12-00-00_01234567-89ab-cdef-0123-456789abcdef",
        "created_at": "2026-10-14T12:00:00Z"
      }
    ],
    "session_modes": {
      "2026-10-14T12-00-00_01234567-89ab-cdef-0123-456789abcdef": "edit"
    }
  },
  "total": 1
}
```

- `state.config` is the policy above. `state.grants` is every grant, in stored order, or `[]`. `state.session_modes` maps a conversation id to `ask`, `edit` or `allow`, or `{}`.
- `requests` is the history page, newest `created_at` first, every `status`, after the `session_id` filter. `total` is the filtered count before paging. `next_offset` is the index just after this page; it equals `total` when this page reaches the end.
- `pending` is the number of requests with `status` `pending`, across every conversation. `pending_requests` is those requests, newest first, not paged and not filtered. A waiting request is also in `requests` when the filter and the page include it.
- Request fields that are present when set: `tool_identity` (string), `tool_wide` (true), `policy_session` (parent conversation id), `tool_label` (text to show), `once_only` (true), `scope`, `reason`, `resolver`, `finished_at`. They are omitted when empty. `unattended` false is still present.
- `tool` is the tool name. For an external agent it is `codex`, `grok`, `pi` or `claude`. `tool_identity` is then `agent:<agent>`, or `agent:<agent>:<operation>` when the grant covers one operation type; `tool_label` is the agent name, or that name, ` · `, and a short operation phrase. A plugin tool uses a `plugin:` identity. `tool_label` is otherwise a short label such as `exec_command · process guard` or `Computer use / ` plus the app name.
- `resource` is the text to show for the target. For `write` and `edit` it is the absolute file path. Otherwise it is a description of the call, often the arguments as JSON text. `input` is the tool arguments as a JSON value. `cwd` is the working directory. `run_id` identifies the run that asked. `tool_call_id` identifies the call inside that run.
- `resolver` is set when this route did not make the decision. It is `hook:` plus a handler id when a hook allowed that one call, or `panel:` plus a plugin id when a plugin panel did; those records are `approved` with `scope` `once`. It is `policy:unattended` on a `denied` record that the unattended policy refused (including a once-only request in an unattended run).
- On a grant, `tool_identity`, `tool_wide` and `tool_label` are omitted when empty, and `session_id` is omitted when the scope is `global`. `cwd` and `key` may be empty strings. `created_at` is when the grant was stored.

Errors:

- `503` approvals are not running
- `503` `approval storage unavailable: ` followed by the storage error

Events: none.

### PUT /v1/approval-settings

Auth: any key
Effect: Replaces the global attended mode and the unattended policy. Grants and per-conversation modes stay. When the stored policy actually changes, every waiting request expires and is not run under the new policy; `reason` is `approval policy changed; retry the operation`. Sending the policy that is already stored changes nothing and expires nothing.

Request:

```json
{"mode": "ask", "unattended": "deny"}
```

- `mode` (string, required) is `ask`, `edit` or `allow`.
- `unattended` (string, required) is `deny`, `wait` or `allow`.
- Both fields are replaced together. Omitting either one, or any other value, is refused. Unknown fields are ignored. The body is one JSON object of at most 4096 bytes.

Response: `200`

```json
{"ok": true}
```

Errors:

- `400` invalid settings (the body is not a JSON object of at most 4096 bytes)
- `409` invalid approval mode
- `409` the approval store has faulted; the message is the storage error. If waiting requests were being expired, the new policy may already be stored
- `503` approvals are not running

`400` and `409` `invalid approval mode` do not publish the event. A saved policy publishes once, and each request it expires publishes once more.

Events: `approvals.changed`

### PUT /v1/sessions/{id}/approval-mode

Auth: any key
Effect: Sets or clears this conversation's override of the attended mode. The global policy and grants stay. When the stored override actually changes, waiting requests for this conversation, and waiting requests whose `policy_session` is this conversation, expire and are not run under the new mode. `reason` is `conversation approval mode changed; retry the operation`. Sending the mode already stored, or clearing an override that is already absent, changes nothing and expires nothing.

Path: `id` conversation id.

Request:

```json
{"mode": "allow"}
```

- `mode` (string) is `ask`, `edit` or `allow`. An empty string, or a body whose `mode` is absent, removes the override so the conversation inherits `config.mode`.
- Any other value is refused. Unknown fields are ignored. The body is one JSON object of at most 4096 bytes.

Response: `200`

```json
{"ok": true}
```

Errors:

- `404` conversation not found
- `409` the conversation is a child of another conversation; change the parent. The message is `子任务权限跟随父对话，请在父对话调整`
- `400` invalid mode (the body is not a JSON object of at most 4096 bytes)
- `409` invalid approval mode
- `409` the approval store has faulted; the message is the storage error. If waiting requests were being expired, the new mode may already be stored
- `503` approvals are not running

The conversation is checked before the body, so a bad body on a missing conversation is `404`, and on a child conversation is `409`. Those, and `409` `invalid approval mode`, do not publish the event. A saved mode publishes once, and each request it expires publishes once more.

Events: `approvals.changed`

### POST /v1/approvals/{approval}/decision

Auth: any key
Effect: Decides one waiting request. `once` and `reject` affect that request only. `session` and `global` store one grant and also decide every other waiting request that grant covers. An approved call then continues; this response does not include the tool result. A rejected call does not run. `reason` on a rejection is `rejected by user`.

Path: `approval` request `id`.

Request:

```json
{"scope": "once"}
```

- `scope` (string, required) is `once`, `session`, `global` or `reject`.
- `once` records `approved` with `scope` `once` and stores no grant.
- `reject` records `rejected`, stores no grant, and leaves every other waiting request alone.
- `session` stores a grant with `scope` `session` and `session_id` equal to the request's `session_id` (the child conversation, when the card is the child's). `global` stores a grant with `scope` `global` and no `session_id`. Each matching waiting request is `approved` with that same `scope`. One whose run is already cancelled is `cancelled`, with `reason` `context canceled` or `context deadline exceeded`. One whose check fails is `expired`, with `reason` the check. Those outcomes still record that `scope`. A tool-wide grant replaces grants of that same tool, in the same scope and the same `session_id`, that it covers. A grant that is not tool-wide replaces only a non-tool-wide grant of the same tool and the same `key` in that scope.
- `once_only` true ignores `session` and `global`: the request is approved `once`, and no grant is stored. Offer `once` and `reject` for that card.
- A missing or unknown `scope` is refused. Unknown fields are ignored. The body is one JSON object of at most 4096 bytes.

Response: `200`

```json
{"ok": true}
```

- The decided request's `scope` is `once`, `session` or `global` when it was approved, and omitted when it was rejected. `finished_at` is set.

Errors:

- `400` invalid decision (the body is not a JSON object of at most 4096 bytes)
- `409` invalid approval scope
- `409` approval is no longer pending (unknown id, already decided, or already expired)
- `409` the run was cancelled before the decision. The message is `context canceled`, or `context deadline exceeded` when the run's deadline passed. The record is `cancelled` with `reason` `task cancelled`
- `409` a check failed before the decision. The record is `expired` and `reason` is the same message. The message is whatever check failed, including `working directory changed while waiting for approval`, `file target changed while waiting for approval`, `file appeared while waiting for approval`, `file changed while waiting for approval`, `MCP configuration or tools changed; retry with the current tool list`, or `invalid project cwd "` plus the directory, then `": workspace unavailable; restore the directory or choose another location`
- `409` the approval store has faulted; the message is the storage error. A grant for this decision may already be stored; refetch
- `503` approvals are not running

A `409` that cancels or expires the request still publishes the event. A `409` for a bad scope, a request that is not waiting, or a store fault before anything is saved does not.

Events: `approvals.changed`

### DELETE /v1/approval-grants/{grant}

Auth: any key
Effect: Removes one grant. Every waiting request then expires, not only requests that grant covered, and a call that already started is left running. `reason` is `authorization was revoked; retry the operation`.

Path: `grant` grant `id`.

Request: no body.

Response: `200`

```json
{"ok": true}
```

Errors:

- `409` file does not exist (no grant has this id)
- `409` the approval store has faulted; the message is the storage error. If waiting requests were being expired, the grant may already be gone
- `503` approvals are not running

A missing grant does not publish the event. A removed grant publishes once, and each request it expires publishes once more.

Events: `approvals.changed`
