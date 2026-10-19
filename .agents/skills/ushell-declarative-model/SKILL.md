---
name: ushell-declarative-model
description: U-Shell declarative description language (target state plus current status and required refactoring). Load when designing or changing portfolio.json/module.json content or the UShell.PortfolioDescription contract - workspaces, use cases, static assignments, commands and command types, use-case instance identity, appearance/modal, RuntimeTags, ApplicationScope definitions and the mapDynamic mapping semantics. Requires ushell-architecture.
---

# U-Shell declarative model

Bounded context: **declarative language**. Contract repository: `UShell.PortfolioDescription`
(.NET shared project + npm `ushell-portfoliodescription`). Runtime behavior is described in
`ushell-interaction-dynamics` and `ushell-shell-runtime`.

Snapshot used for all status blocks: 2026-10-19, PortfolioDescription `7bf83f4`,
ReactFrontend `c71aa36`.

## 1. Resource model

Portfolio and module descriptions are **plain HTTP GET resources** (`portfolio.json`,
`module.json`). They may be static files or generated dynamically (controller, database,
BFF). The browser cannot tell the difference. No RPC protocol may be required to obtain them.

A portfolio references modules (`ModuleDescriptionUrls`). Modules may be remote. A module is a
description, not a code bundle; it may reference remote widgets.

### Status quo
- Implemented as described. `PortfolioDescription.FromEmbeddedResFile`, `ModuleDescription.Build` and `PortfolioHosting.Mvc` support static and dynamic provisioning.

## 2. Workspaces

A workspace is a long-lived working context. It has a key, a title, an icon and an
**appearance** (`default`, `sidebar`, ...). It may declare `RequiredApplicationScopes`: the
names of ApplicationScope dimensions that **must have a value** before the workspace is usable.
If not satisfied, the shell offers the scope selection instead of the workspace content.

### Status quo
- `WorkspaceAppearance` (default `"default"`) and the legacy `IsSidebar` exist in the contract. ReactFrontend evaluates neither.
- ReactFrontend checks `RequiredApplicationScopes` **against RuntimeTags** (`intialRuntimeTags`) and shows "You do not have the required permissions". That mixes two concepts.

### Refactoring & compatibility
- UB-06: evaluate `WorkspaceAppearance`; treat `IsSidebar == true` as alias for appearance `sidebar`.
- UB-09: evaluate `RequiredApplicationScopes` against ApplicationScope values. Behavior change, so verify productive portfolios that might rely on the tag check. If any do, keep the tag check as fallback when no scope with that name is defined.

## 3. Use cases and static assignments

A use case description has a key, title (placeholders may resolve from the UoW), icon,
`WidgetClass` and `UnitOfWorkDefaults` (a dynamic parameter object, see section 7).

`WidgetClass` addresses:
- a built-in or CommonComponent widget by name (e.g. `guifad`, `guifadFuse`, `scopeSelection`, `login`, `debug`),
- a federated remote widget (JSON `{scope, module, url}`, see `UsecaseDescription.SetRemoteWidget`),
- in future a WebComponent (keep the contract open for it).

`StaticUsecaseAssignment` binds a use case permanently to a workspace with its own
`InitUnitOfWork`.

### Instance identity
Whether `start-usecase` creates a new instance or re-activates an existing one is decided by
an **instance key** computed from a pattern:

| Pattern | Effect |
|---|---|
| `{uid}` | always a new instance |
| `{usecaseKey}` | application-wide singleton |
| `Customer:EditOrCreate` | one instance for this purpose |
| `EditCustomer-{customerId}` | one instance per record (placeholder resolved from the new UoW) |

The productive field for the pattern is `UsecaseDescription.SingletonActionkey`.

### Status quo
- `SingletonActionkey` exists in the contract but is not evaluated.
- ReactFrontend re-uses an existing instance if the use case is static for that workspace, or if the new UoW is **shallowly equal** to an existing instance's UoW (`WorkspaceManager.areEqual`).

### Refactoring & compatibility
- UB-05: evaluate `SingletonActionkey` as an instance-key pattern. When empty, keep today's UoW-equality behavior unchanged.

## 4. Appearance of use cases (modal)

A use case can be shown inside its workspace (tab) or as a modal dialog. Modality is a
property of **how a use case is started**, with a default on the use case:

- A `start-usecase` command without `TargetWorkspaceKey` starts the use case modally over the current context. `"."` addresses the workspace of the source use case.
- A use case may declare a default appearance for the case where the command does not decide.

### Status quo
- ReactFrontend opens a use case modally when `UsecaseDescription.IconName == "modal"` (a hack). A command without `TargetWorkspaceKey` is not handled as modal.
- `CommandDescription.TargetWorkspacePath` (with magic value `<CURRENT>`) exists in the contract and is unused.

### Refactoring & compatibility
- UB-06: introduce an explicit, optional appearance for use cases and the "no target workspace = modal" rule. Keep `IconName == "modal"` as a deprecated alias. Clarify `TargetWorkspacePath` versus `TargetWorkspaceKey` and merge or deprecate one of them.

## 5. Commands

Commands are the only declarative way to trigger behavior. Shared attributes:

| Attribute | Semantic |
|---|---|
| `UniqueCommandKey` | identifier, also used by widgets to fire the command |
| `Label`, `Description`, `IconKey`, `IconKeyChecked`, `Semantic` | presentation |
| `WarningToConfirm` | shell shows a confirmation before execution |
| `RequiredRuntimeTagsForAvailability` | AND-linked tags, otherwise disabled |
| `RequiredRuntimeTagsForVisibility` | AND-linked tags, otherwise hidden |
| `CheckedRepresentingRuntimeTag` | checked state |
| `MenuOwnerUsecaseKey` | empty = global menu, otherwise toolbar of that use case |
| `MenuFolder` | `/`-separated path. `MY` = user menu. Empty root falls back to the misc folder for the global menu |
| `LocateAfterCommand`, `LocationPriority` | ordering |
| `CommandType` | see below |

Command types (target):

| CommandType | Purpose | Type-specific attributes |
|---|---|---|
| `activate-workspace` | navigate to a workspace, initializing static use cases | `TargetWorkspaceKey` |
| `start-usecase` | start or re-activate a use case; pipe values into its UoW | `TargetWorkspaceKey` (empty = modal), `TargetUsecaseKey`, `InitUnitOfWork` |
| `usecase-action` | invoke an action on the current use case: either a shell-known action (`endUsecase`, `commit`, `discard`, ...) or an action the widget registered | `ActionName`, arguments via dynamic parameter object |
| `server-command` | start an asynchronous server-side command (see `ushell-server-commands`) | endpoint/service reference, command name, arguments via dynamic parameter object |
| `set-runtime-tag` | modify RuntimeTags: `tag` sets, `!tag` toggles, `-tag` removes | `TagsToSet` |
| `switch-scope` | set an ApplicationScope dimension | `TargetScopeKey`, `TargetScopeValue` (literal or `mapDynamic` URI) |
| `navigate` | navigate to a router link or external URL | `RouterLink` |

Historical name: the 2019 concept called the server command `backend-action`. The shell
should accept it as an alias.

### Status quo
- ReactFrontend implements `activate-workspace`, `start-usecase`, `switch-scope`. `navigate` ignores `RouterLink`. `usecase-action`, `set-runtime-tag` and any server command type fall through to `throw "invalid command type"`.
- `RequiredRuntimeTagsForVisibility` is evaluated in the global menu only. The use-case `CommandBar` evaluates no tag at all.
- `WarningToConfirm`, `RequiredRuntimeTagsForAvailability`, `IconKeyChecked`, `LocateAfterCommand`, `LocationPriority` are not evaluated.
- `ActionName` exists in the contract; there are no attributes for server commands yet.

### Refactoring & compatibility
- UB-03 `usecase-action`, UB-04 `set-runtime-tag`, UB-13 `navigate`, UB-30 server command type, UB-10 command metadata (confirm, availability, checked, ordering) in menu **and** CommandBar.
- All additive: unknown types already throw today, so new types cannot break existing portfolios.
- Contract additions for server commands must be added to .NET and TS together.

## 6. RuntimeTags

RuntimeTags are boolean predicates with three sources:

1. `PortfolioDescription.IntialRuntimeTags` (sic, productive spelling),
2. tags derived from token claims/scopes (`AuthenticatedAccess.RuntimeTagsFromTokenScope`) - the declarative bridge from authentication to UI permissions,
3. `set-runtime-tag` commands and the Host API.

User-set tags (source 3) are persisted per portfolio. Token-derived tags are re-computed
whenever tokens change. RuntimeTags never carry values and are never used for ApplicationScope checks.

### Status quo
- Sources 1 and 2 work: token-derived tags are appended to the in-memory `intialRuntimeTags` array.
- Source 3 does not exist. There is no separate runtime tag store, so token-derived tags and initial tags cannot be told apart or removed.

### Refactoring & compatibility
- UB-04: introduce a runtime tag store in the shell, initialized from source 1, merged with 2, mutated by 3. Keep the portfolio field as the initial value only.

## 7. Dynamic parameter objects and `mapDynamic`

Wherever the contract carries an `IDynamicParamObject` (`UnitOfWorkDefaults`,
`InitUnitOfWork`, datasource/datastore provider arguments, command arguments), the object may
contain a `mapDynamic` array of `{ use, for }` entries.

Resolution:

```text
static/default values  +  mapDynamic entries
        -> resolve each "use" through its provider channel
        -> write the value to the "for" path (nested paths allowed)
        -> drop the mapDynamic array
        -> effective object for the target
```

Provider channels (target):

| Channel | Source |
|---|---|
| `unitOfWork://path` | UoW of the **source use case** that triggered the command |
| `commandArgs://path` | arguments passed with the command invocation (e.g. by a widget firing it) |
| `applicationScope://name` | current **value** of an ApplicationScope dimension. `applicationScope://name.label` and `.entry` give access to label or full entry |
| `setting://group.field` | well-known settings source |
| `userInput://name` | value asked from the user in a shell dialog (makes resolution asynchronous) |
| `factory://uuid` / `UUID` / `UID64` / `now` | generated values |

Rules:
- `mapDynamic` is **not** an expression language. New behavior means a new registered provider or factory, never `eval`.
- An unresolvable entry leaves the default value untouched and is reported through the shell logging facade.
- Resolution of `userInput://` may cancel the whole command if the user aborts.

Interaction semantics (when which source UoW is used, the return path) are in `ushell-interaction-dynamics`.

### Status quo (`ReactFrontend/ArgumentMapper.ts`)
- Implemented: `unitOfWork://`, `commandArgs://`, `factory://uuid|UUID|now`, `applicationScope://`.
- `applicationScope://name` returns the **whole `ApplicationScopeEntry` object**, not its value.
- `setting://` is commented out (no settings source). `userInput://` logs "under construction". `factory://UID64` is missing.
- Resolution is synchronous. Unresolved values are logged with `console.error`.

### Refactoring & compatibility
- UB-07 `userInput://` (requires async resolution path), UB-08 `setting://` plus settings source, UB-14 `factory://UID64`.
- UB-09 `applicationScope://`: the value-versus-entry change is **potentially breaking**. Before switching the default, search productive portfolios for `applicationScope://`. Safe path: keep the current return for the bare form, add `.value` / `.label` sub-paths first, then decide the default with a version marker.

## 8. ApplicationScope definitions

`PortfolioDescription.ApplicationScope` declares dimensions (`ApplicationScopeEntry`):
`Name`, `Label`, `InitialValue`, `KnownValues` (value -> label), `DependentScopeNames`,
`DependentScopeConstraints`, `IsVisible`, `SwitchScopeCommand`.

Target: the description holds the **definition and initial value** only. The **current
value** is runtime state of the shell, persisted per portfolio, and changed through
`switch-scope`, the scope selection use case or the Host API. Propagation to backends is
covered by `ushell-data-auth-scope`.

### Status quo
- The runtime mutates `InitialValue` as current value and persists the whole entry list in `localStorage` (`appScope_<applicationTitle>`).
- `PortfolioManager.SetAppScope` creates an entry for an unknown key but never adds it to the list (bug).
- Values may be objects (selected records); transports flatten them to a label.

### Refactoring & compatibility
- UB-22/UB-23: separate definition from current value in the runtime. Migrate the persisted format on load so existing browser state survives.

## 9. Authentication and token-source declarations

`PortfolioDescription` declares `PrimaryUiTokenSourceUid`, `AuthTokenConfigs` (dictionary by
token source UID, type from AuthTokenHandling), `AuthenticatedAccess` and `AnonymousAccess`.
Datasources, datastores and services reference token sources by UID. Behavior is described in
`ushell-data-auth-scope`.

## 10. Legal and branding

`CiDescriptionUrl`, `LegalContactMdUrl`, `UserAgreementMdUrl`, `UserAgreementVersion`,
`LogoUrlLight`/`LogoUrlDark` declare branding and legal content. Target: the shell renders
legal contact and user agreement as built-in workspaces/use cases. Agreeing to a version sets
a RuntimeTag that commands can require.

### Status quo
- `LegalContactMdUrl`, `UserAgreementMdUrl`, `UserAgreementVersion`, `CiDescriptionUrl` are not evaluated by ReactFrontend.

### Refactoring & compatibility
- UB-71: additive implementation in the shell.
