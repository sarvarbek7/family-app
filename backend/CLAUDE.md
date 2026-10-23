# backend — API server

Read the root [CLAUDE.md](../CLAUDE.md) first for repo-wide rules.

## Responsibility

- Owns the authoritative copy of all data and the business rules for the Family app.
- Exposes an HTTP API consumed by `frontend/` (Vue) and `mobile/` (Kotlin Multiplatform, local-first).
- Handles authentication and authorization. Clients are never trusted for permission checks.
- Guarantees that a user only ever sees and changes their own family's data.

## Stack

- Language and runtime: C# on .NET 10.
- Framework: ASP.NET Core 10 with Minimal APIs. No MVC controllers.
- Database: PostgreSQL.
- ORM: Entity Framework Core 10 with the Npgsql provider (`Npgsql.EntityFrameworkCore.PostgreSQL`).
- Complex queries: Dapper over Npgsql.
- Database naming: snake_case via `EFCore.NamingConventions`.
- Identity: ASP.NET Core Identity for users and passwords, with JWT bearer tokens issued by this backend.
- Validation: ASP.NET Core 10 built-in Minimal API validation. No validation library.
- Primary keys: UUID version 7.
- Test framework: not decided. xUnit is the default choice unless decided otherwise.
- API documentation: built-in OpenAPI support via `Microsoft.AspNetCore.OpenApi`.

Do not add MediatR or AutoMapper. Both moved to commercial licenses. Use plain handler classes and hand-written mapping instead. Check the license of any new package before adding it, and prefer packages from Microsoft or with a permissive open source license.

## Project structure

The solution is split into one project per layer, each in its own folder. The `FamilyApp` prefix is the assumed solution name.

```
backend/
  src/
    FamilyApp.Domain/          entities, enums, length constants, domain rules
    FamilyApp.Application/     use-case handlers, request and response DTOs, interfaces
    FamilyApp.Infrastructure/  DbContext, entity configurations, migrations, Dapper queries, Identity, JWT issuing
    FamilyApp.Api/             Program.cs, Minimal API endpoints, auth and OpenAPI setup
  tests/
    FamilyApp.UnitTests/
    FamilyApp.IntegrationTests/
```

- **Dependencies point inward only.** Domain references nothing. Application references Domain. Infrastructure references Application and Domain. Api references Application, and Infrastructure only to register services at startup.
- **Domain stays framework-free.** No EF Core, ASP.NET Core or Dapper references in Domain. The Fluent API rule below keeps entities clean.
- **Inside each project, organize by feature folder**, such as `Families/` or `Members/`, using the same feature names in every project.
- **Endpoints never touch the `DbContext` or Dapper directly.** They call an Application handler. Handlers use interfaces that Infrastructure implements.
- **Each project registers its own services** in one extension method, such as `AddApplication()` and `AddInfrastructure()`, called from `Program.cs`.

## Commands

Run from `backend/`. Confirm these once the solution is scaffolded.

- Restore: `dotnet restore`
- Run locally: `dotnet run --project src/FamilyApp.Api`
- Watch mode: `dotnet watch --project src/FamilyApp.Api`
- Build: `dotnet build`
- Test: `dotnet test`
- Format: `dotnet format`
- Add migration: `dotnet ef migrations add <Name> --project src/FamilyApp.Infrastructure --startup-project src/FamilyApp.Api`
- Apply migrations: `dotnet ef database update --project src/FamilyApp.Infrastructure --startup-project src/FamilyApp.Api`

## Conventions

- Enable nullable reference types and treat warnings as errors.
- Make every class `sealed` by default, including entities, DTOs, handlers, services and configurations. Leave a class unsealed only when it is designed as a base class, and mark it `abstract` where that fits. Static classes are already sealed. Mock through interfaces in tests rather than unsealing classes.
- Write request and response DTOs as `sealed record` types.
- Use `async`/`await` end to end and pass `CancellationToken` through to I/O calls.
- Return `ProblemDetails` for errors so both clients can handle them the same way.
- Never expose database entities from endpoints. Map to DTOs by hand.
- Bind configuration with the options pattern instead of reading `IConfiguration` directly in services.

## Endpoints

- **Group endpoints per feature** with `MapGroup`, in a static class per feature such as `FamilyEndpoints` with a `MapFamilyEndpoints()` extension method.
- **Keep endpoints thin.** An endpoint binds the request, calls one Application handler and turns the result into an HTTP response.
- **Use `TypedResults`** and declare the possible results in the return type, such as `Results<Ok<FamilyResponse>, NotFound>`, so the OpenAPI document is accurate.
- **Family data lives under a family-scoped route.** Endpoints that read or change one family's data sit in a group under `/api/families/{familyId}`. The group has a membership endpoint filter that runs before every endpoint in it. Endpoints about the user across families, such as listing their own families, sit under `/api/me`.
- **Require authorization by default.** Call `RequireAuthorization()` on every group. Mark the few public endpoints, such as login, register and token refresh, with `AllowAnonymous()` explicitly.

## Validation

- **Use the built-in validation.** Register it with `builder.Services.AddValidation()`. Invalid requests get a 400 validation problem response automatically.
- **Put validation attributes on request DTOs**, using the length constants from Domain, such as `[MaxLength(FamilyMember.NameMaxLength)]`. Implement `IValidatableObject` on the DTO for rules that compare several fields.
- **Rules that need the database**, such as uniqueness or existence checks, live in the Application handler and return `ProblemDetails`.

## Authentication

ASP.NET Core Identity handles users. The backend issues its own JWTs. Never hand-write password hashing, token signing or other crypto.

- **Users are Identity users with GUID keys.** The user entity is `sealed class User : IdentityUser<Guid>`. The `DbContext` derives from `IdentityDbContext<User, IdentityRole<Guid>, Guid>`, and its `OnModelCreating` calls `base.OnModelCreating(modelBuilder)` first.
- **Use Identity's managers** for registration, password checks, lockout, email confirmation and password reset.
- **Do not use `MapIdentityApi`.** Its bearer tokens are an opaque Identity format, not JWTs.
- **Issue JWT access tokens** from one token service in Infrastructure, using `Microsoft.IdentityModel.JsonWebTokens`. Access tokens are short-lived, about 15 minutes. They carry the user ID as `sub` and roles. They do not carry family IDs, because memberships change, and a token would keep a removed member's access until it expires.
- **Use refresh tokens for long sessions.** They are random values, stored only as a hash in the database, and rotated on every use. If a used refresh token is presented again, revoke that whole token chain.
- **Validate everything on incoming tokens:** issuer, audience, lifetime and signing key.
- **The signing key is a secret.** It lives in user-secrets locally and an environment variable in deployment.

## Family data isolation

A user can belong to several families at once, such as the family they grew up in and the family they started. Memberships also end, for example after a divorce. Every family-owned row is still filtered by `family_id` automatically, and each request acts for exactly one family. This is the most important security rule in the backend.

### Members and accounts

A family member and a user account are different things. Many members never log in, such as young children, grandparents or pets.

- **A `FamilyMember` is a person or pet in one family.** It holds the member's kind, display name, relationship in that family, a join date and an optional end date. It is family data, so it implements `IFamilyOwned`.
- **The member's kind is an enum**, such as `Person` or `Pet`, so pets can be added without a new entity. Pets never have an account.
- **A member may be linked to a user account** through a nullable `UserId`. Only linked members can log in and act for the family.
- **A user's membership in a family is an active `FamilyMember` row linked to their account.** Everywhere this file says "membership", it means that.
- **The same real person can be a member of several families.** Each family gets its own `FamilyMember` row, and each row can be linked to the same account. You can be a child in your parents' family and a parent in your own, so the relationship is per row, never on the user.
- **A child of separated parents can be in both families**, as one row in each. When that child later creates an account, link both rows to it.
- **Link accounts to existing members; never duplicate them.** When a grandparent or a grown-up child signs up, they claim their existing member row through an invitation from the family. Do not create a second row for them.
- **At most one active member row per user in each family.** Enforce it with a partial unique index on `family_id` and `user_id` where `user_id` is set and the end date is empty.
- **A member is active while it has no end date.** Ending a membership, for example after a divorce, sets the end date instead of deleting the row, so the history stays.
- **Family data belongs to the family, not to the member who created it.** When a member leaves, the data they added stays with the family.
- **Collect little about members without accounts.** Children's data is especially sensitive. Store only what a feature needs, and do not ask for email or phone numbers for members who will not log in.
- **Relationship and permissions are separate.** The relationship, such as parent, child or grandparent, describes the family. What a linked user is allowed to do in the app is a separate permission level, which is not decided yet.

### Choosing the current family

- **The family picker is a client concept.** After login, and from the profile menu, the user chooses which family they are managing. The clients then send every family request to that family's route. The backend never stores an "active family" in the token or session, so switching families needs no new token.
- **The current family comes from the route**, never from the token. The membership endpoint filter on the `/api/families/{familyId}` group reads `familyId` and checks that the current user has an active membership in that family.
- **Check membership on every request**, from the database. Do not cache the result in the token. A short in-memory cache of a minute or less is acceptable if it is cleared when a membership changes.
- **Return 404 when there is no active membership**, the same as for a family that doesn't exist, so nobody can probe which families exist.
- **On success, the filter stores the family ID in a scoped `ICurrentFamily` service.** The `DbContext` property `CurrentFamilyId` reads from that service.

### The query filter

- **Mark owned entities.** Every entity that belongs to a family implements the Domain interface `IFamilyOwned`, with a `Guid FamilyId` property stored as `family_id`.
- **Apply the filter to every owned entity.** In `OnModelCreating`, after applying configurations, loop over all entity types implementing `IFamilyOwned` and add a named query filter called `family`. The filter compares `FamilyId` with the `CurrentFamilyId` property on the `DbContext`. It must reference that `DbContext` property, not a captured value, so EF Core reads it on every query.
- **The filter fails closed.** Outside a family-scoped route, `CurrentFamilyId` is null and the filter matches nothing. It never falls back to showing everything.
- **The membership check is a deliberate filter bypass.** It runs before a current family is known, so it queries `FamilyMember` with `IgnoreQueryFilters(["family"])`, limited explicitly to the current user's ID, the requested family and active rows. Listing the user's own families under `/api/me` works the same way. The `Family` entity itself is not `IFamilyOwned`.
- **Set and check `FamilyId` on save.** A `SaveChanges` interceptor sets `FamilyId` on new owned entities from `ICurrentFamily`. It throws if any added or modified entity has a different family, or if there is no current family.

### Exceptions and other access paths

- **Cross-family views are explicit.** A view across all the user's families, such as every upcoming event, lives under `/api/me`. It uses `IgnoreQueryFilters(["family"])` plus an explicit condition limiting rows to families where the user has an active membership. Add a comment saying why.
- **Bypass the filter only on purpose.** Besides cross-family views, use `IgnoreQueryFilters(["family"])` only in admin features or background jobs, with a comment saying why. Never ignore all filters.
- **Dapper bypasses query filters.** Every Dapper query that touches a family-owned table must include `family_id = @FamilyId` for each such table, including joined ones. Cross-family Dapper queries join active memberships for the user instead.
- **Index `family_id`** on every owned table, usually as the first column of composite indexes. Index `family_members` by `user_id` for the membership check.

### Tests

- **Every family-scoped endpoint gets an integration test** proving that a user who is not an active member of that family gets 404, both for reading and for changing data.
- **Test the ended-membership case.** A user whose membership ended loses access on their next request.
- **Test a user in two families.** Data from one family never appears when acting for the other.

## Health data encryption

Health data is medical data and is stored only in encrypted form. Never store it in plaintext, not even temporarily.

Health data is anything about a member's body, health or care, including:

- body measurements: height, weight, head circumference and other body sizes
- body composition: fat, muscle, water and bone percentages and similar values
- the baby feeding journal: breastfeeding, bottle and solid food entries
- sleep records
- medical examinations, doctor visits, diagnoses, test and lab results
- vaccinations, medications, allergies, illnesses and symptoms
- notes and files attached to any of the above

This list is not complete. Any new feature that records something about a member's body, health or care is health data. When unsure, treat it as health data.

Disk and backup encryption on the database server is still required, but it is not enough on its own: anyone with database access, a dump or a leaked backup would read plaintext. So the backend encrypts health data itself.

### What is encrypted

- **Everything except the bookkeeping columns is encrypted.** All health records live in one `health_records` table, so not even the table name shows what kind of record a row is. A row keeps only `id`, `family_id`, `member_id`, `created_at` and `updated_at` in plaintext, because family isolation, per-member loading and sync need them. The record kind, the time the event happened, and every value, unit, amount, note and detail go into one encrypted payload column.
- **One payload per row.** Serialize the record as a `sealed record` to JSON, encrypt it, and store the result in a `bytea` column named `payload`. The JSON holds the record kind, which says which payload type it is, and a payload schema version, so the shape can change later without a migration for old rows.
- **No SQL over health data.** PostgreSQL cannot filter, sort or aggregate encrypted values, and it cannot even tell feedings from sleep records. Load a member's rows, decrypt them in the handler, then filter by kind and time range and compute charts, averages and trends in memory. Sync loads rows by `updated_at`. Dapper is not useful for health data.
- **Files are encrypted too.** Attachments such as examination reports, lab results or scans are encrypted with the same family key before they are written to file storage, which is not decided yet. File names and content types are part of the encrypted data. Storage object names are random IDs.
- **Never put health data anywhere else in plaintext**: not in other columns, indexes, search tables, caches, logs, exception messages, analytics, notifications, emails or OpenAPI examples with real data.

### How it is encrypted

- **Envelope encryption with one data key per family.** Each family gets a random 256-bit data encryption key (DEK). The DEK is stored only wrapped, that is encrypted, by a master key encryption key (KEK), in a `family_keys` table with a key version. A per-family key keeps one family's data unreadable with another family's key, and deleting a family's key row makes all of that family's health data unrecoverable, including in old backups once those expire.
- **The KEK is a secret** and never sits in the database. Locally it lives in user-secrets; in deployment it comes from a key management service or an environment variable. The hosting provider and KMS are not decided yet. The KEK has an ID, so it can be rotated by rewrapping the DEKs without touching the health rows.
- **Cipher: AES-256-GCM** with .NET's `System.Security.Cryptography.AesGcm`, a fresh random 12-byte nonce for every encryption, and the 16-byte tag. Store the nonce, tag, DEK version and ciphertext together in the `payload` value in one documented byte layout.
- **Bind ciphertext to its row.** Pass the row's `id`, `family_id` and `member_id` as associated data, so a payload copied into another row or another family fails to decrypt instead of showing wrong data.
- **Do not use ASP.NET Core Data Protection** for this. It is meant for short-lived payloads such as cookies and antiforgery tokens, not for data kept for years.
- **Never hand-roll crypto beyond this.** No custom cipher modes, no reused nonces, no keys derived from passwords or IDs, no encryption with the KEK directly.

### Where the code lives

- **Domain** entities store the ciphertext as a `byte[]` property and stay free of crypto code.
- **Application** defines an interface such as `IHealthDataProtector` with methods that turn a typed payload into bytes and back for a given family and row. Handlers call it explicitly on write and read. Do not hide encryption in an EF Core value converter, because converters cannot get the current family's key.
- **Infrastructure** implements it, together with the key store that loads, unwraps and briefly caches DEKs in memory. Never cache unwrapped keys outside process memory.
- **A decryption failure is an error, not empty data.** Log the row ID and key version, never the payload, and return a 500 `ProblemDetails`.

### Tests

- Round-trip each payload type through the protector.
- Prove that a payload moved to another row or another family fails to decrypt.
- Prove that the `payload` column in the database does not contain the plaintext values.
- Prove that rotating the KEK keeps old rows readable.

## Health data audit log

Every access to health data is recorded in an audit log: who viewed, created, changed, deleted or exported which health record, and when. GDPR treats health data as special category data. The log lets us prove the data is protected, find out which records were exposed after a breach or a hacked account, and tell a family who has seen a member's records.

### What is logged

- **Reads are logged as well as changes.** Log every action on health data: `view`, `create`, `update`, `delete`, `export` and `decrypt_failed`. Viewing a list or chart logs one entry per request, with the IDs of all records returned.
- **Every kind of access is logged**: family members through the API, sync from the mobile app, background jobs, and support or admin access. Staff access is the most important to log, and records the staff user and a reason.
- **One entry holds IDs and facts, never health data.** Columns: `id`, `occurred_at`, `actor_type` (user, staff or system), `actor_user_id`, `family_id`, `member_id`, `record_ids` as a `uuid[]`, `action`, `client` (web, mobile, sync or job), `request_id`, `ip_address`, and an optional `reason` for staff access. No payloads, values, notes, file names or record kinds, because the kind is encrypted on the record itself.
- **The table is `health_audit_log`.** It implements `IFamilyOwned`, so the family filter applies when a family reads its own log.

### How it is written

- **Handlers write the entry explicitly** through an Application interface such as `IHealthAuditLog`, next to the `IHealthDataProtector` call. Do not rely on middleware or EF Core interceptors to guess what was accessed.
- **Changes and their audit entry commit together.** Write the entry in the same transaction as the create, update or delete.
- **Fail closed.** If the audit entry cannot be written, the request fails and no health data is returned or changed.
- **Append-only.** The app's database role gets only `INSERT` and `SELECT` on `health_audit_log`, never `UPDATE` or `DELETE`. Set this in a migration. No endpoint or handler changes or deletes entries.
- **Fixed retention.** Entries are deleted after a fixed retention period of 3 years. Keep the period in configuration, bound with the options pattern, so it can change without a code change. Only a separate retention job deletes them, with its own database role, and only entries past retention. The audit log is personal data itself, since it holds user IDs and IP addresses.
- **Deleting a family keeps its audit log** until retention ends, because it may be needed to investigate what happened before the deletion.

### Tests

- Every endpoint and sync path that reads or changes health data writes the right audit entry, including reads.
- A request whose audit entry fails to write returns an error and changes nothing.
- An audit entry never contains health values. Check the stored row against the plaintext values used in the test.
- The app's database role cannot update or delete audit entries.

## IDs

The mobile app is local-first, so clients must be able to create records offline.

- **All primary keys are `Guid`**, stored as PostgreSQL `uuid`, and generated as UUID version 7 with `Guid.CreateVersion7()`. Version 7 IDs are time-ordered, which keeps indexes efficient.
- **The database never generates IDs.** Configure keys with `ValueGeneratedNever()`.
- **Create endpoints accept a client-supplied ID.** Retrying a create with the same ID returns the existing record instead of an error. If the ID already belongs to another family, return 404, so nothing leaks about other families' data.
- **Never expose sequential integer IDs.**

## Data access

EF Core is the default. Dapper is for queries that are hard or slow to express in LINQ.

- **Use EF Core for** all writes, simple and moderate reads, and anything that needs change tracking.
- **Use Dapper for** complex reads such as reporting, aggregations, recursive CTEs, window functions, bulk reads, or any query where the LINQ translation is unclear or slow.
- **Writes go through EF Core** so validation, concurrency checks, family checks and audit logic stay in one place. Use Dapper for a write only for a measured bulk operation, and say why in a comment.
- **EF Core owns the schema.** Every table, column and index change goes through an EF Core migration, including ones only Dapper queries use.
- **Share the connection.** Dapper queries get their connection from the `DbContext` with `Database.GetDbConnection()`. Inside a transaction, pass the EF Core transaction from `Database.CurrentTransaction.GetDbTransaction()` so both run in it.
- **Keep Dapper SQL in Infrastructure query classes** per feature, such as `FamilyQueries`, behind an Application interface. Map results to dedicated read models, not EF Core entities.
- **Parameterize every Dapper query.** Never build SQL by concatenating or interpolating user input.
- **Lazy loading is banned.** Do not add the `Microsoft.EntityFrameworkCore.Proxies` package, call `UseLazyLoadingProxies()`, or inject `ILazyLoader` or a lazy-loading delegate into entities. It hides N+1 queries, blocks threads with synchronous database calls, and lets serialization trigger unplanned queries.
- **Load related data with `Select` projections by default.** Read handlers project straight into response DTOs with `Select`, so EF Core fetches only the needed columns and tracks nothing.
- **Use `Include` only when you need full entities to modify.** When a query includes more than one collection, add `AsSplitQuery()` to avoid a row explosion from joined collections.
- **Read-only queries that return entities** use `AsNoTracking()`. Projections don't need it.
- **Configure entities with the Fluent API only.** Do not put data annotation attributes on entity classes, such as `[Key]`, `[Required]`, `[MaxLength]`, `[Table]`, `[Column]`, `[Index]` or `[ForeignKey]`. Keys, required fields, lengths, indexes, relationships and constraints all go in Fluent API configuration. Validation attributes on request DTOs are still fine, since DTOs are not entities.
- **One configuration class per entity.** Each entity gets a `sealed` class implementing `IEntityTypeConfiguration<TEntity>`, named after the entity, such as `FamilyMemberConfiguration`. It lives in Infrastructure, in the feature folder matching its entity's folder in Domain.
- **Give bounded strings a max length.** Every string property gets an explicit decision in its configuration. Values with a natural limit, such as names, emails, phone numbers, codes and titles, get `HasMaxLength()`. Free text such as notes or descriptions may stay unbounded, with a short comment saying it is intentional. Keep each limit in a `const` on the Domain entity, such as `FamilyMember.NameMaxLength`, and use the same constant in request DTO validation so the API and the database agree.
- **Apply configurations by reflection.** `OnModelCreating` calls `modelBuilder.ApplyConfigurationsFromAssembly(typeof(AppDbContext).Assembly)` and never lists configurations one by one. Do not configure individual entities inline in `OnModelCreating`. The only code there besides that call is the Identity base call and cross-cutting loops such as the family filter.
- **All database names are snake_case.** Enable it with `UseSnakeCaseNamingConvention()` on the `DbContext` options, so a `DbSet<FamilyMember> FamilyMembers` maps to the table `family_members` and a `CreatedAt` property to the column `created_at`. Identity's tables get snake_case names too. Do not override names with `ToTable()` or `HasColumnName()` unless there is a specific reason.
- **Write Dapper SQL in plain snake_case** with no quoted identifiers. Alias columns only when a read model property doesn't match after Dapper's underscore mapping.
- **Turn on Dapper's underscore mapping** once at startup with `DefaultTypeMap.MatchNamesWithUnderscores = true`, so `created_at` maps to `CreatedAt`.

## Rules

- The API contract lives here. The OpenAPI document is the source of truth. Breaking changes are cross-domain work that must update both clients.
- Validate all input at the API boundary.
- Schema changes go through migrations, never manual edits to a live database.
- The PostgreSQL connection string is a secret. It lives in user-secrets locally under `ConnectionStrings:Default`, and in an environment variable in deployment.
- Configuration comes from `appsettings.json` plus environment variables. Secrets go in user-secrets locally and environment variables in deployment, never in `appsettings*.json`.
- Family data is personal data. Do not log request bodies, tokens or personal fields.
