# Files, attachments and search

These routes let a frontend upload images and ordinary files into a conversation, browse and search files on the Core host, open a live workspace file, fetch a presented snapshot, search the owner's conversation text, and preview a skill document from a URL.

A conversation id that Core creates looks like `2026-10-14T12-00-00_01234567-89ab-cdef-0123-456789abcdef`: the UTC time to the second with `:` changed to `-`, then `_`, then a UUID. Any id of 1 to 160 letters, digits, `-` and `_` is accepted. A lane of that conversation is `<id>~<lane>` while the lane is open. The lane name is 1 to 48 characters from `a-z`, `0-9`, `_` and `-`, does not start with `_` or `-`, and is not `main`. `{id}` below accepts either form. Image uploads and ordinary-file uploads are stored under the conversation id and shared by its lanes.

JSON responses are `application/json`. A JSON error, unless an endpoint says otherwise, is:

```json
{
  "error": "session not found",
  "failure": {
    "code": "core_request_failed",
    "message": "session not found",
    "http_status": 404,
    "details": "session not found"
  }
}
```

`error` is the sentence listed below, cut at 2048 characters, with credential-like text and URL queries removed. `failure.message` and `failure.details` repeat it, and `failure.http_status` is the response status. A `413` is different: `failure.code` is `request_too_large`, `failure.message` is `The request exceeds the service's size limit. Reduce the input or switch models.`, and `error` plus `failure.details` keep the specific sentence. If that sentence itself contains `HTTP` and a status number, `failure` describes that status instead, and the HTTP status of the response can differ.

Responses allow any browser origin. A preflight `OPTIONS` needs no access key and returns `204`.

Three routes share one workspace choice, read from the query string or, for reveal, from the same fields in the body. `session_id` wins when it is present: an unknown or invalid id is `404` `session not found`, and a conversation with no workspace behaves as an empty workspace. Otherwise `project_id`: an unknown id is `404` `project not found`, and the workspace is that project's directories, including its library. Otherwise `cwd`, used as the working directory alone. With none of the three, the workspace is empty. When the chosen workspace has a library, that directory is created if it is missing; if it cannot be created the response is `500` with the error text.

File search and reference listing walk the working directory and then each additional root that can be listed. A missing additional root is skipped. The library and the conversation's own directory stay in the set.

### GET /v1/directories

Auth: any key
Effect: Lists one directory on the Core host. Nothing is created, except that an existing project's library directory is created when `path` is that library and it is missing.

Query: `path` (absolute path on the Core host; omitted or empty lists the home directory, or `400` when that directory cannot be named). `files` (`1` includes files). `hidden` (`1` includes names that start with `.`; only affects `items`). `limit` (present switches to one page; an integer, values below 1 become 1, values above 2000 become 2000; a non-numeric value becomes 1). `offset` (page start, used only with `limit`; omitted or non-numeric is 0; a negative value selects from the start). `ignore` (`1`, and only with `limit` on an absolute path, drops `.git` and names excluded by ignore files).

Request: no body.

Response: `200`

Without `limit`, directory paths only. `entries` lists subdirectories, including names that start with `.`, up to the first 2000 names the filesystem returns. `truncated` is true when the directory has more than 2000 names.

```json
{
  "path": "/home/example/project",
  "parent": "/home/example",
  "entries": ["/home/example/project/src"],
  "truncated": false,
  "host": "core"
}
```

With `files=1`, `items` is added: directories first, then regular files, each sorted by name. A symlink is followed. Names that start with `.` are omitted unless `hidden=1`. Anything that is not a file or directory is omitted. One element of `items`:

```json
{
  "name": "notes.txt",
  "path": "/home/example/project/notes.txt",
  "kind": "file",
  "size": 12,
  "mtime": 1760438400
}
```

`kind` is `dir` or `file`. `size` is bytes. `mtime` is Unix seconds. `host` is the Core machine's host name, or `""` when it cannot be read. `parent` is the parent directory. `path` and every path in `entries` and `items` are absolute on the Core host.

With `limit`, the same object also has `total` and `offset`. `total` counts names after the hidden and ignore filters, before the page. `offset` echoes the requested offset. The page is directories first, then files, sorted by lower-case name and then by the original name. Only that page is stat'ed: `entries` holds the directory paths on the page, and `items` holds both kinds. A name that cannot be stat'ed is counted in `total` and left out of the page. `truncated` is true when the directory has more than 20000 names, before filtering. Symlinks are followed; a symlink that is not a file or directory is left out. `ignore=1` drops `.git` and names excluded by `.gitignore`, `.ignore`, `.rgignore`, and `.git/info/exclude`, collected from the nearest git root down to this directory.

Errors:

- `400` the path is not an absolute directory that can be listed, or it is not a directory. The text is `use an absolute directory path on the Core host`, or `invalid project cwd "<path>": <reason>` (`not a directory`, or the operating-system error).
- `500` a project library at `path` could not be created

Events: none.

### POST /v1/directories

Auth: admin
Effect: Creates one directory on the Core host. An existing directory is left as it is and still succeeds. Sends `Cache-Control: no-store`.

Request:

```json
{ "path": "/home/example/project/notes", "parents": false }
```

- `path` (string) is an absolute directory path on the Core host, at most 4096 bytes, with no NUL and no `..` segment. Required for success.
- `parents` (boolean, optional, default false). False creates exactly one level and the parent must already be a directory. True creates missing parents as well.
- The body is JSON of at most 65536 bytes. Unknown fields are ignored.

Response: `200`

```json
{ "path": "/home/example/project/notes", "name": "notes", "created": true }
```

- `path` is the cleaned absolute path. `name` is its last segment.
- `created` is false when that directory already existed.

Errors:

- `400` `invalid request` (body is not JSON, or is larger than 65536 bytes)
- `400` `invalid directory path: <reason>` (`path is empty or contains NUL`, `path is too long`, `use an absolute directory path on the Core host`, `folder name is required`, `path must not contain ..`)
- `404` `parent folder does not exist` (also when the parent exists but is not a directory)
- `409` `a file with this name already exists`
- `403` `permission denied: Core cannot create a folder here`
- `500` any other failure, with the error text

Events: none.

### GET /v1/file-search

Auth: any key
Effect: Fuzzy-searches file names under the workspace roots and returns the best matches. A root that cannot be read is skipped.

Query: `query` (string, trimmed; compared in lower case). `limit` (integer, default 50, maximum 100; below 1 or non-numeric becomes 50). `hidden` (`1` includes names that start with `.`). `session_id`, `project_id`, `cwd` choose the workspace, as above.

Request: no body.

Response: `200`

```json
{
  "files": [
    {
      "type": "file",
      "path": "/home/example/project/src/notes.txt",
      "label": "src/notes.txt",
      "root": "/home/example/project"
    }
  ],
  "partial": false
}
```

- An empty workspace, or a query that is empty after trimming, returns `files: []` and `partial: false` and does not walk. A non-empty query on a workspace that is not usable is `400` instead.
- The query is split on whitespace. Every term must appear in order as a subsequence of the path relative to the root. A match in the file name ranks above a match only in a directory. The score is not returned. Results are best match first, then `label` ascending, cut to `limit`.
- `path` is absolute on the Core host. `label` is relative to `root` with forward slashes. `type` is always `file`.
- Hidden names and `.git` are skipped unless `hidden=1` (`.git` stays skipped). Ignore files apply, the same set as `ignore=1` on a directory page. Symlinks are not followed.
- The walk stops after 3 seconds or 20000 files. `partial` is then true. It is also true when a root could not be read. If the client goes away during the walk, the response is not written.

Errors:

- `404` `session not found` or `project not found` (workspace choice)
- `400` `invalid project cwd "<path>": workspace unavailable; restore the directory or choose another location` when the query is non-empty and the workspace cannot be used
- `500` the conversation's workspace could not be read, its library could not be created, or the project list could not be read, with the error text

Events: none.

### GET /v1/reference-files

Auth: any key
Effect: Lists up to 50 files under the workspace roots whose relative path contains `query`. This is a literal substring search, not the fuzzy file search.

Query: `query` (string; empty matches every file until the cap; comparison ignores case). `session_id`, `project_id`, `cwd` choose the workspace, as above.

Request: no body.

Response: `200`

```json
{
  "files": [
    {
      "type": "file",
      "path": "/home/example/project/src/notes.txt",
      "label": "src/notes.txt"
    }
  ],
  "partial": false
}
```

- An empty workspace returns `files: []` and `partial: false`.
- `path` is the symlink-resolved absolute path inside the workspace. `label` is the path relative to the root it was found under. `type` is `file`.
- Directories whose name starts with `.`, and directories named `node_modules`, are not entered. Symlinks are skipped. Other hidden files are included.
- At most 50 files. `partial` is true when the walk stops early: 50 files are already collected and more entries remain, more than 20000 entries were visited, or an entry could not be read.

Errors:

- `404` `session not found` or `project not found`
- `400` `invalid project cwd "<path>": workspace unavailable; restore the directory or choose another location`
- `500` the conversation's workspace could not be read, its library could not be created, the project list could not be read, or a root could not be walked (the error text)

Events: none.

### POST /v1/reveal-path

Auth: admin
Effect: Asks the Core host's file manager to show a file or directory inside the workspace. On macOS and Windows the file is selected in its folder; on other systems the path is opened. Nothing is copied or deleted. Sends `Cache-Control: no-store`. This only makes sense when the person is at the Core host.

Request:

```json
{
  "session_id": "2026-10-14T12-00-00_01234567-89ab-cdef-0123-456789abcdef",
  "project_id": "",
  "cwd": "",
  "path": "/home/example/project/notes.txt"
}
```

- `session_id`, `project_id` and `cwd` choose the workspace the same way as the query on file search. An empty workspace is rejected.
- `path` (string) must be absolute and must resolve, after symlinks, to something that exists inside that workspace.
- The body is JSON of at most 65536 bytes. Unknown fields are ignored.

Response: `200`

```json
{ "ok": true }
```

Errors:

- `400` `invalid request`
- `404` `session not found` or `project not found`
- `500` the conversation's workspace could not be read, its library could not be created, or the project list could not be read, with the error text
- `400` `path must be absolute inside the workspace` (the workspace has no working directory, or `path` is not absolute)
- `400` `path is outside this workspace` (missing, not resolvable, or resolved outside the workspace)
- `500` `could not open the file manager on the Core host: <detail>`, or the operating-system error if the path disappears after the check

Events: none.

### GET /v1/session-search

Auth: any key
Effect: Finds conversations whose message text contains `query`. Returns the first match of each lane, newest conversation ids first, at most 20. Archived conversations are skipped. So are background Bot tasks that the conversation list hides, and a conversation started by another conversation that still exists. Bot chats stay, and are marked.

Query: `query` (string, trimmed). Empty or whitespace returns no hits and does not scan. Longer than 256 Unicode code points is `400`.

Request: no body.

Response: `200`

```json
{
  "hits": [
    {
      "session_id": "2026-10-14T12-00-00_01234567-89ab-cdef-0123-456789abcdef",
      "message_id": "msg_01234567-89ab-cdef-0123-456789abcdef",
      "role": "user",
      "time": "2026-10-14T12:00:00.000Z",
      "text": "notes from the review",
      "title": "Review",
      "bot": true
    }
  ],
  "has_more": false,
  "partial": false
}
```

- The match is a case-insensitive literal substring. Searched text is user and assistant text, a tool name plus its arguments, and tool-result text. A session-search tool's own call and result are skipped, as are internal records and messages with no such text.
- Conversations are visited by descending id, so ids that start with a timestamp come newest first, at most 1000 files and 128 MiB of transcripts. Inside a conversation the main lane comes first, then each open side lane. A hit whose row is only on a side lane has `session_id` `<id>~<lane>`.
- `message_id` is that message's id (`msg_` plus a UUID, or an older `legacy_...` id). `time` is the stored timestamp. `role` is `user`, `assistant` or `tool_result`.
- `text` is at most 240 Unicode code points, taken from a window that starts at most 120 code points before the match and is itself at most 2000. A title-only match is dropped, so `hits` can be shorter than the scan.
- `title` is present only when the conversation has a custom title. `bot` is present and true for the Bot's main chat, a channel chat, or a side lane of one.
- `has_more` is true when the scan stopped early: 20 lanes, the excerpt budget, or a message longer than the window. This route has no parameter to read the rest.
- `partial` is true when only the newest 1000 files were scanned, the 128 MiB budget was reached, or a transcript could not be read (including one larger than 64 MiB). Hits found before that are still returned.
- If the client goes away during the scan, the response is not written.

Errors:

- `400` `query is too long`
- `500` the scan failed, with the error text

Events: none.

### POST /v1/sessions/{id}/attachments

Auth: any key
Effect: Stores one image on the conversation. The same bytes stored again return `200` and are not written a second time. The image is removed when the conversation is deleted.

Path: `id` conversation id, or an open lane. Unknown is `404`.

Query: `name` (optional file name). The stored name is the last segment, and `\` counts as a separator. Empty or `.` becomes `image`. Longer than 255 bytes is `400`.

Request: the raw image bytes, not a multipart body. The request `Content-Type` is ignored. At most 20 MiB.

Response: `201` when the image is new, `200` when those bytes were already stored. The `name` in a `200` is the name from this request.

```json
{
  "id": "0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef",
  "name": "photo.png",
  "mime_type": "image/png",
  "size": 12345
}
```

- `id` is the lowercase hex SHA-256 of the bytes (64 characters). Later routes use this id, not the name.
- `mime_type` is `image/png`, `image/jpeg`, `image/gif` or `image/webp`, from the bytes.
- `size` is the byte length.

PNG, JPEG, GIF and WebP only. Width and height must each be at most 16384, and the product at most 40 million. Empty bytes are rejected. One conversation holds at most 100 MiB of these files.

Errors:

- `404` `session not found`
- `413` `Image exceeds the 20 MiB upload limit` (the body is larger than 20 MiB, or it could not be read)
- `400` `Each image must be between 1 byte and 20 MiB`
- `400` `Invalid image; supported formats are PNG, JPEG, GIF and WebP`
- `400` `Image dimensions exceed the limit (40 million pixels or 16384 pixels per side)`
- `400` `Image name is too long`
- `413` `This conversation has reached its 100 MiB image storage limit`
- `500` `Cannot read image storage`, `Cannot inspect image storage`, or `Cannot create image storage`
- `507` `Cannot save image: the disk is full` or `Cannot save image metadata: the disk is full`
- `503` storage is unavailable. The body adds `"code": "storage_unavailable"` and the response sets `Retry-After: 5`. `error` is `Cannot save image: <storage message>` or the metadata equivalent.
- `500` `Cannot save image: <detail>` or `Cannot save image metadata: <detail>` for any other failed write

Events: none.

### GET /v1/sessions/{id}/attachments/{attachment}

Auth: any key
Effect: Returns the stored image bytes for this conversation.

Path: `id` conversation id or open lane. `attachment` the 64-character lowercase hex id from the upload.

Request: no body.

Response: `200`, body is the raw image. `Content-Type` is the stored image type. Also `X-Content-Type-Options: nosniff` and `Cache-Control: private, no-store`. Not JSON.

Errors:

- `404` `session not found`
- `404` `Invalid attachment ID` (not 64 lowercase hex characters)
- `404` `Image attachment not found`, `Invalid image attachment`, `Cannot read image attachment`, `Image attachment checksum mismatch`, `Image metadata not found`, `Invalid image metadata`, or the same image-validation sentences as upload when the stored bytes no longer decode

Events: none.

### DELETE /v1/sessions/{id}/attachments/{attachment}

Auth: any key
Effect: Removes one stored image from the conversation. The image is deleted only when it still reads as a valid attachment. The transcript is not changed.

Path: `id` conversation id or open lane. `attachment` the image id.

Request: no body.

Response: `200`

```json
{ "deleted": true }
```

Errors:

- `404` `session not found`, or any `404` from reading the image (same sentences as GET)
- `500` `Cannot remove image`

Events: none.

### POST /v1/sessions/{id}/documents

Auth: any key
Effect: Writes one ordinary file, or creates one directory, under the conversation's workspace at `.gyres-uploads/<conversation id>/<batch>/`. The bytes are not sent to the model. If the conversation has no workspace, a scratch workspace is created. Repeating the same path and the same bytes succeeds. A different file at that path is refused and nothing is overwritten.

Path: `id` conversation id or open lane.

Query: `batch` (required, a 36-character UUID). `name` (required relative path, at most 1024 bytes, no leading `/`, no `\`, NUL, CR or LF; each segment is 1 to 255 bytes, not empty, `.` or `..`, and contains no `:`). `kind` (`directory` creates a directory and does not read the body; any other value, or omitted, stores a file).

Request: for a file, the raw bytes, not multipart, at most 128 MiB. For a directory, no body is read.

Response: `201`

```json
{
  "type": "file",
  "path": "/home/example/project/.gyres-uploads/2026-10-14T12-00-00_01234567-89ab-cdef-0123-456789abcdef/01234567-89ab-cdef-0123-456789abcdef/notes.txt",
  "label": "notes.txt"
}
```

- `type` is `file` for a single file whose `name` has no `/`. It is `directory` when `kind=directory` or when `name` contains `/`. The directory reference points at the first segment, so every file of one folder upload returns the same directory path; clients deduplicate it.
- `path` is absolute on the Core host. `label` is that first segment, cut to 200 Unicode code points.
- The same bytes already stored at that path return `201` as well. The conversation holds at most 512 MiB and 2048 files of uploads. An upload that would pass either limit stores nothing.

Errors:

- `404` `session not found`
- `400` `Invalid upload batch`
- `400` `Invalid upload path`
- `400` the workspace cannot be used, with that error text (including `missing workspaces directory`)
- `400` `Workspace unavailable`
- `413` `Conversation uploads exceed 512 MiB or 2048 files`
- `413` `File upload failed or exceeds 128 MiB per file / 512 MiB per conversation` (too large, or the body could not be read). Both use the `413` failure shape above.
- `409` `Upload path is not a regular file`
- `409` `A different file already exists in this upload`
- `500` `Cannot create upload storage`, `Cannot inspect upload storage` (including a symlink inside the upload folder), `Cannot verify previous upload`, or `Cannot save uploaded file`

Events: none.

### GET /v1/sessions/{id}/document-preview

Auth: any key
Effect: Returns a short preview of one workspace file or directory: at most 128 KiB of text, or at most 200 names. The file is not modified. Sends `Cache-Control: no-store`.

Path: `id` conversation id or open lane.

Query: `path` (absolute path on the Core host). It must resolve, after symlinks, to a regular file or a directory inside the working directory or a workspace root.

Request: no body.

Response: `200`

A directory. `entries` are names in filesystem order, with `/` appended to directories. `truncated` is true when more than 200 names exist. `size` is the directory's size as reported by the filesystem, not the preview length.

```json
{
  "name": "src",
  "size": 4096,
  "truncated": false,
  "entries": ["main.go", "assets/"]
}
```

A file with no NUL byte in the first 128 KiB. `text` is that prefix as UTF-8, with invalid sequences replaced by U+FFFD. `truncated` is true when the file is longer than 128 KiB. A file that contains a NUL byte omits `text`.

```json
{
  "name": "notes.txt",
  "size": 12,
  "truncated": false,
  "text": "hello"
}
```

`name` is the base name.

Errors:

- `404` `session not found`
- `404` `Workspace unavailable` (the conversation has no workspace)
- `400` `File is outside the workspace or unavailable` (missing, relative, outside the roots, or not a file or directory)
- `400` `Workspace unavailable` (a root that contains the file cannot be opened)
- `400` `File unavailable`
- `500` `Cannot read directory` or `Cannot preview file`

Events: none.

### GET /v1/sessions/{id}/presented-files/{file}

Auth: none
Effect: Returns one presented snapshot. The bytes are a copy taken when the file was presented; later edits to the original are not visible. No access key is required, and a missing or wrong key is still accepted. `HEAD` returns the same headers and no body.

Path: `id` conversation id, or an open lane. Snapshots are stored on the conversation, so a lane reads the same files. `file` the snapshot id, 32 lowercase hex characters.

Query: `download` (`1` forces a download). Any other value leaves the default.

Request: no body.

Response: `200`, the snapshot bytes. `Content-Type` is the snapshot's media type. `Content-Disposition` is `inline` for `image/png`, `image/jpeg`, `image/gif`, `image/webp`, `image/bmp`, `application/pdf`, `video/mp4`, `video/webm` and `video/ogg`, and `attachment` for everything else, including HTML and SVG. `download=1` forces `attachment`. The `filename` parameter is the snapshot's file name.

Also `X-Content-Type-Options: nosniff`, `Cache-Control: private, no-cache`, `ETag: "<file id>"`, and `Access-Control-Expose-Headers: Accept-Ranges, Content-Length, Content-Range`.

A `Range` header returns `206` with that range. `If-None-Match` or `If-Modified-Since` can return `304` with no body.

Errors:

- `404` the conversation, the id, or the snapshot is missing or unreadable. The body is plain text `404 page not found`, not the JSON envelope.
- `416` the range cannot be satisfied, plain text, not the JSON envelope

Events: none.

### POST /v1/sessions/{id}/presented-files/{file}/save-to-library

Auth: admin
Effect: Copies the presented snapshot into the library of the conversation's project. The original snapshot stays. An existing library file is never overwritten. Sends `Cache-Control: no-store`.

Path: `id` conversation id or open lane. `file` the 32-character snapshot id.

Request: no body.

Response: `200`

```json
{
  "path": "/home/example/data/projects/01234567-89ab-cdef-0123-456789abcdef/library/notes (2).txt",
  "project_id": "01234567-89ab-cdef-0123-456789abcdef"
}
```

- `path` is the absolute path written. The file name is the snapshot's base name, with surrounding spaces removed: characters before U+0020, U+007F, `/`, `\` and `:` become `_`; leading `.` and spaces are removed; an empty name becomes `file`; longer than 200 bytes is cut on a character boundary. If that name exists, ` (2)`, ` (3)`, … is inserted before the last dot suffix, through ` (999)`.
- `project_id` is the conversation's project.

Errors:

- `404` `session not found`
- `500` the workspace could not be read, with the error text
- `400` `this conversation does not belong to a project with a library`
- `404` `file not found`
- `500` the snapshot could not be opened, or the library write failed, with the error text. That includes `project library unavailable`, `<path> must be a real directory`, and `too many files named <name> in the library`.

Events: none.

### POST /v1/sessions/{id}/preview

Auth: any key
Effect: Mints a short-lived URL for one regular file in the conversation's workspace. The URL needs no access key. Grants are kept in memory only: one hour, extended on every successful read, at most 4096 (the oldest is dropped), and all are dropped when Core restarts. Sends `Cache-Control: no-store`.

Path: `id` conversation id or open lane.

Request:

```json
{ "path": "/home/example/project/outputs/index.html" }
```

- `path` (string) is an absolute Core-host path that must resolve, after symlinks, to a regular file inside the working directory or a workspace root. A directory is rejected.
- The body is JSON of at most 65536 bytes. Unknown fields are ignored. A missing or empty `path` is `400`.

Response: `200`

```json
{
  "url": "/v1/preview/0123456789abcdef0123456789abcdef/index.html",
  "expires_at": "2026-10-14T13:00:00Z",
  "name": "index.html",
  "content_type": "text/html; charset=utf-8",
  "size": 128
}
```

- `url` is relative to Core. The token is 32 lowercase hex characters. The file name is percent-encoded (`index page.html` becomes `index%20page.html`).
- `expires_at` is one hour after this call, UTC. A later read of the token does not change this value; it does extend the grant.
- `name` is the base name. `size` is bytes. `content_type` is chosen as on the GET below.
- The grant covers the file's directory, so relative assets of an HTML page load. It does not cover the parent directory.

Errors:

- `404` `session not found`
- `400` `path is required`
- `404` `Workspace unavailable`
- `400` `File is outside the workspace or unavailable` (missing, relative, outside the roots, a directory, or not a regular file)
- `400` `File unavailable`
- `500` the token could not be generated, with the error text

Events: none.

### GET /v1/preview/{token}/{path...}

Auth: none
Effect: Serves one regular file under the directory granted by the token. No access key is required. Each successful read extends that grant by one hour. `HEAD` returns the same headers and no body. A directory listing is not served.

Path: `token` the 32-character token. `path` the file name, or a relative path below that directory (`assets/app.js`). `{path...}` is required.

Request: no body.

Response: `200`, the file bytes. `Content-Type` is `text/html; charset=utf-8` for `.html` and `.htm`, `image/svg+xml` for `.svg`, `text/javascript; charset=utf-8` for `.js` and `.mjs`, and `text/plain; charset=utf-8` for `.md`, `.markdown`, `.txt`, `.log`, `.csv` and `.tsv`. Any other extension uses the standard type for that extension, or `application/octet-stream`.

HTML and SVG also send `Content-Security-Policy: sandbox allow-scripts allow-forms allow-popups allow-modals allow-downloads` (no `allow-same-origin`). Other types send no such policy.

Also `X-Content-Type-Options: nosniff`, `Cache-Control: no-store`, `Referrer-Policy: no-referrer`, and `Access-Control-Expose-Headers: Accept-Ranges, Content-Length, Content-Range, ETag`. `ETag` is `"<hex size>-<hex nanosecond mtime>"`.

A `Range` returns `206`. `If-None-Match` or `If-Modified-Since` can return `304` with no body. A path that is not in canonical form is answered with `301` to the cleaned path; that cleaned path no longer includes the token.

`..`, a percent-encoded `..`, a symlink that leaves the directory, a directory, and a missing name are not served.

Errors:

- `404` unknown or expired token, empty path, a NUL in the path, the conversation is gone, or the name is not a regular file in the granted directory. Plain text `404 page not found`, not the JSON envelope.
- `416` the range cannot be satisfied, plain text, not the JSON envelope

Events: none.

### GET /v1/sessions/{id}/computer-use/frames

Auth: any key
Effect: Lists progress screenshots for this conversation or lane, oldest first. They exist only in memory, at most 10, and each is dropped 10 minutes after `at`. They are not part of the transcript. Core restart drops them, and deleting the conversation drops them. A frame is recorded only while the viewer is enabled, and only from a capture of the window being operated. Sends `Cache-Control: no-store`.

Path: `id` conversation id or open lane.

Request: no body.

Response: `200`

```json
{
  "frames": [
    {
      "id": "0123456789ab",
      "at": "2026-10-14T12:00:00Z",
      "app": "Notes",
      "label": "After click",
      "mime": "image/png"
    }
  ]
}
```

- `id` is 12 lowercase hex characters. `at` is UTC. `app` is the application name. `label` is the window title, or `After ` plus the preceding action when one was recorded (and that action's element number, when it named one), cut at 120 bytes on a UTF-8 boundary. `mime` is the image type.
- The list does not include the pixels. An image larger than 2 MiB is not kept. When computer use is not available on this Core, `frames` is `[]`.

Errors:

- `404` `session not found`

Events: none.

### GET /v1/sessions/{id}/computer-use/frames/{frame}

Auth: any key
Effect: Returns one progress screenshot as base64. Same lifetime as the list. Sends `Cache-Control: no-store`.

Path: `id` conversation id or open lane. `frame` the 12-character frame id.

Request: no body.

Response: `200`

```json
{ "mime": "image/png", "data": "iVBORw0KGgo=" }
```

- `data` is standard base64 of the image bytes. `mime` is the image type.
- When computer use is not available on this Core, the response is `200` with `{"frames":[]}`, not `404`.

Errors:

- `404` `session not found`
- `404` `frame expired` (unknown, older than 10 minutes, or already dropped)

Events: none.
