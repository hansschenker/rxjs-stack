# Dataflow architecture contract

Design revision: 2026-09-15. Applies to `hansschenker/rxjs-stack`, reached through the supplied `rxjs-full` URL. This is the target contract for the [implementation roadmap](roadmap.md), not a claim that the audited code already satisfies every rule.

## 1. One model, two runtime boundaries

```text
Browser sources
DOM events / startup / transport events
                 |
                 v
          typed input messages
                 |
                 v
       serialized state transitions <--------------------+
                 |                                      |
          shared current state                          |
                 |                                      |
         +-------+----------------+                     |
         |                        |                     |
  coherent viewModel       accepted intents /           |
         |                 transition snapshots         |
         v                        |                     |
  owned DOM bindings        effect policies             |
         |                        |                     |
         v                        v                     |
        DOM               HTTP / other adapters         |
                                  |                     |
                            result messages ------------+

Server boundary:
HTTP request -> validation -> server effect -> store transition
                                               |
                                       shared snapshot stream
                                               |
                                       owned SSE response
                                               |
Browser boundary: bytes -> decoder -> snapshot message ---+
```

Rendering is already an effect on the DOM. It is a separate sink from network/storage effects, not a prerequisite for them. Re-rendering never repeats a write. An effect result can change state, which can change the view, without the view initiating the effect.

The same Observable protocol is used inside both runtimes. Across the network, the protocol is a separately defined message format with validation, ordering, recovery, and lifetime rules. RxJS subscription cancellation does not itself transmit a server-cancel command or reverse an accepted mutation.

## 2. Vocabulary and module responsibilities

| Term | Meaning | Responsibility |
|---|---|---|
| Event | Something happened: submit, response, snapshot, disconnect. | Capture at a source boundary. |
| Intent | The domain meaning of a requested action. | Named pure interpretation functions. |
| Message / existing `Action` | Typed value accepted by the app state machine. | Discriminated union; transport shape is not assumed identical. |
| State | Current remembered application facts. | Pure reducer accumulated once per app scope. |
| Derived value | Projection from a state snapshot. | Named selector; no external effects. |
| Effect description/policy | Which operation is relevant and how competing work is treated. | Explicit stream composition. |
| Effect execution | Actual HTTP, storage, timer, or DOM interaction. | Owned adapter/subscription boundary. |
| Server `Effect` | Existing stream-to-stream request/response function type. | Preserve its repository meaning; do not conflate it with a side-effect descriptor. |
| Scope | Owner of a mounted app, component, request, or connection. | Controls start and disposal. |

Do not force a new `Command` vocabulary. Preserve `Action` compatibility where appropriate, while documenting intent versus completed fact. Domain functions may return ordinary values; they do not all need to become Observables.

## 3. Construction, activation, and disposal

Creating a program describes its graph and dependencies. Starting or mounting it subscribes/connects that graph. Importing a feature module must not start it.

A proposed `createTodoProgram`/`mountTodoApp` split is an implementation direction, not an existing public API. Implement the smallest useful function-based surface first. No application class hierarchy is needed.

The runtime owns state accumulation, effects, rendering bindings, source listeners, timers, and feedback. Libraries/adapters may subscribe internally where necessary, but each subscription must have a named owner and a teardown path. A rule of “exactly one subscribe call in the entire codebase” would be the wrong constraint.

Startup order is contractual: connect the state owner and downstream consumers before startup work or external sources can emit. Synchronous sources must work; network latency must not accidentally supply the initialization barrier.

Shutdown first prevents new accepted inputs, then releases child resources and queued/scheduled work, and finally releases state ownership. Disposal is idempotent. A newly constructed/mounted instance starts with its own state.

## 4. State, sharing, and ordering

The application model remains a pure reducer with `scan`. State is not a mutable public Subject. A Subject may bridge external input at the boundary; downstream consumers receive read-only Observable access.

Use explicit replay and connection policies. A mounted root holds the state connection for its own lifetime, including periods with no view subscribers. Additional consumers must not repeat state accumulation or start new requests. On disposal, release the graph and its replayed data references.

RxJS 7.8.2's `shareReplay` implementation defaults `refCount` to false and does not reset on completion. These are operator semantics, not automatically the application's desired lifetime. The implementation must test the chosen policy against late subscription, child removal, app disposal, and remount. Source reference: [RxJS 7.8.2 shareReplay](https://github.com/ReactiveX/rxjs/blob/7.8.2/src/internal/operators/shareReplay.ts).

For reentrant feedback, select an explicit serialized ingress policy. A result produced synchronously while processing an input must not overtake that input halfway through propagation. A queue-based scheduler can implement this, but its exact location and effects must be tested; scheduling alone is not a transaction system.

Where an effect depends on state, prefer a transition snapshot associated with its input. Avoid relying on which subscriber happened to run first. For coupled view fields, derive one coherent view model from one state emission. Do not claim arbitrary RxJS graphs are glitch-free.

## 5. Error, completion, and cancellation are different

An expected operation failure becomes a typed result value for the application, while the intent-processing graph remains alive. A source's terminal `error` still ends that particular subscription; recovery is an explicit policy at an appropriate boundary.

Cancellation releases owned work and blocks obsolete results. It is not `complete`, not a successful result, and not evidence of remote rollback. Finalization records cleanup for completion, error, and unsubscription; a trace needs separate cause information to identify cancellation correctly.

A client request adapter must check HTTP status, handle contract-appropriate empty bodies, validate decoded data, and preserve structured failures. Unsubscribing should abort underlying work where the platform adapter supports it. A mutation that may already have committed is not automatically retried.

Unexpected runtime faults must be reported with scope context and have defined cleanup. Do not silently convert every exception into `EMPTY` and hide the failure from the UI or operator.

## 6. Temporal and concurrency policies

| Boundary | Initial policy |
|---|---|
| Read refresh/search | Latest relevant read replaces its predecessor. |
| Create submission | Ignore duplicate submissions while one accepted create is pending. |
| Accepted Todo mutations | Ordered queue, with explicit finite queue capacity/admission behavior. |
| Independent work | Explicit overlap limit and separate admission/queue bound. |
| Rendering | Synchronous targeted commits initially. Any animation-frame policy is named and injected. |
| Reconnect | One policy owner, explicit delays/limits, cancelled with the scope. |
| Slow SSE consumers | A documented bounded transport policy; full snapshots may be coalesced, domain events may not be silently discarded. |

Source clocks and schedulers supply time. Operators apply the requested delay, selection, ordering, and concurrency policies. Tests inject time instead of waiting for arbitrary wall-clock sleeps.

## 7. Rendering contract

The existing `h()` factory constructs nodes; keep that useful primitive. Build a stable shell, bind derived values, and reconcile Todo rows by key. Each row owns its bindings and event adapters. Removing it disposes those resources while preserving unrelated nodes.

DOM handlers are adapters: capture a value, perform required immediate event handling, emit an intent. They do not initiate nested HTTP subscriptions. Named pure functions decide which domain intent that input represents.

A render changes only the required DOM state. It must preserve native input behavior, focus, selection, and drafts. Render work may be represented as an Observable of commit notifications where useful, but ordinary pure view construction must not be artificially wrapped in streams just for uniform syntax.

## 8. Server contract

Keep the current Observable HTTP source, router-as-stream-transformer, app context, middleware, and Zod validation. Strengthen them rather than replacing Node HTTP with a different server framework.

The server app owns its store independent of the number of SSE subscribers. Each HTTP request owns its operation; each live response owns its source subscription. Ending one live client does not stop the store or other clients. App stop closes all owned resources and waits for the server to release its port.

Guard the entire request boundary. Route matching and effect construction can fail before an operator-level `catchError` is reached. These failures must produce a response or an explicit connection outcome without ending service for later requests.

Server domain transitions use named pure functions; IDs and current time enter through injected boundary functions. Start with the in-memory store already in the repository. Its lack of durability is explicit; persistence adapters are not implied by an Observable interface.

## 9. Live-state consistency decision

Use the existing HTTP/SSE foundation for the first complete loop. Do not make a WebSocket rewrite a prerequisite.

The reference mode uses live server snapshots as the authoritative Todo collection. A mutation response records operation success/failure; it does not independently append the same item that an SSE snapshot will include. On a fresh live connection, receive a current full snapshot rather than racing an initial GET against unversioned pushes.

A new typed snapshot envelope should identify the server epoch and snapshot revision. Within an epoch, revisions are monotonic. After a restart/new epoch, a fresh snapshot is accepted and the demo explicitly acknowledges that an in-memory store may have reset.

The old bare-array SSE route must either remain available beside a new typed live contract or change through an explicit coordinated migration. Runtime `unknown` decoding is required at the transport boundary.

Reconnection repairs current collection state by resnapshot. It is not an exactly-once event log, and it does not prove that a disconnected mutation was or was not committed. Do not add optimistic writes until this simpler authoritative model is proven.

## 10. Scope boundary

This plan proves an RxJS-first application architecture, not an entire replacement for Angular, Next, or Nest. It does not include SSR, a routing framework, a new auth product, ORM abstractions, CLI scaffolding, plugins, code generation, a distributed backpressure protocol, or rich devtools.

Correctness, validation, bounded resources, and cleanup are not optional “production extras.” They are part of the core proof. Additional platforms and persistence can be added later as adapters, without changing the machine's domain-independent coordination model.

**Completion means:** the reference application demonstrates the whole loop, tests verify its temporal/lifetime contracts, and a fresh checkout can run the documented build. It does not mean every desired web-framework feature exists.
