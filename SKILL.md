---
name: make-pages-interactive
description: Turn static HTML pages into a live commenting surface that any coding agent can monitor and update. Use when the user asks to make pages interactive, comment on a page, set up inline feedback, stop its local server, or remove the feedback layer.
---

# Make Pages Interactive

Turns any folder of HTML files into a place where the user can leave inline comments on text selections, elements, or the whole page. Comments POST to a local JSONL inbox. The coding agent watches that inbox, edits the HTML in response, appends to `feedback/history.json`, and the page auto-reloads with a walkthrough of what changed.

This skill is agent-agnostic. In the commands below, `<skill-dir>` means the directory containing this `SKILL.md`. Resolve it from the skill location supplied by the host agent; do not assume a Claude Code, Codex, or other provider-specific install path.

## When to invoke

User says any of:
- "make this page interactive" / "make these pages interactive" → **Setup flow**
- "add feedback to this page" / "let me comment on this page" → **Setup flow**
- "set up feedback on <dir>" → **Setup flow**
- "stop the feedback server" / "kill the server" / "shut it down" → **Stop flow**
- "remove the feedback layer" / "make pages static again" → **Removal flow**
- "update the make-pages-interactive skill" → **Update flow**

## Setup flow (when user wants to make pages interactive)

1. **Identify the target directory.** Usually the user's current working directory or a folder they named. If ambiguous, ask.
2. **Inject the feedback tags** into every `*.html` in that directory:
   ```
   python <skill-dir>/scripts/inject.py <dir>
   ```
   Add `--recursive` if the pages live in subfolders. The script is idempotent — safe to re-run. It also creates `<dir>/feedback/inbox.jsonl` and `<dir>/feedback/history.json` if missing.
3. **Pick a port.** Default 5050. Before starting, check what's there:
   ```
   curl -s --max-time 2 http://localhost:5050/info
   ```
   - JSON with `artifact_dir` matching this `<dir>` → reuse it, skip to step 5.
   - JSON with a *different* `artifact_dir` → port is held by another exploration. Either ask the user to free it (`lsof -ti:5050 | xargs kill`) or use port 5051, 5052, … (try the next port; tell the user the URL).
   - No response → port 5050 is free.
4. **Start the server as a long-running background process** using the host agent's shell/process facility:
   ```
   python <skill-dir>/lib/server.py <dir> --port <chosen>
   ```
   The server auto-shuts-down on parent death or 10 min of idle, so you don't need to manage its lifecycle.
5. **Tell the user the URL.** For example: `http://localhost:5050/index.html` (use whatever filename they actually have — `index.html`, `report.html`, etc.). If they have multiple pages, list the top-level ones.
6. **Start a Monitor on the inbox** so new comments notify the agent immediately:
   ```
   Monitor on path: <dir>/feedback/inbox.jsonl
   ```
   Do not poll; let the Monitor notification arrive.

## Responding to a feedback batch

When a new batch arrives in `inbox.jsonl`:
- Read the entry. Each comment has a stable `cf_id` and a selector pointing to the exact element/text the user commented on.
- Edit the relevant HTML files to address each comment. Wrap each modified region with `<span data-cf-change="ch-<short-slug>">…</span>` (or add `data-cf-change` to an existing wrapping element) so the post-reload walkthrough can find the change. One anchor per change.
- **Append** a new batch object to the end of `<dir>/feedback/history.json` (newest = last; the library walks from the end to find the latest batch). Schema:
  ```json
  {
    "batch_id": "b-<timestamp-or-slug>",
    "timestamp": "<ISO 8601>",
    "comments": [ /* echo back the inbox comments you addressed */ ],
    "changes": [
      {
        "id": "ch-<slug>",
        "in_response_to": ["<cf_id from inbox>"],
        "anchor": "ch-<slug>",   // must match a data-cf-change in the HTML
        "title": "short, concrete",
        "description": "longer prose (hidden in UI, just for the record)"
      }
    ]
  }
  ```
- The page polls `history.json`, sees the new batch, auto-reloads (scroll position preserved), and offers the user a walkthrough of the changes. The "processing…" banner clears automatically when any `in_response_to` matches a submitted comment id.

### Post in-flight status while you work

When you receive feedback and start working, POST a short status string so the user sees what you're doing instead of just a generic spinner:

```
POST /status
{"comment_id": "<cf_id from inbox>", "message": "Filing 2 receipts to Google Sheet (~30s)"}
```

To clear an entry early, POST the same `comment_id` with `message: null` or `""`. Entries are auto-pruned by the server after 10 min so a crashed agent never leaves a stuck "working" message.

`history.json` remains the source of truth for "done" — the status message is decoration only. The banner clears the moment a matching batch lands in `history.json`, regardless of whether you cleared the status entry.

## On startup in a directory that already has feedback

If you find `<dir>/feedback/inbox.jsonl` and `<dir>/feedback/history.json` and the skill has been invoked in this session:
1. Scan inbox for comment ids.
2. Scan history's `changes[*].in_response_to` union — those are already processed.
3. If unprocessed comments exist, tell the user the count and ask whether to process now.
4. Either way, start a Monitor on the inbox.

## Stop flow (user wants to kill the server)

1. Identify the port. If you started the server in this session, you know it. Otherwise check `curl -s http://localhost:5050/info` (try 5051, 5052 if 5050 returns nothing or a different artifact).
2. Kill it: `lsof -ti:<port> | xargs kill` (use `kill -9` only if a plain kill doesn't free the port within a few seconds — the server traps SIGTERM and exits cleanly).
3. Confirm: `lsof -i :<port>` should be silent.
4. If you also started a Monitor on the inbox in this session, it can remain active; the file will not receive new entries while the server is stopped.

Note: in most cases the user does not need to stop the server manually. It auto-shuts down when its parent agent process dies (usually within 5–10 seconds) or after 10 minutes without client requests. Manual stop is for reclaiming the port immediately.

## Update flow (user wants the latest lib/)

```
python <skill-dir>/scripts/update.py
```
Runs `git pull --ff-only` inside the skill dir. Requires git-clone install (the script tells the user how to re-install if not).

## Removal flow (clean static copy)

If the user wants their HTML back to a clean, server-independent state:
```
python <skill-dir>/scripts/inject.py <dir> --remove
```
Strips both tags from every `*.html`. Leaves the `feedback/` directory alone (delete manually if not wanted).

## Files in this skill

```
<skill-dir>/
├── SKILL.md              # this file (agent-facing)
├── README.md             # GitHub-facing docs (human readers)
├── LICENSE
├── lib/
│   ├── feedback.js       # client library: selection + commenting + tour
│   ├── feedback.css      # styles
│   └── server.py         # stdlib-only HTTP server
└── scripts/
    ├── inject.py         # idempotent tag injection / removal
    └── update.py         # git pull --ff-only
```

## Table rendering (auto-applied)

`feedback.js` automatically enhances every `<table>` on any served page. No markup changes needed.

| Feature | What it does |
|---|---|
| **Horizontal scroll** | Wraps table in `.cf-table-wrap` so wide tables scroll instead of overflowing |
| **Sticky first column** | First `td`/`th` stays visible while scrolling right; background auto-detected to match the page theme |
| **Sortable columns** | Click any `<th>` in a `<thead>` to sort ↑/↓; handles numbers (including K/M suffixes), text, and mixed content |

Tables inside the feedback UI root (`#claude-feedback-root`) are skipped. Enhancement is idempotent.

## Gotchas

- The injected `<link>` and `<script>` reference absolute paths `/lib/feedback.css` and `/lib/feedback.js`. These resolve through `server.py`, which routes `/lib/*` to the skill's own `lib/` directory. So pages only work when opened through this server — opening the HTML file directly in a browser will silently fail to load the feedback widget (the page itself still renders).
- `history.json` order matters: append (don't prepend). The library walks from the end to find the latest batch for the walkthrough.
- `anchor` values must match a `data-cf-change` attribute actually present in the HTML. Typos here cause "anchor not found" warnings post-reload.
