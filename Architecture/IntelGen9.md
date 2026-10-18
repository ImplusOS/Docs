# Intel Gen9.5 Display Driver (`IntelGen9_Display_Driver`)

*Status: compiles and loads; **not verified on real silicon**.*

`Kernel/Source/Drivers/Display/IntelGen9/` is a display driver module for the
integrated graphics in Intel 9th-generation ("Gen9.5") parts — Coffee Lake and
its Whiskey Lake / Comet Lake siblings — and in the mobile/desktop 9th-gen
SKUs that share the same display and blitter engines (UHD 620/630 class). It
does **not** replace `ImplusOS_Generic_Display_Driver` on machines it does not
recognise, and it does not modeset on machines it does.

## 1. What it changes

The generic driver draws every pixel with the CPU and copies rectangles into
the firmware's linear framebuffer. This driver keeps the same interface and
the same double-buffered present protocol, but when a blitter (BCS) engine can
be brought up and proven working, the CPU only *composes* into the driver's
back buffer; the per-frame copy into the scanout framebuffer is done by the
GPU as one `XY_SRC_COPY_BLT` per damaged rectangle.

Nothing else changes: the firmware's display pipeline (pipes, planes, PLLs,
ports) is never touched, the aperture stays exactly where the firmware put it,
and the window manager continues to talk to the same `driver_display_t`
vtable.

```
WM compositor ──> back buffer (driver-owned)
                        │
              present_rects(rects, n)
                        │
        ┌───────────────┴────────────────┐
        │ blitter verified this boot?    │
        └───────────────┬────────────────┘
             yes │                no │
                 v                   v
        BCS XY_SRC_COPY_BLT     CPU memcpy per rect
        (per rect, fenced)      (exactly the generic path)
```

## 2. Why it is written defensively

The development environment for this project boots in QEMU, which has no
Intel GPU. No line of this driver can be exercised by the maintainers' test
loop, so the design assumption is that **any single register offset or command
encoding may be wrong**, and the cost of being wrong must not be a dead
machine or a broken display:

- **Device allowlist.** `probe()` returns false unless the PCI device is
  vendor `0x8086` and one of the ids in `g_ig9_devices[]` (Gen9.5
  desktop/mobile GT2 parts). The class code is logged but not required:
  firmware has been seen to enumerate the iGPU as `0x038000` ("other
  display controller") instead of the usual `0x030000`. On any other
  machine the module never issues an MMIO access and the generic driver
  stays selected.
- **The device search is deferred to `probe()`.** The prepare pass
  (`driver_module_init()` of every module) runs before any module is
  activated — i.e. before the PCI bus driver can answer a device query.
  The search therefore happens the first time `probe()` runs, and the
  module declares `.deps = { "PCI_Driver.ELF" }` so it is activated after
  the bus. `driver_module_init()` itself only logs
  `[IntelGen9] module init`, which keeps a staged-but-unmatched module
  visible on the serial console.
- **Firmware framebuffer only.** The driver requires
  `set_framebuffer()` (called by the display manager with the firmware's
  linear framebuffer) before doing anything; there is no modesetting path and
  no fallback that reprograms a pipe.
- **D0 / memory decode before the first BAR0 read.** A power-management
  capability is located and the function is confirmed in D0, memory decoding
  and bus mastering are enabled — a function parked in D3 can hang the bus on
  real hardware when its BAR is read.
- **Every wait is deadline-bounded.** Forcewake acknowledge, ring drains and
  blit fences all poll with a timeout and *fail closed*: a timeout reverts the
  one write that was issued (forcewake), or disables GPU use for the rest of
  the boot and CPU-copies the same rectangles instead.
- **Staged self-tests, on memory the driver owns.** Acceleration is only
  enabled after:

  | stage | what runs | what it proves |
  |---|---|---|
  | 0 | two `MI_NOOP`s + `MI_USER_INTERRUPT` queued, ring HEAD watched to advance | forcewake, ring registers, doorbell |
  | 1 | `XY_COLOR_BLT` into a scratch page, `MI_STORE_DWORD_IMM` fence | the blitter executes and writes where we asked |
  | 2 | `XY_SRC_COPY_BLT` from a second scratch page | the exact direction `present_rects()` uses |

  Every self-test reads its scratch page back with the CPU. If a stage fails,
  the driver logs over serial and reports itself as CPU-only; the display
  keeps working.

- **The framebuffer's graphics address is discovered, not guessed.** The
  firmware has already mapped the scanout framebuffer in the GGTT and left
  its graphics address in the display plane's surface register
  (`PLANE_SURF`: pipes A/B/C, primary and sprite planes, address in bits
  [31:12]). The driver reads those registers first and accepts a value only
  when the GGTT entry it names is present *and* resolves to a page inside
  the framebuffer's own physical range, so a misread register can never
  become a blit target. If that fails it falls back to scanning existing
  GGTT entries for the framebuffer's physical address (read-only).
  Whichever succeeds also supplies the PTE flags used as the template for
  the entries the driver creates, instead of inventing a PTE bit layout.
  Private entries are taken from the top of the table — placed inside the
  table size the firmware reports in `GMCH_CTRL[7:6]` (PCI config `0x50`,
  Gen8+ layout) — and are checked empty before and after the write.
- **Encodings checked against the reference drivers.** The BCS ring
  registers (`0x22000 + 0x30/0x34/0x38/0x3C`), the 2D blit commands
  (`XY_COLOR_BLT` = six dwords, `XY_SRC_COPY_BLT` = eight dwords, both
  carrying the BR13 dword with depth and pitch) and the GEN4
  `MI_STORE_DWORD_IMM` fence (four dwords, GGTT address) were checked
  against i915 and xf86-video-intel, not against an older guess.
- **Cache coherency.** DMA memory here is cacheable write-back, so ring
  writes are flushed (`clflush` + release fence) before the tail is
  written, the tail write is followed by a posting read, and fence polling
  flushes the CPU's copy before each read.

## 3. Enable / disable

The driver is selected by the normal display-driver selection order: the
manager calls `set_framebuffer()` → `probe()` → `init()` on candidates in
`load_priority` order (this driver: 60, generic: 90), and the first driver
that reports ready wins.

To disable it without rebuilding:

- **At build time**, delete
  `Kernel/Source/Drivers/Display/IntelGen9/Makefile` (the driver build
  discovers modules by `find … -name Makefile`) or remove the `.ELF` from the
  staged image, then rebuild the image.
- **At runtime**, nothing needs to change: on a machine that is not
  allowlisted, or where any bring-up step fails, the driver reports "not
  ready" and the generic driver is used instead.

Because the driver never modesets, there is no "GPU mode / software mode"
switch to flip: if acceleration is unavailable, the same rectangles are
copied by the CPU into the same framebuffer.

## 4. Verification status

| item | status |
|---|---|
| x86_64 + arm64 module builds, no warnings | done |
| module loads and declines on non-Intel / QEMU guests | done (booted live image; logs `module init` + `probe: no supported GPU`) |
| MMIO/ring/blit encodings | checked against i915 + xf86-video-intel; **not run on silicon** |
| blitter self-tests on Gen9.5 hardware | **not run** |
| accelerated present on Gen9.5 hardware | **not run** |

The serial console tells the whole story. On a machine that is not
allowlisted (including QEMU) the module now always shows itself:

```
[IntelGen9] module init
[IntelGen9] probe: no supported GPU
```

(an Intel display device that is not in the allowlist is logged with its
device id and class code immediately before that line). On an allowlisted
part a successful bring-up is expected to read something like:

```
[IntelGen9] module init
[IntelGen9] found UHD 630 (Coffee Lake) id 0x00003e92
[IntelGen9] bringup: BAR0 0xde000000 size 0x01000000
[IntelGen9] bringup: MMIO mapped
[IntelGen9] ggtt entries 0x00100000
[IntelGen9] fb: plane 0x0007019c -> ggtt 0x003d0000
[IntelGen9] ring: ggtt 0x003fa000
[IntelGen9] ring alive
[IntelGen9] blitter verified, accelerated present enabled
```

and any failure logs the stage it failed at (`bringup: BAR0 unusable`,
`bringup: MMIO map failed`, `framebuffer not in GGTT`,
`bringup: no forcewake ack, CPU path only`, `ring setup failed`,
`ring selftest failed`, `present: fence timeout, GPU disabled`) before
falling back; in that case the session behaves exactly like the generic
driver's.
