> **Sync contract**: A condensed version of this document lives in the workspace `AGENTS.md` under `## F#`. When this file is updated, update that section too.

# F# Notes — Patterns, Learnings, and Scott Wlaschin's Influence

This file is a living document. Add to it whenever you encounter a meaningful F# error, learn a pattern, or encounter a nuance that isn't obvious. It applies to all NEXUS projects.

---

## Scott Wlaschin — Domain Modeling Made Functional

Scott Wlaschin ([@ScottWlaschin](https://twitter.com/ScottWlaschin)) is the author of *Domain Modeling Made Functional* (Pragmatic Programmers) and the creator of [fsharpforfunandprofit.com](https://fsharpforfunandprofit.com). His teachings are a foundational influence on how this codebase models the domain.

### Core Ideas

**Make illegal states unrepresentable.**
Use the type system to encode invariants. If a value can never be null, never make it nullable. If a field can only hold two possible values, use a discriminated union — not a string, not a bool, not an int. The compiler enforces what the code can't express in prose.

```fsharp
// Bad — "kind" can hold any string
type Actor = { Name: string; Kind: string }

// Good — only valid kinds compile
type ActorKind =
    | Human          of role: string
    | Automation     of name: string
    | ExternalSystem of name: string
```

**The domain model is the design.**
Types are not DTOs or database rows. They are executable documentation. If a non-developer can read the type definitions and understand the domain, the model is right.

**Composition over inheritance.**
F# has no inheritance for data types. You compose behavior by combining functions and types. Functions are values — pass them in records, return them from functions, store them in maps.

**Total functions over partial ones.**
A function that can fail on certain inputs is a lie. If a function might not return a result, encode that in the return type: `option`, `Result`, or a custom DU. Never throw exceptions for domain failures — `Result<'T, string>` or `Result<'T, DomainError>` is the right tool.

```fsharp
// Total — caller is forced to handle both cases
type CommandHandler<'TState, 'TCommand, 'TEvent> =
    Event<'TState> list -> Command<'TCommand> -> Result<Event<'TEvent> list, string>
```

**Parse, don't validate.**
Validate at the boundary. Once data enters the domain, it is already valid by construction. Don't repeat null checks and range checks inside domain functions — use smart constructors and wrapper types at the edge.

---

## Key fsharpforfunandprofit.com Patterns

### Railway-Oriented Programming (ROP)

Scott's term for chaining `Result`-returning functions. The "happy path" flows straight through; errors are carried along the railway without explicit checking at every step.

```fsharp
// Bind (>>=) threads Result values together
let (>>=) result f =
    match result with
    | Ok v    -> f v
    | Error e -> Error e

// Or use the built-in computation expression
let workflow input =
    result {
        let! validated = validate input
        let! processed = process validated
        return processed
    }
```

Use this pattern in CommandHandlers that have multiple validation steps.

### Single-Case Discriminated Unions for Wrapper Types

Wrap primitives to prevent mixing them up. A `CustomerId` and an `OrderId` are both `int` under the hood but they are not interchangeable.

```fsharp
type CustomerId = CustomerId of int
type OrderId    = OrderId    of int

// Unwrap with pattern matching
let (CustomerId id) = someCustomerId
```

### Active Patterns

Use active patterns to make `match` expressions more readable when the logic for a case is non-trivial.

```fsharp
let (|IsAutomation|IsHuman|) (actor: Actor) =
    match actor.Kind with
    | Automation _ -> IsAutomation
    | _            -> IsHuman
```

---

## F# Compiler Behavior — Things to Know

### Compile Order Is Explicit and Mandatory

Unlike C#, F# files compile in the order listed in the `.fsproj`. A type used before it is defined is a compile error. This is by design — it prevents circular dependencies.

```xml
<Compile Include="Core.fs" />      <!-- Actor, Event, Command, View -->
<Compile Include="Handlers.fs" />  <!-- uses Event, Command, View from Core.fs -->
<Compile Include="GWT.fs" />       <!-- uses Event, Command, View -->
<Compile Include="Slice.fs" />     <!-- uses CommandHandler, EventHandler, GWT -->
<Compile Include="Path.fs" />      <!-- uses SliceRef -->
<Compile Include="Grouping.fs" />  <!-- uses SliceRef -->
```

If you get `The type 'X' is not defined` and the type clearly exists — check the compile order in the `.fsproj`.

### `module` vs `namespace`

- `namespace` — types only, no `let` bindings at the top level. Use for type definitions.
- `module` — can contain both types and `let` bindings. Required when the file contains functions.

### Generic Type Parameters Must Be Used

F# infers generics. If you write a generic function and the type parameter cannot be resolved, you get an "unexpected type variable" error. Provide explicit type annotations at the binding site when inference is ambiguous.

```fsharp
// Explicit annotation on the binding resolves ambiguity
let mySlice : CommandSlice<Actor, StateType, CmdType, EventType> =
    ...
```

### `rec` Is Explicit

Recursive functions and mutually recursive types require the `rec` keyword. Without it, a function cannot call itself.

```fsharp
let rec groupingToTest registry grouping =
    match grouping with
    | Flow(name, refs)     -> testList name (List.map (refToTest registry) refs)
    | Workflow(name, children) -> testList name (List.map (groupingToTest registry) children)
    | Area(name, workflows)    -> testList name (List.map (groupingToTest registry) workflows)
```

For mutually recursive types, use `and`:

```fsharp
type A = { B: B }
and  B = { A: A }
```

### `open` Scoping

`open` is scoped to the file. Each file that needs a namespace must open it explicitly. There is no global `using` equivalent — this is intentional.

### Pattern Matching Is Exhaustive

The compiler warns (and can error) on incomplete match expressions. When you add a case to a discriminated union, the compiler identifies every match expression that needs updating.

```fsharp
type Grouping =
    | Flow     of name: string * slices: SliceRef list
    | Workflow of name: string * children: Grouping list
    | Area     of name: string * workflows: Grouping list
    // Adding a new case here flags every incomplete match immediately
```

### `DateTimeOffset` vs `DateTime`

Always use `DateTimeOffset` for timestamps. `DateTime` is ambiguous — it doesn't carry timezone information. `DateTimeOffset` always knows its offset from UTC.

```fsharp
OccurredAt = DateTimeOffset.UtcNow   // correct
OccurredAt = DateTime.UtcNow         // wrong type
```

In EventModeling GWT example data, use `DateTimeOffset.MinValue` as a sentinel — it signals "this timestamp is example data, not a real occurrence time." Test adapters strip `OccurredAt` before comparing events.

---

## Expecto Notes

### Test Functions Must Be `unit -> unit`

`testCase` takes a `string` label and a `unit -> unit` function. The function body uses `Expect.*` assertions. Any unhandled exception or `failtest` call fails the test.

```fsharp
testCase "my test" <| fun () ->
    Expect.equal actual expected "message"
```

### `<|` vs Parentheses

`testCase "label" <| fun () -> ...` is idiomatic. It avoids nested parentheses. `<|` is function application, right-associative: `f <| x` is `f x`.

### `testList` Nests Tests

`testList "name" [ test1; test2 ]` groups tests. The name appears in the output hierarchy.

### `runTestsWithCLIArgs`

The entry point for an Expecto test executable. Reads standard Expecto CLI arguments (`--filter`, `--summary`, etc.) and runs the provided `Test` value.

---

## Property-Based Testing

### Hedgehog

Hedgehog is the primary property-based testing tool. It generates random inputs, shrinks failing cases to the minimal counterexample, and integrates cleanly with Expecto via `testCase`.

**Core pattern**
```fsharp
testCase "label" <| fun () ->
    Property.checkBool <| property {
        let! x = Gen.int32 (Range.linear 0 100)
        return x >= 0
    }
```

`property { }` is a computation expression. Bind generators with `let!`. The final `return` takes a `bool` — `false` triggers shrinking and failure.

**`Property.check` vs `Property.checkBool`**

F# 5+ CE optimization: `property { let! x = gen; return bool }` uses `BindReturn` and produces `Property<bool>`, not `Property<unit>`. Use `Property.checkBool` when the property body ends with `return bool`. Use `Property.check` only when the body ends with `unit` (i.e. assertions via `Expect.*` rather than returning a bool).

**Common generators**
```fsharp
Gen.int32  (Range.linear 0 100)             // int in [0, 100]  (NOT Gen.int — renamed in 0.13.0)
Gen.string (Range.linear 1 20) Gen.alphaNum // non-empty alphanumeric string
Gen.bool                                    // true or false
Gen.list   (Range.linear 0 10) itemGen      // list of 0–10 items
Gen.choice [ gen1; gen2; gen3 ]             // randomly picks one generator
Gen.constant x                              // always produces x
Gen.map    f gen                            // transform output
gen { let! x = gen1; let! y = gen2; return ... }  // compose generators
```

**Generators for EventModeling types**

`EventModeling.Testing.Generators` provides ready-made generators for `Actor`, `ActorKind`, `Event<'T>`, `Command<'T>`, and `View<'T>`. Use these in consuming project tests rather than rewriting them.

**When to use Hedgehog**
- Invariants that must hold for all inputs (e.g. "event name is never empty")
- Roundtrip properties (serialize then deserialize yields same value)
- Domain laws (commutativity, associativity, idempotence)
- Handler correctness over a range of inputs rather than one GWT example

---

### CsCheck

CsCheck is a .NET property-based testing library used where Hedgehog falls short. Its strengths are:

- **Regression file support** — persists the minimal failing example to disk; reruns it first on subsequent runs, catching regressions immediately
- **Shrinking complex .NET types** — strong shrinkers for collections, strings, and types that Hedgehog's shrinking handles less well
- **Specific .NET type generators** — built-in generators for `DateTime`, `Guid`, `Uri`, collections, etc.

**F# usage**

CsCheck is a C# library; its API is usable from F# but less idiomatic. `Sample` is a static method on `Check`, not on `Gen<T>`:

```fsharp
open CsCheck

testCase "label" <| fun () ->
    Check.Sample(Gen.Int.[0, 100], fun n ->
        if n < 0 then failtest $"Expected non-negative, got {n}"
    )
```

`Gen.Int.[min, max]` uses F#'s `.[idx]` syntax to call the C# indexer on `GenInt`, returning `Gen<int>`. `Check.Sample(gen, action)` runs the property — the F# lambda is coerced to `Action<T>` automatically. Throw (or call `failtest`) to signal a failing case.

> Full API notes in `libs/cscheck.md`.

**When to use CsCheck over Hedgehog**
- When a failing Hedgehog case doesn't shrink to a readable minimal example
- When you need a generator for a complex .NET type that Hedgehog doesn't cover
- When regression replay (the persisted seed file) is important for CI stability

---

## Validation — Positive Match Only

Define what IS valid. Reject everything that does not match. Never enumerate what is invalid.

A denylist is always incomplete — forbidding `$` and `%` still allows `^`, `~`, zero-width spaces, and anything else not yet imagined. An allowlist is complete by definition: if it is not in the valid set, it is rejected.

**`String.forall` — character-level allowlist**
```fsharp
// Bad — denylist; always incomplete
let isValid (s: string) = not (s.Contains("$")) && not (s.Contains("%"))

// Good — allowlist; complete by definition
let isValid (s: string) =
    s.Length > 0 &&
    s |> String.forall (fun c -> Char.IsLetterOrDigit c || c = '-' || c = '_')
```

**Regex — anchored whitelist pattern**
```fsharp
open System.Text.RegularExpressions

// ^ and $ anchor to the full string — without them, [a-z]+ matches a substring of "abc123"
let validPattern = Regex(@"^[a-zA-Z0-9\-_]+$")

let isValid (s: string) = validPattern.IsMatch(s)
```

Always anchor regex patterns with `^` and `$`. An unanchored pattern tests for a matching *substring*, not a matching *input*.

**Smart constructor — positive validation at the boundary**
```fsharp
type ValidName = private ValidName of string

module ValidName =
    let create (s: string) =
        if s.Length > 0 && s |> String.forall (fun c -> Char.IsLetterOrDigit c || c = '-' || c = '_')
        then Ok (ValidName s)
        else Error $"Name must match [a-zA-Z0-9\\-_]+ — got: '{s}'"

    let value (ValidName s) = s
```

The error message states what IS valid, not what was rejected — same positive framing.

**Validate once, at the boundary only**

Validate at system entry points: user input, API requests, file imports. Inside the domain, values are already valid by construction (see: Parse, don't validate). Do not re-validate inside domain functions.

---

## Errors Encountered and Solutions

<!-- Add entries here as they occur. Format:
### Error: <compiler message or symptom>
**Context**: what you were doing
**Cause**: why it happened
**Fix**: what resolved it
-->

### Error: `The value, constructor, namespace or type 'int' is not defined`
**Context**: Using `Gen.int (Range.linear ...)` in a Hedgehog `property` block
**Cause**: In Hedgehog 0.13.0, `Gen.int` was renamed to `Gen.int32`
**Fix**: Replace `Gen.int` with `Gen.int32`

### Error: `This expression was expected to have type 'unit' but here has type 'bool'` inside `property { }`
**Context**: `Property.check <| property { let! x = gen; return someBoolean }`
**Cause**: F# 5+ CE `BindReturn` optimization — `property { let! x = gen; return bool }` produces `Property<bool>`, not `Property<unit>`. `Property.check` expects `Property<unit>`
**Fix**: Use `Property.checkBool` instead of `Property.check` when the property body ends with `return bool`

### Error: `Type constraint mismatch. The type 'Property<unit>' is not compatible with type 'Property<bool>'` — `for` loop inside `property { }`
**Context**: `property { let! n = gen; for _ in 1..n do sideEffect(); return bool }`
**Cause**: A `for` loop inside a `property { }` CE is desugared into CE `For` bindings that produce `Property<unit>`. The compiler then cannot reconcile the trailing `return bool` with the `Property<unit>` already in scope.
**Fix**: Replace the `for` loop with `List.init n (fun _ -> sideEffect())` bound to `_`. `List.init` is eager, runs all side effects, and is a plain F# expression — invisible to the CE:
```fsharp
// Wrong
property { let! n = gen; for _ in 1..n do store.Append sid [e] |> run |> ignore; return bool }

// Right
property { let! n = gen; let _ = List.init n (fun _ -> store.Append sid [e] |> run); return bool }
```

### Error: `The type 'Gen<_>' does not define the field, constructor or member 'Sample'`
**Context**: Calling `.Sample(...)` on a CsCheck `Gen<T>` value
**Cause**: `Sample` is a static method on `CsCheck.Check`, not an instance method on `Gen<T>`
**Fix**: `Check.Sample(gen, fun n -> ...)` with `open CsCheck`

### Error: `NotSupportedException: Deserialization of types without a parameterless constructor` — `[<CLIMutable>]` on private nested type
**Context**: `type private MyDto = { ... }` with `[<CLIMutable>]` inside a module, deserialized with `System.Text.Json`
**Cause**: `private` nested types are not visible enough for STJ's reflection-based deserialization, even with `[<CLIMutable>]`
**Fix**: Remove `private` from the type declaration. The type is still module-scoped; `private` only prevents STJ from instantiating it via reflection.
```fsharp
// Wrong — STJ cannot deserialize
[<CLIMutable>]
type private MyDto = { Field: string }

// Right — STJ can instantiate
[<CLIMutable>]
type MyDto = { Field: string }
```
