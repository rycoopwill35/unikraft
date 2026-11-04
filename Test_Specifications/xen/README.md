# Xen specifications and how the Kiro steering is organized

Specifications under this folder are generated with Kiro and reviewed by the
team. Kiro is guided by steering files in `.kiro/steering/`, organized in
three layers instead of one file per source file.

## The three layers

| Layer | Files | Loads when you work on | What it holds |
| --- | --- | --- | --- |
| 1. Rules | `xen-test-specifications.md` | anything in `plat/xen/` or `Test_Specifications/xen/` | evidence, traceability, document layout, ID and test-claim rules |
| 2. Groups | `xen-group-*.md` (6 files) | files of one kind of code | what requirements look like for that kind of code, which tests fit, and one reviewed example to copy |
| 3. File add-ons | `xen-file-*.md` | one or a few specific files | facts and traps the group file missed for those files |

Write a layer 3 add-on only after a file's specification failed review with
its group file alone. That keeps the steering small and puts per-file work
where the code actually needs it.

## Groups

| Group | Source files in `plat/xen/` | Specs go in | Reviewed example |
| --- | --- | --- | --- |
| Interface headers | `pal/**`, `include/uk/plat/xen/*.h`, `arch/**` (32 files) | `interface/` | `interface/tlb.md` |
| Arch-split subsystems | `x86/*.c`, `arm/*.c`, non-hypercall `include/xen-x86/*.h` and `include/xen-arm/*.h` (24 files) | `x86/`, `arm64/`, `shared/` | `x86/arch_events.md`, `x86/arch_time.md` |
| Hypercall wrappers | `include/xen-*/hypercall*.h`, `arm/hypercalls*.S` (6 files) | `hypercalls/` | `hypercalls/x86_64.md` |
| Boot entry and linker scripts | `x86/*.S`, `arm/entry*.S`, `arm/link*.lds.S` (6 files) | `boot/` | `boot/x86_64/link64.md` |
| Core runtime | top-level `*.c`, `include/common/*.h` (6 files) | `core/` | `core/shutdown.md` |
| Build and configuration | `Config.uk`, `Makefile.uk`, `Linker.uk` (3 files) | `build/` | none yet; scope to confirm with the sponsor |

## File add-ons in place

| Add-on | Covers | Origin |
| --- | --- | --- |
| `xen-file-arch-x86_64.md` | `arch/x86_64/` headers | trimmed from the earlier `xen-arch.md` |
| `xen-file-x86-events-time.md` | `x86/arch_events.c`, `x86/arch_time.c` | the earlier `xen-x86.md` |
| `xen-file-x86-setup.md` | `x86/setup.c` | the earlier `xen-x86-setup.md`, with its file pattern fixed |

## Checking that steering loads

`fileMatch` steering only loads automatically in the Kiro IDE and Kiro CLI V3.
Older CLI versions (V1/V2) only auto-load `inclusion: always` files. Before
generating, open a source file in Kiro and confirm the expected steering files
appear in the context.
