# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run dev          # Start dev server with Turbopack
npm run build        # Production build
npm run lint         # ESLint via next lint
npm run test         # Run tests with Vitest
npm run setup        # First-time setup: install, Prisma generate + migrate
npm run db:reset     # Reset database (destructive)
```

Both `dev` and `build` require `NODE_OPTIONS='--require ./node-compat.cjs'` (already embedded in the npm scripts).

Run a single test file: `npx vitest run src/components/chat/__tests__/MessageList.test.tsx`

## Architecture

UIGen is an AI-powered React component generator with live preview. Users describe components in chat; Claude generates code using tool calls; the result renders live in an iframe.

### Request flow

1. User submits a prompt in `MessageInput` → posted to `POST /api/chat` with serialized virtual FS and optional `projectId`
2. `/api/chat/route.ts` reconstructs the VirtualFileSystem from JSON, then calls Claude with two tools: `str_replace_editor` and `file_manager`
3. The Vercel AI SDK's `onToolCall` callback fires **mid-stream** for each tool call — `FileSystemContext.handleToolCall()` applies changes to the in-memory FS immediately, so the preview updates while Claude is still generating
4. `PreviewFrame` watches a `refreshTrigger` counter; each FS change increments it, causing the frame to re-run `jsx-transformer.ts` and inject a new `srcdoc` into the iframe
5. On `onFinish`, `/api/chat` saves the full FS + message history to the database if a `projectId` and authenticated user exist

### Virtual file system

`/lib/file-system.ts` — A `VirtualFileSystem` class backed by a `Map<string, FileNode>`. No disk I/O during generation. Key details:
- `serialize()` / `deserializeFromNodes()` convert to/from a flat JSON object for DB storage; deserialization sorts paths so parents are always created before children
- `viewFile(path, [startLine, endLine])` supports range viewing (used by the `view` command of `str_replace_editor`)
- All imports in generated code should use the `@/` alias (e.g. `@/components/Button`), which the transformer resolves to blob URLs

### JSX transformer & iframe preview

`/lib/transform/jsx-transformer.ts` converts the virtual FS into a single self-contained HTML document:
- **Babel** (`@babel/standalone`) transpiles JSX/TSX to JS in-browser
- **Import map**: each file becomes a `Blob` → `URL.createObjectURL()` entry; relative and `@/` imports are mapped to those blob URLs; bare specifiers (e.g. `react`, `lucide-react`) are resolved to `https://esm.sh/`
- Missing imports get empty placeholder modules (prevents hard crashes)
- Runtime errors are caught by an injected React `ErrorBoundary`
- Syntax/transform errors are displayed in a formatted error box inside the iframe

### AI tools

`/lib/tools/str-replace.ts` — `str_replace_editor` with commands: `view`, `create`, `str_replace`, `insert`, `undo_edit`. Errors are returned as strings so Claude can self-correct.

`/lib/tools/file-manager.ts` — `file_manager` with commands: `rename`, `delete`.

### State management

- `ChatProvider` (`/lib/contexts/chat-context.tsx`) — wraps Vercel's `useChat` hook; serializes FS into each request; delegates tool execution to `FileSystemContext`
- `FileSystemProvider` (`/lib/contexts/file-system-context.tsx`) — owns the VirtualFileSystem instance; exposes `handleToolCall`; auto-selects `/App.jsx` (or first root file) when no file is selected

### Provider & mock mode

`/lib/provider.ts` returns the real `claude-sonnet-4-5` model when `ANTHROPIC_API_KEY` is set; otherwise returns a `MockLanguageModel` with hardcoded component templates. `/api/chat` sets `maxSteps: 40` for real Claude and `maxSteps: 4` for the mock to avoid infinite loops.

Set `ANTHROPIC_API_KEY` in `.env` to use real Claude.

### Auth & persistence

- JWT in HTTP-only cookies (7-day expiry); helpers in `/lib/auth.ts` using `jose`
- Server actions in `/actions/` handle sign-up/in/out and project CRUD
- Prisma schema (`/prisma/schema.prisma`):
  ```prisma
  model User {
    id        String    @id @default(cuid())
    email     String    @unique
    password  String
    createdAt DateTime  @default(now())
    updatedAt DateTime  @updatedAt
    projects  Project[]
  }

  model Project {
    id        String   @id @default(cuid())
    name      String
    userId    String?          // nullable — anonymous projects have no owner
    messages  String   @default("[]")   // JSON: serialized chat history
    data      String   @default("{}")   // JSON: serialized VirtualFileSystem
    createdAt DateTime @default(now())
    updatedAt DateTime @updatedAt
    user      User?    @relation(fields: [userId], references: [id], onDelete: Cascade)
  }
  ```
  `Project.userId` is nullable so anonymous users can have projects; `messages` and `data` are JSON strings (not native JSON columns, SQLite limitation)
- Middleware (`/middleware.ts`) protects `/api/projects` and `/api/filesystem`
- Anonymous users: `anon-work-tracker.ts` persists work in `sessionStorage`; on sign-in, `useAuth` converts it to a real project via `handlePostSignIn`

## Code style

Only add comments to genuinely complex or non-obvious logic. Omit comments on self-evident code.

### Path alias

`@/*` maps to `./src/*` (configured in `tsconfig.json`).
