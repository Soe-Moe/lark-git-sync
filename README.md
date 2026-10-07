# lark-git-sync

A Claude Code skill that turns a repository's git history for a week, month or custom date range into clear, feature-level task records in a **Lark Base (Bitable)** table, with **subtasks** where the work naturally splits.

- Collects commits for `this-week`, `last-week`, `this-month`, `last-month`, a custom `YYYY-MM-DD..YYYY-MM-DD`, or `since-last-sync`
- Detects vague commit messages (`fix`, `update`, `wip`, ...) and reads the commit's files, `CHANGELOG.md` and a bounded diff to describe what actually changed
- Groups commits by ticket key, conventional-commit scope and module into feature tasks; splits multi-part work (DB / API / UI, etc.) into linked subtasks
- Skips commits already synced (by commit SHA) and can attach new subtasks to parent tasks from earlier periods
- Shows a preview for approval before anything is written to Lark

## Requirements

- [Claude Code](https://docs.claude.com/en/docs/claude-code)
- The official Lark MCP server ([`@larksuiteoapi/lark-mcp`](https://github.com/larksuite/lark-openapi-mcp)) configured in Claude Code, with the Base record tools enabled (e.g. `-t preset.base.default`)
- A Lark custom app added to your Base as a collaborator with edit rights

## Install

**As a plugin (recommended)**

```
/plugin marketplace add <your-github-user>/lark-git-sync
/plugin install lark-git-sync@lark-git-sync
```

**Or copy the skill manually**

```bash
# for you, in every project
mkdir -p ~/.claude/skills && cp -r skills/lark-git-sync ~/.claude/skills/

# or for one project, shared with the team via git
mkdir -p .claude/skills && cp -r skills/lark-git-sync .claude/skills/
```

## Lark Base table setup

Create a table with these fields (names must match exactly):

| Field | Type |
|---|---|
| Title | Text (primary field) |
| Summary | Text |
| Type | Single select: Feature, Fix, Refactor, Perf, Docs, Test, Chore |
| Level | Single select: Task, Subtask |
| Parent Task | Link to the same table (single record) |
| Module | Text |
| Repo | Text |
| Branch | Text |
| Authors | Text |
| Start Date | Date |
| End Date | Date |
| Commit Count | Number |
| Commit SHAs | Text |
| Period | Text |
| Status | Single select: Done (plus any others you use) |

If the MCP server exposes `bitable_v1_appTableField_create`, the skill offers to create missing fields for you. Turn on the parent-child view in the table to see subtasks nested.

## Usage

In a git repo, ask Claude Code, for example:

```
sync last-week to lark
sync this-month to lark --author aung --dry-run
sync 2026-09-01..2026-09-15 to lark --flat
```

On the first run it asks for your Base table URL and stores it in `.claude/lark-sync.json` in the repo.

| Flag | Effect |
|---|---|
| `--all-branches` | Include commits from all local branches (default: current branch) |
| `--author <name\|email>` | Only that author's commits |
| `--flat` | No subtasks |
| `--dry-run` | Preview only, never write |

## Safety

- Nothing is written to Lark without your approval in the same run.
- The skill never changes git history or your working tree; it only writes `.claude/lark-sync.json`.
- Summaries never include secrets, `.env` contents or customer data from diffs.

## License

MIT
