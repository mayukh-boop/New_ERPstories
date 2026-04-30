# 🚀 ERP Backend Architecture Guide (Spring Boot 3 + Java 21)

---

## ✅ MANDATORY COMPLETION CHECKLIST (Every Prompt — No Exceptions)

After implementing ANY feature, API change, bug fix, or business logic modification, ALL of the following MUST be completed before the task is considered done. This applies automatically to every prompt without needing to be reminded:

| # | What | Where |
|---|------|--------|
| 1 | **Test cases** — add/update unit tests covering happy path + all error scenarios | `src/test/java/com/codeverse/cedge/` |
| 2 | **Swagger/OpenAPI** — update `@Operation`, `@Tag`, description on all new/modified endpoints | Controller `@Operation` annotations |
| 3 | **Postman Collection** — add/update requests; include `event` scripts to auto-save tokens from responses | `CEdge_Postman_Collection.json` |
| 4 | **CLAUDE.md** — update architecture docs when new tables, flows, patterns, or APIs are introduced | This file |

---

## 📌 Overview

This document defines the **architecture, coding standards, design principles, and development guidelines** for building a scalable, maintainable, and enterprise-grade ERP backend system.

---

## 🧱 Tech Stack

### Core Technologies

| Layer | Library / Tool | Version |
|---|---|---|
| Framework | Spring Boot | 3.2.4 |
| Language | Java | 21 |
| Build tool | Maven | 3.x |
| Security | Spring Security + JWT (JJWT) | JJWT 0.12.5 |
| Database | PostgreSQL | managed by Spring Boot BOM |
| ORM | Spring Data JPA + Hibernate (multi-tenant) | managed by Spring Boot BOM |
| Schema migration | Flyway | managed by Spring Boot BOM |
| Mapping | MapStruct | 1.5.5.Final |
| Boilerplate | Lombok | 1.18.32 |
| API docs | SpringDoc OpenAPI (Swagger UI) | 2.5.0 |
| Email | Spring Boot Starter Mail (JavaMailSender) | managed by Spring Boot BOM |
| Connection pool | HikariCP | managed by Spring Boot BOM |

### DevOps & Tools

* CI/CD: Jenkins
* Containerization: Docker
* Version Control: Bitbucket (Git)

---

## 🧠 Architecture Style

### Layered Architecture (Strict Separation)

```
Controller → Service → Repository → Database
              ↓
           DTO / Mapper
```

### Modules (Recommended ERP Structure)

* Auth Module
* User & Role Management
* Tenant Management
* Inventory
* Orders / Sales
* Finance / Billing
* Audit & Logging

---

## 🔐 Authentication & Security

### JWT based Authentication

* Stateless authentication using JWT tokens
* JWT claims include:

  * `sub` → userId (Long, stored as string)
  * `tenantSchema` → schema name (used by Hibernate for DB routing)
  * `tenantId` → tenant Long ID as string (used to scope tenant-level queries)
  * `role` → role name for the selected tenant (e.g. `SUPER_ADMIN`, `ADMIN`, `USER`) — read from `user_tenants.role_id` → `public.roles.name`; SUPER_ADMIN uses `users.role` directly
  * `roleId` → role PK from `public.roles` — informational only, never used for server-side security decisions; use `role` name for `hasAuthority()` checks

### Role Naming Convention

* Role names have **no** `ROLE_` prefix: `SUPER_ADMIN`, `ADMIN`, `USER`
* Use `hasAuthority('SUPER_ADMIN')` in Spring Security — never `hasRole()`
* Default role on registration: `ADMIN`
* Default role when creating a user via API: `USER`

### Two-Step Login Flow

1. **POST /auth/login** `{ email, password }` → validates credentials, returns `preAuthToken` + list of tenants user belongs to (each with their single `role`)
2. **POST /auth/select-tenant** `{ preAuthToken, tenantSchemaName }` → validates membership, issues JWT access token (response body) + refresh token (**HttpOnly cookie**)

This supports one user belonging to multiple companies (tenants).

### Registration Flow (Three-Step: Send OTP → Verify OTP → Register)

Email is verified **before** any account is created — no unverified user records in the database.

**Step 1 — POST /auth/send-otp** `{ email }`:
1. Validates email is not already registered
2. **Cooldown check** — if a previous OTP was created within the last 60 s (configurable: `app.otp.cooldown-seconds`), throws `ValidationException` with seconds remaining
3. Deletes any previous OTP rows for this email (`DELETE` — not mark-used)
4. Generates a 6-digit OTP, saves to `public.email_verification_otps` keyed by **email** (no user FK — user doesn't exist yet), expiry = `app.otp.expiry-seconds` (default 10 min)
5. Sends OTP email via JavaMailSender
6. Returns `ApiResponse<Void>`


**Step 2 — POST /auth/verify-otp** `{ email, otp }`:
1. Finds the most recent OTP for the email
2. Validates OTP value + not expired
3. **Deletes the OTP row** (not mark-used — row is removed immediately after use)
4. Generates a UUID `registrationToken`, saves to `public.registration_tokens` (30-min expiry, single-use)
5. Returns `{ registrationToken }` — frontend passes this to Step 3

**Step 3 — POST /auth/register** `{ registrationToken, password, firstName, lastName, company: { companyName, companyCode, companyGst, companyEmail, companyPhone, companyDialCode, companyCity, companyState, companyCountry, companyAddress, decimalDigits } }`:
1. Validates registrationToken (not used, not expired)
2. Extracts email from token record; marks token used
3. Safety-checks email uniqueness (race condition guard)
4. Generates a unique tenant schema name: clean companyCode + `_XXXX` (4-digit suffix), e.g. `vim corp` → `vim_corp_6721`
5. Creates `public.tenants` row and provisions the schema (Flyway)
6. Saves company details in `public.companies` (linked to new tenant, `parent_company_id = NULL` — root company)
7. Creates `public.users` row (username = email, status = **`ACTIVE`**, `firstLogin = true`, `role = 'ADMIN'`)
8. Creates `public.user_tenants` membership (with `company_id` set to the new company's ID and `role_id` = ADMIN role ID from `public.roles`)
9. **Auto-seeds tenant schema via `TenantDefaultDataService.seedDefaultData()`** (after the public-schema transaction commits, in a dedicated transaction with `TenantContext` set):
   * `role_module` — `ADMIN` role (full privileges) and `USER` role (VIEW-only)
   * `role_module_menu` — one row per (role × module × link) for every active `public.module` / `public.module_wise_links` entry; ADMIN gets `VIEW,ADD,EDIT,DELETE`, USER gets `VIEW`
   * `company` (tenant schema) — tenant-scoped company profile with `companyName`
   * `app_user` — the registering admin mapped to the ADMIN role
10. **Returns `ApiResponse<Void>` — no tokens.** User logs in via the two-step login flow.

### First Login Flag
- `firstLogin = true` is set on every new user at registration
- On the first successful `POST /auth/select-tenant`, the flag is cleared (`false`) and `AuthResponse.firstLogin = true` is returned
- Frontend uses `firstLogin: true` to trigger the onboarding / guided tour
- All subsequent logins return `firstLogin: false`

### Forgot Password Flow (Three-Step: Send OTP → Verify OTP → Reset Password)

**Step 1 — POST /auth/forgot-password** `{ email }`:
1. Validates email belongs to an existing user (throws if not found or PENDING_VERIFICATION)
2. **Cooldown check** — if a previous reset OTP was created within the last 60 s (configurable: `app.otp.cooldown-seconds`), throws `ValidationException` with seconds remaining
3. Deletes any previous reset OTP rows for this email (`DELETE` — not mark-used)
4. Generates a 6-digit OTP, saves to `public.password_reset_otps` keyed by **email**, expiry = `app.otp.expiry-seconds` (default 10 min)
5. Sends OTP email via JavaMailSender (red-themed — distinct from registration OTP)
6. Returns `ApiResponse<Void>`

**Step 2 — POST /auth/verify-reset-otp** `{ email, otp }`:
1. Finds the most recent reset OTP for the email
2. Validates OTP value + not expired
3. **Deletes the OTP row** (not mark-used — row is removed immediately after use)
4. Generates a UUID `passwordResetToken`, saves to `public.password_reset_tokens` (30-min expiry, single-use)
5. Returns `{ passwordResetToken }` — frontend passes this to Step 3

**Step 3 — POST /auth/reset-password** `{ passwordResetToken, newPassword }`:
1. Validates passwordResetToken (not used, not expired)
2. Marks token used **before** updating password (prevents race conditions)
3. Finds user by email from token record
4. Encodes and saves new password
5. Revokes all active refresh tokens (`revokeAllByUserId`) — forces re-login
6. Returns `ApiResponse<Void>`

### Token Strategy

* **Access Token**: Short-lived (30 mins), contains both `tenantSchema` + `tenantId`; stateless — never stored in DB; validated by signature only on every request. Returned in the response **body**.
* **Refresh Token**: Stored in `public.refresh_tokens` (configurable, default 24 h via `${JWT_REFRESH_EXPIRY_MS}`), single-use rotation; bound to a specific `tenant_id`. **Delivered as an HttpOnly cookie** (`refresh_token`) — never in the response body.
* **Pre-Auth Token**: 5 min, single-use, stored in `public.pre_auth_tokens`
* **Email Verification OTP**: configurable expiry (default 10 min), stored in `public.email_verification_otps` (keyed by email); **deleted** from DB on use or resend
* **Password Reset OTP**: configurable expiry (default 10 min), stored in `public.password_reset_otps` (keyed by email); **deleted** from DB on use or resend
* **OTP Cooldown**: configurable (default 60 s via `app.otp.cooldown-seconds`); both OTP flows enforce this between resend requests
* **Password Reset Token**: 30 min, single-use, stored in `public.password_reset_tokens`

### HttpOnly Cookie — Refresh Token Delivery

The refresh token is set via `Set-Cookie` (never in the response JSON body):

```
Set-Cookie: refresh_token=<value>; HttpOnly; SameSite=Strict; Path=/cedge/api/v1/auth; Max-Age=86400
```

* **Cookie name**: `refresh_token`
* **Path**: `/cedge/api/v1/auth` — cookie is only sent to `/auth/refresh` and `/auth/logout`
* **Secure**: controlled by `app.cookie.secure` (`${COOKIE_SECURE:false}`) — set `true` in production (HTTPS)
* **SameSite**: `Strict` — prevents CSRF
* Endpoints that **set** the cookie: `POST /auth/select-tenant`, `POST /auth/force-login`, `POST /auth/refresh`
* Endpoints that **read** the cookie (`@CookieValue`): `POST /auth/refresh`, `POST /auth/logout`
* Endpoints that **clear** the cookie (Max-Age=0): `POST /auth/logout`
* `AuthResponse.refreshToken` is annotated `@JsonIgnore` — the field exists internally (used by the controller to write the cookie) but is never serialised to JSON

### CORS Configuration

CORS in `SecurityConfig` must use `setAllowedOrigins()` (not `setAllowedOriginPatterns()`) when `allowCredentials(true)` is set — wildcard origins are forbidden by the browser with credentials.

Current allowed origin: `http://localhost:5173` (Vite dev server). Update for production deployment.

### Tenant Switching

* A JWT access token is always scoped to **exactly one tenant** (`tenantSchema` + `tenantId` claims are fixed at issue time — cannot be changed without issuing a new token)
* **Existing tokens cannot be reused for a different tenant.** The access token routes all DB queries to the baked-in schema; the refresh token has `tenant_id` stored in DB and always re-issues tokens for that same tenant
* To switch tenants, the user MUST run the full two-step login again: `POST /auth/login` → `POST /auth/select-tenant` with the new `tenantSchemaName`
* The `preAuthToken` from step 1 is **single-use** — it is marked `used=true` after the first `select-tenant` call and cannot be reused to pick a second tenant; a fresh `POST /auth/login` is required
* The old tenant's refresh token is NOT automatically revoked on switch — remains active until expiry or explicit logout via `POST /auth/logout`
* The old tenant's access token remains valid until it expires (30 min) — this is by design (stateless JWT, cannot be remotely invalidated)

### Security Rules

* Use Spring Security filters
* Validate token on every request
* Always validate:

  * User global status (ACTIVE / DISABLED / LOCKED)
  * Tenant status (ACTIVE / DISABLED)
  * User-tenant membership status (ACTIVE / DISABLED)

---

## 🏢 Multi-Tenancy Strategy

### Approach

**Shared DB + Separate Schema + Global User Identity**

* Each tenant → separate schema (for future business data: inventory, orders, etc.)
* Users and roles live in `public` schema — they are **global**, not per-tenant
* Hibernate Multi-Tenancy routes business-data queries to correct schema

### User-Tenant Model

* `public.users` → one global row per person (username = email, email must be unique globally); `role VARCHAR(50)` retained for SUPER_ADMIN detection only — all other roles use `user_tenants.role_id`
* `public.roles` → role definitions; `tenant_id = NULL` = system roles (SUPER_ADMIN, ADMIN, USER); `tenant_id = X` = custom roles created by that tenant's admin
* `public.tenants` → one row per tenant; schema name is auto-generated from company code
* `public.companies` → company details linked to tenants (company name, code, GST, email, phone, tenantId, `parent_company_id`); root companies from registration have `parent_company_id = NULL`; child companies created via "Add Company" API have `parent_company_id` pointing to the parent
* `public.user_tenants` → membership: which user belongs to which tenant; includes `company_id` FK and `role_id` FK → `public.roles(id)` — role is **per-tenant** (a user can be ADMIN in one company and USER in another)
* Role resolution: SUPER_ADMIN → `users.role`; all others → `user_tenants.role_id` → `public.roles.name` at `selectTenant` time, embedded in JWT

### Tenant Resolution

* `TenantContext` holds two `ThreadLocal` values:
  * `schemaName` (String) → used by `TenantConnectionProvider` to call `connection.setSchema()`
  * `tenantId` (Long) → used by services to scope queries on public-schema entities only
* Both are extracted from the JWT on every request by `JwtAuthenticationFilter`
* `TenantContext.clear()` is ALWAYS called in `finally` blocks

### Master Table Schema Routing (No `tenant_id` Columns)

All 13 master/business tables live in the **tenant schema**, not in `public`. Hibernate's `TenantConnectionProvider` already routes every query to the correct tenant schema — no explicit `tenant_id` column or filter is needed.

**Tables in the tenant schema** (no `tenant_id` column):
`color_master`, `uoms_master`, `fabric_construction_types`, `vendors_master`, `brand_masters`, `lot_masters`, `transport_masters`, `fabric_masters`, `fabric_multiple_uoms`, `fabric_inventory_limits`, `process_master`, `process_type_master`, `employee_masters`

**How it works:**
1. JWT contains `tenantSchema` claim (e.g. `acme_corp_1234`)
2. `JwtAuthenticationFilter` calls `TenantContext.setTenant(schemaName, tenantId)` on every request
3. `TenantConnectionProvider` calls `connection.setSchema(schemaName)` for every Hibernate session
4. Hibernate issues all queries against the active schema — master table rows are automatically isolated per tenant

**The key toggle — `@Table(schema = "public")`:**
* **With** `@Table(schema = "public")` → entity always queries `public` schema (used for auth/RBAC: `users`, `roles`, `tenants`, `companies`, etc.)
* **Without** `@Table(schema = "public")` → entity queries whatever schema `TenantConnectionProvider` has set (used for ALL master/business entities)

**Flyway migration paths:**
* `db/migration/public/` → runs on application startup against the `public` schema (auth, RBAC, module/menu config)
* `db/migration/tenant/` → runs via `TenantSchemaService.provisionTenantSchema()` when a new tenant registers (all master tables)

**Rules for master entities:**
* ❌ Do NOT add `@Table(schema = "public")` to master entities — this breaks tenant isolation
* ❌ Do NOT add `tenant_id` columns to master tables in the tenant schema
* ❌ Do NOT call `entity.setTenantId(...)` on master entities — the field does not exist
* ❌ Do NOT call `TenantContext.getTenantId()` in master service methods — schema routing handles isolation
* ✅ Only use `TenantContext.getTenantId()` in services that query **public-schema** entities (User, Role, Company, RoleModule, etc.)

---

## 🧩 Design Principles

### SOLID Principles

1. **S - Single Responsibility**

   * One class = one responsibility

2. **O - Open/Closed**

   * Extend behavior without modifying code

3. **L - Liskov Substitution**

   * Subtypes must be replaceable

4. **I - Interface Segregation**

   * Avoid fat interfaces

5. **D - Dependency Inversion**

   * Depend on abstractions, not implementations

---

## 🏗️ Design Patterns

### 1. MVC Pattern

* Controller → API layer
* Service → Business logic
* Repository → Data access

---

### 2. Singleton Pattern

* Used for:

  * Configuration classes
  * Utility services (Spring beans are singleton by default)

---

### 3. Constructor Injection (MANDATORY)

```java
@Service
public class UserService {
    private final UserRepository userRepository;

    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }
}
```

✔ No field injection
✔ Improves testability

---

### 4. Factory Method Pattern

Used for:

* Creating objects based on type (e.g., payment types, notification types)

---

### 5. DTO Pattern

* Never expose entities directly
* Use DTOs for API communication

```java
UserDTO → API Response
User → Entity
```

---

### 6. MapStruct (MANDATORY)

* Use MapStruct for mapping DTO ↔ Entity

```java
@Mapper(componentModel = "spring")
public interface UserMapper {
    UserDTO toDto(User user);
    User toEntity(UserDTO dto);
}
```

✔ No manual mapping
✔ Compile-time safety

---

### 7. Lombok Usage

Use:

* @Getter, @Setter
* @Builder
* @RequiredArgsConstructor

Avoid:

* @Data (for entities)

---

### 8. Complex Queries & Data Access Strategies

To maintain strict architectural boundaries (`Service -> Repository`), we define strict rules on how to handle complex `EntityManager` queries, native SQL, and cross-module reporting to avoid cluttering repositories or blindly creating redundant DAO layers.

**Rule 1: Standard CRUD (90% of cases)**
* Always use Spring Data JPA `Repository` interfaces. 
* **DO NOT** create a separate `DAO` class just to wrap standard `.save()` or `.findById()` methods.
* **Flow:** `Service -> Repository`

**Rule 2: Complex Specific Queries (The Custom Repository Pattern)**
* For highly dynamic, complex queries targeting a single entity (e.g., multi-filter searches using `CriteriaBuilder` or custom Native SQL), do NOT create a separate `<Entity>DAO`. Instead, use the **Spring Custom Repository Pattern**.
  1. Create a custom interface: `ProductRepositoryCustom`
  2. Create the implementation: `ProductRepositoryImpl implements ProductRepositoryCustom` (Inject `EntityManager` here).
  3. Extend both in the main repo: `public interface ProductRepository extends JpaRepository<Product, Long>, ProductRepositoryCustom`
* **Flow:** `Service -> ProductRepository -> ProductRepositoryImpl`

**Rule 3: Cross-Module Analytics (The CQRS Lite DAO Pattern)**
* For massive dashboards, financial reports, or aggregations that span multiple disparate modules/tables, projecting into a single entity repository is an anti-pattern. 
* For these specific, **read-only bulk operations**, you MAY create a separate `dao` package or dedicated DAO classes.
* Create a dedicated class (e.g., `FinancialReportDAO`) annotated with `@Repository` and inject `EntityManager`.
* Result sets MUST map directly to DTOs, preventing large networks of managed Entities from loading into memory.
* **Flow:** `Service -> FinancialReportDAO -> Native SQL / Criteria -> DTOs`

**Strict Agent Instruction:** ANY software agent or developer making architectural decisions MUST verify which rule applies before introducing new structural layers, bypassing repositories, or injecting `EntityManager` directly into services.

---

## 📂 Project Structure

```
com.codeverse.cedge
 ├── config
 ├── security
 ├── controller
 ├── service
 ├── repository
 ├── entity
 ├── dto
 ├── mapper
 ├── exception
 ├── util
 └── tenant
```

---

## ⚙️ Coding Standards

### General Rules

* Use meaningful class and method names
* Follow camelCase naming
* Avoid hardcoded values → use constants

### API Standards

* Use REST conventions
* Proper HTTP status codes
* Standard response format:

```json
{
  "status": "SUCCESS",
  "message": "Operation completed",
  "data": {}
}
```

### Pagination Standard (ALL List Endpoints)

Every endpoint that returns a list MUST be paginated. No unbounded lists are allowed.

**Controller pattern** (consistent across all list endpoints):
```java
@GetMapping
public ResponseEntity<ApiResponse<PageResponse<FooDTO>>> getAll(
        @RequestParam(defaultValue = "0")    int page,
        @RequestParam(defaultValue = "10")   int size,
        @RequestParam(defaultValue = "id")   String sortBy,  // sensible default per entity
        @RequestParam(defaultValue = "desc") String sortDir) {
    Sort sort = sortDir.equalsIgnoreCase("asc") ? Sort.by(sortBy).ascending() : Sort.by(sortBy).descending();
    return ResponseEntity.ok(ApiResponse.success(service.getAll(PageRequest.of(page, size, sort))));
}
```

**Currently paginated endpoints:**

| Endpoint | Default sort |
|----------|-------------|
| `GET /api/v1/users` | `createdAt desc` |
| `GET /api/v1/tenants` | `createdAt desc` |
| `GET /api/v1/tenants/my` | `id desc` |
| `GET /api/v1/roles` | `name asc` |
| `GET /api/v1/masters/vendor` | `createdAt desc` |

**Flat list endpoints (no pagination — for dropdowns):**

| Endpoint | Auth | Returns |
|----------|------|---------|
| `GET /api/v1/companies/list` | Any authenticated | `List<CompanyListItemDTO>` — `{ companyId, companyName, tenantId, tenantSchemaName }` |

---

## 🔐 Role Module Permissions API

Manages per-role, per-link permission flags (`canView`, `canCreate`, `canEdit`, `canDelete`, `canList`). All endpoints under `/api/v1/role-modules/`.

### Endpoints

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| `GET` | `/role-modules/for-role/{roleId}` | Any authenticated | Returns `{ roleModuleId, permissions[] }` for the role in the caller's tenant context |
| `GET` | `/role-modules/for-role/{roleId}/per-company` | `SUPER_ADMIN` | Returns one `CompanyRolePermissionsDTO` per root company for a **system role** (ADMIN/USER) — see below |
| `GET` | `/role-modules/available-modules` | Any authenticated | All active modules with their active links; used to populate Add Link / Add Module panels |
| `POST` | `/role-modules/permissions` | `ADMIN` or `SUPER_ADMIN` | Adds a single link permission row |
| `PUT` | `/role-modules/permissions/{id}` | `ADMIN` or `SUPER_ADMIN` | Updates flags on an existing permission row |
| `DELETE` | `/role-modules/permissions/{id}` | `ADMIN` or `SUPER_ADMIN` | Removes a single permission row |

### Per-Company View (`GET /role-modules/for-role/{roleId}/per-company`)

Used by the Settings > Role Permissions page when SUPER_ADMIN views a system role (ADMIN/USER — where `public.roles.company_id IS NULL`).

- Returns one `CompanyRolePermissionsDTO` per **root company** (`parent_company_id IS NULL`), sorted by company name
- If a company has no `role_module` row for the given role: one is **auto-created** (with no permissions) so SUPER_ADMIN can immediately add mappings
- Throws `ValidationException` if `roleId` resolves to a custom/tenant-scoped role (`company_id IS NOT NULL`)
- Throws `ResourceNotFoundException` if the role doesn't exist

```java
// CompanyRolePermissionsDTO (dto/rolemodule/CompanyRolePermissionsDTO.java)
{
  companyId:    Long
  companyName:  String
  tenantId:     Long
  tenantCode:   String
  roleModuleId: Long
  permissions:  List<RoleModuleMenuDTO>
}
```

### Common Mistakes to Avoid (Role Module Permissions)

* ❌ Calling `/for-role/{roleId}/per-company` with a custom-role ID — only system roles (`company_id IS NULL`) are valid; throws `ValidationException` otherwise
* ❌ Assuming per-company data is scoped to the caller's `TenantContext` — this endpoint reads all root companies from `public.companies` without a tenant filter
* ❌ Forgetting to auto-create `role_module` rows in the per-company endpoint — any company without an existing row must get one created so the frontend can immediately add permissions

**Service pattern**: accept `Pageable`, return `PageResponse<DTO>`.  
**Repository pattern**: use Spring Data `Page<Entity> findBy...(Pageable pageable)` overloads; for custom JPQL queries add a `Pageable` overload alongside the `List<>` overload.

---

## ❗ Exception Handling

* Global Exception Handler using `@ControllerAdvice`
* Custom exceptions:

  * ResourceNotFoundException
  * UnauthorizedException
  * ValidationException

---

## 📊 Logging

* Use SLF4J
* Log levels:

  * INFO → business flow
  * DEBUG → development
  * ERROR → failures

---

## 🧪 Testing Strategy

* Unit Tests → Service layer
* Integration Tests → Controller + DB
* Use JUnit + Mockito

---

## 🐳 Docker Guidelines

### Dockerfile (Backend)

* Use lightweight JDK image
* Multi-stage build recommended

### Best Practices

* Externalize configs
* Use environment variables

---

## 🔁 CI/CD Pipeline (Jenkins)

### Pipeline Stages

1. Checkout Code
2. Build (Maven)
3. Run Tests
4. Build Docker Image
5. Push Image
6. Deploy

---

## 🌿 Git Strategy (Bitbucket)

### Branching Model

* main → production
* develop → integration
* feature/* → new features
* bugfix/* → fixes

### Rules

* Pull Request mandatory
* Code review required
* No direct push to main

---

## 🔒 Security Best Practices

* Never expose secrets in code
* Use environment variables
* Enable CORS properly
* Validate all inputs

---

## 📈 Performance Considerations

* Use pagination for All Pages
* Avoid N+1 queries
* Use caching where needed

---

## 🧾 Audit & Logging (ERP Critical)

* Track:

  * Created By
  * Updated By
  * Timestamp
* Maintain audit logs for:

  * Orders
  * Inventory changes
  * User actions

---

## 🔄 Concurrent Session Management

* Every user has `max_concurrent_sessions` field (default: 1, stored in **global** `public.users` table)
* Session limit is checked in **step 2** (`POST /auth/select-tenant`), not in step 1 login
* Count active (non-revoked, non-expired) refresh tokens in `public.refresh_tokens` for that user
* If count >= max: return `SESSION_LIMIT_REACHED` status with active session list + a `forceLoginToken` (5-min, single-use, stored in `public.force_login_tokens`)
* `POST /api/v1/auth/force-login` validates the token, revokes ALL previous sessions, then issues new tokens
* Use `@Transactional` on session count check to prevent race conditions
* Force-login token is single-use — mark `used=true` on first redemption and reject further use

---

## 📋 MDC Logging Standard

* `MdcFilter` runs first (Order=1) — injects `requestId` (UUID), `requestUri`, `requestMethod` into MDC
* `JwtAuthenticationFilter` enriches MDC with `userId`, `tenantId`, `roles` after token validation
* MDC is always cleared in `finally` blocks to prevent thread-local leaks
* Log pattern must include `[reqId=%X{requestId}] [tenant=%X{tenantId}] [user=%X{userId}]`
* Use async Logback `AsyncAppender` (queue=512) wrapping file appender for non-blocking I/O
* `X-Request-Id` header is returned on every response for client-side correlation

---

## ⚡ Performance & Scalability Rules

* HikariCP pool: `minimum-idle=5`, `maximum-pool-size=20` (tune per load)
* Add DB indices on all FK columns and frequently filtered columns (see Flyway migrations)
* All write operations must be `@Transactional`; read-only queries use `@Transactional(readOnly = true)`
* Use `@Version` (optimistic locking) on `User` and `Tenant` entities to prevent lost updates
* All list endpoints must support pagination — never return unbounded lists
* All timestamps are stored as UTC in PostgreSQL (`TIMESTAMPTZ`). API responses are formatted in the app timezone (`app.timezone`, default `Asia/Kolkata`). Never change the storage format — only the serialization layer (Jackson) converts UTC→IST at response time.
* `TenantContext` uses `ThreadLocal` — always clear in `finally` block to prevent leaks in thread pools
* Design is fully stateless — horizontal scaling requires no sticky sessions

---

## 🧪 Test Coverage Requirements

* Service layer: unit tests with Mockito (`@ExtendWith(MockitoExtension.class)`)
* JWT token: dedicated `JwtTokenProviderTest` covering generate/validate/extract
* Controller layer: integration tests with `MockMvc` + `@SpringBootTest`
* All auth scenarios must have explicit test cases (28+ scenarios defined in test plan)
* Mandatory coverage: login happy path, wrong password, disabled user, disabled tenant, session limit, force-login, register duplicates, token refresh/expiry/revoke, logout, concurrent race condition
* Never mock the database in integration tests — use real repository layer

---

## 🔐 Refresh Token Policy

* Expiry: 1 day (`86400000` ms) — configurable via `${JWT_REFRESH_EXPIRY_MS}`
* Stored in **public schema** (before tenant context is established) in `public.refresh_tokens`
* Single-use rotation: on every `POST /auth/refresh`, the old token is revoked and a new token is issued (both DB row and cookie rotated)
* Delivered as an `HttpOnly; SameSite=Strict` cookie — never in JSON response body
* Read via `@CookieValue(name = "refresh_token")` in `AuthController` — not `@RequestBody`
* Tracks: `device_info`, `ip_address`, `last_used_at` for session management UI
* Access token expiry: 30 minutes (`1800000` ms)
* Cookie `Secure` flag: controlled by `app.cookie.secure` (`${COOKIE_SECURE:false}`)

---

## 🗂️ File & Image Upload Standard (MANDATORY)

> **This section is mandatory for every feature that stores files or images. No exceptions.**

### S3 Storage Path Format

All uploads MUST follow this exact key format:

```
{tenantCode}/{menuCode}/{timestamp}!_!{imageName}
```

Examples:
```
vim_corp_6721/MASTERS_FABRIC/1712345678901!_!fabric_logo.png
vim_corp_6721/INV_MGMT_STOCK/1712345678901!_!item_photo.jpg
public/USER_PROFILE/1712345678901!_!avatar.png
```

- `tenantCode` — the tenant's schema name (e.g. `vim_corp_6721`)
- `menuCode` — the menu/feature code where the upload belongs (e.g. `MASTERS_FABRIC`, `INV_MGMT_STOCK`)
- `timestamp` — `System.currentTimeMillis()` at the moment of upload
- `!_!` — fixed separator between timestamp and image name (never use `_` or `-` as separator)
- `imageName` — original file name sanitised (spaces → underscores, special chars stripped), extension preserved

### Database Storage Rule

**Store only the `{timestamp}!_!{imageName}` value in the DB column — never the full S3 path, S3 URL, or CloudFront URL.**

```java
// ✅ CORRECT — store only timestamp!_!imageName
entity.setImageName("1712345678901!_!fabric_logo.png");

// ❌ WRONG — never store full paths or URLs
entity.setImageName("vim_corp_6721/MASTERS_FABRIC/1712345678901!_!fabric_logo.png");
entity.setImageName("https://cdn.example.com/vim_corp_6721/MASTERS_FABRIC/1712345678901!_!fabric_logo.png");
```

### CloudFront URL Construction (Read Time)

At read time, construct the full CloudFront URL in a **common utility class** — never inline in services or mappers.

```java
// FileUrlUtil.java  (in com.codeverse.cedge.util)
public static String buildCloudFrontUrl(String tenantCode, String menuCode, String storedValue) {
    if (storedValue == null || storedValue.isBlank()) return null;
    // storedValue is already in the form:  {timestamp}!_!{imageName}
    return cloudfrontBaseUrl + "/" + tenantCode + "/" + menuCode + "/" + storedValue;
}
```

- `cloudfrontBaseUrl` comes from `app.cloudfront.base-url` in `application.yml`
- The URL is **constructed directly from the stored value** — no S3 lookup is performed
- This is intentional: if a file was never successfully uploaded, its value was never saved in the DB (see upload rule below)

### Upload Flow (Write Path) — STRICT ORDER

```
1. Receive file from client
2. Sanitise the original file name (spaces → underscores, strip special chars)
3. Build the stored value:  {System.currentTimeMillis()}!_!{sanitisedName}
4. Build the full S3 key:   {tenantCode}/{menuCode}/{storedValue}
5. Upload to S3 via AmazonS3.putObject()
6. Confirm S3 upload SUCCESS (no exception thrown / ETag present)
7. ONLY THEN: save the storedValue (e.g. "1712345678901!_!fabric_logo.png") to the DB column
8. If S3 upload fails: throw exception, do NOT save anything to DB
```

**Never save the stored value to the DB before the S3 upload confirms success.** A failed upload followed by a DB write produces a broken CloudFront URL with no backing file.

### Implementation Checklist (per upload feature)

| # | What |
|---|------|
| 1 | Sanitise file name: strip special chars, replace spaces with `_`, preserve extension |
| 2 | Build stored value: `System.currentTimeMillis() + "!_!" + sanitisedName` |
| 3 | Build full S3 key: `tenantCode + "/" + menuCode + "/" + storedValue` |
| 4 | Upload to S3, wait for confirmed success |
| 5 | Store **only the stored value** (e.g. `1712345678901!_!logo.png`) in the DB column |
| 6 | On reads, call `FileUrlUtil.buildCloudFrontUrl(tenantCode, menuCode, storedValue)` |
| 7 | Inject `tenantCode` from `TenantContext` / JWT claim — never hard-code |
| 8 | Inject `menuCode` from the calling service context — it is the caller's responsibility |

### Common Mistakes to Avoid (File Uploads)

* ❌ Using any separator other than `!_!` between timestamp and image name — `_` and `-` are reserved chars in file names themselves
* ❌ Storing the full S3 key (`tenantCode/menuCode/...`) in the DB — store only `{timestamp}!_!{imageName}`
* ❌ Storing the full CloudFront or S3 URL in the DB
* ❌ Performing any S3 existence check at read time — the URL is constructed directly; trust the write path
* ❌ Saving the stored value to the DB before the S3 upload succeeds
* ❌ Building CloudFront URLs inline in services or mappers — always use `FileUrlUtil`
* ❌ Using a different path format in different features — the `{tenantCode}/{menuCode}/` prefix is universal

---

## ⚠️ Common Mistakes to Avoid

* ❌ Exposing entities in APIs
* ❌ Field injection
* ❌ No DTO mapping
* ❌ Long-lived JWT tokens
* ❌ Mixing business logic in controllers
* ❌ Not clearing `TenantContext` in finally block (causes schema bleed in thread pools)
* ❌ Returning unbounded lists (always paginate)
* ❌ Skipping `@Transactional` on writes (causes dirty reads under concurrent load)
* ❌ Creating `users` or `roles` tables in tenant schemas — they are GLOBAL in `public` schema
* ❌ Putting `@Table` without `schema = "public"` on User/Role/UserTenant entities (causes them to route to tenant schema)
* ❌ Using single-step login — always two steps: `/login` then `/select-tenant`
* ❌ Registering first, verifying email after — registration is three steps: `/send-otp` → `/verify-otp` → `/register`; user is created ACTIVE only after OTP is verified
* ❌ Returning JWT tokens from `/register` — registration returns `ApiResponse<Void>`; user logs in separately after registration
* ❌ Keying `email_verification_otps` by `user_id` — OTPs are keyed by **email** (no FK); the user does not exist yet at OTP time
* ❌ Skipping registration token validation — `/register` must validate `registrationToken` (not used, not expired) from `public.registration_tokens`
* ❌ Allowing login for `PENDING_VERIFICATION` users — `validateUserStatus()` must check this status and throw `UnauthorizedException`
* ❌ Flat company fields in `RegisterRequest` — company details are nested inside a `CompanyRequest` object validated with `@Valid`
* ❌ Reading role from `public.users.role` for non-SUPER_ADMIN users — role is now per-tenant on `public.user_tenants.role_id`; only SUPER_ADMIN continues to use `users.role` as a system flag
* ❌ Not setting `role_id` on `UserTenant` when creating memberships — always resolve the role ID from `public.roles` (`findByNameAndTenantIdIsNull`) and set it on the membership row
* ❌ Passing a single `roleId` in `CreateUserRequest` — the field is now `List<TenantMembershipRequest> tenantMemberships` (one row per company/tenant); callers must provide at least one membership
* ❌ Filtering `GET /api/v1/users` by `TenantContext` when the caller is SUPER_ADMIN — SUPER_ADMIN must bypass tenant scoping and call `findAllWithSearch(search, pageable)`; ADMIN calls `findAllByTenantIdWithSearch(tenantId, ACTIVE, search, pageable)`
* ❌ Calling `GET /api/v1/users` without support for `?search=` — search is an optional query param; pass `null` to skip filtering; the JPQL query handles `NULL` via `:search IS NULL OR ...`
* ❌ Using `GET /api/v1/roles` without `?companyId=` support — SUPER_ADMIN can pass `companyId` to get roles scoped to a specific company (system roles + that company's custom roles); used in per-company role dropdowns
* ❌ Creating a `POST /api/v1/users` or `PUT /api/v1/users/{id}` request without ADMIN/SUPER_ADMIN authorization — both endpoints are guarded by `@PreAuthorize("hasAuthority('SUPER_ADMIN') or hasAuthority('ADMIN')")`
* ❌ Letting an ADMIN create/update a user with a foreign `tenantId` — ADMIN callers must only provide their own `tenantId` in `tenantMemberships`; any other `tenantId` throws `UnauthorizedException`
* ❌ Creating a child company without setting `parent_company_id` — when adding a company via the API, it must reference the parent tenant's company via `parent_company_id`
* ❌ Not setting `company_id` on `UserTenant` when creating memberships — always resolve `company_id` from the tenant's company record and set it on the membership row
* ❌ Calling `TenantDefaultDataService.seedDefaultData()` without setting `TenantContext` first — must call `TenantContext.setTenant(schemaName, tenantId)` before the call and `TenantContext.clear()` in a `finally` block after
* ❌ Mixing public-schema and tenant-schema JPA operations in the same `@Transactional` / `TransactionTemplate` — Hibernate opens a Session once per tenant identifier; switching `TenantContext` mid-transaction has no effect on the already-bound Session; use a separate `@Transactional` method (with `TenantContext` pre-set) for tenant schema writes
* ❌ Using `ROLE_` prefix on role names — roles are `SUPER_ADMIN`, `ADMIN`, `USER` (no prefix); use `hasAuthority()` not `hasRole()`
* ❌ Validating username uniqueness — only email is unique globally; usernames may duplicate
* ❌ Using UUID as primary key type — all PKs and FKs are `Long` (`BIGSERIAL` in SQL, `GenerationType.IDENTITY` in JPA). UUID is only used for opaque token *values* (`UUID.randomUUID().toString()`), never as an ID type
* ❌ Using `UUID.fromString()` in service/filter code — parse IDs with `Long.parseLong()`. `UUID.fromString()` is not used anywhere in this project
* ❌ Reusing `email_verification_otps` for password reset — use the separate `password_reset_otps` table; they are distinct flows with different email templates
* ❌ Marking OTP `used=true` and keeping the row — OTP rows are **deleted** on use (`emailVerificationOtpRepository.delete(otp)`). The `used` column no longer exists on OTP tables (removed in V15 migration)
* ❌ Allowing OTP resend without cooldown check — both `sendOtp` and `forgotPassword` must check `createdAt` of the latest OTP and reject if within `app.otp.cooldown-seconds` (default 60 s)
* ❌ Updating password before marking the reset token as used — always mark `used=true` first to prevent race conditions
* ❌ Not revoking sessions after password reset — `revokeAllByUserId()` must be called after a successful reset so the user must re-authenticate with the new password
* ❌ Allowing forgot password for `PENDING_VERIFICATION` users — they have not completed registration; throw `UnauthorizedException`
* ❌ Assuming a user can switch tenants without re-issuing tokens — a JWT is scoped to one tenant; to work under a different tenant the user must call `POST /auth/select-tenant` again with the new `tenantSchemaName` to get a fresh token pair
* ❌ Expecting logout to immediately invalidate the access token — access tokens are stateless; they remain valid until expiry (max 30 min); only the refresh token is revoked on logout
* ❌ Sending `refreshToken` in the request body for `/auth/refresh` or `/auth/logout` — these endpoints read the token from the `refresh_token` HttpOnly cookie via `@CookieValue`; there is no `@RequestBody` on these endpoints
* ❌ Using `allowedOriginPatterns("*")` with `allowCredentials(true)` — browsers block wildcard origins when credentials are enabled; always set a specific origin (e.g. `http://localhost:5173`) in `SecurityConfig`
* ❌ Setting `app.cookie.secure=true` in a plain HTTP dev environment — the browser will silently drop the cookie; keep `COOKIE_SECURE=false` locally and `true` only in HTTPS production
* ❌ Using the wrong cookie path — the `refresh_token` cookie path is `/cedge/api/v1/auth`; requests to any other path will not carry the cookie
* ❌ Returning `refreshToken` in the JSON response body — `AuthResponse.refreshToken` is `@JsonIgnore`; the controller extracts it via `auth.getRefreshToken()` to write the cookie, never to return it to the client

---

## 🏷️ Vendor Master API

Full CRUD module for managing vendors, customers, and supplier records per tenant.

### Table

`vendors_master` (tenant schema) — isolated per tenant via schema routing; `is_deleted` soft-delete flag.

| Column | Type | Notes |
|--------|------|-------|
| `id` | BIGSERIAL | PK |
| `tenant_id` | BIGINT | FK → tenants |
| `vendor_type` | VARCHAR(20) | `VENDOR` / `CUSTOMER` / `VENDOR_CUSTOMER` |
| `type` | VARCHAR(10) | `DOMESTIC` / `OVERSEAS` |
| `name` | VARCHAR(100) | NOT NULL |
| `address` | TEXT | NOT NULL |
| `country` / `country_code` | VARCHAR | display + ISO code |
| `state` / `state_code` | VARCHAR | display + ISO code |
| `city` | VARCHAR(100) | NOT NULL |
| `postal_code` | VARCHAR(10) | digits only |
| `phone` / `alt_phone` / `whatsapp_phone` | VARCHAR(15) | nullable |
| `supply_type` | VARCHAR(20) | B2B / EXPWP / EXPWOP / SEZWP / SEZWOP / DEXP |
| `currency` | VARCHAR(10) | nullable |
| `gst_no` | VARCHAR(15) | nullable, stored uppercase |
| `email` | VARCHAR(50) | nullable |
| `contact_person` | VARCHAR(50) | nullable |
| `custom_fields` | JSONB | `[{ "label": "...", "value": "..." }]` array |
| `is_deleted` | BOOLEAN | soft-delete flag |
| audit cols | TIMESTAMPTZ | created_at, updated_at, created_by, updated_by |

### Endpoints

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/api/v1/masters/vendor` | Paginated list; `search` filters by name, `gstNo` filters by GST; `search` takes priority |
| `GET` | `/api/v1/masters/vendor/{id}` | Single vendor by ID |
| `POST` | `/api/v1/masters/vendor` | Create vendor — all core fields required |
| `PUT` | `/api/v1/masters/vendor/{id}` | Update all fields |
| `DELETE` | `/api/v1/masters/vendor/{id}` | Soft-delete (sets `is_deleted = true`) |

### Key files

| Layer | File |
|-------|------|
| Entity | `entity/publicschema/VendorsMaster.java` |
| JSONB converter | `entity/publicschema/CustomFieldsConverter.java` |
| Repository | `repository/VendorsMasterRepository.java` |
| DTOs | `dto/vendor/VendorDTO`, `CreateVendorRequest`, `UpdateVendorRequest` |
| Service | `service/vendor/VendorService` + `VendorServiceImpl` |
| Controller | `controller/VendorController.java` |
| Migration | `db/migration/public/V15__vendor_master_full.sql` |
| Tests | `test/.../vendor/VendorServiceTest.java` (14 test cases) |

### Common Mistakes to Avoid (Vendor Master)

* ❌ Using `vendorRepository.findByTenantId()` — the method is `findByTenantIdAndIsDeletedFalse()` (soft-delete aware)
* ❌ Storing GST number in original case — always uppercase via `gstNo.toUpperCase()` before save
* ❌ Storing blank strings as empty strings for optional fields — use `StringUtils.hasText()` to coerce empty strings to `null`
* ❌ `custom_fields` column type is `jsonb`, not `text` — use `CustomFieldsConverter` on the entity field
* ❌ Bypassing the new `VendorService` via the old `FabricService.getVendors()` — those methods have been removed; use `VendorService` directly

---

## 🗂️ Master Data APIs

All 13 master tables live in the **tenant schema** (not `public`). Schema routing via `TenantConnectionProvider` provides automatic tenant isolation — no `tenant_id` column or filter needed on any master entity.

| Domain | Base path | Key entities |
|--------|-----------|--------------|
| Fabric | `/api/v1/masters/fabric` | `FabricMaster` + sub-resources: colors, UOMs, construction types, inventory limits |
| Color | `/api/v1/masters/fabric/colors` | `ColorMaster` |
| UOM | `/api/v1/masters/fabric/uoms` | `UomsMaster` |
| Construction Type | `/api/v1/masters/fabric/construction-types` | `FabricConstructionType` |
| Vendor | `/api/v1/masters/vendor` | `VendorsMaster` (soft-delete, `is_deleted`) |
| Brand | `/api/v1/masters/brand` | `BrandMaster` (soft-delete, `is_deleted`) |
| Lot | `/api/v1/masters/lot` | `LotMaster` (soft-delete, `is_deleted`) |
| Transport | `/api/v1/masters/transport` | `TransportMaster` (soft-delete, `is_deleted`) |
| Process Type | `/api/v1/masters/process-type` | `ProcessTypeMaster` + nested `ProcessMaster` |
| Employee | `/api/v1/masters/employee` | `EmployeeMaster` (soft-delete, `is_deleted`) |

### Common Mistakes to Avoid (Master Data)

* ❌ Adding `@Table(schema = "public")` to any master entity — breaks tenant isolation (all master data would go to public schema, shared across all tenants)
* ❌ Adding a `tenant_id` column to master tables in tenant-schema Flyway migrations — the schema itself is the isolation boundary
* ❌ Calling `TenantContext.getTenantId()` in master service methods to build a filter predicate — schema routing already isolates the data; adding a tenantId predicate causes queries to fail (no such column)
* ❌ Adding `tenantId` field to master entity classes — they don't have this column in the tenant schema
* ❌ Filtering `findAll()` results by tenant in master repositories — all records in the tenant schema already belong to that tenant
* ❌ Soft-delete pattern on `FabricMaster`, `ColorMaster`, `UomsMaster`, `FabricConstructionType`, `ProcessMaster`, `ProcessTypeMaster` — these use hard-delete; only Vendor, Brand, Lot, Transport, Employee use soft-delete

---

## ✅ Summary

This ERP backend must be:

* Scalable
* Secure
* Maintainable
* Multi-tenant ready
* Production-grade

Follow:

* SOLID principles
* Clean architecture
* Proper design patterns

---

