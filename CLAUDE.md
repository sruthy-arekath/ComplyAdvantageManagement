# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

**ComplyAdvantage Cleanup Tool** (project `ComplyAdvantageManagement`): an internal ASP.NET Core MVC app (`net10.0`, controllers + Razor views) that helps the compliance team clean up the ComplyAdvantage (CA) Mesh portal so that it only holds active brokers and active customers.

It does four things:
1. **Source list**: loads the list of active brokers and active customers from Matrix/Salesforce. Phase 1 uses Excel uploads (one file for brokers, one for customers). The "Fetch from API" option exists in the UI but is disabled until a source API is available.
2. **CA extract**: pulls every customer record from the CA Mesh API and consolidates it into an Excel workbook (Brokers / Customers / Unclassified sheets plus a Summary).
3. **Compare**: compares the CA extract with the source lists and produces the cleanup workbook: inactive records, duplicates, entries to delete and the remaining records.
4. **Delete (Phase 2, NOT YET IN SCOPE)**: removing or closing records in CA via the API.

Every result is shown in the browser as a table and can be downloaded as `.xlsx`.

**Requirements come from the user's prompts.** There is no separate requirements document: treat each request as a requirement, implement it, and record the resulting behaviour and business rules in `docs/TECHNICAL_DOCUMENTATION.md`, which is the cumulative record of what the tool does and why. If a request conflicts with earlier documented behaviour or with this file, point out the conflict and ask before changing either. The hard safety rules below still apply unless the user explicitly lifts them. When code and documentation disagree, trust the code and fix the documentation in the same task.

**Current state**: the project is still the stock `dotnet new mvc` template (`HomeController` with Index/Privacy/Error). The `docs/` folder and the test project don't exist yet. The layout under "Architecture" is the target that the scaffolding task creates. If a doc this file refers to is missing, say so and continue from this file rather than inventing its contents.

## Hard safety rules (read first)

- **Phase 1 is read-only against ComplyAdvantage.** The only CA calls allowed are `POST /v2/token`, `GET` endpoints, and (if permission is granted) `POST /v2/exports` plus its status/download `GET`s. Never write code that deletes, closes, updates, transitions or re-screens a CA customer, and never call such an endpoint yourself, even for testing. The `ReadOnlyGuardHandler` enforces this in code; don't weaken or bypass it. Phase 2 will lift this deliberately, with its own plan and sign-off.
- **Business rules are not yours to decide.** Whether a record is deleted, closed or just has monitoring disabled, how a CA record is classified as broker vs customer, and which duplicate to keep are compliance decisions (Mark for brokers, Lamiaa for customers). Implement them as configurable rules with the documented defaults, and list anything unconfirmed under "Open questions" instead of guessing.
- **Secrets**: CA `Username`, `Password` and `Realm` are bound from the `ComplyAdvantage` config section. Use `dotnet user-secrets` in Development and environment variables / the secret store elsewhere. Never write real credentials, tokens or customer data into `appsettings*.json`, source files, docs, plan files, test fixtures or commit messages.
- **Personal data**: CA and source records contain names and identifiers of real people and companies. Never log names, identifiers, tokens or passwords; log counts, run IDs and page numbers only. Test fixtures use obviously fake data. Generated workbooks live under `App_Data/runs/` and must never be committed.

## Commands

Run from the repo root (the web project is in a subfolder, so `run`/`watch` need `--project`):

```bash
dotnet build ComplyAdvantageManagement.slnx
dotnet test ComplyAdvantageManagement.slnx
dotnet run --project ComplyAdvantageManagement         # only when the user asks
dotnet watch run --project ComplyAdvantageManagement   # hot reload, only when the user asks
```

`dotnet run` serves http://localhost:5286 (`--launch-profile https` for https://localhost:7014). There is no test project yet; the scaffolding task adds it. Once it exists, run a single test or class with `dotnet test ComplyAdvantageManagement.slnx --filter "FullyQualifiedName~<ClassOrMethodName>"`.

**Validation** for every change:
- `dotnet build ComplyAdvantageManagement.slnx` succeeds with no errors and no new warnings.
- `dotnet test ComplyAdvantageManagement.slnx` passes. The comparison and pagination logic must have unit tests; add or update them with every change to that logic.
- Don't run `dotnet run` yourself unless asked; it blocks the session.
- For UI changes, tell the user exactly what to check in the browser (page, action, expected result).

## Architecture

**Existing MVC wiring** (keep it; register new DI services and middleware in `Program.cs`):
- `Program.cs` registers `AddControllersWithViews()` and one conventional route, `{controller=Home}/{action=Index}/{id?}`. There are no attribute routes, Razor Pages or minimal-API endpoints yet.
- Views: `Views/<Controller>/<Action>.cshtml`, shared layout `Views/Shared/_Layout.cshtml` (top navbar with the navigation links, `RenderSectionAsync("Scripts")` for per-page scripts), `_ViewStart.cshtml` sets the layout, `_ViewImports.cshtml` holds shared `@using`s and the tag helpers. `_ValidationScriptsPartial.cshtml` pulls in jQuery unobtrusive validation for forms.
- Errors: non-Development uses `UseExceptionHandler("/Home/Error")` plus `UseHsts()`; `UseHttpsRedirection()` is on. No status-code pages are configured.
- Front-end assets: Bootstrap, jQuery, jquery-validation and jquery-validation-unobtrusive under `wwwroot/lib/`, `wwwroot/css/site.css`, `wwwroot/js/site.js`, and CSS isolation (`_Layout.cshtml.css` is bundled into `ComplyAdvantageManagement.styles.css`). Served via `MapStaticAssets()` with `.WithStaticAssets()` on the route; use `asp-append-version="true"` on local assets.

**Target layout** (the scaffolding task creates it; update this section if it changes):

```
ComplyAdvantageManagement.slnx
ComplyAdvantageManagement/          ASP.NET Core MVC app (net10.0)
  Controllers/                      thin controllers: Home, SourceLists, CaExtract, Compare, Runs
  Views/<Controller>/               Razor views per controller; Views/Shared/_Layout.cshtml for nav
  Services/ComplyAdvantage/         typed HttpClient, token service, pagination, read-only guard
  Services/SourceLists/             ISourceListProvider (ExcelUpload now, MatrixApi/SalesforceApi later)
  Services/Comparison/              pure comparison engine (no I/O)
  Services/Excel/                   workbook read/write (ClosedXML)
  Services/Runs/                    run storage under App_Data/runs/<runId>/
  Models/                           DTOs and view models
ComplyAdvantageManagement.Tests/    xUnit tests for comparison, pagination, Excel parsing
docs/                               TECHNICAL_DOCUMENTATION.md, plans/, api-samples/
```

Conventions:
- **Controller → Service → upstream.** Controllers and views stay thin: controllers bind input, call a service and return a view model; views only render. Business logic lives in services; the comparison engine is pure and fully unit-testable.
- **Options pattern** for all config (`ComplyAdvantageOptions`, `ComparisonOptions`, `SourceListOptions`), validated on start (`ValidateOnStart`).
- **CA client**: typed `HttpClient` registered with `AddHttpClient`, standard resilience handler (retry with backoff on 429/5xx, honour `Retry-After`), `CaTokenHandler` (caches the bearer token, refreshes on expiry or 401 once), and `ReadOnlyGuardHandler`.
- **CA pagination rule**: `GET /v2/customers` returns at most 10,000 records per query, however you page. `total_count` can be larger (e.g. 20,978). Never assume paging alone gets everything. The extractor splits the pull into `created_at_from`/`created_at_to` windows, recursively halving any window whose `total_count` exceeds the cap, pages each window with the configured `page_size`, de-duplicates by `customer_identifier`, and verifies the final count against the overall `total_count`. A mismatch is surfaced in the UI and the Summary sheet, never hidden.
- **Entity shape**: individuals come back with a `person` object and companies with a `company` object. Model both; display name is built from whichever is present.
- **Source providers** implement `ISourceListProvider`. The API providers are stubs that report "not available" until the source API exists; the UI button is disabled from `SourceListOptions:ApiEnabled`.
- **Runs**: each upload/extract/compare is stored under `App_Data/runs/<runId>/` with a `run.json` manifest (timestamps, counts, config snapshot, no personal data). Controllers load results from the run by `runId` (in the route), so results survive a refresh and can be re-downloaded (`File(...)` result with the `.xlsx` content type).
- **Long-running work**: a full CA extract can take minutes, far longer than a request should block. Run it as a background job (hosted service) keyed by `runId`, redirect to a status view, and have that view poll a JSON status action for progress (page/window counts only).
- **Excel**: ClosedXML. Header row bold and frozen, auto-filter on, column widths set. Every workbook has a `Summary` sheet first.
- **Logging**: Serilog with structured templates; no string interpolation in log calls; no personal data (see safety rules).
- **Front end**: Razor views + Bootstrap + the jQuery already in `wwwroot/lib`; no SPA framework. Build forms with tag helpers (`asp-controller`/`asp-action`, which emit the antiforgery token) and validate it on every POST (register `AutoValidateAntiforgeryTokenAttribute` globally, or `[ValidateAntiForgeryToken]` per action). File uploads use `enctype="multipart/form-data"` and bind to `IFormFile`. Large tables are paged and searchable. Page them on the server from the stored run, not by rendering every row. Use the POST-Redirect-GET pattern after uploads and actions.
- Keep changes aligned with the existing template structure; don't introduce unrelated frameworks or architectural patterns unless asked.

## Documentation rule (applies to every task)

Documentation is part of "done". Any task that changes behaviour, config, endpoints, data flow, Excel layouts or business rules must update `docs/TECHNICAL_DOCUMENTATION.md` in the same task, including its changelog table. A task that changes nothing documentable says so in its completion summary. The `reviewer` agent treats missing or stale documentation as a finding.

## Git workflow

- Repository: GitHub, `origin` = https://github.com/sruthy-arekath/ComplyAdvantageManagement (use the `gh` CLI for PRs). `.gitignore` covers `bin/`, `obj/`, `.vs/`, `*.user`, `App_Data/runs/`, `*.xlsx`, local appsettings and secrets. If a test fixture genuinely needs a committed `.xlsx`, add a `!` exception for that path instead of force-adding it.
- Base branch: `development` (created from `main`). Nobody commits directly to it or to `main`.
- Branches: `task-<number>-<short-description>` or `bug-<number>-<short-description>`. Never invent a DevOps number; if none is given, ask.
- Create new branches from the latest base: `git fetch origin development` then `git switch -c <branch> --no-track origin/development`.
- Before changing code: check `git status`; if unrelated uncommitted changes exist, stop and ask.
- No history-rewriting or destructive git commands (`reset --hard`, `rebase`, `push --force`, amending pushed commits) without explicit confirmation.
- Commits use the user's configured Git author only. No AI attribution (`Co-Authored-By`, "Generated with Claude Code", etc.) in commits or PRs.
- Never commit `App_Data/runs/**`, uploaded Excel files, `appsettings.*.local.json`, user-secrets or `.claude/settings.local.json`.

**Hand-off commands** (after a completed task, wait for one of these; don't commit/push/PR on your own):
- `commit`: verify branch, stage only this task's files, commit with the task number, `git push -u origin <branch>`, create a PR to `development` (or give the compare link, title and description if the CLI isn't available). Never merge.
- `commit only`: commit, no push or PR.
- `push`: push only. `create PR`: PR only.

## How to work

Judge the level by complexity and risk, not file count.

**Small tasks** (one or two files, a clear bug fix, a view tweak, a config or doc update): normal session, no subagents. Still update the docs if behaviour changed.

**Medium tasks** (controller + view + service + model, a contained investigation): the user switches on Plan mode when they want it.

**Large / cross-cutting tasks**: use `/feature` (`.claude/skills/feature/SKILL.md`) when the user invokes it. It runs `architect` → plan in `docs/plans/` → waits for `approved` → branch → implement → build + test → `reviewer` → fixes → `tech-writer` → hand-off. Examples: scaffolding the solution, the CA extractor, the comparison engine, a new source provider, anything touching the read-only guard, and all of Phase 2.

If a request looks large or high-risk and `/feature` wasn't invoked, don't start implementing. Say it's a good candidate for `/feature` and ask.

Use `architect`, `reviewer` and `tech-writer` only from `/feature` or when the user names them. `/update-docs` runs `tech-writer` alone after small or medium tasks.

`.claude/settings.json` pre-allows `dotnet build/test/restore` and read-only git, and denies destructive git commands, `curl` DELETE/PATCH/PUT, and reading `App_Data/runs/**` and `secrets.json`. These rules only cover the Bash tool, so they back up the safety rules above; they don't replace them.

**Completion summary** for every task: what was implemented, changed files, build and test results, diff summary, current branch, documentation updated (or why not), review findings and open questions. Then stop.
