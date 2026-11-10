---
name: revit-addin-dotnet
description: Use for any .NET 8 / C# development work in this Revit add-in solution — writing or reviewing MediatR commands/queries and handlers, FluentValidation validators, business rules under Adapters/RevitNetAdapters/Systems/<SystemName>/Rules, Revit API adapter/manager code (Autodesk.Revit.SDK, geometry placement, family insertion), Localizer/resx-based error messages, Serilog logging (Debug/File/Email sinks), dependency injection registration, WPF/MVVM UI in RevitIntegrationProject.WpfUI, or xUnit/TestUI tests. Trigger on phrases like "add a new command", "add a new system/panel type", "business rule", "Revit adapter", "Revit family placement", "pipeline behavior", "validator", "MediatR handler", "manager class", "localizer", "CustomBusinessException", "add a query", "Hd150/Multitek/Panemax/Panex", or any request to extend Application/Infrastructure/WpfUI projects in this solution.
---

# .NET 8 Revit Add-in Development (RevitIntegrationProject)

You are working in a Clean-Architecture-style .NET 8 solution for an Autodesk Revit add-in (Revit API 2025 via `Autodesk.Revit.SDK` 2025.0.2.419). Always ground new code in the existing conventions below rather than inventing new patterns.

## Solution layout

- `src/core/RevitIntegrationProject.Application` (net8.0) — CQRS layer: MediatR requests/handlers, FluentValidation validators, pipeline behaviors, cross-cutting contracts. No Revit API references here.
- `src/infrastructure/RevitIntegrationProject.Infrastructure` (net8.0) — Revit API adapters, Serilog logging sinks, DI registrations. References `Autodesk.Revit.SDK`, `Microsoft.Data.SqlClient`, Serilog packages.
- `src/presentation/RevitIntegrationProject.WpfUI` (net8.0-windows7.0) — WPF/MVVM front end, invoked from within Revit's ribbon/dockable panels.
- `src/tests/RevitIntegrationProject.XUnitTest` — xUnit automated tests.
- `src/tests/RevitIntegrationProject.TestUI` — manual test harness project.

## Application layer conventions (`src/core/.../Application`)

Each Revit "system" (hardware family group — currently `Hd150`, `Multitek`, `Panemax`, `Panex`, plus `Family` and `Global`) gets its own folder under `Features/<System>/`:
- `Features/<System>/Commands/<Name>Command.cs` — contains, in one file:
  1. The command class implementing `IRequest<Response>` (or a more specific `IResponse`-derived type) with plain settable properties.
  2. `<Name>CommandHandler : BaseRequestHandler<TCommand>` — a thin handler that just forwards to `IRevitService<TCommand>.Execute(request)`. Do not put business logic in handlers.
  3. `<Name>CommandValidator : AbstractValidator<TCommand>` — FluentValidation rules with Turkish `.WithMessage(...)` strings matching the existing tone (e.g. "Yükseklikler alanı boş olamaz").
- `Features/<System>/Queries/` follows the same one-file-per-query pattern (see `GetFamilyPathQuery`).
- `Features/<System>/Dtos/` for DTOs implementing `IDto`.
- Results: use `Response` (bare success/failure flag), `SuccessResponse` (carries `Data`), `ErrorResponse` — do not invent new response wrapper types.
- Exceptions: throw `CustomBusinessException` for domain/business rule violations, `CustomValidationException` for validation failures (already thrown automatically by the validation pipeline behavior), `CustomAuthorizationException` for auth failures.
- Pipeline behaviors live in `Pipelines/Validations/RequestValidationBehavior` and `Pipelines/Authorizations/AuthorizationBehavior`, registered as `IPipelineBehavior<TRequest, TResponse>` via MediatR — new cross-cutting concerns should be added as new pipeline behaviors here, not sprinkled into handlers.
- `IRevitService<TRequest>` is the seam between Application and Infrastructure — Application only depends on this interface, never on Revit API types directly.

## Infrastructure layer conventions (`src/infrastructure/.../Adapters/RevitNetAdapters`)

Each system has its own subtree: `Systems/<SystemName>/Rules`, `Systems/<SystemName>/Services`, `Systems/<SystemName>/Localizer` (or `Localizers`, naming is not fully consistent — check the existing folder before adding a new one):
- **Rules** (`<Name>BusinessRules : IBusinessRule`): each public method validates one concern, throws `CustomBusinessException` with a localized message on failure, and returns `this` to support fluent chaining (`Rule.ValidateX(...).ValidateY(...)`). Family-existence checks go through `Mediator.Send(new GetFamilyPathQuery(...))` and inspect the `SuccessResponse.Data` — never hardcode a family path without this existence check.
- **Services** (`<Name>Manager : BaseManager<TRule>, IRevitService<TCommand>`): the actual Revit API orchestration — geometry math (`XYZ`, `Line`, `UnitUtils.Convert{From,To}InternalUnits` with `UnitTypeId.Centimeters`), calling `Rule.*` validations up front, then placing families via `RevitDocument.InsertAdaptiveFamily` / `InsertGenericFamily`, setting instance parameters via `SetParametersAdaptiveFamily` / `SetParametersGenericFamily`, and returning a `Response`/`SuccessResponse`. Constructor takes `(IMediator mediator, RevitDocument revitDocument, TRule rule)` and calls `base(...)`. Managers are typically `internal`.
- **Localizer**: static class per feature (e.g. `PanexColumnLocalizer`) in the `Localizer` namespace, backed by a `.resx` + `.tr-TR.resx` pair, exposing each string as a static property via `ResourceManager.GetString(nameof(...))`. Always add new user-facing/error strings this way — never hardcode Turkish/English strings inline in Rules or Managers except as a last-resort `throw new Exception(...)` for truly unexpected states.
- `RevitDocument` (`Adapters/RevitNetAdapters/Models/RevitDocument.cs`) and `RevitDocumentHelpers` are the shared Revit document/UIDocument access point — reuse existing helper methods (`GetActiveDocumentAsync`, `SelectElement`, `GetWallBaseCorners`, `GetRotateAngleForXDirection`, etc.) instead of duplicating Revit API calls inline.
- Unit convention: all user-facing dimensions are centimeters; convert to/from Revit's internal feet units at the Infrastructure boundary only, using `UnitUtils.ConvertFromInternalUnits` / `ConvertToInternalUnits` with `UnitTypeId.Centimeters`.

## Logging

Serilog-based, under `Infrastructure/Logging/Serilog/Logger/`: `FileLogger`, `DebugLogger`, `EmailLogger`, each implementing a small interface (`IFileLogger`, `IDebugLogger`, `IEmailLogger`) declared in `Application/CrossCuttingConcerns/Logging` and extending a shared `LoggerServiceBase`. File logging writes to `%APPDATA%/Revit Logs/log.txt` with daily rolling and a 5MB size cap. Use `Serilog.Sinks.PeriodicBatching` for the email sink's batching. Do not introduce a different logging framework (e.g. NLog, log4net) — extend the existing Serilog sink pattern.

## Dependency injection / configuration

- `Microsoft.Extensions.DependencyInjection` + `Microsoft.Extensions.Configuration`(.Json) + `Microsoft.Extensions.Options.ConfigurationExtensions`, target net8.0.
- Application-side registrations live in `DependencyResolvers/ApplicationServiceRegistrations.cs` — register new validators, MediatR handlers, and pipeline behaviors there following the existing pattern rather than ad hoc `services.AddX` calls scattered across the codebase.
- Serialization: `Newtonsoft.Json` is the standard (not `System.Text.Json`) — match existing usage.

## WPF UI (`src/presentation/RevitIntegrationProject.WpfUI`, net8.0-windows7.0)

MVVM pattern; UI sends MediatR commands/queries into the Application layer and displays `Response`/`ErrorResponse` results back to the user (typically via Revit `TaskDialog` or WPF dialogs). Keep Revit API calls out of ViewModels/Views — route everything through Application commands/queries and the `IRevitService` seam.

## Testing

- `src/tests/RevitIntegrationProject.XUnitTest` — automated xUnit tests; prefer testing Application-layer validators, business rules, and pure logic (e.g. `CreateColumnCornerConnectorConnectionHeights`-style calculation helpers) here, since Revit API calls generally require a live Revit session.
- `src/tests/RevitIntegrationProject.TestUI` — manual/interactive test harness for exercising Revit-dependent flows that can't be easily unit-tested.

## Out of scope for this skill

- Figma/design or general WPF visual styling/theming work (use a dedicated UI/design skill if one exists).
- DevOps/CI pipeline, packaging, or Revit add-in manifest/deployment concerns.
- Deep SQL schema design for `Microsoft.Data.SqlClient` usage (only reference it as an existing dependency; don't assume a specific data-access pattern beyond what's found in code).

## When extending an existing System (e.g. Panex, Panemax, Hd150, Multitek)

Follow the exact existing file for that system as the template (same folder names, same one-file-per-command layout, same fluent business-rule chaining, same Localizer resx pattern) rather than introducing a new structural variant. When adding a brand-new System, mirror the folder structure of the most similar existing one (`Rules/`, `Services/`, `Localizer/`, `Features/<System>/Commands/`).
