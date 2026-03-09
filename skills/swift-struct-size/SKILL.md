---
name: swift-struct-size
description: Analyze and optimize Swift struct memory layout (alignment, padding, size, stride) on 64-bit Apple platforms. Use when tasks involve reducing memory in large arrays/caches, explaining unexpected struct stride, reordering stored properties safely, handling optionals/reference/existential layout pitfalls, and validating `MemoryLayout` results during performance tuning.
---

# Swift Struct Size

## Overview

Diagnose and reduce Swift struct memory footprint by identifying padding, proposing safe field order changes, and validating real byte usage with `MemoryLayout`.

## Workflow

1. Confirm scope and constraints.
- Identify high-volume structs first (arrays, caches, decoded models).
- Confirm whether field order is externally observable (`Codable`, C interop, binary formats, ABI-sensitive code).
- Optimize only when memory pressure is measurable or expected to be large (for example: savings of even 1 byte across millions of instances can matter, otherwise prioritize readability).

2. Measure current layout.
- Print `MemoryLayout<T>.size`, `.stride`, and `.alignment`.
- Estimate impact with: `(oldStride - newStride) * instanceCount`.
- Optionally inspect offsets with `MemoryLayout.offset(of:)` for stored properties.
- Treat nested structs as normal fields and use the nested type's own `size` and `alignment`.

3. Locate padding pressure.
- Treat `size` as raw payload bytes and `stride` as per-instance memory footprint.
- Find internal gaps where a small field is followed by a high-alignment field.
- On 64-bit Swift, use the concrete baseline table in [Field Order And Struct Size Notes](references/field-order-and-struct-size.md) instead of guessing type sizes.
- Do not assume `Optional<T>` behaves like `T`: size may be unchanged (`String?`, `Bool?`) or larger (`Int?`, `Int32?`) depending on spare-bit/extra-inhabitant encoding.

4. Reorder for density.
- Group higher-alignment fields first, then smaller-alignment fields.
- Keep semantically related fields together only when readability is more important than byte savings.
- Push tiny fields (`Bool`, tiny enums) toward the end to reduce repeated internal gaps.
- Place reference-type fields (`MyClass`) in the 8-byte alignment group (pointer-sized on 64-bit).
- Consider unconstrained existentials (`any P`) high-cost fields (commonly 40 bytes on 64-bit; add ~8 bytes per extra protocol in `any P1 & P2 & ...`) and place them with the largest/aligned group.
- Consider class-constrained existentials (`any P & AnyObject`) separately from `any AnyObject` (typically 16+ bytes vs 8 bytes).
- For enums, measure the concrete enum type with `MemoryLayout` instead of inferring from cases.

5. Verify and report.
- Re-run probes after reordering.
- Report before/after `size` and `stride`, plus realistic memory savings.
- If no gain appears, record why (already near-optimal or trailing alignment dominates).

6. Respect ABI/public stability.
- Do not reorder fields in `@frozen` public structs unless ABI impact is explicitly intended.
- In ABI-stable frameworks/libraries, treat public layout changes as breaking and version accordingly.

## Sorting Rule (Swift Only, 64-bit)

Sort stored properties by alignment first, then by size (descending) inside each group:

- `alignment = 8`: apply these tiers in order.
  - Tier 1 (largest): unconstrained `any P` existentials (`40, 48, 56...`).
  - Tier 2: multi-protocol class-constrained existentials (`any P1 & P2 & AnyObject`: `24, 32...`).
  - Tier 3: `String`, `String?`, `Character`, and single-protocol class-constrained existentials (`any P & AnyObject`: `16`).
  - Tier 4: integer/float optionals like `Int?`, `UInt?`, `Int64?`, `UInt64?`, `Double?` (`9`).
  - Tier 5: reference-shaped values (`any AnyObject`, class references) and fixed-width primitives (`Int64`, `UInt64`, `Int`, `UInt`, `Double`) (`8`).
- `alignment = 4`: optionals like `Int32?`, `UInt32?`, `Float?`, then `Int32`, `UInt32`, `Float`.
- `alignment = 2`: optionals like `Int16?`, `UInt16?`, `Float16?`, then `Int16`, `UInt16`, `Float16`.
- `alignment = 1`: optionals like `Int8?`, `UInt8?` (`2`), then `Int8`, `UInt8`, `Bool`, `Bool?`, and many tiny enums (`1`).
- Nested struct fields: use the nested type's measured `alignment` and `size`.

Use this pattern unless you must preserve external layout contracts.

`Optional<T>` rule of thumb:
- `Optional<T>.alignment == T.alignment`.
- `Optional<T>.size` often grows by 1 for plain integer/float payloads (for example `Int`, `UInt`, `Int64`, `UInt64`, `Int32`, `UInt32`, `Int16`, `UInt16`, `Int8`, `UInt8`, `Double`, `Float`, `Float16`).
- `Optional<T>.size` is often unchanged for payloads with spare representations (for example references, `Bool`, `String`, many enums with unused bit patterns).
- ABI rule: extra inhabitants of a payload that are not consumed by the enum's no-data cases become extra inhabitants of the outer single-payload enum.
- For `Optional<Optional<T>>`, if `Optional<T>` has spare representations available after encoding `.none`, nested optionality reuses them instead of adding another byte.
- For plain integer/float payload families in ABI-stable Swift, nested optionals are guaranteed to stay in the same size tier as the single optional.
- Always verify concrete nested forms (`Int??`, `String??`) with `MemoryLayout` in target builds.
- Always validate with `MemoryLayout` for the concrete `T`.

```swift
// Bad: small -> large -> small -> large
struct UserMetricsBad {
    let isActive: Bool
    let name: String
    let isPremium: Bool
    let loginCount: Int
}

// Better: large/aligned first, tiny fields last
struct UserMetricsGood {
    let name: String
    let loginCount: Int
    let isActive: Bool
    let isPremium: Bool
}
```

## When Not To Optimize

- Skip reordering if data volume is small and memory is not a bottleneck.
- Skip reordering if it harms domain readability more than it saves memory.
- Skip reordering in API/ABI-sensitive types unless the breaking impact is accepted.
- Always prefer measurement-driven optimization over speculative field shuffling.

## Quick Probe

```swift
func printLayout<T>(_ t: T.Type) {
    print("\(T.self): size=\(MemoryLayout<T>.size) stride=\(MemoryLayout<T>.stride) alignment=\(MemoryLayout<T>.alignment)")
}

// Example:
// printLayout(MyStruct.self)
// if let offset = MemoryLayout<MyStruct>.offset(of: \MyStruct.someStoredProperty) {
//     print(offset)
// } else {
//     print("offset unavailable")
// }
//
// Notes:
// - `offset(of:)` returns `nil` for computed properties.
// - `offset(of:)` supports stored `let` and `var` properties.
// - `offset(of:)` can also return `nil` across resilience boundaries (for example public non-`@frozen` types in library/framework modules).
```

## References

Use [Field Order And Struct Size Notes](references/field-order-and-struct-size.md) for article-grounded examples and alignment rules.
