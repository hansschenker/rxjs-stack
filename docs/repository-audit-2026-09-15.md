# RxJS-Stack repository audit — 2026-09-15

## Scope and evidence

The requested `hansschenker/rxjs-full` repository resolves through GitHub to **`hansschenker/rxjs-stack`**, repository ID `1238476691`. This audit concerns that repository only. It is not an audit of the separate `rxjs-fullstack` project and does not assume its Hono-based architecture.

Baseline inspected: [`cbc91eefbccdeaf17221d06c75bdd237fe5e5499`](https://github.com/hansschenker/rxjs-stack/commit/cbc91eefbccdeaf17221d06c75bdd237fe5e5499), `main`, authored 2026-05-15. The revision changes documentation only. Proposed paths and APIs in the new roadmap are not claims about existing implementation.

Sources were read through the connected GitHub API: README, the complete `docs/roadmap.md`, CLAUDE.md, package.json, CHANGELOG, CI configuration, the client and server modules cited below, shared contracts, and representative unit/integration tests. Source trees and open pull requests were also inspected. Existing dated `docs/superpowers/` plans were located, not exhaustively audited; they remain unchanged.

## What already works as a foundation

The repository is a framework-free TypeScript application with RxJS 7, custom JSX, a Node HTTP Observable source, an MVU client, and Zod validation. Its server includes application/service context, route helpers and grouping, error/response helpers, authentication wrapping, and testing utilities. Shared route contracts generate typed finite HTTP client methods. There are both server and client SSE primitives.

Sources: [README](../README.md), [server app](../src/server/core/app.ts), [server router](../src/server/core/router.ts), [client API](../src/client/api.ts), [shared routes](../src/shared/routes.ts), [CHANGELOG](../CHANGELOG.md).

These are assets to preserve. The revised direction is not a replacement server, replacement renderer framework, or migration away from RxJS 7. The work is chiefly to connect the pieces with explicit ownership, effect policies, coherent state, and a verified live loop.

## Findings and implementation ownership

### A01 — The old roadmap and implemented version labels diverged

**Observed:** the old roadmap assigned broad full-stack ambitions to v0.3 and production-hardening ambitions to v0.4. The changelog instead records v0.3 authentication and v0.4 SSE. `package.json` says `1.0.0`, also used for the initial historical release.

**Consequence:** a version label alone cannot be used as evidence that all roadmap features are complete. The new plan uses M00–M09 and records acceptance evidence separately from release numbering.

**Action:** M00 reconciles status and documentation without rewriting historical release facts. The old roadmap is archived byte-for-byte. M09 finalizes a coherent delivery description.

Sources: [original roadmap archive](archive/roadmap-before-dataflow-2026-09-15.md), [CHANGELOG](../CHANGELOG.md), [package.json](../package.json).

### A02 — Client composition is still concentrated in imperative entry-point wiring

**Observed:** `src/client/main.tsx` starts the initial request directly, handles submission with an `exhaustMap`, subscribes to update/delete requests inside TodoItem handlers, and dispatches result actions from callbacks. Some update/delete errors become `EMPTY` without a visible error state. No root disposal is present in this entry point.

**Consequence:** there are useful local stream policies, but no single explicit owner and testable feedback contract for the application. A subscription inside an adapter is not inherently wrong; an unowned operation inside rendering/event wiring is the concern here.

**Action:** M01 introduces ownership; M02 defines the state loop; M03 extracts effects; M07 reduces the entry point to assembly.

Source: [main.tsx at baseline](https://github.com/hansschenker/rxjs-stack/blob/cbc91eefbccdeaf17221d06c75bdd237fe5e5499/src/client/main.tsx).

### A03 — State has a useful reducer but a module-global lifetime

**Observed:** `todo.state.ts` exports one `Subject`, one `scan`/`startWith`/`shareReplay(1)` pipeline, and one dispatch function. Actions primarily describe successful CRUD results or a generic error. Pending/loading/connection/form state and a per-instance lifetime are absent from this model.

**Risk inferred from source:** the entry point starts its first request before subscribing to state. Actual network asynchrony normally delays that result, but a synchronous test service could expose an initialization-order gap. That case needs a regression test rather than an assumption that asynchronous timing will always protect startup.

**Action:** M02 creates isolated state instances, deliberate initial emission and root ownership, coherent selectors, and explicit reentrant feedback handling. Merely changing the `shareReplay` configuration is not a complete lifetime design.

Sources: [state](https://github.com/hansschenker/rxjs-stack/blob/cbc91eefbccdeaf17221d06c75bdd237fe5e5499/src/client/todo.state.ts), [state tests](https://github.com/hansschenker/rxjs-stack/blob/cbc91eefbccdeaf17221d06c75bdd237fe5e5499/src/client/todo.state.test.ts), [RxJS 7.8.2 sharing implementation](https://github.com/ReactiveX/rxjs/blob/7.8.2/src/internal/operators/shareReplay.ts).

### A04 — Rendering rebuilds all Todo rows

**Observed:** every state emission clears `listEl.innerHTML` and constructs fresh TodoItem nodes. `h.ts` creates ordinary DOM elements and static listeners; it is not an Observable-binding or keyed-reconciliation runtime.

**Consequence:** targeted updates and child-owned cleanup still need implementation. Node identity and input/focus behavior should become measured acceptance criteria, not presumed performance benefits.

**Action:** M04 retains the JSX factory and adds narrow owned bindings and keyed rows. Rendering must not initiate network work.

Sources: [main.tsx](https://github.com/hansschenker/rxjs-stack/blob/cbc91eefbccdeaf17221d06c75bdd237fe5e5499/src/client/main.tsx), [h.ts](https://github.com/hansschenker/rxjs-stack/blob/cbc91eefbccdeaf17221d06c75bdd237fe5e5499/src/client/h.ts).

### A05 — HTTP status is not checked before reporting data

**Observed:** `requestCore$` calls `fromFetch`, then chooses `undefined` for every DELETE or parses JSON for other methods. There is no `res.ok`/status guard. The API test response double contains only `json()` and does not model status or abort behavior.

**Consequence derived directly from the branch logic:** a failed DELETE can flow through the success path; an error JSON body can be treated as a successful domain result. Generic TypeScript return types do not check the network payload. Body-consumption cancellation also needs explicit tests rather than inference from the `fromFetch` name.

**Action:** M03 repairs status/empty-body handling, structured errors, runtime decoding, and cancellation through body consumption. Tests must include realistic responses and failed operations.

Sources: [API implementation](https://github.com/hansschenker/rxjs-stack/blob/cbc91eefbccdeaf17221d06c75bdd237fe5e5499/src/client/api.ts), [API tests](https://github.com/hansschenker/rxjs-stack/blob/cbc91eefbccdeaf17221d06c75bdd237fe5e5499/src/client/api.test.ts).

### A06 — SSE exists, but the application does not yet consume it

**Observed:** the server exposes `GET /todos/stream`; the client has a cold `fromEventSource` adapter. `main.tsx` does not use that adapter. Its `onerror` closes the connection and errors the subscription. The stream route is declared without a typed live payload, while the general client builder assumes JSON responses. Incoming SSE JSON is asserted as `T`, not runtime-validated.

**Action:** M06 separates finite and streaming contracts, defines one shared app-owned connection and one reconnect owner, validates snapshots, and routes them through state. The chosen first consistency model is authoritative live snapshots, not duplicate updates from both HTTP success and SSE.

Sources: [server entry point](https://github.com/hansschenker/rxjs-stack/blob/cbc91eefbccdeaf17221d06c75bdd237fe5e5499/src/server/main.ts), [SSE client](https://github.com/hansschenker/rxjs-stack/blob/cbc91eefbccdeaf17221d06c75bdd237fe5e5499/src/client/sse.ts), [route contracts](https://github.com/hansschenker/rxjs-stack/blob/cbc91eefbccdeaf17221d06c75bdd237fe5e5499/src/shared/routes.ts), [client entry point](https://github.com/hansschenker/rxjs-stack/blob/cbc91eefbccdeaf17221d06c75bdd237fe5e5499/src/client/main.tsx).

### A07 — Server request/response lifetime boundaries need strengthening

**Observed:** `applySse` subscribes and then registers cleanup on the request's `close` event. The subscription is not returned to the app owner. `createServer` only returns `server.close()` as teardown. The bootstrap has a root error logger, while some matching/construction failures occur outside the route effect's `catchError`. `start()` does not await listening and `stop()` does not await server closure.

**Risks to verify:** request completion is not a sufficient model of a long-lived response; root shutdown must own live subscriptions; one malformed URL or synchronous handler/middleware failure should not terminate service for subsequent requests; startup/shutdown races need real-server tests. The current SSE unit test manually triggers a mocked request-close callback, so it does not establish actual Node HTTP lifetime behavior.

**Action:** M05 owns the response/request/app hierarchy, complete per-request error isolation, start/stop readiness, finite response cardinality, and bounded transport buffering. M00 records regression cases early.

Sources: [HTTP adapter](https://github.com/hansschenker/rxjs-stack/blob/cbc91eefbccdeaf17221d06c75bdd237fe5e5499/src/server/core/http.ts), [bootstrap](https://github.com/hansschenker/rxjs-stack/blob/cbc91eefbccdeaf17221d06c75bdd237fe5e5499/src/server/core/bootstrap.ts), [router](https://github.com/hansschenker/rxjs-stack/blob/cbc91eefbccdeaf17221d06c75bdd237fe5e5499/src/server/core/router.ts), [app](https://github.com/hansschenker/rxjs-stack/blob/cbc91eefbccdeaf17221d06c75bdd237fe5e5499/src/server/core/app.ts), [HTTP unit tests](https://github.com/hansschenker/rxjs-stack/blob/cbc91eefbccdeaf17221d06c75bdd237fe5e5499/src/server/core/http.test.ts).

### A08 — The server store is already injectable; preserve that work

**Observed:** `createTodoStore()` creates an independent BehaviorSubject-backed store, and the server entry point injects a fresh store into its app. Compatibility exports also create a module-global default store. CRUD effects perform get/transform/set operations and generate time/IDs inline.

**Action:** M05 preserves the factory and service boundary, extracts pure transitions, injects clock/ID sources, and owns state progression independently of connected SSE clients. Do not describe the actual server entry point as using the global default store; it does not.

Sources: [store](https://github.com/hansschenker/rxjs-stack/blob/cbc91eefbccdeaf17221d06c75bdd237fe5e5499/src/server/todos/todo.store.ts), [effects](https://github.com/hansschenker/rxjs-stack/blob/cbc91eefbccdeaf17221d06c75bdd237fe5e5499/src/server/todos/todo.effect.ts), [server main](https://github.com/hansschenker/rxjs-stack/blob/cbc91eefbccdeaf17221d06c75bdd237fe5e5499/src/server/main.ts).

## Validation performed and limitations

### Repository CI evidence

The latest inspected push-triggered **CI** run for the baseline commit, [run 25921869496](https://github.com/hansschenker/rxjs-stack/actions/runs/25921869496), completed successfully on 2026-05-15. Its job reports successful dependency installation, typecheck, and test steps. This is historical remote evidence, not a fresh execution performed during this audit.

README/CHANGELOG report 180 tests across 19 files. That count was not independently recounted by running the suite. Existing real HTTP tests cover malformed/oversized JSON and missing-item responses; the inspected SSE lifecycle tests use mocks. Source: [HTTP integration tests](https://github.com/hansschenker/rxjs-stack/blob/cbc91eefbccdeaf17221d06c75bdd237fe5e5499/src/server/http.integration.test.ts).

### Local execution limitation

A local clone was attempted, but the execution container could not resolve `github.com`. Repository files were inspected through the GitHub connector instead. No local `npm ci`, repository typecheck, repository test suite, browser end-to-end test, or production build was successfully executed in this audit. M00 remains necessary.

### Independent platform probe

A small standalone Node **22.16.0** HTTP probe was executed locally with built-in Node modules. It was not a run of this repository. It consumed a GET request, attached an SSE-style request-close listener afterward, kept the response open, and then deliberately disconnected the client.

Observed relative timing in that single run:

| Approximate time | Observation |
|---|---|
| 12 ms | An early `IncomingMessage.close` listener fired. |
| 12 ms | Request body had been consumed; the late SSE-style request-close listener was attached. |
| 14 ms | Client received the first event chunk. |
| 73–74 ms | The response remained writable; a later chunk reached the client. |
| 134 ms | Client intentionally disconnected. |
| 135 ms | `ServerResponse.close` fired. |

The late request-close listener did not run. This supports prioritizing a real response-lifetime regression in M05; it does **not** prove the outcome of every repository/network configuration. Exact millisecond timing is incidental. The ordering and cleanup boundary are the relevant observations. The probe source and JSON result accompany the downloadable audit bundle, outside production code.

## Non-claims

This is not an exhaustive security audit, a benchmark, proof of production readiness, or a completed implementation of the milestones. No dependency PRs were merged, no runtime code was modified, and no package/release/deployment action is part of this documentation revision.

The recommended next implementation work is **M00 baseline and M01 ownership**, with HTTP-status and real SSE-lifetime regressions characterized immediately. Those concrete gaps take precedence over adding new framework features.
