# Tool approvals

Some tool calls wait until a person decides. The request shows up on the run stream and on the Core-wide event stream; `GET /v1/approvals` is the list to render, and `POST /v1/approvals/{approval}/decision` records the decision. Closing the page does not approve, reject, or cancel a waiting request. Any client with a key can decide it.

A call that a grant or the policy already allows does not appear here. Read-only calls, the `subagent` tool, and a fixed set of list and read actions never ask.

## 1. Notice a waiting request

On the run stream, a frame with `"type":"approval"` carries the request. `status` `pending` is a card to show. The same shape is sent again when the request is decided. `id` is 32 lowercase hex characters.

```json
{
  "type": "approval",
  "approval": {
    "id": "a1b2c3d4e5f60718293a4b5c6d7e8f90",
    "session_id": "2026-10-14T12-04-05_3f2a1b0c-4d5e-6789-abcd-ef0123456789",
    "run_id": "0123456789abcdef0123456789abcdef",
    "tool_call_id": "call_01",
    "tool": "exec_command",
    "resource": "{\"command\":\"printf hello\"}",
    "input": { "command": "printf hello" },
    "cwd": "/work/demo",
    "unattended": false,
    "status": "pending",
    "created_at": "2026-10-14T12:00:00Z"
  }
}
```

Show `tool_label` when it is present, otherwise `tool`. `resource` is the target text (for `write` and `edit`, the absolute file path). `input` is the tool arguments. `once_only: true` means this card cannot be covered by a grant: offer only allow-once and reject.

The Core-wide stream does not include the request. It sends `approvals.changed` with no `session_id`, including more than once for one request. Refetch the list. The stored request can still change after the last `approval` frame.

```bash
curl -N -sS -H "Authorization: Bearer $GYRES_KEY" \
  http://127.0.0.1:15100/v1/events
```

```
data: {"type":"approvals.changed"}
```

## 2. Load the list

```bash
curl -sS -H "Authorization: Bearer $GYRES_KEY" \
  "http://127.0.0.1:15100/v1/approvals?session_id=2026-10-14T12-04-05_3f2a1b0c-4d5e-6789-abcd-ef0123456789&offset=0"
```

`200` includes:

- `pending` — how many requests are `pending`, across every conversation.
- `pending_requests` — those requests, newest first. Not paged, and not filtered by `session_id`. Filter this array yourself when the card belongs to one conversation. Render these.
- `requests` — one page of history, every status, newest `created_at` first. `session_id` keeps only rows for that conversation. The page holds at most 100. There is no page-size parameter.
- `total` — the filtered history count. `next_offset` is the next `offset`; it equals `total` on the last page. A missing, negative, or non-integer `offset` counts as 0.
- `state.config` — `mode` is `ask`, `edit`, or `allow` (initial `ask`). `unattended` is `deny`, `wait`, or `allow` (initial `deny`).
- `state.session_modes` — conversation id to `ask`, `edit`, or `allow`. A missing entry inherits `config.mode`.
- `state.grants` — remembered allows. `scope` `global` omits `session_id`. `scope` `session` includes it.

`status` on a request is `pending`, `approved`, `rejected`, `denied`, `expired`, or `cancelled`. `scope` on an approved request is `once`, `session`, or `global`. `finished_at` is set once it leaves `pending`.

`503` means approvals are not running, or approval storage is unavailable.

## 3. Decide

`scope` is `once`, `session`, `global`, or `reject`. Unknown fields are ignored. The body is at most 4096 bytes.

```bash
curl -sS -X POST \
  -H "Authorization: Bearer $GYRES_KEY" \
  -H 'Content-Type: application/json' \
  -d '{"scope":"once"}' \
  http://127.0.0.1:15100/v1/approvals/a1b2c3d4e5f60718293a4b5c6d7e8f90/decision
```

`200`:

```json
{ "ok": true }
```

- `once` approves that request and stores no grant.
- `reject` records `rejected` with reason `rejected by user`. Other waiting requests stay. The call does not run.
- `session` stores a session grant and approves every other waiting request that grant covers. The grant's conversation is the request's `session_id`.
- `global` stores a grant with no `session_id` and approves every waiting request it covers.
- `once_only: true` ignores `session` and `global`: the request is approved `once`, and no grant is stored.

The response does not include the tool result. An approved call continues inside the run. Refetch `GET /v1/approvals` after `approvals.changed`. `DELETE /v1/approval-grants/{grant}` removes one grant (`200` `{"ok":true}`) and expires every waiting request, not only the ones that grant covered.

## 4. Set the conversation's approval mode

This overrides `config.mode` for one conversation. `mode` is `ask`, `edit`, or `allow`. An empty string, or a JSON object with no `mode`, clears the override so the conversation inherits the global mode.

```bash
curl -sS -X PUT \
  -H "Authorization: Bearer $GYRES_KEY" \
  -H 'Content-Type: application/json' \
  -d '{"mode":"allow"}' \
  http://127.0.0.1:15100/v1/sessions/2026-10-14T12-04-05_3f2a1b0c-4d5e-6789-abcd-ef0123456789/approval-mode
```

`200` is `{"ok":true}`. When the stored override actually changes, waiting requests for this conversation expire and are not run under the new mode. Reason: `conversation approval mode changed; retry the operation`. Sending the mode already stored changes nothing. A child conversation cannot set its own mode (`409`, message `子任务权限跟随父对话，请在父对话调整`); change the parent. A request with `policy_session` uses that parent's mode. It is a subtask's request (native or external-agent child): show it, and let it be answered, in the parent conversation too, or a parent waiting for that child stalls unseen.

`ask` asks unless a grant covers the call. `edit` does the same and also allows `write` and `edit` inside the conversation's working directory or a workspace root. `allow` allows the attended call without asking. `once_only` is never covered by a mode or a grant. `PUT /v1/approval-settings` with `{"mode":"ask","unattended":"deny"}` replaces the global policy and expires every waiting request when the stored policy changes.

## 5. User questions

A question is not an approval. The run asks and waits. `GET /v1/events` sends `session.question` with `session_id` and `question_id`. The run stream sends `user_question` with the same batch id. Questions live only in memory.

```bash
curl -sS -H "Authorization: Bearer $GYRES_KEY" \
  http://127.0.0.1:15100/v1/sessions/2026-10-14T12-04-05_3f2a1b0c-4d5e-6789-abcd-ef0123456789/user-questions
```

`200` is `{"questions":[...]}`. Each batch has `id` (a UUID), `tool_call_id`, `questions` (1 to 3), `created`, and `deadline`. Each question has `id`, `header`, `question`, and `options` (2 or 3 of `label` and `description`). Use `deadline`. It is 60 seconds after `created`, unless a longer wait was started. After it, the turn continues with the first option of each question and the batch disappears.

Answer every question in the batch before the deadline. The value does not have to be one of the option labels.

```bash
curl -sS -X POST \
  -H "Authorization: Bearer $GYRES_KEY" \
  -H 'Content-Type: application/json' \
  -d '{"answers":{"library":"net/http"}}' \
  http://127.0.0.1:15100/v1/sessions/2026-10-14T12-04-05_3f2a1b0c-4d5e-6789-abcd-ef0123456789/user-questions/018f3a2b-7c4d-7e5f-a6b7-8c9d0e1f2a3c/answer
```

`200` is `{"ok":true}`. The run stream then sends `user_question_resolved` with `source` `user` or `timeout`, and `answers` mapping each question id to the chosen label. `source` `cancelled` means the wait was cancelled. `409` `question already resolved` or `question not found` means the batch is gone; refetch the list.

## Edge cases

- `409` `invalid approval scope` — `scope` was missing or not one of the four values.
- `409` `approval is no longer pending` — unknown id, already decided, or already expired. Refetch.
- `409` `context canceled` or `context deadline exceeded` — the run was cancelled. The record is `cancelled`.
- A check can expire the request before your decision (`409`), including `working directory changed while waiting for approval` and `file changed while waiting for approval`. The record is `expired` and `reason` is that message.
- A restart stores a still-waiting request as `expired` with reason `Core restarted; this operation was not replayed`.
- Unattended `deny` stores the request as `denied` and does not ask. Unattended `wait` leaves it `pending`. Unattended `allow` runs it with no request. `unattended: false` is still present on a normal card.
- `400` means the body is not one JSON object of at most 4096 bytes.

## See also

- [Tool approvals](../http/approvals.md) — `GET /v1/approvals`, `POST /v1/approvals/{approval}/decision`, `PUT /v1/sessions/{id}/approval-mode`
- [approvals.changed](../events.md#approvalschanged) and the run-stream [approval](../events.md#approval) frame
- [session.question](../events.md#sessionquestion), [user_question](../events.md#user_question), [GET /v1/sessions/{id}/user-questions](../http/runs.md#get-v1sessionsiduser-questions), [POST …/answer](../http/runs.md#post-v1sessionsiduser-questionsqidanswer)
- [Errors](../errors.md)
