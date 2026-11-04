---
inclusion: fileMatch
fileMatchPattern: ["plat/xen/arch/x86_64/**/*", "Test_Specifications/xen/interface/x86_64/**/*.md"]
---

# File add-on: x86_64 Xen arch headers (`except.h`, `auxsp.h`, `sysctx.h`)

## How this file fits

This is a layer 3 add-on. Follow `.kiro/steering/xen-test-specifications.md`
and `.kiro/steering/xen-group-interface-headers.md` first; they carry the
general rules for reading, tracing, and testing interface headers. This file
only lists facts and traps specific to the three x86_64 headers in
`plat/xen/arch/x86_64/include/uk/plat/xen/arch/`, found while writing the
original steering for this folder.

The arm64 headers are out of scope unless the user asks. Do not transfer arm64
behavior into an x86_64 requirement or the reverse.

Write one specification per header:
`Test_Specifications/xen/interface/x86_64/<header>.md` (`except.md`,
`auxsp.md`, `sysctx.md`), with IDs `REQ-ARCH-X86-<TOPIC>-NNN` and
`TEST-ARCH-X86-<TOPIC>-NNN`, where `<TOPIC>` is `EXCEPT`, `AUXSP`, or `SYSCTX`.
If the PAL layer for the same topic has a specification, keep the two
consistent: when a PAL requirement assumes a backend behavior, the matching
requirement here should state it, or the gap should be listed.

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


## Test notes specific to these headers

- For the accessors and IRQ control functions, propose an isolated unit test
  with a stub `HYPERVISOR_shared_info` whose `vcpu_info[0]` the test controls,
  and a stub `uk_plat_xen_irqs_handle_pending` that counts calls. Cover pending
  and not-pending cases, and zero and nonzero flags for `uk_plat_xen_restore_irqf`.
- Ordering around `__barrier()` can be shown only by inspection or disassembly,
  not by a stubbed unit test.
- Check the event-name agreement with a configured build plus `nm` or
  `objdump -h` on the linked image before saying anything about runtime effect.
- Typical obligations: "`UK_PLAT_XEN_EXCEPT_EVENT_DEBUG` shall expand to
  `xen_except_event_debug`"; "when `uk_plat_xen_save_irqf` is called, it shall
  return the previous `evtchn_upcall_mask` value and leave the mask set to `1`".
  Verify each against the current source before using it.
