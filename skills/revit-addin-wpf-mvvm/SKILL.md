---
name: revit-addin-wpf-mvvm
description: Use for WPF/MVVM presentation-layer work in RevitIntegrationProject.WpfUI — adding or editing a Window/View + ViewModel + Controller for a System (Hd150, Multitek, Panemax, Panex), wiring RelayCommand/AsyncRelayCommand bindings, adding a new Ribbon button via [Ribbon]/RibbonHelper, shared Components (CustomButton, CustomComboBox, CustomCheckBox, CustomGroupCheckBox, CustomGridWithSelectedItem), FontAwesome.Sharp icons, value Converters (EnumLocalizationConverter), Localization.xaml/Localization.tr.xaml resource dictionaries, BaseViewModel/INotifyPropertyChanged patterns, ExternalCommandData.ShowView, BaseExternalEventHandler/RequestHandler MediatR dispatch from the UI, appsettings.json-based DI via ServiceRegistrations, Assets/Images/Ribbon or Assets/Images/Views/<System> imagery, or exercising a view through the TestUI manual harness. Trigger on phrases like "add a new WPF view", "new dialog for <System>", "ViewModel", "Controller" (WpfUI), "Ribbon button/icon", "CustomButton/CustomComboBox/CustomGridWithSelectedItem", "FontAwesome icon", "RelayCommand", "AsyncRelayCommand", "Localization.xaml", "ShowView", "TestUI harness". Do NOT use this skill for Application/Infrastructure/MediatR command-handler/business-rule work in the core layers — see revit-addin-dotnet for that.
---

# WPF/MVVM UI (RevitIntegrationProject.WpfUI)

`src/presentation/RevitIntegrationProject.WpfUI` targets `net8.0-windows7.0`, `UseWPF=true`. This skill covers only the presentation layer's own conventions — never put Revit API calls or business logic here; route everything through Application MediatR commands/queries via the `IRevitService` seam described in `revit-addin-dotnet`.

## Per-System View/ViewModel/Controller triad

Each business "System" (`Hd150`, `Multitek`, `Panemax`, `Panex`) has its own folder in three parallel trees:
- `Views/<System>/<Name>View.xaml` + `.xaml.cs` — a `Window` subclass. Code-behind is intentionally minimal: constructor just calls `InitializeComponent()`. All logic lives in the ViewModel, set via `<Window.DataContext><vm:XxxViewModel/></Window.DataContext>` in XAML (parameterless-constructable ViewModels).
- `ViewModels/<System>/<Name>ViewModel.cs` — `partial class XxxViewModel : BaseViewModel`. Owns `ObservableCollection<T>` properties for grid/list bindings, `ICommand` properties backed by `RelayCommand`/`RelayCommand<T>`/`AsyncRelayCommand`, and a `GenerateViewModel()`-style constructor-time initializer that seeds `Available*` option lists and sensible default `Selected*` values (see `ViewModels/Panex/PanexColumnViewModel.cs`).
- `Controllers/<System>/<Name>Controller.cs` — the Revit-facing entry point: `[Transaction(TransactionMode.Manual)]`, `[Ribbon(systemName, groupName, commandName, categoryImageName:, imageName:)]`, `public class XxxController : BaseController<XxxView>`. `BaseController<TView>` (`Controllers/BaseController.cs`) implements `IExternalCommand.Execute` by calling `commandData.ShowView<TView>()` and returning `Result.Succeeded` — do not override `Execute` unless a system genuinely needs pre/post logic beyond opening the window.

When adding a new dialog: create all three files, follow the exact existing sibling for that System as the template, and add the `[Ribbon(...)]` attribute so `RibbonHelper.GenerateRibbonPanels` (called from `App.OnStartup`) picks it up automatically via reflection — there is no manual ribbon registration list to update.

## Commands and MVVM plumbing

- `ViewModels/RelayCommand.cs` defines `RelayCommand`, `RelayCommand<T>`, `AsyncRelayCommand`, `AsyncRelayCommand<T>` (global namespace, no `RevitIntegrationProject.WpfUI.ViewModels` prefix needed) — always reuse these; do not add a new command implementation or take a MVVM Toolkit dependency.
- `ViewModels/BaseViewModel.cs` implements `INotifyPropertyChanged`, `INotifyPropertyChanging`, `INotifyCollectionChanged` with protected `OnPropertyChanged(name, oldValue, newValue)` / `OnPropertyChanging` / `OnCollectionChanged` helpers and a `CustomPropertyChangedEventArgs` carrying old/new values — extend this base rather than re-implementing `INotifyPropertyChanged` per ViewModel. (Note: many properties in existing ViewModels are plain auto-properties without explicit `OnPropertyChanged` calls — match the surrounding file's existing style rather than retrofitting full notification everywhere.)
- Draw/submit commands are typically `AsyncRelayCommand` wrapping a private `DrawAsync()` that builds an Application-layer `*Command` object (e.g. `PanexColumnCommand`), sets its properties from the ViewModel's selections, then calls `await App.ExternalEventHandler.SendAsync(command)` and shows `MessageBox.Show((result as ErrorResponse).ErrorMessage)` when `!result.Success`. This is the WPF-to-MediatR bridge — never call `IMediator` directly from a ViewModel.

## Controllers as the Revit-command / MediatR bridge

- `Controllers/BaseController.cs`: `BaseController<TView> : IExternalCommand where TView : Window` — thin, just opens the view via `Helpers/ViewHelper.ShowView<TView>()`.
- `Helpers/ViewHelper.cs`: `ShowView<TView>(this ExternalCommandData)` loads the correct `Localization.xaml` / `Localization.tr.xaml` merged dictionary based on `CultureInfo.CurrentCulture`, sets `WindowStartupLocation.CenterOwner`, parents the window to Revit's main window via `WindowInteropHelper.Owner = uiApp.MainWindowHandle`, then `view.Show()` (non-modal). Reuse this helper for any new view rather than writing bespoke `Show()`/owner-wiring code.
- `Handlers/RequestHandler.cs` (`BaseExternalEventHandler` implementation, registered as `App.ExternalEventHandler`): the single place that calls `IMediator.Send`, wrapping it in try/catch over `CustomAuthorizationException`, `CustomValidationException`, `CustomBusinessException`, and a generic fallback — each branch logs via `IFileLogger`/`IEmailLogger`/(`#if DEBUG` `IDebugLogger`) using `LogHelper.GetLogDetail(command)` and returns an `ErrorResponse`. Do not duplicate this try/catch in ViewModels — always go through `App.ExternalEventHandler.SendAsync(command)`.

## Ribbon registration

- `Attributes/RibbonAttribute.cs`: `[Ribbon(systemName, groupName, commandName, isVisible = true, categoryImageName = null, imageName = null)]`, applied to the Controller class.
- `Helpers/RibbonHelper.cs`: `GenerateRibbonPanels(this UIControlledApplication)` (called once from `App.OnStartup`) reflects over the executing assembly for `[Ribbon]`-decorated types, creates one `RibbonPanel` per distinct `systemName` under a single "INTEK BIM" tab, one `PulldownButton` per distinct `groupName` within that panel, and one `PushButton` per controller type. Images are loaded from `Assets/Images/Ribbon/<imageName>` (falls back to `NoImage32x32.png` if missing/not found on disk). When adding a new command, just add the attribute — do not hand-edit ribbon-building code.

## Shared components (`ViewModels/Components`, `Views/Components`)

Reusable `UserControl`s live in `Views/Components/` (namespace `RevitIntegrationProject.WpfUI.Components`, XAML prefix conventionally `xmlns:toolbox="clr-namespace:RevitIntegrationProject.WpfUI.Components"`):
- `CustomButton` — wraps a `Button` with rounded/circular style variants (`IsCircular`), a `fa:IconImage` from **FontAwesome.Sharp** (`Icon="{Binding IconName}"`, e.g. `IconName="CompassDrafting"`) shown/hidden via `ShowIcon`, plus `ButtonText`, `ButtonCommand`, `ButtonCommandParameter`, `ButtonOpacity`. Reuse this instead of a bare `<Button>` for any command-triggering action button.
- `CustomComboBox`, `CustomCheckBox`, `CustomGroupCheckBox`, `CustomGridWithSelectedItem` — labeled combo/checkbox/grid wrappers with a consistent `Label`/`LabelWidth`/`LabelHeight` + bound `DataSource`/`SelectedItem`/`SelectedItems` API. `CustomGridWithSelectedItem` is the standard pattern for an "available options -> add to selected list, editable grid, remove" UI (see `PanexColumnView.xaml`'s five Height/Width grids) — prefer it over hand-rolling a `DataGrid` + `ComboBox` + add/remove buttons.
- New shared controls belong in `Views/Components` (view) with any dedicated view-model-ish helper types under `ViewModels/Components`; keep System-specific UI out of `Components`.

FontAwesome.Sharp: referenced as a NuGet package (`FontAwesome.Sharp` 6.6.0) and also explicitly resolved at runtime in `App.OnResolveAssembly` (loads the DLL from the executing assembly's directory if the default resolver fails) — do not remove that resolver hook, it exists to work around Revit's own assembly-loading context.

## Converters, Attributes, Helpers, Models

- `Converters/EnumLocalizationConverter.cs` — `IValueConverter` that looks up `TryFindResource($"{Prefix}.{EnumTypeName}.{value}")` (default `Prefix = "Enum"`) and falls back to `value.ToString()`. Use this for binding any enum property to a localized display string instead of a per-enum converter or inline `IValueConverter`.
- `Helpers/LogHelper.cs`, `Helpers/PathHelpers.cs` — shared formatting/path utilities; check here before adding a new helper class.
- `Models/` — plain UI-only DTOs/view models for lists (`ListItemModel`, `SelectedListItemModel`, `WallModel`, `PanelModel`, `InnerCornerModel`); `Models/Enums/PanelElementType.cs` for UI-facing enums. These are distinct from Application-layer DTOs — keep WPF-only shapes here rather than reusing Application `Dtos` types directly in bindings.
- `DependencyResolvers/ServiceRegistrations.cs` — builds the single `IServiceProvider` for the whole add-in: loads `appsettings.json` next to the assembly via `ConfigurationBuilder`, chains `AddApplicationServices()` + `AddInfrastructureServices()` (from the core/infrastructure layers), registers `BaseExternalEventHandler` as the concrete `RequestHandler`, and binds `EmailLoggerSettings`. `App.cs` (`IExternalApplication`) builds this provider once in its constructor and exposes it as `App.ServiceProvider` / `App.ExternalEventHandler` — new services should be added to this one registration method, not scattered `ServiceCollection` calls elsewhere.

## Assets and localization

- `Assets/Images/Ribbon/*.png` — ribbon tab/pulldown/button icons, referenced by filename string in `[Ribbon(...)]`; keep a `NoImage16x16.png`/`NoImage32x32.png` fallback available.
- `Assets/Images/Views/<System>/<Feature>/*.jpg` — reference diagrams shown inline in a View (e.g. `Image Source="/RevitIntegrationProject.WpfUI;component/Assets/Images/Views/Panex/Column/ColumnPlan.jpg"`), added to the `.csproj` as `<Resource Include=... CopyToOutputDirectory="Always">`. When adding a new View with reference imagery, add both the `Resource` csproj entry and the pack-URI `Image` binding.
- `Resources/Localization.xaml` / `Resources/Localization.tr.xaml` — `DynamicResource` string tables merged into each Window at show-time by `ViewHelper.ShowView` based on `CultureInfo.CurrentCulture`. All user-facing static strings in XAML should be `{DynamicResource Key}` bound to entries in both dictionaries, not hardcoded (Turkish literals do still appear in some existing XAML/code — match the nearest existing file, but prefer the resource-dictionary approach for new UI).

## Manual test harness (`src/tests/RevitIntegrationProject.TestUI`)

`net8.0-windows`, `OutputType=WinExe`, `UseWPF=true`. References `RevitIntegrationProject.Application` and `RevitIntegrationProject.WpfUI` directly (not just Infrastructure) so Views/ViewModels can be instantiated and interacted with outside of a live Revit session — this is a manual/dev harness for eyeballing a dialog's layout and bindings, not an automated test project (that's `RevitIntegrationProject.XUnitTest`, out of scope here). When a WPF view needs to be sanity-checked visually, wire it up here rather than only testing inside Revit.

## Out of scope

- MediatR command/query/handler authoring, FluentValidation validators, business rules, Revit API adapters/managers, Serilog logging internals, and Application/Infrastructure DI registration — see `revit-addin-dotnet`.
- General visual re-theming/design-system work beyond matching existing Components' look and feel.
- Automated xUnit testing (`RevitIntegrationProject.XUnitTest`).
