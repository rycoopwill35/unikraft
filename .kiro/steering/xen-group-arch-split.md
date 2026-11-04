---
inclusion: fileMatch
fileMatchPattern: ["plat/xen/x86/*.c", "plat/xen/arm/*.c", "plat/xen/include/xen-x86/mm.h", "plat/xen/include/xen-x86/mm_pv.h", "plat/xen/include/xen-x86/setup.h", "plat/xen/include/xen-x86/smp.h", "plat/xen/include/xen-x86/traps.h", "plat/xen/include/xen-arm/mm.h", "plat/xen/include/xen-arm/os.h", "plat/xen/include/xen-arm/setup.h", "plat/xen/include/xen-arm/smp.h", "plat/xen/include/xen-arm/traps.h", "Test_Specifications/xen/x86/**/*.md", "Test_Specifications/xen/arm64/**/*.md", "Test_Specifications/xen/shared/**/*.md"]
---

# Group: architecture-split subsystems

## What belongs here

C files that implement one platform subsystem separately for each
architecture (`arch_events`, `arch_time`, `mm`, `setup`, `sysctx`, `traps`),
and the per-architecture headers that support them. The x86 and ARM versions
usually implement the same interface, so requirements can often be written
once and then refined per architecture.

Follow `.kiro/steering/xen-test-specifications.md` first. This file adds the
rules for this kind of code.

## Where specifications go

- x86_64 behavior: `Test_Specifications/xen/x86/<file>.md`, IDs `REQ-X86-<TOPIC>-NNN`.
- arm64 behavior: `Test_Specifications/xen/arm64/<file>.md`, IDs `REQ-ARM64-<TOPIC>-NNN`.
- Behavior shared by both built implementations:
  `Test_Specifications/xen/shared/<subsystem>.md`, IDs `REQ-XEN-<TOPIC>-NNN`.

Keep the existing `REQ-X86-EVENT`, `REQ-X86-TIME`, and `REQ-X86-SETUP` IDs.

Reviewed examples to copy: `Test_Specifications/xen/x86/arch_events.md`
(hooks and call order) and `Test_Specifications/xen/x86/arch_time.md`
(state changes through hypercalls and event channels).

## Step 1: build the pairing table from the build rules

Before writing anything, read the source lists in `plat/xen/Makefile.uk` and
the `PLAT_XEN` dependencies in `plat/xen/Config.uk`, and record which file is
built for which architecture. A file that no selectable configuration builds
gets a one-line "not built" note in the coverage gaps, not requirements.

As of this writing the rules read as follows; recheck them, do not copy them:

| Subsystem | x86_64 (`CONFIG_ARCH_X86_64`) | arm64 (`CONFIG_ARCH_ARM_64`) |
| --- | --- | --- |
| events hooks | `x86/arch_events.c` | `arm/arch_events.c` |
| time | `x86/arch_time.c` | none in `plat/xen/`; `plat/common/arm/time.c` and `generic_timer.c` are built instead |
| memory | `x86/mm.c` | `arm/mm.c` |
| setup | `x86/setup.c` | `arm/setup64.c` |
| system context | `x86/sysctx.c` | `arm/sysctx.c` |
| traps | `x86/traps.c` | none selected |

Observations to confirm and report, not to fix:

- `Config.uk` limits `PLAT_XEN` to `ARCH_X86_64 || ARCH_ARM_64`, so the
  `CONFIG_ARCH_ARM_32` source lines in `Makefile.uk` cannot be selected
  together with Xen. That covers `arm/setup.c`, `arm/traps.c`,
  `arm/arch_time.c`, and also `arm/traps64.c`, which is listed under
  `CONFIG_ARCH_ARM_32` despite its name.
- A subsystem is "paired" only when both architectures build a file for it.

## Step 2: decide shared or per-architecture

Write a shared `REQ-XEN-` requirement only when all of these hold:

1. Both architectures build an implementation (Step 1).
2. Both implementations perform the same observable obligation, and you can
   cite the exact function in each file.
3. One test procedure, run once per architecture, can check it.

Each shared requirement cites both source locations. Anything that differs
between the two (register names, hypercall versus system register, extra
steps) becomes a per-architecture requirement that refines the shared one and
names its ID. Never move behavior from one architecture to the other because
the function names match.

## What requirements look like

- **Hook bodies:** empty hooks are described as having no state-changing
  statements, with the caller that invokes them.
- **Call order:** when a common function calls architecture hooks, state the
  order and test it as an ordered trace.
- **State changes through Xen:** "shall request a timer expiry of X", "shall
  bind VIRQ Y and unmask the returned port". Capture the hypercall or event
  channel argument as the test oracle.
- **Failure paths:** if the source checks a return value, specify the branch.
  If it does not, record the missing check as an open question; do not invent
  recovery behavior.
- **Configuration guards:** name every `CONFIG_*` or `XEN_PARAVIRT` condition.

## Test methods for this group

- **Source inspection and build/link check** for hooks and symbol resolution.
- **Isolated unit test with mocked hypercalls and event channels** (Ceedling
  style, like the Boeing reference procedures) for logic and argument values.
  Control inputs such as the clock reading so the expected value is exact.
- **Ordered call trace** through mocks for sequences.
- **Xen guest integration test** only for claims that need real Xen behavior;
  describe it as proposed and list its prerequisites.

## Review before finishing

- The pairing table was rechecked against the current `Makefile.uk`.
- Shared requirements cite both architectures; per-architecture requirements
  link to the shared one they refine.
- No requirement exists for a file that is not built.
