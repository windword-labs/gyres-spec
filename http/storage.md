# Storage, export and import

These routes tell a frontend where this Core keeps its data, move that data to another directory, delete a previous location, reopen Core when the directory is missing or broken, delete old archived conversations, and download or restore one zip of the whole Core. Every route needs the administrator key.

JSON responses are `application/json` and send `Cache-Control: no-store`.

While Core is opening its data directory, or reopening after a move, import, retry or reset, every request is `503` with `code` `core_starting`, `error` `Core is starting; try again in a moment`, and `Retry-After: 5`. The reopen starts about 300 ms after the response that asked for it. Poll `GET /v1/storage` until `started_at` changes; that value is the UTC time this process opened.

While a move or an import is in progress, `GET /v1/storage` still returns `200`. Every other route in this file returns `503` with `code` `storage_maintenance`, `error` `Core is moving its data directory; try again when it finishes`, and `Retry-After: 5`.

While the data directory is unavailable, routes under `/v1/storage` still answer. Export, import and archive cleanup return `503` with `code` `storage_unavailable`. A Core that could not open its data directory at startup serves the storage routes only; the other routes here get that same `503`, and `error` names the reason, the path and the message.

A hard failure latches unavailable at once and emits `storage.unavailable`. Those reasons are `invalid`, `missing`, `not_directory`, `not_writable`, `read_only`, `full` and `identity_mismatch`. A transient failure (`timeout`, or `io_error` from the check itself) does not latch on the first miss. Other routes can wait up to 10 seconds for the check, then return `503` with `code` `storage_slow` and an `error` beginning `storage is slow to respond; waiting`. `GET /v1/storage` then has `mode` `slow`. The wait becomes a latch after 20 seconds without a success, counted from the first failing check, and `unavailable.reason` is then `timeout` or `io_error`. Routes under `/v1/storage` are not held by that wait.

Those `503` bodies, and any other error built the same way, look like this. `failure.message` and `failure.details` repeat `error`.

```json
{
  "error": "Core is moving its data directory; try again when it finishes",
  "code": "storage_maintenance",
  "failure": {
    "code": "core_request_failed",
    "message": "Core is moving its data directory; try again when it finishes",
    "http_status": 503,
    "details": "Core is moving its data directory; try again when it finishes"
  }
}
```

An error with no top-level `code` has `error` and `failure` only. `failure.http_status` is the HTTP status. Where a route describes its own body, that body is the whole response.

`storage.space` is `{"type":"storage.space","level":"low","volume":"data"}`. `level` is `ok`, `low` or `critical`. `volume` is `data` or `home`. `storage.unavailable` is `{"type":"storage.unavailable","reason":"missing"}`. `sessions.changed` carries `session_id`.

Seven directories move with the data location, in this order: `sessions`, `workspaces`, `schedules`, `tasks`, `bot`, `code`, `projects`. Everything else stays in the home directory.

Free space has two levels. `low` means free space is below the larger of 2 GiB and 5% of the volume. `critical` means free space is below 512 MiB. Leaving `low` takes 256 MiB above its line; leaving `critical` takes 128 MiB above 512 MiB. A `low` or `critical` level does not change `mode` and does not by itself reject a request.

`running` entries, where a route returns them, use `kind` and `id`. `title` is omitted when empty. The array is not ordered.

- `run`: a conversation run. `id` is the conversation id, shaped `2026-10-14T12-00-00_` plus a UUID.
- `memory_review`: a background memory review. `id` is `memory`.
- `complete`: a model request still in flight. `id` is `complete`.
- `task`: a running task receipt. `title` is the agent name when it has one.
- `subagent`: a running receipt that belongs to a parent conversation. `title` is the agent name when it has one.
- `schedule`: a schedule occurrence that has not finished. `title` is the schedule name.

### GET /v1/storage

Auth: admin
Effect: Reports where data is stored, how full the disks are, and whether a move, an import or a storage failure is in progress. In normal mode this read also checks free space on the data volume and the home volume.

Request: no body.

Response: `200`

Normal mode. `categories` is always the seven directories, in the order above. Sizes are omitted outside normal mode.

```json
{
  "mode": "normal",
  "slow": null,
  "home": "/srv/gyres",
  "data_dir": "/srv/gyres-data",
  "default": false,
  "restart_required": false,
  "started_at": "2026-10-14T12:00:00.123456789Z",
  "categories": [
    {"name": "sessions", "bytes": 1048576, "files": 12},
    {"name": "workspaces", "bytes": 0, "files": 0},
    {"name": "schedules", "bytes": 0, "files": 0},
    {"name": "tasks", "bytes": 0, "files": 0},
    {"name": "bot", "bytes": 0, "files": 0},
    {"name": "code", "bytes": 0, "files": 0},
    {"name": "projects", "bytes": 4096, "files": 1}
  ],
  "total_bytes": 1052672,
  "total_files": 13,
  "free_bytes": 20000000000,
  "space": {
    "data": {"free_bytes": 20000000000, "total_bytes": 500000000000, "level": "ok"},
    "home": {"free_bytes": 8000000000, "total_bytes": 250000000000, "level": "low"},
    "same_volume": false,
    "checked_at": "2026-10-14T12:00:01Z"
  },
  "migration": {
    "state": "idle",
    "bytes_done": 0,
    "bytes_total": 0,
    "files_done": 0,
    "files_total": 0
  },
  "unavailable": null,
  "cleanup": {"path": "/srv/gyres-old", "bytes": 4096}
}
```

A move in progress. Category sizes, `total_bytes`, `total_files` and `free_bytes` are absent. Poll about once a second.

```json
{
  "mode": "maintenance",
  "slow": null,
  "home": "/srv/gyres",
  "data_dir": "/srv/gyres",
  "default": true,
  "restart_required": false,
  "started_at": "2026-10-14T11:00:00.100000000Z",
  "migration": {
    "state": "running",
    "phase": "copying",
    "target": "/srv/gyres-data",
    "bytes_done": 4096,
    "bytes_total": 1052672,
    "files_done": 1,
    "files_total": 13
  },
  "unavailable": null,
  "cleanup": null
}
```

The data directory cannot be used. `default_data` is present only in this mode when `data_dir` is not the home path.

```json
{
  "mode": "unavailable",
  "slow": null,
  "home": "/srv/gyres",
  "data_dir": "/srv/gyres-data",
  "default": false,
  "restart_required": false,
  "started_at": "2026-10-14T12:00:00.123456789Z",
  "migration": {
    "state": "idle",
    "bytes_done": 0,
    "bytes_total": 0,
    "files_done": 0,
    "files_total": 0
  },
  "unavailable": {
    "reason": "missing",
    "path": "/srv/gyres-data",
    "message": "the directory does not exist (is the disk connected?)"
  },
  "default_data": {
    "path": "/srv/gyres",
    "has_data": true,
    "bytes": 1048576,
    "moved_at": "2026-10-13T18:00:00Z"
  },
  "cleanup": {"path": "/srv/gyres"}
}
```

- `mode` is `unavailable` when `unavailable` is set, otherwise `maintenance` while a move or import holds the Core, otherwise `slow` while a transient failure is being waited out, otherwise `normal`.
- `slow` is null when nothing is being waited out. Otherwise `reason` is `timeout` or `io_error`, `path` is the data directory, `message` is the check error, and `since` is the UTC time of the first failing check.
- `home` and `data_dir` are absolute paths. `default` is true when those two strings are equal.
- `started_at` is UTC. Fractional seconds appear when the clock has them. It changes on every reopen.
- `restart_required` is true when `migration.state` is `restart_required`. Maintenance stays on until the new process answers.
- `categories[].name`, `bytes` and `files` are logical file sizes. `total_bytes` and `total_files` sum them. `free_bytes` is free space on the data volume; it is omitted when that read fails.
- `space` is omitted until a check has succeeded. A failed check on this read keeps the previous `space` and still returns `200`. `checked_at` is UTC, whole seconds. `same_volume` is true when both paths are on one volume; `home` and `data` then carry the same numbers. On one shared volume a level change emits `storage.space` for `data` and for `home`.
- `migration.state` is `idle`, `running`, `failed` or `restart_required`. `phase` is omitted while idle. A data move uses `copying`, `verifying`, `rewriting`, `switching`, then `done`. An import sets `kind` to `import` and uses `unpacking`, `verifying`, `placing`, `rewriting`, `switching`, then `done`. `target` is the destination path, omitted for an import. `error` is set when `state` is `failed`. Byte and file counters stay `0` until the copy or unpack has measured work.
- `unavailable.reason` is one of the latch reasons above. `free_bytes` and `needed_bytes` appear when a retry still finds a full disk: `needed_bytes` is 536870912 (512 MiB). `message` is then `The disk still has only <free> free. Free up space until at least <needed> is free, then try again.` Sizes in that sentence use `GiB`, `MiB`, or a byte count.
- `unavailable.reason` can also be `stopping` after a retry that is waiting for runs to finish. `message` is `tasks are still stopping (N); try again shortly`.
- `default_data.path` is the home path. `has_data` is true when the home still has files in the seven directories, even when `bytes` is `0`. `moved_at` is present when the recorded previous location is the home path. It is UTC, whole seconds.
- `cleanup` is null when no previous location is waiting. Otherwise `path` is that location. `bytes` is included only in normal mode and is the size of the seven directories there. A directory that is a link is not added to `bytes`.

Errors:

- `500` the seven directories could not be measured. Normal mode only. The body is `error` and `failure`.

Events: `storage.space`

A level change found by this read emits `storage.space`. The first observation of `ok`, and a read that does not change a level, emit nothing.

### POST /v1/storage/check

Auth: admin
Effect: Checks whether the data can be moved to `path`. Nothing is copied and the target directory is not created. `ok` false is still `200`; read `items`.

Request:

```json
{ "path": "/srv/gyres-data" }
```

- `path` is required. Surrounding whitespace is removed. Any other field is ignored. The body is at most 4096 bytes.
- `allow_non_empty` is accepted and ignored here. `POST /v1/storage/migrate` is what acts on it.

Response: `200`

```json
{
  "ok": true,
  "path": "/srv/gyres-data",
  "default": false,
  "items": [
    {"id": "absolute", "ok": true, "message": "The path is absolute."},
    {"id": "usable", "ok": true, "message": "The directory exists and is empty."},
    {"id": "writable", "ok": true, "message": "The location is writable."},
    {"id": "separate", "ok": true, "message": "The location is separate from the current data directory."},
    {"id": "space", "ok": true, "message": "21474836480 bytes free, 537952256 needed."},
    {"id": "idle", "ok": true, "message": "Nothing is running."}
  ],
  "running": [],
  "total_bytes": 1052672,
  "allocated_bytes": 1081344,
  "reserve_bytes": 536870912,
  "required_bytes": 537952256,
  "free_bytes": 21474836480
}
```

- `ok` is true only when every item has `ok` true. A warning item still has `ok` true.
- `path` is the cleaned absolute path. A relative path is returned unchanged.
- `default` is true when `path` is the home directory, including when a link resolves to it. That is a move back to the default location. On success the saved data directory is cleared; it is not set to the home path.
- `items` stay in this order: `absolute`, `usable`, `writable`, `separate`, `space`, then `size` when measuring the current data failed, then `cleanup` when a previous location is still recorded, then `idle`.
- A relative path fails `absolute` with `The path must be absolute.` The next four items fail with `Not checked: the path must be absolute first.`
- `usable` passes when the directory exists and is empty, or when it can be created under an existing parent (`The directory will be created.`). A path that exists and is not a directory fails. A directory that cannot be read fails. Names `.DS_Store`, `.localized`, `Thumbs.db`, `desktop.ini` and names starting with `._` are ignored.
- A directory that already holds any other entry passes `usable` with `warning` true and sets `existing`. `existing.count` is the full count. `existing.names` is at most the first five, in filename order. `message` is `The directory is not empty (N items); Gyres will copy its folders next to them and leave them untouched.`
- `usable` fails when any entry name matches a moved directory or `.gyres-storage-id`, ignoring case. `message` is `The directory already holds <name> from a Gyres data directory; choose another folder.`
- For a move back, `usable` passes with `The default location holds no earlier data.` when each of the seven directories is missing or empty. Empty directories and the ignored names do not count. A file or a link in place of one of them counts. Otherwise `usable` fails and `leftover` is set.
- `leftover.categories` lists each home directory that still has files, with `name`, `bytes` and `files`. `leftover.bytes` and `leftover.files` are the sums. `recorded` is true when that leftover is the previous location of a move away from the home, and `moved_at` is the UTC time of that move when known. `message` then says to delete the old data first. When `recorded` is false, `message` names the folders and says they must be moved or deleted by hand.
- `writable` probes the directory, or the existing parent when the target does not exist yet. A permission error is quoted. On macOS a permission error also appends a note about the first-access prompt for a removable volume.
- `separate` fails when the target is the current data directory, is inside it, contains it, or overlaps a moved directory that is itself a link. For a move back it fails with `Gyres already uses the default location.` or when current data overlaps one of the seven directories in the home. A data directory that lives inside the home is allowed.
- `space` compares `free_bytes` with `required_bytes`. `free_bytes` is `0` when the path is not absolute or the free-space read fails. `allocated_bytes` rounds each current file up to 4 KiB (an empty file counts as 4 KiB). `reserve_bytes` is 536870912. `required_bytes` is the larger of the stored size plus 10% and the stored size plus `reserve_bytes`. When short, `short_bytes` is the gap and `message` is `Needs <required> bytes (the data as stored plus 10%, keeping at least <reserve> bytes free) but only <free> bytes are free: short by <short> bytes.` A passed check says `<free> bytes free, <required> needed.`
- `size` fails with `Cannot measure current data: ` plus the reason. `cleanup` fails with `Clean up the previous location (<path>) before moving the data again.` `idle` fails with `<N> task(s) are still running; wait for them or stop them first.` `running` lists that work.

Errors:

- `400` `path required`. Invalid JSON, a body over 4096 bytes, or a missing or blank `path`. The body is `error` and `failure`.

Events: none.

### POST /v1/storage/migrate

Auth: admin
Effect: Runs the same check as `POST /v1/storage/check`, then copies the seven directories to `path` and reopens Core on that location. The previous location is kept and recorded for `POST /v1/storage/cleanup`.

Request:

```json
{ "path": "/srv/gyres-data", "allow_non_empty": false }
```

- `path` is required, with the same rules as the check.
- `allow_non_empty` is optional and defaults to false. It is required when the check sets `existing`. It does not override a failed item. A move back that still has `leftover` stays failed; this flag does not delete that leftover.

Response: `202`

```json
{
  "state": "running",
  "phase": "copying",
  "target": "/srv/gyres-data",
  "bytes_done": 0,
  "bytes_total": 0,
  "files_done": 0,
  "files_total": 0
}
```

- This is the `migration` object. Totals fill in on a later `GET /v1/storage`, after the copy has measured the directories. `kind` is omitted.
- For a move back, `target` is the home path.
- Phases are `copying`, `verifying`, `rewriting`, `switching`. Success sets `state` to `restart_required` and `phase` to `done`, then Core reopens. Failure sets `state` to `failed` and `error` to the reason, leaves `phase` where it stopped, and ends maintenance.
- Poll `GET /v1/storage` until `started_at` changes. A failed move leaves the previous data in place.

Errors:

- `400` `path required`, same as the check.
- `503` `storage_unavailable`, `the current data directory is unavailable; retry it or switch back to the default first`. `Retry-After: 5` and the coded failure body.
- `409` `storage_maintenance`, `a migration is already running`, when this call loses the race to start maintenance. `Retry-After: 5` and the coded failure body. Once maintenance is on, a later call is the shared `503` instead.
- `422` the check did not pass, or it passed with `existing` and `allow_non_empty` is false. Maintenance is turned off again. The body has no `code` and no `failure`. `error` is `the target failed the pre-check`, or `the target is not empty; confirm with allow_non_empty` when `check.ok` is true and `existing` is set. `check` is the full `200` body of `POST /v1/storage/check`.

Events: `storage.unavailable`

Emitted only if the data directory fails during the copy.

### POST /v1/storage/cleanup

Auth: admin
Effect: Deletes the seven directories at the recorded previous location, then clears that record. The directory now in use is not deleted. A moved directory that is a link is removed as a link; the directory it points at is kept.

Request: no body.

Response: `200`

```json
{
  "path": "/srv/gyres-old",
  "removed": ["sessions", "workspaces", "projects"]
}
```

- `path` is the location that was recorded.
- `removed` lists the seven names that were present, in the order given above. Names that were already absent are left out. An empty result is `[]`.

Errors:

- `409` `storage_maintenance`, `the data directory is not ready`, when the data directory is unavailable or maintenance is on. `Retry-After: 5` and the coded failure body. While a move is already in progress the shared `503` usually answers first.
- `404` `no old location to clean up`. `error` and `failure` only.
- `409` `the current data directory is not healthy: ` plus the reason. The current location must still be the registered data directory.
- `409` `the old location is not the original data directory`.
- `409` `the old location overlaps the current data directory: ` plus the directory path. Nothing further is deleted after this refusal. Directories earlier in the list may already have been removed when a later delete returns `500`.
- `500` the delete or the record update failed. `error` is `delete <path>: <reason>` or the underlying error. The record remains until a later call succeeds. Call the route again.

Events: `storage.unavailable`

Emitted only if the health check latches the data directory.

### POST /v1/storage/retry

Auth: admin
Effect: Re-checks an unavailable data directory and reopens Core when it is usable again. When storage is already available this returns the same `200` document as `GET /v1/storage` and does not reopen.

Request: no body.

Response: `202`

```json
{ "recovered": true }
```

Core reopens. Until `started_at` changes, requests are `503` `core_starting`.

A retry that does not reopen returns `200` and the `GET /v1/storage` document, with `mode` still `unavailable`. The `unavailable` object is the result of this attempt.

- `reason` `stopping` and `message` `tasks are still stopping (N); try again shortly` while a run slot is still held. Call again when it is clear. A full disk is remembered: the next attempt still requires 512 MiB free.
- `reason` `full` when the disk is still below 512 MiB. `free_bytes` and `needed_bytes` are set, and `message` is the free-space sentence from `GET /v1/storage`. If free space cannot be read, `message` is `Cannot read the free space: ` plus the reason, without those two numbers.
- Any other check failure uses that failure's `reason` and `message`. `path` is the configured data directory.
- Success requires an existing writable directory. When Core has a saved identity for that directory, the identity file there must match. A directory Core has never opened successfully is accepted by this route once it is an existing writable directory. A directory with a different identity is refused with `identity_mismatch`.

Core also runs this check every 5 seconds while latched, and reopens by itself when the directory is back with its saved identity and, for a full disk, 512 MiB free. That timer does not adopt a directory that has never been opened successfully. A different identity is not adopted.

Errors: none.

Events: none.

### POST /v1/storage/reset

Auth: admin
Effect: Clears a custom data location and reopens Core on the home directory. Files at the custom location are left where they are. This is only for the unavailable state.

Request: no body.

Response: `202`

```json
{ "reset": true, "data_dir": "/srv/gyres" }
```

- `data_dir` in this body is the home path the new process will use. The saved custom location is cleared, so the next `GET /v1/storage` reports `default` true and `data_dir` equal to `home`.
- A cleanup record that names the home path is removed, because that path is now the live data. A record that names some other path is kept.
- Whatever the home directory already holds is what comes back. The two locations are not merged.

Errors:

- `409` `the data directory is available; use migration to move it`.
- `409` `already using the default data directory`.
- `409` `tasks are still stopping; try again shortly`, while a run slot is still held.
- `500` the setting could not be saved.

Each of these is `error` and `failure` only.

Events: none.

### GET /v1/archive-cleanup

Auth: admin
Effect: Reports the automatic deletion of archived conversations and the last run, if one has been recorded.

Request: no body.

Response: `200`

```json
{
  "enabled": false,
  "days": 90,
  "last_run": {
    "at": "2026-10-14T03:00:00Z",
    "trigger": "auto",
    "days": 90,
    "deleted": 2,
    "skipped": 1,
    "freed_bytes": 1048576
  }
}
```

- `enabled` is false when the setting has never been saved. `days` is `30`, `90` or `180`. A missing setting, a missing file, or a stored value outside those three is reported as `90`.
- `last_run` is omitted when no run has been recorded, the record cannot be read, or `at` is empty.
- `at` is UTC, whole seconds. `trigger` is `auto` or `manual`. `days` is the value used for that run. `deleted`, `skipped` and `freed_bytes` are numbers. `error` is omitted when the run listed the archive successfully.
- While `enabled` is true, Core checks five minutes after it opens and then every hour. It runs when the last run, manual or automatic, is at least 24 hours before `at`. It does not start during a move or while storage is unavailable.

Errors: none.

Events: none.

### PUT /v1/archive-cleanup

Auth: admin
Effect: Saves the automatic cleanup switch and the age threshold. The response is the same document as `GET /v1/archive-cleanup`.

Request:

```json
{ "enabled": true, "days": 30 }
```

- `days` is required and must be `30`, `90` or `180`.
- `enabled` defaults to false when omitted, which turns the switch off.
- Unknown fields are rejected. The body is at most 128 KiB.

Response: `200`

The `GET /v1/archive-cleanup` document, after the save.

Errors:

- `400` `invalid settings request`. Invalid JSON, unknown fields, or a body over 128 KiB.
- `400` `days must be 30, 90 or 180`.
- `500` the setting could not be saved.

Each of these is `error` and `failure` only.

Events: none.

### POST /v1/archive-cleanup/run

Auth: admin
Effect: Deletes archived conversations older than the saved `days`, including when the switch is off. The response is the `GET /v1/archive-cleanup` document, with this run as `last_run` and `trigger` `manual`.

Request: no body.

Response: `200`

The `GET /v1/archive-cleanup` document.

- Age uses the transcript's modification time. A conversation is deleted when that time is before now minus `days` times 24 hours.
- Bot chats are left in place and are not counted.
- A conversation that is running or pinned counts in `skipped` and is kept.
- A conversation that is restored, or whose transcript time changes after the run listed it, is kept and counts in neither `deleted` nor `skipped`.
- `freed_bytes` sums, before each delete, that conversation's files in the sessions directory, its presented files, its widgets and its agent binding files. The scratch work directory is not counted and is not removed.
- One run at a time. A second call waits for the first and then returns `200`.
- If a move starts or storage becomes unavailable during the run, the run stops after the conversation it is on and records the counts so far. `error` stays empty in that case.
- A failure to list the archive is still `200`. `last_run.error` is the reason, and the counts stay `0`.

Errors: none.

Events: `sessions.changed`

One event per conversation removed, with that conversation's `session_id`. A run that removes nothing emits none.

The download and the upload share one lock. A second export or import gets `409` `transfer_busy`.

The package is a zip. `manifest.json` is the last entry. Its `format` is `gyres-export` and `format_version` is `1`. A client that only saves or uploads the body can treat the zip as opaque. The fields below are what import checks and what `GET /v1/import` repeats from the last package.

```json
{
  "format": "gyres-export",
  "format_version": 1,
  "gyres_version": "1.2.3",
  "created_at": "2026-10-14T12:00:00Z",
  "source": {
    "os": "linux",
    "home": "/srv/gyres",
    "data": "/srv/gyres-data",
    "parts": {"sessions": "/srv/gyres-data/sessions"}
  },
  "categories": [
    {"id": "conversations", "files": 12, "bytes": 1048576},
    {"id": "settings", "files": 1, "bytes": 512}
  ],
  "secrets_to_reenter": [
    {"kind": "provider", "name": "Northwind"},
    {"kind": "plugin", "name": "example", "keys": ["token"]},
    {"kind": "mcp", "name": "Example tools"},
    {"kind": "channel", "name": "feishu"}
  ],
  "excluded": [
    "access keys",
    "TLS certificates and keys",
    "provider API keys",
    "plugin secrets",
    "MCP credentials and OAuth tokens",
    "Bot channel connections",
    "logs",
    "caches (browser, runtimes)",
    "updates",
    "storage records",
    "external agents' own histories"
  ],
  "entries": [
    {"path": "conversations/sessions/2026-10-14T12-00-00_018f3a2c-7b14-4c3e-9a10-6d2e4f8a1b00.jsonl", "size": 128, "sha256": "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa", "mode": 384}
  ]
}
```

- `source.parts` maps every part id (`sessions`, `workspaces`, `visualizations`, `projects`, `memories`, `bot`, `skills`, `schedules`, `tasks`, `code`, `plugins`, `approvals`) to its directory. The example shows one. `source.resolved`, when present, maps a part id to that same directory with symlinks resolved, and only for parts where the two paths differ.
- `categories[].id` is `conversations`, `projects`, `memory`, `bot`, `skills`, `schedules`, `tasks`, `code`, `plugins`, `approvals`, then `settings`. `files` and `bytes` count packed files. Empty parts are still listed. `excluded` is always the list in the example.
- `mode` on a file entry is the permission bits as a number. `384` is owner read and write.
- `secrets_to_reenter[].kind` is `provider`, `plugin`, `mcp` or `channel`. `keys` is present for a plugin and lists the secret names that had a value, not the values. `provider` uses the provider name, or its id when the name is empty. `channel` uses the kind, `feishu` or `discord`.
- API keys, the data location, the listen address, LAN discovery, TLS and trusted proxies are removed from the packed settings. Access keys, certificates, plugin secrets, MCP credentials, Bot channel connections, logs and caches are not packed. `excluded` names those groups.
- A file entry omits `type` and includes `size` and a 64-character `sha256`. `mode` is the permission bits. `mtime` is omitted when zero. A directory has `type` `dir`. A link has `type` `link` and `target`, and is not a zip entry of its own.
- Paths are relative and use `/`. They sit under `conversations/sessions`, `conversations/workspaces`, `conversations/visualizations`, `projects/projects`, `memory/memories`, `bot/bot`, `skills/skills`, `schedules/schedules`, `tasks/tasks`, `code/code`, `plugins/plugins`, `approvals/approvals`, or `settings/` with one of `config.json`, `mcp-servers.json`, `codex.json`, `grok.json`, `pi.json`, `claude.json`, `hooks.json`, `running-input.json`. `bot/bot/channels` and `plugins/plugins/.tmp` are not valid entries.

### GET /v1/export

Auth: admin
Effect: Reports the current or last export. The counters stay after the export finishes; `idle` means no export has run in this process.

Request: no body.

Response: `200`

```json
{
  "state": "running",
  "files_done": 4,
  "files_total": 13,
  "bytes_done": 4096,
  "bytes_total": 1052672,
  "at": "2026-10-14T12:00:00Z"
}
```

- `state` is `idle`, `running`, `done` or `failed`.
- `files_total` and `bytes_total` are measured from the part directories when the export starts. They include files the zip then leaves out, so `files_done` can finish below `files_total` when `state` is `done`. Settings files are in the zip and are not part of these counters.
- `at` is UTC and is omitted while `idle`. It is the start time while `running`, and the finish time when `done` or `failed`.
- `error` is present only when `state` is `failed`.

```json
{
  "state": "idle",
  "files_done": 0,
  "files_total": 0,
  "bytes_done": 0,
  "bytes_total": 0
}
```

Errors: none.

Events: none.

### POST /v1/export

Auth: admin
Effect: Streams a zip of this Core's data. Export does not enter maintenance. Nothing may be running, and no other export or import may be in progress.

Request: no body.

Response: `200`

`Content-Type: application/zip`. `Content-Disposition: attachment; filename="gyres-export-20261014-120000.zip"`. The timestamp is the Core host's local time, `YYYYMMDD-HHMMSS`. `Cache-Control: no-store`. The body is the package described above.

The status is `200` as soon as the body starts. A read error after that aborts the connection. `GET /v1/export` then has `state` `failed` and `error` set. Poll that route for `files_done` while the download runs.

Errors:

- `409` work is still running. The body has no `failure` object:

```json
{
  "error": "1 task(s) are still running; wait for them or stop them first",
  "code": "transfer_busy",
  "running": [{"kind": "run", "id": "2026-10-14T12-00-00_018f3a2c-7b14-4c3e-9a10-6d2e4f8a1b00"}]
}
```

- `409` `an export or import is already running`, with `code` `transfer_busy` and no `running` field.

Events: none.

### GET /v1/import

Auth: admin
Effect: Reports whether an import is in progress, whether this Core is fresh enough to accept one, and the outcome of the last import.

Request: no body.

Response: `200`

```json
{
  "running": false,
  "not_fresh": [],
  "last": {
    "state": "done",
    "at": "2026-10-14T12:05:00Z",
    "gyres_version": "1.2.3",
    "created_at": "2026-10-14T12:00:00Z",
    "categories": [
      {"id": "conversations", "files": 12, "bytes": 1048576}
    ],
    "secrets_to_reenter": [
      {"kind": "provider", "name": "Northwind"}
    ]
  }
}
```

- `running` is true only while an import holds the transfer lock, including during the upload before `POST /v1/import` answers. An export in progress leaves it false. After the import is accepted, maintenance hides this route until the import fails or Core reopens. Follow progress on `GET /v1/storage`.
- `not_fresh` is always an array. An empty array means a package can be imported. Otherwise the ids are the categories that already hold data, in this order, skipping empty ones: `conversations`, `projects`, `memory`, `bot`, `skills`, `schedules`, `code`, `plugins`.
- `conversations`: a non-empty transcript (`*.jsonl`) under sessions, or any other user file under workspaces or visualizations.
- `projects`: `projects.json` is not a JSON array, or that array is not empty, or projects has any other user file.
- `memory`: a Markdown file that is not empty or whitespace.
- `bot`: a non-blank `MEMORY.md`, or any file under `incidents/`.
- `skills`, `schedules` and `code`: any user file. `plugins`: any user file outside `.tmp`.
- Ignored names are the same as on the storage check, plus `.lock`. A link, any other non-directory, and a directory that cannot be read count as data. What a new Core writes for an empty Bot chat, default settings and blank memory files does not.
- `last` is omitted when no import has been recorded or the record cannot be read.
- `last.state` is `done` or `failed`. `at` is when this Core finished the attempt. `created_at` and `gyres_version` come from the package. `created_at` is omitted when the package had none. `error` is omitted unless `state` is `failed`.
- `categories` and `secrets_to_reenter` are copied from the package. `skipped_links` lists package paths of links that were not created because the target would fall outside this Core's home or data directory. It is omitted when there are none.

Errors: none.

Events: none.

### POST /v1/import

Auth: admin
Effect: Reads a package from the request body, checks it, and replaces this Core's data with the package. The Core must be fresh and idle. On success Core reopens. Poll `GET /v1/storage` for `migration.kind` `import`. After the reopen, `GET /v1/import` carries `last`.

Request: the zip bytes. The content type is not checked. This is not JSON and not a multipart form.

- The upload must fit in the free space of the data volume above a 512 MiB reserve. An empty body is refused.

Response: `202`

```json
{
  "kind": "import",
  "state": "running",
  "phase": "unpacking",
  "bytes_done": 0,
  "bytes_total": 1048576,
  "files_done": 0,
  "files_total": 12
}
```

- `files_total` and `bytes_total` count file entries in the manifest. Directories and links are not included. `target` is omitted.
- Phases are `unpacking`, `verifying`, `placing`, `rewriting`, `switching`. Success sets `state` to `restart_required` and `phase` to `done`, then Core reopens. Failure sets `state` to `failed` and `error`, ends maintenance, and records `last.state` `failed` with the same error.
- The package replaces every part it knows. A part the package does not carry becomes empty.
- This machine keeps its data location, listen address, LAN discovery, TLS, trusted proxies, and any API key already stored here. Providers that exist only here are kept. MCP connections come back without credentials.
- Paths that pointed at the source Core are rewritten onto this Core. A link whose target would fall outside the home or the data directory is skipped and named in `last.skipped_links`.
- If the data directory sits inside a `.gyres-import` directory under the home, the import fails after the `202` with `the data directory <path> is inside the import staging folder; move it first`.

Errors:

The `409` and `422` bodies below have `error` and `code`, plus the extra fields named. They have no `failure` object. A refusal before the `202` leaves this Core unchanged.

- `503` `storage_unavailable`, `the data directory is not ready`, when this call finds storage unavailable or maintenance already on. `Retry-After: 5` and the coded failure body.
- `409` `transfer_busy`, `an export or import is already running`.
- `409` `transfer_busy`, with `running`, when work is in progress. `error` is `<N> task(s) are still running; wait for them or stop them first`.
- `409` `import_not_fresh`, with `categories` set to the `not_fresh` ids. `error` is `Import needs a fresh Gyres: this one already holds <names>. Merging into existing data is not supported yet.`
- `409` `transfer_busy`, `a data move is running`, when maintenance cannot be entered. If the second idle and fresh check fails after maintenance starts, maintenance ends and the `409` above is returned instead.
- `422` `import_upload`. `error` is `Cannot receive the package: ` plus the reason. An empty body uses `the request body is empty`. Too little free space uses `the data disk has only <size> free` or `not enough free space on the data disk (<size> free)`.
- `422` `import_invalid`. `error` is one of:
  - `this file is not a Gyres export package (not a readable zip)`
  - `this file is not a Gyres export package (no manifest.json)`
  - `this file is not a Gyres export package` when `format` is not `gyres-export`
  - `cannot read manifest.json: <reason>` or `manifest.json is damaged: <reason>`
  - `this package was made by a newer Gyres (format N); update Gyres, then import it`
  - `this package has an unknown format version (N)` when the version is below 1
  - `the package lists <name> twice`
  - `the package has an invalid or repeated entry: "<path>"`
  - `the package is incomplete: <path> is missing`
  - `the package is damaged: <path> does not match the manifest` when size or `sha256` does not match
  - `the package has an invalid entry: "<path>"`
  - `the package has an unknown entry type "<type>"`
  - `the package holds <name>, which the manifest does not list`
- `422` `import_space`. `error` is `Cannot read free space: <reason>`, or `Not enough free space at <directory>: the import needs <need> and keeps <reserve> free, but only <free> is free.` The data volume and the home volume are checked separately. On one volume the two needs are added. Each side must keep 512 MiB free. Need is the packed files rounded up to 4 KiB.

Events: `storage.unavailable`

Emitted only if the data directory fails during the import.
