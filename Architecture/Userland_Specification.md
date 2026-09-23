# Userland Specification — ImplusOS

*Last reviewed: 2026-08-29.*

## 1. Overview

The ImplusOS userland runs in Ring 3 (x86_64) / EL0 (arm64) with a separate
address space per process. All kernel services are accessed via architecture-specific
trap instructions (x86_64: `SYSCALL`; arm64: `SVC`). The init process
(`Userland.ELF`) spawns system and user applications.

## 2. Init Process (`Userland/Userland.c`)

The init process is the first user-space process loaded by the kernel.

### Boot Sequence

1. Render the boot screen (framebuffer snapshot / gradient fill), load the
   system font, show the localized welcome text and boot count, fade in.
2. `service_load_all()` — load every service listed in
   `Userland/Service/services.list` (currently `com.ImplusOS.posix`,
   `com.ImplusOS.netstack`) via `Userland/Service/service_client.h`.
3. Spawn `com.ImplusOS.windowmanager` and set its priority; poll
   `window_get_wm_pid()` until it registers.
4. Once the WM is up, spawn `com.ImplusOS.sysnotif` (notification daemon).
5. Free boot-time buffers and enter the idle loop (`sleep_ms()` forever).

`spawn_with_fallbacks()` tries a list of candidate ELF paths in order, so a
component can move without breaking init.

### Entry Point

User-space binaries use `_start()` as their entry point (no `main()`).
The linker script (`Userland.ld`) sets up the correct entry address.

## 3. Syscall API (`Userland/API/`)

The userland exposes typed C wrapper functions for each syscall category:

### API Headers

| Header | Category | Key Functions |
|---|---|---|
| `Serial.h` | Serial I/O | `serial_putchar()`, `serial_puts()` |
| `Process.h` | Process | `process_spawn()`, `process_yield()`, `process_exit()`, `process_get_current_pid()`, `process_waitpid()`, `process_getppid()`, `sleep_ms()`, `get_uptime_ms()` |
| `File.h` | File I/O | `file_open()`, `file_read()`, `file_write()`, `file_close()`, `file_seek()`, `file_mkdir()`, `file_opendir()`, `file_readdir()` |
| `Graphics.h` | Display | `get_display_width()`, `get_display_height()`, `draw_pixel()`, `draw_fill_rect()`, `draw_present()`, `get_framebuffer()` |
| `Input.h` | Input | `input_read_keyboard()`, `input_read_mouse()` |
| `Memory.h` | Memory | `user_malloc()`, `user_free()`, `user_memcpy()`, `user_memset()`, `user_mmap()` |
| `IPC.h` | IPC | `ipc_send()`, `ipc_receive()` |
| `Window.h` | Window Mgr | `window_register_service()`, `window_get_wm_pid()` |
| `Network.h` | Networking | `udp_send()`, `tcp_connect()`, `tcp_send()`, `tcp_recv()`, `tcp_close()` |
| `Socket.h` | Sockets | `socket_create()`, `socket_connect()`, `socket_bind()`, `socket_listen()`, `socket_accept()`, `socket_send()`, `socket_recv()`, `socket_get_info()`, `socket_set_option()`, `socket_get_option()`, `socket_shutdown()` |
| `Time.h` | Time | `get_rtc_time()`, `get_uptime_ms()`, `nanosleep()` |
| `Audio.h` | Audio | `audio_open()`, `audio_get_info()`, `audio_write()`, `audio_drain()`, `audio_close()` |
| `SystemInfo.h` | System Info | `get_cpu_info()`, `get_memory_info()`, `get_disk_info()`, `get_device_info()`, `get_graphics_info()`, `get_arch_info()`, `get_system_info()` |
| `KVM.h` | Virtualization | `kvm_open()`, `kvm_ioctl()`, `kvm_close()`, `kvm_mmap()` |
| `WiFi.h` | Wi-Fi | `wifi_scan_start()`, `wifi_get_scan_results()`, `wifi_connect()`, `wifi_disconnect()`, `wifi_get_status()` |
| `OSDebug.h` | Debug | `get_proc_debug_info()`, `get_os_debug_info()` (boot profiler, per-process stats) |
| `PnP.h` | Devices | Plug-and-play / device enumeration helpers |
| `Error.h` | Errors | `os_errno` definitions |
| `WM_Protocol.h` | WM Protocol | Window manager message definitions |

Helper libraries also live under `Userland/API/`: `FreeType.c/.h` (font
rendering), `Jpeg.c/.h` (libjpeg wrapper), `Zlib.h` (zlib wrapper) and
`XMLParser.c/.h` (UI-layout parser).

### Syscall Mechanism (x86_64)

All wrappers call through `Userland/Syscalls.c` which dispatches to the
architecture-specific implementation in `libc/I_libc/src/sys/$(ARCH)/hal_syscall.c`
(x86_64: `syscall` instruction; arm64: `SVC #0`). The x86_64 implementation:

```c
static inline uint64_t syscall6(uint64_t num, uint64_t a1, uint64_t a2,
                                 uint64_t a3, uint64_t a4, uint64_t a5) {
    uint64_t ret;
    __asm__ volatile(
        "mov %[a4], %%r10\n\t"
        "mov %[a5], %%r8\n\t"
        "syscall"
        : "=a"(ret)
        : "a"(num), "D"(a1), "S"(a2), "d"(a3),
          [a4]"r"(a4), [a5]"r"(a5)
        : "rcx", "r11", "r10", "r8", "memory"
    );
    return ret;
}
```

The arm64 variant uses `SVC #0` with the syscall number in register `X8` and
arguments in `X0`–`X5`. The arch-specific implementation lives in
`libc/I_libc/src/sys/$(ARCH)/hal_syscall.c`.

## 4. POSIX Compatibility Layer (`Userland/Service/com.ImplusOS.posix/`)

A comprehensive POSIX API mapping layer enabling portable C programs. It is
built as a **hot-loadable service** (`com.ImplusOS.posix.so`), loaded at init
from `services.list` and unloadable at runtime via `service_unload()`. See
`Userland/Service/com.ImplusOS.posix/README_POSIX.md`.

### Supported APIs

| Module | Functions | Notes |
|---|---|---|
| **File I/O** | `open`, `creat`, `read`, `write`, `close`, `lseek`, `pipe`, `dup`, `dup2`, `stat`, `fstat`, `mkdir`, `unlink`, `opendir`, `readdir`, `closedir` | O_CREAT, O_TRUNC, O_APPEND, O_RDWR, O_NONBLOCK |
| **Process** | `getpid`, `getppid`, `fork`, `execve`, `execv`, `execvp`, `waitpid`, `wait`, `kill`, `_exit` | `fork()` spawns fresh process (no memory sharing) |
| **Signal** | `signal`, `sigaction`, `sigprocmask`, `sigpending`, `raise`, `sigemptyset`, `sigfillset`, `sigaddset`, `sigdelset` | SA_RESETHAND, SA_NODEFER, full signal numbers 1–31 |
| **Threads** | `pthread_create`, `pthread_join`, `pthread_detach`, `pthread_self`, `pthread_cancel`, `pthread_once`, `pthread_mutex_*`, `pthread_cond_*`, `pthread_key_*` | Mutex types: NORMAL, ERRORCHECK, RECURSIVE. 64 TLS keys × 128 threads |
| **Networking** | `socket`, `bind`, `connect`, `listen`, `accept`, `send`, `recv`, `sendto`, `recvfrom`, `shutdown`, `setsockopt`, `getsockopt`, `htons/ntohs/htonl/ntohl`, `inet_aton/inet_addr/inet_ntoa` | AF_INET only, SOCK_STREAM + SOCK_DGRAM |
| **Time** | `clock_gettime`, `clock_getres`, `nanosleep`, `gettimeofday`, `time`, `gmtime_r`, `localtime_r`, `mktime`, `difftime` | CLOCK_MONOTONIC, CLOCK_REALTIME |
| **Memory** | `mmap`, `munmap`, `mprotect` | munmap/mprotect are no-ops |
| **I/O Mux** | `fcntl`, `ioctl`, `select`, `poll` | select/poll: sleep-based stub |

### errno Mapping

| Kernel Status | Value | POSIX errno |
|---|---|---|
| `OS_STATUS_NOT_FOUND` | -2 | `ENOENT` (2) |
| `OS_STATUS_IO_ERROR` | -5 | `EIO` (5) |
| `OS_STATUS_ACCESS_DENIED` | -13 | `EACCES` (13) |
| `OS_STATUS_FAULT` | -14 | `EFAULT` (14) |
| `OS_STATUS_INVALID_ARG` | -22 | `EINVAL` (22) |
| `OS_STATUS_LIMIT_REACHED` | -24 | `EMFILE` (24) |
| `OS_STATUS_NOT_SUPPORTED` | -95 | `ENOTSUP` (95) |
| `OS_STATUS_INTERNAL` | -255 | `EIO` (5) |

### Limitations

1. `fork()` spawns a fresh process — no copy-on-write or shared memory
2. `mmap()` is anonymous-only natively; file-backed mmap copies data
3. `munmap()` / `mprotect()` are no-ops
4. `select()` / `poll()` sleep-based stubs (all open FDs reported ready)
5. `kill()` supports self-signal only; cross-process returns ENOSYS
6. IPv4 only (no IPv6)

## 5. Application Structure

### Directory Convention

Applications follow reverse-domain naming (`com.ImplusOS.<appname>`) and each
lives directly under `Userland/Application/` with its own `Makefile`:

```
Userland/Application/
├── com.ImplusOS.windowmanager/   compositing window manager
│   ├── Makefile
│   ├── Core/  Compositor/  SceneGraph/  Decoration/  Animation/
│   ├── Input/  IPC/  Font/  Theme/  UI/  Linux/  Tests/
│   └── Resource/{Background.png, Fonts/, Icons/, Themes/,
│                 Applications/*.desktop, Apps/{apps.list, desktop.icons}}
├── com.ImplusOS.linuxapp/        generic X11 host for a foreign Linux program
├── com.ImplusOS.sysnotif/        notification daemon
└── BusyBox/                      BusyBox port (Resource/busybox binary)
```

The hot-loadable **services** are a separate tree, `Userland/Service/`
(`com.ImplusOS.ldso`, `com.ImplusOS.dynmain`, `com.ImplusOS.posix`,
`com.ImplusOS.netstack`), each with its own `Makefile` including
`Userland/Service/Common.mk`.

### Build System

Each application `Makefile` is `include ../../AppCommon.mk`
(i.e. `Userland/AppCommon.mk`), which provides the common objects (libc,
`Syscalls.c`, `Userland/API/*` helpers, `Library/` crypto/unicode). Services
include `Userland/Service/Common.mk`.

### Application ELF Loading

1. App/service ELFs are staged onto the boot filesystem under `/Userland/…`.
2. Init spawns apps via `process_spawn(path)`; services are brought in by
   `service_load_all()` (which resolves `/Userland/Service/<name>/<name>.so`).
3. The kernel loads the ELF and creates a process with a separate address space.
4. Entry point is `_start()` (not `main()`).

## 6. System Applications

### Window Manager (`com.ImplusOS.windowmanager`)

- Registers as the WM service via the `window_register_service()` syscall.
- Compositor with a scene graph, window decorations, animation, themes, a
  taskbar and a Start menu.
- Manages window stacking and focus; forwards keyboard/mouse events to the
  focused window over IPC.
- Renders via direct framebuffer access.

#### Design system (`Userland/API/Source/Material.[ch]`, `Theme/WM_Theme.c`)

The shell is drawn entirely in **Material Design 3** tokens. Nothing in the
window manager names a colour directly; every surface, outline and piece of
text names an M3 role, so the whole desktop is retinted by changing one
value.

- **Colour.** One seed colour expands into five tonal palettes (primary,
  secondary, tertiary, neutral, neutral-variant) plus M3's fixed error
  palette, which `m3_scheme_init()` maps onto the standard role names
  (`primary`, `on_primary_container`, `surface_container_high`,
  `outline_variant`, ...) for either the light or the dark scheme. Tones are
  produced in CIE LCh(ab) -- hue and chroma held, chroma reduced until the
  tone is inside sRGB -- which lands within a few code points of M3's
  published baseline palettes; `Tests/MaterialTest.c` checks that against
  the specification. The default seed is M3's own baseline, `#6750A4`.
- **Shape.** The M3 shape scale (`WM_M3_SHAPE_EXTRA_SMALL` 4 through
  `EXTRA_LARGE` 28, plus `FULL` for a stadium), resolved against each
  component's box by `m3_shape_radius()`.
- **Elevation.** Levels 0-5. In the light scheme a level tints its container
  toward `surface_tint`; in the dark scheme the surface-container tones
  carry it instead, as M3 specifies. Every raised surface draws its key
  light through `wm_canvas_elevation_shadow()`, so a level looks the same
  height wherever it appears.
- **State layers.** Hover 8%, focus 10%, pressed 10%, dragged 16% of the
  "on" colour over the component's own container (`m3_state_layer()`).
- **Type.** The M3 type scale in pixels (`title_large`, `body_medium`,
  `label_small`, ...), with a theme-wide multiplier.
- **Motion.** M3's emphasized-decelerate curve on the way in and
  emphasized-accelerate on the way out, with the M3 duration tokens
  (`m3_ease()`, `M3_DURATION_*`).

Themes are `.theme` files (`Resource/Themes/Material.theme`,
`Material-Dark.theme`). A theme names a `seed`, a `mode` (`light`/`dark`), a
`type_scale` and the four metrics that are the shell's own -- title bar
height, window corner radius, shadow spread, taskbar height -- and may
override individual colour roles by their M3 name. The window manager reads
`/var/System/shell.theme` first, then the shipped light theme; failing both
it keeps the baseline defaults. The `WM_SET_THEME` IPC message re-seeds the
scheme from the accent colour it carries.

The token module is shared, not the shell's own: the login screen and every
application built on ImUI generate the same scheme from the same theme file
(`m3_load_desktop_theme()`, `MaterialTheme.c`). The shell applies a new seed
at once, over the `WM_SET_THEME` message; applications pick it up the next
time they start. `com.ImplusOS.settings` is what sends both.

`AppUITheme.h` holds the same roles as compile-time constants, frozen at the
baseline seed, for a program that wants to match the shell without linking
the palette generator.

#### Applications and ImUI

First-party applications are built on **ImUI** (`Userland/API/Source/ImUI.[ch]`),
a small retained-mode toolkit whose every widget is a Material Design 3
component drawn from the tokens above: text/filled/tonal/outlined buttons
(`imui_set_variant()`), filled text fields, navigation-drawer-style list
items, outlined cards, filled toolbars and the M3 type scale
(`imui_set_type_role()`). The scheme comes from the desktop's theme file at
startup, so an app follows a retint without knowing about it.

| Application | What it is |
|---|---|
| `com.ImplusOS.editor` | text editor |
| `com.ImplusOS.filemanager` | file browser |
| `com.ImplusOS.serialmonitor` | COM1 debug log viewer |
| `com.ImplusOS.settings` | desktop colour scheme (seed + light/dark), CPU/memory/graphics, devices, disks |
| `com.ImplusOS.processmanager` | running processes with live CPU share and memory, and a way to end one |

#### Application list (`Linux/WM_DesktopEntry.c`)

The Start menu and the desktop icons are built from **freedesktop.org desktop
entries**, the same `.desktop` files a Linux distribution uses. This is what
lets a program from the staged Debian runtime appear on the desktop without a
line of C being written for it.

Search directories, highest precedence first:

| Directory | Holds |
|---|---|
| `/var/System/applications` | the user's own entries and overrides (writable tmpfs) |
| `/Userland/com.ImplusOS.windowmanager/Resource/Applications` | the entries shipped with the image |
| `/usr/local/share/applications` | local additions |
| `/usr/share/applications` | whatever `Vendor/LinuxRuntime` staged from the Debian packages |

An entry is identified by its **desktop file ID** (its filename), and the
first directory that supplies that ID decides — including deciding to hide it
with `NoDisplay=true`. Shipping `Resource/Applications/debian-xterm.desktop`
therefore replaces the entry the `xterm` package installs, without touching
the packaged file.

Keys read: `Type` (only `Application` is offered), `Name`, `Comment`,
`GenericName`, their `[ja]` forms (preferred, matching the rest of the UI),
`Exec`, `TryExec`, `Icon`, `Categories`, `NoDisplay`, `Hidden`. `Exec` has its
field codes (`%f`, `%U`, …) stripped and a bare program name resolved against
`/usr/local/bin`, `/usr/bin`, `/bin`, `/usr/games`; an entry whose program is
not on the image is dropped, which is what keeps Debian's descriptions of
software this image does not ship out of the menu.

Two ImplusOS-specific keys:

| Key | Meaning |
|---|---|
| `X-ImplusOS-Linux=true` | what finally runs is a foreign glibc binary, even though `Exec` names a native launcher. Draws the "Linux" chip in the Start menu. |
| `X-ImplusOS-Geometry=WxH` | the window size the host launcher should ask for. |

`Icon` is resolved to a PNG under `/usr/share/icons/{hicolor,Adwaita}/<size>/apps`
or `/usr/share/pixmaps` (largest size first; SVG and XPM cannot be decoded), and
failing that against the standard freedesktop icon names — `utilities-terminal`,
`system-file-manager`, `accessories-text-editor`, … — which are drawn with the
shell's own vector iconography (`UI/WM_Icons.c`). A foreign program also gets
its executable's basename tried as an icon name, which is how Debian's xterm
ends up with the `xterm.png` its package ships even though its entry asks for
`mini.xterm` (an XPM).

Legacy fallbacks, used only when the scan finds nothing at all — that is, on a
damaged image: `Resource/Apps/apps.list`
(`Name|Path|Badge|Category|Comment|Argument`), then a small built-in table.

The desktop icons come from `/var/System/desktop.icons`, falling back to
`Resource/Apps/desktop.icons` (`Label|Path|Argument`, all but the path
optional). They fill top to bottom and wrap into further columns, and an icon
whose target matches a Start-menu entry is drawn with that entry's icon and
name.

`[wm] launcher: N app(s) from M desktop entr(y|ies), K running Linux binaries`
is logged to COM1 on every boot.

### Linux Application Host (`com.ImplusOS.linuxapp`)

The generic host for a foreign X11 program: it creates a window, brings up (or
joins) Xorg with that window's backing store as the KMS scanout target, spawns
the program as an X client and pumps input into `/dev/input/event*` — all of
which is `Userland/API/XSession.c`. It takes

```
[Title|][WxH|]/usr/bin/program --flags
```

as its launch argument, which is what the Start menu passes for an entry whose
`Exec` is not a native `.ELF`.

The launchers that remain program-specific are the ones that do something this
cannot: `Chromium` drops privileges to uid 1000 before spawning (its sandbox
refuses to start as root), `Doom` and `Terminal` predate this and keep their
own tuned command lines, and `com.ImplusOS.gtk3demo` runs a Wayland compositor
rather than an X server.

### Notification Daemon (`com.ImplusOS.sysnotif`)

- Spawned by init once the WM has registered.
- Displays system notifications / toasts through the WM.

### BusyBox (`Userland/Application/BusyBox/`)

- BusyBox port; `Resource/Applications/com.ImplusOS.busybox.desktop` points at
  `/Userland/BusyBox/BusyBox.ELF`. The `busybox` binary resource is gitignored
  and supplied at build time.

## 7. Utility Libraries

### XML Parser (`Userland/API/XMLParser.c`)

Simple XML parser for UI layout files, used by apps that load `layout.xml`
resources.

### DNS Resolver (`Userland/Service/com.ImplusOS.netstack/DNS/DNS.c`)

Userland DNS resolver that performs queries over UDP. Built as the hot-loadable
`com.ImplusOS.netstack` service.

### Dynamic Linker (`com.ImplusOS.ldso` + `com.ImplusOS.dynmain`)

Userland dynamic-linking support for ELF `.so` loading.

### Shared crypto / Unicode (`Library/`)

`Library/Crypto`, `Library/Unicode`, `Library/Identifier` are compiled into
userland binaries (and the kernel) — TLS primitives, hashes, signatures,
CP932/UTF-8 conversion, UUIDs.

## 8. libc (`libc/I_libc/`)

Minimal freestanding C library providing:

| Header | Functions |
|---|---|
| `string.h` | `memcpy`, `memset`, `memmove`, `memcmp`, `strlen`, `strcmp`, `strncmp`, `strcpy`, `strncpy`, `strcat`, `strchr`, `strstr` |
| `stdlib.h` | `atoi`, `itoa`, `abs`, `strtol`, `strtoul` |
| `stdio.h` | `printf`, `sprintf`, `snprintf`, `putchar`, `puts` |
| `math.h` | Basic math functions |
| `errno.h` | `errno` variable |
| `assert.h` | `assert()` macro |

Additional POSIX-compatible headers:
- `unistd.h`, `fcntl.h`, `signal.h`, `time.h`, `pthread.h`
- `sys/socket.h`, `sys/stat.h`, `sys/mman.h`, `sys/types.h`, `sys/wait.h`
- `netinet/in.h`, `arpa/inet.h`
- `dirent.h`, `poll.h`

The architecture-specific trap sequence is in
`libc/I_libc/src/sys/{x86_64,arm64}/hal_syscall.c`.

## 9. Linker Script (`Userland/Userland.ld`)

The userland linker script is shared across architectures and sets:
- Entry point: `_start`
- Code base: `0x4000000000`
- Text, rodata, data, bss sections
- No standard library startup code
