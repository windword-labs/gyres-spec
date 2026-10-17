# External agents

External agents are Codex, Grok, pi and Claude Code installed on the computer that runs Core. A conversation can be handed to one of them. These routes save that setup, report whether the program is installed and signed in, report usage for Codex, Grok and Claude, and for Codex, Grok and Claude Code attach a session that was started in a terminal.

The path values are `codex`, `grok`, `pi` and `claude`. Settings and status are Core-wide, one object per agent. Usage splits into the conversation's own totals and the account's quota.

An error from these handlers is `application/json`:

```json
{"error":"invalid Codex settings","failure":{"code":"core_request_failed","message":"invalid Codex settings","http_status":400,"details":"invalid Codex settings"}}
```

`error`, `failure.message` and `failure.details` are the same text. A `500` message can include a path on the Core host.

Settings and status require an administrator key. Their responses, including errors, send `Cache-Control: no-store`. `PUT` replaces the whole settings object: a missing boolean is saved as false, a missing string as empty, and then the rules below run. The body is at most 8192 bytes. A body that is not JSON, has an unknown field, or is larger than that is rejected.

- `enabled`: the agent can run only when true.
- `executable`: empty, or the bare command name (`codex`, `grok`, `pi` or `claude`), means automatic, and is saved as `""`. Core then uses the newest install it finds. Any other value must be an absolute path.
- `approval_scope`: `agent` (one approval covers every operation of that agent) or `operation` (each operation type is remembered separately). Anything else, including empty, is saved as `agent`.
- `allow_members`: when false (the default), an ordinary access key cannot run this agent, and cannot list or attach its terminal sessions.

`allow_unattended`, on Grok and Claude only, defaults to false. While it is false, a scheduled task or a channel message cannot use that agent.

A status response is the last detection, returned immediately, plus:

- `checked_at`: UTC `2006-01-02T15:04:05Z`, or `null` until the first detection finishes.
- `checking`: true while a detection is running.
- `step`: `""` when idle. While checking it is `program`, then `account`, then `models`. pi goes from `program` to `models`.
- `executable`: the install the saved settings resolve to right now, or `null` when nothing can run. `auto` is true when the setting is automatic. `version` is omitted when unknown.
- `executables`: every install found right now, or `[]`. `source` is a short label for where it was found. `version` is omitted when unknown.

`path`, `version`, `installed`, `logged_in`, `error` and the model list come from the last finished detection and stay visible while `checking` is true. `error` is omitted when that detection did not fail. `GET` starts a detection only when the agent is enabled and none has finished. `POST` starts one unless one is already running; it does not queue a second. `PUT` starts one, and a detection already running is followed by another so the result matches the settings just saved. Neither request waits for the detection.

`input_tokens` on a usage total includes cached input and cache writes. `cache_write_tokens` is omitted when it is 0. `cost_usd` is the agent's own estimate in USD, not a bill; `cost_source` is `agent_estimate` when `cost_usd` is a number, and both are `null` when the agent gave no estimate. A turn that changes a conversation's usage announces `session.agent_usage` with `session_id`. Reading usage does not.

An account object:

- `kind`: `chatgpt`, `api_key`, `subscription` or `unknown`.
- `plan`: omitted when empty.
- `windows`: rate-limit windows, or `[]`. For Codex, `id` is `5h` for 300 minutes, `weekly` for 10080, otherwise `Nd`, `Nh` or `Nm` from the length, or `primary` / `secondary` when the length is missing. A window with no length and `used_percent` 0 is left out. For Claude, `id` is the limit kind Claude names (`session`, `weekly_all`, `weekly_scoped`, and any later kind as sent), several windows may share an `id`, and there is no `duration_mins`; `label` is the model a scoped limit applies to, `severity` is Claude's level (`normal` or a warning level) and `is_active` is true for the limit Claude marks active. Each of these three is omitted when empty or false, and Codex sends none of them. `resets_at` is UTC, omitted when unknown. `duration_mins` is omitted when unknown.
- `credits`: omitted when the account has none. `balance` is the agent's own credit unit, not a currency, and is omitted when it is not a number. `has_credits` and `unlimited` are booleans.
- `unavailable_reason`: `not_signed_in`, `api_key` or `not_supported`. Omitted when `kind` is `chatgpt` and nothing is wrong. This is why `windows` is empty.
- `updated_at`: UTC time of the last successful read, omitted when there is none.

### GET /v1/codex/settings

Auth: admin
Effect: Returns the Codex settings. When none have been saved, the defaults below.

Request: no body.

Response: `200`

```json
{
  "enabled": false,
  "executable": "",
  "sandbox": "workspace-write",
  "approval_scope": "agent",
  "approvals_reviewer": "user",
  "allow_members": false
}
```

- `sandbox` is `read-only` or `workspace-write`. The default is `workspace-write`.
- `approvals_reviewer` is `user` (ask the owner) or `auto_review` (Codex's automatic reviewer). The default is `user`. `auto_review` takes effect only when the detected CLI version is at least `0.144.6`; an older or unknown version still asks the owner.

Errors:

- `500` the saved settings cannot be read

Events: none.

### PUT /v1/codex/settings

Auth: admin
Effect: Replaces the Codex settings and starts a detection. The saved object is returned.

Request:

```json
{
  "enabled": true,
  "executable": "",
  "sandbox": "read-only",
  "approval_scope": "operation",
  "approvals_reviewer": "auto_review",
  "allow_members": false
}
```

- `sandbox` is required: `read-only` or `workspace-write`. An empty value is rejected, not replaced with the default.
- `approvals_reviewer` may be omitted or `""`; it is then saved as `user`. Any other value than `user` or `auto_review` is rejected.
- See the shared settings rules for `enabled`, `executable`, `approval_scope` and `allow_members`.

Response: `200`

```json
{
  "enabled": true,
  "executable": "",
  "sandbox": "read-only",
  "approval_scope": "operation",
  "approvals_reviewer": "auto_review",
  "allow_members": false
}
```

The body is the saved settings, after automatic executables are stored as `""`, `approval_scope` is normalised, and an empty reviewer is stored as `user`.

Errors:

- `400` invalid Codex settings
- `400` use automatic or an absolute codex executable path
- `400` invalid Codex sandbox
- `400` invalid Codex approval reviewer
- `500` the settings could not be saved

Events: none.

### GET /v1/codex/status

Auth: admin
Effect: Returns the last Codex detection without waiting. Starts one only when Codex is enabled and no detection has finished.

Request: no body.

Response: `200`

```json
{
  "path": "/usr/local/bin/codex",
  "version": "0.144.6",
  "installed": true,
  "logged_in": true,
  "auth_type": "chatgpt",
  "models": [
    {
      "serviceTiers": [{"id": "standard", "name": "Standard", "description": ""}],
      "defaultServiceTier": "standard",
      "id": "codex-model",
      "model": "codex-model",
      "displayName": "Codex model",
      "isDefault": true,
      "defaultReasoningEffort": "medium",
      "supportedReasoningEfforts": [{"reasoningEffort": "low"}, {"reasoningEffort": "medium"}]
    }
  ],
  "checked_at": "2026-10-14T12:00:00Z",
  "checking": false,
  "step": "",
  "executable": {"path": "/usr/local/bin/codex", "version": "0.144.6", "auto": true},
  "executables": [{"path": "/usr/local/bin/codex", "version": "0.144.6", "source": "PATH"}]
}
```

- Before the first detection, `installed` and `logged_in` are false, `models` is `[]`, `checked_at` is `null`, and `path`, `version`, `auth_type` and `error` are omitted.
- `logged_in` is true when Codex has an account, or when it does not require a sign-in. `auth_type` is that account's type (`chatgpt` and `apiKey` are the ones Core treats as known) and is omitted when there is no account.
- Model fields keep the names Codex reports. `serviceTiers` and `supportedReasoningEfforts` are `null` when Codex omitted them. `model` is the id a conversation uses; `displayName` is the label.
- A missing program sets `installed` false and `error` to `Codex executable was not found`. A later failure sets `installed` true and `error` to the failure text. Not being signed in leaves `error` omitted.

Errors:

- `500` the saved settings cannot be read
- `500` the status could not be encoded

Events: none.

### POST /v1/codex/status

Auth: admin
Effect: Starts a Codex detection, unless one is already running, and returns the current snapshot immediately. The previous detection stays in the body until the new one finishes; `checking` is then true and `step` starts again at `program`.

Request: no body.

Response: `200` the same object as `GET /v1/codex/status`.

```json
{
  "path": "/usr/local/bin/codex",
  "version": "0.144.6",
  "installed": true,
  "logged_in": true,
  "auth_type": "chatgpt",
  "models": [],
  "checked_at": "2026-10-14T11:00:00Z",
  "checking": true,
  "step": "program",
  "executable": {"path": "/usr/local/bin/codex", "version": "0.144.6", "auto": true},
  "executables": [{"path": "/usr/local/bin/codex", "version": "0.144.6", "source": "PATH"}]
}
```

`models` is the previous list, not cleared at the start of the new detection. With no previous detection, `checked_at` is `null` and the detection fields are the empty ones.

Errors:

- `500` the saved settings cannot be read
- `500` the status could not be encoded

Events: none.

### GET /v1/grok/settings

Auth: admin
Effect: Returns the Grok settings. When none have been saved, the defaults below.

Request: no body.

Response: `200`

```json
{
  "enabled": false,
  "executable": "",
  "sandbox": "off",
  "approval_scope": "agent",
  "allow_unattended": false,
  "allow_members": false
}
```

- `sandbox` is `off`, `workspace` or `read-only`. The default is `off`. An empty value stored earlier is read back as `off`.

Errors:

- `500` the saved settings cannot be read

Events: none.

### PUT /v1/grok/settings

Auth: admin
Effect: Replaces the Grok settings and starts a detection. The saved object is returned.

Request:

```json
{
  "enabled": true,
  "executable": "",
  "sandbox": "workspace",
  "approval_scope": "agent",
  "allow_unattended": false,
  "allow_members": false
}
```

- `sandbox` is required: `off`, `workspace` or `read-only`. An empty value is rejected, not saved as `off`.
- See the shared settings rules for the other fields.

Response: `200`

```json
{
  "enabled": true,
  "executable": "",
  "sandbox": "workspace",
  "approval_scope": "agent",
  "allow_unattended": false,
  "allow_members": false
}
```

Errors:

- `400` invalid Grok settings
- `400` use automatic or an absolute grok executable path
- `400` invalid Grok sandbox
- `500` the settings could not be saved

Events: none.

### GET /v1/grok/status

Auth: admin
Effect: Returns the last Grok detection without waiting. Starts one only when Grok is enabled and no detection has finished.

Request: no body.

Response: `200`

```json
{
  "path": "/usr/local/bin/grok",
  "version": "1.0.50",
  "installed": true,
  "logged_in": true,
  "plan": "pro",
  "models": [
    {
      "id": "grok-model",
      "name": "Grok model",
      "description": "General",
      "reasoning_efforts": ["low", "high"],
      "default_reasoning_effort": "high",
      "context_tokens": 200000
    }
  ],
  "checked_at": "2026-10-14T12:00:00Z",
  "checking": false,
  "step": "",
  "executable": {"path": "/usr/local/bin/grok", "version": "1.0.50", "auto": true},
  "executables": [{"path": "/usr/local/bin/grok", "version": "1.0.50", "source": "PATH"}]
}
```

- Before the first detection, `installed` and `logged_in` are false, `models` is `[]`, `checked_at` is `null`, and `path`, `version`, `plan` and `error` are omitted.
- `logged_in` is true when Grok reports an authenticated account. `plan` is the subscription tier and is omitted when empty. No email or other account identity is included.
- On a model, `description`, `reasoning_efforts`, `default_reasoning_effort` and `context_tokens` are omitted when empty or zero. `id` is the model a conversation uses.
- A missing program sets `error` to `Grok executable was not found` and `installed` false. A later failure sets `installed` true and `error` to the failure text.

Errors:

- `500` the saved settings cannot be read
- `500` the status could not be encoded

Events: none.

### POST /v1/grok/status

Auth: admin
Effect: Starts a Grok detection, unless one is already running, and returns the current snapshot immediately. The previous detection stays until the new one finishes; `checking` is then true and `step` starts again at `program`.

Request: no body.

Response: `200` the same object as `GET /v1/grok/status`, with `checking` true while the new detection runs.

```json
{
  "path": "/usr/local/bin/grok",
  "version": "1.0.50",
  "installed": true,
  "logged_in": true,
  "plan": "pro",
  "models": [],
  "checked_at": "2026-10-14T11:00:00Z",
  "checking": true,
  "step": "program",
  "executable": {"path": "/usr/local/bin/grok", "version": "1.0.50", "auto": true},
  "executables": [{"path": "/usr/local/bin/grok", "version": "1.0.50", "source": "PATH"}]
}
```

Errors:

- `500` the saved settings cannot be read
- `500` the status could not be encoded

Events: none.

### GET /v1/pi/settings

Auth: admin
Effect: Returns the pi settings. When none have been saved, the defaults below.

Request: no body.

Response: `200`

```json
{
  "enabled": false,
  "executable": "",
  "approvals": false,
  "approval_scope": "agent",
  "allow_members": false
}
```

- `approvals`: when true, tool calls are asked through Core before they run. When false, pi runs them without that check. The approval scope applies only while this is true.

Errors:

- `500` the saved settings cannot be read

Events: none.

### PUT /v1/pi/settings

Auth: admin
Effect: Replaces the pi settings and starts a detection. The saved object is returned. An empty object is valid and saves pi disabled, with automatic executable and `approval_scope` `agent`.

Request:

```json
{
  "enabled": true,
  "executable": "/usr/local/bin/pi",
  "approvals": true,
  "approval_scope": "agent",
  "allow_members": false
}
```

Response: `200`

```json
{
  "enabled": true,
  "executable": "/usr/local/bin/pi",
  "approvals": true,
  "approval_scope": "agent",
  "allow_members": false
}
```

`executable` is `""` when the request asked for automatic. `approval_scope` is `agent` unless the request sent `operation`.

Errors:

- `400` invalid pi settings
- `400` use automatic or an absolute pi executable path
- `500` the settings could not be saved

Events: none.

### GET /v1/pi/status

Auth: admin
Effect: Returns the last pi detection without waiting. Starts one only when pi is enabled and no detection has finished. pi has no account: `logged_in` means at least one model can be used.

Request: no body.

Response: `200`

```json
{
  "path": "/usr/local/bin/pi",
  "version": "0.42.0",
  "installed": true,
  "logged_in": true,
  "providers": ["openai"],
  "models": [
    {
      "id": "openai/pi-model",
      "name": "Pi model",
      "provider": "openai",
      "thinking_levels": ["off", "minimal", "low", "medium", "high"],
      "images": true
    }
  ],
  "checked_at": "2026-10-14T12:00:00Z",
  "checking": false,
  "step": "",
  "executable": {"path": "/usr/local/bin/pi", "version": "0.42.0", "auto": false},
  "executables": [{"path": "/usr/local/bin/pi", "version": "0.42.0", "source": "PATH"}]
}
```

- `providers` is always an array, `[]` when there are no models. Each entry is a provider name taken from the model list.
- `id` is `provider/id`. `thinking_levels` is present only for a reasoning model, and is then `off`, `minimal`, `low`, `medium`, `high`. `images` is true when the model accepts image input.
- A missing program sets `error` to `pi executable was not found`. No models, with no `error`, means `logged_in` is false.

Errors:

- `500` the saved settings cannot be read
- `500` the status could not be encoded

Events: none.

### POST /v1/pi/status

Auth: admin
Effect: Starts a pi detection, unless one is already running, and returns the current snapshot immediately. The previous detection stays until the new one finishes; `checking` is then true and `step` starts again at `program`.

Request: no body.

Response: `200` the same object as `GET /v1/pi/status`.

```json
{
  "installed": false,
  "logged_in": false,
  "providers": [],
  "models": [],
  "checked_at": null,
  "checking": true,
  "step": "program",
  "executable": null,
  "executables": []
}
```

Errors:

- `500` the saved settings cannot be read
- `500` the status could not be encoded

Events: none.

### GET /v1/claude/settings

Auth: admin
Effect: Returns the Claude settings. When none have been saved, the defaults below.

Request: no body.

Response: `200`

```json
{
  "enabled": false,
  "executable": "",
  "permission_mode": "default",
  "approval_scope": "agent",
  "allow_unattended": false,
  "allow_members": false
}
```

- `permission_mode` is `default` (ask before edits and commands) or `acceptEdits` (edits in the workspace pass; commands still ask). Anything else is read back as `default`.

Errors:

- `500` the saved settings cannot be read

Events: none.

### PUT /v1/claude/settings

Auth: admin
Effect: Replaces the Claude settings and starts a detection. The saved object is returned.

Request:

```json
{
  "enabled": true,
  "executable": "",
  "permission_mode": "acceptEdits",
  "approval_scope": "agent",
  "allow_unattended": false,
  "allow_members": true
}
```

- `permission_mode` other than `acceptEdits`, including empty, is saved as `default`.

Response: `200`

```json
{
  "enabled": true,
  "executable": "",
  "permission_mode": "acceptEdits",
  "approval_scope": "agent",
  "allow_unattended": false,
  "allow_members": true
}
```

Errors:

- `400` invalid Claude settings
- `400` use automatic or an absolute claude executable path
- `500` the settings could not be saved

Events: none.

### GET /v1/claude/status

Auth: admin
Effect: Returns the last Claude detection without waiting. Starts one only when Claude is enabled and no detection has finished. The result names the kind of sign-in, never a credential.

Request: no body.

Response: `200`

```json
{
  "path": "/usr/local/bin/claude",
  "version": "1.0.80",
  "installed": true,
  "logged_in": true,
  "auth_method": "claude.ai",
  "plan": "Claude Max",
  "models": [
    {
      "id": "claude-model",
      "name": "Claude model",
      "description": "General",
      "efforts": ["low", "medium", "high"]
    }
  ],
  "checked_at": "2026-10-14T12:00:00Z",
  "checking": false,
  "step": "",
  "executable": {"path": "/usr/local/bin/claude", "version": "1.0.80", "auto": true},
  "executables": [{"path": "/usr/local/bin/claude", "version": "1.0.80", "source": "PATH"}]
}
```

- `auth_method` is the sign-in kind Claude reports: `claude.ai` (a subscription), `api_key`, `oauth_token`, a cloud provider's name, or `none`. It is omitted when empty.
- `api_key_source` is omitted above. When present, it names where an API key comes from and never includes the key. `api_provider` is omitted when empty.
- `key_overrides_subscription` is present and true only when a signed-in API key is the active credential, so usage is billed to the API rather than a subscription. Otherwise the field is absent.
- `plan` is the subscription's display name when the model list provided one, otherwise the raw tier. It is omitted when empty.
- On a model, `description` and `efforts` are omitted when empty. `id` is the id or alias Claude accepts. `efforts` is the effort names Claude reports for a model that supports them.
- A missing program sets `error` to `Claude executable was not found`. A sign-in that cannot be read sets `error` to `Could not read Claude's sign-in state: ` plus the failure. Signed out leaves `logged_in` false and `error` omitted.

Errors:

- `500` the saved settings cannot be read
- `500` the status could not be encoded

Events: none.

### POST /v1/claude/status

Auth: admin
Effect: Starts a Claude detection, unless one is already running, and returns the current snapshot immediately. The previous detection stays until the new one finishes; `checking` is then true and `step` starts again at `program`.

Request: no body.

Response: `200` the same object as `GET /v1/claude/status`.

```json
{
  "path": "/usr/local/bin/claude",
  "version": "1.0.80",
  "installed": true,
  "logged_in": false,
  "models": [],
  "checked_at": "2026-10-14T11:00:00Z",
  "checking": true,
  "step": "account",
  "executable": {"path": "/usr/local/bin/claude", "version": "1.0.80", "auto": true},
  "executables": [{"path": "/usr/local/bin/claude", "version": "1.0.80", "source": "PATH"}]
}
```

Errors:

- `500` the saved settings cannot be read
- `500` the status could not be encoded

Events: none.

### GET /v1/sessions/{id}/agent-usage

Auth: any key
Effect: Returns the Codex, Grok or Claude usage of one conversation, plus that agent's account quota. Any other conversation, including pi, answers with empty parts. Codex account data may wait on a read shared with the account route, up to 20 seconds; a failed read keeps the previous account. Claude account data waits only for the first read since Core started (up to 30 seconds); later an answer older than 60 seconds is served while it is read again, and that read announces `session.agent_usage` for this conversation.

Path: `id` conversation id, `2026-10-14T12-02-00_` plus a UUID. An unknown id or a conversation that does not exist is `404`.

Request: no body.

Response: `200`

```json
{
  "agent": "codex",
  "account": {
    "kind": "chatgpt",
    "plan": "pro",
    "windows": [
      {"id": "5h", "used_percent": 42, "resets_at": "2026-10-14T18:00:00Z", "duration_mins": 300},
      {"id": "weekly", "used_percent": 87, "resets_at": "2026-10-18T00:00:00Z", "duration_mins": 10080}
    ],
    "credits": {"balance": 62500, "has_credits": true, "unlimited": false},
    "updated_at": "2026-10-14T12:00:00Z"
  },
  "session": {
    "model": "codex-model",
    "turns": 9,
    "input_tokens": 171000,
    "cached_input_tokens": 142500,
    "cache_write_tokens": 800,
    "output_tokens": 13200,
    "reasoning_tokens": 4000,
    "total_tokens": 184200,
    "cost_usd": 0.83,
    "cost_source": "agent_estimate",
    "last_turn": {"total_tokens": 8100, "cost_usd": 0.06},
    "context": {"used_tokens": 62000, "window_tokens": 200000},
    "updated_at": "2026-10-14T12:00:00Z"
  }
}
```

```json
{"agent": null, "account": null, "session": null}
```

- `agent` is `codex`, `grok` or `claude`, or `null` when this conversation is not one of those.
- `session` is `null` until the agent has reported usage. Totals are cumulative as the agent reports them. For a session attached from a terminal they include turns from before the attach. `turns` counts turns finished here for Codex and Claude, and every turn of the session for Grok, because Grok numbers them itself.
- `last_turn` is `null` until a turn has been counted. Its `total_tokens` is that turn's own total. Its `cost_usd` is `null` when that turn's estimate is unknown (the previous estimate was missed, or a thread or session attached from a terminal had usage but no earlier estimate).
- `context` is `null` when no window is known. For Codex, `used_tokens` is the last request's total and `window_tokens` is the model window. For Grok, `used_tokens` is the last turn's input divided by the number of model calls in that turn (exact for one call, lower than the final call when there were several), and `window_tokens` is the context size of the model selected for the conversation, or the session's own figure. Grok's `context` is also `null` when that last turn made no model call. For Claude, `used_tokens` is the last request's input including cache reads and writes, and `window_tokens` is the context window Claude reports for the turn's model.
- `account` is the account object described above. It is `null` for Codex until a read has succeeded. For Grok it is always present: `kind` is `subscription`, `windows` is `[]`, `unavailable_reason` is `not_supported`, and `plan` is the tier from the last Grok detection, omitted when empty. Grok sends no `credits` or `updated_at`.
- A Codex account with `kind` `api_key` has `unavailable_reason` `api_key` and `windows` `[]`. `kind` `unknown` uses `not_signed_in` when nobody is signed in, or `not_supported` for any other account type.
- Claude's account is `null` until a read has succeeded (also while Claude is disabled). A subscription sign-in has `kind` `subscription` and `plan` (such as `max`); its `windows` are the plan limits, or `[]` with `unavailable_reason` `not_supported` when Claude reported none. An API key is `kind` `api_key` with `unavailable_reason` `api_key`. Signed out is `kind` `unknown` with `not_signed_in`; another sign-in (a cloud provider) is `unknown` with `not_supported`. `credits` is present only when Claude's extra usage is enabled: `has_credits` is true, `balance` is what is left of the monthly limit, or `null` when unknown, and `unlimited` is true when there is no monthly limit.

Errors:

- `404` session not found
- `500` a binding cannot be read
- `500` cannot read agent usage

Events: none.

### GET /v1/local-agents/{agent}/account-usage

Auth: any key
Effect: Returns the same account object as the conversation usage route, before a conversation exists. `codex` may wait on the shared read (20 seconds, at most once a minute; a failed read keeps the last account). `grok` does not start Grok; it answers from the last detection. `claude` shares the conversation route's cache and waits only for the first read.

Path: `agent` is `codex`, `grok` or `claude`.

Request: no body.

Response: `200`

```json
{
  "kind": "chatgpt",
  "plan": "pro",
  "windows": [
    {"id": "5h", "used_percent": 42, "resets_at": "2026-10-14T18:00:00Z", "duration_mins": 300}
  ],
  "credits": {"balance": 62500, "has_credits": true, "unlimited": false},
  "updated_at": "2026-10-14T12:00:00Z"
}
```

```json
null
```

- Codex and Claude are `null` until a read has succeeded. The fields are the account object on `GET /v1/sessions/{id}/agent-usage`.
- Grok is never `null`:

```json
{"kind": "subscription", "windows": [], "unavailable_reason": "not_supported"}
```

`plan` is added when the last Grok detection reported a tier.

Errors:

- `404` unknown external agent

`pi`, `claude` and any other name use this same `404`.

Events: none.

### GET /v1/local-agents/{agent}/terminal-sessions

Auth: any key (admin for an agent that does not allow non-administrators)
Effect: Lists sessions the agent started on this computer outside Core, newest first, 30 a page. Codex includes threads from the CLI, an editor extension, a one-shot run and the desktop app, and leaves subagent threads out. Grok includes its stored sessions. Claude Code includes every session it saved on this computer (read from its own session files; subagent transcripts and sessions without a prompt or a reply are left out). A successful page may be reused for 15 seconds; whether a session is open, its directory is missing, or it is already attached is computed on every response. The request waits for the agent, up to 20 seconds.

Path: `agent` is `codex`, `grok` or `claude`. `pi` is known but unsupported.

Query: `scope` is `dir` or `all`. Omitted, it is `all`, or `dir` when `cwd` is present. `cwd` is the directory for `scope=dir` and must be absolute; it is ignored for `all`. `scope=dir` lists sessions whose directory is `cwd` or anywhere below it, comparing real paths: both sides are cleaned (a trailing `/` removed) and symbolic links are resolved as far as the path exists, then compared exactly (case-sensitively, also on a case-insensitive volume). `/a/bc` is not below `/a/b`. `cursor` is the `next_cursor` of the previous page, omitted for the first page. Do not parse it.

Request: no body.

Response: `200`

```json
{
  "sessions": [
    {
      "agent": "grok",
      "id": "thread-7c2e9a",
      "title": "Fix the login form",
      "cwd": "/home/dev/app/web",
      "relative_cwd": "web",
      "updated_at": "2026-10-14T11:40:00Z",
      "preview": "The login form rejects valid passwords",
      "model": "grok-model",
      "sandbox": "workspace",
      "message_count": 12,
      "in_use": false,
      "cwd_missing": false,
      "attached_conversation_id": "2026-10-13T09-15-22_018f3a2b-7c4e-7a1b-8d2e-9a0b1c2d3e4f"
    }
  ],
  "next_cursor": "page-2"
}
```

- `sessions` is `[]` when there is nothing on this page. `next_cursor` is omitted on the last page. Rows Core started itself are removed, so a page can be shorter than 30 while `next_cursor` is still set. Keep paging until it is absent.
- `id` is the agent's own session id. `cwd` is the agent's own directory string and must be sent back unchanged to attach. `title` may be `""`. Codex and Claude titles are one line and at most 100 characters, using the thread name (Claude: the name given with `/rename`, else Claude's generated title) or else the start of its first message. Grok titles are as Grok reports them.
- `relative_cwd` (`scope=dir` only) is the session's directory below `cwd`, `/` separated, such as `web` or `mywork/gyres`. It is omitted when the session is in `cwd` itself and for `scope=all`. It is for display; attach with `cwd`.
- `updated_at` is UTC RFC3339 for Codex and Claude (for Claude, when its session file was last written). For Grok it is the timestamp string Grok reported.
- `preview` is the start of the first prompt on one line, at most 120 characters, ending with `…` when cut. It is omitted when empty.
- `model` is omitted when empty. `sandbox` and `message_count` are filled for Grok when it has a summary, and omitted for Codex (Codex does not report them on this list). Claude has no sandbox; its `model` is the full model ID of the last reply (such as `claude-opus-5-5`) and `message_count` counts prompts and replies with text. `message_count` is also omitted when 0.
- `in_use` is true when a terminal has the session open. `cwd_missing` is true when `cwd` is not a directory on the Core host. `attached_conversation_id` is set when this session was attached and that conversation still exists. A session Core started stays hidden after its conversation is deleted. An attached terminal session is listed again after its conversation is deleted, and can be attached again.

Errors:

- `400` scope must be dir or all
- `400` cwd must be an absolute directory on the Core host
- `400` Terminal sessions are not supported for pi yet
- `400` Codex is disabled in Settings → External agents
- `400` Grok is disabled in Settings → External agents
- `400` Claude is disabled in Settings → External agents
- `400` Codex executable was not found
- `400` Grok executable was not found
- `400` invalid cursor
- `403` Ask an administrator to turn on "Allow non-administrators" for Codex in Settings → External agents.
- `404` unknown external agent
- `500` the agent could not list sessions, or the bindings could not be read

The `403` names the agent (`Codex`, `Grok` or `Claude`). `pi` is rejected with the unsupported `400` before that check. `invalid cursor` is the answer to a cursor Core did not issue (Claude, and `scope=dir` for every agent). The disabled and missing-program `400` name whichever agent was asked.

Events: none.

### POST /v1/local-agents/{agent}/terminal-sessions/{external_id}/attach

Auth: any key (admin for an agent that does not allow non-administrators)
Effect: Binds a terminal session to a conversation and copies its history into the transcript. A new conversation is created when this session is not already attached. Attaching it again returns the conversation it already has; if that import never finished, the history is imported again and rows already written are not repeated. A failed create removes the new conversation. The call waits for the import and stops after 5 minutes.

Path: `agent` is `codex`, `grok` or `claude`. `external_id` is the `id` from the list. A blank id is rejected.

Request:

```json
{"cwd": "/home/dev/app", "project_id": "018f3a2b-7c4e-7a1b-8d2e-9a0b1c2d3e4f"}
```

- `cwd` is required. It must be absolute and must be the listed `cwd` in that exact spelling, with nothing added. The body is at most 16384 bytes. Extra fields are ignored.
- `project_id` is optional, a project UUID. When set, the conversation joins that project and `cwd` must sit inside one of its directories. When omitted, the conversation's workspace is `cwd` itself.

Response: `200`

```json
{
  "session_id": "2026-10-14T12-02-00_018f3a2b-7c4e-7a1b-8d2e-9a0b1c2d3e4f",
  "existing": false
}
```

- `session_id` is the conversation. `existing` is false when this call created it, and true when the session was already attached to a conversation that still exists.
- Imported transcript rows carry `importedFrom` `terminal`. They are history the agent already has, so they are not sent again. A Codex thread whose sandbox is neither `read-only` nor `workspace-write` continues under the saved Codex sandbox. A Claude session continues under the saved Claude permission mode and Gyres approvals, whatever permission mode it ran with in the terminal; its origin has no `sandbox`.
- The conversation's title is set from the session when it has one and the conversation has none. Codex keeps the thread's model and effort when that model is offered; an effort the model does not offer is dropped, and a model that is not offered is replaced by the first offered model with the effort cleared. With no model list yet, both are kept. Grok keeps the session's model when it has one, and otherwise uses the first model Grok offers. Claude maps the session's full model ID to the offered model of the same ID or family (`claude-opus-5-5` becomes `opus`), else the first offered model; effort follows the Codex rule.

Errors:

- `400` cwd must be the session's directory as listed
- `400` missing session id
- `400` cwd must be the session's directory exactly as listed
- `400` Terminal sessions are not supported for pi yet
- `400` Codex is disabled in Settings → External agents
- `400` Grok is disabled in Settings → External agents
- `400` Claude is disabled in Settings → External agents
- `400` Codex executable was not found
- `400` Grok executable was not found
- `400` project not found
- `400` current directory must be inside a project directory
- `400` invalid project cwd
- `400` create the project library, or create the conversation directory, when that directory cannot be created
- `403` Ask an administrator to turn on "Allow non-administrators" for Codex in Settings → External agents.
- `404` unknown external agent
- `404` Claude session file not found
- `404` invalid Claude session ID
- `409` This Codex session already belongs to a Gyres conversation
- `409` This session is open in a terminal. Exit it there first, then attach it.
- `409` The session's directory no longer exists on the Core host
- `409` turn already running
- `500` the history could not be imported, or the conversation could not be created

`invalid project cwd` includes the path and the reason. The "already belongs" `409` names the agent (`Codex`, `Grok` or `Claude`) and is a session Core itself started. `turn already running` also covers a conversation that cannot accept the import right now. The `403` names the agent, as on the list route.

Events: `sessions.changed`, `session.messages`, `session.settings`

A new conversation announces `sessions.changed`. Imported rows announce `session.messages`. Saving a model announces `session.settings`, and setting a title announces `sessions.changed` again. An already imported session announces nothing. A failed create announces `sessions.changed` as the new conversation is removed.
