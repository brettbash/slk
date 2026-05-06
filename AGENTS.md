# AGENTS.md — slk

> Unofficial Slack TUI built in Go with [bubbletea](https://github.com/charmbracelet/bubbletea) and [lipgloss](https://github.com/charmbracelet/lipgloss).
> Module: `github.com/gammons/slk` · Go 1.26.1

---

## Essential Commands

| Command | Description |
|---------|-------------|
| `make build` | Build `bin/slk` from `./cmd/slk` |
| `make test` | Run all tests with race detector (`go test ./... -v -race`) |
| `make lint` | Run `golangci-lint run ./...` |
| `make run` | Build then execute `./bin/slk` |
| `make clean` | Remove `bin/` |
| `go install github.com/gammons/slk/cmd/slk@latest` | Install from source |

Releases are cut via [GoReleaser](https://goreleaser.com) (`.goreleaser.yaml`) on version tags (`v*`). The CI workflow is `.github/workflows/release.yml`.

---

## Code Organization

```
cmd/slk/              # Main entry point, CLI flags, workspace orchestration
internal/
  ui/                 # Bubbletea TUI: app model, keymaps, modes, panels
    messages/         # Message rendering, Block Kit parsing, selection
    sidebar/          # Channel sidebar (sections, collapse, staleness)
    compose/          # Message input / compose box
    channelfinder/    # Ctrl+T fuzzy channel finder
    workspace/        # Workspace rail (1-9 switcher)
    threadsview/      # Involved-threads list pane
    thread/           # Thread reply pane
    styles/           # Theme system (12 built-in + custom drop-ins)
    imgrender/        # Inline image rendering pipeline
    ...               # Overlays: reactionpicker, themeswitcher, presencemenu, etc.
  slack/              # Slack API wrapper, WS connection manager, event dispatch
    mrkdwn/           # Slack mrkdwn -> plain text conversion
  cache/              # SQLite persistence (messages, users, channels, threads, reactions)
  config/             # TOML config loading, workspace ordering, section matching
  service/            # Workspace manager, message service, section store, mute store
  image/              # Image protocols (kitty, sixel, half-block), cache, fetcher
  avatar/             # Avatar render cache (half-block / kitty)
  emoji/              # Emoji width probing & terminal calibration cache
  notify/             # OS desktop notifications
  slackfmt/           # MPDM name formatting
```

---

## Architecture & Control Flow

### Bubbletea v2 App Model

The TUI follows the standard bubbletea pattern: **Model → Update → View**.

- **`internal/ui/app.go`** defines `App` (the root `tea.Model`). It owns the layout (workspace rail + sidebar + messages + thread + compose + status bar) and routes all `tea.Msg` traffic.
- Messages between components are typed structs (e.g., `MessagesLoadedMsg`, `WorkspaceSwitchedMsg`, `NewMessageMsg`). Search `type.*Msg struct` in `internal/ui/app.go` for the full catalog.
- **Modal system**: `internal/ui/mode.go` defines modes (`ModeNormal`, `ModeInsert`, `ModeCommand`, `ModeSearch`, `ModeChannelFinder`, …). Key handling branches on the current mode.

### Main Orchestration (`cmd/slk/main.go`)

`main()` is large but linear:

1. Parse CLI flags (`--add-workspace`, `--version`, etc.).
2. Emoji width calibration (`internal/emoji`) — one-time terminal probe.
3. Load TOML config from `~/.config/slk/config.toml`.
4. Init SQLite cache (`~/.local/share/slk/cache.db`).
5. Load workspace tokens from `~/.local/share/slk/tokens/`.
6. Build `ui.App`, wire callbacks, start `tea.NewProgram(app)`.
7. **Launch each workspace in its own goroutine**:
   - `connectWorkspace()` → REST bootstrap (channels, users, unread counts).
   - `p.Send(ui.WorkspaceReadyMsg{...})` → TUI becomes interactive.
   - Start WebSocket via `slackclient.ConnectionManager` (auto-reconnect with exponential backoff).
   - Background tasks: custom emoji fetch, browseable public channels, unresolved DM resolution.

### Workspace Context & Callback Wiring

`WorkspaceContext` (in `cmd/slk/main.go`) holds all per-workspace state: Slack client, user maps, channels, custom emoji, mute store, section store, presence/DND state.

`wireCallbacks()` injects workspace-specific closures into `App` (send message, fetch history, upload files, etc.). When the user switches workspaces (`app.SetWorkspaceSwitcher`), callbacks are rewired to the new context.

### Real-Time Events

`rtmEventHandler` (`cmd/slk/main.go`) implements `slackclient.EventHandler`. It bridges Slack WebSocket events into bubbletea messages via `p.Send()`. Every incoming message is cached to SQLite immediately.

`internal/slack/connection.go` (`ConnectionManager`) manages the WebSocket lifecycle with exponential backoff (1s → 30s).

### Slack Client (`internal/slack/client.go`)

Wraps `slack-go/slack` with browser-cookie auth (`xoxc` token + `d` cookie). Uses a `SlackAPI` interface so tests can mock the underlying library. Handles enterprise grid workspaces via `apiBaseURL` discovered from `auth.test`.

### Cache Layer (`internal/cache/`)

SQLite via `modernc.org/sqlite` (CGO-free). WAL mode enabled.

Key tables: `workspaces`, `users`, `channels`, `messages`, `reactions`, `thread_summaries`.

**Cache-first rendering**: On channel open, `loadCachedMessages()` reads up to 50 messages from SQLite first (offline-capable). Then `fetchChannelMessages()` hits the network, writes through to SQLite, and sends `MessagesLoadedMsg`.

Important: `fetchChannelMessages` returns **`nil` on network failure**, `[]messages.MessageItem{}` on empty channel. The UI distinguishes these so a failed refresh doesn't wipe the cache view.

---

## Naming Conventions & Style

- **Packages**: all lowercase, no underscores. The Slack client package is `slackclient` (not `slack`) to avoid conflict with `github.com/slack-go/slack`.
- **Test files**: `*_test.go` in the same package. Table-driven tests preferred.
- **Mocks**: hand-rolled structs (e.g., `mockSlackAPI`, `mockEventHandler`). No mock generation framework.
- **Comments**: Extensive inline docs explaining *why*, especially around Slack API quirks, nil-safety, and fallback behavior.
- **Error handling**: Log with `log.Printf` and degrade gracefully — the TUI must never crash on a failed Slack API call.
- **TOML keys**: snake_case. Config struct tags match TOML keys exactly.

---

## Testing Approach

- Run with race detector: `go test ./... -v -race`
- Tests cover UI models (sidebar, messages, app), Slack client behavior, cache operations, config parsing, and event handlers.
- UI tests often construct models directly and call `Update()` with specific messages, then assert on the resulting state.
- The `cmd/slk/` tests exercise `rtmEventHandler` methods with nil `db`/`program` to verify guard clauses.

---

## Important Gotchas

### Debug Logging
- `SLK_DEBUG=1` → writes to `/tmp/slk-debug.log`. Without this, `log.SetOutput(io.Discard)` so stderr doesn't pollute the terminal.
- `SLK_DEBUG_WS=1` → logs unknown WebSocket event types (useful for reverse-engineering Slack events).

### Nil-Safe Stores
- `WorkspaceContext.SectionStore` and `WorkspaceContext.MuteStore` may be nil (bootstrap failure or feature disabled). **Always nil-check before use.**
- `SectionStore` nil → sidebar falls back to config-glob sections.
- `MuteStore` nil → all channels render as unmuted.

### Image Rendering
- Three protocols: **kitty graphics** (sharp pixels), **sixel**, **half-block** (universal fallback).
- Auto-detected via `imgpkg.Detect()`. If kitty is detected, a probe runs **before** bubbletea takes over the terminal (raw mode required).
- Avatars use kitty when available; inline images use the detected protocol.
- Slack file thumbnails require **both** `Authorization: Bearer <xoxc>` header **and** the workspace `d` cookie. Cross-workspace files (Slack Connect) try each registered team's auth.

### Clipboard
- `golang.design/x/clipboard` works on X11/macOS/Windows.
- **Wayland**: shells out to `wl-paste` from the `wl-clipboard` package. If `WAYLAND_DISPLAY` is set but `wl-paste` is missing, image paste is disabled with a warning.

### Emoji Width Calibration
- One-time ~30-second probe that renders emoji and measures terminal response.
- Cached per terminal emulator in `~/.cache/slk/emoji/`.
- Skip with `--no-emoji-probe`; force rerun with `--probe-emoji`.

### Slack Timestamp Format
- Timestamps are strings like `"1700000001.000000"` (seconds.microseconds). Parse by splitting on `.` and converting the first part to `int64` for `time.Unix()`.
- `ThreadTS` on a message indicates it belongs to a thread; `ReplyCount` is the number of replies.

### raw_json Cache Field
- Messages stored in SQLite include a `raw_json` column containing the full Slack message JSON.
- This allows reconstructing `attachments`, `blocks`, and `legacy_attachachments` on cache read with full fidelity.
- Pre-schema-migration rows have empty `raw_json` and render as text-only.

### DND State Semantics
- Slack's `dnd_enabled` flag means "has a DND schedule configured", NOT "currently in DND".
- Currently in DND only when: (a) manual snooze is active, or (b) current time falls inside the next scheduled window.
- `dnd.endDnd` ends the current session for the rest of the day; the schedule re-engages at its next window.

### Thread Reply Fetching
- `GetConversationReplies` returns the parent message at index 0, followed by replies.
- `fetchThreadReplies` strips the parent and returns replies-only.
- `loadCachedThreadReplies` includes the parent at index 0 — callers must slice `[1:]` when sending to `ThreadRepliesLoadedMsg`.

### Unresolved DMs
- On first connect, DM channels whose user isn't in the cached user list get an `UnresolvedDM` entry.
- A background goroutine calls `resolveUser()` to fetch their profile and sends `DMNameResolvedMsg` to update the sidebar live.
- Bot/app DMs may initially show as `Type="dm"` and flip to `Type="app"` asynchronously when the bot flag is discovered.

### Config Section Precedence
- `use_slack_sections` defaults to **true** (global and per-workspace).
- When enabled, Slack-native sidebar sections shadow any `[sections.*]` config globs.
- Workspace-level config (in `[workspaces.<slug>]`) overrides global config for that workspace.
- Workspace keys can be user-chosen slugs (with explicit `team_id`) or legacy raw team IDs.
