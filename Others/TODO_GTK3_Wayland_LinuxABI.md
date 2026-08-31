# ImplusOS — GTK3 / Wayland 外来 Linux ABI 実行プラン

> **ステータス: 2026-08-29。G1/G2 完了（W1 達成）。G3（Wayland コンポジタ）第1弾
> 実装 = カーネル共有メモリ配線 K1–K5 ＋ ハンドロール compositor。QEMU 検証待ち。**
>
> 調査基準日: 2026-08-29
> 目的: 外来の動的リンク GTK3 / Wayland Linux バイナリ（第一目標は Debian trixie の
> `gtk-3-examples` の `gtk3-demo` / `gtk3-widget-factory`）を、既存の glibc 動的リンク
> 基盤（[`TODO_glibc_Port.md`](TODO_glibc_Port.md)）の上で起動する。
> カーネル Linux syscall 互換層の一般的拡充は [`TODO_Chromium_LinuxABI.md`](TODO_Chromium_LinuxABI.md)、
> ランタイム同梱機構は [`TODO_glibc_Port.md`](TODO_glibc_Port.md) が担当。本書はそれらに
> 乗る「GTK3/Wayland 固有」の差分だけを扱う。

---

## 0. 前提と到達点の定義

`Vendor/LinuxRuntime` はもともと Chromium の `DT_NEEDED` 閉包を解決していて、
副産物として glib/gobject/gio・pango・cairo・libX11・libxkbcommon は既に閉包内に
あった。足りないのは **libgtk-3 / libgdk-3 / libgdk_pixbuf / libepoxy /
libharfbuzz / libcairo-gobject / libwayland-client(+cursor,egl) と X の補助
ライブラリ**、および GTK が実行時に読む **非 .so データ**（GSettings スキーマ、
フォント）。

**現状の到達点（G2 完了時点）**:
ImplusOS にはまだ Wayland コンポジタも X サーバも無いので、`gtk3-demo` は
`ld.so → libc → GTK スタック全 63 本ロード → gtk_init → gdk_display_open` まで
進んで **"cannot open display" で終了コード 1**。ここまでで
「glibc 動的リンク + GTK3 ランタイム同梱」は成立とみなす。
実描画は G3〜G5（コンポジタ）が要る。

ホスト（x86_64 Linux）で **ステージした実体だけ**を使った素振り:

```
$ ld-linux-x86-64.so.2 --library-path <stage>/usr/lib/x86_64-linux-gnu \
    <stage>/usr/bin/gtk3-demo
(gtk3-demo:NNNN): Gtk-WARNING **: cannot open display:
[exit 1]
```

`gtk3-demo` の最大 glibc 要求は `GLIBC_2.38`、同梱 ld.so は `GLIBC 2.41-12` で充足。

---

## 1. 生成物・変更一覧（G1/G2）

| パス | 種別 | 内容 |
|---|---|---|
| `Vendor/LinuxRuntime/packages.seed.txt` | 変更 | GTK3/Wayland パッケージ 21 件を seed に追加（`gtk-3-examples` 起点＋dlopen/データ pkg 明示） |
| `Vendor/LinuxRuntime/packages.lock` | 再生成 | 93→**119** パッケージ（trixie / snapshot `20250901T000000Z` にピン、sha256 込み） |
| `Vendor/LinuxRuntime/closure.txt` | 再生成 | 114→**133** soname、未解決 0 |
| `Vendor/LinuxRuntime/stage-gtkdata.sh` | 新規 | `.so` 以外の GTK 実行時データを STAGE_DIR へ配置（下記） |
| `Vendor/LinuxRuntime/Makefile` | 変更 | `gtkdata` ターゲット追加（`stage` の後段） |
| `Makefile` | 変更（1 行） | `linux_runtime_stage` の `-C` 呼び出しに `gtkdata` を追加 |
| `Userland/Application/com.ImplusOS.gtk3demo/` | 新規 | `/usr/bin/gtk3-demo` を `process_spawn` するネイティブ・ランチャ（Doom/Chromium と同型） |
| `Userland/Application/com.ImplusOS.windowmanager/Resource/Apps/apps.list` | 変更 | `GTK3 Demo` を追加（`APP_DIRS` 自動 glob なので Makefile 改修不要） |
| `Kernel/Core/process/ProcessManager_Create.c` | 変更 | 外来 Linux ABI の既定 envp（`glibc_envp`）に `HOME` / `XDG_*` / `GDK_BACKEND=wayland,x11` / `GSETTINGS_SCHEMA_DIR` / `GSETTINGS_BACKEND=memory` / `FONTCONFIG_*` を追加。ネイティブ経路は不変、Chromium にも無害（GDK_BACKEND は Ozone が無視） |

### `stage-gtkdata.sh` が STAGE_DIR に置くもの

- `/usr/bin/gtk3-demo` `gtk3-widget-factory` `gtk3-demo-application` `gtk3-icon-browser`
  （Debian の実バイナリ無改変。git にはコミットしない＝`.deb` 方針と一貫）
- `/usr/share/glib-2.0/schemas/gschemas.compiled`
  （`libgtk-3-common` + `gsettings-desktop-schemas` + glib の `*.gschema.xml` 38 本を
  ホスト `glib-compile-schemas` でコンパイル。未コンパイルだと GLib が
  `g_settings_new` で abort する）
- `/usr/share/fonts/truetype/dejavu/*.ttf` + `/etc/fonts/fonts.conf`
  （フォント皆無だと Pango がテキストを一切描けない。cachedir は `/tmp/fontconfig`）
- gtk3-demo 同梱リソース（`/usr/share/` 配下）

### 既知の未配置（G4 で対応）

- **gdk-pixbuf `loaders.cache`**: ホストに `gdk-pixbuf-query-loaders` が無く生成不可。
  ラスタ画像ローダ不在 → gtk3-demo のアイコン/画像は出ない（ウィンドウ自体は出る）。
  対策案: 同梱済み `libgdk-pixbuf-2.0-0` の deb に `gdk-pixbuf-query-loaders` を含む
  `libgdk-pixbuf2.0-bin` を `packages.lock` に足し、ステージ後にホスト上で実行して
  正しいパスの `loaders.cache` を生成（このホストは x86_64 Linux なので実行可能）。
- **Adwaita アイコンテーマ**: 巨大なので未同梱。`hicolor` フォールバックのみ。
- **`/etc/machine-id`**: EtcFS が既に静的供給（glibc port G4）。

---

## 2. フェーズ

### G1 — GTK3 / Wayland `.so` 閉包の vendoring 【完了 2026-08-29】

- [x] `packages.seed.txt` に GTK3/Wayland 系を追加。
- [x] `make -C Vendor/LinuxRuntime resolve` → `packages.lock` 119 / `closure.txt` 133、未解決 0。
- [x] `make -C Vendor/LinuxRuntime fetch` sha256 照合完走（`.deb` 合計 91 MiB）。
- [x] `make -C Vendor/LinuxRuntime stage` 自己検査 `placed=133 missing=0`。
- [x] `gtk3-demo` + `gtk3-widget-factory` の推移的 NEEDED 63 本がステージ tree で全解決を確認。

### G2 — テストアプリ配線 + 起動環境 【完了 2026-08-29】

- [x] `stage-gtkdata.sh` + `gtkdata` ターゲット、トップ Makefile から無条件委譲。
- [x] `com.ImplusOS.gtk3demo` ランチャ（`make app_build` 通過）。
- [x] `apps.list` に `GTK3 Demo`。
- [x] `glibc_envp` に GTK/XDG/フォント環境変数（`make kernel` 通過）。
- [x] ステージ実体だけでホスト素振り → `cannot open display` 到達を確認。
- [x] **QEMU 実起動（1回目、2026-08-29 セッション17）**: `.so` 閉包 約60本すべて
      in-OS で open→mmap 成功、locale-archive も。glibc 初期化を完走し GLib の
      スレッドプール起動まで到達 → `clone` が `EAGAIN` を返し
      `GLib-ERROR: pool-spawner` で `abort()`。
      → `is_valid_user_entry()` の許可レンジがヒープ窓（`.so` の実マップ先
      `0x41_xxxx_xxxx`）を外していたのが原因。`[0x1000, USER_STACK_BASE)` へ
      拡大して修正（`TODO_glibc_Port.md` セッション17）。要再起動確認。
- [x] **QEMU 実起動（2回目、セッション18）**: pool-spawner スレッドは立ったが
      `ppoll` 未実装（ENOSYS）で GLib メインループが暴走し
      `GLib-WARNING: poll(2) failed` をシリアルへ吐き続けて激遅に。
      → `poll`(7)/`ppoll`(271) を Linux 互換層に実装
      （`TODO_glibc_Port.md` セッション18）。`-DLINUX_SYSCALL_TRACE` も外した。
- [x] **QEMU 実起動（3回目、セッション19）**: `poll`/`ppoll` OK・警告スパム消滅。
      次は **間欠的 #PF（RIP=0/RBP=0/CR2=0）**＝スレッド生成の SMP レース
      （READY 公開後に子 RSP を差し替えていた）。
      → `process_create_thread_ex()` に `user_stack` 引数を追加し READY 前に設定
      （`TODO_glibc_Port.md` セッション19）。
- [x] **QEMU 実起動（4回目、セッション19 検証）**: 全 pthread が正しいスタックで立ち、
      GLib 静穏、`gtk_init` 完走 → `gdk_display_open` 失敗
      `Gtk-WARNING: cannot open display:` で正常終了（クラッシュ/ハング無し、
      カーネル巻き込み無し）。**＝ G1/G2 完了、W1 達成**。以降は実描画＝G3。

### G3 — Wayland コンポジタ（最小）  【第1弾実装済み、QEMU 検証待ち — TODO_glibc_Port.md セッション20】

**方式 (a) で実装**: libwayland 非依存のハンドロール compositor
`Userland/Application/com.ImplusOS.waylandcompositor/`（ネイティブ ELF）。
`/tmp/wayland-0` で listen（ネイティブ AF_UNIX syscall 220–229）、committed
`wl_shm` バッファを `Window.h` の backing store に blit。

**カーネル前提として実装した K1–K5**:
- K1: memfd を `SharedMemory.c` の共有オブジェクトで裏打ち。`linux_mmap` の
  memfd 経路で実共有ページをマップ（従来はスナップショットコピーで無意味）。
- K2: `SCM_RIGHTS` を実装（`sendmsg` が memfd→共有ハンドル化＋grant、`recvmsg`
  が受信側に memfd fd を install）。従来は生 fd 整数コピーで無意味だった。
- K3: AF_UNIX リング 1 KiB→256 KiB、`recv` の EAGAIN/EOF 区別。
- K4: `SYSCALL_MEMFD_SHM_HANDLE`(269) ＋ `os_memfd_shm_handle()`。
- K5: `poll`/`epoll` が AF_UNIX fd（0x8000+）を認識（`unix_socket_poll`）。
  これが無いと libwayland がディスプレイ fd を死んだ接続と誤認する。

**未了**: 実 QEMU 起動での動作確認、resize（プール再作成）、gdk-pixbuf、
入力（U3）、複数クライアント、`wl_keyboard.keymap` の fd 送信（server→client）。

- [ ] 旧・方式決定メモ（参考）:
  - (a) `libwayland-server`（同梱済み）を使うネイティブ ImplusOS アプリを新設し、
        `wl_display` / `wl_compositor` / `wl_shm` / `xdg_wm_base` / `wl_seat` /
        `wl_output` を実装、クライアントの `wl_shm` バッファを
        `com.ImplusOS.windowmanager` のシーングラフへブリッジ。
  - (b) Weston を外来 Linux バイナリとして持ち込み（headless/fbdev バックエンド）。
        udev/dbus/input 依存が重い。
  - → 既定は (a)。`libwayland-server` は AF_UNIX + `wl_display` イベントループなので
     ImplusOS の `IPC/UnixSocket.c` + epoll 互換で動くはず（要検証）。
- [ ] `$XDG_RUNTIME_DIR/wayland-0`（`/tmp/wayland-0`）で listen。
      envp の `XDG_RUNTIME_DIR=/tmp` は G2 で設定済み。
- [ ] `wl_shm` プール = `memfd_create` + `mmap`（TmpFS `/dev/shm` 経路、glibc port
      で実装済み）。共有メモリのクロスプロセス可視性は
      `TODO_glibc_Port.md` セッション10 の遅延コミット mmap の制約に注意。
- [ ] キーボード/ポインタは `com.ImplusOS.windowmanager` の入力ルーティングから
      `wl_keyboard` / `wl_pointer` へ変換。xkbcommon のキーマップは
      同梱 `libxkbcommon` + データ（`xkeyboard-config` を `packages.lock` に追加）。

### G4 — GTK 実行時データの補完

- [ ] gdk-pixbuf `loaders.cache`（§1 の対策案）。
- [ ] `xkeyboard-config`（`/usr/share/X11/xkb`）を `packages.lock` に追加、xkbcommon 用。
- [ ] 最小 `hicolor` + 必要なら `Adwaita` の一部アイコン（サイズ検討）。
- [ ] `/usr/share/gtk-3.0/settings.ini`（`gtk-font-name` を DejaVu に、
      `gtk-icon-theme-name=hicolor`）。

### G5 — 起動検証マイルストーン（QEMU、ユーザー側）

- [x] **W1**（2026-08-29 セッション19）: `gtk3-demo` が `cannot open display` で
      正常終了。glibc + GTK3 スタック 約60本のロード・再配置・初期化、
      GSettings/フォント/locale 読み込み、pthread 生成、GLib メインループ、
      `gtk_init` まで in-OS で通ることを確認。
- [ ] **W2**: 最小コンポジタ起動、`wayland-info` 相当（`weston-info` を
      `packages.lock` に追加）が `wl_compositor` / `wl_shm` / `xdg_wm_base` を列挙。
- [ ] **W3**: `gtk3-demo` のメインウィンドウが `com.ImplusOS.windowmanager` 上に描画。
- [ ] **W4**: `gtk3-widget-factory` でウィジェット・テーマ・フォント描画が崩れない。
- [ ] **W5**: ポインタ/キーボード入力がクライアントに届き、ボタン等が反応。

### G6 — syscall ギャップ埋め（運用）

`-DLINUX_SYSCALL_TRACE` で顕在化した `ENOSYS`/`ENOTSUP` を潰す。

**セッション17 の実測トレースで判明済み:**

| # | syscall | 現状 | 対応 |
|---|---|---|---|
| — | `clone`(56) スレッド生成 | `is_valid_user_entry()` の範囲外で `EAGAIN` → **GLib pool-spawner 致命** | **修正済み**（`TODO_glibc_Port.md` セッション17。低位レンジを `[0x1000, USER_STACK_BASE)` へ） |
| 7 / 271 | `poll` / `ppoll` | `ENOSYS` → **GLib メインループが暴走**（ビジーループ＋警告スパム） | **修正済み**（`TODO_glibc_Port.md` セッション18。`syscall_poll_one_fd` ＋ `linux_poll_common`。8ms スライスで縮退ブロック） |
| — | `clone`(56) 子 RSP | READY 公開後に差し替え → SMP レースで子が RIP=0 へ #PF | **修正済み**（`TODO_glibc_Port.md` セッション19。`process_create_thread_ex(..., user_stack)` で READY 前に設定） |
| 435 | `clone3` | `ENOSYS`（glibc は `clone` にフォールバックするので当面 OK） | 低優先。`clone` 経路が安定したら実装 |
| 13 | `rt_sigaction(sig 32/33)` | `signum >= 32` で `EINVAL`（glibc NPTL の SIGCANCEL/SIGSETXID）。今回は glibc が許容して継続 | `OS_CONFIG_SIGNAL_HANDLER_MAX_PER_PROCESS` を 34+ に拡張 → ハンドラ配列を広げてから 32/33 を受理（no-op 配送でよい） |
| 257 | `openat("/usr/share/zoneinfo/UTC")` | `ENOENT`（未同梱）。glibc 内蔵 UTC にフォールバック、非致命 | G4 で `tzdata` の `Etc/UTC` だけステージするか、EtcFS で供給 |

**まだ見えていない想定候補**（`gdk_display_open` 以降で出るはず）:
`recvmsg`/`sendmsg` の `SCM_RIGHTS`（Wayland の fd 受け渡し＝**現状未実装、
G3 の要**、`TODO_Chromium_LinuxABI.md` と共通）、`memfd_create` の
`MFD_ALLOW_SEALING`＋`fcntl(F_ADD_SEALS)`（`wl_shm`）、`ppoll`、`timerfd_*`、
`signalfd4`、`eventfd2` の semaphore モード、`POLLxxx` の網羅
（`wl_display` イベントループ）。

---

## 3. リスク / 未解決

- **`SCM_RIGHTS`（AF_UNIX の fd パッシング）が未実装**。Wayland の `wl_shm` /
  dmabuf / `wl_data_device` は fd を `sendmsg` の補助データで渡す。G3 の前提条件。
  `Kernel/IPC/UnixSocket.c` に補助データ経路の追加が要る。
- **共有メモリのクロスプロセス・コヒーレンシ**。`TODO_glibc_Port.md` の
  「VFS にページキャッシュ無し」「遅延コミット mmap は VMA 木無し」の制約下で
  `wl_shm` プールをコンポジタ・クライアント間で正しく共有できるかは要実証。
- **単一 ISO サイズ**。GTK3 閉包でステージは 232→**254 MiB**。`INSTALL_DISK_IMAGE_SIZE_MB`
  = 2560 の余裕内。xkeyboard-config / アイコンで G4 が +数十 MiB。
- **QEMU 実起動がこの環境で不可**。W1〜W5 はユーザー検証。実装側の完了線は
  ビルド通過＋ステージ実体でのホスト素振り＋トレース設計まで。

---

## 4. 参照

- glibc 動的リンク基盤: [`TODO_glibc_Port.md`](TODO_glibc_Port.md)（実装ログ §11 セッション1–15）
- カーネル Linux ABI 拡充: [`TODO_Chromium_LinuxABI.md`](TODO_Chromium_LinuxABI.md)
- vendoring 機構: `Vendor/LinuxRuntime/README.md`, `resolve.sh`, `stage.sh`, `stage-gtkdata.sh`
- 外来 ABI 既定 envp: `Kernel/Core/process/ProcessManager_Create.c`（`glibc_envp`）
- ランチャ雛形: `Userland/Application/{Doom,Chromium}/Start.c`
- ネイティブ・コンポジタ（ブリッジ先）: `Userland/Application/com.ImplusOS.windowmanager/`
- AF_UNIX: `Kernel/IPC/UnixSocket.c`
