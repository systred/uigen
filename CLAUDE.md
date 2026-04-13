# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run setup          # One-time setup: install deps, generate Prisma client, run migrations
npm run dev            # Start dev server with Turbopack
npm run build          # Production build
npm run lint           # Run ESLint
npm run test           # Run Vitest tests
npm run db:reset       # Reset database
```

Run a single test file: `npx vitest --run src/lib/__tests__/file-system.test.ts`
Filter by test name: `npx vitest --run -t "test name pattern"`

`npm run setup` must be run before `npm run dev` on a fresh clone — it generates the Prisma client to `src/generated/prisma` and applies migrations.

## Environment Variables

- `ANTHROPIC_API_KEY`: Optional. If not set, a mock provider generates dummy components instead.
- `JWT_SECRET`: Optional. Defaults to `"development-secret-key"` in development.

## Architecture

UIGen is a Next.js 15 (App Router) application that lets users describe React components in natural language and previews them in real-time using Claude AI.

### Core Data Flow

1. User types a prompt → `ChatProvider` (`src/lib/contexts/chat-context.tsx`) sends it to `POST /api/chat`
2. The API route (`src/app/api/chat/route.ts`) calls the LLM (Anthropic or mock) with a system prompt from `src/lib/prompts/generation.tsx`
3. The model uses two tools to write code:
   - `str_replace_editor` (`src/lib/tools/str-replace.ts`) — creates/edits files
   - `file_manager` (`src/lib/tools/file-manager.ts`) — renames/deletes files
4. Tool calls stream back to the client, where `FileSystemProvider` (`src/lib/contexts/file-system-context.tsx`) applies them to the in-memory `VirtualFileSystem` (`src/lib/file-system.ts`)
5. `PreviewFrame` (`src/components/preview/PreviewFrame.tsx`) detects changes, transforms JSX via Babel (`src/lib/transform/jsx-transformer.ts`), maps imports to CDN URLs (esm.sh), and renders the result in a sandboxed iframe

### Key Architectural Decisions

- **Virtual File System**: All file operations are in-memory — no files written to disk. The VFS state is serialized as JSON into the `data` field of the `Project` DB record when saving.
- **Preview rendering**: JSX is transformed in-browser using Babel standalone. Third-party imports are resolved to `esm.sh` CDN URLs via an import map injected into the iframe.
- **Provider abstraction**: `src/lib/provider.ts` returns either the real Anthropic client or a `MockLanguageModel` based on whether `ANTHROPIC_API_KEY` is set.
- **Auth**: JWT sessions stored as httpOnly cookies (7-day expiration). Anonymous users can work without signing in; their in-progress work is tracked in sessionStorage via `src/lib/anon-work-tracker.ts`. Middleware only protects `/api/projects` and `/api/filesystem` routes.
- **Refresh trigger**: `FileSystemProvider` increments a `refreshTrigger` counter after each tool call to signal `PreviewFrame` to re-render. This is the coupling point between chat streaming and live preview.
- **Preview entry point detection**: `PreviewFrame` checks for entry files in order: `/App.jsx`, `/App.tsx`, `/index.jsx`, `/index.tsx`, `/src/App.jsx`, `/src/App.tsx`.
- **Node.js 25+ compatibility**: `node-compat.cjs` (loaded via `NODE_OPTIONS` in package.json) removes global `localStorage`/`sessionStorage` on the server to prevent SSR crashes.

### Database

SQLite via Prisma. Schema has two models: `User` and `Project`. Projects store chat history (`messages`) and filesystem state (`data`) as JSON strings. Run `npx prisma studio` to inspect data.

### Path Alias

`@/*` resolves to `./src/*`.
