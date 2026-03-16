# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Code Style

- Use comments sparingly. Only comment complex or non-obvious code. Do not comment self-explanatory code.

## Commands

```bash
# Initial setup (install deps, generate Prisma client, run migrations)
npm run setup

# Development server (with Turbopack)
npm run dev

# Build for production
npm run build

# Lint
npm run lint

# Run all tests
npm run test

# Reset database
npm run db:reset
```

To run a single test file with vitest:
```bash
npx vitest run src/lib/__tests__/<file>.test.ts
```

## Environment

Copy `.env` and set `ANTHROPIC_API_KEY`. If absent, the app uses a `MockLanguageModel` that simulates component generation without hitting the API.

## Architecture

UIGen is an AI-powered React component generator. Users describe components in a chat interface; Claude generates/modifies code in a virtual file system; the result is rendered live in a sandboxed iframe.

### Three-pane layout

```
MainContent (src/app/main-content.tsx)
├── ChatPanel (src/components/chat/)       — 35% width
└── Preview/Code Panel (src/components/)  — 65% width
    ├── PreviewFrame (preview/) — live iframe render
    └── Editor (editor/)       — Monaco editor + file tree
```

### Data flow

1. User sends message → `ChatInterface` → `POST /api/chat`
2. API route streams Claude responses via Vercel AI SDK (`streamText`)
3. Claude calls tools (`str_replace_editor`, `file_manager`) to manipulate files
4. Tool calls dispatch to `FileSystemContext`, updating the in-memory `VirtualFileSystem`
5. `PreviewFrame` detects changes, transforms JSX via Babel standalone, re-renders iframe
6. For authenticated users, messages and file state are persisted to SQLite via Prisma

### Key abstractions

| File | Purpose |
|------|---------|
| `src/lib/file-system.ts` | `VirtualFileSystem` — in-memory FS with CRUD, serialization |
| `src/lib/contexts/file-system-context.tsx` | React context wrapping VirtualFileSystem; dispatches AI tool calls |
| `src/lib/contexts/chat-context.tsx` | Chat state via Vercel AI SDK `useChat`; handles tool invocations |
| `src/lib/provider.ts` | `LanguageModelV1` adapter — Anthropic in prod, `MockLanguageModel` in dev |
| `src/lib/tools/` | Tool definitions (`str_replace_editor`, `file_manager`) passed to `streamText` |
| `src/app/api/chat/route.ts` | Streaming chat endpoint; persists state on completion |
| `src/actions/index.ts` | Server actions for auth (signUp, signIn, signOut, getUser) |
| `src/middleware.ts` | JWT auth guard on `/api/projects` and `/api/filesystem` routes |

### Database (Prisma + SQLite)

Schema is defined in `prisma/schema.prisma` — reference it whenever you need to understand the structure of data stored in the database.

Two models: `User` (email/password auth) and `Project` (stores `messages` and `data` as JSON strings).

### Routing

- `/` — redirects authenticated users to their latest project; shows `MainContent` for anonymous users
- `/[projectId]` — protected project page, loads project from DB
- `/api/chat` — streaming AI chat endpoint
