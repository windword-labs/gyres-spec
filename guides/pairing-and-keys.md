# Pairing and access keys

Almost every Core route needs an access key. The owner can give you one, or an administrator can mint a one-time pairing code that this device redeems for its own ordinary key. Store that secret, send it as `Authorization: Bearer $GYRES_KEY`, then read your role and check `api_version` before you build the rest of the UI.

Set `GYRES_KEY` to the secret you stored. The examples below use `http://127.0.0.1:15100`. A real Core may be a LAN address or an `https://` origin; `GET /v1/identity` lists the origins this Core publishes.

## 1. Use a key the owner gives you

Put the secret in the client's credential store and export it for the examples:

```bash
export GYRES_KEY='the-secret-the-owner-gave-you'
```

An ordinary key and the administrator key use the same header. The administrator key is the only one that can create a pairing code.

## 2. Or pair this device

Pairing only adds a key. It does not list, rename, revoke, or regenerate keys, and the new key is not an administrator key.

An administrator creates one code. The request has no body. The response is `201`:

```bash
curl -sS -D - -X POST \
  -H "Authorization: Bearer $GYRES_KEY" \
  http://127.0.0.1:15100/v1/pairing
```

```json
{
  "code": "K3NQW4E6R7T2Y5PAMZXC7HJSBD",
  "core_url": "http://192.0.2.10:15100",
  "expires_at": "2026-10-14T12:02:00Z",
  "id": "K3NQW4E6R7T2Y5PA"
}
```

Show `code` to the device. It is 26 characters from the base32 alphabet (`A–Z` and `2–7`), returned only here. `id` (16 characters from that alphabet) is how you poll and cancel; it cannot be redeemed. `expires_at` is two minutes after creation. `core_url` is the address the device should call. At most five codes are active. Each works once. Restarting Core drops every code.

Poll until the device has redeemed it. An unknown id is `expired`, not `404`.

```bash
curl -sS -H "Authorization: Bearer $GYRES_KEY" \
  http://127.0.0.1:15100/v1/pairing/K3NQW4E6R7T2Y5PA/status
```

```json
{ "state": "pending" }
```

`state` becomes `redeemed` after a successful redeem, and the body then includes `name`. `expired` means the code lapsed, was cancelled, or is unknown.

The device redeems with no access key. Letter case does not matter; surrounding whitespace is ignored. `device_name` is optional.

```bash
curl -sS -X POST http://127.0.0.1:15100/v1/pairing/redeem \
  -H 'Content-Type: application/json' \
  -d '{"code":"K3NQW4E6R7T2Y5PAMZXC7HJSBD","device_name":"iPhone"}'
```

`201`:

```json
{
  "key_id": "0123456789abcdef01234567",
  "name": "iPhone · 10月4日",
  "secret": "gyres_0123456789abcdef01234567_0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef"
}
```

`secret` is shown only in this response. It is `gyres_`, then `key_id` (24 lowercase hex characters), then `_`, then 64 lowercase hex characters. `name` is the cleaned label, then ` · `, then Core's local month and day. If `device_name` is empty, Core derives a label from the `User-Agent` header.

## 3. Store the secret

Replace `GYRES_KEY` with `secret` and keep it in the client's own credential store. Later requests send:

```bash
curl -sS -H "Authorization: Bearer $GYRES_KEY" \
  http://127.0.0.1:15100/v1/identity
```

## 4. Read the role

`GET /v1/identity` (`200`, `Cache-Control: no-store`):

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

`role` is `admin` or `user`. Hide administrator-only actions when it is `user`. `instance_id` is stable for this Core across restarts (empty when it cannot be stored). `addresses` is never null; `loopback` is always present. Use `instance_id` to tell two addresses of one machine apart.

## 5. Check compatibility

```bash
curl -sS -H "Authorization: Bearer $GYRES_KEY" \
  http://127.0.0.1:15100/v1/version
```

```json
{ "version": "dev", "api_version": 1 }
```

`version` is the Core build (`dev` on a development build, otherwise its version string). `api_version` is an integer and goes up by one on every breaking change. A frontend built for N works with a Core that reports N. When Core reports a higher number, read the entries above N in [CHANGELOG.md](../CHANGELOG.md). A `404` on this route means the Core is older than this spec. Ignore JSON fields you do not know; Core adds fields without notice.

## 6. Handle 401

A missing `Authorization: Bearer` header, an unknown key, or a revoked key is `401` with the plain-text body `unauthorized`. Read the status, not the body: this response is not JSON. Clear the stored secret and ask the owner for a key, or pair again. A key revoked while a request or stream is open ends that request within about a quarter of a second.

## Edge cases

- `POST /v1/pairing` with an ordinary key is `403` and the JSON message `administrator key required`.
- Core that only listens on this computer returns `409` and stores no code. The body is not the usual error envelope:

```json
{
  "code": "loopback_only",
  "error": "Core only listens on this computer; phones cannot connect"
}
```

- `429` on create means five codes are already active. `429` on redeem means ten failed attempts from this client address in ten minutes; the response sets `Retry-After: 600`, and a correct code is refused too while the limit applies.
- `410` on redeem means the code is invalid, expired, or already used. The body is the same for all three, and each counts as a failed attempt.
- `500` `could not create access key` means the code was already consumed. Redeeming again is `410`. Status shows `redeemed` with `name` `""`.
- `400` on redeem means the body is not one JSON object of `code` and `device_name`, or is larger than 4096 bytes.
- `DELETE /v1/pairing/{id}` (administrator) cancels an unused code and returns `204`. A redeemed code stays redeemed.
- While Core is still opening, `GET /v1/identity` and `GET /v1/version` wait up to 10 seconds and may then answer `503` with `code` `core_starting` and `Retry-After: 5`. That starting answer does not require a key. These two routes keep answering while the data directory is unavailable or being moved.
- `403` with the plain-text body `invalid host` means the `Host` header is not an IP address, `localhost`, or Core's configured HTTPS domain.

## Before sending the key to a new address

Once paired, a client usually knows several addresses for its Core (`addresses` from `GET /v1/identity`, local-network discovery). Before it sends the key to an address it did not just use, it can ask that address to prove it holds the key: `POST /v1/identity/challenge` with the key id, a fresh nonce and the origin it is calling, no key attached. Verify the HMAC with your own key and check `instance_id`; send the key only when both match. For a newly discovered address also require `target_ok` true. A Core older than this route answers `401`. Details and test vectors: [POST /v1/identity/challenge](../http/system.md#post-v1identitychallenge).

## See also

- [Device pairing](../http/pairing.md)
- [GET /v1/identity](../http/system.md#get-v1identity) and [GET /v1/version](../http/system.md#get-v1version)
- [Authentication and access errors](../errors.md#authentication-and-access-errors) and [coded errors](../errors.md#coded-errors)
- [Changelog](../CHANGELOG.md)
