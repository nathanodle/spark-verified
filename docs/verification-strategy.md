# Verification Strategy

The stages that follow [`spec-strategy.md`](spec-strategy.md) for a project that will
be formally verified. Written for Rust verified with Kani.

```
Stage 1  Spec                 spec-strategy.md. Stay here as long as possible.
Stage 2  The Bridge           System description, sweep, claims.
Stage 3  Formalize            Compiled models, one feature at a time.
Stage 4  Implement and prove  Code and harnesses.
         Reentry              Rules for reopening discussion once formal things exist.
```

Parts marked **PENDING** are not designed yet.

---

## The Idea

One chain runs from what the user wants, to what the system claims, to how it is
designed, to formal statements, to code, to evidence. Each link is a translation and
can lose something. A prover secures only the last link: the code satisfies the formal
statements. This process exists to manage the links above it, and to keep the chain
connected when something changes.

Two rules pull in opposite directions on purpose:

- **Minimal** governs what exists. Every feature, option, and field is surface that
  must be specified, modeled, and proved. Keep the design as small as intent allows.
- **Complete** governs what is said about what exists. The formal coverage of that
  small design should have no gaps. Never trim the formal layer for brevity, and never
  pad the design for robustness.

---

## Before You Design

Read this before stage one design work. These are architecture decisions; changing
them late is expensive.

**Know the prover.** Kani is a bounded model checker. A passing harness shows that for
every input and state within stated bounds, no assertion fails, nothing panics, and no
undefined behavior is reachable. It says nothing beyond the bound. It cannot show that
something eventually happens, and it does not model weak memory ordering between cores.

**Provability constraints.** Each is a default. Record it as adopted or rejected in the
stack spec's decisions.

1. **Logic lives in pure code.** No `unsafe`, no heap, no hardware dependence. It
   builds on the development host, where the prover runs.
2. **Hardware sits behind traits,** so a harness can substitute a model device.
3. **Encoding is explicit.** Wire and register layouts are built with byte and bit
   operations, never by overlaying a struct or depending on native endianness. The host
   and the target may differ.
4. **Capacity is a parameter.** Fixed-size structures take their size as a const
   generic, so a harness can verify a small instance of the same code.
5. **Loops have stated bounds.**
6. **Every piece of state has one owner** (a core or a thread). Owners exchange data
   only through single-producer single-consumer queues. An interrupt handler shares
   nothing with the code it interrupts.
7. **Every `unsafe` block cites the trust assumption it relies on.**

---

## Stage 2: The Bridge

Stage one ends with settled specs and a pile of verification notes. The bridge turns
them into the claims the project will stand behind.

### System Description

One document, `docs/spec/system-description.md`, the single input every pass of the
claims sweep reads. It says what the system is, then gives the ten lists the standard
lenses walk. It is derived from the stage-one specs: rewrite it before a sweep rather
than maintaining it between sweeps.

### Sweeps

A sweep is the tool for any list that must be complete. It is required once, to build
the claims. Use it anywhere else a list must have no gaps, such as every way a
feature's state can change, or every assumption a driver makes.

One author's passes all draw from the same distribution, so repeating them finds the
same things. A sweep counters that three ways: a **lens** narrows a pass so it goes
deep on one thing, **fresh agents** keep passes from anchoring on each other, and
**outside material** - the human's context, or a standards document - moves a pass
somewhere the author would not have gone alone.

**Procedure**

1. **Freeze the input.** Copy what the sweep reads into the sweep directory.
2. **Gather lenses.**
   - The standard set, below.
   - Lenses specific to the project.
   - Lenses from the human. Ask before showing any results, ask again after the merge,
     and take more whenever they are offered. Raw input is welcome: a worry, an
     anecdote, a device they own. Each becomes its own lens. Offer back what the human
     has parked; those items are this kind of material.
   - Documents. Walking a standard section by section is a lens.
3. **Run one pass per lens, each in a fresh agent** with no conversation history. The
   pass runs in a scratch directory holding only the frozen input and its one lens, so
   there is nothing else to find. Its output is a file of numbered entries. It is
   exhaustive: no prioritizing, no summarizing, no cap. Where the harness can restrict
   an agent's tools, denying search is optional further hardening. Each pass is also
   given the rules for every pass, below.
4. **Merge pairwise.** Take the merged list so far and one pass. Give every entry in
   the pass one verdict: same as a merged entry, refines one, or new. One pair per
   turn. Three rules:
   - Merging links, never deletes. A merged entry lists every source entry.
   - When in doubt, it is new. A leftover duplicate costs a line; a wrong merge loses a
     distinct finding.
   - Every pass entry ends up as a source of exactly one merged entry. The check script
     enforces this.
5. **Dispose.** The human rules on every merged entry. Nothing is left undecided.

The share of the human's lens results that the standard passes had already found is the
best available signal of coverage. Overlap among the agent's own passes proves less.

**Lenses**

A lens has three parts: where the pass stands, the list it walks, and the question it
asks at every step. A standard lens names its list by kind, and the system description
supplies the list. Two lenses that walk the same list are twins; each lens below walks
a different one.

This is the standard set for the claims sweep. It is untried; expect it to change
after the first sweeps. Other sweeps choose their own lenses.

| Lens | Stands | Walks | Asks at each step |
|------|--------|-------|-------------------|
| Boundaries | Outside, on the far side | Each point where anything enters or leaves | Of everything that could cross here, either way, what must never be accepted or emitted? |
| Journeys | Riding one unit of work | Each step of each path through the system | What if this step is skipped, repeated, reordered, cut off halfway, or handed something unexpected? |
| State | Beside one stored thing | Each piece of state | What must always be true of it? What follows if it is stale, duplicated, missing, or wrong? |
| Limits | At one finite thing | Each size, count, and range | What happens at empty, full, one past, and wraparound? Who can drive it there, and who pays? |
| Moments | At one point in the system's life | Each distinct moment: start, first setup, change, overload, failure, restart | What is possible now that is impossible at any other time? |
| Separations | Between two things meant to stay apart | Each such pair, in both directions | Every way one could affect the other or learn about it |
| Agreements | Between two things meant to say the same | Each pair of copies | When can they differ, for how long, and what happens meanwhile? |
| Reliances | At one thing depended on but not controlled | Each outside dependency | What if it is false, late, or different from expected? |
| Exclusions | At one thing the system says it does not do | Each exclusion | The world does it anyway. Where does it arrive, and what happens? |
| Permissions | As one party, using only what it is allowed | Each party and each allowed action | What harm can be assembled from allowed actions alone? |

Some overlap is kept on purpose. Boundaries, Exclusions, and Permissions meet at the
edge of the system; State and Agreements meet at stored data. Watch the overlap at the
merge.

A walk too long for one pass is split across several passes. **PENDING: at what
length.**

**Rules for every pass**

1. State each finding as "must never" or "must always". Never as a fix, and never
   ranked.
2. "Nothing, because ..." is a full answer for a step. No step owes a finding.
3. Where the description already settles a step, cite it and move on. Do not restate
   the design as a finding.
4. If the list being walked looks incomplete, say so. That is a finding.

**Layout**

```
docs/sweeps/<date>-<name>/
├── input.md              # Frozen copy of what the sweep read
├── lenses.md             # Lenses used, including the human's own words
├── passes/
│   └── NN-<lens>.md      # One per pass. Entries are headings: "### E12: title"
└── merged.md             # Table: | # | Entry | Sources |   rows M1, M2, ...
```

Raw passes are an archive. Normal work reads `merged.md` and what came of it, never
`passes/`.

### Claims

`docs/spec/claims.md` is the anchor for everything formal. A claim is something the
system says it does or never does.

Candidates come from two places: the "Must never" notes across the stage-one specs, and
the merged entries of the claims sweep. The human disposes of each.

```markdown
| ID | Status | Claim | From |
|----|--------|-------|------|
| C-001 | Accepted | No packet is forwarded between two zones. | net/spec/policy#V1, sweeps/2026-11-claims/merged#M4 |
| C-002 | Rejected, because no service listens there | The WAN address rate-limits logins. | sweeps/2026-11-claims/merged#M9 |
```

- **Status** is `Accepted`, `Rejected, because ...`, or `Deferred, because ...`. A
  rejection is a decision. A silent omission is a hole.
- **Prefer removing to adding.** When a hazard is accepted, first consider the response
  that shrinks the surface: drop, refuse, do not support.
- **Accepted findings flow back while it is still cheap.** If one changes the design,
  it becomes a decision in the affected spec. Each spec it bears on gets a "Must never"
  note, and the claim names those notes. This is what later forces every affected
  feature to deal with the claim.

The same document holds trust assumptions that are not about a model (the toolchain,
the boot loader):

```markdown
| ID | Assumption | Mitigation | From |
|----|------------|------------|------|
| TA-PLAT-001 | The compiler emits code faithful to the source. | Pinned version | spec/stack#D1 |
```

---

## Stage 3: Formalize

One feature at a time. Start with the one most likely to expose a mistake upstream,
while little else rests on those decisions.

### The Model

Each subsystem directory gains a `model.rs` beside its specs: the abstract state and
the rules over it, independent of how it will be implemented. A table is a map, not a
trie. It is `no_std`, has no `unsafe`, and uses const-generic capacities.

One crate compiles every model by path, so the spec is type-checked before any
implementation exists and there is exactly one copy:

```rust
// model/src/lib.rs
#![no_std]
#![forbid(unsafe_code)]

#[path = "../../docs/spec/model.rs"]
pub mod platform;

#[path = "../../docs/net/spec/model.rs"]
pub mod net;
```

### Formal Items

Three kinds, each with a global ID that is never reused:

| Kind | ID | What it is |
|------|----|------------|
| Invariant | `INV-<TAG>-nnn` | Always true of the state between operations |
| Property | `PROP-<TAG>-nnn` | A system-level claim resting on invariants and contracts |
| Trust assumption | `TA-<TAG>-nnn` | Relied on without proof |

`<TAG>` is the subsystem directory name in uppercase without hyphens, or `PLAT`.

**A formal item's home is the Rust file,** beside the code that states it. Markdown
refers to it by ID and never restates it. A trust assumption about a model lives with
that model; any other lives in `claims.md`.

```rust
/// INV-NET-002: no two live entries with translation share an external port.
/// From: C-004, net/spec/flow#V3
/// Disposition: Verify, N=4
pub fn inv_unique_ext_port<const N: usize>(s: &State<N>) -> bool { ... }
```

- **From** names what the item exists for: the claim it serves, the item it helps
  prove, and any notes it accounts for.
- **Disposition** is one of:
  - `Verify, N=<bound>` - a harness will check it at that instance size
  - `Test, because <reason>`
  - `Trust, mitigated by <how>`

Nothing is verified by accident and nothing is downgraded silently: a disposition other
than Verify carries its reason on the same line.

**Verify at a bound below the real capacity** needs a one-line argument in the doc
comment that the code does not depend on the size: the structure is a const-generic
instance of the same code, or the operation touches at most a fixed number of entries.
Without that argument the disposition is Test.

### Operations and Contracts

An operation is named by its Rust path (`net::flow_insert`); the compiler keeps names
unique. Each gets a requires and an ensures in the model, either as `pre_`/`post_`
predicates or as a pure reference function the implementation must agree with.

Every way the state can change needs a contract: initialization, timeouts, error
paths, updates arriving from another owner. One unlisted change breaks the induction.
If that list must be complete and is not obviously so, sweep it.

### Accounting for Notes

Once stage one is left, every standing verification note and every filled-in
Verification Impact cell must be accounted for.

- In a subsystem, a formal item names it in its `From`.
- In the platform specs, a claim or trust assumption in `claims.md` names it. This
  happens when `claims.md` is written.

Anything else is dismissed, in the formal home of its directory: the model file, or
`claims.md` for the platform specs.

```rust
//! Dismissed: net/spec/flow#V5, because C-002 was rejected.
```

The reason must name what made the note unnecessary: a rejected or deferred claim, a
decision that replaced it, the note it duplicates. It may not cite an accepted claim,
so accepting a claim later brings its dismissed notes back. Dismissals are the one
place an early thought can be dropped, so `tools/check dismissed` lists them all and
the human reviews them at the gate.

---

## Stage 4: Implement and Prove

The implementation provides `fn abs(&Impl) -> Model`, mapping its concrete state to the
model. A harness checks one item, and is named for it: `check_inv_net_002`.

```rust
let mut s: Impl = kani::any();
kani::assume(inv(&abs(&s)));            // start from any valid state
let args = kani::any();
kani::assume(pre_op(&abs(&s), &args));
let old = abs(&s);
let r = s.op(args);
assert!(inv(&abs(&s)));                 // invariant preserved
assert!(post_op(&old, &abs(&s), &args, &r));
```

- **Guard against vacuity.** An assumption no state satisfies makes every proof pass.
  Use `kani::cover!` to show the interesting paths are reachable.
- **Every assumption in a harness is an invariant, a requires, or a cited trust
  assumption.** Nothing else may be assumed.
- **Write discoveries back before continuing.** Proving finds things the spec missed:
  a helper invariant, a stronger requires, a new assumption. Each gets an ID, a `From`,
  and a disposition in the model file. This is expected, and it is additive.
- **Status is computed, never written.** Whether an item is verified is the result of
  running the harnesses. No document records it by hand.

---

## Reentry

Reopening discussion after formal things exist is normal. Its cost is whatever rests on
the thing reopened.

- **Additive** reentry adds without moving anything upstream: a helper invariant, a new
  note. It is routine.
- **Revising** reentry changes something upstream: a decision, a claim, a contract.
  Keep it rare, and handle it as an event:
  1. Mark the item `REOPENED` at its home.
  2. Run `tools/check rests-on <item>` and read what depends on it before discussing.
  3. Decide.
  4. Update everything downstream in the same sitting.
  5. Remove the mark. `tools/check` must come back clean before the reentry is closed.

Nothing that rests on a reopened item may pass a gate.

---

## References

Three kinds, with different jobs.

| Kind | Answers | Form |
|------|---------|------|
| Navigation | What should I read before working on this? | Related Specs tables, the index. Unchanged from stage one. |
| Source | How do we know this fact? | `[source: ...]` or `[believed]`, in the sentence. |
| Trace | Where did this come from, and what is it for? | `From`, as below. |

**Trace references follow four rules.**

1. **Every item has one home and a stable address.**

   | Item | Address | Example |
   |------|---------|---------|
   | Note, decision | document path under `docs/`, no extension, `#`, local number | `net/spec/flow#V3`, `spec/stack#D2` |
   | Sweep entry, merged entry | the same form | `sweeps/2026-11-claims/passes/03-wan#E12`, `sweeps/2026-11-claims/merged#M4` |
   | Claim | `C-nnn` | `C-004` |
   | Formal item | its ID | `INV-NET-002` |

2. **Every reference points upstream.** The thing made later names the thing it came
   from. Earlier documents are never edited to point forward.
3. **There is one relation: From.**
4. **Every downstream view is computed, never written.** What rests on an item, which
   notes are unaccounted for, which claims nothing serves, the registry of all items.

**Hand-kept references carry only what tools cannot see.** Relations among formal items
already live in code: a harness name embeds the ID it checks, a harness that assumes an
invariant shows the dependency, an `unsafe` comment cites its trust assumption. Do not
write those again in a table.

---

## Gates

**Leave stage one** (project). The exit test in `spec-strategy.md`.

**Start formalizing a feature.**
1. The claims sweep is merged and every merged entry is disposed of
2. Every note and impact cell in the platform specs is accounted for
3. The human has approved `claims.md`, including its dismissals
4. The feature's specs are ✅ Complete
5. No parked item due at the claims sweep or at this feature's formalization is still
   parked

**Start implementing a feature.**
1. Its model compiles
2. Every note and impact cell in its specs is accounted for
3. Every formal item has a `From` and a disposition
4. `tools/check` reports no errors and nothing open for this feature except missing
   harnesses
5. The human has approved the model, including its dismissals

**A feature is done.**
1. Every Verify item has a passing harness at its bound
2. Every Test item has passing tests
3. Discoveries are written back
4. `tools/check` reports no errors and nothing open for this feature

**The project is done.** Every accepted claim is served by at least one formal item,
and the human has reviewed whether those items are enough for each claim.

---

## The Check Script

`tools/check` is one dependency-free Python script. It reads the spec directories under
`docs/`, the sweeps, and the Rust sources.

```
tools/check                    # errors and open items
tools/check registry           # every claim and formal item, generated
tools/check rests-on <item>    # everything that depends on an item
tools/check dismissed          # every dismissal, for review
```

**Errors** (exit status 1): an ID or address with two homes; a `From` or dismissal that
points at nothing; a claim or formal item with no `From`; a missing or malformed status
or disposition; a closed parked item with no outcome; a trust assumption with no
mitigation; a dismissal with no reason,
with a reason that names nothing, or that cites an accepted claim; a sweep entry
merged into more than one merged entry.

**Open** (must be empty at the relevant gate): parked items; sweep entries not yet merged; merged
entries not yet disposed of; notes of kind Open; notes and impact cells not accounted
for, in any directory that has a model or `claims.md`; accepted claims nothing serves;
`[believed]` facts; anything resting on a `REOPENED` item; Verify items with no
harness.

---

## Templates

| Template | Purpose |
|----------|---------|
| `system-description.md.template` | The input to the claims sweep |
| `claims.md.template` | Claims and platform trust assumptions |
| `model.rs.template` | A subsystem's model - copy beside its specs |
