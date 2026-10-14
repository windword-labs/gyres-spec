# Device pairing

Device pairing gives a new device its own ordinary access key. An administrator creates a short-lived code; the device redeems that code with no access key and receives a new secret once. Pairing only adds a key. It cannot list, rename, revoke or regenerate keys, and the new key is not an administrator key.

A code lives only in memory. Restarting Core drops it. At most 5 codes are active at once: not yet redeemed, and not yet expired. Each code lasts 2 minutes, works once, and is kept only as a hash. The pairing `id` is a different value from the code. It cannot be redeemed. Status and cancel refer to a code by this `id`.

The code itself is 26 characters from the RFC 4648 base32 alphabet (`A–Z` and `2–7`). The `id` is 16 characters from that same alphabet. Letter case does not matter when redeeming; surrounding whitespace is ignored.

JSON responses are `application/json`. Every response from these routes sends `Cache-Control: no-store`. An error body is `{"error":"<message>","failure":{"code":"core_request_failed","message":"<message>","http_status":<status>,"details":"<message>"}}`, except the `409` on `POST /v1/pairing`, which has `code` and no `failure`.

### GET /v1/pairing/{id}/status

Auth: admin
Effect: Reports whether this pairing code is still waiting, has been redeemed, or is no longer usable. An unknown id is reported as expired.

Path: `id` pairing id returned when the code was created.

Request: no body.

Response: `200`

```json
{ "state": "pending" }
```

```json
{ "state": "redeemed", "name": "iPhone · 10月4日" }
```

```json
{ "state": "expired" }
```

- `state` is `pending` while the code is unused and `expires_at` is still ahead, `redeemed` after the code was accepted (including when the key could not be saved), or `expired` when the code has lapsed, was cancelled, is unknown, or has been dropped from memory. This route does not answer `404`.
- `name` is present only when `state` is `redeemed`. It is the new key's name, or `""` if the code was consumed but the key was not saved. The code and the secret are never included.
- `redeemed` stays visible until more than 10 minutes after redemption; a later read returns `expired`. An unused code that has expired keeps returning `expired` until more than 10 minutes after `expires_at`, and then the same way.

Errors: none.

Events: none.

### POST /v1/pairing

Auth: admin
Effect: Creates one pairing code and the address a device should use to reach this Core. When no such address can be named, nothing is created.

Request: no body.

Response: `201`

```json
{
  "code": "K3NQW4E6R7T2Y5PAMZXC7HJSBD",
  "core_url": "http://192.0.2.10:15100",
  "expires_at": "2026-10-14T12:02:00Z",
  "id": "K3NQW4E6R7T2Y5PA"
}
```

- `code` is the one-time pairing code, returned only here. Show it to the device; a later status read will not repeat it.
- `id` identifies this code for status and cancel. It is not the code.
- `expires_at` is 2 minutes after creation, UTC.
- `core_url` is the address the device should use. The request is first taken as it arrived: scheme `https` when the connection is TLS, otherwise `http`, and the host header. When the peer is a trusted proxy, exactly one forwarding family is present, and those headers are well formed, that view is replaced by the proxy's scheme and host (`Forwarded`, or `X-Forwarded-Proto` together with `X-Forwarded-Host`). Both families at once, an untrusted peer, or a malformed header leave the connection unchanged.
- If that host is a non-loopback, non-unspecified IP, or any other non-empty name other than `localhost` in any case, `core_url` is the scheme plus that host, port included.
- Otherwise, if Core is serving HTTPS, `core_url` is that HTTPS address. It uses the name Core serves, and includes `:<port>` only when the port is not 443.
- Otherwise the listen address decides. Loopback only (`127.0.0.1`, `localhost`, IPv6 loopback, or no address set) stores no code and returns `409`. Every interface (the IPv4 or IPv6 wildcard) uses `http://` plus one IPv4 address of this computer and the HTTP port, private addresses first; if none is up, or the addresses cannot be listed, no code is stored and the response is `409`. Any other listen address uses `http://` plus that address and the HTTP port.

Errors:

- `409` Core only listens on this computer; phones cannot connect. No code is stored. The body is not the usual error envelope:

```json
{
  "code": "loopback_only",
  "error": "Core only listens on this computer; phones cannot connect"
}
```

- `429` too many pairing codes are active; wait or cancel one

Events: none.

### POST /v1/pairing/redeem

Auth: none
Effect: Exchanges one valid pairing code for a new ordinary access key and returns its secret once. The code is consumed before the key is saved, so a second request, a concurrent request, or a failed save cannot mint another key from it.

Request:

```json
{ "code": "K3NQW4E6R7T2Y5PAMZXC7HJSBD", "device_name": "iPhone" }
```

- `code` (string) is the pairing code. Required for success. Missing, empty, wrong, expired, cancelled and already-used values all fail the same way. Comparison trims surrounding whitespace and ignores letter case.
- `device_name` (string, optional) is a label for the new key. Leading and trailing whitespace is removed, runs of whitespace become one space, and control characters and other non-printable characters are dropped. What remains is cut to 40 Unicode code points and trimmed again. If nothing remains, the `User-Agent` header is used, matched in this order, case-insensitive: `iphone` becomes `iPhone`, then `ipad` becomes `iPad`, then `android` becomes `Android`, then `macintosh` becomes `Mac`, then `windows` becomes `Windows`, then `linux` becomes `Linux`, otherwise `新设备`. The stored name is that label, then ` · `, then Core's local month and day with no leading zeros, written `10月4日`.
- The body is one JSON object of at most 4096 bytes and may contain only these fields.

Response: `201`

```json
{
  "key_id": "0123456789abcdef01234567",
  "name": "iPhone · 10月4日",
  "secret": "gyres_0123456789abcdef01234567_0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef"
}
```

- `secret` is the new access key, shown only in this response. It has the form `gyres_`, then `key_id`, then `_`, then 64 lowercase hex characters. Send it later as `Authorization: Bearer <secret>`.
- `key_id` is 24 lowercase hex characters.
- `name` is the label stored for the key, including the date suffix. The same text is what status returns while the redeemed record remains.

Errors:

- `404` pairing is not available
- `400` invalid request (body is not one JSON object of the fields above, or is larger than 4096 bytes)
- `429` too many attempts; try again later. Ten failed redeems from one client address in the past 10 minutes. That address is the connecting address, or the client address a trusted proxy reports under the same trust rules as `core_url`. While the limit applies, a correct code is refused too. The response sets `Retry-After: 600`. The refusal does not itself add another failure.
- `410` pairing code is invalid, expired or already used. Each of these counts as a failed attempt. The body is the same for a wrong code, an expired code and a used code.
- `500` could not create access key. The code is already consumed; redeeming it again returns `410`, and status shows `redeemed` with `name` `""`.

Events: none.

### DELETE /v1/pairing/{id}

Auth: admin
Effect: Cancels an unused pairing code so it can no longer be redeemed and no longer counts toward the limit of 5. A code already redeemed is left redeemed. An unknown or already expired id is still a success.

Path: `id` pairing id returned when the code was created.

Request: no body.

Response: `204` no body.

Errors: none.

Events: none.
