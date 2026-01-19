# Repository Guidelines

## Project Structure & Module Organization
- `apps/web/` is the primary Next.js app (App Router). Routes live in `apps/web/app/`, API handlers in `apps/web/app/api/`, UI in `apps/web/components/`, hooks in `apps/web/hooks/`, utilities in `apps/web/utils/`, and Prisma schema in `apps/web/prisma/`.
- `packages/` contains shared libraries (e.g., `@inboxzero/*`).
- `docs/` hosts deployment and environment guides; `docker/` has compose files and scripts; `scripts/` holds repo utilities.
- Tests are mostly colocated as `*.test.ts(x)` plus suite folders under `apps/web/__tests__/` (AI and E2E).

## Build, Test, and Development Commands
- `pnpm install` installs workspace dependencies (Node >= 22; see `.nvmrc`).
- `pnpm dev` runs the web app via Turborepo; `pnpm build` builds it.
- `pnpm lint` runs Biome checks; `pnpm check` and `pnpm fix` run Ultracite.
- `npm run setup` runs the interactive setup CLI and writes `apps/web/.env`.
- `pnpm test` runs the web test pipeline; for targeted runs use:
  - `cd apps/web && pnpm test --run` (unit tests)
  - `cd apps/web && pnpm test-ai` (AI tests)
  - `cd apps/web && pnpm test-e2e` / `pnpm test-e2e:flows` (E2E suites)
- `docker compose -f docker-compose.dev.yml up -d` starts Postgres/Redis for local dev.

## Coding Style & Naming Conventions
- TypeScript/React with 2-space indentation and double quotes (Biome).
- Use the `@/` alias for imports within `apps/web`.
- Components use PascalCase; tests follow `*.test.ts(x)` naming.

## Testing Guidelines
- Vitest is the primary runner; AI and E2E suites are opt-in via scripts.
- Add or update tests alongside behavior changes and prefer unit tests for core logic.

## Commit & Pull Request Guidelines
- Commit subjects are short and imperative; common prefixes include `feat:`, `fix:`, `docs:`, `refactor:`, `chore:`.
- PRs should explain the change, link relevant issues, and note test commands run.
- Include screenshots or short recordings for UI changes.

## Configuration & Security
- Keep secrets out of Git; use `apps/web/.env` and update `apps/web/.env.example` when adding new variables.
- See `docs/hosting/environment-variables.md` and `ARCHITECTURE.md` for deeper reference.
