---
name: ushell-server-commands
description: U-Shell server commands (target state plus current status and refactoring) - generic, configurable, asynchronous server-side actions that the shell can start, observe (progress, polling, cancellation) and gate (availability, permissions) without custom UI code. Covers the frontend command type, the declaration in module descriptions, the generic progress UI in CommonComponents and the backend contract in UShell.ServerCommands.Contract. Load for any "run something on the server" requirement. Requires ushell-architecture and ushell-declarative-model.
---

# U-Shell server commands

Backend contract and hosting: `UShell.ServerCommands.Contract` (.NET, ASP.NET Core
integration). Frontend: command type in the declarative model, execution in ReactFrontend,
progress UI in CommonComponents.

Snapshot for status blocks: 2026-10-19, ServerCommands.Contract `bfa24c2`, ReactFrontend
`c71aa36`, PortfolioDescription `7bf83f4`.

## 1. Target behavior

A module declares that a server-side action exists and where it is executed. Users trigger it
like any other command (menu, toolbar, widget, Host API):

```text
command (type server-command)
  -> resolve arguments (dynamic parameter object: unitOfWork://, applicationScope://, userInput://, ...)
  -> confirmation (WarningToConfirm) if declared
  -> CanStartExecution? -> otherwise show reason, do not start
  -> StartExecution(commandName, string[] args, syncWaitMs)
  -> outcome immediately, or poll GetLatestExecutionState (respect NextPollNotBefore)
  -> progress in a shell-owned, non-blocking progress surface (StatusMessage, CurrentStep/TotalSteps)
  -> cancel if CancellationPossible -> RequestCancellation
  -> final state: Completed / Canceled / Failed / Rejected*
  -> optional follow-up: ApplyResult into the source UoW, refresh a datasource, run another command
```

Rules:
- Execution is always asynchronous. The UI never blocks; the user may continue working.
- Running executions survive navigation and are shown in a central place (e.g. a notification/progress area). Restoring after reload is desirable: persist `executionId`s and resume polling.
- Availability (`CanStartExecution`) can feed the command's enabled state and an overlay reason (concurrency lock, threshold lock, no permission, unavailable).
- The endpoint is a declared service (`ServiceDescription`) with token source and ApplicationScope side channel, so authentication and ambient context work as for every other call (`ushell-data-auth-scope`).
- Arguments are a string array per the contract. Rich payloads are not the purpose; pass record IDs and let the server load data.

## 2. Declaration (target, names not final)

```json
{
  "uniqueCommandKey": "recalculateInvoices",
  "label": "Recalculate",
  "commandType": "server-command",
  "menuOwnerUsecaseKey": "invoiceBrowser",
  "warningToConfirm": "Recalculate all selected invoices?",
  "serverCommand": {
    "serviceUid": "billing-commands",
    "commandName": "RecalculateInvoices",
    "arguments": {
      "ids": null,
      "mapDynamic": [ { "use": "unitOfWork://selectedIds", "for": "ids" } ]
    },
    "refreshDatasourcesOnCompletion": [ "Invoice" ]
  }
}
```

Open decisions: attribute names; mapping of a dynamic parameter object onto the positional
`string[]` (by `ArgumentNames` from `GetCommandDescriptions`); whether `backend-action` is
accepted as an alias type name (recommended).

## 3. Backend contract (`IServerCommandExecutor`)

| Member | Semantic |
|---|---|
| `GetCommandDescriptions()` | lists commands with `CommandName`, `ArgumentNames`, `Description` (unique per endpoint) |
| `CanStartExecution(name, out reason)` | `CommandAvailabilityInfo`: 0 possible, 5 concurrency lock, 6 threshold lock, 7 no permission, 8 permanently unavailable |
| `StartExecution(name, string[] args, syncWaitMs, out state)` | enqueues; may return the final outcome if it finished within `syncWaitMs` |
| `GetLatestExecutionState(executionId, out state)` | polling |
| `RequestCancellation(executionId)` | cooperative cancellation |

`ServerCommandExecutionState`: `ExecutionId`, `CommandName`, `InvocationState`
(0 queued, 1 in progress, 2 completed, 3 canceled, 4 failed, 5-8 rejected), `StatusMessage`,
`CurrentStep`, `TotalSteps`, `CancellationPossible`, `CancellationRequested`,
`NextPollNotBefore`.

ASP.NET Core hosting: `services.AddServerCommandExecutor(registrar => ...)` registers a worker
based executor as hosted service and exposes it as a **UJMW** controller at route `commands`
(customizable).

### Status quo
- Backend contract and ASP.NET Core hosting exist (.NET 4.8 / 8 / 10 targets).
- The repository contains a TypeScript package `ushell-servercommands-contract` (`js/`), but `IServerCommandsExecutor` in it is still an empty interface.
- No frontend support: no command type, no attributes in `CommandDescription`, no client proxy usage, no progress UI.

### Refactoring & compatibility
- UB-30 (contract): additive `server-command` attributes in PortfolioDescription (.NET + TS); mirror `IServerCommandExecutor` and its DTOs in the TS package.
- UB-31 (runtime): execution and polling service in ReactFrontend, using the declared service proxy.
- UB-32 (CommonComponents): generic progress/notification component for running executions.
- UB-33: persistence and resume of running executions after reload.
- Out-parameters map to UJMW response fields. Verify the exact client-side shape against the UJMW skill before implementing the proxy.
