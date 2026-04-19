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

Within a single `open type`, the operator list must land in the bucket in
**source-declaration order**: the operator written first must be at the head
of the bucket. This is why the insertion uses `List.foldBack`, not a left
fold and not `NameMultiMap.initBy`.

The in-source comment at `NameResolution.fs:1283–1299` documents this. The
key facts:

- Overload resolution (`ResolveOverloading` in `MethodCalls.fs`) is
  order-sensitive when two candidates are equally applicable — the first
  wins.
- `NameMultiMap.initBy` groups via `Seq.groupBy`, which does not preserve
  within-group order relative to source; using it here silently flips which
  overload is picked for homograph operators declared on the same holder.
- `List.foldBack f [a; b; c] init` adds `c` first, then `b`, then `a` — so
  `a` ends up at the head of the bucket. That matches source order.

The pinned regression test is
`tests/.../ExtensionConstraints/testFiles/OpenTypeOperatorHomographOrder.fs`.

## Consumer: `SelectExtensionMethInfosForTrait`

The bucket is read by `SelectExtensionMethInfosForTrait`, at
`src/Compiler/Checking/NameResolution.fs:1707`. During SRTP/trait-constraint
solving the compiler asks "for trait with member name `nm`, which extension
candidates are available?" and this function returns:

1. Per-support-type results from `SelectExtMethInfosForType` (classical
   extension members registered against each support type).
2. The full list from `NameMultiMap.find nm nenv.eOpenedTypeOperators`,
   each paired with the first support type to avoid creating synthetic
   duplicates that would confuse overload resolution.

Crucially, the returned list from the `NameMultiMap` must also be in source
order — which follows automatically from the insertion invariant above. The
consumer comment at `NameResolution.fs:1715–1723` re-states this and
cross-references the insertion site.

The result then feeds overload resolution in `MethodCalls.fs`.

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
