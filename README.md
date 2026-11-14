# SPARK (Verified)

**S**pec **P**owered **A**gent-**R**eady **K**it, extended for software that will be
formally verified. Built on [SPARK](https://github.com/nathanodle/spark).

## What This Is

SPARK's core idea: **discuss before writing, update as you build**. You build a
specification through conversation, then implement against it.

This kit keeps that as its first stage and adds the stages that lead to verified
software. It targets Rust verified with [Kani](https://github.com/model-checking/kani).
A project that will not be verified can use stage one alone.

```
Stage 1  Spec                 Plain SPARK, plus notes on what will matter for proof.
Stage 2  The Bridge           A completeness sweep produces the system's claims.
Stage 3  Formalize            Compiled models, invariants, contracts.
Stage 4  Implement and prove  Code and harnesses.
```

Two rules shape all of it: design **minimally**, formalize **completely**.

## Status

The standard lens set for sweeps is written but untried. Expect it to change after the
first sweeps.

## Usage

1. Clone this repo (or use as a GitHub template)
2. Open a conversation with Claude
3. Tell Claude what you're building

## What's In The Box

```
CLAUDE.md                     # Bootstrap file (gets replaced)
CLAUDE.md.template            # Template for the real project guide
docs/
├── spec-strategy.md          # Stage one
├── verification-strategy.md  # Later stages
├── examples/                 # Worked examples (none yet)
└── spec/templates/
    ├── index, stack, hardware, build-order, feature, parked    # Stage one
    ├── system-description, claims                      # The bridge
    └── model.rs                                        # Formalization
tools/check                   # Reference checker, dependency-free Python
```
