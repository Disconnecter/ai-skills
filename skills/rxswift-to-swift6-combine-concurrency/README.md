# RxSwift to Swift 6/Combine Concurrency Skill

Codex skill for migrating RxSwift codebases to Swift 6, Combine, and modern Swift Concurrency patterns.

Invocation name: `$rx-migrate`

## What this skill covers

- RxSwift trait and operator migration (`Observable`, `Single`, `Completable`, `Maybe`, `Infallible`, `Driver`, `Signal`)
- Subject migration (`PublishSubject`, `BehaviorSubject`, `ReplaySubject`)
- Relay migration (`BehaviorRelay`, `PublishRelay`)
- RxAction/`Action` command pattern migration
- UIKit/RxCocoa binding migration (`.rx.tap`, `.rx.text`, `.rx.items`, control events)
- Networking migration (`Single` -> `async throws`, URLSession async APIs, retry/timeout)
- Migration planning with phased rollout and rollback points
- Swift 6 concurrency hardening (`Sendable`, actors, `@MainActor`, `nonisolated(unsafe)`, cancellation)
- Observation adoption guidance (`@Observable` migration + `withObservationTracking` usage)
- Bridge-based incremental migration using RxSwift 6.5 async interop

## Files

- `SKILL.md`: skill trigger description and workflow
- `references/rxswift-combine-concurrency-map.md`: API mapping and migration caveats
- `references/phased-migration-playbook.md`: phased execution strategy
- `agents/openai.yaml`: UI metadata for skill catalogs

## Install

### From `ai-skills` monorepo (preferred)

Clone monorepo:

```bash
git clone https://github.com/Disconnecter/ai-skills.git
cd ai-skills
```

Use this skill at:

- `skills/rxswift-to-swift6-combine-concurrency/SKILL.md`
- `skills/rxswift-to-swift6-combine-concurrency/references/rxswift-combine-concurrency-map.md`
- `skills/rxswift-to-swift6-combine-concurrency/references/phased-migration-playbook.md`

For Claude Code, add to project `CLAUDE.md`:

```markdown
## Skills

@/absolute/path/to/ai-skills/skills/rxswift-to-swift6-combine-concurrency/SKILL.md
@/absolute/path/to/ai-skills/skills/rxswift-to-swift6-combine-concurrency/references/rxswift-combine-concurrency-map.md
@/absolute/path/to/ai-skills/skills/rxswift-to-swift6-combine-concurrency/references/phased-migration-playbook.md
```

### Install via `npx skills`

```bash
npx skills add https://github.com/Disconnecter/ai-skills --skill rx-migrate
```

### Manual install (any AI CLI)

If your CLI does not have native skill installation, load these files directly:
- `skills/rxswift-to-swift6-combine-concurrency/SKILL.md` as primary instructions
- `skills/rxswift-to-swift6-combine-concurrency/references/rxswift-combine-concurrency-map.md`
- `skills/rxswift-to-swift6-combine-concurrency/references/phased-migration-playbook.md`
- `skills/rxswift-to-swift6-combine-concurrency/agents/openai.yaml` only if your CLI supports metadata catalogs

## Source documentation

- RxSwift main repo: https://github.com/ReactiveX/RxSwift/tree/main
- Action (RxSwiftCommunity) repo: https://github.com/RxSwiftCommunity/Action
- Swift Concurrency guide: https://docs.swift.org/swift-book/documentation/the-swift-programming-language/concurrency/
- Observation framework: https://developer.apple.com/documentation/observation
