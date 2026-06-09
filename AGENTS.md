# AGENTS.md

## Remotes & branches

| Remote | URL |
|--------|-----|
| `origin` | `git@github.com:gabeschw/whatsapp-bridge.git` |
| `upstream` | `git@github.com:verygoodplugins/whatsapp-mcp.git` |

- `main` — tracks upstream bridge verbatim. No custom changes ever.
- `custom` (default) — your patches on top of `main`. All work here.

Sync from upstream: `git checkout main && git fetch upstream && git checkout upstream/main -- whatsapp-bridge/ && git commit` then `git checkout custom && git rebase main`.

## Commands

Run everything from `whatsapp-bridge/` (the Go module is there):

```bash
cd whatsapp-bridge
go run .                         # dev
go build -o whatsapp-bridge      # build binary
go test ./... -count=1           # tests (CGO required for sqlite3)
golangci-lint run                # lint (errcheck, govet, ineffassign, unused)
```

Module name: `whatsapp-client` (not `whatsapp-bridge`). CGO must be enabled (default on macOS).

## Architecture

Single Go binary — REST API backed by whatsmeow + SQLite.
- `main.go` — REST endpoints (`/api/send`, `/api/download`, `/api/react`, `/api/health`, `/api/typing`) + event loop + batch mode
- `webhook.go` — outgoing webhook payloads for inbound messages/reactions
- `auth.go` — bearer token auth + host check

Two SQLite DBs under `store/` (gitignored):
- `whatsapp.db` — whatsmeow session/contacts. Treat as opaque.
- `messages.db` — chats, messages, calls. Schema created by the bridge.

## Conventions

- **Never commit without user confirmation.** Not even in build mode. Stage and show, then ask.
- **Conventional commits** — `feat:`, `fix:`, `chore:`, `docs:`, `refactor:`, `test:`
- **One concern per commit.** No mixed changes.
- **No drive-by formatting.** Only reformat files you're already changing.
- **Security-sensitive changes** (auth, file paths, network bind, command exec) call out explicitly.
- **Cite files with `path:line`** when discussing code.

## Gotchas

- **JIDs.** WhatsApp uses `user@s.whatsapp.net` (DM), `user@g.us` (group), `numeric@lid` (link-ID). The `whatsmeow_lid_map` table maps LID ↔ phone. Sender resolution bugs are the most common issue.
- **Store directory** (`store/`) is gitignored but expected at runtime. First run creates it.
- **History sync** is controlled by the phone, not the bridge. The `--full-history-pair` flag requests more, but the phone decides.
- **Outbound calls** are invisible to linked devices — don't promise call capture for them.
- **Reactions** arrive as `ReactionMessage` stanzas, stored as `media_type="reaction"` with emoji in `content` and reacted-to message ID in `filename`.
- **Batch mode** (`--batch` flag) connects, collects messages until idle, then exits. Used by `sync_and_report`.
- **Media files** are stored at `store/{chat_jid}/{timestamp}-{messageID}.{ext}`. Don't hand-construct these paths — use `/api/download` endpoint.
- **`inserted_at`** column on messages table tracks when each row was first inserted. Updated only from `CURRENT_TIMESTAMP` on INSERT; never overwritten.
- **WAL mode** is NOT set on the SQLite connection strings (unlike the original monorepo). Both DBs use basic journaling.

## Env vars

| Variable | Default | Purpose |
|----------|---------|---------|
| `WHATSAPP_BRIDGE_PORT` | `8080` | REST API port |
| `WHATSAPP_BRIDGE_TOKEN` | `store/.bridge-token` | Bearer token |
| `WHATSAPP_MEDIA_ROOTS` | `~/.local/share/whatsapp-mcp/outbox` | Allowed media paths |
| `WEBHOOK_URL` | `http://localhost:8769/whatsapp/webhook` | Outgoing webhook |
| `FORWARD_SELF` | `true` | Forward self-sent messages |
