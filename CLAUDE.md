# Nexus Workspace

> Sync contract: when `FSHARP.md` or `NEXUS-EventModeling/EVENT_MODELING.md` is updated, update the corresponding condensed section below.

## NEXUS Components

> Full reference: `NEXUS-LOGOS/docs/architecture/components.md`

| Component | Role |
|---|---|
| **NEXUS-EventModeling** | F# vocabulary — Actor, Command, Event, View, Slice, GWT; the Event Model lens in library form |
| **NEXUS-STRATUM** | Event persistence — `IEventStore` with pluggable backends (InMemory, FileSystem, Dropbox, etc.) |
| **NEXUS-FnHCI** | Universal HCI abstraction — adapters for HTML, Blazor (.NET 10), TUI, LCD, mobile, desktop, etc. |
| **NEXUS-LOGOS** | NEXUS knowledge base — principles, ADRs, concept definitions, methodology docs |
| **NEXUS-ATLAS** | Modeling workspace — applies lenses to Truth Graph → Spec; renders event timelines |
| **NEXUS-FORGE** | Language-agnostic code generator — Spec + lenses → F#, JS, C++, Rust, or any target |
| **CORTEX** | External AI layer — Claude, GPT, Gemini; reads LOGOS, drives ATLAS sessions, runs FORGE |
| **NEXUS-admin** | Workspace scaffolding — templates, checklists, new-project bootstrap |

**Truth Graph** is the real Source of Truth — a lens-independent graph of facts built from LOGOS records. The Event Model is one lens applied to it; FnHCI targets and API contracts are other lenses. Full pipeline: `NEXUS-LOGOS/docs/architecture/pipeline.md`

## External Libraries

When working with any external library or tool:
1. **Research before writing code** — check the assembly XML docs (`.xml` alongside the `.dll` in the NuGet cache), the library's GitHub repo (README, source, issues), and official docs site
2. **Record what you learn** — create or update `libs/<library>.md` in this workspace; it applies to all NEXUS projects
3. **Record errors** — add failures and their fixes to both `libs/<library>.md` and `FSHARP.md ## Errors Encountered`

`libs/` is the growing reference for every library used across NEXUS. Future sessions and other AI should read it before using a library.

## Structure

- All projects live under /home/ivan/nexus/
- Each project is a subdirectory
- Language: F# on .NET 10.0 across all projects

## Git Workflow

> Full reference: `WORKFLOW.md` (this directory).

- Always branch — `feature/`, `experiment/`, `fix/`, `docs/`, `refactor/`
- Commit often — small commits capturing progression as it happens
- Merge to `main` with `--no-ff` — always; preserves branch topology
- Abandoned work: merge to `graveyard` with `--no-ff`, then delete the branch
- Delete branches after merge — history lives in the merge commit
- Never rewrite pushed history — no rebase, no amend, no force push
- Stop hook auto-commits + pushes at end of each Claude session (per-project `.claude/settings.json`)

## Event Modeling

> Condensed from `NEXUS-EventModeling/EVENT_MODELING.md`. Update that file AND this section together.

- **Actor**: source of decisions and actions — `Human of role`, `Automation of name`, `ExternalSystem of name`
- **Command**: intent expressed by an Actor; handled by a hidden pure CommandHandler; blue box
- **Event**: immutable fact, past tense, never changes; the backbone and the only coupling point; orange box
- **View**: shaped data produced by a hidden EventHandler for an Actor to consume; green box
- **CommandSlice**: `Actor → Command → Event(s)` — data flows top-down; state change
- **ViewSlice**: `Event(s) → View → Actor` — data flows bottom-up; state view
- **Cadence**: CommandSlice → ViewSlice → CommandSlice → ViewSlice (sine wave)
- **GWT**: Given/When/Then — simultaneously the business spec and the test suite; example data IS the tests
  - CommandSlice GWT: Given = prior events (state), When = command, Then = resulting events
  - ViewSlice GWT: Given = events, When = filter/sort/transform criteria, Then = shaped view
- **Flow**: smallest grouping — short sequence of slices for one focused interaction
- **Workflow**: series of Flows or nested Workflows for a larger process; nestable, no depth limit
- **Area**: top-level grouping of related Workflows
- Groupings are header rows above the grid — purely organizational, no logic
- **Slice immutability**: finalized slices are replaced, never modified; no code shared between slices
- **Conway's Law**: event rows can mirror team/system/bounded-context boundaries
- Handlers are hidden; GWT is visible — GWT rows are the specification handlers implement
- Handlers are pure functions — no side effects, no I/O, no state

## F#

> Condensed from `FSHARP.md`. Update that file AND this section together.

**Scott Wlaschin / Domain Modeling Made Functional**
- Make illegal states unrepresentable — encode invariants in types, not prose
- Use discriminated unions instead of strings/bools/ints for domain concepts
- Total functions: if a function can fail, encode it in the return type (`Result`, `option`) — never throw for domain failures
- Parse, don't validate: validate at the boundary; inside the domain everything is already valid by construction
- Validate with positive match only — define the valid set (`String.forall`, anchored regex `^...$`, smart constructors); never denylist
- Use records for all data types — immutable by default
- Compose functions; F# has no data inheritance

**Compiler rules**
- Compile order in `.fsproj` is mandatory — a type used before it is defined is a compile error
- `namespace` for type-only files; `module` when the file contains `let` bindings/functions
- Recursive functions require `rec`; mutually recursive types require `and`
- Pattern matching is exhaustive — adding a DU case will flag every incomplete match
- `open` is file-scoped — every file that needs a namespace must open it explicitly

**Types and time**
- Always `DateTimeOffset` UTC — never `DateTime`; timezone conversion is a view concern
- `Result<'T, string>` for failures; the string is a business rule violation message
- Single-case DUs wrap primitives to prevent mixing incompatible IDs (e.g. `CustomerId of int`)

**Testing — Expecto + Hedgehog + CsCheck**
- Full stack ships with `EventModeling.Testing` — reference it to get all three
- `testCase "label" <| fun () -> ...` — idiomatic Expecto; `<|` avoids nested parens
- `testList "name" [...]` — groups tests; `runTestsWithCLIArgs [] args allTests` — entry point
- Hedgehog: `property { let! x = gen; return bool }` + `Property.checkBool` inside `testCase`
- Hedgehog: use for invariants, roundtrips, domain laws; `Gen.int32`, `Gen.string`, `Gen.choice`
- CsCheck: use where Hedgehog falls short — regression files, complex shrinking, .NET type generators
- CsCheck F#: `open CsCheck` then `Check.Sample(Gen.Int.[min, max], fun n -> if bad then failtest "...")`
- Full API notes and gotchas: `libs/hedgehog.md`, `libs/cscheck.md`
