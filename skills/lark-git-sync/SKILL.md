---
name: "lark-git-sync"
description: "Sync the current repo's git commits for a week, month or custom date range into a Lark Base table as feature-grouped tasks and subtasks, via the Lark MCP server, with preview before writing."
---

# Lark Git Sync

Turn the current repository's git history for a chosen period into clear, feature-level task records in a Lark Base (Bitable) table. Work that naturally splits into parts becomes a parent task with linked subtasks. Vague commit messages are clarified by reading the actual changes. Nothing is written to Lark until the user approves a preview.

## Inputs

Parse the range from the user's invocation. If none is given, ask once with these options:

| Argument | Meaning (timezone Asia/Bangkok, weeks start Monday) |
|---|---|
| `this-week` (default) | Monday 00:00 of this week -> now |
| `last-week` | Previous Monday 00:00 -> previous Sunday 23:59:59 |
| `this-month` | 1st of this month 00:00 -> now |
| `last-month` | 1st -> last day of previous month |
| `YYYY-MM-DD..YYYY-MM-DD` | Custom inclusive range |
| `since-last-sync` | From `lastSyncedUntil` in the config file -> now |

Optional flags: `--all-branches` (default: current branch only), `--author <name|email>` (default: all authors), `--flat` (no subtasks, one level only), `--dry-run` (preview only, never write).

Always echo the resolved absolute range back, e.g. `Range: 2026-09-28 00:00 -> 2026-10-04 23:59 (+07:00), branch: develop`.

## Step 0 - Preconditions

1. Confirm the working directory is a git repo: `git rev-parse --show-toplevel`. Stop with a clear message if not.
2. Run `git fetch --quiet` only if the user asked to include remote branches; otherwise use local history.
3. Confirm the Lark MCP tools are available. Expected (official `@larksuiteoapi/lark-mcp`) tool names, matched by suffix since prefixes vary by install:
   - `bitable_v1_appTableField_list`
   - `bitable_v1_appTableRecord_search`
   - `bitable_v1_appTableRecord_create`
   - `bitable_v1_appTableRecord_update`
   - optional: `bitable_v1_appTableRecord_batchCreate`, `bitable_v1_appTableField_create`, `wiki_v2_space_getNode`
   If the record tools are missing, stop and tell the user to enable them (e.g. start lark-mcp with `-t preset.base.default` or list the tools with `-t`). Do not fall back to raw HTTP calls.

## Step 1 - Config (first run only)

Config lives at `<repo>/.claude/lark-sync.json`:

```json
{
  "baseUrl": "https://<tenant>.larksuite.com/base/<app_token>?table=<table_id>",
  "appToken": "<app_token>",
  "tableId": "<table_id>",
  "repoName": "<defaults to repo folder name>",
  "lastSyncedUntil": null
}
```

If missing, ask the user for the Base table URL and parse `app_token` (path segment after `/base/`) and `table_id` (`table=` query param). If the URL is a `/wiki/<token>` link, resolve the real `app_token` with `wiki_v2_space_getNode` (`obj_token`). Write the file and remind the user that the Lark app used by the MCP server must be added to the Base as a collaborator with edit rights (Base -> ... -> Add document app).

## Step 2 - Verify table schema

Call `appTableField_list` and check these fields exist (names exact):

| Field | Type | Content |
|---|---|---|
| Title | Text (primary) | Clear task title, imperative, <= 80 chars |
| Summary | Text | 2-5 line description of what changed and why |
| Type | Single select | Feature / Fix / Refactor / Perf / Docs / Test / Chore |
| Level | Single select | `Task` / `Subtask` |
| Parent Task | Link to this same table (single record) | Set only on subtasks; points at the parent task record |
| Module | Text | Area/scope, e.g. `auth`, `admin-panel`, `hs-code-api` |
| Repo | Text | `repoName` from config |
| Branch | Text | Branch the commits came from |
| Authors | Text | Comma-separated commit author names |
| Start Date | Date | Earliest commit time in the record (ms epoch) |
| End Date | Date | Latest commit time in the record (ms epoch) |
| Commit Count | Number | Commits in the record (parent = total of its subtasks) |
| Commit SHAs | Text | Short SHAs, comma-separated (used for de-duplication) |
| Period | Text | e.g. `2026-W40`, `2026-09`, or `2026-09-01..2026-09-15` |
| Status | Single select | Default `Done` |

`Parent Task` must be a single-link field (type 18) whose target table is this same table; that is how Lark Base models parent-child (subtask) hierarchy. Tell the user they can turn on the parent-child / hierarchy display in the table view to see subtasks nested.

If fields are missing: create them with `appTableField_create` when that tool is available (ask first), otherwise list exactly what the user must add and stop. Never rename or delete existing fields. If only `Level` / `Parent Task` are missing and the user does not want to add them, continue in `--flat` mode.

## Step 3 - Collect commits

```bash
git log <branch-or---all> --no-merges \
  --since="<start ISO+07:00>" --until="<end ISO+07:00>" \
  --date=iso-strict \
  --pretty=format:'%h%x1f%an%x1f%ae%x1f%ad%x1f%s%x1f%b%x1e'
```

Split records on `\x1e` and fields on `\x1f`. Apply `--author` filtering if requested. For every commit also collect `git show --stat --format= <sha>` (files and line counts).

If zero commits: report it and stop.

## Step 4 - De-duplicate against Lark

Call `appTableRecord_search` with filter `Repo` is `<repoName>` (paginate with `page_token` until done; `page_size` 500). From the results build:

- a set of every short SHA in existing `Commit SHAs` fields; drop any collected commit already in the set and report how many were skipped;
- a list of existing parent tasks (`Level` = `Task`): `record_id`, Title, Module, and any ticket key in Title/Summary. Used in Step 6c to attach new subtasks to work that started in an earlier period.

## Step 5 - Clarify unclear commits

Treat a commit message as **unclear** if any of these hold:

- Subject shorter than 15 characters, or only punctuation/emoji
- Subject is a generic word or phrase alone: `fix`, `fixes`, `update`, `updates`, `wip`, `changes`, `misc`, `minor`, `refactor`, `cleanup`, `test`, `temp`, `asdf`, `.`, `commit`, `save`, `final`, `review comments`, `fix bug`, `update code`
- Conventional-commit prefix with no meaningful object, e.g. `fix: fix`, `chore: update`
- Subject does not name any thing (module, feature, file, endpoint, screen) that changed

For each unclear commit, read the change itself, in this order, stopping once intent is clear:

1. `git show --stat --format=%B <sha>` - body and touched files
2. `CHANGELOG.md` / `CHANGELOG` / `docs/changelog*` in the repo: entries added by this commit (`git show <sha> -- CHANGELOG.md`) or entries dated within the range
3. `git show <sha> --format= -U3 -- <files>` limited to the 5 most-changed non-generated files, and at most ~400 diff lines total. Skip lockfiles, `dist/`, `build/`, generated code, migrations snapshots, and binary files.

Write a one-line clarified description per unclear commit from what the code actually does. Never invent intent the diff does not show; if still unclear, describe the mechanical change (e.g. "Adjust validation in `order.service.ts` createOrder") and mark it `(inferred)` in the preview.

## Step 6 - Group into tasks and subtasks

### 6a. Group commits into feature-level tasks

Cluster commits using these signals, strongest first:

1. Same ticket/issue key in subject, body or branch name (e.g. `TT-123`, `#45`)
2. Same conventional-commit scope (`feat(auth): ...`)
3. Same top-level module/directory and related intent within the range
4. Follow-up fixes to a feature committed in the same range belong to that feature, not a separate Fix task

Do not list trivial chores (formatting-only, version bumps) as separate tasks; fold them into the nearest group or one `Chore: maintenance` task.

### 6b. Decide which tasks get subtasks

Split a task into subtasks (skip entirely with `--flat`) only when **all** of these hold:

- the task has at least 3 commits, and
- its commits fall into at least 2 distinct, separately describable pieces of work, each with at least 1 non-trivial commit. Typical splits:
  - by layer: DB schema/migration/seed, backend API/service, frontend screen/UI, admin panel, mobile app
  - by deliverable: separate endpoints, separate screens, separate integrations (e.g. payment, mailer)
  - by phase: implementation vs. tests vs. docs, only when each is substantial

Keep a task flat when it is a single concern (one bug fix, one small refactor, one screen tweak), when a piece would be only trivial work, or when splitting would just restate commit messages one-to-one. Aim for 2-6 subtasks per parent; never more than 8 (merge the smallest).

### 6c. Attach to existing parents

If a new group clearly continues an existing parent task from Step 4 (same ticket key, or same Module and the same feature by title/summary), propose adding its work as new subtasks under that existing parent instead of creating a duplicate parent. Mark these `-> existing: <Title>` in the preview; the user confirms.

### 6d. Fill the records

- **Parent task:** Title = the feature/outcome; Summary = overall what and why; Level = `Task`; Type = dominant intent; Module; Authors = all authors across subtasks; Start/End Date = span of all subtasks; Commit Count = total; Commit SHAs = empty (SHAs live on subtasks); Period.
- **Subtask:** Title = the specific piece (e.g. "Add HS code search endpoint"), not a repeat of the parent title; Summary = what this piece changed; Level = `Subtask`; its own Type, Module, Authors, dates, Commit Count, Commit SHAs, Period; Parent Task = parent's record.
- **Flat task (no subtasks):** Level = `Task`, Parent Task empty, Commit SHAs filled.
- Every commit appears in exactly one leaf record (a flat task or a subtask). Write all content in English.

## Step 7 - Preview and confirm

Show a nested markdown table, subtasks numbered and indented under their parent:

```
| #   | Type    | Title                                 | Module  | Commits | Authors    | Dates         |
| 1   | Feature | HS code lookup in admin panel         | hs-code | 7       | Aung, Mia  | 09-29 -> 10-03 |
| 1.1 | Feature |   - Seed hs_statcode table            | backend | 2       | Aung       | 09-29 -> 09-30 |
| 1.2 | Feature |   - Add HS code search API            | backend | 3       | Aung       | 09-30 -> 10-02 |
| 1.3 | Feature |   - HS code search screen in admin    | admin   | 2       | Mia        | 10-02 -> 10-03 |
| 2   | Fix     | Fix duplicate booking on retry        | booking | 1       | Mia        | 10-01          |
```

Then each record's Summary and its commit list (SHA + original subject, with clarified text for unclear ones). Ask the user to:

- **Approve** - write all
- **Edit** - change titles/summaries/types; merge or split groups; promote a subtask to its own task; move a task under another task as a subtask; flatten a parent; drop records. Re-show the preview after edits.
- **Cancel** - write nothing

With `--dry-run`, stop after the preview.

## Step 8 - Write to Lark (parents first)

1. Create all **new parent tasks and flat tasks** first (`appTableRecord_batchCreate` in chunks of up to 500 when available, otherwise `appTableRecord_create` per record). Capture each returned `record_id` and map it to its preview number.
2. Then create all **subtasks**, setting `Parent Task` to `["<parent record_id>"]` (link fields take an array of record IDs). For subtasks attached to an existing parent (6c), use that parent's `record_id` from Step 4.
3. For existing parents that received new subtasks, `appTableRecord_update` their `End Date`, `Commit Count` and `Authors` to include the new work.

- Dates are millisecond epoch numbers; single-select values are plain strings matching the option name.
- If a parent fails to create, do not create its subtasks; report them as skipped. If a write fails, report which records failed and the error; do not retry blindly more than once.
- On success, update `lastSyncedUntil` in the config to the range end (ISO, +07:00).

## Step 9 - Report

One short summary: range, commits found, commits skipped as already synced, parent tasks created, subtasks created (and how many attached to existing parents), flat tasks created, any failures, and the Base URL from config.

## Rules

- Never write to Lark before explicit approval in the current run.
- Never modify git history, branches, or the working tree; this skill is read-only on the repo apart from `.claude/lark-sync.json`.
- Never put secrets, tokens, `.env` contents, or customer data from diffs into Summaries.
- Keep diff reading bounded (Step 5 limits) to keep runs fast on large ranges.