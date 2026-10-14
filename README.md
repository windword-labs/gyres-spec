# Gyres spec

这是 Gyres Core 对外接口的公开说明：HTTP 接口、事件流和入门指南，供想给自己运行的 Gyres Core 做前端（网页、手机、命令行）的开发者使用，主要写给编码 Agent 读。全部为英文 Markdown，只描述最新版本。用法：把 `llms.txt`（索引）或 `llms-full.txt`（全部内容合成一个文件）交给你的 Agent，或者从下面的 Guides 开始读。

---

This repository is the contract for building your own frontend for **Gyres
Core**: the HTTP endpoints, the Server-Sent Events streams and short guides.
It is written for coding agents first and humans second. It describes the
latest Gyres release only; breaking changes are listed in
[CHANGELOG.md](CHANGELOG.md).

## Start here

- Agents: read [llms.txt](llms.txt) (index) or [llms-full.txt](llms-full.txt)
  (every file in one).
- Humans: the [guides](#guides), then the endpoint reference.

## Basics

- Core is an HTTP server the owner runs on their own machine (desktop app or
  standalone `gyresd`). Its address is whatever the owner gives you, for
  example `http://127.0.0.1:15100`, a LAN address, or an `https://` domain.
- Every request except a few marked `Auth: none` sends
  `Authorization: Bearer <access key>`. Keys come from the owner or from
  [device pairing](guides/pairing-and-keys.md). One key, the administrator key,
  may also manage Core (`Auth: admin`); every other key is an ordinary key.
  `GET /v1/identity` tells you which kind you hold.
- Requests and responses are JSON unless an endpoint says otherwise. Ignore
  fields you do not know; Core adds fields without notice.
- Live updates come from two SSE streams: the Core-wide `GET /v1/events` and a
  conversation's `GET /v1/sessions/{id}/events`. See [events.md](events.md).
- Check compatibility with `GET /v1/version`: it returns
  `{"version": "<Core version>", "api_version": <integer>}`. `api_version` goes
  up by one on every breaking change; compare it with the number your
  frontend was built for and read [CHANGELOG.md](CHANGELOG.md). Owners' Cores
  update themselves, your frontend does not.

## Guides

- [Pairing and access keys](guides/pairing-and-keys.md)
- [List conversations](guides/list-conversations.md)
- [Send a message and stream the reply](guides/send-and-stream.md)
- [Stop a run and reattach to it](guides/stop-and-reattach.md)
- [Tool approvals](guides/approvals.md)

## Reference

- [Errors](errors.md): the error body, coded errors, auth failures.
- [Events](events.md): every SSE event and its fields.
- HTTP endpoints, by area:
  - [External agents](http/agents.md)
  - [Tool approvals](http/approvals.md)
  - [Bot and chat channels](http/bot.md)
  - [Plugins, MCP, hooks, computer use and host screen](http/extensions.md)
  - [Files, attachments and search](http/files.md)
  - [Models and model settings](http/models.md)
  - [Device pairing](http/pairing.md)
  - [Projects and skills](http/projects.md)
  - [Runs: sending, streaming, stopping](http/runs.md)
  - [Scheduled tasks](http/schedules.md)
  - [Conversations](http/sessions.md)
  - [Storage, export and import](http/storage.md)
  - [System](http/system.md)

Every endpoint uses the same layout: `### METHOD /path`, then `Auth:`
(`none`, `any key` or `admin`), `Effect:`, `Request`, `Response`, `Errors` and
`Events`.

## What is not covered

Routes only the Core host itself uses (the desktop shell's own credential,
host-only HTTPS settings, update handoff, internal routes), the OAuth redirect
page and a developer test page are deliberately left out. Plugins may add
their own pages and tools; their contents are not part of this spec.

