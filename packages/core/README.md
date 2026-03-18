# @anagnole/claude-cli-wrapper

A shared library for spawning, parsing, and managing [Claude Code CLI](https://code.claude.com) sessions from TypeScript/Node.js. Exposes the full power of the CLI — MCP servers, permission modes, tool control, git worktrees, cost limits, and more.

## Install

```bash
npm install @anagnole/claude-cli-wrapper
```

Requires the Claude CLI (`claude`) to be installed and available in your PATH.

## Basic usage

```typescript
import { spawnClaude, NdjsonParser } from "@anagnole/claude-cli-wrapper";

// Spawn a Claude CLI process
const child = spawnClaude({
  prompt: "Explain this codebase",
  model: "claude-sonnet-4-6",
  streaming: true,
});

// Parse streaming NDJSON output
const parser = new NdjsonParser();
child.stdout.on("data", (chunk) => {
  for (const event of parser.feed(chunk.toString())) {
    console.log(event);
  }
});
```

## Full CLI options

```typescript
import { spawnClaude } from "@anagnole/claude-cli-wrapper";

const child = spawnClaude({
  prompt: "Fix the auth bug",
  model: "claude-sonnet-4-6",
  streaming: false,

  // System prompt
  systemPrompt: "You are a senior engineer.",
  appendSystemPrompt: true,

  // Session management
  resumeSessionId: "uuid-from-previous-run",
  // or: continueConversation: true,
  // or: forkSession: true,

  // Permissions & tools
  permissionMode: "bypassPermissions",
  allowedTools: ["Bash(git *)", "Read", "Edit"],
  disallowedTools: ["Write"],
  // cliTools: "" to disable all, "default" for all

  // MCP servers
  mcpConfig: "/path/to/mcp-config.json",
  strictMcpConfig: true,

  // Workspace
  workingDirectory: "/path/to/project",
  worktree: "feature-branch",
  addDirs: ["../shared-lib"],

  // Safety limits
  maxTurns: 10,
  maxBudgetUsd: 1.00,

  // Other
  effort: "high",
  fallbackModel: "claude-haiku-4-5",
  claudePath: "/custom/path/to/claude",
  jsonSchema: { type: "object", properties: { answer: { type: "string" } } },
});
```

## Session management

`SessionMap` tracks CLI sessions by hashing message history. On resume, it returns the session ID **and** the original spawn options — so system prompts, MCP configs, permissions, etc. are automatically restored.

```typescript
import { spawnClaude, SessionMap, type MessageParam } from "@anagnole/claude-cli-wrapper";

const sessions = new SessionMap();

// First request — no session to resume
const messages: MessageParam[] = [
  { role: "user", content: "Hello" },
];

const spawnOpts = {
  model: "claude-sonnet-4-6",
  systemPrompt: "You are a helpful assistant.",
  mcpConfig: "/path/to/mcp.json",
  permissionMode: "bypassPermissions",
};

const child = spawnClaude({ ...spawnOpts, prompt: "Hello", streaming: false });
// ... collect response, get cliSessionId ...

// Store session with its options
sessions.store(
  [...messages, { role: "assistant", content: "Hi!" }],
  "cli-session-uuid",
  "claude-sonnet-4-6",
  spawnOpts,  // stored for replay on resume
);

// Second request — session + options restored automatically
const nextMessages: MessageParam[] = [
  { role: "user", content: "Hello" },
  { role: "assistant", content: "Hi!" },
  { role: "user", content: "What did I say?" },
];

const hash = SessionMap.hashContext(nextMessages);
const saved = sessions.lookup(hash, "claude-sonnet-4-6");

if (saved) {
  // saved.options contains: systemPrompt, mcpConfig, permissionMode, etc.
  const child = spawnClaude({
    ...saved.options,
    resumeSessionId: saved.sessionId,
    prompt: "What did I say?",
    streaming: false,
  });
}
```

## Transform helpers

```typescript
import {
  extractPrompt,      // Get last user message as text
  extractSystem,      // Flatten system prompt blocks to string
  warnUnsupported,    // Log warnings for unsupported API params
  buildResponse,      // CLI JSON result → Anthropic API response shape
  generateMsgId,      // Generate msg_... IDs
  createStreamState,  // Initialize SSE streaming state
  transformEvent,     // CLI NDJSON event → Anthropic SSE strings
} from "@anagnole/claude-cli-wrapper";
```

## Subpath imports

For lighter imports that avoid pulling in all modules:

```typescript
import { spawnClaude } from "@anagnole/claude-cli-wrapper/cli";
import { NdjsonParser } from "@anagnole/claude-cli-wrapper/parser";
import { SessionMap } from "@anagnole/claude-cli-wrapper/session";
import type { MessagesRequest } from "@anagnole/claude-cli-wrapper/types";
```

## All exports

```typescript
// CLI
spawnClaude, SpawnOptions, NdjsonParser

// Session
SessionMap, SessionLookup

// Transform
extractPrompt, extractSystem, warnUnsupported
buildResponse, generateMsgId
createStreamState, transformEvent, StreamState

// Types
ContentBlock, SystemBlock, MessageParam,
MessagesRequest, MessagesResponse, Usage, ApiError, apiError
```

## Companion: API server

This package also comes with a companion HTTP server (`@anagnole/claude-api-server`) that wraps the core library as an Anthropic Messages API-compatible endpoint. See the [monorepo README](https://github.com/anagnole/claude-api) for details.
