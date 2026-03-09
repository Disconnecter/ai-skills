# ai-skills

Monorepo for multiple AI skills.

GitHub: `https://github.com/Disconnecter/ai-skills`

## Included Skills

- `rxswift-to-swift6-combine-concurrency` (invocation: `$rx-migrate`)
- `swift-struct-size` (invocation: `$swift-struct-size`)

## Repository Layout

```text
skills/
  rxswift-to-swift6-combine-concurrency/
    SKILL.md
    agents/openai.yaml
    references/
  swift-struct-size/
    SKILL.md
    agents/openai.yaml
    references/field-order-and-struct-size.md
```

## Install This Monorepo

```bash
git clone https://github.com/Disconnecter/ai-skills.git
cd ai-skills
```

## Install Via `npx skills`

Install from GitHub and select one skill:

```bash
npx skills add https://github.com/Disconnecter/ai-skills --skill rxswift-to-swift6-combine-concurrency
npx skills add https://github.com/Disconnecter/ai-skills --skill swift-struct-size
```

Install both at once:

```bash
npx skills add https://github.com/Disconnecter/ai-skills \
  --skill rxswift-to-swift6-combine-concurrency \
  --skill swift-struct-size
```

## Install Each Skill

### 1) rxswift-to-swift6-combine-concurrency

Skill files:

- `skills/rxswift-to-swift6-combine-concurrency/SKILL.md`
- `skills/rxswift-to-swift6-combine-concurrency/references/rxswift-combine-concurrency-map.md`
- `skills/rxswift-to-swift6-combine-concurrency/references/phased-migration-playbook.md`

Claude Code (`CLAUDE.md`):

```markdown
## Skills

@/absolute/path/to/ai-skills/skills/rxswift-to-swift6-combine-concurrency/SKILL.md
@/absolute/path/to/ai-skills/skills/rxswift-to-swift6-combine-concurrency/references/rxswift-combine-concurrency-map.md
@/absolute/path/to/ai-skills/skills/rxswift-to-swift6-combine-concurrency/references/phased-migration-playbook.md
```

Manual (any AI CLI):

- Use `skills/rxswift-to-swift6-combine-concurrency/SKILL.md` as primary instructions.
- Load `skills/rxswift-to-swift6-combine-concurrency/references/` as supporting context.

### 2) swift-struct-size

Skill files:

- `skills/swift-struct-size/SKILL.md`
- `skills/swift-struct-size/references/field-order-and-struct-size.md`

Claude Code (`CLAUDE.md`):

```markdown
## Skills

@/absolute/path/to/ai-skills/skills/swift-struct-size/SKILL.md
@/absolute/path/to/ai-skills/skills/swift-struct-size/references/field-order-and-struct-size.md
```

Manual (any AI CLI):

- Use `skills/swift-struct-size/SKILL.md` as primary instructions.
- Load `skills/swift-struct-size/references/field-order-and-struct-size.md` as supporting context.
