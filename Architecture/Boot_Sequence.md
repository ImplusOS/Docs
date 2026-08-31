# Boot Sequence — ImplusOS

*Last reviewed: 2026-08-29 (reflects P3/P4/P5 of `Docs/Others/TODO_OS_Refactor.md`;
the 19-phase list re-verified against `Kernel/Core/kernel_main.c`).*

## 1. From firmware to `kernel_main`

```
UEFI firmware
   → Build/Loader/BOOTX64.EFI          (UEFI loader shim, BootLoader/)
   → Build/BootManager/BOOTMANAGER.EFI (BootManager/, loads Kernel_Main.ELF + drivers)
   → Kernel_Main.ELF entry (kernel_main, e = kernel_main per the linker script's -e flag)
       → kernel_main_after_stack_switch()   (Kernel/Core/kernel_main.c)
```

`kernel_main()` itself does the minimum needed to get onto a known-good
kernel stack (see `hal_arch_switch_stack_and_jump()`), then everything else
happens in `kernel_main_after_stack_switch()`, which is instrumented
phase-by-phase with `boot_profile_begin()`/`boot_profile_end(name, ...)`.

## 2. The 19 boot phases

In order, as recorded in `g_boot_profile[]` (`KERNEL_BOOT_PROFILE_MAX = 64`
slots, only 19 used today):

| # | Phase name | What happens | Key call(s) |
|---|---|---|---|
| 0 | `cpu_tables` | GDT/IDT (x86_64) or exception vectors (arm64) | `arch_ops_get()->init_cpu_tables()` |
| 1 | `pmm` | Physical memory bitmap allocator from the UEFI memory map | `init_physical_memory()` |
| 2 | `paging` | Kernel page tables, remap the boot framebuffer | `init_paging()` |
| 3 | `heap` | Kernel heap (`malloc`/`free` backing) | `memory_init()` |
| 4 | `acpi_interrupts` | RSDP/MADT parsing, IOAPIC/LAPIC or GIC bring-up | `acpi_init()`, `platform_interrupts_configure()` |
| 5 | `timer` | System timer HAL selection + start | `timer_init(ops->get_timer_hal())` |
| 6 | `syscall` | SYSCALL/SYSRET MSRs, per-CPU syscall stacks, `Kernel/Compat` registry | `syscall_init()` |
| 7 | `smp` | AP (application processor) bring-up | `smp_init()` |
| 8 | `driver_module_load` | **DeviceRegistry/BusRegistry come into existence here** — scans the ESP's `Kernel/Driver/` for `.ELF` modules and loads them | `driver_module_manager_init()`, then `platform_builtin_drivers_register()` |
| 9 | `driver_module_critical` | Calls each loaded module's critical-path init under IRQs disabled | `driver_module_init_critical()` |
| 10 | `input_init` | PS/2 and USB HID input drivers | `input_manager_init()` |
| 11 | `disk_io_init` | Block device probing (AHCI/NVMe/VirtIO-Blk/USB Mass Storage) | `disk_io_init()` |
| 12 | `fs_init` | VFS mount table: FAT32, exFAT, ISO9660, DevFS, TmpFS, ProcFS, EtcFS | `all_fs_initialize()` |
| 13 | `display_init` | Framebuffer/GPU driver selection | `driver_manager_display_init()` |
| 14 | `process_manager` | Process table, scheduler | (process manager init) |
| 15 | `kernel_services` | Remaining kernel-internal services | |
| 16 | `userland_elf` | Loads and starts `Userland.ELF` (the init process — spawns WindowManager, Shell, apps) | ELF loader + `process_manager_create()` |
| 17 | `driver_module_deferred` | Non-critical-path module init deferred from phase 9 | `driver_module_init_deferred()` |
| 18 | `audio_network_init` | Audio subsystem + network stack bring-up, plus `Kernel/Drivers/Module/NetworkBuiltinDrivers.c` registration | `audio_manager_init()`, `network_stack_init()`, `network_builtin_drivers_register()` |

Phase 8 is the single most important dependency boundary in this list: it is
the earliest point at which `DeviceRegistry` exists, which is why ACPI/Timer
(already initialized in phases 4–5) only get their `DEVICE_TYPE_PLATFORM`
DeviceRegistry entries *after* it, in phase 8 — not because their own
initialization is delayed, but because there is nothing to register them
into any earlier. See `Docs/Architecture/Driver_Module_Guide.md` and
`Docs/Others/TODO_OS_Refactor.md` phase P5 (9.1) for the full "built-in
driver" rationale.

## 3. Reading a real `boot_profile_dump()`

`kernel_main.c` calls `boot_profile_dump("boot")` once boot completes,
writing one line per phase to the serial console (COM1, 115200 baud). A real
capture (QEMU, OVMF, q35, from this refactor's own regression testing) looks
like:

```
[boot:profile] reason=boot count=19
[boot:profile] 00 name=cpu_tables start_us=0 duration_us=0
[boot:profile] 01 name=pmm start_us=0 duration_us=0
[boot:profile] 02 name=paging start_us=0 duration_us=0
[boot:profile] 03 name=heap start_us=0 duration_us=0
[boot:profile] 04 name=acpi_interrupts start_us=0 duration_us=0
[boot:profile] 05 name=timer start_us=0 duration_us=470
[boot:profile] 06 name=syscall start_us=571 duration_us=456
[boot:profile] 07 name=smp start_us=1035 duration_us=51314
[boot:profile] 08 name=driver_module_load start_us=106232 duration_us=1065
[boot:profile] 09 name=driver_module_critical start_us=107867 duration_us=13239
[boot:profile] 10 name=input_init start_us=121171 duration_us=1178725
[boot:profile] 11 name=disk_io_init start_us=1300029 duration_us=3818831
[boot:profile] 12 name=fs_init start_us=5118872 duration_us=3662
[boot:profile] 13 name=display_init start_us=5124221 duration_us=19582
[boot:profile] 14 name=process_manager start_us=5143812 duration_us=22514
[boot:profile] 15 name=kernel_services start_us=5166334 duration_us=627
[boot:profile] 16 name=userland_elf start_us=5167112 duration_us=12918
[boot:profile] 17 name=driver_module_deferred start_us=5180042 duration_us=244
[boot:profile] 18 name=audio_network_init start_us=5180292 duration_us=7614
```

Reading it: `start_us` is time-since-boot when the phase began; `duration_us`
is that phase's own wall-clock cost. `smp` (phase 7, ~50ms here) and
`input_init`/`disk_io_init` (phases 10–11, over a second combined in this
particular QEMU capture) dominate boot time — the former is AP bring-up
handshaking, the latter is driver hardware-probe timeouts (expected to vary
a lot machine-to-machine; a real NVMe drive probes much faster than QEMU's
emulated AHCI in this trace). Phases with `duration_us=0` either genuinely
complete in under a microsecond or aren't wrapped tightly enough around the
actual work to show timing yet — both are fine to leave as-is unless
profiling that specific phase becomes a real need.

`kernel_boot_profile_count()`/`kernel_boot_profile_get(index, out)`
(`Kernel/Core/kernel_main.c`) expose this same table to callers other than
the serial dump — `OSDebug` reads it for its boot-timing view.

## 4. What phase P4 changed here (and what it deliberately didn't)

Phases 0 and 5 used to pick between two arch-specific implementations via
`#ifdef PLATFORM_X86_64`/`PLATFORM_ARM64` directly in
`kernel_main_after_stack_switch()`. Both now go through `arch_ops_t`
uniformly (`init_cpu_tables()`, `get_timer_hal()`) — see
`Docs/Others/TODO_OS_Refactor.md` phase P4 (8.1) for the specific before/after
diffs. **The boot phase list, its order, and each phase's timing
characteristics are unchanged** by that refactor — only how the kernel
*chooses* which arch-specific function to call changed, never *when* it's
called or what it does once chosen.
