# React Mount Infrastructure (Phase 2 — Footer pilot)

Date: 2026-05-27
Status: Draft
Scope: zeppelin-web-angular

## Summary

Promote the ad-hoc React micro-frontend integration used by `PublishedParagraph`
into a small reusable Angular abstraction, then migrate the notebook paragraph
footer as the first consumer. Future per-component migrations (Phase 2 —
notebook and interpreter modules) reuse this infrastructure instead of
copying ~70 lines of script-loading and mount/unmount logic per component.

## Goals

- Eliminate the per-component boilerplate currently in
  `src/app/pages/workspace/published/paragraph/paragraph.component.ts:54-274`.
- Make the React module contract support live props updates without remounting
  React roots.
- Migrate `NotebookParagraphFooterComponent` to React behind a
  `?reactFooter=true` query-param gate, with rollback to the existing Angular
  footer when the flag is absent.
- Make the next per-component migration cost one new React module + a directive
  binding, not new infra.

## Non-goals

- Refactoring `PublishedParagraph` to use the new directive. That happens in a
  follow-up PR; this design is forward-compatible with that refactor.
- Migrating any other notebook component beyond the footer.
- Changing the Module Federation runtime, build, or shared-deps configuration.
- Visual or UX changes to the footer; the React version must render the same
  text and styling.
- Bundle-splitting the React remote or pre-loading `remoteEntry.js` outside the
  pages that already gate on a flag.

## Background

### Current pilot

The `PublishedParagraph` pilot ships React via Webpack Module Federation:

- React app exposes `./PublishedParagraph` from
  `projects/zeppelin-react/webpack.config.js:71-73`.
- The exposed module exports `mount(element, props): () => void` —
  `projects/zeppelin-react/src/pages/PublishedParagraph.tsx:49-67`.
- Angular host loads `remoteEntry.js` once and stores `unmountReact` in a field
  on the component:
  `src/app/pages/workspace/published/paragraph/paragraph.component.ts:54-274`.

The pilot is one-shot: PublishedParagraph is read-only, so mount happens once
per route visit and unmount on `ngOnDestroy`. Props never update after mount.

### Why the pilot pattern doesn't generalize

- `reactScriptLoaded` is an **instance field** on the Angular component, not a
  page-wide singleton. Copy this into the footer and a notebook with 50
  paragraphs will inject `<script src="…/remoteEntry.js">` 50 times.
- `mount()` returns only an unmount fn. Notebook footer inputs change at
  runtime (`dateStarted`, `dateFinished`, `status`). With the current contract,
  the only way to reflect updates is unmount + mount, which churns React roots
  on every paragraph state change.
- Each integration today reads `environment.reactRemoteEntryUrl`, creates a
  `<script>`, waits for `window.reactApp`, calls `container.get(…)`, calls the
  factory, calls `mount(...)`, stashes the unmount. Repeating this in 30+
  components is a code-review and maintenance loss.

### Footer specifics

`NotebookParagraphFooterComponent`
(`src/app/pages/workspace/notebook/paragraph/footer/footer.component.ts`)
takes six `@Input`s and computes two derived strings in `ngOnChanges` via
date-fns. It is rendered once per paragraph at
`src/app/pages/workspace/notebook/paragraph/paragraph.component.html:113-120`.
There are no outputs, no template projection, no DOM measurement. The Angular
template wraps the result in `<div class="footer">` styled by
`footer.component.less`.

## Design

### Module contract change (React side)

Today:

```ts
export const mount = (element, props) => {
  const root = createRoot(element);
  root.render(<Component {...props} />);
  return () => root.unmount();
};
```

New contract for all exposed components:

```ts
export type ReactHostCallbacks = {
  onError?: (error: unknown) => void;
};

export type ReactMountHandle<P> = {
  update: (props: P & ReactHostCallbacks) => void;
  unmount: () => void;
};

export function mount<P>(
  element: HTMLElement,
  props: P & ReactHostCallbacks
): ReactMountHandle<P>;
```

`mount` is **synchronous**. The async work (script load, `container.get`)
happens on the Angular side; by the time `mount` is called, the module is
in memory. If the React component throws during render, the exposed module
wraps its tree in an error boundary that calls `props.onError(err)`. The
Angular directive treats `onError` as a signal to log + optionally render
fallback content.

`update` calls `root.render(<Component {...newProps} />)` on the same root.
React preserves component state and reconciles. `unmount` calls
`root.unmount()`.

This is the standard React 18 root API. The pilot `PublishedParagraph` will
migrate to this contract; until then, the Angular wrapper treats the missing
`update` defensively (see "Compatibility" below).

### Angular side: two pieces

**`ReactRemoteLoaderService`** (Angular `providedIn: 'root'` service):

- Fields:
  - `private containerPromise: Promise<RemoteContainer> | null = null`
  - `private modulePromises = new Map<string, Promise<ExposedModule>>()`
- `loadContainer(): Promise<RemoteContainer>` — appends the
  `environment.reactRemoteEntryUrl` `<script>` once, caches the resulting
  `window.reactApp` container promise. Subsequent calls return the same
  promise. On `script.onerror` or rejection, clears `containerPromise` AND
  empties `modulePromises` (a stale failed container would otherwise poison
  every module). Next call re-injects the script.
- `loadModule<T>(exposedKey: string): Promise<T>` — calls `loadContainer()`
  then `container.get(exposedKey)()`. Caches each module promise by key.
  If the promise rejects, the entry is **evicted** from `modulePromises`
  so the next caller can retry. A failed-and-cached module promise is
  worse than no cache.

**`ReactMountDirective`** (selector `[zeppelinReactMount]`):

Internal state machine:

```ts
private latestProps: Record<string, unknown> = {};
private destroyed = false;
private loading = false;
private handle: ReactMountHandle | null = null;
```

- `@Input('zeppelinReactMount') module!: string` — e.g.
  `'./ParagraphFooter'`. The `module` input is **read once** on first
  change; subsequent changes to it are not supported (an error is logged
  via `onError`, see contract below). Dynamic module swap is YAGNI for
  this scope and the failure mode is sneaky.
- `@Input() reactProps: Record<string, unknown> = {}`.

Behavior:

- `ngOnChanges`: update `latestProps = props`. If `handle` exists, call
  `ngZone.runOutsideAngular(() => handle.update(latestProps))`. Otherwise,
  if not `loading` and not `destroyed`, kick off the load (see below).
  The directive **never** captures `reactProps` by closure; only
  `latestProps` is read after each await.
- Load flow (use `try/catch/finally`; a thrown error must not wedge
  `loading` in the `true` state):
  1. `loading = true`
  2. `try { await loader.loadModule(module); }`
     - On rejection: surface via `latestProps.onError`. Skip steps 3–4.
  3. **Re-check `destroyed`** after the await. If true, abort without
     calling `mount()`.
  4. Inside `ngZone.runOutsideAngular`, wrap `mount(host, latestProps)`
     in its own `try/catch`. On success: store `handle`. On throw:
     surface via `onError`; leave `handle` null.
  5. `finally { loading = false; }`
- `ngOnDestroy`: set `destroyed = true`. If `handle` exists, call
  `handle.unmount()` exactly once. If still loading, the destroyed flag
  short-circuits the post-await mount.

The directive owns the host `HTMLElement` via `ElementRef`.

Consumer usage (in `notebook/paragraph/paragraph.component.html` near the
existing footer at line 113):

```html
<div
  *ngIf="shouldUseReactFooter; else angularFooter"
  data-testid="react-paragraph-footer"
  zeppelinReactMount="./ParagraphFooter"
  [reactProps]="{
    dateStarted: paragraph.dateStarted,
    dateFinished: paragraph.dateFinished,
    dateUpdated: paragraph.dateUpdated,
    showExecutionTime: !paragraph.config.tableHide && !viewOnly,
    showElapsedTime: paragraph.status === 'RUNNING',
    user: paragraph.user,
    onError: onReactFooterError
  }"
></div>
<ng-template #angularFooter>
  <zeppelin-notebook-paragraph-footer
    data-testid="angular-paragraph-footer"
    …existing bindings…
  ></zeppelin-notebook-paragraph-footer>
</ng-template>
```

Both branches get `data-testid` so E2E can deterministically pick one.

**Fallback flag must be child-local, not the `@Input`.** `useReactFooter`
is now an `@Input` from `NotebookComponent`, so the parent's next change
detection pass would overwrite a local mutation. Use a separate
`reactFooterFailed = false` field and a getter:

```ts
@Input() useReactFooter = false;
reactFooterFailed = false;

get shouldUseReactFooter(): boolean {
  return this.useReactFooter && !this.reactFooterFailed;
}

readonly onReactFooterError = (error: unknown): void => {
  console.error('React footer error', error);
  this.reactFooterFailed = true;
  this.cdr.markForCheck();
};
```

The template gates on `shouldUseReactFooter`. The error handler is an
**arrow property**, not a method, so the function identity passed via
`reactProps.onError` keeps its `this` binding even when Angular re-evaluates
the template expression.

**Object-literal churn.** `[reactProps]="{ ... }"` constructs a new object
each change-detection pass, so `ngOnChanges` fires every cycle even when
no semantic property changed. For the footer this is acceptable — props
are simple scalars and `handle.update → root.render` is a cheap reconcile.
If profile measurement later shows churn matters, two mitigations are
available without changing the directive contract:

1. The consuming component exposes a memoized `reactFooterProps` getter
   that only mutates when relevant inputs change.
2. The directive shallow-compares props (excluding callback identity)
   before calling `handle.update`.

Neither is added in this pilot.

### Compatibility with the current pilot

The directive cannot assume `update` exists on the returned handle until all
exposed modules adopt the new contract. The directive treats two shapes:

- `{ update, unmount }`: full contract. Use both.
- `() => void` (legacy): wrap as `{ update: () => {}, unmount: legacyFn }`.
  This is a no-op update; safe for the existing read-only pilot.

`PublishedParagraph` keeps shipping unchanged until its follow-up refactor.

### Activation gate

Per-component query-param: `?reactFooter=true`.

**Read location:** `NotebookComponent.ngOnInit` already subscribes to
`ActivatedRoute.queryParamMap` with `takeUntil(destroy$)` at
`src/app/pages/workspace/notebook/notebook.component.ts:423-433`. Extend that
existing subscription to also read `reactFooter` and store
`useReactFooter: boolean` on the component. Pass it down to each
`<zeppelin-notebook-paragraph>` via a new `@Input() useReactFooter`. The
gate is **not** read in `NotebookParagraphComponent` directly — a notebook
with 50 paragraphs subscribing to the route 50 times is needless, and
`NotebookParagraphComponent`'s existing subscriptions are not properly
cleaned up (its `destroy$` is declared but not completed in `ngOnDestroy`),
so adding more there is a bad trade.

Use `queryParamMap.get('reactFooter') === 'true'`. Use `queryParamMap`, not
`queryParams`, because the rest of the notebook code already does (e.g.
the existing `paragraph` param at notebook.component.ts:428). All existing
query params (`paragraph`, `revisionId`, etc.) must continue to work
alongside `reactFooter`; manual testing must include
`?paragraph=<id>&reactFooter=true`.

This intentionally does **not** reuse `?react=true` (which currently means
"published paragraph in React" and is read only in
`PublishedParagraphComponent`). A future "enable all" gate can compose
later; this PR keeps the blast radius unambiguous.

### React component

`projects/zeppelin-react/src/components/paragraph/ParagraphFooter.tsx`:

- Pure presentational. Same prop shape as the Angular version's `@Input`s,
  same date-fns output strings.
- Styles live in a co-located `ParagraphFooter.css` next to the component,
  ported from `footer.component.less`. The React tree cannot inherit
  Angular's view-encapsulated CSS, so duplicating the rules is the simplest
  path. The original color comes from `@text-color-secondary` via the
  `themeMixin` in `footer.component.less:13`. The pilot ports this as a
  static hex value and accepts that **theme switches at runtime won't
  reflect in the React footer** — acknowledged limitation for the pilot,
  to be reassessed when more than one component has been migrated.
  Loaded via `style-loader` (already wired in `webpack.config.js:62-65`).
- No AntD. Uses date-fns directly.
- **Dependency:** `date-fns` must be added to
  `projects/zeppelin-react/package.json`. It is currently NOT a React-app
  dep (it's only in the host Angular app's `package.json:49`). Match the
  host's version for **behavior parity** (same format strings, same
  locale behavior). The Module Federation `shared` config currently only
  shares `react`/`react-dom`; the React remote will bundle its own
  `date-fns` regardless of the host. Adding `date-fns` to `shared` to
  deduplicate is out of scope; revisit only if bundle size becomes a
  pain point.
- Wraps its render tree in an error boundary that calls `props.onError`
  with the caught error. The boundary renders nothing on error (Angular
  will fall back). The boundary only catches **render and lifecycle**
  errors inside its subtree — module-evaluation errors, async/event-handler
  errors, and loader rejections still need the directive's own try/catch
  paths described above.
- Exports `mount(element, props): ReactMountHandle`.

Expose in `projects/zeppelin-react/webpack.config.js`:

```js
exposes: {
  './PublishedParagraph': './src/pages/PublishedParagraph',
  './ParagraphFooter': './src/components/paragraph/ParagraphFooter'
}
```

Re-export from `projects/zeppelin-react/src/main.ts`.

The `remoteEntry.json` generation in `webpack.config.js:97-117` is
**informational only** — no Angular code reads it (verified via repo
grep). Add the new exposed module to that listing for consistency, but
not as a blocker.

## Implementation plan

### Slice 1 — infrastructure only

- Add `ReactRemoteLoaderService` and `ReactMountDirective` under
  `src/app/share/react-mount/` (new folder; `share` is already an Angular
  module per `src/app/share/share.module.ts`).
- Declare and export `ReactMountDirective` from `share.module.ts`. The
  service is `providedIn: 'root'` and needs no module wiring. Notebook
  already imports the share module (`notebook.module.ts`), so the directive
  is reachable from notebook templates without further changes. The
  interpreter module gains access the same way when it joins Phase 2.
- Unit tests for the service (script-once behavior, error path) and the
  directive (mount/update/unmount lifecycle, props change).

### Slice 2 — React footer

- Add `ParagraphFooter.tsx` in the React project.
- Expose in webpack config; re-export from `main.ts`.
- Confirm visual parity in dev (`?reactFooter=true`) against current Angular
  footer.

### Slice 3 — host integration

- In `notebook.component.ts`, extend the existing `queryParamMap`
  subscription to also read `reactFooter` and store `useReactFooter` on
  the component.
- Pass `useReactFooter` into each `<zeppelin-notebook-paragraph>` as an
  `@Input`.
- In `paragraph.component.html`, gate the footer markup as shown above
  (using `shouldUseReactFooter`, not the raw `@Input`).
- Add the arrow-property `onReactFooterError` to `paragraph.component.ts`
  that logs the error, sets `reactFooterFailed = true`, and calls
  `cdr.markForCheck()`. The `shouldUseReactFooter` getter then returns
  `false` and the Angular fallback renders.
- E2E smoke: `playwright` step that loads a notebook with
  `?paragraph=<id>&reactFooter=true` and asserts the React footer
  (`[data-testid="react-paragraph-footer"]`) renders. Verify the
  `paragraph` param still selects the correct paragraph.

### Slice 4 — docs

- Rewrite `projects/zeppelin-react/README.md` "Adding a new React module"
  section to teach the directive + handle pattern, not the legacy one-shot
  mount.
- Add a short "React mount infrastructure" section explaining the directive,
  the service, and the contract.

Slices 1–4 land as a single PR. Slicing here is for review structure, not
sequencing.

## Testing

### Service (`ReactRemoteLoaderService`)

- Only one `<script>` element is appended even under concurrent
  `loadContainer` calls.
- `script.onerror` rejects, clears `containerPromise` AND empties
  `modulePromises`. A subsequent `loadContainer` issues a fresh
  `<script>`.
- A rejected `loadModule(key)` evicts that key from the module cache; the
  next caller retries.
- Successful `loadModule` returns a cached module promise on subsequent
  calls without calling `container.get` again.

### Directive (`ReactMountDirective`, via TestBed)

- Happy path: mount called once with initial props; one `update` per
  `reactProps` change; `unmount` on destroy. **`mount` is never called
  twice**, no matter how many prop changes occur.
- Props change while module is loading: only the **latest** props win;
  earlier prop values are not passed to `mount`.
- Destroy while module is loading: `mount` is **never** called after
  destroy; no `update`/`unmount` is invoked either.
- Loader rejects (`loadModule` throws): no mount; `onError` is invoked
  on `reactProps`. Module cache is empty, so a subsequent retry attempts
  load again.
- `mount` throws synchronously: directive does not leave a half-mounted
  state; `handle` stays null; `onError` invoked.
- `update` throws: error reported via `onError`; the next `ngOnDestroy`
  still calls `unmount` exactly once.
- Legacy module shape (returns a bare `() => void`) is tolerated: a
  no-op `update` wrapper, real `unmount`.
- Module input change after first mount: explicitly unsupported. The
  directive logs an error via `onError`; behavior is undefined beyond
  "doesn't crash".

### React component (`ParagraphFooter`)

- Renders `executionTime` text for the `FINISHED` case.
- Renders `elapsedTime` text for `RUNNING`.
- Renders `outdated` suffix when `dateUpdated > dateStarted`.
- Renders nothing visible when both `showExecutionTime` and
  `showElapsedTime` are false.
- Error boundary calls `onError` and renders nothing when the inner
  component throws.

### E2E (Playwright)

- `?reactFooter=true`: `[data-testid="react-paragraph-footer"]` is
  attached; `[data-testid="angular-paragraph-footer"]` is not.
- No `reactFooter` param: Angular footer is attached; React footer is
  not.
- `?paragraph=<id>&reactFooter=true`: paragraph selection still works
  (existing notebook `paragraph` param behavior is preserved).

## Risks / open questions

- **Style scoping.** Captured in the React component section above: pilot
  ships static hex colors, theme runtime-switching is **not supported** in
  the React footer for this pilot, to be reassessed when more components
  are migrated.
- **Elapsed time refresh.** The existing Angular footer computes
  `elapsedTime` once per `ngOnChanges`, so the displayed value only updates
  when parent inputs change. The React version should match this behavior
  (no setInterval). Improving it is out of scope.
- **Server-side dev proxy.** `proxy.conf.js` and `environment.ts` already
  route `remoteEntry.js` to `http://localhost:3001`. No new dev infra.
- **Production asset path.** `environment.prod.ts` serves `remoteEntry.js`
  from `/assets/react/`. Existing pilot already validates this path; no
  change.
- **CHANGELOG / issue tracking.** Follow the existing
  `[ZEPPELIN-####]` commit prefix convention; this work needs a new JIRA
  ticket. The pilot was `[ZEPPELIN-6371]`.

## Out of scope (explicitly)

- PublishedParagraph refactor onto the directive.
- Notebook paragraph control, code-editor, progress, action-bar, sidebar.
- Interpreter module migrations.
- Removing the old Angular footer code (kept as the fallback path during
  rollout).
