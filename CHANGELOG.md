# Changelog

Breaking changes to the client interface (HTTP endpoints, request and response
fields, events, auth levels), newest first, with what a frontend must change.
Additions are not listed; they never break a client that ignores unknown
fields.

Every entry raises `api_version` (reported by `GET /v1/version`) by one.
A frontend built for `api_version` N works with a Core that reports N. When
Core reports a higher number, read the entries above N here.

Each entry:

    ## api_version N (Gyres X.Y.Z)

    - What changed (endpoint, field or event).
      Migration: what a client must do.

## api_version 4 (Gyres 0.97.6)

- `POST /v1/skills/preview` is gone; `POST /v1/skills/inspect` (admin)
  lists every skill bundle of a URL: a raw `SKILL.md`, a GitHub repository,
  tree, blob or raw URL, or a zip / tar.gz archive. `POST /v1/skills/install`
  now takes `bundles`, `expected_digests` (path → digest) and `replace`
  instead of `expected_digest`, writes whole bundles and returns
  `installed` (a list) instead of `skill`.
  Migration: inspect, let the owner choose bundles, then install them with
  the digests from the inspection.
- `POST /v1/skills/uninstall` removes the whole bundle when its history keeps
  every file, and returns `removed_bundle`.
  Migration: none needed; show the new field if useful.

## api_version 3 (Gyres 0.97.0)

- Run stream event `claude_activity` is renamed `agent_activity` and is now
  sent by Codex and Grok conversations too (their own subagents), with the
  same fields. A Codex or Grok task leaves out what that agent does not report
  (`tool_use_id`, and for Codex `subagent_type`).
  Migration: handle `agent_activity` as you handled `claude_activity`; keep
  accepting `claude_activity` for Cores before 0.97.0.

## api_version 2 (Gyres 0.89.0)

- `GET`/`PUT /v1/bot`: `channel_provider_id` and `channel_model` are gone,
  replaced by `bot_provider_id` and `bot_model`, the one model of every Bot
  chat (main, side and channel chats). The task model now falls back to it.
  Migration: read and write `bot_provider_id`/`bot_model`; Core fills them once
  from the main chat's saved model, else the old channel model.
- `GET`/`PUT /v1/sessions/{id}/settings` in a Bot chat: `settings` and
  `effective` are the Bot model, `channel_model` is gone and `bot` is new; a
  `PUT` there sets the Bot model for every Bot chat (or, with `target: "task"`,
  the task model).
  Migration: drop `channel_model`; treat a model switch in a Bot chat as a
  Bot-wide change.

## api_version 1 (the first Gyres release with `GET /v1/version`)

First published version of this spec. `GET /v1/version` is new; a Core that
answers it with `404` is older than this spec.
