# Users

A Core has a list of users, local to the host. A user has an id, a name and a creation time; no password and no role. Every access key belongs to one user, and a key sees only its user's conversations. The role (administrator or ordinary) stays on the key, so a user may hold administrator keys and ordinary ones. The first user of every Core has the id `owner` (named `Owner` until renamed); other users have 24 lowercase hex characters. Names are 1–80 characters after trimming and unique among the Core's users, compared without letter case.

These routes are for administrator keys only. Any administrator key may use them, whatever its user. There is no route that deletes a user, creates an access key outside [device pairing](pairing.md), or returns a key's secret or hash. Every user reads their own user id and name in `GET /v1/identity` ([system](system.md)) and renames themselves with `PATCH /v1/identity/user` below, which any key may use.

On this release a user other than the first may use their own conversations, search, events, approvals of their own conversations, and the host-wide settings every chat needs. Bots, Projects, schedules, Tasks, channels, MCP management and approval settings belong to the first user and answer `403` for them.

Every response from these routes sends `Cache-Control: no-store`. Coded errors carry `{"error": "<message>", "code": "<code>"}`.

### GET /v1/users

Auth: admin
Effect: Lists every user with their access keys.

Request: no body.

Response: `200`

```json
{
  "users": [
    {
      "id": "owner",
      "name": "Owner",
      "created_at": "2026-11-27T08:00:00Z",
      "first": true,
      "key_count": 1,
      "keys": [
        {
          "id": "0123456789abcdef01234567",
          "name": "Laptop",
          "role": "admin",
          "user_id": "owner",
          "created_at": "2026-11-27T08:00:00Z",
          "last_used_at": "2026-11-27T09:12:00Z",
          "revoked": false
        }
      ]
    }
  ]
}
```

- Users come first user first, then by creation time.
- `first` is true for the first user only.
- `keys` lists the user's keys, revoked ones included (`revoked` true), oldest first. `last_used_at` is omitted for a key never used. `handoff_id` is present on a revoked key whose work went to another key. A key carries no secret and no hash.
- `key_count` counts the user's keys that are not revoked.

Errors:

- `403` administrator key required

Events: none.

### POST /v1/users

Auth: admin
Effect: Adds a user with a new id. The new user has no keys yet; give them one with [device pairing](pairing.md) (`user_id`) or on the Core host.

Request:

```json
{ "name": "Ann" }
```

- `name` (string) is required. The body is one JSON object of at most 4096 bytes with only this field.

Response: `201`, the user in the shape of a `GET /v1/users` row (`keys` empty, `key_count` 0).

Errors:

- `400` invalid request; or `code` `user_name_invalid` when the name is empty or longer than 80 characters
- `409` `code` `user_name_taken`: another user already has this name (compared trimmed and without letter case)
- `403` administrator key required

Events: none.

### PATCH /v1/users/{id}

Auth: admin
Effect: Renames a user. The id never changes, and the first user can be renamed too.

Path: `id` user id.

Request:

```json
{ "name": "Ann Lee" }
```

Response: `200`

```json
{ "id": "0123456789abcdef01234567", "name": "Ann Lee", "created_at": "2026-11-27T08:00:00Z", "first": false }
```

Errors:

- `400` invalid request, or `code` `user_name_invalid`
- `404` `code` `user_not_found`
- `409` `code` `user_name_taken`. A user may keep their own name with other letter case.
- `403` administrator key required

Events: none.

### PATCH /v1/identity/user

Auth: any key
Effect: Renames the user the request's key belongs to, whatever the key's role. It takes no user id, so it never renames another user; administrators rename anyone with `PATCH /v1/users/{id}`. Names follow the same rules.

Request:

```json
{ "name": "Ann Lee" }
```

- `name` (string) is required. The body is one JSON object of at most 4096 bytes with only this field.

Response: `200`, as for `PATCH /v1/users/{id}`.

```json
{ "id": "0123456789abcdef01234567", "name": "Ann Lee", "created_at": "2026-11-27T08:00:00Z", "first": false }
```

Errors:

- `400` invalid request, or `code` `user_name_invalid`
- `404` `code` `user_not_found`: the key belongs to no user that can be read (a Core without users)
- `409` `code` `user_name_taken`. A user may keep their own name with other letter case.

Events: none.

### PUT /v1/access-keys/{id}/user

Auth: admin
Effect: Gives an active access key to another user. The secret, role and name stay the same. Requests of that key that are open when it moves (an event stream, for example) end within a second, and its next request acts as the new user: it sees the new user's conversations and none of the old ones.

Path: `id` access key id (24 lowercase hex characters, as in `GET /v1/users`).

Request:

```json
{ "user_id": "0123456789abcdef01234567" }
```

Response: `200`, the key in the shape of a `GET /v1/users` key.

Errors:

- `400` invalid request
- `404` `code` `user_not_found`, or `code` `key_not_found` (no such key, or it is revoked)
- `403` administrator key required

Events: none.
