# Driver Module Guide — ImplusOS

*Last reviewed: 2026-08-29.*

## 1. Overview

ImplusOS uses a **loadable driver module system**. Drivers are compiled as **PIC
(Position-Independent Code) ELF shared objects** that the kernel loads at boot
time. The kernel provides a vtable of kernel services (`driver_binary_t`), and
drivers export an initialization function that returns a descriptor with the
driver's API.

## 2. Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                        Kernel Core                           │
│                                                              │
│  ┌──────────────────────────────────────────────────────────┐│
│  │               Driver Module Manager                      ││
│  │                                                          ││
│  │  1. Load ELF from boot filesystem                        ││
│  │  2. Parse ELF headers, apply relocations                 ││
│  │  3. Find driver_module_init symbol                       ││
│  │  4. Call driver_module_init(kernel_api)                   ││
│  │  5. Receive driver_module_descriptor_t                    ││
│  │  6. Attach driver to Driver Manager                      ││
│  └──────────────────────────────────────────────────────────┘│
│                                                              │
│  ┌──────────────────────────────────────────────────────────┐│
│  │               Driver Manager                             ││
│  │                                                          ││
│  │  Categorizes drivers by kind:                            ││
│  │  - PCI, FAT32, Display, Input, USB, NIC                  ││
│  │  Provides unified access: driver_manager_get_*()         ││
│  └──────────────────────────────────────────────────────────┘│
│                                                              │
│  ┌──────────────────────────────┐ ┌─────────────────────────┐│
│  │  Kernel-resident glue        │ │   Module Framework      ││
│  │  (Drivers/Module/ -- talks   │ │   (load/unload/         ││
│  │   to a loaded driver's       │ │    init/reload)          ││
│  │   vtable via DriverManager)  │ │                          ││
│  └──────────────────────────────┘ └─────────────────────────┘│
└──────────────────────────────────────────────────────────────┘
```

> **Note (2026-08-24)**: The `Client`/`Server` directory split described in
> earlier revisions of this document has been removed
> (`Docs/Others/TODO_OS_Refactor.md`, phases P1/P2). Loadable driver
> implementations now live directly under `Kernel/Drivers/<Category>/<Driver>/`
> (no `Server/` segment, including `Kernel/Drivers/FileSystem/{FAT32,ISO9660,exFAT}/`);
> the kernel-resident gateway code that used to live under
> `Kernel/Drivers/Client/` now lives in `Kernel/Drivers/Module/` alongside
> `DriverManager`/`BusRegistry`/`DeviceRegistry`, including the per-filesystem
> `*_VFS_Bridge.c/.h` files that replaced the old per-FS `*_Client.c` +
> `*_VFS_Adapter.c` pairs. `Kernel/Drivers/Client/` and `Kernel/Drivers/Server/`
> no longer exist anywhere in the tree. A full rewrite of this guide's prose
> (not just the paths) is tracked under phase P7 of the same TODO.

## 3. Driver Categories

| Category | `device_type_t` | Example Drivers | Key Interface |
|---|---|---|---|---|
| PCI Bus | `DEVICE_TYPE_PCI` | `PCI_Driver.ELF` | `pci_driver_t` |
| Filesystem | `DEVICE_TYPE_FILESYSTEM` | `FAT32_Driver.ELF`, `exFAT_Driver.ELF`, `ISO9660_Driver.ELF` | `fat32_driver_t` / `iso9660_driver_t` |
| Display | `DEVICE_TYPE_DISPLAY` | `ImplusOS_Generic_Display_Driver.ELF`, `VirtIO_Driver.ELF` | `driver_display_t` |
| Input | `DEVICE_TYPE_INPUT` | `PS2_Driver.ELF` | `driver_input_t` |
| USB Host | `DEVICE_TYPE_USB` | `USB_Driver.ELF` | `usb_master_vtable_t` |
| NIC | `DEVICE_TYPE_NIC` | `VirtIONet_Driver.ELF`, `I219V` (Intel I219-V), `WiFi` / `AX900_Driver.ELF` (Wi-Fi, with `Firmware/AX900` blob) | `driver_nic_t` |
| Block | `DEVICE_TYPE_BLOCK` | `AHCI_Driver.ELF`, `NVMe_Driver.ELF`, `VirtIOBlk_Driver.ELF` | `driver_storage_t` |
| Audio | `DEVICE_TYPE_AUDIO` | `AC97_Driver.ELF`, `HDA_Driver.ELF`, `VirtIOSound_Driver.ELF` | (audio API) |
| Serial / RTC | (platform) | `Kernel/Drivers/Serial/`, `Kernel/Platform/rtc/` | — |

Source layout: driver modules live under `Kernel/Drivers/<Category>/<Driver>/`
(`Audio/`, `Block/`, `Bus/{PCI,USB}/`, `Display/`, `FileSystem/`, `Input/`,
`NIC/`, `Wi-Fi/`, `Serial/`). `Kernel/Drivers/ExampleDriver/` is a
minimal template. (The CMOS RTC is a port-IO platform HAL, not a driver
module, and lives under `Kernel/Platform/rtc/`.) The on-demand USB/PCI VID→module map is
`Kernel/Drivers/Manifest/DriverDB.txt`.

## 4. Kernel API (`driver_binary_t`)

The kernel passes this vtable to every driver during initialization. Most functions
are architecture-neutral; where noted, some are specific to x86_64 or arm64.

```c
typedef struct {
    // Timer
    void (*timer_msleep)(uint32_t ms);
    uint32_t (*timer_hz)(void);
    uint64_t (*timer_ticks)(void);

    // Memory allocation
    void *(*malloc)(uint64_t size);
    void (*free)(void *ptr);

    // DMA (physically contiguous memory)
    void *(*dma_alloc)(size_t size, uint64_t *phys_out);
    void (*dma_free)(void *ptr, size_t size);
    uint64_t (*virt_to_phys)(void *virt);

    // Memory operations
    void *(*memset)(void *s, int c, size_t n);
    void *(*memcpy)(void *dst, const void *src, size_t n);

    // I/O ports (x86_64 only; arm64 uses MMIO via map_mmio_virt)
    uint8_t (*inb)(uint16_t port);
    void (*outb)(uint16_t port, uint8_t value);
    uint32_t (*inl)(uint16_t port);
    void (*outl)(uint16_t port, uint32_t value);

    // Block I/O
    bool (*disk_read)(uint32_t lba, uint8_t *buffer, uint32_t sector_count);
    bool (*disk_write)(uint32_t lba, const uint8_t *buffer, uint32_t sector_count);

    // PCI configuration space
    uint32_t (*pci_read_config)(uint8_t bus, uint8_t device, uint8_t func, uint8_t offset);
    void (*pci_write_config)(uint8_t bus, uint8_t device, uint8_t func,
                              uint8_t offset, uint32_t value);

    // MMIO mapping
    void *(*map_mmio_virt)(uint64_t phys_addr);

    // Debug output
    void (*serial_write_char)(char c);
    void (*serial_write_string)(const char *str);
    void (*serial_write_uint32)(uint32_t val);
} driver_binary_t;
```

## 5. Driver Descriptor

Every driver module must export:

```c
typedef struct {
    const void *driver_api;    // Pointer to the driver's API struct
    void (*shutdown)(void);    // Called when driver is unloaded
} driver_module_descriptor_t;

typedef const driver_module_descriptor_t *(*driver_module_init_fn_t)(
    const driver_binary_t *api
);
```

The kernel looks for the symbol `driver_module_init` in the loaded ELF.

## 6. Writing a New Driver

### 6.1 Source Structure

Create a directory under `Kernel/Drivers/<Category>/<DriverName>/`:

```
Kernel/Drivers/
└── MyCategory/
    └── MyDriver/
        ├── Makefile
        ├── MyDriver.c
        └── MyDriver.h
```

### 6.2 Implementation Template

```c
// MyDriver.c
#include "Drivers/Module/DriverBinary.h"

static const driver_binary_t *g_api = NULL;

// Your driver's API implementation
static bool my_driver_init(void) {
    g_api->serial_write_string("MyDriver: Initializing...\n");
    // Initialize hardware, allocate resources
    return true;
}

static void my_driver_shutdown(void) {
    g_api->serial_write_string("MyDriver: Shutting down\n");
    // Release resources
}

// Fill in your driver-specific vtable
static const driver_display_t my_display_ops = {
    .name       = "MyDriver",
    .probe      = my_driver_probe,
    .init       = my_driver_init,
    .is_ready   = my_driver_is_ready,
    .width      = my_driver_width,
    .height     = my_driver_height,
    .draw_pixel = my_driver_draw_pixel,
    .fill_rect  = my_driver_fill_rect,
    .present    = my_driver_present,
    // ...
};

static const driver_module_descriptor_t descriptor = {
    .driver_api = &my_display_ops,
    .shutdown   = my_driver_shutdown,
};

// Required export symbol
const driver_module_descriptor_t *driver_module_init(const driver_binary_t *api) {
    g_api = api;
    return &descriptor;
}
```

### 6.3 Makefile Template

```makefile
include ../../module.mk
```

The path depends on how deep `MyDriver/` sits under `Kernel/Drivers/` --
count the `../` needed to reach `Kernel/Drivers/module.mk` from your driver's
own directory (two levels for the common `Kernel/Drivers/MyCategory/MyDriver/`
layout shown above). The `module.mk` file provides standard build rules for
driver modules; it discovers every `*.c` file under your driver's own
directory automatically, so keep driver-only sources there and nothing else.

### 6.4 Kernel-Resident Gateway Code

A loadable driver module cannot be called directly by other kernel code or by
other driver modules (each is a separately linked, separately loaded ELF with
its own address space). If your driver needs to be reachable from elsewhere
in the kernel by something other than the generic `DriverManager`/
`BusRegistry`/`DeviceRegistry` lookup (see `Kernel/Drivers/Module/PCI_Client.c`,
`NIC.c`, or `Display_Main.c` for worked examples), add that resident glue code
directly to `Kernel/Drivers/Module/` -- it is compiled straight into the
kernel binary alongside `DriverManager` itself (see
`DRIVER_RESIDENT_DIRS` in `Kernel/Drivers/Makefile`), not built as a separate
module:

```
Kernel/Drivers/Module/
├── MyDriver_Gateway.c   -- looks up the loaded module via
│                           driver_manager_find(DEVICE_TYPE_MYCATEGORY, "MyDriver.ELF")
│                           and forwards calls to its vtable
└── MyDriver_Gateway.h
```

Most new drivers do **not** need this: reaching a driver through
`DriverManager`'s existing `driver_manager_get_by_kind()`/`driver_manager_find()`
directly from the calling code is preferred where it is sufficient, and this
kernel-resident-glue pattern should stay the exception, not the default (see
`Docs/Others/TODO_OS_Refactor.md` phase P1 for why the former `Client`/
`Server` split was retired).

### 6.5 Registration

The driver is automatically discovered when placed in `Kernel/Driver/` on the
boot filesystem. The `driver_module_manager_init()` function loads all ELFs
from the boot info's loaded files, and `driver_module_init_all()` calls each
driver's init function.

## 7. Driver Interface Specifications

### 7.1 Display Driver (`driver_display_t`)

```c
typedef struct {
    const char *name;
    bool (*probe)(void);                                    // Check if hardware is present
    bool (*init)(void);                                     // Initialize the display
    bool (*is_ready)(void);                                 // Is display ready for rendering?
    uint32_t (*width)(void);                                // Display width in pixels
    uint32_t (*height)(void);                               // Display height in pixels
    void (*draw_pixel)(uint32_t x, uint32_t y, uint32_t color);
    void (*fill_rect)(uint32_t x, uint32_t y, uint32_t w, uint32_t h, uint32_t color);
    void (*present)(void);                                  // Swap/flush buffers
    bool (*set_framebuffer)(const driver_boot_framebuffer_t *fb);
    void *(*get_framebuffer)(void);                         // Direct framebuffer access
} driver_display_t;
```

### 7.2 Input Driver (`driver_input_t`)

```c
typedef struct {
    void (*init)(void);
    void (*poll)(void);                                     // Poll for new events
    int32_t (*read_keyboard)(driver_keyboard_event_t *out); // 0 = no event, 1 = event
    int32_t (*read_mouse)(driver_mouse_event_t *out);       // 0 = no event, 1 = event
    void (*drain_keyboard)(driver_keyboard_event_t *tmp,
                           void (*forward)(driver_keyboard_event_t *));
    void (*drain_mouse)(driver_mouse_event_t *tmp,
                        void (*forward)(driver_mouse_event_t *));
} driver_input_t;
```

### 7.3 Keyboard Event

```c
typedef struct __attribute__((packed)) {
    uint16_t keycode;      // Scancode
    uint8_t  pressed;      // 1 = press, 0 = release
    uint8_t  ascii;        // ASCII character (0 if non-printable)
    uint8_t  modifiers;    // SHIFT | CTRL | ALT | CAPS
    uint8_t  reserved[3];
} driver_keyboard_event_t;
```

Modifier flags:
- `DRIVER_KBD_MOD_SHIFT` (bit 0)
- `DRIVER_KBD_MOD_CTRL` (bit 1)
- `DRIVER_KBD_MOD_ALT` (bit 2)
- `DRIVER_KBD_MOD_CAPS` (bit 3)

### 7.4 Mouse Event

```c
typedef struct __attribute__((packed)) {
    uint16_t x;            // Absolute X position
    uint16_t y;            // Absolute Y position
    uint8_t  buttons;      // Button state bitmask
    int8_t   wheel;        // Scroll wheel delta
    uint8_t  reserved[2];
} driver_mouse_event_t;
```

### 7.5 Storage Driver (`driver_storage_t`)

```c
typedef struct {
    bool (*read_sectors)(uint32_t lba, uint8_t *buffer, uint32_t sector_count);
    bool (*write_sectors)(uint32_t lba, const uint8_t *buffer, uint32_t sector_count);
} driver_storage_t;
```

### 7.6 NIC Driver (`driver_nic_t`)

```c
typedef struct {
    bool (*init)(void);
    bool (*is_ready)(void);
    uint16_t (*mtu)(void);
    void (*get_mac)(uint8_t mac_out[6]);
    bool (*send_frame)(const uint8_t *frame, uint16_t frame_len);
    void (*poll)(void);
    void (*set_rx_callback)(driver_nic_rx_callback_t cb);
} driver_nic_t;
```

### 7.7 USB Master (`usb_master_vtable_t`)

```c
typedef struct {
    driver_input_t   input;    // USB HID input
    driver_storage_t storage;  // USB Mass Storage
    driver_usb_t     usb;      // Low-level USB transfer API
} usb_master_vtable_t;
```

## 8. Existing Driver Modules

### PCI Driver (`PCI_Driver.ELF`)

- Source: `Kernel/Drivers/Bus/PCI/PCI_Main.c`
- Enumerates PCI bus devices
- Provides config space read/write

### FAT32 Driver (`FAT32_Driver.ELF`)

- Source: `Kernel/Drivers/FileSystem/FAT32/FAT32_Main.c`
- Full FAT32 filesystem implementation
- Read/write files, directories, create/delete
- BPB parsing, cluster chain walking, FAT table management

### PS/2 Driver (`PS2_Driver.ELF`)

- Source: `Kernel/Drivers/Input/PS2/PS2_Input.c`
- PS/2 keyboard and mouse input
- Scancode translation to ASCII
- Mouse packet parsing

### USB Driver (`USB_Driver.ELF`)

- Source: `Kernel/Drivers/Bus/USB/`
- Host controller support: OHCI, UHCI, EHCI, XHCI
- Device classes: HID (keyboard/mouse), Mass Storage
- Hub enumeration and device setup

### VirtIO GPU Driver (`VirtIO_Driver.ELF`)

- Source: `Kernel/Drivers/Display/VirtIO/`
- VirtIO-GPU display driver (double-buffered)

### VirtIO-Net Driver (`VirtIONet_Driver.ELF`)

- Source: `Kernel/Drivers/NIC/VirtIONet/`
- VirtIO-Net NIC driver (Ethernet frame send/receive)

### Intel I219-V Driver

- Source: `Kernel/Drivers/NIC/I219V/`
- Intel I219-V (e1000e-class) wired NIC

### AX900 Wi-Fi Driver

- Source: `Kernel/Drivers/Wi-Fi/AX900/` (also `Kernel/Drivers/NIC/WiFi/`)
- Wi-Fi NIC presenting a `driver_nic_t`; needs the firmware blob staged from
  `Kernel/Drivers/Firmware/AX900/`. WPA/association crypto comes from
  `Library/Crypto/`. This is the driver that already uses the newer
  `bus_matches[]` + `probe()` binding model (see
  `Docs/Others/TODO_OS_Refactor.md` §5.3)

### Generic Display Driver (`ImplusOS_Generic_Display_Driver.ELF`)

- Source: `Kernel/Drivers/Display/ImplusOS_Generic/ImplusOS_Generic.c`
- Fallback framebuffer driver using the boot framebuffer from UEFI GOP
- Double-buffered rendering

### AHCI Driver (`AHCI_Driver.ELF`)

- Source: `Kernel/Drivers/Block/AHCI/`
- AHCI SATA controller driver
- Provides block-level read/write for disk I/O

### NVMe Driver (`NVMe_Driver.ELF`)

- Source: `Kernel/Drivers/Block/NVMe/`
- NVMe solid-state storage driver
- High-performance block I/O

### VirtIO Block Driver (`VirtIOBlk_Driver.ELF`)

- Source: `Kernel/Drivers/Block/VirtIOBlk/`
- VirtIO block device driver for QEMU virtual storage

### exFAT Driver (`exFAT_Driver.ELF`)

- Source: `Kernel/Drivers/FileSystem/exFAT/exFAT_Main.c`
- exFAT filesystem implementation, **read-only** (write support is a
  tracked follow-up, see `Docs/Others/TODO_OS_Refactor.md` 6.3)
- Boot sector parsing, MBR/GPT partition detection, FAT chain walking with
  the `NoFatChain` contiguous-allocation optimization, File Directory
  Entry / Stream Extension / File Name entry-set parsing

### ISO9660 Driver (`ISO9660_Driver.ELF`)

- Source: `Kernel/Drivers/FileSystem/ISO9660/`
- ISO9660 (CD-ROM) filesystem implementation
- Used for booting from CD-ROM images

### AC97 Audio Driver (`AC97_Driver.ELF`)

- Source: `Kernel/Drivers/Audio/AC97/`
- Intel AC97 audio controller driver

### HDA Audio Driver (`HDA_Driver.ELF`)

- Source: `Kernel/Drivers/Audio/HDA/`
- Intel High Definition Audio driver

### VirtIO Sound Driver (`VirtIOSound_Driver.ELF`)

- Source: `Kernel/Drivers/Audio/VirtIOSound/`
- VirtIO sound device driver for QEMU virtual audio

## 9. Hot Reload

Drivers can be unloaded and reloaded at runtime:

```c
// Unload a driver
driver_manager_unload_module("MyDriver");

// Reload a driver
driver_manager_reload_module("MyDriver");
```

This calls the driver's `shutdown()` callback, unlinks from the Driver Manager,
then re-loads and re-initializes the driver ELF.

## 10. Driver Selection

The `DriverSelect` module (`Kernel/Drivers/Module/DriverSelect.c`) handles:
- Setting the boot framebuffer from UEFI GOP
- Selecting the best available display driver (VirtIO-GPU preferred, fallback to generic)
