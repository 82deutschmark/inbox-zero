# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Inbox Zero is an AI-powered email assistant built as a Next.js 16 monorepo using Turborepo. It helps users reach inbox zero by organizing emails, pre-drafting replies, tracking follow-ups, and automating email management.

**Tech Stack:** Next.js 16 (App Router), Prisma (PostgreSQL), Upstash Redis, Tailwind CSS, shadcn/ui, Vercel AI SDK, Better Auth

**Repository Structure:**
- `apps/web/` - Main Next.js application (production)
- `apps/unsubscriber/` - Fastify/Playwright service for browser automation (not in production use)
- `packages/` - Shared packages (tinybird, loops, resend, tinybird-ai-analytics, tsconfig)

## Essential Commands

**Monorepo (run from root):**
- Setup environment: `npm run setup` (interactive CLI that generates `.env` file)
- Install dependencies: `pnpm install`
- Run all checks: `pnpm check`
- Fix formatting: `pnpm fix` (uses Ultracite which wraps Biome)

**Development (run from root):**
- Dev server: `pnpm dev` (runs `apps/web` only)
- Build: `pnpm build` (builds `apps/web`)
- Lint: `pnpm lint`

**Testing (run from `apps/web`):**
- All unit tests: `cd apps/web && pnpm test --run`
- Single test: `cd apps/web && pnpm test path/to/test.test.ts --run`
- AI tests (uses real LLM): `cd apps/web && pnpm test-ai`
- E2E tests: `cd apps/web && pnpm test-e2e`
- E2E flow tests: `cd apps/web && pnpm test-e2e:flows`

**Database (run from `apps/web`):**
- Migrate: `cd apps/web && pnpm prisma migrate dev`
- Deploy migrations: `cd apps/web && pnpm prisma migrate deploy`
- Studio: `cd apps/web && pnpm prisma studio`
- Generate client: `cd apps/web && pnpm prisma generate`

**Local Docker:**
- Dev databases: `docker compose -f docker-compose.dev.yml up -d` (PostgreSQL + Redis)
- Full stack: `NEXT_PUBLIC_BASE_URL=http://localhost:3000 docker compose --profile all up -d`

## Architecture Patterns

### Authentication & Authorization

All API routes handling user data MUST use appropriate middleware:
- `withEmailAccount` - For email-scoped operations (provides `emailAccountId`, `userId`, `email`)
- `withAuth` - For user-scoped operations (provides `userId`)
- `withEmailProvider` - Like `withEmailAccount` but also provides `emailProvider` instance
- `withError` - For public endpoints or custom auth (webhooks, cron jobs)

**Critical:** All database queries MUST be scoped to the authenticated user's data:

```typescript
// ✅ Correct - scoped to user's account
const rule = await prisma.rule.findUnique({
  where: { id: ruleId, emailAccount: { id: emailAccountId } }
});

// ❌ Wrong - missing user scoping (security vulnerability)
const rule = await prisma.rule.findUnique({
  where: { id: ruleId }
});
```

**Cron endpoints:** Must validate `CRON_SECRET` using `hasCronSecret(request)` or `hasPostCronSecret(request)` to prevent unauthorized execution.

### Data Flow Pattern

1. **GET API Route** - Fetch data, always wrapped with auth middleware
2. **Server Action** - Mutations (create/update/delete) using `next-safe-action`
3. **Data Fetching** - Client uses SWR hooks
4. **Form Handling** - React Hook Form + Zod validation + `useAction` hook

See `apps/web/CLAUDE.md` for detailed fullstack workflow examples.

### AI Implementation

AI features use the Vercel AI SDK with multiple LLM providers (OpenAI, Anthropic, Google, Bedrock, Groq, OpenRouter, Ollama).

**Key files:**
- `apps/web/utils/ai/` - AI rule processing, categorization, action generation
- `apps/web/utils/llms/` - Core LLM utilities and model configurations
- `apps/web/__tests__/` - AI tests (not run by default, use real LLMs)

**Standard pattern:**
```typescript
import { createGenerateObject } from "@/utils/llms";
import { getModel } from "@/utils/llms/model";

const modelOptions = getModel(emailAccount.user);
const generateObject = createGenerateObject({
  userEmail: emailAccount.email,
  label: "Feature Name",
  modelOptions,
});

const result = await generateObject({
  ...modelOptions,
  system: "System prompt defining role",
  prompt: "User prompt with data",
  schema: zodSchema,
});
```

**Best practices:**
- Always use Zod schemas for response validation
- Use scoped loggers with `createScopedLogger`
- Include `emailAccount.about` for user context
- Format data with XML-like tags in prompts
- Add retry logic with `withRetry` for transient failures

### Gmail API Integration

**Never call Gmail API directly.** Always use wrapper functions from `apps/web/utils/gmail/`:
- `apps/web/utils/gmail/message.ts` - Message operations
- `apps/web/utils/gmail/thread.ts` - Thread operations
- `apps/web/utils/gmail/label.ts` - Label operations

This abstraction enables future support for additional providers (Outlook, ProtonMail).

### Logging

Use request-scoped loggers provided by middleware:

```typescript
export const POST = withEmailAccount("route-name", async (request) => {
  const logger = request.logger; // Already has userId, emailAccountId, email
  logger.info("Processing request");

  // Enrich with additional context
  logger = logger.with({ messageId: body.messageId });

  // Pass to helper functions
  await processEmail(emailId, logger);
});
```

Server actions receive logger through context:
```typescript
export const myAction = actionClient
  .metadata({ name: "myAction" })
  .action(async ({ ctx: { logger, emailAccountId } }) => {
    logger.info("Action executed");
  });
```

Only use `createScopedLogger` for standalone scripts/tests outside middleware chains.

### State Management

- **Global state:** Jotai atoms in `apps/web/store/`
- **Queue management:** AI, archive, categorization queues in `apps/web/store/`
- **Client data fetching:** SWR hooks in `apps/web/hooks/`

### Security

See `.cursor/rules/security.mdc` for comprehensive security guidelines.

**Critical requirements:**
- All API routes must use auth middleware
- All queries must filter by user/account ownership
- Use `SafeError` for user-facing errors to avoid information disclosure
- Validate all inputs with Zod schemas
- Never expose sensitive data in responses or logs

## AI Personal Assistant Architecture

The AI assistant converts user prompt files to database rules with two-way sync:

**Why database rules?**
- AI only decides if conditions match (not generating arbitrary actions)
- Track individual rule execution frequency
- Precise action definitions without LLM interference

**Trade-offs:**
- Two-way sync complexity between prompt file and DB rules
- Prompt file isn't source of truth for LLM (some info like style guidelines in `about` field)
- This evolved architecture may be restructured in the future

**Features built on AI assistant:**
- **Reply Tracking:** Special rule type integrated with assistant (enables global updates like cold email blocker)
- **Cold Email Blocker:** Separate from assistant, runs LLM check for first-time senders

## Project-Specific Guidelines

**File organization:**
- Pages: `apps/web/app/(app)/PAGE_NAME/page.tsx`
- API routes: `apps/web/app/api/`
- Server actions: `apps/web/utils/actions/`
- Components: `apps/web/components/` (UI components in `components/ui/`)
- Utils: `apps/web/utils/`
- Test files: Co-located next to source (only E2E and AI tests in `__tests__/`)

**Code style:**
- Path imports: Use `@/` alias
- Components: PascalCase, functional with hooks
- Helper functions: Place at bottom of files
- No mid-file dynamic imports
- Infer types from Zod schemas with `z.infer`
- Balance DRY vs WET: Avoid premature abstraction (duplicating 2-3 times is fine)
- Self-documenting code: Only comment "why" not "what"
- Don't export types used only within same file
- No import/export re-export patterns

**Component patterns:**
- Use shadcn/ui components when available
- Mobile-first responsive design
- Use `LoadingContent` component for async data with loading/error states

**Database (Prisma):**
- Import: `import prisma from "@/utils/prisma";`
- Schema: `apps/web/prisma/schema.prisma`
- Always scope queries to user's data

**Environment variables:**
- Add to: `.env.example`, `apps/web/env.ts`, and `turbo.json`
- Client vars: Prefix with `NEXT_PUBLIC_`
- See `docs/hosting/environment-variables.md` for comprehensive list

**Testing:**
- Use Vitest, run with `pnpm test --run` (not `npx vitest`)
- Mock server-only: `vi.mock("server-only", () => ({}))`
- Mock Prisma: `vi.mock("@/utils/prisma")` and import from `@/utils/__mocks__/prisma`
- Helpers: `import { getEmail, getEmailAccount, getRule } from "@/__tests__/helpers"`
- Never mock the Logger

## External Services

**Required:**
- PostgreSQL (via Prisma)
- Redis (Upstash or self-hosted)
- Google Cloud (Gmail API, People API, Pub/Sub for webhooks)
- At least one LLM provider (OpenAI, Anthropic, Google, etc.)

**Optional:**
- Tinybird (analytics)
- Lemon Squeezy or Stripe (payments)
- PostHog (feature flags, analytics)
- Resend (transactional emails)
- Sentry (error tracking)
- Microsoft OAuth (Outlook support)

## Common Workflows

**Adding a new feature:**
1. Create GET API route with `withEmailAccount` and type-safe response
2. Create validation schema in `utils/actions/*.validation.ts`
3. Create server action in `utils/actions/*.ts`
4. Create SWR hook for data fetching
5. Build form component with React Hook Form + Zod
6. Add tests

**Database changes:**
1. Update `apps/web/prisma/schema.prisma`
2. Run `cd apps/web && pnpm prisma migrate dev --name description`
3. Commit migration files
4. Update affected queries and types

**Adding environment variables:**
1. Add to `apps/web/.env.example`
2. Add validation to `apps/web/env.ts`
3. Add to `turbo.json` env array
4. Document in `docs/hosting/environment-variables.md`

## Additional Resources

- Full architecture: `ARCHITECTURE.md`
- Detailed rules: `.cursor/rules/` (27 rule files covering specific patterns)
- App-specific guide: `apps/web/CLAUDE.md`
- Hosting guides: `docs/hosting/`
- Setup video: https://youtu.be/hVQENQ4WT2Y
