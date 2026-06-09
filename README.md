# WhatsApp Bridge

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Go 1.24+](https://img.shields.io/badge/go-1.24+-00ADD8.svg)](https://go.dev/)

Standalone Go bridge for WhatsApp Web — handles message send/receive, media download, webhook forwarding, and local SQLite storage via a REST API.

## Features

- **REST API** — send/receive messages, manage media, send reactions, query chats
- **Webhook Integration** — forward incoming messages and reactions to external services
- **Media Support** — auto-download incoming media, serve via download endpoint
- **Call History** — capture incoming voice/video calls into SQLite
- **Local Storage** — all data in local SQLite, no cloud dependency
- **Batch Sync** — connect, collect new messages, and exit (for scripts/cron)
- **Launchd Integration** — install as a macOS background service

## Installation

### Prerequisites

- Go 1.24+

### Quick Start

```bash
git clone https://github.com/gabeschw/whatsapp-bridge.git
cd whatsapp-bridge/whatsapp-bridge
go run .
```

On first start, the bridge prints and stores a local REST API token at
`store/.bridge-token`. Scan the QR code with WhatsApp on your phone to authenticate.

### Updating

Pull the latest changes, then:

| You changed | What to do |
|---|---|
| **Bridge code** and you run `go run .` | Nothing — `go run` recompiles each launch. Just restart the bridge. |
| **Bridge code** and you run a built binary | `cd whatsapp-bridge && go build -o whatsapp-bridge && ./whatsapp-bridge` |

Updates do **not** require re-pairing or deleting `whatsapp.db` — your session and message history are preserved.

## Configuration

Copy `.env.example` to `.env` and configure as needed:

| Variable | Default | Description |
|---|---|---|
| `WHATSAPP_BRIDGE_PORT` | `8080` | Port for Go bridge REST API |
| `WEBHOOK_URL` | `http://localhost:8769/whatsapp/webhook` | Webhook for incoming messages |
| `FORWARD_SELF` | `true` | Forward messages sent by self |
| `WHATSAPP_BRIDGE_TOKEN` | generated in `store/.bridge-token` | Bearer token required for bridge REST calls |
| `WHATSAPP_MEDIA_ROOTS` | `~/.local/share/whatsapp-mcp/outbox` | Path-list of directories allowed for outbound media files |

### Bridge authentication and media paths

The bridge requires bearer-token authentication for every `/api/*` request and
accepts only exact loopback Host headers for its configured port. This protects
the local REST API from other local processes and browser DNS-rebinding attacks.

On first start, the bridge generates a 256-bit token, writes it to
`store/.bridge-token` with owner-only permissions, and prints a
setup banner. For split deployments or containers that do not share the
repository directory, set `WHATSAPP_BRIDGE_TOKEN` in the environment.

Outbound `media_path` values are confined to `WHATSAPP_MEDIA_ROOTS`. The default
outbox is `~/.local/share/whatsapp-mcp/outbox`, created on bridge startup. Move
files there before calling the send endpoints, or set `WHATSAPP_MEDIA_ROOTS` to a
colon-separated list of absolute directories.

### Run automatically on macOS

macOS users can install optional per-user `launchd` jobs that start the Go
bridge at login and monitor it every 60 seconds for API health, disconnects, and
QR relink signals.

```bash
scripts/install-launchd-macos.sh
```

The installer builds `whatsapp-bridge/whatsapp-bridge` with `go build` when Go is
available, writes generated support files to
`~/Library/Application Support/whatsapp-mcp/`, writes LaunchAgents to
`~/Library/LaunchAgents/`, and writes logs to `~/Library/Logs/whatsapp-mcp/`.
It safely reloads only these labels:

- `com.whatsapp-mcp.bridge`
- `com.whatsapp-mcp.bridge-monitor`

To customize the launchd environment, export values before running the installer.
Re-run the installer after changing them.

```bash
export WHATSAPP_BRIDGE_PORT=8080
export WEBHOOK_URL=http://localhost:8769/whatsapp/webhook
export FORWARD_SELF=false
export WHATSAPP_MEDIA_ROOTS="$HOME/.local/share/whatsapp-mcp/outbox"
scripts/install-launchd-macos.sh
```

Verify the jobs and inspect logs:

```bash
launchctl print gui/$(id -u)/com.whatsapp-mcp.bridge
launchctl print gui/$(id -u)/com.whatsapp-mcp.bridge-monitor
tail -n 100 ~/Library/Logs/whatsapp-mcp/bridge.err.log
tail -n 100 ~/Library/Logs/whatsapp-mcp/monitor.err.log
```

The monitor sends a macOS notification once per failure type until recovery. It
alerts when the bridge LaunchAgent is unloaded, the token is missing, the health
endpoint is unreachable, WhatsApp is disconnected, or recent logs indicate that
QR relinking is needed.

Uninstall the generated LaunchAgents and support files with:

```bash
scripts/uninstall-launchd-macos.sh
```

Uninstall preserves `whatsapp-bridge/store/`, including WhatsApp session DBs,
message DBs, media, and `.bridge-token`. Logs are left in
`~/Library/Logs/whatsapp-mcp/` for manual cleanup.

### CLI flags (Go bridge)

| Flag | Default | Description |
| ---- | ------- | ----------- |
| `--full-history-pair` | `false` | Request full history at pair time. Only takes effect on a fresh pair (no existing `whatsapp.db`); no-op for already-paired sessions. The phone ultimately decides the actual history window sent — see [Requesting full history](#requesting-full-history) below. |
| `--batch` | `false` | Run in batch mode: connect, collect new messages until idle, then exit. Reads existing session; requires a prior pairing in normal mode. |
| `--batch-idle-timeout` | `15` | Seconds without new messages before batch sync completes. |
| `--batch-max-duration` | `300` | Hard limit in seconds for batch sync. |

### Requesting full history

whatsmeow's default pairing asks for "recent sync" — roughly the last 3 months, with the exact window decided by the phone. If you want to pull more history at pair time:

```bash
# Stop the bridge
launchctl bootout gui/$UID/com.whatsapp-mcp.bridge    # or however you manage it

# Back up, then remove the auth session (keeps messages.db intact)
cp whatsapp-bridge/store/whatsapp.db{,.bak}
rm whatsapp-bridge/store/whatsapp.db

# Re-pair with the flag
cd whatsapp-bridge
./whatsapp-bridge --full-history-pair
# Scan the QR with WhatsApp → Settings → Linked Devices → Link a Device
# Wait for "History sync complete" in the logs (can take 10-30 minutes)
# Ctrl+C when sync has quiesced, then restart under your normal process manager
```

Caveats:

- **The phone decides the actual cap.** The flag requests up to 10 years / 100 GB, but WhatsApp's iOS primary device enforces its own retention policy. iPad companion is documented at ~1 year max; other linked devices appear to follow similar logic.
- **Only effective on a fresh pair.** With `whatsapp.db` already present, no new pair handshake fires and the flag is a no-op.
- **Messages the phone has deleted are not recoverable** — auto-expire, low-storage cleanup, and manual delete all leave no trace for the phone to share.

## Call History

The bridge captures incoming WhatsApp voice and video calls live into a
dedicated `calls` table in `messages.db`. When a 1:1 call arrives
(`CallOffer`) or a group call is announced (`CallOfferNotice`), a row is
inserted with `result='in_progress'`. Subsequent `CallAccept` /
`CallReject` / `CallTerminate` events update the row — final result becomes
`answered`, `rejected`, `missed`, or `ended` depending on the event
sequence. See the state-machine comment above `StoreCallOffer` in `main.go`
for the exact transitions.

### Caveats

- **Outbound calls are not captured.** WhatsApp's primary device handles
  calls it initiates without notifying linked devices, so the bridge never
  sees an event for them.
- **Call results only reflect what the bridge saw.** If the bridge is
  offline when a call happens, the events are lost.
- **1:1 calls default to `call_type='voice'`.** `CallOffer` events don't
  expose media type directly (it's buried in the binary call data). Group
  calls via `CallOfferNotice` include a `Media` field and are recorded
  accurately as voice or video.

## Schema

### Chats

```sql
CREATE TABLE chats (
    jid TEXT PRIMARY KEY,
    name TEXT,
    last_message_time TIMESTAMP,
    ephemeral_expiration INTEGER NOT NULL DEFAULT 0,
    ephemeral_setting_timestamp INTEGER NOT NULL DEFAULT 0
);
```

### Messages

```sql
CREATE TABLE messages (
    id TEXT,
    chat_jid TEXT,
    sender TEXT,
    content TEXT,
    timestamp TIMESTAMP,
    is_from_me BOOLEAN,
    media_type TEXT,        -- 'image', 'video', 'audio', 'document', 'reaction'
    filename TEXT,          -- for media: saved filename; for reactions: reacted-to message ID
    url TEXT,
    media_key BLOB,
    file_sha256 BLOB,
    file_enc_sha256 BLOB,
    file_length INTEGER,
    deleted_at TIMESTAMP,
    quoted_message_id TEXT,
    inserted_at TIMESTAMP,  -- set on first insert, never overwritten
    PRIMARY KEY (id, chat_jid),
    FOREIGN KEY (chat_jid) REFERENCES chats(jid)
);
```

Reactions are stored as regular rows with `media_type='reaction'`, the emoji in `content`, and the target message ID in `filename`.

### Calls

```sql
CREATE TABLE calls (
    call_id TEXT,
    chat_jid TEXT,          -- group JID for group calls, call creator JID for 1:1
    from_jid TEXT,          -- JID of whoever started the call
    timestamp TIMESTAMP,    -- call start time
    is_from_me BOOLEAN,
    call_type TEXT,         -- 'voice' or 'video'
    is_group BOOLEAN,
    result TEXT,            -- 'in_progress' | 'answered' | 'ended' |
                            --   'missed' | 'rejected'
    duration_sec INTEGER,   -- computed when the call terminates
    ended_at TIMESTAMP,
    reason TEXT,            -- terminate reason string from whatsmeow
    PRIMARY KEY (call_id, chat_jid)
);
```

### Caveats

- **Outbound calls are not captured.** WhatsApp's primary device handles
  calls it initiates without notifying linked devices, so the bridge never
  sees an event for them.
- **Call results only reflect what the bridge saw.** If the bridge is
  offline when a call happens, the events are lost.
- **1:1 calls default to `call_type='voice'`.** `CallOffer` events don't
  expose media type directly (it's buried in the binary call data). Group
  calls via `CallOfferNotice` include a `Media` field and are recorded
  accurately as voice or video.

## Development

### Building

```bash
cd whatsapp-bridge
go build -o whatsapp-bridge
./whatsapp-bridge

# During development (avoids stale binaries)
go run .
```

### Linting

```bash
cd whatsapp-bridge
golangci-lint run
```

### Running Tests

```bash
cd whatsapp-bridge
go test ./...
```

## Troubleshooting

### Authentication Issues

- **QR Code Not Displaying**: Restart the bridge. Check terminal QR code support.
- **Device Limit Reached**: Remove a linked device from WhatsApp Settings > Linked Devices.
- **No Messages Loading**: Initial sync can take several minutes for large chat histories.
- **Out of Sync**: Back up `whatsapp-bridge/store`, then move
  `whatsapp-bridge/store/whatsapp.db` aside and re-authenticate. Keep
  `messages.db` unless you intentionally want to discard local message history.
- **Bridge returns 401 Unauthorized**: Restart the bridge so it creates
  `store/.bridge-token`. If your API caller cannot read that file, set
  `WHATSAPP_BRIDGE_TOKEN` to the same value in both environments.
- **Bridge returns 403 Forbidden for Host**: Use `WHATSAPP_API_URL` with
  `http://127.0.0.1:<port>/api`, `http://localhost:<port>/api`, or
  `http://[::1]:<port>/api`; custom hostnames and missing ports are rejected.
- **Bridge returns 403 Forbidden for media_path**: Move the file into
  `~/.local/share/whatsapp-mcp/outbox` or add its absolute parent directory to
  `WHATSAPP_MEDIA_ROOTS`.

### App State / LTHash Conflicts

Some WhatsApp account state is managed by whatsmeow in
`whatsapp-bridge/store/whatsapp.db`. If the bridge reports errors like:

```text
SendAppState failed: server returned error updating app state (regular_low):
<error code="409" text="conflict"/>
failed to verify patch v12345: mismatching LTHash
```

then WhatsApp's app-state patch chain for the linked device is out of sync.
This usually affects operations that write chat settings such as archive,
mute, or pin state. Incoming and outgoing messages may still work because
message storage lives separately in `messages.db`.

The practical recovery path is to reset the whatsmeow session and re-pair:

```bash
# Stop the bridge first.
launchctl bootout gui/$UID/com.whatsapp-mcp.bridge    # or however you manage it

# Back up the whole runtime store.
cp -a whatsapp-bridge/store whatsapp-bridge/store.bak.$(date +%Y%m%d%H%M%S)

# Reset only the whatsmeow session/app-state DB.
mv whatsapp-bridge/store/whatsapp.db whatsapp-bridge/store/whatsapp.db.lthash.bak

# Restart the bridge and scan the new QR code.
cd whatsapp-bridge
./whatsapp-bridge    # or `go run .` during development
```

Do not remove `whatsapp-bridge/store/messages.db` for this recovery unless you
also want to delete the local message archive.

### Windows

Windows requires CGO for go-sqlite3. Install [MSYS2](https://www.msys2.org/) and enable CGO:

```bash
go env -w CGO_ENABLED=1
go run .
```

## Credits

Derived from [verygoodplugins/whatsapp-mcp](https://github.com/verygoodplugins/whatsapp-mcp),
originally created by [Luke Harries](https://github.com/lharries/whatsapp-mcp).

## Links

- [whatsmeow](https://github.com/tulir/whatsmeow) — WhatsApp Web API library for Go
