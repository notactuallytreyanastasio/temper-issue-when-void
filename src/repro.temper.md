# Bug: `when` with `is Type` arms infers `Void` instead of the arm type

Temper `fbcc92f` — confirmed 2026-05-30

## Summary

When every arm of a `when` expression uses an `is TypeName` pattern,
the type checker assigns `Void` to the whole expression regardless of
what the arms return. The expression cannot be used as a return value,
so every sealed-interface dispatch must be rewritten as if-else chains.

## Repro — arms returning String

    export sealed interface Shape {}
    export class Circle() extends Shape {}
    export class Square() extends Shape {}

The function below produces:
`@G: Cannot assign to String from Void`

    export let describe(s: Shape): String {
      when (s) {
        is Circle -> "circle";
        is Square -> "square";
      }
    }

## Repro — arms returning sealed subtypes

    export sealed interface Status {}
    export class Active()                    extends Status {}
    export class Done(public winner: String) extends Status {}
    export class Drawn()                     extends Status {}

Also produces `Cannot assign to Status from Void`:

    export let classify(code: Int): Status {
      when (code) {
        0    -> new Active();
        1    -> new Done("white");
        else -> new Drawn();
      }
    }

## Workaround — if-else narrowing compiles and passes

    export let describeOk(s: Shape): String {
      if (s is Circle) { "circle" } else { "square" }
    }

    export let classifyOk(code: Int): Status {
      if (code == 0) {
        new Active()
      } else if (code == 1) {
        new Done("white")
      } else {
        new Drawn()
      }
    }

    test("if-else workaround") {
      assert(describeOk(new Circle()) == "circle") { "circle" };
      assert(describeOk(new Square()) == "square") { "square" };
      assert(classifyOk(0) is Active) { "active" };
      assert(classifyOk(1) is Done)   { "done" };
      assert(classifyOk(2) is Drawn)  { "drawn" };
    }

## Expected behaviour

`describe` and `classify` should compile. The `when` expression type should
be inferred as the common type of the arms (`String` and `Status`).

## Context

Found porting the chess9 TypeScript engine to Temper. Every sealed-interface
dispatch (move kind, board status, move error — ~10 functions) required
the if-else workaround. See: https://github.com/notactuallytreyanastasio/chess9
