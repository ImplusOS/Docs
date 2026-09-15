# ImplusOS に Chromium を実用動作させるための TODO リスト

> **現況追記 (2026-09-06) — QEMU 実起動でのヘッドレス bring-up:**
>
> この日、初めて **QEMU 実機起動での逐次デバッグ**ができた（KVM 有効）。
> `Userland/Application/Chromium/Chromium.ELF`（`--headless=new --dump-dom
> about:blank`）を GUI 操作なしで走らせるため、init に
> `/Userland/autostart.list`（1 行 1 ELF、既定では未配置）を追加し、
> `make image_livecd AUTOSTART=/Userland/Chromium/Chromium.ELF` で焼く。
>
> **結果**: Chromium は起動し、ポリシー/variations/プロファイル/signin/
> 拡張機能（PDF Viewer 等）/Mojo/PartitionAlloc/NSS/GLib/ANGLE フォールバック
> まで通るようになった。**`-smp 1` では 10 分以上まったく落ちない**。
> `-smp 2` 以上では 15〜90 秒でメモリ破壊由来の SIGSEGV に至る（下記 §10）。
>
> **本セッションで潰した実バグ**（いずれも Chromium 固有ではなく、Linux ABI
> 全般に効く）:
>
> 1. **`open("/proc/self/fd/<n>")` 未実装** — Chromium の共有メモリは memfd
>    (`O_RDWR`) で、ReadOnlySharedMemoryRegion の読み取り専用ハンドルは
>    `open("/proc/self/fd/<n>", O_RDONLY|O_CLOEXEC)` で作られる
>    (`base::subtle::CreateAnonymousRegion`)。ProcFS は readlink しか対応して
>    おらず open が失敗していた。`syscall_file_reopen_fd()` を新設し、
>    同じオブジェクトを別 fd に別アクセスモードで張り直す。
> 2. **`fcntl(F_ADD_SEALS)` / `F_GET_SEALS` 未実装** — 失敗すると Chromium は
>    「この kernel の memfd は使えない」と判断して一時ファイル経路に落ちる。
>    `kernel_memfd_t.seals` を追加、SEAL_SEAL と SHRINK/GROW を ftruncate で
>    強制。
> 3. **`open(path, O_RDWR|O_CREAT)` が新規作成時に `O_WRONLY` を返す** —
>    `syscall_file_creat()` が固定で `O_WRONLY` open していた。
>    `PlatformSharedMemoryRegion::TakeOrFail()` は `fcntl(F_GETFL)` で
>    アクセスモードを検査して CHECK 失敗＝ブラウザ即死。`syscall_file_creat_ex()`
>    を新設し呼び出し側のフラグで開く。
> 4. **futex の待ちスロット枯渇と非 Linux errno** — スレッド消滅時にスロットが
>    解放されず 128 個を食い潰し、`FUTEX_WAIT` が `-ENOMEM` を返していた。
>    glibc の `futex_wait()` は 0/EAGAIN/EINTR 以外を `futex_fatal_error()`
>    （"The futex facility returned an unexpected error code."）で致命扱いに
>    するのでプロセスが死ぬ。スロット数を `OS_CONFIG_PROCESS_MAX_COUNT` に、
>    死んだ待ち手の GC（`futex_gc_locked()`）を追加、枯渇時は EINTR。
> 5. **`shared_memory_map()` の参照カウント漏れ** — 同一アドレス空間が同じ
>    オブジェクトを 2 回 map すると同じアドレスを返すが `references` を増やさず、
>    片方の unmap で領域が解放されていた。`shared_mapping_t.map_refs` を追加。
> 6. **`munmap()` が共有メモリの帳簿を見ていない** — Linux ABI の munmap は
>    `process_user_munmap()` に直行しており、shm の mapping 表にアドレスが
>    残ったままユーザアロケータへ返却されていた。次に同じオブジェクトを map
>    すると、すでに他人のものになった stale アドレスが返る。
>    `shared_memory_unmap_any()` を追加して munmap から呼ぶ。
> 7. **`madvise(MADV_DONTNEED/FREE)` が no-op** — PartitionAlloc は
>    「decommit した領域は次に読むと 0」という Linux の挙動に依存し
>    (`DecommittedMemoryIsAlwaysZeroed()` が true)、recommit 時に memset を
>    省く。結果 `calloc()` が汚れたメモリを返し、libxcb の
>    `xcb_connection_t.setup` がゴミポインタになって `free()` で #GP。
>    mmap アリーナ内の**プライベート無名ページのみ**その場でゼロ化する実装に
>    変更（ファイル裏付け／共有ページは Linux 同様に内容を保持）。
> 8. **SMP の偽ページフォルト** — 他 CPU が張ったばかりのページに対して古い
>    TLB エントリで faulting するのは x86 では正常で、OS 側が許容する必要が
>    ある。`paging_access_is_now_permitted()` を追加し、既に許可されている
>    アクセスなら invlpg して再開する（従来はプロセスを終了させていた）。
>
> **副次的な改善**: `Chromium/Start.c` の `--v=1` を既定オフに
> (`-DCHROMIUM_VERBOSE_LOG=1` で復活)。VERBOSE1 の数千行を 115200 baud の
> COM1 に流すのが起動時間を支配していた。


> **現況追記 (2026-08-29):**
> - Linux ABI 互換レイヤーの実体は P6 リファクタで移動済み。現在のパスは
>   **`Kernel/Compat/Linux/Syscall_LinuxCompat.c`**（本文中の
>   `Kernel/Core/syscall/Syscall_LinuxCompat.c` は旧パス）。ABI 判定は
>   `Kernel/Core/elf/ELF_Loader.c`、ディスパッチ分岐は `compat_registry`
>   経由（`Docs/Architecture/Compat_Layers.md` 参照）。
> - **本ツリーには `libc/glibc` サブモジュールも `chrome-linux/` も存在しない。**
>   本文の §9 以降が前提とする glibc 同梱・ビルド済み Chromium 同梱
>   （`Userland/Application/com.ImplusOS.chrome/`、`WITH_GLIBC=1` /
>   `WITH_CHROME=1` イメージ経路）は未反映。カーネル側の Linux syscall
>   実装（バケット A + 一部 B/C、`OS_CONFIG_FILE_MAX_FD=256` 等）は反映済み。
> - QEMU での実起動完走テストは未実施のまま。

> 対象: `Kernel/Compat/Linux/Syscall_LinuxCompat.c` を中心とする Linux ABI 互換レイヤー
> 前提: x86-64 Long Mode + UEFI ブート、モノリシックカーネル、SYSCALL/SYSRET ABI
> 本ドキュメントの調査基準日: 2026-08-20
> 最終更新: 2026-08-28 — バケット A（自己完結型 syscall）を実装しカーネルをコンパイル検証、glibc 2.41 を `libc/glibc` に submodule 化しフルビルド完走（§9）。バケット B/C は根拠を明記して見送り（§4ter）。追加で `FUTEX_LOCK_PI`/`UNLOCK_PI`（§3.6）、`-DLINUX_SYSCALL_TRACE` syscall トレース（§6）、glibc 用 `/etc` テキストファイル群（§9.2）を実装。
>
> 追記 2026-08-28（同日・第2セッション、「codeable items only」方針でユーザ承認）— バケット B/C には手を付けず、自己完結でコンパイル検証のみ可能な残 `[ ]` 項目を一括実装:
> - §3.4 EPOLLET を実エッジトリガ化（`epoll_entry_t.last_ready`、立ち上がりエッジのみ通知・ERR/HUP は常時）。
> - §3.5 SIGCHLD をハンドラ設置時のみ親へ配送＋`wait4`/`waitid` の POSIX wait ステータス変換（`process_t.exit_by_signal/exit_term_signal`、`process_waitpid_ex`、`process_exit_current_signaled`、PF→SIGSEGV 経路も追随）。
> - §3.5 SIGPIPE 既定動作（reader 不在パイプ書込／peer 切断 TCP 送信で `SIGPIPE`＋`EPIPE`、`MSG_NOSIGNAL` 尊重、`OS_STATUS_BROKEN_PIPE`=-32 追加）。
> - §3.6 robust list（`exit_robust_list` 相当）: pthread 終了時に自スレッド所有の robust mutex へ `FUTEX_OWNER_DIED` を立て 1 waiter を起床。
> - §3.7 非ブロッキング connect の失敗検出: ソケット層に O_NONBLOCK 追跡を新設（socket fd は file fd テーブル外のため FIONBIO/`F_SETFL` が従来 EFAULT で失敗していたのを修正）、非ブロッキング TCP `connect` は `EINPROGRESS` を返す、`SOCK_NONBLOCK`/`accept4`(288) 対応、非ブロッキング `recv` は空バッファ時に EOF ではなく `EAGAIN`。
> - §3.10 shebang（`#!`）: `process_execve` で1段だけ解釈しインタプリタへ再ターゲット。
> - §4 FD_CLOEXEC: `execve` が全 fd を閉じていた（stdio や Mojo 継承 fd まで消えていた）のを、`FD_CLOEXEC` 付きのみ閉じる `syscall_file_close_cloexec_for_pid` に変更。ソケットは per-fd cloexec ビットが無いため常時継承（POSIX 既定）。
> - §4 getifaddrs: I_libc に実装（`lo` 合成＋UDP connect/getsockname で主 IF アドレス検出）。
> - §4/§3.4 POSIX 層 `posix_io.c` の poll/select アイドルバックオフ上限を 100ms→16ms に縮小（真のイベント駆動化はスケジューラ制約で epoll と同じく不可）。
> - §9.2 auxv に `AT_HWCAP`（CPUID leaf1 EDX）/`AT_CLKTCK`（`timer_hz()`）を追加。
> - §9.2 イメージ配線: `WITH_GLIBC=1 make image` で `make glibc_image_stage` を挟み `/lib64`・`/usr/lib` をOSイメージへ同梱（既定はオフ＝重いビルドを既定経路に入れない、というドキュメント方針を維持）。
>
> 追記 2026-08-28（第3セッション、ユーザ指示で「B・C も実施」「ビルド済み Chromium をそのまま同梱」）:
> - **バケット B / COW fork**: 実装（`Memory_Main` に物理ページ参照カウント配列＋`pmm_page_ref_*`、`Paging_Main` に `paging_cow_clone_user_range`／`paging_handle_cow_fault`＋`PAGE_COW`(bit11)、`IDT_Main` の PF ハンドラに write フォルト時の COW フック、`process_clone_address_space` の Linux ABI 経路で COW clone→失敗時は従来の eager copy にフォールバック）。**`KERNEL_COW_FORK`（`kernel/config.h`）で切替、既定 0（無効）**。理由: PTE エイリアス＋SMP TLB コヒーレンシ＋物理アロケータを同時に触る変更で、この環境では QEMU 起動検証ができないため。`-DKERNEL_COW_FORK=1` でフルビルド・`-Werror` 通過は確認済み。有効化は実機/QEMU 検証後。
> - **バケット B / MAP_SHARED（ファイル）ライトバック**: `Syscall_LinuxCompat.c` に登録表（`g_linux_mshared`、96 エントリ、専用 spinlock）を追加。書込可能な `mmap(MAP_SHARED, <file>)` を記録し（fd を dup して保持）、`msync(2)`（番号 26、新規）と `munmap(2)` と プロセス終了時に領域内容を `linux_pwrite64` でファイルへ書き戻す。**プロセス間ライブ・コヒーレンシは無い**（page cache が無いため）—単一ライタの「mmap 経由のファイル書き込み」が成立するだけ。Chromium が依存する共有メモリ（memfd/`/dev/shm`）は tmpfs 実体で従来どおりコヒーレント。
> - **バケット C / タイムゾーン**: `EtcFS.c` に `/etc/localtime` を追加。バイト厳密に妥当な IANA `Etc/UTC` TZif（`/usr/share/zoneinfo/Etc/UTC` の 114 バイトコピー）を静的埋め込み。不正 TZif で glibc の tzset がクラッシュするという §4ter の懸念は「妥当な実物」なら回避できる。
> - **バケット C / フォント**: `EtcFS.c` に `/etc/fonts/fonts.conf`（fontconfig、generic family → "Noto Sans JP" マッピング、`<dir>/usr/share/fonts</dir>`）。`WITH_CHROME=1` のイメージ経路で実 TTF（`BootManager/Resource/Fonts/NotoSansJP-Regular.ttf`）を `/usr/share/fonts` にステージ。
> - **ビルド済み Chromium の同梱**: `Userland/Application/com.ImplusOS.chrome/`（新規、ビルドせず repo ルートの `chrome-linux/` をステージするだけ）。`WITH_CHROME=1 make image` で APP_DIRS に加わり `/Userland/com.ImplusOS.chrome/` に展開、`WITH_GLIBC=1` を強制、`INSTALL_DISK_IMAGE_SIZE_MB` を 1536 に拡大、Noto フォントを配置。`chrome-headless` ランチャ／`run.txt`／`README.md` を同梱。既定 `make` は不変（560MB を既定経路に入れない）。
> - **バケット C / P2 GUI**: プリビルドバイナリに対しては **`--headless=new` が唯一の現実的経路**（カスタム Ozone プラットフォームは Chromium 本体を再ビルドしないと足せない＝~100GB のソースツリーが必要でこの環境には無い）。ランチャは `--no-sandbox --disable-gpu --use-gl=swiftshader --no-zygote --single-process --headless=new` を既定にした。
>
> 追記 2026-08-28（第4セッション、ユーザ指示「Userland 起動 15 秒後に Chrome を自動起動」「WM の Applist にも追加」「実際に起動するかテスト」）:
> - **自動起動**: `Userland/Userland.c` の `_start` 末尾で WM/sysnotif 起動後に `sleep_ms(15000)` → `chrome-launch.ELF` を `process_spawn`（存在しなければ静かにスキップ）。
> - **WM Applist**: `apps.list` と `WM_Assets.c` の `default_apps[]` に「Chrome (headless)」を追加（`/Userland/com.ImplusOS.chrome/chrome-launch.ELF`、バッジ `CH`）。
> - **ネイティブランチャ**: `Userland/Application/com.ImplusOS.chrome/chrome-launch.c`（新規、ビルドする）。`process_spawn` は引数を 1 個しか渡せないため、この小さなネイティブ ELF が `execve("/Userland/com.ImplusOS.chrome/chrome", argv, envp)` でヘッドレスのフルコマンドライン＋環境（HOME/XDG_*=/dev/shm、LD_LIBRARY_PATH=/lib64:/usr/lib:/Userland/com.ImplusOS.chrome、LANG/LC_ALL=C、TZ=UTC、FONTCONFIG_PATH=/etc/fonts）を渡す。
> - **ELF ローダの前提バグを 2 件修正**（Chrome を読む前に必ず踏む）:
>   1. `PROCESS_ELF_MAX_SIZE` 20MiB→768MiB（Chrome は 465MB）。`vfs_file_t.size` が uint32 なので上限は 4GiB。
>   2. `ELF_Loader.c` の PT_LOAD 読み込みが `malloc(p_filesz)` でセグメント全体をカーネルヒープに確保していた（Chrome の ~200MB セグメントで即 OOM）。256KB のステージングバッファ経由でストリーミングするよう書き換え。
> - **PIE / 動的リンカのロードバイアス対応**（第4セッションで実装。ユーザ提供の起動ログ `elf_err=segment: vaddr out of range` を受けて）:
>   - `ELF_Loader.c` を `elf_load_image_biased(cr3, path, policy, bias_override, is_interp, out)` に内部リファクタ（公開 API `elf_loader_load_from_path` はラッパ）。
>   - `ET_DYN` かつ最下位 PT_LOAD が `USER_CODE_BASE` 未満（＝base 0 リンクの真の PIE）のとき、**メイン実行体を `USER_CODE_BASE`(0x40_0000_0000)**、**インタプリタ（ld.so）をコード領域上端の予約窓 `USER_CODE_LIMIT - 128MiB`(0x40_7800_0000)** にロードバイアスを付けて配置。全 PT_LOAD の `p_vaddr`／entry／phdr アドレスにバイアスを加算。既に高位アドレスにリンク済みの in-tree カスタム ld/.so（`com.ImplusOS.ldso` 等）はバイアス 0 のまま（回帰なし）。ET_EXEC ネイティブアプリも従来どおり。
>   - `elf_loaded_image_info_t` に `load_bias` / `interp_base` を追加。`initialize_elf_user_stack_ex` の auxv `AT_BASE` を `image_info->interp_base` に、`AT_ENTRY`/`AT_PHDR` は既にバイアス済みの値を使用。
>   - メイン画像がインタプリタ窓に食い込む場合は `main image overlaps interpreter window` で失敗。
> - **⚠️ 実起動の完走テストは依然不可**（この環境で QEMU 起動ができない）。これで ELF ローダは通過するはずだが、この先に (a) Chrome の静的画像 ~333MB を eager map する物理メモリ負荷、(b) ld.so → glibc → Chrome `main` で顕在化する Chrome 固有 syscall ギャップ、が続く見込み。`-DLINUX_SYSCALL_TRACE` ＋ シリアルログで逐次潰していく段階。

---

## 1. 現在の実装状況サマリ

### 1.1 アーキテクチャ（3層構造）

```
Chromium (Linux ネイティブ ELF, glibc 依存)
  │  syscall (番号は x86_64 Linux ネイティブ)
  ▼
Kernel/Core/syscall/Syscall_LinuxCompat.c   … Linux ディスパッチ (88 syscall 実装)
  ▼
Kernel ネイティブ API (VFS / Process / TCP / epoll / futex …)
```

- **ABI 判定**: `ELF_Loader.c` が `EI_OSABI==ELFOSABI_LINUX` または Linux エントリヒントで `linux_abi` を設定し、`Syscall_Dispatch.c` の `syscall_dispatch()` が `PROCESS_ABI_LINUX` なら `linux_syscall_dispatch()` へ分岐。
- **PT_INTERP (動的リンカ) 対応済み**、ET_EXEC/ET_DYN 両対応。
- **実装済みの主要 syscall** (Linux 番号): read/write/open/close、stat/fstat/lstat、lseek、mmap/mprotect/munmap/brk、rt_sigaction/sigprocmask、ioctl(FIONBIO/FIONREAD)、readv/writev、access、pipe、dup/dup2、nanosleep、getpid/getppid/gettid、socket 系(AF_INET+TCP のみ)、clone/fork/vfork、execve、exit/exit_group、wait4、kill/tkill/tgkill、uname("Linux 6.1.0-implus" 偽装)、fcntl、ftruncate/getcwd/chdir、rename/mkdir/rmdir/creat/unlink、gettimeofday/getrlimit、getuid 系(0 固定)、prctl/arch_prctl(FS)、setrlimit、time、futex(WAIT/WAKE/WAIT_BITSET)、getdents64、set_tid_address、clock_gettime(REALTIME/MONOTONIC)、epoll_*、openat(AT_FDCWD のみ)/newfstatat、set_robust_list/rseq、timerfd/eventfd/signalfd、epoll_create1、prlimit64、getcpu、getrandom、memfd_create。

### 1.2 既知の未実装・制限（調査で確認済み）

| 項目 | 現状 | 影響 |
|---|---|---|
| FD 上限 | `OS_CONFIG_FILE_MAX_FD=32` (config.h) / 最大 256 | **致命的**。Chromium は単プロセスで 50 FD 超を開く |
| プロセス上限 | `OS_CONFIG_PROCESS_MAX_COUNT=64` (最大 256) | マルチプロセス Chromium が上限に近い |
| mremap | `-38 ENOSYS` スタブ | **致命的**。glibc の realloc/malloc arena が使用 |
| RLIMIT_AS | prlimit64 が **256MB** を報告 | glibc のメモリ計算が狂い mmap 失敗に繋がる |
| epoll_wait | 登録エントリを**無条件即時返却** (timeout 無視) | **致命的**。Chromium のイベントループがビジーループ化 |
| ファイル mmap | 読み込み専用のコピーのみ。MAP_SHARED/ライトバック無し | memfd/shm での共有メモリが機能しない |
| socket | **AF_INET+SOCK_STREAM のみ**。UDP / AF_UNIX 不可 | DNS 直アクセス (UDP) と Mojo IPC (AF_UNIX) が不可能 |
| シグナル配信 | シグナルフレームに **RIP のみ保存**、RDI=signum のみ。SA_SIGINFO/altstack 非対応 | glibc のハンドラが正しく復帰できず誤動作 |
| futex | WAIT/WAKE/WAIT_BITSET のみ。WAKE_OP/REQUEUE/PI 非対応 | pthread_cond の broadcast 等が機能しない |
| clock_gettime | REALTIME / MONOTONIC のみ | MONOTONIC_RAW/BOOTTIME/CPUTIME_ID が ENOSYS |
| madvise / mincore / statx / sysinfo / statfs | 未実装 (ENOSYS) | glibc/V8/Chromium が多用。statx は glibc の stat 実装の要 |
| sched_getaffinity | 未実装 | Chromium の CPU 数検出が失敗し worker 数が狂う |
| fork | **COW 未実装 (全ページ実コピー)** | zygote→renderer の多重フォークが極端に低速 |
| スレッド | スタック 1MB、上限 256、FS base は親からコピー (clone の tls 引数無視) | Chromium の多数スレッドで上限抵触の恐れ |
| /proc, /dev, devfs | **存在しない** | `/proc/self/maps` `/proc/meminfo` `/dev/urandom` 等を読めない |
| /etc (/etc/hosts, resolv.conf, localtime, fonts) | 提供なし | 独自 DNS resolver・時間・フォント解決ができない |
| vDSO / auxv 完全性 | AT_SYSINFO_EHDR 有無不明。auxv の完全性未検証 | Chromium は vDSO 無しでも動作可。要検証 |
| libc / 動的リンカ | 自前 libc は最小構成。Chromium は glibc 前提 | 動的リンク Chromium は ld-linux + glibc が必要 |

---

## 2. 実行方針の前提（TODO を読む前に）

Chromium を ImplusOS で"実用的に"動かすための現実的な方針:

1. **`--no-sandbox --disable-gpu` を必須**とする（seccomp/sandbox/GPU は対象外）。
2. 初段は **静的リンク版 Chromium**（glibc や ld-linux のポーティングを回避）。動的リンク版を目指す場合は `libc/I_libc` とは別に **glibc のポーティング** が別プロジェクトとして必要。
3. GUI は 2 段階: まず **headless モード** (`--headless=new`) で ABI を安定化させ、その後に **Ozone カスタムプラットフォーム** で画面描画を実現。
4. ネットワークは IPv4 のみ。IPv6 は明示的に無効化して運用。

---

## 3. P0 — ブート（起動）の可否を左右する必須項目

### 3.1 リソース上限の拡張

- [x] **FD テーブル拡張**: `OS_CONFIG_FILE_MAX_FD` を 32→256 に引き上げ済み（`Kernel/include/kernel/config.h`）。`prlimit64`/`getrlimit` の `RLIMIT_NOFILE` 報告値も実値(256)に連動させた（`Syscall_LinuxCompat.c`）。`Syscall_Socket.c` の `SOCKET_FD_BASE`(512) はこのグローバル fd テーブルと衝突しないことをコメントで明記。`Userland/POSIX` 側は元々 1024 エントリで対応済み。
- [x] **プロセス上限引き上げ**: `OS_CONFIG_PROCESS_MAX_COUNT` を 64→256 に引き上げ済み。`g_process_spaces_static`/`WaitQueue` 等は元々この定数でサイズを取っているため追随。
- [x] **スレッド上限・スタック拡張**: スレッドスタックを 1MB→8MB に拡張（`ProcessManager_Create.c PROCESS_THREAD_STACK_SIZE`）。heap 直下からの割当設計はレビュー済み（プロセス上限256・8MBスタックでも heap 領域(約29.75GB)に対して十分小さく安全）。TLS キー数の引き上げは未着手。
- [x] **RLIMIT_AS の実態整合**: 実効的な mmap 上限が無いため `RLIMIT_AS`/`RLIMIT_DATA`/`RLIMIT_RSS`/`RLIMIT_MEMLOCK` を `RLIM_INFINITY` 報告に変更。`prlimit64` の setrlimit 方向(`new_limit != 0`)も従来は `ENOTSUP` で失敗していたのを黙って受理するよう修正（glibc/Chromium 起動時の `setrlimit(RLIMIT_CORE, 0)` 等を通すため）。

### 3.2 メモリ管理

- [x] **mremap の実装**: `Syscall_VM.c` に実装(既定は「新規領域確保+コピー+旧領域解放」方式)。理由はコード内コメント参照(既存の bump/free-list アロケータに「その場で伸長」する安全な手段が無いため)。**本セッションで MREMAP_FIXED / MREMAP_DONTUNMAP も実装**（`syscall_vm_mremap5`、§4 参照）。
- [x] **MAP_SHARED / ファイル mmap のライトバック**: 第3セッションで実装（登録表＋`msync`(26)／`munmap`／exit での `linux_pwrite64` 書き戻し）。**プロセス間ライブ・コヒーレンシは無し**（page cache 不在）。匿名共有（memfd/`/dev/shm`）は従来どおり tmpfs 実体で機能。
- [x] **madvise**: no-op で 0 を返す実装(`Syscall_LinuxCompat.c` linux_madvise)。
- [x] **mincore**: 全ページ resident(1)として返す簡易実装。
- [x]/[~] **COW fork**: 第3セッションで実装。ただし **`KERNEL_COW_FORK`（`kernel/config.h`）既定 0 で無効**（詳細は §4quater / 冒頭追記）。有効時は物理ページ参照カウント＋`PAGE_COW` PTE＋PF ハンドラ COW フック＋`process_clone_address_space` の COW clone（失敗時 eager copy フォールバック）。`-DKERNEL_COW_FORK=1` でのフルビルド／`-Werror` 通過は確認済み、起動検証は未。

### 3.3 疑似ファイルシステム（procfs / devfs / tmpfs）

- [x] **devfs (/dev)**: `Kernel/Core/vfs/DevFS.c` を追加し `/dev/null,zero,full,urandom,random,tty` を実装、`/dev` prefix で `vfs_mount`。stat() は `S_IFCHR` を返すよう `Syscall_LinuxCompat.c` を調整。既知の制約: 各デバイスは `vfs_file_t.size` が uint32_t のため疑似的に 64MB の「ストリーム長」を持つ near-unbounded 実装（真の無限ストリームではない）。`/dev/urandom` は 4KB 未満の短い read が同一 fd 内の同じ read-through キャッシュ窓に収まると同一乱数バイト列を返し得る（`Syscall_File.c` の汎用キャッシュ層に起因、既知の制限としてコード内コメントに明記）。
- [x] **tmpfs + /dev/shm**: `Kernel/Core/vfs/TmpFS.c` を追加、`/dev/shm` にマウント。malloc/realloc ベースのフラットな名前空間（サブディレクトリ非対応）。
- [x] **procfs (/proc)**: `Kernel/Core/vfs/ProcFS.c` を追加。実装済み: `/proc/self/{maps,status,stat,cmdline}`、`/proc/self/exe` と `/proc/self/fd/<n>`（`readlink`/`readlinkat` 経由、fd/N は元パスを保持していないため `anon_inode:[...]` プレースホルダ）、`/proc/meminfo`, `/proc/cpuinfo`, `/proc/stat`, `/proc/version`, `/proc/sys/kernel/random/boot_id`, `/proc/sys/vm/overcommit_memory`。制約: 自プロセス(`self`または自分のpid)のみ対応、他プロセスの `/proc/<pid>/...` は非対応。`/proc` ディレクトリ自体の列挙(opendir/readdir)は未実装。内容は open() 時に一度生成される（Linuxのような「readのたびに再生成」ではない）。
- [x] **/proc/self/maps の実装**: `ProcessManager.h` の固定ユーザ空間レイアウト定数(`USER_CODE_BASE/LIMIT`, `USER_HEAP_BASE`, `USER_STACK_BASE/TOP`)と `process_get_heap_cursor()` から3行（code/heap/stack）を合成。新規ヘルパ `process_count_threads()` を追加し `/proc/self/status` の `Threads:` に使用。

### 3.4 epoll の実イベント駆動化（最優先）

- [x] `syscall_epoll_wait` を実データ駆動に変更(`syscall_epoll_wait_ex`として全面書き換え)。既存の `syscall_file_poll()`/`syscall_socket_poll()`(パイプ/ファイル/timerfd/memfd/signalfd/ソケットの実際のreadiness、UDPの実装追加分含む)を使って毎回本物の状態を計算するようにした。**重要な設計上の制約**: 本カーネルのスケジューラは「syscallの途中でブロックし後で"そのC関数の続き"から再開する」ことができない(`process_run_next_on_current_cpu()`のenter_user_modeは一方向ジャンプであり関数呼び出しとして戻ってこない)ため、真の「fdのwait queueに登録し起床させる」方式は不可能と判断。代わりに、何も準備できていない場合は `process_sleep_current_ms(8ms)` + `request_switch` で実際にCPUを他プロセスへ譲り(以前の `hal_cpu_pause()` によるビジーループを解消)、要求されたtimeoutより早く0を返す設計とした。これは本物のイベントループがEINTRによる早期リターンを許容する設計になっている(はず)ことを前提とした安全側の妥協。詳細な設計判断はコード冒頭のコメント参照。
- [x] eventfd の read/write/close を新規実装(元々未実装で読み書きできなかった)。EFD_SEMAPHORE対応、非ブロッキング専用(カウンタ0時はEAGAIN)。
- [x] EPOLLET/EPOLLOUT/EPOLLERR/EPOLLHUP: EPOLLOUT/ERR/HUPは実装。**EPOLLETを実エッジトリガ化**(`epoll_entry_t.last_ready` に前回報告した readiness マスクを保持し、`ready & ~last_ready` の立ち上がりビットのみ返す。`EPOLLERR`/`EPOLLHUP` は Linux 同様に常時報告。`EPOLL_CTL_MOD` と再 `ADD` で `last_ready` をリセットしエッジを再武装)。ドレインまで読み続ける正しい ET 消費者を前提とする点は不変。
- [x] 非ブロッキング socket + `poll()`(POSIX層 `posix_io.c`): アイドルバックオフ上限を 100ms→16ms に縮小し新規 readiness の検出遅延を短縮。真のイベント駆動化はスケジューラ制約(syscall 途中でブロックして再開不可)で epoll と同じく不可、既存のポーリング方式を維持。

### 3.5 プロセス・シグナル

- [x] **シグナルフレームの完全化**: 全面書き換え(`write_signal_frame_locked` を新設)。実装済み: SA_SIGINFO 時の siginfo_t+ucontext_t 構築(SIGSEGVでcr2/si_addrを伝搬)、ハンドラ呼び出し前のsigmask退避(sa_mask+自シグナルのブロック、SA_NODEFER考慮)とrt_sigreturnでの復元、SA_ONSTACK(sigaltstack)、SA_RESETHAND。SA_RESTARTは値の保存のみで、システムコール自動再開ロジック自体は未実装(既知の制限)。**同期的なページフォルト由来のSIGSEGV**もこの経路で配送されるようになった(下記参照)。既知の制約: uc_mcontextのcs/gs/fs/ss/fpstateは常に0(セグメントレジスタとFPU/XSAVE状態は未捕捉)。restorer(sa_restorer)未指定の場合は旧来どおり「ハンドラの`ret`で直接元の命令へ戻る」簡易フォールバックになる(glibcは常にrestorerを設定するため実害は小さい想定)。
- [x] **rt_sigreturn の完全実装**: `process_signal_rt_sigreturn()` を追加、`LINUX_SYS_RT_SIGRETURN`(15)にディスパッチ。ucontext全体(GPレジスタ+RIP+RFLAGS+RSP+sigmask)を復元。**重要な設計上の注意**: この関数はスケジューラ境界を経由せず「発行中の同一syscall」の生フレームを直接操作するため、`proc->saved_rsp`/`saved_user_rsp`(スケジューラ境界でのみ更新される)ではなく、ディスパッチャから渡される生の`saved_rsp`と`syscall_get_user_rsp()`/`syscall_set_user_rsp()`を使う(`linux_clone`と同じパターン)。
- [x] **sigaltstack(2)**: `process_sigaltstack()` を追加、`LINUX_SYS_SIGALTSTACK`(131)にディスパッチ。スレッドごとに独立(スレッド生成時は継承せずリセット)。
- [x] **SIGSEGV配送の新設**: `Arch/x86_64/cpu/IDT_Main.c` のページフォルトハンドラに `process_signal_deliver_fault_now()` 呼び出しを追加(`ProcessManager_Create.c`)。ISRの生レジスタ配列(SAVE_REGS)とCPU割り込みフレームを直接書き換えてハンドラへ`iretq`で復帰する方式(アセンブリ自体は無変更)。ハンドラ未登録の場合は従来どおりプロセス終了。SIGBUSは未配送(x86のページフォルトはSIGSEGVにのみ対応付け)。
- [x] SIGCHLD の確実な配信と wait4 の status 変換: `process_t` に `exit_by_signal`/`exit_term_signal` を追加し、`process_waitpid_ex()`(既存 `process_waitpid` は薄いラッパ)で終了原因を返す。`linux_wait4`/`linux_waitid` を POSIX wait ステータス(`WIFEXITED`/`WIFSIGNALED`/`WTERMSIG`)へ正しく符号化(従来は生の `exit_status` をそのまま返しており終了コード 1 が「シグナル 1 で kill」と誤解釈されていた)。SIGCHLD は**親がハンドラを設置している場合のみ**配送(既定 disposition の SIGCHLD を pending にすると本カーネルの「未ハンドラ=致命」ロジックで親が死ぬため)。PF 由来の SIGSEGV も `process_exit_current_signaled(11)` 経由で `WTERMSIG==SIGSEGV` を報告。
- [x] SIGPIPE の既定動作: reader 不在のパイプへの `write` と peer 切断済み TCP への `send` で、現プロセスへ `SIGPIPE` を post しつつ `EPIPE`(新 `OS_STATUS_BROKEN_PIPE`=-32)を返す。`MSG_NOSIGNAL`(0x4000) 指定時はシグナルを抑止。既定 disposition なら pending-signal 経路がプロセスを終了、`SIG_IGN`/ハンドラなら `EPIPE` のみ(POSIX 準拠)。

### 3.6 スレッド・TLS・futex

- [x] **futex の拡張** (`Kernel/Core/syscall/Syscall_Futex.c`):
  - [x] `FUTEX_WAKE_OP` — 実装済み(オペコード/比較のデコードとuaddr2への原子更新)。
  - [x] `FUTEX_REQUEUE` / `FUTEX_CMP_REQUEUE` — 実装済み。
  - [x] `FUTEX_PRIVATE_FLAG` の受理と無視 — 元々 `FUTEX_CMD_MASK` で自動的にマスクされ実装済みだった(見落とし修正: doc記載の未実装は誤りだった)。
  - [x] `FUTEX_LOCK_PI`/`FUTEX_UNLOCK_PI`/`FUTEX_TRYLOCK_PI`/`FUTEX_LOCK_PI2` — 実装済み（`syscall_futex_lock_pi`/`syscall_futex_unlock_pi`）。**優先度継承そのものは無い**（RR スケジューラのため）。実装したのは PI の**所有権プロトコル**のみ: futex ワードに所有者 TID（下位30bit）＋`FUTEX_WAITERS`、`LOCK_PI` は所有者になるまでブロック、`UNLOCK_PI` はキュー先頭の待機者へ所有権を直接受け渡してから起床。spurious wake 時は `EAGAIN` を返して glibc に syscall 再試行させる。`timeout`（PI は絶対 timespec）は未武装＝`pthread_mutex_timedlock` の PI 版は無期限ブロック扱い（既知の制限、WAIT の timeout 扱いと同方針）。dispatch 側で cmd 6/13 も `request_switch` 対象に追加。
  - [x] (doc未記載だったが追加) `FUTEX_WAKE_BITSET` も実装。
- [x] **clone の CLONE_SETTLS 対応**: `process_create_thread_ex()` を新設し、TLS(fs_base)をスレッドが `PROCESS_STATE_READY` になる**前**(同一ロック区間内)に設定するようにした。SMP環境でスレッド作成直後に他CPUがすぐにディスパッチしうるため、ロック外での事後設定はレースになると判断し採用しなかった。
- [x] **robust list との整合**: `process_thread_exit_current()` に `exit_robust_list(2)` 相当を実装。終了するスレッドの `robust_list_head` から `struct robust_list_head { next; futex_offset; pending; }` を辿り(上限 2048 エントリ)、各エントリ+`list_op_pending` について `futex_word = entry + futex_offset` を読み、下位30bit が自 TID なら `(word & FUTEX_WAITERS) | FUTEX_OWNER_DIED` に書き換えて `WAITERS` があれば 1 waiter を `FUTEX_WAKE`。これで兄弟スレッドの glibc `pthread_mutex_lock` が `EOWNERDEAD` を得て mutex を回収できる。スレッド自身のアドレス空間内で動くため `copy_from/to_user` で直接読み書き。致命シグナルでの異常終了経路(スレッドコンテキスト外)は未対応。

### 3.7 ネットワーク

- [x] **UDP ソケット**: `Syscall_Socket.c` に `SOCKET_TYPE_DGRAM` を追加(既存の `Network/udp/UDP.c` のユーザ向けAPIを利用)。`socket(AF_INET,SOCK_DGRAM)`/`bind`/`connect`(デフォルト送信先の記録のみ、ハンドシェイク無し)/`sendto`/`recvfrom`(実際の送信元IP/ポートを`udp_user_recv`のヘッダから復元して報告)/`send`/`recv`(connect済みの場合のデフォルト送信先を使用)/`close`(UDPバインディング解放)を実装。`FIONREAD`/`poll`もUDPキューの件数を反映するよう更新。新規ヘルパ `udp_user_available()` を `UDP.c` に追加。
- [x] **AF_UNIX ソケット**: 調査の結果、`Kernel/Drivers/Client/UnixSocket/UnixSocket.c` に SCM_RIGHTS(fd受け渡し)対応込みの実装が**既に存在**していたが、ネイティブABI経由のみでLinux ABI側には未接続だった状態を発見・修正。`socket(AF_UNIX,...)`/`bind`/`connect`/`listen`/`accept`/`send`/`recv`/`close` をfd範囲(`UNIX_SOCK_FD_BASE`=0x8000)またはsockaddrのfamilyで振り分けて接続。新規 `unix_socket_pair()` を追加し `socketpair`(Linux番号53、AF_UNIX)を実装。**副次的に発見・修正した既存バグ**: Linux ABIの`close()`が`syscall_socket_close()`(TCP/UDP)を一度も呼んでおらず、ソケットfdをclose()してもTCP接続もソケットテーブルスロットも解放されていなかった(プロセス終了時の一括クリーンアップでのみ解放)。既知の制約: UnixSocket.c自体に所有プロセスチェックが無い(fd番号さえ分かれば他プロセスのUnixソケットを操作できてしまう、既存コードの制約でありこの変更では未修正)。`sendmsg`/`recvmsg`(Linux番号46/47、SCM_RIGHTS込み)もAF_UNIX向けに接続した。接続時に**既存のバグを発見・修正**: `unix_socket_sendmsg`/`recvmsg`内部の`msghdr`パース用ローカル構造体が `msg_iovlen`/`msg_controllen` を`uint32_t`(4バイト)と誤って仮定していたが、実際のglibc ABI(x86-64)ではどちらも`size_t`(8バイト)であり、そのままでは実際のglibc生成`struct msghdr`を渡すとフィールドが4バイトずれて誤読される状態だった。オフセット/サイズを実ABIに合わせて修正済み。トップレベルの`msghdr`構造体自体はサイズ検証しているが、`msg_iov`/`msg_control`が指す先のバッファ検証は無い(`unix_socket_sendmsg`/`recvmsg`側の既存の制約、今回は未修正)。
- [x] **非ブロッキング connect の失敗検出**: `tcp_connect` は実は非ブロッキング(SYN を撃って即 return、状態は `SYN_SENT`)。ソケット層に O_NONBLOCK 追跡を新設(`kernel_socket_t.nonblocking`)—socket fd は `FILE_MAX_FD`(256)外の `SOCKET_FD_BASE`(512)帯にあるため `syscall_file_get_status_flags` が常に EFAULT を返し、**従来 `ioctl(FIONBIO)`/`fcntl(F_SETFL)` がソケットに対して失敗していた**のを修正(`syscall_socket_fd_in_range`/`syscall_socket_set_nonblocking`/`syscall_socket_get_status_flags`)。非ブロッキング TCP `connect` は `EINPROGRESS`(-115) を返し、以後 `tcp_poll` が `SYN_SENT` 中は POLLOUT を返さず/`CLOSED` で POLLERR、`SO_ERROR` が `ECONNREFUSED`/`ETIMEDOUT` を合成(既存)。`SOCK_NONBLOCK`(socket 生成時) と `accept4`(288, `SOCK_NONBLOCK` 適用) を実装。非ブロッキング `recv` は接続が生きたまま空バッファのとき `0`(EOF 誤認) ではなく `EAGAIN` を返す(`tcp_poll` で FIN 未着を判定)。同様に peer 切断済み `send` は EIO ではなく `EPIPE`。
- [x] IPv6 は **明示的に無視**(EAFNOSUPPORT)。`linux_socket_create`/`linux_copy_sockaddr_in`/`linux_copy_sockaddr_un` は AF_INET/AF_UNIX 以外の family を全て `EAFNOSUPPORT` で弾く（AF_INET6=10 含む）。Chromium 側は `--host-resolver-rules` 等で IPv4 のみ構成。

### 3.8 時間系

- [x] **clock_gettime のクロック追加** (`Syscall_Clock.c`): CLOCK_MONOTONIC_RAW / CLOCK_BOOTTIME / CLOCK_PROCESS_CPUTIME_ID / CLOCK_THREAD_CPUTIME_ID / CLOCK_REALTIME_COARSE / CLOCK_MONOTONIC_COARSE を追加。MONOTONIC 系は全て同一の tick ベース（`timer_ticks()/timer_hz()`）を共有し、BOOTTIME は suspend 状態が無いため MONOTONIC と一致。CPUTIME 系は per-task 課金機構が無いため MONOTONIC で近似（V8/glibc は相対デルタにしか使わないため許容）。`clock_getres` も全クロックで tick 分解能を返すよう更新。
- [x] CLOCK_MONOTONIC の基準を tick 起点に統一。`Syscall_Clock.c` の MONOTONIC 計算・`linux_clock_nanosleep`・既存の timerfd/nanosleep が全て同じ `timer_ticks()` を参照するようにし、相互にドリフトしないことをコード内コメントで明記。`gettimeofday`(RTC 起点) は REALTIME 側なので別系統のままで正しい。

### 3.9 その他必須 syscall

- [x] **statx (Linux 番号 332)**: `struct statx` 相当(256バイト、Linux uapi と同一レイアウト)を実装。`AT_EMPTY_PATH`+fd 経由(fstat相当)と `AT_FDCWD`+path 経由の両方に対応。
- [x] **sysinfo**: ⚠️ TODO記載の番号(153)は誤り、正しい x86_64 syscall番号は **99**。`get_total_memory_pages()`/`get_free_memory()` から totalram/freeram、`timer_ticks()` から uptime を報告。
- [x] **statfs (137) / fstatfs (138)**: 簡易実装（tmpfs相当の固定値 + 空きメモリから概算のブロック数）。`statvfs` はlibc側でstatfsをラップする想定のため未着手。
- [x] **sched_getaffinity (204) / sched_setaffinity (203)**: `smp_get_cpu_count()` から CPU 数を反映した affinity mask を返す。setaffinity は要求を受理するのみ(実際のCPUピン留めは非対応)。
- [x] **sendfile (40)**: `syscall_file_read`/`syscall_file_write` を使ったユーザ空間非経由のコピーループとして実装。
- [x] **getitimer / setitimer (36/38)**: SIGALRM配信機構が無いため「常にdisarm」を返す/受理するのみのスタブ。実際のインターバルタイマ配信は未実装(既知の制限)。
- [x] **readlink (89) / readlinkat (267)**: `/proc/self/exe` と `/proc/self/fd/<n>` の解決に対応（procfs実装と同時に追加、TODO原文にはなかったが密結合のため本ステップで実施）。

### 3.10 実行環境（ファイル類・起動経路）

- [x]/[ ] **動的リンク対応**: 調査の結果、`ELF_Loader.c` の PT_INTERP 処理(インタプリタELFを`elf_loader_load_from_path`で再帰ロードし、そのentryをプロセスentryとして採用)は**既に完全に機能する状態**であることを確認。共有ライブラリの検索・mmap・シンボル解決は ld.so 自身がユーザ空間で(本セッションで整備した openat/mmap(MAP_FIXED込み)/mprotect/brk 等の既存syscallを使って)行うものであり、カーネル側に追加のコード変更は不要と判断した。残る作業は「実物の `ld-linux-x86-64.so.2`/`libc.so.6` バイナリをブートイメージに配置する」というパッケージング作業であり、これはコード変更ではなく外部バイナリの入手・ライセンス上の判断(配布可否)が絡むため、ユーザーの判断を要するとして本セッションでは見送った。
- [x] **execve の改善(一部)**: 相対パスの解決(CWD基準)は既に実装済みで確認した。**PATH探索は実はカーネルの責務ではない**(Linuxの生execve(2)もPATH探索をしない。execvp()がユーザ空間のlibcでPATH探索してから素のexecve()を呼ぶ設計であり、doc記載は誤解を含んでいた可能性がある)。argv/envpのサイズ上限は`EXECVE_STRTOTAL_MAX`/`EXECVE_ARG_MAX`で既にガードされていることを確認。
- [x] `#!`(shebang)解釈: `process_execve` で対象ファイル先頭 2 バイトが `#!` なら 1 行目からインタプリタ(＋任意の 1 引数、Linux 同様に空白以降を丸ごと 1 引数)を取り、`argv` を `[interp, (arg,) script_path, 元 argv[1..]]` に組み替えてインタプリタへ再ターゲット。解釈は 1 段のみ(`#!/bin/sh` → `/bin/sh` が ELF、の典型ケース。ネストしたスクリプトインタプリタは ELF ローダの「exec format error」になる)。Chromium 本体には無関係だが busybox/dash 系ヘルパスクリプトで有用。
- [x] **/etc/hosts と /etc/resolv.conf**: `Kernel/Core/vfs/EtcFS.c` を追加(動的生成方式、doc記載の代替案を採用)。`/etc` prefix でマウント。`OS_CONFIG_NET_IPV4_ADDR`/`GATEWAY`(kernel/config.h)から実際の設定値を反映。resolv.confのnameserverはgateway(デフォルト10.0.2.2 = QEMU usermodeネットワーキングの内蔵DNSプロキシ)を使用。
- [x] **フォント**: `EtcFS.c` に `/etc/fonts/fonts.conf`（fontconfig、generic family → "Noto Sans JP"、`<dir>/usr/share/fonts</dir>`）。実 TTF は `BootManager/Resource/Fonts/NotoSansJP-Regular.ttf`（5.4MB、Latin+JP）を `WITH_CHROME=1` イメージ経路で `/usr/share/fonts` に配置。Skia/Chromium の generic family 解決に十分。CJK 以外の広範なカバレッジが要る場合は追加フォントを同ディレクトリに置くだけ。
- [x] **タイムゾーン**: `EtcFS.c` に `/etc/localtime` を追加。バイト厳密に妥当な IANA `Etc/UTC` TZif（システムの `/usr/share/zoneinfo/Etc/UTC` の 114 バイトそのまま）を静的埋め込みしたので、glibc の tzset / ICU が local zone を UTC に解決できる。他ゾーンが必要なら同じ方式で該当 TZif を足す。

---

## 4. P1 — 実用レベルの安定動作に必要な高優先項目

- [ ] **inotify** (Linux 番号 253-255): `Syscall_LinuxCompat.c` に **スタブ実装済み**（`inotify_init`/`inotify_init1` は空マスクの signalfd（＝常に readiness=false で既存の poll/epoll 配管に乗るディスクリプタ）を返し、`inotify_add_watch` は単調増加の wd を返す、`inotify_rm_watch` は 0）。実イベント配送は無いので Chromium の `FilePathWatcher` はイベントを一切受け取らず、liveness が必要な箇所は手動ポーリングにフォールバックする。真のファイル変更通知は VFS レイヤの hook が必要で未着手。
- [x] **getsockname / getpeername の完全性**: `Syscall_LinuxCompat.c` `linux_getsockname_common` を新設し番号 51/52 に接続。`syscall_socket_get_info()` から local/remote の IP/ポートを引いて `sockaddr_in` を構築、`addrlen` の切り詰めにも対応。AF_UNIX の fd は family のみの無名アドレス（`sun_path` 長 0）を返す（socketpair/未bind の Unix ソケットに対する Linux の挙動と一致）。`getifaddrs` は libc 側（`posix.c`）の責務のため未着手。
- [x] **waitid**（番号 247）: `linux_waitid` を実装。`process_waitpid()` にマップし、`siginfo_t`(128B) に si_signo=SIGCHLD / si_code=CLD_EXITED|CLD_KILLED|CLD_STOPPED / si_pid / si_status を構築。P_ALL/P_PID に対応（P_PGID は best-effort で P_ALL 相当）。WNOHANG で対象なしのときは siginfo をゼロ埋めして成功を返す（POSIX 準拠）。`wait4` の rusage 出力は引き続きゼロ無視（`getrusage`(98) 自体はゼロ埋め実装を追加）。
- [ ] **プロセスグループ/セッション** (setpgid/setsid/getpgrp/getpgid/getsid): **実装済み（簡易モデル）**。セッション/プロセスグループの追跡機構が無いため「全プロセスが自分自身のグループ長／セッション長」としてモデル化：getpgid/getpgrp/getsid は自 pid を返し、setpgid は 0 を受理、setsid は自 pid を返す。sandbox 無効前提では十分。
- [x] **prctl 拡張**: `linux_prctl_ext` を追加し、`linux_prctl` の default から委譲。PR_SET_DUMPABLE/PR_GET_DUMPABLE、PR_SET_PDEATHSIG/PR_GET_PDEATHSIG、PR_SET_KEEPCAPS、PR_CAPBSET_READ、PR_SET_NO_NEW_PRIVS/PR_GET_NO_NEW_PRIVS、PR_SET_SECCOMP（sandbox 対象外につき成功を偽装）、PR_GET_SECCOMP（0=非seccomp）、PR_SET_TIMERSLACK、PR_SET_THP_DISABLE、PR_SET_PTRACER、PR_SET_VMA 等を受理（no-op で 0）。未知の option は従来どおり `ENOTSUP`。
- [ ] **ioctl TCGETS/TCSETS/TIOCGWINSZ**: **実装済み**。`linux_ioctl_tty` を新設し `syscall_ioctl_ex` の先頭で分岐。TCGETS/TCSETS*/TCGETA 系/TCFLSH/TIOCSCTTY/TIOCGPTN 等は **ENOTTY** を返す（＝`isatty()`/`base::IsTerminal()` が期待する非 tty シグナル）。TIOCGWINSZ は fd 0/1/2 に対してのみ 80x24 の `winsize` を返し、それ以外は ENOTTY。TIOCGPGRP/TIOCSPGRP も ENOTTY。
- [ ] **personality** (ADDR_NO_RANDOMIZE): **実装済み**。番号 135 は常に 0（PER_LINUX）を返し、ADDR_NO_RANDOMIZE 等の指定は黙って受理（切り替える ASLR が存在しない）。
- [x] **mremap の MREMAP_FIXED / MREMAP_DONTUNMAP**: `Syscall_VM.c` に `syscall_vm_mremap5`（第5引数 new_address 付き）を新設し、番号 25 のディスパッチを arg5 込みに変更。MREMAP_FIXED は `paging_map_user_range_alloc()` で指定アドレスに直接フレームを割り当て→重なり分をコピー→（MREMAP_DONTUNMAP 未指定なら）旧領域を解放。MREMAP_DONTUNMAP 単独（FIXED 無し）は通常アロケータで移設しつつ旧マッピングを保持。
- [x] **recvmmsg / sendmmsg**（番号 299/307）: `linux_sendmmsg`/`linux_recvmmsg` を実装。`sendmsg`/`recvmsg` と同じく **AF_UNIX 限定**（Mojo IPC）。`mmsghdr[]`(x86-64 で 64B/要素、msghdr 56B + msg_len) を走査して各要素の `msg_len` を書き戻す。
- [x] **/proc/sys/kernel/threads-max、pid_max 等**の静的報告: `ProcFS.c` に `g_procfs_static_scalars[]` テーブルを追加。threads-max / pid_max / osrelease / ostype / hostname / cap_last_cap / ngroups_max / yama.ptrace_scope / vm.max_map_count / vm.overcommit_ratio / vm.mmap_min_addr / net.core.somaxconn / fs.pipe-max-size / fs.file-max / fs.nr_open を報告。加えて `/proc/uptime`、`/proc/loadavg`、`/proc/filesystems`、`/proc/self/limits`、`/proc/self/oom_score{,_adj}` を追加。
- [x] **FD_CLOEXEC の完全な伝播**: `pipe2`/`dup3`/`fcntl(F_DUPFD_CLOEXEC)` は cloexec フラグを設定済み。**重大バグを発見・修正**: `process_execve` が `syscall_file_close_all_for_pid` を呼び *全* fd(stdin/stdout/stderr や Mojo で渡された pipe/socket まで)を閉じていたため、exec 後のプロセスは fd を一切継承できなかった。`FD_CLOEXEC` 付きのみ閉じる `syscall_file_close_cloexec_for_pid` に差し替え(ディレクトリハンドルは POSIX opendir が暗黙 cloexec なので従来どおり全クローズ)。ソケットは per-fd cloexec ビットが無いため常時継承(POSIX 既定。exec 失敗時は `process_exit_current` が一括解放)。
- [x] **poll/select の精度向上**: `posix_io.c` の poll/select アイドルバックオフ上限を 100ms→16ms に縮小し新規 readiness の検出遅延を改善。完全なイベント駆動化はスケジューラ制約で epoll と同じく不可のため、既存の適応バックオフ・ポーリング方式を維持(真の wait queue 化は §3.4 の設計注記参照)。
- [x] **getifaddrs の完全性**: I_libc `posix.c` に実装。`lo`(127.0.0.1/8) を合成し、主 IPv4 IF は「UDP ソケットを任意の公開アドレスへ connect → `getsockname`」の定石でアドレスを検出(データグラムなのでパケットは飛ばない)、`/24` 前提で netmask を付与。カーネル側 NIC 列挙 API は不要。`getsockname/getpeername` は前セッションで実装済み。

### 4bis. 本セッションで追加した glibc 実行に必須の周辺 syscall（TODO 原文外だが密結合）

`Syscall_LinuxCompat.c` に以下を追加（いずれも既存プリミティブの薄いラッパか定義済みの no-op）。目的は「実 glibc でリンクされたバイナリ（busybox/dash 等）を ENOSYS で即死させない」こと（section 6 P3 のベンチ前提）:

- **pread64 / pwrite64**（17/18）— fd オフセットを変えずに読み書き（glibc stdio・ld.so・Chromium が多用）。
- **pipe2 / dup3**（293/292）— O_CLOEXEC/O_NONBLOCK 対応（glibc の `pipe()`/`dup2()` は実際にはこれらを呼ぶ）。
- **clock_nanosleep**（230）— TIMER_ABSTIME 対応込みで tick sleep にマップ。
- **epoll_pwait / epoll_pwait2**（281/441）— epoll_wait と同一（sigmask は同期シグナル配送のため無視）。pwait2 の timespec timeout は ms に変換。
- **faccessat / faccessat2**（269/439）— AT_FDCWD 限定で `syscall_access` にマップ（glibc の `access()` はこれ経由）。
- **mkdirat / unlinkat / renameat / renameat2**（258/263/264/316）— AT_FDCWD 限定。
- **fsync / fdatasync / syncfs / sync / flock / fadvise64**（74/75/306/162/73/221）— 0 を返す。
- **mlock / munlock / mlockall / munlockall / mlock2**（149-152/325）— 0（swap/reclaim 無し）。
- **membarrier**（324）— CMD_QUERY で portable コマンドの mask、それ以外は 0。
- **getrusage**（98）— ゼロ埋め。
- **umask**（95）— 022 を返し新値は無視。
- **sched_getscheduler/setscheduler/getparam/setparam/get_priority_max/min、getpriority/setpriority**（140-147）— SCHED_OTHER・nice 0 相当の固定値。
- **setuid/setgid/setre*/setres*、getresuid/getresgid、getgroups/setgroups、capget/capset、syslog**（103/105/106/113-120/125/126 …）— 単一ユーザ(uid0)・sandbox 無し前提の no-op / 固定値。
- **rt_sigpending**（127）— 空シグナルセットを返す。
- **fchmod/fchmodat/chmod/fchown/chown/lchown/fchownat/utimensat**（90-94/260/268/280）— 0（パーミッション/所有者/時刻は保持しない）。

---

## 4ter. 意図的に見送った項目（バケット B/C）と根拠

ユーザ判断（2026-08-28）により、以下は「この環境ではブート検証ができず、盲目的に実装するとかえって危険」なため **本セッションではコード変更せず、根拠のみ記録**する方針とした。

> **更新 2026-08-28（第3セッション）**: ユーザ指示により B・C も実装した。COW fork は「コードは入れたが `KERNEL_COW_FORK` 既定 0」＝**危険な変更を既定経路から外す**という形で、下表の懸念（QEMU 検証なしでの物理メモリ破壊リスク）に対処している。P2 のカスタム Ozone だけは依然として着手不能（プリビルドバイナリには足せない・ソースツリーが無い）。

| 項目 | 状態 | メモ |
|---|---|---|
| **COW fork** | ✅実装 / 既定無効 | 参照カウント（`Memory_Main`）＋`PAGE_COW`＋PF フック＋COW clone を実装。`KERNEL_COW_FORK`（`kernel/config.h`）で切替、既定 0。QEMU 検証後に有効化する想定。`-DKERNEL_COW_FORK=1` フルビルド／`-Werror` 通過済み。 |
| **MAP_SHARED / ファイル mmap ライトバック** | ✅実装（制限あり） | 登録表＋`msync`(26)／`munmap`／exit での書き戻し。プロセス間ライブ・コヒーレンシは無し（page cache 不在）。匿名共有（memfd/`/dev/shm`）は tmpfs 実体で従来どおり。 |
| **フォント（TTF/OTF 実バイナリ）** | ✅実装 | `EtcFS.c` に `/etc/fonts/fonts.conf`。実 TTF は `BootManager/Resource/Fonts/NotoSansJP-Regular.ttf` を `WITH_CHROME=1` で `/usr/share/fonts` にステージ。 |
| **タイムゾーン（/etc/localtime）** | ✅実装 | 妥当な IANA `Etc/UTC` TZif（114B、システムの実体コピー）を `EtcFS.c` に静的埋め込み。 |
| **ビルド済み Chromium の同梱** | ✅実装 | `Userland/Application/com.ImplusOS.chrome/`（ステージのみ）。`WITH_CHROME=1 make image` で `/Userland/com.ImplusOS.chrome/` に展開＋glibc 自動同梱＋イメージ拡大。 |
| **P2 / Ozone カスタムプラットフォーム** | ⛔着手不能 | プリビルドバイナリには追加不可（Chromium 本体の再ビルド＝~100GB ソースが必要、この環境に無い）。プリビルドに対しては `--headless=new` が唯一の経路。 |
| **P3 テスト基盤の一部** | 一部 | syscall トレースは §6 で済。計測フック（VmSize/FD）とテストバイナリ一式は未。 |

---

## 4quater. COW fork（第3セッションで実装・既定は無効）

実装済み。構成:

- `Kernel/MemoryManagement/Memory_Main.c` — 物理ページ 1 バイト参照カウント配列（`memory_init_page_refcounts()` でヒープ確保後に arm、`pmm_page_ref_inc/dec/get`）。`free_page()` は refcount ≥2 のフレームを即解放せず参照だけ落とす。規約: 0/1 = 単独所有、2 = 共有開始。255 で飽和（そのフレームは以後回収しない＝稀なリーク）。
- `Kernel/Arch/x86_64/mmu/Paging_Main.c` — `PAGE_COW`(bit11)。`paging_cow_clone_user_range()` は present な user ページを子へ read-only 共有＋refcount++、書込可だった親 PTE を RO+COW にダウングレード、最後に `smp_tlb_shootdown_all()`。`PAGE_EXTERNAL`（共有メモリ/MMIO）ページは従来どおり deep copy。`paging_handle_cow_fault()` は write フォルトで、最後の所有者なら RW を戻すだけ、そうでなければ新フレームへコピーして `free_page(old)`。
- `Kernel/Arch/x86_64/cpu/IDT_Main.c` — PF ハンドラの `PF_USER` 経路先頭（swap/SIGSEGV より前）で write フォルト時に `paging_handle_cow_fault()`。
- `Kernel/Core/process/ProcessManager_Create.c` — `process_clone_address_space()` の Linux ABI 経路で COW clone を試み、失敗時は従来の `paging_copy_present_user_range()` にフォールバック。
- `Kernel/Core/kernel_main.c` — heap 初期化直後に `memory_init_page_refcounts()`。

**すべて `#if KERNEL_COW_FORK`（`kernel/config.h`、既定 0）でガード**。無効時は参照カウント表すら確保せず、`free_page()` の追加分岐も no-op（`g_page_refcount==NULL`）。`-DKERNEL_COW_FORK=1` でフルビルド・`-Werror` 通過を確認済み。**既定 0 の理由**: PTE エイリアス＋SMP TLB コヒーレンシ＋物理アロケータを同時に触るため、QEMU 起動検証ができないこの環境で既定 ON にするのは危険。実機/QEMU で fork 多用ワークロード（zygote→renderer）を確認してから 1 にする。既知の弱点: clone と exit の同時進行で稀にフレームを 1 枚リークしうる（破壊ではない）。huge page (2MiB) 領域は COW せず deep copy にフォールバック。

---

## 5. P2 — GUI 描画（headless からの発展）

> **プリビルド Chromium が前提のこのタスクでは、カスタム Ozone プラットフォームは実装不能**（Chromium 本体の再ビルド＝~100GB ソースが必要で、この環境に無い）。`chrome-linux/chrome` が持つ Ozone バックエンドは headless / X11 / Wayland のみ。したがって GUI 経路は **`--headless=new`（`chrome-headless` ランチャの既定）** のみ。以下は「ImplusOS 用 Chromium を将来自前ビルドする」場合の設計メモとして残す。

- [~] **headless での安定化**: `chrome-headless` ランチャ（`Userland/Application/com.ImplusOS.chrome/`）が `--headless=new --no-sandbox --disable-gpu --use-gl=swiftshader --no-zygote --single-process` で起動。DOM/ネットワーク/JS の実挙動確認は QEMU 起動が前提で未。
- [ ] **Ozone カスタムプラットフォームの設計**: ⛔プリビルドには足せない。自前ビルド時に `ui/ozone/platform/implusos` を新設し `Userland/API/Window.h`/`Graphics.h` へブリッジ。
- [~] **SwiftShader**: `libvk_swiftshader.so` / `libGLESv2.so` / `libEGL.so` は `chrome-linux/` に同梱済み。ランチャは `--use-gl=swiftshader --use-angle=swiftshader` 指定。
- [ ] **入力**: ⛔同上（Ozone InputBackend は本体ビルドが必要）。
- [ ] **ネイティブ DRM/EVDEV syscall と Ozone の統合**: ⛔同上。
- [ ] **画面サイズ/スケーリング/DPR**: ⛔同上。

---

## 6. P3 — 検証・テスト基盤

> QEMU でのブート検証がこの環境の対象外のため、以下は「コードは書けるが緑にできない」もの中心。§9 の glibc 移植が緑判定の前提。

- [ ] **Linux ABI テストバイナリ一式**: 静的・動的 PIE の最小 ELF、getpid/statx/clock_gettime/futex/シグナル/スレッドの網羅テスト。
- [x] **syscall トレース**: 実装済み。`Kernel/Compat/Linux/Syscall_LinuxCompat.c` の `linux_syscall_dispatch` に `LINUX_TRACE_ENTER`/`LINUX_TRACE_EXIT` を追加。`-DLINUX_SYSCALL_TRACE` 付きでカーネルをビルドすると、全 Linux-ABI syscall の「番号・6引数・戻り値」を COM1 に出力（`[lx] #<num> (<a1>,...,<a6>)` と `[lx] #<num> = <ret>`、いずれも16進）。マクロ未定義時はゼロコスト。allocation-free / lock-free で raw syscall 経路から安全に呼べる。ビルド例: `make kernel` の CFLAGS に `-DLINUX_SYSCALL_TRACE` を足すか、`CI=1` を使わずに `arch.mk` に一時追加。
- [ ] **glibc の直接実行**: 自前 libc ではなく実 glibc でコンパイルしたバイナリ（`/bin/busybox`、`/bin/dash` 等）の実行を最初のベンチマークにする。→ §9 で glibc を submodule 化しクロスビルド経路を整備済み。実バイナリのイメージ配置とブート確認が次段。
- [ ] **メモリ・FD 使用量の計測**: Chromium 各プロセスの VmSize/スレッド数/FD 数を監視。
- [ ] **性能計測**: フォーク時間、mmap スループット、epoll レイテンシ（COW 導入効果の確認）。

---

## 7. 推奨実装順序（ロードマップ）

| 段階 | 内容 | 完了基準 |
|---|---|---|
| **Step 1** ✅実装済み(動作確認は対象外) | FD/プロセス/スレッド上限拡張、RLIMIT_AS 修正、devfs+urandom 追加、statx/sysinfo/statfs/sched_getaffinity/madvise 追加 | busybox と静的バイナリが起動・終了できる |
| **Step 2** ✅実装済み。第3セッションで MAP_SHARED ファイルライトバック（msync/munmap/exit）と COW fork（`KERNEL_COW_FORK` 既定 0）を追加 | mremap(FIXED/DONTUNMAP 含む)、mmap MAP_SHARED、futex 拡張、シグナルフレーム完全化+rt_sigreturn、clock_gettime クロック追加 | glibc の malloc/pthread_cond が正常動作 |
| **Step 3** ✅epoll/UDP/AF_UNIX 済み。第2セッションで EPOLLET 実エッジ化・ソケット O_NONBLOCK 追跡・非ブロッキング connect の `EINPROGRESS`・`accept4`・非ブロッキング recv の `EAGAIN` を追加 | epoll イベント駆動化、非ブロッキング socket、SO_ERROR、UDP ソケット | 非同期ネットワーク IO が成立 |
| **Step 4** ✅コード実装完了。COW fork は `KERNEL_COW_FORK` 既定 0（QEMU 検証待ち）。SIGCHLD/wait ステータス変換・SIGPIPE・robust list も第2セッションで追加 | COW fork、AF_UNIX + socketpair、procfs (/proc/self/maps 等) | **headless Chromium が起動する**（ブート検証は対象外） |
| **Step 5** ✅ /etc ファイル群・フォント（Noto TTF ステージ + fonts.conf）・timezone（UTC TZif）・動的リンカ配線（`WITH_GLIBC`/`WITH_CHROME`）を実装。glibc は §9 で submodule 化 | /etc ファイル群、フォント、timezone、動的リンカの整備 | `--headless=new` でページを描画・実行できる（要ブート検証） |
| **Step 6** 🟡プリビルド Chromium を `com.ImplusOS.chrome` app として同梱＋`--headless` ランチャを用意。カスタム Ozone プラットフォームはプリビルドには足せない（本体再ビルドが必要＝別プロジェクト） | Ozone カスタムプラットフォーム、SwiftShader、入力 | GUI で Chromium が実用動作 |

---

## 8. 関連ソースファイル索引

| ファイル | 役割 |
|---|---|
| `Kernel/Core/syscall/Syscall_LinuxCompat.c` | Linux ABI ディスパッチ本体 (88 syscall) |
| `Kernel/Core/syscall/Syscall_Dispatch.c` | ABI 分岐 + ネイティブディスパッチ |
| `Kernel/Core/syscall/Syscall_Epoll.c` | epoll/eventfd (要再設計) |
| `Kernel/Core/syscall/Syscall_Futex.c` | futex (要拡張) |
| `Kernel/Core/syscall/Syscall_Socket.c` | socket syscall (TCP のみ) |
| `Kernel/Core/syscall/Syscall_VM.c` | mprotect/munmap/mremap(スタブ) |
| `Kernel/Core/syscall/Syscall_File.c` | fd テーブル/pipe/timerfd/memfd/signalfd |
| `Kernel/Core/syscall/Syscall_Clock.c` | clock_gettime/getres |
| `Kernel/Core/process/ProcessManager_Create.c` | fork/execve/waitpid/signal/clone |
| `Kernel/Core/elf/ELF_Loader.c` | linux_abi 判定 + PT_INTERP |
| `Kernel/Core/vfs/VFS.c` | VFS (procfs/devfs 追加先) |
| `Kernel/include/kernel/config.h` | リソース上限のコンパイル時定義 |
| `Userland/Service/com.ImplusOS.posix/src/posix_fdtable.c` | ユーザー側 FD テーブル (1024) |
| `Userland/Service/com.ImplusOS.posix/src/posix_io.c` | select/poll/fcntl |
| `libc/I_libc/src/posix.c` | POSIX 名ラッパー層 (4275 行) |
| `Userland/Service/com.ImplusOS.netstack/DNS/DNS.c` | ユーザー側 DNS resolver (UDP/TCP) |
| `libc/glibc/` | **glibc submodule**（upstream, pinned `glibc-2.41`） |
| `libc/build-glibc.sh` | glibc クロスビルド／ステージングスクリプト |
| `libc/README-glibc.md` | glibc 移植方針・ビルド手順 |

---

## 9. glibc の移植（submodule 方式・Linux ABI ビルド）

> 方針決定（ユーザ判断 2026-08-28）: **upstream glibc を無改変で submodule 化し、`x86_64-linux-gnu` としてクロスビルド**する。ImplusOS 固有の `sysdeps` port は作らない。理由: カーネルの Linux syscall 互換層（`Kernel/Compat/Linux/`）が既にジェネリックな Linux syscall を受けるため、glibc 側は「素の Linux バイナリ」として動けばよく、**カーネル依存が最小**（維持すべき glibc パッチが無く、未対応機能は互換層の `ENOSYS` として一点に集約される）。`libc/I_libc`（最小 freestanding libc）はカーネル／ネイティブ userland 用として不変。glibc は **外来 Linux バイナリ専用**。

### 9.1 完了した作業（本セッション）

- [x] **submodule 追加**: `libc/glibc` ← `https://sourceware.org/git/glibc.git`、タグ `glibc-2.41`（branch `release/2.41/master`）に固定。`.gitmodules` 登録済み。
- [x] **クロスビルドスクリプト** `libc/build-glibc.sh`（`configure` / `build` / `install` / `stage <dir>` / `clean` / `all`）。
  - `--host=x86_64-linux-gnu`（`x86_64-elf` ではなく Linux ABI ターゲット）
  - `--enable-kernel=5.15.0`（`linux_uname()` が返すバージョンに整合、pre-5.15 の fallback syscall 経路をコンパイルアウト）
  - `libc_cv_slibdir=/lib64`（動的リンカを `/lib64/ld-linux-x86-64.so.2` に）
  - `MAKEINFO=:`（texinfo マニュアルをスキップ）、`--disable-nscd --without-selinux --disable-profile`
  - out-of-tree ビルド（`Build/glibc/obj`）、DESTDIR install（`Build/glibc/sysroot`）、submodule 作業ツリーには一切書き込まない
- [x] **Makefile ターゲット**: `make glibc` / `make glibc_configure` / `make glibc_stage` / `make glibc_clean`。`all`/`image` の依存には**あえて含めない**（ビルドが重く、実 glibc バイナリを出荷するときだけ必要）。
- [x] **フルビルド完走**: 本セッションでこの環境上で `make glibc` を実走し成功。`Build/glibc/sysroot/lib64/` に `libc.so.6`(11.5MB)、`ld-linux-x86-64.so.2`(1.4MB)、`libm.so.6`、`libpthread.so.0`、`librt.so.1`、`libresolv.so.2` ほかを生成。`file libc.so.6` は `for GNU/Linux 5.15.0` と表示され `--enable-kernel` と整合。gcc 15.2.0 / binutils（system）でビルド。
- [x] **staging 検証**: `make glibc_stage GLIBC_STAGE_DIR=<tmp>` が 10 個のランタイム `.so` + `/etc/ld.so.conf` を `<tree>/lib64` と `<tree>/usr/lib` に配置することを確認（シンボリックリンクは実体に解決してコピー）。
- [x] **ドキュメント** `libc/README-glibc.md`。

### 9.2 残作業

- [x] **イメージへの配線（opt-in）**: `WITH_GLIBC=1 make image` で `install_payload` が `glibc_image_stage`(＝`make glibc` → `build-glibc.sh stage`)を先に走らせ、`Build/x86_64/glibc/image-stage/{lib64,usr/lib}` を install payload と OS パーティションイメージの `/lib64`・`/usr/lib` に mcopy する。**既定はオフ**(`WITH_GLIBC` 未設定なら従来どおり `all`/`image` に一切影響しない)—ドキュメントの「重い glibc ビルドを既定経路に入れない／まず QEMU で 1 本確認」方針をそのまま踏襲。`/etc/ld.so.conf` は EtcFS が供給するのでイメージには入れない。
  - 第3セッション追加: `WITH_CHROME=1` はこの `WITH_GLIBC=1` を **強制**（`override`）し、さらに `com.ImplusOS.chrome` app を APP_DIRS に追加、`INSTALL_DISK_IMAGE_SIZE_MB` を 1536 に、`NotoSansJP-Regular.ttf` を `/usr/share/fonts` にステージする。`make image` 単体（引数なし）は完全に不変。
- [x] **ランタイム補助ファイル（テキスト分）**: `Kernel/Core/vfs/EtcFS.c` に `g_etcfs_static_files[]` を追加。`/etc/nsswitch.conf`（`files dns`）、`/etc/ld.so.conf`（`/lib64` `/usr/lib` `/usr/local/lib`）、`/etc/passwd`（root/nobody）、`/etc/group`、`/etc/host.conf`、`/etc/gai.conf`（IPv4 優先）、`/etc/shells`、`/etc/os-release` を静的生成。`/etc/hosts`・`/etc/resolv.conf` は従来どおり動的生成。
- [~] **ロケール**: `C.UTF-8` locale-archive は未着手（バイナリ資産）。**`/etc/localtime` は実装済み**（第3セッション、EtcFS に妥当な `Etc/UTC` TZif）。
- [ ] **動的リンカ経路の実機確認**: `ELF_Loader.c` の PT_INTERP 処理は実装済み（§3.10）。`WITH_GLIBC=1`/`WITH_CHROME=1` で実 `ld-linux-x86-64.so.2` がイメージに入るようになった。共有ライブラリ検索・mmap(MAP_FIXED)・シンボル解決がユーザ空間で完結することのブート検証が次段（この環境の対象外）。
- [ ] **最初のマイルストーン**: `x86_64-linux-gnu` でコンパイルした `/bin/busybox` または `/bin/dash`、そして本命の `/Userland/com.ImplusOS.chrome/chrome --headless=new` の実行（§6 / P3）。ここで顕在化した `ENOSYS` を `Syscall_LinuxCompat.c` に随時追加する。
- [ ] **TLS/スレッド**: glibc の NPTL は `set_robust_list`/`rseq`/`clone(CLONE_SETTLS)` を使用（いずれもカーネル側実装済み）。`FUTEX_LOCK_PI`/`UNLOCK_PI` も §3.6 で実装済み（所有権プロトコルのみ、優先度継承は無し）。`robust list` の実死亡時処理（`FUTEX_OWNER_DIED` の伝播）は未検証。
- [~] **`__libc_start_main` の auxv 依存**: `AT_PHDR`/`AT_PHENT`/`AT_PHNUM`/`AT_ENTRY`/`AT_BASE`/`AT_PAGESZ`/`AT_RANDOM`(16B)/`AT_SECURE`/`AT_UID`〜`AT_EGID`/`AT_EXECFN` は既に積まれていることを確認。本セッションで **`AT_HWCAP`(CPUID leaf1 EDX) と `AT_CLKTCK`(`timer_hz()`、0 なら 100)** を追加(`initialize_elf_user_stack_ex`)。実バイナリでの通し検証は QEMU ブートが前提のため未実施。

---

## 10. 残課題 — SMP でのメモリ破壊（2026-09-06 時点の唯一のブロッカー）

`-smp 1` では headless Chromium が長時間安定して動くのに、`-smp 2` 以上では
15〜90 秒で必ず死ぬ。死に方は毎回違う（NULL / 3 / 0xFFE7CE… といったゴミ
ポインタへの書き込み、PartitionAlloc の `FreeInUnknownRoot` でのゴミ free、
`base::sequence_manager` 内の NULL デリファレンス）。**症状が非決定的で
CPU 数に依存する**ので、機能不足ではなくカーネルの SMP レースである。

### 10.-10 追記 (2026-09-14, 夜) — 時刻・プロファイル・入力・「空白ウィンドウのハング」

#### (a) ゲストと Chromium の時刻がずれていた（修正済み）

- `clock_gettime(CLOCK_REALTIME)` / `gettimeofday` / `time` が呼ぶたびに RTC を読んでおり、
  CMOS の更新中に読むと秒が飛ぶうえ、秒未満が常に 0 だった。
- `clock_realtime_ns()`（Syscall_Clock.c）を追加: 起動時に RTC を 1 回（2 回一致するまで）
  読んで基準にし、以後は `timer_monotonic_ns()` で進める。RTC アクセスは spinlock で保護。
- Linux 側の `FUTEX_WAIT`/`FUTEX_WAIT_BITSET` のタイムアウトが timespec ポインタの値そのもの
  （≒20 分）として扱われていた。相対／絶対（CLOCK_REALTIME フラグ含む）を正しく変換し、
  期限切れは値を確認してから `ETIMEDOUT`/`EAGAIN` を返すようにした。

#### (b) 「プロファイルを読み込めませんでした」ダイアログ（修正済み）

- 共有メモリ／プロファイル DB 周りで fd が尽きていた（`EMFILE`）。fd テーブルを 512 に拡張。
  ただし 192..255 は AF_UNIX ソケットの番号帯なので、ファイル表はそこを飛ばす（下の (d)）。
- tmpfs で「開いたまま unlink」されたファイルが即座に消えていた（Chromium の共有メモリは
  /dev/shm に作って即 unlink する）。最後の close まで実体を残すようにした。
- `MAP_FIXED` の匿名／ファイル mmap が既存のページを外さずに上書きしていた。先に
  `paging_unmap_range` する。TLB シュートダウン前にフレームを解放していた経路も、
  シュートダウン後にまとめて解放するよう変更（PartitionAlloc のメタデータ破壊の一因）。
- SQLite の「pwrite した内容を読み取り専用 mmap で読む」に対応するため、ファイル書き込みを
  同じアドレス空間の MAP_SHARED 写像へ反映する write observer を追加。

#### (c) マウス・キーボードを Chromium 内で使えるようにした

- XSession がウィンドウの入力（キー・ポインタ）を購読し、新 syscall `SYSCALL_EVDEV_INJECT`
  (275) で /dev/input/event0,1 に注入する。ポインタは EV_ABS（0..65535）で渡し、
  xorg.conf で `IgnoreRelativeAxes`/`IgnoreAbsoluteAxes` を指定（タッチスクリーン扱いされない
  ようにデバイスは REL_X/REL_Y も広告する）。
- PS/2: マウスパケット内の 0xFA/0xFE（dx/dy = -6, -2）を ACK/RESEND と誤認して捨てていた。
- USB HID（`make run_*` の既定: qemu-xhci + usb-kbd/usb-mouse）で入力が一切来なかった。
  `usb_storage_init()` がストレージ不在時に `usb_core_init()` を再実行し、そのたびに
  ルートポート表とアドレス割り当てを初期化していた。既に列挙済みのキーボード／マウスの
  ポートがリセットされ、スロットを保持したままなので Address Device が "port already
  assigned"（TRB_ERROR）で失敗し、以後ホットプラグ監視が約 1 秒ごとにポートリセットを
  繰り返してデバイスの割り込み転送を潰していた。再実行時は未列挙のポートだけを再試行し、
  失敗は 3 回で打ち切る（抜かれたらリセット）。切断時は xHCI スロットを解放する。
- USB キーボードの修飾キー（Ctrl など）はレポートの byte 0 のビットとしてしか扱っておらず、
  キーイベントとして出ていなかった。X からは Ctrl が押されていないので Ctrl+L が「l」になった。
  ビットの変化を 0xE0..0xE7 のキー押下／解放として発行する。
- /dev/input/event* への write（xf86-input-evdev の EV_LED 等）が EIO だった。受け付けて捨てる。

#### (d) クリック直後に Chromium が終了していた（修正済み）

- 症状: クリックした直後に "X connection error received" で終了。ログでは Chromium 自身の
  スレッドが X 接続と ProcessSingleton の listen ソケット（fd 0xC2..0xC4）を close していた。
- 原因: fd 192..255 を AF_UNIX に割り当てているのに、pipe / memfd（作成・SCM_RIGHTS 受信）/
  timerfd / signalfd / ディレクトリ fd の割り当てループはその帯を飛ばしていなかった。
  fd が 190 を超えると memfd などに 0xC2 が割り当てられ、Linux の `close()` は番号だけで
  `unix_socket_close()` に振り分けるので、memfd を閉じると同じ番号のソケットが壊れた。
- 修正: 全割り当てループで `fd_in_unix_hole()` を確認する。

#### (e) 空白（白／黒）ウィンドウのまま止まるハング

- 停止中の stall ダンプを 3 回分比べると、毎回 Chromium のメインスレッドを含む複数スレッドが
  **同じアドレスの futex で FUTEX_WAIT したまま**（同じユーザ RIP）で、他は epoll/ppoll を
  回っているだけだった。ロック解放時の FUTEX_WAKE が失われている。
- 原因: `process_block_current()` はタスクを BLOCKED にするだけで、実際の睡眠は syscall の
  出口で起きる。待機キューのエントリを消すのは FUTEX_WAKE（requeue）かタイムアウトだけだが、
  BLOCKED のタスクを起こす経路は他にもある（poll-wait の登録ビットが残ったままの notify、
  走行中に積まれた wake credit など）。その場合タスクはエントリを残したままユーザ空間に戻り、
  glibc が再び FUTEX_WAIT して 2 個目のエントリを積む。後の FUTEX_WAKE(1) が古い方に当たると、
  寝ていないタスクを「起こし」、本当に寝ているタスクは永久に起きない。
  poll-wait の起床はスレッドグループ ID 宛てだった（= メインスレッドのスロット）ので、
  メインスレッドが毎回巻き込まれていたのと合う。
- 修正:
  - Linux の FUTEX_WAIT は必ず syscall を再実行（restart）させ、再入時にエントリがまだ
    キューにあれば再び寝る。エントリが消えていれば 0（タイマーが消したなら ETIMEDOUT）を返す
    （`syscall_futex_linux_resume()`）。ネイティブ ABI の経路は変更なし。
  - Poll_Wait の登録・起床をスレッド ID 単位に変更（`process_sleep_current_ms()` と
    `process_wake_pid()` はスロット単位で動くため）。
- 結果（KVM `-smp 4`, `-m 4096`, 起動ごとに Chromium の UI が 20〜30 秒安定するまで観測）:
  - 修正前（Poll_Wait をスレッド単位にした版／しない版の A/B を含む計 9 回）: 空白のまま
    タイムアウト 5 回（A/B 6 回中 3 回、別計測 1 回中 1 回、診断ビルド 2 回中 1 回）。
  - futex 修正後: 6 回連続で UI 表示・安定、空白期間は 2.1〜4.3 秒。USB 入力テスト（クリック、
    文字入力、Ctrl+L、新しいタブ）も通過。

#### (f) Chromium 終了後にデスクトップへ黒い矩形が残る（修正済み）

- ミラー解除後も Xorg はクライアント切断でサーバ再生成してフリップを続け、KMS はミラーが
  無いと実フレームバッファへ blit していた。一度ミラーを持ったセッションが解除した後は、
  新しいミラー登録か DRM クローズまでフリップを捨てる（`g_mirror_released`）。

#### 残っている問題

- USB mouse（相対座標）は QEMU 側の加速の影響で位置が合わない場合がある（ハーネスは相対移動で
  操作している。実機のタブレット／絶対座標デバイスは未対応）。
- CJK フォントが無いので日本語のサジェストが豆腐になる。
- tmpfs ファイルの MAP_SHARED を共有ページで実装する経路（`LINUX_TMPFS_SHARED_MMAP`）は
  起動時にデッドロックしたため無効のまま。

### 10.-9 追記 (2026-09-14, 夕) — Chromium の起動を約 5 倍に高速化（ウィンドウ→UI 中央値 90.6 秒 → 17.2 秒）

#### 効かなかったもの（A/B で確認して取り下げ）

- `--password-store=basic` と fontconfig の走査範囲縮小（`/usr/share/fonts/truetype` のみ・
  DejaVu・`<rescan>` 削除）: 対照 6 回と交互に起動して、ウィンドウ→UI 中央値 70.7 秒 vs 72.8 秒で
  差なし。D-Bus の NameHasOwner 失敗は即座に返っており、待ち時間の原因ではなかった
  （§10.-7 の「D-Bus 待ち ~42 秒」という見立ては誤り）。しかもこの組では Chromium が
  PartitionAlloc の free 中に user-mode #GP（`FreeInUnknownRoot`、壊れたポインタが ASCII 文字列）
  で 6 回中 2 回落ちたため、変更は入れていない。
- ホストが 4 コアなので、計測ハーネスの画面取得を 1 秒間隔にするとゲスト全体が遅くなる
  （デスクトップ到達 19→26 秒）。2 秒間隔に戻して計測した。
  また、別セッションの基準値とは比べられない（ホストの状態で大きく変わる）。必ず同条件で交互に起動する。

#### 原因の特定（QEMU モニタで全 vCPU のレジスタを 0.5 秒ごとに採取）

ウィンドウ表示〜空白ページの間、**CPU サンプルの約 80% がカーネル、ユーザーモードは約 1%**。
Chromium はほとんど動いていなかった。カーネル内の内訳（シンボルは `Kernel_Main.ELF.sym`）:

| RIP | 関数 | 意味 |
|---|---|---|
| 343 | `hal_cpu_pause` | spinlock の空回り |
| 166 | `lapic_get_id` の MMIO 読み出し直後 | VM exit |
| 36 | `hpet_monotonic_ns` の MMIO 読み出し直後 | VM exit（QEMU プロセスまで出る） |
| 57 | `hal_io_in8/out32/in32` | ポート I/O exit |

スタックを見ると、`smp_get_current_cpu_id()`（LAPIC ID の MMIO）が 1 回の syscall で何度も
（`syscall_set_user_rsp`・スケジューラ・TSS rsp0 更新・`process_get_current_pid`・spinlock の
TLB ポーリング）、`timer_monotonic_ns()`（HPET の MMIO）がスケジューラと poll/futex 待ちで
呼ばれ、しばしばプロセステーブルのロックを握ったまま exit していた。残りの CPU はその
ロックの後ろで回っていた。

#### 修正（Kernel）

- `smp_get_current_cpu_id()`: 各 CPU が起動時に `MSR_TSC_AUX` へ index+1 を書き、`RDTSCP`
  （KVM ではネイティブ実行）で読む。0（未設定）なら従来どおり LAPIC ID。
- `timer_monotonic_ns()`: `timer_init()` で TSC を HPET に対して 50 ms 校正し、以後は TSC を使う。
  CPU 間でわずかにずれても逆行しないよう、全 CPU で返した最大値にクランプ。

#### 結果（同一イメージ・カーネルだけ違う 2 種を交互に 6 回ずつ、KVM `-smp 4`）

| | ウィンドウ→空白ページ | 空白ページ→UI | **ウィンドウ→UI** | UI 安定 |
|---|---|---|---|---|
| 修正前 | ~39〜45 秒 | ~45〜52 秒 | **中央値 90.6 秒**（88〜93、1 回は 253 秒） | 5/6（1 回タイムアウト） |
| 修正後 | ~11〜13 秒 | ~4〜32 秒 | **中央値 17.2 秒**（15〜45） | 5/6（1 回は UI 表示後に QEMU プロセスが終了。stderr 空・OOM なし、原因未特定） |

電源投入→UI は中央値 114.2 秒 → 38.6 秒。

#### 残っている問題

- 空白ページ・黒いウィンドウのまま止まるハング: この日の旧カーネルでは 7 回中 4 回と多かった
  （9 月 13 日の 25 回連続では 1 回）。停止中は全 CPU がカーネル内で spinlock を回っており、
  デッドロックの可能性がある。TSC 化の後は 6 回中 0 回だが、原因の特定・修正はまだ。
  → 10.-10 (e) で原因（FUTEX_WAKE の取りこぼし）を特定し修正した。
- プロファイル中、QEMU プロセスが UI 表示後に理由不明で終了した（1 回）。ハーネスが
  終了コードを記録するようにしたので、次に起きたらシグナルか正常終了か分かる。
- PartitionAlloc の free 中の user-mode #GP（上記）。ヒープ破壊の出所（カーネルの
  mmap/madvise/スレッド切り替えなど）は未調査。
- 起動時のプロファイル採取は `info registers -a` を 0.5 秒ごと、スタックは `cpu N` + `x/24gx`。
  スタックを取るとゲストがかなり遅くなるので、時間計測とは別の起動で行うこと。

### 10.-8 追記 (2026-09-14) — SMP の即死（ハング／無音リセット／#DF）を解消。GUI 付き 25 連続起動で 0 件

GUI 付き QEMU（KVM, `-smp 4`, `-m 4096`）を毎回コールドブートし、画面キャプチャで
デスクトップ／Chromium ウィンドウ／UI 完成の時刻を測る連続起動ハーネスを作って
計測した。シリアルが 60 秒止まったら QEMU モニタで全 vCPU のレジスタとスタックを
採取し、`-action reboot=shutdown,shutdown=pause` でリセットを「停止」に変えて
リセット瞬間の状態も採れるようにした。

#### 症状（修正前、13 回）

| 結果 | 回数 |
|---|---|
| UI 到達・安定 | 6（全回で「プロファイルを開けません」ダイアログ） |
| UI 到達後にリセット | 1 |
| **Chromium 起動 1 秒以内に死亡**（ハング 2・無音リセット 2・#DF 1） | **5** |
| 空白ウィンドウのまま Chromium が止まる | 1 |

#### 決め手になった観測

- リセット時の vCPU: **`CS=0008 CS64 CPL=0`、`RIP=0x8019`、`RSP=0`**。#DF のパニックでは
  **`RIP=0x800C`**。0x8000 は SMP トランポリン。16 ビット用のバイトを 64 ビットとして
  実行すると `31 E4` = `xor esp,esp`（RSP=0）、`0F 01 16` = `lgdt [rsi]`（0x800C）、
  `mov cr0` で落ちる（0x8019）。INIT/SIPI ではなく、**ロングモードのまま 0x8000 へ飛んでいる**。
- その CPU の **RBP**（トランポリンは RBP を壊さない）が、**別の CPU の生きている
  RSP の 0x28 バイト上**を指していた＝**2 つの CPU が同じカーネルスタックに乗っていた**。
  後から来た CPU のフレームが戻りアドレスを上書きし、`ret` が 0x8000 に着地する。
  同じ CPU がロックを握ったまま死ぬとハング、例外が積めないとトリプルフォルト。

#### 原因

`process_schedule_on_syscall()` は次のタスクを選び、旧タスクを「leaving」にしてロックを
外すが、`syscall_entry` が新しい RSP を載せるまで**旧タスクのスタック上で動き続ける**。
ところが `scheduler_pid_running_on_other_cpu()` は「current」しか見ておらず「leaving」を
見ていなかったので、その間に別 CPU が旧タスクを再開できた。割り込み禁止なので数命令の
窓だが、**vCPU はホストにその途中で数ミリ秒止められうる**。

#### 修正（Kernel）

- `scheduler_pid_running_on_other_cpu()` が「leaving」も拒否する。
- `syscall_entry` が `mov rsp, rax` の直後に自 CPU の leaving を消す（拒否が長引かない）。
- `process_detach_dead_current_locked()` が死んだタスクのスタックをスロットから外して
  その CPU に退避（同じスロットへの次の fork が、アイドル中の CPU が乗っているスタックを
  再利用しないように）。これ単独ではほとんど効かなかった（8 回中 4 回即死）。
- ついでに: `open(dir, O_RDONLY)`（O_DIRECTORY なし）を許可（LevelDB の SyncParent）、
  tmpfs のスロット表を 256 → 4096（ピーク 224 以上に達していた）。

#### 結果（修正後、GUI 付き 25 連続起動）

| 結果 | 回数 |
|---|---|
| UI 到達・安定 | **24** |
| 即死（ハング／リセット／パニック） | **0** |
| 空白ウィンドウのまま停止（ラン 1） | 1 |

電源投入→UI 完成は 83〜96 秒（中央値 ~87 秒）。

#### 残っている問題

- **プロファイルエラーのダイアログは全回で出る。** Web Data（SQLite）の初期化失敗
  （`token_service_table.cc:216 Failed to load tokens (invalid SQL statement)`）が原因と
  見ている。tmpfs のピークは 256 未満の回でも出るので、スロット枯渇ではなかった。
  GCM Store の `LockFile::16`（EIO）も残る。失敗している syscall はまだ特定していない。
- 空白ウィンドウのまま Chromium が止まる回（25 回中 1）。CPU はロック待ちで回っており
  アイドルではない。
- `process_run_next_on_current_cpu()`（フォルト処理後の一方通行 iretq）も leaving を立てるが、
  次にその CPU がスケジュールするまで消えない。生きたタスクがこの経路で切り替わると
  他 CPU で走れなくなる可能性がある（未確認・未修正）。

### 10.-7 追記 (2026-09-13, 夜) — ウォームリセット後の KSTACK パニック修正、ランチャーの再作成

§10.-6 の状態をまっさらな環境（リポジトリを clone し、`Resource/` だけ手で配置）
から再現しようとして当たったもの。

#### (a) ウォームリセット後に init が `[KSTACK]` で必ず死ぬ（カーネル、修正済み）

```
[OS] [KSTACK] kernel stack overflow pid=0x0000000000000001 name=Userland.ELF base=0x0000000003945A10
```

電源投入直後の起動では出ず、**同じ QEMU プロセス内でリセットした 2 回目の
起動**で毎回、同じアドレスで出る。Chromium 実行中の無音リセットの後に
これが続くので、「Chromium を動かすとマシンが二度と上がらない」ように見えた。

原因: `process_manager_init()` はプロセステーブルを `malloc()` で取るが、
カーネルヒープは確保領域をゼロにしない。`reset_process_slot()` は
`kernel_stack_base` を**意図的に**消さない（§10.-2 で「スタックはスロットの
持ち物」にしたため）。電源投入時はたまたまメモリがゼロなので全スロットが
スタックを新規確保するが、ウォームリセットでは RAM に**前回ブートの
テーブル**が残っており、ヒープ配置も決定的なので、もっともらしい
`kernel_stack_base` が見えて確保が飛ばされ、今回のブートで別用途に渡した
メモリの上で init が走っていた。

修正 (`Core/process/ProcessManager_Create.c`): 確保直後に
`memset(g_processes, 0, ...)`。

再現手順（Chromium 不要）: 起動 → デスクトップ到達 → QEMU モニタで
`system_reset` → 2 回目の起動。修正前は毎回 `[KSTACK]`、修正後はデスクトップまで
上がる。実機でも電源投入時の RAM がゼロである保証はないので、同じ穴だった。

#### (b) ランチャーがリポジトリに存在しなかった

`Userland/.gitignore` が `/Application/Chromium` を**ディレクトリごと**除外して
いたため、§10.-6 までのランチャー（`Start.c` / `Makefile`）は一度もコミットされて
いなかった。`Userland/Application/Chromium/{Start.c,Makefile}` を Doom と同じ
`XSession` 経路で書き直し、除外を `Resource/`（Chromium 本体、~500 MB）に限定した。

#### (c) `--use-gl=swiftshader` は Chromium 155 では無効

最初に書いたランチャーは `--use-gl=swiftshader --use-angle=swiftshader` を渡して
おり、約 50 秒後にインプロセス GPU スレッドが死んでいた:

```
ERROR:ui/gl/init/gl_factory.cc:110] Requested GL implementation (gl=none,angle=none) not found in allowed implementations: [(gl=egl-angle,angle=default)].
FATAL:ui/gl/init/gl_factory_ozone.cc:62] NOTREACHED hit. Expected Mock or Stub, actual:0
[OS] [#GP] user-mode #GP -> terminating ... name=Chrome_InProcGp
```

（FATAL の文字列は `#GP` ダンプのユーザスタック上に ASCII で残っていた。）
§2 の方針どおり `--disable-gpu` にすると GL 実装を要求しなくなり、描画は
ソフトウェア合成になる。この組み合わせのとき、§10.-6 に書いた無音リセットも
3 ブート中 2 回起きていたが、`--disable-gpu` 後の計測では起きていない
（まだ 1 回のみ、結論は出していない）。

#### 到達点（KVM, `-smp 4`, `-m 4096`）

電源投入 → デスクトップ → Chromium 起動、90 秒時点でタブストリップ・
オムニボックス・ツールバー・about:blank まで描画済み、6 分間リセット・
パニック・プロセス終了なし。

#### 残っている既知の問題

- 「Something went wrong when opening your profile」ダイアログが出る。
  `--user-data-dir=/tmp/chrome-profile` 配下で LevelDB がディレクトリを開けない
  （`GCM Store: Unable to open directory`）。§10.-6 の「プロファイルエラー 0 件」は
  再現していない。
- Chromium が終了するとランチャーは `xsession_close()` で Xorg を kill するが、
  `SYSCALL_TKILL` はシグナルを積むだけで、Xorg は数分経っても終了しない。
  ミラーは先に外れるので、Xorg が 1024x680 の黒い画面をパネルに直接描き続ける。
- TCG（KVM なし）では 18 分待っても描画に至らなかった。計測には KVM が要る。

#### 環境構築で要ったもの（ドキュメント外）

- `genisoimage`（xorriso は UDF を作れない）。root がなければ
  `apt-get download genisoimage` を展開して `ISO_MASTER=` で渡せる。
- `Vendor/Library/zlib/zconf.h`（`zconf.h.in` をコピー）。
- `OVMF_CODE_4M.fd` をリポジトリ直下に（`/usr/share/OVMF/` からリンク）。

### 10.-6 追記 (2026-09-13) — **Chromium がブラウザ UI を描画し、動き続けるようになった**

タブストリップ・オムニボックス・ツールバー・about:blank のページ領域まで
描画され、起動から 6 分経っても落ちない。下の 4 件が効いた。§10.-5 の
epoll / タイマ修正が前提。

#### (a) `sched_getaffinity` がスレッド ID を受け付けなかった（これが描画の本命）

```c
int32_t current = process_get_current_pid();          /* = アドレス空間の所有者 */
if (pid != 0u && (int32_t)pid != current) return LINUX_ESRCH;
```

Linux のこの API の "pid" は**スレッド ID** で、glibc はそれに依存している:
`pthread_getattr_np()` は内部で `__pthread_getaffinity_np()` →
`sched_getaffinity(pd->tid, ...)` を呼ぶ。所有者 pid と比べていたので、
**メインスレッド以外からの呼び出しが全部 ESRCH** になっていた。

この errno 1 個で Chromium は描画できなかった。glibc は affinity のエラーを
そのまま返し、V8 の `base::Stack::GetStackStart()` はスレッドのスタック境界を
取得できず、`Heap::CollectGarbage()` の冒頭にある
`CHECK(isolate_->IsOnCentralStack())` が落ちる。つまり**レンダラスレッドで
最初に GC が走った瞬間に Chromium が abort** していた:

```
# Fatal error
# Check failed: isolate_->IsOnCentralStack().
#4  v8::internal::Heap::CollectGarbage(...)
#5  v8::internal::HeapAllocator::CollectGarbageAndRetryAllocation(...)
#9  v8::internal::Runtime_AllocateInYoungGeneration(...)
```

修正 (`Compat/Linux/Syscall_LinuxCompat.c`): `linux_sched_pid_is_self()` を
追加し、0・自スレッド・所有者・**同一スレッドグループの兄弟スレッド**を
受け付ける。`sched_setaffinity` と `prlimit64` も同じ規則に揃えた
（どちらも同じ比較をしていた）。

**症状の切り分けに使った手順**（同種の問題に再度当たったとき用）:
1. `[OS] [#GP]` のユーザスタックダンプに V8 のメッセージ文字列が載っている。
   リトルエンディアンで復号すると `Check failed: isolate_->IsOnCentralStack()`。
2. バックトレースの `#0` が `base::debug::CollectStackTrace` なのは既知なので、
   `nm` 出力 (`chrome.syms`) 中のその symbol の vaddr との差でロードベースが
   出る。この環境では **0x4000000000**（`[code]` 窓の先頭）。
3. 残りのフレームを同じベースで引くと上の呼び出し列になる。

#### (b) `madvise(MADV_DONTNEED)` が読み取り専用ページに書き込んでいた

ゼロを読ませるためにその場で memset していた（§3.8 の経緯）。しかし
PartitionAlloc は span を decommit するとき **`mprotect(PROT_NONE)` の後に**
`madvise(MADV_DONTNEED)` を呼ぶので、対象は往々にして書き込み不可。
カーネルモードの書き込みが present・読み取り専用のページで fault し、
`PAGE_FAULT: Page fault in kernel mode` でマシンが落ちていた
（`CR2=0x16201300000`, `pte=0x800000004B194065` = P=1/RW=0/U=1/NX, `lastsys=28`）。

修正: Linux と同じく**ページを破棄する**（`paging_unmap_range()`、連続する
区間ごとに 1 回だけシュートダウン）。次のアクセスでゼロフォルトするので
呼び出し側が読む値は変わらず、フレームは返り、読み取り専用ページへ書かなく
なる。

**副作用**: 破棄したページは absent になり、デマンドゼロ経路は absent な
ユーザページを「プログラムが最後に指定した保護」を見ずに書き込み可能で
マップする。したがって `mprotect(PROT_NONE)` + `madvise()` で decommit した
span は、フォルトせずアクセス可能なゼロとして読める。PROT_NONE で捕まえる
はずの use-after-free が見逃される。これは PROT_NONE 予約の未触ページに
ついては元から同じ（`Arch/x86_64/cpu/IDT_Main.c` のデマンドページング
コメント参照）。領域ごとの保護を持つのが両方まとめた本筋の直し方。

#### (c) `fcntl` のロック系コマンドが `ENOTSUP` だった

LevelDB は DB を開く前に `<db>/LOCK` を `fcntl(F_SETLK)` で押さえる。
`F_GETLK`/`F_SETLK`/`F_SETLKW` と OFD 版が未実装だったため、**Chrome
プロファイルを構成する 20 個以上の DB が全滅**し、ブラウザは
「Something went wrong when opening your profile.」のダイアログ付きで
起動していた。

修正: `F_SETLK`/`F_SETLKW`/`F_OFD_SETLK`/`F_OFD_SETLKW` は無条件に成功、
`F_GETLK`/`F_OFD_GETLK` は `l_type = F_UNLCK`（競合なし）を返す。
**これは実装ではなく割り切り**で、カーネルはロック表を持たない。単一の
ロック取得者が期待する契約だけを満たす（LevelDB はプロセス内の二重ロックを
自前の表で防いでいる）。別プロセスが同じ DB を開くのは止められない。

#### (d) CPL3 由来のフォルトでカーネルがパニックしていた

`general_protection_fault_handler()` は `from_user && pid >= 0` のときだけ
プロセスを殺し、それ以外はパニックしていた。`process_get_current_pid()` は
**アドレス空間の所有者**を返すので、マルチスレッドプロセスが最後のスレッドを
畳んでいる最中（`release_process_resources()` が
"threads still running on other CPUs" で破棄を遅延している窓）は解決できず、
**ユーザ空間の CHECK 失敗がカーネルパニックになっていた**（レポートの RIP が
ユーザアドレスなのが目印）。

修正 (`Arch/x86_64/cpu/IDT_Main.c`):
- `from_user` なら pid/tid が引けなくてもパニックしない。引けるならスレッドを
  終了させ、引けなければこの CPU を再スケジュールに回す。
- ページフォルトも同様。加えて**ユーザアドレスに対するカーネルモードの
  フォルト**は、サービスできなければそのプロセスを SIGSEGV で終了させる
  （memcpy の途中は巻き戻せないので再開はできないが、デスクトップは生き残る）。
- 診断として、諦める直前に PTE のフラグ・file-backed か否か・その
  スレッドの最後の Linux syscall 番号を出す。これが (b) の特定に直結した。

#### (e) `POLL_WAIT_MAX_DECLINES` を 1 に戻した

セッション途中（撤回したカーネル内待機ループの作業中）に 1 → 8 に上げてい
たのを元に戻した。`poll_wait_park()` は「スキャン中にイベントが来ていた」と
判断するとスリープを見送るが、generation はグローバルなので、待機者が複数
いると毎回トリップする。8 だと最大 8 回連続でスリープを飛ばし、実質的な
スピンになる。

実測（アイドル待機中の syscall/秒、SMP=4）:

| スレッド | declines=8 | declines=1 |
|---|---|---|
| chrome (main) | 1,300 | 329 |
| Xorg | 316 | 83 |

#### 到達点（SMP=4, MEM=4096, QEMU/KVM、出荷構成＝診断なし・`--v=1` なし）

| 経過 | 状態 |
|---|---|
| 〜25 s | デスクトップ（WM）起動 |
| 〜80 s | Chromium (X11) ウィンドウの全面描画 |
| 180 s | **ブラウザ UI 完成**（タブ・オムニボックス・ツールバー・空白ページ） |
| 390 s | 変化なし・生存（`#GP` 0 件、パニック 0 件、プロファイルエラー 0 件） |

#### 残っている既知の問題

- ログに残る唯一のエラーは LevelDB の `SyncParent`（4 件）。
  `open("<dir>", O_RDONLY)` がディレクトリを開けないため。動作への影響はない。
- 起動時間のばらつきが大きい（同じイメージで UI 到達が 150〜400 秒超）。
  内訳の大半は Xorg の起動で、ホスト側の負荷にも左右される。
- **プリエンプションが syscall 境界にしか無い。** `process_timeslice_expired()`
  を見ているのは `Syscall_Dispatch.c` の 1 箇所だけで、タイマ割り込みからの
  切り替え経路が無い（`process_schedule_on_syscall()` は syscall フレームの
  `saved_rsp` を差し替える方式なので、割り込みフレームからは再開できない）。
  CPU バウンドなユーザスレッドは syscall を出すまで CPU を離さない。
  現状の Chromium は poll ループで頻繁に syscall を出すので露見しにくいが、
  スケジューラの素性としてはここが一番の穴。統一した切り替え経路を入れるのが
  次の大きめの仕事。
- `-no-reboot` だけで QEMU を回すと、実行によっては途中で QEMU が rc=0 で
  終了する。`-d cpu_reset` は起動時の 8 件しか出ないのでリセットではなく
  **ゲストからのシャットダウン要求**。計測には `-no-shutdown` を併用のこと
  （`runq.sh` に追加済み）。
- AP のタイマ割り込みは CPU0 以外で早期 return され捨てられている（§10.-5）。

### 10.-5 追記 (2026-09-12) — epoll の 2 バグを修正。Chromium が初めてページを描画した

**結論: Chromium が X11 ウィンドウにピクセルを出すようになった。** 描画が
止まっていた原因は Chromium 側ではなく、カーネルの epoll に 2 件のバグが
あったこと。どちらも「マシン全体がアイドルなのに誰も前進しない」という
同じ症状に見えるため、fd レベルのダンプを足すまで切り分けられなかった。

#### (a) `epoll_ctl(ADD)` が不正な fd を受け入れていた → X サーバが CPU を独占

`syscall_epoll_ctl()` は fd を一切検証していなかった。一方
`epoll_poll_fd()` は「どのテーブルにも属さない fd」に `EPOLLERR` を返し、
`EPOLLERR` はレベル/エッジに関係なく必ず報告される。つまり**ゴミ fd が 1 個
入るだけで `epoll_wait()` が永久に即時復帰する**。

実測: Xorg の epoll セットに `fd = -22`（`= -EINVAL` をそのまま fd として
登録したもの）が入っていた。出どころは Xorg の `dbus-core` モジュールで、
`/run/dbus/system_bus_socket` への接続に失敗した戻り値を `SetNotifyFd()` に
fd として渡している（10 秒ごとに再試行するので永続する）。

```
[epoll] 0x4000 n=6 ... fd4294967274/w0x00000001/r0x00000008
```

結果、Xorg が 1 CPU を丸ごと燃やし、SMP=2 では Chromium が READY のまま
5 秒に数 syscall しか進めない（餓死）。

修正 (`Core/syscall/Syscall_Epoll.c`):
- `epoll_fd_is_addressable()` を追加し、負の fd とどのテーブルにも属さない
  fd を `EBADF` で弾く（Linux の契約どおり）。
- 併せて `ADD` の重複を `EEXIST`、`MOD`/`DEL` の未登録を `ENOENT`、未知の
  `op` を `EINVAL` にした（いずれも黙って成功していた）。

効果: Xorg の syscall 数が 300 秒あたり **226,251 → 3,268**。状態も
RUNNING 固定から BLOCKED になった。

#### (b) EPOLLET を「ポーラ側のレベル変化」で模倣していた → X サーバが永久に待つ

`epoll_check_once()` のエッジ判定は `ready & ~last_ready` だった。これは
**ポーラが観測した瞬間のレベル遷移**であって、Linux が エッジを立てる
「データ到着」ではない。レベルが立ったままの 2 回目の到着は遷移として
見えないので、**エッジが永久に出なくなる**。

実測: Xorg はクライアントソケットを `EPOLLET|EPOLLIN`（`w=0x80000001`）で
登録する。Chromium の要求 20 バイトがカーネルのリングに滞留したまま、
Xorg もChromium も眠り続けた。

```
[epoll] 0x4000 ... fd200/w0x80000001/r0x00000001   ← 読めるのに配送されない
[usock] fd200 own=6 peer=199 c q=20                ← 20 バイト滞留
[poll]  tid7 fd199/w0x00000001/r0x00000004         ← Chromium は応答待ち
```

修正:
- `IPC/UnixSocket.c`: `unix_sock_t` に `rx_seq` を追加。受信キューへの追記
  ごとにインクリメントし、`unix_socket_rx_seq()` で公開する。これが
  「到着イベント」そのもの。
- `Core/syscall/Syscall_Epoll.c`: `epoll_entry_t` に `last_seq` を追加し、
  ET の配送条件を
  `(ready & ~last_ready) | (seq != last_seq ? ready : 0) | (ready & (ERR|HUP))`
  にした。カウンタを持たない種類の fd は `seq` が常に 0 なので従来動作。

効果: X の往復レイテンシがサブミリ秒に戻り、Chromium が `PutImage` で
ピクセルを送り始めた。

#### 失敗した試み — epoll/poll/select の「カーネル内待機ループ」

`epoll_wait` が 1 ms ごとに 0 を返してユーザ空間から再発行される形が
syscall の無駄だと考え、syscall 内で「スキャン→park→再スキャン」を
回す形に書き換えた。**これは悪化させた**（Xorg の起動が 20 秒 → 170 秒、
X の往復が 2〜11 秒）。

理由: **`process_sleep_current_ms()` はブロックしない。** BLOCKED を立てて
起床期限を記録し `process_scheduler_request_reschedule()` を呼んで*戻る*
だけで、実際の切り替えはユーザ空間に戻る途中で起きる
(`process_run_next_on_current_cpu()` は `enter_user_mode()` への片道
ジャンプで、呼び出しから戻らない)。したがってこれをループで囲んでも
スリープはせず、**プロセスを BLOCKED と表示したまま全速でスピンし、
毎周で自分の起床期限を書き換える**。要求はソケットに載ったまま、サーバは
CPU を燃やして「待っている」ふりをする。

`Syscall_Epoll.c` 冒頭のコメントにこの理由を明記した。1 周だけ park して
0 を返す既存の形が、このスケジューラでは正しい。

#### 診断の追加（すべて既定で無効）

`-DPROCESS_STALL_DUMP=1` に加えて:
- ハートビートを**スレッド単位**にした（従来は `process_get_current_pid()`
  = メモリ所有者単位で、Chromium の全スレッドが 1 スロットに集約されて
  「chrome は生きている」以上のことが分からなかった）。スレッド名・最後の
  syscall 番号・第1引数・ユーザ復帰 RIP・累計回数を出す。
- `-DPROCESS_STALL_DUMP_FDS=1` で `[epoll]`（各 epoll セットの fd と現在の
  readiness）、`[poll]`（空振りした poll の fd セット）、`[usock]`（各
  AF_UNIX 端点の滞留バイト数）、`[wire]`（直近 48 件の送受信と先頭 4 バイト、
  ms タイムスタンプ付き）を出す。X プロトコルを直読みできるので、
  「要求が届いていない」のか「応答が来ていない」のかが確定する。
  **タイマ割り込みからの大量シリアル出力なので、測定したい時間そのものを
  歪める。計測用の実行では必ず切ること。**

#### (c) LAPIC タイマの校正が PIT 依存で、ブートごとに最大 17.7 倍ずれていた

`lapic_switch_to_local()` (`Arch/x86_64/timer/LAPIC_Timer.c`) は LAPIC タイマ
の周波数を PIT ティック 10 回分で測っていた。問題は 2 つ:

1. `g_ticks` をタイマ開始**前**に読んでいた（それだけで約 10% 誤差）。
2. より致命的に、**`g_ticks` が 10 進むことは実時間が 10 周期経ったことを
   意味しない**。起動中は割り込み禁止区間が長く、その裏に溜まった PIT 割り
   込みが解除直後にまとめて到着する。窓が数十マイクロ秒に潰れ、`elapsed`
   が極小になり、LAPIC にその分だけ短い周期が設定される。
   `elapsed < 1000` のガードは 100 倍の誤りを通してしまう。

実測（QEMU, SMP=4, 要求 250 Hz）:

| 実行 | 実際のティック率 | 要求比 |
|---|---|---|
| A | 250 Hz | 1.00x |
| B | 286 Hz | 1.14x |
| C | **4439 Hz** | **17.7x** |

4439 Hz は割り込み処理だけでマシンを潰す。**Chromium の起動時間が同じ
イメージで一桁ばらついていた主因はこれ。**

修正: HPET で校正する（自由走行カウンタなので遅延も滞留もしない。
`timer_monotonic_ns()` が既に信頼している）。50 ms の窓で LAPIC カウンタの
減少量を測り、`per_second / g_timer_hz` を周期に使う。HPET が無い機械では
従来の PIT 方式に落ちるが、その場合もティックはカウンタ開始**後**に読む。

効果: `initial` がブート間で 249,864 / 249,874（誤差 0.004%）に安定。
ティック率も要求比 1.014x に収まった（従来 1.00〜17.7x）。

#### (d) ゲストの CLOCK_MONOTONIC がティックカウンタ由来で約 14.5% 速かった

`clock_monotonic_now()` は `timer_ticks() / timer_hz()` を使っていた。これは
「想定した割り込み率」の精度しかなく、(c) の校正ずれもそのまま乗る。HPET
と比べて約 14.5% 速く、Xorg の起動直後のログが `[ 886.996]`（実際は 200 秒
程度）になっていた。

修正 (`Core/syscall/Syscall_Clock.c`, `Syscall_Futex.c`, `Syscall_File.c`):
- MONOTONIC 系はすべて `timer_monotonic_ns()`（HPET 優先）由来にした。
- futex タイムアウトと timerfd の「起動からの ms」も同じ基準に統一。
  同じ時計を読んでから相対タイムアウトを指定する呼び出しが正しく動く。
- `clock_getres` は HPET がある場合に 1 µs を報告する（従来はティック周期
  = 4 ms。これを丸め単位に使う呼び出しがティック単位で余分に待っていた）。

効果: Xorg の初回ログが `[ 886.996]` → `[ 17.465]`、400 秒の実行で最終
タイムスタンプが実時間と一致（362 秒）。

#### 効果（(c)+(d) 適用後）

Chromium がブラウザ起動を完走するようになった。同一実行のログで:

- Blink レンダラが JS モジュールを実行（`modulator_impl_base.cc`）
- ツールバー WebUI を構築（`chrome://webui-toolbar.top-chrome/strings.m.js`）
- Privacy Sandbox の attestation を 263 件パース
- プロファイルの LevelDB を開く

それ以前は `scheduler_loop_quarantine_config` の 2 行で止まっていた
（ログ 7 行 → **260 行**）。

#### 現状と残り

- SMP=4 で Chromium のウィンドウ全面が描画される（起動から約 78 秒）。
  修正前は SMP=2 で約 420 秒、SMP=4 では届く前に落ちていた。
- ページ内容（ツールバー・about:blank の中身）までは未到達。
- 描画直後に `Chrome_InProcRendererThread` が user-mode #GP で落ちる
  (`rip=0x400EA9B161` — ImplusOS ネイティブ userland のコード領域)。
  (e) で Chromium 側の NOTREACHED は消えたので、これは別要因。次の当たり所。
- **カーネル側の堅牢性**: 上の #GP で chrome が exit した直後、
  `[proc] address space kept: threads still running on other CPUs` の状態で
  残存スレッドが同じ命令をフォルトし、ユーザフォルトとして処理されずに
  カーネルパニック（`[OS] [PANIC] Fatal exception / general_protection`）に
  なる。死にかけのプロセスのスレッドは終了させるべきで、パニックさせて
  はいけない。
- 進行が約 14.5 秒周期で途切れていたのは (c) のタイマ暴走が主因。
- SMP=4 で「静かなリセット」は**再現しなかった**（`-d cpu_reset` の 8 件は
  すべて起動時の power-on / AP INIT）。
- AP のタイマ割り込みは `lapic_timer_handler()` / `timer_core_handler()` が
  CPU0 以外を早期 return するため捨てられている。`process_on_timer_tick()`
  は CPU0 でしか走らないので、AP 上の CPU バウンドなスレッドがプリエンプト
  されるかは割り込み復帰パス依存。未検証。ここは次の候補。
- headless (`--headless=new --dump-dom`) は Chromium 自身の
  `FATAL: ui/gl/init/gl_factory_ozone.cc:62] NOTREACHED hit. Expected Mock
  or Stub, actual:0` で落ちる。SwANGLE が `DisplayVkXcb` を選び
  `xcb_connect()` に失敗して GL 実装が None になるため。`--use-gl=stub`
  を渡すのが筋（未検証）。

### 10.-4 追記 (2026-09-08, 夜) — 停止中のスレッド状態を実測。inotify は無関係だった

`-DPROCESS_STALL_DUMP=1`（タイマ駆動。syscall が止まると既存の
`[hb]` ヒートビートは出ないので、タイマから叩く）で停止中の全スレッドの
状態と最後に発行した Linux syscall 番号を採取した。結果:

```
[stall] free=0xD083D  0:s2 1:s6 2:s6 3:s6 4:s6 5:s6
        6:s2:#0xE8:n0x7BD2      <- Xorg,    epoll_wait を 31,698 回
        7:s1:#0xE8:n0x2C69      <- chrome,  epoll_wait を 11,369 回
```

（`s` は `PROCESS_STATE_*`: 1 READY / 2 RUNNING / 6 BLOCKED。`#` は最後の
syscall 番号、`n` は発行回数。）

わかったこと:

- **Xorg も Chromium も `epoll_wait`(232) を回し続けている**。デッドロックでは
  なく、イベントループの往復レートがそのままスループットの上限になっている。
  他のスレッドは全部 BLOCKED で、正常。
- **`inotify` は無関係**だった（§10.-3 の推測は外れ）。止まる位置が
  `file_path_watcher_inotify.cc` の直後に見えたのは、単にそれ以降ログを出す
  コードが少ないだけ。
- **メモリ枯渇でもない**。停止中も空き 3.2 GB（`free=0xD083D` ページ）。

**計測環境の注意**: ゲスト RAM を 8 GB にすると 1.08 GB の ISO がホストの
ページキャッシュから追い出され、起動時間が 15 秒 → 200〜300 秒に化ける。
`-m 4096` にすると再現性のある数字が出る（`MEM=4096`）。

**現状**:

| 構成 | 挙動 |
|---|---|
| デスクトップ `-smp 4` / `-smp 16` | 21〜26 秒で起動、安定 |
| Chromium `-smp 1` | 落ちないが遅い。ログ 1 行 / 実時間 1 分程度で進む。signin までは到達、描画には未到達 |
| Chromium `-smp 2` 以上 | Xorg/Chromium の起動中に無音でリセット（トリプルフォルト）。再現は非決定的 |

**残る 2 つの課題**（どちらも未解決）:

1. **epoll/poll の往復レート**。`EPOLL_POLL_SLICE_MS`(8) と
   `POLL_WAIT_MAX_DECLINES`(1) の組み合わせで、待ちの半分が必ずスライス分
   眠る。両方を緩める実験はしたが、当時は下記 2 の影響と混ざって評価できな
   かった。2 を先に潰してから再評価するのが正しい順序。
2. **`-smp >1` での無音リセット**。二重フォルトのパニック出力すら出ないので
   トリプルフォルト。以下は実測で**排除済み**:
   - TLS の取り違え（全スレッドの `fs_base` は相異なる）
   - syscall のスピン（400 秒で 100 万回未満）
   - ELF ロードのコスト（exec → `main()` が 6 秒）
   - AP の IST/TSS 未設定（`ap_entry_c()` が per-CPU に設定済み）
   - 物理メモリ枯渇（3.2 GB 空き）
   - カーネルスタックの use-after-free と、実行中アドレス空間の破棄
     （どちらも §10.-2 で修正済み。それでも残る）

### 10.-3 追記 (2026-09-08, 後半) — 描画には未到達。止まる位置は毎回同じ

GUI（`--ozone-platform=x11`）でウィンドウ枠は出るが、**ページは描画されない**。
`-smp 4` で観測される停止位置は毎回まったく同じで、Chromium のログは

```
ERROR:base/files/file_path_watcher_inotify.cc:925] Failed to read /proc/sys/fs/inotify/max_user_watches
ERROR:dbus/bus.cc:405] Failed to connect to the bus: ... /run/dbus/system_bus_socket
ERROR:dbus/bus.cc:405] Failed to connect to the bus: ... /nonexistent
```

の 3 行で止まる（Xorg 側は生きていて dbus 再接続を 10 秒ごとに出し続ける）。
そこから先へ進まないか、20〜60 秒後にゲストが無音でリセットする。

**次に当たるべき有力な線**: `inotify` が**イベントを一切配送しない**こと
（§4）。`inotify_init` は「空マスクの signalfd」＝**永久に readiness が立たない
ディスクリプタ**を返す実装なので、そこを待つスレッドは永久に待つ。止まる位置が
`file_path_watcher_inotify.cc` の直後で毎回一致するのは偶然にしては出来すぎて
いる。VFS に変更通知のフックを入れて実イベントを流すか、少なくとも
`inotify_init` の fd を「常に readable（読むと 0 件）」にして待ちが解ける形に
するのが最短の検証になるはず。

**排除できた仮説**（いずれも実測で確認済み、再調査不要）:

- TLS の取り違え — `-DTHREAD_TLS_TRACE=1` で全スレッドの `fs_base` を出力。
  すべて相異なり、8.4 MB 間隔で正しく並んでいた。
- syscall のスピン — `-DLINUX_SYSCALL_PROFILE=1` で計数。400 秒で 100 万回に
  届かない。
- ELF ロードのコスト — exec 発行から Chromium の `main()` 到達まで **6 秒**。
- OS の起動 — 電源からデスクトップまで **14〜20 秒**（`-smp 4` / `-smp 16`）。
- AP の IST/TSS 未設定 — `ap_entry_c()` は `init_gdt()` /
  `init_idt_per_cpu()` を呼んでおり、#DF は IST1 で per-CPU に張られている。

### 10.-2 追記 (2026-09-08) — 起動時間の内訳と、SMP のリセット源 2 件

**まず計測した**（`-smp 4`、ホストのページキャッシュを温めた状態）:

| 区間 | 実時間 |
|---|---|
| 電源 → デスクトップ（loginui が WM を起こす） | **14〜18 秒** |
| → `chrome` の exec 発行 | +4 秒 |
| → Chromium の `main()` 到達（465 MB の ELF ロード＋ld.so 完了） | **+6 秒** |
| → ブラウザ初期化開始 | +0 秒 |

つまり **OS の起動も 465 MB の ELF ロードも遅くない**。当初「9 分かけても
起動しない」と書いたのは誤りで、実体は次の 2 つだった:

1. **ホストのページキャッシュ**。1.08 GB の ISO を焼き直した直後の初回起動は
   195〜296 秒かかるが、2 回目は 16 秒。イメージを作り直したあとに計測すると
   ゲストが遅いように見える。計測前に `dd if=... of=/dev/null` で温めること。
2. **ゲストのリセット**。Chromium 起動の 4〜60 秒後に、シリアルに何も出さずに
   マシンが再起動していた（＝トリプルフォルト）。

`[sysprof]`（`-DLINUX_SYSCALL_PROFILE=1`）で syscall 数を数えたところ、
400 秒で 100 万回に届かない。**syscall ループで焼いているのではない**。

**リセット源として直したもの**:

- **カーネルスタックが使用中に解放されていた**。スレッドを reap する CPU と、
  そのスレッドをまだ実行していた CPU は別で、`free()` されたスタックのページが
  即座に他の用途へ回っていた。CPU ごとに 1 個だけスタックを退避する仕組み
  (`g_parked_stack`) はあったが、埋まっていると次の死亡は無防備になる。
  **カーネルスタックはプロセススロットが保持して再利用する**ようにした
  （解放しない）。スレッド生成から 128 KiB の malloc も 1 つ消える。
- **実行中のアドレス空間が破棄されていた**。`release_process_resources()` は
  他 CPU で実行中のスレッドを `continue` で飛ばしたあと、共有 CR3 の
  ページテーブルを**無条件に** `paging_destroy_process_space()` していた。
  Chromium は数十スレッドを 4 CPU に散らしたまま終了するので毎回踏む。
  実行中のスレッドが残っている場合は破棄を見送る（数ページのリークと引き換えに
  リブートを避ける）。

**それでも Chromium はブラウザ初期化の途中で極端に遅くなる**、あるいは
V8 のレンダラスレッドで `Check failed: isolate_->...CentralStack...` に当たる。
GUI（`--ozone-platform=x11`）ではウィンドウ枠までは出るが、ページは描画されない。

### 10.-1 追記 (2026-09-07, 後半) — **SMP のメモリ破壊は解消**

原因は SMP レースではなく、**アドレス指定の取り違え**だった。

`Arch/x86_64/mmu/Paging_Main.c` の `resolve_pd_table(cr3, pdpt_index)` は
`pdpt_index = (addr >> 30) & 0x1FF` だけでページディレクトリを引いていた。
つまり **PML4 インデックスを見ていない**。すべてのユーザ写像が 512 GiB 未満に
あった頃はそれで足りたが、mmap アリーナが `USER_MMAP_BASE`（1 TiB =
PML4 スロット 2）へ移った時点で破綻していた:

- アリーナのアドレスが **PML4 スロット 0 側の無関係なアドレスの PD** に解決される
- `paging_unmap_range()` はその PD を歩き、**実行体がそこに張っていたフレームを
  解放**し、最後に PD エントリごと 0 にする
- 2 MiB 単位で写像が消え、次に触ると demand-zero の新品ゼロページが返る
  → NULL ポインタ、ゼロで埋まった構造体、ゼロ領域への jmp

PartitionAlloc はアリーナを絶え間なく munmap するので、被害量は
「Chromium がどこまで進んだか」に比例した。`-smp 1` で無傷だったのは
レースだからではなく、単に遅くてそこまで到達しなかったから。

**確認方法**（再利用可能）: `-DPAGING_LOST_MAPPING_TRACE=1` でビルドすると、
コード領域 (`0x40_0000_0000`–`0x40_8000_0000`) に demand-zero フォルトが来た
時点で `[lostmap]` としてページテーブルの状態を吐く。実行体の PT_LOAD は
ELF ローダが eager にマップするので、ここに demand-zero が来ること自体が
「写像が消えた」証拠になる。`pde=0x0` かつ `pd_alloc=1`（テーブル自体は
確保済み）から、PD エントリが明示的に 0 にされたと特定できた。

**修正**: `walk_pd_table(cr3, virt_addr)` / `refresh_pdpt_entry()` を新設し、
実ページテーブルを PML4 から歩くようにした。PML4 盲目だった
`resolve_pd_table()` と `update_pdpt_user_flag()` は削除（同じ罠を再度
踏まないため）。

破壊が止まった結果、Chromium は以前落ちていた地点を越え、そこから先は
**機能不足**で止まるようになった。以下は続けて直したもの:

- **`getrlimit(2)` が read-only ページへ書けてしまう** — Chromium の
  `base::internal::CheckMemoryReadOnly()` は「`mprotect(PROT_READ)` した
  ページを `getrlimit` の出力バッファに渡し、**EFAULT が返ること**」で保護を
  検証し、返らなければ `CHECK` で落とす
  (`protected_memory_posix.cc:46`)。カーネルは読み取り専用のユーザページにも
  書けてしまうので素通りしていた。`paging_user_range_is_writable()` /
  `process_user_buffer_is_writable()` を追加し、`getrlimit`/`prlimit64` の
  出力と `read(2)` の書き込み先で検査する。
  *`copy_to_user()` 全体でこれを行うのが本筋だが、データを返すすべての
  syscall にページテーブル walk が乗って体感で遅くなるため見送っている。*
- **共有メモリの mmap が毎回同じアドレスを返す** — `shared_memory_map()` は
  同一アドレス空間からの 2 回目の map で既存の写像を使い回す（ネイティブの
  `SYS_SHM_MAP` は冪等な契約で、コンポジタがそれに依存している）。しかし
  `mmap(2)` はそうではなく、Chromium の `base::SharedMemoryTracker` は
  マップ先アドレスをキーに記録して unmap 時に `CHECK` する
  (`shared_memory_tracker.cc:62`)。Linux 経路だけ `shared_memory_map_new()`
  で毎回新しい範囲を張るようにした。

- **`memfd_create` / `eventfd2` がフラグを検証していない** — Mojo は
  `ChannelLinux::KernelSupportsUpgradeRequirements()` で
  `memfd_create(name, 0xFFFFFFFF)` と `eventfd2(-1)` を**わざと不正なフラグで
  呼び**、EPERM/EINVAL/ENOSYS のいずれかで失敗することを要求する
  (`channel_linux.cc:947`)。フラグを無視して成功していたので `CHECK` で落ちて
  いた。既知ビット以外は EINVAL にし、`MFD_CLOEXEC` も反映するようにした。
- **memfd のシールが強制されていない** — `F_SEAL_WRITE`/`F_SEAL_FUTURE_WRITE`
  を記録するだけで `write()` を止めていなかった。また封印された `ftruncate`
  は Linux では **EPERM** だが EACCES を返していた（Mojo の検査は EPERM 系し
  か受け付けない）。

**現在の到達点 (`-smp 4`, headless)**: メモリ破壊なし（`[lostmap]` 0 件）、
カーネルパニックなし、Chromium の FATAL なし。ブラウザは
`WebContentsViewAura` すなわち実際のコンテンツ層まで進むようになった。
そこから先は **V8 のレンダラスレッドで
`Check failed: isolate_->...CentralStack...`** で停止する（`ImmediateCrash()`
の `int3`/`ud2` を踏む）。V8 がスレッドのスタック境界を
`pthread_getattr_np()` 経由で取得し、それが実際の RSP と合っていないのが
疑わしい（`/proc/self/maps` の stack 行は固定定数から合成している。§3.3）。

**速度も課題**。`-smp 4` で 9 分回しても起動が終わらない。支配的なのは
epoll/poll の 8ms スリープ（`Syscall_Epoll.c` の設計上の制約、§3.4）と、
ブロックデバイス I/O。

### 10.0 追記 (2026-09-07, 前半) — SMP 修正 2 件（破壊は当時継続）

**修正 1: USB マスストレージの転送に排他が無かった**（`Drivers/Bus/USB/USB_Main.c`）。
`MassStorage.c` は全コマンドで **1 個のバウンスバッファ**・CBW タグ・
選択中デバイス番号を共有し、しかも CBW → データ → CSW の 3 段転送を行う。
`filemap_handle_fault()` は（ブロックドライバがブロックしてよいように）
意図的にロックを落としてから読むので、**複数 CPU が同時にこのドライバへ入る**。
結果、シーケンスが交錯し、互いのバウンスバッファの中身を読み取る＝
**ページに他のフォルトのデータが入る**。デマンドページングが増えるまで顕在化
しなかっただけで、Chromium (465 MiB) はこれを大量に踏む。
`usb_storage_read/write/flush` を「譲るロック」で直列化した（転送中に
`timer_msleep()` でスケジューラへ譲るため、スピンロックでは自己デッドロック
する）。直列化のコストは `FILEMAP_READAHEAD_PAGES` を 16→64（256 KiB）に
拡大し、`pmm_alloc_pages()` 失敗時に 1 ページへ落ちるのをやめて半減リトライに
することで相殺した。

**修正 2: TLB シュートダウン**（§10.1、下記）。CPU ごとの要求スロット＋
確認応答待ちに作り直し、**待ち先を「その CR3 を実際に載せている CPU だけ」に
限定**した（PCID 無しの x86 では CR3 ロードで TLB 全体がフラッシュされるので、
他のアドレス空間を走らせている CPU は古い翻訳を持ち得ない）。計測では
IPI 1 回あたりの待ちは平均 ~540 スピン、最大 ~87k スピンで、実用上問題ない。

**測定結果 (`-smp 4`, headless, `--dump-dom about:blank`)**:

| | 修正前 | 修正後 |
|---|---|---|
| デスクトップ起動 (`-smp 4` / `-smp 16`) | 可 | 可（回帰なし） |
| `-smp 1` の Chromium | 落ちない | 落ちない（回帰なし） |
| `-smp 4` の Chromium | quota DB 付近で SIGSEGV | **同じ深さまで到達するが依然 SIGSEGV** |

つまり **SMP のメモリ破壊はまだ残っている**。上の 2 件はいずれも実在する
データ破壊バグで、直す価値はあったが、Chromium を殺している主因ではなかった
（あるいは主因の一部でしかなかった）。次に当たるべきところは §10.3。

### 10.1 TLB シュートダウンが非同期かつ排他なし（修正済み）

`Arch/x86_64/smp/SMP_Main.c` の `smp_tlb_shootdown()` には**独立した 2 つの
欠陥**があった。

1. **確認応答を待たない。** IPI を投げてすぐ戻る。呼び出し元はその直後に
   ページを解放したり、保護を狭めたり、アドレス範囲をアロケータへ返したり
   する。他の CPU はまだ古い翻訳を持っているので、そこへ書き続ける。
2. **要求スロットがグローバルに 1 個で排他がない。** 2 CPU が同時に
   シュートダウンすると `vaddr`/`pages` を上書きし合い、片方のフラッシュが
   丸ごと失われる。

Chromium は数十スレッドが 4 CPU の上で mmap/mprotect/munmap を絶え間なく
呼ぶので、両方が常時起きる。`-smp 1` で無傷、`-smp >1` で破壊、という観測と
完全に一致する。

**対処**: `smp_tlb_shootdown_cr3(cr3, vaddr, pages)` に作り直した。

- CPU ごとの要求スロット `g_tlb_slot[]` ＋応答行列 `g_tlb_seen[][]` で (2) を解消。
- 全 CPU が応答するまで戻らない（(1) を解消）。
- 待ち先は `g_cpu_cr3[]`（`paging_switch_cr3()` が公開）と照合し、**その
  アドレス空間を載せている CPU だけ**に限定。PCID 無しの x86 では CR3 ロードで
  TLB 全体がフラッシュされるので、これで正しさは保たれ、16 vCPU 構成でも
  待ちが現実的になる。
- 待機中の CPU は `tlb_service_peers()` で相手の要求も処理するので、
  同時シュートダウンでデッドロックしない。
- `spinlock_lock()` は待機中（64 回に 1 回）`smp_tlb_poll()` を呼ぶ。この
  カーネルのスピンロックはほぼ全て割り込み禁止で取られるため、待っている
  CPU はシュートダウン IPI を受け取れず、そのままでは送信側と待ち合う。

**重要な副次的発見**: 「権限を広げる方向の変更ではシュートダウンを省く」
最適化を入れると Chromium が前進しなくなる。古い制限的な TLB エントリが
残ったままフォルトを繰り返すためで、`paging_access_is_now_permitted()` の
偽フォルト処理だけでは足りない。**PTE が変化したら必ずシュートダウンする**
こと（`paging_protect_user_range` は 1 ページごとではなく範囲でまとめて 1 回）。

### 10.2 調査済みで**シロ**だったもの

- `process_user_reserve()` / `process_user_alloc()` — いずれも
  `g_process_table_lock` の下でバンプ割り当て。重複配布はない。
- コンテキストスイッチでの `fs_base`/`gs_base` 退避・復元
  (`process_schedule_*` ↔ `activate_process_context()`) — 対になっている。
- 偽ページフォルト — §上記 8 で対処済み（単体では SMP 破壊は止まらない）。

### 10.3 次に当たるところ

- `Syscall_File.c` の fd テーブル: `syscall_file_read()`/`write()` などが
  `g_file_table_lock` を取る前に `g_files[fd].used` を読む箇所がある。
- `paging_map_user_page()` のシュートダウン条件
  （`replaced_live_mapping` のときだけ）。
- `filemap_handle_fault()` の同一ページ同時フォルト（2 CPU が同じページを
  同時に埋める経路）。

再現手順:

```bash
make image_livecd AUTOSTART=/Userland/Chromium/Chromium.ELF
# 落ちる（15〜90 秒）
make run_uefi_usb QEMU_DISPLAY=none
# 落ちない
make run_uefi_usb QEMU_DISPLAY=none QEMU_SMP=1
```
