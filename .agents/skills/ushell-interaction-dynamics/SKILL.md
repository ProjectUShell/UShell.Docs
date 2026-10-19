---
name: ushell-interaction-dynamics
description: How U-Shell use cases interact at runtime (target state plus current status and refactoring) - UnitOfWork lifecycle and ownership, piping values and states from a source use case into new use cases via commands and mapDynamic, returning results to the caller, use-case actions, user input during flows, dirty/commit semantics, events, persistence and session restore. This is the core of U-Shell's value beyond hosting widgets. Load for any work on flows between use cases or on WorkspaceManager/ArgumentMapper/WidgetHost behavior. Requires ushell-architecture and ushell-declarative-model.
---

# U-Shell interaction dynamics

Bounded context spans the widget contract (`UShell.ModuleBase.JS`) and the shell runtime
(`UShell.ReactFrontend`). Declarative syntax is in `ushell-declarative-model`.

Snapshot used for all status blocks: 2026-10-19, ReactFrontend `c71aa36`, ModuleBase.JS `6d72d1c`.

## 1. The UnitOfWork is the state

Every use-case instance owns exactly one UnitOfWork (UoW), owned by the shell:

- **Created** from `UnitOfWorkDefaults` (resolved via `mapDynamic`) and overlaid with the `InitUnitOfWork` of the starting command or static assignment.
- **Handed** to the widget on initialization (`IWidget.state.unitOfWork`).
- **Changed** by the widget and published through `IWidgetHost.populateChangedState`. The shell is the single writer of the persisted copy.
- **Persisted** continuously so that a browser restart restores all open use cases with their UoWs.
- **Terminated** with the instance. Static use cases are never terminated.

What belongs into the UoW: everything whose loss on restore would change what the user was
working on. Examples: selected record, filter, sort order, wizard step, active sub-view,
master/detail selection, not yet committed input, dirty marker. Purely ephemeral rendering
state may stay local. Secrets never go into the UoW.

The UoW must be JSON-serializable.

### Status quo
- Implemented in `WorkspaceManager`: creation from defaults + init data, `updateUsecaseState`, persistence per workspace in `localStorage` (`<applicationTitle>|<workspaceKey>`), restore on load, terminate.
- `UsecaseState` (ModuleBase) carries `title`, `usecaseInstanceUid`, `usecaseKey`, `parentWorkspaceKey`, `fixed`, `unitOfWork`.

## 2. Execution context of a command

Every command execution happens in an **execution context**:

```text
ExecutionContext
├── sourceUsecaseInstance   the use case the command was triggered from (may be none)
├── sourceUnitOfWork        its current UoW        -> unitOfWork://
├── commandArgs             arguments of the call  -> commandArgs://
└── applicationScope, settings, runtime tags (global)
```

The source is determined by **where** the command was triggered, never by the widget:

| Trigger | Source use case |
|---|---|
| Toolbar of a use case (`MenuOwnerUsecaseKey`) | that use case |
| Widget calls `widgetHost.fireEvent(commandKey, args)` | the use case hosting that widget; `args` become `commandArgs` |
| Global menu / user menu | none (`unitOfWork://` resolves to unset) |
| Host API | as specified by the host call (default none) |
| Command fired from a modal use case | the modal use case |

This is what makes flows declarative: the browser does not know the editor, the editor does
not know the browser, **the configuration owns the connection**.

```text
Browser (static)                           Editor (dynamic)
UoW.selectedCustomer.id ──┐
                          │ command "edit": start-usecase
                          │ InitUnitOfWork.mapDynamic:
                          │   unitOfWork://selectedCustomer.id -> customerId
                          └───────────────────────────────────> UoW.customerId
```

### Status quo
- `WorkspaceManager.executeCommand(command, input, sourceUseCaseUow)` has the right shape.
- **Only the use-case `CommandBar` passes the source UoW.** `WidgetHost.fireEvent`, the global menu and `tryStartGuifad` pass `{}`. So `unitOfWork://` is empty whenever a widget fires a command, which is the most common case for dynamic flows.
- `fireEvent` passes the widget's `args` as `input`, so `commandArgs://` already works from widgets.
- The source use-case instance itself (not just its UoW) is not passed on, so nothing can be returned to it.

### Refactoring & compatibility
- UB-01: the `WidgetHost` instance handed to a widget must be bound to that widget's use-case instance and pass instance and current UoW into `executeCommand`. Purely additive; commands that do not use `unitOfWork://` behave as before.
- UB-01: replace the positional `sourceUseCaseUow` parameter with an execution-context object internally. Keep the old signature as an overload while modules call it directly (`globalWorkspaceManager` is exposed on `globalThis`).

## 3. Returning results to the caller

A dynamic use case started from a source use case can **return a result**: a selection
dialog, a "create new" editor returning the new ID, a lookup returning a record. The return is
declared on the **starting command**, symmetric to `InitUnitOfWork`:

```text
start-usecase command
├── InitUnitOfWork        source -> target   (unitOfWork://, commandArgs://, ...)
└── ApplyResult           target -> source   (result:// reads from the finished target UoW
                                              or the explicit result object)
```

- A use case finishes with a result through the shell action `endUsecase` (with result) or `commit`. Cancel/discard returns no result and applies nothing.
- `ApplyResult` is a dynamic parameter object whose `mapDynamic` entries write into the **source UoW**. The shell then publishes the changed source UoW to its widget.
- If the source use case was terminated in the meantime, the result is dropped and logged.

This is a design proposal. Attribute names (`ApplyResult`, `result://`) are not yet fixed.

### Status quo
- Not implemented generically. The `scopeSelection` widget is hard-wired in `WorkspaceManager.renderWidget` to call `switchScope` on selection, a special case of this pattern.
- Modal use cases are terminated via `terminateModal` without a result channel.

### Refactoring & compatibility
- UB-02: add the result channel to the contract (additive optional attribute on `CommandDescription`, new `result://` channel) and to the runtime. Re-implement `scopeSelection` on top of it later; keep the hard-wired path until then.

## 4. Use-case actions

A widget can expose named actions (`selectAll`, `resetFilter`, `save`, ...). The shell
exposes well-known actions for every use case (`endUsecase`, `commit`, `discard`, `refresh`).
A `usecase-action` command invokes them. Arguments come from a dynamic parameter object, so
actions can receive values from the UoW or ApplicationScope.

This keeps toolbars shell-owned (placement, enablement, confirmation, keyboard shortcuts)
while the widget implements the behavior.

Widget side (target contract, see `ushell-widget-contract`): the widget registers its action
handlers with its WidgetHost. The action list may be dynamic.

### Status quo
- Not implemented. `CommandDescription.ActionName` exists; the runtime throws for `usecase-action`. ModuleBase has no action registration.

### Refactoring & compatibility
- UB-03: additive in ModuleBase (optional registration API) and ReactFrontend (command type).

## 5. User input during a flow

`userInput://name` in any dynamic parameter object makes the shell ask the user for a value
in a shell-owned dialog before executing the command (e.g. a key date for a report). Several
`userInput` entries of one command are collected in one dialog. Cancelling cancels the command.

The dialog is a generic CommonComponent. The input definition (label, type, default) is
declared next to the mapping. The exact syntax is open.

### Status quo
- Not implemented ("this channel is under construction"). Resolution is synchronous today.

### Refactoring & compatibility
- UB-07: requires an asynchronous resolution path in `ArgumentMapper` and `executeCommand`. Keep a synchronous fast path when no `userInput://` is present.

## 6. Dirty state, commit and close

The well-known UoW attribute `changes` (boolean) marks a use case as dirty:

- the shell appends `*` to the tab title,
- closing a dirty use case asks for confirmation (save / discard / cancel),
- `commit` and `discard` are well-known use-case actions,
- restoring a session restores the dirty state.

### Status quo
- Not implemented. Title placeholders are resolved from the UoW (`getTitleForUseCase`), which is a usable base.

### Refactoring & compatibility
- UB-11: additive.

## 7. Events

Commands are the way to **trigger** behavior. Events are the way to **observe** shell state
changes. Widgets may subscribe to shell events through their WidgetHost:

- `applicationScopeChanged`
- `runtimeTagsChanged`
- `unitOfWorkChanged` (the shell changed this widget's UoW, e.g. by an applied result)
- `tokenChanged` (per token source)
- `usecaseActivated` / `usecaseDeactivated`

Widgets do not exchange events with each other directly. Cross-use-case interaction always
goes through commands and the UoW.

### Status quo
- `IWidgetHost.fireEvent(name, args)` executes the **command** with that key. That is fine and stays, but "event" is a misleading name.
- `subscribeEvent` throws "Method not implemented".
- After a scope switch the runtime re-initializes the datastores and rebuilds the menu (`onAppScopeChangedMethod`) but does not notify widgets.

### Refactoring & compatibility
- UB-12: implement `subscribeEvent` with the event set above. Keep `fireEvent` as is; optionally add an alias with a clearer name (`executeCommand`).

## 8. Session persistence and restore

The shell restores after reload or restart:

- open workspaces and their use-case instances,
- each UoW including dirty state,
- the active workspace and use case,
- ApplicationScope values and user-set RuntimeTags,
- token state (according to token source configuration).

Persisted formats are versioned. A newer shell migrates older formats on load; it never
discards user state silently.

### Status quo
- UoWs per workspace and ApplicationScope are persisted in `localStorage`. Formats carry no version. User-set RuntimeTags do not exist yet.

### Refactoring & compatibility
- UB-15: introduce a format version on first change of any persisted structure. Unversioned data is treated as version 0.

## 9. Reference flows (for tests and demos)

1. **Browse -> edit**: static browser, `edit` command in its toolbar, `unitOfWork://selected.id -> recordId`, generic editor.
2. **Widget-fired create with return**: widget fires `create`, modal editor starts, `ApplyResult` writes the new ID back into the browser's `selected.id`.
3. **Key-date report**: global menu command, `userInput://keyDate`, report use case.
4. **Scope-dependent workspace**: workspace requires scope `tenant`; scope selection appears until a tenant is chosen; datasources re-evaluate the side channel.
5. **QR widget port**: a small custom widget (e.g. QR code display/scan) that reads its payload from the UoW and returns a scanned value via `ApplyResult`. The legacy QR widget is to be located (UB-61) and ported as the reference custom widget.
