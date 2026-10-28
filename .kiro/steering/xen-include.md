# Xen Platform Headers (`plat/xen/include`)

## Overview

The `plat/xen/include` directory contains the header files used by Unikraft's
Xen platform layer (`plat/xen`). These headers define the interfaces that let a
Unikraft unikernel run as a Xen guest: talking to the hypervisor, managing
memory, handling events, and setting up the platform during boot.

They are platform-internal headers. Application and library code should use the
generic platform API in `include/uk/plat/` instead of including these files
directly, so the same code can build for KVM, Xen, or other platforms.

## Directory Layout

| Path | Purpose |
|------|---------|
| `plat/xen/include/xen-x86/` | x86_64-specific definitions (hypercall wrappers, page tables / memory layout, traps, setup) |
| `plat/xen/include/xen-arm/` | ARM / ARM64-specific equivalents of the x86 headers |
| `plat/xen/include/` (top level and remaining subfolders) | Architecture-independent Xen platform code, including the Xen public interface definitions (hypercalls, event channels, grant tables, shared info) |

Architecture-specific headers are owned by the arch maintainers
(`/plat/*/include/*-x86` and `/plat/*/include/*-arm` in `.github/CODEOWNERS`),
while the rest of `plat/xen` is owned by the Xen platform maintainers.

## Key Responsibilities

- **Hypercalls** – Wrappers for calling into the Xen hypervisor (the guest
  equivalent of system calls).
- **Memory management** – Page frame / machine frame translation and memory
  layout definitions needed for paravirtualized (PV) guests.
- **Event channels** – Xen's virtual interrupt mechanism, used for console,
  timers, and device notifications.
- **Grant tables** – Shared-memory mechanism used by split drivers (e.g.,
  netfront / blkfront in `drivers/xen`).
- **Shared info / start info** – Structures Xen passes to the guest at boot.

## How It Fits in the Build

1. When the Xen platform is selected in `make menuconfig`
   (`Platform Configuration → Xen guest image`), `plat/xen/Makefile.uk` adds
   `plat/xen/include` and the matching architecture folder (`xen-x86` or
   `xen-arm`) to the include path.
2. Platform sources in `plat/xen/` include these headers to implement the
   generic `ukplat_*` API.
3. Xen drivers in `drivers/xen/` build on top of the event channel and grant
   table interfaces defined here.

## Lifecycle Notes

- **Maintenance:** Changes here can break both x86 and ARM Xen builds. Recent
  upstream issues include ARM64 build failures (e.g., missing definitions when
  networking is enabled), so changes should be tested on both architectures.
- **Testing:** Build `app-helloworld` with `plat: xen` for x86_64 and arm64
  after modifying any header.
- **Related docs:** See `xen-pal.md` for the Xen platform abstraction layer.
