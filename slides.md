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

What's new — and what matters

<img src="/assets/Angular Wordmark Gradient.png" alt="Angular" class="mx-auto mt-6 h-18 w-auto" />

<!-- <div class="pt-12">
  <span class="px-2 py-1 rounded cursor-pointer bg-white/10 hover:bg-white/20" @click="$slidev.nav.next">
    Press space to start <carbon:arrow-right class="inline"/>
  </span>
</div> -->

<div class="abs-br m-6 text-sm opacity-50">
  v21.2.14 stable · v22.0.0 stable · June 2026
</div>

<!--
~35–45 minute talk if covered fully; easy to trim toward ~30 by skipping a few comparison slides.
Audience: medium-to-advanced Angular devs who want to get up to date.
Three big stories: zoneless defaults, Signal Forms graduating, WebMCP / agentic UI.
Optional skips if we're tight: linkedSignal PR, adjacent ecosystem, honorable mentions.
-->

---
layout: center
class: text-center
---

# Where we are now

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

### Angular 22 — Released (June 2026)

The **graduation** release:
- Signal Forms → **stable**
- **OnPush** as default change detection
- `resource()`, `rxResource()`, `httpResource()` → **stable**
- `injectAsync()` **Developer Preview**
- `@Service()` **public API**
- WebMCP for agentic UI
- FetchBackend default for `HttpClient`

Current major: **22.x**

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

# Bootstrap: before vs after

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

<div class="text-xs opacity-70 pt-2">
The migration shape is simple once you separate versions: in <code>v20</code> you add <code>provideZonelessChangeDetection()</code>, in <code>v21+</code> you usually remove overrides, and only legacy apps opt back into Zone.js explicitly.
</div>

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
Example adapted from Vasco Cavalheiro, <a href="https://angular-university.io/angular-linkedsignal"><em>angular-university.io/angular-linkedsignal</em></a>
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
Until this lands → community <code>delegatedSignal</code> remains the practical workaround.<br/>
After it lands → read-only stores + Signal Forms becomes a one-liner.
</div>

</v-click>

<!--
State this plainly: PR is OPEN and not merged. Keep the three explicit callouts so nobody hears this as shipped.
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

# Login form: before vs after

<div class="text-sm pt-2 opacity-80">
Use case: a common auth/profile form where the model itself should stay the source of truth.
</div>

<div class="text-xs pt-2 opacity-60">
Step 1 in the progression: <code>Reactive Forms</code> → <code>Signal Forms</code>
</div>

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

<div class="text-xs opacity-70 pt-2">
The real win is not just fewer lines — the signal model becomes the single source of truth, and the usual <code>valueChanges</code> subscription + teardown boilerplate disappears.
</div>

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

# Signal Forms: what changed at stabilization

<div class="text-xs pt-2 opacity-60">
Step 2 in the progression: <code>experimental Signal Forms</code> → <code>stable, production-shaped Signal Forms</code>
</div>

<div class="grid grid-cols-2 gap-6 pt-4 text-sm">

<div>

- `touched` is now an **input**, with `touch()` as the output for custom controls
- `markAsTouched()` now marks **descendants too**
- Dynamic behaviors and validators now consistently use a **`when`** option
- `getError()` gives typed, targeted error access

</div>

<div>

- `reloadValidation()` re-runs async validators
- `debounce(field, 'blur')` is now supported
- `validateAsync()` / `validateHttp()` get their own `debounce` option
- `minDate()` / `maxDate()` arrive, and legacy custom controls interop improves

</div>

</div>

<div class="text-xs opacity-70 pt-4">
These are the kinds of changes that make v22 feel less like “same API, now stable” and more like “same API, now genuinely production-shaped.”
</div>

<!--
This is the missing “why stabilization matters” slide. It turns the v22 Signal Forms story from a badge into practical migration guidance.
-->

---

# Async validation: before vs after

<div class="text-sm pt-2 opacity-80">
Use case: username or email availability checks without hand-rolling a <code>valueChanges</code> pipeline.
</div>

<div class="text-xs pt-2 opacity-60">
Step 3 in the progression: <code>manual RxJS validation glue</code> → <code>validateHttp()</code>
</div>

````md magic-move {lines: true}
```typescript
// Before — Reactive Forms + manual RxJS debounce pipeline
export class SignupComponent {
  private readonly destroyRef = inject(DestroyRef);
  private readonly fb = inject(FormBuilder);
  private readonly http = inject(HttpClient);

  protected readonly form = this.fb.group({
    email: ['', [Validators.required, Validators.email]]
  });

  protected readonly emailTaken = signal(false);

  constructor() {
    this.form.controls.email.valueChanges.pipe(
      debounceTime(400),
      distinctUntilChanged(),
      takeUntilDestroyed(this.destroyRef),
      switchMap(email => this.http.get<{ taken: boolean }>(
        `/api/users/check?email=${email}`
      ))
    ).subscribe(result => this.emailTaken.set(result.taken));
  }
}
```

```typescript
// v22 — Signal Forms + validateHttp() with built-in debounce
export class SignupComponent {
  protected readonly model = signal({ email: '' });

  protected readonly signupForm = form(this.model, form => {
    required(form.email);
    email(form.email);

    validateHttp(form.email, {
      request: email => `/api/users/check?email=${encodeURIComponent(email.value())}`,
      debounce: 400,
      errors: result => result.taken ? { emailTaken: true } : null,
    });
  });
}
```
````

<div class="text-xs opacity-70 pt-2">
The v22 story is not just “forms are stable” — it’s “common <code>valueChanges + debounceTime + switchMap</code> glue becomes framework-shaped.”
</div>

<!--
This slide makes the Signal Forms validation story concrete. The audience should immediately see how much custom RxJS ceremony disappears for common async validators.
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
Commit <code>41b1410c</code>. Source: Brian Treese, <a href="https://briantree.se/angular-signal-forms-number-inputs"><em>briantree.se/angular-signal-forms-number-inputs</em></a>
</div>

<!--
The keydown handler is the MDN-recommended pattern. Browsers are inconsistent at enforcing numeric input even with inputmode.
-->

---
layout: section
---

# Part 4 — Templates, UI & core APIs

The day-to-day code changes around the big stories

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
- **Production-ready in v22**
- Headless directives for WAI-ARIA patterns like toolbar, tabs, menu, listbox, select, and tree
- Signal Forms integration + test harnesses are part of the v22 story
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

**Developer Preview in v22.** Import on demand, still resolve through Angular DI.
Best fit: root-provided services you only need on specific interactions.

<div class="text-sm pt-2 opacity-80">
Use case: a heavy service like PDF export, markdown rendering, or charts that should load only after user intent.
</div>

<div class="text-xs pt-2 opacity-60">
Step 1 in the progression: <code>inject()</code> / <code>Injector.get()</code> boilerplate → <code>injectAsync()</code>
</div>

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

<div class="text-xs opacity-70 pt-2">
Use <code>injectAsync()</code> when the dependency is off the critical path but should still come from Angular DI. Keep plain <code>inject()</code> for services every render path needs immediately.
</div>

<!--
Real bundle-size wins. Markdown editors, syntax highlighters, charting libs, PDF generators are the canonical wins.
Mention that @Service() also appears in the v22 public API as a service-focused decorator, but keep @Injectable() as the documented default until the final v22 docs make migration guidance explicit.
-->

---

# Root singleton: before vs after

<div class="text-sm pt-2 opacity-80">
Use case: a root-provided service or store that mostly exposes read state to the rest of the app.
</div>

<div class="text-xs pt-2 opacity-60">
Step 2 → 3 in the progression: <code>@Injectable({ providedIn: 'root' })</code> → <code>@Service()</code> → <code>@Service() + httpResource()</code>
</div>

````md magic-move {lines: true}
```typescript
// Before — classic root singleton
@Injectable({ providedIn: 'root' })
export class UserStore {
  private readonly http = inject(HttpClient);

  load() {
    return this.http.get<User[]>('/api/users');
  }
}
```

```typescript
// v22 — same intent, more explicit name
@Service()
export class UserStore {
  private readonly http = inject(HttpClient);

  load() {
    return this.http.get<User[]>('/api/users');
  }
}
```

```typescript
// Even better — if this singleton is really a read store
@Service()
export class UserStore {
  readonly selectedUserId = signal(42);

  readonly user = httpResource<User>(() =>
    `/api/users/${this.selectedUserId()}`
  );
}
```
````

<div class="text-xs opacity-70 pt-2">
For the common “auto-provided root singleton” case, <code>@Service()</code> says the thing more clearly. If that singleton really behaves like a signal-driven read store, exposing <code>httpResource()</code> can be even nicer. If you need custom provider scope, imperative mutations, or other DI options, <code>@Injectable()</code> and plain <code>HttpClient</code> still stay in the toolbox.
</div>

<!--
Keep this pragmatic. The point is naming clarity for the common case, not religious conversion away from @Injectable. The third state is intentionally narrower: only use it when the singleton owns a signal-driven read model.
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

- **Better migration bridge in v22:** add `zone.js/plugins/vitest-patch` if you need `fakeAsync` / `flush` / `waitForAsync` to keep working temporarily.
- Long-term target is still native `async` + Vitest fake timers.
- Migration leaves a TODO and generates a Markdown report
- Browser Mode works via `ng add @vitest/browser-playwright` / `webdriverio`

</div>

</div>

<!--
The story changed in v22: fakeAsync is no longer a hard stop, it's a temporary bridge. Mention the patch, then recommend migrating away from it.
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
    <li>Use <code>zone.js/plugins/vitest-patch</code> as a bridge, then replace Zone helpers with native <code>async</code></li>
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
    <a href="https://angular.dev/llms.txt"><code>angular.dev/llms.txt</code></a>, <a href="https://angular.dev/llms-full.txt"><code>llms-full.txt</code></a>, copy-paste system prompts at <a href="https://angular.dev/ai/develop-with-ai"><code>angular.dev/ai/develop-with-ai</code></a>
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

<div class="pt-3 text-xs opacity-70">
Angular Agent Skills have been around since the v21-era AI push, but v22 is where the Angular team makes them feel much more official and prominently spotlights <code>angular-developer</code> and <code>angular-new-app</code> via <code>github.com/angular/skills</code>.
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
Status: <strong>experimental</strong> in v22. WebMCP spec itself is in flux, currently needs a visible browser context, and today typically means Chrome flags / origin-trial-era ergonomics. Treat as preview, not production.
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
Deep dives: Manfred Steyer's "Agentic Angular" 6-part series at <a href="https://www.angulararchitects.io/en/blog/"><em>angulararchitects.io/en/blog</em></a><br/>
Reference repo: <a href="https://github.com/angular-architects/flights42"><em>github.com/angular-architects/flights42</em></a>
</div>

</v-click>

---
layout: section
---

# Part 7 — Router, resources & upgrade reality

The defaults and migration details that hit real apps

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
- `getError()` / `reloadValidation()` / blur-debounce round out the stable story

</div>

<div>

### Async data

- `resource()` / `rxResource()` / `httpResource()` are **stable**
- `httpResource()` is eager and read-oriented; keep plain `HttpClient` for mutations
- `chain()` + SSR cache IDs make resource composition much more practical

### Selectorless components

- Still evolving
- Treat as experimental for now

### Compiler

- `strictTemplates` is now default
- TypeScript **6.0+**; Node 20 dropped, Node 26 supported

</div>

</div>

---

# Single read: before vs after

<div class="text-sm pt-2 opacity-80">
Use case: a details page where one signal (like <code>userId</code>) drives one backend read.
</div>

<div class="text-xs pt-2 opacity-60">
Step 1 in the progression: <code>HttpClient + RxJS</code> → <code>httpResource()</code>
</div>

````md magic-move {lines: true}
```typescript
// Before — manual loading state + subscription cleanup
export class UserPage {
  private readonly http = inject(HttpClient);
  protected readonly userId = signal(42);
  protected readonly user = signal<User | null>(null);
  protected readonly loading = signal(false);

  constructor() {
    effect((onCleanup) => {
      this.loading.set(true);

      const sub = this.http
        .get<User>(`/api/users/${this.userId()}`)
        .subscribe(user => {
          this.user.set(user);
          this.loading.set(false);
        });

      onCleanup(() => sub.unsubscribe());
    });
  }
}
```

```typescript
// v22 — declarative read path with httpResource()
export class UserPage {
  protected readonly userId = signal(42);

  protected readonly user = httpResource<User>(() =>
    `/api/users/${this.userId()}`
  );
}
// user.value(), user.isLoading(), user.error(), user.status()
```
````

<div class="text-xs opacity-70 pt-2">
Use <code>httpResource()</code> for reads that should track signals and own loading/error state. Keep plain <code>HttpClient</code> for mutations and imperative workflows.
</div>

<!--
First land the simple case: one signal drives one read. Once the audience buys this, the next slide can escalate to dependent reads and explain why chain() exists.
-->

---

# Dependent reads: before vs after

<div class="text-sm pt-2 opacity-80">
Use case: a page where the second read depends on data from the first one — for example, load a user, then load posts by that user's id.
</div>

<div class="text-xs pt-2 opacity-60">
Step 2 → 3 in the progression: <code>HttpClient + RxJS</code> → <code>resource()</code> → <code>chain()</code>
</div>

````md magic-move {lines: true}
```typescript
// Step 1 — classic HttpClient + RxJS chain
export class UserPostsPage {
  private readonly http = inject(HttpClient);
  protected readonly userId = signal(42);

  protected readonly vm = toSignal(
    toObservable(this.userId).pipe(
      switchMap(id =>
        this.http.get<User>(`/api/users/${id}`).pipe(
          switchMap(user =>
            this.http.get<Post[]>('/api/posts', {
              params: { authorId: user.id }
            }).pipe(
              map(posts => ({ user, posts }))
            )
          )
        )
      )
    ),
    { initialValue: undefined }
  );
}
```

```typescript
// Step 2 — resources help, but dependency handling is still manual
export class UserPostsPage {
  protected readonly userId = signal(42);

  protected readonly user = resource({
    params: () => ({ id: this.userId() }),
    loader: ({ params }) => this.userApi.get(params.id),
  });

  protected readonly posts = resource({
    params: () => {
      if (!this.user.hasValue()) return undefined;
      return { authorId: this.user.value().id };
    },
    loader: ({ params }) => this.postsApi.listByAuthor(params.authorId),
  });
}
// Works, but loading/error propagation is manual and easy to get wrong.
```

```typescript
// Step 3 — v22 chain() composes dependent resources explicitly
export class UserPostsPage {
  protected readonly userId = signal(42);

  protected readonly user = resource({
    params: () => ({ id: this.userId() }),
    loader: ({ params }) => this.userApi.get(params.id),
  });

  protected readonly posts = resource({
    params: ({ chain }) => {
      const user = chain(this.user);
      return { authorId: user.id };
    },
    loader: ({ params }) => this.postsApi.listByAuthor(params.authorId),
  });
}
// If user is idle/loading/error, posts mirrors that dependency state automatically.
```
````

<div class="text-xs opacity-70 pt-2">
This is the right use case for <code>chain()</code>: the second read depends on a resolved field from the first read. The progression is the point: <code>RxJS chain</code> → <code>manual resource composition</code> → <code>dependency-aware resource composition</code>.
</div>

<!--
Tell this as a three-beat story. First: "yes, we all wrote the RxJS version". Second: "resources already help". Third: "chain() is the missing piece for dependent reads".
Mention the upstream behavior explicitly: idle→idle, loading→loading, error→dependency error, resolved→normal load.
-->

---

# Router inputs: before vs after

<div class="text-sm pt-2 opacity-80">
Use case: a routed page that wants path params and query params as normal component inputs instead of manual <code>ActivatedRoute</code> plumbing.
</div>

<div class="text-xs pt-2 opacity-60">
Useful enough to show — many Angular devs know path params, fewer remember that <code>withComponentInputBinding()</code> can bind query params, route data, and resolvers too.
</div>

````md magic-move {lines: true}
```typescript
// Before — read route state manually
export class UserPage {
  private readonly route = inject(ActivatedRoute);

  protected readonly id = toSignal(
    this.route.paramMap.pipe(map(params => params.get('id') ?? '')),
    { initialValue: '' }
  );

  protected readonly tab = toSignal(
    this.route.queryParamMap.pipe(map(params => params.get('tab') ?? 'overview')),
    { initialValue: 'overview' }
  );
}
```

```typescript
// After — let the router bind directly to inputs
export const appConfig: ApplicationConfig = {
  providers: [provideRouter(routes, withComponentInputBinding())]
};

export class UserPage {
  readonly id = input.required<string>();
  readonly tab = input('overview');
}
// Path params, query params, route data, and resolvers can all bind.
```

```typescript
// v22 — tune the binding behavior if needed
export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(routes, withComponentInputBinding({
      queryParams: false,
      unmatchedInputBehavior: 'undefinedIfStale'
    }))
  ]
};
```
````

<div class="text-xs opacity-70 pt-2">
This is not just syntactic sugar: it moves route state into the same <code>input()</code>-first mental model as the rest of modern Angular. v22 makes it more practical by adding options like <code>queryParams</code> and <code>unmatchedInputBehavior</code>.
</div>

<!--
Worth showing. Plenty of Angular devs know route params can become inputs, but the broader binding sources and new v22 options are not yet universal knowledge.
-->

---

# Nested route params: before vs after

<div class="text-sm pt-2 opacity-80">
Use case: deeply nested routes where parent params should be reachable without <code>route.parent?.parent?</code> archaeology.
</div>

<div class="text-xs pt-2 opacity-60">
Another router progression: <code>manual parent traversal</code> → <code>inherited params by default</code> → <code>explicit opt-out</code>
</div>

````md magic-move {lines: true}
```typescript
// Before — default was 'emptyOnly'
export class IssueDetailsPage {
  private readonly route = inject(ActivatedRoute);

  protected readonly teamId =
    this.route.parent?.parent?.snapshot.paramMap.get('teamId');
}
```

```typescript
// v22 — default is paramsInheritanceStrategy: 'always'
export class IssueDetailsPage {
  private readonly route = inject(ActivatedRoute);

  protected readonly teamId =
    this.route.snapshot.paramMap.get('teamId');
}
```

```typescript
// If you relied on the old behavior, restore it explicitly
export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(
      routes,
      withRouterConfig({ paramsInheritanceStrategy: 'emptyOnly' })
    )
  ]
};
```
````

<div class="text-xs opacity-70 pt-2">
This is a real quality-of-life breaking change: less <code>route.parent?.parent?</code> plumbing for nested routes. If you depended on the old behavior, opt out in app config with <code>withRouterConfig({ paramsInheritanceStrategy: 'emptyOnly' })</code>.
</div>

<!--
This lands the router default change with an example people have actually written, plus the actual config escape hatch. Emphasize that it's great, but still worth auditing.
-->

---

# Small v22 wins worth caring about

<div class="grid grid-cols-2 gap-4 pt-2 text-[0.86rem] leading-snug">

<div v-click class="rounded-xl border border-white/10 bg-sky-500/10 p-3">
  <div class="font-semibold">Router signals</div>
  <div class="mt-1"><code>router.currentNavigation()</code> is a signal — loading bars without subscribing to router events.</div>
</div>

<div v-click class="rounded-xl border border-white/10 bg-purple-500/10 p-3">
  <div class="font-semibold">Signal Forms polish</div>
  <div class="mt-1"><code>getError()</code>, <code>reloadValidation()</code>, and <code>debounce(..., 'blur')</code> replace a lot of custom glue.</div>
</div>

<div v-click class="rounded-xl border border-white/10 bg-emerald-500/10 p-3">
  <div class="font-semibold">Resources compose</div>
  <div class="mt-1"><code>chain()</code> and SSR cache IDs make dependent resources much less awkward.</div>
</div>

<div v-click class="rounded-xl border border-white/10 bg-orange-500/10 p-3">
  <div class="font-semibold">DI gets more focused</div>
  <div class="mt-1"><code>injectAsync()</code> lazy-loads expensive services; <code>@Service()</code> appears as public API, but <code>@Injectable()</code> remains the safe documented default.</div>
</div>

<div v-click class="rounded-xl border border-white/10 bg-rose-500/10 p-3">
  <div class="font-semibold">HTTP progress is explicit</div>
  <div class="mt-1"><code>reportUploadProgress</code> / <code>reportDownloadProgress</code> replace the vague old <code>reportProgress</code> boolean.</div>
</div>

<div v-click class="rounded-xl border border-white/10 bg-cyan-500/10 p-3">
  <div class="font-semibold">Route-scoped agent tools</div>
  <div class="mt-1">WebMCP tools on routes need injector auto-cleanup, otherwise agents can see tools from pages the user already left.</div>
</div>

</div>

<!--
This is the practical "small stuff" slide: a grab bag of features that are too small for their own section but useful in real apps. Keep the tone pragmatic: adopt these opportunistically, not via a big-bang migration.
-->

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
- **Router params now inherit by default** — `paramsInheritanceStrategy: 'always'`
- **`HttpClient.reportProgress` deprecated** → split into upload/download
- **Signal Forms `min`/`max`** no longer accept strings
- **TypeScript 6.0+**; Node 20 dropped, Node 26 supported
- **Webpack-era builders deprecated** as Angular leans into the application builder path
- **Selectorless** still in flux — APIs may continue to evolve across minors

</v-clicks>

---

# TypeScript 6: the tsconfig bits that bite

<div class="grid grid-cols-2 gap-6 pt-4 text-sm">

<div>

### Defaults that changed

- <code>types</code> now defaults to <code>[]</code>
  <br/>→ add <code>["node"]</code>, test-runner globals, etc. when names suddenly vanish
- <code>rootDir</code> now defaults to <code>.</code>
  <br/>→ set <code>"rootDir": "./src"</code> if output starts landing in <code>dist/src/...</code>
- <code>strict</code> now defaults to <code>true</code>
  <br/>→ usually already true in Angular CLI apps, less boring in custom workspaces

</div>

<div>

### Deprecations to clean up

- <code>baseUrl</code> deprecated → move prefixes directly into <code>paths</code>
- <code>moduleResolution: "node"</code> deprecated
  <br/>→ prefer <code>bundler</code> for web apps, <code>nodenext</code> for Node
- <code>ignoreDeprecations: "6.0"</code> is a bridge, not a strategy
- <code>ts5to6</code> codemod can rewrite <code>baseUrl</code> / <code>rootDir</code>

</div>

</div>

<div class="text-xs opacity-70 pt-4">
Angular 22 only says “TypeScript 6.0+ required”, but the real upgrade pain usually shows up in custom <code>tsconfig</code> files, not in component code.
</div>

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
- ✅ **Roll out v22 on one real feature first** — `onpush_zoneless_migration` is still the safest dress rehearsal.
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
| **v19** | Upgrade **immediately** — v19 is EOL and no longer receives security patches. |
| **v20** | Direct upgrade to v21 — 2–8 hours for most apps. |
| **v21.0 / 21.1** | Bump to **21.2.14** (latest patches, Signal Forms compat bridge). |
| **v16 or earlier** | Treat as 4–12 week re-architecture. Lean on MCP `modernize` tool. |
| **Adopting v22** | Start with one production slice. Audit libs for explicit `changeDetection`, then pilot `injectAsync()` / resource adoption on that slice. |

</div>

---

# Go deeper

<div class="grid grid-cols-2 gap-4 pt-4 text-sm">

<div>

### Must-watch
- **Alex Rickabaugh — Signal Forms deep dive** (YouTube `hKkiivsyrHA`, Jan 2026)
- **OnPush-by-default RFC #66779** — Jan 2026
- **Steyer — "Agentic Angular" 6-part series** at [angulararchitects.io](https://www.angulararchitects.io/en/blog/)

### Release recaps
- **Angular team** — [blog.angular.dev/announcing-angular-v22-c52bb83a4664](https://blog.angular.dev/announcing-angular-v22-c52bb83a4664)
- **Ninja Squad** — [blog.ninja-squad.com/2026/06/03/what-is-new-angular-22.0](https://blog.ninja-squad.com/2026/06/03/what-is-new-angular-22.0)
- **Angular.love** — [angular.love/angular-22-key-features-and-changes](https://angular.love/angular-22-key-features-and-changes)
- **Brian Treese** — [briantree.se](https://briantree.se/) for practical patterns
- **Marmicode Cookbook** — practical Angular testing migration recipes

</div>

<div>

### Official docs
- [angular.dev/tutorials/signals](https://angular.dev/tutorials/signals) — canonical learning order
- [angular.dev/guide/zoneless](https://angular.dev/guide/zoneless)
- [angular.dev/guide/animations](https://angular.dev/guide/animations)
- [angular.dev/guide/aria/overview](https://angular.dev/guide/aria/overview)
- [angular.dev/reference/migrations](https://angular.dev/reference/migrations)
- [angular.dev/ai/develop-with-ai](https://angular.dev/ai/develop-with-ai) + [`/ai/mcp`](https://angular.dev/ai/mcp)
- [next.angular.dev/api/core/provideExperimentalWebMcpTools](https://next.angular.dev/api/core/provideExperimentalWebMcpTools)
- [typescriptlang.org/docs/handbook/release-notes/typescript-6-0.html](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-6-0.html)
- [devblogs.microsoft.com/typescript/announcing-typescript-6-0](https://devblogs.microsoft.com/typescript/announcing-typescript-6-0)

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
