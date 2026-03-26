# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Setup

```bash
npm run setup        # Install deps, generate Prisma client, run migrations
```

Set `ANTHROPIC_API_KEY` in `.env` (optional — without it, a mock model returns static responses).

## Commands

```bash
npm run dev          # Dev server with Turbopack (uses NODE_OPTIONS shim via node-compat.cjs)
npm run dev:daemon   # Dev server in background, logs to logs.txt
npm run build        # Production build
npm run lint         # ESLint
npm run test         # Vitest
npm run db:reset     # Force-reset SQLite database
```

Run a single test file: `npx vitest run src/lib/__tests__/file-system.test.ts`

## Architecture

UIGen is a Next.js 15 App Router application. The core flow: user describes a component in chat → request streams to `/api/chat` → Claude responds with tool calls → virtual file system updates → iframe rerenders the component.

### Key Subsystems

**Chat & AI** (`src/app/api/chat/route.ts`, `src/lib/provider.ts`)
- The `/api/chat` route streams responses using Vercel AI SDK with Anthropic Claude
- Claude uses two tools: `str_replace_editor` (create/modify files) and `file_manager` (manage file structure)
- `lib/provider.ts` returns either the real Anthropic model or `MockLanguageModel` when no API key is set
- System prompts live in `src/lib/prompts/`

**Virtual File System** (`src/lib/file-system.ts`, `src/lib/contexts/FileSystemContext.tsx`)
- All generated files are stored in-memory — nothing is written to disk
- `VirtualFileSystem` class manages file CRUD; serialized to JSON for database persistence
- `FileSystemContext` wraps this for React consumption

**Preview** (`src/components/preview/PreviewFrame.tsx`, `src/lib/transform/jsx-transformer.ts`)
- Generated components are transformed in-browser using Babel Standalone (JSX/TS → JS)
- Transformed code runs in an isolated iframe with an import map pointing to esm.sh for React and other CDN deps

**Authentication** (`src/lib/auth.ts`, `src/middleware.ts`, `src/actions/index.ts`)
- JWT sessions via `jose`, stored in HTTP-only cookies, 7-day expiry
- Passwords hashed with bcrypt
- Middleware protects `[projectId]` routes; unauthenticated users work anonymously with state in localStorage (`lib/anon-work-tracker.ts`)

**Database** (`prisma/schema.prisma`)
- SQLite via Prisma with two models: `User` and `Project`
- `Project.messages` and `Project.data` are JSON columns storing chat history and virtual file system state

### Path Alias

`@/*` maps to `./src/*` (configured in `tsconfig.json`).

### UI Components

Shadcn/UI components (Radix UI primitives + Tailwind) live in `src/components/ui/`. Use `cn()` from `src/lib/utils.ts` for conditional class merging.
