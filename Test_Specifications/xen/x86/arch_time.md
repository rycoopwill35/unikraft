# x86 Xen time and timer interface — example specification

> **Example only.** This document demonstrates the intended structure for
> source-derived requirements and tests. It is not an approved safety artifact.

## Source scope

- `plat/xen/x86/arch_time.c`
- `plat/xen/events.c` for virtual-interrupt binding support.

## Source observations

- `ukplat_time_init()` binds the Xen timer virtual interrupt and unmasks its
  event channel.
- `timer_handler()` schedules the next timer operation one platform tick after
  the current monotonic time.
- `ukplat_time_fini()` clears a pending timer operation and unbinds the timer
  event channel.

## Requirements

### REQ-X86-TIME-001 — timer virtual-interrupt initialization

When the x86 Xen platform time interface is initialized, the implementation
shall bind `VIRQ_TIMER` to its timer handler and unmask the resulting event
channel.

**Source trace:** `plat/xen/x86/arch_time.c:ukplat_time_init` (lines 217–222).

### REQ-X86-TIME-002 — periodic timer rescheduling

When the timer event handler runs, the implementation shall request the next
Xen timer expiry at the current monotonic time plus `UKPLAT_TIME_TICK_NSEC`.

**Source trace:** `plat/xen/x86/arch_time.c:timer_handler` (lines 206–212).

### REQ-X86-TIME-003 — timer interface finalization

When the x86 Xen platform time interface is finalized, the implementation
shall clear any pending Xen timer operation before unbinding the timer event
channel.

**Source trace:** `plat/xen/x86/arch_time.c:ukplat_time_fini` (lines 224–229).

## Tests

### TEST-X86-TIME-001 — timer-interrupt initialization trace

**Traces to:** REQ-X86-TIME-001

**Method:** instrumented integration test or hypercall/event-channel trace

**Preconditions:** x86 Unikraft image is running as a Xen guest and the timer virtual interrupt is available.
**Steps:**

1. Start the platform time interface.
2. Capture the virtual-interrupt bind operation and the event-channel mask state.

**Expected result:** `VIRQ_TIMER` is bound to the timer handler and the returned event channel is unmasked.

### TEST-X86-TIME-002 — timer rescheduling trace

**Traces to:** REQ-X86-TIME-002

**Method:** instrumented integration test or hypercall trace

**Preconditions:** The timer handler can be invoked and monotonic time can be observed.
**Steps:**

1. Record the monotonic time immediately before the timer handler executes.
2. Capture the timer-operation value requested by the handler.

**Expected result:** The requested expiry equals the handler's monotonic-time value plus `UKPLAT_TIME_TICK_NSEC`, subject to the measurement method's documented sampling tolerance.

### TEST-X86-TIME-003 — timer-finalization trace

**Traces to:** REQ-X86-TIME-003

**Method:** instrumented integration test or hypercall/event-channel trace

**Preconditions:** The platform time interface has been initialized.
**Steps:**

1. Finalize the platform time interface.
2. Capture the timer-operation and event-channel operations in order.

**Expected result:** The implementation requests timer value `0` before it unbinds the timer event channel.
