# SPARK (Verified)

**S**pec **P**owered **A**gent-**R**eady **K**it, verified edition.

A project bootstrapping template for AI-assisted development of software that will be
formally verified. Forked from [SPARK](https://github.com/nathanodle/spark).

## What This Is

SPARK's core idea: **discuss before writing, update as you build**. You build a
specification through conversation, then implement against it.

This fork adds one thing: **the spec carries everything the prover needs except the
proof itself.** Abstract models, invariants, contracts, properties, and trust
assumptions are spec artifacts. They are written and approved before implementation,
compiled from day one, and tracked to verified status after.

Target: Rust projects verified with [Kani](https://github.com/model-checking/kani).
Templates lean toward bare-metal and OS work (hardware interface models, per-core state
ownership) because that was the first use. The methodology is general.

## Usage

1. Clone this repo (or use as a GitHub template)
2. Open a conversation with Claude
3. Tell Claude what you're building

## How It Works

1. **Claude learns the methodology** from `docs/spec-strategy.md`
2. **You discuss the big picture**, including the verification approach: prover,
   assurance level, what is trusted, how state is owned
3. **Claude creates the real `CLAUDE.md`** from `CLAUDE.md.template`
4. **Platform specs get written**: hardware, stack, verification, property registry
5. **Subsystem specs get written**, each with a `model.rs` that compiles in the `model/`
   crate: state, invariants, contracts, properties, trust assumptions, harness plan
6. **Implementation only when the gate is met**: no open questions, model compiles,
   every property has a disposition, user has approved
7. **Done only when the verification gate is met**: harnesses pass at their planned
   bounds, the registry says so, every `unsafe` cites a trust assumption

## What's In The Box

```
CLAUDE.md                     # Bootstrap file (gets replaced)
CLAUDE.md.template            # Template for real project guide
docs/
├── spec-strategy.md          # The methodology
└── spec/
    └── templates/
        ├── index.md.template         # Spec overview (platform or subsystem)
        ├── hardware.md.template      # Board, SoC, memory map, boot, bench
        ├── stack.md.template         # Toolchain, target, prover, build matrix
        ├── verification.md.template  # Prover, assurance levels, TCB, ID scheme, rules
        ├── properties.md.template    # Property registry / traceability matrix
        ├── build-order.md.template   # Implementation order with property gates
        ├── subsystem.md.template     # Copy per subsystem
        └── model.rs.template         # Abstract model skeleton, copy beside each subsystem
```

## The Methodology

See `docs/spec-strategy.md`. Key principles:

- **Discuss before writing** - Specs emerge from conversation
- **Living documents** - Update specs as you build
- **The spec is the prover's input** - Models, invariants, contracts live in the spec
- **Nothing is verified by accident** - Every property has a disposition: verify, test, or trust
- **Trust is declared** - Every assumption has an ID and is cited from code
- **One owner per state** - Cross-core or cross-thread traffic is through explicit rings only
- **Build order with gates** - Steps are gated on verified properties, not on compiling code
- **Capture decisions** - What you chose, what you considered, why, and what it did to provability
