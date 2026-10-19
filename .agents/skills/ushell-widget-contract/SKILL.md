---
name: ushell-widget-contract
description: The U-Shell widget contract (target state plus current status and refactoring) - IWidget, IWidgetHost, UsecaseState and what a widget may rely on from the shell - state publication, command execution, events, action registration, data/service access, tokens, ApplicationScope, RuntimeTags, logging, module-relative resources. Load when writing or reviewing any widget (built-in, CommonComponent or remote/federated), when changing UShell.ModuleBase.JS, or when changing the WidgetHost implementations in ReactFrontend or CommonComponents (LocalWidgetHost). Requires ushell-architecture.
---

# U-Shell widget contract

Bounded context: **widget contract**. Contract repository: `UShell.ModuleBase.JS` (npm
`ushell-modulebase`). Implementations: `ReactFrontend/portfolio-handling/WidgetHost.ts`
(shell) and `CommonComponents/data/LocalWidgetHost.ts` (stand-alone/debug).

Snapshot for status blocks: 2026-10-19, ModuleBase.JS `6d72d1c`, ReactFrontend `c71aa36`,
CommonComponents.JS `47f1629`.

## 1. Mental model

A widget is a **custom control against the shell runtime**. It receives:

- `state: UsecaseState` - identity of its use-case instance plus the UoW it works on,
- `widgetHost: IWidgetHost` - its only door into shell capabilities.

A widget must work correctly when it is destroyed and recreated with the same `state` (tab
switch, reload, session restore). A widget must never:

- read or write `localStorage` for application state,
- acquire, parse or store tokens itself,
- build service URLs or authorization headers itself,
- navigate the shell or start other use cases imperatively,
- talk to other widgets directly.

The contract is technology-neutral: React federation is today's loading mechanism, but the
same contract must be implementable for WebComponents later. Avoid React-specific types in
`ushell-modulebase`.

## 2. `IWidgetHost` - target capability set

Each widget gets a WidgetHost **bound to its use-case instance**. That binding lets the shell
know the source of every command, event and state change.

| Capability | Member (productive name where it exists) | Target semantic |
|---|---|---|
| Publish state | `populateChangedState(state)` | The shell persists the new UoW and treats it as current. Partial updates are not allowed; always pass the full state. |
| Execute command | `fireEvent(commandKey, args)` | Executes the command with this widget's use case as source; `args` are `commandArgs`. See `ushell-interaction-dynamics`. |
| Observe | `subscribeEvent(name, handler)` | Shell events (scope, tags, UoW changed by shell, token, activation). Returns or allows unsubscription. |
| Register actions | *(new)* `registerActions(...)` | Handlers for `usecase-action` commands targeting this use case. |
| Finish | *(new)* `endUsecase(result?)` | Ends the own use case; the result feeds `ApplyResult`. |
| Data | `getDataSourceForEntity(entity, store?)`, `tryGetDataSource`, `getDataSource(uid)`, `getSchemaRoot()` | Shell-resolved, authenticated, scope-aware datasources (FUSE contract). |
| Services | `getServiceByName(name)`, `createServiceProxy(url, tokenSourceUid?)` | Shell-resolved UJMW proxies with token and side channel already wired. Prefer declared services over raw URLs. |
| Tokens | `getAccessToken(tokenSourceUid)` | Returns a **valid** token. Expired tokens are renewed transparently. `null` only if authentication failed or was cancelled. |
| ApplicationScope | *(exists on class only)* `getApplicationScopeValues()`, `switchScope(key, value)` | Read the current values; request a switch (shell validates). |
| RuntimeTags | *(new)* `getRuntimeTags()` | Read-only. Changing tags goes through commands. |
| Navigation helpers | *(exists on class only)* `pushBreadcrumbItem`, `forceBreadcrumbItem` | Shell-owned breadcrumb for widgets with internal navigation. |
| Resources | *(declared, not implemented)* `tryGetModuleBaseUrl(usecaseKey)` | Module-relative assets, resolved against the module's `module.json` URL (UB-82, UB-83, see `ushell-hosting`). |
| Layout preference | `allignWidgetNavPanelLeft` | Global preference for a widget's own nav panel side. |
| Logging | *(new)* `log` facade | Central logging; widgets do not build logging islands. |

Contract evolution is additive. New members are optional on the interface or added with a
feature check, so existing remote widgets compiled against older `ushell-modulebase` keep
working, and new widgets degrade gracefully on older shells.

### Status quo
- `IWidgetHost` declares: `populateChangedState`, `subscribeEvent`, `fireEvent`, `getAccessToken`, `getDataSource`, `getDataSourceForEntity`, `getServiceByName`, `createServiceProxy`, `tryGetModuleBaseUrl`, `allignWidgetNavPanelLeft`, plus `tryGetDataSource`/`getSchemaRoot` via `IDataSourceManagerWidget`.
- ReactFrontend `WidgetHost`:
  - `subscribeEvent`, `getDataSource(uid)`, `getServiceByName` and the `allignWidgetNavPanelLeft` getter throw "not implemented".
  - `getAccessToken` returns the stored token without renewal.
  - The class is **not bound** to a use-case instance, so `fireEvent` passes no source UoW.
  - It has extra public members that are not part of the interface: `getApplicationScope`, `getApplicationScopeValues`, `switchScope`, `pushBreadcrumbItem`, `forceBreadcrumbItem`, static `fireEvent1`. Widgets can only reach them by casting.
- `tryGetModuleBaseUrl` **is implemented** in ReactFrontend (corrected 2026-10-19): it finds the use case in the merged module description and returns the module base path that `PortfolioLoader` assigned to it. The lookup is by `usecaseKey` over all modules, so it is ambiguous when two modules use the same key (DD-08).
- CommonComponents provides `LocalWidgetHost` for running widgets outside the shell, but it is **not exported** from the package, so module authors cannot use it (DD-04).
- Built-in widgets rendered by the shell receive the prop `widget`; federated remote widgets receive `inputData` (DD-01).
- ReactFrontend consumes `ushell-modulebase` `^1.0.25`.

### Refactoring & compatibility
- UB-01: bind WidgetHost to the use-case instance and pass source context.
- UB-12: implement `subscribeEvent`.
- UB-03: add action registration. UB-02: add `endUsecase(result)`.
- UB-20: make `getAccessToken` honor its documented renewal promise.
- UB-21: implement `getDataSource(uid)` and `getServiceByName` from module descriptions.
- UB-16: promote the class-only members to the interface as optional members. Do not remove them from the class.
- UB-16: implement or default `allignWidgetNavPanelLeft` instead of throwing.
- Keep `LocalWidgetHost` in sync with every interface change. It is the reference for module developers running widgets stand-alone.

## 3. `UsecaseState`

| Field | Semantic |
|---|---|
| `usecaseInstanceUid` | runtime identity of the instance |
| `usecaseKey` | type (`UsecaseDescription.UsecaseKey`) |
| `parentWorkspaceKey` | owning workspace |
| `title` | current title (placeholders resolved) |
| `fixed` | static use case, cannot be terminated |
| `unitOfWork` | the state (see `ushell-interaction-dynamics`) |

Target additions (optional fields, additive): `instanceKey` (resolved instance-key pattern),
`appearance` (`tab` / `modal`), and a reference to the source instance for result return.

### Status quo
- Matches the first table. No additions yet.

## 4. Widget loading

| WidgetClass form | Loaded by |
|---|---|
| Built-in name (`guifad`, `guifadFuse`, `scopeSelection`, `login`, `debug`, `portfolioChooser`) | `WorkspaceManager.renderWidget` switch |
| JSON `{scope, module, url}` | federation (`ModuleLoader`, `FederatedComponentProxy`) |
| WebComponent tag/URL | future; keep the contract open |

Target: built-in widgets are resolved through a registry rather than an `if` cascade, so
CommonComponents can contribute widgets declaratively.

### Status quo
- Hard-wired `if` cascade in `renderWidget`. The federated path passes `inputData` to the remote component.

### Refactoring & compatibility
- UB-17: introduce a widget registry. Keep every existing built-in name working.
