# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# First-time setup (install deps, generate Prisma client, run migrations)
npm run setup

# Development server (Turbopack)
npm run dev

# Build for production
npm run build

# Lint
npm run lint

# Run all tests
npm test

# Run a single test file
npx vitest run src/lib/__tests__/file-system.test.ts

# Reset the database (destructive)
npm run db:reset

# Regenerate Prisma client after schema changes
npx prisma generate

# Create and apply a new migration
npx prisma migrate dev
```

## Architecture

UIGen is an AI-powered React component generator with live preview. Users describe a component in chat; the AI uses tool calls to write files into a virtual file system; the UI renders the result in an iframe sandbox.

### Request flow

1. User submits a message in `ChatInterface`
2. `ChatProvider` (`src/lib/contexts/chat-context.tsx`) calls `/api/chat` via Vercel AI SDK's `useChat`, serializing the current virtual file system with each request
3. `src/app/api/chat/route.ts` reconstructs the VFS, calls `streamText` with two tools (`str_replace_editor`, `file_manager`), and streams back tool calls + text
4. On the client, tool calls are intercepted by `onToolCall` and dispatched to `FileSystemContext` (`src/lib/contexts/file-system-context.tsx`), which updates the in-memory VFS
5. `PreviewFrame` watches the VFS via `refreshTrigger`, transforms all JSX/TSX files with Babel Standalone (`src/lib/transform/jsx-transformer.ts`), builds an import map with blob URLs, and renders a self-contained HTML page in an `<iframe>`

### Virtual File System

`src/lib/file-system.ts` — `VirtualFileSystem` is a pure in-memory tree. It has no disk I/O. The AI always receives the full serialized VFS on each turn. Every project must have a `/App.jsx` as the entry point; all local imports use the `@/` alias.

### AI provider

`src/lib/provider.ts` — When `ANTHROPIC_API_KEY` is set, uses `claude-haiku-4-5` via `@ai-sdk/anthropic`. Without a key, `MockLanguageModel` returns static component scaffolding so the app is fully usable without credentials.

### Tools given to the AI

- `str_replace_editor` (`src/lib/tools/str-replace.ts`) — view, create, str_replace, insert on VFS paths
- `file_manager` (`src/lib/tools/file-manager.ts`) — rename and delete VFS paths

### Authentication

JWT-based, stored in an `httpOnly` cookie. Logic lives in `src/lib/auth.ts` (server-only). Anonymous users can use the app without signing up; their work is tracked in `src/lib/anon-work-tracker.ts`.

### Persistence

Prisma + SQLite (`prisma/dev.db`). Schema: `User` → many `Project`s. Each `Project` stores the full chat `messages` array and VFS `data` as JSON strings. The Prisma client is generated into `src/generated/prisma/`.

### Key contexts

| Context | File | What it owns |
|---|---|---|
| `FileSystemContext` | `src/lib/contexts/file-system-context.tsx` | VFS instance, selected file, tool-call dispatcher |
| `ChatContext` | `src/lib/contexts/chat-context.tsx` | AI chat state via `useAIChat` |

Both are initialized in `MainContent` (`src/app/main-content.tsx`), which is the client shell shared by the index route (anonymous) and the `[projectId]` route (authenticated).

### Preview rendering

`src/lib/transform/jsx-transformer.ts` handles two things:
- `transformJSX` — Babel-transforms a single JSX/TSX file to JS (strips TypeScript, handles CSS imports)
- `createImportMap` — Transforms all VFS files, creates blob URLs, builds an ES module import map. Third-party packages are resolved via `https://esm.sh/`. Missing local imports get placeholder modules so the preview doesn't hard-crash.
- `createPreviewHTML` — Returns a full HTML document that loads Tailwind from CDN, injects the import map, and bootstraps `ReactDOM.createRoot`

### Node compatibility shim

`node-compat.cjs` is required via `NODE_OPTIONS` in all npm scripts to polyfill Node built-ins that Next.js 15 + Turbopack needs on Windows.

