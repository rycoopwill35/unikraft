# Xen power-management shutdown operations: example specification

> **Example only.** This document demonstrates the intended structure for
> source-derived requirements and tests in the core-runtime group. It is not
> an approved safety artifact.

## Source scope

- `plat/xen/shutdown.c`: built unconditionally for the Xen platform
  (`plat/xen/Makefile.uk`, `LIBXENPLAT_SRCS-y`).
- Dependencies used to interpret behavior:
  - `HYPERVISOR_sched_op`: `plat/xen/include/xen-x86/hypercall64.h` (x86_64,
    `static inline`) or `plat/xen/include/xen-arm/hypercall.h` with
    `plat/xen/arm/hypercalls64.S` (arm64).
  - `xencons_flush`: `drivers/xen/xencons/console.c`, only under `CONFIG_LIBXENCONS`.
  - `uk_pm_ops_register` and `struct uk_pm_ops`: `lib/ukpm/pm.c`,
    `lib/ukpm/include/uk/pm.h`.
  - `UK_BOOT_EARLYTAB_ENTRY`: `lib/ukboot/include/uk/boot/earlytab.h`.

## Source observations

- `xen_shutdown(reason)` fills a `struct sched_shutdown` with `reason`,
  flushes the Xen console when `CONFIG_LIBXENCONS` is enabled, issues
  `HYPERVISOR_sched_op(SCHEDOP_shutdown, &sched_shutdown)`, ignores the
  hypercall's return value, and returns `-EIO`.
- `xen_halt`, `xen_restart`, and `xen_crash` each call `xen_shutdown` with
  `SHUTDOWN_poweroff`, `SHUTDOWN_reboot`, and `SHUTDOWN_crash` respectively
  and return its result.
- `xen_pm_ops` sets `syshalt`, `sysrestart`, and `syscrash`; it does not set
  `syssuspend`.
- `xen_register_pm_ops` registers `xen_pm_ops` and returns the result of
  `uk_pm_ops_register`. It is installed as an early boot table entry with
  priority `UK_PRIO_EARLIEST`.

## Requirements

### REQ-CORE-SHUTDOWN-001: shutdown request

When `xen_shutdown` is called with a reason code, the implementation shall
issue `HYPERVISOR_sched_op` with command `SCHEDOP_shutdown` and a
`struct sched_shutdown` whose `reason` field equals that code.

**Source trace:** `plat/xen/shutdown.c:xen_shutdown` (lines 54–65; hypercall at line 62).

### REQ-CORE-SHUTDOWN-002: console flush before shutdown

When `CONFIG_LIBXENCONS` is enabled and `xen_shutdown` is called, the
implementation shall call `xencons_flush` before issuing the shutdown
hypercall.

**Source trace:** `plat/xen/shutdown.c:xen_shutdown` (lines 58–62).

### REQ-CORE-SHUTDOWN-003: result when the hypercall returns

When the shutdown hypercall returns to `xen_shutdown`, the implementation
shall return `-EIO`, regardless of the hypercall's return value.

**Source trace:** `plat/xen/shutdown.c:xen_shutdown` (lines 62–64).

### REQ-CORE-SHUTDOWN-004: reason code per operation

When `xen_halt`, `xen_restart`, or `xen_crash` is called, the implementation
shall call `xen_shutdown` with `SHUTDOWN_poweroff`, `SHUTDOWN_reboot`, or
`SHUTDOWN_crash` respectively, and return its result.

**Source trace:** `plat/xen/shutdown.c:xen_halt` (lines 67–70), `xen_restart` (lines 72–75), `xen_crash` (lines 77–80).

### REQ-CORE-SHUTDOWN-005: operations table

The implementation shall provide a `struct uk_pm_ops` whose `syshalt`,
`sysrestart`, and `syscrash` members point to `xen_halt`, `xen_restart`, and
`xen_crash`, and whose `syssuspend` member is not set.

**Source trace:** `plat/xen/shutdown.c:xen_pm_ops` (lines 82–86); `lib/ukpm/include/uk/pm.h:struct uk_pm_ops`.

### REQ-CORE-SHUTDOWN-006: registration at early boot

During early boot, the implementation shall register `xen_pm_ops` through
`uk_pm_ops_register` from an early boot table entry with priority
`UK_PRIO_EARLIEST`, and shall return the registration result unchanged.

**Source trace:** `plat/xen/shutdown.c:xen_register_pm_ops` (lines 88–91), `UK_BOOT_EARLYTAB_ENTRY` (line 93).

## Tests

### TEST-CORE-SHUTDOWN-001: shutdown hypercall arguments and result

**Traces to:** REQ-CORE-SHUTDOWN-001, REQ-CORE-SHUTDOWN-003

**Method:** isolated unit test with mocks (Ceedling style)

**Status:** Proposed, not run

**Preconditions:** A proposed harness that compiles `shutdown.c` with a test
include path providing a mock `HYPERVISOR_sched_op` in place of the real
`hypercall.h` (the x86_64 wrapper is `static inline`, so it cannot be mocked
at link time). The mock records `cmd` and a copy of `*arg`, and returns a
value set by the test. `CONFIG_LIBXENCONS` is disabled.

**Steps:**

1. Set the mock to return `0`. Call `xen_shutdown(0x5A)` (reach the `static`
   function by including `shutdown.c` in the test unit).
2. Read the recorded `cmd` and `reason`, and the return value.
3. Repeat with the mock returning `-1`.

**Expected result:** The mock was called once per call with `cmd ==
SCHEDOP_shutdown` and `reason == 0x5A`. Both calls return `-EIO`.

### TEST-CORE-SHUTDOWN-002: console flush ordering

**Traces to:** REQ-CORE-SHUTDOWN-002

**Method:** isolated unit test with an ordered mock trace

**Status:** Proposed, not run

**Preconditions:** Harness from TEST-CORE-SHUTDOWN-001, built with
`CONFIG_LIBXENCONS` enabled and a mock `xencons_flush` that appends to the
same call log as the hypercall mock.

**Steps:**

1. Call `xen_shutdown(SHUTDOWN_poweroff)`.
2. Read the call log.

**Expected result:** The log is exactly `xencons_flush`, then `HYPERVISOR_sched_op`.

### TEST-CORE-SHUTDOWN-003: reason code per operation

**Traces to:** REQ-CORE-SHUTDOWN-004, REQ-CORE-SHUTDOWN-005

**Method:** isolated unit test with mocks

**Status:** Proposed, not run

**Preconditions:** Harness from TEST-CORE-SHUTDOWN-001.

**Steps:**

1. Call `xen_pm_ops.syshalt()`, `xen_pm_ops.sysrestart()`, and
   `xen_pm_ops.syscrash()` in turn.
2. After each call, read the recorded `reason` and the return value.
3. Check `xen_pm_ops.syssuspend`.

**Expected result:** Recorded reasons are `SHUTDOWN_poweroff`,
`SHUTDOWN_reboot`, and `SHUTDOWN_crash` in that order; each call returns
`-EIO`; `syssuspend` is `NULL`.

### TEST-CORE-SHUTDOWN-004: early boot registration

**Traces to:** REQ-CORE-SHUTDOWN-006

**Method:** isolated unit test with mocks, plus configured build and image inspection

**Status:** Proposed, not run

**Preconditions:** Harness from TEST-CORE-SHUTDOWN-001 with a mock
`uk_pm_ops_register` that records its argument and returns a value set by the
test. For the build step: an x86_64 Xen configuration and `objdump`.

**Steps:**

1. Set the mock to return `-EEXIST` and call `xen_register_pm_ops(NULL)`.
2. Build the Xen platform library and inspect the object file for
   `shutdown.c` with `objdump -h` and `objdump -r`. The macro places each entry
   in a section named `.uk_boot_earlytab` followed by the expanded priority
   (`lib/ukboot/include/uk/boot/earlytab.h`).

**Expected result:** The mock received `&xen_pm_ops` and the call returned
`-EEXIST`. The object file has a `.uk_boot_earlytab<N>` section, where `<N>`
is the expansion of `UK_PRIO_EARLIEST`, with a relocation that points to
`xen_register_pm_ops`.

## Assumptions

- Xen acts on `SCHEDOP_shutdown` and does not normally return to the guest.
  The code only handles the case where it does return.
- How the early boot table runs its entries, and what it does with a nonzero
  result, is defined in `lib/ukboot` and is not established here.

## Open questions

- `xen_shutdown` does not inspect the hypercall's return value; every return
  is reported as `-EIO`. Is that sufficient for the caller in `lib/ukpm`?
- `syssuspend` is not provided. Is suspend out of scope for the Xen platform?

## Coverage gaps

- Behavior on a running Xen guest (that the domain actually stops) needs a
  Xen guest integration test, not proposed here.
