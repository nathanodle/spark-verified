# Project Bootstrap

This is a fresh project template that uses a detailed specification process you may not
be familiar with. The software built here will be **formally verified**, and the spec
carries everything the prover needs except the proof itself.

**Read `docs/spec-strategy.md` NOW to understand the methodology, then begin interacting
with the user.**

## Important: Discussion First

**Default mode is discussion, not implementation.** Your job is to help the user think
through and spec their project before writing code.

- Don't jump to implementation suggestions
- Don't write code until specs are solid and models compile
- Ask questions, explore the problem space, discuss tradeoffs
- See "Gates" in `docs/spec-strategy.md` for readiness criteria

## Quick Start

1. Ask user: **What are you building?** (few sentences is fine)
2. Discuss the big picture - what problem, what hardware, what "verified" means here
3. Discuss the verification approach early: prover, assurance level, what is trusted,
   how state is owned across cores or threads
4. Create `CLAUDE.md` from `CLAUDE.md.template` with what you know
5. Create the `model/` crate and fill out platform specs in `docs/spec/` through
   conversation: hardware, stack, verification, properties
6. Identify subsystems, register their tags, fill out their specs and `model.rs` files
7. Only suggest implementation when a subsystem meets the implementation gate

Templates are in `docs/spec/templates/` - copy and fill as needed.
