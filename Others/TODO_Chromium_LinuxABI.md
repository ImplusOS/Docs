# ImplusOS に Chromium を実用動作させるための TODO リスト

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