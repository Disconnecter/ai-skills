# RxSwift/Combine/Concurrency Mapping

## Source Anchors

- RxSwift Traits (repo path): https://github.com/ReactiveX/RxSwift/blob/main/Documentation/Traits.md
- RxSwift Concurrency Bridge (6.5+): https://github.com/ReactiveX/RxSwift/blob/main/Documentation/SwiftConcurrency.md
- Swift Language Guide (Concurrency): https://docs.swift.org/swift-book/documentation/the-swift-programming-language/concurrency/
- Action (RxSwiftCommunity): https://github.com/RxSwiftCommunity/Action

## Type Mapping

| RxSwift | Combine | Swift Concurrency |
| --- | --- | --- |
| `Observable<Element>` | `AnyPublisher<Element, Error>` | consume via `for try await value in observable.values` |
| `Single<Element>` | one-shot `AnyPublisher<Element, Error>` | `try await single.value` |
| `Completable` | representation option: `AnyPublisher<Void, Error>` ⚠️ `Completable` emits no elements; this publisher type can emit `Void` values (stricter completion-only shape is closer to `AnyPublisher<Never, Error>`) | `try await completable.value` |
| `Maybe<Element>` | representation option: `AnyPublisher<Element, Error>` constrained to at most one output (for example, `prefix(1)` or custom wrapper) ⚠️ completion-without-emission is not `nil` | `try await maybe.value` (returns `nil` only in the async bridge; treat as boundary adaptation, not a semantic identity) |
| `Infallible<Element>` | `AnyPublisher<Element, Never>` | `for await value in infallible.values` |
| `PublishSubject<Element>` | `PassthroughSubject<Element, Error>` | `AsyncThrowingStream` continuation |
| `BehaviorSubject<Element>` | `CurrentValueSubject<Element, Error>` | actor state + async getter/stream |
| `ReplaySubject<Element>` | no built-in replay-N subject; use `multicast` with a custom replay buffer/replaying subject (or third-party equivalent) | actor-owned ring buffer + `AsyncStream` replay-on-subscribe bridge |
| `PublishRelay<Element>` | `PassthroughSubject<Element, Never>` | event-only `AsyncStream` continuation |
| `BehaviorRelay<Element>` | `CurrentValueSubject<Element, Never>` | actor-backed state + `AsyncStream` snapshot/event bridge |
| `Driver<Element>` | `AnyPublisher<Element, Never>` + main-delivery + replay latest | `for await value in driver.values` with explicit `@MainActor` UI hop (`driver.values` bridge does not enforce main delivery) |
| `Signal<Element>` | `AnyPublisher<Element, Never>` + main-delivery + no replay | `for await value in signal.values` with explicit `@MainActor` UI hop (`signal.values` bridge does not enforce main delivery) |
| `Action<Input, Element>` | no built-in equivalent; compose a command type with `isEnabled`, `isExecuting`, and `errors` plus single-flight execute semantics | no built-in equivalent; prefer actor-backed `AsyncCommand` with `execute(_:) async throws -> Output` (Rx `Element` maps to blueprint `Output`), explicit gate errors, and an optional async error stream for UI observation |

## Trait Guarantees to Preserve

- `Driver` must preserve three guarantees: no error, observe on main scheduler, and shared side effects with replay latest (`share(replay: 1, scope: .whileConnected)`).
- `Signal` must preserve: no error, main scheduler delivery, shared side effects without replay (`share(scope: .whileConnected)`).
- `Combine` `.share()` alone is not equivalent to `Driver` replay-latest semantics; use explicit replay strategy (for example `multicast` with a replaying/current-value subject) and verify connection lifetime behavior.
- `multicast` replay strategies require explicit `.connect()` lifecycle management. Store the returned connection `Cancellable` in lifecycle-owned storage (`Set<AnyCancellable>` or a retained property), or releasing it early will terminate upstream sharing.
- If upstream restart on re-subscription is not acceptable (stateful pipelines, `Driver`-equivalent sharing), use manual `.connect()` with an owner-held connection.
- Use `.autoconnect()` only when side effects are idempotent/cheap and restart is harmless; it disconnects when the last subscriber cancels and reconnects later, restarting upstream work. A replaying subject can keep the last emitted value, but upstream side effects still restart.
- Do not replace `Driver`/`Signal` with plain publisher/async streams without reintroducing these guarantees explicitly.
- `Maybe` maps are easy to get wrong in Combine: completion without emission is not identical to emitting `nil`. Treat optional-output mapping as an explicit, lossy boundary choice.
- For `Maybe<Optional<T>>`, the async bridge returns `T??` (`nil` can mean "no emission", `.some(nil)` means "emitted nil"). Use an explicit wrapper type when semantic parity matters.

Example `Driver`-like construction in Combine (replay latest + main delivery + no failure):

```swift
@MainActor
final class DriverLikeHost {
    private var sharingConnection: AnyCancellable?
    let driverLike: AnyPublisher<String, Never>

    init(upstream: AnyPublisher<String, Error>) {
        let connectable = upstream
            .receive(on: DispatchQueue.main)
            // Tradeoff: maps upstream errors to completion (no error escapes),
            // so the stream stops producing future values after the first error.
            .catch { _ in Empty<String, Never>() }
            .map(Optional.some)
            .multicast(subject: CurrentValueSubject<String?, Never>(nil))

        self.driverLike = connectable
            .compactMap { $0 } // drop the required initial nil sentinel
            .eraseToAnyPublisher()

        self.sharingConnection = connectable.connect() // retain for sharing lifetime
    }
}
```

`CurrentValueSubject` requires an initial value (`nil` sentinel above), unlike Rx `Driver` which has no mandatory initial emission.
For `ObservableObject`-based ViewModels, `@Published` can also act as replay-latest transport, with the same initial-value tradeoff.
Releasing `sharingConnection` cancels upstream sharing; existing `driverLike` subscribers receive no further values.
The example's `.catch { Empty() }` satisfies "no error" by converting failures into completion; if you need post-error recovery, replace it with an explicit fallback/retry strategy.
In Swift 6 strict mode, accessing `driverLike` from non-`@MainActor` contexts crosses actor isolation; verify your SDK's Combine `Sendable` conformances (or keep usage main-actor confined) to avoid migration-time Sendability errors.

## Lifecycle Mapping

| RxSwift | Combine | Swift Concurrency |
| --- | --- | --- |
| `DisposeBag` | `Set<AnyCancellable>` | `Task` handle(s) + cancellation |
| `.disposed(by:)` | `.store(in:)` | hold task and call `cancel()` |
| `CompositeDisposable` | grouped `AnyCancellable` set + explicit cancel-all helper | cancellation bag object that cancels all child tasks |
| `SerialDisposable` | single replaceable `AnyCancellable?`; replacement-cancel requires explicit `oldValue?.cancel()` before overwrite (for example via `didSet`) | single replaceable `Task<Void, Never>?`; replacement-cancel requires explicit `oldTask?.cancel()` before overwrite |
| subscription disposal on deinit | cancellables released on deinit | task cancellation in `deinit` or owner lifecycle |

## Operator Equivalents (Common Cases)

| RxSwift | Combine | Concurrency Notes |
| --- | --- | --- |
| `.map` | `.map` | direct equivalent |
| `.flatMapLatest` | `.map { ... }.switchToLatest()` | in async code, cancel previous `Task` before starting new |
| `.flatMap` | `.flatMap` | concurrency equivalent often `withThrowingTaskGroup` |
| `.filter` | `.filter` | direct equivalent |
| `.debounce` | `.debounce` | async equivalent: cancel the previous sleep task on each new event to restart the countdown; run work only when sleep completes |
| `.throttle` | `.throttle` | async equivalent requires custom gate logic |
| `.distinctUntilChanged()` | `.removeDuplicates()` | async equivalent compare before yield |
| `.combineLatest` | `.combineLatest` | async equivalent custom combiner/actor state |
| `.zip` | `.zip` | async equivalent run two async ops then tuple |
| `.merge` | `.merge` / `Publishers.MergeMany` | async equivalent `withTaskGroup` and emit as children complete |
| `.amb` / race-first | custom first-emission publisher | async equivalent `withTaskGroup` + first completed child wins, cancel others, then drain/discard remaining child results; on Swift 5.9+ prefer `withThrowingDiscardingTaskGroup` when applicable |
| `.catchError` | `.catch` | async equivalent `do/catch` |
| `.retry` | `.retry` | async equivalent loop with retry/backoff |
| `.timeout` | `.timeout` | async equivalent race operation vs `Task.sleep`, cancel loser |
| `.scan` | `.scan` | async equivalent: accumulate into actor or local variable across iterations |
| `.buffer` | approximate mapping only: `.collect(.byTime(...))` or `.collect(.byTimeOrCount(...))` | async equivalent is approximate: accumulate into array over timed window using `AsyncStream` + actor buffer; verify overflow/backpressure semantics per call site |
| `.window` | approximate mapping only: custom publisher splitting upstream into inner publishers | async equivalent is approximate: partition into `AsyncStream` per window using actor-managed continuation handoff; verify boundary/closure semantics per call site |
| `.observeOn(MainScheduler.instance)` | `.receive(on: DispatchQueue.main)` | prefer `@MainActor` for UI mutation |
| `.subscribeOn` | `.subscribe(on:)` | async equivalent should usually remain structured tasks instead of detached tasks |

## UI Binding Guidance

- Replace `bind(to:)` with explicit state mutation functions, `assign(to:on:)`, or `@MainActor` state updates.
- Keep ViewModel output transport-specific at boundaries; do not leak Rx-specific traits into domain layer.
- Use `@MainActor` on ViewModel methods mutating published UI state.
- For UIKit bindings, replace `.rx.*` binders with target-action, delegate, diffable data source updates, and explicit `@MainActor` state update methods.
- On iOS 17+ / macOS 14+ with `@Observable` types, `withObservationTracking` can replace simple RxCocoa bindings that primarily observe a small synchronous property set; prefer this over Combine for new UIKit code on supported OS versions.
- `withObservationTracking` requires Observation-compatible types; migrate `ObservableObject`/`@Published` models to `@Observable` first where adoption is intended.
- `withObservationTracking` is single-fire for `onChange`; schedule re-registration from `onChange` to keep observing.
- `onChange` does not guarantee main-actor execution. Hop back to `@MainActor` (for example via `Task { @MainActor in ... }`) before mutating UI state.
- Keep all properties you want tracked inside the apply closure. Moving reads (for example `render(...)`) into `onChange` will not register tracking for future updates.
- Observation tracking is registered by reads of `@Observable`-backed properties in the apply closure. Passing a value snapshot to a helper and reading sub-properties from that copy does not register additional observation dependencies.
- `assign(to:on:)` retains `on` strongly; if the target object also owns the upstream publisher, this creates a retain cycle. Prefer `assign(to: &$published)` (iOS 14+) for `@Published` targets or use `sink` with `[weak self]`. In `@MainActor` ViewModel pipelines, `assign(to: &$published)` is also preferred because updates stay actor-isolated without an extra `receive(on: DispatchQueue.main)` hop.
- Bridge pattern (`@Observable` -> Combine): publish tracked snapshots into a subject, then expose `AnyPublisher` for legacy Combine consumers.
Observation loop example (`withObservationTracking` re-registration):

```swift
@MainActor
func bindObservation() {
    withObservationTracking {
        render(viewModel.state)
    } onChange: { [weak self] in
        Task { @MainActor in
            self?.bindObservation() // re-subscribe for next change
        }
    }
}
```

Bridge example (`@Observable` -> Combine publisher):

```swift
@MainActor
final class ObservationPublisherBridge {
    private let subject = CurrentValueSubject<ViewState, Never>(.initial)
    private let viewModel: ViewModel // ViewModel must be `@Observable` (Observation framework), not only `ObservableObject`.

    var publisher: AnyPublisher<ViewState, Never> { subject.eraseToAnyPublisher() }

    init(viewModel: ViewModel) {
        self.viewModel = viewModel
        track()
    }

    private func track() {
        withObservationTracking {
            subject.send(viewModel.state)
        } onChange: { [weak self] in
            Task { @MainActor in self?.track() }
        }
    }
}
```

## Caveats

- `Driver` guarantees (main scheduler, no error, replay latest) are not automatic in Combine.
- Some Rx operators have no exact Combine equivalent; preserve semantics via small custom publishers or explicit async orchestration.
- Avoid converting every stream to `AnyPublisher` too early; keep concrete types internally when possible, erase at boundaries.
- `observable.values` iteration suspends until completion; for never-ending streams, structure cancellation explicitly (for example iterate in a caller-owned `Task` and cancel it on lifecycle teardown, bound with operators like `prefix(_:)` before bridging, or wrap multiple loops in a task group with explicit cancellation).
- Detached tasks are not a transparent replacement for Rx schedulers because they do not inherit parent isolation/priority/task-local context.

## Companion Library Notes

- `RxCocoa`: plan explicit replacement of `.rx.*` binders and control events (`.rx.tap`, `.rx.text`, `.rx.items`) with Combine publishers or async handlers.
- `RxRelay`: migrate `PublishRelay`/`BehaviorRelay` to `Never`-failing subjects or actor-owned state/event channels.
- `RxDataSources`: migrate to `UITableViewDiffableDataSource` / `UICollectionViewDiffableDataSource` (or SwiftUI lists) and preserve section/item identity behavior. Reproduce `AnimatableSectionModelType`/`IdentifiableType` contracts with stable `Hashable` identity and equality rules for snapshot items.
- Ensure equality and hashing are consistent for diffing: inconsistent `Equatable`/`Hashable` contracts cause incorrect diff animations and update glitches.
- Preserve section-level identity separately from item identity when mapping multi-section snapshots; section identity drift breaks animated section updates.
- Example (`RxDataSources` -> diffable):
```swift
enum Section: Hashable { case main(UUID) }
struct Row: Hashable {
    let id: UUID
    let title: String

    // Preserve stable item identity for diffing; content changes should be
    // handled via snapshot reconfigure/reload instead of identity churn.
    // iOS 15+: `snapshot.reconfigureItems([row])`; iOS 13-14: `snapshot.reloadItems([row])`.
    static func == (lhs: Row, rhs: Row) -> Bool { lhs.id == rhs.id }
    func hash(into hasher: inout Hasher) { hasher.combine(id) }
}

let dataSource = UITableViewDiffableDataSource<Section, Row>(tableView: tableView) { tableView, indexPath, row in
    let cell = tableView.dequeueReusableCell(withIdentifier: "Cell", for: indexPath)
    cell.textLabel?.text = row.title
    return cell
}

var snapshot = NSDiffableDataSourceSnapshot<Section, Row>()
snapshot.appendSections([.main(sectionID)])
snapshot.appendItems(rows, toSection: .main(sectionID))
dataSource.apply(snapshot, animatingDifferences: true)
```
- `RxGesture`: replace with UIKit/AppKit gesture recognizers (`UITapGestureRecognizer`, etc.) using gesture target-action wiring, and keep explicit `@MainActor` event routing.
- `RxKeyboard`: on iOS 15+, prefer `UIKeyboardLayoutGuide` for layout-driven keyboard handling; use keyboard notifications only when you need animation/frame event details for custom behavior.
- Example (`RxGesture` -> UIKit):
```swift
let tap = UITapGestureRecognizer(target: self, action: #selector(handleTap))
view.addGestureRecognizer(tap)
```
- Example (`RxKeyboard` -> `UIKeyboardLayoutGuide`):
```swift
if #available(iOS 15.0, *) {
    inputBar.bottomAnchor.constraint(equalTo: view.keyboardLayoutGuide.topAnchor).isActive = true
}
```
- `RxAction` / `Action`: no framework replacement exists; hand-roll a command type or use a plain `async` method. Explicitly reproduce `isExecuting`, `isEnabled`, and `errors` state that `Action` provided automatically.
- `RxBlocking` (tests): replace `.toBlocking().first()` / `.toArray()` assertions with async test functions (`async` XCTest or Swift Testing `@Test`) and use `XCTestExpectation` only when async refactoring is not yet possible.
- `RxTest` (tests): replace `TestScheduler`/`TestableObserver` style tests with injected `Clock`-based timing and controlled `AsyncStream`/`AsyncThrowingStream` continuations (see playbook Testing Strategy items 4-6).

## Action Replacement Blueprint

`Action` replacement is not time-based debounce. The essential guarantee is single-flight execution (do not start a second run while one is active), plus `isExecuting`, `isEnabled`, and `errors` state.
Platform note: `AsyncStream.makeStream` used below requires Swift 5.9+ / iOS 17+. For earlier targets, use `AsyncStream(bufferingPolicy:_:)` and capture the continuation from the initializer closure.

```swift
import Foundation

enum CommandGateError: Error {
    case disabled
    case alreadyExecuting
    case invalidated
}

actor AsyncCommand<Input: Sendable, Output: Sendable> {
    struct State: Sendable {
        let isEnabled: Bool
        let isExecuting: Bool
    }

    typealias Work = @Sendable (Input) async throws -> Output

    private var isExecuting = false
    private var isEnabled = true
    private var isInvalidated = false
    private let work: Work
    private var errorContinuations: [UUID: AsyncStream<Error>.Continuation] = [:]
    private var stateContinuations: [UUID: AsyncStream<State>.Continuation] = [:]
    private var isEnabledContinuations: [UUID: AsyncStream<Bool>.Continuation] = [:]

    init(isEnabled: Bool = true, work: @escaping Work) {
        self.isEnabled = isEnabled
        self.work = work
    }

    func setEnabled(_ enabled: Bool) {
        guard !isInvalidated else { return }
        isEnabled = enabled
        emitState()
        emitIsEnabled()
    }

    func state() -> (isEnabled: Bool, isExecuting: Bool) {
        (isEnabled, isExecuting)
    }

    func states() -> AsyncStream<State> {
        let id = UUID()
        // Keeps latest snapshot only; slow consumers can miss intermediate transitions.
        let (stream, continuation) = AsyncStream<State>.makeStream(
            bufferingPolicy: .bufferingNewest(1)
        )
        stateContinuations[id] = continuation
        continuation.yield(currentState())
        continuation.onTermination = { [weak self] _ in
            // If self is alive, remove this continuation from storage.
            // If self is nil, the actor is already deallocated and storage is gone.
            Task { await self?.removeStateContinuation(id: id) }
        }
        return stream
    }

    func isEnabledStates() -> AsyncStream<Bool> {
        // WARNING: emits only explicit setEnabled changes.
        // For button gating during execution, derive from states():
        // state.isEnabled && !state.isExecuting
        let id = UUID()
        let (stream, continuation) = AsyncStream<Bool>.makeStream(
            bufferingPolicy: .bufferingNewest(1)
        )
        isEnabledContinuations[id] = continuation
        continuation.yield(isEnabled)
        continuation.onTermination = { [weak self] _ in
            // If self is alive, remove this continuation from storage.
            // If self is nil, the actor is already deallocated and storage is gone.
            Task { await self?.removeIsEnabledContinuation(id: id) }
        }
        return stream
    }

    func errors() -> AsyncStream<Error> {
        let id = UUID()
        let (stream, continuation) = AsyncStream<Error>.makeStream(
            bufferingPolicy: .bufferingNewest(1)
        )
        errorContinuations[id] = continuation
        continuation.onTermination = { [weak self] _ in
            // If self is alive, remove this continuation from storage.
            // If self is nil, the actor is already deallocated and storage is gone.
            Task { await self?.removeErrorContinuation(id: id) }
        }
        return stream
    }

    func execute(_ input: Input) async throws -> Output {
        try Task.checkCancellation()
        var didBegin = false
        try beginExecution()
        didBegin = true
        defer {
            if didBegin { endExecution() }
        }

        do {
            try Task.checkCancellation()
            return try await work(input)
        } catch let error as CancellationError {
            throw error
        } catch {
            for continuation in errorContinuations.values {
                continuation.yield(error)
            }
            throw error
        }
    }

    func invalidate() {
        isInvalidated = true
        isEnabled = false
        emitState()
        emitIsEnabled()

        for continuation in errorContinuations.values {
            continuation.finish()
        }
        errorContinuations.removeAll()

        for continuation in stateContinuations.values {
            continuation.finish()
        }
        stateContinuations.removeAll()

        for continuation in isEnabledContinuations.values {
            continuation.finish()
        }
        isEnabledContinuations.removeAll()
    }

    private func beginExecution() throws {
        // Keep gate checks + state flip in one actor turn (no await) to avoid TOCTOU.
        guard !isInvalidated else { throw CommandGateError.invalidated }
        guard isEnabled else { throw CommandGateError.disabled }
        guard !isExecuting else { throw CommandGateError.alreadyExecuting }
        isExecuting = true
        emitState()
    }

    private func endExecution() {
        isExecuting = false
        emitState()
    }

    private func removeErrorContinuation(id: UUID) {
        errorContinuations.removeValue(forKey: id)
    }

    private func removeStateContinuation(id: UUID) {
        stateContinuations.removeValue(forKey: id)
    }

    private func removeIsEnabledContinuation(id: UUID) {
        isEnabledContinuations.removeValue(forKey: id)
    }

    private func currentState() -> State {
        State(isEnabled: isEnabled, isExecuting: isExecuting)
    }

    private func emitState() {
        let snapshot = currentState()
        for continuation in stateContinuations.values {
            continuation.yield(snapshot)
        }
    }

    private func emitIsEnabled() {
        for continuation in isEnabledContinuations.values {
            continuation.yield(isEnabled)
        }
    }
}
```

Call `execute` from a caller-owned `Task` and keep the handle so you can cancel in-flight work on lifecycle events (`viewWillDisappear`, `deinit`, coordinator pop). If you already use Combine/SwiftUI observation, a class-based `ObservableObject` or `@Observable` command wrapper is a valid UI-facing variant.
Use `state()` only for point-in-time checks (for example, unit-test assertions); use `states()` for UI observation to avoid stale snapshot reads.
`states()` uses `.bufferingNewest(1)`: slow consumers may miss intermediate transitions (for example `isExecuting` `true` quickly followed by `false`). Consume promptly for loading UI, or modify the blueprint buffering policy (for example `.bufferingOldest(N)` or `.unbounded`) when transition loss is unacceptable.
WARNING: `isEnabledStates()` emits only explicit `setEnabled` changes; it does not encode execution lifecycle. Do not bind button enabled state directly to this stream when execution should disable input. Derive UI gating from `states()` (`state.isEnabled && !state.isExecuting`).
Because actors are re-entrant, a point-in-time `state()` read can transiently observe `isExecuting == true` around cancellation boundaries before deferred reset is observed; prefer `states()` transition assertions for behavioral checks.
Call `await command.invalidate()` in an explicit lifecycle hook before releasing the command owner so `states()`/`errors()` streams finish cleanly. Because `deinit` cannot `await`, fallback from `deinit` must use `Task { await command.invalidate() }` as fire-and-forget cleanup.
`invalidate()` does not cancel an already-running `execute` call. Cancel the caller-owned task handle separately when teardown should stop in-flight work.
Recommended teardown order: cancel the in-flight task handle first, let it unwind if feasible (`await task.result`), then call `invalidate()`. This preserves a final `isExecuting == false` transition while streams are still active.
If you call `invalidate()` mid-flight before task unwind, observers can see `State(isEnabled: false, isExecuting: true)` as the last emitted value before stream completion.
After `invalidate()`, new `execute` calls are rejected with `CommandGateError.invalidated`.
`setEnabled(_:)` becomes a no-op after `invalidate()`.
For latest-wins behavior, keep a replaceable task handle at the caller and cancel it before starting a new `execute`.
Because cancellation is cooperative, the canceled run may still be unwinding while `isExecuting == true`, so the next `execute` can throw `CommandGateError.alreadyExecuting`.
Handle this race explicitly: either accept the drop (treat `alreadyExecuting` like `CancellationError`), or await previous-task completion before launching the replacement (or retry after a yield/backoff) when strict latest-wins execution is required.
`work` implementations should rethrow `CancellationError` (do not map it into domain errors) so cancellation stays out of the user-error channel.
If `work` captures the owning ViewModel/ViewController that also stores the command, capture `[weak self]` (or route through an isolated service) to avoid a retain cycle.
If `work` performs multi-step or looped processing, it should call `Task.checkCancellation()` between steps. When bridging callback-based APIs, wrap cancellation cleanup with `withTaskCancellationHandler(operation:onCancel:)`.
If legacy models are not yet `Sendable`, keep the command and its call path actor-confined (`@MainActor` or dedicated actor), introduce sendable DTO snapshots at boundaries, and use `@unchecked Sendable` only as a temporary documented exception.
`errors()` uses `.bufferingNewest(1)`: if multiple errors are produced before consumption, older errors are dropped. To avoid this, modify the blueprint buffering policy (for example `.bufferingOldest(N)` or `.unbounded`) and/or consume faster.
`errors()` is hot and non-replaying: a new subscriber receives only errors emitted after subscription, not past errors.

## Bridge Patterns (Documented in RxSwift 6.5+)

1. Rx -> async sequence:
```swift
for try await value in observable.values { ... }
```
Use explicit cancellation for potentially infinite streams; code after `for try await` runs only when the observable completes normally or exits by throwing.
2. Async sequence -> Rx:
```swift
let stream = AsyncStream<Int> { continuation in ... }
let observable = stream.asObservable()
```
For never-ending async streams, pair with explicit cancellation and lifecycle ownership.
3. Async result -> Rx `Single`:
```swift
let single = Single.create {
    try await work()
}
```
Requires RxSwift 6.5+ (`Single.create` async-closure overload).
