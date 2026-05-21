# Dashboard Procurement Improvement Plan

Last updated: 2026-04-24

## Purpose

This document is the shared execution plan for improving the `dashboard_procurement` repository after the latest repo review.

Use this file as the source of truth before making broad backend, frontend, deployment, or documentation changes. Work should stay phase-based and verifiable. Do not jump into later phases before the current phase has passed its verification gate.

## Current Product Scope

The app is an integrated procurement and logistics dashboard for managing:

- Master data: vendors, mills, zones, products, MOTs, and UOMs.
- Contract data: dedicated fix, dedicated var, and oncall routing.
- Excel import/export flows.
- Authenticated dashboard, overview metrics, search, and admin user management.
- Vendor/mill/contract agreement history through audit logs.

Current contract records are treated as **Plan Contract** data. **Actual Contract** is a planned future expansion. The UI can expose a Plan/Actual switch, but backend/database work for actual contracts should wait until the actual contract arrangement is finalized.

## Confirmed Stack

### Frontend

- Framework: Next.js App Router.
- Language: TypeScript.
- UI/data framework: Refine.
- UI library: Ant Design.
- Charts: Recharts.
- HTTP client: Axios through `frontend/src/lib/api-client.ts`.
- Date handling: Dayjs.
- Styling: Ant Design tokens, global CSS, Tailwind config, local font files.
- Main app shell: `frontend/src/providers/refine-provider.tsx`.
- Main protected routes: `/overview`, `/vendors`, `/mills`, `/zones`, `/contracts/dedicated-fix`, `/contracts/dedicated-var`, `/contracts/oncall`, `/import`, `/admin/users`, `/explorer`.

### Backend

- Language: Go.
- Framework: Gin.
- ORM: GORM.
- Database driver: PostgreSQL.
- Auth: JWT signed with HMAC, stored in HttpOnly cookie by the backend.
- CORS: `gin-contrib/cors`.
- Excel handling: `excelize`.
- Decimal math: `shopspring/decimal`.
- Config loading: environment variables plus `godotenv` for local development.
- API root: `/api/v1`.
- Backend module root: `backend/`.

### Database

- Primary database: PostgreSQL.
- Production target: Neon PostgreSQL.
- Local target: PostgreSQL service in Docker Compose or local PostgreSQL.
- Runtime schema is currently driven by GORM `AutoMigrate`.
- `backend/migrations/001_initial_schema.sql` exists as reference documentation but is stale and must not be treated as authoritative until updated.

### Deployment and Local Infra

- Production frontend target: Vercel.
- Production backend target: Railway.
- Production database target: Neon.
- Local compose file: `docker-compose.yml`.
- Known issue: local Compose references `frontend/Dockerfile`, but that file is missing.

## Execution Rules

1. Read this `plan.md` before starting any new phase.
2. Keep changes scoped to the active phase unless the user explicitly expands the scope.
3. Preserve existing frontend style: Ant Design tokens, compact dashboard layout, Refine resource patterns, and shared API client.
4. Preserve cookie-based auth: frontend uses `withCredentials`, backend sets HttpOnly cookie.
5. Do not expose secrets in chat or docs.
6. Exclude `wipe_data.go` from review-driven planning unless the user explicitly asks to include it.
7. After each phase, run the verification gate listed for that phase and record any blocker before moving on.
8. On Windows, prefer `cmd /c npm ...` for frontend commands and repo-local `GOCACHE` for Go validation.

## Execution Batches

The 10 main phases are executed in a 3-3-4 sequence. Phase 0 is a safety gate before the first batch and must run before implementation.

- Batch 1: Phase 0 gate, then Phase 1, Phase 2, and Phase 3.
- Batch 2: Phase 4, Phase 5, and Phase 6.
- Batch 3: Phase 7, Phase 8, Phase 9, and Phase 10.

Current active scope: **Batch 3 completed; all planned batches are implemented**.

Status on 2026-04-24:

- Batch 1 implementation is complete.
- Batch 1 code/config verification gates passed.
- `docker compose up --build` still requires a real local `.env`; `.env.example` intentionally contains rejected placeholder secrets.
- Batch 2 implementation is complete.
- Batch 2 automated verification gates passed.
- Manual upload/re-import, rollback, and missing-ID CRUD checks still require a running local database/API session.
- Batch 3 implementation is complete.
- Batch 3 automated verification gates passed.
- Manual audit-history checks, new master-data page browser checks, and full local DB/API workflow checks still require a running local database/API session.
- Frontend test runner selection remains deferred; current frontend verification is typecheck, lint, and production build.

## Phase 0: Baseline and Safety

Status: Completed on 2026-04-24.

### Objective

Make the current repo state explicit before implementing fixes.

### Tasks

- Confirm repo root with `git rev-parse --show-toplevel`.
- Check dirty files with `git status --short`.
- Confirm current env templates:
  - `backend/.env.example`
  - `frontend/.env.example`
- Confirm local secrets are ignored:
  - `backend/.env`
  - upload directories
  - frontend build artifacts
- Re-run baseline validation.

### Verification Gate

Run:

```powershell
git status --short
cd backend
$env:GOCACHE=(Join-Path (Get-Location) ".gocache")
$env:GOTELEMETRY="off"
go test ./...
cd ../frontend
cmd /c npx tsc --noEmit --pretty false
cmd /c npm run lint
cmd /c npm run build
```

Expected:

- Backend compiles/tests without code errors.
- Frontend typecheck and lint pass.
- If `npm run build` fails with `spawn EPERM` only inside sandbox, rerun outside sandbox before treating it as an app bug.

## Phase 1: Deployment and Local Run Reliability

Status: Completed on 2026-04-24.

### Objective

Make the documented local and deployment paths truthful and repeatable.

### Problems Addressed

- `docker-compose.yml` references a missing `frontend/Dockerfile`.
- Compose defaults use rejected placeholder secrets.
- Frontend README is still generic create-next-app text.
- Deployment docs do not fully explain current local and production requirements.

### Tasks

- Decide one local frontend strategy:
  - Add `frontend/Dockerfile`, or
  - Remove/disable the `rygell-web` Compose service and document non-Docker frontend startup.
- Fix Compose env defaults so backend can start only when required secrets are supplied.
- Update `DEPLOYMENT.md` with:
  - Docker path.
  - Non-Docker local path.
  - Required backend env values.
  - Required frontend env values.
  - Cross-origin cookie requirements.
- Replace `frontend/README.md` with project-specific run instructions.
- Add a clear local run checklist.

### Target Files

- `docker-compose.yml`
- `DEPLOYMENT.md`
- `frontend/README.md`
- `backend/.env.example`
- `frontend/.env.example`
- Optional: `frontend/Dockerfile`

### Verification Gate

Run:

```powershell
docker compose config
cd frontend
cmd /c npx tsc --noEmit --pretty false
cmd /c npm run lint
cmd /c npm run build
cd ../backend
$env:GOCACHE=(Join-Path (Get-Location) ".gocache")
$env:GOTELEMETRY="off"
go test ./...
```

If Compose is intended to run the full stack, also verify:

```powershell
docker compose up --build
```

## Phase 2: Auth and Authorization Hardening

Status: Completed on 2026-04-24.

### Objective

Remove fragile username-based admin checks and make auth behavior explicit.

### Problems Addressed

- Admin-only API access is tied to hard-coded username `admin`.
- `ADMIN_USERNAME` is configurable, but admin checks do not use it.
- There is no role model, so future admin/non-admin behavior is brittle.

### Tasks

- Choose the smallest safe authorization model:
  - Option A: compare against configured `ADMIN_USERNAME`.
  - Option B: add a role field to users and check role claims/server-side user record.
- Apply the selected model consistently to:
  - List users.
  - Create users.
  - Delete users.
  - Update user passwords.
- Keep JWT validation HMAC-safe.
- Keep HttpOnly cookie behavior.
- Add tests for admin and non-admin access behavior.

### Target Files

- `backend/internal/config/config.go`
- `backend/internal/handlers/auth_handler.go`
- `backend/internal/services/users_services.go`
- `backend/internal/models/users.go`
- `backend/internal/repositories/user_repository.go`
- New backend tests as needed.

### Verification Gate

Run:

```powershell
cd backend
$env:GOCACHE=(Join-Path (Get-Location) ".gocache")
$env:GOTELEMETRY="off"
go test ./...
go vet ./...
go build ./cmd/server
```

Expected:

- Admin endpoints reject non-admin users.
- Configured admin remains able to manage users.
- Login/logout/me flow still works through cookie auth.

## Phase 3: Database Schema and Migration Truth

Status: Completed on 2026-04-24.

### Objective

Make schema documentation and runtime schema agree.

### Problems Addressed

- `backend/migrations/001_initial_schema.sql` is stale.
- Runtime schema is mostly GORM-driven.
- Future deploy or manual DB bootstrap could drift from code.

### Tasks

- Decide whether migrations are documentation-only or executable.
- If documentation-only:
  - Rename or clearly mark the file as reference-only.
  - Update it to match current models.
- If executable:
  - Add a real migration strategy.
  - Avoid destructive schema changes without backup steps.
- Update schema for:
  - Users.
  - Vendor extended fields.
  - All current contract columns.
  - Audit logs.
  - Indexes needed for search/filter paths.

### Target Files

- `backend/migrations/001_initial_schema.sql`
- `backend/internal/models/*.go`
- `backend/internal/database/database.go`
- `DEPLOYMENT.md`

### Verification Gate

Run:

```powershell
cd backend
$env:GOCACHE=(Join-Path (Get-Location) ".gocache")
$env:GOTELEMETRY="off"
go test ./...
go build ./cmd/server
```

Manual verification:

- Start against a clean local database.
- Confirm AutoMigrate or migration path creates all required tables and columns.
- Confirm `/api/v1/health` works.

## Phase 4: Import Integrity and Contract Deduplication

Status: Completed on 2026-04-24. Automated gate passed; manual upload/re-import verification still needs a running local database/API session.

### Objective

Make Excel import safe, repeatable, and auditable.

### Problems Addressed

- Import is not transactional.
- Master data can be created before contract bulk insert fails.
- Contract dedupe helpers exist but are not used.
- Re-importing the same file can duplicate contracts.

### Tasks

- Wrap each confirm import in a database transaction.
- Decide dedupe rules per contract type:
  - SPK number only.
  - SPK plus vendor/mill.
  - File-specific import batch rules.
- Use existing `FindDedicatedFixBySPK`, `FindDedicatedVarBySPK`, and `FindOncallBySPK`, or replace with better unique lookup helpers.
- Return import result with:
  - Inserted count.
  - Skipped duplicate count.
  - Updated count if update mode is supported.
  - Row-level errors.
- Avoid partial success that silently leaves orphan master data.
- Add tests for:
  - Successful import.
  - Duplicate import.
  - Failed import rollback.

### Target Files

- `backend/internal/services/import_service.go`
- `backend/internal/repositories/master_repository.go`
- `backend/internal/repositories/contract_repository.go`
- `backend/internal/handlers/import_handler.go`
- Backend tests for import behavior.

### Verification Gate

Run:

```powershell
cd backend
$env:GOCACHE=(Join-Path (Get-Location) ".gocache")
$env:GOTELEMETRY="off"
go test ./...
```

Manual verification:

- Upload and confirm a valid Excel file.
- Re-import the same file.
- Confirm duplicates are skipped or updated according to the chosen rule.
- Confirm failed imports do not leave partial master/contract records.

## Phase 5: Contract Data Model: Plan Contract and Actual Contract

Status: Completed on 2026-04-24.

### Objective

Prepare a clear path from current Plan Contract data to future Actual Contract data.

### Current State

- Existing contract records are Plan Contract data.
- Overview should show Plan vs Actual.
- Actual Contract backend/database design is intentionally deferred until the business arrangement is finalized.

### Tasks Now

- Keep UI wording consistent:
  - Plan Contract.
  - Actual Contract.
- Keep Actual Contract controls visible but inactive until backend/data model exists.
- Avoid creating fake actual contract tables or endpoints before requirements are clear.
- Document expected future questions:
  - Is actual contract a separate table or status/type on existing contract rows?
  - Does actual contract mirror all plan fields?
  - Does actual contract require approval/import flow?
  - How should variance be calculated?
  - What source creates actual contract data?

### Future Tasks After Actual Contract Requirements Are Ready

- Add backend model or fields for actual contract data.
- Add API endpoints or filters for actual contracts.
- Add frontend routes/forms/tables for actual contract mode.
- Update overview charts to compare real plan and actual values.
- Add variance metrics and exception lists.
- Add tests for Plan/Actual switching and data separation.

### Target Files

- `frontend/src/app/overview/page.tsx`
- `frontend/src/components/contracts/ContractModeSwitch.tsx`
- `frontend/src/app/contracts/**/page.tsx`
- `frontend/src/app/contracts/**/create/page.tsx`
- `frontend/src/app/contracts/**/edit/[id]/page.tsx`
- Future backend files after requirements are confirmed.

### Verification Gate

Run:

```powershell
cd frontend
cmd /c npx tsc --noEmit --pretty false
cmd /c npm run lint
cmd /c npm run build
```

Expected:

- Plan Contract is active.
- Actual Contract is visible but disabled until implemented.
- No route or auth regression.

## Phase 6: CRUD Correctness and Data Integrity

Status: Completed on 2026-04-24. Automated gate passed; manual missing-ID CRUD verification still needs a running local database/API session.

### Objective

Make update/delete behavior predictable and safe.

### Problems Addressed

- Some repository update methods use `Save`, which can act like an upsert.
- Deletes do not consistently confirm affected rows.
- Wrong IDs can appear successful.

### Tasks

- Replace risky `Save` update paths with explicit updates after checking existence.
- Return `404` for missing records.
- Return clear validation errors for bad input.
- Preserve Refine-compatible response shapes.
- Add tests for:
  - Update existing record.
  - Update missing record.
  - Delete existing record.
  - Delete missing record.

### Target Files

- `backend/internal/repositories/master_repository.go`
- `backend/internal/repositories/contract_repository.go`
- `backend/internal/handlers/master_handler.go`
- `backend/internal/handlers/contract_handler.go`
- `backend/internal/services/master_service.go`
- `backend/internal/services/contract_service.go`

### Verification Gate

Run:

```powershell
cd backend
$env:GOCACHE=(Join-Path (Get-Location) ".gocache")
$env:GOTELEMETRY="off"
go test ./...
go vet ./...
```

Manual verification:

- Update and delete known valid records.
- Try update/delete with missing IDs.
- Confirm frontend table actions still refresh correctly.

## Phase 7: Audit Log Completeness

Status: Completed on 2026-04-25. Automated gate passed; manual audit-history workflow verification still needs a running local database/API session.

### Objective

Make audit history trustworthy.

### Problems Addressed

- Audit insert errors are ignored.
- `NewData` is not populated.
- Vendor/mill agreement endpoints log notes without confirming entity existence.

### Tasks

- Return or log audit write failures intentionally.
- Store both old and new data where relevant.
- Check vendor/mill existence before writing agreement notes.
- Decide whether audit failure should block the main operation.
- Add tests for audit log creation and retrieval.

### Target Files

- `backend/internal/models/audit_log.go`
- `backend/internal/repositories/audit_repository.go`
- `backend/internal/services/contract_service.go`
- `backend/internal/handlers/contract_handler.go`

### Verification Gate

Run:

```powershell
cd backend
$env:GOCACHE=(Join-Path (Get-Location) ".gocache")
$env:GOTELEMETRY="off"
go test ./...
```

Manual verification:

- Update a contract.
- Add an agreement note.
- Fetch audit history.
- Confirm old/new data and notes are present as expected.

## Phase 8: Missing Master Data UI

Status: Completed on 2026-04-25. Products, MOTs, and UOMs are treated as user-managed because backend CRUD already exists and contract forms depend on them.

### Objective

Expose all backend-supported master data in the frontend if users need to manage it manually.

### Problems Addressed

- Backend exposes products, MOTs, and UOMs.
- Contract forms use products/MOTs/UOMs.
- Frontend navigation currently does not provide first-class management pages for all of them.

### Tasks

- Decide if products, MOTs, and UOMs should be user-managed or import-only.
- If user-managed:
  - Add Refine resources.
  - Add list/create/edit pages.
  - Add navigation entries.
  - Match existing vendors/mills/zones UI pattern.
- If import-only:
  - Document this clearly.
  - Keep them out of navigation intentionally.

### Target Files

- `frontend/src/providers/refine-provider.tsx`
- `frontend/src/components/layout/sider.tsx`
- New pages under:
  - `frontend/src/app/products`
  - `frontend/src/app/mots`
  - `frontend/src/app/uoms`

### Verification Gate

Run:

```powershell
cd frontend
cmd /c npx tsc --noEmit --pretty false
cmd /c npm run lint
cmd /c npm run build
```

Manual verification:

- Navigate to each added page.
- Create, edit, search, and delete records if enabled.
- Confirm contract forms can select newly created records.

## Phase 9: Test Coverage Expansion

Status: Completed on 2026-04-25 for backend high-risk coverage. Frontend test runner selection remains deferred per the current verification gate.

### Objective

Add practical tests for the highest-risk backend and frontend flows.

### Current State

- Backend `go test ./...` passes but reports no test files.
- Frontend typecheck/lint/build passes, but there is no visible user-flow test coverage.

### Backend Test Targets

- Auth:
  - Login.
  - Logout.
  - `/auth/me`.
  - Admin-only user management.
- Contract CRUD:
  - List with pagination.
  - List with `q` search.
  - Create/update/delete.
- Import:
  - Valid import.
  - Duplicate import.
  - Failed import rollback.
- Audit:
  - Contract update audit.
  - Agreement note audit.

### Frontend Test Targets

- Login redirect flow.
- Overview renders after auth.
- Contract list search and pagination.
- Plan/Actual switch rendering.
- Import wizard happy path UI.
- Admin users page behavior.

### Verification Gate

Run:

```powershell
cd backend
$env:GOCACHE=(Join-Path (Get-Location) ".gocache")
$env:GOTELEMETRY="off"
go test ./...
cd ../frontend
cmd /c npx tsc --noEmit --pretty false
cmd /c npm run lint
cmd /c npm run build
```

Add frontend test commands after a test runner is selected.

## Phase 10: Dependency and Repo Hygiene

Status: Completed on 2026-04-25. Automated gate passed; `npm install` still reports existing peer warnings from Refine Kbar and 9 npm audit findings that were not force-fixed.

### Objective

Reduce avoidable maintenance and security risk.

### Problems Addressed

- Some frontend dependencies are used transitively instead of declared directly.
- React runtime and React type versions should be aligned.
- Tracked cookie files should be removed if not intentionally needed.
- Build artifacts and upload files should remain ignored.

### Tasks

- Add direct dependencies for directly imported packages.
- Align React type packages with React runtime version.
- Remove tracked runtime artifacts if not required:
  - `cookie.txt`
  - `cookie2.txt`
- Confirm ignored paths:
  - `backend/.env`
  - `backend/uploads/`
  - `backend/cmd/server/uploads/`
  - `backend/.gocache/`
  - `frontend/.next/`
  - `frontend/node_modules/`
  - `frontend/tsconfig.tsbuildinfo`
  - `frontend/next-env.d.ts`

### Target Files

- `frontend/package.json`
- `frontend/package-lock.json`
- `.gitignore` files as needed.
- Tracked cookie files, if removed intentionally.

### Verification Gate

Run:

```powershell
cd frontend
cmd /c npm install
cmd /c npx tsc --noEmit --pretty false
cmd /c npm run lint
cmd /c npm run build
cd ..
git status --ignored --short
```

## Final Acceptance Checklist

The repo is considered improved when:

- Local run instructions are truthful.
- Production deploy instructions are truthful.
- Compose either works or is clearly scoped to backend/database only.
- Admin access is not hard-coded to username `admin` unless explicitly documented as the chosen rule.
- Schema reference matches runtime models or is clearly marked documentation-only.
- Import is transactional or explicitly handles partial failure safely.
- Duplicate import behavior is defined and tested.
- Update/delete endpoints return correct not-found behavior.
- Audit logs contain reliable old/new/note data.
- Products/MOTs/UOMs are either manageable in UI or intentionally documented as import-only.
- Backend has meaningful tests for auth, contracts, import, and audit.
- Frontend passes typecheck, lint, and production build.

## Known Validation Notes

- Backend validation should run from `backend/`, not repo root.
- Use repo-local Go cache on this Windows machine:

```powershell
$env:GOCACHE=(Join-Path (Get-Location) ".gocache")
$env:GOTELEMETRY="off"
```

- Frontend validation should prefer:

```powershell
cmd /c npx tsc --noEmit --pretty false
cmd /c npm run lint
cmd /c npm run build
```

- If Next build or dev fails with `spawn EPERM` only under sandbox restrictions, rerun outside sandbox before treating it as a code failure.
