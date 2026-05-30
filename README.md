# temper-issue-when-void

Repro for a type inference bug in Temper `fbcc92f`.

## The bug

`when` expressions with `is TypeName` arms and no `else` arm always type as `Void`,
regardless of what the arms return. The same expressions compile and work correctly
when rewritten as if-else chains.

```
when (s) {
  is Circle -> "circle";
  is Square -> "square";
}
// @G: Cannot assign to String from Void
```

`when` with an `else` arm is not affected. The bug is specific to the `is TypeName`
pattern when no `else` is present.

## Root cause, maybe?

**File:** `interp/src/commonMain/kotlin/lang/temper/interp/IfTransform.kt`
**Lines:** 221–234

```kotlin
// `if` chains that don't end in an `else` should be typed as Void.
// See LoopTransform comments on typing as for the problems with control-flow
// constructs that start with a condition and which don't reliably follow it
// with a typeable tree.
if (controlFlow != null && !hasFinalElse) {
    controlFlow = ControlFlow.StmtBlock(
        controlFlow.pos,
        listOf(
            controlFlow,
            ControlFlow.Stmt(
                macroCursor.referenceToVoid(controlFlow.pos.rightEdge),
            ),
        ),
    )
}
```

After `IfTransform` builds the nested `ControlFlow.If` chain from the `when` arms,
this block wraps the entire chain in a `ControlFlow.StmtBlock` and appends a trailing
`ControlFlow.Stmt(void)` as a sequential statement after the if-chain.

`findUnsetTerminalExpressions` in `UnsetTerminalExpressions.kt` walks exit paths and
finds this trailing `void` as the last statement on every path. `MakeResultsExplicit`
assigns only `void` to the result variable. The `Typer` infers `Void`. The actual arm
body expressions live inside `if` branches visited before the trailing `void` and are
not selected as terminals.

This behaviour is correct for a bare `if` without `else` used as a statement. It is
wrong for `when` used as an expression with `is TypeName` patterns, where the type
should reflect the common type of the arms (or `arm_types | Void` if the match is
not exhaustive).

## Why `when` with `else` works

When `hasFinalElse = true`, lines 221–234 do not execute. The innermost synthetic
else clause (lines 190–195) receives a `void` reference, which correctly participates
in the union type, and the arm types are found normally.

## The fix, maybe?

Remove lines 221–234 entirely.

The `void` inserted at lines 190–195 in the innermost synthetic else clause already
marks the "fell through all arms" exit path as `Void`. The trailing sequential `void`
is redundant and solely responsible for shadowing the arm body terminals.

Once lines 221–234 are removed, `findUnsetTerminalExpressions` finds the arm body
expressions alongside the innermost else's `void`, so the inferred type becomes
`arm_types | Void` rather than just `Void`. This is the minimal fix and requires no
changes to `findUnsetTerminalExpressions`, `MakeResultsExplicit`, or the `Typer`.

## How to reproduce

```
git clone https://github.com/notactuallytreyanastasio/temper-issue-when-void
cd temper-issue-when-void
temper build
# @G: Cannot assign to String from Void
# @G: Cannot assign to Status from Void
```

The workaround (if-else chains) is in `src/repro.temper.md` alongside the failing
cases so the difference is visible in one file.

## Version

Temper `fbcc92f`. Reproduced 2026-05-30 on macOS (Apple M-series).

## Context

Found porting [chess9](https://github.com/notactuallytreyanastasio/chess9) to Temper.
Every sealed-interface dispatch function (move kind, board status, move error —
~10 functions total) required the if-else workaround.
