# Phased Migration Playbook

## Phase 0: Baseline

1. Lock current behavior with tests around critical flows.
- If no tests exist, write characterization tests first: call the code, capture actual output, assert it. These tests document current behavior rather than intended behavior and act as a regression baseline throughout migration.
2. Record performance-sensitive chains and threading expectations.
3. Add a migration tracker for remaining Rx types/imports per module.
4. Classify streams by completion behavior:
- finite (request-like),
- optional result (`Maybe`),
- completion-only (`Completable`),
- effectively infinite UI/event streams,
- command/effect streams (`Action`): trigger-driven single-flight execution with explicit `isExecuting` and `errors` state.
- queue-trigger command streams (`CommandQueue` pattern): FIFO-triggered effect processing that must be planned with explicit teardown semantics.

## Phase 1: Boundary Isolation

1. Create adapter protocols at module boundaries.
2. Keep Rx implementation behind adapters.
3. Introduce destination interfaces (`async` APIs and/or `Publisher` APIs).
4. Use RxSwift 6.5 bridge APIs to migrate incrementally:
- `observable.values` / trait `.value` on the async side,
- `asObservable()` and `Single.create { try await ... }` on the Rx side (`Single.create` async-closure overload requires RxSwift 6.5+).

Example direction:
- `protocol UserServiceRx { func user(id: String) -> Single<User> }`
- `protocol UserService { func user(id: String) async throws -> User }`
- Provide temporary adapter implementations both ways only where needed.

### Monolithic-Target Variant (No Module Boundaries)

1. Create feature-local seams instead of module seams:
- facade protocols per feature (`FeatureServiceRx` -> `FeatureService`),
- wrapper types around legacy singletons.
2. Migrate one vertical slice at a time (entrypoint -> ViewModel -> service) behind feature flags.
3. Keep legacy and migrated paths side-by-side temporarily, then delete legacy path per feature.

## Phase 2: Domain/Core Migration

1. Convert service and repository internals first.
2. Prefer `async/await` for request/response and transactional logic.
3. Keep stream-centric modules on Combine until domain migration is stable.
4. Prefer structured concurrency (`async let`, `withTaskGroup`) for fan-out work.
5. Avoid defaulting to `Task.detached`; detached tasks do not inherit parent actor isolation, priority, or task-local values.
6. Networking-specific migration:
- Prefer `URLSession` async APIs directly (`data(for:)`, `bytes(for:)`) and map `Single<Response>` to `async throws -> Response`.
- Recreate retry/timeout/reachability chains explicitly (retry loops with backoff, timeout races, `NWPathMonitor`-backed gating).
- For `retryWhen`-style behavior, use error-aware loops (`do/catch` + error inspection + conditional delay) instead of naive fixed-count retries.
- For one-shot callback APIs, use `withCheckedThrowingContinuation` + `withTaskCancellationHandler` for cancel cleanup.
- Nesting pattern: use `withCheckedThrowingContinuation` inside the `operation:` closure of `withTaskCancellationHandler`, and store cancellation/resume state in a shared reference so `onCancel:` can safely resume with `CancellationError` exactly once.
- The shared reference must be thread-safe (for example lock-guarded or atomic), because `onCancel:` and the legacy callback can race.
- Use `withCheckedThrowingContinuation` only when the callback contract is exactly-once completion; do not use it for zero/multi-callback APIs.
- For streaming or multi-callback APIs, use `AsyncThrowingStream`; for fan-out/fan-in orchestration, use `withThrowingTaskGroup`/`withThrowingDiscardingTaskGroup` (`withThrowingDiscardingTaskGroup` is Swift 5.9+ / iOS 17+).

## Phase 3: Presentation Migration

1. Migrate ViewModels and presenters from `Driver`/`Signal` to:
- `@Published` + Combine pipelines, or
- `@MainActor` state + async event handling.
- For raw trait-less `Observable<T>` outputs crossing the ViewModel boundary (`searchResults`, `isLoading`, etc.), choose one output contract per module and migrate consistently:
  - `@Published` for `ObservableObject`-based ViewModels,
  - `AnyPublisher` for Combine-heavy architectures,
  - actor-isolated state + `AsyncStream` for concurrency-first designs.
- For ViewModel inputs previously modeled as `PublishRelay`/`PublishSubject`/`BehaviorRelay`/`BehaviorSubject`, choose an explicit input surface:
  - prefer direct async methods for one-shot user intents,
  - use `PassthroughSubject`/`CurrentValueSubject` only at UI-boundary composition points in Combine-heavy modules,
  - use actor-owned `AsyncStream` continuations for concurrency-first event pipelines.
- Keep subject/relay-like input plumbing out of domain services; terminate it at the ViewModel boundary.
2. Replace `DisposeBag` with `AnyCancellable` storage or task cancellation.
3. Migrate `Action`-backed ViewModel commands explicitly:
- replace `action.execute(input)` with `await command.execute(input)` (or a publisher-backed command trigger),
- preserve single-flight execution semantics (do not allow concurrent command runs unless intentionally changed),
- expose and bind `isExecuting`, `isEnabled`, and `errors` to UI state.
- replace `button.rx.tap.bind(to: action.inputs)` with target-action (or `UIAction`) that starts a caller-owned `Task` invoking `await command.execute(...)`.
- on iOS 14+, prefer `UIControl.addAction(_:for:)` with `UIAction` closures over selector-only target-action wiring.
- for ViewController-owned tasks, capture `[weak self]`, store task handles, and cancel on `viewWillDisappear`/`deinit`; for ViewModel-owned tasks, keep strong `self` and cancel in lifecycle teardown.
- choose and document trigger policy per command: ignore-while-running, latest-wins (cancel previous), or queue; do not let this policy be implicit.
- latest-wins pattern: keep `var currentTask: Task<Void, Never>?`, cancel it before launching a new task, and treat `CancellationError` as a non-error path (do not surface as user-visible failure unless required).
- cooperative-cancellation race: a replacement `execute` can still throw `CommandGateError.alreadyExecuting` before prior unwind completes; either accept that dropped replacement (same handling as cancellation) or await/retry before relaunch.
- for `Task<Void, Never>` latest-wins flows, handle thrown domain errors inside the task body (for example route to UI error state/logging); do not silently drop them via blanket `try?` unless explicitly intended.
- keep `currentTask` in an isolated owner (`@MainActor` ViewController/ViewModel or an actor); storing it on a non-isolated type risks Swift 6 data races.
- queue pattern: keep an actor-backed FIFO + drain loop (`isDraining` flag). `enqueue` appends and starts drain if idle; drain executes one item at a time and exits when queue is empty.
- queue sketch:
```swift
actor CommandQueue {
    private var jobs: [@Sendable () async -> Void] = []
    private var isDraining = false
    private var isInvalidated = false
    private var drainTask: Task<Void, Never>?

    func enqueue(_ job: @escaping @Sendable () async -> Void) {
        guard !isInvalidated else { return } // queue is single-use after invalidate
        jobs.append(job)
        guard !isDraining else { return }
        isDraining = true
        drainTask = Task { [weak self] in
            await self?.drain()
        }
    }

    func invalidate() {
        isInvalidated = true
        drainTask?.cancel()
        drainTask = nil
        jobs.removeAll()
        isDraining = false
    }

    private func drain() async {
        defer {
            isDraining = false
            drainTask = nil
            // If more jobs arrived while draining, restart before releasing idle state.
            if !isInvalidated, !jobs.isEmpty {
                isDraining = true
                drainTask = Task { [weak self] in
                    await self?.drain()
                }
            }
        }

        while !jobs.isEmpty {
            if Task.isCancelled { return }
            let next = jobs.removeFirst()
            await next() // actor is re-entrant while awaiting
        }
    }
}
```
- Do not add arbitrary early exits from `drain()` (except cancellation/teardown paths); otherwise queued work can be stranded.
- Call `invalidate()` from the owner lifecycle to cancel pending drain work and break queue retention.
- `invalidate()` drops queued (not-yet-started) jobs immediately; a currently executing job stops only if it cooperatively observes cancellation.
- Do not call `enqueue` after `invalidate`; this queue sketch is intentionally single-use after teardown.
4. Migrate UIKit/RxCocoa bindings explicitly:
- replace `.rx.text`, `.rx.tap`, `.rx.controlEvent` with target-action or publisher equivalents,
- replace `.rx.items` / RxDataSources with diffable data sources,
- preserve identity and section-diff behavior in table/collection updates.

## Phase 4: Strict Concurrency Hardening (Swift 6)

1. Resolve all `Sendable` and actor-isolation diagnostics.
2. Isolate mutable shared state into actors or `@MainActor` types.
3. Remove unsafe cross-thread shared references.
4. Make cancellation explicit and cooperative:
- check cancellation in long-running work,
- stop child work when parent task is canceled.

### Phase 4 Concrete Recipes

1. Third-party or legacy modules not annotated for concurrency:
- use `@preconcurrency import LegacyModule` as a temporary compatibility bridge,
- track and remove `@preconcurrency` imports after upstream annotations/migration.
2. Methods that should bypass actor isolation for immutable/constant members:
- mark specific APIs `nonisolated` when they do not touch actor-isolated mutable state.
- Example:
```swift
actor CacheNamespaceStore {
    private let namespace: String

    init(namespace: String) {
        self.namespace = namespace
    }

    // Without `nonisolated`, synchronous call sites must use `await`,
    // and protocol requirements that are non-async cannot be satisfied.
    nonisolated func cacheNamespace() -> String {
        namespace // immutable value; safe to expose without actor hop
    }
}
```
3. UI migration hotspots requiring guaranteed main-actor context:
- isolate types or methods with `@MainActor`,
- use `MainActor.assumeIsolated { ... }` only for proven main-thread contexts and only as a short-term escape hatch. **Warning:** `assumeIsolated` crashes at runtime if called off the main thread — it is not a safe fallback, only a precondition assert.
4. Types you do not own and cannot make `Sendable`:
- avoid crossing concurrency domains with that type directly,
- introduce sendable DTO/value snapshots at boundaries,
- use actor confinement to contain non-sendable references.
5. Last-resort escape hatch:
- `@unchecked Sendable` only with explicit invariants documented and tests covering concurrent access.
6. Property-level isolation escape hatch:
- use `nonisolated(unsafe)` only for narrow legacy interop cases where isolation cannot be expressed directly; document invariants and add tracking for removal.
- choose by problem type:
  - Property-level case: use `nonisolated(unsafe)` for specific stored properties that must be reachable from nonisolated contexts (for example a legacy delegate handle), and keep them write-once/read-only after initialization.
  - Type-level case: use `@unchecked Sendable` for whole types that must cross concurrency domains when you can prove thread-safety invariants.
- Example:
```swift
final class LegacySDKWrapper {
    // Swift 6 error without `nonisolated(unsafe)`: stored property of non-Sendable
    // type 'SDKHandle' cannot be used from nonisolated/Sendable contexts.
    // Written once at init, read-only after; removal tracked in issue #123.
    nonisolated(unsafe) let handle: SDKHandle

    init(handle: SDKHandle = SDKHandle()) {
        self.handle = handle
    }
}
```
7. Region-based isolation transfer option (Swift 6):
- when passing a non-`Sendable` value across isolation boundaries as a one-way ownership transfer, prefer `sending` before `@unchecked Sendable` or broad DTO duplication.
- Example:
```swift
actor Processor {
    func ingest(_ payload: sending LegacyPayload) async {
        // payload ownership transferred here; caller must not use it afterward.
    }
}
```
- Use this only when post-send aliasing is not required; otherwise keep actor confinement or DTO snapshots.

### Phase 4 Exit Criteria

1. All strict-concurrency diagnostics in migrated scope are resolved or explicitly waived with tracked rationale.
2. Every `@preconcurrency` import has an issue/ticket for removal.
3. Every `@unchecked Sendable` site has documented invariants and concurrency tests.
4. Every `nonisolated(unsafe)` site has a tracked removal ticket plus documented write-once/read-only invariants.
5. Main-thread UI updates are actor-isolated (`@MainActor` or equivalent) with no ad-hoc thread hopping.
6. Non-`Sendable` transfer sites are evaluated for `sending` adoption (or have documented rationale when `sending` is not applicable).

## Phase 5: Rx Removal

1. Remove remaining `import RxSwift` / `import RxCocoa` / `import RxRelay` / `import RxBlocking` / `import RxTest` / `import RxDataSources` / `import RxGesture` / `import RxKeyboard` / `import Action` / `import RxAction` (also check test targets).
2. Delete compatibility adapters and obsolete extensions.
3. Remove Rx dependencies from package/project manifests.
4. Audit the full dependency graph for transitive Rx re-introduction:
- SPM: run `swift package show-dependencies` and confirm no RxSwift/RxCocoa/RxRelay/RxDataSources/RxGesture/RxKeyboard/Action nodes remain.
- SPM: check root `Package.resolved` (if present) for lingering Rx/Action packages.
- Xcode: check `<Project>.xcodeproj/project.xcworkspace/xcshareddata/swiftpm/Package.resolved` (or `<Workspace>.xcworkspace/xcshareddata/swiftpm/Package.resolved`) and framework link phases.
- CocoaPods: check `Podfile.lock` for RxSwift/RxCocoa/RxRelay/RxBlocking/RxTest/RxDataSources/RxGesture/RxKeyboard/Action entries.
- Carthage: check `Cartfile.resolved` for RxSwift/RxCocoa/RxRelay/RxDataSources/RxGesture/RxKeyboard/Action entries.
5. Run full tests and smoke checks.
- Smoke-check means launching the app on simulator/device and manually exercising critical user journeys (login, primary navigation, one write flow, one error flow, one retry flow).
- Validate command-driven UI behavior specifically: button enabled/disabled transitions, loading indicators, and surfaced errors.
6. If the project uses SwiftLint (or another Swift linter), run it and resolve lint violations introduced by migration changes.

## Rollback Strategy

1. Keep migration slices small (feature by feature).
2. Feature-flag risky rewrites where behavior changes are possible.
3. Preserve old implementation behind adapter until replacement is proven.

## Definition of Done

1. Zero Rx imports in production targets and test targets (including `import RxRelay`, `import RxBlocking`, `import RxTest`, `import RxDataSources`, `import RxGesture`, and `import RxKeyboard`), and zero `import Action` / `import RxAction` in migrated scope.
2. No strict-concurrency warnings in migrated modules.
3. Equivalent behavior validated by tests and manual acceptance checks.
4. Dependency graph no longer includes RxSwift/RxCocoa/RxRelay/RxDataSources/RxGesture/RxKeyboard/Action in any target (verified via `swift package show-dependencies`, Xcode resolved packages, and lockfiles for CocoaPods/Carthage when used).
5. Replaced `Driver`/`Signal` sites preserve documented guarantees (main delivery, no error, and replay behavior where required), validated by automated tests for replay/no-error/main-delivery behavior.
6. All `Action`/`RxAction` call sites are replaced and validated by tests for command guarantees (single-flight or declared trigger policy, error surfacing, and lifecycle cancellation behavior).
7. On iOS 17+ / macOS 14+ targets, each eligible simple binding site has a documented adopt/reject decision for `@Observable`/`withObservationTracking` (with rationale) and behavior-test coverage.
   Eligibility criteria: `@Observable` model available (or explicitly migrated), deployment target supports Observation, and the binding primarily observes a small synchronous property set.
8. Command owners include explicit `invalidate()` lifecycle call sites (for both `AsyncCommand` and `CommandQueue` when used), or documented fire-and-forget teardown from `deinit`, so command/queue streams finish and do not hang consumers.

## Testing Strategy During Migration

1. Keep behavior tests at boundaries before and after each migration slice.
2. For async replacements, prefer async tests (`async` XCTest methods or Swift Testing `@Test` async functions) over callback-only assertions.
3. Use `XCTestExpectation` only for callback/delegate boundaries that cannot yet be expressed as async sequences.
4. Preserve deterministic time-based tests by injecting time/scheduler abstractions:
- use the `Clock` protocol for injectable time (`SuspendingClock` for user-perceived timers like debounce/backoff, `ContinuousClock` for elapsed-time behavior that should not pause during device sleep, manual/test doubles in tests),
- replace direct `Task.sleep` calls with clock wrappers,
- make debounce/throttle intervals injectable,
- avoid real-time sleeps in unit tests.
- minimal test-double sketch:
```swift
protocol SleepClocking: Sendable {
    func sleep(for duration: Duration) async throws
}

struct LiveSleepClock: SleepClocking {
    private let base = SuspendingClock()
    func sleep(for duration: Duration) async throws {
        try await base.sleep(for: duration)
    }
}

actor TestSleepClock: SleepClocking {
    private var waiters: [CheckedContinuation<Void, Never>] = []

    func sleep(for duration: Duration) async throws {
        await withCheckedContinuation { waiters.append($0) }
    }

    func advance() {
        guard !waiters.isEmpty else { return }
        waiters.removeFirst().resume()
    }
}
```
- caveat: call `advance()` only after confirming the task-under-test is suspended in `sleep(for:)` (for example with an `AsyncLatch`/entry signal), otherwise the advance can be dropped and the test may hang.
- caveat: `advance()` is duration-blind in this sketch and resumes waiters FIFO by registration order, not by requested `Duration`; if duration ordering matters, prefer a real test clock.
- caveat: this sketch uses `Duration`/`SuspendingClock` (Swift 5.7+ / iOS 16+). For iOS 15-era targets, use a `TimeInterval`-based test clock abstraction or a `DispatchQueue.asyncAfter` wrapper instead.
- alternative: if your project already uses `pointfreeco/swift-clocks`, use its `TestClock` for deterministic time advancement.
5. Replace Rx `TestScheduler` intent, not syntax:
- assert ordering, cancellation, and terminal events with controlled clocks or manual async stream continuations,
- validate race/timeout behavior with deterministic task orchestration.
6. Replace Rx-style mocks/stubs with async-aware test doubles:
- change protocols from stream-returning methods to `async`/`async throws` methods where appropriate,
- replace `Observable.just(...)` stubs with direct async returns for single-result operations,
- for multi-event behavior, use controlled `AsyncStream`/`AsyncThrowingStream` continuations in tests.
7. Remove Rx from `@testable`-importing test targets:
- replace Rx-based test-only stubs/helpers used with `@testable import` modules,
- ensure no `import RxSwift` / `import RxCocoa` / `import RxRelay` / `import RxBlocking` / `import RxTest` / `import RxDataSources` / `import RxGesture` / `import RxKeyboard` / `import Action` / `import RxAction` remain in those test targets.
8. Add explicit tests for `Action` replacements (`AsyncCommand` or equivalent):
- assert single-flight behavior (second trigger while executing is ignored/rejected by design),
- assert `isExecuting` transitions (`false -> true -> false`) around execution by consuming `states()` (avoid sleep/polling),
- assert failures are emitted to the command error channel and UI observers receive them.
- error-channel test sketch:
```swift
enum MyError: Error { case example }
let command = AsyncCommand<String, String> { _ in
    throw MyError.example
}

let errors = await command.errors()
var errIt = errors.makeAsyncIterator()
let run = Task { try await command.execute("input") }
let received = await errIt.next()
// assert received matches expected domain error
_ = await run.result
```
- assert cancellation behavior: when lifecycle cancellation fires, the in-flight command task is canceled and no stale UI update is applied after dismissal.
- cancellation test technique: cancel the caller-owned task, await its result (`_ = await run.result`), then assert the final observed command state and completion behavior (for example `isExecuting == false` followed by stream termination where `await it.next() == nil` after teardown). Stream termination in this pattern is caused by `invalidate()` finishing continuations, not by task cancellation alone.
- for deterministic ordering, synchronize `work` entry with a controlled continuation signal before asserting `started`; do not rely on `Task.yield()`.
- example pattern:
```swift
actor AsyncLatch {
    private var isOpen = false
    private var waiters: [CheckedContinuation<Void, Never>] = []

    func wait() async {
        if isOpen { return }
        await withCheckedContinuation { waiters.append($0) }
    }

    func open() {
        guard !isOpen else { return }
        isOpen = true
        for waiter in waiters {
            waiter.resume()
        }
        waiters.removeAll()
    }
}

let enteredLatch = AsyncLatch()
let releaseLatch = AsyncLatch()

let command = AsyncCommand<String, String> { input in
    await enteredLatch.open()
    await releaseLatch.wait()
    return input
}

let states = await command.states()
var it = states.makeAsyncIterator()
let initial = await it.next()
#expect(initial?.isExecuting == false)

let input = "testInput"
let enteredWait = Task { await enteredLatch.wait() }
let run = Task { try? await command.execute(input) }
_ = await enteredWait.result // deterministic: work has started
let started = await it.next()
#expect(started?.isExecuting == true)

await releaseLatch.open()
_ = await run.result
let ended = await it.next()
#expect(ended?.isExecuting == false)
```
9. Define concurrency tests for each `@unchecked Sendable` site:
- run concurrent read/read and read/write access patterns with `withTaskGroup` (or equivalent) against realistic call paths,
- assert documented invariants under contention (ordering, no torn state, no lost updates),
- repeat the stress scenario enough times to increase race-window coverage.
- run these tests under Thread Sanitizer (Xcode Scheme -> Diagnostics -> Thread Sanitizer) to surface races that may not fail on every run.
