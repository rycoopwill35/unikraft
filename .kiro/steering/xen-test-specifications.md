---
inclusion: fileMatch
fileMatchPattern: ["Test_Specifications/xen/**/*.md", "plat/xen/**/*"]
---

# Xen capstone test-specification conventions

## Purpose and scope

Help write reviewable, source-derived requirements and proposed tests for this
fork of Unikraft. Store the resulting documents in `Test_Specifications/xen/`,
in the folder that the file's group steering names (see the table below).
Cover x86_64 first; cover arm64 only when asked.
Work on specification documents by default. Source, configuration, and build
changes are separate tasks and must not be introduced just to make a proposed
test pass.

## How the steering is layered

Steering for `plat/xen/` has three layers. Each layer adds to the one above it
and never contradicts it.

1. **Rules (this file).** Evidence, traceability, document structure, ID and
   test-claim rules for every Xen specification.
2. **Group files (`xen-group-*.md`).** One per kind of code. A group file says
   what requirements for that kind of code look like, which test methods fit,
   and points at one reviewed example specification to copy. Every source file
   in `plat/xen/` belongs to exactly one group.
3. **File add-ons (`xen-file-*.md`).** Optional, and only for a file whose
   generated specification failed review under its group file alone. An add-on
   lists the file-specific facts and traps that the group file missed. Do not
   create an add-on before the group file has been tried on that file.

| Group file | Source files | Specification folder | ID prefix |
| --- | --- | --- | --- |
| `xen-group-interface-headers.md` | `pal/**`, `include/uk/plat/xen/*.h`, `arch/**` | `Test_Specifications/xen/interface/` | `REQ-IF-<TOPIC>`; arch layer `REQ-ARCH-<ARCH>-<TOPIC>` |
| `xen-group-arch-split.md` | `x86/*.c`, `arm/*.c`, non-hypercall headers in `include/xen-x86/` and `include/xen-arm/` | `Test_Specifications/xen/x86/`, `.../arm64/`, `.../shared/` | `REQ-X86-<TOPIC>`, `REQ-ARM64-<TOPIC>`, shared `REQ-XEN-<TOPIC>` |
| `xen-group-hypercalls.md` | `include/xen-x86/hypercall*.h`, `include/xen-arm/hypercall.h`, `arm/hypercalls*.S` | `Test_Specifications/xen/hypercalls/` | `REQ-HCALL-<ARCH>` |
| `xen-group-boot-link.md` | `x86/*.S`, `arm/entry*.S`, `arm/link*.lds.S` | `Test_Specifications/xen/boot/` | `REQ-BOOT-<ARCH>-<TOPIC>` |
| `xen-group-core-runtime.md` | `*.c` at the top of `plat/xen/`, `include/common/*.h` | `Test_Specifications/xen/core/` | `REQ-CORE-<TOPIC>` |
| `xen-group-build-config.md` | `Config.uk`, `Makefile.uk`, `Linker.uk` | `Test_Specifications/xen/build/` | `REQ-BUILD-<TOPIC>` |

Paths in the "Source files" column are relative to `plat/xen/`. `<ARCH>` is
`X86` or `ARM64`. Each `REQ-` prefix has a matching `TEST-` prefix.

If a file seems to fit two groups, pick the group whose test method fits the
file's main claims, and say so in the specification's source scope.

## Evidence before requirements

- Read the target implementation, relevant declarations, direct callers, and
  build-selection rules before describing behavior.
- Use this checkout as the authority for implemented behavior. Clearly label
  external requirements, assumptions, suspected defects, and proposed changes.
- Cite repository-relative source paths and function or macro names for every
  requirement. If including line numbers, verify them against the current file;
  symbols remain the primary anchor.
- Distinguish observed implementation behavior from intended behavior. A
  source-derived requirement documents what the code does; it does not establish
  that the behavior is correct or sufficient for a safety standard.
- Do not infer error handling, concurrency guarantees, supported architectures,
  or timing bounds from function names or comments alone.

## Document structure

Follow the example specification that the file's group steering names. Every
example uses this layout, first set by `Test_Specifications/xen/x86/arch_events.md`:

1. Title and draft/example status; retain the existing example disclaimer.
2. Source scope, including dependencies needed to interpret behavior.
3. Source observations that explain the relevant control flow.
4. Requirements with stable IDs, a trigger/precondition, one observable
   obligation using "shall", and a source trace.
5. Tests with stable IDs, requirement links, method, preconditions, ordered
   steps, and an explicit expected result.
6. Assumptions, coverage gaps, or open questions when present.

Keep existing IDs stable. For new entries, use the `REQ-` and `TEST-` prefix
that the file's group gives in the table above, followed by `-NNN`; check
every document under `Test_Specifications/xen/` for collisions. Every requirement needs at least one proposed test, and every test
must name the requirements it checks. Do not silently broaden an existing ID's
meaning when adding unrelated behavior.

## Test quality and execution claims

- Select source inspection, an isolated test with controlled dependencies, a
  build/link check, or a Xen guest integration test based on the actual claim.
  State what each method can demonstrate.
- Identify required configuration, initial state, controlled inputs, captured
  operations, expected values, and ordering. Include boundary and failure paths
  when the source contains them; mark absent handling as a gap.
- Separate a proposed harness from one that exists. Do not invent runnable
  commands, fixtures, mocks, or test results.
- Report tests as proposed/not run until executed. For actual runs, record the
  command, configuration, environment, result, and evidence location. A Markdown
  review or successful build is not proof of guest runtime behavior.
- Report missing application configuration, toolchain, Xen host, or harness as
  prerequisites. Do not install dependencies or alter the environment merely to
  complete a documentation task.

## Review before finishing

Recheck source traces, ID uniqueness, requirement/test coverage, exact API and
configuration names, and whether the expected results are measurable. Summarize
the documented scope and remaining gaps without claiming full Xen coverage.
