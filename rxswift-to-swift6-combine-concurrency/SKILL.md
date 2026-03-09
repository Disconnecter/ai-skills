---
name: rx-migrate
description: Migrate RxSwift-based Swift codebases to Swift 6-ready Combine and modern Swift Concurrency patterns (async/await, Task groups, actors, Sendable). Use when tasks involve replacing Observable/Single/Completable/Maybe/Infallible/Subject/BehaviorRelay/PublishRelay/Driver/Signal/Action chains, removing Rx dependencies, applying RxSwift 6.5 async bridges (`values`, primitive `.value`, `asObservable()`), fixing strict-concurrency diagnostics, migrating RxAction/Action command patterns, or executing phased migration plans with temporary compatibility adapters.
---

# RxSwift To Swift 6 Combine Concurrency

## Overview

Migrate existing RxSwift architecture to Swift 6-safe async/await and/or Combine without breaking behavior. Prefer incremental, test-backed changes and use short-lived adapter layers to keep rollout safe.

Follow official guarantees from RxSwift traits and Swift Concurrency semantics instead of approximating behavior.

## Workflow Decision

1. Choose migration target:
- Use `async/await` + `AsyncSequence` for new feature work and call-driven flows.
- Use `Combine` when existing architecture is stream-heavy, operator-centric, or integrates with Apple publisher APIs.
- Use hybrid (Combine at boundaries, async/await in domain/service layer) for large codebases.
- For Action-heavy ViewModels, choose command shape up front: `@MainActor` async command methods/types for UI-driven effects, or Combine-backed command publishers when existing binding architecture is already publisher-centric.
2. Keep behavior parity first; optimize operator style only after tests pass.
3. Remove all Rx/Action dependencies only after all call sites and tests are migrated.
4. Prefer structured concurrency (`async let`, task groups) over detached tasks unless work must run independently of parent cancellation/priority/actor context.

## Execution Workflow

1. Inventory Rx surface area.
- Search for `import RxSwift`, `import RxCocoa`, `import RxRelay`, `import RxBlocking`, `import RxTest`, `import RxDataSources`, `import RxGesture`, `import RxKeyboard`, `import Action`, `import RxAction`, `Observable`, `Single`, `Completable`, `Maybe`, `Infallible`, `Subject`, `BehaviorRelay`, `PublishRelay`, `Driver`, `Signal`, `Action`, `DisposeBag`, `.subscribe`, `.bind`, `.disposed(by:)`, `.execute(`, `.rx.`.
- Search UIKit/Cocoa binding usage (`.rx.text`, `.rx.tap`, `.rx.items`, `.rx.modelSelected`, `.rx.controlEvent`) and RxDataSources usage.
- Group by layer: UI binding, domain orchestration, networking, persistence.
2. Use RxSwift 6.5 bridging as transition glue.
- For throwing streams, iterate with `for try await value in observable.values`.
- For non-throwing traits (`Infallible`, `Driver`, `Signal`), iterate with `for await`.
- For primitive traits, use `try await single.value`, `try await maybe.value` (returns `nil` when `Maybe` completes without emission; see Critical Mapping Snapshot below), `try await completable.value`.
- Wrap existing async sequences with `asObservable()` and async single-result work with `Single.create { try await ... }` (RxSwift 6.5+).
3. Define temporary compatibility boundary.
- Keep adapters at module edges, not scattered through feature code.
- Delete adapters once the destination layer is fully migrated.
4. Migrate by primitive and usage style.
- Use [RxSwift/Combine/Concurrency Mapping](references/rxswift-combine-concurrency-map.md) for one-to-one substitutions.
5. Enforce Swift 6 concurrency safety during migration.
- Add `Sendable` where crossing task/actor boundaries.
- Isolate mutable shared state with actors.
- Mark UI-facing APIs `@MainActor`.
- Use `sending` parameter/return annotations for one-way transfer of non-`Sendable` values when ownership can move safely (region-based isolation), before escalating to `@unchecked Sendable` or heavy DTO copying.
6. Replace disposal lifecycle patterns.
- Convert `DisposeBag` ownership to `Task` cancellation or `AnyCancellable` storage, and map `CompositeDisposable`/`SerialDisposable` sites to grouped cancel-all and replaceable-task patterns (for `SerialDisposable`, explicitly cancel the old value before assigning the new one).
7. Validate and clean up.
- Run tests after each feature migration slice.
- Remove Rx dependencies and dead adapters only when no imports remain.
- Audit `@testable import` test targets and remove Rx-based test helpers/stubs so Rx does not linger in tests.
- Replace `RxBlocking` assertions (`.toBlocking().first()`, `.toArray()`) with async tests (`async` XCTest / Swift Testing) or `XCTestExpectation` at callback-only boundaries.

## Critical Mapping Snapshot

Use this snapshot even if reference files are unavailable.

- `Observable<Element>`: map to `AnyPublisher<Element, Error>` or consume via `for try await value in observable.values`.
- `Single<Element>`: map to `async throws -> Element` (`try await single.value`).
- `Maybe<Element>`: keep "0-or-1 emission then complete" semantics; avoid implicitly converting "no emission" into `nil` unless that is an explicit boundary decision.
- `Infallible<Element>`: map to `AnyPublisher<Element, Never>` or consume via `for await value in infallible.values`.
- `Completable`: map to `async throws -> Void` (`try await completable.value`).
- `PublishSubject<Element>`: map to `PassthroughSubject<Element, Error>` or `AsyncThrowingStream` continuation.
- `BehaviorSubject<Element>`: map to `CurrentValueSubject<Element, Error>` or actor state + async getter/stream.
- `ReplaySubject<Element>`: no built-in replay-N subject in Combine; use `multicast` with a custom replay buffer/replaying subject, or actor-owned ring buffer + `AsyncStream` replay bridge.
- `Driver<Element>`: preserve no-error + main-scheduler delivery + replay latest sharing semantics (async bridge caveat: `driver.values` still needs an explicit `@MainActor` UI hop).
- `Signal<Element>`: preserve no-error + main-scheduler delivery + no replay semantics (async bridge caveat: `signal.values` still needs an explicit `@MainActor` UI hop).
- `BehaviorRelay<Element>`: map to `CurrentValueSubject<Element, Never>` or `@MainActor` state container.
- `PublishRelay<Element>`: map to `PassthroughSubject<Element, Never>` or event-only async stream.
- `Action<Input, Element>` (RxAction): no direct equivalent; replace with `@MainActor` async command methods or a custom command type that preserves single-flight execution (no concurrent execute), `isExecuting`, `isEnabled`, and `errors`.

### Action Replacement Baseline

Use this baseline when replacing `Action`:

1. Expose `execute(_:) async throws -> Output` (or an explicit `Result` return).
2. Apply explicit gate checks with separate failure reasons:
- `guard !isInvalidated else { throw CommandGateError.invalidated }`
- `guard isEnabled else { throw CommandGateError.disabled }`
- `guard !isExecuting else { throw CommandGateError.alreadyExecuting }`
3. Set `isExecuting = true` only after gate checks pass, and protect cleanup with a `didBegin` flag so `defer` resets state only when execution actually started.
4. Emit domain failures to an `errors` stream (or explicit error state) and also propagate error to the caller; do not route `CancellationError` to the error stream.
5. Start execution from a caller-owned `Task`, keep the task handle, and cancel on lifecycle teardown.
6. Bind button enabled/loading/error UI to those same command states. Derive button enabled from `states()` (`state.isEnabled && !state.isExecuting`), not from `isEnabledStates()` alone.
7. If `Input`/`Output` cannot yet be made `Sendable`, use temporary boundaries (`@MainActor` confinement or sendable DTO snapshots). Use `@unchecked Sendable` only with documented invariants and tracking.

## Guardrails

- Avoid full-repo big-bang rewrites.
- Avoid mixing two reactive models in the same function body unless bridging.
- Prefer explicit error mapping over silently converting errors to empty streams.
- Keep threading intent explicit (`@MainActor`, `Task`, `receive(on:)`).
- Avoid `Task.detached` as a default replacement for Rx scheduling; detached tasks do not inherit parent actor isolation, priority, or task-local values.
- Treat cancellation as cooperative: propagate cancel intent and check cancellation in long-running async work.
- For callback-based or delegate-backed async bridges, use `withTaskCancellationHandler(operation:onCancel:)` to release resources when tasks are canceled.

## Output Requirements

- Produce a migration plan with phases and rollback points.
- Show concrete before/after snippets for each changed API surface.
- Call out behavioral differences when operator semantics do not match exactly.
- Track residual Rx usage until it reaches zero.

## References

- Use [Phased Migration Playbook](references/phased-migration-playbook.md) for rollout strategy.
- Use [RxSwift/Combine/Concurrency Mapping](references/rxswift-combine-concurrency-map.md) for API substitutions and caveats.
