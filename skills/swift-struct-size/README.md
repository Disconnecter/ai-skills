# swift-struct-size

A Codex skill for optimizing Swift struct memory layout by reordering stored properties based on `MemoryLayout` size/alignment behavior on 64-bit Apple platforms.

Invocation name: `$swift-struct-size`

## What This Skill Does

- Explains padding, alignment, `size`, and `stride` in practical terms.
- Applies a deterministic sorting rule for struct fields: sort by `alignment` descending, then by `size` descending.
- Covers common layout traps: `Optional<T>`, class references, `any P` existentials, enums, and nested structs.
- Uses concrete 64-bit baseline values for common Swift types, including optionals.
- Includes ABI safety guidance (`@frozen`, public framework types) and a "when not to optimize" workflow.

## Skill Files

- `SKILL.md`: Main skill instructions and workflow.
- `references/field-order-and-struct-size.md`: Baseline type table and article-based notes.
- `agents/openai.yaml`: Skill metadata and policy (`display_name`, `short_description`, `default_prompt`, `allow_implicit_invocation`).

## Install (Different AI CLI Tools)

### From `ai-skills` monorepo (preferred)

Clone monorepo:

```bash
git clone https://github.com/Disconnecter/ai-skills.git
cd ai-skills
```

Use this skill at:

- `skills/swift-struct-size/SKILL.md`
- `skills/swift-struct-size/references/field-order-and-struct-size.md`

For Claude Code, add to project `CLAUDE.md`:

```markdown
## Skills

@/absolute/path/to/ai-skills/skills/swift-struct-size/SKILL.md
@/absolute/path/to/ai-skills/skills/swift-struct-size/references/field-order-and-struct-size.md
```

### Install via `npx skills`

```bash
npx skills add https://github.com/Disconnecter/ai-skills --skill swift-struct-size
```

### Manual install (any AI CLI)

If your CLI does not have native skill installation, load these files directly:

- `skills/swift-struct-size/SKILL.md` as primary instructions
- `skills/swift-struct-size/references/field-order-and-struct-size.md` as supporting context
- `skills/swift-struct-size/agents/openai.yaml` only if your CLI supports metadata catalogs

## Usage Trigger Examples

- "Optimize this Swift struct layout to reduce memory."
- "Why is this struct stride bigger than size?"
- "Reorder these fields by alignment and size."
- "Check if moving `Bool` fields to the end saves memory."
