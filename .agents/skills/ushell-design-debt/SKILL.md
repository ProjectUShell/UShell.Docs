---
name: ushell-design-debt
description: Catalog of the known design debt, contract violations and inconsistencies of the productive U-Shell implementation - undocumented conventions a module author must know today, divergences between the target state and the code, and the compatibility risk of removing each one. Load together with ushell-architecture for any work that touches productive shell behavior, that writes a module.json against built-in widgets, or that decides whether something may be changed. It is a running inventory, not a backlog: it names the debt and the safe way to live with it until it is resolved.
---

# U-Shell design debt

U-Shell is **in productive use**. It was built with a strong focus on getting finished, and it
now has to grow into the target state described by the other skills in this repository. That
gap is normal and it is not a reproach; this skill makes it visible so it can be closed in
small, safe steps instead of being rediscovered by every developer.

Inventory date: 2026-10-19. Verified against ReactFrontend `dd373ad`, CommonComponents.JS
`6eee61e`, ModuleBase.JS `df6a0f4`, PortfolioDescription `7bf83f4`.

## 1. How to use this skill

1. **Before writing a `module.json` or a widget:** read section 3. Several conventions the
   productive shell relies on are not expressed in any contract. Writing against them is
   currently correct; writing against the target state alone produces a module that does not run.
2. **Before changing productive behavior:** read the entry for what you touch. Each entry
   names its compatibility risk and the agreed interim rule.
3. **When you find new debt:** add an entry here, in the same shape, and raise a backlog item
   in `doc/backlog.md`. Do not fix it opportunistically while working on something else.

**The consolidation rule.** We are in a consolidation phase. The priority is to *know* the
debt, not to remove it. Removing an entry is a deliberate, separately decided piece of work.
Until then the interim rule in each entry is the correct behavior, and code that follows it
is not wrong.

**Never break existing consumers.** Portfolios, modules and remote widgets are deployed. Every
resolution listed below has to be additive, opt-in, or guarded by an alias, exactly as
`ushell-architecture` §6 requires.

## 2. Severity and risk legend

| Mark | Meaning |
|---|---|
| **blocker** | A documented workflow does not work at all today |
| **trap** | Works, but a developer following the documentation gets it wrong |
| **drift** | Code and target state disagree; both are defensible |
| **untyped** | A contract is extended at runtime through `any`, invisible to consumers |

| Compat | Meaning |
|---|---|
| additive | Resolvable without touching existing behavior |
| alias | Old form has to stay as a documented alias |
| ⚠ | Needs a check of productive portfolios/modules before anything changes |

## 3. Debt a module author must know today

These are the entries that change what you write. They are the most valuable part of this
skill.

### DD-01 `widget` vs. `inputData` - two prop conventions (trap, alias)

The shell passes its built-in widgets the prop `widget: IWidget`
(`WorkspaceManager.renderWidget`: `<GuifadModule widget={input} />`), but a **federated remote
widget receives a single prop named `inputData`** (`FederatedComponentProxy`:
`<Component inputData={props.inputData} />`, assigned in `parseWidgetClass`).

`ushell-widget-contract` and `ushell-module-development` only mention `inputData`, so a
developer who reads the contract and then opens a built-in wrapper as an example gets the prop
name wrong in one of the two directions.

*Interim rule:* write remote widgets against `inputData`; write wrappers that live inside
CommonComponents against `widget`.
*Resolution:* keep `inputData` forever (UB-52 template, existing modules depend on it); a
clearer name may only be added alongside it.

### DD-02 Built-in widgets have an undocumented, inconsistent UnitOfWork contract (trap, ⚠)

Each built-in `WidgetClass` reads a fixed set of UoW fields. That set *is* the contract a
`module.json` author writes against, and it is documented nowhere. The field names are not
consistent between widgets.

| `WidgetClass` | Reads from the UoW | Writes back |
|---|---|---|
| `guifad` (`GuifadModule`) | `entityName`, `record`, `layoutDescription` | yes, via `persistUow` -> `populateChangedState` |
| `guifadFuse` | `fuseUrl`, **`rootEntityName`**, `routePattern` (default `"body"`) | no |
| `scopeSelection` (`EntitySelection`) | `entityName`, `scopeKey`, `layoutDescription` | no; triggers `switchScope` |
| `EntityTableModule` | `entityName`, `layoutDescription`, `pageSizes` (default `[50,100,200,500]`) | no |
| `debug`, `demo` | the whole widget | no |
| `portfolioChooser`, `login` | nothing (shell state only) | no |

Note `entityName` versus `rootEntityName` for the same concept.

*Interim rule:* use exactly the field names in this table; they are productive.
*Resolution:* additive alias fields, never a rename. A rename breaks deployed portfolios.

### DD-03 `guifadFuse` takes a backend URL out of the UnitOfWork (drift, ⚠)

`guifadFuse` is rendered with `fuseUrl={uow.fuseUrl}` and does not receive the widget at all.
This contradicts anti-pattern "widget-owned authentication or hard-coded service URLs"
(`ushell-architecture` §8) and hostability rule M-7 (`ushell-hosting`): backend URLs belong
into declared datastores, not into a UoW.

The same pattern appears in the UniversalBFF user-management reference module
(`BackendService.baseUrl` from `unitOfWork.backendApiUrl`), so it currently reads as the
sanctioned way to do it. It is not.

*Interim rule:* do not copy this into new modules. Declare a datastore and use
`tryGetDataSource` / `getDataSourceForEntity`.
*Resolution:* a datastore-based variant beside `guifadFuse`, old behavior kept while any
portfolio still uses it.

### DD-04 Half of CommonComponents is unreachable through the package (blocker, additive)

`src/index.tsx` is the published surface of npm `ushell-common-components`. Components that
`ushell-common-components` lists as available are **not exported**, among them
`LocalWidgetHost`, `GuifadModule`, `EntityTableModule`, `EntitySelection`, `Dialog`,
`Notification`, `LoadingScreen`, `ErrorScreen`, `TreeView`, `MultiPanelLayout`, the
`schema-editor`, `dashboard` and `card-board` components, and most connectors in `src/data/`
(`FuseConnector`, `LocalDataSourceManager`, `ObjectGraphDataSource*`, `EntitySchemaService`).

This blocks a documented workflow: `ushell-module-development` §3 tells module authors to
develop widgets stand-alone with `LocalWidgetHost`, which cannot be imported.

*Interim rule:* if a module bundle needs one of these, export it from `src/index.tsx` first
(purely additive) and release a new CommonComponents version. Do not work around it as in
DD-05.
*Resolution:* complete the public surface; see the CommonComponents skill for the exact list.

### DD-05 The shell deep-imports into `dist/cjs` (trap, additive)

Because of DD-04 the shell itself bypasses the package surface:

```ts
import GuifadModule from "ushell-common-components/dist/cjs/components/guifad/_Templates/GuifadModule";
import EntitySelection from "ushell-common-components/dist/cjs/_Organisms/EntitySelection";
```

It works for the shell, but it is not a pattern a module bundle may copy: a deep **CJS**
import into a federated bundle can pull a second React instance into the page and breaks the
shared-singleton rule M-5 (`ushell-hosting`).

*Interim rule:* module bundles import only from the package root. Never deep-import.
*Resolution:* falls away with DD-04.

## 4. Contract and runtime debt

### DD-06 The module source URL is smuggled onto the use case via `any` (untyped, additive)

`PortfolioLoader` merges all module descriptions into one and, while doing so, assigns the
module's base path onto each use case through a cast:

```ts
(c as any).moduleFinalPathWithoutFilename = moduleFinalPathWithoutFilename;
```

Two productive features depend on this invisible field: `WorkspaceManager.resolveRemoteUrl`
(relative federated widget URLs) and `WidgetHost.tryGetModuleBaseUrl`. It is not part of
`UsecaseDescription` in `UShell.PortfolioDescription`, so nothing type-checks it and no other
consumer knows it exists.

*Interim rule:* rely on the two behaviors, not on the field name.
*Resolution:* declare an optional, host-set property on the contract (.NET and TS together)
and keep the current name as the serialized form.

### DD-07 Only use cases get the module base - datastores and services do not (drift, additive)

The assignment in DD-06 happens **only** for `md.usecases`. Commands, datastores,
datasources, services and workspaces are pushed into the merged description without it.

Consequence: hostability rule M-4 ("all URLs inside `module.json` may be relative to the
module.json URL") holds today **only for federated widget URLs**. A relative datastore or
service URL does not resolve against the module.

*Interim rule:* in `module.json`, widget URLs may be relative; datastore, datasource and
service URLs must be absolute or host-supplemented.
*Resolution:* UB-82 completion - carry the source URL per module description instead of per
use case.

### DD-08 Merging module descriptions loses module identity (drift, ⚠)

`PortfolioLoader` concatenates the arrays of every module into one `ModuleDescription`. After
loading, there is no module boundary any more: `PortfolioManager.GetModule()` returns the
merged whole. DD-06 exists precisely to smuggle the lost information back in.

This also means keys are global across modules. Two modules with the same `usecaseKey`,
`workspace` key or `UniqueCommandKey` collide silently.

*Interim rule:* prefix keys per module (the `moduleScopingKey` is the natural prefix).
*Resolution:* keep the modules addressable after loading; needed by UB-82, UB-83 and by any
per-module overlay (decision D-09).

### DD-09 `WidgetHost` is not bound to a use-case instance (drift, additive)

Already tracked as UB-01, repeated here because it is the root cause of several surprises:
`fireEvent` carries no source use case, so `unitOfWork://` piping cannot reliably identify the
source UoW, and the shell cannot attribute a state change to the instance that caused it.

*Interim rule:* do not design a module around widget-fired commands that pipe from the source
UoW until UB-01 lands.

### DD-10 Interface members exist only on the implementation class (trap, additive)

`WidgetHost` has public members that `IWidgetHost` does not declare:
`getApplicationScope`, `getApplicationScopeValues`, `switchScope`, `pushBreadcrumbItem`,
`forceBreadcrumbItem`, static `fireEvent1`. A widget can only reach them by casting its host
to `any`, which every module then does, which makes the real surface impossible to see.

Conversely `subscribeEvent`, `getDataSource(uid)`, `getServiceByName` and the
`allignWidgetNavPanelLeft` getter are declared but `throw "not implemented"`.

*Interim rule:* a widget uses only `populateChangedState`, `fireEvent`, `getAccessToken`,
`getDataSourceForEntity`, `tryGetDataSource`, `getSchemaRoot`, `createServiceProxy`,
`tryGetModuleBaseUrl`. Everything else is either a cast or a throw.
*Resolution:* UB-16 (promote to optional interface members), UB-12, UB-21.

### DD-11 `RequiredApplicationScopes` is checked against RuntimeTags (drift, ⚠)

Tracked as UB-09. Listed here because it is a **semantic** confusion, not a missing feature:
two of the three state concepts that `ushell-architecture` §4 says must never be mixed are
mixed in productive code, and the resulting message to the user is about permissions.

### DD-12 `ApplicationScope` current value is stored in `InitialValue` (drift, ⚠)

Tracked as UB-22/UB-23. The runtime mutates the description object and persists the whole
entry list in `localStorage` under `appScope_<applicationTitle>`. Definition and runtime state
are therefore the same object, and a portfolio change cannot update an initial value for users
who already have persisted state.

*Interim rule:* treat `InitialValue` in a deployed portfolio as "value for new users only".

### DD-13 Modality rides on `IconName == "modal"` (trap, alias)

Tracked as UB-06. A presentation field carries behavior. It works and portfolios use it.

*Interim rule:* use it; it is the only way to get a modal today.
*Resolution:* explicit appearance, with `IconName == "modal"` kept as a documented alias.

### DD-14 Unimplemented command types fail loudly, by accident (drift, additive)

`usecase-action`, `set-runtime-tag` and every server-command type fall through to
`throw "invalid command type"`. That is the desired fail-closed behavior, but it is reached by
falling off the end of an `if` cascade rather than by an explicit decision, and `navigate`
silently ignores `RouterLink` instead of throwing.

*Interim rule:* do not declare these command types in a productive `module.json` yet.

### DD-15 Built-in widget resolution is an `if` cascade (drift, additive)

Tracked as UB-17. `renderWidget` resolves built-ins by a chain of string comparisons, with
mixed case handling (`widgetClass == "guifad"` but `widgetClass.toLowerCase() == "login"` and
`toLocaleLowerCase() == "demo"`). Widget names are therefore case-sensitive for some values
and not for others.

*Interim rule:* spell built-in widget names exactly as in DD-02.

## 5. Documentation and repository debt

### DD-16 `package.json` version is not the published version (trap)

In the JS repositories the `version` field in `package.json` is not maintained; KVU in `vers/`
owns the published version. CommonComponents reads `1.0.23` in `package.json` while the shell
consumes `^1.3.42`. Never reason about compatibility from `package.json`.

### DD-17 Status blocks age silently (drift)

Two status blocks in this skill set were already stale at this inventory: `ushell-hosting` M-4
and `ushell-widget-contract` (`tryGetModuleBaseUrl`) described behavior as missing that exists
in `PortfolioLoader` / `WidgetHost`. Both are corrected as of this date.

*Interim rule:* a status block is evidence with a date, never a fact. Re-verify against code
before relying on one, as `ushell-architecture` §1 already requires.

### DD-18 Three implementation repositories carry no copy of the mandatory entry skill (drift)

`UShell.ReactFrontend`, `UShell.CommonComponents.JS` and `UShell.ModuleBase.JS` have no
`.agents/skills/include-additional-mandatory-skills/` copy, which section 7 of that template
requires for every repository of the open-source layer. `UShell.PortfolioDescription` has one.
An agent entering one of those repositories directly does not learn where the leading skills
are.

## 6. Relation to the backlog

`doc/backlog.md` holds *what will be done*, with IDs, priority and ownership. This skill holds
*what is true today* and how to live with it. Where an entry has a backlog item, it is named.
Entries without one (DD-01, DD-02, DD-05, DD-16, DD-17) are either resolved by documentation
alone or wait for a decision.

New entries: give the next free DD number, state the evidence (file and symbol), the severity,
the compatibility risk, the interim rule, and the resolution direction. Never write a
resolution into this skill as if it had happened.
