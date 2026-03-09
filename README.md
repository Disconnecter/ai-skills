# ai-skills

Monorepo for multiple AI skills.

GitHub: `https://github.com/Disconnecter/ai-skills`

## Included Skills

- `rxswift-to-swift6-combine-concurrency` (invocation: `$rx-migrate`)
- `swift-struct-size` (invocation: `$swift-struct-size`)

## Install This Monorepo

```bash
git clone https://github.com/Disconnecter/ai-skills.git
cd ai-skills
```

## Install Each Skill

### 1) rxswift-to-swift6-combine-concurrency

Skill files:

- `rxswift-to-swift6-combine-concurrency/SKILL.md`
- `rxswift-to-swift6-combine-concurrency/references/rxswift-combine-concurrency-map.md`
- `rxswift-to-swift6-combine-concurrency/references/phased-migration-playbook.md`

Claude Code (`CLAUDE.md`):

```markdown
## Skills

@/absolute/path/to/ai-skills/rxswift-to-swift6-combine-concurrency/SKILL.md
@/absolute/path/to/ai-skills/rxswift-to-swift6-combine-concurrency/references/rxswift-combine-concurrency-map.md
@/absolute/path/to/ai-skills/rxswift-to-swift6-combine-concurrency/references/phased-migration-playbook.md
```

Manual (any AI CLI):

- Use `rxswift-to-swift6-combine-concurrency/SKILL.md` as primary instructions.
- Load `rxswift-to-swift6-combine-concurrency/references/` as supporting context.

### 2) swift-struct-size

Skill files:

- `swift-struct-size/SKILL.md`
- `swift-struct-size/references/field-order-and-struct-size.md`

Claude Code (`CLAUDE.md`):

```markdown
## Skills

@/absolute/path/to/ai-skills/swift-struct-size/SKILL.md
@/absolute/path/to/ai-skills/swift-struct-size/references/field-order-and-struct-size.md
```

Manual (any AI CLI):

- Use `swift-struct-size/SKILL.md` as primary instructions.
- Load `swift-struct-size/references/field-order-and-struct-size.md` as supporting context.

## Standalone Repo Installs (Optional)

If you prefer standalone skill repos:

- Rx migration skill: `npx skills add Disconnecter/rxswift-to-swift6-combine-concurrency-skill`
- Struct size skill: `npx skills add Disconnecter/swift-struct-size`
