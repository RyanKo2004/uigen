# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

UIGen is an AI-powered React component generator with live preview. Users describe React components in a chat interface, and the AI generates them in real-time using a virtual file system. The app supports both authenticated users (with project persistence) and anonymous sessions.

**Tech Stack**: Next.js 15 (App Router), React 19, TypeScript, Tailwind CSS v4, Prisma (SQLite), Anthropic Claude AI (via Vercel AI SDK), Monaco Editor

## Common Commands

### Development
```bash
npm run dev              # Start dev server with Turbopack
npm run dev:daemon       # Start dev server in background (logs to logs.txt)
npm run build            # Build for production
npm run start            # Start production server
npm run lint             # Run ESLint
```

### Testing
```bash
npm test                 # Run all tests with Vitest
```

### Database
```bash
npm run setup            # Install deps + generate Prisma client + run migrations
npm run db:reset         # Reset database (destructive)
npx prisma generate      # Generate Prisma client after schema changes
npx prisma migrate dev   # Create and apply new migration
```

## Architecture

### Virtual File System

The core innovation is a **VirtualFileSystem** (`src/lib/file-system.ts`) that operates entirely in memory - no files are written to disk. This enables:

- Real-time component creation and editing without I/O overhead
- Seamless undo/redo through serialization
- Project persistence via JSON storage in the database

The VFS is managed through React Context (`src/lib/contexts/file-system-context.tsx`) and synchronized with the AI chat system.

### AI Integration Flow

1. **API Route** (`src/app/api/chat/route.ts`): Receives chat messages + serialized file system
2. **System Prompt** (`src/lib/prompts/generation.tsx`): Instructs AI to create React components with `@/` import alias
3. **AI Tools**: The AI has two tools to manipulate the file system:
   - `str_replace_editor` (`src/lib/tools/str-replace.ts`): Create files, view files, search/replace content, insert lines
   - `file_manager` (`src/lib/tools/file-manager.ts`): Rename/move files and delete files
4. **Tool Call Handling**: Client-side context (`file-system-context.tsx`) intercepts tool calls and updates the VFS
5. **Project Persistence**: On completion, messages and VFS are saved to database (authenticated users only)

### Preview System

The preview system (`src/components/preview/PreviewFrame.tsx`) transforms and renders components in an iframe:

1. **JSX Transformation** (`src/lib/transform/jsx-transformer.ts`): Uses Babel standalone to transpile JSX/TSX to ES modules
2. **Import Map Generation**: Creates ES module import map with:
   - React/ReactDOM from esm.sh CDN
   - Local files as blob URLs
   - `@/` alias support for local imports
   - Third-party packages from esm.sh
3. **Live HTML Generation**: Builds complete HTML with import map, Tailwind CDN, error boundaries, and syntax error display
4. **Hot Reload**: Automatically updates when VFS changes (via `refreshTrigger`)

**Important**: Entry point is `/App.jsx` (or `/App.tsx`) which must export a React component as default. No HTML files are used.

### Authentication & Projects

- **Auth System** (`src/lib/auth.ts`): JWT-based authentication with bcrypt password hashing
- **Database Schema** (`prisma/schema.prisma`):
  - `User`: Email/password authentication
  - `Project`: Stores serialized messages + VFS data
- **Anonymous Work**: Tracked in localStorage (`src/lib/anon-work-tracker.ts`) with prompt to sign up
- **Server Actions** (`src/actions/`): Handle project CRUD operations

### Mock Provider

If no `ANTHROPIC_API_KEY` is set, a mock AI provider (`src/lib/provider.ts`) generates static component examples. This allows the app to run without API credentials but returns pre-defined Counter/Form/Card components instead of true AI generation.

## Development Guidelines

### File System Operations

- Always use the VFS through `useFileSystem()` hook - never bypass it
- All imports for local files must use `@/` alias (e.g., `import Counter from '@/components/Counter'`)
- Entry point must be `/App.jsx` or `/App.tsx` with a default export

### Testing

- Tests use Vitest with React Testing Library
- Coverage includes: VFS operations, JSX transformation, context providers, UI components
- Run tests frequently during development

### Adding AI Capabilities

When modifying AI behavior:
1. Update system prompt in `src/lib/prompts/generation.tsx`
2. Tool schemas are defined using Zod in `src/lib/tools/`
3. Test with both real API and mock provider
4. Update `maxSteps` in `src/app/api/chat/route.ts` if needed (current: 40 steps, 10k max tokens)

### Preview System Debugging

- Check browser console in iframe for import errors
- Syntax errors are displayed prominently in preview pane
- Import map is logged to console on load failure
- CSS imports are extracted and injected as `<style>` tags
