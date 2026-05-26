# Angular 21 & Angular 22: A Technical Overview for Medium-to-Advanced Developers

## TL;DR

- **Angular 21 (released Nov 20, 2025)** is the "consolidation" release that flips the framework defaults: **zoneless change detection is on by default**, **Vitest replaces Karma as the default test runner**, **HttpClient is auto-provided**, and **Signal Forms ship as experimental** in `@angular/forms/signals`. Three minor releases (21.0/21.1/21.2) iterated heavily on Signal Forms, added template arrow functions, and introduced the `ChangeDetectionStrategy.Eager` alias in preparation for v22.
- **Angular 22** is in release-candidate stage as of this writing (v22.0.0-rc.1 was tagged May 20, 2026 on GitHub; latest stable is 21.2.13). Its headline themes are confirmed: **Signal Forms graduate to stable** (per the v22.0.0-next.11 commit "graduate signal forms APIs to public API"), **OnPush becomes the default change detection strategy** (RFC #66779), new experimental signal primitives (`debounced()`, `injectAsync()`), and continued maturation of the Resource APIs. Selectorless components and `httpResource`/`rxResource` stability are widely discussed but not yet definitively confirmed as "stable" in primary v22 release notes.
- **The AI story is the differentiator.** Two layers: (1) the `ng mcp` CLI server makes Claude/Copilot/Cursor better at *writing* Angular code; (2) v22's experimental **WebMCP** (`provideExperimentalWebMcpTools`) makes your *running app* AI-callable — your Signal Forms become tools an agent can fill out on the user's behalf via the W3C `navigator.modelContext` standard.

---

## Executive Summary

Late 2025 / mid 2026 is the moment Angular's multi-year "signal-first, zoneless, AI-aware" strategy finally consolidates. The bets the team made between v16 (Signals) and v20 (zoneless stable, Vitest experimental, Resource API in preview) are now the *default* developer experience: a fresh `ng new` in v21 scaffolds a zoneless, Vitest-tested, control-flow-using application with no `provideHttpClient()`, no `*ngIf`, no `BehaviorSubject`, and an MCP config already in `.vscode/mcp.json.template`.

Angular 22, in RC at the time of writing and tracking toward GA in May–June 2026, is the version where the experimental layer becomes contract: Signal Forms move to public API, the `Default` change detection strategy is renamed `Eager` and `OnPush` becomes the new default, and new primitives like `debounced()`, `injectAsync()`, and `provideExperimentalWebMcpTools` extend the signal-first / agentic-tooling story. WebMCP in particular is a tell about Angular's bet on the *agentic UI* era — your forms become AI-callable tools via the browser-level `navigator.modelContext` standard. The framework is not chasing React or Svelte anymore — it has a coherent identity around fine-grained reactivity, explicit defaults, and AI-friendly conventions.

For teams: v21 is the upgrade target *today*; v22 is the planning target for Q3 2026. The migration path from v19/v20 → v21 → v22 is well-tooled (`ng update`, MCP `modernize` and `onpush_zoneless_migration`, schematics for NgClass/NgStyle/CommonModule/Jasmine→Vitest), and the breaking changes are mostly behavioral (zoneless default, OnPush default) rather than API removals.

---

## Angular 21 — What Shipped, Grouped by Category

Angular 21 was released **November 20, 2025** by Jens Kuehlers and Mark "Techson" Thompson. The official release event also introduced **Angie**, Angular's new mascot. Latest stable in the 21 line at time of writing is **21.2.13** (May 13, 2026).

### 1. Zoneless & Change Detection — *The Big Shift*

Zoneless is the headline architectural change of v21. It's the result of a four-version arc (experimental in v18 → developer preview in v20 → stable in v20.2 → **default in v21**) and it changes the mental model for how Angular knows when to re-render.

#### Status timeline
| Version | Status |
|---|---|
| v18 | Experimental (`provideExperimentalZonelessChangeDetection`) |
| v20 | Developer Preview (renamed to `provideZonelessChangeDetection`) |
| **v20.2** | **Stable** |
| **v21** | **Default for new apps** — `ng new` scaffolds without zone.js |
| v22+ | Continues as default; Zone.js opt-in remains supported but increasingly legacy |

#### What Zone.js was actually doing for you
Zone.js patched every async primitive in the browser — `setTimeout`, `setInterval`, `Promise.then`, every DOM event, `XMLHttpRequest`, `fetch`, `requestAnimationFrame`, MutationObserver — so that *after* any of them ran, Angular would run change detection across the entire component tree. It's why `this.count = this.count + 1` "just worked" in a click handler. The price: ~12–33 KB of polyfill, an extra microtask after every async operation, change detection runs that don't actually need to happen, debugger stack traces buried under `zone.js` frames, and broken `async/await` ergonomics (you had to wrap in `NgZone.run()` for callbacks from non-patched sources).

#### What replaces it in a zoneless app
Angular runs change detection only when it's *told to*. The notifications come from:
- **Signals** — when a `signal()`/`computed()`/`linkedSignal()`/`resource()` consumed in a template changes
- **`AsyncPipe`** — which calls `markForCheck()` internally on emission
- **Explicit `ChangeDetectorRef.markForCheck()`** — for the rare case you need to push manually
- **Native template events** — `(click)`, `(input)`, etc. still trigger CD on the bound component
- **`afterRenderEffect()`** for DOM-measurement scenarios

If your component updates state outside these channels (a `setTimeout` callback that mutates a plain field, an RxJS subscription that pushes to a `BehaviorSubject` not consumed via `AsyncPipe`, a third-party library callback) — **the UI silently won't update**. This is the #1 zoneless gotcha.

```typescript
// New default — no providers needed for zoneless
bootstrapApplication(AppComponent);

// Opt back in to Zone.js if you must (e.g., legacy third-party deps)
bootstrapApplication(AppComponent, {
  providers: [provideZoneChangeDetection({ eventCoalescing: true })]
});
```

#### Practical migration plan (for existing apps still on Zone.js)
1. **Audit `NgZone` usages** — `grep -r "NgZone\|zone\.js" src/` — every `NgZone.run()` / `NgZone.runOutsideAngular()` becomes a question: *what would this look like with a signal or `AsyncPipe`?*
2. **Move components to `OnPush` first.** Not strictly required for zoneless, but the cleanest dress rehearsal — if it works under OnPush + Zone.js, it'll work zoneless. Use the MCP `onpush_zoneless_migration` tool for a step-by-step plan.
3. **Convert state mutations to signals** in any component that fails the OnPush test. `BehaviorSubject` + `subscribe` + plain field → `signal()` + template binding.
4. **Flip the provider** in `app.config.ts`:
   ```typescript
   // Before
   providers: [provideZoneChangeDetection({ eventCoalescing: true })]
   // After
   providers: [provideZonelessChangeDetection()]
   ```
5. **Remove `zone.js` from `angular.json`** — both the `build` and `test` polyfills arrays. Delete any `import 'zone.js'` from `polyfills.ts`.
6. **Uninstall:** `npm uninstall zone.js`.
7. **Verify in the browser console:** typing `Zone` should throw `Zone is not defined`.

#### Common gotchas
- **Third-party callbacks** (e.g., Stripe, Google Maps, charting libraries that fire callbacks from non-Angular contexts) — wrap their callbacks in something that updates a signal, or use `ChangeDetectorRef.markForCheck()`.
- **`fakeAsync` in tests doesn't work without Zone.js patches** — use Vitest's fake timers instead. Karma/Jasmine tests using `fakeAsync` are the single biggest migration blocker for some teams.
- **`setInterval`-driven UI updates** that previously triggered CD by accident now silently freeze the UI. Move them to a signal that the template reads.
- **Server-side rendering** — zoneless makes hydration stability easier to reason about. `provideStabilityDebugging()` (v21.1) helps diagnose if hydration stalls past 9s.

#### `ChangeDetectionStrategy.Eager` alias — *v21.2, prep for v22*
`Default` is now deprecated and aliased to a new `Eager` strategy in preparation for OnPush-by-default in v22 (RFC #66779). An automatic migration in v22 will set `Eager` on components without an explicit `changeDetection` property to preserve behavior. Components you've already set to `OnPush` are untouched. After migration you can delete the explicit `OnPush` line since it becomes the default.

#### Bundle-size & performance context
The official v21 announcement frames the benefits as *"better Core Web Vitals, native async-await, ecosystem compatibility, reduced bundle size, easier debugging and better control"* without committing to specific numbers. Community estimates put the Zone.js savings at 12–33 KB depending on minification and tree-shaking. The change-detection savings are workload-dependent — apps with many `setTimeout`/RxJS-driven flows that never actually changed UI state will see the biggest wins.


### 1b. Signal Primitives Recap — What's Stable by v21

No new primitives were added in v21, but if your team came in at v17 or v18, the v19/v20 cycle quietly stabilized the whole signal-reactivity stack that Signal Forms now builds on. Here's the state of play in v21.

| Primitive | Introduced | Stable since | Purpose |
|---|---|---|---|
| `signal()` / `computed()` / `input()` / view queries | v16–v17 | v17–v19 | The core building blocks |
| `effect()` | v16 | **v20** | Side-effects in response to signal changes |
| `toSignal()` | v16 | **v20** | RxJS `Observable` → `Signal` bridge |
| **`linkedSignal()`** | v19 (experimental) | **v20** | Writable signal that auto-resets when a source changes |
| `model()` | v17.2 | v19 | Two-way bindable signal input (parent↔child) |
| `resource()` / `rxResource()` | v19 (experimental) | TBD v22 (likely) | Signal-native async data loading |
| `httpResource()` | v20.0 (experimental) | TBD v22 (likely) | Signal-native HTTP |

#### `linkedSignal` — the missing piece between `signal` and `computed`

`linkedSignal` is a **`WritableSignal<T>`** (it has been since v19 — "writable" is the whole point). The pattern it solves, in one sentence: *I want a value that auto-derives from a source signal, but the user can also override it manually, and when the source changes the override resets.*

**Why the dedicated primitive exists.** Angular University's framing (Vasco Cavalheiro, Feb 2026) is the clearest take: before `linkedSignal` you'd reach for `effect()` to imperatively `.set()` a writable signal whenever a source signal changed — but that pattern feels wrong because (a) writing signal values inside effects is now allowed but easily creates write loops, (b) the dependency between source and derived signal isn't visible from the signal declarations, (c) `effect()` is meant for *side effects*, not reactive value computation. `linkedSignal` makes the dependency declarative and removes the footgun.

**Canonical example — shopping cart quantity reset.** Each product in the catalog has a default quantity. When the user picks a product, the quantity field should reset to that product's default — but the user must still be able to override the quantity manually.

```typescript
import { signal, linkedSignal } from '@angular/core';

products = [
  { code: 'BEGINNERS', title: 'Angular for Beginners',   defaultQuantity: 10 },
  { code: 'SIGNALS',   title: 'Angular Signals In Depth', defaultQuantity: 20 },
  { code: 'SSR',       title: 'Angular SSR In Depth',     defaultQuantity: 30 }
];

selectedProduct = signal<string>('BEGINNERS');

// linkedSignal: writable AND auto-resets when source changes
quantity = linkedSignal({
  source: this.selectedProduct,
  computation: (productCode) =>
    this.products.find(p => p.code === productCode)?.defaultQuantity ?? 1
});

onQuantityChanged(q: string) { this.quantity.set(parseInt(q, 10)); }  // user override — works
onProductSelected(code: string) { this.selectedProduct.set(code); }    // changes source → quantity resets
```

**The naive `effect()` alternative — and why it's worse.** Same behavior, more rope:

```typescript
quantity = signal(1);

constructor() {
  effect(() => {                                          // ❌ imperative reactive update
    const code = this.selectedProduct();
    untracked(() => {                                     // ❌ have to remember untracked() to avoid loops
      this.quantity.set(
        this.products.find(p => p.code === code)?.defaultQuantity ?? 1
      );
    });
  });
}
```

The dependency between `selectedProduct` and `quantity` is invisible at the field declaration site — you only see it by reading the constructor. With `linkedSignal`, the relationship is right there in the type.

**Advanced: preserving user overrides across source changes.** The `computation` function also gets the *previous* value, which lets you write smarter reset logic — e.g., keep the user's selection if it still exists in the new source:

```typescript
selectedOption = linkedSignal({
  source: shippingOptions,
  computation: (newSource, previous) => {
    if (previous && newSource.includes(previous.value)) return previous.value;
    return newSource[0];
  }
});
```

**Decision rule (slide-ready):**
- Read-only derivation → **`computed()`**
- Plain mutable state → **`signal()`**
- Writable + auto-derives from a source + resets when source changes → **`linkedSignal()`**
- Two-way parent↔child binding → **`model()`**

**Why this matters for v21/v22:** Signal Forms uses linkedSignal-style patterns internally for field state — a field's `value()` is writable but tracks the underlying data signal. If `linkedSignal`'s mental model clicks, Signal Forms' design stops looking magic. The official v21 Signals tutorial (`angular.dev/tutorials/signals`) walks `signal` → `computed` → `linkedSignal` → `model` as the canonical learning order. See also Vasco Cavalheiro's deep-dive: `blog.angular-university.io/angular-linkedsignal`.

**Common pitfall:** reaching for `linkedSignal` when `computed` would do. It's a *specialized* primitive for a specific scenario, not a default. Most reactive state in your app should still be plain `signal()` + `computed()`.

**🔮 Coming next: the missing `set` option — PR #68708 (open, not yet merged)**

Until now, `linkedSignal` could **derive** state from a source, but it couldn't **write changes back** to that source. That gap matters when a store exposes read-only signals (the recommended pattern) but you want to bind them into a Signal Form, which needs writable signals. The two ideas — read-only stores and 2-way form binding — don't naturally fit together. Community workarounds emerged (Manfred Steyer, Kobi Hari, and Michael Egger-Zikes prototyped a `delegatedSignal` helper that pairs a derivation function with an update handler).

Now Alex Rickabaugh (alxhub, Angular core team) has opened **PR #68708 — `feat(core): add custom set option to linkedSignal`** on branch `alxhub:ls-set`. Per the PR description: *"Introduce a custom `set` option in `linkedSignal` options to allow overriding and customizing the default write-back behavior of writable signals. This lets developers route updates back to the source of truth (e.g., converting Fahrenheit back to Celsius) or perform other side effects like updating properties inside a parent signal."* The custom callback also receives the standard signal setter as its second parameter (`rawSet`) for direct internal mutation when needed.

Expected usage pattern (subject to change while the PR is in review):

```typescript
// Store exposes read-only signals
private readonly store = inject(FlightStore);

// linkedSignal with custom `set` — derives from store, writes back via store action
protected readonly filter = linkedSignal({
  source: () => ({
    from: this.store.from(),
    to:   this.store.to()
  }),
  // when consumer calls filter.set(...), this runs instead of mutating local state
  set: (value) => this.store.updateFilter(value.from, value.to)
});

// Now this Just Works — bidirectional, no `delegatedSignal` helper needed
protected readonly filterForm = form(this.filter);
```

**Status:** PR open as of May 14, 2026, targeting `angular:main`. Steyer (May 14): *"Now there is a PR that adds a `set` option to linkedSignal, allowing it to handle write-backs as well as derived reads."* **Not yet in any v22 RC build at time of writing** — track PR #68708 and the v22 changelog. If it lands before v22 GA, it removes the single biggest friction point between read-only stores (NgRx Signal Store, custom Signal Stores) and Signal Forms. If it slips, the `delegatedSignal` userland helper remains the workaround.

Slide-worthy framing: *"linkedSignal currently does reactive **reads**. PR #68708 makes it do reactive **writes** too. When it lands, stores + Signal Forms become a one-liner."*

### 2. Signal Forms (Experimental — `@angular/forms/signals`)

This is the headline feature of v21. Signal Forms is a brand-new, signal-native forms API designed to replace the reactive-forms / template-driven duality with a single, type-safe, schema-validated model. Marked **experimental** in v21; **graduating to stable in v22** per the v22.0.0-next.11 commit "feat: graduate signal forms APIs to public API".

#### Canonical example

```typescript
import { signal } from '@angular/core';
import { form, required, email, minLength } from '@angular/forms/signals';

private readonly credentials = signal({ email: '', password: '' });

protected readonly loginForm = form(this.credentials, form => {
  required(form.email, { message: 'Email is required' });
  email(form.email, { message: 'Email is not valid' });
  required(form.password, { message: 'Password is required' });
  minLength(form.password, 6, {
    message: pw => `Password should have at least 6 characters but has only ${pw.value().length}`
  });
});
```

```html
<form>
  <input [formField]="loginForm.email" />   <!-- v21.1+ ([field] renamed to [formField]) -->
  <input [formField]="loginForm.password" type="password" />
  @for (err of loginForm.email().errors(); track err.kind) {
    <p class="error">{{ err.message }}</p>
  }
</form>
<pre>{{ loginForm().value() | json }}</pre>
```

**Why it matters:** No more `FormGroup`/`FormControl` ceremony, no `valueChanges.pipe(takeUntilDestroyed())`, no separate `ControlValueAccessor`. Validation is declared as a schema function, errors are signals, and the underlying data signal is the single source of truth (two-way bound automatically).

#### Signal Forms evolution across v21 minors
| Feature | Version |
|---|---|
| Initial `form()` API, `required/email/minLength/...` validators | **21.0** |
| `[field]` directive renamed to **`[formField]`**; `field` property → `fieldTree` in `ValidationError` | **21.1** |
| `provideSignalFormsConfig({ classes: ... })` for auto CSS classes (incl. `NG_STATUS_CLASSES` for `ng-valid`/`ng-touched` etc.) | **21.1** |
| Custom *directive* controls via `FormValueControl` / `FormCheckboxControl` interfaces | **21.1** |
| `focusBoundControl()` + `errorSummary()` for accessible error focusing | **21.1** |
| **`FormRoot` directive** (`<form [formRoot]="loginForm">`) and **`submission` option** with `action`/`onInvalid`/`ignoreValidators` | **21.2** |
| **`transformedValue(parse, format)`** for raw↔model conversion in custom controls | **21.2** |
| **`registerAsBinding()`** to focus arbitrary elements in composite controls | **21.2** |
| **`SignalFormControl`** in `@angular/forms/signals/compat` for incremental bottom-up migration inside an existing `FormGroup` | **21.2** |
| Reactive `validateStandardSchema(() => zodSchema)` accepting signals | **21.2** |
| Dev-mode warning for hidden fields still rendered (`NG01916`) | **21.2** |
| Native number input `null` handling + `parse` errors | **21.2** |

A compat shim `@angular/forms/signals/compat` provides `compatForm()` accepting reactive controls — useful for incremental migration.

### 3. Templates & Control Flow

| Feature | Version | Status |
|---|---|---|
| Auto control-flow migration on `ng update` (`*ngIf`/`*ngFor`/`*ngSwitch` → `@if`/`@for`/`@switch`) | 21.0 | Stable |
| `RegExp` literals supported in templates (`{{ /\d+/.test(id()) }}`) | 21.0 | Stable |
| `@defer (on viewport({ trigger, rootMargin: '50px' }))` with IntersectionObserver options | 21.0 | Stable |
| Compiler diagnostic: required input read before init (compile-time error now) | 21.0 | Stable |
| Compiler diagnostic: unreachable/duplicated/inefficient `@defer` triggers | 21.0 | Stable |
| **Multi-`@case` fall-through** in `@switch` | 21.1 | Stable |
| **Spread operator** in templates: `[currentUser, ...admins]` and `sum(...counters)` | 21.1 | Stable |
| **Arrow functions** in template expressions: `count.update(n => n + 1)` | 21.2 | Stable |
| **`instanceof`** in template expressions | 21.2 | Stable |
| **Exhaustive `@switch`** via `@default never;` | 21.2 | Stable |

```html
<!-- 21.1: multi-case + spread -->
@switch (status) {
  @case ('draft')
  @case ('pending') { <p>Your document is not yet published</p> }
  @case ('published') { <p>Your document is live</p> }
  @default { <p>Unknown status</p> }
}
<button (click)="sum(...counters)">Sum all</button>

<!-- 21.2: arrow + exhaustive switch -->
<button (click)="count.update(n => n + 1)">+1</button>
@switch (status()) {
  @case ('idle') { <p>Idle</p> }
  @case ('loading') { <p>Loading</p> }
  @default never;   <!-- TS2322 if 'error' is reachable -->
}
```

### 4. Migrations & Style-Guide Schematics

```bash
ng generate @angular/core:ngclass-to-class-migration
ng generate @angular/core:ngstyle-to-style-migration
ng generate @angular/core:common-to-standalone    # drop CommonModule
ng generate refactor-jasmine-vitest               # Jasmine → Vitest (with --browser-mode, --add-imports, --include)
ng add tailwindcss                                # falls back to npm install + schematic
ng new my-app --style tailwind                    # built-in Tailwind setup
ng new my-app --file-name-style-guide=2016        # nostalgic mode: user.component.ts, UserComponent suffix
ng new my-app --test-runner=karma                 # opt back into Karma
```

### 5. Compiler & Core

- **`SimpleChanges<T>` generic** — type-safe `ngOnChanges` if you opt in.
- **`typeCheckHostBindings`** is now **on by default** (catches type errors in host bindings/listeners).
- **`ExperimentalIsolatedShadowDom`** view encapsulation strategy (experimental) — true bidirectional style isolation including from global styles.
- **`FormArrayDirective`** (classic forms) — finally allows an array as the top-level form node without wrapping in a `FormGroup`.

### 6. HTTP

- `HttpClient` is **provided in the root injector by default**. You no longer need `provideHttpClient()` unless customizing (interceptors, withFetch, etc.). Testing setup also collapses to just `provideHttpClientTesting()`.
- New `responseType` (mirrors Fetch's `Response.type`) and `referrerPolicy` options on the Fetch backend and `HttpResource` options.
- `HttpErrorResponse` exposes the underlying Fetch `responseType` — easier CORS diagnostics.

### 7. Router

| Feature | Version |
|---|---|
| `router.navigateByUrl('/', { scroll: 'manual' \| 'after-transition' })` per-navigation scroll override | 21.0 |
| Standalone `isActive()` returning a `Signal<boolean>` (tree-shakeable); `Router.isActive()` deprecated | 21.1 |
| Experimental `withExperimentalAutoCleanupInjectors()` — auto-destroys unused route injectors | 21.1 |
| Experimental `withExperimentalPlatformNavigation()` — opt-in integration with the browser Navigation API | 21.1 |
| `provideStabilityDebugging()` — diagnoses hydration stuck after 9s (auto-on in dev w/ `provideClientHydration()`) | 21.1 |
| `TrailingSlashPathLocationStrategy` / `NoTrailingSlashPathLocationStrategy` | 21.2 |
| `canMatch` receives third arg: `PartialMatchRouteSnapshot` with `params`/`queryParams`/`fragment` | 21.2 |

### 8. Accessibility — `@angular/aria` (Developer Preview)

A new package of headless, behavior-only ARIA primitives from the Angular Material team. The official Angular v21 announcement (blog.angular.dev, "Announcing Angular v21", Nov 20 2025) states: *"To start you have access to a set of 8 UI patterns encompassing 13 components that are completely unstyled and can be customized with your own styles."* Directives include `Accordion`, `Autocomplete`, `ComboBox`, `Grid`, `ListBox`, `Menu`, `Multiselect`, `Select`, `Tabs`, `Toolbar`, `Tree`.

```bash
npm install @angular/aria
```

Built on top of the CDK; handles keyboard interactions, ARIA attributes, focus management, screen-reader support. Targets EU European Accessibility Act compliance requirements.

### 9. Testing — Vitest is the New Default

`@angular/build:unit-test` builder is now **stable**, with Vitest as the default runner. Karma and Jasmine are still supported (`--test-runner=karma`), but `web-test-runner` and `jest` experimental builders are **deprecated and will be removed in v22**.

```jsonc
// angular.json (minimal config)
"test": { "builder": "@angular/build:unit-test" }
```

Key options: `coverage*`, `browsers` + `browserViewport` (Vitest 4 Browser Mode in real browsers), `reporters`, `setupFiles`, `providersFile`, `ui`, `watch`, `filter`, `list-tests`, `runnerConfig` for a custom `vitest-base.config.ts`. v21.2 adds a `headless` option and `ng add @vitest/browser-playwright|webdriverio|preview`.

**`fakeAsync` does not work in Vitest** (requires a patched Zone.js variant that doesn't exist for Vitest). Migrations leave a TODO and generate a detailed Markdown report by default.

#### Angular's built-in Vitest vs `@analogjs/vitest-angular`

Both run Vitest, but they're architecturally different: Angular's builder constructs its Vitest config **in-memory** from `angular.json`; AnalogJS uses a real `vite.config.mts` with `@analogjs/vite-plugin-angular`. That single architectural choice drives every difference below.

| Feature | AnalogJS | Angular built-in |
|---|---|---|
| Angular versions | v17+ | v21+ |
| Maintainer | Community (Brandon Roberts et al.) | Angular team |
| Builders / Schematics / Migrations | ✅ | ✅ |
| Browser Mode (Playwright / WDIO) | ✅ | ✅ |
| Fully configurable | ✅ | ⚠️ merged via `runnerConfig`, not pass-through |
| **VS Code / JetBrains Vitest extension** | ✅ | ❌ |
| Vitest CLI (`npx vitest`) | ✅ | ❌ |
| Vitest Workspaces | ✅ | ❌ |
| Custom environments / providers | ✅ | ❌ |
| Module mocking (`vi.mock`) | ✅ | ❌ |
| Buildable libs in monorepos | ✅ | ❌ |

*Source: AnalogJS docs comparison table, March 2026.*

**Pros — Angular's built-in builder**
- Zero config out of the box; `ng new` scaffolds it for you.
- Maintained by the Angular team — internal compiler/build improvements land here first.
- Schematics handle the Jasmine→Vitest migration, browser-mode setup, and Karma fallback.
- v22 RC (PR `angular/angular-cli#31729`, Nov 2025) improved `runnerConfig` merging: user `setupFiles`, Vite plugins, and partial test config now compose more predictably with the builder's in-memory defaults.

**Cons — Angular's built-in builder**
- **No VS Code Test Explorer / JetBrains gutter-icon support.** The Vitest extension can't discover the in-memory config, so no per-test gutter icons or one-click test runs.
- Can't run `npx vitest` directly — you go through `ng test`.
- No Vitest Workspaces (relevant for Nx monorepos).
- Module mocking with `vi.mock()` is limited.
- Third-party libs with ESM directory imports (Ionic, some legacy Material patterns) still hit ESM resolution bugs — see `ionic-team/ionic-framework#30982` and `angular/angular-cli#30429`, both open Feb 2026.

**Pros — `@analogjs/vitest-angular`**
- Real `vite.config.mts` → full IDE integration (VS Code Vitest extension, JetBrains Vitest runner) works out of the box.
- Full Vitest feature surface: workspaces, custom environments, module mocking, in-source testing, the lot.
- Battle-tested across v17–v22; works in Nx monorepos with buildable libs.
- AnalogJS 2.0 (Nov 2025) ships ESM-only, with a smaller install footprint and support for Vitest 3 and 4.

**Cons — `@analogjs/vitest-angular`**
- Community-maintained — Angular internal improvements have to be ported into the Vite plugin after the fact.
- Extra dependency surface vs. the Angular team's "batteries included" default.
- Slightly more setup (a real `vite.config.mts` + `test-setup.ts`).

**Common limitations (both setups)**
- `fakeAsync` / `flush` / `waitForAsync` don't work — Zone.js patches aren't applied to Vitest's `it`/`describe`. Use Vitest fake timers instead.
- Both can target the *same* `@angular/build:unit-test` builder if you prefer — AnalogJS supports either approach.

**Decision matrix**
- Greenfield Angular 21/22 app, standard testing needs → **Angular built-in**.
- VS Code Test Explorer is non-negotiable, you do heavy `vi.mock`, or you're on Nx with buildable libs → **AnalogJS**.
- Hybrid: Angular's `@angular/build:unit-test` with `runnerConfig: "vitest-base.config.ts"` gives you middle ground — more configurable, but doesn't fix the IDE Test Explorer gap.

### 10. CLI Quality-of-Life

- `ng serve --define VERSION="'1.0.0'"` — compile-time variable replacement at dev-server time (parity with build).
- `ng version --json` for automation.
- **Built-in Prettier** in v21.2: new apps get a `.prettierrc`, `ng generate` and `ng update` migrations auto-format changed files.

---

## AI Improvements (Across v21 and v22)

Angular's AI investment in 2025–2026 is conceptually three-layered: **(1) curated context for LLMs**, **(2) agentic tooling via MCP**, and **(3) measurable quality via the Web Codegen Scorer**.

### Curated context: `llms.txt` and best-practice prompts
- `angular.dev/llms.txt` — index of links to key resources.
- `angular.dev/llms-full.txt` — compiled, long-form description of how Angular works and best practices.
- `angular.dev/ai/develop-with-ai` — copy-paste system prompt for any LLM ("You are an expert in TypeScript, Angular, and scalable web application development … Use signals for state management … Implement lazy loading for feature routes … Do NOT use the `@HostBinding` and `@HostListener` decorators.")

### The MCP server (`ng mcp`)
Ships with the Angular CLI. Configured per-IDE via JSON in `.vscode/mcp.json`, `~/.cursor/mcp.json`, JetBrains AI Assistant settings, etc.

| Tool | First in | Stability | What it does |
|---|---|---|---|
| `list_projects` | v20.2 / v21.0 | Stable | Lists Angular projects in the workspace |
| `get_best_practices` | v20.2 / v21.0 | Stable | Returns Angular's latest best-practice prompt |
| `find_examples` | v20.2 / v21.0 | **Stable in v21.0** | Code examples (now with frontmatter `title`/`summary`/`keywords` and per-package/version filtering) |
| `search_documentation` | v20.2 / v21.0 | Stable | Live search of angular.dev, version-scoped |
| `modernize` | v20.2 / v21.0 | Experimental | Suggests + **runs** modernizations (signal migrations, control-flow migration, etc.) |
| `ai_tutor` | **v21.0** | Experimental | Interactive lesson — Smart Recipe Box; v21.1 added a Signal Forms lesson |
| `onpush_zoneless_migration` | **v21.0** | Experimental | Step-by-step migration plan to OnPush + zoneless |
| `build`, `devserver.start`, `devserver.stop`, `devserver.wait_for_build`, `test`, `e2e` | **v21.1** | Experimental | Lets agents build, run dev server, run unit/e2e tests programmatically |

Enable all experimental tools with `ng mcp --experimental-tool=all`. New workspaces get a `.vscode/mcp.json.template` pre-configured. Code examples are now embedded directly in `@angular/forms` via a SQLite database, with more packages to follow.

### WebMCP & Agentic UI — Apps that expose tools to AI agents

This is the architectural bet behind v22's biggest experimental API. It's worth understanding even if you're not adopting it today, because it reframes what a frontend application *is*.

#### The shift in direction
- **Traditional MCP** (the `ng mcp` server above): an AI agent talks to a *server* somewhere — your IDE asks the Angular CLI's MCP server for best practices, code examples, build commands. AI ↔ server.
- **WebMCP**: an AI agent calls your *live web app in the browser* directly. The user opens your app, types *"add 'feed the cats' and 'book flight to Hawaii' to my to-do list, then mark 'renew passport' as done"* to an AI assistant, and watches the app update in real time without typing a single task by hand. AI ↔ browser tab.

Same protocol family (MCP / function calling), opposite direction. The traditional MCP server makes Claude/Copilot/Cursor smarter at *writing* Angular code. WebMCP makes the *running app* itself agent-callable.

#### The underlying browser standard
WebMCP is built on the emerging **W3C ModelContext API** (`navigator.modelContext`). When a page registers a tool, it pins it to this browser-level object. The user's AI agent — a Chrome built-in agent, a browser extension like "WebMCP — Model Context Tool Inspector," or a desktop agent like Claude with browser access — reads from the same object to discover what the current page can do, then calls those tools via LLM function calling. The browser is the broker; your app and the agent never need to know about each other directly.

This means WebMCP isn't an Angular-only thing — it's a web platform feature Angular is wrapping idiomatically. React, Vue, Svelte, and vanilla JS apps can all participate via the same `navigator.modelContext` API.

#### Angular's WebMCP support (Experimental in v22)

The official API is **`provideExperimentalWebMcpTools`** (per `next.angular.dev/ai/webmcp`); some early blog posts use the shorter `provideWebMcpTools` name. Both refer to the same thing. There's also a lower-level `declareWebMcpTool()` primitive for advanced cases.

```typescript
// app.config.ts — global tools available everywhere
import { provideExperimentalWebMcpTools } from '@angular/core';
import { z } from 'zod';

export const appConfig: ApplicationConfig = {
  providers: [
    provideExperimentalWebMcpTools([
      {
        name: 'addTodo',
        description: 'Add a new item to the user\'s to-do list',
        inputSchema: z.object({
          title: z.string().describe('Short task title'),
          dueDate: z.string().datetime().optional()
        }),
        execute: async ({ title, dueDate }) => {
          const todoService = inject(TodoService);
          await todoService.add({ title, dueDate });
          return { success: true, id: todoService.lastId() };
        }
      }
    ])
  ]
};
```

The tools run **inside the Angular injection context** — `inject(TodoService)`, `inject(Router)`, signals, the whole DI tree are available. That's the framework's value-add over raw `navigator.modelContext`.

#### Route-scoped tools — must use auto-cleanup
Tools registered on a route need to *unregister* when the user navigates away. Otherwise the AI agent thinks they're still available and calls them on the wrong page. The pattern:

```typescript
// app.config.ts
provideRouter(routes, withExperimentalAutoCleanupInjectors())

// feature.routes.ts — tools only registered while user is on /todos
export const todosRoutes: Routes = [{
  path: 'todos',
  providers: [
    provideExperimentalWebMcpTools([ /* todo-specific tools */ ])
  ],
  component: TodosPage
}];
```

Without `withExperimentalAutoCleanupInjectors()`, route-scoped tools leak across navigations — a documented footgun.

#### Signal Forms auto-generate WebMCP tools — *the killer integration*
Per the official docs: *"when defining a Signal Form using `form`, pass the `experimentalWebMcpTool` configuration option to opt-in to an implicit WebMCP tool. Angular will inspect your form's data model and automatically generate a JSON schema for connected AI agents."*

```typescript
const checkoutForm = form(this.cartData, schema, {
  experimentalWebMcpTool: {
    name: 'completeCheckout',
    description: 'Fill and submit the checkout form on the user\'s behalf'
  }
});
```

That's it — your form is now AI-callable. The agent gets a JSON schema derived from the form's data model, validates against your existing Signal Forms schema, and the form's submission flow handles the rest. Zero extra plumbing.

**Important security caveat (from official docs):** Angular does NOT validate that the agent's inputs match your declared JSON schema. Validate explicitly in your `execute` function or in the form's schema before acting on the data.

#### What this means for product design — "agentic UI"
"Agentic UI" is the broader pattern WebMCP enables: applications designed so an AI agent can complete tasks the user describes in natural language, instead of clicking through wizards. Manfred Steyer (Angular GDE, AngularArchitects) has been the loudest voice on this in the Angular community — see his eBook at `agentic-angular.com` and the AngularArchitects workshop *"Agentic UI with Angular: Architecture & Patterns"* (`angulararchitects.io/en/training/agentic-ai-with-angular-architecture-patterns`).

The broader standards landscape Steyer points to:
- **MCP** — tool calling protocol (Anthropic, now widely adopted)
- **AG-UI** — agent-generated UI protocol
- **A2UI** — agent-to-UI communication protocol
- **MCP Apps** — server-driven MCP application pattern

For Angular teams the v22 takeaway is narrower: **WebMCP turns your existing Signal Forms and DI-aware services into AI-accessible tools with near-zero friction.** Whether your product wants that capability is a UX/business decision, not a technical one — but the framework primitive is now in place.

#### Status & stability
- `provideExperimentalWebMcpTools` — **Experimental in v22.** APIs subject to change even outside major versions.
- The WebMCP spec itself is very early — the Angular docs explicitly warn it's "undergoing frequent changes."
- Brian Treese's take (`briantree.se/angular-webmcp-tools`, May 2026): *"I'd treat this as a preview of where Angular and browser-based AI tooling are heading rather than a production recommendation today."*

Recommended adoption: pilot in an internal tool or a feature flag, watch the spec for breaking changes, hold off on customer-facing surfaces until the experimental tag is removed.

#### Adjacent ecosystem — AG-UI, A2UI, MCP Apps *(not Angular 21/22 features — context only)*

WebMCP is one slice of a broader agentic-frontend story. Three other standards are worth knowing about so you can answer the inevitable Q&A question, but they're **framework-agnostic open standards, not Angular core APIs** — there's no `provideAgUi()` in v22.

| Standard | Problem it solves | Direction | Angular integration |
|---|---|---|---|
| **WebMCP** | External AI agent (Chrome, extension, desktop Claude) calls *your app's* tools | Agent → app | `provideExperimentalWebMcpTools` (v22 experimental) ✅ |
| **AG-UI** | Standardized streaming protocol between your frontend and *your own* agent backend (runs, text deltas, tool calls) | App ↔ your agent | Community SDK + adapters for LangGraph, OpenAI Agents SDK, CrewAI, etc. |
| **A2UI** | LLM generates UI structure at runtime (component selection + data); rendered via a catalog | Agent → app UI | Community Angular renderer; usually transported inside AG-UI `ACTIVITY_SNAPSHOT` messages |
| **MCP Apps** | Server-driven MCP application pattern | Server → agent | Manfred Steyer's workshop covers patterns |

The clean mental separation: **WebMCP is about your app being a tool**, **AG-UI is about talking to your agent**, **A2UI is about the agent generating UI**. Different layers, often combined in one app.

For Angular teams that want to go deep here, Manfred Steyer's six-part "Agentic Angular" series is the canonical resource — covers AG-UI conceptually, the TypeScript SDK, Angular integration, A2UI, the AG-UI + A2UI combo via `ACTIVITY_SNAPSHOT`, and custom component catalogs. Reference repo: `github.com/angular-architects/flights42`. Articles at `angulararchitects.io/en/blog` (filter by Steyer, April–May 2026).

This is out of scope for v21/v22 release notes but in scope for "where is Angular heading in the agentic era." Treat as further reading, not a v22 adoption checklist.

### Web Codegen Scorer
Open-source tool published by the Angular team to score the quality of AI-generated web code. Simona Cotin (Google Engineering Manager for Angular), interviewed by Loraine Lawson in The New Stack (Oct 29 2025), described the closed loop: *"We iterated on the form of this specific prompt by basically running it against the evals tool, and we perfected it and improved it until it code generation got in the 97 to 100 score with these instructions for LLMs."* That score is only achievable when the AI actually has the prompts (i.e., when the MCP server is connected). This is the closed loop that gives Angular's AI story teeth beyond marketing.

---

## Angular 22 — What's Coming (or Just Shipped)

**Status note (May 22, 2026):** Angular 22 is currently in release candidate. **v22.0.0-rc.1** was tagged on GitHub on May 20, 2026 (`angular/angular`), with corresponding RC1 builds in `angular/angular-cli` and `angular/components`. The latest *stable* release in the v21 line is **21.2.13**. GA is widely expected in late May or June 2026 but no primary source has yet confirmed a final GA date. Features below are sourced from `v22.0.0-next.*` and `v22.0.0-rc.*` release notes and the Angular team's RFCs.

### Signals & Reactivity

#### Signal Forms — **Stable in v22** ✅
The headline graduation. Per the **v22.0.0-next.11** commit `7745365910`: *"feat: graduate signal forms APIs to public API"* under `forms`. Everything we built in v21.x — `form()`, `[formField]`, `FormRoot`, `submission`, `transformedValue`, `SignalFormControl`, `errorSummary`, `focusBoundControl`, `registerAsBinding`, `validateStandardSchema`, `provideSignalFormsConfig` — is now public API and covered by the deprecation policy.

#### Better numeric inputs — *Stable in v22*

Bonus fix that lands with the Signal Forms graduation: per commit `41b1410cb8a333a2ce6569483cd10866effc154d`, Signal Forms now correctly handles binding *text inputs* to *numeric* model fields. This sounds minor but it unlocks the MDN-recommended pattern of using `type="text" inputmode="numeric"` instead of `type="number"`, which is the right answer for most "numeric-looking" fields (age, postal code, CVV) where you don't actually want a spinner UI or the mousewheel-changes-value behavior of `<input type="number">`.

**The problem in v21.** Bind `<input type="text" inputmode="numeric">` to a `number | null` field in a Signal Form and you got a type mismatch — text inputs produced strings, the model wanted numbers. So everyone defaulted to `<input type="number">`, which carries real UX baggage: spinner controls nobody wants for a CVV, mousewheel scroll silently changes the value when the input is focused, and MDN explicitly [recommends avoiding it in many cases](https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Elements/input/number#using_number_inputs).

**v22's fix.** Signal Forms now accepts text inputs for numeric fields, auto-converts the string to a `number`, and crucially maps empty input to `null` (not `''`) so your `number | null` typing stays clean.

```typescript
interface SignupFormData {
  username: string;
  email: string;
  age: number | null;
}

protected model = signal<SignupFormData>({ username: '', email: '', age: null });

protected signupForm = form(this.model, s => {
  required(s.age, { message: 'Age is required' });
  min(s.age, 18,  { message: 'You must be at least 18' });
  max(s.age, 120, { message: 'Please enter a valid age' });
});
```

```html
<!-- v22 — this finally works correctly -->
<input
  type="text"
  inputmode="numeric"
  [formField]="signupForm.age"
  (keydown)="onAgeKeydown($event)" />
```

```typescript
// Keyboard handler — MDN warns browsers are inconsistent at enforcing numeric input
// even with the correct inputmode, so enforce it yourself
protected onAgeKeydown(event: KeyboardEvent) {
  const allowed = ['Backspace', 'Delete', 'Tab', 'Escape', 'Enter', 'ArrowLeft', 'ArrowRight'];
  if (allowed.includes(event.key)) return;
  if (!/^\d$/.test(event.key)) event.preventDefault();
}
```

**Why this is a slide-worthy moment.** It's a small fix that eliminates a class of subtle UX bugs that slip through code review — the kind your QA team raises three weeks after launch ("the age field changes when I scroll the page on mobile?"). For most apps, `type="text" + inputmode="numeric"` + schema validation is now the correct default. Use `type="number"` only when you genuinely have an incremental quantity where the spinner adds value (and even then, consider the mousewheel side effect). Source: Brian Treese, *"Better Numeric Inputs in Angular (Signal Forms + Angular 22)"* (Apr 2026, `briantree.se/angular-signal-forms-number-inputs`).

#### `Resource` / `rxResource` / `httpResource` — **Stability TBD; widely reported as stable**
Multiple secondary sources (CMARIX, Angular Academy, Brian Treese) report the Resource family graduating to stable in v22, and the v21.2 `ResourceSnapshot<T>` / `resourceFromSnapshots()` additions are clearly the kind of polish that precedes stabilization. However, **I could not find a primary v22.0.0-next/rc commit explicitly graduating `httpResource`/`rxResource` to public API at time of writing**. Treat as: *expected stable at v22 GA; verify in the official changelog before relying on it.*

Canonical examples:

```typescript
// resource — base signal-native async API
const userId = signal('42');
const user = resource({
  params: () => ({ id: userId() }),
  loader: async ({ params }) => fetch(`/api/users/${params.id}`).then(r => r.json())
});
// user.value(), user.isLoading(), user.error(), user.status()
```

```typescript
// httpResource — signal-native HTTP
const search = signal('');
const results = httpResource<User[]>(() =>
  `/api/users?q=${encodeURIComponent(search())}`
);
// Change `search`, request re-fires. No subscribe, no switchMap, no destroy.
```

```typescript
// rxResource — bridge to Observables
const id = signal(1);
const post = rxResource({
  params: () => ({ id: id() }),
  stream: ({ params }) => this.http.get<Post>(`/api/posts/${params.id}`)
});
```

#### `debounced()` — **Experimental in v22**
A signal-native debounce that returns a `Resource<T>`. Confirmed experimental (per Brian Treese, Fatima Amzil's Level Up Coding deep dive, and the v22-next changelog).

```typescript
import { debounced, resource } from '@angular/core';

protected readonly query = signal('');
protected readonly debouncedQuery = debounced(this.query, 1000);
// debouncedQuery.value(), .status(), .isLoading(), .error()

protected readonly products = resource({
  params: () => this.debouncedQuery.value() || undefined,
  loader: ({ params }) =>
    firstValueFrom(this.http.get<{products: Product[]}>(`/api/search?q=${params}`))
      .then(r => r.products)
});
```

Note: this `debounced()` (from `@angular/core`) is *different* from `debounce()` in `@angular/forms/signals`, which configures form-field UI-update frequency with `number | 'blur' | Debouncer<T>`.

#### `@Service` decorator — **Stable in v22**
Per the v22.0.0-next.11 commit `7aad302c3e`: *"fix: mark service decorator as stable"*. A new alternative to `@Injectable({ providedIn: 'root' })` with more explicit semantics.

### Dependency Injection

#### `injectAsync()` — **Developer Preview in v22**
Introduced in **v22.0.0-next.10** (commit `444b024d49`: *"feat(core): Add a `injectAsync` helper function"*). Lazy-loads a service via a dynamic import while still going through the injector and caching the result.

```typescript
import { Component, injectAsync, signal } from '@angular/core';

@Component({ /* ... */ })
export class PostEditorComponent {
  protected readonly content = signal('');
  protected readonly previewHtml = signal('');

  private markdownService = injectAsync(
    () => import('../markdown.service').then(m => m.MarkdownService)
  );

  async preview() {
    const svc = await this.markdownService();
    this.previewHtml.set(svc.render(this.content()));
  }
}
```

Replaces the old idiom of `Injector.get()` after a dynamic `import()` + manual promise caching. Supports prefetch options for triggers like `onIdle`.

### Change Detection

#### OnPush as default — Confirmed for v22 per RFC #66779
The RFC, "[Complete] RFC: Setting OnPush as the default Change Detection Strategy" (GitHub Discussion #66779, posted January 27, 2026 by @MarkTechson and @alxhub, now in Complete status), describes the exact mechanics:

```typescript
enum ChangeDetectionStrategy {
  OnPush,
  Eager,                      // new explicit name for old "Default"
  Default = Eager             // alias, deprecated
}
```

`ng update` to v22 will:
1. Add `changeDetection: ChangeDetectionStrategy.Eager` to any component lacking an explicit setting (preserving current behavior).
2. Rename `ChangeDetectionStrategy.Default` → `Eager` in your code.
3. Leave `OnPush` components untouched.

After migration, you can safely delete the `OnPush` line from components you'd already set it on. **Be aware:** library components without an explicit strategy will *silently* become OnPush in v22 — a documented footgun for `chart.js`-style wrappers that update state in a `subscribe()` without `markForCheck()`.

### Templates & Compiler (v22 prerelease additions)

- **Stricter template type checking** continues; `typeCheckHostBindings` is on by default since v21.
- Compiler-CLI **adds Node.js 26 support** (v22.0.0-next.11 commit `b8d3f36ed9`).
- TypeScript 5.9 baseline.

### HTTP

- **`FetchBackend` is the default** in v22. If you still need XHR-only features (e.g., upload progress in the classic API), opt back in with `provideHttpClient(withXhr())`.
- New `reportUploadProgress` and `reportDownloadProgress` options; the old `reportProgress` boolean is **deprecated**.
- **Breaking:** in Signal Forms, `min` and `max` validators **no longer accept string values** — must be `number | null`.

### Testing & Tooling

- `web-test-runner` and `jest` experimental builders are **removed** in v22.
- Vitest is the production-grade default. Browser Mode + Playwright/WebDriverIO are first-class via `ng add @vitest/browser-playwright|webdriverio|preview`.
- Vitest decision (Angular built-in vs AnalogJS) does **not** change with v22 — the in-memory-config architecture is the same; PR `angular/angular-cli#31729` improves merging but doesn't expose a real Vitest config. If you need VS Code Test Explorer or Vitest Workspaces, AnalogJS remains the answer in v22.

### Selectorless Components — Likely Developer Preview in v22 (not confirmed stable)

The long-running RFC for selectorless components ("eliminate the need for selectors in templates by referencing components directly via class names") has not, per primary-source v22 release notes I could find, landed as *stable* in v22. Multiple secondary sources confidently list it as a v22 feature; the Angular team's actual 2025 roadmap commitment was "run an RFC, gather feedback, prototype". Realistic expectation for v22 GA: **experimental or developer preview at most.** Mark this as "watch this space" rather than "ship today."

Anticipated syntax:

```typescript
import { UserCard } from './user-card';

@Component({
  imports: [UserCard],
  template: `<UserCard [name]="userName()" />`   // no string selector
})
export class ProfilePage {}
```

### WebMCP / Agentic UI — Experimental in v22
`provideExperimentalWebMcpTools()` + Signal Forms auto-tool generation. Covered in detail in the **AI Improvements → WebMCP & Agentic UI** section above. Headline: your Signal Forms become AI-callable tools with zero extra code. Treat as preview-quality, watch the spec.

---

## Deprecations & Breaking Changes

### Angular 21
- **`HammerModule`, `HammerGestureConfig`, `HAMMER_GESTURE_CONFIG` removed.** The HammerJS integration was deprecated in v20; v21 completes removal. Apps relying on it must install HammerJS directly + wire it manually, or migrate to Pointer Events.
- **`*ngIf` / `*ngFor` / `*ngSwitch` officially deprecated** (deprecated in v20, ongoing in v21) — auto-migration runs on `ng update`.
- **`NgClass` / `NgStyle` officially deprecated** in favor of `[class]` and `[style]` bindings — optional schematics provided.
- **`CommonModule` discouraged** — optional `common-to-standalone` migration.
- **Karma/Jasmine no longer the default** (still supported).
- **`Router.isActive()` deprecated** (v21.1) in favor of standalone `isActive()` signal.
- **Node.js minimums tightened**: per the Angular CLI `engines` field, Angular 21 requires the same `^20.19.0 || ^22.12.0 || ^24.0.0` range as v20; secondary sources (Arc.dev) cite **Node 22.22.0+ or 24.13.1+** as the practical floor for the latest 21.x patches. **TypeScript 5.9** is required.
- **NgModule** continues its slow death — increasingly hard-deprecated in tooling.

### Angular 22 (RC)
- **`ChangeDetectionStrategy.Default` aliased to `Eager`** and the underlying default flips to **`OnPush`**. Behavior of unmigrated apps is preserved by an `ng update` schematic, but **library components without an explicit strategy will silently become OnPush** — a real risk for older third-party packages.
- **`web-test-runner` and `jest` experimental builders removed**.
- **`HttpClient.reportProgress` deprecated**; use `reportUploadProgress` / `reportDownloadProgress`.
- **Signal Forms `min`/`max` validators no longer accept strings** — numeric only (breaking change vs v21.x experimental).
- **`Node.js 26` supported**; final word on Node 20 dropping awaits the GA changelog.
- **Selectorless components** are still in flux — APIs may shift right up to GA.

---

## Migration Notes — What Existing Apps Need to Know

### Upgrading to Angular 21 from v19 or v20
1. **Run `ng update @angular/core @angular/cli` first.** The schematics handle zoneless opt-in (adds `provideZoneChangeDetection()` if needed), HammerModule removal, control-flow migration, NgClass/NgStyle migrations (optional), and Jasmine→Vitest if you accept it.
2. **Audit Zone.js dependencies** before flipping zoneless: older RxJS-heavy libraries, some Angular Material legacy patterns, and analytics SDKs may assume Zone is running. Practical recipe: upgrade to v21 with `provideZoneChangeDetection()` left in, then run the MCP `onpush_zoneless_migration` tool to plan the OnPush+zoneless transition.
3. **Update Node and TypeScript** before touching app code. Node 22+, TypeScript 5.9.
4. **For Vitest:** `fakeAsync` tests must be rewritten manually. Use `--browser-mode` if you want real-browser tests (highly recommended for component tests). **Decide between Angular's built-in builder and AnalogJS** *before* running the migration (see §9 above) — switching later means rewriting `vite.config.mts` ↔ `angular.json` config.
5. **For Signal Forms:** experimental in v21; don't bet production forms on it yet, but new feature work is a fair candidate. Use the `@angular/forms/signals/compat` `SignalFormControl` for bottom-up migration inside an existing `FormGroup` (v21.2).

### Upgrading to Angular 22 (planning)
1. **Plan around the OnPush default flip.** Audit any in-house or vendored components without an explicit `changeDetection`. The auto-migration will preserve behavior by adding `Eager`, but ideally you've already gone OnPush before v22 lands.
2. **Stabilize Signal Forms code.** If you've been using v21.x Signal Forms, expect renames you already absorbed (`[field]`→`[formField]`, `field`→`fieldTree`) plus the `min`/`max` numeric-only constraint.
3. **Re-evaluate Selectorless eagerly only if it actually lands stable.** Watch the GA changelog before bulk migrations.
4. **Plan for `injectAsync()`** for non-critical-path services (analytics, markdown, chart libraries). Real bundle-size wins.
5. **Remove `web-test-runner` and `jest` experimental builder configs** before upgrading.
6. **Audit `HttpClient` progress reporting** — switch `reportProgress: true` to the new explicit `reportUploadProgress`/`reportDownloadProgress`.

### Decision matrix
- **On v17/v18:** Upgrade to v20 first, then v21. Arc.dev's upgrade guide is explicit: *"Teams on v17 or v18 should budget 3–6 days across two staged upgrade phases."*
- **On v19/v20:** Direct upgrade to v21 is realistic in 2–8 hours for most apps.
- **On v16 or earlier:** Treat as a 4–12 week re-architecture; lean on the MCP `modernize` tool plus AI assistants with Angular's best-practice prompts loaded.

---

## Recommendations

**Now (mid-2026):**
- **Adopt Angular 21 if you're on v19 or v20.** The upgrade is mechanical, the migration tools are mature, and you immediately benefit from zoneless defaults, Vitest, auto-provided HttpClient, control-flow migration, and the MCP server.
- **Install and connect the Angular MCP server** in your team's IDE configurations (Cursor, VS Code Copilot, Claude Code, JetBrains AI Assistant). The codegen-quality delta is meaningful; the curated prompts at angular.dev/ai + MCP context push AI-generated Angular code from "guesses based on Angular 14 training data" to "follows current Angular best practices."
- **Run the `onpush_zoneless_migration` MCP tool** on at least one feature module to get a concrete plan. This is also your dress rehearsal for the v22 OnPush-default flip.
- **Move new forms to Signal Forms** (still experimental in v21, but the API surface that v22 graduates is largely settled by v21.2). For existing reactive forms, the `SignalFormControl` compat bridge in v21.2 enables one-control-at-a-time migration.
- **Replace any new `HttpClient + subscribe` code with `httpResource()` or `resource()`**. The bundle-size and DX wins compound across the codebase.

**Plan for v22 GA (Q3 2026):**
- **Set a deadline on Karma/Jasmine retirement.** v22 won't remove them, but you're in legacy territory.
- **Audit third-party component libraries for explicit `changeDetection`.** Open issues against libraries that omit it before the v22 upgrade — they're the silent-OnPush-footgun candidates.
- **Pick 1–2 expensive services to convert to `injectAsync()`** and measure initial-bundle impact. Markdown editors, syntax highlighters, charting libs, PDF generators are the canonical wins.

**Benchmarks that should change your plan:**
- If a primary-source release note confirms `httpResource`/`rxResource` graduating to stable in v22 GA → accelerate the "all new HTTP code via Resource APIs" decision.
- If Selectorless components land as developer preview in v22 GA → start a pilot in a new feature module only.
- If Angular v19 is your current version → upgrade *now*; it reaches EOL on **May 19, 2026** and stops receiving security patches.

---

---

## Talks, Sessions & Further Watching

Curated picks for going deeper, with a clear preference for primary sources (Angular team members) and recognized GDEs.

### Signal Forms — canonical sessions

- **"Angular Signal Forms with Core Maintainer Alex Rickabaugh"** — YouTube `hKkiivsyrHA`, Jan 28 2026. The single most important watch on this topic. Alex (Angular core team, the person who built it) covers *why* Signal Forms exists, the design decisions, validators-as-schema, the `[formField]` directive, and how the team manages breaking changes during the experimental phase. ~45 min, deep and authoritative.
- **"Signal Forms meet the Signal Store"** — Michael Egger-Zikes (Angular GDE, AngularArchitects), ng-India April 2026, also at NG Poland Nov 2026. Slide deck on Speaker Deck. The canonical talk on combining Signal Forms with NgRx Signal Store for app-wide state — directly relevant if your app has more than trivial forms. Code repo: `github.com/mikezks/ng-india-2026`.
- **ng-conf 2025 LIVE Angular Team Keynote** — Mark Thompson, Alex Rickabaugh, Minko Gechev. YouTube `vdRKAtymFds`, Oct 17 2025. Sets the strategic context for v21/v22 (signal-first, zoneless, AI integration). Not Signal-Forms-specific but the framing slide.
- **"Live Q/A with the Angular Team | August 2025"** — Alex Rickabaugh and Mark Thompson. YouTube `R82ZAgL3BGU`. Audience questions on Signal Forms in its mid-experimental phase — useful for understanding what changed between then and v21.2.

### Signals & Resource API

- **Official v21 Signals tutorial** — `angular.dev/tutorials/signals`. Walks `signal` → `computed` → `linkedSignal` → `model` → `effect` → `toSignal`. Replaces older fragmented blog content. Mandatory if anyone on your team came in pre-v18.
- **Angular v20 announcement (Minko Gechev)** — `blog.angular.dev/announcing-angular-v20-b5c9c06cf301`. The release that stabilized `effect`, `linkedSignal`, and `toSignal`. Useful context for "how we got here."
- **JS Party podcast #310 — "Angular Signals with Pavel Kozlowski & Alex Rickabaugh"** — `changelog.com/jsparty/310`. Long-form history of why the Angular team chose signals; complementary to the Signal Forms talk above.

### Zoneless & change detection

- **Official zoneless guide** — `angular.dev/guide/zoneless`. Authoritative on which notifications trigger change detection in a zoneless app.
- **OnPush-by-default RFC #66779** — `github.com/angular/angular/discussions/66779` (MarkTechson, alxhub, Jan 2026). The primary source for the v22 OnPush flip. Read this before planning the v22 upgrade.
- **Brygida Fiejdasz — "No Zone, No Problem: Building Angular Apps without Zone.js"** — NG Poland 2026. Practical migration patterns.
- **Francesco Borzì — "How to migrate your Angular app to Zoneless"** — Feb 2026 on Medium. Battle-tested patterns from real migrations (Keira3, Royal BAM Group). Recommended over the more generic "remove zone.js" guides.
- **George Hulpoi — "Angular Zoneless: Migrating off Zone.js without breaking your UI"** — DEV Community. Covers the third-party callback hotspot in detail.
- **MCP `onpush_zoneless_migration` tool** — run it via `ng mcp` from your IDE for a project-specific migration plan.

### AI / MCP — IDE integration (traditional MCP)

- **Angular AI guide** — `angular.dev/ai/develop-with-ai`. The best-practice prompt + setup instructions for VS Code, Cursor, JetBrains, Claude Code.
- **`angular.dev/ai/mcp`** — MCP server tool list, per-IDE config.
- **"The Angular MCP Server in Angular 21: How AI Meets the CLI"** — Rahul Anandeshi on Medium. Practical walkthrough.
- **The New Stack interview with Simona Cotin** (Oct 29 2025) on the Web Codegen Scorer and the closed loop with `llms.txt`.

### WebMCP & Agentic UI — apps that expose tools to agents

- **Official Angular WebMCP guide** — `next.angular.dev/ai/webmcp`. The primary source. Covers `provideExperimentalWebMcpTools`, Signal Forms auto-tool generation, route-scoped tools with auto-cleanup.
- **"Your Web App, Meet Your AI Agent — Angular WebMCP"** — Fatima Amzil on Medium (May 2026). Full TODO app walkthrough showing the W3C `navigator.modelContext` standard, the Chrome WebMCP extension, and a working agent demo.
- **"Angular v22 WebMCP Tools Explained"** — Brian Treese (`briantree.se/angular-webmcp-tools`, May 2026). Practical DI-context use with `inject()`-aware tools. Pragmatic perspective on production-readiness.
- **Manfred Steyer — "Agentic UI with Angular" (eBook)** — `agentic-angular.com`. The broader pattern context: MCP, AG-UI, A2UI, MCP Apps. Recommended read before pitching agentic features internally.
- **AngularArchitects workshop** — *"Agentic UI with Angular: Architecture & Patterns"* (`angulararchitects.io/en/training/agentic-ai-with-angular-architecture-patterns`). The hands-on version of the eBook.
- **W3C ModelContext API** — the underlying browser standard. Specs still in flux; treat WebMCP as an early-adoption signal, not a production commitment.

**Adjacent ecosystem deep-dives (framework-agnostic standards, not Angular core):**
- **Steyer's "Agentic Angular" 6-part series** — `angulararchitects.io/en/blog` (April–May 2026). The canonical Angular-flavored walkthrough of AG-UI and A2UI:
  1. *Understanding AG-UI: The Standard for Agentic User Interfaces* (Apr 20)
  2. *AG-UI in Practice: The SDK for TypeScript* (Apr 23)
  3. *Implementing AG-UI with Angular* (Apr 27)
  4. *A2UI: How AI Generates Dynamic UIs at Runtime* (Apr 30)
  5. *Integrating A2UI with AG-UI in Angular* (May 1)
  6. *Custom Catalogs in A2UI: Your Own Components for AI-Generated UIs* (May 5)
- **Reference repo** — `github.com/angular-architects/flights42`. Flight-booking app with `agentic`, `a2ui-dynamic`, and `a2ui-dsl` branches showing different integration levels.
- **AG-UI spec** — `docs.ag-ui.com` (backed by CopilotKit, adapters for 14+ agent frameworks).

### Release recaps (third-party, in priority order)

1. **Ninja Squad recaps** — `blog.ninja-squad.com` covers every minor (21.0, 21.1, 21.2). The most reliable third-party source; technically precise, ships within days of each release.
2. **Angular.love** — `angular.love`. Good for thematic deep-dives (e.g., the Aria writeup).
3. **AngularArchitects blog** — `angulararchitects.io/blog`. Manfred Steyer and Michael Egger-Zikes. Strong on architecture and Signal Store integration.
4. **The Codersclan** — `thecodersclan.com/blog/angular-21-new-features-explained-real-examples`. Beginner-to-intermediate angle if you have mixed-experience attendees.

### Conferences to track (2026)

- **NG Belgrade** — May 7–8 2026 (already past). Recordings on YouTube. Signal-Store + Signal-Forms workshop content available.
- **NG Poland** — Nov 17 2026. Confirmed Signal Forms content (Egger-Zikes), zoneless (Fiejdasz), incremental hydration (Thalhammer), and an Angular team Q&A with Minko Gechev.
- **NG Baguette Conf 2026** — French Angular conf, CFP open.
- *Note:* NG-DE 2026 (Berlin) was cancelled.

---

## Caveats

- **Angular 22 is in RC** at the time of writing (v22.0.0-rc.1, May 20, 2026; latest stable is 21.2.13). Any v22 feature listed as "stable" reflects the trajectory of v22-next/rc commits and RFCs, not a shipped GA. **Signal Forms graduation is the most rigorously confirmed via the v22.0.0-next.11 commit "graduate signal forms APIs to public API"; `httpResource`/`rxResource` stability and Selectorless components are widely reported by secondary sources but not yet definitively confirmed in primary v22 release notes.**
- Several third-party blogs (CMARIX, Medium re-posts) write about v22 in past tense ("Angular 22 is here", May 16 2026). As of the GitHub `angular/angular` releases page on May 20, 2026 only RC builds exist. Treat confident "Angular 22 shipped X" claims with appropriate skepticism until you see a `v22.0.0` (no `-rc`) tag.
- **Angular Aria component count**: Ninja Squad enumerates 11 directives; the Angular team's own announcement says "8 UI patterns encompassing 13 components"; Angular.love says "12 patterns". The official figure from blog.angular.dev is 8 patterns / 13 components.
- **MCP `find_examples`** stability claim is from Ninja Squad's v21.0 recap; verify in the angular.dev MCP docs before quoting on a slide.
- **Performance benchmarks** for v21/v22 (specific TTI/INP/bundle-size numbers) were not surfaced from primary Angular team sources — most numbers in third-party posts are derived or estimated. The "~33 KB" zone.js bundle-size figure is Arc.dev's estimate, not an official Angular team number. Don't put specific percentages on a slide without a primary source.
- **The YouTube video referenced by the user** (`MbkjTNg2rcg`) could not be transcribed; treat its specific claims as needing direct viewing.