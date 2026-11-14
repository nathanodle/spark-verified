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
than maintaining it between sweeps. It is derived after the human has been asked, so
that what they said is in it.

### Sweeps

A sweep is the tool for any list that must be complete.

One author's passes all draw from the same distribution, so repeating them finds the
same things. A sweep counters that three ways: a **lens** narrows a pass so it goes
deep on one thing, **fresh agents** keep one piece of work from anchoring on another, and
**outside material** - the human's context, or a standards document - moves a pass
somewhere the author would not have gone alone.

**When to sweep**

The agent's sense that a list is complete is not evidence, so the trigger is never the
agent's judgment of its own list. Sweeps happen at fixed moments:

| Moment | List | Sweep |
|--------|------|-------|
| The bridge | The claims | Required |
| Formalizing a feature | Every way its state can change | Required |
| The bridge, and formalizing a driver | Trust assumptions | Ask the human |
| A reopening that adds surface | What it touches | Ask the human |

Anywhere else, a list deserves a sweep only when all three hold:

1. **A gap is silent.** No compiler, proof, or test would reveal it.
2. **A gap is costly.** Something formal rests on the list being complete.
3. **The list is open.** It cannot be derived mechanically. If a search or a tool can
   produce it, use that instead.

Never skip a sweep silently, and never launch one silently. Before running, say what
is being swept, with which lenses, and how many agents it will start, and get a yes. For a small
subject the agent may propose fewer lenses: it names each one and asks. An opposite
pair is never split.

Nothing is swept, or left unswept, by accident. The record sits where the list lives,
in the model file or in `claims.md`:

```rust
//! Swept: state changes, sweeps/2026-12-net-changes
//! Not swept: trust assumptions, because the driver touches one register.
```

A required sweep has one way out: a feature that holds no state has no state changes
to sweep, and says so in a `Not swept` line.

A sweep means separate fresh agents and a merge. Considering the lenses in one reply
is not a sweep, is not called one, and supports no claim of completeness.

**Procedure**

1. **Ask the human.** See "Asking the human" below. This comes first, before anything
   is frozen and before any result exists to steer them.
2. **Settle and freeze the input.** For the claims sweep, write any facts from the
   human's answers into the specs they concern, then derive the system description.
   For a general sweep the input is the feature's specs. Copy what the sweep reads
   into the sweep directory.
3. **Gather lenses.**
   - The standard set for this kind of sweep, below.
   - Category lenses, for the claims sweep only, including what the human said. See
     "Category lenses" below.
   - Other lenses specific to the project.
   - Documents. Walking a standard section by section is a lens.
4. **Run each lens as one pass, with one fresh agent per step.** A pass is one lens's
   whole walk. Each step of the walk goes to its own agent, with no conversation
   history. No agent is ever on its second step, so a long walk cannot thin out toward
   its end, and no step is shaped by the ones before it. Each agent works in a scratch
   directory holding only the frozen input, its lens, its one step, and the rules for
   every pass, so there is nothing else to find. It is exhaustive within its step: no
   prioritizing, no summarizing, no cap. Where the harness can restrict an agent's
   tools, denying search is optional further hardening.

   The agents' outputs for one lens are assembled, mechanically and in walk order,
   into that lens's pass file. Entry numbers are assigned then. A step that found
   nothing keeps its "Nothing, because" line in the file.

   For a general lens the walk does not exist until someone writes it. One agent per
   lens writes the walk first. Then one agent per step runs.
5. **Merge pairwise.** Take the merged list so far and one pass. Give every entry in
   the pass one verdict: same as a merged entry, refines one, or new. One pair per
   turn. Three rules:
   - Merging links, never deletes. A merged entry lists every source entry.
   - When in doubt, it is new. A leftover duplicate costs a line; a wrong merge loses a
     distinct finding.
   - Every pass entry ends up as a source of exactly one merged entry. The check script
     enforces this.
6. **Ask the human again,** showing the merged list. Anything new means running the
   new cells: a new row against every lens, or a new column against every row. Give
   the agent count and ask before running them. Take more input whenever it is
   offered.
7. **Dispose.** The human rules on every merged entry. Nothing is left undecided.

The share of the human's results that the standard passes had already found is the
best available signal of coverage. Overlap among the agent's own passes proves less.

**Lenses**

A lens has three parts: where the pass stands, the list it walks, and the question it
asks at every step. Two lenses that walk the same list with the same question are
twins. Both standard sets are untried; expect them to change after the first sweeps.

*Claims lenses.* Each names its list by kind, and the system description supplies the
list.

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
| Pairs | At one item of a list | Every other item of the same list. Run over the State list, and over the parties in Permissions. | Are these two related in a way the description does not state? Must they agree, stay apart, or happen in a set order? |

Some overlap is kept on purpose. Boundaries, Exclusions, and Permissions meet at the
edge of the system; State and Agreements meet at stored data. Watch the overlap at the
merge.

Pairs is the completeness check for Separations and Agreements, which walk only the
pairs the description declares. It walks a list against itself, so it runs only on
short lists. Like the category lenses, it can be staffed two ways:

| Way | Each agent gets | Agents, for n items |
|-----|-----------------|---------------------|
| By row (default) | One item. It walks every other item. | n |
| By cell | One pair | n(n-1)/2 |

By row, every pair is seen from both ends. A relationship reported from one end only
is either a real find or a miss by the other agent. Before the sweep, give the human
the agent count for each way and ask which they want.

*Category lenses.* For the claims sweep only, and built at sweep time. A standard
claims lens asks an open question at a part of the system: what could go wrong here? A
category lens asks a pointed one: could this part contribute to this particular harm?
The harm is fixed in advance and every part is tested against it. The two kinds of
question cross-check each other.

A **risk category** is a kind of harm, stated from the side of whoever suffers it. It
is not a single risk, and not a kind of cause. It must have an edge: it has to be
possible to tell whether a given finding belongs to it.

1. A fresh agent drafts the categories. It reads the frozen system description and the
   human's answers, verbatim, and nothing else. Categories are derived, not recalled:
   from who can be harmed by this system, and what each stands to lose. Harms the
   human named go on the list in the human's words. The agent may add categories
   beside them. It may not reword or merge them.
2. The human edits the list. It is short, so this is a good use of their attention. A
   worry of theirs can become a category in their own words.
3. The categories and the system description form a grid: every row of every list
   against every category. Each cell asks the same question: could this row contribute
   to this harm, and how? There are two ways to staff the grid.

| Way | Each agent gets | Agents |
|-----|-----------------|--------|
| By row (default) | One row and every category. It walks the categories. | One per row |
| By cell | One row and one category | One per row, per category |

By row costs one agent per row whatever the number of categories. The order of the
categories is rotated from agent to agent, so that no category is always last. By cell
gives every cell an agent to itself, and costs as many times more as there are
categories.

Before the sweep, give the human the agent count for each way and ask which they want.
Either way, the results are assembled into one pass file per category.

A remark from the human that runs as it stands (see "Asking the human") is one more
column of the grid. It is put to every row, in the human's words: with this in mind,
what could go wrong here? It gets its own pass file, which shows what the human's
input found that the standard lenses did not.

The approved categories are recorded in the sweep's `lenses.md`.

*General lenses.* For any other list. Each is a direction to count from. It does not
know what is being listed; the sweep states that once, as its target. The pass builds
its walk from the input, and asks at each step which members arise there.

| Lens | Walks | Asks at each step |
|------|-------|-------------------|
| Said | Each part of the input, in its own order | Which members does this state outright? |
| Unsaid | Each statement of the input | What does this take for granted, and is that a member? |
| Forward | Each thing that can act or happen | Which members does it produce? |
| Backward | Each thing that can be affected or end up different | Which members could have led here? |
| Time | Each moment, from before the start to after the end | Which members exist only now? |
| Outside | Each point where the subject meets its surroundings | Which members come from the other side? |
| Failure | Each thing that can fail, be absent, or come late | Which members arise only when it does? |
| Edges | Each clause of the membership test, relaxed one at a time | What meets every clause but this one, and should it count? |
| Pairs | From one part of the subject, every other part | Which members arise only between these two? |

Said and Unsaid are an opposite pair, and so are Forward and Backward. Opposite lenses
reach the same set from two ends and cross-check each other: a member found from one
end only is either a real find or a gap in the other walk. Pairs is staffed by row or
by cell, as in the claims set.

**Rules for every pass**

1. Never give a fix, and never rank.
2. "Nothing, because ..." is a full answer for a step. No step owes a finding. An
   agent whose whole job is one step will feel that it owes one. It does not.
3. If the list being walked looks incomplete, say so. That is a finding.

The claims sweep adds:

4. State each finding as "must never" or "must always".
5. Where the description already settles a step, cite it and move on. Do not restate
   the design as a finding.

A general sweep adds:

4. The sweep states its target once: what counts as a member, and at what grain.
   Every pass uses it unchanged.
5. The walk is written by its own agent before any step runs.
6. No pass sees an existing draft of the list.
7. Entries are members, never kinds of member.

**Asking the human**

The human's context is the one input the agent cannot produce. Do not ask for
"lenses". Ask for raw material in their own terms, and route it.

- **No menus.** Ask open questions in plain text. Any options offered are the agent's
  own distribution handed back.
- **No examples from the project's own field.** They anchor. Show the form of a useful
  answer with an example from an unrelated field.
- **No pressing.** "Nothing comes to mind" is a full answer. Record that the ask was
  made.
- Their words go into `lenses.md` verbatim.

*The first sweep in a project* (no sweep directory exists yet) starts with an
introduction in plain words: what will happen, why it is split up, why their input
matters, and what it will cost them. For example:

> I want to run a check called a sweep. I will start many separate helpers. Each one
> looks at one small part of the system from a single angle and lists everything that
> could go wrong there. Then I combine the lists and you rule on each item.
>
> One reviewer misses the same things every time, which is why I split it up. But the
> helpers all think like me, and you do not. What you know about your own situation is
> the one input I cannot produce. Rough, half-formed answers are the most useful kind.
>
> This will start a few hundred helpers, each with one small job, and leave you a list
> to rule on, which may be long. I am asking you first, before showing you anything,
> so my results do not steer you.

Give the real agent count in place of the rough one.

*The questions,* asked before any results are shown:

1. What is unusual about where and how this will be used, and who or what will be
   around it?
2. Which outcome would bother you most?
3. Which part do you trust least, including anything you think I got wrong?

Then show the form: "For a bridge, a useful answer might be: the river floods every
spring." Offer back anything they have parked.

*After the merge,* show the merged list and ask: What did you expect to see that is
not here? Does anything on it remind you of something else?

*Later sweeps* skip the introduction: one line of reminder, then the questions.

*General sweeps* ask one lighter question: Is there anything about this you suspect
gets forgotten, or anything odd about how it is used?

*What happens to their answers.* Route, never rewrite. An angle is worth having
because of how it is phrased, so it travels verbatim. A fact is worth the same in any
voice.

| What they said | Where it goes | Wording |
|----------------|---------------|---------|
| It names a harm | The category list | Theirs |
| It is a fact about the system or its surroundings | The spec it concerns. The system description picks it up from there. | The spec's own, with the specifics kept |
| Anything else | It runs as it stands: one more column in the category grid. In a general sweep, one more lens, put to every part of the input. | Theirs |

- When unsure, it runs as it stands.
- One remark can take more than one route.
- Show the human the routing before the sweep runs, and let them correct it.

**Layout**

```
docs/sweeps/<date>-<name>/
├── input.md              # Frozen copy of what the sweep read
├── lenses.md             # Target or risk categories, lenses used, the human's own words
├── passes/
│   └── NN-<lens>.md      # One per lens, assembled from its agents. Entries: "### E12: title"
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
paths, updates arriving from another owner. One unlisted change breaks the induction,
and nothing would reveal the gap. So this list is swept for every feature, with the
general lenses and the target "every way this feature's state can change". Each merged
entry becomes a contract or is dismissed. A contract names the entries it came from:

```rust
/// Contract: net::flow_insert
/// From: sweeps/2026-12-net-changes/merged#M3
pub fn pre_flow_insert<const N: usize>(s: &State<N>, a: &InsertArgs) -> bool { ... }
```

### Accounting for Notes

Once stage one is left, every standing verification note and every filled-in
Verification Impact cell must be accounted for. Any one of these settles it:

- **A formal item or contract names it** in its `From`.
- **Every claim that names it was rejected or deferred.** The ruling on the claim is
  the ruling on the note, and needs no second line. If such a claim is accepted later,
  the note is unsettled again.
- **In the platform specs, a claim or trust assumption in `claims.md` names it,**
  whatever the claim's status. An accepted claim carries the note forward. This
  happens when `claims.md` is written.

Anything else is dismissed, in the formal home of its directory: the model file, or
`claims.md` for the platform specs.

```rust
//! Dismissed: net/spec/flow#V5, because it duplicates net/spec/flow#V2.
```

The reason must name what made the note unnecessary: the note it duplicates, or a
decision that replaced it. It may not cite an accepted claim. A dismissal is the one
place an early thought can be dropped on the formalizer's own judgment, so
`tools/check dismissed` lists them all and the human reviews them at the gate.

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
  3. Decide. If the change adds surface, such as a new boundary, party, or piece of
     state, ask the human whether to sweep again.
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
4. The human has ruled on sweeping the project's trust assumptions
5. The feature's specs are ✅ Complete
6. No parked item due at the claims sweep or at this feature's formalization is still
   parked

**Start implementing a feature.**
1. Its model compiles
2. Its state changes are swept, and every merged entry is a contract or dismissed
3. For a driver, the human has ruled on sweeping its trust assumptions
4. Every note and impact cell in its specs is accounted for
5. Every formal item has a `From` and a disposition
6. `tools/check` reports no errors and nothing open for this feature except missing
   harnesses
7. The human has approved the model, including its dismissals

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
points at nothing; a claim, formal item, or contract with no `From`; a `Swept` line that
points at no sweep; a `Not swept` line with no reason; a missing or malformed status
or disposition; a closed parked item with no outcome; a source that cites a spike record that does not
exist; a trust assumption with no
mitigation; a dismissal with no reason,
with a reason that names nothing, or that cites an accepted claim; a sweep entry
merged into more than one merged entry.

**Open** (must be empty at the relevant gate): parked items; a model with no
state-change sweep; no ruling on sweeping the project's trust assumptions; sweep entries not yet merged; merged
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
