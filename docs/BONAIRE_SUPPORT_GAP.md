# Bonaire (GCN 2 / GFX7 / CIK) Support Gap Analysis

Status: planning document; not a claim of Bonaire support.

This driver currently uses Southern Islands/GFX6-era initialization and register headers in the active path (including `gmc_6_0_*` and SI firmware/init routines). Bonaire must remain rejected before register programming until the GFX7/CIK path has been implemented and validated. A PCI-ID addition alone is not support.

## Reference implementation map

Linux source paths below are reference implementations, not code to copy blindly. Check the revision being used and cross-check register semantics against AMD documentation.

| Haiku area / concern | Linux reference | Required work and acceptance evidence |
|---|---|---|
| ASIC classification, startup order, reset, common CIK glue | [amdgpu/cik.c](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/amd/amdgpu/cik.c); [legacy radeon](https://github.com/torvalds/linux/tree/master/drivers/gpu/drm/radeon) | Match Bonaire PCI IDs/revisions and feature flags; confirm unsupported revisions fail closed. Test early failures and complete unwind. |
| Graphics engine / CP / GFX rings / RLC / compute queues | [amdgpu/gfx_v7_0.c](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/amd/amdgpu/gfx_v7_0.c); [GFX7 register definitions](https://github.com/torvalds/linux/tree/master/drivers/gpu/drm/amd/include/asic_reg/gca) | Implement GFX7 register programming, reset and readiness polling, firmware validation/loading, ring setup, fences, interrupts, and timeouts. Do not reuse GFX6 sequences without register-level proof. |
| Memory controller, VRAM, GART, GPU page tables and TLBs | [amdgpu/gmc_v7_0.c](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/amd/amdgpu/gmc_v7_0.c); [AMDGPU VM](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/amd/amdgpu/amdgpu_vm.c); [GMC register definitions](https://github.com/torvalds/linux/tree/master/drivers/gpu/drm/amd/include/asic_reg/gmc) | Port/implement CIK/GMC7-specific setup; validate VRAM size and address placement, GTT aperture, PTE/PDE format and flags, physical address width, cache coherency, TLB invalidation, fault behavior, and map/unmap rollback. |
| SDMA engines | [amdgpu/cik_sdma.c](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/amd/amdgpu/cik_sdma.c) | Implement firmware load, per-engine register offsets, rings, fences, interrupts, recovery, and DMA copy validation. Treat as a separate milestone from graphics CP. |
| Interrupt controller | [amdgpu/cik_ih.c](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/amd/amdgpu/cik_ih.c); GFX7/GMC7 IRQ setup | Validate interrupt-ring allocation, writeback, enable/disable ordering, acknowledgement, spurious interrupts, teardown and partial-init cleanup. |
| Display and AtomBIOS | [amdgpu/dce_v8_0.c](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/amd/amdgpu/dce_v8_0.c); [legacy radeon](https://github.com/torvalds/linux/tree/master/drivers/gpu/drm/radeon) | Verify the exact display IP and board routing; validate AtomBIOS, connectors, mode setting, framebuffer scanout, hotplug and multi-display. Do not infer display support from successful GPU initialization. |
| Video decode/encode (optional scope) | [amdgpu UVD/VCE sources](https://github.com/torvalds/linux/tree/master/drivers/gpu/drm/amd/amdgpu) | UVD/VCE require their own firmware, engine initialization, submission and validation. Do not claim video acceleration if these paths are absent. |
| Power management (optional scope) | [amdgpu CIK sources](https://github.com/torvalds/linux/tree/master/drivers/gpu/drm/amd/amdgpu) | Validate SMC/clock/voltage/thermal paths and suspend/resume. Initially it may be safer to keep dynamic power management disabled if a documented safe fixed state is available. |

## Firmware inventory to verify

Linux's GFX7 code declares Bonaire-specific `bonaire_pfp.bin`, `bonaire_me.bin`, `bonaire_ce.bin`, `bonaire_rlc.bin`, and `bonaire_mec.bin`. Other CIK blocks have separate firmware requirements, including:

- GMC/MC: `bonaire_mc.bin` (see [gmc_v7_0.c](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/amd/amdgpu/gmc_v7_0.c)).
- SDMA: `bonaire_sdma.bin` and `bonaire_sdma1.bin` (see [cik_sdma.c](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/amd/amdgpu/cik_sdma.c)).
- VCE: `bonaire_vce.bin` where VCE support is implemented (see [amdgpu_vce.c](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/amd/amdgpu/amdgpu_vce.c)).
- UVD: `bonaire_uvd.bin` (see [amdgpu_uvd.c](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/amd/amdgpu/amdgpu_uvd.c)); validate the firmware header/version and loader requirements before packaging it.

The firmware directory name differs between Linux driver generations: current amdgpu uses `amdgpu/`; the legacy radeon driver used `radeon/`. Do not rename files or mix generations without confirming expected firmware headers, versions and loader behavior. Obtain and redistribute firmware only under its applicable licensing and distribution terms; do not fabricate or modify firmware blobs.

Firmware presence alone is insufficient. The Haiku loader must validate headers and lengths, handle missing/unsupported versions, and unwind correctly on failure.

## Address-space and memory correctness

The current `AddressSpace::LookupPte()` uses a two-level software structure with `pageDirLen * pageTableLen * B_PAGE_SIZE` nominal coverage (currently documented in code as 8 GiB). This is an implementation limit, not a claim about Bonaire's maximum GPU VA or x86-64 system VA.

Before increasing coverage:
1. Confirm Bonaire's VM page-table depth, PTE/PDE format, address fields and context registers from the applicable AMD/CIK definitions and Linux GMC7 code.
2. Ensure allocated directory/table storage sizes and alignment match hardware requirements, including physical-address masking and table-base register encoding.
3. Confirm the programmed context start/end addresses and page-table depth describe the same geometry as `LookupPte()`.
4. Verify PTE cache flags, read/write permissions, system-memory snooping, fragment mappings, and the required HDP/TLB invalidation sequence.
5. Add tests for first/last page, end overflow, high GPU VA, overlapping maps, invalid GTT PTEs, partial failures, mismatched unmaps, and repeated map/unmap. Verify rollback leaves no stale PTEs.
6. Audit all allocator size-rounding and `offset + size` arithmetic for overflow; check GART page-table allocation before dereferencing it; make partial initialization cleanup safe.

The existing 8 GiB guard is a defensive bound for the current software implementation. It must not be removed until the replacement geometry is implemented and tested.

## Staged acceptance criteria

1. **Fail-closed baseline:** unsupported Bonaire initialization returns an error before any GPU register writes; destructor and failure cleanup are safe.
2. **Memory-controller bring-up:** GMC7/VRAM/GART initialize with validated readback and no VM faults; allocation/map/unmap stress passes.
3. **Minimal GFX7 bring-up:** required firmware is validated and loaded; CP starts, a minimal ring/fence round trip succeeds, and timeouts/reset paths are safe.
4. **Interrupt and SDMA:** each independently enabled feature passes repeated start/stop and error recovery tests.
5. **Display:** at least one known Bonaire board boots to a stable display; modeset, framebuffer scanout and recovery are tested.
6. **Optional features:** UVD, VCE, compute and dynamic power management are individually tested before being advertised.
7. **Release evidence:** Haiku-native build logs, device PCI/revision, firmware versions, register/VM diagnostics, test results, and known limitations are recorded.

## Official AMD references

- [AMD Radeon Sea Islands 3D/Compute Register Reference Guide (70660)](https://docs.amd.com/v/u/en-US/CIK_3D_registers_v2)
- [AMD Southern Islands Acceleration Programming Guide](https://www.amd.com/content/dam/amd/en/documents/radeon-tech-docs/programmer-references/si_programming_guide_v2.pdf)

Use the Sea Islands/CIK material and matching GFX7/GMC7 definitions for Bonaire-specific programming. Southern Islands documentation is background only where the behavior is confirmed to carry over.

## Current verification status

This document is a source-based plan. No Haiku-native build, Bonaire hardware test, or full GFX7 port is asserted here. Keep the Bonaire rejection in place until the relevant stages pass.
