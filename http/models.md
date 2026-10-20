# Models and model settings

These routes are how a client lists models, sets the default model and the provider list, and reads or writes the Core-wide switches that sit next to chat: context mode, pre-prompt text, the fallback chain, recap, recovery, what happens to a message sent while a turn is running, and the owner's memory. Per-conversation model choice is a different set of routes.

JSON responses are `application/json`. An error body is `{"error":"<message>","failure":{"code":"core_request_failed","message":"<message>","http_status":<status>,"details":"<message>"}}`. Every administrator route except `POST /v1/complete` sends `Cache-Control: no-store`.

The Settings default is one provider and that provider's first active model. `GET /v1/chat-defaults` and `PUT /v1/chat-defaults` are a view of that default. A model choice object is:

```json
{ "provider_id": "acme", "model": "demo-model", "reasoning_effort": "", "speed": "" }
```

`reasoning_effort` and `speed` are empty on this default. `speed` elsewhere is a service-tier id.

External agents use the provider ids `local-codex`, `local-grok`, `local-pi`, `local-claude` and `local-cursor`. They are not providers. The default model, the fallback chain and `POST /v1/complete` refuse them.

### GET /v1/chat-defaults

Auth: any key
Effect: Returns the Settings default model, and the model a new conversation would use when that default is missing.

Request: no body.

Response: `200`

```json
{
  "defaults": {
    "provider_id": "acme",
    "model": "demo-model",
    "reasoning_effort": "",
    "speed": ""
  },
  "effective": {
    "provider_id": "acme",
    "model": "demo-model",
    "reasoning_effort": "",
    "speed": ""
  }
}
```

- `defaults` is the Settings default: the selected provider, or else the first enabled provider that has an active model, and that provider's first active model. It is JSON `null` when no enabled provider has an active model. `reasoning_effort` and `speed` are always `""`.
- `effective` is the same object when `defaults` is set. When `defaults` is null, it is the first provider row of `GET /v1/models`, with empty `reasoning_effort` and `speed`. When that list has no provider row, `provider_id` is empty and `model` is the configured connection's model, or `""`.

Errors:

- `500` Core has no home directory

Events: none.

### GET /v1/context-default

Auth: admin
Effect: Returns the context mode stored for conversations created after this read. `summary` keeps a compacted view; `window` keeps the recent window. A conversation that already has its own mode is left as it is. Anything other than `window` is reported as `summary`, including a missing value.

Request: no body.

Response: `200`

```json
{ "mode": "summary" }
```

- `mode` is `summary` or `window`.

Errors:

- `500` the setting could not be read

Events: none.

### GET /v1/fallback-models

Auth: admin
Effect: Returns the global fallback chain. It is off when it has never been saved. While `enabled` is true, a turn whose model Core resolved can move to this chain after that model's own retries are spent and the failure happened before any output. A model named on the turn request stays on that model. Each later entry is tried once, in order, while the new failure is still a reason below. An entry equal to a model already tried on this request is skipped, and so is one that is no longer an active model of an enabled provider. The next request starts again from the conversation's own model.

A failure moves along the chain when it is a rate limit, an HTTP status of 500 or higher, an overload or unavailable service with no status, a timeout before output, a network failure, a missing model, an authentication failure, or an insufficient balance. Context overflow, a bad request, malformed output, a cancel, output that already started, and a failure inside Core stay on the original error.

Request: no body.

Response: `200`

```json
{
  "enabled": false,
  "models": []
}
```

- `enabled` is the switch. `false` still returns any saved `models`.
- `models` is tried in order. Each item is `provider_id` and `model`. The array is empty when nothing is saved, never JSON `null`.

Errors: none.

Events: none.

### GET /v1/memory

Auth: admin
Effect: Returns the owner's memory snapshot, the review settings, and the pending proposals and review jobs. Review transcripts are cleared before the response is sent. This is the owner memory, not one project's memory: project text is not in `snapshot`.

Request: no body.

Response: `200`

```json
{
  "snapshot": {
    "user": "Prefers short answers.",
    "memory": "The demo service listens on port 15100.",
    "version": "0000000000000000000000000000000000000000000000000000000000000000"
  },
  "state": {
    "generation": 1,
    "config": {
      "enabled": true,
      "review_enabled": true,
      "interval": 10,
      "provider_id": "",
      "model": "",
      "skill_learning": false
    },
    "proposals": [
      {
        "id": "a1b2c3d4e5f6789012345678",
        "operation": {
          "action": "add",
          "target": "memory",
          "content": "Ship notes go in the weekly file.",
          "reason": "Stated as a standing rule."
        },
        "source": {
          "session_id": "2026-10-14T12-02-00_550e8400-e29b-41d4-a716-446655440000",
          "origin": "review"
        },
        "version": "0000000000000000000000000000000000000000000000000000000000000000",
        "created_at": "2026-10-14T12:02:00Z"
      }
    ],
    "jobs": [
      {
        "id": "b1c2d3e4f506789012345678",
        "session_id": "2026-10-14T12-02-00_550e8400-e29b-41d4-a716-446655440000",
        "through": "msg_550e8400-e29b-41d4-a716-446655440000",
        "provider_id": "acme",
        "model": "demo-model",
        "project": "",
        "transcript": "",
        "status": "failed",
        "attempts": 3,
        "error": "the model did not answer",
        "next_attempt": "0001-01-01T00:00:00Z",
        "created_at": "2026-10-14T12:00:00Z",
        "generation": 1
      }
    ],
    "counters": {},
    "audit": [],
    "forgotten": []
  },
  "directory": "/data/memories/owner"
}
```

- `snapshot.user` and `snapshot.memory` are the two memory texts. Entries inside a text are separated by a blank line, `---`, and a blank line. `version` is the id of this text; send it back on `PUT /v1/memory`.
- `state.generation` increases when an edit drops queued reviews.
- `config.enabled` turns memory on. `review_enabled` turns the background review on. With no saved state both are true, and `interval` is 10. `interval` is how many completed turns pass before a review is queued. Empty `provider_id` and `model` together mean the review uses the conversation's model; a saved pair names one model. `skill_learning` adds skill writing to chat and to the review.
- `proposals` is JSON `null` until the first proposal. `id` is 24 hexadecimal characters. `operation.action` is `add`, `replace` or `remove`. `operation.target` is `user`, `memory` or `project`. `add` carries `content`. `replace` carries `content` and `old_text`. `remove` carries `old_text`. `old_text` is one whole existing entry. `reason` is why it was proposed. `source.origin` is `chat`, `review` or `desktop`. `source` may also carry `session_id`, `message_id`, `project`, `project_id` and `review_generation`. `version` is the snapshot the proposal was made against. `created_at` is UTC.
- `jobs` is JSON `null` until the first review. `transcript` is always `""` here. `status` is `pending`, `running`, `done`, `failed` or `superseded`. `attempts` counts runs. `error` is present after a failure; a review dropped because the owner edited memory says `Owner changed memory; older review discarded`. `next_attempt` is UTC, or `0001-01-01T00:00:00Z` when unset. `through` is the message the review has read through. `review_provider_id` and `review_model`, when present, are the model a running review was given. At most 50 finished jobs are kept.
- `counters` maps a conversation id to `{count, last_message_id}` and, when skill learning is counting tool calls, `tools`. It is `{}` when empty.
- `audit` is the latest 100 notes, oldest first once the cap is reached. Each note is `{at, action, target, source, detail}`. `action` is `edit`, `propose`, `approve`, `reject`, `add`, `replace`, `remove`, or `review_` plus a job status. A save made through `PUT /v1/memory` records `edit` with `source.origin` `desktop` and `detail` `Owner edited memory`.
- `forgotten` lists opaque hashes of entries an edit removed, so a later review does not put the same entry back. JSON `null` until the first one.
- `directory` is the absolute directory of this memory.

Errors:

- `500` memory could not be opened or read

Events: none.

### GET /v1/models

Auth: any key
Effect: Lists every model the chat can send, and the model id of the connection Core would call when a request names no provider. Provider rows come from each enabled provider's active list, in saved order. External-agent rows follow, in the order Codex, Grok, pi, Claude, and only for an agent that is switched on, signed in, and whose last check reported no error. An ordinary key does not receive an agent's rows unless that agent allows non-administrators. The first time an agent has no saved check, this response waits up to 10 seconds for one; a check older than 10 minutes is refreshed in the background and this response still returns the previous rows.

Request: no body.

Response: `200`

```json
{
  "models": [
    {
      "provider_id": "acme",
      "provider_name": "Acme",
      "provider_host": "192.0.2.20",
      "name": "Demo",
      "model": "demo-model",
      "limits": { "context_window": 128000, "max_output": 8192 },
      "limits_source": "catalog",
      "images_supported": true,
      "image_input": true,
      "reasoning": true,
      "reasoning_levels": ["low", "high"]
    }
  ],
  "current": "demo-model"
}
```

- `models` is an array, `[]` when there are none.
- `provider_id` and `model` identify the row. `provider_name` is the provider's name, or its id when the name is empty. For an external agent it is `Codex (local)`, `Grok (local)`, `pi (local)` or `Claude (local)`. `name` is the display name, or `model` when none was stored.
- `provider_host` is the hostname of the provider's base URL, lowercased, without port, path or credentials. Omitted when the URL has no host. External-agent rows omit it.
- `limits.context_window` and `limits.max_output` are token counts. Either number is omitted when unknown, so `limits` may be `{}`. `limits_source` is `manual` (set on the provider), `provider` (from the last model fetch), `catalog` (a known model id) or `unknown`. External-agent rows send `limits` `{}` and `limits_source` `""`.
- `images_supported` is the owner's image switch for that model. Absent means the owner has not set it. Core enforces this on send. `image_input` is that switch when it is set, otherwise what the provider's model list said. `reasoning` and `reasoning_levels` come from that list. All three are omitted when unknown.
- `reasoning_efforts` is the effort values a request may send, with `default_reasoning_effort` when the agent has one. On a provider row it is `reasoning_levels`, with no `default_reasoning_effort` (unset effort sends none, the model's own default). External-agent rows include them when the agent reported any. Both are omitted when empty.
- `service_tiers` is `[{id, name, description}]` and `default_service_tier` is one of those ids. Only Codex rows carry them, and only when the check reported tiers. `id` is the value to send as `speed` on a conversation.
- Claude rows include `images_supported: true`.
- `current` is a model id, with no provider. It is the configured connection's model when that connection has both a base URL and a model; otherwise the first active model of the first enabled provider; otherwise `""`. Saving providers or the Settings default keeps that connection aligned with the selected provider's first active model.

Errors:

- `500` provider settings could not be read

Events: none.

### GET /v1/pre-prompt

Auth: admin
Effect: Returns the optional text added to the system instructions of a model request. A non-empty model entry wins over that provider's entry, which wins over `global`. An empty string does not win. The next request uses whatever is saved at that moment. The text is an instruction, and it is not written into the conversation.

Request: no body.

Response: `200`

```json
{
  "global": "Answer in English.",
  "providers": { "acme": "Prefer metric units." },
  "models": { "acme": { "demo-model": "Keep the answer under four sentences." } }
}
```

- `global` is omitted when empty.
- `providers` maps a provider id to text. `models` maps a provider id to a map of model id to text. Both are `{}` when unset, never JSON `null`.

Errors:

- `500` the setting could not be read

Events: none.

### GET /v1/provider-settings

Auth: admin
Effect: Returns the provider list, which provider is selected, and the revision a later save must send back. API keys are included. Send the whole document back on `PUT /v1/provider-settings`; fields left out are deleted.

Request: no body.

Response: `200`

```json
{
  "providers": [
    {
      "id": "acme",
      "catalog_id": "acme",
      "name": "Acme",
      "protocol": "openai",
      "api_format": "chat_completions",
      "enabled": true,
      "base_url": "https://192.0.2.20/v1",
      "api_key": "example-key",
      "available": ["demo-model", "demo-small"],
      "active": ["demo-model"],
      "model_names": { "demo-model": "Demo" },
      "model_images": { "demo-model": true },
      "image_transport": "inline",
      "model_limits": { "demo-model": { "context_window": 128000, "max_output": 8192 } },
      "detected_models": {
        "demo-model": { "name": "Demo", "vendor": "acme", "image_input": true, "reasoning": true, "reasoning_levels": ["low", "high"] }
      }
    }
  ],
  "selected_id": "acme",
  "revision": "0000000000000000000000000000000000000000000000000000000000000000"
}
```

- `providers` is `[]` when none are saved. `selected_id` is `""` when none is selected. The Settings default is the selected provider's first `active` model, or the first enabled provider that has one.
- `id` is unique and must not be `local-codex`. `name` is the label. `catalog_id`, when set, picks a known limit table. `custom` is present only when it is true, for a provider the owner added by URL.
- `protocol` is what `POST /v1/provider-settings/fetch` calls. `api_format` is `chat_completions` or `responses`, and is omitted when unset. Unset means chat completions.
- `enabled` with an empty `base_url` is refused on save. `base_url` is an `http` or `https` URL with a host and with no user name, password, query or fragment.
- `api_key` is the credential. It is returned in full. `available` is the last fetched id list. `active` is the models chat may use, in order; the first entry is that provider's contribution to the Settings default. Either list is JSON `null` when missing. Empty maps are omitted.
- `model_names` maps a model id to a display name. `model_images` maps a model id to whether the owner allows images. `image_transport` is `inline` or `files`, and is omitted when unset. `files` is allowed only with `api_format` `responses`.
- `model_limits` maps a model id to `{context_window, max_output}`. A saved limit must have `context_window` from 4096 to 10000000 and `max_output` from 0 to 10000000. `detected_model_limits` and `detected_models` are what the last fetch reported. `detected_models` values use `name`, `vendor`, `image_input`, `reasoning` and `reasoning_levels`, each omitted when the fetch did not say.
- `revision` is 64 hexadecimal characters. It changes when the providers change, and when the connection copied from the selected provider changes. `PUT` must send the value from this response.

Errors:

- `500` provider settings could not be read

Events: none.

### GET /v1/recap-settings

Auth: any key
Effect: Reports whether conversation recaps are active. Recaps are active only when the recap switch is on and the recap plugin can run. While it is off, no recap is written and a conversation's recap reads as empty.

Request: no body.

Response: `200`

```json
{ "enabled": true }
```

Errors: none.

Events: none.

### GET /v1/recovery-settings

Auth: any key
Effect: Returns whether interrupted work continues on its own after Core starts. The default is on.

Request: no body.

Response: `200`

```json
{ "auto_resume": true }
```

- `auto_resume` true continues an interrupted turn after restart. False leaves it waiting.

Errors:

- `503` recovery is not running

Events: none.

### GET /v1/running-input-settings

Auth: any key
Effect: Returns what a follow-up message does while a turn is running. `steer` folds it into the running turn. `queue` holds it until that turn finishes. A missing setting means `steer`.

Request: no body.

Response: `200`

```json
{ "mode": "steer" }
```

- `mode` is `steer` or `queue`.

Errors:

- `500` the stored mode is neither `steer` nor `queue`, or the setting could not be read

Events: none.

### POST /v1/complete

Auth: admin
Effect: Runs one model turn for an existing conversation and returns the whole reply as text. There are no tools and no stream. The call is limited to 90 seconds and is recorded on that conversation's usage. The model is the one Core would call for the conversation (an external-agent conversation uses the Settings default when that default is a usable provider model). `provider_id`, `model` and `reasoning_effort` replace that choice when they are non-empty.

Request:

```json
{
  "session_id": "2026-10-14T12-02-00_550e8400-e29b-41d4-a716-446655440000",
  "purpose": "summarize",
  "system": "Answer in one sentence.",
  "max_tokens": 1024,
  "messages": [
    { "role": "user", "text": "Where did the work stop?" }
  ]
}
```

- `session_id` is required. It is an existing conversation id.
- `purpose` is required. It is 1 to 32 characters: a lowercase letter, then lowercase letters, digits or underscores. It must not be `chat`, `compaction`, `memory_review` or `recap`.
- `system` is optional. `messages` is required. Each item is `role` (`user`, `assistant` or `system`) and `text`. `system` roles are appended to `system`, separated by a blank line. The last `user` or `assistant` item must be `user`, and there must be one.
- `max_tokens` defaults to 1024 when omitted or 0, and cannot be above 4096.
- `provider_id` and `model` optionally name an active model of an enabled provider. A non-empty `reasoning_effort` is rejected. Unknown fields are rejected. The body is at most 1 MiB.

Response: `200`

```json
{
  "text": "The work stopped after the settings page.",
  "provider_id": "acme",
  "model": "demo-model",
  "usage": {
    "input_tokens": 120,
    "output_tokens": 18,
    "cache_read_tokens": 0,
    "cache_write_tokens": 0
  }
}
```

- `text` is the reply's text, pieces joined by newlines. It may be `""`. A reply cut off at the output limit is an error, not a short `text`.
- `provider_id` and `model` are the model that answered.
- `usage` counts tokens. `cache_usage_reported` is present only when true.

Errors:

- `400` invalid json
- `400` session_id is required: usage is recorded on a conversation
- `400` purpose must match `^[a-z][a-z0-9_]{0,31}$` and not be one of Core's (`chat`, `compaction`, `memory_review`, `recap`)
- `400` max_tokens is at most 4096
- `400` message role must be user, assistant or system
- `400` messages must end with a user message
- `400` `model <provider_id>/<model>: <reason>` when that model is an external agent, is not an active model of an enabled provider, names a reasoning effort, or has a base URL or API format Core cannot call
- `404` session not found
- `409` no model Core can call for this conversation: set a Settings default model or pass provider_id and model
- `429` too many completions are already running (at most 4 at once)
- `499` request ended
- `502` the model failed, the request was aborted, or the reply was cut off at the output token limit
- `503` storage unavailable; reconnect and retry
- `503` Core is moving its data directory; try again when it finishes
- `504` the model did not answer in time

Events: none.

### POST /v1/memory/jobs/{id}/retry

Auth: admin
Effect: Queues a failed memory review again. Attempts and the error are cleared, and the next try is not delayed. Any other status is refused. Refetch `GET /v1/memory` to see the job.

Path: `id` review job id from `state.jobs`.

Request: no body.

Response: `200`

```json
{ "ok": true }
```

Errors:

- `409` failed review not found, or memory could not be opened

Events: none.

### POST /v1/memory/proposals/{id}

Auth: admin
Effect: Approves or rejects one pending memory proposal, then removes it. Approve applies the operation when `version` still matches the current text. A mismatch leaves the proposal in place. Reject removes it and changes no text. Refetch `GET /v1/memory`. Approving a `project` proposal writes that project's memory; this route's owner snapshot does not include project text.

Path: `id` proposal id from `state.proposals`.

Request:

```json
{ "approve": true }
```

- `approve` is required. `true` applies the proposal. `false` rejects it. Unknown fields are rejected.

Response: `200`

```json
{ "ok": true }
```

Errors:

- `400` invalid settings request
- `400` approve required
- `409` proposal not found
- `409` memory changed since proposal; reject it and review current entries
- `409` the operation does not apply: missing content, the entry is gone or not unique, the text would pass 1375 characters for user or 2200 for memory or project, the action or target is not one of the values above, or one operation contains more than one entry
- `409` memory could not be opened

Events: none.

### POST /v1/provider-settings/fetch

Auth: admin
Effect: Asks a provider for its model list and returns the ids, display names, limits and the extra facts the list reported. Nothing is saved. The client copies what it wants to keep into `PUT /v1/provider-settings`.

Request:

```json
{
  "provider_id": "acme",
  "protocol": "openai",
  "base_url": "https://192.0.2.20/v1",
  "api_key": "example-key"
}
```

- All four fields may be empty. A known `provider_id` fills each empty field from the saved provider. An unknown `provider_id` with an empty `base_url` is refused. A `base_url` on the request is used as sent.
- `protocol` defaults to `openai`. `anthropic`, `google`, `custom` and `ollama` are recognised. Any other value is called as an OpenAI-compatible list. `custom` tries `{base}/models`, then `{base}/v1/models` when the base path does not already end in a version segment such as `v1`. `ollama` sends no key. The call gives up after 20 seconds.
- Unknown fields are ignored.

Response: `200`

```json
{
  "models": [
    {
      "id": "demo-model",
      "name": "Demo",
      "limits": { "context_window": 128000, "max_output": 8192 }
    }
  ],
  "detected_models": {
    "demo-model": {
      "name": "Demo",
      "vendor": "acme",
      "image_input": true,
      "reasoning": true,
      "reasoning_levels": ["low", "high"]
    }
  }
}
```

- `models` is `[]` when the provider lists none. `name` falls back to `id`. `limits` is omitted when `context_window` is outside 4096–10000000 or `max_output` is outside 0–10000000.
- `detected_models` is always an object, `{}` when the list said nothing beyond ids. A key is a model id. `name` is omitted when it equals the id. `vendor`, `image_input`, `reasoning` and `reasoning_levels` are omitted when the list did not say. `image_input` and `reasoning` are booleans, not display text.

Errors:

- `400` invalid json
- `400` unknown provider
- `500` saved provider settings could not be read
- `502` the provider list could not be fetched or parsed, including a missing base URL

Events: none.

### PUT /v1/chat-defaults

Auth: admin
Effect: Sets the Settings default to one active model of an enabled provider. That provider becomes the selected provider, and the model becomes its first active model. Other providers are unchanged. Returns the same object as `GET /v1/chat-defaults`.

Request:

```json
{ "provider_id": "acme", "model": "demo-model", "reasoning_effort": "", "speed": "" }
```

- `provider_id` and `model` are required. The model must be an active model of an enabled provider. An external agent is refused.
- `reasoning_effort` and `speed` must be empty. Unknown fields are ignored. The body is at most 4096 bytes.

Response: `200`

The body is the `GET /v1/chat-defaults` object after the save. `defaults` and `effective` are the saved provider and model, with empty `reasoning_effort` and `speed`.

Errors:

- `400` invalid json
- `400` provider_id and model are required
- `400` the default model must be a model of a provider, not an external agent
- `400` unknown or unavailable model
- `400` reasoning_effort and speed currently require an external agent model
- `500` Core has no home directory, or the setting could not be saved

Events: `chat_defaults.changed` when the default model changed; otherwise none.

### PUT /v1/context-default

Auth: admin
Effect: Stores the context mode used for conversations created after the save. Conversations that already have a mode are unchanged. Returns the same object as `GET /v1/context-default`.

Request:

```json
{ "mode": "window" }
```

- `mode` must be `summary` or `window`. Unknown fields are rejected. The body is at most 128 KiB.

Response: `200`

```json
{ "mode": "window" }
```

Errors:

- `400` invalid settings request
- `400` mode must be summary or window
- `500` the setting could not be saved or read

Events: none.

### PUT /v1/fallback-models

Auth: admin
Effect: Replaces the fallback chain. Entries that repeat an earlier provider and model are dropped. Each kept entry must be an active model of an enabled provider at the time of the save. Returns the same object as `GET /v1/fallback-models`.

Request:

```json
{
  "enabled": true,
  "models": [
    { "provider_id": "acme", "model": "demo-model" },
    { "provider_id": "acme", "model": "demo-small" }
  ]
}
```

- `enabled` defaults to false when omitted. `models` defaults to an empty chain when omitted, so a partial body clears the other half. At most 8 entries are accepted, counted before duplicates are dropped.
- Unknown fields are rejected. The body is at most 128 KiB.

Response: `200`

The body is the saved `{enabled, models}` object. `models` is `[]` when the chain is empty.

Errors:

- `400` invalid settings request
- `400` the fallback chain takes at most 8 models
- `400` not an active model of an enabled provider: `<provider_id>/<model>`
- `500` the setting could not be saved

Events: none.

### PUT /v1/memory

Auth: admin
Effect: Replaces the owner's memory settings and the two memory texts, when `version` is still the one from `GET /v1/memory`. Proposals, jobs and audit are not part of the body. An edit that drops an existing entry records that entry as forgotten and marks queued, running and failed reviews `superseded`. Returns the same object as `GET /v1/memory`, including the new `version`. Use that `version` for the next save.

Request:

```json
{
  "version": "0000000000000000000000000000000000000000000000000000000000000000",
  "user": "Prefers short answers.",
  "memory": "The demo service listens on port 15100.",
  "config": {
    "enabled": true,
    "review_enabled": true,
    "interval": 10,
    "provider_id": "",
    "model": "",
    "skill_learning": false
  }
}
```

- `config`, `user`, `memory` and `version` are all required. `user` and `memory` may be `""`. `version` may not.
- `config.interval` is an integer from 1 to 100. `provider_id` and `model` must both be set or both be empty. The route does not check that the model is on the provider list.
- `user` is at most 1375 characters and `memory` at most 2200, counted in Unicode code points.
- Unknown fields are rejected. The body is at most 128 KiB.

Response: `200`

The body is the `GET /v1/memory` object after the save. Job `transcript` values are `""`.

Errors:

- `400` invalid settings request
- `400` complete memory snapshot required
- `409` review interval must be between 1 and 100
- `409` choose both a provider and a model
- `409` memory exceeds its character limit
- `409` memory changed; refresh before saving
- `500` memory could not be opened, or the saved view could not be read. A failed read after a successful save still returns `500`; the save has been kept.

Events: none.

### PUT /v1/pre-prompt

Auth: admin
Effect: Replaces the pre-prompt text. Omitted `global`, `providers` or `models` are cleared. Returns the same object as `GET /v1/pre-prompt`.

Request:

```json
{
  "global": "Answer in English.",
  "providers": { "acme": "Prefer metric units." },
  "models": { "acme": { "demo-model": "Keep the answer under four sentences." } }
}
```

- Each text is at most 65536 bytes. A provider id or model id that is empty or only whitespace is refused.
- Unknown fields are rejected. The body is at most 128 KiB.

Response: `200`

The body is the stored object. `providers` and `models` are `{}` when empty. `global` is omitted when empty.

Errors:

- `400` invalid settings request
- `400` global pre-prompt is too long
- `400` invalid provider pre-prompt
- `400` invalid model pre-prompt provider
- `400` invalid model pre-prompt
- `500` the setting could not be saved or read

Events: none.

### PUT /v1/provider-settings

Auth: admin
Effect: Replaces the provider list and the selected provider, when `revision` is still the one from `GET /v1/provider-settings`. The connection `GET /v1/models` reports as `current` is then taken from the selected provider, or from the first enabled provider that has an active model. `"providers": []` clears the list and that connection. Returns the saved document, including a new `revision`.

Request:

The body is the `GET /v1/provider-settings` object. Send every provider you want to keep, including `api_key` and the detected maps. A field you omit on a provider is stored empty.

- `revision` must be the current value. An empty or stale revision saves nothing.
- Provider `id` values are non-empty and unique, and none is `local-codex`. `selected_id` is empty or one of those ids.
- `base_url` is empty only when `enabled` is false. When set, it is `http` or `https`, has a host, and has no user name, password, query or fragment.
- `api_format` is empty, `chat_completions` or `responses`. `image_transport` is empty, `inline` or `files`, and `files` requires `responses`.
- Every `model_limits` entry has `context_window` from 4096 to 10000000 and `max_output` from 0 to 10000000. A `model_images` key must be a non-empty model id.
- Unknown fields are rejected, including a second JSON value after the object. The body is at most 2 MiB.

Response: `200`

The body is the saved provider document, the same shape as `GET /v1/provider-settings`, with the new `revision`.

Errors:

- `400` invalid provider settings
- `400` unexpected data after provider settings
- `400` provider id must be nonempty and unique
- `400` local-codex is reserved for the local agent
- `400` `<id>`: base URL required
- `400` `<id>`: use an HTTP(S) provider URL without credentials, query or fragment
- `400` `<id>`: invalid API format
- `400` `<id>`: invalid image transport
- `400` `<id>`: Files image transport requires Responses
- `400` `<id>`: image capability requires a model id
- `400` `<id>`: invalid model limits
- `400` selected provider does not exist
- `400` the settings could not be written
- `409` `配置已在其他窗口更新，请重新加载后再修改` (the document changed since `revision` was read)

Events: `chat_defaults.changed` when the Settings default model changed; otherwise none.

### PUT /v1/recap-settings

Auth: admin
Effect: Turns the recap switch on or off. The response is whether recaps are active after the save, which can stay false when the switch is on but the recap plugin cannot run.

Request:

```json
{ "enabled": false }
```

- `enabled` is required and is a boolean. Unknown fields are ignored. The body is at most 4096 bytes.

Response: `200`

```json
{ "enabled": false }
```

Errors:

- `400` enabled is required
- `404` plugin not found
- `500` the plugin list could not be read, or the switch could not be saved

Events: none.

### PUT /v1/recovery-settings

Auth: any key
Effect: Sets whether interrupted work continues on its own after Core starts. Returns the stored object. An ordinary key may change it.

Request:

```json
{ "auto_resume": false }
```

- `auto_resume` is required and is a boolean. Unknown fields are ignored. The body is at most 4096 bytes.

Response: `200`

```json
{ "auto_resume": false }
```

Errors:

- `400` invalid settings
- `409` update is preparing; try again after restart
- `500` the setting could not be saved
- `503` recovery is not running

Events: none.

### PUT /v1/running-input-settings

Auth: any key
Effect: Sets what a follow-up message does while a turn is running. `steer` folds it into the running turn. `queue` holds it until that turn finishes. An ordinary key may change it. A later `GET` returns this value.

Request:

```json
{ "mode": "queue" }
```

- `mode` must be `steer` or `queue`. Unknown fields are ignored. The body is at most 4096 bytes.

Response: `200`

```json
{ "mode": "queue" }
```

Errors:

- `400` mode must be steer or queue
- `500` Core has no home directory, or the setting could not be saved

Events: none.
