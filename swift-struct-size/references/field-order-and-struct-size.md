# Field Order And Struct Size Notes

Source article:
`https://appsell.su/blog/den-apps-1/razrabotka/kak-poryadok-parametrov-mozhet-vliyat-na-razmer-struktury-v-swift-137` (published March 7, 2026).

## 64-bit Baseline (`MemoryLayout`)

Use these values as a practical baseline on modern 64-bit Apple targets:

| Type | size | stride | alignment |
|------|------|--------|-----------|
| `Int8` | 1 | 1 | 1 |
| `Int16` | 2 | 2 | 2 |
| `Int32` | 4 | 4 | 4 |
| `Int64` | 8 | 8 | 8 |
| `Int` | 8 | 8 | 8 |
| `UInt8` | 1 | 1 | 1 |
| `UInt16` | 2 | 2 | 2 |
| `UInt32` | 4 | 4 | 4 |
| `UInt64` | 8 | 8 | 8 |
| `UInt` | 8 | 8 | 8 |
| `Float` | 4 | 4 | 4 |
| `Double` | 8 | 8 | 8 |
| `Float16` | 2 | 2 | 2 |
| `Bool` | 1 | 1 | 1 |
| `Character` | 16 | 16 | 8 |
| `String` | 16 | 16 | 8 |
| `String?` | 16 | 16 | 8 |
| `Int?` / `Optional<Int>` | 9 | 16 | 8 |
| `UInt?` | 9 | 16 | 8 |
| `Int64?` | 9 | 16 | 8 |
| `UInt64?` | 9 | 16 | 8 |
| `Double?` | 9 | 16 | 8 |
| `Int32?` | 5 | 8 | 4 |
| `UInt32?` | 5 | 8 | 4 |
| `Float?` | 5 | 8 | 4 |
| `Int16?` | 3 | 4 | 2 |
| `UInt16?` | 3 | 4 | 2 |
| `Float16?` | 3 | 4 | 2 |
| `Int8?` | 2 | 2 | 1 |
| `UInt8?` | 2 | 2 | 1 |
| `Bool?` | 1 | 1 | 1 |
| `MyClass` reference | 8 | 8 | 8 |
| `MyClass?` / `Optional<MyClass>` | 8 | 8 | 8 |
| `any AnyObject` (no witness table) | 8 | 8 | 8 |
| `any P` (unconstrained existential) | 40 | 40 | 8 |
| `any P & AnyObject` (1 protocol + class constraint) | 16 | 16 | 8 |

## Core Ideas

- Do not assume struct byte size equals the sum of field sizes.
- Expect compiler-inserted padding to satisfy alignment.
- Understand that field order can increase or decrease internal padding.
- Distinguish:
  - `MemoryLayout<T>.size`: payload bytes without final trailing padding.
  - `MemoryLayout<T>.stride`: bytes occupied per instance in arrays/buffers.

## Example Pattern

Given mixed fields (`String`, `Int`, `Bool`), placing `Bool` before 8-byte aligned fields can force extra internal padding.

- Less efficient pattern: `Bool`, `String`, `Bool`, `Int`
- More efficient pattern: `String`, `Int`, `Bool`, `Bool`

Using the article assumptions (`String`=16, `Int`=8, `Bool`=1), the first ordering can reach 40-byte stride while the grouped ordering can reach 32-byte stride.

## Practical Rule

- Sort by alignment groups first (`8`, `4`, `2`, `1`), then by size descending inside each group.
- Place high-cost fields early with explicit intra-group order for `alignment = 8`: unconstrained existentials (`40+`) -> multi-protocol class-constrained existentials (`24, 32...`) -> `16`-byte values (`String`, `String?`, `Character`, `any P & AnyObject`) -> `9`-byte optionals (`Int?`, `UInt?`, `Int64?`, `UInt64?`, `Double?`) -> `8`-byte values (class references, `any AnyObject`, integer/float primitives).
- For `alignment = 1`, place size-2 optionals (`Int8?`, `UInt8?`) before size-1 fields (`Bool`, `Bool?`, `Int8`, `UInt8`, tiny enums).
- Measure enums and nested structs directly with `MemoryLayout` rather than guessing.

## Optionals And Existentials

- `Optional<T>` layout is not uniform across `T`.
- `Optional<T>.alignment == T.alignment`.
- If `T` has spare bit patterns (extra inhabitants), `Optional<T>` can remain the same size as `T` (for example `Bool?`, `String?`, class references).
- If `T` has no spare bit patterns (common for integers/floats), `Optional<T>` typically needs an additional tag byte (for example `Int?`, `UInt?`, `Int64?`, `UInt64?`, `Int32?`, `UInt32?`, `Int16?`, `UInt16?`, `Int8?`, `UInt8?`, `Double?`, `Float?`, `Float16?`).
- ABI rule: extra inhabitants of the payload type that are not consumed by the enum's no-data cases become extra inhabitants of the outer single-payload enum.
- For nested optionals, if `Optional<T>` still has spare representations after encoding `.none`, `T??` reuses them instead of growing.
- For plain integer/float payload families in ABI-stable Swift, nested optionals are guaranteed to stay in the same size tier as the single optional.
- Verify concrete nested forms (`Int??`, `String??`) with `MemoryLayout` in target builds.
- Unconstrained `any P` values are commonly 40 bytes on 64-bit and can dominate struct layout.
- Multi-protocol existentials add witness-table words (on 64-bit, typically +8 bytes per additional protocol; example: `any P1 & P2` is commonly 48 bytes).
- Class-constrained existentials (`any P & AnyObject`) are much smaller than unconstrained value existentials and should be measured separately.
- For class-constrained existentials, each additional protocol typically adds one witness-table word (+8 bytes on 64-bit): `any P & AnyObject` ~= 16, `any P1 & P2 & AnyObject` ~= 24.
- Distinguish bare `any AnyObject` (object reference only, typically 8 bytes; ABI-compatible with Objective-C `id`/`id<Protocol>`) from `any P & AnyObject` (reference + witness table, typically 16+ bytes).

## Notes

- Values above are representative for 64-bit Apple targets; always verify in the target build with `MemoryLayout`.
- `String` is a 16-byte value type with small-string optimization (short UTF-8 payload can stay inline; larger content spills to heap).
- `Character` is also a 16-byte value, but representation details are runtime-specific; treat layout as fixed-size value plus possible out-of-line storage.
- Reordering can affect readability and serialization assumptions; treat it as a measured optimization pass.

## Canonical References

- Swift ABI docs directory: https://github.com/swiftlang/swift/tree/main/docs/ABI
- Swift ABI TypeLayout document (see Single-Payload Enums section): https://github.com/swiftlang/swift/blob/main/docs/ABI/TypeLayout.rst
- Swift ABI stability manifesto: https://github.com/swiftlang/swift/blob/main/docs/ABIStabilityManifesto.md
