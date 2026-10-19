# About the 'Project UShell'

This is the idea of building a 'universal' shell in the form of an SPA that can host any web application. The current paradigm of configurable application modules (keyword 'composite application') in combination with the "WebComponents" standard enables us to create a product which is put together from several decentrally hosted components. In addition, an interaction between these modules should be made possible. In addition, it should also be possible to define various standard use cases, such as processing a list of data records in the form of simple CRUD operations, purely configuratively (see the [CORE-CONCEPT](https://github.com/ProjectUShell/UShell.Docs/blob/master/core-idea-kornsw.md)). In the latter case, no frontend development is necessary at all - only a web service in the backend...

# About this Repo

`UShell.Docs` is the **top-down entry point** of Project UShell. It holds the leading
description of the U-Shell **target state** for all repositories of the organization, written
primarily for AI-driven development (Agent Skills format) and readable for humans.

The implementation repositories carry only slim, implementation-specific skills that point
back here.

## Map

| Path | Content |
|---|---|
| [`.agents/skills/ushell-architecture`](.agents/skills/ushell-architecture/SKILL.md) | **Root entry (always load first):** vision, glossary, context map, ownership, authority and compatibility rules, skill index |
| [`.agents/skills/ushell-declarative-model`](.agents/skills/ushell-declarative-model/SKILL.md) | Portfolio/module description language: workspaces, use cases, commands, instance identity, RuntimeTags, ApplicationScope definitions, `mapDynamic` |
| [`.agents/skills/ushell-interaction-dynamics`](.agents/skills/ushell-interaction-dynamics/SKILL.md) | How use cases interact: UnitOfWork, piping, result return, actions, user input, dirty state, events, restore |
| [`.agents/skills/ushell-widget-contract`](.agents/skills/ushell-widget-contract/SKILL.md) | `IWidget` / `IWidgetHost` / `UsecaseState` |
| [`.agents/skills/ushell-data-auth-scope`](.agents/skills/ushell-data-auth-scope/SKILL.md) | Datastores/datasources/services, token sources, seamless token refresh, ApplicationScope via UJMW side channels |
| [`.agents/skills/ushell-server-commands`](.agents/skills/ushell-server-commands/SKILL.md) | Asynchronous server-executed commands |
| [`.agents/skills/ushell-host-api`](.agents/skills/ushell-host-api/SKILL.md) | JavaScript Host API for embedding into .NET WebView hosts |
| [`.agents/skills/ushell-hosting`](.agents/skills/ushell-hosting/SKILL.md) | Hostability contract of shell and module bundles, canonical module layout, delivery via UniversalBFF, static hosting, FrontendBundle.Mvc, WebView |
| [`.agents/skills/ushell-shell-runtime`](.agents/skills/ushell-shell-runtime/SKILL.md) | Layer requirements for the ReactFrontend harness, consolidated status |
| [`.agents/skills/ushell-common-components`](.agents/skills/ushell-common-components/SKILL.md) | Layer requirements for CommonComponents |
| [`.agents/skills/ushell-module-development`](.agents/skills/ushell-module-development/SKILL.md) | Building new modules and module bundles from scratch |
| [`.agents/skills/ushell-legacy-integration`](.agents/skills/ushell-legacy-integration/SKILL.md) | Evolving and integrating legacy implementations, findings from earlier generations |
| [`.agents/skills/ushell-design-debt`](.agents/skills/ushell-design-debt/SKILL.md) | Inventory of known design debt and undocumented conventions of the productive implementation (`DD-xx`) |
| [`doc/backlog.md`](doc/backlog.md) | Cross-repository backlog and open design decisions |
| [`core-idea-kornsw.md`](core-idea-kornsw.md) | Original concept (2019), frozen, externally deep-linked |
| [`epics/`](epics/) | Historical concept epics (`mapDynamic`, token configuration) |
| [`docs/`](docs/) | Older notes |

## Repositories

| Repository | Role |
|---|---|
| [UShell.PortfolioDescription](https://github.com/ProjectUShell/UShell.PortfolioDescription) | Declarative description contract (.NET + npm `ushell-portfoliodescription`) |
| [UShell.ModuleBase.JS](https://github.com/ProjectUShell/UShell.ModuleBase.JS) | Widget contract (npm `ushell-modulebase`) |
| [UShell.ReactFrontend](https://github.com/ProjectUShell/UShell.ReactFrontend) | Shell runtime (harness) |
| [UShell.CommonComponents.JS](https://github.com/ProjectUShell/UShell.CommonComponents.JS) | Generic components and data connectors (npm `ushell-common-components`) |
| [UShell.ServerCommands.Contract](https://github.com/ProjectUShell/UShell.ServerCommands.Contract) | Server command executor contract and ASP.NET Core hosting |
| [UShell.FrontendBundle](https://github.com/ProjectUShell/UShell.FrontendBundle) | Hosting the frontend from a .NET assembly |

## Working with AI agents

Open this repository next to the implementation repositories (same parent folder). Agents
start with `ushell-architecture`, load the matching domain skills, and then the slim skill of
the repository they change. Each domain skill separates the **target state** from a dated
**status quo** and the required **refactoring**. Never break existing consumers: every change
to contracts and runtime behavior must stay backwards compatible.

U-Shell is in productive use and was built with a focus on getting finished, so the code and
the target state differ in places. Those differences are inventoried in `ushell-design-debt`:
load it whenever you write against the running shell rather than against the target state. It
is a running inventory, not a to-do list - the backlog decides what gets resolved and when.
