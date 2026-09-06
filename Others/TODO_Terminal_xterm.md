# ImplusOS — 端末プラン（無改変 xterm + カーネル擬似端末）

> **ステータス: 2026-09-05。T1〜T5 完了・QEMU 実機で検証済み。**
> ImplusOS のデスクトップ上のウィンドウで、Debian trixie の無改変 `xterm` が
> 同梱の静的 BusyBox（`/bin/sh`）を走らせ、`ash` のプロンプトが表示され、
> 行編集のカーソル位置照会（`ESC[6n` → `ESC[1;12R`）まで往復している。

調査基準日: 2026-09-05
対象バイナリ: Debian trixie `xterm 398-1` の `/usr/bin/xterm`（無改変）

関連: [`TODO_Doom_Xorg_MethodA.md`](TODO_Doom_Xorg_MethodA.md)（X サーバ経路。
本プランはその上に乗る）、[`TODO_glibc_Port.md`](TODO_glibc_Port.md)（ランタイム
同梱機構）、[`TODO_Chromium_LinuxABI.md`](TODO_Chromium_LinuxABI.md)（Linux syscall
互換層）。

---

## 0. なぜ xterm か

端末エミュレータの選定より先に決まっていた制約が 2 つある。

- **表示スタックは X 一択**。Debian の無改変 Xorg を KMS シムへ繋ぐ経路は Doom で
  実証済み（`display_kms_set_mirror` で WM ウィンドウにミラー）。Wayland
  compositor は QEMU 実機未検証（`TODO_GTK3_Wayland_LinuxABI.md` §G3 の残件）で、
  そちらを選ぶと未知が二重になる。
- **第三者バイナリは無改変**。ソース追加はしない。

その上で候補を比べると:

| 候補 | 追加パッケージ | 表示 | 判定 |
|---|---|---|---|
| **xterm** | `xterm` `libxft2` `libutempter0`（+ terminfo 用 `ncurses-base`） | X11（実証済） | **採用** |
| st (`stterm`) | `libxft2` `ncurses-term` | X11 | DT_NEEDED 4 本と最小だが Xft+fontconfig 必須、`st` terminfo 必須 |
| foot | `libfcft4` `libutf8proc3` ほか | Wayland | compositor が実機未検証 |
| xfce4-terminal / gnome-terminal | GTK3+VTE+dconf/D-Bus で数十本 | GTK | VTE は pty/termios を最も深く叩く |
| kitty / alacritty | Python3.13 / EGL | — | 論外 |

**決め手は依存の少なさではなくフォント**。xterm は `-fa` を渡さない限り X コア
フォント（`fixed`）で描くので、Xorg 用に `stage-xorg.sh` が既に置いている
`xfonts-base` だけで起動できる。st も foot も 1 文字描く前に Xft +
fontconfig が要る。実際 xterm の直接依存 14 本のうち 12 本は Xorg / GTK3 の閉包に
既にあり、増えたのは `libXft` と `libutempter` の 2 本だけだった。

---

## 1. 唯一の障害は「擬似端末が無かった」こと

端末を何にしようと `openpty()` は必ず呼ばれる。着手前の実測:

- `Core/vfs/DevFS.c` に `/dev/ptmx` も `/dev/pts/*` も無い
- `Compat/Linux/Syscall_LinuxCompat.c` の `linux_ioctl_tty()` が
  `TIOCGPTN` / `TIOCSPTLCK` / `TCGETS` / `TIOCSCTTY` を一律 `ENOTTY`
- `TIOCGWINSZ` は fd 0〜2 に 24x80 を返すだけのダミー

`setsid`(112) / `setpgid`(109) / `poll` / `ppoll` / `pipe` / `fork` / `execve` /
`wait4` は実装済みだったので、足りないのは pty 本体と行規律だけだった。

---

## 2. T1 — カーネル擬似端末（新規）

### 2.1 追加したもの

| パス | 種別 | 内容 |
|---|---|---|
| `Kernel/Source/Core/tty/Pty.h` / `Pty.c` | 新規 | pty ペア 8 組、入出力リング、行規律、termios、winsize、制御端末 |
| `Kernel/Source/Core/tty/Tests/` | 新規 | ホスト側ハーネス（`run.sh` + スタブ）。12 ケース |
| `Core/vfs/DevFS.c` | 変更 | `/dev/ptmx`（動的割り当て）、`/dev/pts/N`（前置一致）、`/dev/tty` を制御端末へ解決 |
| `include/kernel/interfaces/vfs_types.h` | 変更 | `dev_write` と `open_file` フックを追加 |
| `Core/vfs/VFS.c` / `VFS.h` | 変更 | `vfs_dev_write()` / `vfs_open_file()` / `vfs_file_has_dev_write()` |
| `Core/syscall/Syscall_File.c` | 変更 | open が `open_file` を呼ぶ、write が `dev_write` を通る、`syscall_file_is_pty()` |
| `Compat/Linux/Syscall_LinuxCompat.c` | 変更 | pty fd の端末 ioctl を `linux_ioctl_tty()` の一律 ENOTTY より先に振り分け |
| `Core/process/ProcessManager_Create.c` | 変更 | プロセス終了時に `pty_forget_process()` |

### 2.2 実装した端末の中身

- **行規律**: `ICANON`（行単位の受け渡し・`VERASE`/`VKILL`/`VWERASE`/`VEOF`）、
  `ECHO`/`ECHOE`/`ECHOCTL`、`ISIG`（`^C`→SIGINT `^\`→SIGQUIT `^Z`→SIGTSTP を
  前景プロセスグループへ）、`ICRNL`/`INLCR`/`IGNCR`/`ISTRIP`、`IXON`（`^S`/`^Q`）、
  出力側 `OPOST`/`ONLCR`（これが無いとシェルの出力が階段状になる）
- **termios**: Linux の *カーネル* レイアウト（`NCCS=19`・36 バイト）。glibc の
  `struct termios`（NCCS=32）ではない。`tcgetattr` が変換する前の形が正しい。
- **ioctl**: `TCGETS` `TCSETS(W|F)` `TCFLSH` `TIOCGWINSZ` `TIOCSWINSZ`（→SIGWINCH）
  `TIOCGPGRP` `TIOCSPGRP` `TIOCGSID` `TIOCSCTTY` `TIOCNOTTY` `TIOCGPTN`
  `TIOCSPTLCK` `TIOCGPTPEER` `FIONREAD` `TIOCOUTQ` `TIOCPKT`
- **ブロッキング**: `Syscall_File.c` のパイプと同じ流儀（`process_sleep_current_ms`
  でリングを見に行く）。ただし端末は「何時間でも入力を待つ」のが正しいので
  タイムアウトで `EAGAIN` を返すことはせず、データ・EOF・未マスクのシグナル
  （`EINTR`）でだけ戻る。アイドルが続いたら 1 ms → 10 ms に落とす。
- **ハングアップ**: 最後の slave fd が閉じたら master の read が `EIO`（端末
  エミュレータが窓を閉じる合図）、master が閉じたら slave が EOF + SIGHUP。

### 2.3 ホスト側ハーネス

```bash
./Kernel/Source/Core/tty/Tests/run.sh
```

実物の `Pty.c` を `Tests/stub/` の最小カーネル代替（spinlock / usercopy /
ProcessManager）と一緒にホスト `cc` でビルドして走らせる。カバー範囲:
ptmx プロトコル（ロック→`TIOCGPTN`→`unlockpt`→`TIOCGPTPEER`）、行の組み立て、
`DEL` 消去とそのエコー、`^C` のシグナルと入力破棄、`^D` の EOF、`ONLCR`、
raw モード、`TIOCSWINSZ`→SIGWINCH、ハングアップ両方向、poll ビットと `FIONREAD`、
制御端末の親からの継承、ペア枯渇と回復。**QEMU 起動 1 回ずつ潰す種類の不具合を
ここで潰すためのもの**（`TODO_GTK3_Wayland_LinuxABI.md` §4 と同じ発想）。

---

## 3. T2 — vendoring

`packages.seed.txt` に `xterm` と `ncurses-base` を足しただけ。`resolve.sh` は
seed パッケージ内の全 ELF の `DT_NEEDED` を閉包の起点にするので、`libXft.so.2` と
`libutempter.so.0` は自動で拾われる。

- `packages.lock`: 179 → **183** パッケージ
- `closure.txt`: 179 → **182** soname、未解決 **0**
- `stage-xterm.sh`（新規）: `/usr/bin/xterm` 実体、`app-defaults`（`/etc/X11` と
  `/usr/share/X11` の両方。Debian と上流で探索先が違う）、terminfo 10 件
  （`xterm` `xterm-256color` `linux` `vt100` …）
- `Vendor/LinuxRuntime/Makefile` に `xtermdata` ターゲット、トップ Makefile の
  `linux_runtime_stage` に追加

ホストでの素振り（実体のみ使用）:

```
S=Build/x86_64/LinuxRuntime/stage
$S/lib64/ld-linux-x86-64.so.2 --library-path $S/usr/lib/x86_64-linux-gnu \
  $S/usr/bin/xterm -version   -> "XTerm(398)"
```

---

## 4. T3 — シェルと環境

- `/bin/sh` は**既にあった**。Xorg の `Popen()` が xkbcomp を
  `/bin/sh -c` で起動するため、トップ Makefile の `STAGE_POSIX_SHELL` が静的
  busybox を `/bin/sh` として置いている。argv[0] が `sh` なら ash applet になる。
- `/etc/passwd` も既に `root:x:0:0:root:/root:/bin/sh`（`EtcFS.c`）。
- `glibc_envp`（`ProcessManager_Create.c`）に追加:
  `SHELL=/bin/sh`（xterm は `$SHELL` を先に見る。絶対パスが取れないと
  "No absolute path found for shell" で終了する）、`TERM=xterm`、
  `XFILESEARCHPATH`（Xt の app-defaults 探索先）。

---

## 5. T4 — ランチャと X セッションの共有

X サーバは 1 つ（`:0`）しか無いので、2 番目のランチャは**自分で起動せず既存に
相乗りする**必要がある。Doom のランチャに埋まっていたその手順を切り出した:

- `Userland/API/Source/XSession.{c,h}`（新規）: `xsession_open()` /
  `xsession_run()` / `xsession_close()`。既に `/tmp/.X11-unix/X0` が listen して
  いれば「相乗り」フラグを立て、ウィンドウも KMS ミラーも取らず、終了時にも
  相手のサーバを落とさない。
- `Userland/Application/Terminal/`（新規）: 上記を呼ぶだけのランチャ。
- `Userland/Application/Doom/Start.c`: 195 行 → 53 行。X の起動条件・引数・
  ミラー処理は一字一句そのまま `XSession.c` へ移した。
- `windowmanager/Resource/Apps/apps.list` に `Terminal` を追加。

xterm の引数:

```
-fn fixed -bg black -fg white +sb -ut -geometry 80x24 -e /bin/sh
```

`-ut` が要る。これが無いと xterm は utmp 記録のために libutempter の
`/usr/lib/utempter/utempter` を fork+exec して待ち続ける（このイメージには
無いバイナリ）。`+utmp` ではなく `-ut` である点に注意（xterm はこのオプションに
限り `-` が否定形）。

---

## 6. T5 — 実機で見つかった 3 件の不具合

すべてカーネル側。ホストのハーネスでは出ない種類のものばかりだった。

### 6-1. `vfs_mount()` はドライバ構造体を**コピー**する

`devfs_file_is_pty()` を `file->fs_driver == &g_devfs_vfs_driver` で判定して
いたが、`VFS.c` の `g_vfs_drivers[n] = *driver;` はマウント時に構造体を丸ごと
複製する（`prefix` を差し込むため）。したがって同一性比較は**必ず偽**になり、
すべての pty が「pty ではない」と判定され、`TIOCGPTN` が
`linux_ioctl_tty()` の一律 ENOTTY に落ちて `xterm: Error 32`（`ERROR_PTYS`）に
なっていた。ドライバの識別は自分のフック関数のアドレス（`dev_ioctl`）で行う。

### 6-2. `dup()` が fd 0/1/2 を絶対に返さなかった

端末エミュレータの子は必ずこう書く:

```c
for (i = 0; i <= 2; i++) if (i != ttyfd) { close(i); dup(ttyfd); }
```

`dup()` が「空いている最小の fd」を返す前提のコードだが、この OS では fd 0/1/2 に
テーブル項目が無い状態が「コンソール」を意味するため、割り当ては 3 から始まって
いた。子は 3,4,5 を貰い、stdio はコンソールのまま。結果、シェルはプロンプトを
COM1 に吐き、`isatty(0)` も落ちる。

**修正**: プロセスごとに「0/1/2 のうち自分で close したもの」を 1 ビットずつ
覚え（`g_std_closed`）、その fd に限って `dup`/`open`/`F_DUPFD` の候補にする。
誰でも 0 を貰えるようにするのは危険で、実際それを踏んで直した経緯がある
（libwayland の `F_DUPFD_CLOEXEC(fd,0)` が fd 0 を貰って wl_shm が壊れた
＝`TODO_GTK3_Wayland_LinuxABI.md` §4 #12）。「自分で閉じた fd だけ」は
まさに POSIX の規則そのもの。fork で引き継ぎ、プロセス終了で消す。
ついでに `dup`/`dup2`/`F_DUPFD` が `FD_CLOEXEC` を複製していたのも直した
（POSIX ではコピー側のフラグは落ちる）。

### 6-3. `writev()` / `readv()` が fd ≤ 2 を無条件でコンソール扱い

`write()` には「fd 1/2 にファイルが割り当てられていなければコンソール」という
正しい判定があったが、`syscall_writev()` は `if (fd <= 2)` で無条件に COM1 へ
流していた。busybox の行編集はプロンプトを `writev` で書くので、pty を
stdout に持っていても出力だけがシリアルに漏れていた。`readv()` も同様に
`fd <= 2` を即 EOF にしていた。両方 `write()` と同じ判定に揃えた。

### 6-4. ついでに直したもの

- `com.ImplusOS.loginui`: WM の登録待ちが 6 秒固定で、ISO からのコールドブート
  では普通に超える。超えると通知デーモンを起動しないまま先に進んでいた。60 秒へ
  （成功した時点で抜けるので速いときのコストはゼロ）。

---

## 7. 検証

**ホスト**（`./Kernel/Source/Core/tty/Tests/run.sh`）: 12 ケース全通過。

**QEMU**（q35 / OVMF / 16 vCPU / 8 GiB、`-display none` + monitor `screendump`）:
デスクトップ上の「Terminal」ウィンドウに `xterm` が描画され、シリアルには
pty を流れたバイト列がそのまま出る:

```
[pty] ptmx open -> pair 0
[pty] pts open -> pair 0            (xterm 親 → 子 → /dev/tty → 再オープン)
[fd]  close std 0->0 pid=7
[fd]  dup 0x0E->0 pid=7             (ttyfd=14 を 0/1/2 へ)
[execve-ok] pid=7 path=/bin/sh
[pty] slave write pair=0 n=15 "/usr/bin # .[6n"     ← ash のプロンプト
[pty] master read  pair=0 n=15 "/usr/bin # .[6n"    ← xterm が読む
[pty] master write pair=0 n=7  ".[1;12R"            ← xterm のカーソル位置応答
[pty] slave read   pair=0 n=1  "."                  ← 行規律を通って ash へ
```

`ESC[1;12R`（1 行 12 桁）は、xterm が 11 文字のプロンプトを**実際に描画して**
カーソル位置を答えている証拠。画面にも `/usr/bin # ` とカーソルが出る。

**未検証**: 物理キーボードからの打鍵。QEMU の monitor（`sendkey` /
`mouse_move`）からの入力がゲストに届かないため、`-display none` の自動テストでは
打てなかった（マウスカーソルも動かない＝pty とは無関係の入力経路の問題）。
master→slave 方向は xterm 自身の `ESC[1;12R` が行規律を通っており、canonical /
erase / `^C` / raw の各モードはホストのハーネスで押さえてある。GUI 付きで
`make run_uefi_cdrom` を回せば打鍵まで確認できる。

---

## 8. 残件

- **打鍵の実機確認**（上記）。
- xterm の cwd が `/usr/bin`（カーネルが外来 ELF の cwd を実行体のディレクトリに
  するため）。プロンプトが `/usr/bin # ` になるだけで実害は無いが、見た目が悪い。
- `xsession_open()` の相乗り判定は「socket が listen しているか」だけなので、
  2 つのランチャが同時に起動すると両方が Xorg を起こす競合が残る
  （Xorg 自身の `/tmp/.X0-lock` に頼るか、`/run` に排他ファイルを作るか）。
- `Kernel/Source/Core/tty/Pty.c` と `Syscall_File.c` に入れたブリングアップ用
  トレース（`[pty]` / `[fd]`、いずれも出力数に上限あり）は残してある。
  `OS_CONFIG_FOREIGN_TRACE=0` で消える。
- pty ペアは 8 組固定（`PTY_MAX_COUNT`）。1 端末 1 組なので当面足りる。

---

## 9. 参照

- 行規律とペア: `Kernel/Source/Core/tty/Pty.c`
- ノード公開: `Kernel/Source/Core/vfs/DevFS.c`（`/dev/ptmx`・`/dev/pts/N`・`/dev/tty`）
- fd 層: `Kernel/Source/Core/syscall/Syscall_File.c`（`g_std_closed`・`allocate_fd_locked`）
- ioctl 振り分け: `Kernel/Source/Compat/Linux/Syscall_LinuxCompat.c`（`syscall_ioctl_ex`）
- X セッション: `Userland/API/Source/XSession.c`
- ランチャ: `Userland/Application/Terminal/Start.c`
- 同梱: `Vendor/LinuxRuntime/stage-xterm.sh`, `packages.seed.txt`
