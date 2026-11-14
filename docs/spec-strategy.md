# Spec Strategy

A reusable approach to project specification. Applicable to any project.

This is stage one of SPARK (Verified). A project that will not be formally verified
uses this document alone. A project that will be verified starts here too, stays here
as long as it can, and then continues with [`verification-strategy.md`](verification-strategy.md).
Such a project should read that document's "Before You Design" section first, because
some choices that make software provable are architecture decisions.

---

## Philosophy

### Discuss Before Writing
Specs emerge from conversation, not isolation. Start broad, narrow gradually through
discussion. The spec is a record of decisions made, not a document written in a vacuum.

### Specs Are Living Documents
Specs change as understanding deepens and implementation reveals reality. Update them
continuously. A stale spec is worse than no spec - it actively misleads.

### Separate Concerns, Link Liberally
Split specs by concern (stack, hardware, features) but cross-reference heavily. When
working on a feature, you should know exactly which shared specs are relevant.

### Capture Decisions, Not Just Outcomes
Don't just write down *what* was decided - write down *why*: what you chose, what you
considered, and the reasoning. Conversations get lost. Knowing what was rejected
prevents revisiting the same debates. Don't let important decisions live only in
conversation.

### Minimal Shape
Define every system, feature, and statement in the simplest way that matches the
decisions made and the user's intent. Before committing anything - a spec, a feature, a
sentence, code - work out its minimum usable shape, and write that.

Nothing inferred goes in silently. Functionality, options, configurability, future
hooks, and implementation detail the user did not decide are raised, not added.
Everything in a spec should be able to answer "what is this from?" with a decision or
something the user said.

Every spec handed to the user comes with a short **raised, not added** list: the things
the author was tempted to include. The user accepts, parks, or ignores each. Only
accepted items enter the spec.

### Raise What Was Not Asked
The user holds the intent. The agent often holds domain knowledge the user lacks. The
agent has a duty to raise facts, hazards, and consequences the user did not ask about.
Raising puts a question in front of the user. It never puts anything in the spec. The
user decides, and a "no" is recorded like any other decision.

---

## Document Structure

### Three Layers

```
┌─────────────────────────────────────────────┐
│  Planning                                   │
│  - Build order, what to build first and why │
└─────────────────────────────────────────────┘
                      ↓
┌─────────────────────────────────────────────┐
│  Shared Specs                               │
│  - Stack (technologies, toolchain)          │
│  - Hardware or platform (what it runs on)   │
│  - Other shared architecture, as needed     │
└─────────────────────────────────────────────┘
                      ↓
┌─────────────────────────────────────────────┐
│  Feature Specs                              │
│  - One per major feature                    │
│  - References shared specs                  │
│  - Contains feature-specific details        │
└─────────────────────────────────────────────┘
```

### Recommended Files

```
docs/
├── spec-strategy.md              # This file - methodology
├── verification-strategy.md      # Later stages, verified projects only
├── spec/                         # Platform-level shared specs
│   ├── index.md                  # Entry point - links to shared specs + subsystems
│   ├── stack.md                  # Tech choices, toolchain
│   ├── hardware.md               # What it runs on (if applicable)
│   ├── build-order.md            # Implementation order
│   ├── parked.md                 # Topics set down to come back to
│   ├── templates/                # Templates for bootstrapping
│   └── archive/                  # Historical specs (completed build orders, etc.)
│
├── spikes/                       # Spike records
│
├── <subsystem>/                  # One per major subsystem
│   └── spec/
│       ├── index.md              # Subsystem overview + Related Platform Specs table
│       └── [feature].md          # Feature specs for this subsystem
│
└── examples/                     # Worked examples
```

### Subsystems

Specs split into **platform** and **subsystem** layers:

- **Platform specs** (`docs/spec/`) cover what every subsystem shares.
- **Subsystem specs** (`docs/<subsystem>/spec/`) cover features specific to one module.

Every subsystem index starts with a **Related Platform Specs** table pointing back to
the shared specs it depends on.

**When to create a new subsystem directory:**
- It has its own features, state, or domain logic
- It will have multiple spec documents
- It could conceptually be developed somewhat independently

**When NOT to:**
- It's just one feature of an existing subsystem (add a feature spec there instead)
- It's platform infrastructure (belongs in `docs/spec/`)

### Index Structure

The index is the entry point. It shows what the project is (one paragraph), all spec
documents with status, and high-level scope.

---

## Cross-Referencing

Every feature spec starts with a "Related Specs" table:

```markdown
## Related Specs

| Spec | Relevance |
|------|-----------|
| [Stack](../../spec/stack.md) | Technologies used by this feature |
| [Hardware](../../spec/hardware.md) | Registers and memory this feature touches |
```

This tells the reader (human or AI) what to load for context before diving in. These
are navigation references. Verified projects add a second kind later; see References in
`verification-strategy.md`.

---

## Status Tracking

| Status | Meaning |
|--------|---------|
| ✅ Complete | Spec written and approved |
| 🔲 Pending | Placeholder exists, content not written |
| 🔲 Partial | Some sections done, others TODO |
| 🔲 Draft | Written but not reviewed/approved |

Mark sections within docs too.

---

## Decisions Tables

Any spec that involves meaningful choices includes a Decisions table. Rows are numbered
within the document and never renumbered, so they can be pointed at.

```markdown
## Decisions

| # | Decision | Alternatives Considered | Why This Choice |
|---|----------|------------------------|-----------------|
| D1 | Fixed-size table | Heap-backed map | No allocator in the kernel |
```

---

## Facts

A spec states facts about the world: what a board has, what a tool does. Mark where
each one that matters came from, in the sentence that states it:

- `[source: reference manual rev 2, section 5.3]` - read from evidence
- `[believed]` - from memory, the user's or the agent's, not yet checked

A spec built on a believed fact is built on a guess. A **spike** is how a believed fact
becomes a sourced one: a small experiment run to answer one question. Its code is
thrown away. Its record is kept.

Ask the user before running a spike, and state its question first. A spike is not
implementation and is not gated.

The record is one file, `docs/spikes/<date>-<name>.md`, with four parts:

| Part | Holds |
|------|-------|
| Question | One sentence, written before the spike starts. It names the believed facts it will settle. |
| Method | What was done, and with what versions and hardware. Enough to repeat it. |
| Observed | The raw result, verbatim: output, measurements, the configuration that worked. |
| Answer | The fact, in a sentence or two. |

- **Keep Observed and Answer apart.** A conclusion is never written in the voice of an
  observation.
- **Everything the answer rests on is inside the record.** Do not point at a log kept
  somewhere else.
- **A fact settled by a spike cites the record:** `[source: spikes/<date>-<name>]`.
- **Nothing from a spike's code enters the implementation.** Anything worth keeping,
  such as a configuration that worked, is kept as text in the record.

---

## Parking

Things come up while attention is elsewhere, from a passing thought to a topic that
deserves a full spec discussion. **Parking** sets one down without losing it. Either
the user or the agent can park.

`docs/spec/parked.md` is the one list for the whole project. Everything else in a spec
has a home; a parked item does not have one yet. It leaves the list when it is taken up
and its result lands wherever it belongs.

```markdown
| # | By | Item | Take up | Status |
|---|----|------|---------|--------|
| P1 | User | "what about the second trunk" - while laying out zones | stage one | Parked |
| P2 | Claude | Flow capacity is shared across zones - while sizing the table | stage one | Closed: became net/spec/flow#D4 |
```

- **Capture costs one line.** The words go down as said, with what was being worked on
  at that moment, because the surrounding context is what gets lost. No discussion at
  capture.
- **Take up** says when the item must come back: `stage one` (before leaving it),
  `claims sweep`, `formalizing <feature>`, or `any time`. An item of unknown impact is
  `stage one`. Parking must not carry a design change past the point where changes are
  cheap.
- **Items come back** at the start of a session, at natural pauses, and before any
  exit test or gate.
- **An item closes** when it is taken up and its result is written where it belongs, or
  when the user drops it. The Status says which. Nothing closes silently.

---

## Build Order

Every project has a build order document that answers:
1. What do we build first?
2. Why that order? (dependencies)
3. What spec sections need to be done for each step?

---

## What Each Spec Covers

**Stack:** summary table of choices, toolchain details, how things are built and run,
decisions.

**Hardware (if applicable):** the board and processor, memory map, peripherals in and
out of scope, boot path, bench setup, decisions.

**Feature:** related specs, overview, requirements, behavior (the state it holds and
the operations it offers, in prose), error handling, decisions, open questions.

Add a section only when a decision calls for one.

---

## AI/Agent Instructions

Create a `CLAUDE.md` or `AGENTS.md` in project root: project description, spec
organization, key files, constraints, the working rules from the Philosophy section
above, and a reminder to keep docs in sync. See `CLAUDE.md.template`.

---

## Workflow

### Starting a New Project

1. Create `docs/spec/` directory
2. Create `index.md` with project overview
3. Discuss stack choices → write `stack.md`
4. Identify features → create placeholder `[feature].md` files
5. Discuss build order → write `build-order.md`
6. Work through the specs, filling them in through discussion

### New Session / New Contributor

1. Read `CLAUDE.md` or `AGENTS.md` (if AI) or `README.md` (if human)
2. Read `docs/spec/index.md` for overview
3. Check `build-order.md` for current focus
4. Read relevant feature spec + its related specs
5. Continue work

---

## Spec Before Implementation

The default mode is **discussion**, not implementation. Spec first, code second.

A spec section is ready when:

1. **No unresolved TODOs** - no "TODO", "TBD", or placeholders
2. **No open questions** - discussed and answered
3. **User has approved** - reviewed and agreed, not just written
4. **Status is ✅ Complete** - explicitly marked
5. **Dependencies ready** - what it depends on is implemented, or defined well enough
   that you know what it takes in and gives back
6. **Big picture understood** - how it fits the whole, no conflicts with other specs

In a plain project, a ready spec may be implemented. In a verified project, a ready
spec may move on to the later stages; implementation waits for the gates in
`verification-strategy.md`.

Don't over-spec. Performance details and implementation internals can emerge during
implementation. Spec the *what*, not the *how*.

---

## If the Project Will Be Verified

Stage one stays ordinary spec work. Nothing formal is written and nothing gets a global
ID. Four light habits make the later stages possible.

### 1. Provability constraints are architecture decisions

Read "Before You Design" in `verification-strategy.md` before designing. Record each
constraint the project adopts or rejects as a numbered decision in the stack spec.

### 2. Decisions carry their verification impact

Decisions tables gain one column:

```markdown
| # | Decision | Alternatives Considered | Why This Choice | Verification Impact |
|---|----------|------------------------|-----------------|---------------------|
| D2 | One table for all tracked flows | Separate NAT and conntrack tables | Less state | One set of invariants covers both |
```

### 3. Verification notes are written when the thought occurs

Each spec has a Verification Notes section. A note is a sentence or two in plain words,
numbered within the document and never renumbered.

```markdown
## Verification Notes

| # | Kind | Note |
|---|------|------|
| V1 | Must never | A packet crosses from one zone to another. |
| V2 | Relies on | The boot loader hands off with the MMU in a known state. |
```

| Kind | Meaning |
|------|---------|
| Must never | Something the system must never do or allow |
| Relies on | Something assumed about the outside world |
| Must agree | Two things that have to stay consistent |
| Shaped for proof | A design choice made to keep something provable |
| Hard to prove | Something the prover is unlikely to reach |
| Open | A verification question to settle before formalizing |

Keep it DRY. A thought that is the reason for a decision goes in that decision's
Verification Impact cell. Everything else goes in the notes. Neither restates the
other; point instead (`see D2`, `see V1`).

In stage one a note can be rewritten or removed freely, since nothing rests on it yet.
A removed number is not reused. A note of kind Open must be resolved before leaving
stage one: remove it, or turn it into another kind.

Later stages must account for every note and every filled-in impact cell, so nothing
said early is lost.

### 4. Discovery is pulled forward

A change of mind costs whatever rests on the thing changed. In stage one nothing formal
rests on anything, so changes are cheap. Later they are not. So stay in stage one as
long as it keeps producing surprises, and use the tools that provoke them while they
are cheap:

- **Spikes** for believed facts that other things depend on.
- **Sketch models** for the risky parts: a throwaway attempt at writing the state and
  its rules precisely, to surface questions. A sketch is not kept and creates nothing
  to maintain.

### Leaving stage one

Leave when the discovery tools have stopped producing surprises:

1. Every spec meets the readiness criteria above
2. Facts that other things depend on are sourced, not believed
3. The risky parts have been sketched
4. No questions are open, and no note of kind Open remains
5. No parked item with take up `stage one` is still parked

Then continue with The Bridge in `verification-strategy.md`.

---

## Project Structure

Project structure (folder layout, file organization) is one of the **last things to
define** before implementation. Structure follows from decisions, not the other way
around.

**Define structure after:** the big picture is clear, the stack is decided, major
features are identified, and you understand how the pieces relate.

Organize code by *what it does*. Each layer should be able to change without rippling
through the others, and dependencies should point one way. Match the conventions of
your language and framework unless you have a strong reason not to.

---

## Anti-Patterns

- **Monolithic spec** - one giant document is hard to navigate and update
- **Orphan specs** - feature specs with no links to shared specs
- **Write-once specs** - written at the start and never updated
- **Icing** - features, options, and detail nobody decided on
- **Guesses stated as facts** - a believed fact with no marker
- **Vague TODOs** - "TODO: figure this out"; say what is unknown

---

## Template Files

Templates live in `docs/spec/templates/`. Copy them and replace `{{PLACEHOLDER}}` values.

| Template | Purpose |
|----------|---------|
| `index.md.template` | Spec overview and entry point (platform or subsystem) |
| `stack.md.template` | Tech stack |
| `hardware.md.template` | What it runs on |
| `build-order.md.template` | Implementation order |
| `parked.md.template` | The parked list |
| `feature.md.template` | Feature spec - copy once per feature |

Templates for the later stages are listed in `verification-strategy.md`.
