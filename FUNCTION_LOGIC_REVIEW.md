# Dashboard Procurement - Function And Logic Review Map

This document is a repo-wide function map to make code review faster.
It focuses on runtime code (non-test files), and summarizes each function's intent and logic path.

## Reading Order

1. Backend bootstrap and routing
2. Backend middleware and handlers
3. Backend services and repositories
4. Frontend provider, auth, layout, and pages

## Backend

### `backend/cmd/server/main.go`

- `main` - loads environment, validates security-critical config (`JWT_SECRET`, admin password bootstrap rules), initializes DB and migrations, ensures admin account, builds router, starts hardened `http.Server` with timeouts.

### `backend/internal/config/config.go`

- `Load` - materializes `Config` from environment with defaults.
- `getEnv` - returns trimmed env var or fallback.
- `getEnvBool` - parses boolean env var with safe fallback.
- `getEnvInt64` - parses positive int env var with safe fallback.

### `backend/internal/database/database.go`

- `parseLogLevel` - maps text log level to GORM logger level.
- `Connect` - builds DSN from config, opens GORM connection, applies logging config.
- `Migrate` - auto-migrates all model schemas.

### `backend/internal/router/router.go`

- `Setup` - wires dependencies (repos, services, handlers), applies middleware, and registers all API routes.
- `defaultMaxBodyBytes` - resolves global request body limit.
- `defaultImportMaxBytes` - resolves upload body limit.
- `defaultImportJSONBytes` - resolves import-confirm JSON limit.
- `splitCSV` - parses CSV env strings into trimmed slices.

### `backend/internal/middleware/auth.go`

- `AuthMiddleware` - extracts JWT from bearer token or cookie, validates HMAC signature and claims, injects `user_id` and `username` into request context.

### `backend/internal/middleware/cors.go`

- `CORSMiddleware` - builds dynamic CORS config from allowed origins, supports wildcard behavior, and toggles credential allowance accordingly.

### `backend/internal/middleware/security.go`

- `RequestBodyLimit` - wraps request body with `http.MaxBytesReader` for size enforcement, with optional exempt paths.
- `AdminOnly` - blocks non-admin usernames based on request context + configured admin username.
- `StrictOriginForUnsafeMethods` - enforces exact allowlist origin/referer checks on mutating methods.
- `exactFrontendOrigins` - constructs exact origin set from config and safe localhost defaults.
- `originFromReferer` - derives origin tuple (`scheme://host`) from referer URL.
- `isUnsafeMethod` - returns true for `POST/PUT/PATCH/DELETE`.
- `adminUsername` - resolves canonical admin username with fallback.
- `asString` - safe `any` to string helper.

### `backend/internal/handlers/login_limiter.go`

- `newLoginLimiter` - creates in-memory login limiter state.
- `locked` - checks whether IP+username key is currently locked; also prunes stale entries.
- `recordFailure` - increments failure count, applies lockout window, prunes state, and enforces memory cap with oldest-entry eviction.
- `reset` - clears limiter state for a key after successful login.
- `loginKey` - normalizes `(ip, username)` into stable limiter key.
- `prune` - removes expired lockouts and old entries.
- `evictOldest` - drops oldest seen entries when map exceeds cap.

### `backend/internal/handlers/auth_handler.go`

- `NewAuthHandler` - injects user service + config + login limiter.
- `Login` - validates payload, applies lockout check, authenticates, sets HttpOnly cookie, returns user payload.
- `Logout` - clears auth cookie.
- `Me` - returns current user profile using `user_id` from middleware.
- `CreateUser` - admin-only user creation endpoint.
- `ListUsers` - admin-only user listing endpoint.
- `DeleteUser` - admin-only deletion, with guard against deleting currently logged-in admin.
- `UpdateUserPassword` - admin-only password update endpoint.
- `requireAdmin` - shared guard for admin routes in handler.
- `adminUsername` - resolves admin username from config.
- `userPayload` - normalized public user response with computed `is_admin`.
- `setAuthCookie` - writes auth cookie with configured same-site/security behavior and expiry.
- `clearAuthCookie` - removes auth cookie with matching cookie attributes.

### `backend/internal/handlers/master_handler.go`

- `paginateAndRespond` - simple-rest pagination slicer for in-memory list results + `X-Total-Count` headers.
- `respondMasterDataError` - maps not-found to 404, otherwise 500.
- `NewMasterHandler` - dependency injection.
- `GetAllMills/GetAllVendors/GetAllProducts/GetAllZones/GetAllMots/GetAllUoms` - list endpoints with optional `q` search, then paginated response shaping.
- `GetMillByID/GetVendorByID/GetProductByID/GetZoneByID/GetMotByID/GetUomByID` - ID lookup endpoints.
- `CreateMill/CreateVendor/CreateProduct/CreateZone/CreateMot/CreateUom` - create endpoints with JSON bind and service call.
- `UpdateMill/UpdateVendor/UpdateProduct/UpdateZone/UpdateMot/UpdateUom` - update endpoints by ID with payload bind and service call.
- `DeleteMill/DeleteVendor/DeleteProduct/DeleteZone/DeleteMot/DeleteUom` - delete endpoints by ID.

### `backend/internal/handlers/contract_handler.go`

- `NewContractHandler` - dependency injection.
- `parsePagination` - resolves `_start/_end` into bounded `(limit, offset)`, with safe defaults/caps.
- `respondContractDataError` - consistent 404 vs 500 mapping.
- `auditActorFromContext` - resolves trusted audit actor from authenticated context.
- `readContractUpdateBody` - bounded raw body read for update endpoints.
- `GetAllDedicatedFix/GetAllDedicatedVar/GetAllOncall` - filtered list endpoints with pagination and total headers.
- `GetDedicatedFixByID/GetDedicatedVarByID/GetOncallByID` - single contract fetch by ID.
- `CreateDedicatedFix/CreateDedicatedVar/CreateOncall` - create endpoints.
- `UpdateDedicatedFix/UpdateDedicatedVar/UpdateOncall` - bounded-body updates that merge payload onto current entity, clear associations, and submit audited update.
- `DeleteDedicatedFix/DeleteDedicatedVar/DeleteOncall` - delete endpoints.
- `UpdateDedicatedFixAgreement/UpdateDedicatedVarAgreement/UpdateOncallAgreement` - agreement note mutation + audit.
- `GetAuditHistory/GetVendorAuditHistory` - audit retrieval endpoints.
- `UpdateVendorAgreement/UpdateMillAgreement` - agreement note audit insert for vendor/mill entities.

### `backend/internal/handlers/import_handler.go`

- `NewImportHandler` - dependency injection.
- `UploadAndParse` - validates upload, sanitizes filename, persists file, parses sheets, validates rows, returns preview + warnings.
- `sanitizeUploadedExcelFilename` - blocks traversal/absolute/non-excel names.
- `ConfirmImport` - validates request + mode + path safety, imports from edited sheets or saved file, returns result, cleans uploaded file.

### `backend/internal/handlers/export_handler.go`

- `NewExportHandler` - dependency injection.
- `ExportDedicatedFix/ExportDedicatedVar/ExportOncall` - generates excel file and streams response as downloadable attachment.

### `backend/internal/handlers/search_handler.go`

- `NewSearchHandler` - dependency injection.
- `Search` - global search endpoint across contracts + master entities with grouped suggestions.
- `appendContractListResult` - appends synthetic "view all results" item for contract groups.
- `buildContractLabel` - builds compact contract label with SPK and parties.
- `buildContractLabelWithRoute` - extends contract label with origin-destination details.

### `backend/internal/services/password_policy.go`

- `ValidatePassword` - enforces minimum strength and rejects weak/default passwords.

### `backend/internal/services/users_services.go`

- `NewUserService` - builds user service with parsed JWT expiry.
- `EnsureDefaultAdmin` - ensures admin exists without forced sync.
- `EnsureDefaultAdminWithSync` - ensures admin exists; optionally syncs password hash.
- `CreateUser` - validates and hashes password, persists user.
- `Login` - validates credentials and issues signed JWT.
- `GetByID` - reads user by ID.
- `ParseToken` - validates and parses JWT map claims.
- `ListUsers` - returns all users.
- `DeleteUser` - deletes user after existence check.
- `UpdateUserPassword` - validates + hashes new password and updates user.

### `backend/internal/services/master_service.go`

- `NewMasterService` - dependency injection.
- `GetAll*/Get*ByID/Create*/Update*/Delete*` for `Mill/Vendor/Product/Zone/Mot/Uom` - thin business layer delegating to repository.
- `SearchVendorsLimited/SearchMillsLimited/SearchZonesLimited` - delegated search APIs for global search.

### `backend/internal/services/contract_service.go`

- `RunInTransaction` (gorm runner) - executes contract/audit/master operations in shared DB transaction.
- `NewContractService` - initializes contract service with transaction runner.
- `GetAll*/Get*Page/Get*ByID/Create*/Update*/Delete*` for dedicated-fix/dedicated-var/oncall - contract CRUD and read logic with transaction-scoped audited writes.
- `Update*Agreement` for each contract type - note-only update + audit record.
- `sanitizeUpdateMap` - strips association/meta keys and normalizes numeric fields for map-based updates.
- `UpdateDedicatedFixMap/UpdateDedicatedVarMap/UpdateOncallMap` - partial update paths with audit.
- `GetAuditHistory/GetVendorAuditHistory` - audit query delegation.
- `UpdateVendorAgreement/UpdateMillAgreement` - agreement note audit operations for master entities.
- `withMutationTransaction` - shared transactional wrapper for mutation methods.
- `logAudit` - creates audit row with marshaled old/new snapshots.
- `marshalAuditData` - JSON marshaling helper for audit payloads.

### `backend/internal/services/import_service.go`

- `RunInTransaction` (gorm import runner) - wraps import write paths in DB transaction.
- `NewImportService` - builds import service + transaction runner.
- `(*ImportValidationError).Error` - returns human-readable validation error text.
- `IsImportValidationError` - type-check helper for validation errors.
- `ConfirmImport` - parse-then-import entrypoint from file.
- `ConfirmParsedSheets` - import entrypoint from edited in-memory preview sheets.
- `runInTransaction` - chooses transaction runner or direct execution.
- `importParsedSheets` - orchestrates per-sheet import routines and aggregates metrics/errors.
- `resolveVendor/resolveMill/resolveProduct/resolveZone/resolveMot/resolveUom` - maps row text fields to foreign key IDs with find-or-create behavior.
- `contractDedupeKey` - normalized dedupe key builder.
- `formatImportRowError` - standardized row error formatter.
- `importDedicatedFix/importDedicatedVar/importOncall` - sheet-specific row mapping, duplicate detection, model construction, and batch insert.
- `isEmptyRow` - skips fully blank rows.
- `findField` - robust header/value resolver across alias/newline/normalized forms.
- `parseDecimal` - locale-tolerant numeric parser (ID/US style).
- `parseDate` - multi-format date parser with excel serial fallback.

### `backend/internal/services/parser_service.go`

- `NewParserService` - constructor.
- `ParseExcelFile` - opens workbook and parses each sheet into normalized row/header data.
- `parseSheet` - turns one excel sheet into `ParsedSheet`.
- `detectSheetType` - identifies contract type from headers/sheet name.
- `ValidateRow` - row-level required-field and type validations.

### `backend/internal/services/export_service.go`

- `NewExportService` - dependency injection.
- `ExportDedicatedFix/ExportDedicatedVar/ExportOncall` - reads records and writes excel export rows per type.
- `setCellValue/setCellDecimal/setCellTime/setCellProduct/setCellZone/setCellMot/setCellUom` - specialized cell write helpers with null-safe formatting.

### `backend/internal/services/calculation_service.go`

- `NewCalculationService` - constructor.
- `CalcCostPerKGPerKM` - calculates cost per kg-km with divide-by-zero guard.
- `CalcRunningCost` - computes running cost from total cost, payload tons, and distance.
- `CalcDistributedCost` - averages/sums monthly costs into distributed cost figure.
- `CalcTotalCostWithLoadUnload` - derives total logistics cost including loading/unloading components.

### `backend/internal/repositories/user_repository.go`

- `NewUserRepository` - constructor.
- `Create` - inserts user row.
- `FindByUsername` - lookup by username.
- `FindByID` - lookup by ID.
- `Count` - total users.
- `FindAll` - list users.
- `DeleteByID` - deletes by ID.
- `UpdatePasswordHash` - updates password hash column.

### `backend/internal/repositories/master_repository.go`

- `NewMasterRepository` and `DB` - constructor + DB accessor.
- `updateExisting` - generic update helper with not-found behavior.
- `deleteExisting` - generic delete helper with not-found behavior.
- `GetAll*/Get*ByID/Create*/Update*/Delete*` for `Mill/Vendor/Product/Zone/Mot/Uom` - entity-level data access.
- `SearchMillsLimited/SearchVendorsLimited/SearchZonesLimited` - limited search queries for global search.
- `BulkCreateMills/BulkCreateVendors/BulkCreateProducts/BulkCreateZones` - batch insert APIs.
- `FindOrCreateVendorByName/FindOrCreateMillByName/FindOrCreateProductByName/FindOrCreateZoneByName/FindOrCreateMotByName/FindOrCreateUomByName` - import-oriented upsert-like helpers with generated codes.
- `importCodeFromName` - deterministic short code generator from name.
- `normalizeCodeBase` - normalizes source text before code derivation.

### `backend/internal/repositories/contract_repository.go`

- `NewContractRepository` and `DB` - constructor + DB accessor.
- `updateExistingContract/updateExistingContractMap/deleteExistingContract` - generic contract mutation helpers.
- `normalizeSPK` - normalizes SPK identifier.
- `applyCommonContractFilters` - applies shared vendor/mill filters.
- `buildDedicatedFixQuery` - composes dedicated-fix search query with joins and text predicates.
- `GetAll*/Get*Page/Get*ByID/Create*/Update*/Delete*` for dedicated-fix/dedicated-var/oncall - entity-level contract data access.
- `BulkCreateDedicatedFix/BulkCreateDedicatedVar/BulkCreateOncall` - import batch insert APIs.
- `FindDedicatedFixBySPK/FindDedicatedVarBySPK/FindOncallBySPK` - raw SPK lookups.
- `FindDedicatedFixBySPKVendorMill/FindDedicatedVarBySPKVendorMill/FindOncallBySPKVendorMill` - normalized duplicate checks for import.
- `UpdateDedicatedFixMap/UpdateDedicatedVarMap/UpdateOncallMap` - map-based partial updates.

### `backend/internal/repositories/audit_repository.go`

- `NewAuditRepository` and `DB` - constructor + DB accessor.
- `Create` - insert audit log row.
- `GetByEntity` - list logs per entity/type.
- `GetLatestByEntity` - latest log for one entity.
- `GetAll` - recent logs with optional limit.
- `GetByVendor` - merged audit history for vendor + related contracts.

## Frontend

### Core Provider And Auth

#### `frontend/src/providers/refine-provider.tsx`

- `RefineProvider` - app shell bootstrap: session check, auth provider wiring, refine resources registration, conditional login/layout rendering.
- `SessionCheckingState` - loading screen while auth session check runs.
- `RefineAppShell` - wraps `Refine` with data/auth/notification providers and resource map.

#### `frontend/src/lib/auth.ts`

- `getToken` - placeholder token getter (returns `null` in cookie-based auth mode).
- `setAuth` - stores user profile in localStorage.
- `clearAuth` - removes stored user profile.
- `getAuthUser` - reads and parses user profile from localStorage.

#### `frontend/src/lib/api-client.ts`

- `apiClient` - axios client with `withCredentials` enabled.
- response interceptor - on `401`, clears local auth and redirects to `/login`.

#### `frontend/src/contexts/color-mode/index.tsx`

- `ColorModeContextProvider` - manages and persists light/dark mode and exposes `setMode`.

### Shared Components

#### `frontend/src/components/master-data/simple-master-pages.tsx`

- `SimpleMasterList` - generic searchable table list UI for simple master entities.
- `SimpleMasterCreate` - generic create form wrapper.
- `SimpleMasterEdit` - generic edit form wrapper.
- `SimpleMasterForm` - shared create/edit form implementation.

#### `frontend/src/components/layout/global-search-bar.tsx`

- `debounce` - generic debounce with cancel API.
- `buildSearchPath` - appends encoded `q` query.
- `buildContractSearchItems` - fallback contract search suggestion set.
- `getConfig` - resolves display config by resource type.
- `highlightMatch` - highlights matched query segment in labels.
- `GlobalSearchBar` - async grouped autocomplete search with abortable requests, fallback suggestions, keyboard navigation, and route push behavior.

#### `frontend/src/components/layout/index.tsx`

- `CustomLayout` - top-level app layout composition around sider/header/content.

#### `frontend/src/components/layout/header.tsx`

- `Header` - top navbar with collapse trigger, search bar, and theme toggle.

#### `frontend/src/components/layout/sider.tsx`

- `CustomSider` - sidebar navigation + active route handling + logout action.

#### `frontend/src/components/contracts/ContractModeSwitch.tsx`

- `optionLabel` - compact option rendering helper.
- `ContractModeSwitch` - mode selector (`plan`/`actual`) used by explorer/import workflows.

#### `frontend/src/components/contracts/AgreementHistoryPanel.tsx`

- `AgreementHistoryPanel` - loads audit timeline and submits agreement note updates.
- internal `fetchHistory` - retrieves logs for entity.
- internal `handleSubmit` - posts note update and refreshes timeline.

#### `frontend/src/components/common/app-spinner.tsx`

- `AppSpinner` - reusable loading indicator component with optional text/overlay behavior.

### App Pages

#### Thin Wrappers Around Generic Master Components

- `ProductList/ProductCreate/ProductEdit` - wire product pages to `SimpleMaster*`.
- `MotList/MotCreate/MotEdit` - wire MOT pages to `SimpleMaster*`.
- `UomList/UomCreate/UomEdit` - wire UOM pages to `SimpleMaster*`.
- `VendorCreate/VendorEdit`, `MillCreate/MillEdit`, `ZoneCreate/ZoneEdit` - simple create/edit wrappers for specific entities.

#### Full List Pages With Custom Table

- `VendorList`, `MillList`, `ZoneList` - custom searchable table pages using refine `useTable`, location-synced `q`, and action buttons.

#### Contract List Pages

- `DedicatedFixList`, `DedicatedVarList`, `OncallList` - searchable, paginated contract tables with formatting helpers and row actions.
- helper `formatIDR` - currency formatter.
- helper `formatDate` - display formatter for validity date fields.

#### Contract Create/Edit Pages

- `DedicatedFixCreate/DedicatedFixEdit`
- `DedicatedVarCreate/DedicatedVarEdit`
- `OncallCreate/OncallEdit`
- shared page logic in each file: normalize numeric fields, submit via refine form hooks, format/parse IDR inputs where applicable.

#### Detail/Show Pages

- `VendorShow` - composes vendor profile with contract summaries, spend metrics, and vendor audit note timeline.
- `MillShow` - composes mill profile with route coverage map, metrics, contract sections, and audit note timeline.
- `ZoneShow` - zone detail page with refine `useShow` data presentation.

#### `frontend/src/app/import/page.tsx`

- `ImportWizardPage` - full upload -> parse preview -> editable rows -> confirm import workflow.
- `handleModeChange` - resets UI state on plan/actual switch.
- `handleUpload` - uploads excel and loads parser result.
- `handleConfirm` - submits parsed/edited sheets for DB persistence.
- `handleReset` - clears workflow state.
- `handlePreviewCellChange` - immutably edits in-memory preview cell.
- `buildPreviewRows` - adds stable preview row index.
- `buildPreviewColumns` - builds editable column config dynamically from headers.

#### `frontend/src/app/explorer/page.tsx`

- `DataExplorerPage` - unified cross-contract explorer with dynamic filters, visible-column controls, and export.
- helpers `formatIDR`, `formatDate`, `toNumber` - normalization/formatting utilities.
- memo `allRows` - normalizes three contract datasets into one schema.
- memo `filteredRows` - applies type/mill/vendor/date/text filters.
- memo column builders - dynamic table columns and column-visibility logic.
- `handleResetFilters` - resets filter state.
- `csvEscape` + `handleExport` - CSV export pipeline.

#### `frontend/src/app/overview/page.tsx`

- `DashboardOverview` - fetches all key datasets and computes KPI cards, trends, and chart inputs.
- helpers `formatIDR`, `formatNumber`, `formatDate` - display format utilities.
- memos `totalLandedCost`, `avgRunningCost`, `expiringContracts`, `contractPropData`, `trendData`, and related chart datasets - derived analytics state.

#### `frontend/src/app/admin/users/page.tsx`

- `AdminUsersPage` - admin user management page (list/create/delete/password update).
- `fetchUsers` - refreshes user list.
- `onFinish` - creates user.
- `handleDeleteUser` - deletes user with confirmation.
- `openPasswordModal` - opens password reset dialog for selected user.
- `handleUpdatePassword` - submits password change.

#### App Shell / Meta

- `RootLayout` (`app/layout.tsx`) - global HTML/body/app wrapper.
- `Home` (`app/page.tsx`) - initial route component.
- `LoginPage` (`app/login/page.tsx`) - username/password login flow and visual composition.
- `ProcurementLogo` (`app/login/page.tsx`) - inline SVG logo component.
- `manifest` (`app/manifest.ts`) - web manifest metadata provider.

## Notes For Review Sessions

- Backend flow is `handler -> service -> repository`.
- Write operations for contracts/import are transaction-scoped and audit-aware.
- Frontend data access is centralized through `apiClient` and Refine providers.
- Contract pages share repeated numeric normalization behavior. If you refactor one, inspect all three contract mode pages for consistency.
- Import and explorer pages are the highest complexity frontend modules; start review there for logic correctness risk.
