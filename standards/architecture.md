# Architecture: contract-first APIs, data and codebase structure

Tiers: **[MUST]** non-negotiable · **[SHOULD]** strong default · **[EXTREME]** for money, identity, health or family data. Part of [applied-magic](../README.md).

One schema is the truth: the API contract generates the spec, the clients and the docs, so frontends, partners and agents can't drift from the backend.

## Contract-first APIs
- [MUST] Declare every API surface as code-level contracts (route + method + input schema + output schema + declared errors) in one shared contracts package. The schema is the single source of truth. Server handlers, the OpenAPI spec, client SDKs and docs are all derived from it and never written by hand.
- [MUST] Bind server handlers to the contract so the compiler rejects drift: a handler's input type and return type are inferred from the contract, and a handler with no contract entry cannot exist.
- [MUST] Split contracts by audience (public/partner, first-party internal, admin, session/embedded). Each surface has its own entry point, auth scheme and generated spec. Never mix audiences in one router. An internal convenience must not leak into the public spec.
- [MUST] Route definitions hold only path, method, operationId, tags, `.input()`, `.output()` and `.errors()`. Domain shapes live in per-domain **entity schema files** and are imported. Extract an inline object into a named schema when it has more than 3 fields, when a nested object has more than 3 fields, or when another endpoint reuses it. Tiny inputs (`{ id }`, `{ page, limit }`) may stay inline.
- [MUST] Re-export entity schemas through one barrel so every contract surface uses the same named shapes. One name per concept everywhere.
- [MUST] Every schema field carries a description and an example, through one helper, so the generated spec is the documentation. A field with no description is not done.
- [MUST] The contracts package must not import server-only code (ORM, drivers) because clients bundle it. Mirror the few shared enum value lists as `as const` arrays and note why they are duplicated.
- [MUST] Give every operation a stable, explicit `operationId` and a tag. Generated client method names depend on them, so renaming one is a breaking change.
- [SHOULD] Register event/webhook payloads in a typed registry keyed by event type. Spec generation iterates the registry, so adding an event automatically documents it. Webhooks go through the same contract discipline as requests.

```ts
// entities/widget.ts: the shape lives with its domain
export const WidgetSchema = z.object({
  id: d(z.uuid(), "Widget ID", "0192f1c4-..."),
  externalId: d(z.string(), "Caller-defined identifier", "wid_123"),
  amount: d(z.number().int(), "Amount in minor units", 125000),
  expiresAt: d(z.iso.datetime().nullable(), "ISO 8601 UTC; null = never expires", "2027-01-01T00:00:00Z"),
});
// contracts/public.ts: routes only
getWidget: oc.route({ method: "GET", path: "/v2/widgets/{id}", operationId: "getWidgetV2", tags: ["Widget"] })
  .input(z.object({ id: z.uuid() })).output(WidgetSchema).errors(commonErrors),
```

## Generation pipeline
- [MUST] Use one command (`<pm> openapi:generate`) to build the contracts, emit one OpenAPI document per surface and then generate typed clients from those documents. Run it after every contract edit, in the same commit.
- [MUST] Commit the generated specs. They are the reviewable artifact: a PR that changes the API shows a spec diff, and CI diffs it against the base branch.
- [MUST] CI posts a machine-generated API changelog (spec diff of base vs head, per surface) on every PR that touches generated specs. Breaking changes (removed path/field, narrowed type, new required input, changed operationId) MUST fail the check unless the PR is explicitly labelled as an approved breaking change.
- [MUST] CI fails when the regenerated spec differs from the committed one (stale-spec guard).
- [SHOULD] First-party TypeScript frontends consume the contract object directly, through a typed RPC/OpenAPI link plus query-library helpers. That gives end-to-end inference with no codegen step. Use generated SDKs (from the emitted spec) only for consumers outside the monorepo or for non-TS stacks.
- [SHOULD] Publish the spec per environment (servers list for sandbox/live) to a docs renderer on merge to main. Docs are never hand-edited.

```ts
// frontend: one typed client per surface, created once
// (oRPC names; other contract libraries have the same three pieces)
const link = new OpenAPILink(internalContract, { url: env.API_URL, fetch: (u, i) => fetch(u, { ...i, credentials: "include" }) });
export const client: ContractRouterClient<typeof internalContract> = createORPCClient(link);
export const api = createTanstackQueryUtils(client); // api.widget.get.queryOptions({ input: { id } })
```

## Evolution and versioning
- [MUST] Changes to anything outside callers depend on (public/partner routes, webhook payloads, stored data, mobile apps already installed) are additive: optional fields, new endpoints, new enum values only where clients tolerate them. Never delete, rename or repurpose such a route or field in place; add a parameter or a new version. Internal code the compiler sees every caller of may be renamed freely.
- [MUST] Put a version in the path (`/v2/...`) and in the operationId (`...V2`) of a breaking redesign. The old version keeps running until its consumers are migrated. Deprecate in the spec first (`deprecated: true` plus a sunset note) and remove in a separate, announced change.
- [MUST] Be a tolerant reader at the edge. When an input format must change, accept the legacy form, normalize it in the schema transform to the canonical form and document both. Internals only ever see the canonical form.
- [SHOULD] New request fields are optional with a documented server default ("omitted → organization default"). A required addition is a breaking change.

## Errors
- [MUST] Use one typed error type across the backend (code + message + optional typed `data`) that the transport serializes into one envelope. Never throw bare `Error`, framework HTTP exceptions or ad-hoc JSON. They bypass serialization and arrive untyped.
- [MUST] Use a closed code set mapped to HTTP status: `BAD_REQUEST` 400, `UNAUTHORIZED` 401, `FORBIDDEN` 403, `NOT_FOUND` 404, `CONFLICT` 409, `RATE_LIMITED` 429, `INTERNAL_SERVER_ERROR` 500. Each surface declares its error map once (`commonErrors`) and attaches it to every route, so the errors appear in the spec.
- [MUST] Input validation failures are 400. A well-formed ID that resolves to no row, or to a row the caller does not own, is 404. Never 403 for "exists but not yours", because that leaks existence; 403 is only for a resource the caller can see but may not perform this action on.
- [MUST] Messages are safe for the caller and actionable. Internal causes go in `cause` (logged), never in `message`.

## Pagination, IDs, money, time
- [MUST] Every list endpoint takes one shared pagination input with a default and a hard cap on `limit` and returns one shared envelope; never an unbounded array. Admin tables may use `page` + `total`; feeds, syncs, exports and large tables use cursor (keyset) pagination on an indexed, unique sort key, since offsets skip or repeat rows as data changes and `COUNT(*)` gets slow.
- [MUST] Primary keys are server-generated, time-ordered UUIDs (v7); where creation time is itself sensitive, expose a v4 or prefixed random public ID instead, since v7 embeds a timestamp. Path IDs are validated in the schema, so malformed IDs are rejected before any handler runs.
- [MUST] Resources a client creates also carry a client-supplied `externalId`. Uniqueness is scoped to the owner, and a duplicate `externalId` returns `CONFLICT`. This gives idempotent creates and reported events with no separate key store. A retry with the same `externalId` is never a second write.
- [MUST] Money is an integer in minor units plus an explicit currency enum, both in the contract and in the database. Never use floats.
- [MUST] Timestamps on the wire are ISO 8601 in UTC (`Z`). Calendar dates use `YYYY-MM-DD` with no time or zone.
- [SHOULD] Polymorphic resources are discriminated unions on a literal `type` field (`BUSINESS` | `PERSON`), both for input and output.

## Null and optional policy
- [MUST] On the wire, `optional` means the caller may omit the field (with a documented default) and `nullable` means the value is known to be absent. Every nullable output field documents what null means ("null = never underwritten", "null = no expiry"). Outputs always include every key: no omitted keys, only null.
- [MUST] Never pass positional `null` (`find(null, customer)`) in any layer. If the caller genuinely has nothing, make the parameter optional. If both values are needed and one may be absent, take a named options object: `find({ org: null, customer })`.
- [SHOULD] Prefer explicit, destructured null handling over long optional chains that hide which link can actually be null.

## Backend layering
- [MUST] Use these layers: **contract** (shape) → **controller** (auth gate + delegation only) → **service** (orchestration and business rules) → **repository** (every query) → **schema** (tables). Each layer imports only the layer below. A small app may merge service and repository; SQL never sits in a controller or a UI route.
- [MUST] Every ORM/SQL call lives in a repository as a named, domain-worded method (`findActiveByOwner`). A service that imports the DB handle or a table object is in the wrong layer.
- [MUST] Entity types (`Widget`, `CreateWidget` without id, `UpdateWidget` with all fields optional) live in one entity file per domain and are reused by repo and service. A service defines its own input/output types (in a sibling `service.interface` file) only when it reshapes data or orchestrates several repositories.
- [MUST] Controller inputs and outputs are the contract types. Derive return types from the contract (`Outputs["widget"]["get"]`) so they cannot drift.
- [MUST] An aggregate or "view" service exists only to combine two or more entity services into one response. Single-entity reads and all mutations go controller → entity service directly. Never mirror the domain tree with prefixed wrappers (`AdminWidgetService`).
- [MUST] Each third-party provider gets exactly one module: a client that owns config (read once, in its constructor), auth/token refresh, base URL and retries; optional service and mapper; provider types that never cross the module boundary. Consumers import the module. Domain code makes no inline SDK or HTTP calls.
- [MUST] Avoid N+1: batch related lookups (one `IN (...)` or join, then a map), run independent queries with `Promise.all`, select only the columns the contract needs, and flag any new filter or sort column that needs an index.

## Data layer (Postgres + typed ORM)
- [MUST] The schema lives in one database package as typed table definitions. That package is the only owner of tables and migrations. Apps import it and never declare tables of their own.
- [MUST] Migrations exist from the first table: generated from schema diffs, timestamp-prefixed, committed, and applied by one command that the deploy runs before new code serves traffic (a deploy that skips it ships code against the old schema), failing the deploy if it fails. Direct schema push is allowed only on a throwaway local DB, never unattended (it can stop on an interactive rename prompt).
- [MUST] The backfill a schema change needs is a hand-written SQL migration next to it, batched on large tables, with a comment stating why. Repairs of bad data are separate preview-first commands ([build.md](build.md), *Data integrity*), never migrations and never ad-hoc SQL.
- [MUST] A seed command builds an idempotent synthetic dataset covering every role and status for local dev and e2e; production data never seeds dev, and a dev database is never assumed to mirror production (check row counts before reusing IDs across them).
- [MUST] Table conventions: UUIDv7 PK via one helper; `created_at`/`updated_at` not-null with defaults and an on-update hook; snake_case columns mapped to camelCase fields; FKs declared with explicit `onUpdate`/`onDelete`; uniqueness and checks in the schema, not only in code.
- [MUST] CI applies all migrations to a fresh test DB before e2e tests run.
- [SHOULD] Store enums as `text` with a `CHECK` constraint plus a typed guard in the app, rather than native DB enums: removing or renaming a native enum value needs a type rebuild, while a `CHECK` changes in one plain migration.
- [SHOULD] New columns on live tables are nullable (with a comment on what null means for legacy rows) or take a constant default (instant on Postgres 11+); tighten to not-null after the backfill via `CHECK … NOT VALID` then `VALIDATE`, and build indexes `CONCURRENTLY`.

## Monorepo and package boundaries
- [MUST] Once there's more than one deployable or client, use a workspace monorepo with `apps/*` (deployables) and `packages/*` (shared libraries: contracts, database, ui, logger, config presets). Dependency direction runs app → package only. Packages never import apps, and frontend packages never import server packages.
- [MUST] The task graph declares that `build`/`typecheck`/`test` depend on upstream `^build`, so a consumer always compiles against freshly built contracts.
- [MUST] Packages expose explicit subpath exports per surface (`pkg/public`, `pkg/internal`, `pkg/admin`, `pkg/entities`) so a client can only import the surface it is allowed to use.
- [MUST] Use one formatter/linter config at the root (overridden per app only when needed), a pre-commit auto-fix, and the CI order lint → typecheck → build → unit → migrate test DB → e2e (API) → e2e (browser).
- [MUST] Configuration is a schema validated at boot (the process refuses to start on a missing or invalid var). It is read through one module, injected only into the provider that needs it and never read ad hoc from `process.env` (a lint enforces it). Ship a `.env.example` per app. Clients call the canonical API domain from config, never a platform-generated URL.
- [SHOULD] Run isolated sandbox and live modes side by side (separate DBs and config, a predictable port offset) so test data can never touch real data.
- [SHOULD] Track bundle size against a main-branch baseline and comment the delta on PRs.

## Testing the contract
- [MUST] API e2e tests boot the real app module against a freshly migrated and seeded test DB of their own (never a shared dev DB another run or agent is using), call it over HTTP and assert status plus body against the contract. Only true externals (payments, messaging, ledger, storage, key lookup) are replaced by mocks at the provider boundary.
- [MUST] Contract-edge tests cover every ID-taking route (malformed → 400, valid-but-absent → 404, not-owned → 404), duplicate `externalId` → 409, and auth missing/invalid → 401.
- [MUST] Every state or eligibility rule has a service spec with positive, negative and status-boundary cases (one per meaningful status group), with dependencies mocked by type.
- [MUST] When two clients implement the same wire format, signature or encryption (web and mobile), the protocol has a name and version, and both run one shared set of test vectors in CI so they can't drift byte by byte.
- [MUST] Use the replace-don't-layer rule: once a boundary test covers a behaviour, delete the shallow unit tests it supersedes. Tests assert observable outcomes through the public interface and survive refactors.

## UI conventions derived from the contract
- [MUST] Route files are thin page entries: fetch through the typed client, compose named components and hold only local dialog/navigation state. Tables, cards and forms are extracted components. Reuse the app's existing primitives and shared formatters before writing new ones.
- [MUST] Gate every action, badge and conditional section on entity state. Before building it, write down which statuses show it, what other field conditions must hold and what goes wrong if it shows in the wrong state. In invalid states render nothing, not a disabled control or an empty-state placeholder.
- [MUST] Register every new page under the authenticated layout. A route outside the auth layout is public.

## Architecture-improvement heuristics
- [MUST] Scope convention rules to new code. Do not opportunistically refactor pre-existing code to a new rule inside a feature PR. Cleanup is its own task.
- [MUST] Readability: full-word names that say what the thing is or does; one concept per file; a function does one thing (if it needs phase comments, split it; question anything over ~40 lines); comments explain why (business rule, external constraint, invariant), never what, and never carry names, dates or history, which belong in the PR.
- [SHOULD] Prefer **deep modules**: a small, stable interface hiding a large implementation. Deep modules are testable at the boundary and navigable by agents. The signal for a shallow one: understanding one concept means bouncing across many tiny files, the interface is nearly as complex as the implementation, or pure functions were extracted "for testability" while the bugs live in how they are called.
- [SHOULD] Classify each dependency before deepening: (1) in-process → merge and test directly; (2) local-substitutable (in-memory DB, fake FS) → test with the stand-in; (3) remote-but-owned → define a port, with an HTTP adapter in production and an in-memory adapter in tests; (4) true external → inject it as a port and mock it at the boundary.
- [SHOULD] When redesigning an interface, sketch 3+ deliberately different designs (minimal, maximally flexible, optimized for the commonest caller, ports-and-adapters), compare them in prose and recommend one with a reason in a short RFC: problem, chosen interface with usage, dependency strategy, tests to add/delete.

---
