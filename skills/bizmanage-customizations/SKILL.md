---
name: bizmanage-customizations
description: Work with BizManage CRM customization repositories synced by bizmanage-cli. Use when the agent needs to edit, review, explain, validate, pull, or push BizManage instance custom code for objects/views, fields, frontend actions, custom pages, SQL reports, backend scripts, or CLI workflows in any BizManage instance (any subdomain.bizmanagecrm.com).
---

# BizManage Customizations

Use this skill for BizManage instance-specific codebases. These repos are local file representations of platform metadata and code that `bizmanage-cli` syncs to a live BizManage CRM instance at `https://<subdomain>.bizmanagecrm.com`. Everything here is instance-agnostic — it applies to any BizManage tenant, not a specific one. Any status IDs, table names, or business rules specific to one customer's instance belong in that instance's own repo/docs, never in this shared skill.

## Reference Sources

- [Maintained BizManage skills](https://github.com/bizmanagrcrm/bizmanage-skills): upstream customization guidance and the companion CLI skill.
- [BMML documentation](https://github.com/bizmanagrcrm/bmml): custom pages, elements, functions, forms, reports, and platform requests.
- [Server-side scripts](https://github.com/bizmanagrcrm/bmml/tree/main/server-side-scripts): execution, schedules, metadata, and injected modules.
- [Customization examples](https://github.com/bizmanagecrm/EXAMPLE_CUSTOMIZATIONS): four adapted educational snapshots covering construction estimating, property management, donor management, and a customer ordering portal. Start with the selected README and catalog; adapt schema, IDs, and configuration. These are reference workflows, not configured applications or proof of live behavior.

## First Moves

- Check the maintained upstream skill for relevant updates when accessible; reconcile them with the installed copy while preserving useful local additions. Verify version-sensitive guidance against the destination's deployed core.
- Read `bizmanage.config.json` first. It identifies the instance URL, alias, project name, and last pull.
- **Hard alias rule:** every `bizmanage` command that communicates with a platform instance must include the explicit alias from `bizmanage.config.json` via `-a <alias>`. Never rely on the CLI's `default` alias or assume the local project URL selects the remote target. This applies to `pull`, `push`, `test`, `status-detail`, `login`, `logout`, and any future remote command that supports `--alias`.
- Before every pull or push, run `bizmanage status-detail -a <alias>` or `bizmanage test -a <alias>` and verify that the reported instance URL exactly matches `bizmanage.config.json`. Stop if it does not.
- Treat the instance repo as the source for customizations. The usual layout is `src/objects`, `src/backend`, `src/reports`, and `src/pages`.
- Treat `.bizmanage/file-hashes.json` as local CLI cache, not source. Do not edit it manually except for explicit cache troubleshooting. Note it is also alias-agnostic (keyed by content hash, not by which instance was pushed to) — see [[bizmanage-cli]] for the deploy-verification workaround when the same customization needs pushing to more than one instance.
- If the user provides a core-system checkout, use it as read-only reference unless they explicitly ask for core edits.
- Before changing unfamiliar files, inspect nearby examples and the matching core helper route. File shape alone is often not enough.
- Do not run live-impact commands such as `bizmanage push`, `bizmanage pull -f`, report execution, or `scripts/run/...` unless the user explicitly asks.

## File Layout

- Objects/views live under `src/objects/<table-or-view>/definition.json`. `internal_name` is the stable key used by the platform.
- Fresh custom objects (verified core 3.1.452/3.1.484) and SQL reports (3.1.484) derive `internal_name` from `display_name`, overriding the supplied name. Create with a label that internalizes to the intended name; rename the label afterward if needed. Verify a fresh-instance import.
- Fields live under `src/objects/<table-or-view>/fields/<field_internal_name>.json`. Field docs in the core repo are usually at `docs/field.md`.
- Actions live under `src/objects/<table-or-view>/actions/<action_name>.json`, often with an optional same-named `.js` file. A missing `.js` is normal for pure URL/menu actions.
- Backend scripts live as paired `src/backend/<name>.json` metadata and `src/backend/<name>.js` code.
- Reports live as paired `src/reports/<internal_name>.json` metadata and `src/reports/<internal_name>.sql`.
- Pages live as paired `src/pages/<url-or-name>.json` metadata and `src/pages/<url-or-name>.html`.
- Some scripts/pages may refer to live platform records that are not present locally. Do not assume every referenced table, page action, or numeric report ID has a local file.

## Backend Scripts

- The core wraps script code in an async function with `envData`, then destructures `let { user, query, body } = envData`.
- Metadata `modules` controls injected variables. Known module names include `db`, `axios`, `https`, `_`, `moment`, `callScript`, `callReport`, `fieldHelper`, `pdfHelper`, `saveFileHelper`, `projectsHelper`, `deliveriesHelper`, `quotesHelper`, `mailHelper`, `templatesHelper`, `scheduleScriptExecution`, `proposalHelper`, `invoiceHelper`, `statusHelper`, `paymentsHelper`, `aiHelper`, `smsHelper`, `cheerio`, and `recycleBin`.
- Where the deployed core supports `aiHelper`, its `ask`, `generateJson`, and `runWithTools` methods use the execution user. Verify gateway/model availability and capabilities. For classification, prefer schema-bound `generateJson` results that code validates before applying changes.
- For scheduled integrations, verify the deployed scheduler's timezone, overlap, and missed-run behavior. Use explicit business timezones and persistent checkpoints/reconciliation for resumable external writes.
- Check deployed `backend_script` and `ai_tokens` metering when estimating automation costs; backend execution is not necessarily free, and nested script calls can add executions.
- A script must contain a `return` somewhere; core validation rejects scripts without one.
- The native admin **Run Script** control opens a session-authenticated GET `/scripts/run/<name>` (verified 3.1.485). The internal runner ignores metadata `method`; this is the existing UI path for empty-input setup scripts.
- Default timeout is 30 seconds unless metadata has `timeout`. Long-running scripts should declare a suitable timeout.
- On core 3.1.485, script timeout includes pool queueing and worker startup; timed-out queued work can still run. Do not treat a timeout as cancellation or automatically retry external effects.
- Script workers on core 3.1.484 have a one-connection DB pool. Propagate `trx` through every nested read/write, including permission lookups. `db.readOne(table, filter, order, clms, { trx })` takes options fifth; verify the deployed helper signature.
- `db.query` uses Knex bindings (verified 3.1.484): literal `?` inside quoted SQL regexes is still a placeholder. Avoid it, e.g. use `[-]{0,1}` for an optional minus.
- `callScript(name, envData)` waits for another shared worker; concurrent waiting parents can exhaust the worker pool. Prefer durable queued dispatch for background work. `callReport(id, data)` executes a custom report by numeric platform ID.
- On core 3.1.484, `scheduleScriptExecution(name, { after_ms, data: { body }, run_now: true }, user)` persists work without awaiting it. Fast wake hints cover delays up to 5 seconds; longer delays rely on native minute cron, which the isolated test kit disables. Keep long-job concurrency bounded so normal UI requests retain worker capacity.
- Native queue `done_at` records transport completion, including caught script failures. Persist and display business outcomes separately; do not treat Queue Done as successful work.
- On core 3.1.484, `saveFileHelper.save` stores buffers with PDF MIME. Verify a format-aware upload path before storing raw images or audio; changing the filename does not make the MIME correct.
- Backend scripts may contain live credentials or tokens. Do not print secrets in responses, logs, or skill updates.
- Verify script/request logging, native history and webhooks before choosing a credential setter. Keep secrets out of logged inputs/results, return status only, and enforce storage permissions; a hidden setting is not a secret store.
- If the instance repository has `docs/instance-settings.md`, follow its runtime configuration registry and set-only agent procedure. A hidden setting (`visible: false`) is not secret access control; never retrieve or display its value merely to verify configuration.

## Reports

- Report SQL uses named parameters like `:project`, `:year`, or `:month`; those names must match `params[].internal_name` in the report JSON.
- Report types include `list`, `execute`, and `execute-many`. `execute` and `execute-many` can mutate data; treat them as live operations.
- Report parameter objects need `internal_name` and `display_name`. Reserved names include `USER_ID`, `id`, and `autoExecute`.
- `settings.actions` can attach row actions by `view_name` and menu `internal_name`; `settings.csv.file_name` can define CSV filename templates.
- Numeric report IDs used in pages or scripts are live platform IDs. Local report metadata may not include those IDs, so resolve them from the live instance or surrounding page labels before editing.

## Pages And Actions

- Pages are AngularJS/EJS-era HTML snippets rendered by BizManage. Common globals/components include `$urlParams`, `$now`, `goToExternalUrl`, `executeActionByName`, `<bmml-report>`, and `<bmml-data-src>`.
- On core 3.1.493, Admin Custom Pages' "Public" column reflects published status; `access_policy` separately controls internal/external access.
- Core 3.1.484 enables Angular strict DI in both shells. Use explicit dependency annotations with `$injector.invoke` and other dynamically injected callbacks.
- Match select option types to `ng-model`: use numeric `ng-options`/`ng-value` for numeric models. Plain option values are strings and can leave the selection blank.
- Native shell links use `/#/page/<url>`; `/pages/<url>` serves the page template. A hard navigation to `/page/<url>` is not the shell route.
- Native landing URLs on core 3.1.484 must be query-free (`/page/<url>`): default navigation encodes queries into the page path. Set default state inside the page; child menu links can include queries.
- Menu/action docs in the core repo are usually at `docs/menu-item.md`.
- Action JSON is upserted by table plus `action_name`. Preserve `table_name` and `action_name` unless the user wants a new action.
- Action `.js` code runs in the frontend controller context and commonly uses `$http`, `notify`, `Modal`, `record`, `scope`, and route helpers, depending on action type and local examples.
- Action `type` values seen in the wild: `url` (navigate, `value` is a route/hash, can point at a page), `custom-script` (runs the paired `.js` in-browser), `function`, `http`, `group` (nests sub-actions), `ext-url`, `cache`.

### Building a custom page backed by a `custom-script` action

A page that needs its own data/logic (not just `<bmml-data-src>` binding) is usually built as **one hidden action doing the work, one thin page binding to its scope output**:

- Action JSON: `"is_hidden": {"dash": true, "grid": true}`, `"type": "custom-script"` — this keeps it out of row/menu UI; it's only ever invoked programmatically.
- Page HTML triggers it once on load: `<div ng-init="executeActionByName('<table>', '<action_name>')"> ... </div>`. Everything the action assigns onto `scope.*` is then available to bind against in the page's Angular templates (`{{scope.someValue}}`, `ng-repeat`, `ng-click="scope.someFunction()"`, etc. — inside the template you just write the bare name, not `scope.x`).
- Unlike a row action, a page-init action has no `record` — pull whatever context it needs from `scope.$urlParams` or its own data fetch.
- Define any interactive handlers as `scope.someFunction = async () => {...}` inside the action's `.js`; the page's `ng-click`/`ng-model` etc. can reference them directly once the action has run.

### Internal (session-authenticated) requests — no API key, no `restapi/*`

Code running inside a logged-in page/action (as opposed to an external script) talks to the platform same-origin, using the active session — never an API key or the public `restapi/*` prefix:

- Angular's `$http` service: `$http.post('v2/crud/<table>/read', {query: {...}, options: {columns: [...], limit: N}})` mirrors the shape of the public `restapi/crud/<table>/read` query builder, but hits the internal `v2/crud/*` path instead. `read-one`, `create`, `update`, `update-many`, `delete`, and `bulk_update` (`{internal_name, value, ids}` — sets one field across many records in a single call) are the other internal CRUD verbs.
- Plain same-origin `fetch('/<relative-path>', {...})` also works and appears in some built-in actions — either is fine; match whatever convention nearby code already uses.
- `$http.get`/`$http.post` (or `fetch`) against other internal, non-`restapi` routes work the same way when the platform exposes a dedicated endpoint for an action (e.g. a status-change route on a workflow-enabled table) — check whether the target instance already has a similar built-in action before assuming a route exists; internal route names beyond `v2/crud/*` are core-platform-specific and not guaranteed identical across every instance/version.

### Bulk operations with live progress

For loops that call the platform many times in a row, track progress on `scope` and force an Angular digest after every step so the UI updates as it runs, not just at the end:

```js
scope.progress = { current: 0, total: ids.length, failed: [] };
try { scope.$apply(); } catch (ex) { } // first apply before the loop starts

for (const id of ids) {
    try {
        await $http.get(`/some/internal/endpoint/${id}`);
    } catch (e) {
        scope.progress.failed.push(id);
    }
    scope.progress.current++;
    try { scope.$apply(); } catch (ex) { } // wrap every apply: throws if already mid-digest
}
```
Bind a progress bar in the page HTML to `progress.current / progress.total`. Report the outcome with `notify({message, classes: 'alert-success' | 'alert-danger'})`, and re-run the page's own load function afterward to refresh any on-screen counts.

### Dialogs, and where interactive controls have to live

- `Modal.confirm("plain string")` — resolves truthy/falsy; check the result and bail out.
- `Modal.confirm({header, msg, yes, variant})` — the object form **rejects (throws) on cancel** rather than resolving falsy, so it must be wrapped in `try/catch`. `msg` accepts HTML (e.g. `<b>`).
- `Modal.choice('<table>', {data: {header, displayProp}})` picks a record; `Modal.prompt({...})` takes free text.
- `Modal.open(templateUrl, ['$scope', fn], {size})` renders a **server-side template** (`/views/templates/...`), so it is only available for templates that already exist in core — a customization author cannot add one.
- **Consequence:** a modal cannot host custom interactive controls (checkboxes, selects) bound to your own state — its `msg` HTML is not Angular-compiled, so `ng-model` inside it will not bind. When a confirmation needs real input (e.g. "which of these groups should this bulk action touch?"), build the confirmation as an **in-page step** — a card/panel toggled by a `scope` flag, with normal `ng-model` checkboxes and explicit Cancel / Confirm buttons — instead of a modal. A deliberate click on a button labelled with the exact affected count is a sound confirmation.

### Counting records: watch for filtered list reads and null FKs

Two traps when a page summarizes or bulk-operates over records grouped by a lookup/FK column:

- **Object definitions can carry default `filters`.** A `definition.json` may include something like a filter on `archived` (`{"null": true}`), meaning list reads of that table return only unarchived rows. So a lookup table read can return *fewer* rows than the values actually referenced by your records, and any record pointing at a filtered-out row will silently have no matching entry.
- **The FK can be `NULL`.** Records with no value set match neither `status = X` nor `status != X` in SQL (`NULL` comparisons are unknown), so a "not equal to" style filter silently excludes them — while naive client-side JS grouping (`p.status !== TARGET`) silently *includes* them. The two disagree, and the totals stop reconciling.

Always group by the raw value, add an explicit bucket for `null`/unknown keys, and reconcile the per-group counts against the total record count before trusting any bulk action's "X records will change" number. Then show the breakdown to the user before writing anything.

### Custom reports as a data source for pages

A `list` report is a good way to load aggregates (a grouped `COUNT`) instead of reading every row into the browser. Practical constraints:

- Execute a report from a page with `$http.get('cust_report/query/<id>')` (also usable as `<bmml-report id="<id>">` or `<bmml-data-src source-url="/cust_report/query/<id>">`); rows come back as `res.data.rows`.
- **That endpoint is session-only.** Calling `restapi/cust_report/query/<id>` returns 404 and the internal path returns 401 with an API key, so a report cannot be executed with an API key the way `restapi/crud/*` can.
- **Report ids are numeric, per-instance, and are not in the files `bizmanage-cli` pulls.** The pulled `<report>.json` has `internal_name`/`display_name` but no id. So an id referenced from a page is instance-specific: hardcode it in a clearly-named constant (matching platform convention) and have the code **fall back** to a CRUD read when the constant is unset or the call fails — otherwise the page breaks when deployed to another instance.
- Reports are not exposed as a CRUD view: `restapi/crud/cust_reports/read` answers `The view "cust_reports" does not exist`.

### SQL conventions in reports

Reports are PostgreSQL. System tables are unqualified (`projects`, `customers`, `statuses`), customer-created tables live in a `custom.` schema (`custom.leases`, `custom.credit_reports`), and custom fields on any table carry a `__c` suffix (`p.subscription__c`). `order` is a reserved word — quote it (`s."order"`).

This whole pattern (hidden `custom-script` action + thin page + internal `$http`/`fetch` calls + `scope.$apply()`-driven progress) has been built and deployed end-to-end successfully — validated with `bizmanage validate`, and confirmed live with a scoped `push` followed by a `pull -f` back into git showing an empty diff.

## Core Lookup Map

When behavior is unclear, inspect these core-system files if available:

- `routes/cust-fields.js`: fields, views, data-by-internal-name, and actions REST endpoints.
- `routes/custom-pages.js` and `backend/custom-pages/`: page upsert, content, and render behavior.
- `routes/scripts.js` and `backend/custom-scripts/`: custom backend script upsert, execution, validation, module injection, cron, and timeout behavior.
- `routes/cust_report.js` and `backend/cust_reports.js`: custom report CRUD/execution, params, permissions, CSV, and settings.
- `backend/page-builder.js` and `backend/menus.js`: menu/action behavior.
- `docs/field.md`, `docs/menu-item.md`, `docs/CUSTOM_REPORT_SETTINGS.md`, `docs/form.md`: compact reference docs for customization schemas.

## Core Frontend Builds

- In the BizManage core repo, `npm run compile` starts webpack in watch mode when `PRODUCTION` is unset; it is not expected to exit.
- Check for an existing webpack watcher before starting another. Do not stack compile processes.
- An initial rebuild normally finishes within about 30 seconds. Verify the changed bundle in the browser after that; do not wait for the command to terminate.
- Keep the watcher running unless the user asks to stop it. Treat production compilation separately because `PRODUCTION=true` disables watch mode.

## CLI Workflow

- For native test coverage, pin compatible core and runner commits. The selected core's committed `scripts/customization-tests/coverage-contract.json` declares v1 (inline metadata) or v2 (native backend initializers). Never rewrite capability markers to bypass compatibility checks.
- Capture browser action coverage before a hard reload or tab close. End-of-test capture only sees the current document and can miss earlier executed actions.

`bizmanage-cli` uses the `bizmanage` binary. Official docs are at `https://github.com/bizmanagrcrm/bizmanage-cli`; see [[bizmanage-cli]] for the full command reference.

- `bizmanage status --changes-only`: show local changed/new/deleted customization files.
- `bizmanage validate`: validate changed files. Use `bizmanage validate --all` for a full check or `bizmanage validate -f <file>` for one file.
- `bizmanage pull -a <alias>`: pull current platform state. It blocks when local changes exist unless forced.
- `bizmanage push -a <alias>`: push changed files by default after validation and tests.
- Use selectors for narrow live operations: `--include reports`, `--include pages`, `--script <name>`, `--report <name>`, `--object <name>`, `--field table.field`, or `--action table.action`.
- Avoid `--skip-validation`, `--skip-tests`, `--all`, `pull -f`, and `clear-cache` unless the user asks or the need is clear and explained. Exception: `--skip-tests` is expected (not a shortcut) when the project has no `npm test` script configured at all — see [[bizmanage-cli]] troubleshooting.
- To verify a deploy actually landed: pull the same selection back with `-f` and check `git diff --stat` is empty.

## Updating This Skill

Whenever an agent discovers useful BizManage information missing from this skill, add it only after fully verifying that it applies generally across instances. Keep additions short, clear, and practical; update existing guidance instead of duplicating it, and exclude guesses, customer-specific details, and secrets.
