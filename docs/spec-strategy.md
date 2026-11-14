# Spec Strategy (Verified)

A reusable approach to specifying software that will be formally verified. The spec
carries everything the prover needs except the proof itself. Applicable to any Rust
project verified with Kani; templates lean toward bare-metal and OS work.

---

## Philosophy

### Discuss Before Writing
Specs emerge from conversation, not isolation. Start broad, narrow gradually through
discussion. The spec is a record of decisions made, not a document written in a vacuum.

### Specs Are Living Documents
Specs change as understanding deepens and implementation reveals reality. Update them
continuously. A stale spec is worse than no spec - it actively misleads.

### Separate Concerns, Link Liberally
Split specs by concern (hardware, stack, verification, subsystems) but cross-reference
heavily. When working on a subsystem, you should know exactly which shared specs are
relevant.

### Capture Decisions, Not Just Outcomes
Don't just write down *what* was decided - write down *why*. Specs capture what you
chose, what you considered, and why. Conversations get lost. Knowing what was rejected
prevents revisiting the same debates. In a verified project, many decisions are made
specifically to keep something provable; record that too.

### The Spec Is the Prover's Input
Plain spec work asks "what goes in, what comes back" so you can code against an
interface. Verified spec work asks "what is always true, what must hold before, what is
guaranteed after, and what do we trust without proof." Those are prover inputs. They
live in the spec, written and approved before implementation. Implementation encodes
them; it does not invent them.

### Nothing Is Verified by Accident
Every property has a declared disposition: verify, test, or trust. A property with no
disposition is an open question. A property that was meant to be verified and ends up
tested is a recorded decision with a reason, not a quiet downgrade.

### Trust Is Declared
Everything the proofs rest on without proof - hardware behavior, assembly stubs, the
prover itself, barrier placement - is written down as a trust assumption with an ID.
The trusted computing base is the sum of those IDs, and it is visible.

---

## Document Structure

### Four Layers

```
┌─────────────────────────────────────────────┐
│  Planning                                   │
│  - Build order, what to build first and why │
│  - Which properties gate which steps        │
└─────────────────────────────────────────────┘
                      ↓
┌─────────────────────────────────────────────┐
│  Platform Specs                             │
│  - Hardware (board, SoC, memory map)        │
│  - Stack (toolchain, targets, tools)        │
│  - Verification (prover, TCB, rules, IDs)   │
│  - Properties (registry / traceability)     │
└─────────────────────────────────────────────┘
                      ↓
┌─────────────────────────────────────────────┐
│  Subsystem Specs                            │
│  - One per module or driver                 │
│  - References platform specs                │
│  - Owns its formal content (below)          │
└─────────────────────────────────────────────┘
                      ↓
┌─────────────────────────────────────────────┐
│  Formal Layer (inside each subsystem spec)  │
│  - Abstract model (Rust, compiled)          │
│  - Invariants, contracts, properties        │
│  - Trust assumptions, harness plan          │
└─────────────────────────────────────────────┘
```

### Recommended Files

```
docs/
├── spec-strategy.md              # This file - methodology
├── spec/                         # Platform-level shared specs
│   ├── index.md                  # Entry point - links to platform specs + subsystems
│   ├── hardware.md               # Board, SoC, cores, memory map, peripherals, boot, bench
│   ├── stack.md                  # Toolchain, targets, prover version, build matrix
│   ├── verification.md           # Prover, assurance levels, TCB, ID scheme, design rules
│   ├── properties.md             # Property registry - every ID, disposition, status, harness
│   ├── build-order.md            # Platform-level implementation order
│   ├── model.rs                  # Platform-level model types shared by subsystems
│   ├── templates/                # Templates for bootstrapping new subsystems
│   └── archive/                  # Historical specs (completed build orders, etc.)
│
├── <subsystem>/                  # One per subsystem, module, or driver
│   └── spec/
│       ├── index.md              # Subsystem overview + Related Platform Specs table
│       ├── build-order.md        # Subsystem-specific build order (if multi-step)
│       ├── <subsystem>.md        # The subsystem spec (may be split into several docs)
│       └── model.rs              # The subsystem's abstract model, compiled
│
└── <another-subsystem>/
    └── spec/
        └── ...

model/                            # The model crate - includes every model.rs above
├── Cargo.toml
└── src/lib.rs
```

### Subsystems

Specs split into **platform** and **subsystem** layers:

- **Platform specs** (`docs/spec/`) cover what every subsystem shares: the hardware,
  the toolchain, the verification approach, the property registry, and shared model
  types.
- **Subsystem specs** (`docs/<subsystem>/spec/`) cover one module or driver: a memory
  manager, a UART driver, a NAT table, a cross-core ring. Each has its own index, its
  own model, and its own formal content.

Every subsystem index starts with a **Related Platform Specs** table pointing back to
the shared specs it depends on.

**When to create a new subsystem directory:**
- It has its own state, its own invariants, or its own hardware
- It has a single owning core (see State Ownership)
- It could be implemented and verified somewhat independently

**When NOT to create a subsystem directory:**
- It's one operation of an existing subsystem (add it to that spec instead)
- It's platform infrastructure (belongs in `docs/spec/`)

**Bootstrapping a new subsystem:**
1. Create `docs/<subsystem>/spec/`
2. Copy `docs/spec/templates/index.md.template` → fill in overview
3. Copy `docs/spec/templates/subsystem.md.template` → the spec itself
4. Copy `docs/spec/templates/model.rs.template` → `model.rs`
5. Register the subsystem tag in `docs/spec/properties.md`
6. Add the module to `model/src/lib.rs` so the model compiles
7. Add the subsystem to the platform `docs/spec/index.md` subsystems table
8. Fill in through discussion, same as any other spec work

### Index Structure

The index is the entry point. It should show:
1. What the project is (one paragraph)
2. All spec documents with status
3. High-level scope/phases

```markdown
## Spec Components

### Platform Specs
| Document | Status | Description |
|----------|--------|-------------|
| [Hardware](hardware.md) | ✅ Complete | Board and SoC |
| [Stack](stack.md) | ✅ Complete | Toolchain |
| [Verification](verification.md) | 🔲 Draft | Prover, TCB, rules |
| [Properties](properties.md) | 🔲 Partial | Registry |

### Subsystems
| Subsystem | Spec | Verification | Description |
|-----------|------|--------------|-------------|
| [NAT](../nat/spec/index.md) | 🔲 Draft | 0/6 verified | Connection tracking |
```

---

## The Formal Layer

This is what distinguishes a verified spec. Every subsystem spec contains the
following, and the model portion is real Rust that compiles from day one.

### Abstract Model

A mathematical description of the subsystem's state, independent of how it will be
implemented. A routing table is a map from prefix to next hop, not a trie. A buffer
pool is a set of free indices, not a linked list threaded through memory.

The model is written in Rust in `model.rs` beside the spec doc. It uses fixed-capacity
structures parameterized by const generics so a harness can shrink them. It is
`no_std`, forbids `unsafe`, has no dependencies beyond `core`, and builds on the host.

Proofs are refinement: the implementation provides an abstraction function from its
concrete state to the model state, and every property is checked against the model.

### Invariants

What is always true of the model state between operations. Each is a boolean predicate
in `model.rs`, carries an ID in its doc comment, and is stated in one sentence of prose
in the spec doc.

```rust
/// INV-NAT-003: no two live entries share an external (addr, port).
pub fn inv_unique_external<const N: usize>(s: &NatState<N>) -> bool { ... }
```

A subsystem's invariant `inv(s)` is the conjunction of all its INV predicates. Every
operation must preserve it.

### Operations and Contracts

Every operation the subsystem exposes gets a requires clause and an ensures clause,
stated against the model. These become `#[kani::requires]` and `#[kani::ensures]` on
the implementation.

Operations are named by their Rust path in the pure crate: `nat::allocate_port`,
`ring::push`. The compiler enforces uniqueness. Do not invent a separate ID scheme for
operations.

Contracts can be written two ways in `model.rs`; use whichever reads better:

- **Relational:** `pre_op(s, args) -> bool` and `post_op(old, new, args, result) -> bool`
- **Functional:** `model_op(s, args) -> (NatState<N>, Result)` - a pure reference
  implementation the real one must agree with

Functional is clearer for deterministic operations. Relational is needed when the spec
permits several outcomes.

### Properties

System-level claims, usually spanning several invariants and contracts. "A packet is
never forwarded out an interface not named by its route." Each has an ID, a prose
statement, a predicate or harness description, and a list of the INVs and operations it
rests on.

Safety properties only. Liveness ("eventually forwarded") is out of reach for bounded
model checking and is not claimed.

### Trust Assumptions

What the subsystem relies on without proof. Hardware behaves per the reference manual.
An assembly stub preserves registers. A memory barrier is placed correctly. Each gets an
ID, one sentence, and a note on how it is mitigated (review, test on hardware, vendor
documentation).

Every `unsafe` block in the implementation cites a TA ID in its safety comment.

### Hardware Interface Model

Drivers only. The register and descriptor layout the driver believes, stated as the
contract we assume from the reference manual: offsets, bit fields, read/write
semantics, ordering requirements. This is modeled in `model.rs` as types and encoding
predicates, so Kani can check that the driver's encoding logic honors the model. The
model is itself a trust assumption (it is our reading of the manual), and gets a TA ID.

Hardware access in the implementation goes through a trait, so harnesses substitute a
model device for the real one.

### Harness Plan

Per property: what is symbolic, what is assumed, what the bound is, what the unwind
limit is. Not the harness itself - that depends on the concrete API, which is
implementation - but enough that writing the harness is mechanical.

```markdown
| Property | Symbolic | Assumed | Bound | Unwind |
|----------|----------|---------|-------|--------|
| INV-NAT-003 | state, packet header | inv(state) | N=4 | 5 |
```

### Verification Disposition

A table of every INV, PROP, and TA in the subsystem with its disposition and current
status. This is the subsystem's slice of the platform property registry; the two must
agree.

### The Model Crate

A single crate, conventionally `model/`, includes every `model.rs` by path:

```rust
// model/src/lib.rs
#![no_std]
#![forbid(unsafe_code)]

#[path = "../../docs/spec/model.rs"]
pub mod platform;

#[path = "../../docs/nat/spec/model.rs"]
pub mod nat;
```

Subsystem models import shared types with `use crate::platform::*`. The crate builds on
the host with `cargo check -p model` and is a dependency of every pure crate and every
harness. The spec is therefore type-checked before any implementation exists, and
there is exactly one copy of the model.

### The Refinement Pattern

Implementation provides `fn abs(&Impl) -> Model`. A harness for an operation looks like:

```
let mut s: Impl = kani::any();          // symbolic concrete state
kani::assume(inv(&abs(&s)));            // start from a valid state
let args = kani::any();
kani::assume(pre_op(&abs(&s), &args));
let old = abs(&s);
let r = s.op(args);
assert!(inv(&abs(&s)));                 // invariant preserved
assert!(post_op(&old, &abs(&s), &args, &r));
```

This is inductive over time and bounded only in instance size. See Assurance Levels in
the verification spec for what a bounded result means and what argument accompanies it.

---

## Identifiers

| Kind | Form | Example | Lives in |
|------|------|---------|----------|
| Invariant | `INV-<TAG>-nnn` | `INV-NAT-003` | Subsystem spec + `model.rs` doc comment |
| Property | `PROP-<TAG>-nnn` | `PROP-DP-001` | Subsystem spec (or platform for cross-cutting) |
| Trust assumption | `TA-<TAG>-nnn` | `TA-FMAN-002` | Subsystem spec, cited by `unsafe` blocks |
| Operation | Rust path | `nat::allocate_port` | Subsystem spec + the code |

Rules:
- `<TAG>` is a short uppercase subsystem tag registered in `properties.md`. Platform
  tag is `PLAT`.
- Numbers are three digits, assigned in order, never reused. A retired ID stays in the
  registry marked Retired with a reason.
- An ID appears in exactly one spec doc as its home; everywhere else is a reference.
- Operation names must match the real Rust path. If the function moves, the spec is
  updated in the same change.

---

## Status Tracking

### Spec Status

| Status | Meaning |
|--------|---------|
| ✅ Complete | Spec written and approved, formal content present, ready to implement |
| 🔲 Pending | Placeholder exists, content not written |
| 🔲 Partial | Some sections done, others TODO |
| 🔲 Draft | Written but not reviewed/approved |

Mark sections within docs too.

### Verification Disposition and Status

Every INV, PROP, and TA carries two fields.

**Disposition** - the plan:

| Disposition | Meaning |
|-------------|---------|
| Verify | Will be checked by a Kani harness |
| Test | Will be covered by tests on host or hardware, by recorded decision |
| Trust | Assumed without proof, by recorded decision (all TAs are Trust) |

**Status** - the progress:

| Status | Meaning |
|--------|---------|
| 🔲 Specified | Stated in spec with model predicate; no harness yet |
| 🔲 Harnessed | Harness exists; not yet passing, or passing at a bound below plan |
| ✅ Verified (N) | Harness passes at bound N; size-independence argument recorded if N is below capacity |
| ✅ Tested | Tests exist and pass |
| ✅ Trusted | Recorded with mitigation |

### Rollup Rules

- A subsystem spec is ✅ Complete only when every INV, PROP, and TA has a disposition.
- A subsystem is verified only when every item is at a ✅ status matching its
  disposition.
- The platform index shows both axes per subsystem: spec status and `k/n verified`.

---

## Gates

### Implementation Gate

Do not suggest or begin implementation of a subsystem unless ALL of these are true:

1. **No unresolved TODOs** in its spec
2. **No open questions** - discussed and answered
3. **User has approved** - reviewed and agreed, not just written
4. **Status is ✅ Complete** - explicitly marked
5. **Dependencies ready** - what it depends on is implemented or well enough defined
6. **Model compiles** - `model.rs` exists and `cargo check -p model` passes
7. **Invariants enumerated** - every INV has a predicate in `model.rs`
8. **Contracts written** - every operation has requires and ensures against the model
9. **Dispositions assigned** - every INV, PROP, TA has one, with bound for Verify items
10. **Core ownership declared** - the subsystem names its owning core and its rings
11. **Big picture understood** - fits the architecture, no conflicts with other specs

If any are not met, stop and address them first. Don't approximate, don't wing it.

### Verification Gate

A subsystem is not done when the code compiles and runs. It is done when:

1. Every Verify item has a passing harness at its planned bound, and the registry is
   updated to ✅ Verified (N)
2. Every Test item has passing tests and the registry says ✅ Tested
3. Every TA is cited by at least one `unsafe` block or trust boundary in code, or is
   retired
4. The harness plan in the spec matches the harnesses that exist

### Downgrades

Changing a disposition from Verify to Test or Trust is allowed. It is a decision: add a
row to the subsystem's Decisions table stating what was tried, why it did not work, and
what replaces it. Changing a bound downward is the same. Silent downgrades are the
failure mode this process exists to prevent.

---

## Design for Verification

Platform rules every subsystem inherits. They exist so the code is provable, and they
are stated in `verification.md`. The defaults:

1. **Pure crates are pure.** `no_std`, `#![forbid(unsafe_code)]`, no heap, no hardware
   dependency, build and test on the host. All logic lives here.
2. **Hardware is behind traits.** Register access, DMA descriptors, and anything with
   side effects is reached through a trait. Harnesses substitute a model device.
3. **Encoding is explicit.** Wire formats and register layouts are built with explicit
   byte and bit operations (`from_be_bytes`, shifts, masks). Never overlay a struct on a
   buffer, never `transmute`, never depend on native endianness. The host and the target
   may not agree.
4. **Capacity is a parameter.** Fixed-size structures take their capacity as a const
   generic or a named const. Harnesses verify at small N; the deployed N is a different
   instantiation of the same code.
5. **Loops have stated bounds.** Every loop's bound is derivable from a capacity const.
   The harness plan records the unwind limit.
6. **No panic paths on the hot path.** No `unwrap`, `expect`, or unchecked indexing in
   pure crates on the data path. Kani checks absence of panics; the rule keeps the
   harness from finding them.
7. **State has one owner.** See below.
8. **`unsafe` cites its TA.** Every `unsafe` block's safety comment names the TA ID it
   relies on.

Add project-specific rules in `verification.md`.

---

## State Ownership and Concurrency

Verifying concurrent code is an order of magnitude harder than sequential code. The
default discipline avoids it:

- Every piece of state names exactly one owning core (or one owning thread, for hosted
  projects). Only that owner reads or writes it.
- Cross-owner communication is through explicit single-producer single-consumer rings.
  Nothing else is shared.
- The rings are the one place where weak memory ordering matters. Barrier placement in
  the ring implementation is a declared trust assumption, mitigated by review and by
  testing on hardware. Kani does not model weak memory.
- Interrupt handlers either touch no state the main loop touches, or the core does not
  take interrupts. A core that polls and never takes interrupts is sequential, and its
  proofs are easy.

Every subsystem spec declares: owning core, rings it produces to, rings it consumes
from, whether its core takes interrupts.

---

## Build Order

Every project has a build order that answers:
1. What do we build first?
2. Why that order? (dependencies)
3. What spec sections must be complete for each step?
4. Which property IDs must be verified before the next step can start?

Format:

```
Step 1: [Name]
  What: Brief description
  Why first: Dependency explanation
  Spec: Which doc/section
  Gates: INV-X-001, PROP-X-001 verified
  Status: ✅/🔲
      ↓
Step 2: [Name]
  ...
```

Property gates make the build order honest. "Memory manager done" means its invariants
hold, not that it compiles.

---

## Platform Spec Contents

### Hardware Spec

1. **Board and SoC** - what it is, revision, links to reference manuals
2. **Cores** - count, ISA, endianness, caches, what each core is assigned to
3. **Memory map** - RAM, flash, MMIO regions the project touches
4. **Peripherals in scope** - which are used, which are explicitly ignored
5. **Boot path** - what runs before our code, how our image is loaded
6. **Bench** - console, debugger, how to flash, how to recover
7. **Decisions**

### Stack Spec

1. **Summary table** - language, toolchain, target, prover, build tool, debugger
2. **Toolchain** - compiler version pinning, target spec, linker, binutils
3. **Prover** - Kani version, how it is invoked, which crates it runs on
4. **Build matrix** - which crates build for host, target, or both; which run Kani
5. **Development tools** - emulator if any, flashing, serial, tracing
6. **Data flow diagram** - how pieces connect at runtime
7. **Decisions**
8. **Future considerations**

### Verification Spec

1. **Prover and approach** - the tool, what kind of result it produces
2. **Assurance levels** - what "Verified (N)" means, what argument accompanies a bound
   below capacity, what Tested and Trusted mean
3. **Trusted computing base** - the global list of what is trusted: hardware, boot
   loader, toolchain, prover, assembly, plus a pointer to every TA
4. **ID scheme** - as above, with the tag registry pointer
5. **Harness conventions** - naming (`check_inv_nat_003`), location, how contracts and
   stubs are used, how bounds are passed
6. **Design rules** - the platform defaults above plus project-specific additions
7. **Concurrency model** - the ownership discipline for this project, which cores take
   interrupts, which rings exist
8. **Decisions**

### Property Registry

One row per ID across the whole project:

```markdown
| ID | Subsystem | Statement | Disposition | Status | Bound | Harness |
|----|-----------|-----------|-------------|--------|-------|---------|
| INV-NAT-003 | NAT | No two live entries share external (addr, port) | Verify | ✅ Verified (4) | N=4 | crates/nat/src/harness.rs |
```

Plus the tag registry: tag, subsystem, spec path. This is the traceability matrix.
Manual at first; it may be generated from spec docs later, in which case the generator
is the source of truth and the doc says so.

---

## Subsystem Spec Contents

Each subsystem spec covers:

1. **Related specs** - links to platform specs and sibling subsystems
2. **Overview** - what it does, from the system's perspective
3. **Requirements** - must, should, nice to have
4. **Ownership** - owning core, rings produced and consumed, interrupts
5. **Abstract model** - prose description plus pointer to `model.rs`
6. **Invariants** - table of IDs, prose, predicate name
7. **Operations** - each with signature, requires, ensures, and errors
8. **Properties** - table of IDs, prose, what they rest on
9. **Hardware interface model** - drivers only
10. **Trust assumptions** - table of IDs, prose, mitigation
11. **Harness plan** - per property: symbolic, assumed, bound, unwind
12. **Verification disposition** - the subsystem's slice of the registry
13. **Error handling** - what can fail and what happens
14. **Decisions table** - with verification impact column
15. **Open questions** - checklist

---

## Decisions Tables

Any spec that involves meaningful choices includes a Decisions table. In a verified
project, add a column for verification impact:

```markdown
## Decisions

| Decision | Alternatives Considered | Why This Choice | Verification Impact |
|----------|------------------------|-----------------|---------------------|
| Fixed-size NAT table, N=4096 | Hash map with heap | No heap in kernel; bounded | Verified at N=4, size-independence argued |
| Dataplane core polls, no interrupts | NAPI-style hybrid | Sequential core, simple proofs | No concurrency on core 1 |
```

---

## Cross-Referencing

Every subsystem spec starts with a Related Specs table:

```markdown
## Related Specs

| Spec | Relevance |
|------|-----------|
| [Hardware](../../spec/hardware.md) | Registers and memory this subsystem touches |
| [Verification](../../spec/verification.md) | Assurance level, design rules, TCB |
| [Properties](../../spec/properties.md) | Registry rows for this subsystem |
| [Ring](../../ring/spec/index.md) | Command ring this subsystem consumes |
```

---

## Workflow

### Starting a New Project

1. Create `docs/spec/` and the `model/` crate
2. Create `index.md` with project overview
3. Discuss hardware and bench → write `hardware.md`
4. Discuss toolchain and prover → write `stack.md`
5. Discuss assurance level, TCB, ownership → write `verification.md`
6. Identify subsystems → create directories, register tags in `properties.md`
7. Discuss build order → write `build-order.md`
8. Work through build order, filling in subsystem specs and models as you go

### During Development

1. Before implementing: check the subsystem spec and its registry rows
2. During implementing: note any spec changes needed; keep `model.rs` as the source
3. After implementing: run harnesses, update registry status, update specs to reflect
   reality
4. Mark sections complete as you go

### New Session / New Contributor

1. Read `CLAUDE.md` (if AI) or `README.md` (if human)
2. Read `docs/spec/index.md` for overview
3. Check `build-order.md` for current focus
4. Read the relevant subsystem spec, its `model.rs`, and its related specs
5. Continue work

---

## Spec Before Implementation

### Default Mode: Discussion

The default mode is **discussion**, not implementation. Spec first, model second, code
third. Don't suggest implementation until the gates above are met.

### What's NOT Required

Don't over-spec. These can emerge during implementation:
- The harnesses themselves (the plan is enough)
- The abstraction function from concrete to model state
- Performance optimization details
- Implementation internals (spec the *what*, not the *how*)
- Folder layout (see Project Structure)

What IS required, and is not optional in a verified project:
- Every edge case that affects an invariant. "Some you'll discover as you build" is
  true of UI edge cases; it is not true of the condition under which a buffer is freed
  twice.

---

## Project Structure

### The Layering Rule Comes Early, the Layout Comes Last

Plain spec work defers all structure until just before implementation. Here, one
structural rule is a verification constraint and is fixed in `verification.md` early:

```
┌───────────────────────────────────────────────┐
│  Binary / glue                                │
│  Entry point, assembly, linking, main loops   │
│  Target only. Minimal logic.                  │
├───────────────────────────────────────────────┤
│  Hardware crates                              │
│  MMIO, DMA, descriptors. Hold the unsafe.     │
│  Expose traits. Target only, except models.   │
├───────────────────────────────────────────────┤
│  Pure crates                                  │
│  All logic. no_std, no unsafe, no heap.       │
│  Host and target. Kani runs here.             │
├───────────────────────────────────────────────┤
│  Model crate                                  │
│  Abstract state and predicates from the spec. │
│  Host and target. Depended on by everything.  │
└───────────────────────────────────────────────┘
```

Dependencies point downward only. The pure layer never names a register. The hardware
layer never contains a routing decision.

Folder layout, crate names, and module boundaries within a layer are still decided
last, after the subsystems and their relationships are clear.

### Host / Target Matrix

| Layer | Builds on host | Builds on target | Kani | Tests |
|-------|----------------|------------------|------|-------|
| Model | yes | yes | referenced | yes |
| Pure | yes | yes | yes | yes |
| Hardware | models only | yes | encoding logic via stubs | on hardware |
| Binary | no | yes | no | on hardware |

---

## Anti-Patterns

### Don't Do This

- **Monolithic spec** - one giant document is hard to navigate and update
- **Orphan specs** - subsystem specs with no links to platform specs
- **Write-once specs** - written at start and never updated
- **Prose-only invariants** - an invariant without a predicate in `model.rs` is a wish
- **Silent downgrades** - verify became test and nobody wrote down why
- **Undeclared trust** - an `unsafe` block with no TA, a barrier with no TA
- **Model drift** - a copy of the model living in the implementation crate
- **Vague TODOs** - "TODO: figure this out"

### Do This Instead

- **Split by concern** - easier to find, easier to update
- **Cross-reference** - Related Specs table in every subsystem doc
- **Living documents** - update as you learn and build
- **Predicate for every invariant** - compiled, in one place
- **Recorded downgrades** - a Decisions row with the reason
- **TA for every trust boundary** - cited in the code
- **One model** - the crate includes `model.rs` by path
- **Specific unknowns** - "TODO: decide NAT entry timeout and whether it is an INV"

---

## Template Files

Templates live in `docs/spec/templates/`. Copy them and replace `{{PLACEHOLDER}}` values.

| Template | Purpose |
|----------|---------|
| `index.md.template` | Spec overview and entry point (platform or subsystem) |
| `hardware.md.template` | Board, SoC, memory map, peripherals, boot, bench |
| `stack.md.template` | Toolchain, target, prover, build matrix |
| `verification.md.template` | Prover, assurance levels, TCB, ID scheme, design rules |
| `properties.md.template` | Property registry and tag registry |
| `build-order.md.template` | Implementation order with property gates |
| `subsystem.md.template` | Subsystem spec - copy once per subsystem |
| `model.rs.template` | Abstract model skeleton - copy beside each subsystem spec |

### Bootstrapping a New Subsystem

```bash
mkdir -p docs/my-subsystem/spec
cp docs/spec/templates/index.md.template      docs/my-subsystem/spec/index.md
cp docs/spec/templates/subsystem.md.template  docs/my-subsystem/spec/my-subsystem.md
cp docs/spec/templates/model.rs.template      docs/my-subsystem/spec/model.rs
cp docs/spec/templates/build-order.md.template docs/my-subsystem/spec/build-order.md  # if multi-step

# Register the tag in docs/spec/properties.md
# Add `#[path = "../../docs/my-subsystem/spec/model.rs"] pub mod my_subsystem;` to model/src/lib.rs
# Add to platform index subsystems table
```
