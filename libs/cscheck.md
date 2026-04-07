# CsCheck — .NET Property-Based Testing

- **NuGet**: `CsCheck`
- **Version in use**: 4.6.2
- **GitHub**: https://github.com/AnthonyLloyd/CsCheck
- **Target**: `net8.0` (compatible with net10.0)
- **Author**: Anthony Lloyd

---

## When to Use CsCheck vs Hedgehog

| Use CsCheck when | Use Hedgehog when |
|---|---|
| Regression replay matters (persists failing seed to disk) | Standard property/invariant checking |
| Complex .NET type generators (`DateTime`, `Guid`, `Uri`, etc.) | F#-idiomatic CE syntax |
| Shrinking complex types | Composing generators via `gen { }` CE |
| Parallel testing / `Check.SampleParallel` | Domain laws, roundtrips |

---

## Core API

CsCheck is a C# library. All generators are in the `CsCheck.Gen` static class. All runners are in the `CsCheck.Check` static class.

```fsharp
open CsCheck
```

### Generators — `Gen.*`

`Gen` is a static C# class. Primitive generators are **public static fields**:

```fsharp
Gen.Int        // CsCheck.GenInt — ranged int generator
Gen.Long       // CsCheck.GenLong
Gen.String     // CsCheck.GenString
Gen.Bool       // Gen<bool>
Gen.Guid       // Gen<Guid>
Gen.DateTime   // Gen<DateTime>
Gen.DateTimeOffset  // Gen<DateTimeOffset>
```

Apply a range via C# indexer syntax:
```fsharp
Gen.Int.[0, 100]        // Gen<int> in [0, 100]
Gen.Long.[0L, 1000L]    // Gen<long>
Gen.String.[1, 20]      // Gen<string> of length 1–20
```

### Runners — `Check.*`

`Sample` is a **static method on `Check`**, not an instance method on `Gen<T>`:

```fsharp
// Assert — throw / failtest on failure
Check.Sample(Gen.Int.[0, 1000], fun n ->
    if n < 0 then failtest $"Expected non-negative: {n}"
)

// Bool — return false to fail
Check.Sample(Gen.Int.[0, 1000], Func<int, bool>(fun n -> n >= 0))
```

**Common mistake**: calling `.Sample(...)` on `Gen<T>` — `Sample` does not exist on `Gen<T>`. It is on `Check`.

---

## Regression Replay

CsCheck persists failing seeds to disk, allowing CI to re-run the exact failing case on subsequent runs. This is the primary reason to prefer CsCheck over Hedgehog for regression-sensitive tests.

```fsharp
// Seed file is created automatically next to the test assembly when a failure is found
// On next run, the failing seed is replayed first before running new random cases
Check.Sample(Gen.Int.[0, 1000], fun n ->
    // if this fails, CsCheck saves the seed
    Expect.isTrue (someInvariant n) "invariant"
)
```

---

## Known Issues / Gotchas

### `Gen.Int.[0, 1000].Sample(...)` does not compile
`Sample` is on `Check`, not `Gen<T>`. Use:
```fsharp
Check.Sample(Gen.Int.[0, 1000], fun n -> ...)
```

### `open CsCheck` must be present for `Gen.Int.*` to resolve
`Gen.Int` is a static field on the `CsCheck.Gen` class. Without `open CsCheck`, you must qualify it as `CsCheck.Gen.Int.[0, 1000]`.

### F# lambda is coerced to `Action<T>` automatically
F# lambdas passed to `Check.Sample(gen, fun n -> ...)` are implicitly coerced to `System.Action<T>`. No explicit `System.Action<int>(fun n -> ...)` wrapper needed in practice.
