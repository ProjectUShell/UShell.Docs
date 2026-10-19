---
name: ushell-architecture
description: MANDATORY root entry for any U-Shell work in any ProjectUShell repository (ReactFrontend, CommonComponents.JS, ModuleBase.JS, PortfolioDescription, ServerCommands.Contract, modules built on U-Shell). Defines the U-Shell vision, the ownership model, the glossary, the context map (which repository owns which truth), the authority and compatibility rules, and the index of all U-Shell domain skills in UShell.Docs. Load this first, then load the domain skills matching the task.
---

# U-Shell Architecture (root entry)

> Leading source for the U-Shell **target state**. All U-Shell design knowledge lives
> in `ProjectUShell/UShell.Docs/.agents/skills/`. The `.agents/skills` folders inside the
> implementation repositories are deliberately slim and only carry implementation details.

## 1. How to use this skill set

1. Always load this skill first.
2. Load every domain skill from the index (section 9) that touches the task. Most tasks need two or three.
3. Then load the slim implementation skill of the repository you are working in.
4. If the repository is not checked out next to `UShell.Docs`, read the skills from
   `https://github.com/ProjectUShell/UShell.Docs/tree/master/.agents/skills`.

Every domain skill is structured the same way:

```text
## <Topic>
<Target state>                         normative, timeless, the "should"

### Status quo (snapshot <date>, <SHAs>)  what the code does today
### Refactoring & compatibility          what has to change and how to stay compatible
```

Rules for this structure:

- The **target state** is normative. Write it so it stays true after the refactoring is done.
- The **status quo** is evidence with a date and commit SHAs. It ages; re-verify against code before relying on it.
- When a refactoring is completed, delete the status/refactoring block and keep only the target state.
- The cross-repository work list lives in `doc/backlog.md`. Skills reference backlog IDs (`UB-xx`).

## 2. Vision

U-Shell is a universal application shell. Professional web applications are composed from

- declarative application descriptions (portfolio, modules, workspaces, use cases, commands),
- reusable, business-free generic components (CommonComponents),
- decentrally hosted modules,
- and, only where truly necessary, custom widgets.

The central idea is **maximum inversion of control**: the shell owns orchestration. Bootstrap,
module aggregation, navigation, workspaces, use-case lifecycle, use-case state (UnitOfWork),
commands, runtime tags, application scope, persistence/restore, authentication, data and
service resolution, logging integration, dialogs and confirmations, and keyboard/focus behavior
are shell concerns. Improving a behavior once in the shell lets every module benefit.

A widget is a **custom control** plugged into a shell-owned use case, not an autonomous SPA
shown in a tab.

The value U-Shell adds beyond "show a widget in a tab" is the **dynamics**: how use cases
interact. A browser selects a record, a command starts an editor and pipes the selection into
the editor's UnitOfWork, the editor returns a result, the ambient application scope flows into
every backend call. All of this is described declaratively. **The configuration owns the
connections; widgets do not glue each other together.**

Guiding question for every design decision:

> Not "how quickly can I build this in React?" but "how should the U-Shell describe and own
> this use case, and what does the widget really need to contribute?"

## 3. Core abstraction model

Description time:

```text
Portfolio                       one product, loaded by one shell runtime
└── Module (0..n, remote or local)
    ├── Workspaces
    ├── UseCase descriptions     widget + UnitOfWork defaults
    ├── StaticUsecaseAssignments
    ├── Commands
    ├── Datastores / Datasources / Services
    └── Widget references        built-in, CommonComponent, federated, (WebComponent)
```

Runtime:

```text
Shell
├── RuntimeTags                  boolean feature/UI predicates
├── ApplicationScope             persisted ambient context (tenant, location, ...)
├── Token sources                authentication state per token source
└── Workspace
    └── UseCase instance
        ├── UnitOfWork           the complete meaningful state of this instance
        ├── lifecycle            shell-owned
        ├── scoped commands      shell-owned toolbar
        └── Widget               deep inside the ownership hierarchy
```

## 4. Glossary

| Term | Meaning |
|---|---|
| Portfolio | One product/application aggregation loaded by the host shell. Exactly one is active. A chooser portfolio may precede it. |
| Module | Decentralized description contributing workspaces, use cases, commands, data/service declarations and widget references. A module is **not** a React app. It may be 100 % declarative. |
| Module bundle | Optional remote frontend artifact (federated React app) that provides widgets and/or data providers for modules. Covered by `ushell-module-development`. |
| Workspace | Long-lived working context ("desk"), e.g. dashboard, customers, settings. |
| UseCase | Bounded working/transaction scope inside a workspace with its own runtime state. |
| Static UseCase | Permanently present, non-terminable use case of a workspace. |
| Dynamic UseCase | Use case instance with a lifecycle, started by commands. |
| UnitOfWork (UoW) | Shell-managed, serializable state of one use-case instance. |
| Command | Shell-described action with placement, enablement, visibility, confirmation and a command type. |
| `mapDynamic` | Controlled mapping from well-defined providers (`unitOfWork://`, `applicationScope://`, ...) to target paths. Not an expression language. |
| RuntimeTag | Boolean runtime predicate (feature toggle, expert mode, permission-derived flag). |
| ApplicationScope | Persisted, application-wide ambient metadata, propagated to backends via side channels. |
| WidgetHost | Controlled bridge from a widget back into shell capabilities. |
| CommonComponent | Generic, business-free UI or data building block. |
| Server command | Asynchronous, server-executed action, started and observed through the shell. |
| Host API | JavaScript API through which an embedding host (.NET WebView) controls and observes the shell. |
| Token source | Configured origin of an access token (`tokenSourceUid`), issued and validated per configuration. |

The three state concepts must never be mixed:

```text
UnitOfWork        -> state of one use-case instance
ApplicationScope  -> ambient context of the running application
RuntimeTags       -> boolean runtime / feature / UI predicates
```

## 5. Context map and ownership

| Repository | Bounded context | Owns the truth about |
|---|---|---|
| `UShell.Docs` | Architecture | Target state, glossary, cross-repository rules, backlog (this skill set) |
| `UShell.PortfolioDescription` | Declarative language | Shape of portfolio/module/workspace/use-case/command/scope/datasource/token descriptions (.NET + `ushell-portfoliodescription` npm) |
| `UShell.ModuleBase.JS` | Widget contract | `IWidget`, `IWidgetHost`, `UsecaseState`, datasource/datastore interfaces (`ushell-modulebase` npm) |
| `UShell.ReactFrontend` | Shell runtime ("harness") | How the shell fulfills the contracts: loading, menus, workspaces, command execution, UoW piping, persistence, token handling, scope propagation, Host API |
| `UShell.CommonComponents.JS` | Generic building blocks | Business-free UI components, data connectors (FUSE, service proxy), shell-layout primitives (`ushell-common-components` npm) |
| `UShell.ServerCommands.Contract` | Server execution | Executor contract and ASP.NET Core hosting for server commands |
| `UShell.FrontendBundle` | Packaging | Hosting the frontend from a .NET assembly |

Adjacent technologies with their own documentation: FUSE-fx RepositoryContract (generic data
access), UnifiedJsonMessageWrapper / UJMW (RPC and side channels), AuthTokenHandling (token
semantics), SmartAmbience (ambient context on the server), UniversalBFF (recommended hosting
direction).

**Ownership rule for cross-cutting topics:** semantics are defined where they are declared,
behavior is defined where it is implemented. Example: token renewal is *promised* in the widget
contract (ModuleBase), *configured* in PortfolioDescription and *implemented* in ReactFrontend
and CommonComponents. Every other place only links.

**Layer rule:**

- Something that has to be declared in a portfolio or module goes to PortfolioDescription.
- Something a widget may rely on goes to ModuleBase.
- Orchestration and anything needing global shell knowledge goes to ReactFrontend.
- Reusable, business-free UI or transport logic that also works outside the shell (stand-alone module debugging, `LocalWidgetHost`) goes to CommonComponents.
- Business knowledge never goes into any of these repositories.

## 6. Authority and compatibility rules

1. **Target state:** this skill set leads.
2. **Current naming and micro-implementation:** the productive code and contracts lead. Use their names; do not invent aliases.
3. **Historical material** (`core-idea-kornsw.md`, `epics/`, Angular-era prototypes, CtExpert legacy) is evidence of intent, not an automatic requirement.
4. **Adjacent technologies:** their own repositories and skills lead.
5. **Divergence:** when target state and code disagree, do not silently "fix" either side. Record it in the status block and backlog, assess compatibility, and ask before architecture-affecting changes.

**Never break existing consumers.** Portfolios, modules and remote widgets in production
depend on the current contracts. Therefore:

- Contract changes are additive: new optional properties, new command types, new provider channels.
- A semantic change of an existing field needs an opt-in (new field, new value, or version marker) and keeps the old behavior as default.
- Legacy behavior that has to go is kept as a documented alias first (e.g. `iconName == "modal"`), then deprecated, then removed with a major version.
- .NET and JS/TS representations of the description contract change together.
- Persisted browser state (UoW, ApplicationScope, tokens) must stay readable after an upgrade, or be migrated on load.

## 7. Agent decision process for new requirements

1. **Identify the use-case graph:** workspaces, static and dynamic use cases, lifecycles, source and target UoWs, commands, ApplicationScope, RuntimeTags, data/service dependencies.
2. **Try pure configuration** with existing descriptions and CommonComponents.
3. **Inspect CommonComponents** before writing any widget.
4. **Consider a new generic CommonComponent** if the capability is domain-neutral. It may start rough; it must stay business-free.
5. **Custom widget** only for genuine specialization. It still uses the WidgetHost and keeps state in the UoW.
6. **Large embedded external UI** only as a documented escape hatch (legacy, migration, third party).

If a step fails because the platform lacks a generic capability, record a backlog item
instead of building local infrastructure.

## 8. Anti-patterns

- **Giant SPA in a static use case**: the shell degenerates to a launcher.
- **Imperative glue between use cases**: express flows through commands and `mapDynamic`.
- **Arbitrary expressions or JavaScript in descriptions**: destroys analyzability and future visual modeling.
- **Widget-owned state**: meaningful selection/filter/edit state that is lost when the widget is recreated belongs in the UoW.
- **Widget-owned authentication or hard-coded service URLs**: use token sources and declared datasources/services.
- **Business-specific CommonComponents.**
- **Ambient context as business parameters**: ApplicationScope travels through side channels.
- **Loading `portfolio.json` / `module.json` via RPC**: they are plain HTTP GET resources, static hosting must stay possible.

## 9. Skill index (UShell.Docs/.agents/skills)

| Skill | Load when |
|---|---|
| `ushell-architecture` | Always (this skill) |
| `ushell-declarative-model` | Designing or changing portfolio/module/workspace/use-case/command descriptions, command types, instance identity, appearance, RuntimeTags, `mapDynamic` semantics |
| `ushell-interaction-dynamics` | Anything about how use cases interact: UoW lifecycle, piping between use cases, result return, `userInput`, dirty state, events, persistence and restore |
| `ushell-widget-contract` | Writing widgets, changing `IWidget` / `IWidgetHost` / `UsecaseState`, widget-to-shell communication |
| `ushell-data-auth-scope` | Datastores, datasources, services, FUSE access, token sources, seamless token refresh, ApplicationScope propagation over UJMW side channels |
| `ushell-server-commands` | Server-executed asynchronous actions, their command type, progress UI and backend contract |
| `ushell-host-api` | Embedding the shell into a .NET WebView (or another host) and controlling it from outside via JavaScript |
| `ushell-shell-runtime` | Requirements for the ReactFrontend harness layer: what the shell itself must implement |
| `ushell-common-components` | Requirements for the CommonComponents layer: what is generic, what is UI-specific, what must not go there |
| `ushell-module-development` | Building a new U-Shell module from scratch (declarative module, module bundle with widgets and/or data providers) |
| `ushell-legacy-integration` | Evolving existing (legacy) implementations onto the target state, integrating legacy applications, porting features from older generations (Angular prototype, CtExpert) |

## 10. Human documentation

- `readme.md`: landing page and map of this repository.
- `doc/backlog.md`: cross-repository backlog derived from the status blocks.
- `core-idea-kornsw.md`: original concept (2019). Frozen; externally deep-linked; do not rename or move.
- `epics/`: historical concept epics (`mapDynamic`, token configuration). Their normative content is carried forward by the domain skills.

## 11. Maintenance contract

- Update incrementally and keep section structure stable so diffs show semantic changes.
- New architectural knowledge discovered during implementation goes back into the matching domain skill, not only into code.
- Open questions stay visible in status blocks and the backlog until decided. Move them into the target state only once decided.
- Keep implementation-repository skills slim. When they start to contain design knowledge, move it here.
- All skill and documentation content is written in English.
