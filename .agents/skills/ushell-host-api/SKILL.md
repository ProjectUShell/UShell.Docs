---
name: ushell-host-api
description: U-Shell Host API (target state plus current status and refactoring) - the versioned JavaScript API through which an embedding host, primarily a .NET desktop application with a WebView control (WebView2, CefSharp), controls and observes the shell - inbound calls (scope, runtime tags, tokens, commands, use cases), outbound events, embedding modes (headless/seamless), host-provided tokens and an optional host transport for backend calls. Load when embedding U-Shell into a desktop/native host or when touching globalThis exposure, headless mode or host integration. Requires ushell-architecture.
---

# U-Shell Host API

Implementation home: `UShell.ReactFrontend` (a dedicated module, e.g. `host-api/`). A small
.NET helper for hosts may live in a separate package later. This skill defines the
contract; nothing here is product-specific.

Snapshot for status blocks: 2026-10-19, ReactFrontend `c71aa36`.

## 1. Purpose

U-Shell runs as a normal browser SPA **and** embedded in a WebView inside a .NET desktop
application. The embedding host needs to:

- push context into the shell (ApplicationScope, RuntimeTags, tokens it already owns via desktop SSO),
- trigger shell behavior (commands, start/activate use cases, workspaces),
- observe the shell (ready, use-case lifecycle, dirty state, titles, scope changes),
- render the shell in reduced modes (only one use case, no menu) when the host owns navigation,
- optionally carry backend calls itself (no HTTP from the WebView).

All of this goes through **one** documented, versioned API instead of ad-hoc globals.

## 2. Shape (target, names not final)

```text
globalThis.ushell.host                     (present once the shell bootstrapped)
├── apiVersion: "1.x"
├── capabilities(): string[]
├── inbound
│   ├── setApplicationScope({ name: value, ... })
│   ├── setRuntimeTags(["tag", "!toggle", "-remove"])
│   ├── provideToken(tokenSourceUid, token)          host-issued tokens
│   ├── executeCommand(uniqueCommandKey, commandArgs?, sourceUsecaseInstanceUid?)
│   ├── startUsecase(workspaceKey | null, usecaseKey, initUnitOfWork?)
│   ├── activateWorkspace(workspaceKey)
│   ├── closeUsecase(usecaseInstanceUid, force?)
│   └── getSnapshot()                                 open workspaces/use cases, titles, dirty flags
└── outbound (events)
    ├── ready
    ├── usecaseStarted / usecaseActivated / usecaseClosed
    ├── titleChanged / dirtyChanged
    ├── applicationScopeChanged / runtimeTagsChanged
    ├── tokenRequested(tokenSourceUid)                shell asks the host for a token
    └── hostActionRequested(name, args)               commands the host implements (e.g. print, open file)
```

Rules:
- **Inbound calls reuse shell semantics.** `executeCommand` runs the same pipeline as a menu click (confirmation, `mapDynamic`, RuntimeTag gating). The host does not get privileged shortcuts around shell rules.
- **Outbound transport is pluggable.** Detect the host and use its native channel: WebView2 `window.chrome.webview.postMessage`, a CefSharp bound object, or as fallback a DOM `CustomEvent` on `window`. Messages are JSON envelopes `{ apiVersion, type, payload }`.
- **Early calls are queued.** Calls arriving before the portfolio is loaded are buffered and applied after bootstrap (as `setAppScope` already does).
- **Host-provided tokens.** A token source may declare that the host issues its tokens. The shell then raises `tokenRequested` and waits for `provideToken`, including for renewal (`ushell-data-auth-scope`).
- **Host actions.** A command type (e.g. `host-action`) forwards to `hostActionRequested`; the command is hidden or disabled when no host advertises the action in `capabilities`.
- **Versioning.** `apiVersion` follows semver. Additive changes only within a major version.
- **Security.** The API exposes control, not secrets. Outbound events never contain tokens. Provide a portfolio switch to disable the Host API when the shell runs as a public website.

## 3. Embedding modes

| Mode | Activated by | Renders |
|---|---|---|
| default | - | full shell (menus, tabs, workspaces) |
| headless | URL `?headless=...` | workspace area without shell menus |
| seamless use case | URL `?seamless_usecase=<usecaseKey>` (+ init UoW) | exactly one use case; host owns tabs and menus |

## 4. Optional host transport

A desktop host may serve backend calls itself (in-process services, offline data). A
datastore/service provider class `host` sends requests through the Host API and receives
responses asynchronously (request handle -> response). Push messages from the host map to
shell events.

Legacy evidence (CtExpert `w3bstract.DataBackendService`): `SendInMemoryRequest(url, method,
data) -> handle`, `ReceiveInMemoryResponse(handle, data)`, handle `-1` as "hook active" signal,
handle `0` as push channel, services exposed on `window` for CefSharp. The concept proved
useful. The target replaces the magic handle numbers with typed envelopes.

### Status quo
- `globalThis.globalWorkspaceManager` (the full `WorkspaceManager` instance) and `globalThis.setAppScope(values)` are exposed in `App.tsx`. `setAppScope` before bootstrap is buffered (`PortfolioManager.externalAppScope`).
- `?headless` renders only the `Workspace` component.
- The readme documents `?seamless_usecase=`, but `ushell-ts` has no implementation of it.
- `meta[name="portfolioLocation"]` lets a host choose the portfolio URL.
- No outbound events, no host token source, no host transport, no versioning.
- A desktop wrapper concept exists in `ProjectUShell/UShell.ReactFrontend.CefWrapper` (CEF offline hosting). Not analyzed yet; check it before designing UB-41/UB-44.

### Refactoring & compatibility
- UB-40: introduce `globalThis.ushell.host` with version and capabilities.
- UB-40: keep `globalWorkspaceManager` and `setAppScope` as deprecated aliases. Hosts in the field may call them.
- UB-41: outbound event channel with WebView2 / CefSharp / CustomEvent adapters.
- UB-42: host-provided token source and `host-action` command type (additive contract changes in PortfolioDescription).
- UB-43: re-implement `seamless_usecase` in `ushell-ts`.
- UB-44: optional `host` transport provider.
