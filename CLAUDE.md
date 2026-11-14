# Project Bootstrap

This is a fresh project template that uses a detailed specification process you may not
be familiar with.

**Read `docs/spec-strategy.md` NOW to understand the methodology, then begin interacting
with the user.**

## Important: Discussion First

**Default mode is discussion, not implementation.** Your job is to help the user think
through and spec their project before writing code.

- Don't jump to implementation suggestions
- Don't write code until specs are solid
- Ask questions, explore the problem space, discuss tradeoffs
- Keep everything minimal: nothing the user did not decide goes into a spec. Raise it
  instead. See "Minimal Shape" in `docs/spec-strategy.md`.

## Park
"Park" sets a topic down without losing it, so attention can stay on the work at hand.
When the user says "park <something>" anywhere in a message, even mid-thought in the
middle of other work:

1. Add a row to `docs/spec/parked.md`, creating it from the template if it is missing.
   Write their words exactly as said, then what we were working on at that moment.
   Their words are a reminder to them: do not rephrase, expand, or interpret them.
2. Set Take up yourself. If unsure, `stage one`. Do not ask.
3. Confirm in one line (`Parked P7.`) and carry on with the rest of the message as if
   the park were not there. Do not discuss the parked topic, ask about it, or act on it.

You may park your own items the same way, when something is worth raising but would
interrupt the work. Say so in one line at the end of your reply.

Bring parked items back. Read the list at the start of a session. At natural pauses,
name the ones that are due. Before any exit test or gate, clear the ones due there. An
item closes only when it has been taken up and its result written where it belongs, or
when the user drops it. Record which in its Status. Never close one silently.

## Quick Start

1. Ask user: **What are you building?** (few sentences is fine)
2. Ask whether it will be formally verified. If so, read "Before You Design" in
   `docs/verification-strategy.md` before discussing architecture.
3. Discuss the big picture - what problem are they solving, who's it for, how should it work
4. Create `CLAUDE.md` from `CLAUDE.md.template` with what you know
5. Fill out specs in `docs/spec/` iteratively through conversation
6. Only move past stage one when its exit test is met

Templates are in `docs/spec/templates/` - copy and fill as needed.
