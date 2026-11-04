# Xen TLB interface chain: example specification

> **Example only.** This document demonstrates the intended structure for
> source-derived requirements and tests in the interface-headers group. It is
> not an approved safety artifact.

## Source scope

- `plat/xen/pal/include/uk/plat/pal/tlb.h`: PAL layer, architecture independent.
- `plat/xen/include/uk/plat/xen/tlb.h`: Xen layer, architecture independent.
- No per-architecture layer exists for this topic: there is no `tlb.h` under
  `plat/xen/pal/arch/` or `plat/xen/arch/`.
- Consumers (context only, not evidence): `lib/ukpt/include/uk/pt.h`,
  `lib/ukpaging/include/uk/paging.h`, and
  `lib/ukpaging/arch/x86_64/include/uk/paging/arch.h` include the PAL header.
- Configuration context: `plat/Config.uk`, `plat/kvm/Config.uk`,
  `lib/ukpt/Config.uk`, `lib/ukpaging/Config.uk`.

## Source observations

- The PAL header includes the Xen header (`pal/tlb.h` line 11) and defines two
  `static inline` functions, each of which contains a single call to the
  matching Xen function.
- The Xen header defines the two Xen functions as `static inline` with empty
  bodies containing only a `/* No-op */` comment. Its header comment states
  "Xen platform does not support paging; declare no-ops" (line 10).
- In both headers the functions sit inside `#if !__ASSEMBLY__` and inside an
  `extern "C"` block when compiled as C++.
- `lib/ukpt/Config.uk` and `lib/ukpaging/Config.uk` make `LIBUKPT` and
  `LIBUKPAGING` depend on `HAVE_PAGING` (line 3 in each). `HAVE_PAGING` is declared in `plat/Config.uk` (line 69) and, among
  the platform configs, is selected by `plat/kvm/Config.uk` (line 19);
  `plat/xen/Config.uk` does not select it.

## Requirements

### REQ-IF-TLB-001: PAL single-entry flush forwards to Xen

When `uk_pal_tlb_flush_entry` is called with a virtual address, the
implementation shall call `uk_plat_xen_tlb_flush_entry` with that same address
and perform no other statement.

**Source trace:** `plat/xen/pal/include/uk/plat/pal/tlb.h:uk_pal_tlb_flush_entry` (lines 19–23).

### REQ-IF-TLB-002: PAL full flush forwards to Xen

When `uk_pal_tlb_flush` is called, the implementation shall call
`uk_plat_xen_tlb_flush` and perform no other statement.

**Source trace:** `plat/xen/pal/include/uk/plat/pal/tlb.h:uk_pal_tlb_flush` (lines 25–29).

### REQ-IF-TLB-003: Xen single-entry flush is a no-op

When `uk_plat_xen_tlb_flush_entry` is called, the implementation shall execute
no statement and shall not use its `vaddr` argument.

**Source trace:** `plat/xen/include/uk/plat/xen/tlb.h:uk_plat_xen_tlb_flush_entry` (lines 25–29).

### REQ-IF-TLB-004: Xen full flush is a no-op

When `uk_plat_xen_tlb_flush` is called, the implementation shall execute no
statement.

**Source trace:** `plat/xen/include/uk/plat/xen/tlb.h:uk_plat_xen_tlb_flush` (lines 34–38).

### REQ-IF-TLB-005: availability conditions

The four functions in REQ-IF-TLB-001 to REQ-IF-TLB-004 shall be defined only
when `__ASSEMBLY__` is not set, and shall have C linkage when the headers are
compiled as C++.

**Source trace:** `plat/xen/pal/include/uk/plat/pal/tlb.h` (`extern "C"` lines 13–15 and 33–35, `#if !__ASSEMBLY__` lines 17–31); `plat/xen/include/uk/plat/xen/tlb.h` (`extern "C"` lines 14–16 and 42–44, `#if !__ASSEMBLY__` lines 18–40).

## Tests

### TEST-IF-TLB-001: PAL forwarding check

**Traces to:** REQ-IF-TLB-001, REQ-IF-TLB-002

**Method:** source inspection and preprocessor expansion check

**Status:** Proposed, not run

**Preconditions:** A C compiler and the Xen x86_64 include paths from
`plat/xen/Makefile.uk` ("Arch includes" and "PAL includes").

**Steps:**

1. Inspect the bodies of `uk_pal_tlb_flush_entry` and `uk_pal_tlb_flush`.
2. Write a test unit that includes `<uk/plat/pal/tlb.h>` and calls
   `uk_pal_tlb_flush_entry(0x1234000)` and `uk_pal_tlb_flush()`.
3. Run the compiler with `-E` on the test unit.

**Expected result:** In the expanded output, the body of
`uk_pal_tlb_flush_entry` is exactly one call to `uk_plat_xen_tlb_flush_entry`
passing `vaddr`, and the body of `uk_pal_tlb_flush` is exactly one call to
`uk_plat_xen_tlb_flush`.

### TEST-IF-TLB-002: Xen no-op check

**Traces to:** REQ-IF-TLB-003, REQ-IF-TLB-004

**Method:** source inspection and disassembly

**Status:** Proposed, not run

**Preconditions:** Same as TEST-IF-TLB-001, plus `objdump`.

**Steps:**

1. Inspect the bodies of `uk_plat_xen_tlb_flush_entry` and `uk_plat_xen_tlb_flush`.
2. Compile the test unit from TEST-IF-TLB-001 at `-O2` into an object file,
   with the PAL calls placed inside a non-inline test function.
3. Disassemble the test function with `objdump -d`.

**Expected result:** Both Xen bodies contain no statements. The disassembled
test function contains no call instruction and no memory access produced by
the two PAL calls; it consists only of the function return.

### TEST-IF-TLB-003: availability conditions check

**Traces to:** REQ-IF-TLB-005

**Method:** preprocessor expansion check

**Status:** Proposed, not run

**Preconditions:** Same as TEST-IF-TLB-001, plus a C++ compiler.

**Steps:**

1. Preprocess the test unit with `-D__ASSEMBLY__=1` and search the output for
   the four function names.
2. Preprocess it without that define and search again.
3. Preprocess it as C++ and inspect what surrounds the four definitions.

**Expected result:** Step 1 finds none of the four names; step 2 finds all
four; step 3 shows all four inside `extern "C" { ... }`.

## Assumptions

- Xen PV guests manage page tables through hypercalls rather than through this
  API. That design is outside these headers and is not established here.

## Open questions

- In-tree consumers of this API (`lib/ukpt`, `lib/ukpaging`) cannot be enabled on Xen,
  because `plat/xen/Config.uk` does not select `HAVE_PAGING`. Should the
  no-op behavior be documented as a constraint for any future consumer?

## Coverage gaps

- None within the two headers. Consumer behavior in `lib/ukpt` and
  `lib/ukpaging` is out of scope.
