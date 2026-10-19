---
name: ushell-hosting
description: How U-Shell artifacts become hostable (target state plus current status and refactoring) - the hostability contract the shell bundle and every module bundle must fulfill on the React level (base path independence, portfolio location, publicPath, module.json shipped by the module's React project, URL resolution relative to the module.json, SPA fallback, caching, origin independence), the canonical module layout, and how the hosting variants map onto it - UniversalBFF as the main delivery path (assemblies with embedded React artifacts, module providers), static hosting, UShell.FrontendBundle.Mvc and WebView desktop hosts. Load when building or changing how the shell or a module bundle is built, packaged, embedded or served, when writing a module provider for UniversalBFF, or when touching portfolio/module URL resolution. Requires ushell-architecture; usually together with ushell-module-development.
---

# U-Shell hosting integration

Snapshot for status blocks: 2026-10-19, ReactFrontend `dd373ad`, ModuleBase.JS `df6a0f4`,
PortfolioDescription `7bf83f4`, FrontendBundle `e265b2e`, UniversalBFF `b7a7c62`.

## 1. Scope and boundary

This skill owns the **hostability requirements of U-Shell artifacts on the React level**:
what the shell bundle and a module bundle must do (and must not assume) so that any host can
serve them. The ownership rule applies: the artifact defines what it needs; a host fulfills it.

It does **not** document hosts. In particular **UniversalBFF** is the recommended and main
delivery path, but it has its own documentation and skills, which lead for everything about
its module providers, registrars, routing, security and product definitions:

- `KornSW/UniversalBFF`, `doc/2-architecture.md`
- `KornSW/UniversalBFF`, `.agents/skills/most-important-details-for-universalbff/SKILL.md`

Here UniversalBFF only appears as the reference host: which of its places fulfill which part
of the contract. Do not copy its internals into this skill set; link them.

The JavaScript control surface for embedding hosts (scope, tokens, commands, events) is
`ushell-host-api`. This skill covers how the files get there.

## 2. Artifacts

| Artifact | Built by | Content | Hosted as |
|---|---|---|---|
| Shell bundle | `UShell.ReactFrontend/ushell-ts` | `index.html`, chunks, `remoteEntry.js` | static files; packaged for .NET as `UShell.FrontendBundle` (embedded resources) |
| Module bundle | the module's own React project | `remoteEntry.js`, chunks, optional stand-alone `index.html`, **`module.json`** | static files; for UniversalBFF embedded into the module's assembly |
| `portfolio.json` | host (file or generated) | product composition, module URLs | plain HTTP GET |
| Backends | module's .NET project | UJMW / FUSE endpoints | out of scope here, see `ushell-data-auth-scope` |

All frontend artifacts are **plain static files**. Every hosting variant (section 6) is only a
different way of serving them. No artifact may depend on server-side rendering or on a
specific host product.

## 3. Shell hostability contract

The shell bundle must run unchanged under every host. Requirements:

- **H-1 Base path independence.** The shell runs under any base path (`/`, `/products/x/`,
  `app://local/`). It derives every own URL from `document.baseURI`. It is built with
  `<base href="/">` and a relative `publicPath`; it contains no root-absolute URLs to its own
  assets, routes or default files.
- **H-2 Host injection is minimal.** A host injects at most: the base href, the portfolio
  location (`<meta name="portfolioLocation" content="...">`) and an HTML title fallback.
  Everything else (branding, modules, auth, scope) comes from the portfolio. Without the meta
  tag the shell uses `{base}portfolio` (static-hosting convention).
- **H-3 Tolerant injection points.** The injection points are well defined and tolerant:
  hosts match the `<base>` element by pattern (whitespace and self-closing variations), not by
  exact string. The shell must not rely on minification to make a host's string match work.
- **H-4 Deep links.** The shell mount needs SPA fallback (unknown paths below the mount return
  `index.html`). Module mounts do not need it unless they ship a stand-alone app.
- **H-5 Origin independence.** No assumption about scheme, host or port. Custom schemes of
  desktop hosts (`app://local`) and in-memory hosting must work. Cross-origin module bundles
  are allowed (CORS is a host duty, section 6).
- **H-6 Caching.** `index.html`, `portfolio.json` and `module.json` are revalidated on every
  load (`no-cache`); hashed chunks may be cached long-term. `remoteEntry.js` is not hashed and
  must be revalidated, otherwise module updates do not arrive.
- **H-7 Traceability.** The delivered `index.html` carries a version marker (HTML comment) of
  the shell bundle and its package, so a running deployment can be identified.

### Status quo
- H-1: `App.tsx` reads `meta[name="portfolioLocation"]`, otherwise `pickBasePath() + "portfolio"`. `ushell-ts/webpack.config.js` uses `publicPath: ""`.
- H-3: `UShellBundleFileProvider` (`UShell.FrontendBundle/dotnet/src/FrontendBundle.Mvc/`) replaces the exact string `<base href="/"/>` and appends the `portfolioLocation` meta at the same place. It only works because the built `index.html` is minified to that exact form; the source `ushell-ts/src/index.html` contains `<base href="/" />`. UniversalBFF's own `EmbeddedBundleFileProvider` (used for module bundles) already uses a regex.
- H-6: no cache headers are set for the shell bundle; behavior depends on the host defaults.
- H-7: implemented in `UShellBundleFileProvider` (comment with DLL and JS version).

### Refactoring & compatibility
- UB-80: pattern-based injection in `UShellBundleFileProvider` (base href + meta), independent of minification. Additive.
- UB-81: cache-header defaults for embedded bundles (shell and modules) as a documented host duty, implemented in FrontendBundle.Mvc; handoff to UniversalBFF.

## 4. Module bundle hostability contract

- **M-1 publicPath.** A module bundle is built with webpack `publicPath: "auto"`, so the
  federation runtime derives the chunk base from the URL of `remoteEntry.js`. No host
  rewriting is needed. The placeholder `_PUBLIC_PATH_` (replaced by hosts in `.js` and `.html`
  files) is a supported **legacy path** and must keep working.
- **M-2 module.json is a React artifact.** The module's React project owns its `module.json`
  (source e.g. `public/module.json` or `src/module.json`), versioned together with the widgets
  it references. The build emits it into the output. It is the single description of the module.
- **M-3 Host-bound values may be supplemented.** A host may supplement or override only
  host-bound values when serving the module description; it never removes or rewrites
  workspaces, use cases, commands or their semantics. Host-bound values are:
  - `moduleUid` if empty,
  - token source assignments (`tokenSourceUid`) and auth-related declarations of the product,
  - absolute URLs for backends that are not co-located with the module (section 5),
  - product-specific metadata the declarative model defines for that purpose.
  The exact overlay set and its mechanism is decision D-09.
- **M-4 Relative URLs.** All URLs inside `module.json` (federated widget `url`, datastore,
  datasource and service URLs, module-relative resources) may be relative. They are relative
  to **the URL the `module.json` was loaded from**, resolved with standard URL semantics
  (`new URL(relative, moduleJsonUrl)`). Absolute URLs stay unchanged.
- **M-5 Shared singletons.** `react`, `react-dom`, `ushell-modulebase` and
  `ushell-common-components` are shared as singletons (see `ushell-module-development`). A
  bundle must not ship a second React into the page.
- **M-6 Unique federation name.** The federation `name` (the `scope` in `widgetClass`) is
  unique per module, ideally derived from the `moduleScopingKey`. Copy-pasted names collide
  silently in the shell's container registry.
- **M-7 No host knowledge in widgets.** Widgets never compute URLs from `window.location`,
  never read backend URLs from the UnitOfWork and never hard-code mounts. They use declared
  datasources/services and `tryGetModuleBaseUrl` for module-relative resources.

### Status quo
- M-1: the UniversalBFF user-management bundle uses `publicPath: "_PUBLIC_PATH_"`; UniversalBFF's `EmbeddedBundleFileProvider` replaces it with the mount URL.
- M-2/M-3: module descriptions served by UniversalBFF are currently built in C# (`UserManagementModuleProvider.RegisterModule`, with placeholder `WidgetClass = "demo"`). A commented-out line shows the intended `ModuleDescription.FromEmbeddedResFile(...)`. The module's `.csproj` explicitly excludes `webapp-files\*.json` from embedding.
- M-4: the shell resolves only the module.json URLs themselves (relative to the portfolio location, `PortfolioLoader.getFullModuleUrl`). It then merges all module descriptions into one and forgets each module's source URL. The federated widget `url` is used as written; relative values resolve against the document, not the module.
- M-6: the user-management bundle exposes `name: "srViewer"` (copied from another project).
- M-7: `tryGetModuleBaseUrl` is listed in the widget contract but implemented neither in `ushell-modulebase` nor in ReactFrontend. The user-management wrapper sets `BackendService.baseUrl` from `unitOfWork.backendApiUrl`.

### Refactoring & compatibility
- UB-82: keep the source URL per loaded module description; resolve relative URLs inside it (M-4). ⚠ Modules in the field might use relative widget URLs that today resolve against the shell document; check productive modules first, otherwise gate behind a module-level version marker.
- UB-83: implement `tryGetModuleBaseUrl` on top of UB-82 (MB + RF).
- UB-84: module starter template (UB-52) uses `publicPath: "auto"`, ships `module.json` and the canonical layout (section 5).
- Hosts keep replacing `_PUBLIC_PATH_` for existing bundles.

## 5. Canonical module layout

Every host exposes a module under one **module base URL**, with this layout:

```text
{moduleBase}/module.json           module description (M-2)
{moduleBase}/ui/...                module bundle (remoteEntry.js, chunks, assets)
{moduleBase}/api/v{n}/{alias}      co-located backend endpoints (optional)
```

Inside `module.json` the module therefore references `ui/remoteEntry.js` and
`api/v1/<alias>`, independent of where the host mounts it. This is exactly the UniversalBFF
shape (`{BaseUrl}{moduleScopingKey}/module.json`, extension alias `ui`, UJMW route
`{moduleScopingKey}/api/v{apiV}/{endpointAlias}`), so the main path needs no rewriting.

The React project's build output mirrors the layout (`dist/module.json` + `dist/ui/**`), so
static hosting is "copy `dist` to `{moduleBase}`". Whether the layout is mandatory or only
recommended is decision D-10.

Backends that are not co-located (static hosting, external services) are referenced
absolutely; that value is host-bound and may be supplemented by the host (M-3).

## 6. Hosting variants

### 6.1 UniversalBFF (main path)

The module ships as one assembly: backend services, a module provider, and the React build
output as embedded resources. UniversalBFF discovers the provider, registers the backend
endpoints, mounts the embedded files and serves `module.json` and `portfolio.json`.

Where UniversalBFF fulfills the contract (pointers only, details in its repository):

| Contract part | UniversalBFF place |
|---|---|
| Shell bundle, base href, `portfolioLocation` (H-1, H-2) | `dotnet/src/UniversalBFF.AspNetCore/Programm.cs` (`SetupSpaMultiHosting` with `UShellBundleFileProvider`) |
| Module bundle mount `{key}/ui/`, `_PUBLIC_PATH_` replacement (M-1) | `IFrontendModuleRegistrar.RegisterFrontendExtension`, `AspSupport/[EmbeddedFileHandling]/EmbeddedBundleFileProvider*.cs` |
| `module.json` at `{key}/module.json` (M-2, M-3) | `IFrontendModuleRegistrar.RegisterModule`, `ModuleRegistrar` (`IPortfolioService`), `PortfolioController` from `UShell.PortfolioDescription/PortfolioHosting.Mvc` |
| Product composition into `portfolio.json` | `IProductDefinitionProvider`, `ModuleRegistrar.Init.cs` |
| Co-located backends `{key}/api/v{n}/...` | `IBackendServiceRegistrar.RegisterUjmwServiceEndpoint` |
| Reference module | `dotnet/src/UniversalBFF.OobModules.UserManagement` + `js/UniversalBFF.OobModules.UserManagement` |

Expectations of U-Shell towards this path (to be fulfilled on the UniversalBFF side):

- The module provider registers the **embedded `module.json` of the React build** and only supplements host-bound values (M-3). It does not assemble workspaces, use cases or commands in C#.
- The React build output is embedded in every configuration (not only Debug) and includes `module.json`.
- The embedded-namespace pitfalls (case-sensitive, `-` becomes `_`) are UniversalBFF knowledge; see its skill.

### 6.2 Static hosting

Shell and modules as plain files (CDN, nginx, IIS static site). The shell folder contains a
`portfolio/` folder (or the page sets the `portfolioLocation` meta); each module is a copied
`dist` in the canonical layout. The host must configure SPA fallback for the shell mount
(H-4), cache headers (H-6) and CORS for cross-origin module bundles. No placeholder
replacement happens, so module bundles must use `publicPath: "auto"` (M-1).

### 6.3 UShell.FrontendBundle.Mvc without UniversalBFF

An own ASP.NET Core application references the `UShell.FrontendBundle` package and hosts the
shell via `UShellBundleFileProvider` / `UShellHostingSetupExtensions`, typically with
`PortfolioHosting.Mvc` for portfolio and module descriptions (example:
`UShell.FrontendBundle/dotnet/src/FrontendBundle.DemoWebService`). Module bundles are served
statically or from other hosts. Use this only when UniversalBFF does not fit; it duplicates
composition that UniversalBFF provides.

### 6.4 WebView desktop hosts

A desktop application runs the same ASP.NET Core pipeline in memory and serves it to a
WebView under a custom scheme (`UniversalBFF.WebViewHost.net10.0`, default `app://local`), or
uses a CEF-based wrapper (`UShell.ReactFrontend.CefWrapper`). For the artifacts nothing
changes beyond H-5 (origin independence). Controlling the shell from the host is
`ushell-host-api`.

## 7. Checklist for a hostable module

- [ ] Bundle built with `publicPath: "auto"` and a unique federation name.
- [ ] `module.json` lives in the React project and is emitted by the build.
- [ ] All URLs in `module.json` are relative to the module base or genuinely external.
- [ ] Build output follows the canonical layout (`module.json`, `ui/`).
- [ ] No widget reads URLs from `window.location` or the UnitOfWork.
- [ ] For UniversalBFF: the provider registers the embedded `module.json` and adds only host-bound values.
