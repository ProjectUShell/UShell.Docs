---
name: ushell-data-auth-scope
description: U-Shell data access, authentication and ambient-context propagation (target state plus current status and refactoring) - central declaration of datastores/datasources/services, FUSE-based generic data access, token sources and seamless token refresh instead of error messages, the 401-renew-retry rule, and propagation of ApplicationScope to backends through UJMW side channels. Load for work on DatasourceManager, FuseDataStore, ServiceFactory/createServiceProxy, TokenService, login flows or anything that calls a backend from the shell or a widget. Requires ushell-architecture.
---

# U-Shell data, authentication and ApplicationScope propagation

Declarations live in `UShell.PortfolioDescription`. Behavior lives in
`ReactFrontend` (`datasource-handling/`, `authentication/`) and `CommonComponents.JS`
(`data/`). Adjacent technologies with their own skills: FUSE-fx RepositoryContract,
UnifiedJsonMessageWrapper (UJMW), AuthTokenHandling, SmartAmbience.

**Token semantics are owned by the skill `auth-token-handling`** in
`SmartStandards/AuthTokenHandling` (`.agents/skills/auth-token-handling/SKILL.md`). Load it for any
work on token sources, login flows, token renewal or 401 handling. This skill only defines how
the shell *uses* those semantics. Most relevant sections there: §1.3 (lightweight
implementation in another stack, which is what the shell's `TokenService` is), §6
(`AuthTokenConfig` as portable strategy description, §6.4 cross-stack rule), §8.4 and §32.2
(refresh tokens), §12 (fail-closed), §13 (401 vs. 403), §14 (scopes), §20 (UJMW integration),
§27 (security rules, e.g. never log raw tokens).

Snapshot for status blocks: 2026-10-19, ReactFrontend `c71aa36`, CommonComponents.JS
`47f1629`, PortfolioDescription `7bf83f4`.

## 1. Central declaration of data and services

All backend access a module needs is **declared in the module description**, not in widget
code:

| Declaration | Purpose | Key attributes |
|---|---|---|
| `DatastoreDescription` | a repository endpoint serving many entities (one schema) | `Key`, `ProviderClass` (`fuse`, `localstore`, ...), `ProviderArguments` (url, routePattern, entitySchemaUrl, tokenSourceUid, additionalBodyArgs, additionalHeaderArgs, ...) |
| `DatasourceDescription` | one addressable entity source | `DatasourceUid`, `ProviderClass`, `ProviderArguments`, `EntityName` |
| `ServiceDescription` | a named service proxy (UJMW) | `ServiceUid`, `ServiceName`, `ProviderClass`, `ProviderArguments` |

Rules:
- Provider arguments are dynamic parameter objects. They may use `mapDynamic` (`setting://`, `applicationScope://`, ...).
- A declaration references its token source by `tokenSourceUid`. Widgets never pass tokens.
- Widgets obtain data through the WidgetHost (`getDataSourceForEntity`, `getDataSource(uid)`, `getServiceByName`). Generic components work purely on the FUSE `IDataSource` contract (CRUD, paging, sorting, filtering via `LogicalExpression`, entity refs, schema).
- New provider classes are generic and live in CommonComponents (or a module bundle that registers them), never inside a business widget.
- Entity references (ID -> display label, used for foreign keys and selection lists) come from `getEntityRefs`. Generic components resolve labels through it instead of loading full records.

### Status quo
- `DatasourceManager` creates datastores for `fuse` (`FuseDataStore` from CommonComponents) and `localstore`, then resolves datasources by entity name.
- `DatasourceDescription` (by UID) and `ServiceDescription` are declared in the contract but `getDataSource(uid)` and `getServiceByName` are not implemented.
- `CustomDataSource` / `CustomDataStore` exist for module-provided providers.

### Refactoring & compatibility
- UB-21: resolve datasources by UID and services by name. Additive.
- UB-51: define the registration path for provider classes contributed by module bundles (see `ushell-module-development`).

## 2. Token sources

Authentication is configured per **token source** (`AuthTokenConfigs` keyed by
`tokenSourceUid`, semantics from AuthTokenHandling / `epics/authTokensConfig.md`): how a
token is **issued** (OAuth code/popup/iframe/embedded, HTTP-GET, raw input, local generation,
...) and how it is **validated** (local JWT, introspection, ...).
`PrimaryUiTokenSourceUid` represents the application login. Other token sources are acquired
on demand when a datasource or service needs them. Claims/scopes may be mapped to RuntimeTags
(`AuthenticatedAccess.RuntimeTagsFromTokenScope`).

The shell is the only component that issues, stores, renews and deletes tokens.

Relation to `auth-token-handling`:
- The shell's token service is a **Level C** consumer (`auth-token-handling` §1.3). It parses the shared `AuthTokenConfig` shape with the same field names, mode names and semantics; it supports a subset of modes, **rejects unsupported modes explicitly** and never silently reinterprets a known mode (§6.4).
- New issue modes or config fields are proposed to AuthTokenHandling first, because `AuthTokenConfig` is a cross-stack interoperability boundary (§32.4). U-Shell does not extend it locally.
- Fail-closed (§12): a technical failure while obtaining or renewing a token never results in an unauthenticated call that pretends to be authenticated.

## 3. Seamless token refresh (target)

The user must never see a backend error just because a token expired.

1. **Proactive renewal.** The token service knows each token's expiry (`exp` or configuration). Shortly before expiry (configurable skew) it renews in the background using the configured issue mode: refresh token, silent iframe/redirect for OAuth, or re-retrieval for HTTP-GET sources.
2. **Single flight.** Concurrent requests for the same token source share one renewal.
3. **Asynchronous token access everywhere.** Every consumer gets the token through an async provider (`getAccessToken(tokenSourceUid)`), never by reading storage synchronously.
4. **401 rule.** A call answered with HTTP 401 (or the equivalent UJMW fault) triggers exactly **one** renewal of that token source and **one** retry of the original call. A second 401 is not retried. **403 is never retried**: it means a known token lacks authorization, and renewal cannot fix that (`auth-token-handling` §13).
5. **Interactive fallback.** If silent renewal is impossible, the shell opens the login for that token source (overlay). The pending calls wait and resume after a successful login, or fail with a defined "authentication cancelled" result if the user aborts.
6. **No credential storage.** Never persist passwords or password hashes for auto-login; persist only what the token protocol intends (tokens, refresh tokens) with the configured storage policy.

Boundary to `auth-token-handling` §8.4 / §32.2: the refresh-token model there is
deliberately not refined yet ("agents MUST NOT invent broader refresh-token propagation
semantics"). The renewal above is therefore **client-side lifecycle only**: it uses the issue
modes of the token source's `AuthTokenConfig` as defined today and uses a `refresh_token` only
where the configured mode already defines that. A client-side refresh model (timing, storage,
rotation, client vs. server responsibilities) that U-Shell needs is to be contributed to
AuthTokenHandling §32.2, not defined here (D-11).

Lesson from the legacy generation (CtExpert `w3bstract.ServiceBase`): a central
`AutoLoginMethod` hook with "retry once, never loop" worked well and is the pattern to keep.
Its credential storage (MD5 password hash in `localStorage`) is the anti-pattern to drop.

### Status quo
- `TokenService.checkTokenExpired` **deletes** expired tokens (`// TODO refresh token?`).
- `getTokenAndContent` and therefore `IWidgetHost.getAccessToken` return the stored token without checking expiry or renewing.
- `FuseDataStore.getTokenMethod` and the token header factory in `WidgetHost.createServiceProxy` are **synchronous** reads.
- `createServiceProxy` (CommonComponents `ServiceFactory`) maps every non-OK response to `{ return: null, fault: statusText }`. There is no 401 detection and no retry.
- Login flows exist: popup, iframe, embedded OAuth, logon overlay/page, `ProtectedRoute`.

### Refactoring & compatibility
- UB-20, in this order:
  1. Async token provider in ReactFrontend with expiry check, renewal and single flight.
  2. Optional async token callback in CommonComponents (`FuseDataStore`, `createServiceProxy`) **next to** the existing sync callback; the sync path stays for existing consumers.
  3. 401 detection + renew + single retry inside the CommonComponents transports, driven by a shell-provided `onUnauthorized(tokenSourceUid)` callback.
  4. Login overlay with resumable pending calls.
- The response shape `{ return, fault }` of the service proxy must not change.

## 4. ApplicationScope propagation to backends (target)

ApplicationScope values are **ambient context**. They travel to backends through the UJMW
side channel and are restored there into SmartAmbience. They are never added to business
method signatures unless they are genuinely part of the business contract.

```text
ApplicationScope (current values)
   -> every UJMW call made by shell transports (FUSE datastores, service proxies)
      -> request side channel "_": { "<dimension>": "<value>", ... }   (body, default)
         or configured header side channel                              (alternative)
            -> UJMW server -> SmartAmbience -> backend logic
```

Rules:
- The **value** (the key from `KnownValues`, an ID) is transmitted, not the display label. Object-valued scopes transmit their identity value.
- Values are read **per call**, so a scope switch takes effect immediately without re-creating datastores.
- The transport (body `_` or header, header name) is configurable per datastore/service and defaults to the UJMW body side channel.
- Which dimensions are sent can be restricted per declaration (default: all).
- Response side channel data is not interpreted as a scope change. ApplicationScope changes only through the shell.

### Status quo
- `DatasourceManager.tryCreateDataStore` adds `_` to `additionalBodyArgs` for `fuse` stores **once at creation**. Each dimension's value is flattened by `pickAppScopeValueLabel`, which sends the first `*label*` property of object values (`TODO_KRN`: objects not supported as side channel values).
- After a scope switch the runtime re-creates all datastores (`onAppScopeChangedMethod`), which is how the new values reach the backend today.
- `WidgetHost.createServiceProxy` sends `_` with the raw `initialValue` of each dimension per call (objects not flattened). The two transports are inconsistent.
- No header variant, no per-declaration filtering.

### Refactoring & compatibility
- UB-22: one shared side-channel builder used by all transports, evaluated per call, transmitting values. Switching label to value is a **behavior change for existing backends**. Make it opt-in per declaration first (e.g. a provider argument), then change the default with a version note.
- Keep the re-initialization on scope switch until all transports read per call.
