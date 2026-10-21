# Gyres relay

The Gyres relay lets paired devices reach this Core from outside the LAN. Core keeps one outbound WebSocket to the relay; devices open their own socket to it and talk to Core through an end-to-end encrypted tunnel (`Noise_NK_25519_ChaChaPoly_SHA256`, Core's static key pinned from `ck`). The relay sees neither requests nor access keys. Tunnelled requests go through the same authentication as direct ones. The relay is off by default and needs a Gyres account sign-in, which only this Core uses; devices never need an account.

JSON responses are `application/json`. Every response from these routes sends `Cache-Control: no-store`. An error body is `{"error":"<message>","failure":{...}}` unless stated otherwise.

### GET /v1/relay

Auth: admin
Effect: Reports the relay switch, the signed-in account and the connection.

Request: no body.

Response: `200`

```json
{
  "enabled": true,
  "account": { "email": "owner@example.com" },
  "state": "connected",
  "relay_id": "abcdefghijklmnopqrstuvwxyz",
  "address": "https://relay.gyres.app/c/abcdefghijklmnopqrstuvwxyz",
  "client_key": "base64url X25519 public key"
}
```

- `account` is `null` when no account is signed in; it has `error` when the session needs a new sign-in.
- `state` is `off`, `connecting`, `connected` or `error` (then `error` holds the reason).
- `relay_id`, `address` and `client_key` name this Core's relay identity, created on first use.

Errors: none.

Events: none.

### PUT /v1/relay

Auth: admin
Effect: Turns the relay on or off. Turning it on needs a signed-in account.

Request:

```json
{ "enabled": true }
```

Response: `200` with the body of `GET /v1/relay`.

Errors:

- `409` with `{"error":"…","code":"sign_in_required"}` when turning it on without an account.
- `400` invalid request.

Events: none.

### POST /v1/relay/account/start

Auth: admin
Effect: Asks the account center to email a sign-in code.

Request:

```json
{ "email": "owner@example.com" }
```

Response: `204`.

Errors: the account center's status and message; `502` with `code` `account_unreachable` when it cannot be reached.

Events: none.

### POST /v1/relay/account/verify

Auth: admin
Effect: Signs this Core in with the emailed code. Core keeps the account session in its home, never in `config.json`.

Request:

```json
{ "email": "owner@example.com", "code": "123456" }
```

Response: `200` with the body of `GET /v1/relay`.

Errors: `401` invalid code; others as for start.

Events: none.

### DELETE /v1/relay/account

Auth: admin
Effect: Signs out, forgets the session and turns the relay off.

Request: no body.

Response: `200` with the body of `GET /v1/relay`.

Errors: none.

Events: none.

### GET /v1/relay/enroll

Auth: any access key
Effect: Gives the calling device a new relay ticket (replacing its previous one) with the relay address and Core's Noise key, so a device paired before the relay was on can use it.

Request: no body.

Response: `200`

```json
{
  "relay": "https://relay.gyres.app/c/abcdefghijklmnopqrstuvwxyz",
  "ck": "base64url X25519 public key",
  "ticket": "base64url 32 bytes"
}
```

Errors:

- `409` with `{"error":"…","code":"relay_off"}` while the relay is off.
- `403` without an access key (hook tokens and peer keys are refused).

Events: none.
