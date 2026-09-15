# RxJS-Stack: dataflow implementation roadmap

Updated: 2026-09-15. Audited baseline: `cbc91eefbccdeaf17221d06c75bdd237fe5e5499` on `main`.

The supplied `hansschenker/rxjs-full` URL resolves to `hansschenker/rxjs-stack`. This roadmap applies to that repository, not to the separate `rxjs-fullstack` project. It does not rename the repository or replace its Node HTTP architecture.

**Goal:** complete a small, functional, RxJS-7 application architecture in which events, state, derived values, rendering, effects, and server updates participate in explicit, owned dataflow.

**Status:** this revision delivers the architecture and implementation plan only. M00–M09 remain implementation work; none is marked complete merely because this document exists. Existing functionality is preserved as the starting point.

Read with [the architecture contract](dataflow-architecture.md) and [the source audit](repository-audit-2026-09-15.md). The previous roadmap is preserved unchanged in [the archive](archive/roadmap-before-dataflow-2026-09-15.md). Existing dated plans under `docs/superpowers/` remain historical implementation records, not evidence that every item in the old broad roadmap shipped.

## 1. Product decision

Prioritize **a small reactive application core plus a complete reference application**, rather than expanding immediately into a general-purpose, batteries-included framework.

The intended progression is:

```text
events → Observable streams → state streams → derived streams → rendering
                      ↘ effect policies → external work → result events ↗
```

Rendering and other effects are separate branches. A render must not initiate a server write. Results re-enter the state loop as typed messages. Server transport connects two explicitly managed runtimes; it does not transport an Observable object or create automatic distributed cancellation.

Retain RxJS 7, TypeScript, the custom JSX factory, Node HTTP, Zod, typed route contracts, existing middleware/authentication capabilities, and the existing SSE starting point. Prefer named pure domain functions and function-based factories. Do not introduce a component framework, Zone.js, decorators, or a second reactive state engine.

The first completion target is **architectural completeness for a client-rendered, in-memory, real-time Todo application**. Durable persistence and production-readiness are separate claims and require separate work.

## 2. Existing work to preserve

| Existing capability | Source | Treatment |
|---|---|---|
| Observable HTTP source and stream-transforming server `Effect` | `src/server/core/http.ts`, `bootstrap.ts`, `types.ts` | Retain; strengthen ownership and error isolation. |
| App context, route groups, middleware, response helpers, Zod validators | `src/server/core/` | Retain APIs where practical; test regressions. |
| Authentication wrapper | `src/server/core/middleware.ts`, `app.ts`; v0.3 changelog | Do not rebuild an auth framework. |
| Shared route contracts and generated client | `src/shared/routes.ts`, `src/client/api.ts` | Retain inference; add transport correctness and runtime decoding. |
| MVU reducer and accumulated client state | `src/client/todo.state.ts` | Reuse reducer behavior; remove module-global lifetime. |
| Custom JSX and TodoItem | `src/client/h.ts`, `components/todo-item.tsx` | Keep construction; add owned bindings and keyed updates. |
| Observable Todo store and SSE primitives | `src/server/todos/`, `src/client/sse.ts` | Integrate into the real app after lifetime tests. |
| Vitest, DOM tests, HTTP tests, CI | `src/**/*.test.ts`, `.github/workflows/ci.yml` | Extend, not replace. |

The changelog records v0.2 backend work, v0.3 auth, and v0.4 SSE. The old roadmap used those version labels for substantially broader ambitions. `package.json` still says `1.0.0`. Use milestone identifiers below until version policy is reconciled; do not invent a new release number or rewrite release history.

## 3. Milestone overview

| ID | Outcome | Depends on | Status |
|---|---|---|---|
| M00 | Reproducible baseline and reconciled project status | — | Pending |
| M01 | Explicit application/component/source lifetimes | M00 | Pending |
| M02 | Instance-owned state machine and coherent derived streams | M01 | Pending |
| M03 | Effect streams with explicit policies and correct HTTP outcomes | M02 | Pending |
| M04 | Owned, targeted reactive DOM rendering | M02, M03 | Pending |
| M05 | Server request, store, and SSE resource correctness | M00, M01 | Pending |
| M06 | Typed server-to-client live-state synchronization | M03, M05 | Pending |
| M07 | Complete Todo application and reactive form behavior | M04, M06 | Pending |
| M08 | Temporal traces and adversarial integration evidence | M07 | Pending |
| M09 | Documented API, runnable builds, and completion review | M08 | Pending |

Recommended serial order: M00 → M01 → M02 → M03 → M04 → M05 → M06 → M07 → M08 → M09. M05 is independent of the UI milestones after M01 and may be implemented earlier; its disconnect/error regressions should be characterized in M00. Never integrate live SSE before M05 passes.

Tests accompany every milestone. M08 adds cross-cutting evidence; it is not the point at which testing begins.

## M00 — Establish the baseline

**Purpose:** make the next change measurable rather than relying on historical documentation.

**Touch:** `package.json`, lockfile, `.gitignore`, `.github/workflows/ci.yml`, README, CHANGELOG, existing tests; add a short baseline record under `docs/`.

**Tasks**
1. Run a clean install, `npm run typecheck`, and `npm test` on the chosen, explicitly recorded Node 22 toolchain. Record versions, commit, commands, test counts, and limitations.
2. Resolve the documented version/status mismatch without relabeling historical releases. Add navigation from README to this roadmap and identify implemented versus planned features.
3. Remove tracked `node_modules` from the index in a dedicated housekeeping change, preserving the lockfile and without rewriting history. Review the unrelated `@hono/cli` development dependency before retaining or removing it; do not infer a Hono server from its presence.
4. Review open dependency-update PRs individually, especially any security-related updates. Do not auto-merge them as part of the architecture work.
5. Add characterization cases for non-2xx client responses, failed DELETE, synchronous startup results, root disposal, malformed route parameters, request-handler throws, and real SSE disconnect. Demonstrate current failures on a temporary branch; do not merge an intentionally failing main baseline.
6. Decide how to add a client build and server start/build smoke test. Do not silently add a packaging ecosystem.

**Acceptance:** clean-checkout commands have recorded results; known failures have reproducible cases and owners; generated dependencies are not tracked; the current branch status is explicit. Existing CI success is historical evidence, not a substitute for this run.

## M01 — Make lifetime and source ownership explicit

**Purpose:** every subscription and external resource has an owner before further abstraction is added.

**Touch:** new `src/client/runtime/scope.ts`, `src/client/runtime/sources.ts` and tests; extract browser startup from `main.tsx`; adapt `h.ts` only where ownership requires it.

**Tasks**
1. Define a function-based application scope with idempotent disposal and child component scopes. RxJS `Subscription` may implement ownership internally; do not create a parallel subscription system.
2. Make application construction inert. An explicit mount/start operation attaches sources and sinks. Importing feature modules must not subscribe, touch the DOM, open a connection, or issue a request.
3. Adapt DOM events to typed source streams. Capture the required event data synchronously, including `preventDefault` at the event boundary, before any asynchronous policy.
4. Define ownership contracts for event listeners, HTTP operations, EventSource connections, scheduled work, bindings, and child rows. Every resource declares whether it is per subscription, per component, or per app.
5. Register hot sources deliberately. A DOM event producer exists independently; adding a listener does not make the browser event producer cold.
6. Attach HMR disposal at the application boundary. A second mount must not inherit the previous mount's listeners or pending operations.

**Acceptance tests:** create without start performs no external work; one start attaches one owned set of sources; dispose twice is safe; mount/unmount/remount leaves exactly one active listener set; removing one child does not stop its siblings; no event or result updates a disposed app.

**Not included:** a general component framework or automatic dependency tracking.

## M02 — Build the state machine and derived-stream contract

**Purpose:** make memory, ordering, and sharing explicit while preserving the existing pure reducer approach.

**Touch:** `todo.state.ts`, new `todo.model.ts` / `todo.selectors.ts`, `src/client/runtime/program.ts`, state/runtime tests. Names are proposed file locations, not existing APIs.

**Tasks**
1. Replace the module-global `action$`, `state$`, and `dispatch` instance with a factory. Separate app instances and tests receive independent state and lifecycle.
2. Extend the existing `Action` union with UI intents, operation-started/succeeded/failed/cancelled facts where relevant, and server snapshot/connection facts. Keep intent, event, and transport message meanings distinct; no mandatory `Command` vocabulary.
3. Model initial loading, empty data, pending mutations, recoverable errors, form draft state, and connection status explicitly. Prefer readonly domain data and named update functions.
4. Accumulate state with `scan` and emit the initial state deliberately. Keep one shared accumulation alive for the mounted scope. Specify replay and reset behavior rather than using the positional `shareReplay(1)` default as a lifecycle design.
5. Connect the state owner and feedback consumers before enabling event sources or startup effects. Serialize reentrant feedback with a documented ingress policy; synchronous test effects must behave correctly too.
6. Where effects need state, give them a coherent transition record containing the accepted message and the appropriate state snapshot. Do not depend on incidental subscriber order between `actions$` and `state$`.
7. Derive a coherent `viewModel$` from a single state snapshot. Use named selectors and explicit equality. Do not split coupled fields into a diamond and then claim `combineLatest` makes updates transactional.

**Acceptance tests:** reducer determinism and immutability; isolated instances; initial value before the first external action; late subscribers see current state; extra consumers do not repeat reductions; synchronous startup feedback is not lost; coherent derived fields; state survives temporary loss of a view while the app owns it; a new scope resets to its own initial state.

**Important:** `shareReplay({ bufferSize: 1, refCount: true })` alone does not specify app lifetime. The mounted runtime must own the state connection; disposal must release that ownership and references.

## M03 — Separate effect streams and repair HTTP outcomes

**Purpose:** events describe intent, pure functions determine meaning, and effect streams implement explicit execution policies.

**Touch:** new `todo.effects.ts`, `todo.intents.ts`, `src/client/runtime/effects.ts` only if reuse is demonstrated; existing `api.ts`, `todo.service.ts`, relevant shared contracts and tests.

**Tasks**
1. Move network subscriptions and success/error dispatching out of rendering and TodoItem callbacks. Components emit typed intent values. Effect functions return result-event streams; the runtime owns their subscriptions and feeds results to state.
2. Use existing route contracts, but check status before decoding. Failed DELETE must not emit success. Handle expected empty bodies by response contract/status, not by a blanket method shortcut. Preserve structured server errors.
3. Decode external data from `unknown` with shared Zod schemas or explicit decoders. Generic return types and `as T` do not validate HTTP/SSE payloads.
4. Make HTTP work subscription-driven and connect unsubscription to transport abort, including body consumption. Inject the client/transport so tests do not depend on global fetch behavior.
5. Catch recoverable errors inside each operation. A failed save must not terminate future UI intent processing. Unexpected runtime faults are reported and cleaned up, not silently swallowed.
6. Correlate pending operations and their results. Request failure is not the same as cancellation; cancellation is not proof that a server write was rolled back.
7. Ensure extra subscribers to state, view models, or traces cannot execute an effect twice.

**Reference policies**

| Situation | Policy | Required qualification |
|---|---|---|
| Refresh/search reads | `switchMap`: latest request wins | Late results cannot overwrite newer intent. |
| Repeated create submit | `exhaustMap`: ignore while accepted submission is pending | Capture accepted draft; visibly disable/indicate pending. |
| Todo mutations | `concatMap`: serialize accepted writes in the initial implementation | Bound the pending queue; do not auto-retry non-idempotent writes. |
| Independent test/demo work | `mergeMap` with explicit concurrency limit | A concurrency limit alone does not bound queued inputs. |

Start with one simple ordered mutation queue. Per-entity queues are a later optimization only with bounded key lifetime and conflict tests. A newer query is not a reason to cancel an already accepted mutation.

**Acceptance tests:** all relevant non-2xx statuses become failure events; valid 204 succeeds; malformed bodies fail safely; cancel-before-headers and cancel-during-body release resources; policy marble tests assert inner subscription intervals; a failed request does not disable the next intent; no duplicate request from additional consumers.

## M04 — Render from streams with targeted DOM ownership

**Purpose:** make rendering an explicit sink without turning every state emission into a full reconstruction.

**Touch:** `h.ts`, `components/todo-item.tsx`, new `src/client/dom/bindings.ts`, `src/client/dom/keyed-list.ts`, `todo.view.tsx`, and DOM tests.

**Tasks**
1. Retain the small JSX element factory. Add narrow, scope-owned binding functions for text, properties, attributes, classes, and conditional content rather than introducing a second framework.
2. Build a stable shell once. Bind coherent derived values to the relevant DOM locations. Replace `listEl.innerHTML = ''` on every state emission with keyed Todo rows.
3. Give each row a child scope. Updating/reordering an existing key retains node identity; deleting it disposes listeners and bindings exactly once.
4. Event handlers capture input and emit intents only. They must not subscribe to HTTP work or mutate shared state directly.
5. Start with synchronous rendering for straightforward semantics and tests. Any frame coalescing must be a named, injectable policy with cancellation and order tests; do not globally delay all application messages.
6. Preserve focus, selection, unsent input, and native form behavior. Rendering the same view model again must not trigger writes.

**Acceptance tests:** an unrelated error/pending-state change does not rebuild the todo list; an unchanged row retains identity; update touches the intended row; delete tears down its scope; focus and draft survive server updates; removing the view releases all bindings; render never invokes an effect.

**Visible checkpoint:** the existing CRUD app now runs through sources → owned state → derived values → DOM, with effects outside the renderer.

## M05 — Strengthen the server stream runtime

**Purpose:** preserve the existing server composition while giving requests, stores, and live responses reliable lifetimes.

**Touch:** `src/server/core/http.ts`, `bootstrap.ts`, `app.ts`, `router.ts`, `types.ts`, `src/server/todos/todo.store.ts`, `todo.effect.ts`, and live HTTP/SSE tests.

**Tasks**
1. Give each request/response an owned lifetime and cancellation signal. Tie live response teardown to response/socket closure as appropriate; do not equate `IncomingMessage.close` with the end of an SSE response.
2. Make the SSE subscription a child of the response and application lifetime. Register cleanup before synchronous emissions can terminate it; handle disconnect, source complete/error, and app stop idempotently.
3. Catch failures around the entire per-request operation, including route matching, synchronous effect construction, middleware, and response writing. One malformed URL or handler throw must not terminate the server's root request stream.
4. Define finite-response cardinality, empty-handler behavior, and SSE ownership explicitly. An SSE response descriptor can be emitted once while its owned data stream remains active.
5. Make `start()` reflect listen readiness and startup errors; make `stop()` await closure and dispose active responses. Define repeated start/stop behavior and cleanup of startup failures.
6. Preserve the store factory and injected services, but move new domain transition logic into named pure functions. Serialize store transitions through one app-owned update stream and publish coherent snapshots. Preserve compatibility wrappers where required; avoid exported global stores in the application path.
7. Inject ID/time creation at the boundary. Do not hide these effects in domain reducers.
8. Define bounded policies for request admission and slow SSE consumers. For full-state snapshots, a bounded latest-snapshot policy is possible; do not silently drop domain-event streams. Treat transport buffering as an adapter concern, not automatic RxJS backpressure.

**Acceptance tests:** two app instances do not share state; a bad request is followed successfully by a good request; malformed percent encoding is isolated; outstanding reads are disposed on disconnect; SSE remains active after the GET request body completes and stops on response closure; app stop disposes live streams and releases the port; listener/queue/resource counts return to baseline.

**Early priority:** reproduce the request-close concern before M06. The audit includes an independent Node 22 event-order probe, but repository integration must still prove the correction.

## M06 — Close the server-to-client live-state loop

**Purpose:** make the existing SSE capability part of the actual application, with a defined consistency policy.

**Touch:** `src/shared/routes.ts`, new shared live-state schemas/contracts, `src/client/sse.ts`, `todo.effects.ts`, `todo.state.ts`, server store/stream effect, integration tests.

**Tasks**
1. Distinguish finite JSON routes from live stream routes in contracts/client construction. A live route must not be generated as an ordinary `res.json()` call. Keep existing finite-route inference intact.
2. Add an explicit live Todo contract with runtime decoding. Prefer a new versioned live route/format or a documented coordinated migration; do not silently reinterpret the existing bare `Todo[]` SSE payload.
3. For the reference app, choose server snapshots as the authoritative Todo collection. Mutation HTTP results settle pending operation status; live snapshots replace the collection. Do not append once from HTTP and again from SSE.
4. Attach one shared live connection per mounted app. A first connection or reconnection supplies a complete current snapshot. Display loading before the first snapshot and stale/reconnecting state during interruption.
5. Include a server-instance epoch and increasing snapshot revision (or an equivalently tested ordering contract). Reject stale/duplicate snapshots within an epoch; accept a fresh full snapshot on a new epoch. A server restart must not permanently lock out newer data because its revision counter restarted.
6. Make reconnection ownership explicit: either the browser EventSource reconnects or the RxJS adapter does; never both independently. Inject retry timing and define retry limits, terminal errors, and manual recovery.
7. Stop retries and the connection on scope disposal. Validate malformed payloads and define whether they fail/reconnect or produce a recoverable protocol failure.
8. Document transport limits: losing a connection can create a gap; resnapshot repairs collection state, not historical event delivery. No exactly-once or remote rollback guarantee is claimed.

**Acceptance tests:** one update reaches two mounted clients; each accepted write occurs once; identical/older snapshots do not duplicate state; reconnect produces current state; a new server epoch is accepted; extra UI consumers do not open extra connections; unmount during retry opens nothing later; malformed payloads cannot corrupt the model.

## M07 — Complete the reference application

**Purpose:** prove the whole model through user-visible behavior rather than a collection of disconnected utilities.

**Touch:** `src/client/main.tsx`, Todo feature modules, `index.html`, form/view tests, new browser end-to-end tests and scripts.

**Tasks**
1. Reduce `main.tsx` to construction, mount, and disposal of the Todo program. Preserve the existing application rather than scaffolding a second demo.
2. Complete load/create/toggle/delete, filtering, initial loading/empty/error displays, live connection status, and recoverable operation feedback.
3. Represent form draft, validation, and pending state through the same state/effect model. Capture the submitted draft; only clear the relevant accepted draft, not newer text the user typed while the request was pending.
4. Keep client validation pure and server validation authoritative. Use accessible native controls and explicit disabled/error states.
5. Demonstrate two clients, a server update, a network interruption, a failed mutation, and mount/unmount/remount. Document that the in-memory server loses Todos on restart.
6. Do not add a nested router, auth product, ORM, or CLI simply to make this demo larger.

**Acceptance:** a real browser test runs DOM intent → effect → HTTP → server transition → live snapshot → client reducer → targeted DOM update; a second client sees the same state; failures remain recoverable; teardown leaves no active app resources. Form and focus tests pass under server updates.

## M08 — Make time, causality, and cleanup inspectable

**Purpose:** provide evidence that the graph behaves correctly under overlap, cancellation, errors, and repeated mounting.

**Touch:** small opt-in trace hooks in runtime/transport boundaries, `src/testing/` helpers only where needed, new integration/temporal tests, CI.

**Tasks**
1. Trace source receipt, accepted intent, state transition, effect subscribe/next/error/complete/cancel, render commit, connection changes, and scope disposal.
2. Give records a runtime-local sequence number, source/scope identifier, operation correlation ID, and scheduler/clock timestamp. Equal timestamps do not establish order by themselves. Do not assert a global total order across client and server clocks.
3. Keep tracing observational: enabling it must not add an independent subscription that repeats effects. Avoid recording credentials or full sensitive payloads by default.
4. Add virtual-time tests for policies and real transport/browser tests for behavior virtual time cannot prove. Make actual resource ownership assertions rather than assuming `finalize` means cancellation.
5. Exercise bursts, queue limits, delayed bodies, out-of-order results, bad payloads, reconnect/dispose races, synchronous sources, repeated mounting, and unrelated simultaneous requests.
6. Capture a lightweight benchmark and resource baseline. Do not claim universal performance superiority or general glitch freedom.

**Acceptance:** a reproducible trace explains one complete operation and one cancellation; enabling tracing leaves request counts unchanged; owned subscriptions/listeners/connections return to baseline; all adversarial tests and existing tests pass in CI.

## M09 — Completion review and documented delivery

**Purpose:** turn the demonstrated architecture into a maintainable project with an honest scope.

**Touch:** README, this roadmap, architecture guide, CHANGELOG, public exports as needed, package scripts, build/start documentation, CI.

**Tasks**
1. Document the canonical application graph, construction/start/stop boundary, source temperature, sharing scope, state lifetime, effect policies, validation, and server synchronization contract.
2. Document the small function-based public surface actually proven by the Todo app. Avoid publishing speculative abstractions, a plugin system, or a multi-package workspace.
3. Add supported client/server build and start scripts plus a smoke test that does not rely on Vite's development proxy as the production deployment story. Specify same-origin/static hosting or an explicit reverse-proxy arrangement.
4. Reconcile package version, changelog, examples, and architecture terminology. Preserve attribution and historical documents.
5. Review all milestone acceptance evidence at one final commit. Document remaining limitations and migration notes for changed stream contracts.
6. No npm publication, release tagging, repository rename, or production deployment is implied by this planning task. Those are separate actions.

**Acceptance:** a fresh checkout follows documented commands and runs the complete app; tests/typecheck/build/smoke checks pass for the recorded commit; the architecture is demonstrated, not merely described; in-memory persistence, reconnect resnapshot semantics, and supported environments are explicit.

## 4. Old roadmap → revised treatment

| Old ambition | New treatment |
|---|---|
| Backend context, errors, routing, responses, validation, tests | Preserve implemented work; M00 baseline and M05 correctness. |
| Component/runtime improvements and local state | First-class goals in M01, M02, M04. |
| Data fetching and typed contracts | Preserve contracts; M03 lifecycle/outcome correctness and M06 stream contracts. |
| Reactive forms | Concrete Todo behavior in M07 before a reusable forms package. |
| Real-time primitives | Finish existing SSE in M05–M06 before adding WebSockets. |
| Client router, nested routes, guards, lazy loading | Deferred adapter work after the core loop is proven. |
| SSR/hydration | Explicitly client-rendered for this completion target. |
| Authentication expansion and security package | Preserve current middleware; do not defer correctness/security fixes, but defer a new auth product. |
| Persistence/transactions/ORM adapters | Separate follow-on milestone; no database replacement in this plan. |
| Jobs, queues, event-bus products | Defer; implement only the bounded local policies required by this app. |
| Observability/performance | Lightweight trace and resource evidence in M08, not a devtools platform. |
| CLI, code generation, plugins, ecosystem packages | Deferred; not prerequisites for architectural completion. |

## 5. Working agreement

One reviewable implementation change per milestone or independently testable substep. Start with a failing regression where behavior is being corrected, implement the smallest change, run the appropriate tests, and update this roadmap with the evidence commit. Never mark a milestone complete based only on a new module or passing happy-path example.

Do not mix dependency upgrades, repository renaming, a renderer rewrite, and server protocol changes in the same implementation PR. Preserve working code and the narrow proof loop throughout.

**First implementation session:** M00 baseline, followed by M01 ownership. Characterize the HTTP-status and SSE-close failures immediately; M03 and M05 own their fixes, with M05 eligible to run early.
