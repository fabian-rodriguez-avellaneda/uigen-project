# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Setup

```bash
npm run setup        # Install deps, generate Prisma client, run migrations
```

Set `ANTHROPIC_API_KEY` in `.env` to use real Claude AI; otherwise a mock provider is used automatically.

## Commands

```bash
npm run dev          # Dev server with Turbopack
npm run dev:daemon   # Dev server in background, logs to logs.txt
npm run build        # Production build
npm run lint         # ESLint
npm run test         # Run all tests (Vitest)
npm run db:reset     # Force reset SQLite database
```

Run a single test file:
```bash
npx vitest run src/lib/__tests__/file-system.test.ts
```

## Code Style

- Use comments sparingly. Only comment complex code.

## Architecture

UIGen is a Next.js 15 (App Router) app where users chat with Claude to generate React components, with live preview and a code editor.

### Core Data Flow

1. User sends a message → `ChatProvider` (`src/lib/contexts/chat-context.tsx`) calls `/api/chat`
2. `/api/chat` (`src/app/api/chat/route.ts`) streams responses from Claude using Vercel AI SDK
3. Claude uses two tools: `str_replace_editor` (edit files) and `file_manager` (create/delete/move files)
4. Tool calls update the **virtual file system** (`FileSystemProvider` / `src/lib/file-system.ts`)
5. `PreviewFrame` (`src/components/preview/PreviewFrame.tsx`) transforms files via Babel (`src/lib/transform/jsx-transformer.ts`) and renders them in a sandboxed iframe

### Key Abstractions

- **Virtual File System**: In-memory only — no files are written to disk. The `FileSystem` class handles all CRUD. State lives in `FileSystemProvider` context.
- **AI Provider** (`src/lib/provider.ts`): Returns Claude Haiku 4.5 when `ANTHROPIC_API_KEY` is set, or a mock provider for offline development.
- **JSX Transform**: Babel standalone transforms JSX/TSX to JS at runtime, resolves imports to blob URLs, and injects CSS. Runs client-side inside the iframe pipeline.
- **Persistence**: Projects and chat messages are stored in SQLite via Prisma. `messages` and `data` (file system snapshot) are JSON columns on the `Project` model.
- **Auth**: JWT-based via `jose`, with `bcrypt` for passwords. Middleware at `src/middleware.ts` protects routes.

### Layout

The main UI (`src/app/main-content.tsx`) is a three-panel resizable layout:
- **Left**: Chat (`ChatInterface` + `MessageInput`)
- **Right top**: `PreviewFrame` (iframe)
- **Right bottom**: `FileTree` + `CodeEditor` (Monaco)

### Server Actions

`src/actions/` contains Next.js server actions for project CRUD (`create-project`, `get-project`, `get-projects`). These are the only server-side data access layer outside of the API route.

### Anonymous Users

`src/lib/anon-work-tracker.ts` tracks work done by unauthenticated users (stored client-side). On sign-up/sign-in, anonymous work is migrated to the new user account.

### Database Schema

Reference `prisma/schema.prisma` for the full schema. The Prisma client is generated to `src/generated/prisma/` (not the default location).

Two models (SQLite):
- `User`: id, email, password, timestamps
- `Project`: id, name, userId (nullable FK — null for anonymous), messages (JSON string), data (JSON string — file system snapshot), timestamps

Run migrations with `npx prisma migrate dev`.
