# Name Resolution: `open type` Extension Operators

This document describes how extension operators declared on a type and brought
into scope via `open type` participate in F# name resolution, and in particular
how they are used to solve SRTP (statically resolved type parameter) and
extension member constraints.

See RFC FS-1043 for the user-visible feature.

## User-visible feature

A type can declare static operator members:

```fsharp
[<AbstractClass; Sealed>]
type Ops =
    static member inline (+!) (a: int, b: int) = a + b + 1
    static member inline (+!) (a: string, b: string) = a + b + "!"
```

After `open type Ops`, those operators are available unqualified at F# source
level, and they participate in SRTP/trait-constraint resolution for `inline`
generic code:

```fsharp
open type Ops

let r1 = 1 +! 2                    // 4
let r2 = "a" +! "b"                // "ab!"

let inline combine (a: ^T) (b: ^T) = a +! b
let r3 = combine 10 20             // 31
```

Operators from multiple holder types open into the same scope accumulate and
participate in a single overload set.

## Storage: `eOpenedTypeOperators`

The operators collected from all in-scope `open type` declarations live in a
dedicated field on `NameResolutionEnv`:

```fsharp
eOpenedTypeOperators: NameMultiMap<MethInfo>
```

Declared at `src/Compiler/Checking/NameResolution.fs:454` and surfaced on the
`.fsi` at line 237.

The map is keyed by the operator's `LogicalName` (e.g. `op_PlusPlus`,
`op_Multiply`). Each bucket holds every `MethInfo` contributed by any
`open type` in the current resolution scope. Buckets accumulate as more
`open type` declarations come into scope; they shrink when leaving scope the
usual way (name-env lexical shadowing).

These operators are intentionally **excluded** from `eUnqualifiedItems` by
`ChooseMethInfosForNameEnv`. They are kept in their own index because the
constraint solver needs a separate, targeted lookup path during SRTP
resolution; mixing them into the unqualified-value namespace would change
overload-resolution behavior for ordinary value-binding lookup.

## Insertion site

Operators are collected and inserted into the map by
`AddStaticContentOfTypeToNameEnv`, located at
`src/Compiler/Checking/NameResolution.fs:1203`. The operator-filtering and
bucket-insertion block is at lines 1270–1301.

The filter keeps static (non-instance, non-ctor) methods whose
`ApparentEnclosingType` is the opened type and whose `LogicalName` matches
`IsLogicalOpName`:

```fsharp
let operatorMethods =
    allMethInfos
    |> List.filter (fun minfo ->
        not (minfo.IsInstance || minfo.IsClassConstructor || minfo.IsConstructor)
        && typeEquiv g minfo.ApparentEnclosingType ty
        && IsLogicalOpName minfo.LogicalName)
```

Each surviving `MethInfo` is added with `AddMethInfoByLogicalName` — a
`NameMultiMap.add` keyed by `minfo.LogicalName`.

## The order invariant

Within a single `open type`, the operator list lands in the bucket in
**source-declaration order**: the operator written first ends up at the head
of the bucket. This matches the semantics of the previous list-prepend
implementation; the insertion uses `List.foldBack` (not a left fold, not
`NameMultiMap.initBy`) to preserve it.

This invariant is defensive rather than observably pinned. Overload
resolution in `MethodCalls.fs` is order-sensitive when two candidates are
equally applicable, but in practice candidates within the same holder type
disambiguate on parameter types first. The bucket order becomes observable
across multiple `open type` declarations: the most-recently-opened holder
appears first in the bucket (because each new `foldBack` prepends), which
matches the cross-`open` semantics of the prior flat-list implementation.

Design notes:

- `NameMultiMap.initBy` groups via `Seq.groupBy`, which does not preserve
  within-group order relative to source; it is deliberately avoided here.
- `List.foldBack f [a; b; c] init` adds `c` first, then `b`, then `a` — so
  `a` ends up at the head of the bucket. That matches source order.

The coverage test
`tests/.../ExtensionConstraints/testFiles/OpenTypeOperatorHomographOrder.fs`
exercises the >=2-entries-per-bucket path. It does not by itself pin the
within-call source-order tie-break (its three overloads disambiguate on
parameter type). Adding a genuine tie-break test is future work.

## Consumer: `SelectExtensionMethInfosForTrait`

The bucket is read by `SelectExtensionMethInfosForTrait` in
`src/Compiler/Checking/NameResolution.fs`. During SRTP/trait-constraint
solving the compiler asks "for trait with member name `nm`, which extension
candidates are available?" and this function returns:

1. Per-support-type results from `SelectExtMethInfosForType` (classical
   extension members registered against each support type).
2. The full list from `NameMultiMap.find nm nenv.eOpenedTypeOperators`,
   each paired with the first support type to avoid creating synthetic
   duplicates that would confuse overload resolution.

The returned list from the `NameMultiMap` is in source-order — follows
automatically from the insertion site. The result then feeds overload
resolution in `MethodCalls.fs`.

## Test coverage

All under
`tests/FSharp.Compiler.ComponentTests/Conformance/Types/TypeConstraints/ExtensionConstraints/testFiles/`
and registered in `ExtensionConstraintsTests.fs`:

- `BasicExtensionOperators.fs` — baseline: `open type` + operator + SRTP call.
- `ExtensionPrecedence.fs` — most-recent `open` wins.
- `ExtensionAccessibility.fs` — private/internal gates are honored.
- `ScopeCapture.fs` — extension captured at the inline definition site
  survives across module boundaries.
- `OpenTypeOperatorHomographOrder.fs` — pins the within-holder source-order
  invariant documented above.
- `OpenTypeOperatorHomographMultipleHolders.fs` — pins cross-`open type`
  accumulation into a shared bucket.
- `OpenTypeOperatorNestedModule.fs` — lexical scoping of `open type` inside
  a nested module.
- `OpenTypeOperatorShadowing.fs` — a local `let` binding shadows an
  `open type`-provided operator.
- `OpenTypeOperatorSRTPDispatch.fs` — an inline SRTP function dispatches to
  the correct overload when overloads live on *different* holder types.
- `OpenTypeOperatorOverloadByParam.fs` — homograph overloads on a single
  holder differ only by one parameter type; overload resolution picks the
  right one.
- `OpenTypeOperatorCompiledName.fs` — `[<CompiledName>]` on an operator
  changes only the emitted CLR name; the F# `LogicalName` still keys the
  bucket and the F# source symbol still resolves.
- Cross-assembly coverage lives inline in `ExtensionConstraintsTests.fs`
  (see the `open type extension operator crosses assembly boundary` test),
  exercising a library DLL that declares the holder and a consumer that
  `open type`s it.

## Known gotchas

- **Operator symbol rules.** The F# parser classifies operator symbols as
  prefix vs infix by their first character. Names starting with `!` (other
  than the built-in `!=`) or `~` parse as prefix; only the standard infix
  character set (`+ - * / % < > = & | ^ . ? : @ $`) is parsed as infix.
  A holder-declared operator must use symbols the parser will read as the
  intended fixity at the call site.
- **Shadowing by a local `let`.** An ordinary `let inline (op) ...` in the
  current lexical scope hides an `open type` operator of the same symbol —
  because unqualified-value lookup hits `eUnqualifiedItems` before the
  SRTP path consults `eOpenedTypeOperators`. Pinned by
  `OpenTypeOperatorShadowing.fs`.
- **Nested modules.** `open type` inside a nested module is scoped to that
  module. A sibling scope sees neither the opened type nor its operators.
  Pinned by `OpenTypeOperatorNestedModule.fs`.
- **`CompiledName` vs `LogicalName`.** `[<CompiledName("X")>]` renames only
  the emitted CLR method; the F# `LogicalName` (`op_PlusPlus` etc.) is what
  drives both the bucket key and `IsLogicalOpName` filtering, so the F#
  source symbol keeps working. Pinned by `OpenTypeOperatorCompiledName.fs`.
- **Built-ins win.** The built-in operators for primitive types take
  precedence over an extension operator declared on the same primitive —
  even through SRTP. Pinned by the
  `Built-in operator wins over extension on same type` test.

## Measurements

The refactor to a name-indexed bucket (`NameMultiMap<MethInfo>`) replaces a
per-lookup filter over a flat `MethInfo list`. Under a synthetic
FSharpPlus-like stress harness (`SemigroupOp`-style overloads for `list`,
`array`, `Option`, `ResizeArray`, `string`, `HashSet`, `Map`, `seq`, plus
~40 consumer files exercising SRTP dispatch), typecheck phase self-time
was within measurement noise across 5 sample runs on macOS / .NET 10.0
(`dotnet fsc --times`):

| Variant         | Typecheck self (s, median of 5) |
| --------------- | ------------------------------- |
| Pre-refactor    | 1.148                           |
| Post-refactor   | 1.206                           |

Delta is inside run-to-run variance (stddev on both variants ≈ 0.08s),
so this is not a measured perf win on this harness. The refactor stands
on structural grounds — named indexing of a trait-name-keyed bucket is
the right data model for the `SelectExtensionMethInfosForTrait` consumer
— not on wall-clock improvement. Benchmarking on a larger corpus
(FSharpPlus, Fable repo etc., see PLAN.md §0.4) is future work to either
confirm neutrality or surface a target-specific improvement.
