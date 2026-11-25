# Bot and chat channels

The Bot is one persistent assistant: a profile, a memory file, a main chat, optional side chats, one chat per connected channel kind, one Coordinator chat per project, and background tasks. These routes are how a frontend reads and changes that state. Ordinary conversations are a different list; a Bot chat (the main chat, a side chat, a channel chat, a project's Coordinator chat, a hidden task conversation, or a hidden worker's conversation) is omitted from it. Read chats and tasks from `GET /v1/bot`.

A successful body is `application/json`. An error body is `{"error":"<message>","failure":{"code":"core_request_failed","message":"<message>","http_status":<status>,"details":"<message>"}}`. Channel routes send `Cache-Control: no-store`.

`bot.changed` on `GET /v1/events` has no session id. Refetch `GET /v1/bot` after it, and `GET /v1/bot/memory` when the memory may have changed. It is sent when the profile, chat list, task list, supervision agenda, or memory changes, not when a read changes nothing.

## Profile

`GET /v1/bot` and `PUT /v1/bot` return the same object.

- `configured` (bool) is false until the first successful `PUT /v1/bot`, then stays true.
- `name` (string) is `Gyres Bot` until a save. A save stores the trimmed name.
- `description` (string) is built-in delegation guidance until the owner saves one. A stored description that is empty, or exactly `Your persistent local assistant.`, is filled with that guidance once, before any request. A save stores the trimmed description, including `""`, and a later empty save stays empty.
- `main_session_id` (string) is the current main chat, `""` while there is none. Saving the profile (`PUT /v1/bot`) creates it, and so does the first write that needs it (a side chat, a background task, a delivered result, a channel message). `GET /v1/bot` never creates it. Once `configured` is true there is always one. A main chat that was deleted or archived is replaced by a fresh one the same way; an archived one stays archived.
- `bot_provider_id` and `bot_model` (strings, always present) are the Bot model: the model of the main chat and each channel kind's current chat (including chats `/new` creates) and of the side chats branched from them. Both `""` means the Settings default, then the first model Core can use. Their own saved model is not used; `PUT /v1/sessions/{id}/settings` in one of them sets this pair. Any other conversation, a chat `/new` set aside included, runs on its own setting (see `unified_chats`). A project's Coordinator chats (with `project_id`) run on that project's Coordinator model, its lead Bot's Bot model (`coordinator_model` on the project), when set, else on this pair; `PUT /v1/sessions/{id}/settings` there sets the project's Coordinator model unless its `target` is `bot` or `task`, and an empty choice clears it. An external-agent pair is ignored as if it were unset. The first Core start after Bot models were merged fills an empty pair once from the main chat's saved model, else from the old `channel_provider_id`/`channel_model`, which are then dropped.
- `bot_reasoning_effort` and `task_reasoning_effort` (strings, always present) are the reasoning effort of the Bot model and of the task model. `""` is the model's default (no effort is sent). A task without a task model takes the Bot model with its effort. The first Core start after efforts were added copies the main chat's saved effort to an empty `bot_reasoning_effort` once, when that saved setting names the Bot model.
- `task_provider_id` and `task_model` (strings, always present) choose the model for a new background task. Both `""` means the Bot model, then Core's default model resolution. A non-empty pair is pinned for every new task: the fallback chain is not used. A pair that names an external agent is ignored at task creation as if it were unset.
- `system_prompt` (string) is the prompt every conversation that runs as the Bot receives (the pinned chats, the chats `/new` set aside and their lanes, and the supervisor; the current Coordinator appends its section). A plain conversation never gets it, configured or not (#2477, docs/bots.md "Plain conversations"). The string is a fixed introduction, then ` Your name is <name>.` when the name is non-empty, then a space and the description when that is non-empty, then a fixed style section. Memory is not part of this string.
- `chats` (array) is every Bot chat, oldest first. See Chats.
- `channel_chats` (object) maps a kind (`feishu`, `discord`) to the session id of the chat that kind talks to now. Kinds with no chat yet are absent. `{}` when none are bound.
- `chat_rotation` and `side_chat_branches` are always true: this Core accepts `POST /v1/bot/chats/{id}/new` and a `from` on `POST /v1/bot/chats`.
- `unified_chats` is always true: every conversation is one kind (#2198) and `GET /v1/sessions?all=1` lists them with their role (see the sessions reference, "Unified list"). `chats` stays as before: every Bot chat record, unchanged, for older clients. A chat that holds no role has its own model setting: Core wrote the model it followed there when it lost its role, and a side or parallel chat opened now starts on the Settings default. A chat set aside by `/new` stays its Bot's chat, with the Bot prompt, Bot memory and dispatch tools (#2481, docs/bots.md "Earlier chats"); a side or parallel chat of no Bot chat is a plain conversation (#2477): none of them.
- `tasks` (array) is newest first. See Tasks.
- `projects` (array) is every registered project in the order of `GET /v1/projects`, for a project list next to the Bot's chats: `id`, `name`, `available` (the project's directories can be used now), `running` (one of its open chats, the current Coordinator chat or a project chat without `ended`, holds a live turn), and when set `folderless`, `bot_hidden`, `coordinator_session_id` (the current Coordinator chat, absent until `POST /v1/projects/{project}/coordinator` created it) and `run_state` (as for a chat: the Coordinator chat's when it is not `idle`, else the first open chat's that is not `idle`, else `idle`). `[]` when none are registered.
- `agenda` (array, read only) is the Bot's supervision agenda: open items first, then settled ones, each group most recently updated first. See Agenda. `[]` when empty.
- `agenda_error` (string) is present only when the stored agenda could not be read at start. `agenda` is then `[]` and the Bot cannot change the agenda until the file is repaired or removed and Core restarts. The rest of the profile is unaffected.

A session id is UTC time `YYYY-MM-DDTHH-MM-SS`, then `_`, then a UUID. Example: `2026-10-14T12-02-00_3f1c2a90-7b6e-4d11-8a55-0c1e2d3f4a5b`.

External-agent provider ids, refused on `PUT /v1/bot`, are `local-codex`, `local-grok`, `local-pi`, `local-claude`, and `local-cursor`.

## Bot list

A Core has a list of Bots. The Bot every other section of this page describes is the **primary** Bot: it always exists, it is first in the list, and it cannot be deleted. `/v1/bot` and its sub-routes are the primary's. The other Bots are **specialists**: each has its own id, name, description (its persona), Bot model and task model, chats and memory file. A specialist's chats are omitted from `GET /v1/sessions` without `all=1` and from the archived list without `all=1`, as the primary's are; `GET /v1/sessions?all=1` lists them with `bot_id` and their role (see the sessions reference, "Unified list"), and `GET /v1/bots/{id}` keeps listing them in `chats`. A specialist's main chat branches side chats (`POST /v1/bot/chats` with `from` naming an entry of it; the record is in the specialist's `chats`), and `POST /v1/bot/chats/{id}/new` on it gives the specialist a fresh main chat; the chat set aside stays the specialist's, `ended`, on the model it ran on as its own setting. `PATCH` and `DELETE /v1/bot/chats/{id}` accept a specialist's side chats too. A plan may belong to a specialist (`bot_id`, see Schedules).

A conversation that is a specialist's chat runs as that specialist: its prompt is a fixed introduction naming the specialist, then a space and the description when non-empty, then the fixed style section; its context carries that specialist's memory, not the primary's, and the owner memory as every conversation does; it runs on the specialist's Bot model (when unset, the primary's Bot model, then the Settings default), and `PUT /v1/sessions/{id}/settings` there sets the specialist's Bot model (with `target` `task`, its task model). It runs only on configured providers, never on an external agent. A specialist has no background tasks, agenda or supervisor of its own: those stay with the primary. A specialist may have channel apps of its own (see [Channels](#channels)); each talks to its own channel chat of that specialist. Its task model runs the tasks other Bots hand to it and, as a project's lead, the workers it starts.

**Project binding.** A specialist works in at most one project (`project_id`); a project may have several Bots. The primary works outside projects and refuses a binding. A bound specialist's main chat is bound to the project like its Coordinator chat: the project's directories, instructions, memory, library and skills, and the project's list (`project_list`); nothing of other projects. `PUT /v1/bots/{id}` with another `project_id` moves the main chat to that project in place: the same conversation, with a fresh context from its next turn. Its workspace cannot be changed with `PUT /v1/sessions/{id}/workspace` (`409`).

**Project lead.** A project names at most one lead Bot (`lead_bot_id` on the project, see Projects), a specialist bound to it. The lead's main chat is the project's Coordinator chat: it keeps the project's `coordinator_session_id`, is listed with `GET /v1/bot`'s `chats` and `projects` and with the role `coordinator` in `GET /v1/sessions?all=1`, and its side chats are branched as before. The lead runs with the primary's introduction rewritten for a lead, its own name and description, the Coordinator section in its main chat, and the background tasks, radar and agenda tools; its tasks and agenda items are the Core's one list (`GET /v1/bot`'s `tasks` and `agenda`), they report back to the chat that started them and run on the lead's task model, else the primary's task model, else the lead's Bot model, else the primary's. A project's reports and notices that no chat claims (an agenda item of the project, a project Task's notice, a report whose requesting conversation is gone) go to its lead's main chat, and to the primary's main chat when it has none. A bound specialist that does not lead has no background tasks; other Bots can hand it tasks.

**Tasks handed to another Bot.** The primary and a project's lead can hand a background task to another Bot of the list. Such a task is one of `GET /v1/bot`'s `tasks` with `target_bot_id`. It runs in its own hidden conversation (`session_id`), never in the target's main chat, which stays as it was: with the target's prompt, memory and model (its task model, else its Bot model, else the primary's Bot model, else the Settings default), in the target's project when it is bound (`project_id`), unattended. Its outcome is reported to the chat that handed it over; when that chat is gone, to the handing Bot's main chat, then to the lead of the project it was handed from, then to the primary's main chat. Core refuses a hand to the handing Bot itself, to a Bot already on the work's chain since the owner's message, past two hands per chain, and past three per turn.

**Coordinators migrate.** The first start of a Core with the Bot list's projects turns each project's existing Coordinator chat into a specialist bound to the project and leading it: the chat is its main chat, named after the project, with the primary's description and a copy of the primary's memory file, and the project's Coordinator model as its Bot model. A hidden project migrates as well. A project that never opened a Coordinator chat, or whose chat is gone, gets its lead from `POST /v1/projects/{project}/coordinator`, which creates one the same way.

Each Bot in the list:

- `id` (string): `bot-` and 16 lowercase hex characters. The primary has one too; it never changes.
- `primary` (bool): true for the first Bot only.
- `created` (string, UTC RFC3339): when the Bot entered the list. For the primary of a Core that had a Bot before the list existed, the time the list was made.
- `configured`, `name`, `description`, `main_session_id`, `bot_provider_id`, `bot_model`, `bot_reasoning_effort`, `task_provider_id`, `task_model`, `task_reasoning_effort`, `chats`: as in Profile. A specialist is always `configured` and always has a main chat, created with it.
- `project_id` (string): the project a specialist is bound to; `""` for none and always for the primary.
- `lead_of` (string): present when the Bot is that project's lead.
- `error` (string): present only when this Bot's stored state could not be read at start; its changes are refused until the file is repaired and Core restarts.
- `system_prompt` (string): on `GET`, `PUT` and `POST` of one Bot only, not in the list.

## Chats

Each chat:

- `session_id` (string). For a branched side chat this is `<main session id>~sc_<12 lowercase hex>`, a lane of that main chat. It shares the main chat's history through `from_entry` and then continues alone.
- `title` (string). The main chat's stored title is `Main chat`. A channel chat's title is `Feishu` or `Discord`.
- `created` (string, UTC `YYYY-MM-DDTHH:MM:SSZ`).
- `lane` (bool) is present only when true (a branched side chat).
- `from_entry` (string) is present for a branched side chat: the main-chat message id it branched after.
- `ready` (bool) is present only when true. A branched chat is ready once its model snapshot has been copied. A successful branch returns it true.
- `channel` (string) is present for a chat created for a kind (`feishu` or `discord`). It stays after `/new` points the kind at a newer chat.
- `ended` (string, UTC `YYYY-MM-DDTHH:MM:SSZ`) is present after `/new` moved the main-chat pointer, the kind's binding or the project's Coordinator pointer off this chat, or after `PATCH /v1/bot/chats/{id}` closed a project's chat. The chat stays listed and usable; closing it with `closed: false` clears `ended`.
- `project_id` (string) is present on a project's Coordinator chat, on one `/new` set aside, on a project's parallel chat, and on a side chat branched from any of them. The project's current Coordinator chat is the `coordinator_session_id` in `projects`, not the absence of `ended`. A Coordinator chat's `title` is the project's name.
- `parallel` (bool) is present only when true: a project's chat opened next to its Coordinator chat (`POST /v1/bot/chats` with `project_id`). It is never a Coordinator chat and keeps its own `title`. A client shows a project's open chats (no `ended`) as tabs next to its Coordinator chat and the others as its closed chats.
- `untitled` (bool) is present only when true: the chat was opened without a title and `title` is the placeholder `New chat`. The owner's first message in the chat (`POST /v1/sessions/{id}/turns`) replaces it with that message's first line, shortened, and clears `untitled`; so does a rename.
- `running` (bool) is true while a live turn holds the chat, including while that turn is `paused` or `pausing`.
- `run_state` (string) while a live turn holds the chat is `running`, `pausing`, `paused`, or `waiting_children` (`running` that is waiting on child runs). When nothing is live but a task is still pending, it is the pending task's state as [`GET /v1/sessions/{id}/run-state`](runs.md#get-v1sessionsidrun-state) derives it: `paused`, `interrupted`, or `blocked`. Otherwise `idle`.

`settings` is never included. The profile's `chats` always include `running` and `run_state`. `POST /v1/bot/chats` without `from` and `project_id` returns the chat without those two fields.

## Tasks

`POST /v1/bot/tasks` returns a task as stored at creation. `GET /v1/bot` returns the same shape later, with status and result fields filled in.

- `id` (string) is `bot_` plus 32 lowercase hex characters.
- `session_id` (string) is the task's own conversation. A hidden task's conversation is a Bot chat and is omitted from the ordinary conversation list.
- `title` (string). An empty title is stored as `Background task`.
- `prompt` (string).
- `source_session_id` (string) is present when the request named one.
- `provider_id` and `model` (strings) are present when a model was pinned for this task.
- `model_pinned` (bool) is present and true when this task must not use the fallback chain.
- `status` (string) is `queued` in the create response, then `running`. Pause sets `paused`, resume sets `running`, stop sets `cancelled`. A finished run sets `completed` or `failed`.
- `result` (string) is present when `completed`: the last assistant text, or `Task completed. Open the task session for the full transcript.` when that text is empty.
- `error` (string) is present when `failed`. It is the run's error, or `task stopped` when the run was aborted and reported no error. A task already set to `cancelled` is left as `cancelled`.
- `created` and `updated` (strings, UTC `YYYY-MM-DDTHH:MM:SSZ`).
- `visible` (bool) is present and true when the task runs in an ordinary conversation the owner can open: a visible task, or a hidden worker. `POST /v1/bot/tasks` does not set it. Those tasks are not created by this route.
- `unlisted` (bool) is present and true on a hidden worker: a task the Bot started hidden but in a project or directory, on an external agent or on a named model (in a project's Coordinator chat, every hidden task, in that project). Its conversation is an ordinary one (no Bot prompt or tools; the Bot can send it more messages), yet it is omitted from `GET /v1/sessions`, archived included. List a project's workers with `GET /v1/projects/{project}/activity`; open one by its `session_id` as an ordinary conversation.
- `project_id` and `cwd` (strings) are present when the task's conversation works in a project or a directory.
- `adopted` (bool) is present and true when the Bot sent into an existing conversation it did not open. `prompt` is then `""`.
- `executor` (string) is present when a visible task or a hidden worker runs on an external agent: `codex`, `grok`, `pi`, `claude`, or `cursor`.
- `run_id` (string) is present once the run whose outcome is reported back exists.
- `delivered` (bool) is present and true once that outcome was handed to the requesting chat.
- `report` (bool) is present and true on a hidden task whose outcome is reported back to the requesting chat (every hidden task created since #1507).
- `reply` (object) is present when the request arrived from a channel: `channel` (kind), `chat_id`, and `message_id`. Absent means the request came from the app.
- `dispatcher_bot_id` (string) is present on tasks a Bot started since Bot-to-Bot dispatch: the id (`GET /v1/bots`) of the Bot whose conversation started it.
- `target_bot_id` (string) is present on a task handed to another Bot: that Bot's id. Its conversation is a hidden Bot session.
- `dispatch_chain` (array of strings) is present on a task handed to another Bot, and on a task started while handling one: the Bot ids the work passed through since the owner's message, in order, the target last.

## Agenda

The agenda is work the Bot supervises: what the owner delegated to it or asked it to watch. Only the Bot changes it, through its own tools in Bot chats; no route writes it. Each item stores the owner's intent and pointers to the conversations and Tasks doing the work, never copied transcripts.

- `id` (string) is `ag_` plus 16 lowercase hex characters.
- `title` (string), at most 200 characters.
- `intent` (string) is the owner's acceptance criteria and constraints, at most 4,000 characters.
- `targets` (array) of objects with `session_id` (an ordinary conversation, never a Bot chat) and/or `task_id` (a Task, a Bot task among them). A hidden Bot task is always a `task_id` target: given its conversation's `session_id`, Core stores its `task_id`. `[]` when none.
- `origin_session_id` (string) is the Bot chat the item was created from. Omitted when empty.
- `reply` (object) is present when the item was created from a channel message: `channel` (kind), `chat_id`, and `message_id`.
- `project_id` (string) is present when the item was created from a project's Coordinator chat (or a side chat branched from one). Its messages go to that project's current Coordinator chat, and a channel post about it starts with the project's name in brackets.
- `status` (string) is `active`, `waiting_owner` (parked until the owner answers), `completed`, or `cancelled`. The first two are open.
- `stage` (string) is where the work stands and the next planned step, at most 1,000 characters. Omitted when empty.
- `last_reported` (string) is what the owner was last told, at most 1,000 characters. Omitted when empty.
- `next_check_at` (string, UTC `YYYY-MM-DDTHH:MM:SSZ`) is the item's next follow-up. Omitted when unset.
- `precheck` (object) is present when set: `project_id` (a registered Project), and exactly one of `command` or `url` (`http` or `https`), each at most 2,000 bytes. A `command` runs in `project_id`'s folder, else in the working directory of the item's first target that has one; an item whose command has neither is refused. Core adds `every_seconds` (how far a check that found nothing new moves `next_check_at`; omitted for the default of 15 minutes), `last_hash` (the output's SHA-256 when the supervisor last looked), `skips` (checks since then that found nothing new) and `checked_at`, each omitted until set.
- `auto_steps` (int) counts the advances made without the owner since the owner last resumed the item. `max_auto_steps` (int, 1 through 50) is its limit; a new item gets the default.
- `daily_tokens` (int) is a daily token limit: the tokens the item's target conversations and Tasks used since local midnight, plus the supervisor's own (its current and kept sessions). Omitted when unset.
- `idle_wakes` (int) counts the follow-ups (`next_check_at` falling due) that woke the supervisor since it last advanced the item or the owner resumed it. Omitted when 0.
- `waiting_since` (string, UTC RFC3339 with fractional seconds) is when the item last became `waiting_owner`. Omitted until then.
- `created_at` and `updated_at` (strings, UTC `YYYY-MM-DDTHH:MM:SSZ`).

At most 20 items are open at once. Settled items beyond the 50 most recently settled are dropped, earliest settled first. A Bot turn the owner did not start (a background, scheduled, woken, or task-result turn) cannot raise `max_auto_steps`, raise or remove `daily_tokens`, change an item's `title` or `intent`, move a `waiting_owner` item out of waiting (resume, complete or cancel it), or reopen a settled one. When the owner resumes an item, `auto_steps` and `idle_wakes` return to 0. One exception lets such a turn resume, complete or cancel a `waiting_owner` item: Core sees the owner already dealt with it at a target after `waiting_since` — an owner message (not one the Bot sent, not a scheduled run's prompt, not a wake or goal continuation) or an approval the owner gave in a target conversation, a target conversation archived, or a target Task `complete`, `completed` or `cancelled`. It does not reset `auto_steps`.

### Supervisor

Core runs the agenda in the background in one hidden Bot session, the supervisor. It is not in `chats`, the conversation list, the archived list or search (neither the app's nor a model's `session_search`), and neither are the sessions it rotated out of (up to 5 are kept; older ones are deleted, their usage records kept). `DELETE /v1/sessions/{id}` refuses them. The supervisor only coordinates: its turns have the Bot's radar, agenda, task (list, create, send), outbox and wait tools and `session_search`, and no tool that writes files, runs commands, controls the desktop, asks the owner a question, sets goals or schedules, manages skills, starts subagents or shows anything. The supervisor runs one unattended turn at a time on a batch of events: a Bot task or a run in a target conversation settled, a target Task needs attention (blocked, capped, usage or budget limited, or an approval waits; such a Task gets no static notice in the main chat), an item's `next_check_at` fell due, or the owner resumed an item. An outcome or Task notice that no active item watches goes to the chats as before. Events and batches are kept in the claim log, so a restart neither loses nor repeats one.

Each `bot_task` create or send of the supervisor names the item it advances and counts one `auto_steps`; a send reaches only the item's targets and is refused while the owner is active in that conversation (a live run the owner started, or an owner message in the last 3 minutes). At `max_auto_steps`, or when `daily_tokens` is used up, Core refuses, sets `waiting_owner` and tells the owner itself. A follow-up that would wake the supervisor counts too: once `idle_wakes` reaches `max_auto_steps` (follow-ups without an advance), or `daily_tokens` is used up, Core sets `waiting_owner` and tells the owner instead of waking it.

The supervisor's single wake (source `bot_supervisor`, see the runs reference) follows the earliest `next_check_at` of the active items. When it fires, each due item's pre-check runs first: an unchanged output moves the item on by `every_seconds` without a model request. A command pre-check runs unattended: a command that only reads files (one of `cat`, `head`, `tail`, `ls`, `wc`, `stat`, `pwd`, written as plain words without shell syntax) runs without approval; `git` is not among them, since a repository can configure programs that even its reading commands run; any other needs a grant or the unattended `allow` policy. A pre-check that fails wakes the supervisor with the failure in the event's `error`.

The supervisor reaches the owner only through its outbox: the message is recorded in the item's Bot chat (the main chat when that chat is gone), as a delivered record like a scheduled result, and posted to at most one channel (see Notices): the item's `reply` channel, else the one `notify_target` names. An `escalate` message also sets `waiting_owner`; a `complete` one completes the item and sets `last_reported`. Messages are kept in `<data>/bot/outbox.json` until delivered, so a restart finishes a delivery without recording or posting it twice; a post interrupted mid-send is not retried.

A supervisor run that sends nothing through the outbox leaves one display-only row at the end of the main chat: a user-role record with `metadata.internalMessage` `supervisorNotice` and `metadata.supervision` (`runId`, `supervisorSessionId`, and when known `agendaIds`, `targetSessionIds`, `triggers`, `tools`, `status`, `inputTokens`, `outputTokens`, `startedAt`, `finishedAt`; `summary` may be empty). The row never reaches the model and starts no turn; its text is only for a client that does not know it. Clients show consecutive rows as one line and open the runs through `GET /v1/bot/supervisor/runs`.

In a Bot chat turn the owner started, a `bot_task` `send` that succeeds into a target of a `waiting_owner` item also resumes that item (`active`, `auto_steps` 0) and wakes the supervisor, as resuming it does. A turn the owner did not start, and the supervisor, never resume an item this way.

A Bot chat's `bot_task` `create` may name an open item (`agenda_id`): the new task joins the item's targets before its run starts (a visible task's conversation, a hidden task's id), so its outcome goes to the supervisor however fast it settles; without one, its outcome comes back to the chat as before. A `waiting_owner` item takes such work only in a turn the owner started, which resumes it (`active`, `auto_steps` 0).

A `bot_task` `send` into a conversation whose run waits on an approval queues the message and also answers `blocked_on_approval: true`, `pending_approvals` (the waiting tools' labels) and `warning`: the run reads the message only after a person decides the approval. The Bot cannot decide approvals.

After `/new`, a report for a chat it set aside (a task outcome, a supervisor message) goes to the slot's current chat: the main chat's, or that channel kind's current chat, which still answers the channel message.

### Notices

Core's own notices to the owner are never broadcast. Each is recorded in a Bot chat and posted to at most one channel: supervisor messages, needs-attention notices of Tasks, and idle reminders. Channels here are channel apps (instances, see Channel instances), never just a kind. Work that came from a channel app goes back to that app: a Task's notice goes to the app the task was asked from (a Bot task's reply, or the channel chat it was started from or runs in), also when that app talks to a specialist. Everything else follows `notify_target`: `last_active` (the default) posts to the app bound to the primary that the owner last wrote from, a channel kind (`feishu`, `discord`) pins the primary's app of that kind, and `app_only` posts nowhere. A target that is switched off, removed, bound to another Bot since, not configured or has no owner yet keeps the notice in the app (the main chat) only; no other app of the kind stands in. A post is also recorded, once, in that app's current Bot chat, so a reply there has it in context.

A post about work that waits on a person's approval (an idle reminder for an approval, a supervisor `escalate` about an item whose target conversations or Tasks wait on one, a Task's approval notice) is followed by each waiting approval's own card in the owner's direct chat of that kind, the same card a channel turn shows: at most one card per approval and kind. Deciding the approval on any card or in the app updates every card of it; a card for an approval already decided or gone shows as expired when pressed.

Idle reminder: when one of the owner's ordinary conversations (not a Bot chat, a subagent, an archived conversation, a target of an open agenda item, or the open attempt of an unsettled Task, which get Task notices) waits on a tool approval, stopped `interrupted` or `blocked`, or ended `failed` (its `unseen` is `failed`, from a run the owner or an external agent started), and nobody has looked at it for `idle_nudge_minutes`, Core sends one reminder: a record in the main chat and one post as above. Looking is an owner message there or `POST /v1/sessions/{id}/seen` within the last 2 minutes; the wait counts from the later of the condition starting and the last look. A condition older than 24 hours is not reminded. Each condition of a run is reminded once, also across restarts. The records name the conversation (`[Session ID: <id>, Title: <title>]`) and carry `metadata.targetSessionId`. Core never approves, sends or changes anything in that conversation.

## Memory

`content` is the whole memory file, raw. A missing file is `""`.

A line `<!-- gyres:bot-rules -->` that starts the text or follows a newline, and ends the text or is followed by a newline, splits the file. Text before it, trailing whitespace removed, is the owner's notes. Text after it, trimmed, is the Bot-written rules. With no such line, the whole file is the owner's notes.

Rules are entries joined by a blank line, `---`, and a blank line. An entry the Bot writes is the rule, then a newline, then `— `, then the Core host's local date `YYYY-MM-DD`, then ` · `, then `Web`, `Feishu`, or `Discord`.

`version` is the SHA-256 of the raw file bytes as 64 lowercase hex characters, or `""` when `content` is empty. Send it back on `PUT` to avoid overwriting a newer file.

`changes` is the Bot's own writes, newest first, at most the last 200. An owner save does not add one. Each change:

- `id` (string), 16 lowercase hex characters.
- `at` (string, UTC RFC3339).
- `op` (string): `add`, `replace`, or `remove`.
- `source` (string): `Web`, `Feishu`, or `Discord`.
- `session_id` (string), the conversation the write came from, omitted when empty.
- `old` (string), the full entry before a `replace` or `remove`, omitted otherwise.
- `new` (string), the full entry after an `add` or `replace`, omitted otherwise.
- `reason` (string), omitted when empty.
- `undone_at` (string, UTC RFC3339), omitted until this change is undone. Undone changes stay in the list.

`changes` is always an array. `[]` when there are none.

## Channels

Channel routes are administrator only. Kinds, in list order, are `feishu` and `discord`. Credentials are never returned: no Feishu app secret and no Discord token.

**Channel instances.** Each connected app (a Feishu or Lark app, a Discord bot) is a channel instance with an `id`, a `kind` and the Bot it talks to (`bot_id`, an id from `GET /v1/bots`). A Core may have several instances of one kind, but each Bot has at most one instance of each kind, and one app (Feishu `app_id`, Discord application) is connected on one instance only. An instance created without a Bot talks to the primary. A message on an instance becomes a user message in that Bot's channel chat of the instance's kind: one persistent chat per instance, separate from the Bot's main chat, with the Bot's identity, memory, skills and approval rules. Its reply, progress, approval cards and files go back only to the instance the message came from, and only that instance's owner is answered. `/new` in the app, or `POST /v1/bot/chats/{id}/new` on that chat, starts a fresh channel chat for that instance only; the one set aside stays the Bot's chat (listed in its `chats`, `ended` set). Moving an instance to another Bot stops its running turn, drops the messages waiting behind it and sets the old Bot's channel chat aside; the new Bot's chat starts with the next message. A report to the old Bot's chat no longer posts on the moved app. Removing an instance deletes its credentials; its Bot keeps the channel chat for a later app of the kind. A new instance of a kind for the primary takes the kind as its id while no instance has it and that id was not removed while it talked to a specialist; any other new instance gets `<kind>-<8 hex>`, so results and notices meant for a removed app never reach another Bot's app. A scheduled plan names the instance it delivers to (see the schedules routes) and follows it to another Bot; notices go to the instance their work came from, else as `notify_target` says (see Notices).

The first instance of a kind has the kind as its `id` (`feishu`, `discord`); later ones have the kind followed by `-` and 8 hexadecimal characters. A Core from before instances turns its Feishu app and its Discord bot into one instance each, bound to the primary, keeping their chats.

`/v1/bot/channels/{kind}` addresses the primary's instance of that kind, as before instances; `/v1/bot/channel-instances/{id}` addresses any instance. Both take the same bodies and return the same objects.

`GET /v1/bot/channels` and `PUT /v1/bot/channels` return:

- `display.settings`, the stored global display object. Unset fields are omitted, so `{}` means nothing is stored. A field that is present is set, including `false` and `0`.
- `display.builtin`, what an unset field resolves to when the kind has no built-in of its own. See the values below.
- `channels`, one object per kind: the primary's instance of that kind, or the kind not set up.
- `feishu`, the same object as the `feishu` element of `channels`.
- `instances`, one object per channel instance, in the order they were created.

Each channel object:

- `id`, the instance. Omitted while the kind has no instance for the primary.
- `kind` (`feishu` or `discord`).
- `bot_id`, the Bot the instance talks to. Omitted with `id`.
- `chat_session_id`, the instance's current channel chat on that Bot. Omitted before its first message.
- `configured` (bool). False deletes nothing about display: a display override outlives the credentials.
- `enabled` (bool). False when not configured. When configured, false means the credentials are kept and the connection is down.
- `owner_bound` (bool). True after an owner is bound.
- `claim_code` (string, 6 digits, leading zeros kept) is present after a manual setup until the owner claims it. The owner sends a direct message whose text, with surrounding whitespace removed, is exactly this code. The first such sender is the owner. Other messages before that are ignored. After a claim, `owner_bound` is true and `claim_code` is omitted. Scan-to-create binds the scanning user and has no claim code.
- `state` (string) is present while a connection is up: `connecting`, `connected`, `reconnecting`, or `error`. Omitted when the kind is not configured or is switched off. A new connection starts at `connecting`.
- `error` (string) is present when the connection reports one.
- `display` is that instance's display view (the kind's when it has no instance): `override` (stored instance override, `{}` when none), `inherited` (resolved with no override), and `effective` (resolved with the override). `inherited` and `effective` always include every display field.
- Feishu, once configured: `brand` (`feishu` or `lark`), `app_id`, `bot_name`.
- Feishu, while a scan is in progress or has failed and not been cleared: `registration`. `state` is `waiting` or `failed`. While waiting, `url` is the link to show as a QR code and `expires_at` is when it expires (RFC3339, Core host offset), both omitted until the link is known. On failure, `error` is set and `url` is omitted.
- Discord, once configured: `bot_name`, `application_id`, and `invite_url`. `invite_url` is the link that adds the bot to a server the owner is in, which is what lets them open a direct message. Its query has `client_id` set to the application id, `scope` `bot`, `permissions` `67584`, and `integration_type` `0`. Omitted when there is no application id.

Display fields. A value resolves, first one set wins, from the kind override, the global setting, the kind's built-in, then the global built-in. `tool_progress_command` and `followup_grace_seconds` skip the kind override and the kind built-in.

| Field | JSON type | Allowed | Global built-in | Kind built-in |
| --- | --- | --- | --- | --- |
| `tool_progress` | string | `off`, `new`, `all`, `verbose`, `log` | `all` | Feishu `new`, Discord `all` |
| `tool_progress_grouping` | string | `accumulate`, `separate` | `accumulate` | none |
| `show_reasoning` | bool | | false | false |
| `reasoning_style` | string | `code`, `blockquote`, `subtext` | `code` | Discord `subtext` |
| `interim_assistant_messages` | bool | | true | true |
| `long_running_notifications` | bool | | true | true |
| `cleanup_progress` | bool | | false | none |
| `tool_preview_length` | int | 0 through 100000 | 0 | Feishu and Discord 40 |
| `tool_progress_command` | bool | global only | false | none |
| `busy_input_mode` | string | `interrupt`, `steer`, `queue` | `interrupt` | none |
| `followup_grace_seconds` | number | 0 through 60 | 3 | none |

`tool_progress` is how much tool activity a channel shows: `off` none, `new` when the tool changes, `all` every call, `verbose` every call with its arguments, `log` nothing in the chat. `accumulate` edits one progress message; `separate` sends one message per line. `tool_progress_command` lets the owner send `/verbose` in the channel, which moves that kind's `tool_progress` one step along `off`, `new`, `all`, `verbose`, `log`, then back to `off`, and saves the step as the kind's override. `busy_input_mode` is what a direct message does while that kind's chat is busy: `interrupt` stops the running turn (it waits instead when that chat has subagents or is compacting), `steer` adds the message to the running turn or waits if that is refused, `queue` waits. `followup_grace_seconds` is how long after a turn starts a new message waits instead of interrupting; `0` turns that off.

`display.builtin` is the global built-in column: `tool_progress` `all`, `tool_progress_grouping` `accumulate`, `show_reasoning` false, `reasoning_style` `code`, `interim_assistant_messages` true, `long_running_notifications` true, `cleanup_progress` false, `tool_preview_length` 0, `tool_progress_command` false, `busy_input_mode` `interrupt`, `followup_grace_seconds` 3.

A `PUT` of `display` replaces the whole stored object. A field left out is cleared, not kept. `{"display":{}}` clears it. Unknown fields are rejected.

### GET /v1/bot

Auth: any key
Effect: Returns the Bot profile. It creates nothing: before the Bot is saved, `main_session_id` may be `""` and `chats` empty. When a main chat exists, the first read in this process also finishes or drops side chats left half-created or half-deleted.

Request: no body.

Response: `200`

```json
{
  "configured": true,
  "name": "Research Bot",
  "description": "Keeps long-running work moving.",
  "main_session_id": "2026-10-14T12-02-00_3f1c2a90-7b6e-4d11-8a55-0c1e2d3f4a5b",
  "task_provider_id": "",
  "task_model": "",
  "bot_provider_id": "",
  "bot_model": "",
  "bot_reasoning_effort": "",
  "task_reasoning_effort": "",
  "chats": [
    {
      "session_id": "2026-10-14T12-02-00_3f1c2a90-7b6e-4d11-8a55-0c1e2d3f4a5b",
      "title": "Main chat",
      "created": "2026-10-14T12:02:00Z",
      "running": false,
      "run_state": "idle"
    }
  ],
  "channel_chats": {},
  "chat_rotation": true,
  "unified_chats": true,
  "side_chat_branches": true,
  "tasks": [],
  "system_prompt": "You are the user's persistent local Gyres Bot.",
  "agenda": [],
  "notify_target": "last_active",
  "notify_channel": "feishu",
  "idle_nudge_minutes": 10
}
```

`notify_target` is where notices without a channel of their own go (see Notices): `last_active`, `app_only` or a channel kind. `notify_channel` is the channel app id it resolves to now (an app id such as `feishu-1a2b3c4d`, not only a kind), `""` when notices stay in the app. `idle_nudge_minutes` is the idle reminder's wait, `0` when reminders are off; the default is 10.

`system_prompt` above is only the opening sentence. The field is the full assembled prompt: that introduction, then ` Your name is Research Bot.`, a space, the description, then the fixed style section. Before any save, `configured` is false, `name` is `Gyres Bot`, and `description` is the built-in guidance.

Errors:

- `500` the Bot state file is damaged (the message names it), or a half-finished side chat could not be repaired

Events: `bot.changed` when a repair changes the chat list. A read that changes nothing emits nothing.

### PUT /v1/bot

Auth: any key
Effect: Saves the name and description, and optionally the task model or the Bot model. The description sent here is kept as sent, including empty. Also creates the main chat if it does not exist yet, before the profile is saved, so a configured Bot always has one.

Request:

```json
{
  "name": "Research Bot",
  "description": "Keeps long-running work moving.",
  "task_provider_id": "provider-b",
  "task_model": "model-b"
}
```

- `name` (string, required). Trimmed. Empty or whitespace-only is rejected. The body is one JSON object of at most 16384 bytes.
- `description` (string). Trimmed. Omitted is `""` and clears a description already saved. Send the current description back when a save should keep it.
- `task_provider_id` and `task_model` (strings). Omit both to leave the task model unchanged. Send both as `""` to clear it. Send both non-empty to pin it. Any other combination is rejected, including one empty and one non-empty. The same rule applies to `bot_provider_id` and `bot_model`.
- `bot_reasoning_effort` and `task_reasoning_effort` (strings, optional). Omitted keeps the effort while the model stays the same and clears it when the same body changes that model. `""` is the model's default. A non-empty value must be one of the model's `reasoning_efforts` (`GET /v1/models`) and needs that model set.
- `notify_target` (string, optional): `last_active`, `app_only`, or a channel kind of this Core. Omitted keeps the stored value.
- `idle_nudge_minutes` (int, optional): 0 through 1440; `0` turns idle reminders off. Omitted keeps the stored value.
- A non-empty provider id of `local-codex`, `local-grok`, `local-pi`, `local-claude`, or `local-cursor` is rejected. Other fields are ignored.

Response: `200` the profile, same shape as `GET /v1/bot`, after the save and after the main chat exists.

Errors:

- `400` bot name is required. Also for a body that is not JSON or is larger than 16384 bytes.
- `400` task_provider_id and task_model must be set together
- `400` bot_provider_id and bot_model must be set together
- `400` unknown or unavailable model (a Bot model that is not available now)
- `400` unsupported reasoning_effort for this model
- `400` reasoning_effort needs a model
- `400` Bot conversations run on a configured provider, not an external agent
- `400` notify_target must be last_active, app_only or a channel kind
- `400` idle_nudge_minutes must be 0 through 1440
- `500` the profile could not be saved, or the main chat could not be created. A failure while creating the main chat leaves the profile unsaved.

Events: `bot.changed`. Also `sessions.changed` when this call creates the main chat.

### GET /v1/bot/memory

Auth: any key
Effect: Returns the memory file, its version, and the Bot's change log. Changes nothing.

Request: no body.

Response: `200`

```json
{
  "content": "Prefer concise research notes.",
  "version": "ab3dde22904e89dda9261e2d905489478b3d23c55d3bb8612fcd3200fe0cbd1b",
  "changes": []
}
```

`version` above is the SHA-256 of that exact `content`. An empty file uses `""`. See Memory for `content` and `changes`.

Errors:

- `500` the change log could not be read, including when it is corrupt (`change log is corrupt: …`)

Events: none.

### PUT /v1/bot/memory

Auth: any key
Effect: Replaces the whole memory file with `content`. Does not add a change-log entry. Bot conversations pick the new text up on their next turn.

Request:

```json
{
  "content": "Prefer concise research notes.",
  "version": "ab3dde22904e89dda9261e2d905489478b3d23c55d3bb8612fcd3200fe0cbd1b"
}
```

- `content` (string). Stored exactly, including an empty string. The body is one JSON object of at most 262144 bytes. This save is not limited to 3000 characters; that limit applies to the Bot-written section when the Bot writes it and when an undo would grow it.
- `version` (string, optional). When present, it must equal the current version or the save is refused and the file is left unchanged. Omit it, or send JSON `null`, to save unconditionally. `""` matches only an empty file.
- A client that edits the owner's notes and wants to keep the Bot-written rules must send the marker line and the rules after it as part of `content`.

Response: `200` the memory view, same shape as `GET /v1/bot/memory`, with the new `version`.

Errors:

- `400` invalid json. Also when the body is larger than 262144 bytes.
- `409` Bot memory changed; reload before saving
- `500` the file could not be written, or it was written but the view could not be read back (a corrupt change log). In the second case the new content is already stored and `bot.changed` was already sent.

Events: `bot.changed` after a successful write, including when the following read of the change log fails.

### POST /v1/bot/memory/changes/{id}/undo

Auth: any key
Effect: Reverts one Bot write in the rules section and marks that change undone. The owner's notes are left as they are. Undoing an add removes that entry. Undoing a replace puts the old entry back. Undoing a remove appends the old entry.

Path: `id` change id from `changes`.

Request: no body.

Response: `200` the memory view, same shape as `GET /v1/bot/memory`. `content` and `version` match the reverted file. The undone change stays in `changes` with `undone_at` set.

Errors:

- `404` change not found
- `409` this change was already undone
- `409` cannot undo: a later change touched this rule; undo that one first. A later change touches this one when either entry text (`old` or `new`, when non-empty) equals the other's.
- `409` cannot undo: the rule was changed or removed since. An add or replace whose `new` entry is no longer the only exact copy, and a replace whose `old` entry is already present.
- `409` cannot undo: the rule is already there. A remove whose `old` entry is already present.
- `409` cannot undo: unknown change op
- `409` cannot undo: the Bot-written section would exceed 3000 characters; merge or remove old rules first (replace/remove), then retry
- `500` the file or the change log could not be read or written, including a corrupt change log

Events: `bot.changed` on success.

### GET /v1/bots

Auth: any key
Effect: Returns every Bot, the primary first, then the specialists in the order they were created. Creates nothing.

Request: no body.

Response: `200`

```json
{
  "bots": [
    {
      "id": "bot-4f1c2a907b6e4d11",
      "primary": true,
      "created": "2026-11-25T03:30:33Z",
      "configured": true,
      "name": "Research Bot",
      "description": "Keeps long-running work moving.",
      "main_session_id": "2026-10-14T12-02-00_3f1c2a90-7b6e-4d11-8a55-0c1e2d3f4a5b",
      "bot_provider_id": "",
      "bot_model": "",
      "bot_reasoning_effort": "",
      "task_provider_id": "",
      "task_model": "",
      "task_reasoning_effort": "",
      "chats": [],
      "channel_chats": [
        {"instance_id": "feishu", "kind": "feishu", "session_id": "2026-10-14T12-05-00_9a0b1c2d-3e4f-4a5b-8c6d-7e8f9a0b1c2d"}
      ]
    }
  ]
}
```

`channel_chats` lists the channel apps bound to that Bot (see [Channels](#channels)), each with its current channel chat on the Bot. `session_id` is empty until the app's first message. List these chats under the Bot; a chat that an earlier `/new` or a move set aside is in `chats` with `ended` set.

Errors:

- `500` the stored Bot list cannot be used (damaged, written by a newer Core, or missing while specialists exist). The message names the file. The primary keeps working through `/v1/bot`.

Events: none.

### POST /v1/bots

Auth: any key
Effect: Creates a specialist Bot with its main chat.

Request: the body of `PUT /v1/bot`, with the same rules for `name`, `description` and the model fields, and optionally `project_id` (string, a registered project's id; see Project binding). `notify_target` and `idle_nudge_minutes` are the primary's and are ignored here.

```json
{
  "name": "Release Writer",
  "description": "Drafts release notes from merged pull requests.",
  "bot_provider_id": "provider-a",
  "bot_model": "model-a"
}
```

Response: `201` the new Bot, with `system_prompt`.

Errors:

- `400` the same errors as `PUT /v1/bot` for the fields above
- `400`, `409` the project in `project_id` is not registered, or its primary directory cannot be used; the Bot is created, unbound, and the message names it
- `500` the Bot list cannot be used, or the Bot or its main chat could not be saved. Nothing is left behind.

Events: `bot.changed`, `sessions.changed`.

### GET /v1/bots/{id}

Auth: any key
Effect: Returns one Bot, with `system_prompt`. Creates nothing.

Path: `id` a Bot id from `GET /v1/bots`.

Response: `200` the Bot.

Errors:

- `404` bot not found
- `500` the Bot list cannot be used, or this Bot's state file is damaged

Events: none.

### PUT /v1/bots/{id}

Auth: any key
Effect: Saves a Bot's profile. For the primary this is `PUT /v1/bot`, with the same body and effects. For a specialist it saves the name, description and models, and first gives it a fresh main chat when its main chat was deleted or archived.

Request: the body of `PUT /v1/bot`. A specialist ignores `notify_target` and `idle_nudge_minutes`. `project_id` (string, optional) binds a specialist to that project, `""` to none; omitted keeps it. A lead bound elsewhere stops leading its earlier project first.

Response: `200` the Bot, with `system_prompt`.

Errors:

- `400` the same errors as `PUT /v1/bot`
- `400` the primary Bot works outside projects (a non-empty `project_id` for the primary)
- `400` project not found (`project_id`)
- `404` bot not found
- `409` the project's primary directory cannot be used, or the Bot's main chat is running (stop it first)
- `500` the Bot could not be saved

Events: `bot.changed`; `sessions.changed` when a main chat is created.

### DELETE /v1/bots/{id}

Auth: any key
Effect: Deletes a specialist: a project's lead stops leading it first (the project has no lead then), then each of its chats is archived, then removes it from the list with its state and memory file. Its archived chats stay in the archive as ordinary conversations; a conversation unarchived from there no longer runs as that Bot.

Path: `id` a specialist's id.

Response: `200`

```json
{ "id": "bot-0a1b2c3d4e5f6071" }
```

Errors:

- `404` bot not found
- `409` the primary Bot cannot be deleted
- `409` a chat of this Bot is running; stop it first
- `500` the list could not be saved

Events: `bot.changed`, `sessions.changed`.

### GET /v1/bots/{id}/memory

Auth: any key
Effect: `GET /v1/bot/memory` for this Bot: its own memory file and change log. For the primary it is the same file as `/v1/bot/memory`.

Path: `id` a Bot id.

Response: `200` the memory view (see Memory).

Errors:

- `404` bot not found
- `500` as `GET /v1/bot/memory`, or the Bot list cannot be used

Events: none.

### PUT /v1/bots/{id}/memory

Auth: any key
Effect: `PUT /v1/bot/memory` for this Bot. That Bot's chats pick the new text up on their next turn.

Path: `id` a Bot id.

Request: as `PUT /v1/bot/memory`.

Response: `200` the memory view.

Errors:

- `400`, `409`, `500` as `PUT /v1/bot/memory`
- `404` bot not found

Events: `bot.changed`.

### POST /v1/bots/{id}/memory/changes/{change}/undo

Auth: any key
Effect: `POST /v1/bot/memory/changes/{id}/undo` for this Bot's change log.

Path: `id` a Bot id; `change` a change id from that Bot's `changes`.

Request: no body.

Response: `200` the memory view.

Errors:

- `404` bot not found, or change not found
- `409`, `500` as `POST /v1/bot/memory/changes/{id}/undo`

Events: `bot.changed` on success.

### POST /v1/bot/chats

Auth: any key
Effect: Opens a side chat. With `from`, the chat is a lane of the current main chat (or of a project's open chat), sharing history through that message. With `project_id`, the chat is a project's parallel chat: a new empty Bot chat bound to the project like its Coordinator chat (the project's directories, instructions, memory, library and skills, the Bot's prompt and tools, the project's Coordinator model), in its own conversation directory, with `parallel` true. Otherwise the chat is a new empty conversation of its own.

Request:

```json
{ "title": "Research thread" }
```

```json
{ "project_id": "p_3f1c2a90" }
```

```json
{
  "title": "Research thread",
  "from": {
    "session_id": "2026-10-14T12-02-00_3f1c2a90-7b6e-4d11-8a55-0c1e2d3f4a5b",
    "entry_id": "msg_3f1c2a90-7b6e-4d11-8a55-0c1e2d3f4a5b"
  }
}
```

- `title` (string, optional). Trimmed. Omitted or empty, the chat gets the placeholder title `New chat` with `untitled` true, and the owner's first message names it. The body is one JSON object of at most 4096 bytes.
- `from` (object, optional). Omit it, or send JSON `null`, for an empty side chat. `session_id` must be the current main chat (`main_session_id`), not an earlier main chat, or an open chat of a registered project that is not itself a branch: its current Coordinator chat, a parallel chat, or an earlier Coordinator chat opened again, none of them with `ended`. A branch of a project's chat has that project's `project_id` and works in the project.
- `project_id` (string, optional). Ignored when `from` is given. A registered project's id; the chat opens as that project's parallel chat. A project without folders gets its library created first. `entry_id` must be non-empty and must be the assistant message that ends a finished turn on the main chat's current path: not inside a turn that is still running, not a turn whose tool call has no result yet, and not a message that is not that final assistant reply.
- Other fields are ignored.

Response: `200`

An empty side chat:

```json
{
  "session_id": "2026-10-14T12-05-00_3f1c2a90-7b6e-4d11-8a55-0c1e2d3f4a5b",
  "title": "Research thread",
  "created": "2026-10-14T12:05:00Z"
}
```

A branched side chat:

```json
{
  "session_id": "2026-10-14T12-02-00_3f1c2a90-7b6e-4d11-8a55-0c1e2d3f4a5b~sc_6f1a2b3c4d5e",
  "title": "Research thread",
  "created": "2026-10-14T12:05:00Z",
  "lane": true,
  "from_entry": "msg_3f1c2a90-7b6e-4d11-8a55-0c1e2d3f4a5b",
  "ready": true,
  "running": false,
  "run_state": "idle"
}
```

A project's parallel chat opened without a title:

```json
{
  "session_id": "2026-10-14T12-07-00_3f1c2a90-7b6e-4d11-8a55-0c1e2d3f4a5b",
  "title": "New chat",
  "created": "2026-10-14T12:07:00Z",
  "project_id": "p_3f1c2a90",
  "parallel": true,
  "untitled": true,
  "running": false,
  "run_state": "idle"
}
```

`session_id` of a branch is the main session id, `~sc_`, and 12 lowercase hex characters.

Errors:

- `400` invalid json. The body is not JSON or is larger than 4096 bytes.
- `400` a side chat branches from an entry of the Bot main chat or an open chat of a Project
- `404` project not found (`project_id`)
- `409` the project's primary directory is missing or cannot be used (`project_id`); nothing is created
- `400` message `<entry_id>` is not in the main conversation
- `409` the main chat is busy; try again
- `409` this turn is still running; branch from an earlier turn
- `409` this turn has not finished; branch from a finished turn
- `409` branch from the end of a turn
- `503` storage unavailable; reconnect and retry
- `503` Core is moving its data directory; try again when it finishes
- `500` the chat could not be created. If copying the branched chat's model setting fails after the chat was recorded, the record stays and the next `GET /v1/bot` finishes it.

Events: an empty side chat or a parallel chat emits `sessions.changed` for the new conversation and `bot.changed`. It does not create the main chat. A branch emits `session.messages` for the new lane and `bot.changed`. A branch that has to create the main chat also emits `sessions.changed` and `bot.changed` for that chat.

### POST /v1/bot/chats/{id}/new

Auth: any key
Effect: Sets aside the current main chat, the chat a channel kind is bound to now, or a project's current Coordinator chat, and binds a fresh chat with the same model setting and no history. A fresh Coordinator chat is bound to the same project and becomes its `coordinator_session_id`; the memory review of the old one writes that project's memory, and task reports and supervisor messages meant for the old one go to the new one. The old chat stays listed, with `ended` set, and remains usable. Its memory review is queued, immediately if it is idle, otherwise once its current turn finishes. This route does not stop that turn and does not drop messages waiting on a channel.

Path: `id` session id of the current main chat, of the chat a kind is bound to now (`channel_chats`), or of a project's current Coordinator chat.

Request: no body.

Response: `200` the new chat, with `running` false and `run_state` `idle`.

```json
{
  "session_id": "2026-10-14T12-06-00_3f1c2a90-7b6e-4d11-8a55-0c1e2d3f4a5b",
  "title": "Main chat",
  "created": "2026-10-14T12:06:00Z",
  "running": false,
  "run_state": "idle"
}
```

A new main chat has title `Main chat` and `main_session_id` moves to it. A new channel chat has title `Feishu` or `Discord`, `channel` `feishu` or `discord`, and `channel_chats` for that kind moves to it. Refetch `GET /v1/bot` for the updated pointers and the old chat's `ended` time.

Errors:

- `409` only the main chat, a channel's current chat or a Project's Coordinator chat can start a new chat. Also when this chat was already rotated by a concurrent `/new`.
- `409` the project's primary directory is missing or cannot be used, or the project's Coordinator chat changed meanwhile
- `500` the new chat could not be created, including `the chat changed meanwhile; try again` if the binding moved during the write

Events: `sessions.changed` for the new conversation, and `bot.changed`. Also `sessions.changed` for the main chat when this call creates it.

### PATCH /v1/bot/chats/{id}

Auth: any key
Effect: Closes a project's chat, opens it again, or renames a Bot chat. Closing sets `ended` to now: a client lists the chat with the project's closed chats instead of its tabs, task reports for it go to the project's current Coordinator chat, and its memory review is queued (writing the project's memory), immediately if it is idle, otherwise once its current turn finishes. Closing stops nothing and deletes nothing; the chat stays usable. `closed: false` clears `ended`, so the chat is one of the project's open chats again and receives its own reports. A title renames the chat and clears `untitled`.

Path: `id` session id of a chat in `chats`.

Request:

```json
{ "closed": true }
```

```json
{ "title": "Launch email" }
```

- `closed` (boolean, optional). Only a chat with `project_id` can be closed or opened again. Closing a chat that is already closed, or opening one that is open, changes nothing.
- `title` (string, optional). Whitespace is collapsed; 1 to 100 characters. The current main chat cannot be renamed.
- At least one of the two is required. The body is one JSON object of at most 4096 bytes.

Response: `200` the chat as `chats` lists it.

```json
{
  "session_id": "2026-10-14T12-07-00_3f1c2a90-7b6e-4d11-8a55-0c1e2d3f4a5b",
  "title": "Launch email",
  "created": "2026-10-14T12:07:00Z",
  "ended": "2026-10-15T09:00:00Z",
  "project_id": "p_3f1c2a90",
  "parallel": true,
  "running": false,
  "run_state": "idle"
}
```

Errors:

- `400` closed or title is required. Also for a body that is not JSON or is larger than 4096 bytes.
- `400` a Bot chat needs a title of at most 100 characters
- `400` the main chat cannot be renamed
- `400` only a Project's chats can be closed or opened again
- `404` chat not found. The id is not a Bot chat, or its conversation is gone.
- `409` closing the Bot main chat, a project's current Coordinator chat (`/new` sets that one aside instead), or the chat a channel kind is bound to now
- `500` the chat record could not be saved

Events: `bot.changed` on every change; `sessions.changed` for the chat when its title changes.

### DELETE /v1/bot/chats/{id}

Auth: any key
Effect: Deletes a Bot chat other than the current main chat, and drops it from the profile. A branched side chat closes its lane and leaves the main chat's history in place. Deleting the chat a kind is bound to clears that binding; the kind's next message opens a new chat. Undelivered task reports that were aimed at this chat are aimed at the current main chat instead, with no channel reply.

Path: `id` session id of a chat in `chats`, other than the current main chat.

Request: no body.

Response: `200`

```json
{ "session_id": "2026-10-14T12-05-00_3f1c2a90-7b6e-4d11-8a55-0c1e2d3f4a5b" }
```

Errors:

- `404` side chat not found. The id is not a Bot chat, or it is the current main chat. Set the main chat aside with `POST /v1/bot/chats/{id}/new` before deleting the earlier one.
- `404` session not found. The profile still lists the chat but the conversation is already gone. The next `GET /v1/bot` drops that record.
- `409` turn already running
- `409` delete the side chats branched from this chat first: `<titles>`. Titles are comma-separated. This applies to a former main chat that still has branched side chats.
- `409` this is the Coordinator chat of the Project `"<name>"`; start a new chat with /new, then delete this one
- `500` the conversation or the chat record could not be removed, including `Cannot remove conversation images` after the conversation itself was removed

Events: `sessions.changed` and `bot.changed` on success.

### POST /v1/bot/tasks

Auth: any key
Effect: Creates a hidden background task, records it, and starts it. The task uses the primary's task model when one is pinned, otherwise its Bot model, otherwise Core's default model resolution (a `SourceSessionID` of a project lead's chat uses that lead's models first). The response is sent while `status` is still `queued`; the run then moves it to `running`. Creates the main chat first when it does not exist yet. When it settles, its outcome is reported once as a Bot turn in `source_session_id` (or the current main chat when that chat is gone); `report: true` on the task marks this, and `delivered` is set once the report turn is admitted.

Request:

```json
{
  "title": "Quick research",
  "prompt": "Return a short result.",
  "SourceSessionID": "2026-10-14T12-02-00_3f1c2a90-7b6e-4d11-8a55-0c1e2d3f4a5b"
}
```

- `title` (string, optional). Trimmed. Empty becomes `Background task`.
- `prompt` (string, required). Trimmed. Empty is rejected. The body is one JSON object of at most 131072 bytes.
- `SourceSessionID` (string, optional). Stored as sent and returned as `source_session_id`. Note the request key is `SourceSessionID` (any letter case), not `source_session_id`.
- Other fields are ignored. This route cannot open a visible conversation.

Response: `202`

```json
{
  "id": "bot_0123456789abcdef0123456789abcdef",
  "session_id": "2026-10-14T12-07-00_3f1c2a90-7b6e-4d11-8a55-0c1e2d3f4a5b",
  "title": "Quick research",
  "prompt": "Return a short result.",
  "source_session_id": "2026-10-14T12-02-00_3f1c2a90-7b6e-4d11-8a55-0c1e2d3f4a5b",
  "provider_id": "provider-b",
  "model": "model-b",
  "model_pinned": true,
  "status": "queued",
  "created": "2026-10-14T12:07:00Z",
  "updated": "2026-10-14T12:07:00Z"
}
```

`provider_id`, `model`, and `model_pinned` are omitted when no task model is pinned. `source_session_id` is omitted when the request did not send one. Refetch `GET /v1/bot` for `running`, `completed`, `failed`, `paused`, and `cancelled`.

Errors:

- `400` task prompt is required. Also for a body that is not JSON or is larger than 131072 bytes.
- `500` the task or its conversation could not be created

Events: `bot.changed` when the task is recorded and again each time its status changes. `sessions.changed` for the new task conversation, and for the main chat when this call creates it. The run then emits `session.run_started`, `session.task`, `session.messages`, and `session.run_finished` on the task conversation.

### POST /v1/bot/tasks/{id}/{action}

Auth: any key
Effect: Pauses, resumes, or stops one background task. Pause and resume require a live run on the task's conversation. Stop cancels a live run, and also cancels a task that is still pending after the live run is gone.

Path: `id` task id. `action` is `pause`, `resume`, or `stop`.

Request: no body.

Response: `200`

```json
{ "ok": true }
```

Pause sets the task's `status` to `paused`. Resume sets it to `running`. Stop sets it to `cancelled`. The chat's `run_state` follows the live run (`pausing` while pause is still finishing in-flight work, then `paused`).

Errors:

- `404` unknown bot task action
- `404` bot task not found
- `409` task is no longer running. Pause or resume without a live run, or stop when nothing is live and no pending task remains. A task still `queued` that has not started its run returns this.
- `500` the run state or the task record could not be saved. The message `bot task not found` here means the task disappeared after the `404` check. If saving the task record fails, pause, resume, or stop has already been applied to the live run.

Events: `bot.changed`. Pause, resume, and stop also emit `session.task` for the task conversation. Stop of a live run also leads to `session.run_finished`.

### GET /v1/bot/supervisor/runs

Auth: any key
Effect: Lists the supervisor's runs, newest first. Read only.

Query:

- `run_ids` (optional) comma-separated run ids: only those runs, in that order; unknown ids are left out. `limit` and `before` are ignored then.
- `limit` (optional) 1 or more, default 20, at most 100.
- `before` (optional) a run id from an earlier page: the runs older than it.
- `session_id` (optional) a Bot chat: only the runs whose `origin_session_ids` name it (applied before the other parameters).

Response: `200`

```json
{
  "runs": [
    {
      "run_id": "turn_6f1c…",
      "supervisor_session_id": "2026-11-15T12-13-10_aca8…",
      "status": "completed",
      "started_at": "2026-11-15T12:13:10.296Z",
      "finished_at": "2026-11-15T12:13:10.395Z",
      "triggers": ["check_due"],
      "agenda_ids": ["ag_18dcfdca959ed51d"],
      "target_session_ids": ["2026-11-15T12-17-16_18f8…"],
      "tools": ["bot_radar", "bot_agenda"],
      "input_tokens": 5120,
      "output_tokens": 96,
      "summary": "tests are running, look again later",
      "outbox": false,
      "pruned": false,
      "target_tasks": [
        { "task_id": "bot_3c1f…", "title": "Run the tests", "session_id": "2026-11-15T12-14-02_77ab…", "hidden": true }
      ],
      "origin_session_ids": ["2026-11-15T12-01-44_5d02…"]
    }
  ],
  "next_before": "turn_6f1c…"
}
```

A run is one turn of the supervisor session (or of a rotated one Core still keeps); `run_id` is that turn's id (`turn.id` on its records). `status` is the turn's (`running` while it runs). `triggers` are the kinds of the inbox events it took (`run_settled`, `task_attention`, `check_due`, `owner_resumed`); `agenda_ids` and `target_session_ids` the items they name and those items' target conversations; `tools` the tool calls in order. `summary` is the stage the run recorded with `bot_agenda`, else the last thing it said, at most 240 characters. `outbox` is true when it told the owner with `bot_outbox`. `target_tasks` are the Bot tasks and Tasks its events, items and `bot_task` calls name that Core still knows: `title`, and `session_id`, the conversation to open (a Bot task's, or a Task's latest attempt; omitted when there is none). `hidden` is true for a hidden Bot task's conversation, which is not in the conversation list and opens as a Bot task. `[]` when none, and for a pruned run. `origin_session_ids` are the Bot chats (main, channel, side or Coordinator chat) whose agenda items the run took up, each item's `origin_session_id` as the agenda has it now; `[]` when none (a run about no item, or items no longer in the agenda). A run whose supervisor session was deleted after rotation comes from its main-chat notice row (see Supervisor) with `pruned: true`; runs that left no row are gone then. `next_before` is present when older runs remain.

Errors:

- `400` limit must be a positive number
- `404` run not found (`before` is not a listed run)

### GET /v1/bot/supervisor/runs/{run_id}

Auth: any key
Effect: One supervisor run with what it did. Read only.

Response: `200`

```json
{
  "run": { "run_id": "turn_6f1c…", "pruned": false },
  "events": [{ "kind": "check_due", "agenda_ids": ["ag_18dcfdca959ed51d"], "at": "2026-11-15T12:13:10Z" }],
  "records": []
}
```

`run` has the shape of a `GET /v1/bot/supervisor/runs` entry. Runs whose rows are only in a main chat `/new` set aside are found too. `events` are the inbox events the run took. `records` are its transcript records, the same shape as `records` of `GET /v1/sessions/{id}`. A pruned run has `[]` for both.

Errors:

- `404` run not found

### POST /v1/bot/agenda/{id}/cancel

Auth: admin
Effect: Cancels one open item of the Bot's supervision agenda, as the owner (the same change as the Bot's `bot_agenda` cancel in a turn the owner started). The item stays in the agenda as `cancelled`; its targets are not touched.

Request: no body.

Response: `200`

```json
{ "item": { "id": "ag_0123456789abcdef", "status": "cancelled" } }
```

`item` is the whole saved item, the shape of a `GET /v1/bot` `agenda` entry (shortened above).

Errors:

- `403` administrator key required
- `404` agenda item not found
- `409` the item is already `completed` or `cancelled`

Events: `bot.changed`.

### POST /v1/bot/agenda/{id}/check

Auth: admin
Effect: Runs an active item's follow-up now, as when its `next_check_at` falls due: its pre-check runs first when it has one, and the background supervisor is woken with a `check_due` event unless the pre-check output is unchanged. An item without a pre-check always wakes the supervisor. An owner check never counts against the item's automatic-wake limit, and an unchanged pre-check leaves `next_check_at` as it was (only an item that had one moves it on).

Request: no body.

Response: `200`

```json
{ "outcome": "queued", "item": { "id": "ag_0123456789abcdef", "status": "active" } }
```

- `outcome` is `queued` (the supervisor will look at the item) or `unchanged` (the pre-check found nothing new; no model request).

Errors:

- `403` administrator key required
- `404` agenda item not found
- `409` the item is not `active` (a `waiting_owner` item waits for the owner's answer in a Bot chat)
- `500` the check could not be queued

Events: `bot.changed`.

### GET /v1/bot/channels

Auth: admin
Effect: Lists every channel kind, configured or not (the primary's instance of each), every channel instance, plus the global display settings and each display view. Changes nothing. While a Feishu scan is running, poll this route: the scan does not emit an event.

Request: no body.

Response: `200`

```json
{
  "display": {
    "settings": {},
    "builtin": {
      "tool_progress": "all",
      "tool_progress_grouping": "accumulate",
      "show_reasoning": false,
      "reasoning_style": "code",
      "interim_assistant_messages": true,
      "long_running_notifications": true,
      "cleanup_progress": false,
      "tool_preview_length": 0,
      "tool_progress_command": false,
      "busy_input_mode": "interrupt",
      "followup_grace_seconds": 3
    }
  },
  "channels": [
    {
      "kind": "feishu",
      "configured": false,
      "enabled": false,
      "owner_bound": false,
      "display": {
        "override": {},
        "inherited": {
          "tool_progress": "new",
          "tool_progress_grouping": "accumulate",
          "show_reasoning": false,
          "reasoning_style": "code",
          "interim_assistant_messages": true,
          "long_running_notifications": true,
          "cleanup_progress": false,
          "tool_preview_length": 40,
          "tool_progress_command": false,
          "busy_input_mode": "interrupt",
          "followup_grace_seconds": 3
        },
        "effective": {
          "tool_progress": "new",
          "tool_progress_grouping": "accumulate",
          "show_reasoning": false,
          "reasoning_style": "code",
          "interim_assistant_messages": true,
          "long_running_notifications": true,
          "cleanup_progress": false,
          "tool_preview_length": 40,
          "tool_progress_command": false,
          "busy_input_mode": "interrupt",
          "followup_grace_seconds": 3
        }
      }
    },
    {
      "kind": "discord",
      "configured": false,
      "enabled": false,
      "owner_bound": false,
      "display": {
        "override": {},
        "inherited": {
          "tool_progress": "all",
          "tool_progress_grouping": "accumulate",
          "show_reasoning": false,
          "reasoning_style": "subtext",
          "interim_assistant_messages": true,
          "long_running_notifications": true,
          "cleanup_progress": false,
          "tool_preview_length": 40,
          "tool_progress_command": false,
          "busy_input_mode": "interrupt",
          "followup_grace_seconds": 3
        },
        "effective": {
          "tool_progress": "all",
          "tool_progress_grouping": "accumulate",
          "show_reasoning": false,
          "reasoning_style": "subtext",
          "interim_assistant_messages": true,
          "long_running_notifications": true,
          "cleanup_progress": false,
          "tool_preview_length": 40,
          "tool_progress_command": false,
          "busy_input_mode": "interrupt",
          "followup_grace_seconds": 3
        }
      }
    }
  ],
  "feishu": {
    "kind": "feishu",
    "configured": false,
    "enabled": false,
    "owner_bound": false
  },
  "instances": []
}
```

`feishu` is the same object as `channels` for `feishu`, including `display` and, when set, `registration`, `brand`, `app_id`, and `bot_name`. The example omits those repeated fields. A configured Discord object adds `bot_name`, `application_id`, `invite_url`, and, until claimed, `claim_code`. `state` and `error` appear while the connection is up.

Errors: none.

Events: none.

### PUT /v1/bot/channels

Auth: admin
Effect: Replaces the global display settings. Fields left out of `display` are cleared. Kind overrides are unchanged.

Request:

```json
{ "display": { "tool_progress": "verbose", "cleanup_progress": true } }
```

- The body is one JSON object of at most 65536 bytes, and its only field is `display`.
- `display` is one object of the display fields. Unknown fields are rejected. See Channels for types, allowed values, and which fields are global-only. A kind-only restriction does not apply here: `tool_progress_command` and `followup_grace_seconds` may be set.
- `{"display":{}}` clears every global setting.

Response: `200` the same object as `GET /v1/bot/channels`, after the save.

Errors:

- `400` expected {"display": {...}}
- `400` invalid display settings
- `400` tool_progress must be one of off, new, all, verbose, log
- `400` tool_progress_grouping must be one of accumulate, separate
- `400` reasoning_style must be one of code, blockquote, subtext
- `400` tool_preview_length must be between 0 and 100000
- `400` busy_input_mode must be one of interrupt, steer, queue
- `400` followup_grace_seconds must be between 0 and 60
- `500` the settings could not be saved

Events: none.

### PUT /v1/bot/channels/{kind}

Auth: admin
Effect: Sets up the primary's instance of a kind that is not configured yet (creating the instance, bound to the primary), switches a configured one on or off, or replaces its display override. A setup and a display change cannot be in the same body. The display override applies whether or not the kind is configured, and it survives deleting the credentials.

Path: `kind` is `feishu` or `discord`.

Request:

Feishu setup, only while Feishu is not configured. Unknown fields are rejected. The body is at most 131072 bytes.

```json
{
  "brand": "feishu",
  "app_id": "cli_example",
  "app_secret": "...",
  "enabled": true
}
```

- `brand` (string, optional). `feishu` or `lark`. Omitted is stored as `feishu`. Lark makes Core's own messages in this channel English; Feishu uses Chinese.
- `app_id` and `app_secret` (strings, required). Trimmed. The secret is stored and never returned.
- `enabled` (bool, optional). Omitted or true connects immediately. False stores the credentials disconnected.

Discord setup, only while Discord is not configured. Unknown fields are rejected.

```json
{ "token": "...", "enabled": true }
```

- `token` (string, required). Surrounding whitespace is removed, and a leading `Bot ` prefix is removed regardless of letter case. The token is stored and never returned. Discord is asked who the bot is; the response then carries `bot_name`, `application_id`, and `invite_url`.
- `enabled` (bool, optional). Same meaning as for Feishu.

Switch, only when that kind is configured. The body is exactly this one field:

```json
{ "enabled": false }
```

`enabled` must be a JSON boolean. Sending the current value is a success and changes nothing.

Display, configured or not. This replaces the kind's whole override. `{}` deletes it. `tool_progress_command` and `followup_grace_seconds` are rejected.

```json
{ "display": { "tool_progress": "off" } }
```

Display beside the switch, only when the kind is configured. The display is saved first, then the switch. The body has exactly these two fields:

```json
{ "enabled": false, "display": { "tool_progress": "new" } }
```

The credential check waits up to 15 seconds. A manual setup returns a new `claim_code`.

Response: `200` that kind's object, the same shape as one element of `channels` from `GET /v1/bot/channels`.

Errors:

- `404` unknown channel
- `404` this channel is not configured. `{"enabled": true}` or `{"enabled": false}` alone, before setup.
- `400` invalid settings request. Not a JSON object, a body larger than 131072 bytes, or a setup body with an unknown field or a wrong JSON type.
- `400` "display" goes alone or beside "enabled" of a configured channel
- `400` enabled must be true or false
- `400` invalid display settings
- `400` tool_progress must be one of off, new, all, verbose, log
- `400` tool_progress_grouping must be one of accumulate, separate
- `400` reasoning_style must be one of code, blockquote, subtext
- `400` tool_preview_length must be between 0 and 100000
- `400` busy_input_mode must be one of interrupt, steer, queue
- `400` followup_grace_seconds must be between 0 and 60
- `400` tool_progress_command is a global setting
- `400` followup_grace_seconds is a global setting
- `400` app_id, app_secret and brand (feishu or lark) are required
- `400` Feishu rejected these credentials: `<reason>`. Credentials and URL query strings are removed from the reason.
- `400` token is required
- `400` Discord rejected this token: `<reason>`
- `409` a Feishu bot is already connected; disconnect it first
- `409` a Discord bot is already connected; disconnect it first
- `409` another channel instance already uses this Feishu app
- `409` another channel instance already uses this Discord bot
- `500` a claim code could not be created, the credentials or the display could not be saved, or the switch failed. If the display was saved and the switch then failed, the new display is already in place.

Events: none.

### DELETE /v1/bot/channels/{kind}

Auth: admin
Effect: Disconnects the primary's instance of that kind, deletes its stored credentials and removes the instance. The primary keeps its channel chat of that kind: an app set up again continues it, and a turn running there finishes with its answer kept in the chat. The app or bot on Feishu, Lark, or Discord is left in place. A Feishu scan in progress is cancelled. The display override is kept.

Path: `kind` is `feishu` or `discord`.

Request: no body.

Response: `200` that kind's object, with `configured` false and `enabled` false. Deleting a kind that was not configured is the same success.

Errors:

- `404` unknown channel
- `500` the credentials could not be deleted

Events: none.

### POST /v1/bot/channels/{kind}/registration

Auth: admin
Effect: Starts Feishu or Lark scan-to-create for the primary's Feishu instance, creating the instance (bound to the primary) when there is none. Any scan already in progress on that instance is cancelled and replaced. The new app uses the instance's Bot's saved name, or `Gyres Bot` when the name is empty, and the description `Your persistent local assistant.` (not the Bot profile description). The owner can edit that name and description on the scan page. On success the scanning user is the owner, `configured` becomes true, and `registration` disappears. Poll `GET /v1/bot/channels` for the link and the result. Discord has no scan.

Path: `kind` is `feishu` for this flow. `discord` is rejected.

Request: no body.

Response: `202` the Feishu channel object, as soon as the QR link is known, the attempt has failed, the attempt has finished, or 10 seconds have passed. The scan itself is abandoned 11 minutes after it starts.

```json
{
  "kind": "feishu",
  "configured": false,
  "enabled": false,
  "owner_bound": false,
  "registration": {
    "state": "waiting",
    "url": "...",
    "expires_at": "2026-10-14T12:12:00+08:00"
  }
}
```

`url` is the platform's link; the example shows it as `...`. Before the link arrives, `registration` is `{"state":"waiting"}`. On failure within the wait, `state` is `failed`, `error` is the reason, and `url` is omitted. If the scan finishes successfully within the wait, the body is the configured channel with no `registration`, `owner_bound` true, and `brand`, `app_id`, and `bot_name` set. `brand` is `lark` when the scanning tenant is Lark, otherwise `feishu`.

Errors:

- `404` unknown channel
- `404` no registration flow for this channel
- `409` a Feishu bot is already connected; disconnect it first
- `503` channels are not running

Events: none.

### DELETE /v1/bot/channels/{kind}/registration

Auth: admin
Effect: Cancels Feishu scan-to-create, if one is in progress, and returns the Feishu channel. Cancelling when nothing is in progress is a success. Discord has no scan.

Path: `kind` is `feishu` for this flow. `discord` is rejected.

Request: no body.

Response: `200` the Feishu channel object, with no `registration`.

Errors:

- `404` unknown channel
- `404` no registration flow for this channel

Events: none.

### GET /v1/bot/channel-instances

Auth: admin
Effect: Lists every channel instance, in the order they were created. Changes nothing.

Request: no body.

Response: `200` `{"instances": [...]}`, each element the same object as in `instances` of `GET /v1/bot/channels`.

Errors: none.

Events: none.

### POST /v1/bot/channel-instances

Auth: admin
Effect: Creates a channel instance that is not set up yet, bound to a Bot. Set it up afterwards with `PUT /v1/bot/channel-instances/{id}` (the same bodies as `PUT /v1/bot/channels/{kind}`) or, for Feishu, with its registration route.

Request:

```json
{"kind": "feishu", "bot_id": "bot-9d2e4c61a0b3f785"}
```

`bot_id` may be omitted: the instance talks to the primary.

Response: `201` the instance object, `configured` false.

```json
{
  "id": "feishu-1a2b3c4d",
  "kind": "feishu",
  "bot_id": "bot-9d2e4c61a0b3f785",
  "configured": false,
  "enabled": false,
  "owner_bound": false
}
```

The example omits `display`.

Errors:

- `400` invalid channel instance request
- `400` unknown channel (the kind)
- `404` bot not found
- `409` this Bot already has an app of this kind
- `500` the channel list cannot be read or saved

Events: none.

### GET /v1/bot/channel-instances/{id}

Auth: admin
Effect: Returns one instance. Changes nothing.

Request: no body.

Response: `200` the instance object.

Errors:

- `404` unknown channel instance

Events: none.

### PUT /v1/bot/channel-instances/{id}

Auth: admin
Effect: As `PUT /v1/bot/channels/{kind}`, for this instance: sets it up while it is not configured, switches it on or off, or replaces its display override.

Request: the same bodies as `PUT /v1/bot/channels/{kind}`, for the instance's kind.

Response: `200` the instance object.

Errors: as `PUT /v1/bot/channels/{kind}`, and `404` unknown channel instance.

Events: none.

### DELETE /v1/bot/channel-instances/{id}

Auth: admin
Effect: Disconnects the instance, deletes its stored credentials and removes it from the list. Its Bot keeps its channel chat of that kind: a later app of the kind for that Bot continues it, and a turn running there finishes with its answer kept in the chat. The app or bot on the platform is left in place. A Feishu scan in progress is cancelled.

Request: no body.

Response: `200` the kind's object with `configured` false (no `id`).

Errors:

- `404` unknown channel instance
- `500` the credentials or the channel list could not be saved

Events: none.

### PUT /v1/bot/channel-instances/{id}/bot

Auth: admin
Effect: Binds the instance to another Bot. Its running turn stops and the messages waiting behind it are dropped; the earlier Bot keeps its channel chat, set aside; the new Bot's channel chat starts with the next message. Binding to the Bot it already talks to changes nothing.

Request:

```json
{"bot_id": "bot-9d2e4c61a0b3f785"}
```

An empty `bot_id` is the primary.

Response: `200` the instance object.

Errors:

- `400` invalid binding request
- `404` unknown channel instance
- `404` bot not found
- `409` this Bot already has an app of this kind
- `500` the channel list could not be saved

Events: none.

### POST /v1/bot/channel-instances/{id}/registration

Auth: admin
Effect: As `POST /v1/bot/channels/{kind}/registration`, for this Feishu instance. The new app is named after the instance's Bot.

Request: no body.

Response: `202` the instance object.

Errors: as `POST /v1/bot/channels/{kind}/registration`, and `404` unknown channel instance.

Events: none.

### DELETE /v1/bot/channel-instances/{id}/registration

Auth: admin
Effect: As `DELETE /v1/bot/channels/{kind}/registration`, for this instance.

Request: no body.

Response: `200` the instance object, with no `registration`.

Errors: `404` unknown channel instance, `404` no registration flow for this channel.

Events: none.
