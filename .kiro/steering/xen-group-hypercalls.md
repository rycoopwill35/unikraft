---
inclusion: fileMatch
fileMatchPattern: ["plat/xen/include/xen-x86/hypercall*.h", "plat/xen/include/xen-arm/hypercall.h", "plat/xen/arm/hypercalls*.S", "Test_Specifications/xen/hypercalls/**/*.md"]
---

# Group: hypercall wrappers

## What belongs here

The thin wrappers that turn a C call into a Xen hypercall:

- x86: `plat/xen/include/xen-x86/hypercall.h`, which includes
  `hypercall64.h` under `__X86_64__` and `hypercall32.h` under `__X86_32__`.
  The wrappers are `static inline` functions built on the `_hypercallN`
  macros, which `call` into `hypercall_page` at offset
  `__HYPERVISOR_<name> * 32`.
- arm: `plat/xen/include/xen-arm/hypercall.h` declares the wrappers, and
  `plat/xen/arm/hypercalls64.S` (or `hypercalls32.S`) defines each one with a
  `hypercall` assembler macro.

Every wrapper follows the same pattern, so one requirement shape and one test
shape cover the whole group. Do not write one prose section per wrapper.

Follow `.kiro/steering/xen-test-specifications.md` first. This file adds the
rules for this kind of code.

## Where specifications go

- `Test_Specifications/xen/hypercalls/x86_64.md`, IDs `REQ-HCALL-X86-NNN`.
- `Test_Specifications/xen/hypercalls/arm64.md`, IDs `REQ-HCALL-ARM64-NNN`.

Reviewed example to copy: `Test_Specifications/xen/hypercalls/x86_64.md`.

## Read before writing

1. Check which files are built (see the pairing step in
   `xen-group-arch-split.md`). x86-32 and arm32 are not selectable with
   `PLAT_XEN` in `plat/xen/Config.uk`; give `hypercall32.h` and
   `hypercalls32.S` a "not built" note only.
2. Read the mechanism once: the `_hypercallN` macro bodies (which register
   carries which argument, the `"memory"` clobber, the cast of the result) or
   the assembler `hypercall` macro (which register holds the hypercall number
   and which instruction traps to Xen).
3. Find where the mechanism's targets are defined. On x86, `hypercall_page`
   is reserved and exported in `plat/xen/x86/entry64.S` and advertised to Xen
   through an ELF note there. Cite it; whether Xen fills the page is an
   assumption.
4. Hypercall numbers come from `drivers/xen/xenheaders/include/xen/xen.h`,
   and some are remapped there depending on `__XEN_INTERFACE_VERSION__`,
   which `plat/xen/Makefile.uk` sets. Record the version and check the
   remapping block before writing any number into a requirement.
5. List the wrappers in a table: wrapper name, hypercall number macro, macro
   or entry used, argument order, return type. Note any wrapper that builds a
   structure first (for example `HYPERVISOR_shutdown` and `HYPERVISOR_suspend`).
6. For each wrapper, `grep` for callers in `plat/xen/` and `drivers/` so the
   spec can say which wrappers are actually used.

## What requirements look like

- **Mechanism requirement, once per architecture:** "A wrapper built on
  `_hypercall2` shall place its first argument in `rdi`, its second in `rsi`,
  call `hypercall_page + __HYPERVISOR_<name> * 32`, and return `rax` cast to
  the wrapper's return type."
- **Wrapper table requirement:** "Each wrapper in Table N shall invoke the
  hypercall number and argument order listed for it." One requirement plus a
  table is better than one requirement per wrapper.
- **Wrappers with extra logic:** a separate requirement for each wrapper that
  does more than forward (builds a structure, fixes an argument).
- **Return value handling belongs to callers.** Wrappers do not check results.
  Record which callers ignore the result as an open question in the caller's
  own specification, not here.

## Test methods for this group

- **Preprocessor expansion and disassembly:** compile a small test unit that
  calls each wrapper with distinctive constants, then inspect the generated
  code (`objdump -d`) for register setup and the call target offset.
- **Table check:** compare the wrapper table against `__HYPERVISOR_*` values in
  the Xen public headers used by the build.
- **Mocked hypercall page (proposed harness):** link the test against a
  `hypercall_page` stub that records registers, to check argument order
  without Xen.
- **Xen guest integration test:** only to show a real hypercall returns; list
  it as proposed.

Disassembly proves what the guest emits. It does not prove what Xen does with
the call; that is an assumption.

## Review before finishing

- Every built wrapper is in the table or listed as a coverage gap.
- The mechanism requirement cites the macro or assembler macro by name.
- No requirement claims Xen-side behavior.
