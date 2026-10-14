# Plugins, MCP, hooks, computer use and host screen

Plugins, host hooks, MCP servers, computer use and host-screen viewing are Core-wide settings. A plugin is one named add-on: hooks, skills, tools, optional HTML pages and optional MCP servers, turned on or off together. Host hooks are commands Core runs at lifecycle points, separate from plugins. MCP servers are connections whose tools Core can call. Computer use lets an agent operate ordinary apps on the computer running Core. Host screen is a live view of that computer's displays, with optional remote control on the same WebRTC connection.

An add-on runs only when `enabled` is true, `error` is empty and `missing` is empty. One that ships with Core (`source` `bundled`) starts on; one placed on the computer (`source` `plugin`) starts off until an administrator turns it on. `required` is true only for a bundled add-on that cannot be turned off. A directory with the same name replaces the bundled one and is then `source` `plugin`.

JSON responses are `application/json`. Every route here except the four MCP routes and `POST /v1/plugins/{name}/tools/{tool}` sends `Cache-Control: no-store`. A failed request uses the usual error body (`error` plus `failure`) except where an endpoint below shows a different body. Paths in these objects are on the computer running Core.

Changing plugins publishes `plugins.changed`. Read `GET /v1/plugins` again, and `GET /v1/mcp` again when the change can add or remove a plugin's MCP servers.

### GET /v1/plugins

Auth: any key
Effect: Lists every plugin as scanned now, plus the host hooks file. Nothing is changed.

Request: no body.

Response: `200`

```json
{
  "plugins_dir": "/var/lib/gyres/plugins",
  "host_hooks": {
    "path": "/var/lib/gyres/hooks.json",
    "exists": true,
    "error": "",
    "max_continuations": 2,
    "handlers": []
  },
  "plugins": [
    {
      "name": "hello-gyres",
      "version": "1.2.0",
      "description": "Example plugin",
      "source": "plugin",
      "enabled": true,
      "required": false,
      "path": "/var/lib/gyres/plugins/hello-gyres",
      "error": "",
      "warnings": [],
      "missing": [],
      "library_note": "Prefer the project library for notes.",
      "handlers": [
        {
          "id": "hello-gyres/on-run",
          "event": "RunStart",
          "command": ["python3", "/var/lib/gyres/plugins/hello-gyres/hook.py"],
          "mode": "command",
          "timeout_ms": 10000,
          "on_error": "continue",
          "env": { "TOKEN": "" }
        }
      ],
      "skills": [
        { "name": "hello", "path": "/var/lib/gyres/plugins/hello-gyres/skills/hello/SKILL.md" }
      ],
      "tools": [
        {
          "name": "hello-gyres.look",
          "model_name": "hello-gyres__look",
          "description": "Look up a word",
          "side_effect": "read",
          "replay": "",
          "deferred": false,
          "handler": "hello-gyres/look"
        }
      ],
      "mcp": [
        {
          "name": "api",
          "transport": "http",
          "command": "",
          "args": null,
          "url": "https://192.0.2.20/mcp",
          "env": null,
          "headers": ["Authorization"]
        }
      ],
      "settings": [
        {
          "key": "region",
          "type": "string",
          "secret": false,
          "label": "Region",
          "description": "",
          "required": false,
          "value": "us"
        }
      ],
      "web": {
        "panel": "ui/panel.html",
        "page": {
          "file": "ui/page.html",
          "title": "Hello",
          "icon": "puzzle"
        }
      },
      "installed": {
        "source": "git",
        "url": "https://192.0.2.10/plugins/hello-gyres.git",
        "ref": "main",
        "ref_kind": "branch",
        "commit": "0123456789abcdef0123456789abcdef01234567",
        "version": "1.2.0",
        "installed_at": "2026-10-14T12:00:00Z",
        "pinned": false
      }
    }
  ]
}
```

- `plugins_dir` is the directory that holds installed plugins. `plugins` is in name order.
- `host_hooks` is the same object as `GET /v1/hooks`.
- `name` is lowercase letters, digits and single hyphens, at most 64 characters. `source` is `plugin` or `bundled`. `enabled` and `required` are booleans. `path` is the plugin directory.
- `error` is empty when the plugin loaded. Otherwise it is why the plugin contributes nothing. `warnings` does not stop the plugin. A missing or non `MAJOR.MINOR.PATCH` version (an optional `-pre-release` and `+build` are allowed) is a warning, not an error.
- `missing` lists required setting keys that have no value. An empty string counts as no value. A non-empty `missing`, a non-empty `error`, or `enabled` false means the plugin is not running.
- `library_note` is present only when the plugin declares one.
- `commands` are the plugin's slash commands (empty when none): `name`, `description`, optional `descriptions` (by language tag, such as `zh` or `zh-Hant`), `argument_hint`, `argument_hints` and `subcommands` (`name`, `description`, `descriptions`, `takes_argument`). A command is sent as an ordinary message, `/<name> <text>`, in a conversation where the plugin runs; list them in your `/` menu for Gyres conversations (not external agents).
- `handlers` are the plugin's hooks. `id` is `name/` plus the id from the plugin. A command argument that was written `./...` is an absolute path. `mode` is `command` (one process per event) or `process` (one long-lived process). `timeout_ms` and `on_error` (`continue` or `stop`) are filled in when omitted in the file. `env` is present only when the handler has environment keys, and every value is `""`. `tool`, `slash`, `project`, `source`, `purpose` and `executor` appear only when set. Setting values are not included.
- `skills` is empty when the plugin has no skills directory. Each `path` is that skill's file.
- `tools` is what the plugin declares. `name` is `plugin.tool` (the name people see). `model_name` is `plugin__tool` (the name a model calls). The path of `POST /v1/plugins/{name}/tools/{tool}` uses the short tool name, `look` in the example, not `name` or `model_name`. `side_effect` is `read` or `write` (`write` when omitted). `replay` is `safe`, `never`, `interrupt`, or `""`. `handler` is the namespaced handler id. The argument schema is not included.
- `mcp` is the servers the plugin declares. `transport` is `stdio` or `http`. `env` and `headers` are sorted names only (`null` when there are none); values are never included. `args` is an array, or `null` when omitted.
- `settings` is the declared list, which may be empty. `type` is `string`, `number`, `boolean` or `enum`. `secret`, `label`, `description` and `required` are always present. `default` and `options` are present only when declared (`options` is for `enum`). A secret has `set` (boolean) and never `value`. A non-secret has `value` only when a value is currently in force.
- `web` is `null` when the plugin has no pages. Otherwise it has the keys that are declared: `panel` and `widget` are paths such as `ui/panel.html`, and `page` is `{ "file", "title", "icon", "titles"?, "badge"?, "show_at_zero"? }`. `icon` is `alert`, `inbox`, `list`, `chart`, `bell`, `book`, `flag` or `puzzle`. `titles` maps a language tag such as `zh` to a title. `badge` names one of the plugin's own read tools.
- `installed` is present only for a plugin installed from these routes (`source` of the plugin is `plugin`). `source` inside it is `git`, `path` or `archive`. `url` has no user name, password, query or fragment. `ref_kind` is `branch`, `tag`, `commit`, or absent when the fetch did not say. `commit` is the full commit hash. `sha256` is present for an archive and is the hex digest of the zip. `pinned` is present when true. `installed_at` is UTC, to the second. `update`, when present, is the last check: `checked_at`, `available`, and `remote_commit` when one was seen. A later install clears it.

Errors:

- `500` the plugins directory could not be read

Events: none.

### GET /v1/hooks

Auth: any key
Effect: Reads the host hooks file. An invalid file is still `200`: `error` explains it and `handlers` is empty. Environment values are blank. Nothing is changed.

Request: no body.

Response: `200`

```json
{
  "path": "/var/lib/gyres/hooks.json",
  "exists": true,
  "error": "",
  "max_continuations": 2,
  "handlers": [
    {
      "id": "on-run",
      "event": "RunStart",
      "command": ["python3", "hook.py"],
      "mode": "command",
      "timeout_ms": 10000,
      "on_error": "continue",
      "env": { "TOKEN": "" }
    }
  ]
}
```

When the file is missing, `exists` is false, `error` is `""`, `handlers` is `[]`, and `max_continuations` is omitted. When the file exists but cannot be read or is invalid, `exists` is true if the file is still there, `error` is the reason, `handlers` is `[]`, and `max_continuations` is omitted.

- `path` is the hooks file on the computer running Core.
- `max_continuations` is an integer from 0 to 5. When the file omits it, the value is 2.
- Handler fields match the plugin `handlers` above, except `id` is not prefixed and a `./` command is left as written. `env` values are `""`.
- `event` is one of `RunStart`, `RunEnd`, `BeforePrompt`, `BeforeModel`, `AfterModel`, `ModelError`, `BeforeTool`, `AfterTool`, `BeforeCompact`, `AfterCompact`, `BeforeFinish`, `ApprovalRequested`, `ApprovalResolved`, `RunPaused`, `RunResumed`, `BeforeSystemPrompt`, `RunSettled`. `Tool` is not valid in this file.
- `executor`, only on `RunSettled`, is `core`, `codex`, `grok`, `pi` or `claude`.

Errors: none.

Events: none.

### GET /v1/plugins/{name}/ui/{path...}

Auth: any key
Effect: Returns one file under the running plugin's `ui/` directory. The plugin must be enabled, loaded and not missing required settings. The file is not changed.

Path: `name` plugin name. `path` is the file relative to `ui/`, for example `index.html` or `assets/app.js` for `ui/index.html` and `ui/assets/app.js`.

Request: no body.

Response: `200` the file bytes.

- `Content-Type` is `text/html; charset=utf-8` for `.html` and `.htm`, `image/svg+xml` for `.svg`, `text/javascript; charset=utf-8` for `.js` and `.mjs`, `text/plain; charset=utf-8` for `.md`, `.markdown`, `.txt`, `.log`, `.csv` and `.tsv`, otherwise the standard type for the extension, or `application/octet-stream`.
- Also sent: `X-Content-Type-Options: nosniff`, `Referrer-Policy: no-referrer`, and a `Content-Security-Policy` of `sandbox allow-scripts` with `default-src 'none'`. Scripts may come from inline code, `eval`, `blob` and a fixed list of public CDNs. Styles may come from inline code and that list. Fonts may come from `data` and that list. Images and media may come from `data` and `blob`. Workers may come from `blob`. `connect-src`, `frame-src`, `form-action` and `base-uri` are `'none'`.
- The page cannot call Core. A file larger than 512 KiB is not returned.

Errors:

- `404` plugin page not found (unknown or inactive plugin, a path that is empty, absolute, hidden, or contains `..`, `\`, or an empty segment, or a missing file, a non-file, or a file over 512 KiB)
- `500` the plugin list could not be read

Events: none.

### POST /v1/plugins/rescan

Auth: admin
Effect: Scans plugins again and drops enable flags, install records and secrets that belong to names no longer present. Enable flags that only turn a still-present plugin off are kept. Returns the same body as `GET /v1/plugins`.

Request: no body.

Response: `200` the `GET /v1/plugins` body.

Errors:

- `500` the scan failed; the previous list is kept

Events: `plugins.changed`

### POST /v1/plugins/install

Auth: admin
Effect: Fetches a plugin and places it under `plugins_dir` under its manifest name. A new install is enabled. Replacing a plugin that ships with Core leaves the replacement off unless it was already turned on. An install over an existing name needs `replace`. Enable state of a replacement follows that rule; settings already stored for the name are kept. Failure after the files move puts the previous files and install record back.

Request:

```json
{
  "source": "git",
  "url": "https://192.0.2.10/plugins/hello-gyres.git",
  "ref": "main",
  "subdir": "plugin",
  "replace": false,
  "pin": false
}
```

The body is one JSON object of at most 16 KiB. Unknown fields are rejected.

- `source` (string, required) is `git`, `path` or `archive`.
- `url` (string) is the git remote, or the `https` URL of a zip. A git URL may be `https`, `http`, `ssh`, `file`, or `user@host:path`. It must not start with `-` or contain spaces or control characters, and is at most 2048 characters. User information and the query are not stored.
- `path` (string) is an absolute directory on the computer running Core (`source` `path`), or an absolute `.zip` file (`source` `archive`).
- `ref` (string) is a branch, tag or commit for `git` only. Empty means the remote's default branch. At most 256 characters, matching a branch, tag or commit name, with no `..`.
- `subdir` (string) is a relative directory inside the fetched tree. Empty means the root (a zip with a single top-level directory uses that directory).
- `replace` (boolean) must be true to replace a plugin that is already present. Without it, an existing plugin returns `409`.
- `pin` (boolean) keeps a git install on the fetched commit. Ignored unless the install is git.
- `git` uses `url` and `ref`, not `path`. `path` uses `path` only. `archive` uses exactly one of `path` or `url`, and no `ref`. A git fetch or a zip download gives up after 3 minutes. A zip larger than 64 MiB is refused. The unpacked tree is at most 10,000 files and 256 MiB.

Response: `200` the new plugin object, the same shape as one `plugins` entry.

Errors:

- `400` invalid install request, or the source or the fetched files were rejected (the message says which; a git failure and a bad manifest are included)
- `409` `plugin <name> <version> is already installed`, or, when that name ships with Core, `plugin <name> <version> is bundled with Core; installing replaces it and starts turned off`
- `500` the files could not be written, or the scan failed

Events: `plugins.changed` on success.

### PUT /v1/plugins/{name}

Auth: admin
Effect: Turns the named plugin on or off. A bundled plugin is on unless this route has turned it off. A plugin under `plugins_dir` is off unless this route has turned it on. Returns the plugin object.

Path: `name` plugin name.

Request:

```json
{ "enabled": true }
```

At most 4096 bytes. Unknown fields are rejected. `enabled` is required.

Response: `200` the plugin object.

Errors:

- `400` `enabled` is required
- `404` plugin not found
- `409` this plugin is built into Core and cannot be turned off. The body adds a top-level `code`:

```json
{
  "error": "this plugin is built into Core and cannot be turned off",
  "code": "plugin_required",
  "failure": {
    "code": "core_request_failed",
    "message": "this plugin is built into Core and cannot be turned off",
    "http_status": 409,
    "details": "this plugin is built into Core and cannot be turned off"
  }
}
```

- `500` the change could not be saved, or the scan failed

Events: `plugins.changed` on success.

### PUT /v1/plugins/{name}/settings

Auth: admin
Effect: Changes the named plugin's settings. Omitted keys stay as they are. `null` clears a key: a secret is deleted, a non-secret returns to its default. Non-secret values are stored as settings; secrets are stored separately and never come back in a response. Long-lived hook processes of this plugin are stopped so the next event starts them with the new values. Returns the plugin object.

Path: `name` plugin name.

Request:

```json
{ "values": { "region": "us", "api_token": null } }
```

At most 512 KiB. Unknown fields outside `values` are rejected. `values` is required.

- Each key must be declared by the plugin. A non-secret `string` is at most 8 KiB and must not contain a NUL. A `number` is a JSON number. A `boolean` is `true` or `false`. An `enum` must be one of `options`. A secret must be a non-empty string, or `null` to clear it. An empty string is stored for a non-secret string; for a secret it clears the secret.
- A plugin with `error` set is rejected before any value is written.

Response: `200` the plugin object. Secret fields show `set` and not the text.

Errors:

- `400` `values` is required
- `400` invalid settings. The body is not the usual error envelope:

```json
{ "error": "invalid settings", "fields": { "region": "expected a string" } }
```

`fields` maps each rejected key to a reason: `not declared by the plugin`, `invalid JSON`, `expected a string`, `expected a number`, `expected true or false`, `expected one of the options`, or `at most 8 KiB of text`. Values are not echoed.

- `400` `the plugin has an error: ` plus the plugin's `error`
- `404` plugin not found
- `500` the values could not be saved, or the scan failed

Events: `plugins.changed` on success. An empty `values` object still counts as success.

### POST /v1/plugins/{name}/check-update

Auth: admin
Effect: Compares a git install's recorded commit with the commit its ref names now, and stores that result on the install record. A pinned plugin, including one installed from a commit, is not looked up.

Path: `name` plugin name.

Request: no body.

Response: `200`

```json
{
  "current_commit": "0123456789abcdef0123456789abcdef01234567",
  "update_available": true,
  "pinned": false,
  "remote_commit": "abcdef0123456789abcdef0123456789abcdef01",
  "checked_at": "2026-10-14T12:05:00Z"
}
```

- `pinned` is true when the install is pinned or `ref_kind` is `commit`. Those responses omit `remote_commit` and `checked_at`, and `update_available` is false.
- Otherwise `update_available` is whether `remote_commit` differs from `current_commit`. `checked_at` is UTC, to the second. The same result is then visible on the plugin's `installed.update`.

Errors:

- `404` plugin not found
- `400` a bundled plugin ships with Core and is updated with it
- `400` this plugin was placed by hand; there is no source to update it from
- `400` this plugin was installed from a zip archive; update it by installing the new archive with Replace
- `400` this plugin was installed from a directory on the Core host; update it by installing it again
- `400` the recorded source could not be asked (the message says which, including a 3 minute timeout)
- `500` the scan failed, or the result could not be stored

Events: `plugins.changed` when the remote lookup succeeds and the install record can be written, including when that write leaves the record unchanged because the plugin changed. A pinned response publishes nothing.

### POST /v1/plugins/{name}/update

Auth: admin
Effect: Installs the named plugin again from its recorded git source. Settings, secrets and the on/off state stay. A failure puts the previous files back. The new version must load.

Path: `name` plugin name.

Request:

```json
{ "ref": "v1.3.0", "pin": true }
```

At most 4096 bytes. An empty body is allowed. Unknown fields are rejected.

- `ref` (string, optional) fetches that ref instead of the recorded one. Blank space around it is removed. The same ref as the one already recorded is not a change.
- `pin` (boolean, optional) sets whether the install stays on a commit. Omitted keeps the recorded choice. When it stays pinned and `ref` did not change, the recorded commit is fetched again and the recorded ref is kept.

Response: `200` the plugin object.

Errors:

- `400` invalid update request
- `400` the source now holds a different plugin name (the message names both)
- `400` the new version does not load, so the installed one was kept, followed by the load error
- `404` and the other `400` cases of `POST /v1/plugins/{name}/check-update` (no git source)
- `500` the scan or the write failed

Events: `plugins.changed` on success.

### PUT /v1/plugins/{name}/pin

Auth: admin
Effect: Sets whether a git install stays on its commit. Nothing is fetched. Pinning clears the last update check. A plugin whose `ref_kind` is `commit` stays treated as pinned for checks and updates even when this flag is false. Returns the plugin object.

Path: `name` plugin name.

Request:

```json
{ "pinned": true }
```

At most 4096 bytes. Unknown fields are rejected. `pinned` is required.

Response: `200` the plugin object. `installed.pinned` is omitted when false.

Errors:

- `400` `pinned` is required
- `404` and the `400` cases of `POST /v1/plugins/{name}/check-update` (no git source)
- `500` the scan failed, or the flag could not be stored

Events: `plugins.changed` on success.

### POST /v1/plugins/{name}/tools/{tool}

Auth: any key
Effect: Runs one tool declared by that plugin, as a click in the plugin's own page. The call is checked against the tool schema and recorded as approved, with no approval card. The tool's own failure is `200` with `result.is_error` true. A failure before the tool runs is an HTTP error.

Path: `name` plugin name. `tool` the short tool name (`look`), not `plugin.tool` and not `plugin__tool`.

Request:

```json
{
  "session_id": "2026-10-14T12-00-00_01234567-89ab-cdef-0123-456789abcdef",
  "call_id": "look-1",
  "arguments": { "word": "gyre" }
}
```

At most 1 MiB. Unknown fields are rejected.

- `session_id` (string) is the conversation. A lane is `sessionId~laneId`: `laneId` is 1 to 48 characters, lowercase letters, digits, `_` or `-`, not `main`, and the id contains one `~`. Empty means the Bot's main conversation, which is created if needed. The conversation must already exist and its directory must be usable. The id itself is 1 to 160 ASCII letters, digits, `-` or `_`.
- `call_id` (string) is 1 to 64 characters: letters, digits, `_` or `-`.
- `arguments` (object) is required and must match the tool's schema.

At most 8 of these calls run at once across Core.

Response: `200`

```json
{
  "result": {
    "content": [{ "text": "hello" }],
    "is_error": false,
    "details": { "count": 1 }
  }
}
```

- `content` is an array of parts. A text part has `text`. An image part has `image_data` (base64) and `mime_type`. Empty parts are omitted.
- `is_error` is true when the tool, or a hook that rewrote the result, failed. The HTTP status is still `200`.
- `details` is present when the tool returned a JSON value.

Errors:

- `400` invalid request
- `400` arguments must be an object
- `400` `call_id` must be 1 to 64 letters, digits, `_` or `-`
- `400` arguments do not match the tool schema (the message names the tool)
- `404` conversation not found
- `404` plugin tool not found (unknown plugin, plugin not running, or the short name is not one of its tools)
- `409` the Bot is unavailable
- `409` the conversation's directory is unavailable, or the workspace or the hook table cannot be prepared (the message says which)
- `409` when approvals are not running, the conversation binding or the tool access check failed (the message is that error)
- `403` a hook rejected the call, or the call was not approved. An approval refusal starts with `approval refused`
- `429` too many plugin page calls at once; retry shortly
- `499` the client closed the request while approval was in progress
- `500` the Bot conversation could not be created

Events: `approvals.changed` when a write tool is recorded as approved and was not already allowed. A tool with `side_effect` `read` does not publish it. When approvals are not running, the tool still runs if its binding and access checks pass, and nothing is published; a failed check is `409`.

### DELETE /v1/plugins/{name}

Auth: admin
Effect: Deletes a plugin that was placed on this computer, drops its enable flag, install record and secrets, and stops its long-lived hooks. Non-secret setting values for that name are kept. A bundled plugin cannot be removed. Returns the plugin list after the deletion.

Path: `name` plugin name.

Request: no body.

Response: `200` the `GET /v1/plugins` body.

Errors:

- `404` plugin not found
- `400` a bundled plugin cannot be removed; turn it off instead
- `500` the directory, the secrets, or the following scan failed

Events: `plugins.changed` on success. A `500` from the scan after the directory and its secrets were both removed still publishes `plugins.changed` before the error. A failure to delete the directory, or to delete the secrets, publishes nothing.

### GET /v1/mcp

Auth: any key
Effect: Lists every MCP server and the tools it has discovered. Secrets are not included. Nothing is changed.

Request: no body.

Response: `200`

```json
{
  "servers": [
    {
      "id": "0123456789abcdef0123456789abcdef",
      "name": "Notes",
      "kind": "remote",
      "enabled": true,
      "url": "https://192.0.2.20/mcp",
      "timeout_seconds": 120,
      "oauth": true,
      "redirect_url": "http://127.0.0.1:15100/mcp/oauth/callback",
      "client_id": "gyres-client",
      "revision": "abcdef0123456789abcdef0123456789",
      "state": "authorization_required",
      "tools": [],
      "has_secrets": false,
      "auth_url": "https://192.0.2.20/authorize?state=once"
    }
  ]
}
```

- Servers are sorted by `id`.
- `id` is 1 to 64 characters: a lowercase letter or digit, then lowercase letters, digits or `-`. An id this route creates is 32 hex characters. A plugin server's id starts with `plugin-`, and `plugin` is the plugin name. Its `name` is `plugin/server`. Those servers appear while the plugin is running and are not editable here.
- `kind` is `local`, `remote` or `builtin`. `local` has `command`, `args` and `cwd` (an absolute directory). `remote` has `url` (`http` or `https`, no user info, no fragment). `builtin` has `builtin`. The built-in browser uses `builtin` `browser`.
- `enabled` is the stored switch. `timeout_seconds` is 1 to 3600. `include`, when present, is the only tool names that are offered; omitted or empty means all of them. `oauth` is only for `remote`. `redirect_url` and `client_id` belong to OAuth.
- `revision` changes on every successful `PUT`. Send it back on the next save.
- `state` is `disabled` when `enabled` is false. Otherwise `disconnected`, `queued`, `connecting`, `connected`, `authorization_required` or `error`.
- `error` is present when the last connection failed. Secret text in it is replaced with `[redacted]`. An OAuth failure is replaced with a short sentence: registration refused, authorization required, authorization expired, timed out, or a generic OAuth failure.
- `tools` is `{ "name", "description", "enabled" }` in the server's order. `enabled` is false when `include` leaves that tool out.
- `has_secrets` is true when environment values, headers, a client secret, or a plugin secret is stored. The values are not returned.
- `auth_url` is present while an authorization is waiting. Open it, then `POST /v1/mcp/{server}/callback` with the redirect URL if Core did not receive it.

Errors:

- `503` MCP unavailable. The body is `{"error":"MCP unavailable"}` with no `failure` object.

Events: none.

### PUT /v1/mcp

Auth: any key
Effect: Creates or replaces one MCP server, then starts it when `enabled` is true. The response is the full list, including a new `revision`. Plugin servers and the built-in browser have extra refusals below.

Request:

```json
{
  "config": {
    "id": "",
    "name": "Notes",
    "kind": "remote",
    "enabled": true,
    "url": "https://192.0.2.20/mcp",
    "timeout_seconds": 120,
    "oauth": false,
    "include": ["search"]
  },
  "secrets": {
    "headers": { "Authorization": "Bearer example" },
    "env": { "API_TOKEN": "example" },
    "client_secret": ""
  }
}
```

At most 1 MiB.

- `config` is the server. Empty `id` creates one and assigns a 32-hex id. Empty `timeout_seconds` becomes 120. `name` is required, at most 100 characters, and not blank.
- On update, `revision` must be the current one. A mismatch means another save won: reload and retry. A `revision` on an id that does not exist is refused.
- Omit `secrets` to keep the previous secrets and, when kind, URL, client id, redirect URL and the OAuth flag are unchanged, the saved OAuth login. Send `secrets` to replace them and drop that login. Header names and values must not contain CR or LF. OAuth cannot be combined with an `Authorization` header.
- The built-in browser must stay enabled, with no `include`, no OAuth and no secrets. It cannot be replaced by a different server, and a second browser cannot be added.
- `plugin` must be empty. An id starting with `plugin-`, or an existing plugin server, is refused.

Response: `200` the `GET /v1/mcp` body. Connection continues after the response; read again for `state`.

Errors: every body is `{"error":"<message>"}` with no `failure` object.

- `400` invalid MCP configuration
- `400` invalid MCP id or name
- `400` timeout must be 1–3600 seconds
- `400` local MCP requires a command and absolute working directory
- `400` invalid HTTP URL
- `400` missing builtin identifier
- `400` unknown MCP kind
- `400` OAuth requires remote MCP
- `400` OAuth requires a valid redirect URL
- `400` empty tool filter
- `400` MCP configuration changed; reload before saving
- `400` MCP no longer exists
- `400` a plugin's MCP server is managed on its plugin card
- `400` unknown builtin MCP
- `400` invalid MCP header
- `400` choose OAuth or an Authorization header
- `400` the built-in browser cannot be turned off, filtered, given secrets, or given OAuth; the message is `内置浏览器始终启用，全部工具可用，无需配置凭据或授权`
- `400` the built-in browser cannot be replaced; the message is `内置浏览器不可替换`
- `400` a browser server already exists; the message is `内置浏览器已存在`
- `503` MCP unavailable

Events: none.

### POST /v1/mcp/{server}/{action}

Auth: any key
Effect: `connect` queues a connection. `authorize` does the same and, when the server needs OAuth, produces an `auth_url` for a later read. `callback` delivers the redirect URL for a pending authorization. All three return the server list. The connection itself finishes in the background.

Path: `server` the server id. `action` is `connect`, `authorize` or `callback`.

Request: no body for `connect` and `authorize`. For `callback`:

```json
{ "url": "http://127.0.0.1:15100/mcp/oauth/callback?code=example&state=example" }
```

At most 16 KiB. The URL's scheme, host and path must match the server's `redirect_url`, and the `state` query must match the pending authorization. A `code` query is required. An `error` query declines the authorization.

Response: `200` the `GET /v1/mcp` body.

`connect` on a server that needs OAuth does not open `auth_url`. The later `error` is `MCP authorization required; use Settings → MCP`. Use `authorize` when a person should sign in.

Errors: the same `{"error":"<message>"}` body, with no `failure` object.

- `400` unknown MCP action
- `400` invalid callback
- `400` MCP unavailable or disabled
- `400` MCP connection already in progress (`state` is `queued`, `connecting` or `authorization_required`)
- `400` invalid callback URL
- `400` OAuth authorization was declined
- `400` missing authorization code
- `400` callback already received
- `400` no matching authorization request; start authorization again
- `503` MCP unavailable

Events: none.

### DELETE /v1/mcp/{server}

Auth: any key
Effect: Removes one server, its secrets and its saved OAuth login, and disconnects it. Returns the remaining list.

Path: `server` the server id.

Request: no body.

Response: `200` the `GET /v1/mcp` body.

Errors: the same `{"error":"<message>"}` body, with no `failure` object.

- `400` MCP not found
- `400` a plugin's MCP server is removed with its plugin
- `400` the built-in browser cannot be deleted; the message is `内置浏览器不可删除`
- `503` MCP unavailable

Events: none.

### GET /v1/computer-use

Auth: any key
Effect: Reports whether computer use is available on this computer, and the current switches. Nothing is changed. On a computer that cannot run it, the body is the unsupported object below and a `PUT` does not apply.

Request: no body.

Response: `200`

```json
{
  "supported": true,
  "enabled": true,
  "state": "ready",
  "version": "0.34.1",
  "ready": false,
  "mode": "ask",
  "blocked": [],
  "cursor_style": "signature_arc",
  "reduced_motion": "auto",
  "viewer": true
}
```

When this computer cannot run computer use (anything other than macOS on Apple silicon):

```json
{
  "supported": false,
  "enabled": false,
  "state": "unsupported",
  "version": "0.34.1",
  "ready": false,
  "mode": "",
  "blocked": null,
  "cursor_style": "",
  "reduced_motion": "",
  "viewer": false
}
```

- `version` is the driver version this Core installs.
- `enabled` defaults to true when nobody has turned it off. `viewer` is whether progress screenshots are kept, and defaults to true.
- `state` is `unsupported`, `missing` (no driver yet), `installing`, `ready` or `error`. During `installing`, `phase` is `downloading`, `extracting` or `verifying`, and `percent` (0 to 100) is set while a download length is known. `error` is why the last install failed. When a driver is already usable and a newer one is installing, `state` stays `ready`, `updating` is true, and `phase` and `percent` describe that install. `update_error` is why the last such attempt failed.
- `accessibility` and `screen_recording` are booleans once a check has completed, and omitted before that. `ready` is true only when `enabled` is true, a driver is usable, and both grants are true.
- `mode` is `ask` (an approval card per ordinary app; the default), `allow_except` (ordinary apps are allowed except those on `blocked`) or `allowlist` (only apps with a grant). `blocked` is `{ "bundle_id", "name"? }` and is `[]` when empty.
- `cursor_style` is `signature_arc` (the default), `spring_settle`, `magnetic`, `comet_swoop`, `adaptive` or `classic`. `reduced_motion` is `auto` (the default), `on` or `off`.

Errors: none.

Events: none.

### GET /v1/computer-use/apps

Auth: admin
Effect: Lists apps on the computer running Core, with whether each is on the allow list or the block list. Apps that can never be operated are omitted. Requires an installed driver.

Request: no body.

Response: `200`

```json
{
  "apps": [
    {
      "name": "Notes",
      "bundle_id": "com.example.notes",
      "running": true,
      "protected": false,
      "allowed": false,
      "blocked": false
    }
  ]
}
```

- Sorted by `name`, ignoring letter case. `running` is whether the app is running now.
- `protected` is true for apps that can be allowed only on the computer running Core (password managers, System Settings, terminals). `PUT /v1/computer-use` cannot change those.
- `allowed` is true when a global grant exists for that app. `blocked` is true when the app is on the block list. Comparison of bundle ids ignores letter case.

Errors:

- `409` the computer use driver is not installed
- `502` the driver could not list apps (the message is the cause)

Events: none.

### PUT /v1/computer-use

Auth: admin
Effect: Changes one or more switches. Every field is checked before anything is saved. The response is the status after the saves. A driver check started by `refresh` or by turning `enabled` on is not waited for; read `GET /v1/computer-use` again for the grants. On a computer that cannot run computer use, the body is ignored and the unsupported object is returned.

Request:

```json
{
  "enabled": true,
  "viewer": true,
  "mode": "allow_except",
  "cursor_style": "magnetic",
  "reduced_motion": "auto",
  "refresh": false,
  "app": { "bundle_id": "com.example.notes", "name": "Notes", "blocked": true }
}
```

At most 4096 bytes. At least one field must be set.

- `enabled`, `viewer` (booleans, optional).
- `mode` (string, optional): `ask`, `allow_except` or `allowlist`.
- `cursor_style` (string, optional): one of the styles on `GET`.
- `reduced_motion` (string, optional): `auto`, `on` or `off`.
- `refresh` (boolean): when true, starts a grant check and does not by itself change stored settings.
- `app` (object, optional) edits one ordinary app. `bundle_id` is required. Exactly one of `allowed` or `blocked` is set. `name` is optional and is stored on the block list and on a new grant's label. `allowed` true adds a global grant; false removes one if present. `blocked` true adds the app to the block list; false removes it. Gyres itself, the computer-use driver, and any app with `protected` true are refused.

Saves run in this order: `viewer`, `mode`, `cursor_style`, `reduced_motion`, `app`, `enabled`. A `500` on a later field leaves the earlier fields of this request already saved.

Response: `200` the `GET /v1/computer-use` body.

Errors:

- `400` invalid request
- `400` nothing to change
- `400` `cursor_style` must be one of `signature_arc`, `spring_settle`, `magnetic`, `comet_swoop`, `adaptive`, `classic`
- `400` `reduced_motion` must be auto, on or off
- `400` `mode` must be ask, allow_except or allowlist
- `400` app needs `bundle_id` and exactly one of `allowed` or `blocked`
- `403` this app can be changed only at the Core host
- `500` a save failed, or approvals are not running when `allowed` is set

Events: `approvals.changed` when `app.allowed` creates a grant or removes one. A grant that was already present does not publish again.

### GET /v1/host-screen

Auth: admin
Effect: Reports whether viewing is supported and turned on, the permission, the displays, and whether anyone is watching or controlling. Nothing is changed.

Request: no body.

Response: `200`

```json
{
  "supported": true,
  "enabled": true,
  "permission": "granted",
  "viewers": 1,
  "capturing": true,
  "width": 1280,
  "height": 720,
  "fps": 30,
  "displays": [
    {
      "id": 1,
      "name": "Built-in",
      "width": 1512,
      "height": 982,
      "pixel_width": 3024,
      "pixel_height": 1964,
      "main": true,
      "x": 0,
      "y": 0,
      "viewers": 1,
      "capturing": true,
      "stream_width": 1280,
      "stream_height": 720,
      "fps": 30
    }
  ],
  "control": {
    "supported": true,
    "enabled": false,
    "accessibility": "granted",
    "active": false
  }
}
```

- `supported` is true where capture exists (macOS 13 or later). `enabled` is the viewing switch, which is turned on from the computer running Core, not from this route.
- `permission` is `granted` or `denied` when capture is supported, and `unknown` when it is not.
- `viewers` is how many views are open. This route does not list their ids. `capturing`, `width`, `height` and `fps` describe the main display's stream when one is running, otherwise one running stream, otherwise zeros and false.
- `displays` is the main display first. `width` and `height` are points. `pixel_width` and `pixel_height` are backing pixels. `x` and `y` are the top-left corner in the global point space and may be negative. `viewers` is how many views watch that display. `stream_width`, `stream_height` and `fps` are present while that display is being captured.
- `control.supported` is whether this computer can inject input. `control.enabled` is the remote-control switch. `control.accessibility` is `granted`, `denied`, or `unknown` when control is unsupported. `control.active` is true while one viewer holds control.
- If viewing was not prepared, the same keys are returned with `supported` false, `permission` `""`, `displays` null, and `control.accessibility` `""`.

Errors: none.

Events: none.

### POST /v1/host-screen/viewers

Auth: admin
Effect: Accepts one complete WebRTC offer and returns the answer for a new view. Capture of the chosen display starts if this is the first viewer on it. The view stays open after the response, until it is deleted, the peer connection fails or closes, viewing is turned off, or the access key that opened it is revoked or regenerated.

Request:

```json
{
  "sdp": { "type": "offer", "sdp": "v=0\r\n..." },
  "width": 1280,
  "fps": 30,
  "display_id": 0
}
```

At most 256 KiB. There is no separate ICE exchange: put every candidate in the offer. No relay servers are added. Gathering the answer waits at most 15 seconds.

- `sdp.type` and `sdp.sdp` are the offer.
- `width` and `fps` are the size the viewer wants. `0` means the defaults, 1280 and 30. Width is then clamped to 320–4096 and fps to 5–60. Viewers of one display share one stream, at the largest requested width and the highest fps, then capped to an even width no wider than the display's pixel width or 4096.
- `display_id` `0` means the main display. Any other value must be a `displays[].id` from `GET /v1/host-screen`.

Response: `200`

```json
{
  "viewer_id": "01234567-89ab-cdef-0123-456789abcdef",
  "answer": { "type": "answer", "sdp": "v=0\r\n..." }
}
```

- `viewer_id` is the id for `PATCH` and `DELETE`.
- The video is one H.264 track. After the connection is up, the browser may open two data channels. `control` is reliable and ordered. `pointer` is unordered. Messages are JSON text, at most 64 KiB; anything else is ignored.
- `control` accepts `take`, `release`, `button`, `key`, `text`, `clipboard_set` and `clipboard_get`. `pointer` accepts `move` and `scroll` (and the same input messages, limited at the pointer rate).
- `take` asks to control. Optional `force` true takes over from another viewer. Replies on `control` are `{ "t": "control", "state": "granted" }`, `state` `denied` with `reason` `disabled`, `accessibility` or `unsupported`, or `state` `busy` with `reason` `busy`. `release` drops control. A later `{ "t": "control", "state": "released", "reason" }` uses `released`, `taken_over`, `idle`, `disabled`, `accessibility`, `unsupported`, `stopped` or `shutdown`. No reply is sent when the channel itself disconnects. Control ends after 60 seconds without input.
- `move` uses `x` and `y`, each 0 to 1 across the video content after letterbox bars are removed. `button` uses `button` 0 left, 1 middle, 2 right, `down`, and `clicks` from 1 to 3. `scroll` uses `x`, `y`, `dx` and `dy` in pixels; positive `dy` means the content should move down, and each delta is clamped to ±2000. `key` uses `code` (a physical KeyboardEvent code such as `KeyA` or `MetaLeft`) and `down`. Unrecognised codes are ignored. Command combined with Q, W or Tab, and Command+Option+Escape, are not forwarded. `text` is typed characters, control characters removed, at most 256 characters.
- `clipboard_set` sends `text`, at most 48 KiB, and receives `{ "t": "clipboard_set" }` or `{ "t": "clipboard_set", "error": "too_large" }` or `"failed"`. `clipboard_get` receives `{ "t": "clipboard", "text" }` or `{ "t": "clipboard", "error": "failed" }` or `"too_large"`.
- Input is applied only while this viewer holds control. `control` messages are limited to 250 per second after a burst of 120. `pointer` messages are limited to 500 per second after a burst of 120. Releasing a key or button this viewer is holding is not dropped. Only one viewer controls at a time.

Errors:

- `400` invalid request
- `400` invalid WebRTC offer, or display not found
- `403` screen recording permission is missing; grant it on the host
- `409` host screen viewing is turned off; enable it on the host
- `500` the answer was not ready within 15 seconds, or the connection could not be created
- `501` host screen viewing is not supported on this host

Events: none.

### PATCH /v1/host-screen/viewers/{id}

Auth: admin
Effect: Changes what an open view asks for. A non-zero `display_id` different from the current display moves the view. Capture of a display with no viewers left stops after 5 seconds.

Path: `id` the `viewer_id` from `POST /v1/host-screen/viewers`.

Request:

```json
{ "width": 1920, "fps": 30, "display_id": 0 }
```

At most 256 KiB. `sdp` is accepted and ignored. `width` and `fps` use the same defaults and clamps as `POST`: an omitted `0` becomes 1280 and 30, not "leave unchanged". `display_id` `0` means stay on the current display. A non-zero id must be a current display.

Response: `200`

```json
{ "ok": true }
```

Errors:

- `400` invalid request
- `400` display not found
- `404` viewer not found
- `500` the display could not be switched or reconfigured
- `501` host screen viewing is not supported on this host

Events: none.

### DELETE /v1/host-screen/viewers/{id}

Auth: admin
Effect: Closes one view. An id that is not open is still success. The peer connection is closed.

Path: `id` the `viewer_id`.

Request: no body. A body is ignored.

Response: `200`

```json
{ "ok": true }
```

Errors:

- `501` host screen viewing is not supported on this host

Events: none.
