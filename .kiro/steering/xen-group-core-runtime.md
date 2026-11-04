---
inclusion: fileMatch
fileMatchPattern: ["plat/xen/*.c", "plat/xen/include/common/*.h", "Test_Specifications/xen/core/**/*.md"]
---

# Group: core runtime

## What belongs here

C files at the top of `plat/xen/` that are built for every architecture, and
the headers in `plat/xen/include/common/` that declare their interface:

- `events.c` with `include/common/events.h`: event channel binding, masking,
  dispatch, and lifecycle.
- `hypervisor.c` with `include/common/hypervisor.h`: the hypervisor callback
  and pending-event handling.
- `memory.c`: memory and mapping initialization.
- `shutdown.c`: power-management operations that request a Xen shutdown.

This is the code closest to Boeing's reference example: plain C with state,
branches, and calls out to hypercalls, so it suits Ceedling-style unit tests
with mocks.

Follow `.kiro/steering/xen-test-specifications.md` first. This file adds the
rules for this kind of code.

## Where specifications go

`Test_Specifications/xen/core/<file>.md`, for example
`Test_Specifications/xen/core/shutdown.md`. IDs `REQ-CORE-<TOPIC>-NNN`, where
`<TOPIC>` names the file or feature (`SHUTDOWN`, `EVTCHN`, `HYP`, `MEM`).

Reviewed example to copy: `Test_Specifications/xen/core/shutdown.md`.

## Read before writing

1. Confirm the file is in the unconditional source list of
   `plat/xen/Makefile.uk`, then list every `#if`, `#ifdef`, and `CONFIG_*`
   guard inside it. Code under `#if 0` is not built; give it a "not built"
   note. Code under `__X86_*__` or `__ARM_*__` guards is per-architecture
   inside a shared file; say which architecture each branch applies to.
2. For every external function the file calls (hypercall wrappers,
   `uk_*` library calls, architecture hooks), find its definition and decide
   whether the test will mock it. List each mock in the spec.
3. Find how the code is reached: registration macros such as
   `UK_BOOT_EARLYTAB_ENTRY`, `UK_EVENT`, or ops tables, and the callers in
   `plat/xen/x86/`, `plat/xen/arm/`, and `drivers/`. Cite them.
4. Record every global or static variable the functions read or write. Those
   are the state the tests must set up and check.

## What requirements look like

- **Per function, one obligation per requirement:** inputs or state before,
  one observable result after (return value, variable changed, call made with
  given arguments).
- **Branches:** one requirement per branch the code actually has, including
  error returns. If the code ignores a failure (for example a hypercall result
  that is not checked), record that as an open question; do not invent a
  recovery path.
- **Call order:** when the order of calls is the behavior, state it and test it
  as an ordered mock trace.
- **Registration:** if a function is installed through a table or macro,
  specify that it is installed, with what priority, separately from what it does.
- **Configuration:** name every guard, and write a separate requirement for
  behavior that only exists under one.

## Test methods for this group

- **Isolated unit test with mocks (Ceedling style):** the default for this
  group. Mock hypercall wrappers and library calls, set the state variables,
  call the function, and check return value, state, and mock call arguments.
  Use distinctive input values.
- **Ordered call trace:** for lifecycle and dispatch sequences.
- **Configured build plus symbol inspection:** for registration claims (the
  table entry or section exists in the image).
- **Xen guest integration test:** only for end-to-end claims; list it as
  proposed.

A mocked unit test proves the function's logic, not Xen's response to the
calls it makes; that stays an assumption.

## Review before finishing

- Every function in the file is covered or listed as a coverage gap.
- Every mock used by a proposed test is listed with the real symbol it replaces.
- Every guard is named, and `#if 0` code carries only a "not built" note.
