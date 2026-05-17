# AGENTS.md

## Project Overview

ClawBench is a mobile-first AI workstation wrapping AI CLI tools (CodeBuddy, Claude Code, OpenCode, Gemini CLI, Codex, Qoder CLI, VeCLI, DeepSeek TUI, Pi) into a web-accessible platform. Go backend shells out to CLI tools and streams JSON output via SSE; Vue 3 frontend renders the streamed events in real time. Supports SSH tunnel-based port forwarding for remote/mobile access and a scheduled task (cron) system for recurring AI execution.

## Build & Run Commands

```bash
./build.sh                # Full build (Go binary + Vue frontend)
./build.sh --windows      # Cross-compile: Windows amd64
./build.sh --linux        # Cross-compile: Linux amd64
./build.sh --darwin       # Cross-compile: macOS arm64

./dev-server.sh           # Dev mode (Vite HMR proxy to production backend's dev HTTP port)
./dev-server.sh --fg      #   foreground
./dev-server.sh --stop    #   stop
./dev-server.sh --restart #   restart

./server.sh               # Production (port 20000)
./server.sh --fg          #   foreground
./server.sh --stop        #   stop

go build -o clawbench ./cmd/server   # Go binary only
go test ./...                        # All Go tests
go test ./internal/ai/...            # Package-specific
npm test                             # Vitest (all frontend tests)

# Android APK (requires JDK 17)
cd android && JAVA_HOME=/usr/lib/jvm/jdk-17.0.12 ./gradlew assembleDebug    # Debug APK
cd android && JAVA_HOME=/usr/lib/jvm/jdk-17.0.12 ./gradlew assembleRelease  # Release APK
```

## Architecture

### Backend (Go)

**Entry point:** `cmd/server/main.go` — config → port → LoadAgents → auto-discovery → scheduler init.

**Layers:**
- `internal/handler/` — HTTP handlers, SSE streaming (`chat_stream.go`), CRUD endpoints. All `/api/` routes use `middleware.Auth` (localhost bypass for CLI).
- `internal/service/` — Business logic: chat persistence, scheduler (cron via `robfig/cron/v3`), SQLite, ProxyRegistry, session runtime.
- `internal/ai/` — AI backend abstraction. `AIBackend` interface with `ExecuteStream()`. `CLIBackend` is the shared base; each backend provides CLI args and a `LineParser`. `AutoResumeBackend` wraps claude/codebuddy/qoder/deepseek/pi — detects ExitPlanMode and auto-resumes with "继续". `NewBackend()` factory in `factory.go`.
- `internal/model/` — Data models, config structs, structured errors (`NotFound`, `Forbidden`, `Internal`), auto-discovery of AI CLIs.
- `internal/cli/` — CLI subcommands for AI agent self-service: `task` (CRUD + trigger + `list-exec`), `rag` (search), `migrate`.
- `internal/middleware/` — Auth, request logging, panic recovery, request ID.
- `internal/speech/` — TTS providers: MiniMax (cloud), Edge TTS (cloud, free), Piper/Kokoro/MOSS-Nano (local).
- `internal/summarize/` — Text summarization for TTS/task summaries. Supports AI backend CLIs, OpenAI/Anthropic HTTP APIs, and simple text cleanup.
- `internal/ssh/` — SSH tunnel server with direct-tcpip channels, password auth, auto-persisted host key.
- `internal/rag/` — RAG history memory: DuckDB vector store, Ollama BGE-M3 embeddings, chunking, indexing, search, cleanup.
- `internal/terminal/` — Interactive web terminal: PTY sessions, ring buffer replay, concurrent session management.
- `internal/push/` — Push notification abstraction. `JPushClient` wraps JPush API (SDK 6.1.0). AppKey fetched at runtime from server config (not baked into APK). `AppKey()` getter exposed via `/api/push/config` endpoint.
- `internal/ws/` — WebSocket event channel (`/api/ai/events/ws`). Replaces all frontend polling for session/task/queue status. `Manager` handles subscriptions, event broadcast, JPush fallback when client is disconnected, and buffered event replay on reconnect. Push Registration ID is stored at login level via HTTP API (not per-WS-connection).

**Agent system:** `config/agents/*.yaml` defines agents (id, backend, model, system_prompt). Auto-discovery generates configs when none exist (one-time). `config/rules.md` is injected into every agent's system prompt — placeholders `{{AVAILABLE_AGENTS}}`, `{{PORT}}`, `{{PROJECT_PATH}}` are replaced dynamically.

**Data flow (chat):** POST `/api/ai/chat` → resolve agent → `NewBackend()` → `ExecuteStream()` spawns CLI → `LineParser` → SSE events → SQLite persistence.

**Data flow (real-time events):** Backend state changes (session completed, task finished, queue updated) → `ws.Manager.BroadcastEvent()` → if WebSocket connected: send via WS; if disconnected + JPush configured + Registration ID available: send push notification; if disconnected + no JPush: buffer for replay on reconnect. Client sends `ack` for each event. JPush Registration ID is reported via `POST /api/push/register` (login-level lifecycle, not per-WS-connection) and persisted in `Manager` so it survives WS disconnects.

**Push notification flow:** Android `MainActivity.fetchPushConfig()` → `GET /api/push/config` (unauthenticated) → if `jpush_enabled`: `JPushInterface.init(context, config)` with runtime AppKey. After login, frontend calls `POST /api/push/register` (authenticated) with JPush Registration ID — this is a login-level lifecycle event persisted server-side, not tied to any individual WebSocket connection. When app goes to background: if push available → disconnect WebSocket (push delivers events); if push NOT available → keep WebSocket alive in background for real-time events.

**Scheduled tasks:** POST `/api/tasks` → cron trigger → creates chat session → executes AI backend → writes assistant message. `CLAWBENCH_SCHEDULED=1` env var for anti-recursion. AI agents manage tasks via `clawbench task` CLI. Zombie executions auto-cleaned on startup.

**Soft-delete:** Chat sessions/messages use `deleted=1` (not `DELETE FROM`) so RAG can still search them. `CleanupWorker` purges soft-deleted data past retention. Scheduled tasks use hard delete.

### Frontend (Vue 3 + TypeScript)

**Source root:** `web/src/` — No Vue Router, drawer-based single-page layout.

**State management:** Single `reactive()` store in `stores/app.ts` — no Pinia/Vuex.

**Key composables (chat):** `useChatSession` (CRUD), `useChatStream` (SSE + reconnect + polling fallback), `useChatRender` (block parsing + coalescing), `useAutoSpeech` (TTS), `useQuickSend` (SQLite CRUD), `useReconnect` (generic exponential backoff).

**Key composables (events):** `useGlobalEvents` (WebSocket event channel singleton — session/task/queue status, push availability check, background visibility strategy: disconnect WS when push available, keep alive when not).

**Key composables (terminal):** `useTerminalSession` (WebSocket lifecycle), `useTerminalKeys` (modifier state machine), `useTerminalGestures` (touch swipe/pinch), `useTerminalViewport` (xterm.js + soft keyboard avoidance).

**Key components:** `ChatPanel`, `FileManager`/`FileViewer`, `TaskTab` (4-level breadcrumb), `TerminalPanel` (xterm.js + virtual keys + gestures), `GitGraph`, `BottomSheet`, `Lightbox`.

**Vite config:** `hljsThemeWrapper` plugin for light/dark theme coexistence. Root `web/`, output `public/`. Path alias `@` → `web/src/`.

## Key Patterns

- **Module-level singletons:** `useAutoSpeech()`, `useToast()` — instantiate once, share state via module-level refs.
- **SSE reconnection:** 3 attempts → fallback to HTTP polling (2s). 15s heartbeat, 30s timeout. `online` event triggers immediate reconnect.
- **Block coalescing:** Text/thinking events merge into last block of same type; `tool_use` acts as boundary.
- **AutoResumeBackend:** ExitPlanMode → cancel → resume with "继续". Emits `resume_split` for DB finalization.
- **Cancel reason tracking:** `"user"` (explicit) vs `"disconnect"` (SSE gone). `ForceCancelSession` kills zombie CLI processes.
- **Green portable deployment:** All runtime data under `.clawbench/` next to binary. Delete that dir = clean uninstall. Copy binary dir for multi-instance isolation.
- **Zero-config startup:** `config/config.yaml` optional. `model.ApplyDefaults()` fills sensible defaults. Auto-generated password persisted to `.clawbench/auto-password`.
- **Touch device CSS:** Use `@media (hover: hover)` to scope `:hover` styles.
- **Structured errors:** `model.NotFound()`, `model.Forbidden()`, `model.Internal()` constructors.
- **Push-aware background strategy:** `useGlobalEvents.pushAvailable` ref tracks JPush availability. On `visibilitychange` to hidden: if `pushAvailable=true` → disconnect WebSocket (JPush delivers events); if `pushAvailable=false` → keep WebSocket alive in background. Same logic in `useChatStream` for SSE connections.
- **Runtime JPush AppKey:** JPush AppKey is NOT baked into APK. Android fetches it from `GET /api/push/config` at startup and initializes JPush via `JPushConfig.setjAppKey()` + `JPushInterface.init(context, config)`. Unauthenticated endpoint — only exposes `jpush_enabled` and `jpush_app_key`, no secrets.
- **Login-level push registration:** JPush Registration ID is reported via `POST /api/push/register` (authenticated, cookie-based) — not via WS `register` message. The Registration ID is a login-level lifecycle event persisted in `ws.Manager`, surviving WS disconnects/reconnects. WS authentication uses only Cookie (HTTP Upgrade carries cookies), no `?token=` fallback needed.

## Configuration

`config/config.yaml` is entirely optional. See `config/config.example.yaml`.

| Section | Key options |
|---------|------------|
| Server | `port` (20000), `host`, `log_level` ("info"), `watch_dir`, `password` (auto-UUID), `default_agent`, `dev_port` (0=auto, Port+1 when TLS) |
| Upload | `upload.max_size_mb`, `upload.max_files` |
| Chat UI | `chat.initial_messages`, `chat.page_size`, `chat.collapsed_height`, `chat.system_prompt_interval` (10) |
| Session | `session.max_count` |
| TLS | `tls.enabled`, `tls.cert_file`, `tls.key_file` |
| TTS | `tts.engine`, `tts.summarize_backend`, `tts.summarize_model`, `tts.speed`, `tts.voice` |
| Proxy | `proxy.enabled`, `proxy.allowed_ports` |
| SSH | `ssh.enabled`, `ssh.port`, `ssh.host_key` |
| RAG | `rag.enabled`, `rag.ollama_base_url`, `rag.ollama_model` (bge-m3), `rag.chunk_size` (512), `rag.retention_days` (90) |
| Terminal | `terminal.enabled` (true), `terminal.idle_timeout` (10m), `terminal.max_sessions` (10) |
| Tasks | `tasks.summarize_backend`, `tasks.summarize_model` |
| Push | `push.jpush.enabled`, `push.jpush.app_key`, `push.jpush.master_secret` |
