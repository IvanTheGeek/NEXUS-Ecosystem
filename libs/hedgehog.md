# Hedgehog — F# Property-Based Testing

- **NuGet**: `Hedgehog`
- **Version in use**: 0.13.0
- **GitHub**: https://github.com/hedgehogqa/fsharp-hedgehog
- **Targets**: `net6.0` / `netstandard2.0` (compatible with net10.0)

---

## Core Concepts

Hedgehog generates random inputs and automatically shrinks failing cases to the minimal counterexample. It integrates with Expecto via `testCase`.

```fsharp
open Hedgehog
open Expecto

testCase "my invariant" <| fun () ->
    Property.checkBool <| property {
        let! n = Gen.int32 (Range.linear 0 100)
        return n >= 0
    }
```

---

## Computation Expression: `property { }`

The `property { }` CE is defined in `[<AutoOpen>] module PropertyBuilder` inside the `Hedgehog` namespace. `open Hedgehog` brings it into scope.

### Binding generators

```fsharp
property {
    let! x = Gen.int32 (Range.linear 0 100)
    let! s = Gen.string (Range.linear 1 10) Gen.alphaNum
    return x >= 0 && s.Length > 0
}
```

### `Property.check` vs `Property.checkBool`

F# 5+ CE optimization: when the body ends with `let! x = gen; return bool`, the compiler uses `BindReturn` which produces `Property<bool>`, not `Property<unit>`.

| Pattern | Result type | Use |
|---|---|---|
| `property { ... return bool }` | `Property<bool>` | `Property.checkBool` |
| `property { ... Expect.isTrue ...; () }` | `Property<unit>` | `Property.check` |

**Rule**: if the last line of the CE is `return <bool expression>`, use `Property.checkBool`. If the last line is a `unit` assertion (Expecto `Expect.*`), use `Property.check`.

---

## Generators

### Integer

```fsharp
Gen.int32  (Range.linear 0 100)    // renamed from Gen.int in 0.13.0
Gen.int64  (Range.linear 0L 100L)
Gen.int16  (Range.linear 0s 100s)
```

### String

```fsharp
Gen.string (Range.linear 1 20) Gen.alphaNum   // non-empty alphanumeric
Gen.string (Range.linear 0 50) Gen.unicode    // unicode string
Gen.char 'a' 'z'                              // single char in range
Gen.alphaNum                                  // Gen<char> for [a-zA-Z0-9]
```

### Collections

```fsharp
Gen.list (Range.linear 0 10) itemGen    // list of 0–10 items
Gen.array (Range.linear 1 5) itemGen   // non-empty array
Gen.seq (Range.linear 0 5) itemGen     // sequence
```

### Choice / Composition

```fsharp
Gen.choice [ gen1; gen2; gen3 ]     // picks one generator uniformly
Gen.choiceRec nonrecs recs          // for recursive structures
Gen.constant x                      // always produces x
Gen.map f gen                       // transform output
Gen.filter p gen                    // discard values where p is false (use sparingly)
gen { let! x = g1; let! y = g2; return (x, y) }  // compose via CE
```

---

## Running Properties

```fsharp
Property.check       : Property<unit> -> unit
Property.checkBool   : Property<bool> -> unit
Property.checkWith   : PropertyConfig -> Property<unit> -> unit

// Reports (don't throw, return Report)
Property.report      : Property<unit> -> Report
Property.reportBool  : Property<bool> -> Report
```

---

## Known Issues / Gotchas

### `Gen.int` renamed to `Gen.int32` in 0.13.0
```fsharp
Gen.int  (Range.linear 0 100)   // FS0039: 'int' is not defined
Gen.int32 (Range.linear 0 100)  // correct
```

### `return bool` in `property { let! ... }` needs `checkBool`
See `Property.check vs Property.checkBool` section above.

### `open CsCheck` conflicts with Hedgehog `Gen<'T>`
If both `open Hedgehog` and `open CsCheck` are in the same file, the `Gen<'T>` type is ambiguous. The `property` CE `Bind` method requires `Hedgehog.Gen<'a>` — if `Gen` resolves to CsCheck's `Gen`, the CE breaks.
**Solution**: The conflict does NOT occur in practice because CsCheck's `Gen` is a static C# class, not a generic type. However, if namespace conflicts appear, qualify with `Hedgehog.Gen<...>` or use `CsCheck.Gen.*` for CsCheck generators.
