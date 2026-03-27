# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run dev          # Start dev server with Turbopack
npm run build        # Production build
npm run lint         # Run ESLint
npm run test         # Run all tests (Vitest)
npm run setup        # Full setup: install deps + prisma generate + migrate
npm run db:reset     # Reset and re-migrate the SQLite database
```

Run a single test file:
```bash
npx vitest run src/lib/__tests__/file-system.test.ts
```

## Architecture

**UIGen** is an AI-powered React component generator. Users describe what they want in a chat interface; the AI writes code into a virtual file system; the result renders in a live preview iframe.

### Request Flow

1. User sends a message → `POST /api/chat` (`src/app/api/chat/route.ts`)
2. API streams a response using the Vercel AI SDK with Anthropic (`claude-haiku-4-5`) or a mock provider
3. The AI uses two tools: `str_replace_editor` (create/edit files) and `file_manager` (mkdir/rm/mv)
4. Tool calls are applied to the in-memory `FileSystem` and streamed back to the client
5. For authenticated users, the project (messages + file data) is saved to SQLite via Prisma

### Key Abstractions

**Virtual File System** (`src/lib/file-system.ts`) — an in-memory tree of files. All AI-generated code lives here. It is serialized to JSON for persistence in the `Project.data` column.

**Provider** (`src/lib/provider.ts`) — returns a `MockLanguageModel` when `ANTHROPIC_API_KEY` is absent, otherwise wraps the real Anthropic SDK. The mock returns hardcoded demo components to make the app usable without an API key.

**Contexts** — `ChatContext` (`src/lib/contexts/chat-context.tsx`) manages message history and streaming state. `FileSystemContext` (`src/lib/contexts/file-system-context.tsx`) exposes the virtual FS to components.

**AI Tools** (`src/lib/tools/`) — `str-replace.ts` implements the file-editing tool; `file-manager.ts` handles directory operations. Both mutate the `FileSystem` instance.

**JSX Transformer** (`src/lib/transform/jsx-transformer.ts`) — transpiles JSX/TSX in the virtual FS to plain JS that can run in the preview iframe via `@babel/standalone`.

### Database

SQLite via Prisma. Two models:
- `User` — email/password auth (bcrypt + JWT sessions)
- `Project` — stores `messages` (JSON array) and `data` (serialized FileSystem JSON), linked to an optional User

Auth is handled through server actions (`src/actions/`) and `src/lib/auth.ts` (JWT signing). `src/middleware.ts` protects `/api/chat` and project routes.

### Routing

- `/` — home; redirects authenticated users to their latest project or creates a new one
- `/[projectId]` — loads and displays a saved project
- `/api/chat` — streaming POST endpoint for code generation

### Path Alias

`@/*` resolves to `src/*` (configured in `tsconfig.json`).

## Environment

`ANTHROPIC_API_KEY` in `.env` is optional. Without it, the mock provider is used automatically.
