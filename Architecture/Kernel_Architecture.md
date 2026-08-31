# Kernel Architecture — ImplusOS

*Last reviewed: 2026-08-29.*

## 1. Overview

ImplusOS runs a **monolithic kernel** with loadable driver modules on **x86-64 (Long
Mode)** and **arm64 (AArch64)** hardware. The kernel is a single ELF64 binary
(`Kernel_Main.ELF`, linked `-pie`, entry `kernel_main`) that is loaded by the boot
manager (reached via a UEFI loader shim on both architectures, or the legacy-BIOS
path on x86-64) into physical memory and entered with paging already enabled.

```
┌─────────────────────────────────────────────────────────────────┐
│                         Userland (Ring 3)                       │
│  Init (Userland.ELF) → WindowManager, Shell, Apps, ...         │
├───────────────────────── SYSCALL/SYSRET ────────────────────────┤
│                         Kernel (Ring 0)                        │
│                                                                 │
│  ┌───────────┐ ┌─────────┐ ┌─────┐ ┌───────┐ ┌──────────────┐ │
│  │ Process   │ │  VFS /  │ │ IPC │ │ WM    │ │  Network     │ │
│  │ Manager   │ │ FAT32   │ │     │ │Kernel │ │  Stack       │ │
│  └───────────┘ └─────────┘ └─────┘ └───────┘ └──────────────┘ │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │              Driver Module Manager                          ││
│  │  PCI │ FAT32 │ PS/2 │ USB │ VirtIO │ Display │ NIC         ││
│  └─────────────────────────────────────────────────────────────┘│
│                                                                 │
│  ┌───────────┐ ┌─────────┐ ┌─────┐ ┌───────┐ ┌──────────────┐ │
│  │ Memory /  │ │ GDT /   │ │ SMP │ │ ACPI  │ │ Timer /      │ │
│  │ Paging    │ │ IDT     │ │     │ │ APIC  │ │ RTC          │ │
│  └───────────┘ └─────────┘ └─────┘ └───────┘ └──────────────┘ │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│         Boot Manager (BootManager/)  ← UEFI shim / BIOS        │
│  framebuffer → logo → ELF load (kernel+drivers+userland)      │
│  → ACPI/BPB discovery → KASLR → hand off BOOT_INFO            │
└─────────────────────────────────────────────────────────────────┘
```

## 2. Boot Sequence

### 2.1 Loader and boot manager

The first-stage loader is a small UEFI application built with EDK2
(`BootLoader/x86_64/UEFI/Loader.c`, `BootLoader/arm64/UEFI/Loader.c`) — or, on
x86-64 only, a legacy-BIOS stage (`BootLoader/x86_64/BIOS/BiosLoader.c`). It
hands control to the second-stage **boot manager** (`BootManager/`), which:

1. Sets up the framebuffer (UEFI GOP / BIOS VESA) and renders the boot logo + font.
2. Discovers SMBIOS and the ACPI RSDP (v1/v2).
3. Loads `Kernel/Kernel_Main.ELF`, the driver `*.ELF` modules, `Userland/Userland.ELF`
   and the userland ELFs from the boot filesystem. `BootManager/Core/ElfLoader.c`
   parses ELF64 headers and PT_LOAD segments and supports ET_DYN (PIE) with
   arch-specific relocations (`R_X86_64_RELATIVE` / `R_AARCH64_RELATIVE`).
4. Applies KASLR (`BootManager/Core/KASLR_RNG.c`).
5. Captures the boot partition's BPB / start LBA and the boot-drive type.
6. On UEFI, calls `ExitBootServices()`. On arm64, drops EL2→EL1 via
   `BootManager/UEFI/AArch64Trampoline.c`.
7. Jumps to `kernel_main()` with a `BOOT_INFO` structure.

### 2.2 Kernel initialization (`Kernel/Core/kernel_main.c`)

`kernel_main()` does the minimum to get onto a known-good kernel stack (early
`arch_ops_t` init, `serial_init()`, `kernel_panic_init()`), then
`kernel_main_after_stack_switch()` runs **19 instrumented boot phases**, each
wrapped in `boot_profile_begin()` / `boot_profile_end(name)` and dumped over
serial by `boot_profile_dump("boot")`.

| # | Phase | What happens |
|---|---|---|
| 0 | `cpu_tables` | GDT/IDT (x86_64) or exception vectors (arm64), via `arch_ops_get()->init_cpu_tables()` |
| 1 | `pmm` | Bitmap physical page allocator from the UEFI/BIOS memory map |
| 2 | `paging` | Kernel page tables; remap the boot framebuffer |
| 3 | `heap` | Kernel heap (`malloc`/`free` backing) |
| 4 | `acpi_interrupts` | RSDP/MADT parse; IOAPIC/LAPIC or GIC bring-up |
| 5 | `timer` | System timer HAL (`arch_ops_t.get_timer_hal()`) selection + start |
| 6 | `syscall` | `SYSCALL`/`SYSRET` MSRs (or `SVC` vectors), per-CPU syscall stacks, `compat_registry` |
| 7 | `smp` | Application-processor bring-up (AP trampoline / PSCI) |
| 8 | `driver_module_load` | Scan the boot FS `Kernel/Driver/` for `.ELF` modules and load them; `DeviceRegistry`/`BusRegistry` come into existence here |
| 9 | `driver_module_critical` | Each module's critical-path init, IRQs disabled |
| 10 | `input_init` | PS/2 and USB HID input |
| 11 | `disk_io_init` | Block device probing (AHCI/NVMe/VirtIO-Blk/USB Mass Storage) |
| 12 | `fs_init` | VFS mount table: FAT32, exFAT, ISO9660, DevFS, TmpFS, ProcFS, EtcFS |
| 13 | `display_init` | Framebuffer/GPU driver selection |
| 14 | `process_manager` | Process table, scheduler |
| 15 | `kernel_services` | `ipc_init()`, `syscall_file_init()`, remaining kernel services |
| 16 | `userland_elf` | Load and start `Userland.ELF` (the init process) |
| 17 | `driver_module_deferred` | Non-critical module init deferred from phase 9 |
| 18 | `audio_network_init` | `audio_manager_init()`, `network_stack_init()`, `network_builtin_drivers_register()` |

See `Docs/Architecture/Boot_Sequence.md` for a worked `boot_profile_dump()`
capture and phase-timing discussion. `kernel_main.c` no longer contains
`#ifdef PLATFORM_*` branches for phase selection — arch differences go through
`arch_ops_t` (`Kernel/include/interfaces/arch_ops.h`).

## 3. Memory Management

### 3.1 Physical Memory Manager (PMM)

- **Algorithm**: Bitmap-based page allocator
- **Page size**: 4 KiB (4096 bytes)
- **Initialization**: Parses UEFI memory map, marks available pages
- **API**: `alloc_page()`, `free_page()`, `alloc_contiguous_pages()`, `free_contiguous_pages()`
- **Statistics**: `get_free_memory()`, `get_used_memory()`, `get_total_memory_pages()`

### 3.2 Virtual Memory (4-Level Paging)

Both architectures use 4-level page tables:

| Architecture | Levels | Base register |
|---|---|---|
| **x86_64** | PML4 → PDPT → PD → PT | CR3 |
| **arm64** | TTBR0/1 → L0 → L1 → L2 → L3 | Translation Table Base Register |

- **Kernel mapping**: Identity-mapped (virtual == physical) on x86_64; kernel-space high mapping on arm64
- **User mapping**: Per-process address space (x86_64: per-process CR3; arm64: TTBR0_EL1)
- **Page flags**: PRESENT, RW, USER, NX (No-Execute), accessed/dirty bits
- **API**: `paging_create_process_space()`, `paging_destroy_process_space()`, `paging_map_user_page()`, `paging_set_user_access()`, `paging_virt_to_phys()`

### 3.3 Kernel Heap

- Standard allocator interface: `malloc()`, `free()`, `calloc()`, `realloc()`
- Sensitive memory variants: `malloc_sensitive()`, `free_sensitive()` (zeroed on free)
- DMA allocator: `dma_alloc()`, `dma_free()` (physically contiguous, low memory)

### 3.4 User-Space Memory Map (x86_64)

| Region | Start Address | End Address | Size |
|---|---|---|---|
| Code | `0x4000000000` | `0x4080000000` | 2 GiB |
| Heap | `0x4100000000` | `0x47E0000000` | ~30 GiB |
| Stack | `0x47E0000000` | `0x4800000000` | 32 MiB |

> **Note**: The arm64 user-space memory map uses different base addresses
> (defined in `Kernel/Arch/arm64/linker/linker.ld`).

## 4. Process Management

### 4.1 Process Model

- Each process has its own address space (x86_64 CR3 / arm64 TTBR0_EL1)
- Max processes: 256 (`OS_CONFIG_PROCESS_MAX_COUNT`, range 1–256)
- Scheduling: Round-robin with timer-driven preemption
- Process creation: `process_create_user()`, `process_spawn_user_elf()`
- Process states tracked via process table

### 4.2 Capability System

Every process has a bitmask of capabilities:

| Capability | Bit | Controls |
|---|---|---|
| `PROCESS_CAP_SERIAL` | 0 | Serial I/O syscalls |
| `PROCESS_CAP_PROCESS` | 1 | Process creation/spawning |
| `PROCESS_CAP_FILE` | 2 | File operations |
| `PROCESS_CAP_MEMORY` | 3 | Memory allocation |
| `PROCESS_CAP_INPUT` | 4 | Keyboard/mouse input |
| `PROCESS_CAP_SIGNAL` | 5 | Signal handling |
| `PROCESS_CAP_IPC` | 6 | Inter-process communication |
| `PROCESS_CAP_NETWORK` | 7 | Network access |
| `PROCESS_CAP_DISPLAY` | 8 | Display / framebuffer access |

Default: All capabilities granted (`PROCESS_CAP_DEFAULT_MASK`).

### 4.3 File Descriptors

- Per-process FD table (`OS_CONFIG_FILE_MAX_FD` = 256 by default, range 4–256)
- Operations: open, read, write, close, seek, stat, pipe, dup/dup2
- Directory handles: opendir/readdir/closedir
  (`OS_CONFIG_FILE_MAX_DIR_HANDLE` = 256 by default)

## 5. System Call Interface

### 5.1 ABI Convention

| Architecture | Instruction | Entry point |
|---|---|---|
| **x86_64** | `SYSCALL` / `SYSRET` | `Kernel/Arch/x86_64/cpu/Syscall_Entry.asm` |
| **arm64** | `SVC` | `Kernel/Arch/arm64/cpu/ExceptionVectors.S` |

x86_64 convention:
- Syscall number: `RAX`
- Arguments: `RDI` (arg1), `RSI` (arg2), `RDX` (arg3), `R10` (arg4), `R8` (arg5)
- Return value: `RAX`

arm64 convention:
- Syscall number: `X8`
- Arguments: `X0`–`X5`
- Return value: `X0`

### 5.2 Syscall Categories

| Range | Category | Examples |
|---|---|---|---|
| 1–5 | Serial I/O | putchar, puts, write_u64/u32/u16 |
| 6–12 | Process | create, yield, exit, thread_create/exit/join/detach |
| 23–42 | File I/O | open, read, write, close, seek, mkdir, opendir, readdir, unlink |
| 43–60 | Memory, Graphics, IPC | mmap, malloc, free, memcpy, display draw/present, IPC, display topology |
| 100–109 | Network (TCP/UDP) | connect, listen, accept, send, recv, close, get_state |
| 110–124 | Process Ext. | waitpid, getppid, sleep, getcwd, proc info, spawn with args, launch args |
| 130–145 | Socket API | socket, connect, bind, listen, accept, send, recv, get_info, set/get option, shutdown |
| 140 | RTC | get_rtc_time |
| 150–159 | Memory Ext. | mprotect, munmap, mremap, getuid/gid/tid |
| 160–179 | Epoll/Clock/Linux | epoll, clock_gettime, readv, writev, ioctl, fcntl, access, poll |
| 180–195 | Futex/Clone/Signal/SHM | futex, clone, rt_sigaction, fork, execve, shm_create/grant/map/unmap/close |
| 200–207 | Sys Info | cpu, memory, vmem, disk, device, graphics, arch, system info |
| 208–211 | DRM | open, ioctl, close, mmap |
| 212–219 | Evdev + Block | evdev open/read/ioctl/close, disk count, raw block read/write, boot font |
| 220–229 | Unix Socket | socket, bind, listen, accept, connect, send, recv, sendmsg, recvmsg, close |
| 230–234 | Audio | audio open, get_info, write, drain, close |
| 235–236 | Process Sched | get_cpu_usage, set_process_priority |
| 240–243 | KVM | open, ioctl, close, mmap |
| 250–254 | System | shutdown, reboot, shutdown broadcast, get total/used memory |
| 260–261 | Debug | get_proc_debug_info, get_os_debug_info |
| 262–267 | Wi-Fi / DHCP | wifi scan_start / get_scan_results / connect / disconnect / get_status, net_get_dhcp_dns |

Exact numbers and any additions live in `Kernel/Core/syscall/Syscall_Main.h`
(the ranges above are not fully contiguous).

### 5.3 Error Handling

All syscalls return `os_status_t` (int64_t):
- `0` = success (or positive for data like PIDs, FDs, byte counts)
- Negative = error (maps directly to errno via `os_status_to_errno()`)

## 6. Inter-Process Communication (IPC)

### 6.1 Message Passing

- Ring-buffer queue per process
- Max message size: 256 bytes (`IPC_MESSAGE_MAX_SIZE`)
- Max messages per process queue: 128 (`IPC_MAX_MESSAGES_PER_PROCESS`)
- Sender PID attached to each message automatically

### 6.2 API

```c
os_status_t ipc_send_message(int32_t target_pid, const void *message, uint32_t size);
os_status_t ipc_receive_message(ipc_message_t *out_message);
```

### 6.3 Window Manager Protocol

The window manager uses IPC messages for all GUI operations:
- Window creation/destruction (`WM_CREATE_WINDOW`, `WM_DESTROY_WINDOW`)
- Drawing operations (`WM_DRAW_PIXEL`, `WM_DRAW_RECT`, `WM_BLIT_BUFFER`)
- Input forwarding (`WM_KEYBOARD_EVENT`, `WM_MOUSE_EVENT`)
- Window management (`WM_SET_WINDOW_RECT`, `WM_SHOW_WINDOW`, `WM_RAISE_WINDOW`)

## 7. Virtual File System (VFS)

### 7.1 Design

- Prefix-based mount system (e.g., `/` maps to FAT32 driver)
- `VFS_FILE` and `VFS_DIRENT` types alias FAT32 structures
- Case-sensitivity configurable at runtime

### 7.2 VFS Operations

| Operation | Description |
|---|---|
| `vfs_find_file()` | Locate file by path |
| `vfs_read_file()` / `vfs_write_file()` | Full file read/write |
| `vfs_read_at()` / `vfs_write_at()` | Positional read/write |
| `vfs_truncate()` | Truncate file |
| `vfs_creat()` | Create new file |
| `vfs_mkdir()` | Create directory |
| `vfs_unlink()` | Delete file |
| `vfs_opendir()` / `vfs_readdir()` / `vfs_closedir()` | Directory listing |

### 7.3 FAT32 Driver

- Loadable driver module (`FAT32_Driver.ELF`)
- Full read/write support
- Directory operations (create, list, delete)
- BPB parsed from boot partition at boot time

## 8. Network Stack

### 8.1 Protocol Layers

```
┌────────────────────────────────────────┐
│       UDP / TCP / ICMP / DHCP         │
├────────────────────────────────────────┤
│               IPv4                     │
├────────────────────────────────────────┤
│           ARP / Ethernet              │
├────────────────────────────────────────┤
│      NIC Driver (VirtIO-Net)          │
└────────────────────────────────────────┘
```

### 8.2 Features

- **Ethernet**: Frame construction and parsing
- **ARP**: Address resolution (request/reply, cache)
- **IPv4**: Packet routing, fragmentation (basic)
- **UDP**: Connectionless datagrams, port binding
- **TCP**: Full connection lifecycle (connect, listen, accept, send, recv, close, state machine)
- **ICMP**: Echo request/reply (ping)
- **DHCP**: Client for dynamic IP configuration, driven centrally from
  `network_stack_on_timer_tick()` (see `Docs/Architecture/Network_Stack.md`)
- **DNS**: Resolver (userland, `Userland/Service/com.ImplusOS.netstack/DNS/`)
- **TLS**: Record layer + TLS 1.3 key schedule and the underlying crypto
  primitives live in `Library/Crypto/` (shared kernel/userland source)

Each `Kernel/Network/` protocol layer is also registered into `DeviceRegistry`
under `DEVICE_TYPE_NET_PROTOCOL` with a real vtable and a `deps[]` list by
`Kernel/Drivers/Module/NetworkBuiltinDrivers.c` — a hybrid step toward fully
modular protocol drivers (`Docs/Architecture/Network_Stack.md` §2).

### 8.3 Configuration

Default static IP (from `config.h`):
- Address: `10.0.2.15` (`OS_CONFIG_NET_IPV4_ADDR`)
- Netmask: `255.255.255.0` (`OS_CONFIG_NET_IPV4_MASK`)
- Gateway: `10.0.2.2` (`OS_CONFIG_NET_IPV4_GATEWAY`)

## 9. Driver Module System

### 9.1 Architecture

Drivers are **PIC (Position-Independent Code) ELF shared objects** loaded by the kernel at boot time.

```
┌──────────────────────────────────────────┐
│              Kernel Core                  │
│                                           │
│  driver_module_manager_init()             │
│     ↓                                     │
│  Parse ELF, relocate, find entry symbol   │
│     ↓                                     │
│  Call driver_module_init(kernel_api)       │
│     ↓                                     │
│  Driver returns driver_module_descriptor_t │
│     ↓                                     │
│  driver_manager_attach(name, kind, api)   │
└──────────────────────────────────────────┘
```

### 9.2 Kernel API (`driver_binary_t`)

The kernel provides drivers with a vtable of kernel services:
- Timer: `timer_msleep()`, `timer_hz()`, `timer_ticks()`
- Memory: `malloc()`, `free()`, `dma_alloc()`, `dma_free()`, `virt_to_phys()`
- I/O Ports: `inb()`, `outb()`, `inl()`, `outl()`
- Disk: `disk_read()`, `disk_write()`
- PCI: `pci_read_config()`, `pci_write_config()`
- MMIO: `map_mmio_virt()`
- Debug: `serial_write_char()`, `serial_write_string()`, `serial_write_uint32()`

### 9.3 Driver Types

| Type | `device_type_t` | Module Names |
|---|---|---|---|
| PCI Bus | `DEVICE_TYPE_PCI` | `PCI_Driver` |
| Filesystem | `DEVICE_TYPE_FILESYSTEM` | `FAT32_Driver`, `exFAT_Driver`, `ISO9660_Driver` |
| Display | `DEVICE_TYPE_DISPLAY` | `ImplusOS_Generic_Display_Driver`, `VirtIO_Driver` (VirtIO-GPU) |
| Input | `DEVICE_TYPE_INPUT` | `PS2_Driver` |
| USB Host | `DEVICE_TYPE_USB` | `USB_Driver` (OHCI, UHCI, EHCI, XHCI + HID + Mass Storage) |
| NIC | `DEVICE_TYPE_NIC` | `VirtIO_Driver` (VirtIO-Net) |
| Block | `DEVICE_TYPE_BLOCK` | `AHCI_Driver`, `NVMe_Driver`, `VirtIOBlk_Driver` |
| Audio | `DEVICE_TYPE_AUDIO` | `AC97_Driver`, `HDA_Driver`, `VirtIOSound_Driver` |

### 9.4 Hot Reload

Drivers support runtime unload/reload:
- `driver_manager_unload_module(name)`
- `driver_manager_reload_module(name)`

## 10. Platform Support

### 10.1 x86_64

- **GDT**: Kernel code/data (Ring 0), User code/data (Ring 3), TSS (see §13)
- **IDT**: 256 interrupt vectors
- **SMP**: Multi-processor startup via AP trampoline, per-CPU GDT/IDT/TSS
- **VMX**: Intel VT-x support (EPT, VM entry/exit), KVM-style interface — initialized via `ops->virtualization_init()`
- **AHCI**: AHCI SATA controller driver for disk I/O
- **NVMe**: NVMe solid-state storage driver
- **VirtIO-Blk**: VirtIO block device driver
- **PS/2**: `Kernel/Drivers/Input/PS2/` targets the legacy PC/AT PS/2
  controller and is x86_64-only in the sense that no arm64 board this
  kernel targets exposes one — the build system does not ARCH-filter
  driver modules (`DRIVER_MAKEFILES` in the top-level `Makefile` discovers
  every `Kernel/Drivers/<Category>/<Driver>/Makefile` unconditionally), so
  `PS2_Driver.ELF` is still built and staged for an `ARCH=arm64` image; it
  simply never finds its controller and stays inert there (see
  `Docs/Others/TODO_OS_Refactor.md` 8.3 — this is an accepted, harmless
  arch-specific-hardware case, not something the driver model needs to
  prevent)

### 10.2 arm64 (AArch64)

- **Exception levels**: EL2 (boot manager trampoline) → EL1 (kernel) → EL0 (userland)
- **Exception vectors**: `Kernel/Arch/arm64/cpu/ExceptionVectors.S` — 16-entry vector table
- **SMP**: PSCI interface for CPU bringup (`Kernel/Arch/arm64/smp/PSCI.c`)
- **GIC**: Generic Interrupt Controller v3 driver (`Kernel/Arch/arm64/interrupt/GIC.c`)
- **Timer**: ARM Generic Timer (`Kernel/Arch/arm64/timer/GenericTimer.c`)
- **UART**: PL011 serial driver (`Kernel/Arch/arm64/hal/uart_hal.c`)
- **Block**: AHCI/NVMe/VirtIO-Block storage drivers (shared with x86_64)
- **Audio**: AC97/HDA/VirtIO-Sound drivers (shared with x86_64)

### 10.3 Interrupts

| Architecture | Controller | Timer | IPI |
|---|---|---|---|
| **x86_64** | IOAPIC + LAPIC | LAPIC timer | LAPIC IPI (TLB shootdown) |
| **arm64** | GICv3 | Generic Timer | SGI via GIC |

### 10.4 ACPI

- RSDP v1/v2 discovery (used on both architectures)
- MADT parsing (x86_64: LAPIC, IOAPIC enumeration; arm64: GIC structures)
- Used for SMP initialization and interrupt routing

## 11. Debug Infrastructure

### 11.1 Serial Output

- COM1 port (`0x3F8`), 115200 baud
- `serial_write_char()`, `serial_write_string()`
- Primary debug output channel
- Mapped to QEMU's `-serial stdio`

### 11.2 Kernel Printf

- Custom `printf()` implementation (no floating point)
- Output routed to serial

### 11.3 Boot Progress Bar

- Source: `Kernel/Boot/LoadBar.c` / `LoadBar.h`
- Displays a graphical spinner during kernel initialization phases
- Driven by `timer_set_callback()` with a timer-driven update
- Finished before the fade-to-black transition to userland

### 11.4 Panic Handler

- `kernel_panic()` — Prints message to serial, halts all CPUs
- Stack trace output (when available)

## 12. Kernel Configuration

All compile-time configuration in `Kernel/include/kernel/config.h`:

| Macro | Default | Range | Description |
|---|---|---|---|
| `OS_CONFIG_PROCESS_MAX_COUNT` | 256 | 1–256 | Max concurrent processes |
| `OS_CONFIG_FILE_MAX_FD` | 256 | 4–256 | Max file descriptors per process |
| `OS_CONFIG_FILE_MAX_DIR_HANDLE` | 256 | 4–256 | Max directory handles per process |
| `OS_CONFIG_SMP_MAX_CPUS` | 16 | — | Max CPU cores |
| `OS_CONFIG_SMP_ENABLED` | 1 | — | Enable SMP |
| `OS_CONFIG_DRIVER_MODULE_MAX_COUNT` | 64 | — | Max loadable driver modules |
| `OS_CONFIG_LOG_FILE_MAX_BYTES` | 512K | — | Max kernel log size |
| `OS_CONFIG_BOOT_FADE` | 0 | — | Fade-to-black transition to userland (disabled) |
| `OS_CONFIG_SIGNAL_HANDLER_MAX_PER_PROCESS` | 32 | — | Max signal handlers |
| `OS_CONFIG_PENDING_SIGNAL_MAX_PER_PROCESS` | 32 | — | Max pending signals |
| `OS_CONFIG_NET_IPV4_ADDR` | 10.0.2.15 | — | Static IPv4 address |
| `OS_CONFIG_NET_IPV4_MASK` | 255.255.255.0 | — | IPv4 netmask |
| `OS_CONFIG_NET_IPV4_GATEWAY` | 10.0.2.2 | — | IPv4 gateway |

## 13. GDT Segment Layout (x86_64 only)

> arm64 does not use a GDT; privilege separation is handled via exception levels (EL0/EL1).

| Selector | Offset | Description |
|---|---|---|
| `GDT_KERNEL_CODE` | `0x08` | Kernel code (Ring 0, 64-bit) |
| `GDT_KERNEL_DATA` | `0x10` | Kernel data (Ring 0) |
| `GDT_USER_COMPAT_CODE` | `0x18` | User 32-bit compat code (unused) |
| `GDT_USER_DATA` | `0x20` | User data (Ring 3) |
| `GDT_USER_CODE` | `0x28` | User code (Ring 3, 64-bit) |
| `GDT_TSS` | `0x30` | Task State Segment |
