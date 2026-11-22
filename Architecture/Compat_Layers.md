# Compatibility Layers — ImplusOS

*Last reviewed: 2026-11-18 (three Linux ABIs: x86_64, i386, AArch64 -- see §6;
programs for the other CPU run under the translator -- see §7).
The userland POSIX layer lives in libc, `libc/I_libc/Source/src/posix.c`.*

## 1. Two different things are both called "compat" in this codebase

It is easy to conflate these — they solve different problems at different
layers:

| | `Kernel/Compat/` | `libc/I_libc/Source/src/posix.c` |
|---|---|---|
| **Runs in** | Ring 0 (kernel) | Ring 3 (linked into each process) |
| **Translates** | A foreign **syscall-number ABI** (which syscall number means what, how arguments are packed) into calls against ImplusOS's real kernel subsystems | ImplusOS's **native syscall API** into POSIX libc entry points (`open`, `read`, `fork`, `socket`, `pthread_*`, ...) |
| **Chosen by** | `ELF_Loader.c` detecting `EI_OSABI == ELFOSABI_LINUX` in the binary being executed, once, at `exec()` time | Whatever the linked userland binary was compiled against |
| **Example** | `Kernel/Compat/Linux/Syscall_LinuxCompat.c` — Linux's `SYS_read`/`SYS_openat`/... numbers | `libc/I_libc/Source/src/posix.c` — `open()` calling `file_open()` |

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
Kernel/Source/Compat/
├── compat_types.h      -- process_abi_t, compat_dispatch_fn_t, compat_layer_t
├── compat_registry.h/.c -- compat_registry_{init,register,find}()
└── Linux/
    ├── Syscall_LinuxCompat.h  -- linux_syscall_dispatch(), linux_compat_layer_register()
    ├── Syscall_LinuxCompat.c  -- the Linux implementation (x86_64 numbering)
    ├── Linux_CompatCore.h     -- linux_syscall_core(): the entry the other ABIs use
    ├── Linux_FrontEnd.h       -- helpers shared by the two front ends below
    ├── i386/Syscall_LinuxI386.c   -- Linux i386 front end (x86_64 kernel)
    └── arm64/Syscall_LinuxArm64.c -- Linux AArch64 front end (arm64 kernel)
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

## 6. The three Linux ABIs

| ABI | `abi_mode` | Kernel | Trap | Numbering | Data model |
|---|---|---|---|---|---|
| Linux x86_64 | `PROCESS_ABI_LINUX` (1) | x86_64 | `syscall` | x86_64 table | LP64 |
| Linux i386 | `PROCESS_ABI_LINUX_I386` (2) | x86_64 | `int $0x80` | i386 table | ILP32 |
| Linux AArch64 | `PROCESS_ABI_LINUX_ARM64` (3) | arm64 | `svc #0` | asm-generic table | LP64 |

`PROCESS_ABI_IS_LINUX()` (Core/process/ProcessManager.h) is "is this a Linux
program at all" -- the fd table, /proc, the pty layer and the pipe/file code
ask that, not which ABI.

**Which ABI a program gets** is decided by `ELF_Loader.c` at `exec()`:
`ELFCLASS32`/`EM_386` is Linux i386; an `ELFCLASS64` image with
`EI_OSABI == ELFOSABI_LINUX`, or with a non-zero entry below the native code
base `0x4000000000`, is Linux x86_64 on the x86_64 kernel and Linux AArch64 on
the arm64 kernel. A `PT_INTERP` must be of the same ABI as the program.

### 6.1 One implementation, two front ends

`Syscall_LinuxCompat.c` implements Linux once, in the x86_64 numbering and
structure layouts. The i386 and AArch64 layers are front ends: they translate
a call into its x86_64 form, call `linux_syscall_core()`, and translate the
result back. Structures whose layout differs are rebuilt in scratch space just
below the caller's stack pointer (neither ABI has a red zone, and a restarted
call rebuilds it), where the shared code reads and writes it as it would the
program's own buffer. `linux_syscall_core()` leaves the result in a
`linux_call_t` rather than in the caller's registers, and when a blocking call
has to be re-executed it re-arms the caller's *own* syscall number (EAX for
i386, x8 for AArch64) and steps the PC back over the trap instruction (2 bytes
for `int $0x80`, 4 for `svc`).

What the front ends translate:

- **AArch64**: renumbering; `struct stat` (fstat/newfstatat); the four `O_*`
  flags whose values differ (`O_DIRECTORY`, `O_NOFOLLOW`, `O_DIRECT`,
  `O_LARGEFILE`, in `openat`, `openat2`, `fcntl(F_GETFL/F_SETFL)`); `clone`'s
  argument order (tls before child_tid); `PROT_BTI`/`PROT_MTE` (ignored).
  `struct epoll_event` needs nothing: the kernel's own epoll uses the natural
  16-byte layout when built for arm64 (Core/syscall/Syscall_Epoll.h). `rseq`
  returns ENOSYS (the preemption fixup exists only on x86_64).
- **i386**: every structure with a `long`, a pointer or a 32-bit `time_t`
  (timespec/timeval/itimer*, stat/stat64/statfs/statfs64, rusage, sysinfo,
  sigaction, stack_t, iovec, msghdr and its control messages, flock/flock64,
  rlimit, siginfo, linux_dirent, tms), 64-bit arguments split over two
  registers (`_llseek`, `pread64`, `ftruncate64`, `fallocate`, `fadvise64*`,
  `mmap2`'s page offset), the multiplexed `socketcall` and `ipc` (shared
  memory only), the 16-bit uid calls, old `sigaction`/`signal`/`sigprocmask`,
  `select`, `getdents`, and `set_thread_area`/`get_thread_area`. `int`
  arguments that may be negative (descriptors, pids, timeouts, offsets) are
  sign-extended per call. The `*_time64` calls map onto the x86_64 ones
  unchanged.

### 6.2 What is ABI-specific outside Compat/

- **Thread creation**: both front ends create `CLONE_THREAD` threads with
  `process_create_thread_clone()`, which starts the child from a copy of the
  caller's whole register frame (AArch64 glibc keeps the thread function in
  x10/x11 across the `svc`; i386 glibc pops it off the new stack after
  `int $0x80`). Everything that is not a thread goes through the shared
  `clone()`/`fork()`.
- **Signals** (Core/process/ProcessManager_Create.c): i386 `sigframe` and
  `rt_sigframe` (with the 624-byte `_fpstate_32` FXSAVE image) and the AArch64
  `rt_sigframe` (with an `fpsimd_context` record and a frame record), both at
  scheduling boundaries and for synchronous faults; `sigreturn`/`rt_sigreturn`
  for both. Neither ABI has a vDSO here, so a process gets a read-only page of
  return trampolines (`USER32_SIGPAGE`, `USER_ARM64_SIGPAGE`) used when the
  handler has no `SA_RESTORER` -- AArch64 glibc never passes one.
- **Initial stack**: 32-bit argv/envp/auxv words for i386 (with
  `AT_PLATFORM=i686`); `AT_HWCAP = FP|ASIMD` for AArch64.
- **uname**: `machine` is `x86_64`, `i686` or `aarch64` after the program's
  ABI.

### 6.3 Linux i386 on the x86_64 kernel

A 32-bit program runs in compatibility mode with the GDT's 32-bit user code
segment (0x1B); the user data segment has D/B set so it serves as a 32-bit
stack segment. `int $0x80` has a DPL-3 interrupt gate whose stub
(`isr_int80`, Arch/x86_64/cpu/Syscall_Entry.asm) builds the same register
frame as `syscall_entry`, at the same place, and passes EBX..EBP as the six
arguments. All exits back to user mode take the code/stack selectors from
per-CPU state set on every switch (`syscall_set_user_segments()`), and a
32-bit task always leaves through IRETQ, with DS/ES loaded and its ECX handed
back. A trap that does not match the task's ABI (`int $0x80` from a 64-bit
task, `syscall` from a 32-bit one) gets ENOSYS.

**TLS**: `set_thread_area()` descriptors live per thread
(`process_t.compat_tls`) and are copied into GDT entries 12-14 of whichever
CPU runs the thread; the thread's FS/GS selectors are saved when it leaves the
CPU and reloaded (GS through a SWAPGS pair) when it comes back.

**Address space**: a 32-bit program can only address the low 4 GiB, which the
x86_64 kernel identity-maps into every address space. A window of it
(`OS_CONFIG_LINUX_I386_WINDOW_BASE/SIZE`, default 0x08000000 + 256 MiB, which
covers the classic non-PIE link address 0x08048000) has its physical frames
withheld at boot, and an i386 process's page tables have the window's identity
entries removed (`paging_prepare_compat32_window()`), leaving a hole the
process fills demand-zero. Everything the process maps lives inside it
(`USER32_*` in Core/process/ProcessManager.h): the image and brk, a
page-granular reusable mmap area, the small bump heap, ld.so, the stack and the
trampoline page. The reservation costs the window's size in RAM and is skipped
-- with i386 programs then refused by the loader -- when the window is not free
RAM at boot; `OS_CONFIG_LINUX_I386_WINDOW_SIZE=0` turns it off. Multiarch
library paths are in `OS_CONFIG_LINUX_RUNTIME_LIB_PATH_I386`; the image does
not ship an i386 runtime.

### 6.4 Linux AArch64 on the arm64 kernel

AArch64 programs use the same address-space layout as x86_64 Linux programs
(PIE and ld.so are relocated to `USER_CODE_BASE`). Below the user windows the
arm64 TTBR0 tables are the kernel's own and shared, so a fixed-address
(ET_EXEC) program linked low -- the AArch64 default of 0x400000 -- is refused
by the loader; build such programs as PIE or static-PIE. Multiarch library
paths are in `OS_CONFIG_LINUX_RUNTIME_LIB_PATH_ARM64`; the image does not ship
an AArch64 runtime.

The address space of a Linux process on the arm64 kernel (the x86_64
programs run by the translator, section 7, live in it too):

- `[256 GiB, 288 GiB)` code, heap and stack windows; above them, up to
  512 GiB, the window `SYSCALL_GET_DISPLAY_FRAMEBUFFER` maps the display
  framebuffer into (`USER_DISPLAY_FB_BASE`/`USER_DISPLAY_FB_LIMIT`; every
  L1 slot from 256 GiB to 512 GiB is the process's own); and from 1 TiB the
  mmap arena. The kernel places mappings in `[1 TiB, 16 TiB)`
  (`USER_MMAP_BASE`/`USER_MMAP_LIMIT`); `MAP_FIXED`/`MAP_FIXED_NOREPLACE`
  are accepted, demand-zero, up to 128 TiB (`USER_MMAP_FIXED_LIMIT`), the
  47-bit range x86_64 programs assume. Wine probes for free space downward
  from `0x7fffffff0000` one 64 KiB step at a time and used to spend hours
  in refused `mmap()` calls before reaching 16 TiB.
- Below 256 GiB everything is the kernel's identity map, shared by
  every process: fixed mappings there fail, except in the low *shadow
  window* `[0x70000000, 0x80000000)`, whose RAM is withheld at boot. A
  process may map its own pages there (Wine's `KUSER_SHARED_DATA` at
  `0x7ffe0000` and its low reservations); the tables on the way are made
  private to it first (`shadow_unshare_path()` in
  `Arch/arm64/mmu/Paging_Main.c`), never written in the kernel's shared ones.
- `fork()` is copy-on-write, as on x86_64: both sides map the frame
  read-only with a software bit, and the first write copies it (or, for the
  last owner, just takes write access back).
- A process's memory is freed when it exits, not when its parent reaps it:
  once its last thread has left the CPUs, its tables go on a deferred queue
  that frees them when no CPU runs on them any more (per-CPU TTBR0 tracking,
  `paging_space_active_on_any_cpu()`). A zombie nobody waited for used to
  keep its whole address space.
- `PROT_NONE` is not enforced on pages never touched: they are filled with
  zeroes on first access whatever the protection was. Guard pages
  (thread stacks, Wine's) only work once committed.

### 6.5 Not provided (any ABI unless noted)

vDSO; `modify_ldt`; robust futex lists and `rseq` for i386; `rseq` for
AArch64; i386 `shmctl` other than `IPC_RMID`; i386 `sendmmsg`/`recvmmsg`,
`timer_create`, `preadv2`/`pwritev2`; AArch64 ET_EXEC images below
0x4000000000; ioctls whose argument structures contain pointers or `long`s are
passed to the shared code unconverted for i386.

## 7. Programs for the other CPU: the translator

A Linux ELF built for the CPU the kernel was **not** built for -- an x86_64
program on the arm64 kernel, an AArch64 program on the x86_64 kernel -- runs
under a user-mode CPU translator, the same arrangement as binfmt_misc +
qemu-user on Linux (or Rosetta 2 on macOS):

| Kernel | Foreign program | Translator on the image |
|---|---|---|
| arm64 | Linux x86_64 (`EM_X86_64`) | `/Userland/Translator/x86_64.ELF` |
| x86_64 | Linux AArch64 (`EM_AARCH64`) | `/Userland/Translator/aarch64.ELF` |

Source: `Userland/Application/Translator/` (README there for the internals).
i386 programs on the arm64 kernel are not handled.

### 7.1 What the kernel does

Only the dispatch, in `Core/process/ProcessManager_Create.c`
(`process_foreign_translator()`), for both `execve()` and the native spawn
path: when the target is an ELFCLASS64 file for the other CPU and
`OS_CONFIG_TRANSLATOR_DIR/<isa>.ELF` exists, the kernel loads the translator
instead, with

    argv = { <translator>, "-0", <argv[0]>, <program path>, <argv[1..]> }

(the qemu-user command line). The check comes after the `#!` step, so a
script whose interpreter is a foreign binary works as well. The process
keeps the *program's* name and exe path (`ps`, `/proc/<pid>/exe` -- a re-exec
of `/proc/self/exe` comes back through the same dispatch); the launch log
says `exec (translated)` / `spawn (translated)`. A spawned program gets the
library path of its own ISA in `LD_LIBRARY_PATH`
(`OS_CONFIG_LINUX_RUNTIME_LIB_PATH` for x86_64 programs,
`..._ARM64` for AArch64 ones). `OS_CONFIG_TRANSLATOR_DIR` set to `""`, or a
missing translator file, turns it off; the program then fails to load as
before.

To the kernel the translator is an ordinary Linux program of its own CPU
(static, `EI_OSABI = ELFOSABI_LINUX`, linked at `USER_CODE_BASE`), so
everything after the dispatch is the existing Linux ABI of that kernel
(§6): its system calls, signals, threads and faults. The one thing the
arm64 kernel provides for it besides: EL0 may clean and invalidate caches
by address and read `CTR_EL0` (`SCTLR_EL1.UCI`/`UCT`, set in
`Arch/arm64/mmu/Paging_Main.c` as Linux sets them), which the translator's
JIT needs to make the code it writes visible to instruction fetch.

### 7.2 What the translator does

- **Loading.** It maps the program and its `PT_INTERP` itself: one
  anonymous reservation per image, the whole pages of each segment mapped
  privately from the file over it (`MAP_FIXED`; the kernel's FileMap pages
  them in on first touch and, on arm64, shares them between processes),
  and the partial pages at segment ends copied in with `pread`, so
  overlapping segments and odd alignments need nothing more from the
  kernel. (Copying whole segments made every process exec'd from
  Chromium's binary hold its ~350 MB image.) It builds the initial stack and auxiliary vector for the
  guest (`AT_PLATFORM`, `AT_HWCAP` of the guest CPU, no `AT_SYSINFO_EHDR`:
  there is no vDSO) and starts the guest's dynamic linker. Guest and host
  share one address space: a guest pointer is a host pointer, so a system
  call whose arguments are plain memory is a renumbering at most.
- **CPU.** One emulator per guest ISA: x86_64 integer, x87, MMX, SSE, SSE2
  and SSE3 (CPUID reports exactly that, so ifunc-selecting code never picks
  an SSSE3+/AVX path); AArch64 ARMv8.0 integer, FP and Advanced SIMD
  (`AT_HWCAP = FP | ASIMD`; PAC/BTI hints execute as NOPs).
  x86_64 code on the arm64 kernel is compiled to AArch64 machine code by a
  JIT: guest registers in host registers, flags computed only where they
  are read (`cmp` + `jcc` becomes `cmp` + `b.cond`), most SSE as AdvSIMD,
  LOCKed instructions as exclusive load/store loops, translations chained
  to each other and to an inline indirect-branch table, and faults in
  generated code mapped back to the guest instruction and its exact
  register and flag state. AArch64 code on the x86_64 kernel runs through
  a per-thread cache of predecoded "uops" (a handler plus decoded operands)
  instead. In both, whatever has no native form runs through the
  instruction-at-a-time interpreter, which stays the reference
  implementation (`XL_NOBLOCKS=1` runs it alone, `XL_NOJIT=1` the uop
  engine for x86_64). A translation of code in a range mapped executable
  and not writable is trusted until that range is unmapped, remapped or
  made writable; any other is compared with guest memory before it runs,
  so code the guest rewrites (a JIT of its own) is retranslated. Where the two
  hosts' floating point differs from the guest's (NaN propagation, the
  sign of the default NaN, MIN/MAX of NaN and signed zeros, overflowing
  float-to-int conversion) the guest's behaviour is reproduced explicitly.
  The x86 guest's memory ordering is kept x86-like (TSO) with barriers once
  the process has a second thread.
- **System calls.** Guest calls are turned into the canonical form -- Linux
  x86_64 numbering and structure layouts, the choice §6.1 makes in the
  kernel -- and then into the host's own (identity on x86_64; on arm64 the
  calls the generic table dropped become their `*at`/`p*` successors, and
  `struct stat`, `struct epoll_event`, four `O_*` flags, `getdents` and
  `clone`'s argument order are converted). `seccomp`, `rseq`, `io_uring`
  and `modify_ldt` are refused: they would expose guest code or guest
  numbers to the host kernel.
- **Signals.** The guest's dispositions live in the translator. A host
  handler only queues a signal; the interpreter takes it at an instruction
  boundary and builds the guest's own `rt_sigframe` (x86_64 or AArch64
  layout, with its FP state), and the guest's `rt_sigreturn` restores from
  it. A blocking system call is made in a window the handler can bail out
  of with `EINTR`, after which the call is re-run or not according to the
  guest handler's `SA_RESTART`. The host's blocked mask mirrors the guest's
  (minus SIGSEGV/SIGBUS/SIGILL/SIGFPE/SIGTRAP, which must always reach the
  translator); a fault while touching guest memory becomes the guest's
  SIGSEGV on the faulting instruction.
- **Threads and processes.** `clone(CLONE_VM)` becomes a host thread with a
  copy of the caller's guest CPU state; fork is a host fork; vfork runs as a
  fork. `execve()` goes to the host kernel, which dispatches again.

### 7.3 The image

On x86_64 nothing changes: the bundled Linux runtime is x86_64. On arm64 the
same x86_64 runtime (`Vendor/LinuxRuntime`, `Build/x86_64/LinuxRuntime/stage`)
is now bundled too and runs under the translator -- `/bin/sh` (dash),
fastfetch, xterm, ... -- unless the image is built with
`LINUX_RUNTIME_TRANSLATED=0`. No AArch64 runtime is shipped on either
image.

### 7.4 Limits

- Speed: x86_64 programs on arm64 run a few times slower than native
  (a mixed benchmark and a dash loop: 25-30 times faster than the uop
  engine), compute-bound SSE code more slowly than integer code (XMM
  registers stay in memory). AArch64 programs on x86_64 have no JIT: the
  uop engine runs on the order of 10^8-10^9 guest instructions per second,
  roughly 25-50 times slower than native.
- x86_64: SSE3 is the newest extension (enough for Google Chrome's CPU
  check, not for its speed); no SSSE3, SSE4, AVX or BMI. x87 transcendental
  instructions (FSIN, FYL2X, ...) are computed in double precision.
  Fixed-address (non-PIE) programs linked below `0x4000000000` -- the static
  BusyBox, for one -- cannot be placed on the arm64 kernel, whose low page
  tables are shared.
- AArch64: no LSE atomics, SVE, CRC32 or crypto extensions (none are
  reported).
- `vfork()` is a fork; `posix_spawn()` therefore cannot report an `execve`
  failure through its shared-memory channel and reports success instead.
- What runs on arm64 (QEMU HVF, 8 GiB): Chromium headless (`--dump-dom`
  with JavaScript, about 10 s) and with its X11 window through XSession
  (first-run page drawn in about 50 s); Wine 10 (`wine64 cmd.exe /c echo`,
  the prefix created on the first start in about a minute; `winecfg`
  through XSession). Chromium's zygote and crashpad handlers outlive a
  browser that exits, keeping about 1 GiB and the profile's
  ProcessSingleton, so a second launch hands its URL to them and exits.
