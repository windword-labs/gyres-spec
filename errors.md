# Errors

How Core reports a failed request, and how a client should react.

## The JSON error body

Almost every handler error is JSON:

```json
{
  "error": "conversation not found",
  "failure": {
    "code": "core_request_failed",
    "message": "conversation not found",
    "http_status": 404,
    "details": "conversation not found"
  }
}
```

- `error` (string): the human-readable message, with credentials and URL query
  strings removed. Show it to the user; do not match on its wording.
- `failure` (object): a classified form for display.
  - `code`: `core_request_failed` for Core's own errors. When the message
    carries an upstream (model provider) HTTP status, it is classified instead:
    `invalid_request`, `authentication_failed`, `permission_denied`,
    `not_found`, `timeout`, `request_too_large`, `unsupported_media`,
    `rate_limited`, `service_unavailable`, `connection_failed` or
    `request_failed`.
  - `message`: short text for the user.
  - `http_status`: the status (omitted when unknown).
  - `details`: the sanitised original message (omitted when empty).

Rely on the HTTP status first, then `failure.code`, then (only where an endpoint
documents one) a top-level `code`.

## Coded errors

Some errors add a top-level `code` the client is expected to branch on. They
are listed per endpoint under **Errors**. Core-wide ones:

| Status | `code` | Meaning | Client action |
| --- | --- | --- | --- |
| `503` | `core_starting` | Core is still starting. | Retry after `Retry-After` (5 s). |
| `503` | `storage_maintenance` | Core is moving its data directory. | Show a maintenance state; retry. `GET /v1/storage` shows progress. |
| `503` | `storage_unavailable` | The data directory is unavailable (disk missing, permissions). | Show the error; `GET /v1/storage` explains, an administrator fixes it. |
| `503` | `storage_slow` | The data directory did not answer in time. | Retry later. |

While storage is unavailable or under maintenance, `GET /health`,
`GET /v1/identity`, `GET /v1/version` and the `/v1/storage` routes keep
answering so a client can show what is wrong.

## Authentication and access errors

These come from the access layer, before any handler. Some are **plain text**,
not JSON: read the status code, not the body.

| Status | Body | When |
| --- | --- | --- |
| `401` | `unauthorized` (text) | No `Authorization: Bearer <key>`, an unknown key, or a revoked key. Ask the user to reconnect or pair again. |
| `403` | `administrator key required` (JSON) | An `Auth: admin` route called with an ordinary key. Hide admin features for non-admin keys (`GET /v1/identity` tells you the role). |
| `403` | `invalid host` (text) | The `Host` header is not an IP address, `localhost`, or Core's configured HTTPS domain. Connect by IP or by that domain. |
| `403` | `manage configuration on the host` (text) | A route that only the Core host itself may use. |

A key revoked while a request or stream is open ends that request within about
a quarter of a second.

## Other common statuses

- `400`: invalid JSON or an invalid field; the message names it.
- `404`: the conversation, project, plugin, ... does not exist.
- `409`: the request conflicts with the current state (for example a run is
  already active); the endpoint lists when.
- `413`: the body or upload is too large.
- `429`: a limit was hit; honour `Retry-After` when present.
- `5xx`: Core or an upstream failed; the message says which. Retrying an
  idempotent `GET` is safe; check state before retrying a `POST`.

## CORS

Core answers every origin (`Access-Control-Allow-Origin: *`) and allows the
`Authorization`, `Content-Type` and `Range` headers, so a browser frontend on
any origin can call it. A browser page served over HTTPS can only reach a Core
over HTTPS, on `127.0.0.1`, or (with the browser's local-network permission) on
a private LAN address.
