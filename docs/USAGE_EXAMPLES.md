# OpenClaw - Practical Usage Examples

This document provides practical, real-world examples of how to use OpenClaw.

## Table of Contents
1. [Installation & Setup](#installation--setup)
2. [Basic Usage](#basic-usage)
3. [Configuration Examples](#configuration-examples)
4. [Channel Setup Examples](#channel-setup-examples)
5. [Agent Examples](#agent-examples)
6. [Tool Usage Examples](#tool-usage-examples)
7. [Advanced Routing](#advanced-routing)
8. [Custom Plugin Examples](#custom-plugin-examples)

---

## Installation & Setup

### Quick Installation

```bash
# Install OpenClaw globally
npm install -g openclaw@latest

# Or with pnpm
pnpm add -g openclaw@latest

# Or with bun
bun add -g openclaw
```

### Run Onboarding Wizard

```bash
# Interactive setup wizard
openclaw onboard --install-daemon

# This wizard will:
# 1. Create ~/.openclaw directory
# 2. Generate initial config
# 3. Set up authentication
# 4. Configure channels
# 5. Install daemon (launchd/systemd)
```

### From Source (Development)

```bash
# Clone repository
git clone https://github.com/openclaw/openclaw.git
cd openclaw

# Install dependencies
pnpm install

# Build UI and project
pnpm ui:build
pnpm build

# Run in development mode
pnpm openclaw onboard
```

---

## Basic Usage

### Start the Gateway

```bash
# Start gateway server (default port 18789)
openclaw gateway run

# Start with custom port
openclaw gateway run --port 8080

# Bind to LAN (accessible from other devices)
openclaw gateway run --bind lan

# Enable Control UI (web interface)
openclaw gateway run --control-ui

# Verbose logging
openclaw gateway run --verbose
```

### Send a Message

```bash
# Send to WhatsApp
openclaw message send --channel whatsapp --to +1234567890 --message "Hello!"

# Send to Telegram
openclaw message send --channel telegram --to @username --message "Hi there"

# Send to Discord
openclaw message send --channel discord --to channel-id --message "Hello Discord"

# Send with media
openclaw message send --channel whatsapp --to +1234567890 --file ./image.jpg --message "Check this out"
```

### Talk to the Agent

```bash
# Simple question
openclaw agent --message "What's the weather?"

# With high thinking (more reasoning)
openclaw agent --message "Analyze this code" --thinking high

# Interactive mode
openclaw agent

# Use specific agent
openclaw agent --agent researcher --message "Research topic X"

# JSON mode (for programmatic use)
openclaw agent --message "Hello" --json
```

### Manage Sessions

```bash
# List all sessions
openclaw sessions list

# View session details
openclaw sessions view main

# Delete a session
openclaw sessions delete old-session

# Clear session history
openclaw sessions clear main
```

### Check Status

```bash
# System status
openclaw status

# Detailed status with probes
openclaw status --deep

# Check specific channel
openclaw channels status discord

# Health check
openclaw health
```

---

## Configuration Examples

### Minimal Configuration

**File: `~/.openclaw/openclaw.json5`**

```json5
{
  // Single agent with default settings
  agents: {
    main: {
      model: "claude-sonnet-4",
      thinking: "medium"
    }
  },

  // Enable WhatsApp channel
  channels: {
    whatsapp: {
      enabled: true
    }
  },

  // Gateway settings
  gateway: {
    port: 18789,
    bind: "loopback"
  }
}
```

### Multi-Agent Configuration

```json5
{
  agents: {
    // General-purpose assistant
    main: {
      model: "claude-sonnet-4",
      thinking: "medium",
      tools: ["browser", "bash", "message_send"],
      systemPrompt: "You are a helpful personal assistant."
    },

    // Research specialist
    researcher: {
      model: "claude-opus-4",
      thinking: "high",
      tools: ["browser", "bash"],
      systemPrompt: "You are a research assistant. Focus on accuracy and depth."
    },

    // Code reviewer
    coder: {
      model: "claude-opus-4",
      thinking: "high",
      tools: ["bash", "browser"],
      systemPrompt: "You are an expert code reviewer. Analyze code quality and suggest improvements."
    }
  },

  // Route different channels to different agents
  routing: {
    discord: [
      {
        match: { kind: "group", id: "research-channel" },
        agent: "researcher"
      },
      {
        match: { kind: "group", id: "dev-channel" },
        agent: "coder"
      }
    ]
  },

  channels: {
    whatsapp: { enabled: true },
    telegram: { enabled: true },
    discord: { enabled: true }
  }
}
```

### Security Configuration

```json5
{
  // Global DM policy
  dmPolicy: "pairing",  // Require pairing for unknown senders

  // Global allowlist
  allowFrom: [
    "+1234567890",  // WhatsApp number
    "@username",     // Telegram username
    "user-id-123"    // Generic user ID
  ],

  channels: {
    discord: {
      enabled: true,
      dm: {
        policy: "pairing",  // Override global policy for Discord
        allowFrom: ["user-id-456"]  // Discord-specific allowlist
      },
      groups: [
        {
          id: "server-id",
          allowFrom: ["*"],  // Allow all in this server
          mentionGating: true  // Require @mention
        }
      ]
    },

    telegram: {
      enabled: true,
      dm: {
        policy: "closed"  // Block all DMs
      }
    },

    whatsapp: {
      enabled: true,
      dm: {
        policy: "open",
        allowFrom: ["*"]  // Open to everyone (not recommended)
      }
    }
  }
}
```

### Multi-Account Configuration

```json5
{
  channels: {
    telegram: {
      enabled: true,
      accounts: [
        {
          id: "personal",
          apiId: "12345678",
          apiHash: "abcdef1234567890"
        },
        {
          id: "work",
          apiId: "87654321",
          apiHash: "0987654321fedcba"
        }
      ]
    },

    slack: {
      enabled: true,
      accounts: [
        {
          id: "company-workspace",
          token: "${SLACK_BOT_TOKEN_COMPANY}"
        },
        {
          id: "personal-workspace",
          token: "${SLACK_BOT_TOKEN_PERSONAL}"
        }
      ]
    }
  }
}
```

---

## Channel Setup Examples

### WhatsApp Setup

```bash
# Start onboarding
openclaw channels onboard whatsapp

# QR code will be displayed - scan with WhatsApp app
# After scanning, credentials are saved

# Send a test message
openclaw message send --channel whatsapp --to +1234567890 --message "Test"
```

**Config:**
```json5
{
  channels: {
    whatsapp: {
      enabled: true,
      dm: {
        policy: "pairing",
        allowFrom: ["+1234567890"]
      }
    }
  }
}
```

### Telegram Setup

```bash
# Get API credentials from https://my.telegram.org
# Create an app and note the API ID and Hash

# Configure
openclaw config set channels.telegram.accounts.0.apiId "12345678"
openclaw config set channels.telegram.accounts.0.apiHash "your-api-hash"

# Onboard
openclaw channels onboard telegram

# Enter phone number and verification code
```

**Config:**
```json5
{
  channels: {
    telegram: {
      enabled: true,
      accounts: [
        {
          id: "main",
          apiId: "12345678",
          apiHash: "your-hash"
        }
      ],
      dm: {
        policy: "pairing"
      }
    }
  }
}
```

### Discord Setup

```bash
# Create a Discord bot at https://discord.com/developers/applications
# Enable required intents: MESSAGE CONTENT, GUILD MESSAGES, DIRECT MESSAGES
# Copy bot token

# Configure
openclaw config set channels.discord.token "${DISCORD_BOT_TOKEN}"

# Or set environment variable
export DISCORD_BOT_TOKEN="your-bot-token"

# Start gateway
openclaw gateway run
```

**Config:**
```json5
{
  channels: {
    discord: {
      enabled: true,
      token: "${DISCORD_BOT_TOKEN}",
      dm: {
        policy: "pairing"
      },
      groups: [
        {
          id: "server-id",
          allowFrom: ["*"],
          mentionGating: true  // Require @bot mention
        }
      ]
    }
  }
}
```

### Slack Setup

```bash
# Create Slack app at https://api.slack.com/apps
# Enable Socket Mode
# Add scopes: chat:write, channels:history, im:history
# Install to workspace
# Copy Bot User OAuth Token

# Configure
openclaw config set channels.slack.token "${SLACK_BOT_TOKEN}"

# Start gateway
openclaw gateway run
```

**Config:**
```json5
{
  channels: {
    slack: {
      enabled: true,
      token: "${SLACK_BOT_TOKEN}",
      dm: {
        policy: "pairing"
      }
    }
  }
}
```

---

## Agent Examples

### Basic Agent Interaction

```bash
# Simple question
openclaw agent --message "What's 2+2?"
# Response: "2 + 2 = 4"

# Web search
openclaw agent --message "What's the latest news about AI?"

# Code help
openclaw agent --message "Write a Python function to reverse a string"
```

### Agent with Tools

```bash
# Browser automation
openclaw agent --message "Navigate to example.com and take a screenshot"

# Shell commands
openclaw agent --message "List files in the current directory"

# Message sending
openclaw agent --message "Send a message to +1234567890 saying 'Meeting at 3pm'"
```

### Different Thinking Levels

```bash
# Low thinking (faster, less reasoning)
openclaw agent --message "Quick math: 15 * 7" --thinking low

# Medium thinking (balanced)
openclaw agent --message "Summarize this article: ..." --thinking medium

# High thinking (slower, more reasoning)
openclaw agent --message "Analyze the pros and cons of this architecture" --thinking high
```

### Custom Agent

```bash
# Create custom agent config
cat > ~/.openclaw/openclaw.json5 << 'EOF'
{
  agents: {
    pythonExpert: {
      model: "claude-opus-4",
      thinking: "high",
      tools: ["bash", "browser"],
      systemPrompt: "You are a Python expert. Help with Python code, best practices, and debugging."
    }
  }
}
EOF

# Use custom agent
openclaw agent --agent pythonExpert --message "Review this Python code: ..."
```

---

## Tool Usage Examples

### Browser Tools

```bash
# Navigate to URL
openclaw agent --message "Go to https://github.com/openclaw/openclaw"

# Take screenshot
openclaw agent --message "Take a screenshot of example.com"

# Fill form
openclaw agent --message "Navigate to form.example.com and fill in the name field with 'John'"

# Extract data
openclaw agent --message "Go to news.ycombinator.com and list the top 5 stories"
```

### Shell Tools

```bash
# Run commands
openclaw agent --message "Run: ls -la"

# File operations
openclaw agent --message "Create a file called test.txt with content 'Hello World'"

# System info
openclaw agent --message "Show disk usage"

# Multiple commands
openclaw agent --message "Create a directory, cd into it, and create a file"
```

### Message Tools

```bash
# Send messages
openclaw agent --message "Send 'Hello' to +1234567890 via WhatsApp"

# Multi-channel
openclaw agent --message "Send 'Meeting reminder' to both Telegram @user and Discord user-id"

# Scheduled messages
openclaw agent --message "Schedule a message to +1234567890 for tomorrow at 9am"
```

### Canvas Tools

```bash
# Render HTML
openclaw agent --message "Show a bar chart of [10, 20, 30, 40] on the canvas"

# Interactive UI
openclaw agent --message "Create a calculator interface on the canvas"

# Data visualization
openclaw agent --message "Visualize this data: {sales: [100, 200, 150]}"
```

---

## Advanced Routing

### Route by Channel Type

```json5
{
  routing: {
    // Route Discord messages to coder agent
    discord: [
      {
        match: { kind: "group" },
        agent: "coder"
      }
    ],

    // Route WhatsApp to main agent
    whatsapp: [
      {
        match: { kind: "direct" },
        agent: "main"
      }
    ]
  }
}
```

### Route by Specific ID

```json5
{
  routing: {
    telegram: [
      {
        // Specific user to research agent
        match: { kind: "direct", id: "user123" },
        agent: "researcher",
        session: "research-project"
      },
      {
        // Specific group to coder agent
        match: { kind: "group", id: "group456" },
        agent: "coder",
        session: "main"
      }
    ]
  }
}
```

### Complex Routing

```json5
{
  routing: {
    discord: [
      {
        // Research channel
        match: { kind: "group", id: "research-channel-id" },
        agent: "researcher",
        session: "research"
      },
      {
        // Dev channel
        match: { kind: "group", id: "dev-channel-id" },
        agent: "coder",
        session: "coding"
      },
      {
        // Support channel
        match: { kind: "group", id: "support-channel-id" },
        agent: "support",
        session: "support"
      },
      {
        // Default for other Discord channels
        match: { kind: "group" },
        agent: "main",
        session: "main"
      }
    ]
  }
}
```

---

## Custom Plugin Examples

### Simple Channel Plugin

**File: `/extensions/my-channel/index.ts`**

```typescript
import type { ChannelPlugin, OpenClawPluginApi } from 'openclaw/plugin-sdk';

const myChannelPlugin: ChannelPlugin = {
  id: 'my-channel',
  meta: {
    name: 'My Custom Channel',
    icon: 'my-channel.png',
  },
  capabilities: {
    directMessages: true,
    groupMessages: false,
  },
  
  gateway: {
    async start(account, deps) {
      console.log('Starting my channel...');
      
      // Simulate incoming message
      setTimeout(() => {
        deps.onMessage({
          channelId: 'my-channel',
          sessionKey: 'my-channel:direct:user1',
          text: 'Hello from custom channel!',
          sender: { id: 'user1' },
        });
      }, 5000);
      
      return {};
    },
    
    async stop(runtime) {
      console.log('Stopping my channel...');
    },
  },
  
  outbound: {
    async sendMessage(runtime, params) {
      console.log('Sending message:', params.text);
      console.log('To:', params.targetId);
    },
  },
};

export default {
  id: 'my-channel-plugin',
  name: 'My Channel Plugin',
  configSchema: {},
  register(api: OpenClawPluginApi) {
    api.registerChannel({ plugin: myChannelPlugin });
  },
};
```

### Custom Tool Plugin

**File: `/skills/weather-tool/index.ts`**

```typescript
export const tool = {
  name: 'get_weather',
  description: 'Get current weather for a location',
  
  parameters: {
    type: 'object',
    properties: {
      location: {
        type: 'string',
        description: 'City name or zip code',
      },
    },
    required: ['location'],
  },
  
  async execute({ location }: { location: string }) {
    // Call weather API (example with OpenWeatherMap)
    const apiKey = process.env.OPENWEATHER_API_KEY;
    const url = `https://api.openweathermap.org/data/2.5/weather?q=${location}&appid=${apiKey}`;
    
    const response = await fetch(url);
    const data = await response.json();
    
    return {
      temperature: data.main.temp,
      condition: data.weather[0].description,
      humidity: data.main.humidity,
    };
  },
};
```

**Usage:**
```bash
# Add to config
openclaw config set agents.main.tools '["browser", "bash", "get_weather"]'

# Use tool
openclaw agent --message "What's the weather in San Francisco?"
```

---

## Real-World Scenarios

### Scenario 1: Personal Assistant

```json5
{
  agents: {
    main: {
      model: "claude-sonnet-4",
      thinking: "medium",
      tools: ["browser", "bash", "message_send", "canvas"],
      systemPrompt: `You are my personal assistant. Help me with:
        - Scheduling and reminders
        - Research and information lookup
        - Message drafting
        - Task automation`
    }
  },

  channels: {
    whatsapp: {
      enabled: true,
      dm: {
        policy: "pairing",
        allowFrom: ["${MY_PHONE}"]
      }
    },
    telegram: {
      enabled: true,
      dm: {
        policy: "pairing",
        allowFrom: ["@myusername"]
      }
    }
  }
}
```

### Scenario 2: Team Support Bot

```json5
{
  agents: {
    support: {
      model: "claude-sonnet-4",
      thinking: "medium",
      tools: ["browser", "message_send"],
      systemPrompt: `You are a customer support assistant. Help with:
        - Answering common questions
        - Troubleshooting issues
        - Escalating complex problems`
    }
  },

  channels: {
    slack: {
      enabled: true,
      dm: {
        policy: "open",
        allowFrom: ["*"]  // Allow all team members
      }
    },
    discord: {
      enabled: true,
      groups: [
        {
          id: "support-server",
          allowFrom: ["*"],
          mentionGating: true  // Require @mention
        }
      ]
    }
  }
}
```

### Scenario 3: Research Assistant

```json5
{
  agents: {
    researcher: {
      model: "claude-opus-4",
      thinking: "high",
      tools: ["browser", "bash"],
      systemPrompt: `You are a research assistant specialized in:
        - Academic paper analysis
        - Technical documentation review
        - Data gathering and synthesis
        - Citation management`
    }
  },

  routing: {
    discord: [
      {
        match: { kind: "group", id: "research-lab" },
        agent: "researcher",
        session: "lab-research"
      }
    ]
  },

  channels: {
    discord: {
      enabled: true,
      groups: [
        {
          id: "research-lab",
          allowFrom: ["*"],
          mentionGating: true
        }
      ]
    }
  }
}
```

---

## Troubleshooting Examples

### Check System Health

```bash
# Run doctor command
openclaw doctor

# This checks:
# - Configuration validity
# - Channel connectivity
# - Model access
# - Security settings
# - File permissions
```

### Debug Gateway Issues

```bash
# Start with verbose logging
openclaw gateway run --verbose

# Check gateway status
openclaw status --deep

# View gateway logs
tail -f ~/.openclaw/logs/gateway.log
```

### Fix Channel Connection

```bash
# Check channel status
openclaw channels status whatsapp

# Restart channel
openclaw channels restart whatsapp

# Re-authenticate
openclaw channels onboard whatsapp --force
```

### Reset Session

```bash
# Clear session history
openclaw sessions clear main

# Delete and recreate session
openclaw sessions delete main
openclaw agent --message "Hello"  # Creates new session
```

---

This comprehensive guide should help you understand how to use OpenClaw in various scenarios!
