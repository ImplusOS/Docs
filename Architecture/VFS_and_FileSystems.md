# VFS and Filesystems — ImplusOS

*Last reviewed: 2026-08-29 (post phase P2 of `Docs/Others/TODO_OS_Refactor.md`;
re-verified against `Kernel/Core/vfs/VFS.c` and `Kernel/Drivers/FileSystem/`).*

## 1. Overview

`Kernel/Core/vfs/VFS.c` is a thin, name-independent dispatch layer over a set of
mounted `vfs_driver_t` vtables. It does not know that FAT32, ISO9660, or exFAT
exist — it only knows about **prefixes** (path routing) and **media kinds**
(default-filesystem selection). Every real filesystem's code lives entirely
outside `VFS.c`, in `Kernel/Drivers/FileSystem/<Name>/`.

```
┌─────────────────────────────────────────────────────────────────┐
│  Syscall_File.c (open/read/write/...)                            │
└───────────────────────────┬───────────────────────────────────────┘
                             │ vfs_find_file() / vfs_read_at() / ...
┌───────────────────────────▼───────────────────────────────────────┐
│  Core/vfs/VFS.c                                                   │
│    g_vfs_drivers[16]  -- mounted vfs_driver_t table                │
│    vfs_resolve_candidates() -- longest-prefix-match + default-fs   │
└─────┬───────────────┬───────────────┬───────────────┬─────────────┘
      │ prefix=""     │ prefix=""     │ prefix="/dev"  │ prefix="/proc"
┌─────▼─────┐   ┌──────▼──────┐  ┌─────▼─────┐  ┌───────▼───────┐
│ FAT32     │   │ ISO9660     │  │ DevFS     │  │ ProcFS        │
│ VFS_Bridge│   │ VFS_Bridge  │  │ (pseudo)  │  │ (pseudo)      │
└─────┬─────┘   └──────┬──────┘  └───────────┘  └───────────────┘
      │ driver_manager_find()  │ driver_manager_find()
┌─────▼─────┐   ┌──────▼──────┐
│FAT32_Main │   │ISO9660_Main │   (loadable driver modules, .ELF)
│(driver_   │   │             │
│ module_   │   │             │
│ init())   │   │             │
└───────────┘   └─────────────┘
```

## 2. `vfs_driver_t` — the contract every filesystem implements

Defined in `Kernel/include/kernel/interfaces/vfs_types.h`:

```c
typedef enum {
    VFS_MEDIA_KIND_UNKNOWN = 0,
    VFS_MEDIA_KIND_OPTICAL,   /* read-only optical media (ISO9660) */
    VFS_MEDIA_KIND_DISK,      /* writable fixed/removable disk (FAT32, exFAT) */
    VFS_MEDIA_KIND_PSEUDO,    /* DevFS/TmpFS/ProcFS/EtcFS */
} vfs_media_kind_t;

typedef struct vfs_driver {
    const char *fs_type;      /* informational / vfs_set_default_fs(name) lookup only */
    const char *prefix;       /* mount point, "" for the catch-all root */
    vfs_media_kind_t media_kind;
    bool (*find_file)(const char *path, vfs_file_t *out_file);
    bool (*read_file)(vfs_file_t *file, uint8_t *buffer);
    bool (*write_file)(vfs_file_t *file, const uint8_t *buffer);
    bool (*read_at)(vfs_file_t *file, uint32_t offset, uint8_t *buffer, uint32_t size);
    bool (*write_at)(vfs_file_t *file, uint32_t offset, const uint8_t *buffer, uint32_t size);
    bool (*truncate)(vfs_file_t *file, uint32_t new_size);
    uint32_t (*get_file_size)(vfs_file_t *file);
    bool (*creat)(const char *path);
    bool (*mkdir)(const char *path);
    int32_t (*opendir)(const char *path);
    int32_t (*readdir)(int32_t handle, vfs_dirent_t *out_entry);
    int32_t (*closedir)(int32_t handle);
    bool (*close_file)(vfs_file_t *file);
    bool (*unlink)(const char *path);
    void (*list_root)(void);
    void (*set_case_sensitive)(bool enabled);   /* optional, may be NULL */
    bool (*get_case_sensitive)(void);           /* optional, may be NULL */
} vfs_driver_t;
```

A read-only filesystem (ISO9660, exFAT) simply sets `write_file`/`write_at`/
`truncate`/`creat`/`mkdir`/`unlink` to functions that return `false`
unconditionally — `VFS.c` never special-cases "read-only filesystems", it just
calls through the vtable and gets `false` back like any other failure.

## 3. Why `media_kind`, not `fs_type` string comparison

Before this refactor, `vfs_mount()` had a hardcoded
`if (strcmp(driver->fs_type, "iso9660") == 0)` to decide the boot-time default
filesystem — meaning `VFS.c` had to know ISO9660 existed by name. That is gone.
Boot-time default selection now goes through:

```c
bool vfs_set_default_fs_by_kind(vfs_media_kind_t kind);
```

which scans mounted drivers for the first one whose `media_kind` matches. See
`Kernel/Core/kernel_main.c`'s `all_fs_initialize()`:

```c
if (fat_ok)   vfs_mount("", fat32_vfs_get_driver());
if (exfat_ok) vfs_mount("", exfat_vfs_get_driver());
if (iso_ok)   vfs_mount("", iso9660_vfs_get_driver());

/* Optical media wins when present (LiveCD/installer boot); otherwise the
 * first mounted VFS_MEDIA_KIND_DISK driver (FAT32, since it's mounted
 * first above) is the default. */
if (iso_ok) {
    vfs_set_default_fs_by_kind(VFS_MEDIA_KIND_OPTICAL);
} else if (fat_ok || exfat_ok) {
    vfs_set_default_fs_by_kind(VFS_MEDIA_KIND_DISK);
}
```

`vfs_set_default_fs(const char *fs_type)` (the name-based version) still
exists for callers that already have a specific driver name in hand (a
POSIX-style `mount(2)`), but it never drives *internal* VFS policy anymore.

## 4. Path resolution: `vfs_resolve_candidates()`

`vfs_find_file`/`vfs_creat`/`vfs_mkdir`/`vfs_opendir`/`vfs_unlink` all used to
duplicate the same "longest matching mount-point prefix, then fall back to the
default filesystem" logic five times. It is now one shared helper:

```c
static int vfs_resolve_candidates(const char *path,
                                  const vfs_driver_t *out[VFS_MAX_CANDIDATES]);
```

which every one of those five functions calls, then just loops over the
returned candidate list trying each driver in order until one succeeds. This
also means a future filesystem operation only has to be written once against
`vfs_resolve_candidates()`, not copy-pasted a sixth time.

## 5. The Client/Server split is gone — writing a new filesystem driver

Before this refactor, adding a filesystem meant three files: the driver
implementation, a `*_Client.c` proxy (driver-module lookup), and a
`*_VFS_Adapter.c` (type conversion to `vfs_file_t`). That is now two:

1. **The driver itself**, `Kernel/Drivers/FileSystem/<Name>/<Name>_Main.c` —
   a normal loadable driver module (see `Docs/Architecture/
   Driver_Module_Guide.md`), exporting `driver_module_init()` and its own
   native file-handle type (e.g. `exFAT_FILE`).
2. **A `*_VFS_Bridge.c/.h`** in `Kernel/Drivers/Module/` — looks the driver
   module up once via `driver_manager_find(DEVICE_TYPE_FILESYSTEM, "...")`,
   lazily calls its `init()`, and converts between the driver's native
   handle type and `vfs_file_t`. This is the file that actually implements
   `vfs_driver_t` and that `kernel_main.c` mounts.

### Worked example: the smallest possible read-only bridge

This is the shape every bridge follows (trimmed from
`Kernel/Drivers/Module/exFAT_VFS_Bridge.c` — see that file for the complete,
buildable version with all fields wired up):

```c
#include "MyFS_VFS_Bridge.h"
#include "Drivers/FileSystem/MyFS/MyFS_Main.h"
#include "Drivers/Module/DriverBinary.h"
#include "Drivers/Module/DriverManager.h"
#include <stdlib.h>

static const myfs_driver_t *g_myfs_driver = NULL;
static uint8_t g_myfs_initialized = 0;

static bool myfs_ensure_initialized(void)
{
    const device_t *device =
        driver_manager_find(DEVICE_TYPE_FILESYSTEM, "MyFS_Driver.ELF");
    const myfs_driver_t *driver = device ? (const myfs_driver_t *)device->ops : NULL;
    if (!driver) { g_myfs_driver = NULL; g_myfs_initialized = 0; return false; }
    if (g_myfs_driver != driver) { g_myfs_driver = driver; g_myfs_initialized = 0; }
    if (g_myfs_initialized) return true;
    if (!g_myfs_driver->init || !g_myfs_driver->init()) return false;
    g_myfs_initialized = 1;
    return true;
}

bool myfs_init(void) { return myfs_ensure_initialized(); }

static bool myfs_vfs_find_file(const char *path, vfs_file_t *out_file)
{
    if (!myfs_ensure_initialized()) return false;
    MYFS_FILE *f = (MYFS_FILE *)malloc(sizeof(MYFS_FILE));
    if (!f) return false;
    if (!g_myfs_driver->find_file(path, f)) { free(f); return false; }
    out_file->internal_id = (uint64_t)f;
    out_file->size = f->size;
    out_file->driver_data = f;
    return true;
}

/* ... read_file/read_at/get_file_size/close_file the same shape;
 * write_file/write_at/truncate/creat/mkdir/unlink return false for a
 * read-only filesystem, exactly like ISO9660_VFS_Bridge.c/exFAT_VFS_
 * Bridge.c ... */

static const vfs_driver_t g_myfs_vfs_driver = {
    .fs_type = "myfs",
    .prefix = NULL,
    .media_kind = VFS_MEDIA_KIND_DISK,
    .find_file = myfs_vfs_find_file,
    /* ... */
};

const vfs_driver_t *myfs_vfs_get_driver(void) { return &g_myfs_vfs_driver; }
```

Then in `kernel_main.c`'s `all_fs_initialize()`:

```c
bool myfs_ok = myfs_init();
if (myfs_ok) vfs_mount("", myfs_vfs_get_driver());
```

That is the entire integration surface — `VFS.c` itself never needs to change.

## 6. exFAT — read-only implementation notes

`Kernel/Drivers/FileSystem/exFAT/exFAT_Main.c` implements the exFAT format
per the Microsoft "exFAT File System Specification" (2019-08-28), **read-only**
for now (`write_file`/`write_at`/`creat`/`mkdir`/`unlink`/`truncate` all return
`false`; see `Docs/Others/TODO_OS_Refactor.md` 6.3 for the staged-rollout
rationale). Key structural differences from FAT32:

- **Directory entries are 3-part sets**, not single 32-byte records: a
  `0x85` File Directory Entry, followed by exactly one `0xC0` Stream
  Extension (first cluster, size, the `NoFatChain` flag), followed by one or
  more `0xC1` File Name Entries (15 UTF-16LE characters each). See
  `exfat_scan_next()`.
- **`NoFatChain` optimization**: when a stream's data is contiguous, the
  formatter may skip maintaining its FAT chain entries entirely and just set
  a flag; `exfat_cursor_read_entry()`/`exfat_cluster_at_index()` branch on
  this to either walk the FAT (`exfat_fat_get_next_cluster()`) or compute the
  next cluster as `first_cluster + index` directly (O(1), no FAT reads at
  all).
- **No `.`/`..` entries** — unlike FAT12/16/32, exFAT directories never
  contain dot-entries, so `readdir()` needs no special-case filtering for
  them.
- Allocation Bitmap (`0x81`) and Up-case Table (`0x82`) entries exist in the
  spec but are not read by this implementation — they are only needed for
  free-space allocation (irrelevant to a read-only driver) and
  locale-correct case folding (this driver does plain ASCII
  case-insensitive comparison instead, consistent with how the rest of
  ImplusOS's path handling works).

## 7. VFS handle/mount table limits

| Table | Location | Size | Overflow behavior |
|---|---|---|---|
| Mounted filesystems | `VFS.c`'s `g_vfs_drivers[16]` | 16 | `vfs_mount()` returns `false` and logs via `debug_printf` |
| Open directory handles | `VFS.c`'s `g_vfs_directory_handles[32]` | 32 | `vfs_opendir()` returns `-1` (closes the underlying driver handle first) |

As of this refactor, 4 filesystems (FAT32, exFAT, ISO9660, plus whichever
pseudo-filesystems are mounted: DevFS, TmpFS, ProcFS, EtcFS) use 7–8 of the 16
mount slots at boot, leaving headroom for a POSIX-style runtime `mount(2)`.
