# System

Status, identity, disk and process counters, model-usage records, the task-round cap, HTTPS, desktop and standalone updates, and the managed browser and code runtimes. These routes do not change a conversation.

JSON responses are `application/json`. A failed request uses the usual error body (`error` plus `failure`) except where an endpoint shows another body. Do not match on `error` text; branch on the HTTP status and, where an endpoint gives one, on `code`.

While Core is opening, `GET /health` answers `503` at once. Every other route in this file waits up to 10 seconds for Core to finish opening, then answers the same `503`: `code` is `core_starting`, `Retry-After` is `5`. That starting answer does not require an access key. While the data directory is unavailable or being moved, `GET /health`, `GET /v1/identity`, `POST /v1/identity/challenge`, `GET /v1/version`, `GET /v1/tls` and `POST /v1/tls/issue` keep their normal answers. Every other route here then answers the storage `503` (`storage_unavailable`, `storage_maintenance` or `storage_slow`).

Timestamps are UTC, RFC3339, and may include a fraction of a second.

### GET /health

Auth: none
Effect: Reports that this Core process is accepting requests. A `200` means the process is up. It does not mean the data directory is usable, and it does not mean HTTPS is on.

Request: no body.

Response: `200`

```json
{ "status": "ok" }
```

- `status` is the string `ok`.
- `X-Gyres-Core-Instance` is always set. It is an opaque token for this process, or an empty string when Core was not given one. A frontend can ignore it.
- `X-Gyres-Core-Mode` is set only when this Core has a build channel. The value is `production`, `develop` or `local`.
- The `503` returned while Core is still opening does not send these two headers and does not use this body.

Errors:

- `503` Core is still opening (`code` `core_starting`, `Retry-After: 5`). Retry. This route does not wait.

Events: none.

### GET /v1/browser-runtime

Auth: any key
Effect: Reports the managed browser on this Core host: whether it is installed, and how far a running install has got. Nothing is downloaded by this call.

Request: no body.

Response: `200`

```json
{
  "phase": "downloading",
  "percent": 30,
  "progress": "12/40 MB (30%)",
  "state": "installing",
  "supported": true,
  "binary": "/var/lib/gyres/browser/bin/agent-browser",
  "chrome": "/var/lib/gyres/browser/chrome/chrome",
  "version": "0.38.1",
  "log": "/var/lib/gyres/browser/install.log"
}
```

- `state` is `missing` (not installed), `installing`, `ready` or `error`. When the browser program and Chrome are both present and no install is running, the state is `ready` and `error` is cleared.
- `phase` is omitted when there is nothing to report. During an install it is one of `checking`, `binary`, `downloading`, `extracting`, `verifying` and `ready`. It is not saved across a restart: after a restart a ready browser omits `phase`.
- `percent` is 0–100 while a download reports a percentage, and is omitted when the size is unknown or a download is being retried.
- `progress` is a short progress string, such as `12/40 MB (30%)`, or `正在重新下载` while a download is retried. Omitted when empty.
- `error` is omitted when empty. On failure it is the install error. The text can name a path on the Core host.
- `supported` is true on macOS Apple silicon, Linux amd64, Linux arm64, Windows amd64 and Windows arm64.
- `binary` is the path of the browser program on the Core host. It is present even when the file is not there yet.
- `chrome` is the path of the Chrome executable. Omitted when none is published.
- `version` is the browser release this Core installs (`0.38.1`).
- `log` is the path of the install log on the Core host. It is present before the first install.
- The response is not `Cache-Control: no-store`. Poll this route while `state` is `installing`. An install gives up after 15 minutes and then `state` is `error`.

Errors:

- `503` the managed browser is not running in this Core. The body is only `{"error":"Browser unavailable"}`, with no `failure`.

Events: none.

### GET /v1/code-runtime

Auth: any key
Effect: Reports the managed code runtime (a pinned runtime used to run code) on this Core host. Nothing is downloaded by this call.

Request: no body.

Response: `200`

```json
{
  "state": "installing",
  "version": "1.4.2",
  "binary": "/var/lib/gyres/runtimes/bun/1.4.2/bun",
  "downloaded": 1048576,
  "total": 20971520
}
```

- `state` is `missing`, `installing`, `ready` or `error`. Core starts an install when it opens, on a host that supports it.
- `version` is the runtime release this Core installs (`1.4.2`).
- `binary` is the path of that runtime on the Core host.
- `error` is omitted when empty. After a failed install it holds the reason.
- `downloaded` is how many bytes of the archive have been written. `total` is the archive size in bytes, or `-1` when the download did not say a length. Both stay at `0` until a download starts, and are not cleared when `state` becomes `ready`. Use `state`, not these two numbers, to decide readiness.
- Only macOS on Apple silicon can install this runtime. Other hosts come up with `state` `error`.

Errors:

- `503` the code runtime is not running in this Core. The body is only `{"error":"code runtime unavailable"}`, with no `failure`.

Events: none.

### GET /v1/identity

Auth: any key
Effect: Reports which kind of access key this request used, and where this Core can be reached. The response sends `Cache-Control: no-store`.

Request: no body.

Response: `200`

```json
{
  "role": "admin",
  "host": "studio",
  "instance_id": "0123456789abcdef0123456789abcdef",
  "addresses": [
    { "url": "https://core.example.com", "source": "https" },
    { "url": "http://192.0.2.10:15100", "source": "lan" },
    { "url": "http://127.0.0.1:15100", "source": "loopback" }
  ]
}
```

- `role` is `admin` when the key is an administrator key, and `user` for every other accepted key.
- `host` is the operating-system hostname of the Core machine. It is an empty string when that name cannot be read.
- `instance_id` is 32 lowercase hex characters, stable for this Core's home across restarts. Clients that know several addresses use it to recognise one machine. It is an empty string when it cannot be stored.
- `addresses` is never null. Each `url` is an origin a client can use, and `source` says why it is listed. Order, with duplicates removed:
  1. `https` — the HTTPS address, only while HTTPS state is `active` or `renewal_failing`. The port is omitted when it is 443.
  2. `listen` — `http://` plus the HTTP bind address and port, only when HTTP is bound to one non-loopback IPv4 address.
  3. `lan` — one `http://<ipv4>:<port>` for each up, non-loopback IPv4 address, only when HTTP listens on every interface. Private addresses come first.
  4. `loopback` — `http://127.0.0.1:<http port>`, always.
- Only IPv4 is listed, apart from the HTTPS URL, which uses the configured domain. If the machine's addresses cannot be listed, the `lan` entries are skipped and `loopback` remains.

Errors: none.

Events: none.

### GET /v1/resources

Auth: any key
Effect: Reports this process and how much disk Core's files use. The disk figures are measured at most once every 30 seconds; the process figures are read on every call.

Request: no body.

Response: `200`

```json
{
  "sampled_at": "2026-10-14T12:00:00Z",
  "host": "studio",
  "os": "linux",
  "arch": "amd64",
  "home": "/var/lib/gyres",
  "data_dir": "/var/lib/gyres",
  "pid": 1234,
  "logical_cpus": 4,
  "go_version": "go1.25.0",
  "heap_bytes": 1000000,
  "go_reserved_bytes": 8000000,
  "goroutines": 40,
  "active_runs": 1,
  "uptime_seconds": 3600,
  "storage": {
    "measured_at": "2026-10-14T12:00:00Z",
    "bytes": 10,
    "files": 1,
    "partial": false,
    "skipped": 0,
    "categories": [
      {
        "id": "conversations",
        "bytes": 10,
        "files": 1,
        "home_bytes": 10,
        "data_bytes": 0,
        "home_files": 1,
        "data_files": 0,
        "moves": true
      }
    ]
  }
}
```

- `sampled_at` is when this response was built. `host` is the operating-system hostname, or empty when it cannot be read.
- `os` and `arch` name the Core host, such as `linux` and `amd64`, or `darwin` and `arm64`.
- `home` and `data_dir` are absolute paths on the Core host. They are equal when data has not been moved. `pid` is this process. `logical_cpus` is the number of logical CPUs. `go_version` is the Go runtime version. `heap_bytes` is the heap in use. `go_reserved_bytes` is memory reserved from the operating system. `goroutines` is the number of goroutines. `active_runs` is how many conversation runs are in this process now. `uptime_seconds` is whole seconds since this process started.
- `storage.measured_at` is when the disk scan ran. A call within 30 seconds of that time returns the same `storage` object, including a `partial` one. Wait until `measured_at` is 30 seconds old before expecting a new scan.
- `bytes` and `files` are logical sizes of regular files, counted once. Symlinks are not followed and are not counted. `partial` is true when the scan stopped early (2 seconds, 100,000 entries, or an entry that could not be read). `skipped` counts entries that were not measured: unreadable entries, symlinks and files that are not regular.
- `categories` contains only ids that have at least one counted file, largest `bytes` first, then `id` ascending. `home_bytes` / `home_files` are the part under `home`. `data_bytes` / `data_files` are the part under `data_dir` when that path is different; otherwise they are `0` and the whole total is in the home fields. A data directory inside the home is not counted twice.
- `id` is one of `conversations`, `browser`, `code`, `bot`, `runtimes`, `workspaces`, `memory`, `skills`, `plugins`, `visualizations`, `mcp`, `approvals`, `access`, `logs`, `cache`, `backups`, `tasks`, `other`.
- `moves` is true for `conversations`, `workspaces`, `tasks`, `bot` and `code`: those follow the data location. Every other id stays with the home.

Errors:

- `500` Core home unavailable

Events: none.

### GET /v1/task-limits

Auth: any key
Effect: Reads the global cap on how many model rounds a new task may run. The cap is off until someone saves one. A task already running keeps the cap it started with.

Request: no body.

Response: `200`

```json
{ "enabled": false, "max_turns": 100, "goal_max_continuations": 100 }
```

- `goal_max_continuations` caps a goal's automatic continuations ([goal](runs.md#goal)); `100` until someone saves another.
- `enabled` is false until a cap is turned on. While it is false, new tasks are not capped, and `max_turns` is still stored.
- `max_turns` is the stored cap. When no setting has been saved it is `100`. A new task is limited to this many model rounds only when `enabled` is true. One round is one model decision and its tool calls. A tighter limit asked for on a single task is kept; this setting never raises a task's own limit.

Errors:

- `500` the saved setting could not be read

Events: none.

### GET /v1/tls

Auth: admin
Effect: Reports HTTPS: whether it is on, the certificate that is being served, and any order waiting on the owner. Secrets are not included. The response sends `Cache-Control: no-store`.

Request: no body.

Response: `200`

```json
{
  "enabled": true,
  "state": "active",
  "https_url": "https://core.example.com",
  "host": "",
  "port": 443,
  "domain": "core.example.com",
  "source": "acme",
  "cert_file": "",
  "key_file": "",
  "acme": {
    "ca": "letsencrypt",
    "directory_url": "",
    "staging": false,
    "email": "owner@example.com",
    "challenge": "dns-01",
    "http01_port": 80,
    "dns_provider": "cloudflare"
  },
  "credentials": {
    "cloudflare.api_token": { "set": true },
    "eab.kid": { "set": false },
    "eab.hmac": { "set": false }
  },
  "certificate": {
    "issuer": "Example CA",
    "subject": "core.example.com",
    "names": ["core.example.com"],
    "not_before": "2026-10-01T00:00:00Z",
    "not_after": "2026-12-30T00:00:00Z",
    "serial": "0a1b2c3d",
    "staging": false
  },
  "last_attempt_at": "2026-10-01T00:00:00Z",
  "last_renewal_at": "2026-10-01T00:00:00Z",
  "next_renewal_at": "2026-12-01T00:00:00Z",
  "last_error": null,
  "listen_error": null,
  "pending": null,
  "propagation": "checked",
  "handshake_errors": 0
}
```

- `enabled` is whether HTTPS is turned on. `state` is decided in this order: `disabled` when it is off; `error` when the saved settings cannot be read or do not validate; then, for an ACME setup with no certificate yet, `waiting_owner` while a manual TXT record is waiting to be added, `issuing` while an order is running or has been asked for, `pending_issue` when no order has been asked for or the last problem was only a warning, otherwise `error`. With a certificate loaded: `listen_failed` when the HTTPS port is not open; `waiting_owner` again while a manual TXT record waits; `expired` when the certificate is outside its dates; `renewal_failing` for an ACME certificate whose latest problem is not a warning; otherwise `active`.
- `https_url` is `https://` plus `domain`, with `:<port>` only when `port` is not 443, while a certificate is loaded and the listener is up. Otherwise it is null. Identity lists this URL only in `active` and `renewal_failing`.
- `host` is the HTTPS bind address. An empty string means every interface. `port` is the HTTPS port; a saved `0` is reported as `443`.
- `domain` is the name on the certificate, lowercase, without a trailing dot.
- `source` is `files` (the owner's certificate files) or `acme`. `cert_file` and `key_file` are absolute paths when `source` is `files`, and empty otherwise. The key itself is never returned.
- `acme` is the stored ACME settings, present even for `files` (then its strings are empty and `staging` is false). `ca` is `letsencrypt`, `zerossl`, `googletrust` or `custom`; an empty `ca` means Let's Encrypt. `directory_url` is set only for `custom`, and is an `https://` URL with no user information, query or fragment. `staging` selects that authority's test directory. `email` is the account contact and is not a credential. `challenge` is `http-01` or `dns-01`. `http01_port` is omitted when unset, and unset means port 80; `0` means Core opens no port for the challenge. `dns_provider` is `cloudflare`, `alidns`, `tencentcloud` or `manual`, and is empty when the challenge is not `dns-01`.
- `credentials` says only whether a secret is stored (`set`). The secret is never returned. For `files` the object is empty. For ACME it always has `eab.kid` and `eab.hmac`. For `dns-01` it also has that provider's names: `cloudflare.api_token`; `alidns.access_key_id` and `alidns.access_key_secret`; `tencentcloud.secret_id` and `tencentcloud.secret_key`. `manual` adds none.
- `certificate` is null until a certificate is loaded. `issuer` and `subject` are the certificate's common names. `names` is the DNS names on the certificate (null when it has none). `serial` is lowercase hex. `staging` is true when the served certificate came from a test directory. `other_ca` is present and true only for an ACME certificate that was issued by a directory other than the one configured now; it keeps being served until a new order replaces it.
- `last_attempt_at` is when an order was last tried, or null. `last_renewal_at` and `next_renewal_at` are set for an enabled ACME configuration and are null when there is no time yet.
- `last_error` is null when there is no problem. Otherwise `code` and `message` describe it, `at` is when it was recorded, `warning` is true when the problem does not by itself stop a good certificate from being served (the name on the certificate does not match `domain`, or the order was cancelled), and `retry_at` is when Core will try again. `message` is Core's own text and never a credential. `code` is one of `files_unreadable`, `files_mismatch`, `files_expired`, `domain_mismatch`, `config_invalid`, `http01_port_denied`, `http01_port_busy`, `challenge_failed`, `dns_credentials`, `dns_zone_not_found`, `dns_propagation_timeout`, `dns_provider`, `manual_timeout`, `eab_required`, `eab_invalid`, `rate_limited`, `ca_unreachable`, `ca_error`, `cancelled`, `store_failed`.
- `listen_error` is a short reason the HTTPS port could not be opened, or null.
- `pending` is null except during a manual TXT challenge. `kind` is `manual_txt` (add the record), `manual_txt_checking` (it was confirmed; DNS and the authority are checking) or `manual_txt_remove` (the order is done; the record can be removed). `order_id` identifies the order. `records` has one entry: `name` is the DNS name without a trailing dot, `value` is the TXT value. `expires_at` is set for `manual_txt` and is one hour after the record was published. `state` is `waiting_owner` only while `kind` is `manual_txt`.
- `propagation` is omitted except for a `dns-01` certificate that has been issued. `checked` means this host saw the TXT record. `assumed` means it did not, Core waited, and the authority validated anyway.
- `handshake_errors` counts failed TLS handshakes since this process started. A change of this number alone does not send `tls.changed`.

Errors:

- `503` TLS is not available in this Core

Events: none.

### GET /v1/update

Auth: any key
Effect: Reports the desktop app's updater, as that app last described it. A Core with no desktop report in the last 15 seconds answers with every flag false. The response sends `Cache-Control: no-store`.

Request: no body.

Response: `200`

```json
{
  "supported": true,
  "version": "1.2.3",
  "available_version": "1.2.4",
  "ready": true,
  "downloading": false,
  "automatic": true,
  "error": false
}
```

- `supported` is true when this desktop can update itself.
- `version` is the desktop version that is running. An empty string means no fresh report.
- `available_version` is the version already downloaded and ready to install. Omitted when there is none.
- `ready` is true when that package can be installed. `downloading` is true while the desktop is still downloading it.
- `automatic` is true when the desktop will install a ready update by itself.
- `phase` is omitted while idle. `requested` means an install has been asked for. `waiting_steps` means a run or a memory review is still finishing. `waiting_persistence` means a run cannot be saved for a restart. `waiting_idle` means runs are going and will not be suspended. `restarting` means the desktop is restarting. Until the desktop has accepted an install request, `phase` stays `requested` even if the desktop has already moved on.
- `error` is true when the last download or restart failed. There is no error message. While an install request is still waiting for the desktop, `error` is false.
- A report older than 15 seconds is discarded. The body is then `supported`, `ready`, `downloading`, `automatic` and `error` all false, and `version` `""`, with `available_version` and `phase` omitted.

Errors: none.

Events: none.

### GET /v1/update/standalone

Auth: admin
Effect: Reports this standalone Core's own updater. A desktop-bundled Core and a development build do not have one. The response sends `Cache-Control: no-store`.

Request: no body.

Response: `200`

```json
{
  "version": "1.2.3",
  "auto": true,
  "latest": "1.2.4",
  "last_check": "2026-10-14T03:00:00Z"
}
```

- `version` is the version of this Core.
- `auto` is true when a newer release found by the periodic check is installed by itself. It is read from the current settings on every call. `POST /v1/update/standalone` installs a newer release even when `auto` is false.
- `phase` is omitted while idle. `checking` is a check in progress. Then `downloading`, `waiting` (until a safe point to restart), `installing` and `restarting`.
- `latest` is the newest release the last check found. Omitted when that check found nothing newer.
- `last_check` is when a check last finished. Omitted until the first check finishes.
- `last_error` is the last check or install failure. Omitted when the last check succeeded. It can name a rate limit.
- `rate_limited_until` is when an anonymous rate limit resets. Omitted when no limit applies. It is read live, not frozen at `last_check`.

Errors:

- `404` not a standalone Core

Events: none.

### GET /v1/usage

Auth: any key
Effect: Returns recorded model-request attempts, newest first, plus totals for the same filter. Records do not contain prompts, replies or credentials. Deleting a conversation deletes its records. Editing or compacting a conversation does not.

Query: all optional.

- `session` matches that conversation, including every lane of it, or one lane when the value is the conversation id, a tilde, and the lane id.
- `model`, `provider`, `purpose`, `source` and `status` keep rows whose field is exactly that string.
- `from` and `to` are RFC3339. `from` is inclusive and `to` is exclusive (`started` >= `from` and `started` < `to`). When both are set, `to` must be after `from`. A date with no time is rejected.
- `origin` is `interactive`, `scheduled`, `compaction`, `memory_review` or `recap`. Omitted returns every bucket. It narrows the page, the summary, and the session, day and model groups. It does not narrow `breakdown.origins`.
- `executor` is omitted, `gyres`, `all`, `agents`, `codex` or `grok`. Omitted and `gyres` keep Core's own rows (a missing `executor` counts as `gyres`). `agents` keeps every external-agent row. `all` keeps every row. `codex` and `grok` keep that agent.
- `tz` is an integer number of minutes, from -840 to 840, the same value JavaScript's `getTimezoneOffset()` returns. It only names `breakdown.days`: that many minutes are subtracted from the UTC time (a negative value moves the clock forward) and the result is formatted `YYYY-MM-DD`. Omitted means `0`.
- `offset` is how many matching rows to skip. Omitted means `0`. Past the end, `records` is empty and `total` is still the full count.
- `limit` is the page size. Omitted or `0` means 100. Above 200 is treated as 200. The response `limit` is the value after that rule.

A row's origin bucket is `plugins` when `purpose` is anything other than empty, `chat`, `compaction`, `memory_review` or `recap`. Otherwise `compaction`, `memory_review` and `recap` stay in the bucket of that name. Otherwise `source` `scheduled` is `scheduled`, and everything else is `interactive`. `plugins` is not a legal `origin` query value.

Request: no body.

Response: `200`

```json
{
  "records": [
    {
      "version": 1,
      "id": "0123456789abcdef0123456789abcdef",
      "request_id": "fedcba9876543210fedcba9876543210",
      "attempt": 1,
      "session_id": "2026-10-14T12-00-00_01234567-89ab-cdef-0123-456789abcdef",
      "run_id": "abcdef0123456789abcdef0123456789",
      "source": "interactive",
      "provider": "default",
      "model": "example-model",
      "format": "chat_completions",
      "purpose": "chat",
      "started": "2026-10-14T12:00:00Z",
      "finished": "2026-10-14T12:00:02Z",
      "duration_ms": 2000,
      "status": "completed",
      "usage_state": "reported",
      "usage": {
        "cache_usage_reported": true,
        "input_tokens": 100,
        "output_tokens": 20,
        "cache_read_tokens": 40,
        "cache_write_tokens": 0
      }
    }
  ],
  "summary": {
    "cache_known": 1,
    "cache_input": 140,
    "cache_read": 40,
    "requests": 1,
    "known": 1,
    "partial": 0,
    "unknown": 0,
    "duration_ms": 2000,
    "tokens": { "input": 100, "output": 20, "cache_read": 40, "cache_write": 0 },
    "cost_usd": 0,
    "cost_known": 0
  },
  "breakdown": {
    "sessions": [
      {
        "session_id": "2026-10-14T12-00-00_01234567-89ab-cdef-0123-456789abcdef",
        "requests": 1,
        "known": 1,
        "partial": 0,
        "unknown": 0,
        "tokens": { "input": 100, "output": 20, "cache_read": 40, "cache_write": 0 },
        "last": "2026-10-14T12:00:00Z",
        "models": ["example-model"],
        "origins": {
          "interactive": {
            "requests": 1,
            "known": 1,
            "partial": 0,
            "unknown": 0,
            "tokens": { "input": 100, "output": 20, "cache_read": 40, "cache_write": 0 }
          }
        },
        "cost_usd": 0
      }
    ],
    "days": [
      {
        "key": "2026-10-14",
        "requests": 1,
        "known": 1,
        "partial": 0,
        "unknown": 0,
        "tokens": { "input": 100, "output": 20, "cache_read": 40, "cache_write": 0 },
        "last": "2026-10-14T12:00:00Z",
        "cost_usd": 0
      }
    ],
    "models": [
      {
        "key": "example-model",
        "requests": 1,
        "known": 1,
        "partial": 0,
        "unknown": 0,
        "tokens": { "input": 100, "output": 20, "cache_read": 40, "cache_write": 0 },
        "last": "2026-10-14T12:00:00Z",
        "cost_usd": 0
      }
    ],
    "origins": [
      {
        "key": "interactive",
        "requests": 1,
        "known": 1,
        "partial": 0,
        "unknown": 0,
        "tokens": { "input": 100, "output": 20, "cache_read": 40, "cache_write": 0 },
        "last": "2026-10-14T12:00:00Z",
        "cost_usd": 0
      },
      {
        "key": "scheduled",
        "requests": 0,
        "known": 0,
        "partial": 0,
        "unknown": 0,
        "tokens": { "input": 0, "output": 0, "cache_read": 0, "cache_write": 0 },
        "cost_usd": 0
      },
      {
        "key": "compaction",
        "requests": 0,
        "known": 0,
        "partial": 0,
        "unknown": 0,
        "tokens": { "input": 0, "output": 0, "cache_read": 0, "cache_write": 0 },
        "cost_usd": 0
      },
      {
        "key": "memory_review",
        "requests": 0,
        "known": 0,
        "partial": 0,
        "unknown": 0,
        "tokens": { "input": 0, "output": 0, "cache_read": 0, "cache_write": 0 },
        "cost_usd": 0
      },
      {
        "key": "recap",
        "requests": 0,
        "known": 0,
        "partial": 0,
        "unknown": 0,
        "tokens": { "input": 0, "output": 0, "cache_read": 0, "cache_write": 0 },
        "cost_usd": 0
      },
      {
        "key": "plugins",
        "requests": 0,
        "known": 0,
        "partial": 0,
        "unknown": 0,
        "tokens": { "input": 0, "output": 0, "cache_read": 0, "cache_write": 0 },
        "cost_usd": 0
      }
    ]
  },
  "total": 1,
  "offset": 0,
  "limit": 100
}
```

- `records` is the page, newest `started` first. The same `started` is ordered by `id` descending. The array is empty, not null, when nothing matches. `total` counts every matching row, not just the page. `summary` and `breakdown` also cover every matching row.
- `version` is `1`. `id` is 32 lowercase hex and identifies one attempt. `request_id` is the logical request; retries share it and get a new `id` and a higher `attempt`. `session_id` is the conversation. `lane` is omitted for the main lane; otherwise `session_id` is the base conversation and the lane handle is `session_id`, a tilde, and `lane`. `run_id` is the run. `schedule_id` is set when a scheduled task started the run.
- `source` is the label the turn was started with, such as `interactive`, `scheduled` or `background`. `provider` and `model` are what the request was sent to. `fallback_from` is `provider id/model` of the conversation's model when a fallback model served this request instead, and is omitted otherwise.
- `format` is `chat_completions` or `responses` for Core's own request. For an external-agent row it is the agent name, `codex` or `grok`.
- `purpose` is why the request was made: `chat`, `compaction`, `memory_review`, `recap`, or a hook or plugin purpose. Older rows may omit it; that means chat.
- `caller` is omitted for Core's own requests. It is `hook:` plus a handler id, or `key:` plus an access key id, when the request was made for that caller.
- `started` is when the attempt began. `finished` is omitted until it ends. `duration_ms` is the model call, not tool time, and is `0` until the attempt finishes.
- `status` is `running` while the attempt is open, then `completed`, `failed` or `cancelled`. An attempt still `running` when Core restarts is returned as `interrupted`.
- `usage_state` is `reported` when a successful attempt included usage, `partial` when a failed or cancelled attempt included some, and `unknown` when none arrived. Do not read zeros as a bill when `usage_state` is `unknown`. A restarted `running` attempt is returned as `unknown`. An external-agent row is `reported` when its `status` is `completed` and `partial` otherwise.
- `usage.input_tokens` is non-cached input. `output_tokens` includes reasoning tokens when the provider includes them there. `cache_read_tokens` and `cache_write_tokens` are separate. `cache_usage_reported` is true when the provider reported cache figures, and is omitted when false.
- `executor` is omitted for Core (read those rows as `gyres`) and is `codex` or `grok` for an external agent. `cost_usd` is that agent's own estimate for the turn, and `cost_source` is `agent_estimate`. Both are omitted when the agent gave no estimate. Core does not price its own requests.
- `error` on a failed Core attempt is `模型请求失败；详情见所属对话`. On a cancelled Core attempt it is `请求已取消`. `failure`, when present, is the same shape as a request error's `failure` (`code`, `message`, `http_status`, `details`) and is a sanitised provider error. A cancelled attempt omits `failure`.
- `diagnostics`, when present, compares this attempt with the previous one for the same conversation and purpose. It holds hashes only, never the prompt. `kind` is `baseline`, `unchanged`, `append_only`, `changed` or `limited`. `changes` lists what moved: `route`, `context`, `system`, `tools`, `parameters`, `history`. `previous_attempt` is the earlier attempt id. `first_difference` names the first item that differs. `common_items`, `item_count` and `previous_items` count items. `request_bytes` is the payload size. `body_hash`, `system_hash`, `tools_hash` and `parameters_hash` are hex digests. `context_epoch` names the context snapshot. `limited` is true when the comparison stopped at 4096 items.
- `summary.requests`, `known`, `partial` and `unknown` count rows by `usage_state` (`known` is `reported`; anything other than `reported` or `partial` counts as `unknown`). `tokens` sums every matching row: `input`, `output`, `cache_read`, `cache_write`. `duration_ms` sums the attempts. `cache_known` counts rows that are `reported` and have `cache_usage_reported`. For those rows only, `cache_input` is input plus cache read plus cache write, and `cache_read` is the cache-read sum. `cost_usd` sums agent estimates, and `cost_known` counts the rows that had one.
- `breakdown.sessions` is one object per conversation, largest token total first, then latest `last`, then `session_id`. `models` on a session is the distinct model names, sorted. `origins` on a session has only the buckets that had rows. `cost_usd` sums that session's agent estimates.
- `breakdown.days` is sorted by `key` ascending. `key` is the `YYYY-MM-DD` from `tz`. `breakdown.models` is one object per model name (`key`), largest token total first.
- `breakdown.origins` always has six objects, in this order, even when every count is zero and even when `origin` filtered the rest of the response: `interactive`, `scheduled`, `compaction`, `memory_review`, `recap`, `plugins`. A group has `key`, `requests`, `known`, `partial`, `unknown`, `tokens`, `cost_usd`, and `last` (omitted when the group is empty).
- `days`, `models` and `sessions` are empty arrays when they have no rows. They are not null.

Errors:

- `400` invalid time filter
- `400` invalid pagination (`offset` or `limit` is not a whole number, or is negative)
- `400` invalid time range (`to` is not after `from`)
- `400` invalid origin
- `400` invalid executor
- `400` invalid time zone
- `500` cannot read usage records

Events: none.

### GET /v1/version

Auth: any key
Effect: Reports this Core's build version and the interface version. The response sends `Cache-Control: no-store`.

Request: no body.

Response: `200`

```json
{ "version": "dev", "api_version": 1 }
```

- `version` is the Core build version. A development build sends `dev`. A release build sends its version string.
- `api_version` is an integer. It goes up by one on every breaking change listed in CHANGELOG.md. Additions do not change it. A frontend compares it with the `api_version` it was built for. A Core that answers this route with `404` is older than this spec.

Errors: none.

Events: none.

### POST /v1/browser-runtime/install

Auth: any key
Effect: Starts installing the managed browser, if it is not already installed or already installing. The call returns as soon as the job is accepted. Poll `GET /v1/browser-runtime` for progress. A second call while an install is running, or while the browser is already ready, does not start another install.

Request: no body.

Response: `202`

The body is the same object as `GET /v1/browser-runtime`. When a new install starts, `state` is `installing` and `phase` is `checking` until the job moves on. The install gives up after 15 minutes.

Errors:

- `503` the managed browser is not running in this Core. The body is only `{"error":"Browser unavailable"}`, with no `failure`.
- `400` the install could not start. The body is only `{"error":"<reason>"}`, with no `failure`. When this host cannot install it and the browser is not already present, the reason is `automatic installation supports macOS Apple silicon, Linux amd64/arm64, and Windows amd64/arm64`. A later GET then has `state` `error` and the same text in `error`.

Events: none.

### POST /v1/code-runtime/install

Auth: any key
Effect: Starts installing the managed code runtime, if it is not already installed or already installing. The call returns as soon as the job is accepted. Poll `GET /v1/code-runtime`. A second call while an install is running, or while a ready runtime file is present, does not start another install.

Request: no body.

Response: `200`

The body is the same object as `GET /v1/code-runtime`. When a new install starts, `state` is `installing`. The install gives up after 15 minutes.

Errors:

- `503` the code runtime is not running in this Core. The body is only `{"error":"code runtime unavailable"}`, with no `failure`.
- `503` the install could not start. The body is only `{"error":"<reason>"}`, with no `failure`. On a host other than macOS Apple silicon the reason is `managed Bun currently supports macOS arm64`. A later GET then has `state` `error` and the same text in `error`.

Events: none.

### POST /v1/identity/challenge

Auth: none
Effect: Proves that this Core holds a given access key, without the client sending that key. A client runs it before it sends its key to an address it is not sure about: an address found on the local network (the `_gyres._tcp` TXT `id` is public, so anything can claim it), or a saved address that may now belong to another device. Changes nothing. Nothing about the request is logged. The response sends `Cache-Control: no-store`.

Request:

```json
{
  "key_id": "0123456789abcdef01234567",
  "nonce": "000102030405060708090a0b0c0d0e0f101112131415161718191a1b1c1d1e1f",
  "target": "http://192.0.2.10:15100"
}
```

- `key_id` (string, required, 1–64 characters) names the client's key without revealing it. For a key of the form `gyres_<id>_<secret>` it is `<id>`. For any other key it is `legacy-` followed by the first 24 lowercase hex characters of the SHA-256 of the whole key. Another Core holding a peer key `gyp_<id>_<secret>` (see [Bot peers](peers.md)) names it `peer-<id>`; the proof is computed the same way from that key.
- `nonce` (string, required) is 32 to 128 lowercase hex characters, an even number, fresh random for every challenge (send at least 16 random bytes; 32 is recommended).
- `target` (string, required, at most 512 bytes, no control characters) is the `http` or `https` origin the client sent this request to: scheme, host and optional port, no path other than `/`, no query, fragment or user info.
- The body is one JSON object of at most 4096 bytes with only these fields.

Response: `200`

```json
{
  "instance_id": "0123456789abcdef0123456789abcdef",
  "target_ok": true,
  "proof": "d9600a2ec1215ce953bcdd7ef1ea0893fde6ce3012a0f22c1acfa0dd97c9c761"
}
```

- `instance_id` is the same value as in `GET /v1/identity`.
- `target_ok` is true when `target` is one of the origins `GET /v1/identity` lists in `addresses` at this moment. Scheme and host compare without case and a missing port counts as 80 or 443. Forwarding headers are not used.
- `proof` is 64 lowercase hex characters: `HMAC-SHA256(k, m)`, where `k` is the 32-byte SHA-256 of the full access key (UTF-8) and `m` is the UTF-8 text `gyres-identity-v1`, `nonce`, `instance_id`, `target` (exactly as sent) and `1` or `0` for `target_ok`, joined with `\n` (one newline byte, none at the end).
- To verify, compute the same HMAC from your stored key, the nonce you sent, the target you sent and the `instance_id` and `target_ok` received, compare in constant time, and check `instance_id` against the one you saved for this Core. Send the key only when it matches. A relay that forwards your challenge to the real Core gets `target_ok` false, since your target names the relay, and cannot change the flag without breaking the proof. Accept `target_ok` false only for an address you already trust for other reasons (one you saved, for example behind a proxy or a port forward), never for a newly discovered one.
- An unknown or malformed `key_id` answers the same way, with a proof that does not verify under any key, so the route does not reveal which key ids exist. A revoked or replaced key still gets a valid proof; sending it then returns `401`.
- Test vectors: [identity-proof-vectors.json](../identity-proof-vectors.json).

Errors:

- `400` invalid request (body is not one JSON object of the fields above, a field is missing or malformed, or the body is larger than 4096 bytes)
- `429` too many attempts; try again later. More than 60 challenges from one client address within one minute; the address is taken under the same trusted-proxy rules as pairing. The response sets `Retry-After: 60`.
- `404` identity proof is not available (Core runs without access keys). A Core older than this route answers `401`, because it needs a key for every unknown route.

Events: none.

### POST /v1/tls/issue

Auth: admin
Effect: Asks Core to obtain or renew the ACME certificate in the background. Saving HTTPS settings does not start an order; this call does. The first certificate, and a certificate from a newly chosen authority, come only from this call. The response sends `Cache-Control: no-store`.

Request:

```json
{ "force": false }
```

- `force` (boolean, optional) renews even when the current certificate is not due yet. Omitted means false. An empty body is accepted and means false. Unknown fields are ignored. The body is at most 65536 bytes.

Response: `202`

The body is the same object as `GET /v1/tls`, read just after the order was accepted. With no certificate yet, `state` is `issuing`. A renewal of a certificate that is already being served can stay `active` until the order moves it. A refused call does not send `tls.changed`. An accepted call sends it when the status changed, and the order sends it again whenever the status changes after that. `handshake_errors` alone does not. Read `GET /v1/tls` again after the event.

Errors:

- `503` TLS is not available in this Core
- `400` invalid json (a body was sent and it is not JSON, or it is larger than 65536 bytes)
- `409` the order was not started. The body has `error` and `code`, and no `failure`:

```json
{ "error": "the certificate is not due for renewal", "code": "not_due" }
```

- `code` `not_acme`, error `TLS is not enabled with an ACME certificate`: HTTPS is off, the source is not ACME, or the saved settings do not validate.
- `code` `busy`, error `an order is already running`.
- `code` `rate_limited`, error `the certificate authority's rate limit applies until retry_at`. `GET /v1/tls` carries that time on `last_error.retry_at`.
- `code` `not_due`, error `the certificate is not due for renewal`: a current certificate from the configured authority is not yet inside its renewal window, and `force` is false.

Events: `tls.changed`

### POST /v1/update

Auth: admin
Effect: Asks the desktop app to install the update it has already downloaded and marked ready. This call does not download a package. A repeat while that request is still in progress, or while the desktop is already restarting or waiting, is accepted again and does not replace the first request. The response sends `Cache-Control: no-store`.

Request:

```json
{ "version": "1.2.4" }
```

- `version` (string) must be exactly the `available_version` from a fresh `GET /v1/update`. The body is one JSON object of at most 128 KiB and may contain only this field.

Response: `202`

```json
{ "accepted": true }
```

Until the desktop accepts the request, `GET /v1/update` reports `phase` `requested` and `error` false.

Errors:

- `400` invalid settings request (the body is not a JSON object of the field above, or is larger than 128 KiB)
- `409` update is not ready; refresh status. The desktop has not reported in the last 15 seconds, `supported` or `ready` is false, or `version` is empty or is not the reported `available_version`.

Events: none.

### POST /v1/update/standalone

Auth: admin
Effect: Checks now for a newer standalone release and, when one is found, installs it and restarts Core. This runs even when `auto` is false. The call waits for the check, up to 2 minutes. When an install is already past the checking step, the call does not start another check and reports that install. The response sends `Cache-Control: no-store`.

Request: no body.

Response: `200`

```json
{ "latest": "1.2.4", "installing": true }
```

- `latest` is the release the check found. An empty string means this Core is already current.
- `installing` is true when a newer release was found and installation follows, or when an install was already running. It is false when `latest` is empty.

Errors:

- `404` not a standalone Core
- `429` the anonymous rate limit has not reset. The message says when a check is allowed again. There is no `Retry-After` header. `GET /v1/update/standalone` reports the same deadline as `rate_limited_until`.
- `502` the check failed, or it did not finish within 2 minutes

Events: none.

### PUT /v1/task-limits

Auth: any key
Effect: Replaces the global task-round cap. Both fields are required. Tasks that are already running keep the cap they started with; the new values apply to tasks that start afterwards.

Request:

```json
{ "enabled": true, "max_turns": 150 }
```

- `enabled` (boolean) turns the cap on or off. Turning it off keeps `max_turns`.
- `max_turns` (integer) is the cap, from 100 through 4294967295. A fraction, a negative number, or a number above that range is rejected.
- `goal_max_continuations` (integer, optional) is the goal continuation cap, 1 through 10000. Left out, the saved value stays.
- The body is at most 4096 bytes. Unknown fields are ignored. Omitting either field is rejected.

Response: `200`

The body is the saved object, the same shape as `GET /v1/task-limits`.

Errors:

- `400` `enabled 和 max_turns 必须完整提供，次数必须是至少 100 的整数` — the body is not JSON, is larger than 4096 bytes, a field is missing, or `max_turns` is not a whole number in range
- `400` `最大循环次数必须是至少 100 的整数` — both fields were read and `max_turns` is below 100
- `500` the setting could not be saved

Events: none.
