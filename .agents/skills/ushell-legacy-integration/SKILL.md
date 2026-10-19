---
name: ushell-legacy-integration
description: How to evolve and integrate legacy implementations into the U-Shell target state - stepwise migration of hand-built custom UIs and embedded SPAs into shell-owned use cases (integration levels), porting knowledge from earlier generations (2019 concept, Angular prototype, CtExpert legacy shell) without copying their technology, the deprecation path for productive shell behavior, and the concrete findings already extracted from CtExpert. Load when working on existing modules/widgets that predate the target state, when integrating an existing application, or when mining older code for requirements. Requires ushell-architecture.
---

# Evolving and integrating legacy implementations

"Legacy" covers three things:

1. **Pragmatic U-Shell modules**: hand-built custom React UIs, often a whole SPA inside one static use case, with own state, toolbars, auth and URLs.
2. **Earlier generations**: the 2019 concept (`core-idea-kornsw.md`, `epics/`), the Angular U-Shell prototype, CtExpert's Angular/PrimeNG shell (`kswNgShell`), VB.NET MVP use-case controllers, WinForms desktop shells.
3. **External applications** that have to appear inside the shell.

## 1. Principles

- **Never break running consumers.** Every step below is shippable on its own.
- **Older generations are evidence, not requirements.** Extract the generic capability, map it to the target-state skill that owns it, and record a backlog item. Do not port technology (Angular, PrimeNG, VB MVP).
- **Move orchestration outwards, step by step.** Each step moves one responsibility from the custom UI into the shell or into CommonComponents.

## 2. Integration levels

| Level | Description | Typical next step |
|---|---|---|
| L0 | External app via link or iframe | wrap as federated widget, or keep isolated if third party |
| L1 | Federated widget, but internally a complete SPA (own routing, state, auth, URLs) | let it obtain tokens and services through the WidgetHost |
| L2 | Uses WidgetHost for data, services, tokens, ApplicationScope | move meaningful state into the UoW |
| L3 | State lives in the UoW; survives reload | split internal screens into shell use cases; turn internal buttons into commands |
| L4 | Flows between screens are commands + `mapDynamic`; generic screens replaced by CommonComponents | the remaining widget is a true custom control |

Rules for moving up:
- **L1 -> L2**: remove token handling and hard-coded URLs first. This is invisible to users and immediately enables seamless refresh and side-channel propagation.
- **L2 -> L3**: introduce UoW fields incrementally. Read from the UoW with fallback to defaults, so persisted old states keep loading.
- **L3 -> L4**: extract one screen at a time into its own use case. The old internal navigation can coexist until the last screen moved.
- Generic parts discovered on the way (tables, editors, dialogs) go to CommonComponents, not into the module.

## 3. Deprecation path for productive shell behavior

Productive behavior of the shell itself is legacy as soon as the target state differs:

1. **Add** the target mechanism next to the old one (new attribute, new channel, new API).
2. **Alias** the old mechanism onto the new one where possible (e.g. `IconName == "modal"` -> modal appearance, `IsSidebar` -> sidebar appearance, `globalWorkspaceManager` -> Host API).
3. **Deprecate**: document in the target-state skill and in the release notes, and log a warning when used.
4. **Remove** only with a major version of the affected package, after productive portfolios/modules were checked.

Persisted browser state follows the same rule: migrate on load, never discard silently.

## 4. Findings from CtExpert (2026-10-19)

Source: `KornSW/CtExpert`, `PlatformFrontend/CTE.Frontend.SpaBundle/spa` (Angular 9 + PrimeNG,
libraries `kswNgShell`, `kswNgUserAuth`, `kPers`, `w3bstract`). It is **not** the U-Shell
Angular prototype; it has no workspaces, UoW, commands or `mapDynamic`. The U-Shell Angular
prototype and the legacy QR widget are still to be located (UB-61).

Worth keeping (generic):

| Legacy behavior | Target home | Backlog |
|---|---|---|
| Central auto-relogin hook in the service base: on "unauthorized" renew once, retry the original call once, never loop | `ushell-data-auth-scope` §3 | UB-20 |
| Deep link preserved through login (`return` + all query params) | `ushell-shell-runtime` §3 | UB-70 |
| Document title `App - Page`, error page for 404 / access denied / generic | `ushell-shell-runtime` §3 | UB-70 |
| Cookie notice and privacy/imprint page from company info | legal built-ins | UB-71 |
| Menu entries filtered by permission; empty parents removed | RuntimeTags from token scopes + visibility tags | UB-10 |
| Route-to-permission mapping with dynamic scope (`Study:{id}`, `User:{me}`); allow/deny and scope hierarchy were only sketched | backend authorization stays authoritative; frontend uses tags | idea |
| CRUD overview: list + side panel create/edit/delete-confirm, validation issues block submit | CommonComponents catalog | UB-50 |
| List paging, toggle sort, 500 ms debounced search, reload button, per-item buttons | CommonComponents table | ✅ exists |
| Item-referer cache for FK labels and dropdowns | `getEntityRefs` resolver | UB-50 |
| Multiple backends by alias from runtime settings (`settings.js`) | declared datastores/services + static config delivery | ✅ concept exists |
| Desktop host bridge (CefSharp): services on `window`, in-memory request/response by handle, push channel | `ushell-host-api` | UB-40..44 |
| Planned stubs: notifications, persistent messages, log viewer, PDF viewer, report list, CMS content blocks, self sign-up | CommonComponents catalog / backlog | UB-32, ideas |

Do not port:
- MD5 password hash stored in `localStorage` for auto-login.
- Raw token string from `localStorage` as `Authorization` header in every call.
- Date sentinel `1900-01-01` instead of `null`.
- Hard-coded menu building in code (the target is declarative commands).

## 5. Porting CtExpert functionality onto the new platform

Goal: CtExpert's core functionality (studies, assessments, medical companies, visits) runs
as U-Shell modules.

1. Expose the CtExpert backends (VES, DVS, SDS, VPS) as FUSE repository endpoints and UJMW services.
2. Describe one module per bounded context with CommonComponents only (browser + editor flows).
3. Add specialized widgets only where needed (e.g. visit planning, QR).
4. Use ApplicationScope for ambient context such as the medical company, propagated via side channel.

Tracked as UB-60.
