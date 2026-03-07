# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

UIGen is an AI-powered React component generator with live preview. Users describe components in a chat, Claude generates React code via tool calls, and a sandboxed iframe renders the result in real-time.

## Development Commands

```bash
npm run setup          # First-time: install deps + prisma generate + migrate
npm run dev            # Dev server with Turbopack
npm run build          # Production build
npm run lint           # ESLint
npm test               # Vitest (watch mode)
npm test -- path/to/file.test.tsx  # Single test file
npm run db:reset       # Reset SQLite database (destructive)
```

Prisma migrations:
```bash
npx prisma migrate dev --name migration_name
npx prisma generate
```

## Architecture

### Core Flow
1. User sends message → `src/app/api/chat/route.ts` receives it with current file system state
2. Vercel AI SDK streams Claude's response with tool calls (`streamText`, up to 40 steps)
3. Two AI tools modify a `VirtualFileSystem` instance:
   - `str_replace_editor` (`src/lib/tools/str-replace.ts`): create, view, edit files via string replacement
   - `file_manager` (`src/lib/tools/file-manager.ts`): rename, delete files/directories
4. Client renders streamed tool results into the virtual file system and updates preview

### Virtual File System
`src/lib/file-system.ts` — Map-based in-memory tree of `FileNode` objects. No files written to disk. Serializes to JSON for database persistence.

### Preview System
`src/components/preview/PreviewFrame.tsx` renders components in a sandboxed iframe:
- `src/lib/transform/jsx-transformer.ts` converts JSX to ES modules via Babel Standalone
- Creates browser import maps with blob URLs for each module
- Entry point: `/App.jsx` (or `.tsx`, `/index.jsx`, `/index.tsx`)

### Routing
- `/` — Anonymous users get `MainContent` directly; authenticated users redirect to most recent project
- `/[projectId]` — Authenticated project page (redirects to `/` if not logged in)
- `/api/chat` — Streaming chat endpoint (120s max duration)

### State Management
Two React Context providers wrap the app in `MainContent`:
- `FileSystemProvider` (`src/lib/contexts/file-system-context.tsx`) — wraps VirtualFileSystem
- `ChatProvider` (`src/lib/contexts/chat-context.tsx`) — manages messages and AI streaming

### Auth & Persistence
- JWT sessions via `jose` (`src/lib/auth.ts`)
- Prisma + SQLite (`prisma/schema.prisma`): `User` and `Project` models
- Prisma client output: `src/generated/prisma` (not default location)
- Anonymous users can use the app without persistence
- `src/lib/anon-work-tracker.ts` tracks unsaved anonymous work

### Mock Provider
When `ANTHROPIC_API_KEY` is absent, `MockLanguageModel` in `src/lib/provider.ts` returns static component examples (counter, form, card). Useful for dev/testing without API costs. Mock uses 4 max steps vs 40 for real API.

## Key Conventions

- All imports use `@/` alias pointing to `src/`
- UI components from shadcn/ui (new-york style) in `src/components/ui/`, configured via `components.json`
- Icons: `lucide-react`
- AI-generated components must use `/App.jsx` as entry point, Tailwind for styling, default exports
- The generation system prompt is in `src/lib/prompts/generation.tsx`

## Testing

Vitest with jsdom environment and React Testing Library. Tests live in `__tests__` directories next to their source:
- `src/components/**/__tests__/*.test.tsx`
- `src/lib/**/__tests__/*.test.ts`

## Tech Stack

- Next.js 15 (App Router) + React 19 + TypeScript
- Anthropic Claude (claude-sonnet-4-0) via Vercel AI SDK
- Prisma + SQLite
- Tailwind CSS v4
- Monaco Editor
- Babel Standalone (JSX→ESM)
- Radix UI primitives

## Environment Variables

```
ANTHROPIC_API_KEY=...   # Optional; mock provider used if absent
JWT_SECRET=...          # Defaults to 'development-secret-key'
```
