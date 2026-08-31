# Compatibility Layers — ImplusOS

*Last reviewed: 2026-08-29 (post phase P6 of `Docs/Others/TODO_OS_Refactor.md`).
The userland POSIX layer moved to a hot-loadable service at
`Userland/Service/com.ImplusOS.posix/`; paths below reflect that.*

## 1. Two different things are both called "compat" in this codebase

It is easy to conflate these — they solve different problems at different
layers:

| | `Kernel/Compat/` | `Userland/Service/com.ImplusOS.posix/` |
|---|---|---|
| **Runs in** | Ring 0 (kernel) | Ring 3 (linked into each process) |
| **Translates** | A foreign **syscall-number ABI** (which syscall number means what, how arguments are packed) into calls against ImplusOS's real kernel subsystems | ImplusOS's **native syscall API** into POSIX libc entry points (`open`, `read`, `fork`, `socket`, `pthread_*`, ...) |
| **Chosen by** | `ELF_Loader.c` detecting `EI_OSABI == ELFOSABI_LINUX` in the binary being executed, once, at `exec()` time | Whatever the linked userland binary was compiled against |
| **Example** | `Kernel/Compat/Linux/Syscall_LinuxCompat.c` — Linux's `SYS_read`/`SYS_openat`/... numbers | `Userland/Service/com.ImplusOS.posix/src/posix_file.c` — `open()` calling `Syscalls_FileOpen()` |

A Linux-ABI binary running on ImplusOS uses **only** the left column: its own
libc already speaks raw Linux syscall numbers, so ImplusOS intercepts those
numbers directly in the kernel. A native ImplusOS binary linked against a
POSIX-flavored libc uses **only** the right column: its libc calls ImplusOS's
own native syscalls, which never go through `Kernel/Compat/` at all. The two
layers do not call each other and do not need to agree on anything.

This document covers `Kernel/Compat/` only.

## 2. Why a registry instead of a hardcoded `if`

Before this refactor, `Syscall_Dispatch.c`'s SYSCALL entry point had:

```c
extern uint64_t linux_syscall_dispatch(uint64_t, uint64_t, ...);
if (process_get_current_abi_mode() == PROCESS_ABI_LINUX) {
    request_switch |= linux_syscall_dispatch(saved_rsp, num, arg1, ...);
    goto pre_schedule;
}
```

— meaning the one general-purpose syscall dispatcher had to know Linux exists
by name, forward-declare its entry point ad hoc, and would need a second
hardcoded `if` for every future compat layer. `Kernel/Compat/compat_registry.c`
replaces this with a small lookup table `Syscall_Dispatch.c` queries generically:

```c
const compat_layer_t *compat = compat_registry_find(process_get_current_abi_mode());
if (compat != NULL) {
    request_switch |= (int)compat->dispatch(saved_rsp, num, arg1, arg2, arg3, arg4, arg5, arg6);
    goto pre_schedule;
}
```

`Syscall_Dispatch.c` now contains zero Linux-specific identifiers.

## 3. The pieces

```
Kernel/Compat/
├── compat_types.h      -- process_abi_t, compat_dispatch_fn_t, compat_layer_t
├── compat_registry.h/.c -- compat_registry_{init,register,find}()
└── Linux/
    ├── Syscall_LinuxCompat.h  -- linux_syscall_dispatch(), linux_compat_layer_register()
    └── Syscall_LinuxCompat.c  -- the actual ~3000-line Linux syscall table
```

```c
/* Kernel/Compat/compat_types.h */
typedef uint8_t process_abi_t;

typedef uint64_t (*compat_dispatch_fn_t)(uint64_t saved_rsp, uint64_t num,
                                         uint64_t arg1, uint64_t arg2,
                                         uint64_t arg3, uint64_t arg4,
                                         uint64_t arg5, uint64_t arg6);

typedef struct {
    process_abi_t abi;
    const char *name;
    compat_dispatch_fn_t dispatch;
} compat_layer_t;
```

A layer registers a `static const compat_layer_t` describing itself:

```c
/* Kernel/Compat/Linux/Syscall_LinuxCompat.c */
static const compat_layer_t g_linux_compat_layer = {
    .abi = PROCESS_ABI_LINUX,
    .name = "Linux",
    .dispatch = linux_syscall_dispatch,
};

void linux_compat_layer_register(void)
{
    (void)compat_registry_register(&g_linux_compat_layer);
}
```

and something calls that registration function once, early in boot —
`Kernel/Core/syscall/Syscall_Init.c`'s `syscall_init()`:

```c
void syscall_init(void) {
    syscall_init_per_cpu();
    compat_registry_init();
    linux_compat_layer_register();
}
```

## 4. Adding a new compat layer

1. Create `Kernel/Compat/<Name>/` with your syscall table, following
   `Kernel/Compat/Linux/Syscall_LinuxCompat.c`'s shape (it can be as large or
   as small as the ABI needs — the registry doesn't care about size, only
   about the `compat_dispatch_fn_t` signature).
2. Give `Kernel/Core/process/ProcessScheduler.h` a new `PROCESS_ABI_<NAME>`
   value (a plain `uint8_t` constant, matching the existing
   `PROCESS_ABI_IMPLUS`/`PROCESS_ABI_LINUX`).
3. Teach `ELF_Loader.c` to detect the new ABI at `exec()` time (however that
   ABI identifies itself in its own ELF headers — `EI_OSABI` is what Linux
   uses, but this is not universal).
4. Export a `static const compat_layer_t` and a `<name>_compat_layer_
   register()` function, same shape as step 3's Linux example above.
5. Call it from `Syscall_Init.c`'s `syscall_init()`, next to
   `linux_compat_layer_register()`.

`Syscall_Dispatch.c` needs **no changes** for any of this — that is the entire
point of the registry.

## 5. Design decisions worth knowing

- **`compat_dispatch_fn_t`'s signature matches the one real implementation,
  not an idealized generic shape.** It takes `saved_rsp` (the raw saved user
  register frame) plus a syscall number and 6 arguments, because that is
  exactly what `linux_syscall_dispatch()` already needed — a different future
  ABI with a genuinely different argument-passing convention may need this
  typedef extended, and that is fine; there is no value in speculatively
  generalizing it further today.
- **`process_abi_t` is deliberately *not* wired into
  `ProcessScheduler.h`.** That header's `PROCESS_ABI_IMPLUS`/`PROCESS_ABI_LINUX`
  remain plain `#define`s over a bare `uint8_t`, unchanged — the process/
  scheduler subsystem is referenced far too pervasively across the kernel to
  safely introduce a new named type into it as a side effect of this phase.
  `compat_types.h`'s `process_abi_t` is `Kernel/Compat`'s own typed view of
  the same value space; the underlying values are identical, so this is a
  documentation/API-clarity distinction, not a runtime one.
- **The registry has a fixed cap of 4 layers** (`COMPAT_REGISTRY_MAX` in
  `compat_registry.c`) and re-registering the same ABI value replaces rather
  than duplicates the entry — a deliberate, simple design matching this
  kernel's other small fixed-size tables (see `Docs/Others/
  TODO_OS_Refactor.md` 7. item 1's audit of similar tables) rather than a
  dynamically-growable list, since the number of syscall-ABI compat layers a
  hobby OS realistically ships is small and known well ahead of time.
