# Development Guide

> **Comprehensive guide for developers contributing to better-chatbot**

This document explains the internal architecture, code organization, and how to implement new features like agent-to-agent communication protocols (e.g., Google's A2A protocol).

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [Core Systems](#core-systems)
  - [Chat Request Flow](#chat-request-flow)
  - [MCP (Model Context Protocol) Integration](#mcp-model-context-protocol-integration)
  - [Visual Workflows](#visual-workflows)
  - [Custom Agents](#custom-agents)
  - [Mentions System](#mentions-system)
- [Data Layer](#data-layer)
- [Authentication & Authorization](#authentication--authorization)
- [File Storage System](#file-storage-system)
- [Frontend Architecture](#frontend-architecture)
- [Testing Infrastructure](#testing-infrastructure)
- [Adding New Features](#adding-new-features)
  - [Example: Implementing Agent-to-Agent (A2A) Protocol](#example-implementing-agent-to-agent-a2a-protocol)
  - [Example: Adding a New Tool](#example-adding-a-new-tool)
  - [Example: Creating a New API Endpoint](#example-creating-a-new-api-endpoint)
- [Common Development Patterns](#common-development-patterns)
- [Debugging Tips](#debugging-tips)
- [Performance Considerations](#performance-considerations)

---

## Architecture Overview

Better-chatbot is a **full-stack Next.js 15 application** with these architectural layers:

```
┌─────────────────────────────────────────────────────────────┐
│                    Next.js App Router                        │
│  src/app/(chat)  │  src/app/(auth)  │  src/app/(public)     │
└────────────────┬────────────────────────────────────────────┘
                 │
┌────────────────▼────────────────────────────────────────────┐
│                   React Components                           │
│  UI (shadcn) │ Feature Components │ Layouts                 │
└────────────────┬────────────────────────────────────────────┘
                 │
┌────────────────▼────────────────────────────────────────────┐
│                   State Management                           │
│  SWR (Server State) │ Zustand (Client State)                │
└────────────────┬────────────────────────────────────────────┘
                 │
┌────────────────▼────────────────────────────────────────────┐
│                    API Routes                                │
│  /api/chat │ /api/mcp │ /api/workflow │ /api/agent          │
└────────────────┬────────────────────────────────────────────┘
                 │
┌────────────────▼────────────────────────────────────────────┐
│                  Business Logic Layer                        │
│  src/lib/ai │ src/lib/db │ src/lib/auth                     │
└────────────────┬────────────────────────────────────────────┘
                 │
┌────────────────▼────────────────────────────────────────────┐
│              External Services & Storage                     │
│  PostgreSQL │ Vercel Blob/S3 │ LLM APIs │ MCP Servers       │
└─────────────────────────────────────────────────────────────┘
```

---

## Core Systems

### Chat Request Flow

**Entry Point**: `src/app/api/chat/route.ts` - The main chat endpoint that orchestrates all AI interactions.

#### Request Processing Pipeline:

```typescript
1. Authentication Check (getSession)
   ↓
2. Parse Request Body (chatApiSchemaRequestBodySchema)
   ↓
3. Load/Create Chat Thread (chatRepository.selectThreadDetails)
   ↓
4. Load Agent (if mentioned via @agent)
   ↓
5. Load Tools (3 categories - conditional):
   - MCP_TOOLS: From MCP servers (loadMcpTools)
   - WORKFLOW_TOOLS: Published workflows (loadWorkFlowTools)
   - APP_DEFAULT_TOOLS: Built-in tools (loadAppDefaultTools)
   ↓
6. Build System Prompt:
   - User preferences (buildUserSystemPrompt)
   - Agent instructions
   - MCP customizations (buildMcpServerCustomizationsSystemPrompt)
   ↓
7. Stream AI Response (streamText from Vercel AI SDK)
   ↓
8. Save to Database (chatRepository.upsertMessage)
```

#### Key Files:

- **Main Handler**: `src/app/api/chat/route.ts`
- **Tool Loading**: `src/app/api/chat/shared.chat.ts`
- **System Prompts**: `src/lib/ai/prompts.ts`
- **Model Provider**: `src/lib/ai/models.ts`

#### Tool Loading Logic:

```typescript
// Tools are loaded conditionally based on:
// 1. Tool choice mode (auto/manual/none)
// 2. Mentions in the message
// 3. User's allowed servers/tools

const isToolCallAllowed = 
  supportToolCall && 
  (toolChoice != "none" || mentions.length > 0) &&
  !useImageTool;

if (isToolCallAllowed) {
  // Load only the tools needed for this request
  const MCP_TOOLS = await loadMcpTools({
    mentions,
    allowedMcpServers,
  });
  
  const WORKFLOW_TOOLS = await loadWorkFlowTools({
    mentions,
    dataStream,
  });
  
  const APP_DEFAULT_TOOLS = await loadAppDefaultTools({
    mentions,
    allowedAppDefaultToolkit,
  });
}
```

**Why this matters**: This conditional loading prevents sending hundreds of tools to the LLM, which would:
- Waste tokens
- Slow down response time
- Confuse the model with irrelevant options

---

### MCP (Model Context Protocol) Integration

MCP is a protocol for connecting AI models to external tools and data sources. Better-chatbot has deep MCP integration.

#### Architecture:

**Global Manager**: `src/lib/ai/mcp/mcp-manager.ts`
```typescript
// Singleton instance managing all MCP clients
export const mcpClientsManager = globalThis.__mcpClientsManager__;

// Usage in code:
const mcpClients = await mcpClientsManager.getClients();
const mcpTools = await mcpClientsManager.tools();
```

#### Storage Modes:

1. **Database Storage** (default): `src/lib/ai/mcp/db-mcp-config-storage.ts`
   - MCP configs stored in `McpServerTable`
   - Multi-user support
   - Persistent across restarts

2. **File Storage**: `src/lib/ai/mcp/fb-mcp-config-storage.ts`
   - Enabled with `FILE_BASED_MCP_CONFIG=true`
   - Stores configs in JSON files
   - Useful for single-user deployments

#### MCP Client Lifecycle:

```typescript
// 1. Initialization (on app startup)
await mcpClientsManager.init();

// 2. Adding a server
await mcpClientsManager.addClient(id, name, config);

// 3. Getting tools
const tools = await mcpClientsManager.tools();
// Returns: { "servername__toolname": VercelAIMcpTool }

// 4. Calling a tool
const result = await mcpClientsManager.toolCall(
  serverId, 
  toolName, 
  params
);

// 5. Cleanup (on shutdown)
await mcpClientsManager.cleanup();
```

#### Tool ID Convention:

MCP tools are identified as: `{serverName}__{toolName}`

```typescript
import { createMCPToolId } from "src/lib/ai/mcp/mcp-tool-id";

const toolId = createMCPToolId("playwright", "navigate");
// Returns: "playwright__navigate"
```

#### Custom Tool Prompts:

Users can customize how MCP tools behave via `McpToolCustomizationTable`:

```typescript
// Example: Custom prompt for a GitHub tool
{
  serverId: "github-server-id",
  toolName: "create_issue",
  customPrompt: "Always add a 🐛 emoji for bug reports"
}
```

These customizations are injected into the system prompt via `buildMcpServerCustomizationsSystemPrompt()`.

---

### Visual Workflows

Workflows are **visual node graphs** that execute multi-step AI operations and can be published as callable tools.

#### Node Types:

Located in `src/lib/ai/workflow/workflow.interface.ts`:

1. **InputNode**: Entry point, receives workflow parameters
2. **OutputNode**: Exit point, defines workflow return value
3. **LLMNode**: Calls an LLM with messages (supports @mentions to reference other nodes)
4. **ToolNode**: Executes an MCP tool
5. **HttpNode**: Makes HTTP requests
6. **ConditionNode**: Branching logic
7. **TemplateNode**: Text template with variable substitution

#### Execution Flow:

**Executor**: `src/lib/ai/workflow/executor/workflow-executor.ts`

```typescript
// Workflow execution graph
InputNode 
  ↓
LLMNode (generates SQL query)
  ↓
ToolNode (executes query via MCP)
  ↓
LLMNode (formats results)
  ↓
OutputNode (returns formatted data)
```

Each node executor implements:
```typescript
export type NodeExecutor<T extends WorkflowNodeData = any> = (input: {
  node: T;
  state: WorkflowRuntimeState;
}) => Promise<{
  input?: any;   // Input data used
  output?: any;  // Output data produced
}>;
```

#### Publishing as Tools:

**Tool Conversion**: `src/lib/ai/workflow/tool-executor.ts`

When a workflow is published (`visibility: "public"`), it becomes a callable tool:

```typescript
// In chat, users can invoke with @workflow_name
// The workflow executor runs the node graph and returns results
```

#### Storage:

- **Metadata**: `WorkflowTable` (name, description, visibility)
- **Nodes**: `WorkflowNodeDataTable` (node type, configuration)
- **Edges**: `WorkflowEdgeTable` (connections between nodes)

**Repository**: `src/lib/db/pg/repositories/workflow-repository.pg.ts`

---

### Custom Agents

Agents are **pre-configured AI assistants** with custom instructions and tool access.

#### Agent Structure:

```typescript
type Agent = {
  id: string;
  name: string;
  description: string;
  icon: { type: "emoji", value: string };
  instructions: {
    systemPrompt: string;
    mentions: ChatMention[];  // Pre-selected tools
  };
  visibility: "public" | "private";
  userId: string;
};
```

#### Invocation Flow:

```typescript
// 1. User types: @github_manager
// 2. Agent is loaded: rememberAgentAction(agentId)
// 3. Agent's mentions are merged with user mentions
if (agent?.instructions?.mentions) {
  mentions.push(...agent.instructions.mentions);
}
// 4. System prompt includes agent instructions
const systemPrompt = buildUserSystemPrompt(user, preferences, agent);
```

#### Example Use Case: GitHub Manager Agent

```typescript
{
  name: "GitHub Manager",
  instructions: {
    systemPrompt: `You are a GitHub project manager. You have access to:
      - Issue creation and updates
      - PR management
      - Repository queries
      
      Project context: better-chatbot repository
      Always use conventional commit format.`,
    mentions: [
      { type: "mcpServer", serverId: "github-id", name: "github" },
    ]
  }
}
```

**Storage**: `AgentTable` via `agentRepository`

---

### Mentions System

The `@mention` system allows users to **temporarily bind tools** for a specific request.

#### Mention Types:

Defined in `src/types/chat.ts`:

```typescript
type ChatMention = 
  | { type: "mcpTool", name: string, serverId: string }
  | { type: "mcpServer", serverId: string }
  | { type: "defaultTool", name: string }
  | { type: "workflow", workflowId: string }
  | { type: "agent", agentId: string };
```

#### How It Works:

**Input Component**: `src/components/chat-mention-input.tsx`

```
User types: "Search for @web-search React tutorials"
            ↓
Mention detected: { type: "defaultTool", name: "webSearch" }
            ↓
Only webSearch tool is loaded for this request
            ↓
Response generated with web search results
```

#### Benefits Over Tool Selection:

| Tool Selection | Mentions |
|----------------|----------|
| Always available | Available only when mentioned |
| Higher token usage | Lower token usage |
| Slower response | Faster response |
| Good for frequent tools | Good for occasional tools |

**Implementation**: `src/app/api/chat/shared.chat.ts` - `filterMCPToolsByMentions()`

---

## Data Layer

### Repository Pattern

All database access goes through **repository objects** exported from `src/lib/db/repository.ts`:

```typescript
export const chatRepository = pgChatRepository;
export const userRepository = pgUserRepository;
export const mcpRepository = pgMcpRepository;
export const workflowRepository = pgWorkflowRepository;
export const agentRepository = pgAgentRepository;
export const archiveRepository = pgArchiveRepository;
export const bookmarkRepository = pgBookmarkRepository;
```

### Database Schema

**Schema File**: `src/lib/db/pg/schema.pg.ts`

Key tables:
- `ChatThreadTable`: Chat sessions
- `ChatMessageTable`: Individual messages with parts (text, tool calls, tool results)
- `AgentTable`: Custom agents
- `WorkflowTable`: Visual workflows
- `McpServerTable`: MCP server configurations
- `UserTable`: User accounts (Better Auth integration)

### Migrations

**Production**: Use `pnpm db:migrate` (runs `scripts/db-migrate.ts`)
```bash
pnpm db:migrate
```

**Development**: Use `pnpm db:push` (applies schema changes directly)
```bash
pnpm db:push
```

**⚠️ Warning**: Never use `db:push` in production - it can lose data!

### Example Repository Usage:

```typescript
import { chatRepository } from "lib/db/repository";

// Create a thread
const thread = await chatRepository.insertThread({
  id: generateUUID(),
  title: "New Chat",
  userId: session.user.id,
});

// Save a message
await chatRepository.upsertMessage({
  threadId: thread.id,
  role: "user",
  parts: [{ type: "text", text: "Hello" }],
  id: generateUUID(),
});

// Query thread with messages
const threadDetails = await chatRepository.selectThreadDetails(thread.id);
```

---

## Authentication & Authorization

### Better Auth Integration

**Server-side**: `auth/server` (path alias)
```typescript
import { getSession } from "auth/server";

const session = await getSession();
if (!session?.user.id) {
  return new Response("Unauthorized", { status: 401 });
}
```

**Client-side**: `auth/client`
```typescript
import { useSession } from "auth/client";

const { data: session } = useSession();
```

### Roles

Defined in `src/lib/auth/roles.ts`:
- `admin`: Full system access
- `editor`: Can create/edit content
- `user`: Basic access

### Permissions

**Permission Checks**: `src/lib/auth/permissions.ts`

```typescript
import { canEditWorkflow, canManageUsers } from "lib/auth/permissions";

// Check if user can edit workflow
if (!canEditWorkflow(session.user, workflow)) {
  return new Response("Forbidden", { status: 403 });
}
```

Common permission functions:
- `canEditWorkflow(user, workflow)`
- `canEditAgent(user, agent)`
- `canManageUsers(user)`
- `canAccessAdminPanel(user)`

### Middleware

`src/middleware.ts` protects routes:

```typescript
// Unauthenticated users redirected to /sign-in
// Exceptions: /api/auth, /export, static files
```

---

## File Storage System

**Abstraction Layer**: `src/lib/file-storage/index.ts`

### Storage Drivers:

1. **Vercel Blob** (default): `src/lib/file-storage/vercel-blob-storage.ts`
2. **S3**: `src/lib/file-storage/s3-file-storage.ts`

### Usage:

```typescript
import { serverFileStorage } from "lib/file-storage";

// Upload
const { url, key } = await serverFileStorage.upload({
  file: fileBuffer,
  filename: "image.png",
  contentType: "image/png",
});

// Download
const buffer = await serverFileStorage.download(key);

// Delete
await serverFileStorage.delete(key);
```

**⚠️ Important**: Never hardcode `@vercel/blob` imports - always use `serverFileStorage` for portability!

### Configuration:

```env
FILE_STORAGE_TYPE=vercel-blob  # or "s3"
FILE_STORAGE_PREFIX=uploads
BLOB_READ_WRITE_TOKEN=...      # For Vercel Blob
# Or for S3:
FILE_STORAGE_S3_BUCKET=...
FILE_STORAGE_S3_REGION=...
```

---

## Frontend Architecture

### State Management

#### Server State (SWR)

**Location**: `src/hooks/queries/*`

```typescript
// Example: useChatModels hook
import useSWR from "swr";

export function useChatModels() {
  return useSWR<Provider[]>("/api/chat/models", fetcher);
}

// In component:
const { data: models, error, isLoading } = useChatModels();
```

Common SWR hooks:
- `useChatModels()` - Available LLM providers
- `useAgents()` - User's agents
- `useArchives()` - Archived chats

#### Client State (Zustand)

**Location**: `src/app/store/*`

```typescript
// Example: Workflow Store
import { create } from "zustand";

export const useWorkflowStore = create<WorkflowState & WorkflowDispatch>(
  (set) => ({
    workflow: undefined,
    hasEditAccess: false,
    init: (workflow, hasEditAccess) => set({ workflow, hasEditAccess }),
  })
);

// In component:
const { workflow, init } = useWorkflowStore();
```

### Component Organization

```
src/components/
├── ui/                    # shadcn primitives (Button, Dialog, etc.)
├── layouts/               # Page layouts (AppSidebar, etc.)
├── chat-bot.tsx          # Main chat component
├── mcp-dashboard.tsx     # MCP server management
├── workflow/             # Workflow editor
│   ├── workflow.tsx
│   └── nodes/
└── agent/                # Agent management
    ├── agent-list.tsx
    └── agent-editor.tsx
```

### Path Aliases

```typescript
// tsconfig.json paths
import { Button } from "ui/button";           // ui/*
import { getSession } from "auth/server";     // auth/*
import { ChatMessage } from "app-types/chat"; // app-types/*
import { generateUUID } from "lib/utils";     // lib/*
import logger from "logger";                  // logger
```

---

## Testing Infrastructure

### Unit Tests (Vitest)

**Location**: `src/lib/**/*.test.ts`

```bash
pnpm test        # Run once
pnpm test:watch  # Watch mode
```

**Example**:
```typescript
// src/lib/utils.test.ts
import { describe, it, expect } from "vitest";
import { generateUUID } from "./utils";

describe("generateUUID", () => {
  it("should generate unique IDs", () => {
    const id1 = generateUUID();
    const id2 = generateUUID();
    expect(id1).not.toBe(id2);
  });
});
```

### E2E Tests (Playwright)

**Location**: `tests/**/*.spec.ts`

```bash
pnpm test:e2e              # Run all tests
pnpm test:e2e:ui           # Open UI
pnpm test:e2e:first-user   # Test admin role assignment
```

#### Test Setup Orchestration:

See `tests/PLAYWRIGHT-TEST-STRATEGY.md` for details.

```
1. first-user-setup → Clear database
2. first-user → Test first user gets admin role
3. setup → Seed test users (admin, editor, user)
4. chromium → Run standard tests
```

**Test Users**: Created by `scripts/seed-test-users.ts`

**Auth States**: Saved browser sessions in `tests/.auth/`

#### Example Test:

```typescript
// tests/agents/agent-visibility.spec.ts
import { test, expect } from "@playwright/test";

test("agents are visible to creator", async ({ page }) => {
  await page.goto("/agents");
  await page.getByRole("button", { name: "Create Agent" }).click();
  await page.fill('input[name="name"]', "Test Agent");
  await page.click('button[type="submit"]');
  
  await expect(page.getByText("Test Agent")).toBeVisible();
});
```

---

## Adding New Features

### Example: Implementing Agent-to-Agent (A2A) Protocol

Google's A2A protocol allows multiple AI agents to communicate. Here's how to add it to better-chatbot:

#### Step 1: Define A2A Types

Create `src/types/a2a.ts`:

```typescript
import { z } from "zod";

export const A2AMessageSchema = z.object({
  from: z.string(), // Agent ID
  to: z.string(),   // Agent ID
  type: z.enum(["request", "response", "broadcast"]),
  payload: z.any(),
  timestamp: z.date(),
});

export type A2AMessage = z.infer<typeof A2AMessageSchema>;

export type A2AChannel = {
  id: string;
  participants: string[]; // Agent IDs
  messages: A2AMessage[];
};
```

#### Step 2: Create Database Schema

Add to `src/lib/db/pg/schema.pg.ts`:

```typescript
export const A2AChannelTable = pgTable("a2a_channel", {
  id: text("id").primaryKey(),
  name: text("name").notNull(),
  participants: jsonb("participants").$type<string[]>().notNull(),
  createdAt: timestamp("created_at").defaultNow().notNull(),
  userId: text("user_id").notNull(),
});

export const A2AMessageTable = pgTable("a2a_message", {
  id: text("id").primaryKey(),
  channelId: text("channel_id").notNull(),
  from: text("from").notNull(), // Agent ID
  to: text("to"),               // Null for broadcast
  type: text("type").notNull(),
  payload: jsonb("payload").notNull(),
  timestamp: timestamp("timestamp").defaultNow().notNull(),
});
```

#### Step 3: Create Repository

Create `src/lib/db/pg/repositories/a2a-repository.pg.ts`:

```typescript
import { A2AChannelTable, A2AMessageTable } from "../schema.pg";
import { db } from "../db.pg";
import { eq } from "drizzle-orm";

export const pgA2ARepository = {
  async createChannel(channel: typeof A2AChannelTable.$inferInsert) {
    return db.insert(A2AChannelTable).values(channel).returning();
  },
  
  async sendMessage(message: typeof A2AMessageTable.$inferInsert) {
    return db.insert(A2AMessageTable).values(message).returning();
  },
  
  async getChannelMessages(channelId: string) {
    return db
      .select()
      .from(A2AMessageTable)
      .where(eq(A2AMessageTable.channelId, channelId))
      .orderBy(A2AMessageTable.timestamp);
  },
};
```

Export in `src/lib/db/repository.ts`:
```typescript
export const a2aRepository = pgA2ARepository;
```

#### Step 4: Create A2A Manager

Create `src/lib/ai/a2a/a2a-manager.ts`:

```typescript
import { agentRepository, a2aRepository } from "lib/db/repository";
import { customModelProvider } from "lib/ai/models";
import { streamText } from "ai";

export class A2AManager {
  async sendAgentMessage({
    fromAgentId,
    toAgentId,
    channelId,
    message,
  }: {
    fromAgentId: string;
    toAgentId: string;
    channelId: string;
    message: string;
  }) {
    // 1. Load source agent
    const fromAgent = await agentRepository.selectById(fromAgentId);
    
    // 2. Save message to channel
    await a2aRepository.sendMessage({
      id: generateUUID(),
      channelId,
      from: fromAgentId,
      to: toAgentId,
      type: "request",
      payload: { text: message },
      timestamp: new Date(),
    });
    
    // 3. Load target agent
    const toAgent = await agentRepository.selectById(toAgentId);
    
    // 4. Get channel history
    const history = await a2aRepository.getChannelMessages(channelId);
    
    // 5. Generate response from target agent
    const model = customModelProvider.getModel(toAgent.defaultModel);
    const result = await streamText({
      model,
      system: toAgent.instructions.systemPrompt + 
              `\n\nYou are in an agent-to-agent conversation with ${fromAgent.name}.`,
      messages: history.map(msg => ({
        role: msg.from === toAgentId ? "assistant" : "user",
        content: msg.payload.text,
      })),
    });
    
    // 6. Save response
    const responseText = await result.text();
    await a2aRepository.sendMessage({
      id: generateUUID(),
      channelId,
      from: toAgentId,
      to: fromAgentId,
      type: "response",
      payload: { text: responseText },
      timestamp: new Date(),
    });
    
    return responseText;
  }
  
  async broadcastToAgents({
    fromAgentId,
    channelId,
    message,
  }: {
    fromAgentId: string;
    channelId: string;
    message: string;
  }) {
    // Implementation for broadcasting to multiple agents
    // Each agent processes the message in parallel
  }
}

export const a2aManager = new A2AManager();
```

#### Step 5: Create API Endpoint

Create `src/app/api/a2a/channel/[id]/message/route.ts`:

```typescript
import { NextRequest } from "next/server";
import { getSession } from "auth/server";
import { a2aManager } from "lib/ai/a2a/a2a-manager";
import { z } from "zod";

const sendMessageSchema = z.object({
  fromAgentId: z.string(),
  toAgentId: z.string(),
  message: z.string(),
});

export async function POST(
  req: NextRequest,
  { params }: { params: { id: string } }
) {
  const session = await getSession();
  if (!session?.user.id) {
    return new Response("Unauthorized", { status: 401 });
  }
  
  const body = await req.json();
  const { fromAgentId, toAgentId, message } = sendMessageSchema.parse(body);
  
  const response = await a2aManager.sendAgentMessage({
    fromAgentId,
    toAgentId,
    channelId: params.id,
    message,
  });
  
  return Response.json({ response });
}
```

#### Step 6: Create UI Component

Create `src/components/a2a/a2a-channel.tsx`:

```typescript
"use client";

import { useState } from "react";
import useSWR from "swr";
import { Button } from "ui/button";

export function A2AChannel({ channelId }: { channelId: string }) {
  const { data: messages } = useSWR(
    `/api/a2a/channel/${channelId}/messages`,
    fetcher
  );
  const [fromAgent, setFromAgent] = useState("");
  const [toAgent, setToAgent] = useState("");
  const [message, setMessage] = useState("");
  
  const sendMessage = async () => {
    await fetch(`/api/a2a/channel/${channelId}/message`, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ fromAgentId: fromAgent, toAgentId: toAgent, message }),
    });
    setMessage("");
  };
  
  return (
    <div className="space-y-4">
      <div className="messages">
        {messages?.map(msg => (
          <div key={msg.id} className="message">
            <strong>{msg.from}</strong> → {msg.to}: {msg.payload.text}
          </div>
        ))}
      </div>
      
      <div className="input-area">
        <select value={fromAgent} onChange={e => setFromAgent(e.target.value)}>
          {/* Agent options */}
        </select>
        <select value={toAgent} onChange={e => setToAgent(e.target.value)}>
          {/* Agent options */}
        </select>
        <input value={message} onChange={e => setMessage(e.target.value)} />
        <Button onClick={sendMessage}>Send</Button>
      </div>
    </div>
  );
}
```

#### Step 7: Add Route

Create `src/app/(chat)/a2a/page.tsx`:

```typescript
import { A2AChannel } from "@/components/a2a/a2a-channel";

export default function A2APage() {
  return (
    <div>
      <h1>Agent-to-Agent Communication</h1>
      <A2AChannel channelId="default-channel" />
    </div>
  );
}
```

#### Step 8: Run Migration

```bash
pnpm db:push  # In development
# or
pnpm db:migrate  # In production
```

---

### Example: Adding a New Tool

Let's add a "Weather API" tool:

#### 1. Define Tool Interface

Create `src/lib/ai/tools/weather/weather-tool.ts`:

```typescript
import { tool } from "ai";
import { z } from "zod";

export const weatherTool = tool({
  description: "Get current weather for a city",
  parameters: z.object({
    city: z.string().describe("City name"),
    units: z.enum(["celsius", "fahrenheit"]).default("celsius"),
  }),
  execute: async ({ city, units }) => {
    // Call weather API
    const response = await fetch(
      `https://api.weather.com/v1/current?city=${city}&units=${units}`,
      {
        headers: { "API-Key": process.env.WEATHER_API_KEY! },
      }
    );
    
    const data = await response.json();
    
    return {
      temperature: data.temp,
      conditions: data.conditions,
      humidity: data.humidity,
    };
  },
});
```

#### 2. Add to Tool Kit

Edit `src/lib/ai/tools/tool-kit.ts`:

```typescript
import { weatherTool } from "./weather/weather-tool";

export const APP_DEFAULT_TOOL_KIT = {
  // ... existing tools
  weather: {
    tools: { weather: weatherTool },
    description: "Weather information",
  },
};
```

#### 3. Add Tool Name Enum

Edit `src/lib/ai/tools/index.ts`:

```typescript
export enum DefaultToolName {
  // ... existing tools
  Weather = "weather",
}
```

#### 4. Test the Tool

Create `src/lib/ai/tools/weather/weather-tool.test.ts`:

```typescript
import { describe, it, expect, vi } from "vitest";
import { weatherTool } from "./weather-tool";

describe("weatherTool", () => {
  it("should fetch weather data", async () => {
    global.fetch = vi.fn().mockResolvedValue({
      json: async () => ({ temp: 72, conditions: "Sunny" }),
    });
    
    const result = await weatherTool.execute({
      city: "San Francisco",
      units: "fahrenheit",
    });
    
    expect(result.temperature).toBe(72);
    expect(result.conditions).toBe("Sunny");
  });
});
```

---

### Data Visualization Tools

Better-chatbot has a built-in **visualization toolkit** that allows AI models to create interactive charts and tables. This section explains how these tools work and how to add custom visualizations.

#### How Visualization Tools Work

The visualization system has two parts:

1. **Tool Definition** (`src/lib/ai/tools/visualization/`) - Defines the schema and executes with `"Success"`
2. **UI Component** (`src/components/tool-invocation/`) - Renders the actual visualization

**Why return "Success"?** The tool's `input` parameters are what matter - they're passed to the React component for rendering. The `execute` function only needs to validate the schema.

#### Existing Visualization Tools

**1. Pie Chart** - `src/lib/ai/tools/visualization/create-pie-chart.ts`

```typescript
import { tool as createTool } from "ai";
import { z } from "zod";

export const createPieChartTool = createTool({
  description: "Create a pie chart",
  inputSchema: z.object({
    data: z.array(z.object({ 
      label: z.string(), 
      value: z.number() 
    })),
    title: z.string(),
    description: z.string().nullable(),
    unit: z.string().nullable(),
  }),
  execute: async () => {
    return "Success";  // UI component handles rendering
  },
});
```

**2. Bar Chart** - Multi-series support

```typescript
export const createBarChartTool = createTool({
  description: "Create a bar chart with multiple data series",
  inputSchema: z.object({
    data: z.array(z.object({
      xAxisLabel: z.string(),
      series: z.array(z.object({
        seriesName: z.string(),
        value: z.number(),
      })),
    })),
    title: z.string(),
    description: z.string().nullable(),
    yAxisLabel: z.string().nullable(),
  }),
  execute: async () => "Success",
});
```

**3. Interactive Table** - With sorting, filtering, search

```typescript
export const createTableTool = createTool({
  description: "Create an interactive table with data. Supports sorting, filtering, and search.",
  inputSchema: z.object({
    title: z.string(),
    description: z.string().nullable(),
    columns: z.array(z.object({
      key: z.string(),
      label: z.string(),
      type: z.enum(["string", "number", "date", "boolean"]).default("string"),
    })),
    data: z.array(z.object({}).catchall(z.any())),
  }),
  execute: async () => "Success",
});
```

**4. Line Chart** - Time series and trend data

```typescript
export const createLineChartTool = createTool({
  description: "Create a line chart for time series or trend data",
  inputSchema: z.object({
    data: z.array(z.object({
      xAxisLabel: z.string(),
      series: z.array(z.object({
        seriesName: z.string(),
        value: z.number(),
      })),
    })),
    title: z.string(),
    description: z.string().nullable(),
    xAxisLabel: z.string().nullable(),
    yAxisLabel: z.string().nullable(),
  }),
  execute: async () => "Success",
});
```

#### Rendering Pipeline

**1. Tool is called by AI:**
```typescript
// AI decides to use createPieChart tool
{
  toolName: "createPieChart",
  input: {
    title: "Browser Market Share",
    data: [
      { label: "Chrome", value: 65 },
      { label: "Safari", value: 18 },
      { label: "Firefox", value: 10 },
    ],
  }
}
```

**2. Message part is created:**
The tool call becomes a `ToolUIPart` in the message:
```typescript
{
  type: "tool-call",
  toolCallId: "call_xyz",
  toolName: "createPieChart",
  args: { /* input params */ },
  state: "output-available",
  output: "Success",
}
```

**3. Component is rendered:**
In `src/components/message-parts.tsx`:
```typescript
// Dynamic import for code splitting
const PieChart = dynamic(
  () => import("./tool-invocation/pie-chart").then((mod) => mod.PieChart),
  { ssr: false }
);

// Render based on tool name
if (state === "output-available") {
  switch (toolName) {
    case DefaultToolName.CreatePieChart:
      return <PieChart {...input} />;
    case DefaultToolName.CreateBarChart:
      return <BarChart {...input} />;
    case DefaultToolName.CreateTable:
      return <InteractiveTable {...input} />;
  }
}
```

**4. React component renders visualization:**
`src/components/tool-invocation/pie-chart.tsx`:
```typescript
import { Pie, PieChart as RechartsPieChart } from "recharts";

export function PieChart({ title, data, unit, description }: PieChartProps) {
  return (
    <Card>
      <CardHeader>
        <CardTitle>{title}</CardTitle>
        {description && <CardDescription>{description}</CardDescription>}
      </CardHeader>
      <CardContent>
        <ChartContainer config={chartConfig}>
          <RechartsPieChart>
            <Pie data={chartData} dataKey="value" nameKey="label" />
            {/* ... */}
          </RechartsPieChart>
        </ChartContainer>
      </CardContent>
    </Card>
  );
}
```

#### Adding a Custom Visualization Tool

Let's add a **Scatter Plot** visualization:

##### Step 1: Create Tool Definition

Create `src/lib/ai/tools/visualization/create-scatter-plot.ts`:

```typescript
import { tool as createTool } from "ai";
import { z } from "zod";

export const createScatterPlotTool = createTool({
  description: "Create a scatter plot to show correlation between two variables",
  inputSchema: z.object({
    title: z.string().describe("Chart title"),
    description: z.string().nullable().describe("Chart description"),
    xAxisLabel: z.string().describe("X-axis label"),
    yAxisLabel: z.string().describe("Y-axis label"),
    data: z.array(
      z.object({
        x: z.number().describe("X coordinate"),
        y: z.number().describe("Y coordinate"),
        label: z.string().optional().describe("Point label (optional)"),
        group: z.string().optional().describe("Group for color coding (optional)"),
      })
    ).describe("Array of data points"),
    showTrendLine: z.boolean().default(false).describe("Show trend line"),
  }),
  execute: async () => {
    return "Success";
  },
});
```

##### Step 2: Add to Tool Kit

Edit `src/lib/ai/tools/tool-kit.ts`:

```typescript
import { createScatterPlotTool } from "./visualization/create-scatter-plot";

export const APP_DEFAULT_TOOL_KIT = {
  [AppDefaultToolkit.Visualization]: {
    [DefaultToolName.CreatePieChart]: createPieChartTool,
    [DefaultToolName.CreateBarChart]: createBarChartTool,
    [DefaultToolName.CreateLineChart]: createLineChartTool,
    [DefaultToolName.CreateTable]: createTableTool,
    [DefaultToolName.CreateScatterPlot]: createScatterPlotTool,  // Add this
  },
  // ... other toolkits
};
```

##### Step 3: Add Tool Name Enum

Edit `src/lib/ai/tools/index.ts`:

```typescript
export enum DefaultToolName {
  CreatePieChart = "createPieChart",
  CreateBarChart = "createBarChart",
  CreateLineChart = "createLineChart",
  CreateTable = "createTable",
  CreateScatterPlot = "createScatterPlot",  // Add this
  // ... other tools
}
```

##### Step 4: Create React Component

Create `src/components/tool-invocation/scatter-plot.tsx`:

```typescript
"use client";

import * as React from "react";
import { Scatter, ScatterChart as RechartsScatterChart, XAxis, YAxis, ZAxis, Tooltip, CartesianGrid, Legend } from "recharts";
import { Card, CardContent, CardDescription, CardHeader, CardTitle } from "@/components/ui/card";
import { ChartContainer, ChartTooltip, ChartTooltipContent } from "@/components/ui/chart";

export interface ScatterPlotProps {
  title: string;
  description?: string;
  xAxisLabel: string;
  yAxisLabel: string;
  data: Array<{
    x: number;
    y: number;
    label?: string;
    group?: string;
  }>;
  showTrendLine?: boolean;
}

export function ScatterPlot({
  title,
  description,
  xAxisLabel,
  yAxisLabel,
  data,
  showTrendLine = false,
}: ScatterPlotProps) {
  // Group data by group field (if exists)
  const groupedData = React.useMemo(() => {
    const groups = new Map<string, typeof data>();
    
    data.forEach(point => {
      const groupName = point.group || "default";
      if (!groups.has(groupName)) {
        groups.set(groupName, []);
      }
      groups.get(groupName)!.push(point);
    });
    
    return Array.from(groups.entries());
  }, [data]);
  
  // Calculate trend line (simple linear regression)
  const trendLineData = React.useMemo(() => {
    if (!showTrendLine) return null;
    
    const n = data.length;
    const sumX = data.reduce((sum, p) => sum + p.x, 0);
    const sumY = data.reduce((sum, p) => sum + p.y, 0);
    const sumXY = data.reduce((sum, p) => sum + p.x * p.y, 0);
    const sumX2 = data.reduce((sum, p) => sum + p.x * p.x, 0);
    
    const slope = (n * sumXY - sumX * sumY) / (n * sumX2 - sumX * sumX);
    const intercept = (sumY - slope * sumX) / n;
    
    const minX = Math.min(...data.map(p => p.x));
    const maxX = Math.max(...data.map(p => p.x));
    
    return [
      { x: minX, y: slope * minX + intercept },
      { x: maxX, y: slope * maxX + intercept },
    ];
  }, [data, showTrendLine]);

  return (
    <Card>
      <CardHeader>
        <CardTitle>{title}</CardTitle>
        {description && <CardDescription>{description}</CardDescription>}
      </CardHeader>
      <CardContent>
        <ChartContainer
          config={{
            scatter: {
              label: "Data Points",
              color: "hsl(var(--chart-1))",
            },
          }}
        >
          <RechartsScatterChart
            margin={{ top: 20, right: 20, bottom: 20, left: 20 }}
          >
            <CartesianGrid strokeDasharray="3 3" />
            <XAxis 
              type="number" 
              dataKey="x" 
              name={xAxisLabel}
              label={{ value: xAxisLabel, position: "bottom" }}
            />
            <YAxis 
              type="number" 
              dataKey="y" 
              name={yAxisLabel}
              label={{ value: yAxisLabel, angle: -90, position: "left" }}
            />
            <ZAxis range={[50, 400]} />
            <ChartTooltip content={<ChartTooltipContent />} />
            <Legend />
            
            {groupedData.map(([groupName, points], index) => (
              <Scatter
                key={groupName}
                name={groupName}
                data={points}
                fill={`hsl(var(--chart-${(index % 5) + 1}))`}
              />
            ))}
            
            {trendLineData && (
              <Scatter
                name="Trend Line"
                data={trendLineData}
                fill="hsl(var(--muted-foreground))"
                line
                shape="none"
              />
            )}
          </RechartsScatterChart>
        </ChartContainer>
      </CardContent>
    </Card>
  );
}
```

##### Step 5: Register Component in Message Parts

Edit `src/components/message-parts.tsx`:

```typescript
// Add dynamic import (around line 663)
const ScatterPlot = dynamic(
  () => import("./tool-invocation/scatter-plot").then((mod) => mod.ScatterPlot),
  { ssr: false, loading: () => <div>Loading chart...</div> }
);

// Add to switch statement (around line 907)
if (state === "output-available") {
  switch (toolName) {
    case DefaultToolName.CreatePieChart:
      return <PieChart {...input} />;
    case DefaultToolName.CreateBarChart:
      return <BarChart {...input} />;
    case DefaultToolName.CreateLineChart:
      return <LineChart {...input} />;
    case DefaultToolName.CreateTable:
      return <InteractiveTable {...input} />;
    case DefaultToolName.CreateScatterPlot:  // Add this
      return <ScatterPlot {...input} />;
  }
}
```

##### Step 6: Add Tool Icon (Optional)

Edit `src/components/default-tool-icon.tsx`:

```typescript
import { ScatterChart } from "lucide-react";

export function getDefaultToolIcon(name: string) {
  // ... existing conditions
  
  if (name === DefaultToolName.CreateScatterPlot) {
    return <ScatterChart className="size-4" />;
  }
  
  // ... rest of function
}
```

##### Step 7: Test Your Tool

Create `src/lib/ai/tools/visualization/create-scatter-plot.test.ts`:

```typescript
import { describe, it, expect } from "vitest";
import { createScatterPlotTool } from "./create-scatter-plot";

describe("createScatterPlotTool", () => {
  it("should validate correct input", async () => {
    const result = await createScatterPlotTool.execute({
      title: "Height vs Weight",
      xAxisLabel: "Height (cm)",
      yAxisLabel: "Weight (kg)",
      data: [
        { x: 170, y: 70 },
        { x: 180, y: 80 },
        { x: 160, y: 60 },
      ],
      showTrendLine: true,
    });
    
    expect(result).toBe("Success");
  });
  
  it("should validate grouped data", async () => {
    const result = await createScatterPlotTool.execute({
      title: "Sales Analysis",
      xAxisLabel: "Marketing Spend",
      yAxisLabel: "Revenue",
      data: [
        { x: 1000, y: 5000, group: "Q1" },
        { x: 1200, y: 6000, group: "Q1" },
        { x: 1500, y: 7500, group: "Q2" },
      ],
      showTrendLine: false,
    });
    
    expect(result).toBe("Success");
  });
});
```

##### Step 8: Try It in Chat

Ask the AI:
```
"Create a scatter plot showing the relationship between study hours and test scores:
- Student A: 2 hours, 65 points
- Student B: 4 hours, 75 points
- Student C: 6 hours, 85 points
- Student D: 8 hours, 95 points

Include a trend line."
```

The AI will use your new `createScatterPlot` tool! 🎉

#### Key Points for Custom Visualizations

1. **Tool Schema is UI Props**: The `inputSchema` becomes the React component props
2. **Return "Success"**: The execute function just validates, rendering happens client-side
3. **Use Dynamic Imports**: Prevents large chart libraries from bloating the main bundle
4. **Leverage Recharts**: Already included, provides extensive chart types
5. **Add Loading States**: Dynamic imports should have loading fallbacks
6. **Type Safety**: Export prop interfaces from components for type checking
7. **Responsive Design**: Use ChartContainer from shadcn for responsive charts

#### Advanced: Interactive Visualizations

You can make visualizations interactive by adding state and callbacks:

```typescript
export function InteractiveScatterPlot({ data, ...props }: ScatterPlotProps) {
  const [selectedPoint, setSelectedPoint] = React.useState<number | null>(null);
  
  return (
    <RechartsScatterChart onClick={(data) => {
      setSelectedPoint(data.activePayload?.[0]?.payload);
    }}>
      {/* Show details when point is clicked */}
      {selectedPoint && (
        <div className="absolute top-0 right-0 p-4 bg-card">
          <h4>Point Details</h4>
          <p>X: {selectedPoint.x}</p>
          <p>Y: {selectedPoint.y}</p>
        </div>
      )}
    </RechartsScatterChart>
  );
}
```

#### Export Functionality

The `InteractiveTable` component shows how to add export features:

```typescript
// Add export buttons
const exportToCSV = () => {
  const csv = /* generate CSV from data */;
  const blob = new Blob([csv], { type: "text/csv" });
  const url = URL.createObjectURL(blob);
  const a = document.createElement("a");
  a.href = url;
  a.download = `${title}.csv`;
  a.click();
};

return (
  <Card>
    <CardHeader>
      <Button onClick={exportToCSV}>Export CSV</Button>
    </CardHeader>
    {/* ... chart */}
  </Card>
);
```

---

### Example: Creating a New API Endpoint

Add an endpoint to get user statistics:

#### 1. Create Route Handler

Create `src/app/api/user/stats/route.ts`:

```typescript
import { getSession } from "auth/server";
import { chatRepository, agentRepository } from "lib/db/repository";

export async function GET(request: Request) {
  const session = await getSession();
  
  if (!session?.user.id) {
    return new Response("Unauthorized", { status: 401 });
  }
  
  // Gather stats
  const threads = await chatRepository.selectThreadsByUserId(session.user.id);
  const agents = await agentRepository.selectByUserId(session.user.id);
  
  const stats = {
    totalChats: threads.length,
    totalAgents: agents.length,
    totalMessages: threads.reduce((sum, t) => sum + (t.messages?.length || 0), 0),
  };
  
  return Response.json(stats);
}
```

#### 2. Create SWR Hook

Create `src/hooks/queries/use-user-stats.ts`:

```typescript
import useSWR from "swr";

type UserStats = {
  totalChats: number;
  totalAgents: number;
  totalMessages: number;
};

const fetcher = (url: string) => fetch(url).then(r => r.json());

export function useUserStats() {
  return useSWR<UserStats>("/api/user/stats", fetcher);
}
```

#### 3. Use in Component

```typescript
import { useUserStats } from "@/hooks/queries/use-user-stats";

export function UserStatsCard() {
  const { data: stats, isLoading } = useUserStats();
  
  if (isLoading) return <div>Loading...</div>;
  
  return (
    <div>
      <h3>Your Statistics</h3>
      <p>Total Chats: {stats?.totalChats}</p>
      <p>Total Agents: {stats?.totalAgents}</p>
      <p>Total Messages: {stats?.totalMessages}</p>
    </div>
  );
}
```

---

## Common Development Patterns

### Error Handling with `ts-safe`

```typescript
import { safe, errorIf } from "ts-safe";

// Chaining operations safely
const result = await safe(async () => {
    const user = await getUser();
    return errorIf(!user, "User not found");
  })
  .map(user => user.email)
  .orElse("unknown@example.com");
```

### Logging with Context

```typescript
import logger from "logger";
import { colorize } from "consola/utils";

const requestLogger = logger.withDefaults({
  message: colorize("blue", `[Request ${requestId}] `),
});

requestLogger.info("Processing request");
requestLogger.error("Request failed", error);
```

### Object Transformation

```typescript
import { objectFlow } from "lib/utils";

const filtered = objectFlow(tools)
  .filter(tool => tool.enabled)
  .map(tool => ({ ...tool, enhanced: true }));
```

### Conditional Rendering with Safe Access

```typescript
import { safe } from "ts-safe";

const userName = safe(() => user?.profile?.name).orElse("Anonymous");
```

---

## Debugging Tips

### 1. Enable Detailed Logging

Set environment variable:
```env
LOG_LEVEL=debug
```

### 2. Inspect MCP Tool Calls

Add logging in `src/lib/ai/mcp/create-mcp-clients-manager.ts`:

```typescript
logger.info(`Calling tool ${toolName} with params:`, params);
const result = await client.toolInfo.execute(params);
logger.info(`Tool result:`, result);
```

### 3. Debug AI Responses

In `src/app/api/chat/route.ts`:

```typescript
logger.info(`System prompt: ${systemPrompt}`);
logger.info(`Tool count: ${Object.keys(vercelAITooles).length}`);
logger.info(`Messages: ${JSON.stringify(messages)}`);
```

### 4. Inspect Database Queries

Use Drizzle Studio:
```bash
pnpm db:studio
```

### 5. Test E2E with Headed Browser

```bash
pnpm test:e2e -- --headed --debug
```

### 6. Check Network Requests

In browser DevTools, filter by:
- `/api/chat` - Chat requests
- `/api/mcp` - MCP operations
- `/api/workflow` - Workflow execution

---

## Performance Considerations

### 1. Tool Loading Optimization

**Problem**: Loading all MCP tools adds hundreds of tokens to each request.

**Solution**: Use mentions and allowed servers to filter tools.

```typescript
// Good: Only load mentioned tools
const tools = await loadMcpTools({ mentions, allowedMcpServers });

// Bad: Load all tools
const allTools = await mcpClientsManager.tools();
```

### 2. Database Query Optimization

**Use select specific fields**:
```typescript
// Good
const threads = await db
  .select({ id: ChatThreadTable.id, title: ChatThreadTable.title })
  .from(ChatThreadTable);

// Bad: Fetches all columns including large JSON
const threads = await db.select().from(ChatThreadTable);
```

### 3. Client-Side Caching

**Use SWR for automatic caching**:
```typescript
const { data } = useSWR("/api/agents", fetcher, {
  revalidateOnFocus: false,  // Don't refetch on tab focus
  dedupingInterval: 60000,   // Cache for 1 minute
});
```

### 4. Streaming Responses

**Always use streaming for AI responses**:
```typescript
// Good: Streaming
const result = streamText({ model, messages });

// Bad: Waiting for full response
const result = await generateText({ model, messages });
```

### 5. Database Indexes

Add indexes for frequent queries:
```typescript
// In schema.pg.ts
export const ChatMessageTable = pgTable(
  "chat_message",
  { /* columns */ },
  (table) => ({
    threadIdx: index("chat_message_thread_idx").on(table.threadId),
    userIdx: index("chat_message_user_idx").on(table.userId),
  })
);
```

---

## Next Steps

- Review `CONTRIBUTING.md` for PR guidelines
- Check `tests/PLAYWRIGHT-TEST-STRATEGY.md` for testing details
- Explore `src/app/api/chat/route.ts` for the main chat implementation
- Read `src/lib/ai/prompts.ts` to understand prompt engineering patterns

**Happy coding! 🚀**
