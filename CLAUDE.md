# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Initial setup (install deps + generate Prisma client + run migrations)
npm run setup

# Development server (uses Turbopack + node-compat shim)
npm run dev

# Background dev server (logs to logs.txt)
npm run dev:daemon

# Build
npm run build

# Lint
npm run lint

# Run all tests
npm test

# Run a single test file
npx vitest run src/lib/__tests__/file-system.test.ts

# Database reset
npm run db:reset
```

The `NODE_OPTIONS='--require ./node-compat.cjs'` prefix is automatically applied by all npm scripts — it patches Node.js built-ins for compatibility. Do not remove it.

## Environment

Create a `.env` file with:
```
ANTHROPIC_API_KEY=your-key   # optional — omit to use the built-in mock provider
JWT_SECRET=your-secret        # optional — defaults to "development-secret-key"
```

Without `ANTHROPIC_API_KEY`, the app uses `MockLanguageModel` (`src/lib/provider.ts`) which returns static component demos.

## Architecture

### Request / AI flow

1. User sends a message → `ChatContext` (`src/lib/contexts/chat-context.tsx`) serializes the virtual file system and POSTs to `/api/chat`.
2. `src/app/api/chat/route.ts` reconstructs a `VirtualFileSystem` from the serialized state, calls `streamText` (Vercel AI SDK) with two tools:
   - `str_replace_editor` — view/create/str_replace/insert operations on files
   - `file_manager` — rename/delete operations
3. The model streams tool calls back to the client. `FileSystemContext` (`src/lib/contexts/file-system-context.tsx`) receives them via `onToolCall` and mutates the in-memory VFS, triggering a `refreshTrigger` increment.
4. `PreviewFrame` (`src/components/preview/PreviewFrame.tsx`) watches `refreshTrigger`, calls `createImportMap` + `createPreviewHTML` from `src/lib/transform/jsx-transformer.ts` to compile all VFS files via Babel and render them in a sandboxed `<iframe>` using blob URLs and an import map.
5. On `onFinish`, if the user is authenticated and a `projectId` exists, the full message history and serialized VFS are persisted to the SQLite `Project` row via Prisma.

### Virtual File System

`VirtualFileSystem` (`src/lib/file-system.ts`) is a pure in-memory tree (no disk I/O). Files live in a flat `Map<path, FileNode>` plus a tree of parent→children references. `serialize()` / `deserializeFromNodes()` convert between the `Map` and a plain `Record<string, FileNode>` for JSON transport.

### Preview pipeline

`src/lib/transform/jsx-transformer.ts` does all client-side transpilation:
- Transforms TSX/JSX via `@babel/standalone`
- Creates blob URLs for each transformed file
- Builds an ES module import map that maps local paths + `@/` aliases to blob URLs, and unknown third-party packages to `https://esm.sh/<package>`
- Missing local imports get placeholder stub modules (avoids hard crashes)
- CSS files are collected into an inline `<style>` tag

Entry point resolution order: `/App.jsx` → `/App.tsx` → `/index.jsx` → `/index.tsx` → `/src/App.jsx` → first `.jsx`/`.tsx` found.

### Auth

Custom JWT-based auth using `jose` (`src/lib/auth.ts`). Sessions are stored in an `httpOnly` cookie (`auth-token`, 7-day expiry). The middleware at `src/middleware.ts` guards `/api/projects` and `/api/filesystem`. The `/api/chat` route is unauthenticated — project saving is skipped silently when no session exists.

Anonymous users get their work stored in `sessionStorage` (`src/lib/anon-work-tracker.ts`) so it can be offered for import after they sign up.

### Data model

SQLite via Prisma (`prisma/schema.prisma`). Two models:
- `User` — email + bcrypt password
- `Project` — belongs to optional `User`, stores `messages` (JSON array) and `data` (serialized VFS JSON)

Projects can exist without a user (`userId` is optional), but saving from the chat route requires an authenticated session.

### Context providers

`FileSystemProvider` wraps the VFS with React state and exposes `handleToolCall` which translates AI SDK tool call payloads into VFS mutations. `ChatProvider` wraps Vercel AI SDK's `useChat` and feeds `handleToolCall` to `onToolCall`. Both are set up in `src/app/[projectId]/page.tsx` and `src/app/main-content.tsx`.

### Testing

Tests use Vitest + jsdom + React Testing Library. Test files live alongside source in `__tests__` subdirectories. The `@/` path alias is resolved via `vite-tsconfig-paths`.
