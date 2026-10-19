---
name: ushell-common-components
description: Layer requirements for UShell.CommonComponents.JS (npm ushell-common-components) - what belongs into the generic, business-free component and transport library and what does not, the expected configuration-first component catalog (tables, entity browser/editor, CRUD overview, filters, selection, reports, charts, dialogs, user input, progress, QR), the data connectors (FUSE, service proxy, local stores) and their duties for tokens and side channels, plus status and gaps. Load before adding or changing anything in CommonComponents or before writing a custom widget (check here first). Requires ushell-architecture.
---

# U-Shell CommonComponents (generic layer)

Implementation repository: `UShell.CommonComponents.JS`, published as npm
`ushell-common-components`. Consumed by ReactFrontend and by module bundles.

Snapshot for status blocks: 2026-10-19, CommonComponents.JS `47f1629`.

## 1. What belongs here

| Belongs here | Does not belong here |
|---|---|
| Business-free UI components (atoms to templates) | Any domain concept (customer, study, invoice, ...) |
| Generic widgets usable as `WidgetClass` (entity browser/editor, report, dashboard, schema editor) | Shell orchestration: menus of the portfolio, use-case lifecycle, command execution, persistence |
| Data connectors implementing `IDataSource` / `IDataStore` (FUSE, local, object graph) | Token acquisition and storage (only consume a token **callback**) |
| Transport helpers (`createServiceProxy`, FUSE connectors) with hooks for token, 401 handling and side channel | Reading ApplicationScope or tokens from global state (receive them via callbacks/factories) |
| `LocalWidgetHost` and local data managers for stand-alone widget development | Direct access to `WorkspaceManager` or `PortfolioManager` |
| Shell-layout primitives the shell and stand-alone modules share | |

Test for every addition: "Would this component be useful unchanged in a completely different
business application?" If not, it belongs into a module.

Components interact with the shell only through what they are given: props, an `IWidget` /
`IWidgetHost`, `IDataSource`, callbacks. That keeps them usable inside the shell and stand-alone.

## 2. Expected component catalog (configuration-first coverage)

| Capability | Target | Status 2026-10-19 |
|---|---|---|
| Table with paging, sorting, column filters, debounced search | generic over `IDataSource` | ✅ `components/Table`, filter molecules, `SearchBar` |
| Generic entity browser/editor incl. master-detail and relations | `guifad` family, driven by FUSE schema | ✅ `Guifad`, `GuifadFuse`, `GuifadModule`, `EntityTableModule` |
| CRUD overview pattern: list + side panel for create / edit / delete-confirm, validation issues block submit, error inside the panel, reload after close | part of the entity browser | ❓ verify coverage against this pattern (from CtExpert legacy) |
| Entity reference labels (FK display, select options) via `getEntityRefs` with caching | generic resolver | ❓ verify (UB-50) |
| Entity selection (pick a record) | `EntitySelection` | ✅ |
| Reports | `components/report` | ✅ |
| Dashboard, card board | `components/dashboard`, `card-board` | ✅ |
| Schema editor | `components/schema-editor` | ✅ |
| Charts | generic chart widget | ❓ |
| PDF viewing | generic viewer | ❌ |
| Log viewer | generic viewer | ❌ |
| Dialog, notification, loading/error screens | primitives | ✅ `Dialog`, `Notification`, `LoadingScreen`, `ErrorScreen` |
| Confirmation dialog for commands | primitive used by the shell | ❓ (UB-10) |
| `userInput://` dialog (typed fields, collected per command) | generic form dialog | ❌ (UB-07) |
| Server command progress/notification area | generic progress UI | ❌ (UB-32) |
| QR code display (and optionally scan) | generic component; reference port of the legacy QR widget | ❌ (UB-61) |
| Wizard mechanics | generic step container driven by UoW | ❌ |

## 3. Data connectors and transport duties

Connectors in `src/data/`: `FuseDataStore`, `FuseConnector`, `FuseDataSource*`,
`GenericFuseFxRepositoryConnector`, `ServiceFactory` (`createServiceProxy`),
`LocalDataSourceManager`, `LocalStoreDataSource`, `ObjectGraphDataSource*`,
`LocalWidgetHost`, `EntitySchemaService`.

Target duties of every transport (FUSE and service proxy alike):

- obtain the token through an **async** callback supplied by the caller (the shell),
- on 401: call the supplied `onUnauthorized(tokenSourceUid)` callback once, then retry the original request exactly once,
- add the ApplicationScope side channel through a supplied **per-call** factory (body `_` or header, per configuration),
- never parse, store or renew tokens itself,
- keep the public response shapes stable (`{ return, fault }` for the service proxy).

### Status quo
- `FuseDataStore.getTokenMethod` is a static, synchronous token getter. Additional body/header args are factories.
- `createServiceProxy(baseUrl, headersFactory?, additionalBodyPropertiesFactory?)` returns `{ return: null, fault }` for any non-OK response, without 401 handling or retry.

### Refactoring & compatibility
- UB-20: add optional async token callback and `onUnauthorized` hook next to the existing parameters. Existing callers keep working unchanged.
- UB-22: accept a side-channel configuration (transport, dimensions) and a per-call value factory.

## 4. Repository conventions

- Coding and structure guidelines: `guidelines.md` in the repository (atomic design, PascalCase components, Tailwind).
- Folders `src/[Move2LayoutDescription]` and `src/[Move2Logging]` mark code that is planned to move into separate packages. Do not extend them; ask before moving.
- Versioning is driven by the KVU tooling in `vers/` (`versioninfo.json`). The `version` in `package.json` is not the published version. Never edit `vers/changelog.md`.
- Every change to the widget contract must also be reflected in `LocalWidgetHost`.
