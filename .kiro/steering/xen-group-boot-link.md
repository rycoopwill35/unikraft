---
inclusion: fileMatch
fileMatchPattern: ["plat/xen/x86/*.S", "plat/xen/arm/entry*.S", "plat/xen/arm/link*.lds.S", "Test_Specifications/xen/boot/**/*.md"]
---

# Group: boot entry and linker scripts

## What belongs here

Assembly and linker-script files that decide how the image is laid out and
what state the CPU is in before C code runs:

- Entry code: `plat/xen/x86/entry64.S`, `plat/xen/arm/entry64.S`
  (and `entry32.S`, which is not built; see below).
- Default linker scripts: `plat/xen/x86/link64.lds.S`,
  `plat/xen/arm/link64.lds.S` (and `link32.lds.S`, not built).

These files cannot be unit tested the usual way. Their claims are about
symbols, sections, segments, and register state, so they are checked on the
built image.

Follow `.kiro/steering/xen-test-specifications.md` first. This file adds the
rules for this kind of code.

## Where specifications go

`Test_Specifications/xen/boot/<arch>/<file>.md`, for example
`Test_Specifications/xen/boot/x86_64/link64.md`. IDs
`REQ-BOOT-<ARCH>-<TOPIC>-NNN`, where `<TOPIC>` is `LDS` for a linker script
and `ENTRY` for entry code.

Reviewed example to copy: `Test_Specifications/xen/boot/x86_64/link64.md`.

## Read before writing

1. Find how the file is selected: `UK_PLAT_XEN_DEF_LDS` and the source lists in
   `plat/xen/Makefile.uk`, and the link and `objcopy` steps in
   `plat/xen/Linker.uk`. Only x86_64 and arm64 can be selected with
   `PLAT_XEN` (`plat/xen/Config.uk`); give 32-bit files a "not built" note.
2. A `.lds.S` file is run through the C preprocessor first. Find the
   definitions of every macro it uses (`EXCEPTION_SECTIONS`, `TLS_SECTIONS`,
   `DATA_SECTIONS`, `DISCARDS`, and the others) with
   `grep -rn "define <MACRO>" --include=*.h --include=*.lds.S .` and cite them.
   Do not describe what a macro adds without opening its definition.
3. In entry code, record which parts sit under `#ifdef XEN_PARAVIRT` or other
   guards, which symbols are made global, and which functions are called
   before the first C function.
4. Cross-check names between files: the `ENTRY(...)` symbol in the linker
   script must be defined in the entry code; symbols that `objcopy -G` keeps
   global in `Linker.uk` must exist.

## What requirements look like

- **Entry point:** "The x86_64 Xen image shall use `_libxenplat_start` as its
  ELF entry point."
- **Segment flags:** one requirement per program header with its type and
  flags, since each is checked on the image with the same tool.
- **Section order and alignment:** state order and alignment only where the
  script states them (`ALIGN(__PAGE_SIZE)`, `ALIGN(0x8)`).
- **Boundary symbols:** symbols such as `_text`, `_etext`, `_end` and where
  they are assigned.
- **Entry state:** for entry code, the ordered steps before the first C call,
  stated only as the code shows them. Do not claim what the CPU or Xen
  guarantees at entry; list it as an assumption.
- **Conditional parts:** name every `CONFIG_*` or `XEN_PARAVIRT` guard.

## Test methods for this group

- **Configured build plus image inspection:** `readelf -h` (entry point),
  `readelf -l` (program headers and flags), `readelf -S` or `objdump -h`
  (sections, order, alignment), `nm` (boundary symbols).
- **Preprocessed script check:** keep or regenerate the preprocessed linker
  script and compare it to the requirement.
- **Source inspection:** entry code ordering and guards.
- **Xen guest boot test:** only for claims that need a running guest; list it
  as proposed with its prerequisites.

Image inspection proves the layout the linker produced. It does not prove
that Xen loads or maps the image that way; that is an assumption.

## Review before finishing

- Every macro used by a linker script has its definition cited.
- Every symbol named in a requirement was found in the source.
- Guarded parts name their guard; 32-bit files only carry a "not built" note.
