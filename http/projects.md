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
- `source` is `global` for a skill installed for every conversation, the project directory the bundle was found under for a project skill, `builtin` for a skill shipped with Core, or `plugin:<name>` for a skill from an enabled plugin. The install response uses `global` or `project` instead; see `POST /v1/skills/install`.
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
          { "path": "/work/demo", "available": true },
          { "path": "/work/notes", "available": true }
        ]
      }
    }
  ],
  "host_os": "darwin"
}
```

- `projects` is the saved projects in the order they were saved, each with `status`. An empty Core returns `[]`.
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
- `400` the project could not be saved (the message says why)
- `409` another project already has this name

Events: none.

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

Response: `200`

The saved project object, as for `POST /v1/projects`.

Errors:

- `400` invalid project
- `400` the same directory and name failures as create
- `400` project not found
- `409` another project already has this name

Events: none.

### DELETE /v1/projects/{project}

Auth: any key
Effect: Forgets the registration. Directories, library files, instructions, project memory, and conversation bindings are left in place. A later create with the same primary path, or the same name for a project without folders, can take this id again.

Path: `project` project id.

Request: no body.

Response: `200`

```json
{ "ok": true }
```

Errors:

- `404` project not found
- `500` the registration could not be updated

Events: none.

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
- With none of those three, the list is the global skills, plus plugin and built-in skills, and skills that require a library are left out. `manage_scope=all` together with none of those three is the management list: a fresh reading of every available project's primary directory, plus global, plugin, and built-in skills, including skills that require a library. A project whose primary directory is unavailable is skipped.
- `manage_scope` (string) filters the list when it is set. `all` keeps every source. `global` keeps `source` `global`. `builtin` keeps `source` `builtin`. `project` keeps a source that is not `global`, not `builtin`, and does not start with `plugin:`. Any other value leaves `skills` empty. `warnings` is unchanged.
- `q` (string) is a case-insensitive substring of `name` or `description`. It applies only when `manage_scope` is set. An empty `q` does not narrow the list.

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

Errors:

- `400` invalid project cwd "<path>": workspace unavailable; restore the directory or choose another location
- `404` session not found
- `404` project not found
- `500` the workspace or the project list could not be read, or the library directory could not be created

Events: none.

### POST /v1/skills/install

Auth: admin
Effect: Fetches one `SKILL.md` from an `http` or `https` URL and writes it onto the machine running Core. `scope` `global` installs it for every conversation. `scope` `project` installs it under that project's primary directory (the library, when the project has no folders). Scripts in the bundle are not run. The response sends `Cache-Control: no-store`.

Request:

```json
{
  "url": "https://192.0.2.10/skills/brand/SKILL.md",
  "scope": "project",
  "project_id": "3f2a1c0e-6b7d-4e18-9c0a-1b2d3e4f5a6b",
  "expected_digest": "0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef"
}
```

- The body is one JSON object of at most 8192 bytes and may contain only these fields.
- `url` (string) is required, at most 2048 characters, `http` or `https`, with a host and no user name. A local or private address is rejected, including after DNS lookup and on a redirect. The fetch gives up after 20 seconds. The file must be at most 256 KiB and must be a skill file: frontmatter between `---` lines, a `name` and a `description` as in the skill object above, and `requires` containing only `library` when it is present. Other frontmatter fields are ignored.
- `scope` (string) is `global` or `project`.
- `project_id` (string) is required when `scope` is `project`, and must be an existing project whose primary directory is available.
- `expected_digest` (string) is optional. When it is sent, it must equal the digest of the fetched file or nothing is written.

Response: `200`

```json
{
  "skill": {
    "name": "brand",
    "description": "Apply the brand voice",
    "path": "/work/demo/.agents/skills/brand/SKILL.md",
    "source": "project",
    "digest": "0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef"
  },
  "context_reload_required": true
}
```

- `skill` is the installed file. `path` is where it was written. `source` is the request `scope`, `global` or `project`, which is not the `source` a later `GET /v1/skills` returns for a project skill.
- `context_reload_required` is always `true`. A conversation that already has a catalog does not list the new skill until its context is built again.

Errors:

- `400` invalid Skill install request (the body is not a JSON object of these fields, or is larger than 8192 bytes)
- `400` scope must be global or project
- `400` project_id is required for project Skills
- `400` the URL or the file was rejected. Stable messages include `URL is required and must be at most 2048 characters`, `URL must be an http or https address`, `local and private network URLs are not allowed`, `could not fetch Skill: ...`, `Skill URL returned HTTP <status>`, `could not read Skill: ...`, `SKILL.md exceeds 256 KiB`, `missing YAML frontmatter`, `unterminated YAML frontmatter`, `invalid name or description`, and `unknown requires "<value>" (only library)`
- `400` project directories are unavailable
- `404` project not found
- `409` Skill changed after preview; preview it again
- `409` Skill is already installed; remove it before reinstalling
- `500` the project list could not be read, the destination could not be checked, or the file could not be written

Events: none.

### POST /v1/skills/uninstall

Auth: admin
Effect: Removes one global or project skill that was listed for an available project primary directory or for every conversation. The `SKILL.md` file is deleted. The directory that contained it is removed only when that leaves the directory empty; any other file there is kept. Built-in skills, plugin skills, and a skill that exists only under an additional project directory are left in place. The response sends `Cache-Control: no-store`.

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
  "context_reload_required": true
}
```

- `skill` is the catalog entry that was removed. `source` is `global` or the project directory, as in `GET /v1/skills`.
- `context_reload_required` is always `true`. A conversation that already has a catalog keeps listing the skill until its context is built again.

Errors:

- `400` path and expected_digest are required (the body is not a JSON object of these fields, is larger than 8192 bytes, or `path` is empty)
- `404` Skill is not removable
- `409` Skill changed; refresh the list and try again (this is also the result when `expected_digest` is empty)
- `500` the file could not be removed

Events: none.
