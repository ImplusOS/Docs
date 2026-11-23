# ImplusOS で Firefox を動かす — 記録と残課題

`Userland/Application/FireFox` は、無改変の Linux 版 Firefox（Mozilla の
リリース tarball）を Chromium と同じ「方法 A」（Xorg を XSession が WM
ウィンドウに載せる経路、`TODO_Doom_Xorg_MethodA.md` §10）で動かす
ランチャーである。ランチャー・デスクトップエントリ・アイコンは以前から
あったが、`Resource/` に Firefox 本体が無く、置いても起動しなかった。

## 1. 入手と配置 (2026-11-23)

- 版: **Firefox 160.0.1, linux-x86_64, ja**（2026-11-20 リリース）。
- `make -C Userland/Application/FireFox fetch` が Mozilla から取得し、
  sha256 を照合して `Resource/` に展開する（版・ロケール・sha256 は同
  Makefile にピン留め）。`Resource/` は `.gitignore` 対象で、通常の `make`
  はネットワークに触れない。
- arm64 でも同じ x86_64 版を同梱し、トランスレータ
  （`Compat_Layers.md` §7）で動かす。
- 依存ライブラリ（GTK3, libX11 系, libasound, libdbus, fontconfig …）は
  既存の glibc 閉包（`Vendor/LinuxRuntime`）にすべてある。Firefox 自身の
  `.so` は `dependentlibs.list` を自分のディレクトリから dlopen するので
  `LD_LIBRARY_PATH` に足す必要は無い。
- ランチャーは tarball の `firefox`（`/proc/self/exe` に `-bin` を付けて
  execv するだけの 5.5 KiB のスタブ）ではなく `firefox-bin` を直接起動する。
  スタブは tarball で唯一の非 PIE（lld の既定 0x200000 にリンク）で、
  arm64 のトランスレータはこのアドレスにマップできない（§2.4）。
- プログラムプロファイル `Environment/firefox-bin.env`: `GDK_BACKEND=x11`,
  `MOZ_ENABLE_WAYLAND=0`（XSession のディスプレイは X だけ）。

初回起動では Firefox の「ようこそ」ダイアログ（利用規約・プライバシー通知
への同意）が出る。これは Firefox の正常動作で、「続行」を押せば閉じる。

## 2. カーネルとトランスレータで直したもの

### 2.1 フォークサーバーが即死する — recvmsg の制御バッファ上限

症状: 親プロセスの UI は出るが、タブが「タブクラッシュレポーター」になる。

```
IPDL protocol error: File handle not found in message!
[OS] [PF] ... name=forkserver   (MOZ_CRASH: *(int*)0 = __LINE__)
WARNING: Failed to launch tab subprocess
```

Firefox 160 は Linux で子プロセスをフォークサーバー経由で作る。親は
起動要求を `sendmsg(SCM_RIGHTS)` で送り、フォークサーバーは
`MiniTransceiver::RecvData()` で受ける。ここで Firefox は制御バッファの
大きさに `kMaxNumFds` ではなく `kMaxDataSize` を渡しており、
`CMSG_SPACE(4 * 8192)` = 32 KiB の `msg_controllen` で `recvmsg` する。

`IPC/UnixSocket.c` は 4096 バイトを超える制御バッファを「無い」扱いにして
いたため、送られた fd を全部捨てていた。Linux は大きな制御バッファを
受け付け、入るだけ詰めるだけである。修正: `usock_control_len()` で先頭
4096 バイトに切り詰めて使う（4096 バイトでも fd は 1020 本入り、
`UNIX_SOCK_FD_MAX` を十分超える）。送信側も同じ扱いにした。

### 2.2 コンテンツプロセスが即死する — memfd の seal が SCM_RIGHTS で消える

2.1 の修正後、子プロセスが次で死ぬ:

```
IPDL protocol error: Shared memory PlatformHandle is not safe to map
```

Firefox は受け取った共有メモリ fd を `IsSafeToMap()` で検査し、memfd に
`F_SEAL_SHRINK` が付いていなければ拒否する。Linux では seal は inode の
属性なので fd と一緒に届く。ImplusOS の AF_UNIX は共有メモリ fd を
「shm ハンドル」として運び、受信側で新しい `kernel_memfd_t` を作るため、
seal が 0 に戻っていた。修正: `usock_passed.seals` に送信時の seal を
載せ、`usock_adopt()` で受信側の新しい memfd に付け直す。

### 2.3 Xorg が「Linking lock file … failed: Invalid argument」で起動しない

アプリ反復テスト（§3）で、強制終了した Xorg が残した `/tmp/.X0-lock` が
あると次の Xorg が起動しなかった。Xorg の `LockServer()` は一時ファイルを
`link()` でロックファイルに付け、**EEXIST のときだけ** 古いロックかを
調べて取り除く。`syscall_file_link()` は既存の宛先に `OS_STATUS_INVALID_ARG`
（EINVAL）を返していた（コメントには EEXIST とあった）。修正:
`OS_STATUS_ALREADY_EXISTS`（-17）を `kernel/status.h` に追加し、それを返す。

### 2.4 2 回目の起動が「paging map failed (load segment)」で失敗する

起動直後の 1 回目は動き、閉じてもう一度起動すると、ランチャーが何も
言わずに終了 1 していた（2.5 のログでやっと理由が出た）。

`firefox` スタブは ET_EXEC で 0x200000 に固定リンクされている。x86_64 の
低位 4 GiB はカーネルの恒等マップで、プロセス専用ページで覆ってよいのは
起動時に物理フレームを取り置いた範囲（BusyBox 用の 0x400000〜0x800000 など）
だけだった。0x200000 は対象外で、`paging_map_user_range_alloc()` は
「その物理フレームが今たまたま空いていれば借りる」方式で通していた。
1 回目は空いていたが、終了時に返したフレームを次の割り当てが使うと、
2 回目は借りられずに失敗する。

修正: `PAGING_LOW_ET_EXEC_*`（0x200000〜0x400000、`Arch/x86_64/mmu/Paging_Main.h`）
を追加し、`kernel_main` がヒープ作成前に、そのフレームが全部空いていれば
取り置いて shadow window にする（空いていなければ従来どおり）。lld で
リンクした非 PIE は既定でここに置かれるので、Firefox 以外にも効く。
ランチャーは `firefox-bin` を直接起動するようにもした（arm64 対策、§1）。

### 2.5 arm64 でだけ起動直後に落ちる — /proc/self/maps にゲストのスタックが無い

arm64（x86_64 版 Firefox をトランスレータで実行）では、gfxtest を起動した
直後に親プロセスが NULL 書き込みで落ちていた（JIT を切っても同じ）。
`XL_STRACE=1` で見ると、直前に `/proc/self/maps` を読み、`RLIMIT_STACK` を
取っている。glibc の `pthread_getattr_np()` がメインスレッドのスタックを
探す処理で、`__libc_stack_end` を含む行が無いと失敗し、SpiderMonkey が
`MOZ_RELEASE_ASSERT(stackBase)` で意図的に落ちる。

カーネルの `/proc/self/maps` はカーネル自身が配置する [code]/[heap]/[stack]
の 3 行だけを返す。x86_64 ネイティブではゲストのスタックが [stack] の範囲に
あるので見つかるが、トランスレータはゲストのスタックを mmap で別に確保する
ので、どの行にも入らない。

修正（`Userland/Application/Translator/Source/syscall.c`）: ゲストが
`/proc/self/maps`（`/proc/thread-self/maps`, `/proc/<自 pid>/maps`）を
開くと、ホストの内容にゲストスタックの行をアドレス順に挟んだものを memfd で
返す（`/proc/self/exe` の横取りと同じ場所）。ホスト側のスタック行の名前は
`[xlstk]` に変える。

### 2.6 spawn の失敗理由をログに出す

`process_spawn_user_elf_with_env()` の失敗経路（環境変数が大きすぎる・
プロセススロットが無い・ELF ロード失敗）は無言で -1 を返し、ランチャーは
何も言わずに終了 1 していた。`[app] spawn failed: <path>: <理由>` を出す
（ELF ロード失敗は `elf_loader_last_error()` の内容）。

## 3. Linux アプリの反復テスト

テスト用ドライバ（イメージには入れていない）を AUTOSTART で起動し、
各 Linux アプリを順に起動 → 一定時間動かす → WM の閉じるボタンと同じ
経路（`WM_CLOSE_REQUESTED` → XSession → `WM_DELETE_WINDOW` / KillClient）
で閉じる → ランチャーの終了コードと残留プロセス・空きメモリを記録、を
3 ラウンド繰り返し、最後に Firefox / Chromium / Terminal / Doom / gtk3-demo
を同時に起動する。対象は Start メニューにある Linux アプリ全部:
Firefox, Chromium, Terminal, XTerm, UXTerm, Doom, Wine (winecfg),
Wine (explorer), gtk3-demo, gtk3-widget-factory, gtk3-icon-browser,
FastFetch。

テストで見つかって直したもの（2.3〜2.6 のほか）:

- **強制終了した Xorg のロックで display 番号が使えなくなる**: 2.3 で
  EEXIST を返すようにしても、Xorg はロックの pid が生きていれば「使用中」と
  判断する。ここでは pid がすぐ再利用され、次の Xorg 自身が同じ pid を
  もらって自分のディスプレイを拒否していた（`Server is already active for
  display 0`）。`XSession.c` の `clear_stale_lock()` が、誰も listen して
  いないディスプレイについて、ロックの pid が生きた `Xorg` でなければ
  ロックとソケットを消してから Xorg を起動する。
- **所有プロセスが死んだウィンドウが閉じられないまま残る**: WM は持ち主の
  死を知る手段がなく、ランチャーが異常終了すると、そのウィンドウが画面に
  残った。XSession のウィンドウは閉じるボタンが持ち主への
  `WM_CLOSE_REQUESTED` になるので、押しても何も起きない。WM が 1 秒ごとに
  `wm_scene_reap_orphaned_windows()` で持ち主を確かめ、いなければ閉じる
  （pid の再利用に備えて作成時のプロセス名も照合し、閉じる際は pid を
  消してから閉じるので、同じ pid の別プロセスに通知も kill もしない）。

- **UXTerm が起動しない**: Debian の `debian-uxterm.desktop` の
  `Exec=uxterm` はシェルスクリプトで、イメージに無い（あっても ELF しか
  spawn しない汎用ランチャーでは動かず、locale/sed/awk も要る）。
  `windowmanager/Resource/Applications/debian-uxterm.desktop` で上書きし、
  スクリプトが UTF-8 ロケールで最終的に行う
  `xterm -class UXTerm -u8` を直接実行する。
- **macOS ホストでビルドすると GTK3 アプリが全部 SIGABRT**:
  `Gtk:ERROR:gtkiconhelper.c:495 … Unrecognized image file format`。
  macOS ではステージングが Linux の `update-mime-database` を実行できず
  `mime.cache` が作られない。gdk-pixbuf は形式判定を GIO（shared-mime-info）
  に任せるので、PNG すら判定できない。ステージングは既にホストの
  `update-mime-database` を使う代替経路を持っていたので、
  `brew install shared-mime-info` を CLAUDE.md の macOS 手順に加えた。

- **exit_group() の後も他スレッドが走り、閉じた fd を読む**:
  gtk3-widget-factory は合成器が消えると `_exit(1)` するが、終了コードが
  133（SIGTRAP）になった。`process_exit_current()` は fd を閉じる
  `process_teardown()` を先に実行し、兄弟スレッドを止めるのはその後だった。
  別 CPU で走っていた GLib の gmain スレッドが閉じた inotify fd を読み、
  EBADF を致命エラー（G_BREAKPOINT）にしていた。修正: 兄弟スレッドを先に
  DEAD にしてから（poll で寝ているスレッドは fd が閉じても二度と走らない）、
  今まさに別 CPU で走っているものが CPU を離れるのを最大 20 ms 待ち、
  それから teardown する。最初は「別 CPU にいる間は teardown を
  `process_run_pending_teardowns()` に回す」実装にしたが、CPU が止めた
  スレッドの pid を名乗り続けることがあり（`process_scheduler_release_stale_pid()`
  参照）、teardown が永久に走らなかった。親に終了が届かず Wine の
  ランチャーが閉じず、fd が閉じられないまま溜まって数回後には Xorg が
  `/tmp` にロックファイルを作れなくなった。待ちに上限を付け、teardown は
  必ずその場で実行する。あわせて、既に終了中のグループの終了コードを
  後から来たスレッドのシグナルや exit_group が上書きしないようにした。
- **標準入出力への `fcntl(F_GETFL)` が EFAULT**: 0/1/2 は表に項目の無い
  「コンソール」で、引けずに `OS_STATUS_FAULT` がそのまま返っていた
  （GLib: "Error while getting flags for FD: Bad address"）。未 close の
  0/1/2 は O_RDWR、それ以外の未使用 fd は EBADF を返す。

- **tmpfs が 4096 ファイルで満杯になり、以後 X のプログラムが一切起動
  しない**: `/tmp`・`/run`・`/var`・`/dev/shm` は 1 つの表（`TmpFS.c`）を
  共有する。Firefox が正常に動くようになると、そのプロファイルと HTTP
  キャッシュ、Chromium のプロファイル、Wine のプレフィックス（system32 だけで
  約 750 個の DLL）で表が埋まり、wineboot が system32 の途中から
  `create_dest_file failed ... (error=998)` で失敗（Wine のランチャーが
  「設定を更新中」のまま閉じない）、続いて Xorg が `/tmp/.tXn-lock` を
  作れず「no free X display」になった。上限を 16384 に上げ、パス検索を
  全スロットの線形走査からハッシュ表（`g_tmpfs_bucket`）に替えた。

- **arm64: Chromium を閉じた後に zygote が CPU 100% で残り続ける**:
  1 秒に約 80 万回のシステムコールで回っていた（`XL_STRACE=1` で見ると
  `ppoll` = 1 と `clock_gettime` の繰り返し）。相手が閉じた AF_UNIX
  ソケットに `unix_socket_poll()` が POLLHUP だけを返し、POLLIN を立てて
  いなかった。Linux の `unix_poll()` は EOF が読める状態として
  EPOLLIN | EPOLLRDHUP | EPOLLHUP を返す。zygote は POLLIN を見てから
  読むので、ブラウザが消えたこと（EOF）に気付けなかった。HUP と一緒に
  POLLIN（と要求があれば POLLRDHUP）を返す。x86_64 のテストでは
  この残留は出ていなかった（理由は未確認）。
- **arm64: マルチスレッドのゲストが fork した子がトランスレータ内部の
  ロックで永久に回る**: トランスレータの fork は「呼んだスレッドだけが子に
  残る」ホスト fork だが、コード範囲表・シグナル動作表・アトミック用の
  スピンロック（`xl_lock_t`）を他スレッドが握った瞬間に fork すると、子は
  最初の mmap などでそのロックを永久に待つ（コードを読んで見つけたもので、
  再現はしていない）。`xl_sys_clone()` の fork 経路で、pthread_atfork と
  同じく全ロックを取ってから fork し、親子両方で離す。

意図どおりで失敗ではないもの:

- Doom は WM_DELETE_WINDOW を持たないので KillClient で閉じられ、X の
  I/O エラーで終了コード 1 になる。
- x86_64 を macOS の TCG で動かすと、Wine の初回プレフィックス作成
  （wineboot）が Wine 自身の 5 分の待ちを超え、`boot event wait timed out`
  の後 winecfg が DLL を読めずに終わる（ランチャーは再試行する）。速度の
  問題で、arm64 (HVF) では起きない。

### 3.1 最終結果

| 環境 | 結果 |
|---|---|
| arm64（macOS HVF、x86_64 版をトランスレータで実行） | 12 アプリ × 3 ラウンド + 5 アプリ同時起動 = **41 件すべて PASS**、残留プロセス 0 |
| x86_64（macOS TCG） | Wine を除く 10 アプリ × 3 ラウンド + 5 アプリ同時起動 = **35 件すべて PASS**、残留プロセス 0、ユーザープロセスの異常終了ログ 0 |

x86_64 で Wine を外したのは 3 章末の通り速度の問題（初回の wineboot が
TCG では Wine 自身の 5 分の待ちを超える）。arm64 のログにある
winedevice.exe の data abort 5 件は、Wine が自分で例外として処理する
NULL 読み込みで、プロセスは終了していない。

「PASS」は、指定時間動いたあと閉じるボタンと同じ経路で閉じ、ランチャーが
正常な終了コードで終わり（Doom の KillClient による 1、GTK3 の
「合成器との接続断」による 1 を含む）、次のアプリの開始時に生きた
プロセスが残っていないこと。

## 4. 残課題

- glibc の C.UTF-8 ロケールが macOS ホストのビルドでは入らない（host に
  glibc の `localedef` が無い）。GTK と Firefox が
  "Locale not supported by C library" を出して C ロケールで動く。
  Debian の `libc-bin` が持つ `/usr/lib/locale/C.utf8/` をステージすれば
  ホストに依らず入る。
- gdk-pixbuf の `loaders.cache` も同じ理由で macOS ホストでは作られない
  （PNG/JPEG は libgdk_pixbuf に内蔵なので GTK3 の起動には要らない）。
- x86_64 では、親が先に回収された孤児ゾンビは次の `find_free_slot()` まで
  アドレス空間を持ったままになる（arm64 は exit 時に解放）。Firefox/Chromium
  を閉じた直後に数百 MB が残って見えるのはこれ。
- AF_UNIX の半閉じ（`shutdown(SHUT_WR)`）は表現されておらず、wineserver が
  `sock_init: ERROR in sock_check_pollhup()` を出す（EOF 方式に落ちて動く）。
