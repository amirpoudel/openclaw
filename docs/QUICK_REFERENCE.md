# OpenClaw Codebase Analysis - Quick Reference

**Created:** 2026-02-09  
**Repository:** amirpoudel/openclaw (fork of openclaw/openclaw)

## What is OpenClaw?

OpenClaw is a **personal AI assistant platform** that runs on your own devices. It's a multi-channel AI gateway that connects to messaging platforms (WhatsApp, Telegram, Discord, Slack, Signal, etc.) and provides an intelligent AI assistant powered by Claude or other LLMs.

## Key Features

- 🤖 **Multi-channel support**: WhatsApp, Telegram, Discord, Slack, Signal, iMessage, Teams, Matrix, Zalo, WebChat
- 🧠 **Agentic AI**: Uses Pi embedded agent runtime for multi-turn conversations with tools
- 🔧 **Tool system**: Browser automation, shell commands, message sending, canvas, cron jobs
- 🔒 **Security-first**: Pairing-based access control by default
- 📱 **Platform apps**: macOS menu bar app, iOS app, Android app
- 🎯 **Multi-agent routing**: Route different channels to different AI agents
- 💾 **Session management**: Persistent conversation history

## Quick Start

```bash
# Install
npm install -g openclaw@latest

# Setup
openclaw onboard --install-daemon

# Start gateway
openclaw gateway run

# Talk to assistant
openclaw agent --message "Hello!"

# Send message
openclaw message send --channel whatsapp --to +1234567890 --message "Hi"
```

## Core Architecture Components

### 1. Entry & CLI (`src/entry.ts`, `src/cli/`)
- Bootstrap and command routing
- Commander.js-based CLI
- Commands: gateway, agent, config, sessions, message, onboard, doctor

### 2. Gateway Server (`src/gateway/`)
- Central coordination hub (WebSocket/HTTP on port 18789)
- Manages channel lifecycles
- Routes messages between channels and agents
- Provides Control UI (web interface)

### 3. Channel Plugins (`src/channels/`, `extensions/`)
- Plugin-based architecture for messaging platforms
- 15+ adapters per channel (gateway, outbound, auth, onboarding, etc.)
- Built-in: WhatsApp, Telegram, Discord, Slack, Signal, Google Chat
- Extensions: Teams, Matrix, Zalo, BlueBubbles

### 4. Session Router (`src/routing/`)
- Session key format: `<channel>:<kind>:<id>`
- Routes messages to agent sessions based on rules
- Examples: `whatsapp:direct:+1234567890`, `discord:group:channel-id`

### 5. Pi Agent Runtime (`src/agents/`)
- Multi-turn conversation engine (@mariozechner/pi-agent-core)
- Model provider integration (Anthropic, OpenAI)
- Tool invocation and streaming responses
- Sessions stored in YAML files

### 6. Tool System (`src/agents/pi-tools.js`)
- **Browser**: Playwright automation (navigate, click, screenshot)
- **Shell**: Bash command execution
- **Messaging**: Send messages to channels
- **Canvas**: A2UI visual workspace
- **Cron**: Scheduled jobs

### 7. Configuration (`src/config/`)
- Main file: `~/.openclaw/openclaw.json5`
- JSON5 format (comments, trailing commas)
- Zod schema validation
- Environment variable substitution

## Data Flow

```
User Message (WhatsApp/Telegram/etc.)
    ↓
Channel Plugin receives message
    ↓
Gateway Server (server-chat.ts)
    ↓
Session Router determines target agent
    ↓
Pi Agent Runtime processes with LLM
    ↓
Agent may invoke tools (browser, shell, etc.)
    ↓
Response streamed back through Gateway
    ↓
Channel Plugin delivers to user
    ↓
Session saved to ~/.openclaw/agents/<agentId>/sessions/
```

## Directory Structure

```
openclaw/
├── src/                    # TypeScript source
│   ├── cli/                # CLI commands
│   ├── gateway/            # Gateway server
│   ├── agents/             # Agent runtime
│   ├── channels/           # Channel plugin system
│   ├── routing/            # Session routing
│   ├── config/             # Configuration
│   ├── browser/            # Browser automation
│   ├── sessions/           # Session management
│   ├── discord/            # Discord integration
│   ├── telegram/           # Telegram integration
│   ├── whatsapp/           # WhatsApp integration
│   └── ...
├── extensions/             # Extension plugins
│   ├── matrix/
│   ├── msteams/
│   └── ...
├── apps/                   # Platform apps
│   ├── macos/              # macOS app (Swift)
│   ├── ios/                # iOS app (Swift)
│   └── android/            # Android app (Kotlin)
├── docs/                   # Documentation (Mintlify)
├── skills/                 # Bundled skills
└── ui/                     # Web UI (Control UI)
```

## Configuration Example

```json5
{
  agents: {
    main: {
      model: "claude-sonnet-4",
      thinking: "medium",
      tools: ["browser", "bash", "message_send"],
      systemPrompt: "You are a helpful assistant."
    },
    researcher: {
      model: "claude-opus-4",
      thinking: "high",
      tools: ["browser", "bash"],
      systemPrompt: "You are a research assistant."
    }
  },

  routing: {
    discord: [
      {
        match: { kind: "group", id: "research-channel" },
        agent: "researcher"
      }
    ]
  },

  channels: {
    whatsapp: { enabled: true },
    telegram: { enabled: true },
    discord: { enabled: true }
  },

  gateway: {
    port: 18789,
    bind: "loopback"
  },

  dmPolicy: "pairing",  // Require pairing for DMs
  allowFrom: ["+1234567890", "@username"]
}
```

## Key Technologies

- **Runtime**: Node.js ≥22, TypeScript (ESM)
- **Package Manager**: pnpm
- **CLI**: Commander.js, @clack/prompts
- **Gateway**: WebSocket (ws), Hono
- **Agent**: @mariozechner/pi-agent-core
- **Channels**: grammY (Telegram), discord.js, @slack/bolt, Baileys (WhatsApp)
- **Browser**: Playwright
- **Validation**: Zod, @sinclair/typebox
- **Testing**: Vitest

## Development Commands

```bash
# Type-check
pnpm tsgo

# Lint and format
pnpm check

# Fix issues
pnpm lint:fix

# Run tests
pnpm test

# Build
pnpm build

# Dev mode (auto-reload)
pnpm gateway:watch

# Run CLI in dev
pnpm openclaw agent --message "Test"
```

## Extension Points

### 1. Create Channel Plugin
Implement `ChannelPlugin` interface with adapters for gateway, outbound, auth, etc.

### 2. Create Tool
Define tool with name, description, parameters, and execute function.

### 3. Create Skill
Bundle tools and workflows into reusable skills.

### 4. Create Hook
Handle events (transcription, message_received, etc.) via webhooks or scripts.

### 5. Configure Routing
Route channels to specific agents based on rules.

## Security

- **Default DM policy**: `pairing` (unknown senders need approval)
- **Pairing flow**: User gets code → Admin approves with `openclaw pairing approve`
- **Allowlists**: Per-channel and global allowlists
- **Doctor command**: `openclaw doctor` checks for security misconfigurations

## Resources

- **Docs**: https://docs.openclaw.ai
- **GitHub**: https://github.com/openclaw/openclaw
- **Discord**: https://discord.gg/clawd
- **Website**: https://openclaw.ai

## Analysis Documents

This repository now includes three comprehensive analysis documents:

1. **CODEBASE_ANALYSIS.md** (31KB)
   - Complete technical overview
   - Core components explained
   - Architecture patterns
   - Extension points

2. **docs/ARCHITECTURE_DIAGRAMS.md** (30KB)
   - Visual architecture diagrams
   - Message flow diagrams
   - Session management architecture
   - Component interaction patterns

3. **docs/USAGE_EXAMPLES.md** (18KB)
   - Installation and setup
   - Configuration examples
   - Channel setup guides
   - Real-world scenarios
   - Troubleshooting

## Summary

OpenClaw is a sophisticated, modular platform for running a personal AI assistant:

- **Entry Point**: CLI bootstraps and loads commands
- **Gateway**: Central hub managing channels, sessions, and agents
- **Channels**: Plugin-based integrations with messaging platforms
- **Routing**: Routes messages to appropriate agent sessions
- **Agent**: Pi runtime with LLM integration and tool execution
- **Tools**: Browser, shell, messaging, canvas, cron
- **Config**: Centralized JSON5 configuration with validation

The architecture is clean, extensible, and well-organized, making it easy to understand, customize, and extend for various use cases.
