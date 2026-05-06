# Repository Guidelines

## Project Overview

This repository is a Next.js App Router attendance system for a seminar/event. Public users submit identity data and a drawn signature. Admin users authenticate with Google via NextAuth, review attendance records, delete rows, and export Excel data with embedded signatures.

Primary stack:
- Next.js 16 + React 19 + TypeScript
- PostgreSQL via Prisma Client and `@prisma/adapter-pg`
- NextAuth Google provider for admin login
- ExcelJS for attendance export
- Docker production runtime on Node 22 Alpine

## Architecture & Data Flow

### Public attendance flow

1. `src/app/page.tsx` renders public attendance UI.
2. `src/components/AttendanceForm.tsx` collects identity fields and signature data.
3. `src/components/SignaturePad.tsx` captures base64 PNG signature.
4. Form submits JSON to `POST /api/attendance` in `src/app/api/attendance/route.ts`.
5. Route validates input, rate-limits by IP, checks duplicate `nipNrp`, then writes via shared Prisma client.
6. Success UI lives in `src/app/success/page.tsx`.

### Admin flow

1. `src/app/admin/page.tsx` calls `getAdminSession()` from `src/lib/auth.ts`.
2. Unauthenticated admins go to `src/app/admin/login/page.tsx`.
3. `src/components/AdminTable.tsx` fetches paginated/searchable records from `GET /api/attendance` every 15 seconds.
4. Delete actions call `DELETE /api/attendance?id=...`.
5. Export link calls `src/app/api/attendance/export/route.ts`, which reads attendance rows and generates Excel with embedded signature images.

### Shared infrastructure patterns

- `src/lib/prisma.ts`: singleton Prisma client. Import this instead of constructing new clients.
- `src/lib/auth.ts`: NextAuth options, Google provider, `ADMIN_EMAILS` allowlist, `getAdminSession()` helper.
- `src/lib/rate-limit.ts`: in-memory IP rate limiter. Single-instance only; replace with Redis or another shared store before multi-instance deployment.
- `prisma/schema.prisma`: canonical database model definition.

## Key Directories

- `src/app/`: Next.js App Router pages, layouts, and route handlers.
- `src/app/api/attendance/`: attendance CRUD/export API routes.
- `src/app/api/auth/[...nextauth]/`: NextAuth route handler.
- `src/components/`: interactive client components (`AttendanceForm`, `SignaturePad`, `AdminTable`, `Providers`).
- `src/lib/`: shared server utilities for auth, Prisma, and rate limiting.
- `prisma/`: Prisma schema and migration location configured by `prisma.config.ts`.
- Root config files: `package.json`, `tsconfig.json`, `next.config.ts`, `eslint.config.mjs`, `Dockerfile`, `docker-compose.yml`.

## Development Commands

Use npm. `package-lock.json` is present and Docker uses `npm ci`.

```bash
npm ci
npm run dev      # next dev --port 3456
npm run build    # next build
npm run start    # next start
npm run lint     # eslint
npx prisma generate
```

Notes:
- README still shows default Next.js `localhost:3000` guidance, but observed dev port is `3456`.
- Docker runtime also uses port `3456`.
- No project-level `test`, `coverage`, or `e2e` script is currently configured.

## Runtime & Tooling Preferences

- Runtime: Node 22 in Docker (`node:22-alpine`).
- Package manager: npm; do not introduce Bun/Yarn/pnpm without an explicit migration.
- TypeScript is strict (`strict: true`) with `moduleResolution: "bundler"`.
- Path alias: `@/*` maps to `./src/*`.
- Next config uses `output: "standalone"` and keeps Prisma/Postgres packages external through `serverExternalPackages`.
- ESLint extends Next `core-web-vitals` and TypeScript rules.
- Prisma config loads `DATABASE_URL` through `dotenv/config`.

Observed environment variables:
- App/auth/database: `DATABASE_URL`, `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`, `ADMIN_EMAILS`
- Runtime: `NODE_ENV`, `NEXT_TELEMETRY_DISABLED`, `PORT`, `HOSTNAME`
- Docker Postgres: `POSTGRES_DB`, `POSTGRES_USER`, `POSTGRES_PASSWORD`

`docker-compose.yml` expects `.env.production` for the app service. No env example file was observed.

## Code Conventions & Common Patterns

### Next.js boundaries

- Server pages and API route handlers live under `src/app`.
- Interactive React components must start with `'use client'`.
- API handlers return `NextResponse.json(...)` for JSON responses.
- Use `@/` imports for source modules.

### Data access

- Use shared `prisma` from `src/lib/prisma.ts`.
- Prefer Prisma Client queries; no raw SQL pattern was observed.
- Keep database shape aligned with `prisma/schema.prisma`.

### Validation and errors

- Attendance POST validation is inline in `src/app/api/attendance/route.ts`:
  - required string checks
  - max field lengths
  - enum validation
  - signature size limit (`500000` chars)
  - duplicate `nipNrp` check
- Route handlers use `try/catch`, log server errors with `console.error`, and return Indonesian user-facing error JSON.
- Client components use local loading/error state; admin delete failures currently surface with `alert`/console logging.

### Async/state patterns

- Client state uses React local state and `fetch`; no dedicated state library observed.
- `AdminTable` polls attendance data every 15 seconds.
- Rate limiting is process-local. Do not depend on it for distributed deployments.

### Naming and UI

- Component names use PascalCase (`AttendanceForm`, `SignaturePad`, `AdminTable`).
- Route files follow App Router conventions: `page.tsx`, `layout.tsx`, `route.ts`.
- User-facing text is mostly Indonesian; keep new UI/API messages consistent unless changing product language intentionally.

## Important Files

- `src/app/page.tsx`: public attendance entry point.
- `src/components/AttendanceForm.tsx`: public form behavior and submit flow.
- `src/components/SignaturePad.tsx`: signature capture.
- `src/app/api/attendance/route.ts`: attendance create/list/delete API.
- `src/app/api/attendance/export/route.ts`: Excel export.
- `src/app/admin/page.tsx`: admin access gate and table page.
- `src/components/AdminTable.tsx`: admin table, polling, search, pagination, delete, export link.
- `src/lib/auth.ts`: NextAuth config and admin allowlist.
- `src/lib/prisma.ts`: Prisma singleton.
- `src/lib/rate-limit.ts`: process-local rate limiting.
- `prisma/schema.prisma`: database schema.
- `next.config.ts`: standalone Docker build settings.
- `Dockerfile`: production image build/run behavior.
- `docker-compose.yml`: app + Postgres local/container deployment.

## Testing & QA

No test framework, test files, fixtures, coverage config, or e2e setup was observed. `@playwright/test` appears only as a transitive optional dependency in `package-lock.json`; do not treat Playwright as configured.

Required practical checks for AI changes:
- Run `npm run lint` for any non-trivial code change.
- Run `npm run build` when changing Next.js routing, server/client boundaries, Prisma usage, or Docker/runtime-sensitive code.
- Add focused tests or another reproducible verification path when fixing bugs; current repo has no automated safety net.
- For form/API changes, verify both public submit flow and admin list/export behavior.

## Deployment Notes

Docker build flow:

```bash
npm ci
npx prisma generate
npm run build
node server.js
```

Compose provisions PostgreSQL 16 Alpine and builds the app image from `Dockerfile`. Host ports observed:
- app: `3456`
- postgres: `5433 -> 5432`

Cautions:
- `docker-compose.yml` contains concrete Postgres credentials; do not reuse them for shared or production environments.
- No migration execution command was observed in Dockerfile or compose.
- In-memory rate limiting resets on restart and does not coordinate across replicas.
