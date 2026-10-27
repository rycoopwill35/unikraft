# x86 Xen platform setup and boot initialization — draft specification

> **Example/draft only.** This document describes source-derived requirements and proposed tests for the provided `plat/xen/x86/setup.c` excerpt. It is not an approved safety artifact. **All tests are proposed and have not been run.** Source traces use function names rather than line numbers because the current repository revision has not been independently checked.

## Source scope

- Primary implementation: `plat/xen/x86/setup.c` (source text supplied for this draft).
- Referenced interfaces and implementation dependencies include `HYPERVISOR_update_va_mapping`, `HYPERVISOR_set_segment_base`, `ukplat_memregion_list_insert`, `ukplat_memregion_list_coalesce`, `_init_mem_prepare`, `_init_mem_build_pagetable`, `_init_mem_clear_bootstrap`, `_init_mem_set_readonly`, `_init_mem_demand_area`, `init_events`, `xen_traps_init`, `uk_boot_early_init`, and `uk_boot_entry`.
- Relevant declarations, build configuration, and helper implementations must be inspected before treating the requirements as complete or executing tests.

## Source observations

- `_init_shared_info()` maps the hypervisor-provided shared-info physical address into `_libxenplat_shared_info` using `HYPERVISOR_update_va_mapping`, crashes on a nonzero return, and assigns `HYPERVISOR_shared_info` on success.
- `_init_mem()` prepares page-frame bounds, caps `max_pfn` when it meets or exceeds `MAX_MEM_SIZE / __PAGE_SIZE`, calls page-table/bootstrap/read-only setup helpers, inserts a free heap region, optionally inserts an initrd region, coalesces memory regions, and returns `0` if insertions succeed.
- `_init_segbase()` sets three segment bases to zero in order and crashes on the first nonzero hypercall result.
- `_libxenplat_x86bootinfo_setup()` asserts the boot-info pointer, sets bootloader and protocol strings, initializes memory, and copies command-line pointer and length from Xen start information.
- `_libxenplat_x86entry()` initializes segment bases, obtains boot info, initializes traps, assigns the Xen start-info pointer, maps shared info, initializes events, sets up boot info, performs early boot initialization, and enters the boot entry point.

## Requirements

### REQ-X86-SETUP-001 — shared-info mapping

When `_init_shared_info()` is invoked with `HYPERVISOR_start_info` available, the implementation **shall** request a virtual-address mapping of `_libxenplat_shared_info` using the physical address in `HYPERVISOR_start_info->shared_info`, page-table flags formed by `__pte(pa | 7)`, and `UVMF_INVLPG`.

**Source trace:** `plat/xen/x86/setup.c::_init_shared_info`.

### REQ-X86-SETUP-002 — shared-info mapping failure

If `HYPERVISOR_update_va_mapping()` returns a nonzero result during `_init_shared_info()`, the implementation **shall** invoke `UK_CRASH` with that result rather than proceeding to assign `HYPERVISOR_shared_info` on the normal success path.

**Source trace:** `plat/xen/x86/setup.c::_init_shared_info`.

### REQ-X86-SETUP-003 — shared-info pointer assignment

When the shared-info mapping hypercall succeeds, `_init_shared_info()` **shall** assign `HYPERVISOR_shared_info` to the address of `_libxenplat_shared_info`.

**Source trace:** `plat/xen/x86/setup.c::_init_shared_info`.

### REQ-X86-SETUP-004 — memory upper bound

When `_init_mem()` receives a `max_pfn` from `_init_mem_prepare()` that is greater than or equal to `MAX_MEM_SIZE / __PAGE_SIZE`, it **shall** set `max_pfn` to `MAX_MEM_SIZE / __PAGE_SIZE - 1` before building the page table.

**Source trace:** `plat/xen/x86/setup.c::_init_mem`.

### REQ-X86-SETUP-005 — free heap memory region

After memory setup helper calls, `_init_mem()` **shall** construct a free heap memory-region descriptor with physical base `start_pfn << PAGE_SHIFT`, virtual base derived from `to_virt`, length `(max_pfn - start_pfn) << PAGE_SHIFT`, page count `max_pfn - start_pfn`, type `UKPLAT_MEMRT_FREE`, and read/write flags, and request insertion into `bi->mrds`.

**Source trace:** `plat/xen/x86/setup.c::_init_mem`.

### REQ-X86-SETUP-006 — heap insertion error

If insertion of the free heap descriptor returns a negative result, `_init_mem()` **shall** return that result without proceeding to demand-area initialization or initrd processing.

**Source trace:** `plat/xen/x86/setup.c::_init_mem`.

### REQ-X86-SETUP-007 — optional initrd descriptor

When `HYPERVISOR_start_info->mod_len` is nonzero, `_init_mem()` **shall** construct and request insertion of an initrd memory-region descriptor using the page-aligned module start, module offset, module length, computed page count, `UKPLAT_MEMRT_INITRD`, and read/write flags. It **shall** derive the virtual base through `to_virt` when `SIF_MOD_START_PFN` is set and otherwise use the aligned physical base.

**Source trace:** `plat/xen/x86/setup.c::_init_mem`.

### REQ-X86-SETUP-008 — initrd insertion error

If insertion of an initrd descriptor returns a negative result, `_init_mem()` **shall** return that result before coalescing the memory-region list.

**Source trace:** `plat/xen/x86/setup.c::_init_mem`.

### REQ-X86-SETUP-009 — memory region coalescing

After successful heap insertion and, when applicable, successful initrd insertion, `_init_mem()` **shall** invoke `ukplat_memregion_list_coalesce()` and return `0`.

**Source trace:** `plat/xen/x86/setup.c::_init_mem`.

### REQ-X86-SETUP-010 — segment-base initialization

When `_init_segbase()` executes, the implementation **shall** request zero values for `SEGBASE_FS`, `SEGBASE_GS_USER`, and `SEGBASE_GS_KERNEL` in that order, stopping and invoking `UK_CRASH` if any request returns a nonzero result.

**Source trace:** `plat/xen/x86/setup.c::_init_segbase`.

### REQ-X86-SETUP-011 — boot-info metadata

When `_libxenplat_x86bootinfo_setup()` is invoked with a valid boot-info pointer and `_init_mem()` succeeds, it **shall** set the bootloader field to `Xen`, the bootprotocol field to `PVH`, and the command-line pointer and length from `HYPERVISOR_start_info->cmd_line`.

**Source trace:** `plat/xen/x86/setup.c::_libxenplat_x86bootinfo_setup`.

### REQ-X86-SETUP-012 — boot-info memory failure

If `_init_mem()` returns a negative result during `_libxenplat_x86bootinfo_setup()`, the implementation **shall** invoke `UK_CRASH` rather than continuing on the normal command-line setup path.

**Source trace:** `plat/xen/x86/setup.c::_libxenplat_x86bootinfo_setup`.

### REQ-X86-SETUP-013 — x86 Xen entry initialization order

When `_libxenplat_x86entry()` executes, it **shall** call `_init_segbase()`, obtain boot info and crash if it is unavailable, initialize traps, assign `HYPERVISOR_start_info`, initialize shared info, initialize events, set up boot info, call `uk_boot_early_init()`, optionally call `_check_ospke()` when `CONFIG_HAVE_X86PKU` is enabled, and call `uk_boot_entry()` in that source order.

**Source trace:** `plat/xen/x86/setup.c::_libxenplat_x86entry`.

## Tests

**Status for every test below: PROPOSED / NOT RUN.** These are test designs, not claims of an existing harness or observed results.

### TEST-X86-SETUP-001 — shared-info mapping success

**Traces to:** REQ-X86-SETUP-001, REQ-X86-SETUP-003  
**Method:** Isolated function test with controlled hypercall return and captured arguments, if an appropriate harness can be built.  
**Preconditions:** Valid controlled Xen start-info object; mapping hypercall stub returns `0`; accessible shared-info symbol.

**Steps:**
1. Set `start_info->shared_info` to a known physical address.
2. Invoke `_init_shared_info()` through an appropriate test seam.
3. Capture the mapping call arguments and resulting global pointer.

**Expected result:** The mapping request uses the specified address, PTE expression, and invalidation flag; `HYPERVISOR_shared_info` points to `_libxenplat_shared_info`.

### TEST-X86-SETUP-002 — shared-info mapping failure

**Traces to:** REQ-X86-SETUP-002  
**Method:** Isolated error-path test with a controllable `UK_CRASH` interception mechanism.  
**Preconditions:** Hypercall stub returns a nonzero error; crash behavior can be observed safely.

**Steps:**
1. Configure the mapping stub to return a nonzero result.
2. Invoke `_init_shared_info()`.
3. Observe crash invocation and whether the normal pointer assignment is reached.

**Expected result:** `UK_CRASH` is invoked with the returned error; normal success-path pointer assignment is not reached.

### TEST-X86-SETUP-003 — memory PFN cap

**Traces to:** REQ-X86-SETUP-004  
**Method:** Controlled helper test or source inspection if helper substitution is unavailable.  
**Preconditions:** `_init_mem_prepare()` output can be controlled or its caller's code can be inspected.

**Steps:**
1. Test `max_pfn` below, equal to, and above `MAX_MEM_SIZE / __PAGE_SIZE`.
2. Observe the bound passed into `_init_mem_build_pagetable()`.

**Expected result:** Values at or above the limit are capped to `MAX_MEM_SIZE / __PAGE_SIZE - 1`; a value below the limit is not changed by this cap condition.

### TEST-X86-SETUP-004 — heap descriptor and insertion failure

**Traces to:** REQ-X86-SETUP-005, REQ-X86-SETUP-006  
**Method:** Isolated test with captured memory-region insertion calls.  
**Preconditions:** Memory setup helpers are controlled; valid boot-info region list.

**Steps:**
1. Configure `start_pfn` and `max_pfn` with valid values.
2. Capture the first inserted descriptor and compare its fields with the formulas in the requirement.
3. Repeat with the insertion helper returning a negative value.

**Expected result:** The heap descriptor matches the documented fields; on negative insertion result, `_init_mem()` returns that error without continuing to demand-area setup.

### TEST-X86-SETUP-005 — optional initrd handling

**Traces to:** REQ-X86-SETUP-007, REQ-X86-SETUP-008  
**Method:** Controlled memory-region insertion test.  
**Preconditions:** Memory helper calls can be isolated and the Xen module fields can be set.

**Steps:**
1. Test `mod_len == 0` and confirm no initrd descriptor is inserted.
2. Test nonzero `mod_len` with `SIF_MOD_START_PFN` set and clear.
3. Capture descriptor address, offset, size, count, type, and flags.
4. Simulate a negative initrd insertion return.

**Expected result:** The initrd descriptor is present only for nonzero module length, with the documented branch-dependent virtual base; negative insertion error is returned before coalescing.

### TEST-X86-SETUP-006 — memory-region coalescing

**Traces to:** REQ-X86-SETUP-009  
**Method:** Captured helper-call sequence.  
**Preconditions:** Heap insertion succeeds; optional initrd insertion succeeds if enabled.

**Steps:**
1. Invoke `_init_mem()` with controlled successful helper returns.
2. Record whether coalescing is called and capture the return value.

**Expected result:** The region list is passed to `ukplat_memregion_list_coalesce()` and `_init_mem()` returns `0`.

### TEST-X86-SETUP-007 — segment-base calls and failure paths

**Traces to:** REQ-X86-SETUP-010  
**Method:** Controlled hypercall sequence test with crash interception.  
**Preconditions:** `HYPERVISOR_set_segment_base` can be stubbed and calls captured.

**Steps:**
1. Return success for all calls and capture segment identifiers and values.
2. Repeat with failure on the first, second, and third call separately.

**Expected result:** Successful calls occur in `FS`, `GS_USER`, `GS_KERNEL` order with value zero; each failure invokes `UK_CRASH` and prevents later segment-base calls.

### TEST-X86-SETUP-008 — boot-info fields and failure

**Traces to:** REQ-X86-SETUP-011, REQ-X86-SETUP-012  
**Method:** Isolated boot-info setup test.  
**Preconditions:** Valid boot-info object and controlled `_init_mem()` behavior; known Xen command-line string.

**Steps:**
1. Make `_init_mem()` succeed and invoke `_libxenplat_x86bootinfo_setup()`.
2. Inspect bootloader, bootprotocol, command-line pointer, and length.
3. Repeat with `_init_mem()` returning a negative result and capture crash behavior.

**Expected result:** Success sets `Xen`, `PVH`, and the expected command-line pointer/length; failure invokes `UK_CRASH`.

### TEST-X86-SETUP-009 — platform entry call ordering

**Traces to:** REQ-X86-SETUP-013  
**Method:** Source inspection or instrumented integration test in a Xen guest, depending on available harness.  
**Preconditions:** Build configuration is known; instrumented call traces are available for runtime validation.

**Steps:**
1. Inspect `_libxenplat_x86entry()` for the required sequence.
2. If runtime instrumentation is available, capture calls during Xen guest startup.
3. Repeat or inspect with `CONFIG_HAVE_X86PKU` enabled and disabled.

**Expected result:** The initialization order matches the requirement, with `_check_ospke()` included only in the enabled configuration, followed by `uk_boot_entry()`.

## Assumptions, coverage gaps, and open questions

- This draft is based on the source excerpt supplied in the conversation. Confirm the exact file content and symbols on the team's current `staging` branch before committing.
- The filename's initial comments refer to `common.c` and `mm.c`, although the selected repository path is `setup.c`; the executable code and selected path, not these historical comments, define this draft's scope.
- Helper functions, macro definitions, platform configuration, and crash semantics were not independently inspected. Some proposed isolated tests may require new seams, a specialized harness, or a Xen guest environment.
- The code appears to rely on valid page-frame bounds and valid Xen start information; these preconditions are not validated here. No additional safety guarantees are inferred.
- Requirements describe observed implementation behavior, **not** certification suitability, intended safety behavior, or proof of correct runtime operation.
- No tests have been executed, no test results have been collected, and no complete source or requirement coverage is claimed.

## Review before finishing

Verify each source trace against the team's current `plat/xen/x86/setup.c`; confirm symbol and macro names, unique requirement/test IDs, the dependency assumptions, and the feasibility of each proposed test. Obtain teammate/sponsor review before treating this draft as an accepted artifact.
