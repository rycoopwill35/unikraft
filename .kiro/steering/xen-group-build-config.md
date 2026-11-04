---
inclusion: fileMatch
fileMatchPattern: ["plat/xen/Config.uk", "plat/xen/Makefile.uk", "plat/xen/Linker.uk", "Test_Specifications/xen/build/**/*.md"]
---

# Group: build and configuration rules

## What belongs here

- `plat/xen/Config.uk`: the Kconfig options for the Xen platform, their
  dependencies, defaults, and `select` and `imply` relationships.
- `plat/xen/Makefile.uk`: which sources, include paths, and flags are used
  for which configuration.
- `plat/xen/Linker.uk`: how the image is linked, post-processed, and named.

**Scope status:** whether the build rules get their own specifications is an
open question for the sponsor. Until the team confirms it, use this group
mainly as a reference: other groups read these files to find out what is
built. Only write a specification here when asked to.

Follow `.kiro/steering/xen-test-specifications.md` first. This file adds the
rules for this kind of code.

## Where specifications go

`Test_Specifications/xen/build/<file>.md` (`config.md`, `makefile.md`,
`linker.md`). IDs `REQ-BUILD-<TOPIC>-NNN`, where `<TOPIC>` is `CONFIG`,
`MAKE`, or `LINK`.

No reviewed example exists yet. When the first one is written and reviewed,
name it here.

## Read before writing

1. Treat each rule as a mapping from configuration to result, and write it
   down as a table before drafting requirements: option, condition, effect.
2. For `Config.uk`, record each option's type, default with its condition,
   `depends on`, `select`, and `imply`. A `select` forces an option on; an
   `imply` only sets a default the user can change. Do not mix them up.
3. For `Makefile.uk`, record each `-y` or `-$(CONFIG_...)` line and the
   `ifeq`/`ifneq` block around it.
4. Report rules that cannot take effect, such as source lines guarded by an
   architecture option that `Config.uk` does not allow with `PLAT_XEN`.
   Describe them as observations; do not change the build files.

## What requirements look like

- "When `CONFIG_ARCH_X86_64` is `y`, the build shall use
  `plat/xen/x86/link64.lds.S` as the default linker script."
- "When `CONFIG_XEN_HVMLITE` is not `y`, the build shall add `-DXEN_PARAVIRT`
  to the assembler and C flags."
- "`PLAT_XEN` shall only be selectable when `ARCH_X86_64` or `ARCH_ARM_64`
  is set."

One configuration, one observable effect per requirement.

## Test methods for this group

- **Configuration resolution check:** generate a configuration (for example
  with the project's `make menuconfig` or a defconfig) and inspect the
  resulting `.config` for the expected values.
- **Build log inspection:** run a configured build with verbose output and
  check that the expected sources, include paths, and flags appear.
- **Image inspection:** for `Linker.uk`, check the produced file names and the
  symbols kept global.

List the toolchain and configuration each check needs. Do not claim a build
was run without its log.
