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

### From ai-skills monorepo

Clone monorepo:

```bash
git clone https://github.com/Disconnecter/ai-skills.git
cd ai-skills
```

Use this skill at:

- `swift-struct-size/SKILL.md`
- `swift-struct-size/references/field-order-and-struct-size.md`

For Claude Code, add to project `CLAUDE.md`:

```markdown
## Skills

@/absolute/path/to/ai-skills/swift-struct-size/SKILL.md
@/absolute/path/to/ai-skills/swift-struct-size/references/field-order-and-struct-size.md
```

### Standalone repo (Skills-compatible CLI)

### OpenAI/Codex Skills-compatible CLI

After publishing the repo on GitHub, install with:

```bash
npx skills add Disconnecter/swift-struct-size
```

Then invoke in chat:

```text
Use $swift-struct-size to optimize this struct layout.
```

### Claude Code

Clone the repository:

```bash
git clone https://github.com/Disconnecter/swift-struct-size.git
```

Add references to your project `CLAUDE.md`:

```markdown
## Skills

@/absolute/path/to/swift-struct-size/SKILL.md
@/absolute/path/to/swift-struct-size/references/field-order-and-struct-size.md
```

Or load on demand in a session:

```text
@/absolute/path/to/swift-struct-size/SKILL.md
```

### Manual install (any AI CLI)

If your tool does not support a skill registry command, clone the repo and point your tool to:

- `SKILL.md` as the primary instruction file
- `references/field-order-and-struct-size.md` as supporting context
- `agents/openai.yaml` only if your CLI supports skill catalog metadata

## Publish To GitHub

1. Create a new GitHub repository (for example: `swift-struct-size`).
2. Copy this folder into the repo root (or keep it as a subfolder in a larger skills repo).
3. Commit and push:

```bash
git init
git add .
git commit -m "Add swift-struct-size Codex skill"
git branch -M main
git remote add origin https://github.com/Disconnecter/swift-struct-size.git
git push -u origin main
```

4. Add a release tag when ready:

```bash
git tag v1.0.0
git push origin v1.0.0
```

## Recommended Repo Layout

If this is a single-skill repo:

```text
swift-struct-size/
  README.md
  SKILL.md
  agents/openai.yaml
  references/field-order-and-struct-size.md
```

If this is a multi-skill repo:

```text
skills/
  swift-struct-size/
    SKILL.md
    agents/openai.yaml
    references/field-order-and-struct-size.md
```

## Validation Before Publish

Run the validator from the Skill Creator toolkit:

```bash
python3 "${CODEX_HOME:-$HOME/.codex}/skills/.system/skill-creator/scripts/quick_validate.py" ./swift-struct-size
```

Adjust `CODEX_HOME` if your Codex home directory is non-standard. If your environment is missing `PyYAML`, install it (or run with a temporary `PYTHONPATH`) before validating.

## Usage Trigger Examples

- "Optimize this Swift struct layout to reduce memory."
- "Why is this struct stride bigger than size?"
- "Reorder these fields by alignment and size."
- "Check if moving `Bool` fields to the end saves memory."
