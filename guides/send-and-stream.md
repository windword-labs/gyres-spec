# Send a message and stream the reply

`POST /v1/sessions/{id}/turns` starts one turn and returns its progress as `text/event-stream`. Closing that response does not stop the turn. Render text and tool calls from the frames, and when this conversation is already running, add a follow-up with `POST /v1/sessions/{id}/messages` instead of starting a second turn.

The id below is a session id. A lane is that id, `~`, and a lane name. Codex, Grok, pi, and Claude run only on the main lane.

## 1. Start a turn

The body is JSON. `prompt` is required unless `attachments` is non-empty. Omit `provider_id` and `model` to use the conversation's model, then the settings default, then the first available model.

```bash
curl -N -sS -D - -X POST \
  -H "Authorization: Bearer $GYRES_KEY" \
  -H 'Content-Type: application/json' \
  -d '{"prompt":"Fix the parser and add a test"}' \
  http://127.0.0.1:15100/v1/sessions/2026-10-14T12-04-05_3f2a1b0c-4d5e-6789-abcd-ef0123456789/turns
```

Success is `200` with `Content-Type: text/event-stream`. Save the response header `X-Gyres-Run-Stream` (16 lowercase hex characters). Each stored frame is a `data:` line of JSON, an `id:` line (a decimal offset), and a blank line. About every 10 seconds the stream sends `: heartbeat`, which has no `id`. Ignore lines that start with `:`. The stream stays open until the turn finishes. This response always starts at the beginning of the turn.

```
data: {"type":"run_snapshot","session_id":"2026-10-14T12-04-05_3f2a1b0c-4d5e-6789-abcd-ef0123456789","records":[]}
id: 128

data: {"type":"text_delta","text":"Looking at the parser."}
id: 196

data: {"type":"tool_start","call":{"id":"call_1","name":"read","input":{"path":"parser.go"}}}
id: 280

data: {"type":"tool_end","call":{"id":"call_1","name":"read","input":{"path":"parser.go"}},"result":{"content":[{"type":"text","text":"ok"}],"is_error":false}}
id: 420

data: {"type":"message_complete","message":{"content":[{"type":"text","text":"Done."}],"stop_reason":"end_turn","usage":{"input_tokens":12,"output_tokens":4,"cache_read_tokens":0,"cache_write_tokens":0},"error_message":null}}
id: 640

data: {"type":"result","session_id":"2026-10-14T12-04-05_3f2a1b0c-4d5e-6789-abcd-ef0123456789","stop_reason":"end_turn"}
id: 780
```

The turn is accepted before the stream is attached. The call is not idempotent: after an ambiguous failure, `GET /v1/sessions/{id}` before you send again. Core also emits `session.run_started`, and when the turn ends `session.run_finished`, on `GET /v1/events`.

## 2. Render the frames

Apply frames in order. Ignore a `type` you do not know.

- `run_snapshot` — replace the transcript on screen with `records`, then apply later frames. `records` match `GET /v1/sessions/{id}`.
- `agent_start` / `turn_start` — a run and a model step are starting. `turn` starts at 1.
- `text_delta` — append `text`. `thinking_delta` — append `thinking`.
- `tool_input_delta` — append `partial_json` to the call `id`. The first fragment includes `name`.
- `tool_start` — show `call` (`id`, `name`, `input`) as running. `input` is `null` when there are no arguments.
- `tool_end` — show `result.content` and `result.is_error` on that call. A call cut off at the output cap can send `tool_end` without `tool_start`.
- `tool_blocked` — the call did not run. Show `reason` when it is present.
- `message_complete` — replace this step's streamed text with `message`. `error_message` is `null` or a string. `stop_reason` is `end_turn`, `tool_use`, `max_tokens`, `error`, or `aborted`.
- `usage` — token counts for the step. Stored totals still come from the usage routes.
- `steering_message` — a user message entered between steps. Show `message`. A `metadata.internalMessage` row is Core's own text.
- `result` — the outcome, and the last stored frame of a normal finish. `stop_reason` is `end_turn`, `error`, or `aborted`. `error` is omitted on `end_turn` and `aborted`. When the stream then closes, reload `GET /v1/sessions/{id}`.
- `compaction_start` / `compaction_end` — Core is compacting the context before the next model request, which can take many seconds with nothing else streaming. Show a status until `compaction_end` (`ok` false when it failed) or `result`.
- `approval` and `user_question` — see [Tool approvals](approvals.md). `output_discarded` (`reason` `overflow`) drops the reply streamed since the last stored row; a following `run_snapshot` replaces the view.

`agent_end` finishes the loop. Wait for `result`, then reload the conversation. `stream_error` is not stored, has no `id`, and the response then ends. The turn itself continues; reload the conversation and reattach if it is still running ([Stop a run and reattach](stop-and-reattach.md)).

## 3. When a turn is already running

`409` with the message `turn already running` (for an external agent, `turn already running or awaiting recovery`) means a turn is live. Do not start another. Queue a follow-up. The body has no mode field. Core stores it as `steer` or `queue` from the current running-input setting. A missing setting is `steer`. A Grok turn always stores `queue`. `steer` can enter at the next step. `queue` waits until the current work takes a follow-up.

```bash
curl -sS -X POST \
  -H "Authorization: Bearer $GYRES_KEY" \
  -H 'Content-Type: application/json' \
  -d '{"id":"queued_0123456789abcdef0123456789abcdef","prompt":"Also check the tests"}' \
  http://127.0.0.1:15100/v1/sessions/2026-10-14T12-04-05_3f2a1b0c-4d5e-6789-abcd-ef0123456789/messages
```

`200`:

```json
{
  "session_id": "2026-10-14T12-04-05_3f2a1b0c-4d5e-6789-abcd-ef0123456789",
  "queued": true,
  "id": "queued_0123456789abcdef0123456789abcdef"
}
```

`id` is optional, at most 128 characters. Send the same id on a retry; the same content is accepted again and does not create a second message. A different body with that id is `409` `message id already used`. `queued` is true both when the message is waiting and when this id was already accepted. Read the mode that was stored:

```bash
curl -sS -H "Authorization: Bearer $GYRES_KEY" \
  http://127.0.0.1:15100/v1/sessions/2026-10-14T12-04-05_3f2a1b0c-4d5e-6789-abcd-ef0123456789/pending-messages
```

`messages[].mode` is `steer` or `queue`. The same route is how you show the waiting list. `GET /v1/running-input-settings` returns the setting (`{"mode":"steer"}` or `{"mode":"queue"}`); `PUT` the same path with that body changes it for later follow-ups.

A different `409` means nothing is live, so `POST …/messages` will also fail (`no active turn ready for input`). The body is not the usual error envelope:

```json
{
  "error": "Gyres is waiting to be continued; continue or stop it first",
  "code": "awaiting_continuation",
  "agent": "Gyres",
  "state": "interrupted"
}
```

`agent` is `Gyres`, `Codex`, `Grok`, `pi`, or `Claude`. `state` is the run state. Continue with `POST /v1/sessions/{id}/resume` or stop with `POST /v1/sessions/{id}/cancel`, then send a new turn.

## 4. Attach an image

Upload the raw image bytes (not multipart) before the turn. `Content-Type` is ignored. At most 20 MiB. PNG, JPEG, GIF, and WebP only.

```bash
curl -sS -D - -X POST \
  -H "Authorization: Bearer $GYRES_KEY" \
  --data-binary @photo.png \
  "http://127.0.0.1:15100/v1/sessions/2026-10-14T12-04-05_3f2a1b0c-4d5e-6789-abcd-ef0123456789/attachments?name=photo.png"
```

`201` when the bytes are new, `200` when they were already stored:

```json
{
  "id": "0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef",
  "name": "photo.png",
  "mime_type": "image/png",
  "size": 12345
}
```

`id` is 64 lowercase hex characters. Pass it as `attachments` on the turn or the follow-up: `{"prompt":"What is in this image?","attachments":["<id>"]}`. A prompt may be `""` when `attachments` is non-empty. At most 20 images and 40 MiB together. A model that does not accept images is rejected. The same limits apply to `POST …/messages`.

## Edge cases

- `400` `missing prompt` when both the prompt and `attachments` are empty. Whitespace-only prompts are rejected for Codex, Grok, pi, and Claude unless there is an image.
- `409` `子会话请通过父任务追加工作` — this conversation is a child; send further work from the parent.
- `409` `任务尚未完成，请先继续或停止` — a task is still open.
- `409` `{"error":"stopped before the turn started","code":"turn_stopped"}` — nothing was recorded.
- `409` while the turn is still preparing: `POST …/messages` returns `no active turn ready for input`. Retry shortly.
- `403` on a Codex, Grok, pi, or Claude conversation asks an administrator to allow non-administrators. An ordinary key still runs a provider-model turn.
- `404` `session not found`. `503` `storage unavailable; reconnect and retry` or Core is moving its data directory.
- Keep `X-Gyres-Run-Stream` and the last `id:` so a dropped connection can resume. That is [Stop a run and reattach](stop-and-reattach.md).

## See also

- [POST /v1/sessions/{id}/turns](../http/runs.md#post-v1sessionsidturns) and [POST /v1/sessions/{id}/messages](../http/runs.md#post-v1sessionsidmessages)
- [GET /v1/running-input-settings](../http/models.md#get-v1running-input-settings)
- [POST /v1/sessions/{id}/attachments](../http/files.md#post-v1sessionsidattachments)
- [Run events](../events.md#run-events-get-v1sessionsidevents) — `text_delta`, `tool_start`, `tool_end`, `message_complete`, `result`
- [Errors](../errors.md)
