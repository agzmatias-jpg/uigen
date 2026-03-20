# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Development
npm run dev          # Start dev server with Turbopack
npm run dev:daemon   # Start in background, logs to logs.txt
npm run build        # Production build
npm run lint         # ESLint

# Database
npm run setup        # Install deps + generate Prisma client + run migrations
npm run db:reset     # Reset SQLite database

# Testing
npm run test         # Run all Vitest tests
npx vitest run src/path/to/__tests__/file.test.tsx  # Run a single test file
```

Set `ANTHROPIC_API_KEY` in `.env` to use real Claude. Without it, the app falls back to a `MockLanguageModel` that generates static sample components.

## Architecture

UIGen is an AI-powered React component generator with live preview. Users describe components in natural language; Claude generates and modifies code via structured tool calls in real time.

### Request Flow

```
User message → ChatInterface
  → POST /api/chat (with messages + serialized VirtualFileSystem)
  → Claude (claude-haiku-4-5) streams response
  → Tool calls: str_replace_editor / file_manager → VirtualFileSystem mutations
  → FileSystemContext updates React state
  → PreviewFrame re-renders live preview
  → onFinish → serialize FS + messages → save to Prisma DB (if authenticated)
```

### Key Concepts

**VirtualFileSystem** (`src/lib/file-system.ts`): All generated files exist only in memory as a `Map`. No disk writes. The FS is serialized to JSON for DB persistence and deserialized on project load.

**Claude Tools** (`src/lib/tools/`):
- `str_replace_editor` — find/replace content in a file
- `file_manager` — create, delete, rename files

**FileSystemContext / ChatContext** (`src/lib/contexts/`) — React contexts wiring the virtual FS and chat state together across components.

**Live Preview** (`src/components/preview/PreviewFrame.tsx`): Transpiles JSX in-browser using Babel standalone and renders the active component.

**LLM Provider** (`src/lib/provider.ts`): Returns `anthropic('claude-haiku-4-5-20251001')` if `ANTHROPIC_API_KEY` is set, otherwise `MockLanguageModel`.

### Auth

JWT sessions (7-day expiry, HTTP-only cookie) via `jose` + bcrypt. Anonymous users can generate components but cannot save projects. Middleware at `src/middleware.ts` protects routes. Server actions for sign-up/sign-in are in `src/actions/index.ts`.

### Database

SQLite via Prisma. Schema: `User` (id, email, hashed password) → `Project` (id, name, userId, messages as JSON string, data as JSON serialized VirtualFileSystem). Prisma client is generated into `src/generated/prisma/`.

### UI Layout

`src/app/main-content.tsx` — resizable split-pane shell:
- Left panel (35%): `ChatInterface`
- Right panel (65%): Tabs for `PreviewFrame` (live render) and `CodeEditor` (Monaco) + `FileTree`

### Tech Stack

Next.js 15 App Router · React 19 · TypeScript (strict) · Tailwind CSS v4 · Shadcn/ui · Monaco Editor · Vercel AI SDK · Anthropic SDK · Prisma + SQLite · Vitest · JWT (jose) · Babel standalone (in-browser JSX transpilation)

Path alias: `@/*` → `./src/*`
