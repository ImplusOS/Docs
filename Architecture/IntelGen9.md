# Intel Gen9 / Gen9.5 Display Driver (`IntelGen9_Display_Driver`)

*Status: builds (x86_64 + arm64, no warnings); passes the host-side GPU
model (`Tools/IntelGen9Sim`, 12 scenarios); declines cleanly in QEMU.
**Not yet run on real silicon** — see §6 for what to check on first boot.*

`Kernel/Source/Drivers/Display/IntelGen9/` drives the integrated GPU of
Skylake, Kaby Lake, Amber Lake, Coffee Lake, Whiskey Lake, Comet Lake,
Apollo Lake and Gemini Lake parts (the PCI ids in `g_ig9_devices[]`).

## 1. Scope

| Area | What the driver does |
|---|---|
| Mode | Keeps the mode UEFI GOP set. No PLL/DDI/link-training writes (`set_mode` returns false). |
| Display readout | Finds the pipe/plane scanning out the GOP buffer, checks it is linear XRGB8888 with the same stride, reads the DDI transcoder (HDMI/DVI/DP/eDP + port letter), measures refresh from `PIPE_FRMCOUNT`. Reported through `get_monitor_info`/`get_mode_info`. |
| Present | WM composes into a driver-owned back buffer; each damaged rect is copied to scanout by one `XY_SRC_COPY_BLT` on the BCS. CPU copy (WC mapping) when the GPU is not usable. |
| Submission | Execlists (logical ring context + ELSP + CSB), legacy ring buffer as fallback. Blits run from a GGTT ("secure") batch buffer. |
| Recovery | Breadcrumb timeout → RESET_CTL handshake + `GDRST.BLT`, mode restarted, frame copied by CPU; 3 hangs disable the GPU path for the boot. |
| Not done | Modesetting, hotplug, EDID, RCS/3D, media engines, GuC/HuC/DMC, PPGTT for user processes, i915 UAPI, runtime PM. |

## 2. Source layout

| File | Contents |
|---|---|
| `IG9_Regs.h` | Every register offset, bit and command encoding, with its i915 name. Shared with the simulator. |
| `IG9_Uncore.c` | BAR0/BAR2 discovery, D0 + memory decode, MMIO, Gen9 forcewake (GT 0xA188/ack 0x130044, render 0xA278/ack 0x0D84, refcounted, `FORCEWAKE_KERNEL_FALLBACK` kick), deadlines, cache flush, logging. |
| `IG9_Ggtt.c` | GGTT at BAR0 + size/2, size from `GMCH_CTRL.GGMS`, PTE = `phys | 1` (`gen8_ggtt_pte_encode`). Private 64 MiB window at the top of the table with guard pages; detects firmware that fills unused entries with a scratch PTE. |
| `IG9_Engine.c` | BCS: buffers, context image, execlists/legacy enable, request = `MI_BATCH_BUFFER_START_GEN8` (GGTT) + `MI_FLUSH_DW` post-sync seqno, CSB idle wait, reset, self-tests. |
| `IntelGen9.c` | Device table, probe/init, framebuffer location, plane/transcoder readout, present, `driver_module_init`. |

## 3. Bring-up sequence

1. `probe()`: PCI vendor 0x8086 + allowlisted device id. Nothing else touches
   the hardware on a non-matching machine. `is_ready()` stays false until
   `probe()`+`init()` succeed (the display selector adopts any driver that is
   ready right after `set_framebuffer()`).
2. `init()`: map BAR0, read GGMS, take GT forcewake.
3. Locate the framebuffer for the GPU. GOP hands out the **aperture**
   (BAR2/GMADR) address, so GGTT offset = fb − GMADR; every page of the range
   must have a present PTE. A framebuffer given by physical address is looked
   up in the GGTT and must be linearly mapped. Then: matching plane must be
   linear/XRGB8888/same stride, and no fence register may cover the range.
4. Engine: allocate ring (32 KiB), batch (16 KiB), HWSP, context (5 pages),
   scratch PD/PT/page, fence page, test pages; reset the BCS; enable
   execlists; run the self-tests (`MI_STORE_DWORD_IMM`, `XY_COLOR_BLT` with
   out-of-rect pixels checked, `XY_SRC_COPY_BLT` twice). On failure: reset,
   legacy ring, same tests. On failure: reset, CPU path.
5. Map the back buffer into the GGTT; present goes through the blitter.

## 4. Fixes relative to the previous version

The earlier module (BCS on the legacy ring only) could not have worked on
silicon:

- `XY_SRC_COPY_BLT`/`XY_COLOR_BLT` were emitted in the pre-Gen8 8/6-dword
  form; Gen8+ uses 10/7 dwords (64-bit addresses).
- The GOP framebuffer address is an aperture address; the driver searched the
  GGTT for it as a physical address, then fell back to creating a GGTT
  mapping that pointed the GPU at its own BAR.
- `GDRST` bit 1 resets the **render** engine; BCS is bit 3, and Gen8+ needs
  the `RESET_CTL` ready-to-reset handshake first.
- It wrote 0x9A78 (not a forcewake register) and 0x0D88 (the media
  forcewake *ack*) as forcewake requests.
- Gen9 is an execlists part (i915 forces execlists on Gen9+); legacy ring
  submission is kept only as a fallback.
- A “ring drained, so assume the fence landed” shortcut could report success
  for work that never executed.

## 5. Host simulator

```
make -C Tools/IntelGen9Sim test
```

compiles the unmodified driver sources with `-DIG9_HOST_SIM` (MMIO goes to
hooks) against a model of the GPU: PCI/BARs, GGTT + aperture, forcewake,
reset, execlists (context image parsed as LRI, CSB), legacy ring, and a
strict decoder for every command the driver emits (wrong length, PPGTT
batch, unmapped page, ring-straddling command → engine hang). Scenarios:
execlists bring-up + present + monitor info, full-screen present, 500 rects
and ring wrap, legacy fallback, dead engine, hang → reset → recover,
repeated hangs → disabled, scratch-filled GGTT, X-tiled plane, forcewake
fallback kick, framebuffer given as stolen physical address, non-Intel
machine (zero MMIO accesses). The model verifies logic and encodings
against i915-derived expectations; it is not silicon.

## 6. First boot on hardware

Serial (COM1) shows the whole bring-up. A successful run looks like:

```
[IntelGen9] module init
[IntelGen9] found UHD Graphics 630 (Coffee Lake) id 0x00003E92 rev 0x...
[IntelGen9] BAR0 0x...  size 0x01000000, aperture 0x... size 0x10000000
[IntelGen9] ggtt: 1048576 entries (4096 MiB), private window at 0xFC000000
[IntelGen9] scanout: GOP framebuffer via aperture, ggtt 0x...
[IntelGen9] scanout: pipe 0 plane 1 ctl 0x84000000 stride 7680 pipe-src 1920x1080
[IntelGen9] output HDMI-B refresh 60000 mHz
[IntelGen9] engine: ring 0x... batch 0x... ctx 0x... fence 0x...
[IntelGen9] engine: execlists submission verified (store, fill, copy)
[IntelGen9] blitter present enabled (execlists)
```

Every failure path logs its reason (and a `BCS ...:` register dump with
HEAD/TAIL/ACTHD/IPEHR/INSTDONE/RING_MODE/EXECLIST_STATUS/CSB pointer/fence)
and leaves the display on the CPU path, so a bad first boot is still a
working desktop. Things worth reporting from a first boot: the full
`[IntelGen9]` block, whether `execlists` or `legacy ring` was chosen, and
any `no ACTIVE_IDLE event` line.

To disable the module, remove its `.ELF` from the image (or delete
`Kernel/Source/Drivers/Display/IntelGen9/Makefile`); the generic driver is
then used as on any other machine.
