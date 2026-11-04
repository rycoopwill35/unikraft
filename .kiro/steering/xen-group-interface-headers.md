---
inclusion: fileMatch
fileMatchPattern: ["plat/xen/pal/**/*", "plat/xen/include/uk/plat/xen/*.h", "plat/xen/arch/**/*", "Test_Specifications/xen/interface/**/*.md"]
---

# Group: interface header chains

## What belongs here

Headers that form the platform API, as layers that forward to each other:

1. PAL, architecture independent: `plat/xen/pal/include/uk/plat/pal/<topic>.h`
2. PAL, per architecture: `plat/xen/pal/arch/<arch>/include/uk/plat/pal/arch/<topic>.h`
3. Xen, architecture independent: `plat/xen/include/uk/plat/xen/<topic>.h`
4. Xen, per architecture: `plat/xen/arch/<arch>/include/uk/plat/xen/arch/<topic>.h`

Topics share names across layers (`addr`, `auxsp`, `except`, `page`,
`paging`, `pt`, `start`, `sysctx`, `tlb`, plus the PAL-only `ectx` and
`regs`). A chain is all the layers for one topic. Specify a chain as a unit,
because a PAL requirement usually depends on what a lower layer does.

Follow `.kiro/steering/xen-test-specifications.md` first. This file adds the
rules for this kind of code.

## Where specifications go

- Architecture-independent layers of a topic (layers 1 and 3):
  `Test_Specifications/xen/interface/<topic>.md`, IDs `REQ-IF-<TOPIC>-NNN`.
- Per-architecture layers of a topic (layers 2 and 4):
  `Test_Specifications/xen/interface/<arch>/<topic>.md`, IDs
  `REQ-ARCH-<ARCH>-<TOPIC>-NNN` (for example `REQ-ARCH-X86-EXCEPT-001`).

Reviewed example to copy: `Test_Specifications/xen/interface/tlb.md`.

## Read before writing

1. List the current files for the topic in all four layers with
   `find plat/xen -name '<topic>.h'`. Do not rely on a remembered list.
2. Read each layer top to bottom and draw the chain: which PAL item calls or
   aliases which Xen item, and which Xen item is defined in the arch layer.
3. For every item a header declares but does not define, find the definition
   with `grep -rn "<symbol>" --include=*.c --include=*.S --include=*.h .` and
   cite its path. Definitions often sit in `plat/xen/x86/`, `plat/xen/arm/`,
   or a top-level `plat/xen/*.c` file.
4. Find the consumers outside `plat/xen/` with
   `grep -rn "uk_pal_<topic>\|UK_PAL_" --include=*.c --include=*.h lib/ plat/common/`.
   Consumers show what the API is used for; they are not evidence of behavior.
5. Check the include rules in `plat/xen/Makefile.uk` ("Arch includes" and
   "PAL includes") and the `XEN_PARAVIRT` flag, which is defined for assembly
   and C but not C++, and only when `CONFIG_XEN_HVMLITE` is not `y`. That
   is always true on x86_64; on arm64 `XEN_HVMLITE` defaults to `y`, so
   `XEN_PARAVIRT` code is not built there (`plat/xen/Config.uk`).
6. `plat/native/pal/` has headers with the same names. They are a different
   platform. Never use them as evidence for Xen.

## What requirements look like

Interface headers mostly make static claims. Typical obligations:

- **Forwarding:** "`uk_pal_tlb_flush` shall call `uk_plat_xen_tlb_flush` with no
  other statements."
- **Macro expansion:** "`UK_PAL_SENTRY_SYM` shall expand to `uk_plat_xen_sentry`."
- **Accessors:** "`<getter>` shall return `ctx-><field>` unchanged."
- **No-op or stub bodies:** "`uk_plat_xen_tlb_flush` shall have an empty body."
  State what the body contains; do not claim that a no-op is safe.
- **Conditional availability:** name the guard (`__ASSEMBLY__`,
  `XEN_PARAVIRT`, `__cplusplus`, `CONFIG_*`) for every item inside one.
- **Compile-time checks:** record each `#error` and the condition that
  triggers it.

Trace each requirement to the layer that actually does the work. If a PAL
requirement depends on a lower layer, either cite that layer's requirement ID
or list the dependency as an assumption. A group of macros that one test can
check together may share one requirement with a mapping table.

## Test methods for this group

- **Source inspection:** forwarding bodies, guards, include guards, stub bodies.
- **Preprocessor expansion check:** run the compiler with `-E` using the Xen
  include paths for the target architecture, once with and once without
  `XEN_PARAVIRT`, and compare the expansions against the requirement.
- **Compile-time assertion:** `_Static_assert` for type, size, and offset claims.
- **Isolated unit test:** for inline functions with logic, compile the header
  in a test file with stubbed dependencies and use distinctive input values so
  a dropped or swapped value is visible.
- **Configured build plus symbol inspection:** for claims that a symbol or
  section name lines up between producer and consumer.

These methods prove what the header says. They do not prove that hardware or
Xen behaves as the header assumes; list those as assumptions.

## Review before finishing

- Every macro, structure, and inline function in the chain is covered by a
  requirement or listed as a coverage gap.
- Every declared item has its definition cited or is listed as unresolved.
- PAL and Xen layers agree; any mismatch is a labeled, unconfirmed discrepancy.
- Native headers were not used as evidence.
