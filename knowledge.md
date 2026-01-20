# Project knowledge

Inbox Zero is an AI-powered email assistant (Next.js 16 monorepo with Turborepo). Helps users reach inbox zero by organizing emails, pre-drafting replies, tracking follow-ups, and automating email management.

## Quickstart
- Setup: `pnpm install && npm run setup` (interactive CLI generates `.env`)
- Dev databases: `docker compose -f docker-compose.dev.yml up -d`
- Dev server: `pnpm dev`
- Build: `pnpm build`
- Test: `cd apps/web && pnpm test --run`
- Lint/format: `pnpm check` / `pnpm fix` (uses Ultracite/Biome)
- DB migrate: `cd apps/web && pnpm prisma migrate dev`

## Architecture
- **apps/web/** - Main Next.js app (App Router) - production
- **apps/unsubscriber/** - Fastify/Playwright service (not in production)
- **packages/** - Shared packages (tinybird, loops, resend, tinybird-ai-analytics, tsconfig)
- **prisma/** - Database schema and migrations (PostgreSQL)

Key directories in apps/web:
- `app/` - Pages, layouts, API routes
- `components/` - React components (UI in `components/ui/`)
- `utils/actions/` - Server Actions (next-safe-action)
- `utils/ai/` - AI rule processing, categorization
- `utils/gmail/` - Gmail API wrappers (never call Gmail API directly)
- `store/` - Jotai atoms and queues

## Conventions
- **Auth middleware:** Use `withEmailAccount`, `withAuth`, `withEmailProvider`, or `withError` for API routes
- **DB queries:** Always scope to user's data (security requirement)
- **Path imports:** Use `@/` alias
- **Components:** PascalCase, functional with hooks, use shadcn/ui
- **Forms:** React Hook Form + Zod validation + `useAction` hook
- **Data fetching:** SWR hooks
- **Logging:** Use request-scoped logger from middleware, not `createScopedLogger`
- **Tests:** Vitest, co-located with source (E2E/AI tests in `__tests__/`)
- **Environment vars:** Add to `.env.example`, `apps/web/env.ts`, and `turbo.json`

## Tech Stack
Next.js 16, Prisma (PostgreSQL), Upstash Redis, Tailwind CSS, shadcn/ui, Vercel AI SDK, Better Auth

## Things to avoid
- Calling Gmail API directly (use utils/gmail wrappers)
- DB queries without user/account scoping
- Using `createScopedLogger` in middleware chains (use `request.logger`)
- Casting to `any` type
- Premature abstraction (2-3 duplications is fine)
