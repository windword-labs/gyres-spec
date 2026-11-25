# Projects and skills

Projects are the connected Core's named work containers: a display name, zero or more directories on the machine running Core, a library directory Core keeps for that project, instructions, and a memory file. Skills are `SKILL.md` bundles discovered on that same machine. Installing or removing a bundle changes what a new conversation, or a conversation whose context is built again, can list. A conversation that already has a context keeps the catalog from when that context was built.

JSON responses use `Content-Type: application/json`. An error body is `{"error":"<message>","failure":{"code":"core_request_failed","message":"<message>","http_status":<status>,"details":"<message>"}}`.

A project object:

```json
{
  "id": "3f2a1c0e-6b7d-4e18-9c0a-1b2d3e4f5a6b",
  "name": "Demo",
  "primary": "/work/demo",
  "roots": ["/work/demo", "/work/notes"],
  "library": "/data/projects/3f2a1c0e-6b7d-4e18-9c0a-1b2d3e4f5a6b/library"
}
```

- `id` is assigned by Core. A new project receives a lowercase UUID. Creating a project again after it was removed can return the previous id; see `POST /v1/projects`.
- `name` is the display name, trimmed. Comparison for uniqueness ignores letter case.
- `primary` is the project's main directory, an absolute path on the machine running Core. It is always the first element of `roots`.
- `roots` is the directories the project keeps, primary first, then the others in the order they were saved. Duplicates of the same cleaned path appear once. The array is never null.
- `library` is the absolute path of the library directory Core derives for this id. It is present when Core has a data directory for projects. It is not one of `roots` unless the project has no folders. The request field is ignored.
- `folderless` is `true` when the project has no folders of its own. It is omitted when false. For that project, `primary` and `roots` are both the library path, and the library is created when the project is saved:

```json
{
  "id": "3f2a1c0e-6b7d-4e18-9c0a-1b2d3e4f5a6b",
  "name": "Notes",
  "primary": "/data/projects/3f2a1c0e-6b7d-4e18-9c0a-1b2d3e4f5a6b/library",
  "roots": ["/data/projects/3f2a1c0e-6b7d-4e18-9c0a-1b2d3e4f5a6b/library"],
  "folderless": true,
  "library": "/data/projects/3f2a1c0e-6b7d-4e18-9c0a-1b2d3e4f5a6b/library"
}
```

`GET /v1/projects` adds `status` on each project. `POST` and `PUT` return the project without `status`.

```json
{
  "available": true,
  "directories": [
    { "path": "/work/demo", "available": true },
    {
      "path": "/work/notes",
      "available": false,
      "reason": "invalid project cwd \"/work/notes\": no such file or directory"
    }
  ]
}
```

- `available` is true when the primary directory can be used. An additional directory that cannot be opened does not by itself make the project unavailable.
- `directories` lists each distinct root. `path` is that directory. `available` is false when it cannot be opened and listed. `reason` is then present and starts with `invalid project cwd "<path>"`.
- A library path that is not on disk yet counts as available. Core creates it on use. A library path that exists and is not a directory is unavailable.

A skill object:

```json
{
  "name": "brand",
  "description": "Apply the brand voice",
  "path": "/work/demo/.agents/skills/brand/SKILL.md",
  "source": "/work/demo",
  "digest": "0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef"
}
```

- `name` comes from the file's frontmatter: lowercase letters, digits, and single hyphens, at most 64 characters.
- `description` is the frontmatter description.
- `path` is the absolute path of that `SKILL.md` on the machine running Core.
- `source` is `global` for a skill installed for every conversation, the project directory the bundle was found under for a project skill, `builtin` for a skill shipped with Core, or `plugin:<name>` for a skill from an enabled plugin.
- `digest` is 64 lowercase hex characters, the SHA-256 of the file's bytes.

`GET /v1/skills` returns `{"skills":[...],"warnings":[...]}`. Both arrays are present. `warnings` lists discovery problems; a bad bundle is skipped and the others are still returned. Project and global skills come first, ordered by name then path. Plugin skills follow, grouped by plugin name. Built-ins follow those. A project skill hides a global skill of the same name. A name already present hides a plugin skill and a built-in of that name. The same name under two project directories is two entries.

Built-in names are `widgets`, `hooks`, and `computer-use`; `web-search` comes from the bundled `web-search` plugin (source `plugin:web-search`) while that plugin is enabled. `computer-use` is left out when computer use is not available. A workspace or conversation list includes built-ins when Core has started browser support. The unscoped management list (`manage_scope=all` with no `session_id`, `project_id`, or `cwd`) includes built-ins whenever skills can be read from Core's home.

A skill whose frontmatter `requires` contains `library` is listed for a project that has a library, and in that unscoped management list. It is left out of every other list, including a conversation with no project.

### GET /v1/projects

Auth: any key
Effect: Lists the saved projects and whether their directories can be used right now, plus the operating system of the machine running Core.

Request: no body.

Response: `200`

```json
{
  "projects": [
    {
      "id": "3f2a1c0e-6b7d-4e18-9c0a-1b2d3e4f5a6b",
      "name": "Demo",
      "primary": "/work/demo",
      "roots": ["/work/demo", "/work/notes"],
      "library": "/data/projects/3f2a1c0e-6b7d-4e18-9c0a-1b2d3e4f5a6b/library",
      "status": {
        "available": true,
        "directories": [
          { "path": "/work/demo", "available": true, "git": { "branch": "main", "dirty": 3, "ahead": 2, "behind": 0 } },
          { "path": "/work/notes", "available": true }
        ]
      }
    }
  ],
  "host_os": "darwin"
}
```

- Each available folder in `status.directories` (not the library) has `git` when it is inside a git work tree: `branch` (`""` on a detached HEAD), `dirty` (changed and untracked paths, `0` when clean), `ahead` and `behind` (commits against the branch's upstream as last fetched; `0` without an upstream). Core reads it with one local `git status`, never fetches, waits at most 2 seconds per folder and reuses an answer for 10 seconds, so `ahead`/`behind` lag until something fetches. `git` is absent when the folder is not in a repository, git is not installed, or git did not answer in time. Only this route fills it.

- `projects` is the saved projects in the order they were saved, each with `status`. An empty Core returns `[]`.
- Each project may also carry its Bot fields, each present only when set: `lead_bot_id` (the project's lead Bot, a specialist bound to the project, see the Bot list in Bot), `coordinator_session_id` (the project's current Coordinator chat: its lead Bot's main chat, see `POST /v1/projects/{project}/coordinator`), `bot_hidden` (true: the owner hid the project from the main project list; a client shows it in a folded hidden group with its conversations; nothing else changes) and `coordinator_provider_id`, `coordinator_model` and `coordinator_reasoning_effort` (the model the project's Coordinator chats run on: its lead Bot's Bot model; absent: the primary's Bot model).
- `host_os` is the operating system of the machine running Core, such as `darwin`, `linux`, or `windows`.

Errors:

- `500` the project list could not be read

Events: none.

### POST /v1/projects

Auth: any key
Effect: Saves a new project. Directories must already exist; this route does not create them. A project without folders gets a library directory instead, and that directory is created now. The response id may be a previous project's id: the latest removal of the same primary path after `.`, `..`, and extra separators are removed, or, for a project without folders, the latest removal of the same name including letter case. The directory match does not follow symlinks. If no removal was recorded for a project that has folders, a conversation still bound to that primary path can supply the old id. That id is reused once. Anything still stored for the id (conversations bound to it, instructions, library files, project memory) is there again with the recreated project.

Request:

```json
{ "name": "Demo", "primary": "/work/demo", "roots": ["/work/notes"] }
```

```json
{ "name": "Notes", "folderless": true }
```

- The body is one JSON object of at most 65536 bytes. Unknown fields are ignored. `id` and `library` are ignored.
- `name` (string) is required after trimming and at most 200 bytes. Names already saved are unique ignoring letter case.
- `folderless` (boolean) true creates a project with no folders. `primary` and every entry of `roots` must then be empty or absent. Core sets `primary` and `roots` to the new library path in the response.
- Without `folderless`, `primary` (string) is the main directory. `roots` (array of strings) lists additional directories. Each path is trimmed, then `.`, `..`, and extra separators are removed. It must be an absolute directory on the machine running Core and must be listable. The same path is kept once. At most 32 directories, including the primary.
- A project may be saved again later under a name it already has even when another project also has that name. Any other save that matches another project's name is refused.
- Optional Bot fields: `bot_hidden` (boolean) and the Coordinator model `coordinator_provider_id` with `coordinator_model` (strings, sent together; both `""` clear it so the Coordinator follows the Bot model) and `coordinator_reasoning_effort` (string, needs a model). An external agent cannot be the Coordinator model. With a lead Bot it sets the lead's Bot model. `lead_bot_id` (string): a specialist bound to this project becomes its lead, `""` leaves it without one (its unclaimed reports go to the primary); refused on create. `coordinator_session_id` is set by Core only and ignored here.

Response: `200`

The saved project object. `folderless` is present only when true. `library` is present when Core has a data directory for projects.

Errors:

- `400` invalid project (the body is not JSON or is larger than 65536 bytes)
- `400` project name is required (maximum 200 characters)
- `400` a project without folders cannot also have directories
- `400` the project library is unavailable, so a project cannot be created without folders
- `400` use an absolute directory path on the Core host
- `400` invalid project cwd "<path>": the directory is missing, is not a directory, or cannot be listed
- `400` at most 32 project directories
- `400` coordinator_provider_id and coordinator_model must be set together
- `400` coordinator_reasoning_effort needs a model
- `400` `lead_bot_id` is not a specialist bound to this project, or sent on create
- `400` Bot conversations run on a configured provider, not an external agent
- `400` the Coordinator model is not an enabled provider's active model (the message says why)
- `400` the project could not be saved (the message says why)
- `409` another project already has this name

Events: `bot.changed`.

### PUT /v1/projects/{project}

Auth: any key
Effect: Replaces the registration of an existing project with the same checks as create. Directories already on disk are left where they are. Sending `folderless: true` drops the saved directories from the registration. Sending directories without `folderless` turns a project without folders into one that has those directories. The id does not change.

Path: `project` project id.

Request:

```json
{ "name": "Demo", "primary": "/work/demo", "roots": ["/work/notes"] }
```

- Same object and limits as `POST /v1/projects`. The id in the body is ignored; the path id is the project being saved.
- Omitted `folderless` means false. A project without folders is returned with `primary` and `roots` set to the library. Sending that object back is rejected. To save it again without folders, send `folderless: true`, `primary` `""`, and `roots` `[]`.
- The Bot fields are optional as for create: an omitted one keeps its saved value, so a client that edits only the name and folders never changes them. The Coordinator chat is kept.
- A body with only Bot fields (no `name`, `primary`, `roots` or `folderless`), such as `{"bot_hidden": true}`, changes those fields alone and does not check the folders again, so it works while a folder is missing.

Response: `200`

The saved project object, as for `POST /v1/projects`.

Errors:

- `400` invalid project
- `400` the same directory, name and Bot field failures as create
- `400` project not found
- `404` project not found (a body with only Bot fields)
- `409` another project already has this name

Events: `bot.changed`; `session.settings` for the Bot's chats when the Coordinator model changed.

### DELETE /v1/projects/{project}

Auth: any key
Effect: Forgets the registration. Directories, library files, instructions, project memory, and conversation bindings are left in place. The project's Coordinator chats stay Bot chats (listed in `GET /v1/bot`'s `chats`), and its Bots stay bound to its id, the lead included, no longer leading. A later create with the same primary path, or the same name for a project without folders, can take this id again, and its Coordinator chat with it (its Bot leads again).

Path: `project` project id.

Request: no body.

Response: `200`

```json
{ "ok": true }
```

Errors:

- `404` project not found
- `500` the registration could not be updated

Events: `bot.changed`.

### POST /v1/projects/{project}/coordinator

Auth: any key
Effect: Returns the project's Coordinator chat, creating it on first use. A Coordinator chat is the main chat of the project's lead Bot (see the Bot list in Bot), bound to the project's workspace: it runs on the lead's Bot model (never an external agent), gets the lead's prompt and the Bot tools together with the project's instructions, memory, library and skills, and is not listed by `GET /v1/sessions`. With a lead, its main chat is returned (a fresh one when it was deleted or archived). Without one, a specialist is created, bound to the project and named its lead: named after the project, with the primary's description and a copy of its memory file and the project's Coordinator model; its main chat is the saved chat while its conversation exists and is bound to the project, otherwise a Coordinator chat this project had before, otherwise a new one, saved as the project's `coordinator_session_id`. Calling it again returns the same chat. `POST /v1/bot/chats/{id}/new` replaces it with a fresh one.

Path: `project` project id.

Request: no body.

Response: `200`

```json
{ "session_id": "2026-11-16T04-00-00_5b0c2a8e-4f4e-4d55-9a37-0d0f0c7b3a11", "project_id": "3f2a1c0e-6b7d-4e18-9c0a-1b2d3e4f5a6b" }
```

Errors:

- `404` project not found
- `409` the project's primary directory is missing or cannot be used (the message says which)
- `409` the project's Coordinator chat changed meanwhile; try again
- `500` the Bot list cannot be used; nothing is created
- `500` the chat could not be created or saved
- `503` the Bot is not available

Events: `bot.changed`.

### GET /v1/projects/{project}/activity

Auth: any key
Effect: Returns what works and listens for the project, for a Coordinator view: the Bot's tasks and workers in the project, standalone Tasks of the project while an attempt runs, the open agenda items the project's chats created and the active scheduled plans of the project. Read only. Refetch it on `bot.changed`, `schedules.changed`, `approvals.changed` and run ends.

Path: `project` project id.

Request: no body.

Response: `200`

```json
{
  "project_id": "3f2a1c0e-6b7d-4e18-9c0a-1b2d3e4f5a6b",
  "workers": [
    {
      "id": "bot_8d1f0a2b3c4d5e6f708192a3b4c5d6e7",
      "session_id": "2026-11-16T05-00-00_bot_8d1f0a2b3c4d5e6f708192a3b4c5d6e7",
      "title": "Fix the parser",
      "kind": "bot_task",
      "executor": "codex",
      "provider_id": "codex",
      "model": "gpt-5.5",
      "visible": false,
      "status": "running",
      "waiting_approval": true,
      "agenda_id": "ag_0123456789abcdef",
      "created": "2026-11-16T05:00:00Z",
      "updated": "2026-11-16T05:02:00Z"
    }
  ],
  "listening": [
    { "id": "ag_0123456789abcdef", "kind": "agenda", "title": "Keep CI green", "status": "active", "next": "2026-11-16T06:00:00Z" },
    { "id": "job_1", "kind": "schedule", "title": "Nightly check", "status": "active", "next": "2026-11-17T02:00:00Z" }
  ],
  "working_count": 1,
  "listening_count": 2
}
```

- `workers` lists the unsettled ones and those waiting on an approval first, then up to 30 settled ones, each group by `updated`, newest first. `kind` is `bot_task` (a task a Bot chat started: a hidden worker, `visible` false, or a visible task, `visible` true) or `task` (a standalone Task's running attempt; `id` is the Task's id and `session_id` the attempt's conversation). Conversations the Bot only sent a message to are not listed. `executor` is `gyres` for Gyres' own models, otherwise the external agent (`codex`, `grok`, `pi`, `claude`, `cursor`). `status` is a Bot task status (`queued`, `running`, `paused`, `completed`, `failed`, `stopped`, `cancelled`); a standalone Task's attempt is `running`. `waiting_approval` is true while an approval card of the conversation (or of its child agents) waits for a person. `agenda_id` is present when an open agenda item targets the worker.
- `listening`: agenda items (`kind` `agenda`) with `status` `active` or `waiting_owner` and `next` their next check; scheduled plans (`kind` `schedule`) that are active, for the project directly, for a conversation bound to the project, or advancing a Task of the project, with `next` their next run.
- `working_count` is the number of workers listed first; `listening_count` the length of `listening`.

Errors:

- `404` project not found
- `500` the registration or the tasks could not be read

Events: none.

### GET /v1/projects/summary

Auth: any key
Effect: Returns every saved project at once with what waits on the owner there, for a home screen that sorts projects by it: workers waiting on an approval, agenda items waiting for the owner, linked work that failed, and list items waiting to be merged, plus the working count and the last activity. Read only; derived from the same data as `GET /v1/projects/{project}/activity` and `GET /v1/projects/{project}/list`. Refetch it on `bot.changed`, `approvals.changed`, `project_list.changed` and run ends.

Request: no body.

Response: `200`

```json
{
  "projects": [
    {
      "id": "3f2a1c0e-6b7d-4e18-9c0a-1b2d3e4f5a6b",
      "name": "Demo",
      "available": true,
      "coordinator_session_id": "2026-11-16T04-00-00_5b0c2a8e-4f4e-4d55-9a37-0d0f0c7b3a11",
      "running": false,
      "run_state": "idle",
      "attention_count": 2,
      "counts": { "approval": 1, "owner": 0, "failed": 0, "merge": 1 },
      "attention": [
        {
          "kind": "approval",
          "title": "Fix the parser",
          "session_id": "2026-11-16T05-00-00_bot_8d1f0a2b3c4d5e6f708192a3b4c5d6e7",
          "task_id": "bot_8d1f0a2b3c4d5e6f708192a3b4c5d6e7",
          "executor": "codex",
          "approval_ids": ["apr_0123456789abcdef"],
          "item": { "id": "li_0123456789abcdef", "key": "DEM-3", "title": "Fix the parser" },
          "at": "2026-11-16T05:02:00Z"
        },
        {
          "kind": "merge",
          "title": "Ship the right panel",
          "item": { "id": "li_fedcba9876543210", "key": "DEM-1", "title": "Ship the right panel" },
          "prs": [{ "url": "https://github.com/example/repo/pull/12", "number": 12, "state": "open" }],
          "at": "2026-11-16T05:10:00Z"
        }
      ],
      "working_count": 1,
      "last_activity": "2026-11-16T05:10:00Z"
    }
  ]
}
```

- `projects` lists the projects that have something waiting first (`attention_count` above 0), then the others, each group by `last_activity`, newest first. `id`, `name`, `folderless`, `available`, `bot_hidden`, `coordinator_session_id`, `running` and `run_state` are as in the `projects` of `GET /v1/bot`; hidden projects are included with `bot_hidden`.
- `attention_count` is the number of entries; `counts` splits it by kind, every kind present. `attention` lists them, at most 20 (the counts cover all of them), approvals first, then `owner`, `failed` and `merge`, each kind by `at`, newest first:
  - `approval`: a worker of the project (as in `GET /v1/projects/{project}/activity`) with `waiting_approval`, once per worker, with `task_id`, `executor` and the waiting `approval_ids` (see `GET /v1/approvals`); `at` is when the oldest of them was asked. A project chat (its Coordinator, a parallel or side chat) waiting on an approval is listed the same way without `task_id`, titled with the project's name. Each approval is listed once.
  - `owner`: an agenda item of the project that waits for the owner (`agenda_id`; `session_id` is its first conversation target, when it has one); `at` is when it started waiting.
  - `failed`: a list item whose linked worker failed after the item last got a status (the item is `in_progress` for it in `GET /v1/projects/{project}/list`), the newest such worker; giving the item a status again clears it. A failed worker that no list item links is not listed.
  - `merge`: a list item that is `for_review` (`review_reason` `pr`, waiting to be merged) for its own open PRs, listed in `prs`; a parent that only takes the state from its child is not listed again, and an item given `for_review` by hand without an open PR is not listed. `session_id`, `task_id` and `executor` name its most recent linked worker, when there is one.
  - `title` is the worker's or the item's title. `item` (`id`, `key`, `title`) is the list item the entry belongs to or that links the worker. `session_id` is the conversation to open.
- `working_count` is the project's `working_count` in `GET /v1/projects/{project}/activity` (unsettled workers and those waiting on an approval).
- `last_activity` is the newest of the project's chats' last writes, its workers' `updated` and its list items' `updated_at`; absent when the project has none.

Errors:

- `500` the registration, a list or the tasks could not be read

Events: none.

### GET /v1/projects/{project}/list

Auth: any key
Effect: Returns the project's list: short items for the project's work, each with a key such as `GYR-12`, a stored status, links to the work, and an effective status Core derives from that work on every read. Archived items are left out unless `archived=1`. Read only. Refetch it on `project_list.changed`, and also on `bot.changed`, `approvals.changed` and run ends, because the effective status moves with the linked work.

Path: `project` project id.

Query: `archived` (`1` includes archived items).

Request: no body.

Response: `200`

```json
{
  "project_id": "3f2a1c0e-6b7d-4e18-9c0a-1b2d3e4f5a6b",
  "prefix": "GYR",
  "library": "/data/projects/3f2a1c0e-6b7d-4e18-9c0a-1b2d3e4f5a6b/library",
  "items": [
    {
      "id": "li_0123456789abcdef",
      "key": "GYR-1",
      "title": "Ship the right panel",
      "note": "List and Library only",
      "stored_status": "queued",
      "effective_status": "for_review",
      "review_reason": "pr",
      "links": {
        "task_ids": ["bot_8d1f0a2b3c4d5e6f708192a3b4c5d6e7"],
        "session_ids": [],
        "library_files": ["reports/out.md"],
        "prs": [{ "url": "https://github.com/example/repo/pull/12", "number": 12, "state": "open" }]
      },
      "workers": [],
      "children_done": 1,
      "children_total": 2,
      "status_at": "2026-11-16T05:00:00Z",
      "linked_at": "2026-11-16T05:10:00Z",
      "created_at": "2026-11-16T05:00:00Z",
      "updated_at": "2026-11-16T05:10:00Z"
    }
  ]
}
```

- `prefix` is the prefix new keys get: 2 to 5 capital letters, derived from the project's name until it is set (the initials of its Latin words, or the first three letters of one word, else `PRJ`). Keys are never reused.
- `library` is the project's library path. `library_files` are relative to it, with `/`.
- `parent_id`, present on a child, is its parent's `id`. Items nest one level only.
- `stored_status` is the status last given by hand or by the Coordinator: `queued`, `in_progress`, `for_review` (waiting to be merged), `done` or `archived`. `status_at` is when a status was last given, also when the same one is given again; `linked_at` when a link was last added. `completed_at` is present while the stored status is `done` or `archived` after `done`.
- `effective_status` is derived from the linked work, with no review step, in this order: `archived` and `done` as stored; `done` (`done_reason` `merged`) when every linked PR is `merged` and no linked work runs; `in_progress` while linked work is queued, running or paused, waits on an approval (a worker's `waiting_approval`, or its agenda item waits for the owner), or failed (the worker's `status` says so); `for_review` (`review_reason` `pr`, waiting to be merged) when a linked PR is `open`; `in_progress` (`review_reason` `pr_closed`) when a linked PR is `closed` without merging and none is `open` (also when another one is `merged`); `done` (`done_reason` `completed`) when linked work completed; otherwise the stored status. Finished or failed work and PRs (merged ones included) count only when they are newer than `status_at` or were linked since then (links sent again unchanged do not count), so an item whose status was set by hand, for example sent back to `queued`, stays there until something new arrives. Library files do not move the status.
- A parent with children: `children_total` and `children_done` count its children that are not archived. It is `done` when every child is and a child got its status, a link or finished work after the parent's `status_at` (a parent sent back after its children finished keeps its stored status), and otherwise, unless its stored status is `done` or `archived`, takes the most pressing of its own effective status and its children's (`for_review`, then `in_progress`, then `queued`, then `done`; with the child's `review_reason`).
- `workers` has one row per linked task Core still knows, shaped as in `GET /v1/projects/{project}/activity`. A standalone Task between attempts has no `session_id` and the status of its last outcome.
- PR states are what was last sent, or what Core found on GitHub: on a read, PRs stored as `open` are looked up again in the background (a short shared cache; never PRs stored `merged` or `closed`) with the host's own GitHub access (a logged-in `gh` CLI, else `GITHUB_TOKEN` / `GH_TOKEN`, else unauthenticated for public repositories; `github.com` or a GitHub Enterprise host `gh` is logged into). A merge or close found is published as `project_list.changed` and written back on the next read, as if the URL had been linked again with that state. The read never waits for GitHub; a failed lookup keeps the stored state.

Errors:

- `404` project not found
- `500` the registration, the list or the tasks could not be read

Events: none.

### POST /v1/projects/{project}/list

Auth: any key
Effect: Adds an item with the next key, `queued` unless `stored_status` says otherwise.

Path: `project` project id.

Request:

```json
{
  "title": "Ship the right panel",
  "note": "List and Library only",
  "parent_id": "GYR-1",
  "stored_status": "queued",
  "links": { "task_ids": [], "session_ids": [], "library_files": ["plans/panel.md"], "prs": [] }
}
```

- The body is one JSON object of at most 256 KiB. `title` is required, at most 200 characters; runs of whitespace become one space. `note` is optional, at most 2000 characters.
- `parent_id` is the id or key of a top-level item.
- `links` (and `add_links`, which means the same here) name the work: `task_ids` of Tasks Core knows (a Bot task's id or a standalone Task's), `session_ids` of existing conversations, `library_files` relative to the library or absolute inside it, and `prs` with an `http(s)` `url`, an optional `number`, `repo` and `state` (`open`, `merged`, `closed`). At most 20 of each.

Response: `201` the item as `GET` lists it.

Errors:

- `400` the JSON could not be read, the title is missing or too long, the note is too long, the status is unknown, the parent is missing or is itself a child, a library path is outside the library, a Task or conversation is unknown, a PR is malformed, too many links, or the list holds 500 items and none is archived
- `404` project not found
- `500` the list could not be saved

Events: `project_list.changed`.

### PATCH /v1/projects/{project}/list

Auth: any key
Effect: Sets the prefix of keys given from now on. Existing keys stay as they are.

Path: `project` project id.

Request:

```json
{ "prefix": "GYR" }
```

- `prefix` is 2 to 5 capital letters `A`-`Z`.

Response: `200` the list as `GET` returns it.

Errors:

- `400` the prefix is not 2 to 5 capital letters
- `404` project not found
- `500` the list could not be saved

Events: `project_list.changed`.

### PATCH /v1/projects/{project}/list/{item}

Auth: any key
Effect: Changes one item. Fields left out are kept.

Path: `project` project id. `item` the item's id or key (any letter case).

Request:

```json
{
  "title": "Ship the right panel",
  "stored_status": "done",
  "add_links": { "task_ids": ["bot_8d1f0a2b3c4d5e6f708192a3b4c5d6e7"] }
}
```

- `title`, `note`, `parent_id` (`""` makes it top-level) and `stored_status` as in `POST`. An item that has children cannot get a parent. Giving a status always sets `status_at`, also when it is the same status. Archiving an item makes its children that are not archived top-level items.
- `add_links` appends what is new; a PR whose `url` is linked already takes the sent `number`, `repo` and `state`. `links` replaces all links.

Response: `200` the item as `GET` lists it, archived or not.

Errors:

- `400` as for `POST`
- `404` project or item not found
- `500` the list could not be saved

Events: `project_list.changed`.

### DELETE /v1/projects/{project}/list/{item}

Auth: any key
Effect: Removes one item. Its children stay, as top-level items. Its key is not reused.

Path: `project` project id. `item` the item's id or key.

Request: no body.

Response: `204` no body.

Errors:

- `404` project or item not found
- `500` the list could not be saved

Events: `project_list.changed`.

### POST /v1/projects/{project}/reveal

Auth: any key
Effect: Opens the project's primary directory in the file manager of the machine running Core. For a project without folders that directory is the library. Nothing is deleted.

Path: `project` project id.

Request: no body.

Response: `200`

```json
{ "ok": true }
```

Errors:

- `404` project not found
- `500` the project list could not be read, the primary directory cannot be opened, or the file manager could not be started (`could not open the file manager on the Core host: ...`)

Events: none.

### GET /v1/projects/{project}/instructions

Auth: any key
Effect: Returns the project's instructions. A missing file is empty text. The response sends `Cache-Control: no-store`.

Path: `project` project id.

Request: no body.

Response: `200`

```json
{
  "content": "",
  "version": "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855",
  "max_chars": 16000
}
```

- `content` is the instruction text. Line endings are `\n`. Empty means there are no instructions.
- `version` is 64 lowercase hex characters, the SHA-256 of `content`. The empty text uses the SHA-256 of no bytes, including when the file is absent.
- `max_chars` is `16000`.
- A file edited outside this API is read up to 64000 bytes and then cut to the first 16000 characters. `version` matches that returned text.

Errors:

- `404` project not found
- `500` the project list or the instructions could not be read

Events: none.

### PUT /v1/projects/{project}/instructions

Auth: admin
Effect: Replaces the project's instructions when `expected_version` is still current. The text is what a conversation in this project receives the next time its context is built. A context already built keeps the previous text. The response sends `Cache-Control: no-store`.

Path: `project` project id.

Request:

```json
{
  "content": "Always answer in French.",
  "expected_version": "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855"
}
```

- The body is one JSON object of at most 256 KiB.
- `content` (string) is the new text. `\r\n` is stored as `\n`. Empty, absent, or only whitespace clears the instructions and removes the file. The text cannot contain a NUL and is at most 16000 characters after that newline change.
- `expected_version` (string) is required. It must be the `version` from the last read or the last successful save.

Response: `200`

The same object as `GET /v1/projects/{project}/instructions`, with the stored content and its new version.

Errors:

- `400` content and expected_version are required (JSON could not be read, the body is larger than 256 KiB, or `expected_version` is missing)
- `400` instructions cannot contain NUL
- `400` instructions are limited to 16000 characters
- `400` project instructions unavailable
- `400` the instructions could not be saved (the message says why)
- `404` project not found
- `409` instructions changed; refresh before saving
- `500` the project list could not be read

Events: none.

### POST /v1/projects/{project}/library

Auth: admin
Effect: Creates the project's library directory if it is not already there, and returns its path. The response sends `Cache-Control: no-store`.

Path: `project` project id.

Request: no body.

Response: `200`

```json
{ "path": "/data/projects/3f2a1c0e-6b7d-4e18-9c0a-1b2d3e4f5a6b/library" }
```

- `path` is the absolute library path. After `200` that path is a directory.

Errors:

- `404` project not found
- `500` project library unavailable, or the directory could not be created (the message says why)

Events: none.

### GET /v1/projects/{project}/library

Auth: any key
Effect: Lists the files in the project's library, newest first, creating the library directory when it is missing. By default names that start with `.` (at any depth) and the top-level `internal` and `tmp` folders are left out. Symbolic links are not followed or listed. The response sends `Cache-Control: no-store`.

Path: `project` project id.

Query: `all` (`1` lists everything).

Request: no body.

Response: `200`

```json
{
  "path": "/data/projects/3f2a1c0e-6b7d-4e18-9c0a-1b2d3e4f5a6b/library",
  "files": [
    { "path": "/data/projects/3f2a1c0e-6b7d-4e18-9c0a-1b2d3e4f5a6b/library/reports/weekly.md", "rel": "reports/weekly.md", "size": 2048, "modified": "2026-11-16T05:00:00Z" }
  ],
  "truncated": false
}
```

- `rel` is relative to the library, with `/`. `modified` is RFC 3339 in UTC. At most 2000 files are listed; `truncated` is then `true`.

Errors:

- `404` project not found
- `500` the library cannot be created or read

Events: none.

### GET /v1/projects/{project}/memory

Auth: admin
Effect: Returns this project's memory text and the version a save must send back. The response sends `Cache-Control: no-store`.

Path: `project` project id.

Request: no body.

Response: `200`

```json
{
  "content": "Uses pnpm.",
  "version": "0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef",
  "max_chars": 2200,
  "enabled": true
}
```

- `content` is the whole project memory file. A missing file is `""`. The text is not trimmed.
- `version` is 64 lowercase hex characters. It covers this project's text together with the global memory texts, so a change to either makes a later save stale.
- `max_chars` is `2200`.
- `enabled` is whether memory is turned on. The file is still returned when it is false.

Errors:

- `400` invalid memory project ID
- `404` project not found
- `500` the project list or the memory could not be read
- `503` memory is unavailable

Events: none.

### PUT /v1/projects/{project}/memory

Auth: admin
Effect: Replaces this project's memory file when `version` is still current, then returns the same view as the read. The response sends `Cache-Control: no-store`.

Path: `project` project id.

Request:

```json
{
  "content": "Uses pnpm.\n\n---\n\nReports live in reports/weekly.",
  "version": "0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef"
}
```

- The body is one JSON object of at most 65536 bytes.
- `content` (string) is the whole file. Empty is allowed. At most 2200 characters. The text is stored as sent.
- Pieces of the previous text split on a line that is `---` with a blank line on each side, and no longer present in `content`, stay deleted: a later background review does not add that exact text back.
- `version` (string) is required and must be non-empty. Use the `version` from the last read or the last successful save of this project.

Response: `200`

The same object as `GET /v1/projects/{project}/memory`.

Errors:

- `400` content and version are required (JSON could not be read, the body is larger than 65536 bytes, or `version` is missing or empty)
- `400` invalid memory project ID
- `404` project not found
- `409` memory changed; refresh before saving
- `409` maximum 2200 characters
- `409` the memory could not be saved (the message says why)
- `500` the project list could not be read, or the memory was saved and the following read failed
- `503` memory is unavailable

Events: none.

### GET /v1/skills

Auth: any key
Effect: Lists skills for a conversation, a project, a directory, or the whole Core. A conversation that already has a context for its current workspace (same workspace kind, working directory, directories, and project) receives that captured catalog. Otherwise the response is a live reading and does not create a context. Reading skills for a project creates its library directory when it is missing.

Query: `session_id`, `project_id`, `cwd`, `manage_scope`, `q`. When more than one of `session_id`, `project_id`, and `cwd` is set, the first of those three in that order is the one that is used.

- `session_id` (string) is a conversation id. The list is that conversation's catalog, as above. A conversation with no workspace lists global skills (plus plugin and built-in skills under the rules above). A scratch conversation does not list project skills. The conversation's own directory is not scanned.
- `project_id` (string) is a project id. The list is a live reading for that project's directories. The library directory is scanned only when the project has no folders; otherwise skills are read from the project's own directories, skipping an additional directory that is missing.
- `cwd` (string) is an absolute directory on the machine running Core, used as one project directory with no library. Global skills are included. Skills that require a library are not.
- With none of those three, the list is the global skills, plus plugin and built-in skills, and skills that require a library are left out. `manage_scope=all` together with none of those three is the management list: a fresh reading of every available project's primary directory, plus global, plugin, and built-in skills, including skills that require a library and skills a same-named skill of higher precedence (project, then global, then plugin, then built-in) hides from a conversation's catalog. A project whose primary directory is unavailable is skipped.
- `manage_scope` (string) filters the list when it is set. `all` keeps every source. `global` keeps `source` `global`. `builtin` keeps `source` `builtin`. `project` keeps a source that is not `global`, not `builtin`, and does not start with `plugin:`. Any other value leaves `skills` empty. `warnings` is unchanged.
- `q` (string) is a case-insensitive substring of `name` or `description`. It applies only when `manage_scope` is set. An empty `q` does not narrow the list.
- When `manage_scope` is set, each skill also carries `requires` (the conditions from its frontmatter, such as `["library"]`; omitted when none), `versions` (how many versions its history keeps; omitted when none or for plugin and built-in skills) and `learned` (only for a global or project bundle the Agent created: `created_by` `chat` or `review`, `session_id` of the conversation it was created in when recorded, and `reviewed`). `reviewed` is `true` once the owner marked it with `POST /v1/skills/review`, or when it was created more than 14 days ago. The response also carries `warning_details`: the `warnings` as `{path, reason}` objects (`path` is the file or directory, or `builtin <name>`).

Request: no body.

Response: `200`

```json
{
  "skills": [
    {
      "name": "brand",
      "description": "Apply the brand voice",
      "path": "/work/demo/.agents/skills/brand/SKILL.md",
      "source": "/work/demo",
      "digest": "0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef"
    }
  ],
  "warnings": []
}
```

With `manage_scope`:

```json
{
  "skills": [
    {
      "name": "release-notes",
      "description": "Write release notes",
      "path": "/home/me/.gyres/skills/release-notes/SKILL.md",
      "source": "global",
      "digest": "0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef",
      "versions": 2,
      "learned": { "created_by": "chat", "session_id": "s_123", "reviewed": false }
    }
  ],
  "warnings": ["/home/me/.gyres/skills/broken/SKILL.md: missing YAML frontmatter"],
  "warning_details": [{ "path": "/home/me/.gyres/skills/broken/SKILL.md", "reason": "missing YAML frontmatter" }]
}
```

Errors:

- `400` invalid project cwd "<path>": workspace unavailable; restore the directory or choose another location
- `404` session not found
- `404` project not found
- `500` the workspace or the project list could not be read, or the library directory could not be created

Events: none.

### POST /v1/skills/inspect

Auth: admin
Effect: Downloads a URL and lists the skill bundles it holds, without installing anything or running any of it. Accepted: a raw `SKILL.md`; a GitHub repository page, `tree/<ref>/<path>`, `blob/<ref>/<path>` or `raw.githubusercontent.com/.../SKILL.md` URL (Core downloads that ref's archive from `codeload.github.com`, `HEAD` for a repository page, and keeps only the sub-path); a `.zip`, `.tar.gz` or `.tgz` archive (any body that is a zip or gzip archive is read as one). A bundle is a directory with a valid `SKILL.md`; a file belongs to the nearest bundle directory above it. The response sends `Cache-Control: no-store`.

Request:

```json
{ "url": "https://github.com/example/skills", "project_id": "3f2a1c0e-6b7d-4e18-9c0a-1b2d3e4f5a6b" }
```

- `project_id` (string) is optional: a registered, available project whose install directory `targets.project` should name.
- `url` (string) is required, at most 2048 characters, `http` or `https`, with a host and no user info. `localhost`, names ending in `.localhost` or `.local`, and loopback, private, link-local, multicast, unspecified and RFC 6598 (`100.64.0.0/10`) addresses are refused, also after DNS lookup and on every redirect.
- The body is JSON of at most 4096 bytes. Unknown fields are ignored.
- Limits: one download at most 50 MB, an archive walked for at most 512 MB decompressed, a bundle at most 20 MB and 500 files, `SKILL.md` at most 256 KiB. The download times out after 2 minutes.

Response: `200`

```json
{
  "url": "https://github.com/example/skills",
  "download": "https://codeload.github.com/example/skills/tar.gz/HEAD",
  "kind": "github",
  "bundles": [
    {
      "path": "skills/pdf",
      "name": "pdf",
      "description": "Work with PDF files",
      "files": [
        { "path": "SKILL.md", "size": 8123 },
        { "path": "scripts/fill.py", "size": 2040, "exec": true }
      ],
      "size": 10163,
      "digest": "0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef",
      "skipped": [".DS_Store"]
    }
  ],
  "warnings": ["skills/broken/SKILL.md: invalid name or description"],
  "targets": { "global": "/Users/me/.gyres/skills", "project": "/work/demo/.agents/skills" }
}
```

- `targets` are the absolute directories `POST /v1/skills/install` writes to on the machine running Core: `global` for scope `global`, `project` for scope `project` in the given `project_id` (the library's `.agents/skills` for a project without folders). `project` is `null` without `project_id`; either is `null` when that scope is unavailable.

- `kind` is `skill` (the URL was one `SKILL.md`; its bundle `path` is `.`), `github`, `archive`, or `page` (an ordinary web page or another document: `bundles` is empty and a warning says so; this is not an error).
- `bundles` are sorted by `path`, the directory inside the source (`.` for its root). `files` are sorted, relative to the bundle; `exec` marks an executable file. `skipped` lists entries left out: hidden files, links and devices. A bundle over the limits, or whose files differ only by ASCII letter case, has `error` and no `digest`, `files` or `size`, and cannot be installed.
- `digest` is the lowercase hex SHA-256 over every file's path, content hash and executable bit. Install checks it.
- `warnings` lists `SKILL.md` files that are not valid skills.

Errors:

- `400` `invalid URL request`
- `400` the URL was refused or could not be fetched: `URL is required and must be at most 2048 characters`, `URL must be an http or https address`, `local and private network URLs are not allowed`, `could not fetch <url>: ...`, `<url> returned HTTP <status>`, `download exceeds 50 MB`, `<url> is not a zip or tar.gz archive`, an archive that is invalid, expands past 512 MB or has a member with an absolute path, `..` or a backslash
- `400` project directories are unavailable (with `project_id`)
- `403` the key is not an administrator's
- `404` project not found (with `project_id`)

Events: none.

### POST /v1/skills/install

Auth: admin
Effect: Downloads the URL again and writes the chosen bundles (`SKILL.md` and every supporting file) onto the machine running Core: `scope` `global` into `<home>/skills/<name>`, `scope` `project` into that project's primary directory under `.agents/skills/<name>` (the library, when the project has no folders). `<name>` is each bundle's frontmatter name. Each bundle is recorded as an owner version in the skill's history. Nothing is run. The response sends `Cache-Control: no-store`.

Request:

```json
{
  "url": "https://github.com/example/skills",
  "bundles": ["skills/pdf"],
  "expected_digests": { "skills/pdf": "0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef" },
  "scope": "project",
  "project_id": "3f2a1c0e-6b7d-4e18-9c0a-1b2d3e4f5a6b",
  "replace": false
}
```

- The body is one JSON object of at most 65536 bytes and may contain only these fields.
- `url` is checked as in `POST /v1/skills/inspect`.
- `bundles` (array of strings) is required: bundle `path`s from the inspection. Two chosen bundles may not share a name.
- `expected_digests` (object) maps every chosen path to the `digest` the inspection returned. A bundle whose digest changed is refused and nothing is written.
- `scope` (string) is `global` or `project`. `project_id` (string) is required when `scope` is `project`, and must be an existing project whose primary directory is available.
- `replace` (boolean) allows overwriting a skill of the same name in that scope. The current bundle is recorded in its history first and then replaced as a whole. A bundle holding files history cannot keep (hidden files, links, files over the history limits) is never replaced.

Response: `200`

```json
{
  "installed": [
    { "name": "pdf", "path": "/work/demo/.agents/skills/pdf/SKILL.md", "files": 12 }
  ],
  "context_reload_required": true
}
```

- `installed` lists each written bundle: `name`, the `path` of its `SKILL.md`, the number of `files`, and `replaced: true` when it replaced an existing skill.
- When some bundles were written and a later one failed, the response is still `200` with those in `installed` and the failure in `error`.
- `context_reload_required` is always `true`. A conversation that already has a catalog does not list the new skills until its context is built again.

Errors:

- `400` `invalid Skill install request` (the body is not a JSON object of these fields, or is larger than 65536 bytes)
- `400` `scope must be global or project`
- `400` `project_id is required for project Skills`
- `400` the URL was refused or could not be fetched (as for inspect), a chosen bundle is not in the source, is over the limits, has no expected digest, or two chosen bundles share a name
- `400` `project directories are unavailable`
- `403` the key is not an administrator's
- `404` `project not found`
- `409` a bundle changed since it was inspected, or a skill of that name is already installed (send `replace`)

Events: none.

### POST /v1/skills/uninstall

Auth: admin
Effect: Removes one global or project skill that was listed for an available project primary directory or for every conversation. The bundle is recorded in its history first. A bundle directly in the skills directory whose every file the history keeps is removed whole. Otherwise (a folder linked from elsewhere, hidden files such as `.git`, links, files over the history limits) only `SKILL.md` is deleted, and its folder only when that leaves it empty. Built-in skills, plugin skills, and a skill that exists only under an additional project directory are left in place. The response sends `Cache-Control: no-store`.

Request:

```json
{
  "path": "/work/demo/.agents/skills/brand/SKILL.md",
  "expected_digest": "0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef"
}
```

- The body is one JSON object of at most 8192 bytes and may contain only these fields.
- `path` (string) is required. It must be absolute and already cleaned: the same string Core returned in `path`, or another absolute path that resolves to that file, with no `.` or `..` and no trailing separator.
- `expected_digest` (string) must equal that skill's current `digest`.

Response: `200`

```json
{
  "skill": {
    "name": "brand",
    "description": "Apply the brand voice",
    "path": "/work/demo/.agents/skills/brand/SKILL.md",
    "source": "/work/demo",
    "digest": "0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef"
  },
  "removed_bundle": true,
  "context_reload_required": true
}
```

- `removed_bundle` is `true` when the whole folder was removed, `false` when only `SKILL.md` was.
- `skill` is the catalog entry that was removed. `source` is `global` or the project directory, as in `GET /v1/skills`.
- `context_reload_required` is always `true`. A conversation that already has a catalog keeps listing the skill until its context is built again.

Errors:

- `400` path and expected_digest are required (the body is not a JSON object of these fields, is larger than 8192 bytes, or `path` is empty)
- `404` Skill is not removable
- `409` Skill changed; refresh the list and try again (this is also the result when `expected_digest` is empty)
- `500` the file could not be removed

Events: none.

### POST /v1/skills/review

Auth: admin
Effect: Marks a skill the Agent learned as looked at by the owner. Only the bundle's hidden `.gyres.json` marker changes: no history version is recorded and the catalog text is unchanged. The response sends `Cache-Control: no-store`.

Request:

```json
{
  "path": "/home/me/.gyres/skills/release-notes/SKILL.md",
  "expected_digest": "0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef"
}
```

- The body is one JSON object of at most 8192 bytes and may contain only these fields.
- `path` (string) is required, as in `POST /v1/skills/uninstall`: a global or project skill of the management list.
- `expected_digest` (string) must equal that skill's current `digest`.

Response: `200`

```json
{
  "path": "/home/me/.gyres/skills/release-notes/SKILL.md",
  "learned": { "created_by": "chat", "session_id": "s_123", "reviewed": true }
}
```

Errors:

- `400` path and expected_digest are required
- `403` the key is not an administrator's
- `404` Skill not found (not listed, or a plugin or built-in skill)
- `409` Skill changed; refresh the list and try again (also when `expected_digest` is empty)
- `409` this skill was not learned by the Agent

Events: none.

### GET /v1/skills/bundle

Auth: admin
Effect: Reads one skill of the management list for display: the files of its bundle and the text of its `SKILL.md`. Nothing is recorded. Hidden files and folders and symbolic links are left out of the file list. The response sends `Cache-Control: no-store`.

Query:

- `path` (string, required) is a skill's `path` from the management list of `GET /v1/skills` (or an absolute path resolving to it). Any other path is refused.

Request: no body.

Response: `200`

```json
{
  "path": "/home/me/.gyres/skills/release-notes/SKILL.md",
  "files": [
    { "path": "SKILL.md", "size": 1432 },
    { "path": "scripts/check.sh", "size": 210 }
  ],
  "truncated": false,
  "content": "---\nname: release-notes\n..."
}
```

- `files` are paths relative to the bundle folder with `/` separators, sorted, at most 200; `truncated` is `true` when there are more.
- `content` is `SKILL.md` as text, at most 256 KiB.

Errors:

- `403` the key is not an administrator's
- `404` Skill not found
- `500` the bundle could not be read

Events: none.

### GET /v1/skills/history

Auth: admin
Effect: Lists the saved versions of one global or project skill, newest first. Core keeps a version for every change to the skill's directory: before and after each write the Agent or the owner makes through Gyres, and, when a conversation's catalog is built or this endpoint is called, for any change made on disk by something else since the last version. Versions are stored outside the skill directory, so they survive a tool that deletes and copies the skill again. The latest 30 versions are kept per skill. Calling this endpoint first records a change on disk that has no version yet.

Query:

- `path` (string) is required: a global or project `SKILL.md` path as returned by `GET /v1/skills`, for every conversation or for an available project primary directory.

Response: `200`

```json
{
  "path": "/home/owner/.gyres/skills/release/SKILL.md",
  "name": "release",
  "editable": true,
  "versions": [
    { "id": 3, "at": "2026-10-20T08:00:00Z", "source": "external", "action": "found", "files": 2 },
    { "id": 2, "at": "2026-10-19T08:00:00Z", "source": "chat", "action": "edit", "files": 2 },
    { "id": 1, "at": "2026-10-18T08:00:00Z", "source": "external", "action": "found", "files": 2 }
  ],
  "notice": { "at": "2026-10-20T08:00:00Z", "local": 2, "external": 3 }
}
```

- `name` is the skill's directory name.
- `editable` is `false` for a skill directory linked from elsewhere; its history is kept but it cannot be rolled back.
- Each version has an increasing `id`, the time `at`, its `source` (`chat` and `review` for the Agent in a conversation or in background review, `owner` for the owner through Gyres, `external` for a change Gyres did not write) and an `action` (`create`, `edit`, `install`, `rollback`, or `found` for an external change). `restored_from` is the version a rollback restored. `files` counts the files in the version. `partial` is `true` when some files were left out: hidden files and symbolic links are never included, and a version holds at most 200 files, 1 MiB per file and 8 MiB in total.
- `notice` is set when an external change replaced a version written through Gyres: `local` is that version and `external` the one that replaced it. It is `null` otherwise. It is cleared by a rollback or by `POST /v1/skills/history/dismiss`.

Errors:

- `404` Skill has no history (the path is not a listed global or project skill)
- `500` the history could not be read or written

Events: none.

### GET /v1/skills/history/notices

Auth: admin
Effect: Lists the skills with an open notice (see `GET /v1/skills/history`), so a client can mark them in its skill list without fetching every history.

Response: `200`

```json
{
  "notices": [
    { "path": "/home/owner/.gyres/skills/release/SKILL.md", "name": "release", "notice": { "at": "2026-10-20T08:00:00Z", "local": 2, "external": 3 } }
  ]
}
```

- `path` is the skill directory's real location joined with `SKILL.md`, which is what `GET /v1/skills` returns for a skill that is not linked from elsewhere.

Errors: none beyond authentication.

Events: none.

### GET /v1/skills/history/diff

Auth: admin
Effect: Returns a unified diff between two versions of one skill.

Query:

- `path` (string) as in `GET /v1/skills/history`.
- `from` (integer) is the older version id, or `0` for an empty skill.
- `to` (integer) is the newer version id.

Response: `200`

```json
{
  "patch": "--- a/SKILL.md\n+++ b/SKILL.md\n@@ -4,1 +4,2 @@\n 1. run tests\n+2. publish\n",
  "truncated": false
}
```

- `patch` lists changed files in path order; a file that is not UTF-8 text appears as `Binary file <path> changed`. An empty string means the versions are the same.
- `truncated` is `true` when the diff was cut at 512 KiB.

Errors:

- `400` from and to must be version ids (from 0 for an empty bundle)
- `404` Skill has no history
- `404` version `<id>` not found, or a stored copy is missing or damaged

Events: none.

### POST /v1/skills/history/rollback

Auth: admin
Effect: Makes one saved version the skill's content again. Its files are written back with their saved permissions, and files of the current version that the restored one does not have are removed (not when the restored version is `partial`). Files that no version holds, such as hidden files (the Agent's marker file) or files over the limits, are left alone. The rollback is always recorded as a new version, even when it restores content an earlier version already had (`source` `owner`, `action` `rollback`); no version is deleted. It clears the notice. Restoring the version named in `notice.local` re-applies the change an external update replaced.

Request:

```json
{
  "path": "/home/owner/.gyres/skills/release/SKILL.md",
  "version": 2,
  "expected_head": 3
}
```

- The body is one JSON object of at most 8192 bytes and may contain only these fields.
- `version` (integer) is the version to restore.
- `expected_head` (integer) must equal the newest version id, after any change on disk is recorded; otherwise nothing is written.

Response: `200`

```json
{ "history": { "path": "...", "name": "release", "editable": true, "versions": [], "notice": null }, "context_reload_required": true }
```

- `history` is the skill's history after the rollback, shaped as in `GET /v1/skills/history`.
- `context_reload_required` is always `true`: conversations that already have a catalog read the skill again only after their context is built again; a skill body is read from disk whenever the Agent reads it.

Errors:

- `400` path, version and expected_head are required
- `400` version `<id>` not found, this version is already the current one, or a stored copy is missing or damaged
- `400` restored version `<id>`, but could not remove `<files>`: the other files were restored and the result is recorded as a version
- `404` Skill has no history
- `409` the skill changed; refresh its history and try again
- `409` this skill is linked from outside the skills directory; it is managed by the owner

Events: none.

### POST /v1/skills/history/dismiss

Auth: admin
Effect: Clears a skill's notice without changing its files.

Request:

```json
{ "path": "/home/owner/.gyres/skills/release/SKILL.md" }
```

Response: `200`, the history as in `GET /v1/skills/history`, with `notice` `null`.

Errors:

- `400` path is required
- `404` Skill has no history
- `500` the history could not be written

Events: none.
