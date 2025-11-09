# AI Coding Agent Instructions

## Project Overview
Better-chatbot is a Next.js 15 AI chatbot with multi-LLM support (OpenAI, Anthropic, Google, xAI, Ollama), Model Context Protocol (MCP) integration, visual workflows, custom agents, and voice chat. Built with Vercel AI SDK, Drizzle ORM (PostgreSQL), and Better Auth.

## Critical Architecture Patterns

### MCP (Model Context Protocol) Integration
- **MCP Manager**: `src/lib/ai/mcp/mcp-manager.ts` - Global singleton managing MCP clients and tools
- **Storage Modes**: Database-based (default) or file-based (set `FILE_BASED_MCP_CONFIG=true`)
  - DB storage: `src/lib/ai/mcp/db-mcp-config-storage.ts`
  - File storage: `src/lib/ai/mcp/fb-mcp-config-storage.ts`
- **Tool Loading**: MCP tools are dynamically loaded per chat request based on mentions and allowed servers
- **Custom Tool IDs**: Use `createMCPToolId(serverName, toolName)` from `src/lib/ai/mcp/mcp-tool-id.ts`

### Chat Flow Architecture
1. **Request**: `src/app/api/chat/route.ts` receives chat messages with mentions, tool choices, and attachments
2. **Tool Loading**: Three tool categories loaded conditionally:
   - `MCP_TOOLS`: From MCP servers (loaded via `loadMcpTools`)
   - `WORKFLOW_TOOLS`: User-created visual workflows (loaded via `loadWorkFlowTools`)
   - `APP_DEFAULT_TOOLS`: Built-in tools like web search, JS/Python execution (loaded via `loadAppDefaultTools`)
3. **System Prompts**: Merged from user preferences, agent instructions, and MCP customizations via `buildUserSystemPrompt()`, `buildMcpServerCustomizationsSystemPrompt()`
4. **Streaming**: Uses Vercel AI SDK's `streamText` with `smoothStream` for word-chunked responses
5. **Persistence**: Messages saved to DB via `chatRepository.upsertMessage()` with parts converted by `convertToSavePart()`

### Visual Workflows
- **Node Types**: LLM nodes (AI reasoning) and Tool nodes (MCP tool execution)
- **Execution**: `src/lib/ai/workflow/executor/node-executor.ts` orchestrates node execution with data flow
- **Tool Conversion**: Published workflows become callable tools via `src/lib/ai/workflow/tool-executor.ts`
- **Storage**: `workflowRepository` manages workflow structure (nodes, edges) and metadata

### Custom Agents
- **Definition**: Custom system prompts + selected tools + mentions, stored in `agentRepository`
- **Invocation**: `@agent_name` in chat loads agent via `rememberAgentAction()`
- **Tool Merging**: Agent-defined mentions are merged with user mentions before tool loading

### Mentions System (`@`)
- **Types**: `@tool`, `@mcp`, `@workflow`, `@agent` - see `src/types/chat.ts` for `ChatMention` schema
- **Binding**: Mentioned tools are temporarily bound for that specific request (token-efficient)
- **Tool Selection vs Mentions**: Tool selection makes tools always available; mentions bind only for one response

### Repository Pattern
- **Single Source**: All data access through `src/lib/db/repository.ts` exports
- **Available Repos**: `chatRepository`, `userRepository`, `mcpRepository`, `workflowRepository`, `agentRepository`, `archiveRepository`, `bookmarkRepository`
- **Schema**: PostgreSQL schema in `src/lib/db/pg/schema.pg.ts`
- **Migrations**: Use `pnpm db:migrate` (runs `scripts/db-migrate.ts`)

### Authentication & Permissions
- **Better Auth**: Server auth via `auth/server` (`getSession()`), client via `auth/client` (`useSession()`)
- **Roles**: `admin`, `editor`, `user` - defined in `src/lib/auth/roles.ts`
- **Permissions**: Fine-grained checks in `src/lib/auth/permissions.ts` (e.g., `canEditWorkflow()`, `canManageUsers()`)
- **Middleware**: `src/middleware.ts` redirects unauthenticated users except for auth/export/static routes

### File Storage Abstraction
- **Driver**: `serverFileStorage` from `src/lib/file-storage/index.ts` - auto-resolves to Vercel Blob or S3
- **Interface**: `upload()`, `download()`, `delete()` - always use this abstraction, never direct storage calls
- **Configuration**: `FILE_STORAGE_TYPE` env var (defaults to `vercel-blob`)

## Path Aliases (tsconfig.json)
- `ui/*` → `src/components/ui/*`
- `auth/*` → `src/lib/auth/*`
- `app-types/*` → `src/types/*`
- `lib/*` → `src/lib/*`
- `logger` → `src/lib/logger.ts`
- `load-env` → `src/lib/load-env.ts`
- `@/*` → `src/*`

## Development Workflows

### Running the App
```bash
pnpm dev              # Development with hot-reload
pnpm build:local      # Build for local HTTPS (sets NO_HTTPS=1)
pnpm start            # Production mode
pnpm docker-compose:up # Full stack with PostgreSQL
```

### Testing Strategy
- **Unit Tests**: Vitest for logic in `src/lib/**/*.test.ts` - run `pnpm test`
- **E2E Tests**: Playwright in `tests/**/*.spec.ts` - run `pnpm test:e2e`
  - **Critical**: Tests require seeded DB - setup runs automatically via `tests/lifecycle/auth-states.setup.ts`
  - **First-user tests**: Verify admin role assignment - run separately with `pnpm test:e2e:first-user`
  - **Test users**: Seeded by `scripts/seed-test-users.ts` (admin, editor, regular user)
  - **Auth states**: Saved as JSON for reuse across tests
- **Pre-commit**: `pnpm check` runs lint, typecheck, and tests

### Database Operations
```bash
pnpm db:push      # Push schema changes (dev)
pnpm db:migrate   # Run migrations (prod)
pnpm db:studio    # Open Drizzle Studio
pnpm docker:pg    # Start local PostgreSQL
```

### Code Quality
- **Linting**: ESLint + Biome - `pnpm lint` or `pnpm lint:fix`
- **Formatting**: Biome (2 spaces, LF, width 80, double quotes) - `pnpm format`
- **Type Checking**: `pnpm check-types`

## Project-Specific Conventions

### Naming
- **Components**: `PascalCase.tsx` (e.g., `ChatBot.tsx`)
- **Hooks**: `camelCase.ts` with `use` prefix (e.g., `useChat.ts`)
- **Utilities**: `camelCase.ts` (e.g., `fuzzy-search.ts`)
- **Repositories**: `kebab-case-repository.pg.ts` pattern

### Component Organization
- **UI Components**: `src/components/ui/*` - shadcn-based primitives
- **Feature Components**: `src/components/*` - domain-specific (chat, mcp, workflow, agent)
- **Layouts**: `src/components/layouts/*` - page wrappers
- **Route Layouts**: `src/app/(auth)`, `src/app/(chat)`, `src/app/(public)` - group routes with shared layouts

### State Management
- **Server State**: SWR hooks in `src/hooks/*` (e.g., `useMcpServers()`)
- **Client State**: Zustand stores in `src/app/store/*` (e.g., `useWorkflowStore`)
- **Never mix**: Don't use client state for server data - always use SWR for fetching

### Error Handling
- **Safe Operations**: Use `safe()` from `ts-safe` for nullable chains
- **Error Utils**: `errorIf()` from `ts-safe` for conditional errors
- **Logging**: Import `logger` alias, use `.withDefaults()` for context, `colorize()` for readability

### AI SDK Patterns
- **Model Selection**: `customModelProvider.getModel(chatModel)` from `src/lib/ai/models.ts`
- **Tool Definitions**: Always include `description` and `parameters` schema - LLMs use this for decision-making
- **Streaming**: Use `createUIMessageStream` for real-time UI updates with `dataStream.write()` events
- **Tool Choice**: `auto` (default), `manual` (require permission), `none` (disable tools), `required` (force tool call)

## Common Pitfalls

1. **MCP Tool Loading**: Don't load all MCP tools blindly - filter by `allowedMcpServers` and mentions to reduce token usage
2. **Message Parts**: Convert parts with `convertToSavePart()` before DB insert - handles tool call serialization
3. **Authentication**: Always check `session.user.id` matches resource owner before mutations
4. **File Storage**: Never hardcode Vercel Blob - use `serverFileStorage` interface for portability
5. **Migrations**: Use `pnpm db:migrate` for production - `db:push` is dev-only and can lose data
6. **Environment**: Always load `load-env` at top of scripts/configs - it handles `.env` parsing
7. **Server-Only Code**: Mark server-only files with `import "server-only"` to prevent client bundling
8. **Tool Execution**: For manual tool mode, use `excludeToolExecution()` to prevent auto-execution while preserving tool definitions

## PR & Commit Guidelines
- **PR Titles**: Must follow Conventional Commits (`feat:`, `fix:`, `chore:`, `docs:`)
- **Squash Merge**: Used for clean history - only PR title matters for changelog
- **Visuals Required**: Include before/after screenshots for UI changes
- **Test Coverage**: Add unit tests for logic, E2E tests for features/bugs
- **Feature Requests**: Create issue first for discussion before implementing

## Key Files to Reference
- `AGENTS.md` - Repository guidelines (already attached)
- `CONTRIBUTING.md` - Contribution workflow
- `tests/PLAYWRIGHT-TEST-STRATEGY.md` - E2E test orchestration
- `src/app/api/chat/route.ts` - Main chat endpoint (reference implementation)
- `src/lib/ai/prompts.ts` - System prompt builders
- `src/lib/db/repository.ts` - Data access layer
- `src/types/*.ts` - TypeScript schemas (use for validation patterns)
