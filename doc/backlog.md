# U-Shell backlog

Cross-repository work list derived from the status blocks of the skills in
`.agents/skills/`. The skills hold the target state and the details; this list holds IDs,
ownership, priority and compatibility risk.

- **Created:** 2026-10-19 (conceptual phase, no code changed yet)
- **Snapshot:** ReactFrontend `c71aa36`, CommonComponents.JS `47f1629`, ModuleBase.JS `6d72d1c`, PortfolioDescription `7bf83f4`, ServerCommands.Contract `bfa24c2`

**Layers:**

| Code | Repository |
|---|---|
| PD | PortfolioDescription |
| MB | ModuleBase.JS |
| RF | ReactFrontend |
| CC | CommonComponents.JS |
| SC | ServerCommands.Contract |
| FB | FrontendBundle |
| DOC | UShell.Docs |

**Priority:** P1 = core of the target paradigm (dynamics, data/auth, scope, server commands), P2 = completes the paradigm, P3 = polish.

**Compat:** additive = no risk if done as described · opt-in = behavior change behind a switch · ⚠ = needs a check of productive portfolios/modules first.

## Epic A - Interaction dynamics (UoW, piping, actions)

| ID | Item | Layers | Prio | Compat | Skill |
|---|---|---|---|---|---|
| UB-01 | Bind WidgetHost to its use-case instance; pass a full execution context (source instance, source UoW, command args) for widget-fired, toolbar, menu and Host API commands | RF, MB | P1 | additive | interaction-dynamics §2 |
| UB-02 | Result return: `ApplyResult` on start-usecase commands, `result://` channel, `endUsecase(result)` in WidgetHost | PD, MB, RF | P1 | additive | interaction-dynamics §3 |
| UB-03 | `usecase-action` command type; widget action registration; shell actions `endUsecase`/`commit`/`discard`/`refresh` | PD, MB, RF | P1 | additive | interaction-dynamics §4 |
| UB-04 | RuntimeTag store (initial + token-derived + user-set, persisted); `set-runtime-tag` command | RF | P2 | additive | declarative-model §6 |
| UB-05 | Instance identity via `SingletonActionkey` pattern; empty keeps UoW-equality | RF | P2 | additive | declarative-model §3 |
| UB-06 | Appearance: modal by command (no target workspace / `.`), use-case default appearance, `WorkspaceAppearance`; `IconName == "modal"` and `IsSidebar` as aliases | PD, RF | P2 | additive | declarative-model §2, §4 |
| UB-07 | `userInput://` provider with shell dialog; async mapping resolution | RF, CC | P1 | additive | interaction-dynamics §5 |
| UB-08 | `setting://` provider and well-known settings source | PD, RF | P2 | additive | declarative-model §7 |
| UB-09 | `applicationScope://` value semantics (`.value`/`.label` sub-paths first); `RequiredApplicationScopes` checked against scopes, not tags | RF | P1 | ⚠ | declarative-model §2, §7 |
| UB-10 | Command metadata in global menu **and** CommandBar: visibility, availability, checked, confirmation, ordering | RF, CC | P2 | additive | declarative-model §5 |
| UB-11 | Dirty marker `changes`, `*` in title, confirm on close, commit/discard | RF | P2 | additive | interaction-dynamics §6 |
| UB-12 | `subscribeEvent` with shell event set | MB, RF | P2 | additive | interaction-dynamics §7 |
| UB-13 | `navigate` command honoring `RouterLink` | RF | P3 | additive | declarative-model §5 |
| UB-14 | `factory://UID64` | RF | P3 | additive | declarative-model §7 |
| UB-15 | Versioned persisted formats with migration on load | RF | P2 | additive | interaction-dynamics §8 |
| UB-16 | Promote class-only WidgetHost members to optional interface members; no throwing members | MB, RF | P2 | additive | widget-contract §2 |
| UB-17 | Widget registry instead of `if` cascade; bundles can contribute named widgets | RF | P2 | additive | widget-contract §4 |
| UB-18 | Central logging facade for shell, widgets, providers | MB, RF | P3 | additive | shell-runtime R17 |

## Epic B - Data, authentication, ApplicationScope

| ID | Item | Layers | Prio | Compat | Skill |
|---|---|---|---|---|---|
| UB-20 | Seamless token refresh: async provider, expiry check, renewal, single flight; transports detect 401 -> renew once -> retry once; login overlay with resumable calls | RF, CC, MB | P1 | additive | data-auth-scope §3 |
| UB-21 | Resolve datasources by UID and services by name | RF | P2 | additive | data-auth-scope §1 |
| UB-22 | One side-channel builder for all transports, per call, transmitting values; body `_` or header; dimension filter | RF, CC | P1 | opt-in ⚠ (label -> value) | data-auth-scope §4 |
| UB-23 | ApplicationScope runtime: separate definition and current value; fix unknown keys being dropped in `SetAppScope`; migrate persisted format | RF | P1 | additive (migrate) | declarative-model §8 |

## Epic C - Server commands

| ID | Item | Layers | Prio | Compat | Skill |
|---|---|---|---|---|---|
| UB-30 | `server-command` command type (alias `backend-action`) and declaration attributes | PD | P1 | additive | server-commands §2 |
| UB-31 | Execution, availability check and polling service in the shell | RF | P1 | additive | server-commands §1 |
| UB-32 | Generic progress/notification component for running executions | CC | P1 | additive | server-commands §1 |
| UB-33 | Persist and resume running executions after reload | RF | P3 | additive | server-commands §1 |

## Epic D - Host API (embedding in .NET WebView)

| ID | Item | Layers | Prio | Compat | Skill |
|---|---|---|---|---|---|
| UB-40 | `globalThis.ushell.host` with version, capabilities, inbound calls; keep `globalWorkspaceManager` / `setAppScope` as deprecated aliases | RF | P2 | additive | host-api §2 |
| UB-41 | Outbound event channel (WebView2, CefSharp, CustomEvent) | RF | P2 | additive | host-api §2 |
| UB-42 | Host-provided token source; `host-action` command type | PD, RF | P2 | additive | host-api §2 |
| UB-43 | `seamless_usecase` mode in `ushell-ts` | RF | P2 | additive | host-api §3 |
| UB-44 | Optional `host` transport provider | RF, CC | P3 | additive | host-api §4 |

## Epic E - Modules and CommonComponents

| ID | Item | Layers | Prio | Compat | Skill |
|---|---|---|---|---|---|
| UB-50 | Verify/complete CRUD overview pattern and entity-ref label resolver in the generic entity browser | CC | P2 | additive | common-components §2 |
| UB-51 | Provider-class registration for module bundles (datastores/datasources) | RF, MB | P2 | additive | module-development §3 |
| UB-52 | Starter template: module.json + optional bundle + stand-alone harness with `LocalWidgetHost` | new repo or DOC | P2 | - | module-development §3 |

## Epic F - Legacy and CtExpert

| ID | Item | Layers | Prio | Compat | Skill |
|---|---|---|---|---|---|
| UB-60 | Port CtExpert core functionality as U-Shell modules (FUSE endpoints, declarative modules, few custom widgets) | modules | P2 | - | legacy-integration §5 |
| UB-61 | Locate the U-Shell Angular prototype and the legacy QR widget; port QR as reference custom widget/CommonComponent | DOC, CC | P2 | - | legacy-integration §4 |
| UB-62 | Turn the private `AI-Skills` U-Shell skill into a pointer to this repository (single source of truth) | DOC | P2 | - | architecture §1 |

## Epic G - Shell-level UX

| ID | Item | Layers | Prio | Compat | Skill |
|---|---|---|---|---|---|
| UB-70 | Document title, error/not-found/access-denied surface, error boundary per use case, deep link preserved through login | RF | P2 | additive | shell-runtime §3 |
| UB-71 | Legal contact, user agreement (with RuntimeTag), cookie/storage notice from portfolio URLs | RF | P3 | additive | shell-runtime §3 |
| UB-72 | Central keyboard shortcuts and focus handling | RF | P3 | additive | shell-runtime §3 |

## Epic H - Hosting

| ID | Item | Layers | Prio | Compat | Skill |
|---|---|---|---|---|---|
| UB-80 | Pattern-based injection of base href and `portfolioLocation` meta, independent of minification | FB | P2 | additive | hosting §3 |
| UB-81 | Cache-header defaults for embedded shell and module bundles (`no-cache` for html/json/`remoteEntry.js`) | FB | P3 | additive | hosting §3 |
| UB-82 | Keep the source URL per module description; resolve relative URLs in `module.json` against it | RF | P1 | ⚠ | hosting §4 |
| UB-83 | Implement `tryGetModuleBaseUrl` on top of UB-82 | MB, RF | P2 | additive | hosting §4, widget-contract §2 |
| UB-84 | Starter template with `publicPath: "auto"`, `module.json` in the React project and canonical layout (extends UB-52) | DOC | P2 | - | hosting §4, §5 |

Handoffs to UniversalBFF (owned and tracked there, listed for visibility):

- Module providers register the embedded `module.json` of the React build and only supplement host-bound values (D-09); embed the build output in every configuration, including `module.json`.
- User-management reference module: replace C#-built description and `WidgetClass = "demo"`, rename federation `srViewer`, remove `backendApiUrl` from the UnitOfWork.

## Open design decisions

| ID | Question | Affects |
|---|---|---|
| D-01 | Names and shape of the result channel (`ApplyResult`, `result://`, explicit result object vs. target UoW) | UB-02 |
| D-02 | Syntax for declaring `userInput://` fields (label, type, default, validation) | UB-07 |
| D-03 | Default semantics of bare `applicationScope://name` (entry vs. value) and migration path | UB-09 |
| D-04 | Declaration shape of server commands; mapping of named arguments onto `string[]` | UB-30 |
| D-05 | Final event set and naming (`fireEvent` alias) | UB-12 |
| D-06 | Host API namespace, envelope format, security switch | UB-40..42 |
| D-07 | `TargetWorkspacePath` vs. `TargetWorkspaceKey` | UB-06 |
| D-08 | Where the `userInput`, confirmation and progress dialogs live (CC primitives used by RF) | UB-07, UB-10, UB-32 |
| D-09 | Exact set of host-bound values a host may supplement in `module.json`, and the mechanism (server-side overlay vs. separate host document) | UB-82, UniversalBFF |
| D-10 | Canonical module layout (`module.json`, `ui/`, `api/v{n}/`) mandatory or recommended | UB-84 |
| D-11 | Client-side refresh-token model (timing, storage, rotation) needed for seamless refresh; to be decided in AuthTokenHandling §32.2, not in U-Shell | UB-20, AuthTokenHandling |

## Suggested order

1. Close decisions D-01, D-03, D-04 (they shape the contracts).
2. Contract changes in PD and MB together (UB-02, UB-03, UB-16, UB-30), published as minor versions.
3. Runtime core: UB-01, UB-23, UB-20, UB-22.
4. Dynamics completion: UB-03, UB-02, UB-07, UB-31/32.
5. Everything else by priority.
