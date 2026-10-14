# Stop a run and reattach to it

A turn keeps running after the client that started it disconnects. Stop it only with `POST /v1/sessions/{id}/cancel`. Anyone, including that client after a dropped connection, attaches with `GET /v1/sessions/{id}/events`. When the stream ends, reload the transcript.

The id in the examples is `2026-10-14T12-04-05_3f2a1b0c-4d5e-6789-abcd-ef0123456789`.

## 1. Stop

The request has no body.

```bash
curl -sS -X POST \
  -H "Authorization: Bearer $GYRES_KEY" \
  http://127.0.0.1:15100/v1/sessions/2026-10-14T12-04-05_3f2a1b0c-4d5e-6789-abcd-ef0123456789/cancel
```

`200`:

```json
{
  "session_id": "2026-10-14T12-04-05_3f2a1b0c-4d5e-6789-abcd-ef0123456789",
  "canceled": true
}
```

`canceled` is always true on success. The HTTP response can arrive before the run stream has closed. Child tasks of this turn are stopped too. When the turn belongs to a scheduled task, that occurrence is cancelled. Core emits `session.task` and `session.run_finished` on `GET /v1/events`.

A second stop, once nothing is left, is `409` `no active turn`. `404` means the conversation does not exist. `500` `task stopped, but cancellation could not be saved: …` means the turn was still stopped.

## 2. Read run-state until it is idle

```bash
curl -sS -H "Authorization: Bearer $GYRES_KEY" \
  http://127.0.0.1:15100/v1/sessions/2026-10-14T12-04-05_3f2a1b0c-4d5e-6789-abcd-ef0123456789/run-state
```

`200`:

```json
{ "state": "running", "reason": "" }
```

`state` is `idle`, `running`, `pausing`, `paused`, `waiting_children`, `interrupted`, or `blocked`. `reason` is always present and may be `""`. After a successful cancel, follow this route or the run stream until the live turn has left. `state` is then `idle`. Also refetch when `session.task` arrives for this conversation.

`interrupted` means the task stopped and can be continued (`POST /v1/sessions/{id}/resume`). `blocked` means it cannot continue until the owner acts. `paused` is waiting for `resume` or `cancel`. A `204` from the event stream (below) does not by itself mean `idle`.

## 3. Reattach with stream and after

Pass the `X-Gyres-Run-Stream` value you saved (16 lowercase hex characters) and the `id:` of the last frame you handled. Both are required together. The server does not read a last-event id header.

```bash
curl -N -sS -D - \
  -H "Authorization: Bearer $GYRES_KEY" \
  "http://127.0.0.1:15100/v1/sessions/2026-10-14T12-04-05_3f2a1b0c-4d5e-6789-abcd-ef0123456789/events?stream=a1b2c3d4e5f60718&after=128"
```

While a turn is running the response is `200` `text/event-stream`, again with `X-Gyres-Run-Stream`. A matching cursor starts with `run_resume` and no `id:`:

```
data: {"type":"run_resume","session_id":"2026-10-14T12-04-05_3f2a1b0c-4d5e-6789-abcd-ef0123456789"}

data: {"type":"text_delta","text":"Still working."}
id: 256
```

Keep the transcript you already show, then apply only the frames after `run_resume`. Each later stored frame has an `id:`. Heartbeats (`: heartbeat`, about every 10 seconds) do not. `stream` and `after` are accepted only for this turn's own stream, and only when `after` is the end of a frame.

If the cursor does not match, the stream replays the turn from the beginning. The first stored frame is `run_snapshot`. Replace what you show with its `records` before you apply later frames.

The response stays open until the turn finishes or you disconnect. Close without a `result` frame is a dropped connection: reconnect with the same `stream` and the last `id:` you handled. If the progress log breaks, one frame arrives and the response ends:

```json
{ "type": "stream_error", "error": "progress stream unavailable; reload conversation" }
```

That frame has no `id`. The turn is not cancelled. Reload `GET /v1/sessions/{id}`, and attach again if `running` is still set.

## 4. Or resume an external-agent turn with history_cursor

`history_cursor` is the field on `GET /v1/sessions/{id}`. It resumes a Codex, Grok, pi, or Claude turn after the transcript you loaded. A Gyres turn does not resume this way; use `stream` and `after`.

```bash
curl -N -sS \
  -H "Authorization: Bearer $GYRES_KEY" \
  "http://127.0.0.1:15100/v1/sessions/2026-10-14T12-04-05_3f2a1b0c-4d5e-6789-abcd-ef0123456789/events?history_cursor=CURSOR"
```

Replace `CURSOR` with the `history_cursor` you just received. Do not invent one. Query order: `stream` and `after` are tried first; if they do not match, `history_cursor` is tried; anything else replays from `run_snapshot`. A matching cursor yields `run_resume`, then events after that mark. Keep the history you loaded. Approvals and questions that were not saved in the transcript are not resent; read `GET /v1/approvals` and `GET /v1/sessions/{id}/user-questions`.

## 5. 204 means nothing is live

When nothing is running, including a task that is paused, blocked, or interrupted but not live, the response is `204` with no body. The header `X-Gyres-History-Version` is the same opaque string as `history_version` on `GET /v1/sessions/{id}`. Compare it with the transcript you just loaded. The same value means nothing ran in between.

```bash
curl -sS -D - -o /dev/null \
  -H "Authorization: Bearer $GYRES_KEY" \
  http://127.0.0.1:15100/v1/sessions/2026-10-14T12-04-05_3f2a1b0c-4d5e-6789-abcd-ef0123456789/events
```

Then read `run-state`. If you still need to continue or stop a waiting task, that route is the one that says `interrupted`, `paused`, or `blocked`.

`409` `turn is preparing; retry observation` means a turn is starting and has no stream yet. Retry the GET.

## 6. Reload the transcript after the end

A normal finish delivers the remaining frames and closes. The last stored frame is `result` when the run published one. The next `GET …/events` is `204`. Reload once:

```bash
curl -sS -H "Authorization: Bearer $GYRES_KEY" \
  http://127.0.0.1:15100/v1/sessions/2026-10-14T12-04-05_3f2a1b0c-4d5e-6789-abcd-ef0123456789
```

That body is the authoritative transcript. `session.run_finished` on `GET /v1/events` is the signal for every other client. Frame fields are in the events reference. Rendering them is covered in [Send a message and stream the reply](send-and-stream.md).

## Edge cases

- `404` `session not found` on cancel, run-state, and the event stream.
- A replay that starts at `run_snapshot` replaces the on-screen transcript. `run_resume` does not.
- `history_cursor` from an earlier turn, a missing cursor, or a Gyres turn falls through to a full replay.
- Pausing is a different route (`POST /v1/sessions/{id}/pause`). Codex, Grok, pi, and Claude have no pause; stop them with cancel.
- Revoking the access key that started a live turn stops it.

## See also

- [POST /v1/sessions/{id}/cancel](../http/runs.md#post-v1sessionsidcancel), [GET /v1/sessions/{id}/run-state](../http/runs.md#get-v1sessionsidrun-state), [GET /v1/sessions/{id}/events](../http/runs.md#get-v1sessionsidevents)
- [SSE framing](../events.md#sse-framing) and [run events](../events.md#run-events-get-v1sessionsidevents) (`run_resume`, `run_snapshot`, `result`, `stream_error`)
- [GET /v1/sessions/{id}](../http/sessions.md#get-v1sessionsid)
- [session.run_finished](../events.md#sessionrun_finished) and [session.task](../events.md#sessiontask)
- [Errors](../errors.md)
