# OpenClaw Architecture Diagrams

## System Architecture

### High-Level System Overview

```
┌──────────────────────────────────────────────────────────────────────┐
│                         OpenClaw Platform                            │
│                                                                      │
│  ┌────────────────────────────────────────────────────────────┐    │
│  │                    CLI Interface                             │    │
│  │  (openclaw agent, gateway, config, message, onboard, ...)  │    │
│  └────────────┬───────────────────────────────────────────────┘    │
│               │                                                      │
│  ┌────────────▼──────────────────────────────────────────────┐     │
│  │                  Gateway Server                            │     │
│  │  ┌──────────────────────────────────────────────────┐    │     │
│  │  │  WebSocket/HTTP Server (port 18789)              │    │     │
│  │  │  - Real-time message routing                     │    │     │
│  │  │  - Session management                            │    │     │
│  │  │  - Tool execution approvals                      │    │     │
│  │  │  - Control UI (web interface)                    │    │     │
│  │  └──────────────────────────────────────────────────┘    │     │
│  └────────────┬──────────────────────────────────────────────┘     │
│               │                                                      │
│  ┌────────────▼──────────────────────────────────────────────┐     │
│  │              Channel Manager                               │     │
│  │  ┌─────────┬─────────┬─────────┬─────────┬─────────┐    │     │
│  │  │WhatsApp │Telegram │ Discord │  Slack  │ Signal  │    │     │
│  │  └─────────┴─────────┴─────────┴─────────┴─────────┘    │     │
│  │  ┌─────────┬─────────┬─────────┬─────────┬─────────┐    │     │
│  │  │  Teams  │ Matrix  │  Zalo   │iMessage │ WebChat │    │     │
│  │  └─────────┴─────────┴─────────┴─────────┴─────────┘    │     │
│  └────────────┬──────────────────────────────────────────────┘     │
│               │                                                      │
│  ┌────────────▼──────────────────────────────────────────────┐     │
│  │               Session Router                               │     │
│  │  - Parses session keys (channel:kind:id)                 │     │
│  │  - Applies routing rules                                  │     │
│  │  - Maps to agent sessions                                 │     │
│  └────────────┬──────────────────────────────────────────────┘     │
│               │                                                      │
│  ┌────────────▼──────────────────────────────────────────────┐     │
│  │            Pi Agent Runtime                                │     │
│  │  ┌──────────────────────────────────────────────────┐    │     │
│  │  │  Multi-turn conversation engine                  │    │     │
│  │  │  - Session management (YAML storage)             │    │     │
│  │  │  - Model provider integration                    │    │     │
│  │  │  - Tool invocation                               │    │     │
│  │  │  - Response streaming                            │    │     │
│  │  └──────────────────────────────────────────────────┘    │     │
│  └────────────┬──────────────────────────────────────────────┘     │
│               │                                                      │
│  ┌────────────▼──────────────────────────────────────────────┐     │
│  │                Tool System                                 │     │
│  │  ┌─────────┬─────────┬─────────┬─────────┬─────────┐    │     │
│  │  │ Browser │  Shell  │ Message │ Canvas  │  Cron   │    │     │
│  │  │(Playwrg)│  (bash) │  Send   │ (A2UI)  │  Jobs   │    │     │
│  │  └─────────┴─────────┴─────────┴─────────┴─────────┘    │     │
│  └────────────────────────────────────────────────────────────┘     │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

### Message Flow Diagram

```
┌─────────────┐
│    User     │
│ (WhatsApp/  │
│  Telegram)  │
└──────┬──────┘
       │ "What's the weather?"
       │
       ▼
┌──────────────────────────────────────────────────────────────┐
│             Channel Plugin (WhatsApp)                         │
│  - Receives message via Baileys WebSocket                    │
│  - Parses message content                                    │
│  - Extracts sender info                                      │
└──────┬───────────────────────────────────────────────────────┘
       │ deps.onMessage({ 
       │   sessionKey: "whatsapp:direct:+1234567890",
       │   text: "What's the weather?",
       │   sender: { id: "+1234567890" }
       │ })
       ▼
┌──────────────────────────────────────────────────────────────┐
│        Gateway Server - Chat Handler                          │
│  (server-chat.ts)                                            │
│  - Receives message event                                    │
│  - Validates sender (check allowlist/pairing)                │
└──────┬───────────────────────────────────────────────────────┘
       │
       ▼
┌──────────────────────────────────────────────────────────────┐
│           Session Router                                      │
│  (routing/resolve-route.ts)                                  │
│  - Parses: "whatsapp:direct:+1234567890"                    │
│  - Checks routing rules                                      │
│  - Resolves to: agent="main", session="main"                │
└──────┬───────────────────────────────────────────────────────┘
       │ { agentId: "main", sessionKey: "main" }
       ▼
┌──────────────────────────────────────────────────────────────┐
│         Agent Event Handler                                   │
│  (server-chat.ts - createAgentEventHandler)                  │
│  - Loads session from storage                                │
│  - Adds message to transcript                                │
└──────┬───────────────────────────────────────────────────────┘
       │
       ▼
┌──────────────────────────────────────────────────────────────┐
│          Pi Agent Runtime                                     │
│  (agents/pi-embedded-runner/run.ts)                          │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ 1. Load model config (claude-opus-4)                   │ │
│  │ 2. Load tools (browser, bash, message_send, ...)      │ │
│  │ 3. Run agent with user message                         │ │
│  │ 4. Agent decides: "Need to call weather API"          │ │
│  │ 5. Invokes tool: browser_navigate(weather.com)        │ │
│  └────────────┬───────────────────────────────────────────┘ │
└───────────────┼──────────────────────────────────────────────┘
                │
                ▼
┌──────────────────────────────────────────────────────────────┐
│             Tool Executor - Browser                           │
│  (agents/pi-tools.js)                                        │
│  - Launches Playwright browser                               │
│  - Navigates to weather.com                                  │
│  - Extracts weather data                                     │
│  - Takes screenshot                                          │
└──────┬───────────────────────────────────────────────────────┘
       │ Tool result: { temp: "72°F", condition: "Sunny" }
       ▼
┌──────────────────────────────────────────────────────────────┐
│          Pi Agent Runtime (continued)                         │
│  - Agent receives tool result                                │
│  - Generates response: "It's 72°F and sunny today!"         │
│  - Streams chunks via onStream callback                      │
└──────┬───────────────────────────────────────────────────────┘
       │ Streaming chunks...
       ▼
┌──────────────────────────────────────────────────────────────┐
│        Gateway - Response Handler                             │
│  - Collects streaming chunks                                 │
│  - Broadcasts to WebSocket clients (Control UI)              │
│  - Buffers full response                                     │
└──────┬───────────────────────────────────────────────────────┘
       │ Full response: "It's 72°F and sunny today!"
       ▼
┌──────────────────────────────────────────────────────────────┐
│      Channel Plugin - Outbound                                │
│  (whatsapp outbound adapter)                                 │
│  - Formats message for WhatsApp                              │
│  - Calls Baileys sendMessage()                               │
└──────┬───────────────────────────────────────────────────────┘
       │
       ▼
┌─────────────┐
│    User     │
│ Receives:   │
│ "It's 72°F  │
│ and sunny!" │
└─────────────┘
```

### Session Management Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                 Session Storage Layer                         │
│                                                              │
│  ~/.openclaw/agents/                                        │
│    ├── main/                                                 │
│    │   ├── sessions/                                         │
│    │   │   ├── main.yaml                (default session)   │
│    │   │   ├── research.yaml            (custom session)    │
│    │   │   ├── discord-group-123.yaml   (channel session)   │
│    │   │   └── telegram-dm-user456.yaml (channel session)   │
│    │   └── config.yaml                                       │
│    │                                                         │
│    └── researcher/                                           │
│        ├── sessions/                                         │
│        │   ├── main.yaml                                     │
│        │   └── analysis.yaml                                 │
│        └── config.yaml                                       │
│                                                              │
└──────────────────────────────────────────────────────────────┘

Session File Format (YAML):

```yaml
id: main
agentId: main
created: 2026-01-15T10:30:00Z
updated: 2026-02-09T11:27:00Z

# Conversation transcript
transcript:
  - role: user
    content: "What's the weather?"
    timestamp: 2026-02-09T11:20:00Z
  
  - role: assistant
    content: "Let me check the weather for you."
    timestamp: 2026-02-09T11:20:05Z
  
  - role: tool
    name: browser_navigate
    input: { url: "https://weather.com" }
    output: { temp: "72°F", condition: "Sunny" }
    timestamp: 2026-02-09T11:20:10Z
  
  - role: assistant
    content: "It's 72°F and sunny today!"
    timestamp: 2026-02-09T11:20:15Z

# Session metadata
metadata:
  lastChannel: whatsapp
  lastSender: "+1234567890"
  messageCount: 42
```

### Session Key Routing

```
┌──────────────────────────────────────────────────────────────┐
│              Session Key Format                               │
│                                                              │
│  <channel>:<kind>:<id>                                       │
│                                                              │
│  Examples:                                                   │
│  - whatsapp:direct:+1234567890                              │
│  - telegram:direct:user123                                   │
│  - discord:group:channel-id-456                              │
│  - slack:group:C01234ABCDE                                   │
│  - signal:direct:+9876543210                                 │
│                                                              │
└──────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│             Routing Rules (openclaw.json5)                    │
│                                                              │
│  routing: {                                                  │
│    discord: [                                                │
│      {                                                       │
│        match: { kind: "group", id: "123456" },              │
│        agent: "researcher",                                  │
│        session: "main"                                       │
│      }                                                       │
│    ],                                                        │
│    telegram: [                                               │
│      {                                                       │
│        match: { kind: "direct", id: "user789" },            │
│        agent: "main",                                        │
│        session: "personal"                                   │
│      }                                                       │
│    ]                                                         │
│  }                                                           │
│                                                              │
└──────────────────────────────────────────────────────────────┘

Routing Flow:

Input: "discord:group:123456"
       ↓
Parse: { channel: "discord", kind: "group", id: "123456" }
       ↓
Match: routing.discord[0] matches { kind: "group", id: "123456" }
       ↓
Route: { agentId: "researcher", sessionKey: "main" }
       ↓
Session Path: ~/.openclaw/agents/researcher/sessions/main.yaml
```

### Channel Plugin Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                 Channel Plugin Interface                      │
│                                                              │
│  ┌────────────────────────────────────────────────────┐     │
│  │ ChannelPlugin                                      │     │
│  │  - id: "whatsapp" / "telegram" / "discord" / ...  │     │
│  │  - meta: { name, icon }                           │     │
│  │  - capabilities: { dm, groups, voice, files }     │     │
│  └────────────────────────────────────────────────────┘     │
│                                                              │
│  ┌────────────────────────────────────────────────────┐     │
│  │ Adapters (15+ optional interfaces)                │     │
│  │                                                    │     │
│  │  ├─ gateway: { start, stop }                      │     │
│  │  │   - Start/stop channel runtime                 │     │
│  │  │   - Handle incoming messages                   │     │
│  │  │                                                 │     │
│  │  ├─ outbound: { sendMessage, sendFile }          │     │
│  │  │   - Send messages to users                     │     │
│  │  │   - Send files/media                           │     │
│  │  │                                                 │     │
│  │  ├─ auth: { login, logout }                       │     │
│  │  │   - Handle auth flows                          │     │
│  │  │   - Token management                           │     │
│  │  │                                                 │     │
│  │  ├─ onboarding: { wizard }                        │     │
│  │  │   - Interactive setup                          │     │
│  │  │   - Account configuration                      │     │
│  │  │                                                 │     │
│  │  └─ ... (status, probe, audit, etc.)             │     │
│  │                                                    │     │
│  └────────────────────────────────────────────────────┘     │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### Configuration Architecture

```
┌──────────────────────────────────────────────────────────────┐
│            Configuration Loading Pipeline                     │
│                                                              │
│  ~/.openclaw/openclaw.json5                                 │
│         ↓                                                    │
│  ┌─────────────────────────────┐                            │
│  │ 1. Read JSON5 file          │                            │
│  │    - Supports comments      │                            │
│  │    - Trailing commas OK     │                            │
│  └──────────┬──────────────────┘                            │
│             ↓                                                │
│  ┌─────────────────────────────┐                            │
│  │ 2. Resolve includes         │                            │
│  │    - Load referenced files  │                            │
│  │    - Merge configurations   │                            │
│  └──────────┬──────────────────┘                            │
│             ↓                                                │
│  ┌─────────────────────────────┐                            │
│  │ 3. Substitute env vars      │                            │
│  │    - ${VAR} → value         │                            │
│  │    - ${VAR:-default}        │                            │
│  └──────────┬──────────────────┘                            │
│             ↓                                                │
│  ┌─────────────────────────────┐                            │
│  │ 4. Validate with Zod        │                            │
│  │    - Type checking          │                            │
│  │    - Schema validation      │                            │
│  └──────────┬──────────────────┘                            │
│             ↓                                                │
│  ┌─────────────────────────────┐                            │
│  │ 5. Apply defaults           │                            │
│  │    - Agent defaults         │                            │
│  │    - Model defaults         │                            │
│  │    - Session defaults       │                            │
│  └──────────┬──────────────────┘                            │
│             ↓                                                │
│  ┌─────────────────────────────┐                            │
│  │ 6. Load plugin configs      │                            │
│  │    - Channel plugins        │                            │
│  │    - Tool plugins           │                            │
│  └──────────┬──────────────────┘                            │
│             ↓                                                │
│  Final OpenClawConfig object                                │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### Tool System Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                    Tool System                                │
│                                                              │
│  Agent invokes tool                                          │
│         ↓                                                    │
│  ┌─────────────────────────────┐                            │
│  │ Tool Executor               │                            │
│  │ (pi-tools.js)               │                            │
│  │  - Validates arguments      │                            │
│  │  - Checks permissions       │                            │
│  │  - Executes tool            │                            │
│  └──────────┬──────────────────┘                            │
│             ↓                                                │
│  ┌─────────────────────────────────────────────────────┐   │
│  │            Tool Categories                           │   │
│  │                                                      │   │
│  │  ┌───────────────────────────────────────────────┐  │   │
│  │  │ Browser Tools (Playwright)                    │  │   │
│  │  │  - navigate: Go to URL                        │  │   │
│  │  │  - click: Click element                       │  │   │
│  │  │  - type: Type text                            │  │   │
│  │  │  - screenshot: Capture page                   │  │   │
│  │  │  - evaluate: Run JavaScript                   │  │   │
│  │  └───────────────────────────────────────────────┘  │   │
│  │                                                      │   │
│  │  ┌───────────────────────────────────────────────┐  │   │
│  │  │ Shell Tools                                   │  │   │
│  │  │  - bash: Execute shell commands               │  │   │
│  │  │  - read_bash: Read command output             │  │   │
│  │  │  - write_bash: Send input to command          │  │   │
│  │  │  - stop_bash: Stop running command            │  │   │
│  │  └───────────────────────────────────────────────┘  │   │
│  │                                                      │   │
│  │  ┌───────────────────────────────────────────────┐  │   │
│  │  │ Messaging Tools                               │  │   │
│  │  │  - message_send: Send message to channel      │  │   │
│  │  │  - message_list: List messages                │  │   │
│  │  │  - message_reply: Reply to message            │  │   │
│  │  └───────────────────────────────────────────────┘  │   │
│  │                                                      │   │
│  │  ┌───────────────────────────────────────────────┐  │   │
│  │  │ Canvas Tools (A2UI)                           │  │   │
│  │  │  - canvas_render: Render HTML to canvas       │  │   │
│  │  │  - canvas_update: Update canvas content       │  │   │
│  │  │  - canvas_clear: Clear canvas                 │  │   │
│  │  └───────────────────────────────────────────────┘  │   │
│  │                                                      │   │
│  │  ┌───────────────────────────────────────────────┐  │   │
│  │  │ Cron Tools                                    │  │   │
│  │  │  - cron_schedule: Schedule recurring job      │  │   │
│  │  │  - cron_list: List scheduled jobs             │  │   │
│  │  │  - cron_delete: Delete scheduled job          │  │   │
│  │  └───────────────────────────────────────────────┘  │   │
│  │                                                      │   │
│  └──────────────────────────────────────────────────────┘   │
│             ↓                                                │
│  Tool result returned to agent                              │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### Multi-Agent Routing

```
┌──────────────────────────────────────────────────────────────┐
│               Multi-Agent Architecture                        │
│                                                              │
│  ┌────────────────────────────────────────────────────┐     │
│  │ Gateway receives message from Discord #research    │     │
│  │ Session key: "discord:group:research-channel-id"  │     │
│  └────────────┬───────────────────────────────────────┘     │
│               ↓                                              │
│  ┌────────────────────────────────────────────────────┐     │
│  │ Routing Rule Matched:                             │     │
│  │  - Channel: discord                                │     │
│  │  - Match: { kind: "group", id: "research-..." }   │     │
│  │  - Route to: agent="researcher"                    │     │
│  └────────────┬───────────────────────────────────────┘     │
│               ↓                                              │
│  ┌────────────────────────────────────────────────────┐     │
│  │ Load "researcher" Agent                            │     │
│  │  - Model: claude-opus-4                            │     │
│  │  - Thinking: high                                  │     │
│  │  - Tools: [browser, bash]                          │     │
│  │  - System: "You are a research assistant..."      │     │
│  └────────────┬───────────────────────────────────────┘     │
│               ↓                                              │
│  ┌────────────────────────────────────────────────────┐     │
│  │ Load Session: researcher/main.yaml                 │     │
│  │  - Contains research conversation history          │     │
│  │  - Separate from "main" agent sessions             │     │
│  └────────────┬───────────────────────────────────────┘     │
│               ↓                                              │
│  ┌────────────────────────────────────────────────────┐     │
│  │ Run Agent with User Message                        │     │
│  │  - Agent has context from previous research        │     │
│  │  - Uses high thinking for complex reasoning        │     │
│  │  - Can invoke browser/bash tools                   │     │
│  └────────────┬───────────────────────────────────────┘     │
│               ↓                                              │
│  ┌────────────────────────────────────────────────────┐     │
│  │ Response sent back to Discord #research            │     │
│  └────────────────────────────────────────────────────┘     │
│                                                              │
│  Meanwhile, a WhatsApp message to "main" agent:             │
│                                                              │
│  ┌────────────────────────────────────────────────────┐     │
│  │ Gateway receives message from WhatsApp             │     │
│  │ Session key: "whatsapp:direct:+1234567890"        │     │
│  └────────────┬───────────────────────────────────────┘     │
│               ↓                                              │
│  ┌────────────────────────────────────────────────────┐     │
│  │ No specific routing rule → default to "main"       │     │
│  └────────────┬───────────────────────────────────────┘     │
│               ↓                                              │
│  ┌────────────────────────────────────────────────────┐     │
│  │ Load "main" Agent                                  │     │
│  │  - Model: claude-sonnet-4                          │     │
│  │  - Thinking: medium                                │     │
│  │  - Tools: [browser, bash, message_send, ...]      │     │
│  │  - System: "You are a helpful assistant..."       │     │
│  └────────────┬───────────────────────────────────────┘     │
│               ↓                                              │
│  ┌────────────────────────────────────────────────────┐     │
│  │ Load Session: main/main.yaml                       │     │
│  │  - Completely separate from researcher session     │     │
│  │  - Different conversation history                  │     │
│  └────────────┬───────────────────────────────────────┘     │
│               ↓                                              │
│  ┌────────────────────────────────────────────────────┐     │
│  │ Response sent back to WhatsApp                     │     │
│  └────────────────────────────────────────────────────┘     │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

## Component Interaction Patterns

### Plugin Registration Flow

```
App Startup
    ↓
Load plugins from /extensions/
    ↓
For each plugin:
    ├─ Load plugin entry point (index.ts)
    ├─ Call plugin.register(api)
    ├─ Plugin calls api.registerChannel()
    └─ Channel added to registry
    ↓
Gateway starts channels based on config
```

### WebSocket Communication

```
Client (Control UI/CLI)
    ↓
Connect to ws://localhost:18789
    ↓
Send subscription request: { type: "subscribe", topics: ["chat", "status"] }
    ↓
Gateway registers subscription
    ↓
Events occur (message received, status change)
    ↓
Gateway broadcasts to subscribed clients
    ↓
Client receives real-time updates
```

### Pairing Flow (Security)

```
Unknown user sends DM to WhatsApp
    ↓
Gateway checks dmPolicy (default: "pairing")
    ↓
User not in allowlist
    ↓
Generate pairing code (e.g., "ABC123")
    ↓
Send auto-reply: "Please have the admin approve with: openclaw pairing approve whatsapp ABC123"
    ↓
Admin runs: openclaw pairing approve whatsapp ABC123
    ↓
User added to allowlist
    ↓
User can now send messages
```

## Technology Stack Details

### Language & Runtime
- **TypeScript 5.9+**: Strict mode, ESM
- **Node.js ≥22**: Required runtime
- **Bun**: Optional for dev (faster TypeScript execution)

### Build & Package
- **tsdown**: Build tool (outputs to dist/)
- **pnpm**: Package manager (workspace support)
- **Rolldown**: Bundler for UI

### Testing
- **Vitest**: Test runner (fast, ESM-native)
- **@vitest/coverage-v8**: Code coverage
- **Playwright**: Browser testing

### CLI & Terminal
- **Commander.js**: CLI framework
- **@clack/prompts**: Interactive prompts
- **osc-progress**: Progress bars
- **chalk**: Terminal colors

### Messaging Libraries
- **grammY**: Telegram (modern, type-safe)
- **discord.js**: Discord API
- **@slack/bolt**: Slack apps
- **@whiskeysockets/baileys**: WhatsApp Web (no official API)
- **signal-utils**: Signal protocol
- **@line/bot-sdk**: LINE messaging

### Data & Validation
- **zod**: Runtime type validation
- **@sinclair/typebox**: JSON Schema generation
- **ajv**: JSON Schema validation
- **yaml**: YAML parsing (session storage)
- **json5**: JSON5 parsing (config)

### Web & HTTP
- **Hono**: Fast web framework (Control UI)
- **Express**: HTTP server (legacy)
- **ws**: WebSocket server
- **undici**: Modern HTTP client

### Browser & Media
- **playwright-core**: Headless browser
- **sharp**: Image processing
- **pdfjs-dist**: PDF parsing
- **linkedom**: Server-side DOM

### Platform Apps
- **SwiftUI**: macOS/iOS apps
- **Kotlin + Jetpack Compose**: Android app
- **Lit**: Web components (Control UI)

This comprehensive architecture overview should give you a complete understanding of how OpenClaw works!
