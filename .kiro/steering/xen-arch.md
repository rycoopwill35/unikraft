---
inclusion: fileMatch
fileMatchPattern: ["plat/xen/arch/**/*", "plat/xen/include/uk/plat/xen/*.h", "Test_Specifications/xen/arch/**/*.md"]
---

# Xen arch backend headers: directory-level steering

## Purpose and scope

Use this guidance when reviewing any header under `plat/xen/arch/` or drafting/revising specifications under `Test_Specifications/xen/arch/` in this Unikraft fork. Follow `.kiro/steering/xen-test-specifications.md` as the primary project-wide rules. This file adds arch-folder review prompts; it does not replace those rules.

Produce **source-derived, reviewable draft requirements and proposed tests**, not approved safety artifacts. Modify specification documents only unless the user separately requests source changes. Do not claim that any test has been executed without actual execution evidence.

`plat/xen/arch/` holds the Xen platform's architecture-specific backend headers. The PAL headers in `plat/xen/pal/` forward to them, and `xen-pal.md` tells PAL specifications to treat backend behavior as an assumption. **The specifications written under this file are where that backend behavior is actually specified.** Keep the two consistent: when a PAL requirement assumes a backend behavior, the matching arch requirement should state it, or the gap should be listed.

Cover the `x86_64` headers. The `arm64` headers are out of scope unless the user asks; do not transfer arm64 behavior into an x86_64 requirement or the reverse. On arm64 most items forward to native, so their behavior is not established by these headers at all.

## Inspect source and dependencies first

1. List the **current checked-out contents** of `plat/xen/arch/` (for example with `find plat/xen/arch -type f`). The folder contains headers only, in `plat/xen/arch/<arch>/include/uk/plat/xen/arch/`. Do not rely on a remembered file list.
2. Read the architecture-independent parent headers that include each arch header: `plat/xen/include/uk/plat/xen/except.h`, `sysctx.h`, and `auxsp.h`. Record what the parent declares, what it checks with `#error`, and which declarations the x86_64 arch header actually defines.
3. For every symbol an arch header declares but does not define, find the definition with `grep -rn "<symbol>" --include=*.c --include=*.S --include=*.h .` and record its path. Known examples to verify: `uk_plat_xen_auxsp` (`plat/xen/x86/traps.c`, read by `plat/xen/x86/entry64.S`), `uk_plat_xen_irqs_handle_pending` (`plat/xen/hypervisor.c`), and `uk_plat_xen_tlsp_set`, `uk_plat_xen_sysctx_load`, `uk_plat_xen_sysctx_store` (`plat/xen/x86/sysctx.c`).
4. Find the consumers: the PAL wrappers in `plat/xen/pal/` and any other user found with `grep -rn "uk_plat_xen_\|UK_PLAT_XEN_" --include=*.h --include=*.c --include=*.S . | grep -v "^./plat/xen/arch/"`.
5. Inspect the build rules in `plat/xen/Makefile.uk` and `plat/xen/Config.uk`: the "Arch includes" block that adds `arch/$(CONFIG_UK_ARCH)/include`, and the `XEN_PARAVIRT` flag, which is added only when `CONFIG_XEN_HVMLITE` is not `y`. State which items are conditional on these.
6. Use the **actual code** as evidence. Do not infer behavior from symbol names, header comments, or the intended platform architecture alone.

## Traceability rule for this folder

Arch headers are small, but most of what they do depends on code defined somewhere else (shared memory set up at boot, per-CPU variables, out-of-line functions, macros from other headers). Apply these rules strictly:

- **Only state a requirement you can trace to a specific line of source** in this checkout. Cite the repository-relative path and the macro or function name.
- When behavior depends on code outside `plat/xen/arch/`, either open and cite that code, or record the dependency as an explicit assumption. **Do not fill the gap with a guess** about what the other code does.
- If you are unsure whether something is true, list it under **Open questions** instead of writing a requirement.
- Distinguish three kinds of statement and label them: what the header itself does, what a backend or caller does (cite it), and what is assumed about Xen or the hardware (assumption).

## x86_64 behavior to investigate

Verify each item against the current source before turning it into a requirement.

### `except.h`

- **Event identifiers.** The header defines nine `UK_PLAT_XEN_EXCEPT_EVENT_*` macros required by the `#error` checks in the parent `except.h`, plus two x86_64-only macros, `UK_PLAT_XEN_X86_64_EXCEPT_EVENT_ERR_GP_FAULT` and `UK_PLAT_XEN_X86_64_EXCEPT_EVENT_NMI`. Record what each expands to.
- **Producer and consumer names must agree.** `UK_EVENT_HANDLER_PRIO` in `include/uk/event.h` places a handler in a section named after the *expanded* event name. Compare the names that handlers register against (through the PAL and `lib/uklcpu` macro chain, for example `lib/uknofault/maccess.c`) with the names `plat/xen/x86/traps.c` defines and raises with `UK_EVENT` and `UK_EVENT_PTR`. In the current source, `traps.c` uses `UK_PLAT_XEN_EXCEPT_EVENT_ERR_X86_GP` and `UK_PLAT_XEN_EXCEPT_EVENT_NMI`, neither of which is the x86_64 macro name defined in this header. Treat this as a **suspected discrepancy**: describe it with exact citations, propose a build and symbol/section inspection to confirm it, and do not claim a runtime consequence until that evidence exists.
- **Context structures and accessors.** `struct uk_plat_xen_except_err_ctx` and `struct uk_plat_xen_except_irq_ctx` are concretely defined here. Each `get`/`set` accessor reads or writes exactly one field. Record each accessor's field, its declared types, and its `__isr` annotation. Note observed inconsistencies as open questions, not defects, unless a consequence is shown. Examples to check: `uk_plat_xen_except_irq_ctx_get_irq` returns `__u64` while the `irq` field and the setter use `__u32`; the fault-address accessors use the field `cr2`; only the trap number and error code accessors carry an `x86_64` name infix.
- **IRQ control (only under `XEN_PARAVIRT`).** `uk_plat_xen_enable_irq`, `uk_plat_xen_disable_irq`, `uk_plat_xen_save_irqf`, `uk_plat_xen_restore_irqf`, and `uk_plat_xen_irqs_disabled` operate on `evtchn_upcall_mask` and `evtchn_upcall_pending` in `HYPERVISOR_shared_info->vcpu_info[smp_processor_id()]`. For each, trace exactly: which value is written, which value is returned, when `uk_plat_xen_irqs_handle_pending` is called, and where `__barrier()` appears. Note that `restore_irqf` only checks pending events when the restored flags are zero.
- **Limits of what IRQ control establishes.** `__barrier()` in `include/uk/compiler.h` is a compiler barrier only; do not describe it as a hardware memory fence. `smp_processor_id()` in `plat/xen/include/xen-x86/smp.h` is defined as `0`, so these functions always use `vcpu_info[0]`; do not claim per-CPU behavior. Whether masking actually stops event delivery is Xen behavior and must be an assumption.
- **Stubs.** `uk_plat_xen_except_get_except_stack_base` returns `0`, `uk_plat_xen_except_push_nested` and `uk_plat_xen_except_pop_nested` have empty bodies, and `uk_plat_xen_except_init` returns `0`. Describe them as stubs with no state changes, using the header's own comments only as supporting context.
- **Conditional availability.** Everything from `#ifdef XEN_PARAVIRT` to its `#endif` exists only when that flag is defined. In `plat/xen/Makefile.uk` it is added to `ASFLAGS` and `CFLAGS` but not `CXXFLAGS`. Raise as an open question whether a C++ translation unit that reaches these functions through the PAL resolves them.
- **Parent declarations without an x86_64 definition.** The parent `except.h` declares `uk_plat_xen_halt` and `uk_plat_xen_halt_irq`. Check whether any x86_64 definition exists and record the result as a gap if none is found.

### `auxsp.h`

- `UK_PLAT_XEN_AUXSP_SYM` is defined outside `!__ASSEMBLY__`, so it is visible to assembly; record that and find its assembly users.
- `uk_plat_xen_set_auxsp` writes the current CPU's `uk_plat_xen_auxsp` per-CPU variable through `uk_pcpuvar_current_set`. It is not annotated `__isr`, and the header provides no getter. Treat per-CPU storage semantics as a `uk/pcpuvar.h` contract (cite it or list it as an assumption).

### `sysctx.h`

- `uk_plat_xen_tlsp_get` returns the result of `uk_plat_native_tlsp_get` unchanged. That native behavior is an assumption here; do not use native code as evidence for Xen-specific behavior.
- `uk_plat_xen_tlsp_set`, `uk_plat_xen_sysctx_load`, and `uk_plat_xen_sysctx_store` are declared by the parent `sysctx.h` and defined out of line in `plat/xen/x86/sysctx.c`, which issues `HYPERVISOR_set_segment_base` hypercalls. That file belongs to the x86 folder scope. Reference it for traceability, but do not duplicate requirements that belong in `Test_Specifications/xen/x86/`; coordinate instead.

### Header structure

- Each arch header uses an include guard `__UK_PLAT_XEN_ARCH_<NAME>_H__`. Unlike the PAL arch headers, these do **not** contain a `#error "Do not include this header directly"` check; report that as observed.
- Record which items sit inside `#if !__ASSEMBLY__` and `extern "C"`.

## Requirements drafting rules

Write one specification per x86_64 header: `Test_Specifications/xen/arch/x86_64/<header>.md` (`except.md`, `auxsp.md`, `sysctx.md`). Use the same layout as the team's `arch_time.md` and `arch_events.md` examples:

1. Title and explicit **draft/example, not an approved safety artifact** notice.
2. Source scope, the parent headers and out-of-line definitions relied on, and relevant configuration (`CONFIG_ARCH_X86_64`, `XEN_PARAVIRT`, `CONFIG_XEN_HVMLITE`).
3. Concise source observations, including which items are defined inline here, which are only declared, and where each declared item is defined.
4. Requirements with stable unique IDs `REQ-ARCH-X86-<TOPIC>-NNN`, where `<TOPIC>` is `EXCEPT`, `AUXSP`, or `SYSCTX`. Each has a trigger or precondition, one observable obligation using **shall**, and a trace to an exact repository-relative path and macro or function.
5. Proposed tests with stable unique IDs `TEST-ARCH-X86-<TOPIC>-NNN`, links to the requirements they cover, test method, preconditions, ordered steps, and explicit measurable expected results.
6. Assumptions (Xen, shared-info setup, per-CPU storage, native backends), prerequisites, open questions, suspected discrepancies, and coverage gaps.

Typical obligations: macro expansion ("`UK_PLAT_XEN_EXCEPT_EVENT_DEBUG` shall expand to `xen_except_event_debug`"), accessor behavior ("`uk_plat_xen_except_err_ctx_set_handler_err` shall store its `handler_err` argument in `ctx->handler_err`"), IRQ control state changes ("when `uk_plat_xen_save_irqf` is called, it shall return the previous `evtchn_upcall_mask` value and leave the mask set to `1`"), conditional pending-event handling, stub return values, and conditional availability. A group of closely related macros, such as the event identifiers, may share one requirement with a mapping table if one test checks all of them. Preserve existing IDs, check for duplicates across `Test_Specifications/xen/`, and do not invent requirements for behavior the headers do not establish.

## Testing and evidence

- Select an appropriate method for each claim and explain what it proves:
  - **Source inspection** for structure, guards, and stub bodies.
  - **Preprocessor expansion check** (`-E` with the Xen x86_64 include paths and `XEN_PARAVIRT` defined, and again without it) for macro expansion and conditional availability.
  - **Compile-time assertion** (`_Static_assert`) for type and offset claims.
  - **Isolated unit test** for accessors and IRQ control: provide a stub `HYPERVISOR_shared_info` with a controlled `vcpu_info[0]`, and a stub `uk_plat_xen_irqs_handle_pending` that counts calls. Use distinctive values so a dropped, truncated, or swapped value is visible, and cover both pending and not-pending cases, and zero and nonzero flags for `restore_irqf`.
  - **Configured build plus symbol and section inspection** (for example `nm` or `objdump -h` on the linked image) for the event-name agreement check.
- A unit test with stubs proves the header's logic, not Xen's. It cannot prove that masking stops event delivery or that barrier placement prevents a race; ordering around `__barrier()` can be shown only by inspection or disassembly. A build proves the headers compile together, not guest runtime behavior.
- **Do not invent** an existing harness, executable commands, stubs, results, or infrastructure. Describe any harness as proposed and list the include paths, defines, and configuration it would need.
- Label all unexecuted tests **Proposed, not run**. If tests are eventually run, record the command, configuration, environment, result, and evidence location.
- Ensure each requirement has at least one proposed test and each test identifies its covered requirement(s).

## Review checklist

Before finishing, verify:

- Every macro, structure, and inline function in the x86_64 headers in scope is covered by a requirement or listed as a coverage gap.
- Every declared-but-not-defined symbol has its definition located and cited, or is listed as unresolved.
- Every requirement traces to a specific source location; nothing is stated from a name, comment, or guess.
- Conditional items name their condition (`XEN_PARAVIRT`, `__ASSEMBLY__`, `CONFIG_ARCH_X86_64`).
- Xen behavior, per-CPU semantics, and native backend behavior are stated as assumptions, and native files were not used as evidence for Xen.
- Suspected discrepancies are cited exactly and clearly labeled as unconfirmed.
- Requirement and test IDs are unique and cross-referenced, and are consistent with any PAL requirement that relies on this backend.
- Proposed tests are not presented as executed or safety-approved.

Summarize what each arch specification covers, what remains unverified, and any discrepancies between the documents and the current headers. Do not change implementation code as part of a specification-only task.
