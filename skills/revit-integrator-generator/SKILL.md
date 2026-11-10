---
name: revit-integrator-generator
description: Use to scaffold a brand-new Revit "System" (full-stack: Application command/handler/validator, Infrastructure business rules/adapter, WPF view/viewmodel/controller/ribbon) or to add/modify a WPF interface for an existing System in this project, driven by a JSON request and either automatic UI design, an image, or a draw.io diagram. Opens a Jira ticket and follows CLAUDE.md's Jira & Bitbucket Workflow before generating code. Trigger on phrases like "yeni bir sistem oluştur", "yeni panel/sistem ekle", "bu sistem için arayüz oluştur/düzenle", "generate a new system", "scaffold a new Revit system", or any request to generate a full System or its UI from a JSON spec/image/diagram. Do NOT use for ad hoc single-file edits to an existing command/view — see revit-addin-dotnet / revit-addin-wpf-mvvm for that.
---

# revit-integrator-generator

This skill orchestrates end-to-end generation of a Revit "System" (like `Hd150`, `Multitek`, `Panemax`, `Panex`) or a WPF interface addition to an existing System — from an initial choice, through a JSON spec and UI-source question, a Jira ticket, code generation, build/test validation, to a PR-ready branch. It does not invent its own code-generation rules: for actual code, follow `revit-addin-dotnet` (Application/Infrastructure) and `revit-addin-wpf-mvvm` (WPF/MVVM) exactly as written there. For Jira/Bitbucket process, follow CLAUDE.md's "Jira & Bitbucket Workflow" section and reuse the `to-tickets` skill/flow for opening the ticket rather than inventing a parallel ticket-creation path.

## Step 1 — Initial choice

Ask the user, as a single-select question (not free text):

> "Nasıl devam etmek istersiniz?"
> 1. Yeni bir sistem
> 2. Mevcut bir sistem için arayüz

If option 2 is chosen, do not accept a free-text system name. Scan `src/core/RevitIntegrationProject.Application/Features/*` for actual subfolders (the real System names: `Hd150`, `Multitek`, `Panemax`, `Panex`, plus `Family`/`Global` if relevant) and present them as a single-select list. Never target a system that doesn't exist on disk.

## Step 2 — JSON request

Ask the user for a JSON request mapping to the target `IRequest<Response>` command's properties. Show them this schema as an example — it covers a scalar field, a numeric-collection field, a selectable list field with display/value pairs and a default selection, a boolean field, and per-property validation rules. The JSON never names an exact shared component (`CustomComboBox`, `CustomGridWithSelectedItem`, …) — the user can get that wrong, so the skill always decides which component backs each property itself, from `type`/`options`/`validations`, the same way as the "otomatik tasarla" flow in Step 3:

```json
{
  "systemName": "Panex",
  "commandName": "ColumnCornerConnector",
  "properties": [
    {
      "name": "WidthA",
      "type": "double",
      "required": true,
      "validations": [
        { "rule": "GreaterThan", "value": 0, "message": "A genişliği 0'dan büyük olmalıdır" }
      ]
    },
    {
      "name": "Heights",
      "type": "List<double>",
      "required": true,
      "validations": [
        { "rule": "MinCount", "value": 1, "message": "En az bir yükseklik girilmelidir" },
        { "rule": "MaxCount", "value": 10, "message": "En fazla 10 yükseklik girilebilir" }
      ]
    },
    {
      "name": "MaterialType",
      "type": "enum:MaterialType",
      "required": true,
      "options": [
        { "displayText": "Alüminyum", "value": "Aluminum" },
        { "displayText": "Çelik", "value": "Steel" },
        { "displayText": "Galvaniz", "value": "Galvanized" }
      ],
      "selectedIndex": 0,
      "validations": [
        { "rule": "Required", "message": "Malzeme tipi seçilmelidir" }
      ]
    },
    {
      "name": "IsPainted",
      "type": "bool",
      "required": false,
      "defaultValue": false,
      "uiProperty": { "label": "Boyalı mı?" }
    }
  ]
}
```

Field notes:
- `type`: only primitive types / collections / enum names (`double`, `string`, `bool`, `List<T>`, `enum:<EnumName>`). Revit-specific types (`ElementId`, `XYZ`, `Category`, …) never appear in the JSON — any such conversion happens inside the Infrastructure/Manager layer, outside the JSON contract.
- `options` (list/enum fields only): each entry is `{ "displayText": "<shown to user>", "value": "<stored/enum value>" }` — `displayText` maps to the bound `DataSource` item's display text (or a `Localization.xaml` key if the list should be localized), `value` maps to the underlying enum member or stored value. Generate the backing enum (if `type` is `enum:<EnumName>` and the enum doesn't already exist) from these `value` entries.
- `selectedIndex` (list/enum fields only): the zero-based index into `options` used to seed the ViewModel's `Selected<Property>` default, mirroring the `GenerateViewModel()`-style constructor-time initializer pattern (see `ViewModels/Panex/PanexColumnViewModel.cs`). Omit it if there is no sensible default — in that case leave `Selected<Property>` unset and let the validator require an explicit choice.
- `defaultValue` (optional, scalar/bool fields): seeds the property's default in the generated ViewModel/Command; omit if there is no natural default.
- `uiProperty` (optional, free-form): supplementary display metadata that doesn't decide the control type — e.g. `label` (override the auto-derived label text), `readonly`, `multiline`, `placeholder`. Never put a component class name in here; component selection is always the skill's decision, not the JSON's.
- `validations` (optional per property): each entry is `{ "rule": "<RuleName>", "value"?: <threshold>, "message": "<Turkish message>" }`. `rule` is one of: `Required`, `GreaterThan`, `LessThan`, `Range` (needs `min`/`max` instead of `value`), `MinLength`/`MaxLength` (strings), `MinCount`/`MaxCount` (lists), `Regex` (needs `pattern` instead of `value`), or `Custom` (needs a free-text `description` of the business rule instead of `value` — generated as a `CustomBusinessException`-throwing method in `<Name>BusinessRules`, not in the Validator). Map `Required`/`GreaterThan`/`LessThan`/`Range`/`MinLength`/`MaxLength`/`MinCount`/`MaxCount`/`Regex` to `<Name>CommandValidator` (FluentValidation) rules with `.WithMessage(...)` set to `message`, matching the existing Turkish tone (e.g. "Yükseklikler alanı boş olamaz"); map `Custom` to a fluent-chained method on `<Name>BusinessRules` instead, since business rules (not simple field validation) belong in Infrastructure per `revit-addin-dotnet`.

Other rules:
- Validate the JSON yourself before generating anything: required fields present, types recognized, `options`/`selectedIndex` consistent (index in range, no duplicate `value`s), no name collisions. If anything is missing or ambiguous, ask the user — never silently assume a default.
- **New system branch:** before generating, check whether `Features/<System>/Commands/<Name>Command.cs` already exists. If it does, tell the user and ask for a different name or suggest switching to the "existing system interface" flow instead of overwriting.
- **Existing system branch:** if the JSON requires adding a property to an existing Command, first search the codebase for other files referencing that Command (other Views/ViewModels/tests) and report what will be affected before touching it. Don't silently mutate a shared Command.

## Step 3 — UI source

Ask a second single-select question:

> "Arayüzü nasıl oluşturmak istersiniz?"
> 1. Otomatik tasarla
> 2. Bir resim analiz et
> 3. Bir draw.io diyagramı analiz et

**Option 1 — Automatic:** derive a reasonable form layout from the JSON's property types (`bool` → `CustomCheckBox`, enum/list → `CustomComboBox`, numeric collection → `CustomGridWithSelectedItem`, etc.), using the closest existing System's View as the template.

**Option 2 — Image:** ask the user for the image. Extract control/field layout, grouping, label text, and visual style from it.

**Option 3 — draw.io XML:** ask the user for the exported `.xml` file (uncompressed native draw.io export — not PNG/SVG). Parse it as follows:
- Primary method: if an `mxCell`'s `style` attribute contains a draw.io Mockup/Forms shape name (`shape=mxgraph.mockup.forms.*`), read the control type directly from it. This mapping is unverified against a real export — learn from whatever `style` values actually appear rather than trusting a fixed table.
- Fallback: if `style` doesn't carry this, infer the control type from a naming prefix in the cell's `value` (label) text, e.g. `"TextBox: Ad Soyad"`, `"ComboBox: Malzeme Tipi"`.
- **Mandatory cross-check before finalizing any mapping:** for every extracted draw.io control, find the closest existing View (per Step 1's target System, or the nearest similar System if none) that has a field for the same concept, matched by label-text proximity (e.g. the diagram's "Malzeme Tipi" label against an existing View's `MaterialType`/`Malzeme Tipi`-labeled control). Read that real View's actual control type and treat it as the authoritative answer for what the shape/style/prefix *should* resolve to — don't finalize a control-type/component mapping from shape or naming-prefix alone without this cross-check.
- If, after the cross-check, a shape or its inferred component still doesn't clearly map to one of the existing shared components (`CustomButton`, `CustomComboBox`, `CustomCheckBox`, `CustomGroupCheckBox`, `CustomGridWithSelectedItem`) — no matching concept found in any existing View, or the existing View's control type conflicts with the shape/prefix inference — **ask the user** what it represents and which component it should map to. Never guess a best-effort mapping in this situation, even if one inference method produced an answer.
- Use `geometry` (x, y, width, height) for position/order, and `parent` relationships for grouping.
- If the XML is malformed or uses an unsupported shape library, report the failure and offer to fall back to option 1 or 2 — don't produce an empty/default UI silently.

**Component reuse (applies to all three UI sources):**
1. Prefer the existing shared controls in `Views/Components` (`CustomButton`, `CustomComboBox`, `CustomCheckBox`, `CustomGroupCheckBox`, `CustomGridWithSelectedItem`) wherever a control matches.
2. If a needed control has no match, design a new reusable `UserControl` following the existing components' pattern (same `Label`/`LabelWidth`/`LabelHeight` + bound `DataSource`/`SelectedItem` API). **Before adding it to the shared `Views/Components` pool**, summarize the proposed component to the user and get explicit approval — this extends a pool every other System depends on, so it's not a silent decision. Once approved, add it there and use it in the current view.

## Step 4 — Jira ticket (after JSON is collected, before code generation)

Follow CLAUDE.md's Jira & Bitbucket Workflow, via the `to-tickets` skill/flow — do not duplicate ticket-creation logic here:
- Issue type: **Hikaye**, always, not asked per-run (both a new System and a new interface deliver user-visible functionality, consistent with CLAUDE.md's Minor version-bump rule).
- Project: `RI` (`intekkalip.atlassian.net` per CLAUDE.md — verify this against `getVisibleJiraProjects` before the first real run, since it hasn't been confirmed live).
- Ticket title/description: summarize the collected JSON request, target System name, and chosen UI source.
- Start date is set automatically when work begins on the ticket, per the standard workflow.
- Branch: `<tip>/<RI-XXX>-<kebab-case-aciklama>` — Hikaye → `feature/`, auto-linked via Jira's development panel.
- These Jira-side steps don't need user approval; **the first commit/push after branch creation does**.

**If the flow is abandoned mid-way** (user rejects TestUI approval, or says stop, after ticket/branch already exist): leave generated files on disk uncommitted, don't delete the branch, don't auto-close the ticket. Summarize the state ("ticket RI-XXX and branch feature/RI-XXX-... are open, generated files are not committed") and ask the user what they want to do (clean up / leave it / continue).

**On re-invocation:** check whether an open ticket/branch already exists for the same target System/JSON (search branches + Jira) before opening a new ticket — offer to resume instead of duplicating.

## Step 5 — Code generation

**Hard gate before this step starts:** do not write a single line of Application/Infrastructure/WPF code until you have verified — not just stated — that a Jira ticket and its branch already exist for this JSON/System, per Step 4. Concretely: confirm there is an open ticket number and that you are on (or have created) its `feature/<RI-XXX>-...` branch. If either is missing at the point you're about to start generating code, **stop immediately and go do Step 4 first** — do not proceed "temporarily" and create the ticket later, even if the user is pushing straight toward code or iterating on generated code. This is a hard-stop, not a judgment call (see "When to actually stop and ask").

Apply `revit-addin-dotnet` and `revit-addin-wpf-mvvm` conventions exactly; never step outside the existing folder structure or architecture (`docs/analyze/system/folder-structure.md` is the reference).

- **New system:** `Features/<System>/Commands/<Name>Command.cs` (Command+Handler+Validator in one file), `Adapters/RevitNetAdapters/Systems/<System>/{Rules,Services,Localizer}`, `Controllers/ViewModels/Views/<System>/`, the `[Ribbon(...)]` attribute (RibbonHelper discovers it automatically via reflection — never hand-edit ribbon-building code), `Localization.xaml`/`Localization.tr.xaml` entries, DI registrations in `ApplicationServiceRegistrations.cs`/`ServiceRegistrations.cs`.
- **Existing system interface:** only touch that System's `Views/<System>`, `ViewModels/<System>`, and if needed `Controllers/<System>`; if JSON scope requires it, update the relevant Command's properties (per the affected-files check in Step 2).
- **Localization parity:** for every new/changed key, verify it exists in both `Localization.xaml` and `Localization.tr.xaml` before building — fill in any gap.
- Always use the closest existing similar System's file as the literal template — never introduce a new structural variant.

## Step 6 — Build and test (bounded retry)

1. Run `dotnet build`.
2. Write xUnit tests under `src/tests/RevitIntegrationProject.XUnitTest/Tests/<System>/<Name>Tests.cs`, following `docs/analyze/test/naming-conventions.md`: class name `<Command>Tests`, natural-language-style method names (`Throws_an_error_if_...`, `Success_request`), implicit Arrange-Act-Assert, a hand-written fake `IRevitService<TCommand>` under `TestServices/<System>/` (no mocking library). Cover both the Command and its Validator. Run them with `dotnet test`.
3. **If build or tests fail, fix and retry — but cap this at 3 attempts.** After 3 failed attempts, stop, report the current error and the code generated so far to the user, and ask how to proceed. Never loop indefinitely.
4. Register the new View/ViewModel in `src/tests/RevitIntegrationProject.TestUI` as a manual harness entry (registration only — do not attempt to run it yourself). Ask the user to run it manually and visually confirm it, and **wait for their explicit approval before continuing** — this cannot be automated.

## Step 7 — Commit / PR / release

Follow CLAUDE.md's Jira & Bitbucket Workflow exactly:
- Commit message: `[RI-XXX] feat: <Türkçe açıklama>` — every commit/push needs user approval.
- PR: always add Serdar Uludağ and Muammer Yorulmaz as reviewers; description includes the Jira link, a change summary, and test/QA notes (build result, xUnit results, TestUI manual-approval status).
- Opening the PR moves the linked ticket to REVİEW automatically.
- After merge: ask whether to delete the branch, then ask about a release/tag (Hikaye → suggest Minor, per CLAUDE.md's version rule).
- **Never auto-transition the ticket to Done/Tamam.** After merge, leave the ticket in REVİEW (or whatever review-adjacent status it's already in) — moving it to Done is a decision for the human reviewer, not this skill. This is not part of the standard merge follow-up sequence. If the user separately asks for help closing the ticket, first confirm with them via a question before transitioning it to Done — never do it unprompted.
- **Never commit/push/open a PR directly on master.**

## Step 8 — docs/analyze staleness (note only, don't update)

A new System or interface can make `docs/analyze/system/folder-structure.md`, `docs/analyze/backend/*`, and `docs/analyze/frontend/*` stale. Don't update these documents yourself — that's a separate, focused task. Instead, add a note to the PR description: "bu değişiklik `docs/analyze` belgelerini etkileyebilir, `project-analyze` skill'i ile güncellenmesi önerilir."

## When to actually stop and ask

Auto Mode says to keep going on reasonable judgment calls; this skill still hard-stops for the following (everything else — small formatting choices, internal ordering — is your call):
- The two single-select questions (Steps 1, 3).
- JSON schema ambiguity or naming collisions (Step 2).
- An unrecognized control shape in a draw.io diagram, or a control that doesn't clearly cross-check to an existing shared component (Step 3).
- Adding a new component to the shared `Views/Components` pool (Step 3).
- Files affected by mutating an existing Command's properties (Step 2).
- Missing Jira ticket/branch right before code generation would start (Step 4/5 hard gate) — go create them first, don't generate code first and open the ticket later.
- Build/test still failing after 3 attempts (Step 6).
- TestUI manual approval (Step 6).
- Commit/push/PR (Step 7 — required by CLAUDE.md regardless).
- Transitioning the ticket to Done after merge (Step 7) — only with explicit user confirmation, never as part of standard merge follow-up.

## Out of scope

- Ad hoc, single-file edits to an existing command/view outside this JSON+Jira flow — use `revit-addin-dotnet` / `revit-addin-wpf-mvvm` directly.
- General visual re-theming/design-system work, Figma integration.
- CI pipeline work — `bitbucket-pipelines.yml` has no real `dotnet build/test` step yet (see `docs/analyze/test/testing-strategy.md`), so build/test validation here is local only; wiring it into CI is a separate task.
- Updating `docs/analyze/*` documents (see Step 8 — flag only).

## Recommended validation order before relying on this skill for a real System

1. Export one real draw.io diagram with a few known control types and inspect the actual `style` strings — confirms or disproves the Mockup/Forms shape assumption in Step 3.
2. Run JSON→code generation against an existing System's real JSON (e.g. Panex) and diff the output against the real code, instead of starting with a genuinely new System.
3. Open the Jira ticket + branch on a throwaway/test ticket first, confirming the `intekkalip.atlassian.net` project domain.
4. Exercise the build/test retry cap (Step 6) before trusting it unattended.
5. Keep the TestUI manual-approval gate as the human checkpoint it is — don't try to automate past it.
