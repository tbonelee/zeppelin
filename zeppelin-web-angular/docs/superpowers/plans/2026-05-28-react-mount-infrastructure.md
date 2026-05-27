# React Mount Infrastructure (Footer Pilot) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a reusable Angular wrapper around Webpack Module Federation
mount calls, and migrate `NotebookParagraphFooterComponent` to React as the
first consumer behind a `?reactFooter=true` query-param gate.

**Architecture:** A singleton `ReactRemoteLoaderService` injects
`remoteEntry.js` once and caches per-module promises. A `ReactMountDirective`
owns its host element, calls `mount(el, props)` outside the Angular zone,
forwards subsequent prop changes through `handle.update(props)`, and unmounts
on destroy. The React-side `mount()` contract returns `{ update, unmount }`
instead of a bare unmount fn. `NotebookComponent` reads the activation flag
from its existing `queryParamMap` subscription and passes it down to each
paragraph.

**Tech Stack:** Angular 13, React 18, Webpack 5 Module Federation, date-fns 3,
Playwright (E2E only — no unit test runner is configured in this repo).

---

## Testing Strategy Deviation From Spec

The design spec lists unit tests for `ReactRemoteLoaderService`,
`ReactMountDirective`, and `ParagraphFooter`. Reality check: this repo has
**no unit test runner**.

- `ng test` is not configured in `angular.json` (no `test` target).
- No `karma.conf.js`, `test.ts`, or any `.spec.ts` files exist under
  `src/app`.
- The React project (`projects/zeppelin-react/`) has no test runner in
  `package.json` either.

Adding Karma + Jasmine + CI wiring is an order of magnitude more work than
the pilot. It is **out of scope** for this plan and tracked as a follow-up.

In its place, this plan uses:

1. **Playwright E2E tests** (Task 12) for the happy paths and the most
   important edge case (destroy-during-load).
2. **Manual verification steps** documented in each task where appropriate.
3. **Inline code-review focus** on the lifecycle edge cases the spec
   highlights: destroy-while-loading, props-change-while-loading, retry
   after script error.

When unit test infrastructure is added later, the test cases listed in the
spec become the seed list for retroactive coverage.

---

## File Structure

### New files

- `src/app/share/react-mount/react-mount-handle.ts` — type definitions
  shared by service, directive, and (in future) PublishedParagraph.
- `src/app/share/react-mount/react-remote-loader.service.ts` — singleton
  script + module loader.
- `src/app/share/react-mount/react-mount.directive.ts` — element-level
  mount/update/unmount wrapper.
- `src/app/share/react-mount/index.ts` — barrel exports.
- `projects/zeppelin-react/src/components/paragraph/ParagraphFooter.tsx` —
  React footer component.
- `projects/zeppelin-react/src/components/paragraph/ParagraphFooter.css` —
  ported styles.
- `projects/zeppelin-react/src/components/paragraph/ReactErrorBoundary.tsx`
  — error boundary used by the footer mount.
- `projects/zeppelin-react/src/components/paragraph/index.ts` — barrel.
- `e2e/tests/notebook/paragraph/react-footer.spec.ts` — Playwright E2E.

### Modified files

- `projects/zeppelin-react/package.json` — add `date-fns`.
- `projects/zeppelin-react/webpack.config.js` — expose
  `./ParagraphFooter`, update `remoteEntry.json` listing.
- `projects/zeppelin-react/src/main.ts` — re-export `ParagraphFooter` and
  its `mount`.
- `projects/zeppelin-react/src/components/index.ts` — re-export the new
  paragraph barrel.
- `projects/zeppelin-react/README.md` — rewrite "Adding a new React module"
  guidance.
- `src/app/share/share.module.ts` — declare and export
  `ReactMountDirective`.
- `src/app/pages/workspace/notebook/notebook.component.ts` — extend the
  existing `queryParamMap` subscription to read `reactFooter`.
- `src/app/pages/workspace/notebook/notebook.component.html` — pass
  `[useReactFooter]` into each paragraph.
- `src/app/pages/workspace/notebook/paragraph/paragraph.component.ts` —
  add `@Input() useReactFooter`, `reactFooterFailed`,
  `shouldUseReactFooter`, `onReactFooterError`.
- `src/app/pages/workspace/notebook/paragraph/paragraph.component.html` —
  gate the footer rendering.

---

## Task 1: Add `date-fns` dependency to the React remote

**Files:**
- Modify: `projects/zeppelin-react/package.json`

The React remote uses `date-fns` formatting in `ParagraphFooter` (Task 4
ports the existing `formatDistanceStrict`, `format`, `formatDistanceToNow`
calls). The host's `package.json:49` pins `^3.6.0`; match exactly for
behavior parity.

- [ ] **Step 1: Add the dependency**

In `projects/zeppelin-react/package.json`, add to `"dependencies"`
(alphabetically between `@zeppelin/sdk` and `file-saver`):

```json
    "date-fns": "^3.6.0",
```

- [ ] **Step 2: Install**

```bash
cd projects/zeppelin-react && npm install
```

Expected: `date-fns@3.6.x` resolves and `node_modules/date-fns/` is populated.

- [ ] **Step 3: Commit**

```bash
git add projects/zeppelin-react/package.json projects/zeppelin-react/package-lock.json
git commit -m "[ZEPPELIN-####] Add date-fns to zeppelin-react"
```

(Replace `####` with the JIRA ticket number assigned to this work. Use the
same ticket for every commit in this plan.)

---

## Task 2: Define the React mount handle type contract (Angular side)

**Files:**
- Create: `src/app/share/react-mount/react-mount-handle.ts`

This is the type the service consumer and directive both rely on. It also
documents the React-side export shape.

- [ ] **Step 1: Create the file with the contract**

```ts
/*
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *     http://www.apache.org/licenses/LICENSE-2.0
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */

export type ReactProps = Record<string, unknown>;

export interface ReactHostCallbacks {
  onError?: (error: unknown) => void;
}

export interface ReactMountHandle {
  update: (props: ReactProps & ReactHostCallbacks) => void;
  unmount: () => void;
}

export type ReactMountFn = (
  element: HTMLElement,
  props: ReactProps & ReactHostCallbacks
) => ReactMountHandle;

/**
 * Shape of a Module Federation exposed module: a factory returning an
 * object whose `mount` is the entry point.
 */
export interface ReactExposedModule {
  mount: ReactMountFn;
}

/**
 * Legacy shape (used by ./PublishedParagraph until its follow-up
 * refactor): mount returns a bare unmount function.
 */
export type LegacyMountFn = (
  element: HTMLElement,
  props: ReactProps
) => () => void;

export interface LegacyExposedModule {
  mount: LegacyMountFn;
}

export type AnyExposedModule = ReactExposedModule | LegacyExposedModule;
```

- [ ] **Step 2: Commit**

```bash
git add src/app/share/react-mount/react-mount-handle.ts
git commit -m "[ZEPPELIN-####] Add React mount handle type contract"
```

---

## Task 3: Implement `ReactRemoteLoaderService`

**Files:**
- Create: `src/app/share/react-mount/react-remote-loader.service.ts`

Singleton: one script element ever, one container promise, one module
promise per exposed key. Failed promises are evicted so retry works.

- [ ] **Step 1: Create the service**

```ts
/*
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *     http://www.apache.org/licenses/LICENSE-2.0
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */

import { Injectable } from '@angular/core';
import { environment } from '../../../environments/environment';
import { AnyExposedModule } from './react-mount-handle';

interface RemoteContainer {
  get<T>(key: string): Promise<() => T>;
  init?: (shareScope: unknown) => Promise<void>;
}

declare global {
  interface Window {
    reactApp?: RemoteContainer;
  }
}

@Injectable({ providedIn: 'root' })
export class ReactRemoteLoaderService {
  private containerPromise: Promise<RemoteContainer> | null = null;
  private readonly modulePromises = new Map<string, Promise<AnyExposedModule>>();

  loadContainer(): Promise<RemoteContainer> {
    if (this.containerPromise) {
      return this.containerPromise;
    }

    this.containerPromise = new Promise<RemoteContainer>((resolve, reject) => {
      if (window.reactApp) {
        resolve(window.reactApp);
        return;
      }

      const script = document.createElement('script');
      script.src = environment.reactRemoteEntryUrl;
      script.async = true;
      script.onload = () => {
        if (!window.reactApp) {
          reject(new Error('window.reactApp not registered after script load'));
          return;
        }
        resolve(window.reactApp);
      };
      script.onerror = () => {
        reject(new Error(`Failed to load React remote at ${script.src}`));
      };
      document.head.appendChild(script);
    });

    // Clear the container promise AND drain module cache on failure so a
    // future caller can retry. A stale failed container would otherwise
    // poison every module fetch.
    this.containerPromise.catch(() => {
      this.containerPromise = null;
      this.modulePromises.clear();
    });

    return this.containerPromise;
  }

  loadModule<T extends AnyExposedModule>(exposedKey: string): Promise<T> {
    const cached = this.modulePromises.get(exposedKey);
    if (cached) {
      return cached as Promise<T>;
    }

    const promise = (async () => {
      const container = await this.loadContainer();
      const factory = await container.get<T>(exposedKey);
      return factory();
    })();

    this.modulePromises.set(exposedKey, promise);

    // Evict failed module promise so the next caller can retry.
    promise.catch(() => {
      this.modulePromises.delete(exposedKey);
    });

    return promise;
  }
}
```

- [ ] **Step 2: Manual verification (dev console)**

This service has no unit tests. After Task 8 (directive in share module),
verify the loader manually with the browser dev console:

```js
// In the dev console, while serving the app:
const svc = ng.getInjector('zeppelin-app').get('ReactRemoteLoaderService');
// NOTE: this is illustrative — the real path is harder. Defer manual check
// until Task 12 E2E runs.
```

For practical verification, the E2E test in Task 12 exercises the happy
path and one failure path end-to-end. Code review focus for this task:

- `loadContainer` returns the same promise instance for concurrent calls
  before `onload`.
- `script.onerror` clears `containerPromise` AND the module map.
- `loadModule` evicts the cache entry when the inner promise rejects.

- [ ] **Step 3: Commit**

```bash
git add src/app/share/react-mount/react-remote-loader.service.ts
git commit -m "[ZEPPELIN-####] Add ReactRemoteLoaderService"
```

---

## Task 4: Implement `ReactMountDirective`

**Files:**
- Create: `src/app/share/react-mount/react-mount.directive.ts`

State machine fields: `latestProps`, `destroyed`, `loading`, `handle`.
Re-checks `destroyed` after every await. Wraps mount in try/catch and
puts `loading = false` in a `finally`.

- [ ] **Step 1: Create the directive**

```ts
/*
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *     http://www.apache.org/licenses/LICENSE-2.0
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */

import {
  Directive,
  ElementRef,
  Input,
  NgZone,
  OnChanges,
  OnDestroy,
  SimpleChanges
} from '@angular/core';
import { ReactRemoteLoaderService } from './react-remote-loader.service';
import {
  AnyExposedModule,
  LegacyExposedModule,
  ReactExposedModule,
  ReactHostCallbacks,
  ReactMountHandle,
  ReactProps
} from './react-mount-handle';

function isLegacyModule(
  mod: AnyExposedModule,
  handleOrUnmount: unknown
): handleOrUnmount is () => void {
  void mod;
  return typeof handleOrUnmount === 'function';
}

function wrapLegacyHandle(unmount: () => void): ReactMountHandle {
  return {
    update: () => {
      /* legacy modules don't support updates; no-op */
    },
    unmount
  };
}

@Directive({
  selector: '[zeppelinReactMount]'
})
export class ReactMountDirective implements OnChanges, OnDestroy {
  @Input('zeppelinReactMount') module!: string;
  @Input() reactProps: ReactProps & ReactHostCallbacks = {};

  private latestProps: ReactProps & ReactHostCallbacks = {};
  private destroyed = false;
  private loading = false;
  private handle: ReactMountHandle | null = null;
  private mountedModule: string | null = null;

  constructor(
    private readonly host: ElementRef<HTMLElement>,
    private readonly ngZone: NgZone,
    private readonly loader: ReactRemoteLoaderService
  ) {}

  ngOnChanges(changes: SimpleChanges): void {
    this.latestProps = this.reactProps ?? {};

    if (changes.module && !changes.module.firstChange && this.mountedModule) {
      // Module swap after first mount is unsupported. Report via onError
      // and otherwise leave the existing handle in place.
      this.reportError(
        new Error(
          `ReactMountDirective: module input changed after mount ` +
            `(from "${this.mountedModule}" to "${this.module}") — unsupported`
        )
      );
      return;
    }

    if (this.handle) {
      this.ngZone.runOutsideAngular(() => {
        try {
          this.handle!.update(this.latestProps);
        } catch (err) {
          this.reportError(err);
        }
      });
      return;
    }

    if (!this.loading && !this.destroyed && this.module) {
      void this.startLoad();
    }
  }

  ngOnDestroy(): void {
    this.destroyed = true;
    if (this.handle) {
      try {
        this.handle.unmount();
      } catch (err) {
        this.reportError(err);
      }
      this.handle = null;
    }
  }

  private async startLoad(): Promise<void> {
    this.loading = true;
    const moduleKey = this.module;
    try {
      const mod = await this.loader.loadModule<AnyExposedModule>(moduleKey);
      if (this.destroyed) {
        return;
      }
      this.ngZone.runOutsideAngular(() => {
        try {
          const returned = (mod as ReactExposedModule).mount(
            this.host.nativeElement,
            this.latestProps
          );
          if (isLegacyModule(mod, returned)) {
            this.handle = wrapLegacyHandle(returned as unknown as () => void);
          } else {
            this.handle = returned as ReactMountHandle;
          }
          this.mountedModule = moduleKey;
        } catch (err) {
          this.handle = null;
          this.reportError(err);
        }
      });
    } catch (err) {
      this.reportError(err);
    } finally {
      this.loading = false;
    }
  }

  private reportError(error: unknown): void {
    const onError = this.latestProps.onError;
    if (typeof onError === 'function') {
      try {
        onError(error);
      } catch {
        /* swallow callback errors; they shouldn't loop */
      }
    } else {
      console.error('[ReactMountDirective]', error);
    }
  }
}
```

- [ ] **Step 2: Code-review focus**

Walk through these scenarios mentally before moving on:

- **Destroy before load resolves:** `startLoad` checks `this.destroyed`
  after the await. If true, returns without mounting. `ngOnDestroy` sees
  no handle, so no unmount. ✅
- **Props change twice while loading:** Each `ngOnChanges` updates
  `latestProps`. `startLoad` reads `this.latestProps` after the await,
  not a captured value. ✅
- **Module load rejects:** outer try/catch surfaces via `onError`. The
  module cache eviction (Task 3) lets a retry work. ✅
- **`mount` throws synchronously:** inner try/catch keeps `handle` null;
  `onError` fires; no half-mounted state. ✅
- **`update` throws:** caught in `ngOnChanges`; reported via `onError`;
  `unmount` on destroy still runs because `handle` is still set. ✅
- **Module swap after mount:** explicit error via `onError`; no mutation. ✅
- **`loading` after error:** `finally` resets it, so a future `ngOnChanges`
  could re-trigger load if `module` input changes (which we forbid above).
  The combination is safe. ✅

- [ ] **Step 3: Commit**

```bash
git add src/app/share/react-mount/react-mount.directive.ts
git commit -m "[ZEPPELIN-####] Add ReactMountDirective"
```

---

## Task 5: Add barrel and wire into `ShareModule`

**Files:**
- Create: `src/app/share/react-mount/index.ts`
- Modify: `src/app/share/share.module.ts`

- [ ] **Step 1: Create the barrel**

```ts
/*
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *     http://www.apache.org/licenses/LICENSE-2.0
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */

export * from './react-mount-handle';
export * from './react-mount.directive';
export * from './react-remote-loader.service';
```

- [ ] **Step 2: Declare and export the directive from `ShareModule`**

Edit `src/app/share/share.module.ts`:

1. Add import near the other share imports (after `ResizeHandleComponent`
   import):

```ts
import { ReactMountDirective } from './react-mount';
```

2. Add `ReactMountDirective` to the `EXPORT_LIST` array (around line 67):

```ts
const EXPORT_LIST = [
  HeaderComponent,
  NodeListComponent,
  NoteTocComponent,
  PageHeaderComponent,
  SpinComponent,
  ThemeToggleComponent,
  ResizeHandleComponent
];
```

becomes:

```ts
const EXPORT_LIST = [
  HeaderComponent,
  NodeListComponent,
  NoteTocComponent,
  PageHeaderComponent,
  SpinComponent,
  ThemeToggleComponent,
  ResizeHandleComponent,
  ReactMountDirective
];
```

3. Verify `EXPORT_LIST` is already passed to both `declarations` and
   `exports` in the `@NgModule` decorator (it is — line 79 and 80). No
   further edits needed.

- [ ] **Step 3: Build to verify wiring**

```bash
cd /Users/user/my-projects/zeppelin2/zeppelin-web-angular
npm run build:angular -- --configuration development 2>&1 | tail -20
```

Expected: build succeeds with no errors. If `ReactMountDirective` is
unresolved, the import path is wrong.

- [ ] **Step 4: Commit**

```bash
git add src/app/share/react-mount/index.ts src/app/share/share.module.ts
git commit -m "[ZEPPELIN-####] Wire ReactMountDirective into ShareModule"
```

---

## Task 6: Implement the React error boundary

**Files:**
- Create: `projects/zeppelin-react/src/components/paragraph/ReactErrorBoundary.tsx`

Wraps the footer's render tree and reports errors to Angular via
`props.onError`. Renders nothing on error so Angular's fallback path takes
over.

- [ ] **Step 1: Create the file**

```tsx
/*
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *     http://www.apache.org/licenses/LICENSE-2.0
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */

import { Component, ErrorInfo, ReactNode } from 'react';

interface Props {
  onError?: (error: unknown) => void;
  children: ReactNode;
}

interface State {
  hasError: boolean;
}

export class ReactErrorBoundary extends Component<Props, State> {
  state: State = { hasError: false };

  static getDerivedStateFromError(): State {
    return { hasError: true };
  }

  componentDidCatch(error: Error, _info: ErrorInfo): void {
    if (typeof this.props.onError === 'function') {
      try {
        this.props.onError(error);
      } catch {
        /* swallow */
      }
    }
  }

  render(): ReactNode {
    if (this.state.hasError) {
      return null;
    }
    return this.props.children;
  }
}
```

- [ ] **Step 2: Commit**

```bash
git add projects/zeppelin-react/src/components/paragraph/ReactErrorBoundary.tsx
git commit -m "[ZEPPELIN-####] Add ReactErrorBoundary"
```

---

## Task 7: Port footer styles to a co-located CSS file

**Files:**
- Create: `projects/zeppelin-react/src/components/paragraph/ParagraphFooter.css`

Source: `src/app/pages/workspace/notebook/paragraph/footer/footer.component.less:13-21`.
The original uses `@text-color-secondary` from `themeMixin`. Resolve that
to the actual color in the active theme.

- [ ] **Step 1: Resolve the theme color**

Run the dev server and inspect the existing Angular footer in the
browser. From a notebook with a paragraph that finished running, find the
computed color of `.footer .execution-time`. Note the hex value. (At time
of writing this is the ng-zorro default secondary text color; record what
your dev environment shows.)

- [ ] **Step 2: Create the CSS**

```css
/*
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *     http://www.apache.org/licenses/LICENSE-2.0
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */

.zeppelin-react-paragraph-footer {
  color: <hex from Step 1>;
  font-size: 12px;
  margin-top: 12px;
}
```

Replace `<hex from Step 1>` with the value you recorded.

The class is prefixed with `zeppelin-react-` to avoid colliding with the
Angular host's `.footer` class on the same page.

- [ ] **Step 3: Commit**

```bash
git add projects/zeppelin-react/src/components/paragraph/ParagraphFooter.css
git commit -m "[ZEPPELIN-####] Add ParagraphFooter styles"
```

---

## Task 8: Implement the React `ParagraphFooter` component

**Files:**
- Create: `projects/zeppelin-react/src/components/paragraph/ParagraphFooter.tsx`

Mirrors `NotebookParagraphFooterComponent`'s `getExecutionTime` and
`getElapsedTime` logic. `mount` returns `{ update, unmount }`.

- [ ] **Step 1: Create the file**

```tsx
/*
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *     http://www.apache.org/licenses/LICENSE-2.0
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */

import { createRoot, Root } from 'react-dom/client';
import { format, formatDistanceStrict, formatDistanceToNow } from 'date-fns';
import { ReactErrorBoundary } from './ReactErrorBoundary';
import './ParagraphFooter.css';

export interface ParagraphFooterProps {
  dateStarted?: string;
  dateFinished?: string;
  dateUpdated?: string;
  showExecutionTime?: boolean;
  showElapsedTime?: boolean;
  user?: string;
  onError?: (error: unknown) => void;
}

function isOutdated(dateUpdated?: string, dateStarted?: string): boolean {
  return (
    dateUpdated !== undefined &&
    dateStarted !== undefined &&
    Date.parse(dateUpdated) > Date.parse(dateStarted)
  );
}

function computeExecutionTime(props: ParagraphFooterProps): string {
  const { dateStarted, dateFinished, user, dateUpdated } = props;
  if (dateFinished === undefined || dateStarted === undefined) {
    return '';
  }
  const timeMs = Date.parse(dateFinished) - Date.parse(dateStarted);
  if (isNaN(timeMs) || timeMs < 0) {
    return isOutdated(dateUpdated, dateStarted) ? 'outdated' : '';
  }

  const durationFormat = formatDistanceStrict(
    new Date(dateStarted),
    new Date(dateFinished)
  );
  const endFormat = format(new Date(dateFinished), 'MMMM dd yyyy, h:mm:ss a');
  const userLabel = user === undefined || user === null ? 'anonymous' : user;
  let desc = `Took ${durationFormat}. Last updated by ${userLabel} at ${endFormat}.`;
  if (isOutdated(dateUpdated, dateStarted)) {
    desc += ' (outdated)';
  }
  return desc;
}

function computeElapsedTime(dateStarted?: string): string {
  const base = dateStarted ? new Date(dateStarted) : new Date();
  return `Started ${formatDistanceToNow(base)} ago.`;
}

export const ParagraphFooter = (props: ParagraphFooterProps) => {
  const { showExecutionTime, showElapsedTime } = props;
  const executionTime = computeExecutionTime(props);
  const elapsedTime = computeElapsedTime(props.dateStarted);

  return (
    <div className="zeppelin-react-paragraph-footer" data-testid="react-paragraph-footer-content">
      {showExecutionTime && <div className="execution-time">{executionTime}</div>}
      {showElapsedTime && <div className="elapsed-time">{elapsedTime}</div>}
    </div>
  );
};

export interface ParagraphFooterMountHandle {
  update: (props: ParagraphFooterProps) => void;
  unmount: () => void;
}

export const mount = (
  element: HTMLElement,
  initialProps: ParagraphFooterProps
): ParagraphFooterMountHandle => {
  if (!element) {
    throw new Error('Mount element is required');
  }

  const root: Root = createRoot(element);

  const renderWith = (props: ParagraphFooterProps) => {
    root.render(
      <ReactErrorBoundary onError={props.onError}>
        <ParagraphFooter {...props} />
      </ReactErrorBoundary>
    );
  };

  renderWith(initialProps);

  return {
    update: (newProps: ParagraphFooterProps) => {
      renderWith(newProps);
    },
    unmount: () => {
      root.unmount();
    }
  };
};
```

- [ ] **Step 2: Commit**

```bash
git add projects/zeppelin-react/src/components/paragraph/ParagraphFooter.tsx
git commit -m "[ZEPPELIN-####] Add React ParagraphFooter component"
```

---

## Task 9: Add the paragraph barrel and re-export from the React project

**Files:**
- Create: `projects/zeppelin-react/src/components/paragraph/index.ts`
- Modify: `projects/zeppelin-react/src/components/index.ts`
- Modify: `projects/zeppelin-react/src/main.ts`

- [ ] **Step 1: Create the paragraph barrel**

```ts
/*
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *     http://www.apache.org/licenses/LICENSE-2.0
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */

export * from './ParagraphFooter';
export * from './ReactErrorBoundary';
```

- [ ] **Step 2: Re-export from components**

Replace `projects/zeppelin-react/src/components/index.ts`:

```ts
/*
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *     http://www.apache.org/licenses/LICENSE-2.0
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */

export * from './renderers';
export * from './visualizations';
export * from './common';
export * from './paragraph';
```

- [ ] **Step 3: Re-export the new `mount` from main.ts**

Replace `projects/zeppelin-react/src/main.ts`:

```ts
/*
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *     http://www.apache.org/licenses/LICENSE-2.0
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */

export { PublishedParagraph, mount } from './pages/PublishedParagraph';
export {
  ParagraphFooter,
  mount as mountParagraphFooter
} from './components/paragraph/ParagraphFooter';
```

- [ ] **Step 4: Commit**

```bash
git add projects/zeppelin-react/src/components/paragraph/index.ts \
        projects/zeppelin-react/src/components/index.ts \
        projects/zeppelin-react/src/main.ts
git commit -m "[ZEPPELIN-####] Re-export ParagraphFooter from React app"
```

---

## Task 10: Expose `./ParagraphFooter` from Webpack

**Files:**
- Modify: `projects/zeppelin-react/webpack.config.js`

- [ ] **Step 1: Add to `exposes` and `remoteEntry.json`**

In `projects/zeppelin-react/webpack.config.js`:

1. Update the `exposes` object (currently lines 71-73):

```js
exposes: {
  './PublishedParagraph': './src/pages/PublishedParagraph',
  './ParagraphFooter': './src/components/paragraph/ParagraphFooter'
},
```

2. Update the `remoteEntry.json` template (currently lines 103-105). The
   file is informational only (no Angular code reads it), but keep it
   accurate:

```js
exposes: {
  './PublishedParagraph': './PublishedParagraph.tsx',
  './ParagraphFooter': './ParagraphFooter.tsx'
}
```

- [ ] **Step 2: Build the React project**

```bash
cd projects/zeppelin-react && npm run build
```

Expected: `dist/` contains a chunk for `ParagraphFooter`. The build
finishes without errors.

- [ ] **Step 3: Commit**

```bash
git add projects/zeppelin-react/webpack.config.js
git commit -m "[ZEPPELIN-####] Expose ParagraphFooter via Module Federation"
```

---

## Task 11: Wire `useReactFooter` through the notebook

**Files:**
- Modify: `src/app/pages/workspace/notebook/notebook.component.ts`
- Modify: `src/app/pages/workspace/notebook/notebook.component.html`
- Modify: `src/app/pages/workspace/notebook/paragraph/paragraph.component.ts`
- Modify: `src/app/pages/workspace/notebook/paragraph/paragraph.component.html`

This task is the biggest single edit. Follow the steps strictly in order
and commit only at the end.

- [ ] **Step 1: Read `reactFooter` in `NotebookComponent.ngOnInit`**

In `src/app/pages/workspace/notebook/notebook.component.ts`, find the
`ngOnInit` block at line 423-433. Extend the existing `queryParamMap`
subscription. The current code is:

```ts
this.activatedRoute.queryParamMap
  .pipe(
    startWith(this.activatedRoute.snapshot.queryParamMap),
    takeUntil(this.destroy$),
    map(data => data.get('paragraph'))
  )
  .subscribe(id => {
    this.onParagraphSelect(id);
    this.onParagraphScrolled(id);
  });
```

This subscription is already shaped to read one param. Don't break it.
**Add a second subscription** alongside it (immediately after):

```ts
this.activatedRoute.queryParamMap
  .pipe(
    startWith(this.activatedRoute.snapshot.queryParamMap),
    takeUntil(this.destroy$)
  )
  .subscribe(data => {
    this.useReactFooter = data.get('reactFooter') === 'true';
    this.cdr.markForCheck();
  });
```

Also add a field declaration near the other component fields (search for
`sidebarWidth` or similar layout fields):

```ts
useReactFooter = false;
```

- [ ] **Step 2: Pass the flag into each paragraph in the template**

Open `src/app/pages/workspace/notebook/notebook.component.html`. Find the
`<zeppelin-notebook-paragraph` tag (search for the tag name). Add an
`[useReactFooter]` input binding:

```html
<zeppelin-notebook-paragraph
  ...existing inputs...
  [useReactFooter]="useReactFooter"
></zeppelin-notebook-paragraph>
```

- [ ] **Step 3: Add the inputs and fallback flag to the paragraph component**

In `src/app/pages/workspace/notebook/paragraph/paragraph.component.ts`,
add to the class body:

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

Also expose the props object as a getter so the template doesn't
construct a new object literal every change-detection pass:

```ts
get reactFooterProps(): Record<string, unknown> {
  if (!this.paragraph) {
    return { onError: this.onReactFooterError };
  }
  return {
    dateStarted: this.paragraph.dateStarted,
    dateFinished: this.paragraph.dateFinished,
    dateUpdated: this.paragraph.dateUpdated,
    showExecutionTime: !this.paragraph.config.tableHide && !this.viewOnly,
    showElapsedTime: this.paragraph.status === 'RUNNING',
    user: this.paragraph.user,
    onError: this.onReactFooterError
  };
}
```

Ensure `@Input` is imported. Open the imports at the top of the file and
add `Input` to the `@angular/core` import list if it isn't there already.

- [ ] **Step 4: Gate the footer in the paragraph template**

In `src/app/pages/workspace/notebook/paragraph/paragraph.component.html`,
find the footer markup at line 113:

```html
<zeppelin-notebook-paragraph-footer
  [showExecutionTime]="!paragraph.config.tableHide && !viewOnly"
  [showElapsedTime]="paragraph.status === 'RUNNING'"
  [user]="paragraph.user"
  [dateUpdated]="paragraph.dateUpdated"
  [dateStarted]="paragraph.dateStarted"
  [dateFinished]="paragraph.dateFinished"
></zeppelin-notebook-paragraph-footer>
```

Replace with:

```html
<ng-container *ngIf="shouldUseReactFooter; else angularFooter">
  <div
    data-testid="react-paragraph-footer"
    zeppelinReactMount="./ParagraphFooter"
    [reactProps]="reactFooterProps"
  ></div>
</ng-container>
<ng-template #angularFooter>
  <zeppelin-notebook-paragraph-footer
    data-testid="angular-paragraph-footer"
    [showExecutionTime]="!paragraph.config.tableHide && !viewOnly"
    [showElapsedTime]="paragraph.status === 'RUNNING'"
    [user]="paragraph.user"
    [dateUpdated]="paragraph.dateUpdated"
    [dateStarted]="paragraph.dateStarted"
    [dateFinished]="paragraph.dateFinished"
  ></zeppelin-notebook-paragraph-footer>
</ng-template>
```

- [ ] **Step 5: Build to verify**

```bash
npm run build:angular -- --configuration development 2>&1 | tail -30
```

Expected: clean build, no template errors.

- [ ] **Step 6: Manual smoke check**

Start both servers (`npm start`) and visit:

1. `/#/notebook/<note-id>` — should render the Angular footer (no flag).
2. `/#/notebook/<note-id>?reactFooter=true` — should render the React
   footer (open dev tools, confirm `[data-testid="react-paragraph-footer"]`
   exists and `[data-testid="angular-paragraph-footer"]` does not).
3. Run a paragraph in (2) and verify that
   `[data-testid="react-paragraph-footer-content"]` updates from the
   "elapsed" string to the "Took … Last updated by … at …" string when
   the paragraph finishes.

If any step fails, fix before continuing. The E2E (Task 12) will codify
these.

- [ ] **Step 7: Commit**

```bash
git add src/app/pages/workspace/notebook/notebook.component.ts \
        src/app/pages/workspace/notebook/notebook.component.html \
        src/app/pages/workspace/notebook/paragraph/paragraph.component.ts \
        src/app/pages/workspace/notebook/paragraph/paragraph.component.html
git commit -m "[ZEPPELIN-####] Gate notebook paragraph footer behind ?reactFooter=true"
```

---

## Task 12: Add Playwright E2E coverage

**Files:**
- Create: `e2e/tests/notebook/paragraph/react-footer.spec.ts`

Follows the patterns in
`e2e/tests/notebook/published/published-paragraph.spec.ts`. Use the
existing `createTestNotebook` helper.

- [ ] **Step 1: Create the test file**

```ts
/*
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *     http://www.apache.org/licenses/LICENSE-2.0
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */

import { expect, test } from '@playwright/test';
import {
  addPageAnnotationBeforeEach,
  createTestNotebook,
  PAGES,
  performLoginIfRequired,
  waitForNotebookLinks,
  waitForZeppelinReady
} from '../../../utils';

test.describe('React Paragraph Footer', () => {
  addPageAnnotationBeforeEach(PAGES.WORKSPACE.NOTEBOOK);

  let testNotebook: { noteId: string; paragraphId: string };

  test.beforeEach(async ({ page }) => {
    await page.goto('/#/');
    await waitForZeppelinReady(page);
    await performLoginIfRequired(page);
    await waitForNotebookLinks(page);
    testNotebook = await createTestNotebook(page);
  });

  test('without reactFooter flag, Angular footer renders', async ({ page }) => {
    const { noteId } = testNotebook;

    await page.goto(`/#/notebook/${noteId}`);
    await waitForZeppelinReady(page);

    await expect(
      page.locator('[data-testid="angular-paragraph-footer"]').first()
    ).toBeAttached({ timeout: 15000 });
    await expect(
      page.locator('[data-testid="react-paragraph-footer"]')
    ).toHaveCount(0);
  });

  test('with reactFooter=true, React footer renders', async ({ page }) => {
    const { noteId } = testNotebook;

    await page.goto(`/#/notebook/${noteId}?reactFooter=true`);
    await waitForZeppelinReady(page);

    await expect(
      page.locator('[data-testid="react-paragraph-footer"]').first()
    ).toBeAttached({ timeout: 15000 });
    await expect(
      page.locator('[data-testid="angular-paragraph-footer"]')
    ).toHaveCount(0);
  });

  test('reactFooter=true preserves the paragraph query param', async ({ page }) => {
    const { noteId, paragraphId } = testNotebook;

    await page.goto(
      `/#/notebook/${noteId}?paragraph=${paragraphId}&reactFooter=true`
    );
    await waitForZeppelinReady(page);

    await expect(page).toHaveURL(/reactFooter=true/);
    await expect(page).toHaveURL(new RegExp(`paragraph=${paragraphId}`));
    await expect(
      page.locator('[data-testid="react-paragraph-footer"]').first()
    ).toBeAttached({ timeout: 15000 });
  });

  test('navigating away during remoteEntry load does not throw', async ({ page }) => {
    const { noteId } = testNotebook;

    // Delay remoteEntry.js to widen the destroy-while-loading window
    await page.route('**/remoteEntry.js', async route => {
      await new Promise(r => setTimeout(r, 1500));
      await route.continue();
    });

    await page.goto(`/#/notebook/${noteId}?reactFooter=true`);
    // Navigate away before the remote can possibly mount
    await page.waitForTimeout(100);
    await page.goto('/#/');
    await waitForZeppelinReady(page);

    // No uncaught errors recorded — Playwright's page error listener
    // would fail the test if React mount happened after destroy.
    const consoleErrors: string[] = [];
    page.on('pageerror', err => consoleErrors.push(err.message));
    await page.waitForTimeout(2500);

    expect(consoleErrors).toEqual([]);
  });
});
```

- [ ] **Step 2: Run the new tests**

```bash
npm run e2e:fast -- e2e/tests/notebook/paragraph/react-footer.spec.ts
```

Expected: all four tests pass. If any fail, fix the implementation (most
likely missing `data-testid` attributes or a typo in the
`zeppelinReactMount` directive selector).

- [ ] **Step 3: Run the full notebook E2E to catch regressions**

```bash
npm run e2e:fast -- e2e/tests/notebook
```

Expected: no new failures introduced by the changes in Task 11.

- [ ] **Step 4: Commit**

```bash
git add e2e/tests/notebook/paragraph/react-footer.spec.ts
git commit -m "[ZEPPELIN-####] Add E2E coverage for React paragraph footer"
```

---

## Task 13: Update the React project README

**Files:**
- Modify: `projects/zeppelin-react/README.md`

The current "Adding a new React module" section (lines 79-99) teaches the
one-shot `mount(element, props): () => unmount` pattern. That contract is
obsolete with the new directive; an engineer following it would skip
`update`.

- [ ] **Step 1: Replace the "Adding a new React module" section**

Replace lines 79-99 (the section starting with `## Adding a new React module`)
with:

```markdown
## Adding a new React module

The Angular host loads each exposed module through the
`ReactMountDirective` (see `src/app/share/react-mount/`). The contract:

```ts
export interface ReactMountHandle {
  update: (props: Props & { onError?: (e: unknown) => void }) => void;
  unmount: () => void;
}

export function mount(element: HTMLElement, props: Props): ReactMountHandle;
```

1. Create a component (e.g. `src/components/<area>/ExampleFeature.tsx`).
2. Wrap its render tree in `<ReactErrorBoundary onError={props.onError}>`.
3. Export a `mount(element, props)` function that:
   - Creates a single `Root` via `createRoot(element)`.
   - Calls `root.render(<Wrapped {...props}/>)` on initial mount AND on
     every `update(newProps)` call. React's reconciler preserves state.
   - Returns `{ update, unmount }`. `unmount` calls `root.unmount()`.
4. Register in `webpack.config.js` under `exposes`:
   ```js
   exposes: {
     './PublishedParagraph': './src/pages/PublishedParagraph',
     './ParagraphFooter': './src/components/paragraph/ParagraphFooter',
     './ExampleFeature': './src/components/<area>/ExampleFeature'
   }
   ```
5. Re-export from `main.ts`:
   ```ts
   export {
     ExampleFeature,
     mount as mountExampleFeature
   } from './components/<area>/ExampleFeature';
   ```
6. Use from Angular by adding the directive to your template:
   ```html
   <div
     zeppelinReactMount="./ExampleFeature"
     [reactProps]="exampleFeatureProps"
   ></div>
   ```
   `exampleFeatureProps` should be a getter on the host component (not
   an inline object literal) so identity is stable when nothing changed.

The legacy `./PublishedParagraph` module returns a bare unmount fn from
`mount`. The directive tolerates that shape, but new modules should use
the handle contract.
```

- [ ] **Step 2: Add a short "React mount infrastructure" section**

Above the existing "Migration roadmap" section, add:

```markdown
## React mount infrastructure (Angular side)

The Angular host's `src/app/share/react-mount/` exports two pieces:

- `ReactRemoteLoaderService` — loads `remoteEntry.js` once per page,
  caches per-module promises, evicts on error.
- `ReactMountDirective` — owns the host element, mounts outside the
  Angular zone, forwards `[reactProps]` changes through
  `handle.update(...)`, and unmounts on destroy. Re-checks `destroyed`
  after the async load so a navigation during load does not leak a mount.

Both are exported from `ShareModule`. Any notebook or interpreter
template can use the directive without additional wiring.
```

- [ ] **Step 3: Commit**

```bash
git add projects/zeppelin-react/README.md
git commit -m "[ZEPPELIN-####] Update zeppelin-react README for new mount contract"
```

---

## Self-Review

Before handing off, walk through:

1. **Spec coverage:** Tasks 1-13 cover every section of
   `docs/superpowers/specs/2026-05-27-react-mount-infrastructure-design.md`
   except the spec's unit-test enumeration. That deviation is named at
   the top of this plan with rationale.
2. **Placeholder scan:** Two intentional placeholders exist —
   `[ZEPPELIN-####]` (JIRA ticket) and `<hex from Step 1>` (theme color
   captured at dev-time). Both have explicit instructions for filling
   them in.
3. **Type consistency:** `ReactMountHandle` / `ReactProps` /
   `ReactHostCallbacks` are defined in Task 2 and referenced unchanged
   in Tasks 3, 4. The React-side `ParagraphFooterMountHandle` (Task 8)
   has the same shape, intentionally.
4. **No silent regressions:** Task 11 keeps the Angular footer as the
   default (`useReactFooter` defaults to `false`). The pilot is opt-in
   only.
