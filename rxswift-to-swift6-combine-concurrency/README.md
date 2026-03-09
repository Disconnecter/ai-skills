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

### OpenAI/Codex Skills CLI

```bash
npx skills add Disconnecter/rxswift-to-swift6-combine-concurrency-skill
```

This is the canonical install form for Skills-compatible CLIs.

### Claude Code

Clone the repo, then add the skill to your project's `CLAUDE.md` so Claude Code loads it automatically:

```bash
git clone https://github.com/Disconnecter/rxswift-to-swift6-combine-concurrency-skill.git
```

In your project's `CLAUDE.md` (create it at the repo root if it doesn't exist), add:

```markdown
## Skills

@/path/to/rxswift-to-swift6-combine-concurrency-skill/SKILL.md
@/path/to/rxswift-to-swift6-combine-concurrency-skill/references/rxswift-combine-concurrency-map.md
@/path/to/rxswift-to-swift6-combine-concurrency-skill/references/phased-migration-playbook.md
```

Replace `/path/to/` with the actual path where you cloned the repo. Claude Code reads `CLAUDE.md` automatically at session start, so the skill context is always available.

Alternatively, reference the files on demand in any Claude Code session without modifying `CLAUDE.md`:

```
@/path/to/rxswift-to-swift6-combine-concurrency-skill/SKILL.md
```

Then ask Claude to migrate your code. The `SKILL.md` trigger description causes Claude Code to apply the full migration workflow automatically.

### Manual install (any AI CLI)

If your CLI does not support `npx skills add`, use the repo directly:

```bash
git clone https://github.com/Disconnecter/rxswift-to-swift6-combine-concurrency-skill.git
cd rxswift-to-swift6-combine-concurrency-skill
```

Then configure your tool to use:
- `SKILL.md` as the primary instruction/prompt file
- `references/` for supporting migration docs (`map` + `playbook`)
- `agents/openai.yaml` only if your tool supports skill metadata catalogs

## Source documentation

- RxSwift main repo: https://github.com/ReactiveX/RxSwift/tree/main
- Action (RxSwiftCommunity) repo: https://github.com/RxSwiftCommunity/Action
- Swift Concurrency guide: https://docs.swift.org/swift-book/documentation/the-swift-programming-language/concurrency/
- Observation framework: https://developer.apple.com/documentation/observation
