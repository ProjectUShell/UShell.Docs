---
name: ushell-shell-runtime
description: Layer requirements for the U-Shell runtime harness (UShell.ReactFrontend) - everything the shell itself must implement because it needs global knowledge or ownership - bootstrap, portfolio/module loading, menus and placement, workspaces and use-case tabs/modals, command execution pipeline, UoW piping and persistence, RuntimeTag store, ApplicationScope runtime, token service, widget registry and WidgetHost, Host API, logging facade, shell-level UX. Includes a consolidated status overview. Load before changing ReactFrontend or when deciding whether a feature belongs into the shell or into CommonComponents. Requires ushell-architecture.
---

# U-Shell shell runtime (harness layer)

Implementation repository: `UShell.ReactFrontend`. The productive project is `ushell-ts/`
(React + TypeScript + Tailwind, webpack module federation). The `ushell/` folder is an older
JavaScript generation; do not extend it.

Snapshot for status blocks: 2026-10-19, ReactFrontend `c71aa36`.

## 1. What belongs into the shell

A capability belongs into the shell when it

- needs global knowledge (portfolio, all modules, all open use cases, tokens, scope, tags), or
- must behave identically for every widget (lifecycle, toolbars, confirmations, persistence, keyboard), or
- is part of the WidgetHost or Host API.

Everything business-free and reusable without global shell knowledge goes to
CommonComponents (`ushell-common-components`). The shell **consumes** CommonComponents for
its own UI.

## 2. Responsibilities and status overview

| # | Responsibility | Target (details in) | Status 2026-10-19 |
|---|---|---|---|
| R1 | Bootstrap: portfolio location (meta tag / base path / `?portfolio=`), chooser portfolio, module aggregation | `ushell-declarative-model` | ✅ |
| R2 | Menus: global, user (`MY`), use-case toolbar; folders, ordering, visibility, availability, checked, confirmation | `ushell-declarative-model` §5 | ⚠️ visibility only in global menu; ordering, availability, checked, confirmation missing (UB-10) |
| R3 | Workspaces: activation, static use cases, appearance (default/sidebar), required scopes | `ushell-declarative-model` §2 | ⚠️ appearance not evaluated, scopes checked against tags (UB-06, UB-09) |
| R4 | Use-case instances: start, re-activate by instance key, tabs, modal, terminate | `ushell-declarative-model` §3-4 | ⚠️ UoW-equality instead of instance key; modal via `iconName` hack (UB-05, UB-06) |
| R5 | Command pipeline: execution context, `mapDynamic`, confirmation, all command types | `ushell-interaction-dynamics` §2 | ⚠️ 3 of 7 types; no source context for widget-fired commands (UB-01, UB-03, UB-04, UB-13, UB-30) |
| R6 | Result return to source use case | `ushell-interaction-dynamics` §3 | ❌ (UB-02) |
| R7 | `userInput://` dialog, `setting://` provider | `ushell-interaction-dynamics` §5 | ❌ (UB-07, UB-08) |
| R8 | UoW persistence, restore, dirty/commit semantics, versioned formats | `ushell-interaction-dynamics` §6, §8 | ⚠️ persistence ✅, dirty ❌, no versioning (UB-11, UB-15) |
| R9 | RuntimeTag store (initial + token-derived + user-set, persisted) | `ushell-declarative-model` §6 | ⚠️ initial + token-derived only (UB-04) |
| R10 | ApplicationScope runtime: current values, persistence, switch, dependent constraints, scope selection | `ushell-declarative-model` §8 | ⚠️ definition and value mixed; unknown keys dropped (UB-22, UB-23) |
| R11 | Token service: issue, validate, store, renew, single flight, login overlay | `ushell-data-auth-scope` §2-3 | ⚠️ issue/login ✅, renewal ❌ (UB-20) |
| R12 | Data/service resolution: datastores, datasources by UID, services by name, side channel | `ushell-data-auth-scope` §1, §4 | ⚠️ datastores ✅, UID/service lookup ❌, side channel inconsistent (UB-21, UB-22) |
| R13 | Widget registry and WidgetHost bound per use-case instance | `ushell-widget-contract` | ⚠️ `if` cascade; unbound host; several members throw (UB-01, UB-12, UB-16, UB-17) |
| R14 | Remote widget loading (federation), later WebComponents | `ushell-widget-contract` §4 | ✅ federation |
| R15 | Server command execution and progress | `ushell-server-commands` | ❌ (UB-30..33) |
| R16 | Host API and embedding modes | `ushell-host-api` | ⚠️ ad-hoc globals, `headless` only (UB-40..44) |
| R17 | Logging facade for shell, widgets and providers | `ushell-architecture` | ❌ `console.*` only (UB-18) |
| R18 | Shell-level UX: document title, error/not-found surface, deep link preserved through login, legal and user agreement, theme (CI), keyboard/focus conventions | below | ⚠️ partly (UB-70, UB-71, UB-72) |

## 3. Shell-level UX requirements (R18)

These were proven in earlier generations (CtExpert legacy shell) and are generic:

- **Document title** = `ApplicationTitle - <active use case title>`.
- **Not found / access denied / error surface**: an unknown workspace or use case, a missing permission (required tags/scopes) or a widget crash shows a shell-owned, consistent surface with a way back to the landing workspace. A crashing widget must not take down the shell (error boundary per use case).
- **Deep link through login**: a deep link that requires authentication returns to the original target, including its query parameters, after login.
- **Legal**: `LegalContactMdUrl` and `UserAgreementMdUrl` render as built-in use cases. Accepting `UserAgreementVersion` sets a RuntimeTag. The cookie/storage notice is part of this.
- **Theme/CI**: `CiDescriptionUrl`, logos, dark/light mode (exists in `ShellSettings`).
- **Keyboard and focus**: central shortcuts for commands, tab switching, closing; focus moves into a newly activated use case.

### Status quo
- Landing workspace and color mode work. Breadcrumbs exist (`AppBreadcrumb`).
- No document title handling, no error boundary per use case. Legal URLs and user agreement are not evaluated. `ProtectedRoute` exists; whether deep links survive login is unverified.

## 4. Implementation rules for this layer

- Every change must keep existing portfolios, modules and remote widgets working (`ushell-architecture` §6).
- New descriptive attributes are added to `UShell.PortfolioDescription` first (.NET and TS), then consumed here.
- New widget-facing capabilities are added to `UShell.ModuleBase.JS` first (optional members), then implemented here **and** in `LocalWidgetHost` (CommonComponents).
- Do not put reusable UI into this repository when it could live in CommonComponents.
- Keep `WorkspaceManager` from growing further: extract the command pipeline, `mapDynamic` resolution and instance management into separate units as they are touched.
