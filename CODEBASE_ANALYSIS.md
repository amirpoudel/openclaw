# OpenClaw Codebase Analysis - Complete Technical Guide

## Table of Contents
1. [Project Overview](#project-overview)
2. [Architecture Overview](#architecture-overview)
3. [Directory Structure](#directory-structure)
4. [Core Components](#core-components)
5. [Data Flow](#data-flow)
6. [Key Technologies](#key-technologies)
7. [Development Workflow](#development-workflow)
8. [Extension Points](#extension-points)

---

## Project Overview

**OpenClaw** is a personal AI assistant platform that you run on your own devices. It's designed to be:

- **Multi-channel**: Connects to WhatsApp, Telegram, Slack, Discord, Google Chat, Signal, iMessage, Microsoft Teams, Matrix, Zalo, and WebChat
- **Local-first**: Runs on your own infrastructure (laptop, server, Pi)
- **Agentic**: Uses the Pi embedded agent runtime for multi-turn AI conversations
- **Extensible**: Plugin-based architecture for channels, tools, and skills

**Key Features:**
- Voice wake (macOS/iOS/Android)
- Live Canvas for visual workspace
- Multi-agent routing (route different channels to different AI agents)
- Session management (persistent conversation history)
- Tool system (browser control, shell, messaging, cron jobs)
- Onboarding wizard for easy setup

---

## Architecture Overview

### High-Level Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                    CLI Entry Point                            │
│  (entry.ts → run-main.ts → build-program.ts)                 │
└────────────────┬─────────────────────────────────────────────┘
                 │
        ┌────────┴────────┐
        │                 │
   ┌────▼────┐      ┌────▼────────────────┐
   │ Gateway │      │ CLI Commands        │
   │ Server  │      │ - agent             │
   │         │      │ - config            │
   │ WS/HTTP │      │ - sessions          │
   └────┬────┘      │ - message send      │
        │           │ - onboard           │
        │           └─────────────────────┘
        │
   ┌────▼─────────────────────────────────┐
   │  Channel Manager                      │
   │  ┌──────────┬──────────┬──────────┐  │
   │  │ WhatsApp │ Telegram │ Discord  │  │
   │  └──────────┴──────────┴──────────┘  │
   │  ┌──────────┬──────────┬──────────┐  │
   │  │  Slack   │  Signal  │  Matrix  │  │
   │  └──────────┴──────────┴──────────┘  │
   └────┬─────────────────────────────────┘
        │
   ┌────▼─────────────────────────────────┐
   │  Session Router                       │
   │  - Routes messages to agent sessions  │
   │  - Classifies session types           │
   │  - Manages session lifecycle          │
   └────┬─────────────────────────────────┘
        │
   ┌────▼─────────────────────────────────┐
   │  Pi Agent Runtime                     │
   │  ┌────────────────────────────────┐  │
   │  │ Agent Session Manager          │  │
   │  │ - Multi-turn conversations     │  │
   │  │ - Tool invocation              │  │
   │  │ - Streaming responses          │  │
   │  └────────────────────────────────┘  │
   └────┬─────────────────────────────────┘
        │
   ┌────▼─────────────────────────────────┐
   │  Tool System                          │
   │  - Browser (Playwright)               │
   │  - Shell commands                     │
   │  - Message sending                    │
   │  - Canvas control                     │
   │  - Cron jobs                          │
   └───────────────────────────────────────┘
```

### Component Interaction Flow

```
User Message (via WhatsApp/Telegram/etc.)
    ↓
Channel Plugin receives message
    ↓
Channel Manager forwards to Gateway
    ↓
Session Router determines target session
    ↓
Pi Agent Runtime processes message
    ↓
Agent may invoke tools (browser, shell, etc.)
    ↓
Agent generates response
    ↓
Response streamed back through Gateway
    ↓
Channel Plugin delivers to user
```

---

## Directory Structure

```
openclaw/
├── src/                          # TypeScript source code
│   ├── cli/                      # CLI implementation
│   │   ├── program/              # Command registry & builder
│   │   ├── gateway-cli.ts        # Gateway commands
│   │   ├── agent-cli.ts          # Agent commands
│   │   ├── config-cli.ts         # Config commands
│   │   └── ...
│   │
│   ├── gateway/                  # Gateway server
│   │   ├── server.impl.ts        # Main server implementation
│   │   ├── server-chat.ts        # Chat event handling
│   │   ├── server-channels.ts    # Channel lifecycle
│   │   ├── server-ws-runtime.ts  # WebSocket runtime
│   │   └── ...
│   │
│   ├── agents/                   # Agent system
│   │   ├── pi-embedded-runner/   # Pi runtime integration
│   │   ├── pi-tools.js           # Tool definitions
│   │   └── ...
│   │
│   ├── channels/                 # Channel plugin system
│   │   ├── plugins/              # Plugin types & adapters
│   │   └── ...
│   │
│   ├── config/                   # Configuration system
│   │   ├── types.ts              # Config types
│   │   ├── io.ts                 # Config loading/saving
│   │   ├── validation.ts         # Schema validation
│   │   └── ...
│   │
│   ├── routing/                  # Session routing
│   │   ├── session-key.ts        # Session key parsing
│   │   ├── resolve-route.ts      # Route resolution
│   │   └── ...
│   │
│   ├── discord/                  # Discord integration
│   ├── telegram/                 # Telegram integration
│   ├── whatsapp/                 # WhatsApp integration
│   ├── slack/                    # Slack integration
│   ├── signal/                   # Signal integration
│   │
│   ├── browser/                  # Browser automation
│   ├── canvas-host/              # Canvas system
│   ├── media/                    # Media processing
│   ├── sessions/                 # Session management
│   ├── hooks/                    # Webhook system
│   ├── cron/                     # Cron job system
│   │
│   ├── terminal/                 # Terminal UI utilities
│   ├── infra/                    # Infrastructure utilities
│   ├── utils/                    # General utilities
│   │
│   ├── entry.ts                  # Bootstrap entry point
│   └── index.ts                  # Main export
│
├── extensions/                   # Extension plugins
│   ├── matrix/                   # Matrix channel
│   ├── msteams/                  # Microsoft Teams
│   ├── zalo/                     # Zalo channel
│   └── ...
│
├── apps/                         # Platform apps
│   ├── macos/                    # macOS menu bar app
│   ├── ios/                      # iOS app
│   └── android/                  # Android app
│
├── packages/                     # Shared packages
│
├── docs/                         # Documentation (Mintlify)
│   ├── channels/                 # Channel docs
│   ├── concepts/                 # Core concepts
│   ├── platforms/                # Platform-specific docs
│   └── ...
│
├── scripts/                      # Build & utility scripts
├── test/                         # Test utilities
├── skills/                       # Bundled skills
├── ui/                          # Web UI (Control UI)
│
├── openclaw.mjs                  # CLI entry wrapper
├── package.json                  # Package manifest
├── tsconfig.json                 # TypeScript config
├── vitest.config.ts              # Test config
└── tsdown.config.ts              # Build config
```

---

## Core Components

### 1. Entry Point & CLI

**File: `/src/entry.ts`**

The bootstrap entry point that:
- Respawns Node.js with experimental warning suppression
- Normalizes Windows argv for compatibility
- Parses CLI profile arguments (dev, prod, etc.)
- Dynamically imports the main CLI runner

```typescript
// Simplified flow
async function main() {
  const respawn = shouldRespawn();
  if (respawn) {
    // Respawn with clean environment
    return;
  }
  
  // Load profile (dev/prod)
  parseCliProfileArgs(process.argv);
  
  // Import and run CLI
  const { runMain } = await import('./cli/run-main.js');
  await runMain();
}
```

**File: `/src/cli/run-main.ts`**

Main CLI runner that:
- Loads `.env` files
- Validates Node.js version
- Initializes console capture
- Builds Commander.js program
- Installs error handlers

**File: `/src/cli/program/build-program.ts`**

Builds the CLI program:
```typescript
export function buildProgram() {
  const program = new Command('openclaw');
  
  // Register all commands
  registerProgramCommands(program, deps);
  
  return program;
}
```

**File: `/src/cli/program/command-registry.ts`**

Central command registry defining all CLI commands:
- `status`, `health`, `sessions`
- `agent`, `gateway`, `config`
- `message send`, `onboard`, `doctor`
- Channel-specific commands

### 2. Gateway Server

**File: `/src/gateway/server.impl.ts`**

The gateway is the heart of OpenClaw - a WebSocket/HTTP server that:
- Manages channel lifecycles
- Routes messages between channels and agents
- Handles session management
- Provides Control UI (web interface)
- Manages tool execution approvals

**Key Gateway Components:**

```typescript
// Gateway server options
type GatewayServerOptions = {
  bind?: "loopback" | "lan" | "tailnet" | "auto";
  port?: number;
  auth?: { username: string; password: string };
  controlUi?: boolean;  // Enable web UI
  tls?: { cert: string; key: string };
};

// Main server interface
type GatewayServer = {
  close: (opts?: { 
    reason?: string; 
    restartExpectedMs?: number | null 
  }) => Promise<void>;
};
```

**File: `/src/gateway/server-chat.ts`**

Chat event handling:
- Creates agent event handlers for incoming messages
- Manages chat run registry (tracks active conversations)
- Handles streaming responses back to channels

**File: `/src/gateway/server-channels.ts`**

Channel lifecycle management:
- Starts/stops channels based on config
- Manages account switching
- Tracks channel runtime state

```typescript
type ChannelManager = {
  start: (channelId: string) => Promise<void>;
  stop: (channelId: string) => Promise<void>;
  restart: (channelId: string) => Promise<void>;
  getStatus: (channelId: string) => ChannelStatus;
};
```

**File: `/src/gateway/server-ws-runtime.ts`**

WebSocket runtime for real-time communication:
- Handles WebSocket connections
- Message queuing and subscription
- Event broadcasting to connected clients

### 3. Channel Plugin System

**File: `/src/channels/plugins/types.plugin.ts`**

Defines the channel plugin interface:

```typescript
type ChannelPlugin<ResolvedAccount = any, Probe = unknown, Audit = unknown> = {
  id: ChannelId;                  // "discord", "telegram", etc.
  meta: ChannelMeta;              // Display name, icon
  capabilities: ChannelCapabilities;  // Feature flags
  
  // Adapters (optional)
  config?: ChannelConfigAdapter<ResolvedAccount>;
  configSchema?: ChannelConfigSchema;
  gateway?: ChannelGatewayAdapter<ResolvedAccount>;
  onboarding?: ChannelOnboardingAdapter;
  auth?: ChannelAuthAdapter;
  outbound?: ChannelOutboundAdapter;
  messaging?: ChannelMessagingAdapter;
  
  // ... 15+ other adapters
};
```

**Channel Capabilities:**
- `directMessages`: Supports 1-on-1 DMs
- `groupMessages`: Supports group chats
- `voiceMessages`: Supports voice notes
- `fileSharing`: Supports file uploads
- `richMedia`: Supports images/video
- And many more...

**Example Plugin (Discord):**

**File: `/extensions/discord/index.ts`**

```typescript
const discordPlugin: ChannelPlugin = {
  id: "discord",
  meta: {
    name: "Discord",
    icon: "discord.png",
  },
  capabilities: {
    directMessages: true,
    groupMessages: true,
    richMedia: true,
    fileSharing: true,
  },
  gateway: {
    async start(account, deps) {
      // Start Discord bot
      const client = new Client({ intents: [...] });
      await client.login(account.token);
      
      // Register message handlers
      client.on('messageCreate', async (msg) => {
        // Forward to gateway
        deps.onMessage({
          channelId: 'discord',
          sessionKey: `discord:${msg.channel.id}`,
          text: msg.content,
          sender: { id: msg.author.id },
        });
      });
      
      return { client };
    },
    async stop(runtime) {
      await runtime.client.destroy();
    },
  },
  outbound: {
    async sendMessage(runtime, params) {
      const channel = await runtime.client.channels.fetch(params.targetId);
      await channel.send(params.text);
    },
  },
};
```

**Channel Extensions:**

Built-in channels:
- WhatsApp (Baileys library)
- Telegram (grammY framework)
- Discord (discord.js)
- Slack (Bolt SDK)
- Signal (signal-cli)
- Google Chat (Chat API)

Extension channels (in `/extensions/`):
- Microsoft Teams
- Matrix
- Zalo
- BlueBubbles (iMessage)

### 4. Agent System (Pi Runtime)

**File: `/src/agents/pi-embedded-runner/run.ts`**

The Pi embedded agent runtime:

```typescript
type RunEmbeddedPiAgentParams = {
  agentId: string;
  sessionKey: string;
  message: string;
  thinking?: 'low' | 'medium' | 'high';  // Reasoning depth
  model?: string;
  tools?: ToolDefinition[];
  onStream?: (chunk: StreamChunk) => void;
};

async function runEmbeddedPiAgent(
  params: RunEmbeddedPiAgentParams
): Promise<EmbeddedPiRunResult> {
  // 1. Create or load agent session
  const session = await loadSession(params);
  
  // 2. Add user message to session
  session.addMessage({ role: 'user', content: params.message });
  
  // 3. Run agent (may invoke tools)
  const result = await session.run({
    model: params.model,
    tools: params.tools,
    onStream: params.onStream,
  });
  
  // 4. Save session state
  await saveSession(session);
  
  return result;
}
```

**File: `/src/agents/pi-tools.js`**

Tool definitions for the agent:

```typescript
const tools = [
  // Browser automation
  {
    name: 'browser_navigate',
    description: 'Navigate to a URL',
    parameters: { url: 'string' },
    execute: async ({ url }) => {
      // Launch Playwright browser
      const page = await browser.newPage();
      await page.goto(url);
      return { screenshot: await page.screenshot() };
    },
  },
  
  // Shell commands
  {
    name: 'bash',
    description: 'Run a bash command',
    parameters: { command: 'string' },
    execute: async ({ command }) => {
      const result = await exec(command);
      return { stdout: result.stdout };
    },
  },
  
  // Message sending
  {
    name: 'message_send',
    description: 'Send a message to a channel',
    parameters: { 
      channel: 'string',
      target: 'string',
      text: 'string',
    },
    execute: async ({ channel, target, text }) => {
      await sendMessage({ channel, target, text });
      return { success: true };
    },
  },
  
  // Canvas control
  {
    name: 'canvas_render',
    description: 'Render content to canvas',
    parameters: { html: 'string' },
    execute: async ({ html }) => {
      await renderToCanvas(html);
      return { success: true };
    },
  },
];
```

**File: `/src/agents/pi-embedded-runner/run/attempt.ts`**

Handles individual agent attempts:
- Streams responses from language models
- Manages caching for performance
- Handles tool invocation
- Truncates oversized results
- Implements retry logic on errors

### 5. Session Management

**File: `/src/routing/session-key.ts`**

Session key format and parsing:

```typescript
// Session key format: "<prefix>:<kind>:<id>"
type SessionKey = string;

// Examples:
// - "agent:main:main" (main agent session)
// - "discord:group:123" (Discord group chat)
// - "telegram:direct:user456" (Telegram DM)

function parseSessionKey(key: string) {
  const [prefix, kind, id] = key.split(':');
  return { prefix, kind, id };
}
```

**File: `/src/config/sessions/store.ts`**

Session storage implementation:

```typescript
type SessionStore = {
  load: (agentId: string, sessionKey: string) => Promise<Session>;
  save: (agentId: string, sessionKey: string, session: Session) => Promise<void>;
  list: (agentId: string) => Promise<SessionKey[]>;
  delete: (agentId: string, sessionKey: string) => Promise<void>;
};

// Sessions stored in YAML files:
// ~/.openclaw/agents/<agentId>/sessions/<sessionKey>.yaml
```

**File: `/src/routing/resolve-route.ts`**

Routes incoming messages to agent sessions:

```typescript
type RouteResolution = {
  agentId: string;      // Target agent
  sessionKey: string;   // Target session
  createIfMissing: boolean;
};

async function resolveRoute(
  channelSessionKey: string,
  config: OpenClawConfig
): Promise<RouteResolution> {
  // 1. Parse channel session key
  const { channel, kind, id } = parseSessionKey(channelSessionKey);
  
  // 2. Check routing rules
  const rules = config.routing?.[channel];
  
  // 3. Find matching rule
  const rule = rules?.find(r => matches(r, { kind, id }));
  
  // 4. Return target agent + session
  return {
    agentId: rule?.agent ?? 'main',
    sessionKey: rule?.session ?? channelSessionKey,
  };
}
```

### 6. Configuration System

**File: `/src/config/types.ts`**

Main configuration structure:

```typescript
type OpenClawConfig = {
  // Agent configurations
  agents?: Record<string, AgentConfig>;
  
  // Channel configurations
  channels?: Record<ChannelId, ChannelConfig>;
  
  // Gateway settings
  gateway?: GatewayConfig;
  
  // Model configurations
  models?: ModelsConfig;
  
  // Hooks (webhooks, transcription)
  hooks?: HooksConfig;
  
  // Plugin settings
  plugins?: PluginsConfig;
  
  // Security settings
  dmPolicy?: 'open' | 'pairing' | 'closed';
  allowFrom?: string[];
  
  // Routing rules
  routing?: Record<ChannelId, RoutingRule[]>;
};

type AgentConfig = {
  id: string;
  model?: string;
  thinking?: 'low' | 'medium' | 'high';
  tools?: string[];
  systemPrompt?: string;
  sessions?: SessionConfig;
};

type ChannelConfig = {
  enabled?: boolean;
  accounts?: AccountConfig[];
  dm?: {
    policy?: 'open' | 'pairing' | 'closed';
    allowFrom?: string[];
  };
  groups?: GroupConfig[];
};
```

**File: `/src/config/io.ts`**

Configuration loading and saving:

```typescript
async function loadConfig(path: string): Promise<OpenClawConfig> {
  // 1. Read JSON5 file
  const content = await fs.readFile(path, 'utf-8');
  
  // 2. Parse JSON5 (allows comments, trailing commas)
  const raw = JSON5.parse(content);
  
  // 3. Resolve includes
  const config = await resolveIncludes(raw, path);
  
  // 4. Substitute env vars
  const resolved = substituteEnvVars(config);
  
  // 5. Validate schema
  const validated = await validateConfig(resolved);
  
  // 6. Apply defaults
  const final = applyDefaults(validated);
  
  return final;
}
```

**Configuration File Locations:**

Default: `~/.openclaw/openclaw.json5`

Priority order:
1. `OPENCLAW_CONFIG` env var
2. `--config` CLI flag
3. `~/.openclaw/openclaw.json5`
4. `~/.openclaw/openclaw.json`
5. `./openclaw.json5`
6. `./openclaw.json`

**File: `/src/config/validation.ts`**

Schema validation using Zod:

```typescript
import { z } from 'zod';

const AgentConfigSchema = z.object({
  id: z.string(),
  model: z.string().optional(),
  thinking: z.enum(['low', 'medium', 'high']).optional(),
  tools: z.array(z.string()).optional(),
  systemPrompt: z.string().optional(),
});

const OpenClawConfigSchema = z.object({
  agents: z.record(AgentConfigSchema).optional(),
  channels: z.record(z.any()).optional(),
  gateway: z.any().optional(),
  models: z.any().optional(),
});

function validateConfig(config: unknown): OpenClawConfig {
  return OpenClawConfigSchema.parse(config);
}
```

---

## Data Flow

### Message Receive Flow

```
1. User sends message (via WhatsApp/Telegram/etc.)
   ↓
2. Channel Plugin receives message
   |  - WhatsApp: Baileys WebSocket
   |  - Telegram: grammY long polling
   |  - Discord: discord.js event handler
   ↓
3. Channel calls deps.onMessage({ sessionKey, text, sender })
   ↓
4. Gateway server-chat.ts receives message
   ↓
5. Session Router (resolve-route.ts) determines target
   |  - Checks routing rules
   |  - Resolves agent ID + session key
   ↓
6. createAgentEventHandler() in server-chat.ts
   |  - Loads session from storage
   |  - Adds message to session transcript
   ↓
7. Pi Agent Runtime (pi-embedded-runner/run.ts)
   |  - Runs agent with user message
   |  - Agent may invoke tools (browser, shell, etc.)
   |  - Agent generates response
   ↓
8. Response streaming
   |  - Chunks streamed via onStream callback
   |  - Chunks sent to WebSocket clients
   |  - Chunks buffered for final response
   ↓
9. Final response sent back
   |  - Gateway calls channel.outbound.sendMessage()
   |  - Channel plugin sends to user
   ↓
10. Session saved to disk
    - YAML file in ~/.openclaw/agents/<agentId>/sessions/
```

### Message Send Flow (CLI)

```
1. User runs: openclaw message send --to +1234567890 --message "Hello"
   ↓
2. CLI command (message-cli.ts) parses arguments
   ↓
3. Sends HTTP request to gateway
   |  POST /api/message/send
   |  Body: { channel, target, text }
   ↓
4. Gateway API handler (server-methods.ts)
   |  - Validates request
   |  - Looks up channel
   ↓
5. Channel outbound adapter
   |  - channel.outbound.sendMessage()
   ↓
6. Channel sends message
   |  - WhatsApp: Baileys socket.sendMessage()
   |  - Telegram: bot.sendMessage()
   |  - Discord: channel.send()
   ↓
7. Response returned
   |  - Success/error status
   |  - Message ID (if available)
```

### Agent Tool Invocation Flow

```
1. Agent decides to invoke a tool
   |  - During Pi agent run
   |  - Tool: "browser_navigate"
   |  - Args: { url: "https://example.com" }
   ↓
2. Tool executor (pi-tools.js)
   |  - Looks up tool definition
   |  - Validates arguments
   ↓
3. Tool execution
   |  - Browser: Launch Playwright
   |  - Shell: Execute via child_process
   |  - Message: Call gateway API
   ↓
4. Tool result
   |  - Capture output (stdout, screenshot, etc.)
   |  - Truncate if too large
   ↓
5. Result returned to agent
   |  - Added to session as tool result
   |  - Agent continues reasoning
   ↓
6. Agent generates final response
   |  - Based on tool results
   |  - Streamed back to user
```

---

## Key Technologies

### Core Runtime
- **Node.js ≥22**: Runtime environment
- **TypeScript**: Primary language (ESM)
- **pnpm**: Package manager
- **tsdown**: Build tool (outputs to `dist/`)

### CLI Framework
- **Commander.js**: CLI command framework
- **@clack/prompts**: Interactive prompts
- **osc-progress**: Progress bars/spinners

### Gateway
- **WebSocket (ws)**: Real-time communication
- **Express**: HTTP server (minimal, mostly WS)
- **Hono**: Fast web framework for Control UI

### Agent Runtime
- **@mariozechner/pi-agent-core**: Multi-turn agent framework
- **@mariozechner/pi-ai**: AI provider integration
- **@mariozechner/pi-coding-agent**: Coding tools

### Channel Integrations
- **grammY**: Telegram bot framework
- **discord.js**: Discord API wrapper
- **@slack/bolt**: Slack app framework
- **@whiskeysockets/baileys**: WhatsApp Web library
- **signal-utils**: Signal protocol
- **@line/bot-sdk**: LINE messaging

### Browser Automation
- **playwright-core**: Headless browser control
- **linkedom**: DOM manipulation (server-side)

### Media Processing
- **sharp**: Image processing
- **pdfjs-dist**: PDF parsing
- **file-type**: File type detection

### Data Validation
- **zod**: Schema validation
- **@sinclair/typebox**: JSON Schema generation
- **ajv**: JSON Schema validation

### Testing
- **vitest**: Test runner
- **@vitest/coverage-v8**: Code coverage

### Platform Apps
- **Swift**: macOS/iOS apps
- **Kotlin**: Android app (Gradle)
- **Lit**: Web components (Control UI)

---

## Development Workflow

### Setup Development Environment

```bash
# Clone repo
git clone https://github.com/openclaw/openclaw.git
cd openclaw

# Install dependencies
pnpm install

# Build UI
pnpm ui:build

# Build project
pnpm build

# Run onboarding
pnpm openclaw onboard
```

### Development Commands

```bash
# Type-check
pnpm tsgo

# Lint
pnpm lint

# Format
pnpm format

# Fix lint + format
pnpm lint:fix

# Run tests
pnpm test

# Test with coverage
pnpm test:coverage

# Run specific test
vitest run path/to/test.test.ts

# Watch mode (auto-rebuild)
pnpm gateway:watch
```

### Development Mode

```bash
# Run CLI in dev mode (TypeScript, auto-reload)
pnpm openclaw agent --message "Test"

# Run gateway in dev mode
pnpm gateway:dev

# Run with specific profile
OPENCLAW_PROFILE=dev pnpm openclaw agent
```

### Building Platform Apps

```bash
# macOS app
pnpm mac:package
pnpm mac:open

# iOS app (requires Xcode)
pnpm ios:build
pnpm ios:run

# Android app (requires Android SDK)
pnpm android:assemble
pnpm android:install
pnpm android:run
```

### Code Quality

```bash
# Pre-commit checks (runs automatically)
prek install

# Full check
pnpm check  # tsgo + lint + format

# Check documentation
pnpm check:docs  # format + lint + build

# Check lines of code limit
pnpm check:loc --max 500
```

---

## Extension Points

### 1. Creating a Channel Plugin

**File: `/extensions/my-channel/index.ts`**

```typescript
import type { ChannelPlugin, OpenClawPluginApi } from 'openclaw/plugin-sdk';

const myChannelPlugin: ChannelPlugin = {
  id: 'my-channel',
  meta: {
    name: 'My Channel',
    icon: 'my-channel.png',
  },
  capabilities: {
    directMessages: true,
    groupMessages: true,
  },
  
  // Gateway adapter - start/stop channel
  gateway: {
    async start(account, deps) {
      // Initialize your channel client
      const client = new MyChannelClient(account.apiKey);
      
      // Handle incoming messages
      client.on('message', (msg) => {
        deps.onMessage({
          channelId: 'my-channel',
          sessionKey: `my-channel:direct:${msg.senderId}`,
          text: msg.text,
          sender: { id: msg.senderId },
        });
      });
      
      await client.connect();
      return { client };
    },
    
    async stop(runtime) {
      await runtime.client.disconnect();
    },
  },
  
  // Outbound adapter - send messages
  outbound: {
    async sendMessage(runtime, params) {
      await runtime.client.sendMessage({
        to: params.targetId,
        text: params.text,
      });
    },
  },
  
  // Config schema
  configSchema: {
    account: {
      apiKey: { type: 'string', required: true },
    },
  },
};

// Register plugin
export default {
  id: 'my-channel',
  name: 'My Channel Plugin',
  configSchema: {},
  register(api: OpenClawPluginApi) {
    api.registerChannel({ plugin: myChannelPlugin });
  },
};
```

**Install plugin:**

```bash
# Install from npm
openclaw plugins install my-channel-plugin

# Or install from local directory
openclaw plugins install ./extensions/my-channel
```

### 2. Creating a Tool

**File: `/skills/my-tool.ts`**

```typescript
export const tool = {
  name: 'my_custom_tool',
  description: 'A custom tool that does something useful',
  
  parameters: {
    type: 'object',
    properties: {
      input: { type: 'string', description: 'Input parameter' },
    },
    required: ['input'],
  },
  
  async execute({ input }: { input: string }) {
    // Your tool logic here
    const result = await doSomething(input);
    
    return {
      success: true,
      result,
    };
  },
};
```

**Register in config:**

```json5
{
  agents: {
    main: {
      tools: [
        "browser",
        "bash",
        "my_custom_tool"  // Your custom tool
      ]
    }
  }
}
```

### 3. Creating a Skill

**File: `/skills/my-skill/index.ts`**

```typescript
export default {
  id: 'my-skill',
  name: 'My Custom Skill',
  description: 'A skill that does X',
  
  // Skill execution
  async run(params: { input: string }) {
    // Skill logic
    return { output: 'result' };
  },
  
  // Skill metadata
  meta: {
    category: 'productivity',
    tags: ['automation'],
  },
};
```

### 4. Creating a Hook

**File: Configuration**

```json5
{
  hooks: {
    transcription: {
      enabled: true,
      handler: 'webhook',
      url: 'https://my-server.com/transcribe',
      events: ['voice_message']
    },
    
    custom_hook: {
      enabled: true,
      handler: 'script',
      path: './hooks/my-hook.js',
      events: ['message_received']
    }
  }
}
```

**File: `/hooks/my-hook.js`**

```javascript
export async function handle(event) {
  // Process event
  console.log('Received event:', event);
  
  // Return modified event or null
  return event;
}
```

### 5. Extending Configuration

**File: `~/.openclaw/openclaw.json5`**

```json5
{
  // Custom agent with specific model and tools
  agents: {
    researcher: {
      model: 'claude-opus-4',
      thinking: 'high',
      tools: ['browser', 'bash'],
      systemPrompt: 'You are a research assistant...'
    }
  },
  
  // Custom routing rules
  routing: {
    discord: [
      {
        // Route #research channel to researcher agent
        match: { kind: 'group', id: '123456789' },
        agent: 'researcher',
        session: 'main'
      }
    ]
  },
  
  // Multi-account channel config
  channels: {
    telegram: {
      enabled: true,
      accounts: [
        { id: 'personal', apiId: '...', apiHash: '...' },
        { id: 'work', apiId: '...', apiHash: '...' }
      ]
    }
  },
  
  // Security settings
  dmPolicy: 'pairing',  // Require pairing for DMs
  allowFrom: ['user123', 'user456']  // Allowlist
}
```

---

## Additional Resources

### Documentation
- **Official Docs**: https://docs.openclaw.ai
- **Getting Started**: https://docs.openclaw.ai/start/getting-started
- **Configuration**: https://docs.openclaw.ai/gateway/configuration
- **Channels**: https://docs.openclaw.ai/channels
- **Tools**: https://docs.openclaw.ai/tools

### Community
- **Discord**: https://discord.gg/clawd
- **GitHub**: https://github.com/openclaw/openclaw
- **Website**: https://openclaw.ai

### Development
- **Contributing**: See `CONTRIBUTING.md`
- **Code Style**: Oxlint + Oxfmt (run `pnpm check`)
- **Testing**: Vitest (run `pnpm test`)
- **Releasing**: See `docs/reference/RELEASING.md`

---

## Summary

**OpenClaw** is a sophisticated personal AI assistant platform with a modular, extensible architecture:

1. **CLI Entry** → Bootstrap and command routing
2. **Gateway Server** → Central coordination hub (WebSocket/HTTP)
3. **Channel Plugins** → Multi-platform messaging integrations
4. **Session Router** → Routes messages to agent sessions
5. **Pi Agent Runtime** → Multi-turn AI conversations with tools
6. **Tool System** → Browser, shell, messaging, canvas, cron
7. **Configuration** → Centralized config with validation

The platform is designed to be:
- **Local-first**: Run on your own devices
- **Multi-channel**: Connect to many messaging platforms
- **Agentic**: Intelligent, multi-turn conversations
- **Extensible**: Plugin architecture for channels, tools, and skills
- **Secure**: Pairing-based access control by default

The codebase is well-organized with clear separation of concerns, making it easy to understand, extend, and maintain.
