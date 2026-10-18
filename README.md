# Angular AST Architecture Scanner

A **read-only, static architecture discovery tool** for large **Angular 13** enterprise applications (200+ modules, components, services). Built on the **TypeScript Compiler API**, it resolves symbols through the compiler — path aliases, re-exports, and constructor DI — and produces a complete architecture map that a developer who has **never seen the codebase** can use to understand its structure in minutes.

No source is modified. No application is executed. The scan is purely static.

---

## Why this exists

Onboarding to a large Angular monolith is slow. New developers spend days answering questions like:

- *Where does execution start, and which module bootstraps what?*
- *Which NgModules are forRoot vs forFeature, and are their providers scoped correctly?*
- *Where is `HttpClientModule` imported — once, or scattered across features?*
- *Which arrows are lazy-loaded business-feature entry points?*
- *Which services are shared singletons, and is anything leaking between features?*
- *Which component uses which, and which API endpoints does each feature call?*
- *Where is NgRx state registered, and are effects/actions/reducers wired correctly?*

This scanner answers all of those questions by building **six symbol-resolved graphs** and emitting a Markdown report, a JSON model, and Mermaid diagrams — every finding tagged so nothing is mistaken for proof.

---

## What it produces

| Output | Description |
|---|---|
| `ARCHITECTURE-REPORT.md` | Full human-readable report: inventory, six graph sections, enterprise checks, ordered "start here" guide, limitations |
| `architecture-model.json` | Machine-readable model — all six graphs, all checks, all findings (for dashboards/CI) |
| `graphs/modules.mmd` | **G1** — NgModule import/export graph |
| `graphs/routing.mmd` | **G2** — Routes and lazy `loadChildren`/`loadComponent` boundaries |
| `graphs/components.mmd` | **G3** — Component usage graph (via template tags) |
| `graphs/services.mmd` | **G4** — Service constructor-injection graph |
| `graphs/api.mmd` | **G5** — HttpClient call graph (caller → endpoint) |
| `graphs/ngrx.mmd` | **G6** — NgRx Store/Effects/Actions/Reducers/Selectors map |

Every finding in the report carries one of three tags:

- **`[CONFIRMED]`** — read directly from source via the TypeScript compiler
- **`[INFERRED]`** — derived by graph or symbol reasoning
- **`[VERIFY]`** — needs human confirmation (dynamic patterns, runtime behaviour)

---

## The six graphs

### G1 — Module graph
Resolves every NgModule's `imports`, `exports`, `declarations`, and `providers` arrays to actual source files through the compiler. Detects `forRoot`/`forFeature` registrations.

### G2 — Routing graph
Finds `RouterModule.forRoot/forChild`, `Routes` arrays, and `loadChildren`/`loadComponent` lazy boundaries. Every lazy arrow is a **business-feature entry point** — the best place to start reading.

### G3 — Component graph
Parses each component's `templateUrl`/inline `template`, matches tag selectors against declared components/directives, and records component → component usage edges. Attribute selectors, `ngComponentOutlet`, and content projection are flagged `[VERIFY]`.

### G4 — Service dependency graph
Walks every `@Injectable()` constructor, resolves each parameter's type reference to its declaring file, and records service → service injection edges. Classifies each injectable as **Service**, **Guard**, **Interceptor**, or **Resolver**.

### G5 — API call graph
Detects `this.http.get/post/put/patch/delete/request/...` calls, extracts the HTTP verb and URL (string literals), and links caller file → endpoint. Dynamically composed URLs are flagged `[VERIFY]`.

### G6 — NgRx state graph
Finds `StoreModule.forRoot/forFeature`, `EffectsModule.forRoot/forFeature`, `createAction`, `createReducer`, and `createSelector`. Flags `forFeature` without a visible root store registration.

---

## Enterprise checks

Beyond the six graphs, the scanner runs sanity checks that catch classic Angular monolith bugs:

| Check | What it detects |
|---|---|
| **Root provider leak risk** | `providedIn: 'root'` services living inside `Shared/`, `Core/`, or `Feature/` paths — likely re-provided or duplicated |
| **Repeated Shared/Core imports** | Same `SharedModule`/`CoreModule` imported by multiple feature modules — provider scoping bugs |
| **Duplicate singletons by name** | Same class name declared with `providedIn: 'root'` in more than one file — accidental duplicate instances |
| **HttpClientModule outside root** | Imported anywhere other than `AppModule` — creates a second `HttpClient` with a different interceptor chain, silently skipping root interceptors |
| **forFeature without root** | `StoreModule.forFeature` present without a visible `forRoot` — bootstrap-order risk |
| **Lazy boundaries** | Every `loadChildren`/`loadComponent` — the business-feature entry points |

---

## Requirements

- **Node.js** ≥ 14
- **TypeScript** — must match the Angular version. For Angular 13:

  ```bash
  npm install --no-save typescript@4.9
