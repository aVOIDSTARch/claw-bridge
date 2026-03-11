# Claude Universal

# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with
code in this repository.

## Commands

```bash
# Frontend dev server (port 5173, proxies /api → localhost:3001)
npm run dev

# Backend API server (port 3001, binds 127.0.0.1)
npm run server

# Production build
npm run build

# Change admin credentials interactively
npm run set-password
```

Both `npm run dev` and `npm run server` must be running simultaneously during
development.

The backend requires root (or appropriate group membership) to access the
Dovecot SQLite DB, run `systemctl`, read log files, and create maildirs.

## Architecture

Full-stack mail admin panel for foldedbits.net. Frontend is Preact + Tailwind +
DaisyUI (night theme). Backend is Express with Node's built-in `node:sqlite`.

**Do not use `better-sqlite3`** — the system's g++ 9.3 is too old to compile it.
Use `node:sqlite` (built into Node 24).

### Auth flow

`app.jsx` monkey-patches `window.fetch` to auto-inject
`Authorization: Bearer {token}` on all `/api/*` requests. The token is stored in
`localStorage` as `mail_admin_toke>

### Backend (`server/`)

- `index.js` — Express entry point. All `/api/*` routes except `/api/auth`
  require Bearer token validation. Also serves `dist/` for production. Handles
  `--set-password` C>
- `routes/auth.js` — Login/logout; generates 32-byte hex tokens via
  `crypto.randomBytes`.
- `routes/mailboxes.js` — CRUD against SQLite `users` table. Password hashing
  via `doveadm pw -s SHA512-CRYPT`. Creates Maildir at
  `$VMAIL_BASE/{domain}/{username}/`, `ch>
- `routes/queue.js` — Wraps `postqueue -j` (NDJSON), `postsuper -r ALL`,
  `postsuper -d {id}`.
- `routes/logs.js` — Tails log files via `tail -n {lines} {file}` (max 2000
  lines).
- `routes/server-status.js` — `systemctl is-active` per service; disk usage via
  `df -B1 /var/vmail`.
- `credentials.js` — Reads/writes `/etc/mail-admin/credentials.json` with mode
  0o600; bcrypt cost 12.
- `archive-config.js` — Utility for `/etc/mail-admin/archive-config.json`; not
  yet wired to any route.

### Frontend (`src/`)

- `app.jsx` — Root component: sidebar nav, auth state, fetch patch.
- `pages/Login.jsx` — Login form → `POST /api/auth/login`.
- `pages/Mailboxes.jsx` — CRUD UI for mail users. Default domain:
  `foldedbits.net`.
- `pages/Queue.jsx` — Postfix queue viewer with flush and per-message delete.
- `pages/Logs.jsx` — Log viewer with source selector and error/warn keyword
  highlighting.
- `pages/Server.jsx` — Service start/stop/restart; disk usage bar (green <65%,
  yellow 65–85%, red >85%).

### Environment variables

| Variable              | Default                               |
| --------------------- | ------------------------------------- |
| `PORT`                | `3001`                                |
| `HOST`                | `127.0.0.1`                           |
| `MAIL_DB`             | `/etc/dovecot/sql/users.db`           |
| `VMAIL_BASE`          | `/var/vmail`                          |
| `MAIL_ADMIN_CREDS`    | `/etc/mail-admin/credentials.json`    |
| `ARCHIVE_CONFIG_PATH` | `/etc/mail-admin/archive-config.json` |

### DB schema

```sql
users(id, username, domain, password TEXT SHA512-CRYPT, maildir, quota, active)
```

### Services monitored

`postfix`, `dovecot`, `nginx`, `opendkim`, `php7.4-fpm`

---

## Skills Marketplace

Browse **https://www.skilsmp.com** to discover and learn new skills relevant to
the current task. This site is always accessible and should be consulted when
encountering>

**Before installing any skill:**

1. Ask the user for permission — describe what the skill does and why it's
   useful.
2. Assess and document risks in `~/skill.md` (create if it doesn't exist):
   - Security implications (network access, file system access, code execution,
     data exposure)
   - Trust level of the skill source
   - Any dependencies it introduces
   - Whether it could conflict with existing tools or configuration
3. Present the risk summary clearly before proceeding. If risks are significant,
   recommend alternatives or a sandboxed approach.

Only install after explicit user approval.

---

## Working Style

Before implementing anything non-trivial, ask clarifying questions and actively
suggest improvements, extensions, and alternatives — even if not requested.
Surface better>

Default to the most comprehensive, fully-featured solution unless there is a
good reason not to (security risk, significant complexity cost, or architectural
mismatch). W>

Present options and trade-offs using `AskUserQuestion` menus so the user can
steer direction before work begins.

---

## Coding Standards

- **TypeScript always** — all new code must be TypeScript. For existing JS-only
  projects, propose a migration path before touching them.
- **Logging** — implement structured logging at appropriate levels (debug, info,
  warn, error) with meaningful context. Pick a consistent logger per project and
  use it eve>
- **User-readable errors** — every error that reaches a user must explain what
  went wrong and, where possible, how to fix it. No raw stack traces, no cryptic
  codes.
- **Comments** — comment the _why_, not the _what_. Explain non-obvious
  decisions, algorithmic choices, and anything a future reader would have to
  reverse-engineer. Keep >
- **Modularity** — every module does exactly one thing. Prefer many small,
  focused, composable files over large multi-purpose ones. The goal is "lego
  brick" code: self-co>
- **AI agent accessibility** — when designing any module, always suggest how it
  could be exposed as a callable tool for AI agents: typed input/output schemas,
  clear side->
- **Tests alongside code** — write unit tests for each module or function as it
  is built, not after. Every discrete block of code must have comprehensive test
  coverage be>

---

## Project Workflow

### 1. Planning

Before starting any non-trivial task, create a planning document at
`~/plans/<project-slug>.md` (create the folder if needed). The document must be
comprehensive, well-st>

### 2. Progress Logging

At logical milestones (after each major feature, route, or component is
complete):

1. Make a git commit with a descriptive message.
2. Create or update `~/progress/<project-slug>.md` (create `~/progress/` if it
   doesn't exist). Append a timestamped entry summarising what was completed and
   what remains.

The progress file must have the same base name as the plan file.

### 3. Completion & Handoff

When a project is finished:

1. Copy the final plan to `~/completed-projects/<project-slug>.md` (create
   folder if needed).
2. Create `~/completed-projects/<project-slug>-appendix.md` containing:
   - Any incomplete items or known issues
   - Recommended next stages with enough context for a fresh session to continue

---

### Slash Commands

These commands scan `~/completed-projects/` and present interactive menus via
`AskUserQuestion`. After a selection is made, act on it or offer to return to
the prompt.

### `/incompletes`

Scan all `~/completed-projects/*-appendix.md` files. Extract
incomplete/unfinished items from each. Present a numbered menu of projects —
when the user selects one, displ>

**Menu format:** one option per project that has incomplete items, plus a
"Return to prompt" option.

### `/do-more`

Scan all `~/completed-projects/*-appendix.md` files. Extract the recommended
next stages / enhancement ideas from each. Present a menu grouped by project —
when the user >

**Menu format:** one option per project that has suggested enhancements, plus a
"Return to prompt" option.

### `/whats-next`

Combine the outputs of `/incompletes` and `/do-more` into a single menu that
clearly distinguishes entry types:

- Prefix incomplete work items with `[INCOMPLETE]`
- Prefix enhancement suggestions with `[ENHANCEMENT]`

Present all items in one list with project labels. After selection, act on the
chosen item or return to the menu. Always end with a "Return to prompt" option.

---

## Ideas System

Ideas are stored as structured markdown files in `~/ideas/`. Each file covers
one topic and acts as a seed list for future exploration. The format is designed
so the coll>

### Ideas file format

```markdown
# Topic Name

Brief one-line description of what this topic covers.

---

1. **Verbatim:** "Exact text as the user wrote it" **Claude:** Interpretation —
   what the idea is really about, implications, connections to other ideas.

2. **Verbatim:** "Next idea..." **Claude:** ...
```

Each numbered item always has both parts. Never collapse or rewrite the Verbatim
field. The Claude field should surface non-obvious angles and connections.

Occasionally (not on every session) suggest refactoring the `~/ideas/`
collection: merging topics that have grown too similar, splitting topics that
have become too broad>

### `/ideas`

Scan all `~/ideas/*.md` files. Present a top-level menu of topic names (one per
file). On selection, display the numbered idea list from that file and show two
additional>

- **Synthesize** — use `claude-opus-4-6` with extended thinking to generate
  10–15 project ideas that combine or build on the ideas in this topic. Present
  the synthesis in>
- **Return to topic list** — go back to the top-level topic menu.

All menus use `AskUserQuestion`.

### `/add-idea <text>`

Take all text following `/add-idea`. Determine the best matching existing topic
file in `~/ideas/` (or propose a new topic name if none fits — confirm with user
before cr>

1. A new numbered entry with the **Verbatim** field set to the exact text
   provided.
2. A **Claude** field with Claude's interpretation, implications, and
   connections to existing ideas in that file.

Confirm what was added and where before finishing.

### Future app design notes

The ideas system is pre-designed for extraction into a standalone app and
website. Keep these invariants so migration stays clean:

- Each idea item has: `id` (sequential per file), `topic`, `verbatim` (immutable
  after creation), `claude_interpretation`, `created_at`
- Topic files map 1:1 to topic pages in the future app
- `/ideas` → browsing UI; `/add-idea` → capture UI; Synthesize → AI synthesis
  feature (premium tier)
- All writes go through append-only operations — never edit existing idea
  entries, only add new ones
- When implementing the app, use a TypeScript API-first backend (tRPC or REST)
  with the markdown files as the initial data source
  <ctrl-6>
