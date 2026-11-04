# x86_64 Xen default linker script: example specification

> **Example only.** This document demonstrates the intended structure for
> source-derived requirements and tests in the boot-and-link group. It covers
> what `link64.lds.S` states directly; sections added through shared macros are
> listed as a coverage gap. It is not an approved safety artifact.

## Source scope

- `plat/xen/x86/link64.lds.S`: the x86_64 Xen default linker script.
- `plat/xen/Makefile.uk`: selects it as `UK_PLAT_XEN_DEF_LDS` when
  `CONFIG_ARCH_X86_64` is `y` (lines 39–40).
- `plat/xen/Linker.uk`: passes `UK_PLAT_XEN_DEF_LDS` to the final link with
  `-dT` (lines 27–28) and keeps `_libxenplat_start` global with `objcopy -G`
  (line 48).
- `plat/common/include/uk/plat/common/common.lds.h`: segment flag macros and
  the shared section macros used by the script.
- `plat/xen/x86/entry64.S`: defines the entry symbol.

## Source observations

- The script is preprocessed and includes `uk/arch/limits.h` (for
  `__PAGE_SIZE`) and `uk/plat/common/common.lds.h`.
- It sets output format `elf64-x86-64` and architecture `i386:x86-64`,
  declares the program headers, sets the entry point, and lays out text,
  read-only data, constructor arrays, and then a series of shared section macros.
- The entry symbol `_libxenplat_start` is defined in `entry64.S` inside
  `#ifdef XEN_PARAVIRT` (lines 49–61). `XEN_PARAVIRT` is defined when
  `CONFIG_XEN_HVMLITE` is not `y` (`plat/xen/Makefile.uk` lines 98–103).
  `XEN_HVMLITE` has no prompt and nothing selects it; its default is `n` on
  x86_64 (`plat/xen/Config.uk` lines 59–62), so on x86_64 `XEN_PARAVIRT` is
  always defined and the entry symbol is always assembled.

## Requirements

### REQ-BOOT-X86-LDS-001: output format and entry point

When the x86_64 Xen image is linked, the image shall be in `elf64-x86-64`
format for architecture `i386:x86-64`, with `_libxenplat_start` as its entry point.

**Source trace:** `plat/xen/x86/link64.lds.S` (lines 29–30 and `ENTRY(_libxenplat_start)` at line 45); `plat/xen/x86/entry64.S` (`.globl _libxenplat_start` line 56, label line 59).

### REQ-BOOT-X86-LDS-002: program headers

When the image is linked, it shall declare the program headers in Table 1, in
the order listed.

**Table 1**

| Name | Type | Flags in the script | Condition |
| --- | --- | --- | --- |
| `phdrs` | `PT_LOAD`, includes file and program headers | `PHDRS_PF_R` (R) | only when `CONFIG_ELF_PHDRS_IN_PT_LOAD` |
| `text` | `PT_LOAD` | `PHDRS_PF_RX` (R, X) | always |
| `rodata` | `PT_LOAD` | `PHDRS_PF_R` (R) | always |
| `data` | `PT_LOAD` | none stated | always |
| `tls` | `PT_TLS` | none stated | always |
| `tls_load` | `PT_LOAD` | none stated | always |
| `stack` | `PT_GNU_STACK` | `PHDRS_PF_RW` (R, W) | always |

**Source trace:** `plat/xen/x86/link64.lds.S:PHDRS` (lines 32–43); flag values in `plat/common/include/uk/plat/common/common.lds.h` (lines 48–55).

### REQ-BOOT-X86-LDS-003: text section

When the image is linked, the `.text` output section shall start at symbol
`_text`, contain the input sections `.text.boot`, `.text`, and `.gnu.warning`
in that order, be placed in the `text` program header with fill value
`0x2000`, and be followed by symbol `_etext`.

**Source trace:** `plat/xen/x86/link64.lds.S` (lines 58–65).

### REQ-BOOT-X86-LDS-004: read-only data section

When the image is linked, `_rodata` shall be aligned to `__PAGE_SIZE`, the
`.rodata` output section shall contain `.rodata` and `.rodata.*` input
sections in the `rodata` program header, and `_erodata` shall be aligned to
`__PAGE_SIZE` after it.

**Source trace:** `plat/xen/x86/link64.lds.S` (lines 67–75).

### REQ-BOOT-X86-LDS-005: constructor arrays

When the image is linked, `_ctors` shall be 8-byte aligned and followed by a
`.preinit_array` section and an 8-byte aligned `.init_array` section, each
bounded by its hidden `__*_start` and `__*_end` symbols and kept by `KEEP`;
`.init_array` shall place priority-sorted `.init_array.*` and `.ctors.*`
input before unsorted `.init_array` and `.ctors`; `_ectors` shall be aligned
to `__PAGE_SIZE` after them.

**Source trace:** `plat/xen/x86/link64.lds.S` (lines 85–101).

### REQ-BOOT-X86-LDS-006: base and end symbols

When the image is linked, `_base_addr` shall equal address `0x0`, and `_end`
shall be assigned after the `DATA_SECTIONS` content.

**Source trace:** `plat/xen/x86/link64.lds.S` (lines 48–50 and 105–107).

## Tests

### TEST-BOOT-X86-LDS-001: header, entry point, and program headers

**Traces to:** REQ-BOOT-X86-LDS-001, REQ-BOOT-X86-LDS-002

**Method:** configured build plus image inspection

**Status:** Proposed, not run

**Preconditions:** An x86_64 Xen configuration (`CONFIG_PLAT_XEN=y`,
`CONFIG_ARCH_X86_64=y`, `CONFIG_XEN_HVMLITE` not set), built twice: once with
and once without `CONFIG_ELF_PHDRS_IN_PT_LOAD`. `readelf` and `nm` available.

**Steps:**

1. Run `readelf -h` on each debug image (`*.dbg`) and read the class,
   machine, and entry point address.
2. Run `nm` and read the address of `_libxenplat_start`.
3. Run `readelf -l` and list program header types and flags in order.

**Expected result:** Class `ELF64`, machine `Advanced Micro Devices X86-64`;
entry point equals the address of `_libxenplat_start`. Program headers match
Table 1 in order and flags, with the `phdrs` entry present only in the build
with `CONFIG_ELF_PHDRS_IN_PT_LOAD`. Headers with no flags in the script are
recorded as observed (the linker chooses them) and compared against the team's
expectation in the review.

### TEST-BOOT-X86-LDS-002: section contents, order, and alignment

**Traces to:** REQ-BOOT-X86-LDS-003, REQ-BOOT-X86-LDS-004, REQ-BOOT-X86-LDS-005, REQ-BOOT-X86-LDS-006

**Method:** configured build plus image inspection, and linker map review

**Status:** Proposed, not run

**Preconditions:** The build from TEST-BOOT-X86-LDS-001, plus a linker map
(add `-Wl,-Map=<file>` to the final link) and `objdump`.

**Steps:**

1. Run `nm` and read `_base_addr`, `_text`, `_etext`, `_rodata`, `_erodata`,
   `_ctors`, `_ectors`, and `_end`.
2. In the map file, list the input sections placed in `.text`, `.rodata`,
   `.preinit_array`, and `.init_array`, in order.
3. Run `objdump -h` and read the start address of each of those output sections.

**Expected result:** `_base_addr` is `0x0`. `.text.boot` input comes before
any `.text` input in `.text`. `_rodata` and `_erodata` are multiples of
`__PAGE_SIZE`; `_ctors` and `.init_array` are multiples of 8; `_ectors` is a
multiple of `__PAGE_SIZE`. In `.init_array`, sorted priority sections come
before unsorted ones. `_etext` is not greater than `_rodata`, and `_end` is the
highest of these symbols.

## Assumptions

- Xen loads the image according to its program headers and starts it at the
  ELF entry point. Nothing in the script establishes this.

## Open questions

- The `data` and `tls_load` program headers have no flags in the script, so
  the linker chooses them. Should the team state the expected flags?

## Coverage gaps

- The content added by `EXCEPTION_SECTIONS`, `CTORTAB_SECTION`,
  `INITTAB_SECTION`, `EVENTTAB_SECTION`, `TLS_SECTIONS`, `DATA_SECTIONS`,
  `DEBUG_SYMBOLS`, and `DISCARDS` (all defined in
  `plat/common/include/uk/plat/common/common.lds.h`) is not specified here.
- The commented-out `/DISCARD/` block (lines 109–116) is not built.
- `plat/xen/x86/entry64.S` (entry code) needs its own specification.
