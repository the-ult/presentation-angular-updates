---
theme: seriph
title: Angular 21 & 22
info: |
  ## What's new in Angular 21 & 22
  A technical overview for medium-to-advanced Angular developers.

  Source overview doc: angular-21-22-overview.md
class: text-center
highlighter: shiki
drawings:
  persist: false
transition: slide-left
mdc: true
lineNumbers: false
colorSchema: dark
hideInToc: true
fonts:
  sans: 'Inter'
  mono: 'JetBrains Mono'
themeConfig:
  primary: '#DD0031'
seoMeta:
  ogTitle: 'Angular 21 & 22 — What''s New'
  ogDescription: 'Zoneless defaults, Signal Forms, WebMCP, and the road to v22'
---

# Angular 21 & 22

What's new — and what's coming

<img src="/assets/Angular Wordmark Gradient.png" alt="Angular" class="mx-auto mt-6 h-18 w-auto" />

<!-- <div class="pt-12">
  <span class="px-2 py-1 rounded cursor-pointer bg-white/10 hover:bg-white/20" @click="$slidev.nav.next">
    Press space to start <carbon:arrow-right class="inline"/>
  </span>
</div> -->

<div class="abs-br m-6 text-sm opacity-50">
  v21.2.14 stable · v22.0.0-rc.1 · May 2026
</div>

<!--
~20–25 minute talk, full version first. Audience: medium-to-advanced Angular devs who want to get up to date.
Three big stories: zoneless defaults, Signal Forms graduating, WebMCP / agentic UI.
Optional skips if we're tight: linkedSignal PR, adjacent ecosystem, honorable mentions.
-->

---
layout: center
class: text-center
---

# Where we are in mid-2026

<div class="grid grid-cols-2 gap-8 pt-8 text-left">

<div>

### Angular 21 — Nov 20, 2025

The **consolidation** release:
- Zoneless **default**
- Vitest **default** runner
- Templates / compiler polish
- Signal Forms (experimental)
- `@angular/aria` accessibility primitives

Latest stable: **21.2.14**

</div>

<div>

### Angular 22 — RC, GA ~June 2026

The **graduation** release:
- Signal Forms → **stable**
- **OnPush** as default change detection
- `resource()`, `rxResource()`, `httpResource()` → **stable**
- `injectAsync()` **stable**
- WebMCP for agentic UI
- FetchBackend default for `HttpClient`

Current: **22.0.0-rc.1** (May 20, 2026)

</div>

</div>

<!--
This is the orientation slide. Set context fast — most people came in at v17 or v18 and need to know v19/v20 already happened. 30 seconds max.
-->

---
layout: section
---

# Part 1 — Zoneless

The biggest mental model shift since standalone

---

# Zoneless: the timeline

<div class="pt-4">

| Version | Status |
|---------|--------|
| v18 | Experimental |
| v20 | Developer Preview |
| **v20.2** | **Stable** |
| **v21** | **Default for new apps** ⭐ |
| v22 | Continues; Zone.js opt-in |

</div>

<v-click>

<div class="pt-6 text-lg">

`ng new` in v21 scaffolds **without** `zone.js`. If you still need ZoneJS-based change detection, keep `zone.js` in the build and add `provideZoneChangeDetection()` explicitly.

</div>

</v-click>

<!--
Most people in the room are still on Zone.js. The point of this slide: the team isn't testing zoneless anymore, they're shipping it as the default. Behavior of existing apps doesn't change — but the team's center of gravity has shifted.
-->

---

# What Zone.js was actually doing

<div class="grid grid-cols-2 gap-6 pt-4 text-sm">

<div>

### The "magic"

Zone.js **patched every async API**:

- `setTimeout`, `setInterval`
- Every DOM event
- `Promise.then` / `async`-`await`
- `XMLHttpRequest`, `fetch`
- `requestAnimationFrame`

After any of them ran, Angular ran change detection across the **entire component tree**.

</div>

<div>

### The price

- ~12–33 KB of polyfill
- Extra microtask after every async op
- CD runs that don't change anything
- Stack traces buried in `zone.js`
- Async/await + newer browser APIs were awkward under patching

</div>

</div>

<!--
Explain the magic so people understand what they're trading away. Don't skip this — without context the migration looks scary.
-->

---

# What replaces Zone.js

<div class="pt-4">

Change detection runs only when Angular gets an **official notification** — via:

<v-clicks>

- **Updating a signal** that's read in a template
- **`AsyncPipe` / `ChangeDetectorRef.markForCheck()`**
- **Bound host or template listeners** — `(click)`, `(input)`, etc.
- **`ComponentRef.setInput()`**
- **Attaching a view** that was already marked dirty

</v-clicks>

</div>

<v-click>

<div class="mt-8 p-4 border-l-4 border-amber-500 bg-amber-50/10">

⚠️ **The #1 zoneless gotcha:** state mutated outside these notifications (a `setTimeout` callback writing a plain field, or reactive forms state updated via `setValue()` / `patchValue()` and read directly in the template) — **the UI silently won't update.** Bridge it with a signal, `AsyncPipe`, or `markForCheck()`.

</div>

</v-click>

<!--
Walk through the official notification mechanisms slowly. If people remember one thing, it should be this: in zoneless Angular, updates must flow through signals, AsyncPipe/markForCheck, listeners, setInput, or a dirty attached view. Call out reactive forms explicitly — `setValue()` and `patchValue()` do not schedule a refresh on their own.
-->

---

# Bootstrap: before vs now

````md magic-move {lines: true}
```typescript
// v17 / v18 — Zone.js everywhere by default
bootstrapApplication(AppComponent, {
  providers: [
    // (Zone.js was implicit — just imported in polyfills.ts)
  ]
});
```

```typescript
// v20 — opt-in zoneless
bootstrapApplication(AppComponent, {
  providers: [
    provideZonelessChangeDetection() // explicit opt-in
  ]
});
```

```typescript
// v21+ — zoneless is the default ✨
bootstrapApplication(AppComponent);
// No provider needed to enable it.
// Do verify `provideZoneChangeDetection()` is not configured anywhere.
```

```typescript
// v21 — opt BACK IN to Zone.js (legacy / third-party)
bootstrapApplication(AppComponent, {
  providers: [
    provideZoneChangeDetection({ eventCoalescing: true })
  ]
});
// Keep `zone.js` in the build if you opt back in.
```
````

<!--
Magic Move animates between four states. Click through them slowly. The key distinction: v20 is the version where you add `provideZonelessChangeDetection()`. In v21+, zoneless is already the default, so the migration is mostly removing overrides and removing ZoneJS from the build. The last panel is the safety net for teams that can't move yet.
-->

---

# Migrate to zoneless in 3 passes

<div class="grid grid-cols-3 gap-4 pt-2 text-[0.92rem] leading-snug">

<div v-click class="rounded-xl border border-white/10 bg-sky-500/10 p-3">
  <div class="text-[0.68rem] font-semibold uppercase tracking-[0.18em] opacity-60">Phase 1</div>
  <div class="mt-2 text-lg font-semibold">Find blockers</div>
  <ul class="mt-2 space-y-1">
    <li>Search <code>provideZoneChangeDetection</code></li>
    <li>Remove <code>zone.js</code> / <code>zone.js/testing</code></li>
    <li>Replace the stability APIs</li>
  </ul>
</div>

<div v-click class="rounded-xl border border-white/10 bg-fuchsia-500/10 p-3">
  <div class="text-[0.68rem] font-semibold uppercase tracking-[0.18em] opacity-60">Phase 2</div>
  <div class="mt-2 text-lg font-semibold">Rewire updates</div>
  <ul class="mt-2 space-y-1">
    <li>Rehearse with <code>OnPush</code></li>
    <li>Move plain field writes to signals</li>
    <li>Bridge RxJS/forms with <code>AsyncPipe</code> / <code>markForCheck()</code></li>
  </ul>
</div>

<div v-click class="rounded-xl border border-white/10 bg-emerald-500/10 p-3">
  <div class="text-[0.68rem] font-semibold uppercase tracking-[0.18em] opacity-60">Phase 3</div>
  <div class="mt-2 text-lg font-semibold">Verify + clean up</div>
  <ul class="mt-2 space-y-1">
    <li>v20: add the zoneless provider</li>
    <li>v21+: remove overrides</li>
    <li>Tests: <code>fixture.whenStable()</code> + optional no-changes check</li>
  </ul>
</div>

</div>

<v-click>

<div class="mt-3 rounded-xl border-l-4 border-amber-500 bg-amber-50/10 p-3 text-sm leading-snug">

<strong>Keep:</strong> <code>NgZone.run()</code> / <code>runOutsideAngular()</code><br/>
<strong>Remove:</strong> <code>onStable</code>, <code>onUnstable</code>, <code>onMicrotaskEmpty</code>, <code>isStable</code>

</div>

</v-click>

<!--
This is the same official migration guidance, but grouped into three buckets so the audience can actually read it. Narrate it as: find blockers, rewire updates, verify. Say the concrete search/replace targets out loud, and explicitly call out the nuance most teams get wrong — `NgZone.run()` stays, the stability APIs go.
-->

---
layout: two-cols-header
---

# Gotchas you'll actually hit

::left::

### Third-party callbacks

Stripe, Google Maps, chart libraries that fire callbacks from non-Angular contexts.

→ Wrap their callbacks in something that updates a **signal**, or call `markForCheck()` manually.

### Tests that still load `zone.js`

`TestBed` uses zone-based change detection by default when `zone.js` is present.

→ Add `provideZonelessChangeDetection()` in tests and prefer `await fixture.whenStable()` over `fixture.detectChanges()`.

::right::

### `setInterval`-driven UI

Previously triggered CD by accident. Now silently freezes the UI.

→ Move the interval target to a **signal** the template reads.

### Reactive forms state

`setValue()`, `patchValue()`, `FormArray.push()` update form state but do **not** schedule change detection.

→ Reflect through signals or call `markForCheck()` when the template reads it.

<!--
These are the four traps worth mentioning aloud: third-party callbacks, tests still loading ZoneJS, interval-driven UI, and reactive forms state. The chart-library callback and direct reactive-forms bindings are the two most relatable examples.
-->

---
layout: section
---

# Part 2 — Signals primitives

The foundation everything else is built on

---

# Signals timeline — what stabilized when

<div class="text-sm pt-2">

| Primitive | Introduced | Stable since | Purpose |
|---|---|---|---|
| `signal()` / `computed()` | v16 | **v17** | Core state + derivation |
| `input()` | v17.1 | **v19** | Component inputs |
| `effect()` | v16 | **v20** | Side-effects |
| `toSignal()` | v16 | **v20** | RxJS interop |
| `linkedSignal()` | v19 | **v20** | Writable + auto-reset |
| `model()` | v17.2 | **v19** | Two-way input |
| `resource()` / `rxResource()` | v19 | **v22** | Async resources |
| `httpResource()` | v20 | **v22** | HTTP resources |

</div>

<v-click>

<div class="pt-4 text-sm opacity-80">
Cheat sheet: <code>v17</code> core signals, <code>v19</code> inputs/models, <code>v20</code> effects + interop, <code>v22</code> async resources.
</div>

</v-click>

<!--
Quick reference. Useful for Q&A. The story changed since the first draft: the async trio is now officially stable in v22.
-->

---

# Where `linkedSignal()` fits

<div class="pt-2">

The pattern: *derive it by default, let the user override it, reset it when the source changes.*

</div>

<v-click>

<div class="pt-4 grid grid-cols-4 gap-2 text-center text-sm">
  <div class="p-3 bg-blue-500/20 rounded">
    <code class="font-bold">computed()</code><br/>
    <span class="opacity-70">derived only</span>
  </div>
  <div class="p-3 bg-green-500/20 rounded">
    <code class="font-bold">signal()</code><br/>
    <span class="opacity-70">plain writable</span>
  </div>
  <div class="p-3 bg-purple-500/20 rounded ring-2 ring-purple-400">
    <code class="font-bold">linkedSignal()</code><br/>
    <span class="opacity-70">writable + auto-reset</span>
  </div>
  <div class="p-3 bg-orange-500/20 rounded">
    <code class="font-bold">model()</code><br/>
    <span class="opacity-70">parent↔child binding</span>
  </div>
</div>

</v-click>

<!--
Decision rule lives in everyone's head as a 2x2 grid. Show it visually.
-->

---

# Don’t derive values with `effect()`

````md magic-move {lines: true}
```typescript
// ❌ Derived value hidden inside effect()
quantity = signal(1);

constructor() {
  effect(() => {
    const code = this.selectedProduct();
    untracked(() => {                 // 👈 don't forget this
      this.quantity.set(
        this.products.find(p => p.code === code)?.defaultQuantity ?? 1
      );
    });
  });
}
// Dependency is hidden from the field declaration.
// Easy to create loops. Effects are for SIDE effects, not derived state.
```

```typescript
// ✅ With linkedSignal
selectedProduct = signal<string>('BEGINNERS');

quantity = linkedSignal({
  source: this.selectedProduct,
  computation: (code) =>
    this.products.find(p => p.code === code)?.defaultQuantity ?? 1
});

// Reads like a signal, writes like a signal:
onQuantityChanged(q: string) { this.quantity.set(parseInt(q, 10)); }
onProductSelected(code: string) { this.selectedProduct.set(code); }
// Source changes → quantity resets. User edits → quantity stays until then.
```
````

<div v-click class="text-sm opacity-70 pt-2">
Example adapted from Vasco Cavalheiro, <em>angular-university.io/angular-linkedsignal</em>
</div>

<!--
This is THE moment to slow down. The two versions side-by-side via Magic Move makes the value of linkedSignal click instantly.
-->

---

# Maybe next: `linkedSignal({ set })` (open PR)

<div class="pt-2 text-sm">

**The dilemma** (Steyer, May 14, 2026): stores expose **read-only** signals. Signal Forms need **writable** signals. They don't naturally fit.

</div>

<v-click>

```typescript
// ⚠️ PR #68708 — OPEN, NOT YET MERGED — by alxhub (Angular core)
private readonly store = inject(FlightStore);

protected readonly filter = linkedSignal({
  source: () => ({
    from: this.store.from(),
    to:   this.store.to()
  }),
  // when consumer calls filter.set(...), this runs INSTEAD of mutating local state
  set: (value) => this.store.updateFilter(value.from, value.to)
});

// Bidirectional — read-only store ↔ writable Signal Form
protected readonly filterForm = form(this.filter);
```

</v-click>

<v-click>

<div class="pt-2 text-sm opacity-80">
If it lands → read-only stores + Signal Forms become a one-liner.<br/>
If it slips → community <code>delegatedSignal</code> helper remains the workaround.
</div>

</v-click>

<!--
Hedge this honestly: PR is OPEN, not merged. Three explicit callouts so you don't accidentally claim it shipped. If you give this talk after May 2026 GA, check the PR status first.
-->

---
layout: section
---

# Part 3 — Signal Forms

The headline feature of v21 — and the v22 graduation

---

# The forms story so far

<div class="grid grid-cols-3 gap-4 pt-6 text-[0.95rem] leading-snug">

<div class="p-4 bg-gray-500/10 rounded">

### Template-driven

<div class="mt-3 space-y-2">
  <div><code>[(ngModel)]</code></div>
  <div>Quick to start.</div>
  <div>Logic and validation stay in the template.</div>
</div>

</div>

<div class="p-4 bg-gray-500/10 rounded">

### Reactive

<div class="mt-3 space-y-2">
  <div><code>FormGroup</code> / <code>FormControl</code></div>
  <div>Explicit and testable.</div>
  <div>More setup code and RxJS wiring.</div>
</div>

</div>

<div class="p-4 bg-purple-500/20 rounded ring-2 ring-purple-400">

### Signal Forms ⭐

<div class="mt-3 space-y-2">
  <div><code>form()</code> + signal model</div>
  <div>Model-first and type-safe.</div>
  <div>Validator functions. Errors are signals.</div>
</div>

</div>

</div>

<v-click>

<div class="pt-6 text-center">
<span class="px-3 py-1 bg-yellow-500/30 rounded">Experimental in v21</span>
→
<span class="px-3 py-1 bg-green-500/30 rounded">Stable in v22 ✅</span>
</div>

</v-click>

<!--
Frame why a third forms API exists. The duality of template-driven vs reactive was a real pain point — Signal Forms unifies them.
-->

---

# Signal Forms in 3 steps

````md magic-move {lines: true}
```typescript
// Step 1 — define the model as a signal
private readonly credentials = signal({ email: '', password: '' });
```

```typescript
// Step 2 — add schema-style validators
import { form, required, email, minLength } from '@angular/forms/signals';

protected readonly loginForm = form(this.credentials, form => {
  required(form.email, { message: 'Email is required' });
  email(form.email,    { message: 'Email is not valid' });
  required(form.password, { message: 'Password is required' });
  minLength(form.password, 6, {
    message: pw => `Min 6 chars, you have ${pw.value().length}`
  });
});
```

```html
<!-- Step 3 — bind fields and show errors -->
<form>
  <input [formField]="loginForm.email" />
  <input [formField]="loginForm.password" type="password" />

  @for (err of loginForm.email().errors(); track err.kind) {
    <p class="error">{{ err.message }}</p>
  }
</form>
```
````

<!--
Three Magic Move steps. The pattern: data → schema → template. No FormGroup, no FormControl, no subscribe, no destroy.
-->

---

# Same login form, less wiring

````md magic-move {lines: true}
```typescript
// ❌ Reactive Forms — explicit form model + subscriptions
export class LoginComponent implements OnInit, OnDestroy {
  loginForm!: FormGroup;
  private destroy$ = new Subject<void>();

  constructor(private fb: FormBuilder) {}

  ngOnInit() {
    this.loginForm = this.fb.group({
      email:    ['', [Validators.required, Validators.email]],
      password: ['', [Validators.required, Validators.minLength(6)]]
    });

    this.loginForm.valueChanges
      .pipe(takeUntil(this.destroy$))
      .subscribe(value => this.onChange(value));
  }

  ngOnDestroy() {
    this.destroy$.next();
    this.destroy$.complete();
  }
}
```

```typescript
// ✅ Signal Forms — model-first form
export class LoginComponent {
  protected credentials = signal({ email: '', password: '' });

  protected loginForm = form(this.credentials, s => {
    required(s.email);    email(s.email);
    required(s.password); minLength(s.password, 6);
  });

  // credentials() stays as the source of truth.
  // No subscribe. No teardown. Less form wiring.
}
```
````

<!--
This is the side-by-side that lands hardest. Everyone has written the top version dozens of times.
-->

---

# Signal Forms: how v21 refined it

<div class="text-xs pt-2">

| Feature | v21.0 | v21.1 | v21.2 |
|---|:-:|:-:|:-:|
| `form()` + `required`/`email`/`minLength`/`max`/`pattern` | ✅ | | |
| `[field]` renamed to **`[formField]`** | | ✅ | |
| `provideSignalFormsConfig({ classes })` for auto CSS | | ✅ | |
| Custom controls via `FormValueControl` / `FormCheckboxControl` | | ✅ | |
| `focusBoundControl()` + `errorSummary()` (a11y) | | ✅ | |
| **`FormRoot` directive** + **`submission`** option | | | ✅ |
| **`transformedValue(parse, format)`** | | | ✅ |
| **`SignalFormControl`** compat for incremental migration | | | ✅ |
| Reactive `validateStandardSchema(() => zodSchema)` | | | ✅ |

</div>

<v-click>

<div class="pt-4 text-sm opacity-80">
By <code>v21.2</code>, the API had mostly settled. <code>v22</code> then graduates it to public API — <code>v22.0.0-next.11</code>, commit <code>7745365910</code>.
</div>

</v-click>

<!--
Three minor releases of polish. Worth showing because it answers "is this stable enough yet?" Yes — the API surface stopped moving by 21.2.
-->

---

# The numeric input problem

<div class="text-sm pt-2">

`<input type="number">` looks semantic, but it brings UX bugs: spinner controls, **mousewheel edits**, and inconsistent mobile keyboards. MDN recommends avoiding it.

</div>

<v-click>

````md magic-move {lines: true}
```html
<!-- Pre-v22 — type="number" was the practical option -->
<input type="number" [formField]="signupForm.age" />

<!-- 🐛 spinner UI nobody wants
   🐛 mousewheel silently changes value
   🐛 mobile keyboard inconsistent -->
```

```html
<!-- v22 — text UI + inputmode, numeric model ✅ -->
<input
  type="text"
  inputmode="numeric"
  [formField]="signupForm.age"
  (keydown)="onAgeKeydown($event)" />
```
````

</v-click>

<!--
This is a "huh, finally" moment. Every dev in the room has fought this.
-->

---

# The v22 fix: text UI, numeric model

```typescript
interface SignupFormData {
  username: string;
  email: string;
  age: number | null;        // 👈 stays number | null in v22
}

protected model = signal<SignupFormData>({ username: '', email: '', age: null });

protected signupForm = form(this.model, s => {
  required(s.age, { message: 'Age is required' });
  min(s.age, 18,  { message: 'You must be at least 18' });
  max(s.age, 120, { message: 'Please enter a valid age' });
});

protected onAgeKeydown(event: KeyboardEvent) {
  const allowed = ['Backspace','Delete','Tab','Escape','Enter','ArrowLeft','ArrowRight'];
  if (allowed.includes(event.key)) return;
  if (!/^\d$/.test(event.key)) event.preventDefault();
}
```

<div class="text-xs opacity-70 pt-2">
v22 keeps the UI as text, the model as <code>number | null</code>, and empty input as <code>null</code>.<br/>
Commit <code>41b1410c</code>. Source: Brian Treese, <em>briantree.se/angular-signal-forms-number-inputs</em>
</div>

<!--
The keydown handler is the MDN-recommended pattern. Browsers are inconsistent at enforcing numeric input even with inputmode.
-->

---
layout: section
---

# Part 4 — Templates & DX

The small stuff that adds up

---

# Templates got nicer in v21

<div class="text-sm pt-2">

| Feature | Version | Example |
|---|---|---|
| Control flow migration on `ng update` | 21.0 | `*ngIf` → `@if` |
| `RegExp` literals in templates | 21.0 | `{{ /\d+/.test(id()) }}` |
| `@defer` viewport `rootMargin` | 21.0 | IntersectionObserver options |
| **Multi-`@case` fall-through** | 21.1 | shared branches |
| **Spread syntax** in templates | 21.1 | `[...admins]`, `sum(...counters)` |
| **Arrow functions** in template expressions | 21.2 | `(click)="count.update(n => n + 1)"` |
| **`instanceof`** in templates | 21.2 | `{{ x instanceof Date }}` |
| **Exhaustive `@switch`** with `@default never` | 21.2 | TS2322 if a case is reachable |

</div>

---

# Template superpowers, in practice

```html
<!-- v21.1: multi-case fall-through + spread -->
@switch (status) {
  @case ('draft')
  @case ('pending') { <p>Your document is not yet published</p> }
  @case ('published') { <p>Your document is live</p> }
  @default { <p>Unknown status</p> }
}

<button (click)="sum(...counters)">Sum all counters</button>

<!-- v21.2: arrow functions + exhaustive switch -->
<button (click)="count.update(n => n + 1)">+1</button>

@switch (status()) {
  @case ('idle')    { <p>Idle</p> }
  @case ('loading') { <p>Loading</p> }
  @default never;   <!-- compile error if 'error' is reachable -->
}
```

<div class="text-xs opacity-70 pt-2">
Fewer throwaway component methods, more type-checked intent in the template.
</div>

<!--
The arrow function in (click) is the moment everyone goes "wait, we can do THAT now?" Saves a ton of trivial component methods.
-->

---

# Animations: let CSS do the work

```html
@if (isShown()) {
  <aside class="toast"
         animate.enter="toast-in"
         animate.leave="toast-out">
    Saved!
  </aside>
}
```

```css
.toast-in {
  opacity: 1;
  @starting-style { opacity: 0; }
}

.toast-out {
  opacity: 0;
}
```

<div class="grid grid-cols-2 gap-6 pt-4 text-sm">

<div>

- `animate.enter` / `animate.leave` are **Angular hooks** that add CSS classes at the right time
- Prefer **native CSS transitions or keyframes**
- `@starting-style` makes entry transitions work

</div>

<div>

- Angular handles lifecycle; CSS handles motion
- For route transitions, use `withViewTransitions()`
- Router view transitions are still **developer preview**

</div>

</div>

<!--
This slide is here because people still expect "Angular animations" to mean the old runtime DSL first. The current docs put CSS and enter/leave lifecycle hooks front and center.
-->

---
layout: two-cols-header
---

# Angular Aria: accessibility without prebuilt widgets

::left::

```html
<div ngToolbar aria-label="Editor actions">
  <button ngToolbarWidget value="undo">Undo</button>
  <button ngToolbarWidget value="redo">Redo</button>

  <div ngToolbarWidgetGroup role="radiogroup" aria-label="Alignment">
    <button ngToolbarWidget value="left">Left</button>
    <button ngToolbarWidget value="center">Center</button>
    <button ngToolbarWidget value="right">Right</button>
  </div>
</div>
```

::right::

- Install with `npm install @angular/aria`
- Headless directives for WAI-ARIA patterns like toolbar, tabs, menu, listbox, select, and tree
- Built-in keyboard navigation, focus management, and screen-reader semantics
- Best fit: custom design systems that want behavior without pre-styled widgets

<div class="text-xs opacity-70 pt-4">
Think “accessibility primitives for your design system”, not “another component library”.
</div>

<!--
Toolbar is the most legible example on one slide: it shows why Angular Aria exists without requiring a full design-system demo.
-->

---

# v22 makes OnPush the default

<div class="pt-2 text-sm">

The rule in v22 is simple: <code>ChangeDetectionStrategy.OnPush</code> is now the default.<br/>
<code>Eager</code> is the explicit name for the old always-check strategy, and <code>Default</code> becomes a deprecated alias.

</div>

<v-click>

```typescript
// v22 — the enum changes
enum ChangeDetectionStrategy {
  OnPush,
  Eager,           // 👈 new explicit name for old "Default"
  Default = Eager  // deprecated alias
}
```

</v-click>

<v-click>

<div class="pt-4 text-sm">

`ng update` to v22 keeps existing behavior by:

1. Add `changeDetection: ChangeDetectionStrategy.Eager` to components **without** an explicit setting
2. Rename `ChangeDetectionStrategy.Default` → `Eager` in your code
3. Leave existing `OnPush` components untouched

</div>

</v-click>

<v-click>

<div class="pt-4 p-3 border-l-4 border-red-500 bg-red-50/10 text-sm">

⚠️ **Library footgun:** if a library ships components without explicit `changeDetection`, they can flip to OnPush under v22. Audit your dependencies and upstream fixes.

</div>

</v-click>

---

# `injectAsync()`: lazy services without injector boilerplate

**Stable in v22.** Import on demand, still resolve through Angular DI.
Best fit: root-provided services you only need on specific interactions.

````md magic-move {lines: true}
```typescript
// Before — Injector.get() + manual promise caching
export class PostEditorComponent {
  private markdownService?: MarkdownService;
  private loading?: Promise<MarkdownService>;

  constructor(private injector: Injector) {}

  private async getMarkdown() {
    if (this.markdownService) return this.markdownService;
    if (!this.loading) {
      this.loading = import('../markdown.service').then(m => {
        return this.injector.get(m.MarkdownService);
      });
    }
    return this.markdownService = await this.loading;
  }
}
```

```typescript
// v22 — injectAsync()
import { Component, injectAsync, signal } from '@angular/core';

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
````

<!--
Real bundle-size wins. Markdown editors, syntax highlighters, charting libs, PDF generators are the canonical wins.
-->

---
layout: section
---

# Part 5 — Testing

The default changed; the trade-offs did not disappear

---

# Vitest is the default now

<div class="grid grid-cols-2 gap-6 pt-4 text-sm">

<div>

### What changed in v21

- `@angular/build:unit-test` builder is **stable**
- Vitest is the **default** runner
- `web-test-runner` + `jest` experimental builders → **deprecated in v21, removed in v22**
- `ng g @schematics/angular:refactor-jasmine-vitest`
- Karma still works: `ng new --test-runner=karma`

</div>

<div>

### What to watch for

- **`fakeAsync` / `flush` / `waitForAsync` don't carry over** — no Zone.js patches in Vitest. Use native `async` and fake timers.
- Migration leaves a TODO and generates a Markdown report
- Browser Mode works via `ng add @vitest/browser-playwright` / `webdriverio`

</div>

</div>

<!--
The fakeAsync break is the biggest practical migration blocker. Mention it explicitly.
-->

---

# Angular built-in vs `@analogjs/vitest-angular`

<div class="text-xs pt-2">

| Feature | AnalogJS | Angular built-in |
|---|:-:|:-:|
| Angular versions | v17+ | v21+ |
| Builders / Schematics / Migrations | ✅ | ✅ |
| Browser Mode (Playwright / WDIO) | ✅ | ✅ |
| Fully configurable | ✅ | ⚠️ merged via `runnerConfig` |
| **VS Code / JetBrains Vitest extension** | ✅ | ❌ |
| Vitest CLI (`npx vitest`) | ✅ | ❌ |
| Vitest Workspaces (Nx monorepos) | ✅ | ❌ |
| Module mocking (`vi.mock`) | ✅ | ❌ |
| Buildable libs in monorepos | ✅ | ❌ |

</div>

<v-click>

<div class="pt-4 text-sm">

**The key difference:** Angular's builder constructs Vitest config **in-memory** from `angular.json`. The VS Code Vitest extension can't see that config — so no gutter icons and no per-test runs.

</div>

</v-click>

---

# Which Vitest should you pick?

<div class="grid grid-cols-3 gap-4 pt-8">

<div class="p-4 bg-green-500/20 rounded">

### Greenfield + standard tests

→ **Angular built-in**

Zero config, batteries included, the team maintains it.

</div>

<div class="p-4 bg-purple-500/20 rounded">

### VS Code Test Explorer non-negotiable, heavy `vi.mock`, or Nx monorepo

→ **AnalogJS**

Real `vite.config.mts` + every Vitest feature.

</div>

<div class="p-4 bg-blue-500/20 rounded">

### Middle ground

→ **Built-in + `runnerConfig`**

`runnerConfig: "vitest-base.config.ts"`<br/>
v22 improved the merging behavior (PR #31729). Still no IDE gutters.

</div>

</div>

<!--
Don't be evangelistic. Both are fine. The choice is driven by VS Code Test Explorer and module mocking needs.
-->

---

# Testing migration: 20 → 22

<div class="pt-2 text-sm opacity-80">
  <strong>The story:</strong> first make fragile tests fail honestly, then switch the runner, then aim for tests that don't care whether Zone.js is around.
</div>

<div class="grid grid-cols-3 gap-4 pt-4 text-[0.94rem] leading-snug">

<div v-click class="rounded-xl border border-white/10 bg-amber-500/10 p-4">
  <div class="text-[0.68rem] font-semibold uppercase tracking-[0.18em] opacity-60">v20 mindset</div>
  <div class="mt-2 text-lg font-semibold">Make hidden failures loud</div>
  <ul class="mt-3 space-y-2">
    <li><code>flushEffects()</code> → <code>tick()</code> is <strong>not</strong> a mechanical swap</li>
    <li>For async signal/resource work, lean on <code>whenStable()</code></li>
  </ul>
</div>

<div v-click class="rounded-xl border border-white/10 bg-fuchsia-500/10 p-4">
  <div class="text-[0.68rem] font-semibold uppercase tracking-[0.18em] opacity-60">v21 move</div>
  <div class="mt-2 text-lg font-semibold">Switch the runner on purpose</div>
  <ul class="mt-3 space-y-2">
    <li>Karma/Jasmine → Angular's builder + schematic</li>
    <li>Jest → migrate gradually, then replace Zone helpers with native <code>async</code></li>
  </ul>
</div>

<div v-click class="rounded-xl border border-white/10 bg-sky-500/10 p-4">
  <div class="text-[0.68rem] font-semibold uppercase tracking-[0.18em] opacity-60">v22 target</div>
  <div class="mt-2 text-lg font-semibold">Write zoneless-ready tests</div>
  <ul class="mt-3 space-y-2">
    <li>Treat fake timers like global state — install early, always restore</li>
    <li>Browser Mode is optional and incremental</li>
  </ul>
</div>

</div>

<div v-click class="pt-4 text-sm">
  <strong>Docs for the path. Marmicode for the potholes.</strong><br/>
  <span class="text-xs opacity-70">Angular docs are the canonical migration route; Marmicode is great for the weird failures around <code>flushEffects()</code>, timers, Jest → Vitest, and Browser Mode.</span>
</div>

<!--
Deliver this as a three-beat story, not a checklist: v20 makes hidden failures loud, v21 switches the runner, v22 removes the dependency on Zone magic. Land the line: "docs for the path, Marmicode for the potholes."
-->

---
layout: section
---

# Part 6 — AI & Agentic UI

Where Angular is genuinely ahead

---

# Angular's AI story: three layers

<div class="pt-4">

<v-clicks>

<div class="flex items-start gap-4 p-4 bg-blue-500/10 rounded">
  <span class="text-2xl">📄</span>
  <div>
    <strong>Curated context for LLMs</strong><br/>
    <code>angular.dev/llms.txt</code>, <code>llms-full.txt</code>, copy-paste system prompts at <code>angular.dev/ai/develop-with-ai</code>
  </div>
</div>

<div class="flex items-start gap-4 p-4 bg-purple-500/10 rounded">
  <span class="text-2xl">🤖</span>
  <div>
    <strong>Agentic tooling via MCP</strong><br/>
    <code>ng mcp</code> server with tools for examples, modernization, dev server, tests, AI tutor, OnPush+zoneless migration
  </div>
</div>

<div class="flex items-start gap-4 p-4 bg-green-500/10 rounded">
  <span class="text-2xl">📊</span>
  <div>
    <strong>Measurable quality — Web Codegen Scorer</strong><br/>
    Closed loop: prompt → scorer → iterate until 97–100/100. This is why “follows current Angular best practices” can actually be measured.
  </div>
</div>

</v-clicks>

</div>

---

# `ng mcp`: the Angular MCP server

<div class="text-xs pt-2">

| Tool | First in | Stability | What it does |
|---|---|---|---|
| `list_projects` | v20.2 | Stable | Lists projects in workspace |
| `get_best_practices` | v20.2 | Stable | Latest best-practice prompt |
| `find_examples` | v20.2 | **Stable v21** | Code examples, version-scoped |
| `search_documentation` | v20.2 | Stable | Live search of angular.dev |
| `modernize` | v20.2 | Experimental | Suggests + runs modernizations |
| `ai_tutor` | **v21.0** | Experimental | Smart Recipe Box, Signal Forms lesson |
| `onpush_zoneless_migration` | **v21.0** | Experimental | Step-by-step migration plan |
| `build`, `devserver.*`, `test`, `e2e` | **v21.1** | Experimental | Agent can run dev/test/e2e |

</div>

<v-click>

<div class="pt-4 text-sm">
Configured per IDE: <code>.vscode/mcp.json</code>, <code>~/.cursor/mcp.json</code>, or JetBrains AI Assistant settings. New workspaces already get a <code>.vscode/mcp.json.template</code>.
</div>

</v-click>

<!--
"Install and connect the MCP server" is a non-negotiable team action item. Codegen quality difference is meaningful.
-->

---

# WebMCP: your app becomes a tool

<div class="grid grid-cols-2 gap-6 pt-4 text-sm">

<div class="p-4 bg-blue-500/10 rounded">

### Traditional MCP (`ng mcp`)

AI agent ↔ **server somewhere**

Your IDE asks the Angular CLI for examples, docs, migrations.

→ Makes Claude/Copilot **smarter at writing** Angular code.

</div>

<div class="p-4 bg-purple-500/20 rounded ring-2 ring-purple-400">

### WebMCP

AI agent ↔ **your live app in the browser**

User: *"add 'feed the cats' to my to-do list."*<br/>
Watches it happen without typing.

→ Makes your **running app** agent-callable.

</div>

</div>

<v-click>

<div class="pt-4 text-sm">
Same MCP family, opposite direction. Built on the emerging W3C <code>navigator.modelContext</code> API — framework-agnostic, with the browser as broker.
</div>

</v-click>

---

# WebMCP in Angular 22

```typescript
// app.config.ts — global tools, Experimental in v22
import {
  ApplicationConfig,
  inject,
  provideExperimentalWebMcpTools,
} from '@angular/core';

export const appConfig: ApplicationConfig = {
  providers: [
    provideExperimentalWebMcpTools([
      {
        name: 'addTodo',
        description: 'Add a new item to the user\'s to-do list',
        inputSchema: {
          type: 'object',
          properties: {
            title: { type: 'string', description: 'Short task title' },
            dueDate: { type: 'string', format: 'date-time' }
          },
          required: ['title'],
          additionalProperties: false
        } as const,
        execute: async ({ title, dueDate }) => {
          const todoService = inject(TodoService);
          await todoService.add({ title, dueDate });
          return `Added "${title}" to the to-do list.`;
        }
      }
    ])
  ]
};
```

<div class="text-xs opacity-70 pt-2">
Tools run inside Angular's injection context — <code>inject()</code>, signals, the whole DI tree are available. The callback returns content for the agent (typically a string), not an RPC-style success object.
</div>

---

# Signal Forms can auto-generate tools

```typescript
// app.config.ts
import { ApplicationConfig } from '@angular/core';
import { provideExperimentalWebMcpForms } from '@angular/forms/signals';

export const appConfig: ApplicationConfig = {
  providers: [provideExperimentalWebMcpForms()]
};

// checkout.component.ts
const checkoutForm = form(this.cartData, schema, {
  experimentalWebMcpTool: {
    name: 'completeCheckout',
    description: 'Fill and submit the checkout form on the user\'s behalf'
  }
});

// Angular derives JSON schema from the form model
// and registers the form as a WebMCP tool.
```

<v-click>

<div class="pt-4 p-3 border-l-4 border-amber-500 bg-amber-50/10 text-sm">

⚠️ **Security caveat:** Angular does NOT validate agent inputs against your declared schema. Validate explicitly in `execute` or in the form's own validators.

</div>

</v-click>

<v-click>

<div class="pt-4 text-xs opacity-70">
Status: <strong>experimental</strong> in v22. WebMCP spec itself is in flux. Treat as preview, not production.
</div>

</v-click>

---

# Related standards — not Angular v22 features

<div class="text-xs pt-2">

| Standard | Problem it solves | Direction | Angular |
|---|---|---|---|
| **WebMCP** | External agent calls your app's tools | Agent → app | `provideExperimentalWebMcpTools` (v22) ✅ |
| **AG-UI** | Frontend ↔ your own agent backend (runs, streaming, tool calls) | App ↔ your agent | Community SDK |
| **A2UI** | LLM generates UI at runtime, rendered via catalog | Agent → app UI | Community renderer |
| **MCP Apps** | Server-driven MCP application pattern | Server → agent | Workshop content |

</div>

<v-click>

<div class="pt-4 text-sm">
**WebMCP = your app is a tool. AG-UI = talk to your agent. A2UI = agent generates UI.**<br/>
Different layers, often combined. Framework-agnostic standards, not Angular core.
</div>

</v-click>

<v-click>

<div class="pt-4 text-xs opacity-70">
Deep dives: Manfred Steyer's "Agentic Angular" 6-part series at <em>angulararchitects.io/en/blog</em><br/>
Reference repo: <em>github.com/angular-architects/flights42</em>
</div>

</v-click>

---
layout: section
---

# Part 7 — What else is in v21 & v22

The honorable mentions

---

# v21: router, HTTP, and DX polish

<div class="grid grid-cols-2 gap-6 pt-4 text-sm">

<div>

### HTTP

- `provideHttpClient()` is still required
- `withFetch()` is the v21 opt-in for `FetchBackend`
- New `responseType`, `referrerPolicy` options

### Router

- Per-navigation scroll override
- Standalone `isActive()` signal (v21.1)
- `provideStabilityDebugging()` (v21.1)
- Trailing-slash strategies (v21.2)

</div>

<div>

### DX / platform

- `@angular/aria` lands for headless accessible widgets
- `ng serve --define VERSION="'1.0.0'"`
- `ng version --json` for automation
- **Built-in Prettier** (v21.2)
- `HttpErrorResponse` exposes Fetch `responseType`

</div>

</div>

---

# v22: stable resources, new defaults

<div class="grid grid-cols-2 gap-6 pt-4 text-sm">

<div>

### HTTP

- `FetchBackend` is the **default** — `withFetch()` is no longer needed
- `provideHttpClient()` still configures the client
- `reportUploadProgress` / `reportDownloadProgress`
- Old `reportProgress` boolean **deprecated**

### Signal Forms

- `min`/`max` no longer accept strings (number-only)
- Stable APIs + the compat bridge make incremental migration realistic

</div>

<div>

### Async data

- `resource()` / `rxResource()` / `httpResource()` are **stable**
- `httpResource()` is eager and read-oriented; keep plain `HttpClient` for mutations
- `ResourceSnapshot<T>` + `resourceFromSnapshots()` round out composition

### Selectorless components

- Watch this space — RFC still in flux
- Realistic v22: experimental at most

### Compiler

- Node.js 26 supported
- TypeScript 5.9 baseline

</div>

</div>

---

# Deprecations & breaking changes — v21

<v-clicks>

- **`HammerModule` REMOVED.** Apps relying on HammerJS must wire it manually or move to Pointer Events.
- **`*ngIf` / `*ngFor` / `*ngSwitch` deprecated** — auto-migration via `ng update`
- **`NgClass` / `NgStyle` deprecated** → `[class]` / `[style]` bindings
- **`CommonModule` discouraged** → `common-to-standalone` migration
- **Karma/Jasmine no longer default** (still supported)
- **`Router.isActive()` deprecated** (v21.1) → standalone signal
- **TypeScript 5.9 required**; Node 22.22+ / 24.13+ practically

</v-clicks>

---

# Deprecations & breaking changes — v22

<v-clicks>

- **OnPush as default** — auto-migration adds `Eager` to unmigrated components. Watch third-party libs.
- **`web-test-runner` + `jest` experimental builders REMOVED**
- **`HttpClient.reportProgress` deprecated** → split into upload/download
- **Signal Forms `min`/`max`** no longer accept strings
- **Node.js 26 supported**; final word on Node 20 awaits GA
- **Selectorless** still in flux — APIs may shift right up to GA

</v-clicks>

---
layout: center
class: text-center
---

# 🎯 Monday morning takeaways

What to do first when you're back at work

---

# What to do first

<v-clicks>

- ✅ **Start with `ng update`; finish with the named schematics** — Angular already ships first-party migrations for control flow, `CommonModule`, `NgClass` / `NgStyle`, router testing, and Jasmine → Vitest.
- ✅ **Put `ng mcp` in every IDE** — `.vscode/mcp.json` makes the AI stop guessing.
- ✅ **Dry-run v22 on one real feature** — `onpush_zoneless_migration` is the dress rehearsal.
- 🚫 **Stop importing `CommonModule` everywhere** — run `common-to-standalone`.
- 🚫 **Stop reaching for `NgClass` / `NgStyle`** — run `ngclass-to-class` / `ngstyle-to-style`, then stay on `[class]` / `[style]`.
- 🚫 **Stop writing new `*ngIf` / `*ngFor`** — run `control-flow`, then stay on `@if` / `@for` with `track`.
- ✅ **Default new forms to Signal Forms** — use `SignalFormControl` only as the bridge.
- ✅ **Default new read paths to `httpResource()` / `resource()`** — less `subscribe`, less teardown. Keep plain `HttpClient` for mutations.
- ✅ **Write `changeDetection: OnPush` explicitly today** — then v22 becomes boring.
- ✅ **Audit dependencies before v22** — any library without explicit `changeDetection` deserves an issue.

</v-clicks>

<!--
Call out the official migrations reference here: Angular ships named schematics for standalone, control-flow, inject(), signal inputs/outputs/queries, NgClass/NgStyle cleanup, router-testing-module migration, and common-to-standalone. The message is: don't heroically hand-edit code Angular can migrate for you.
-->

---

# Upgrade decision matrix

<div class="text-sm pt-4">

| Where you are | What to do |
|---|---|
| **v17 / v18** | Go **one major at a time** — `(17 →) 18 → 19 → 20 → 21`. Don't skip straight to v20. |
| **v19** | Upgrade **now** — EOL May 19, 2026. No more security patches. |
| **v20** | Direct upgrade to v21 — 2–8 hours for most apps. |
| **v21.0 / 21.1** | Bump to **21.2.14** (latest patches, Signal Forms compat bridge). |
| **v16 or earlier** | Treat as 4–12 week re-architecture. Lean on MCP `modernize` tool. |
| **Planning v22** | Start dry-runs now. Audit libs for explicit `changeDetection`, then pilot `injectAsync()` / resource adoption on one feature. |

</div>

---

# Go deeper

<div class="grid grid-cols-2 gap-4 pt-4 text-sm">

<div>

### Must-watch
- **Alex Rickabaugh — Signal Forms deep dive** (YouTube `hKkiivsyrHA`, Jan 2026)
- **OnPush-by-default RFC #66779** — Jan 2026
- **Steyer — "Agentic Angular" 6-part series** at angulararchitects.io

### Release recaps
- **Ninja Squad** — v21.0 / 21.1 / 21.2 deep dives at `blog.ninja-squad.com`
- **Brian Treese** — `briantree.se` for practical patterns
- **Marmicode Cookbook** — practical Angular testing migration recipes

</div>

<div>

### Official docs
- `angular.dev/tutorials/signals` — canonical learning order
- `angular.dev/guide/zoneless`
- `angular.dev/guide/animations`
- `angular.dev/guide/aria/overview`
- `angular.dev/reference/migrations`
- `angular.dev/ai/develop-with-ai` + `/ai/mcp`
- `next.angular.dev/api/core/provideExperimentalWebMcpTools`

### Migration tools
- `ng update @angular/core @angular/cli`
- `ng mcp --experimental-tool=all`
- MCP `onpush_zoneless_migration`

</div>

</div>

---
layout: end
class: text-center
---

# Thanks!

<div class="text-xl pt-6 opacity-80">
Questions?
</div>
