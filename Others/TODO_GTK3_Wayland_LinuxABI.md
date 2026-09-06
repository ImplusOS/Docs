# ImplusOS — GTK3 / Wayland 外来 Linux ABI 実行プラン

> **ステータス: 2026-09-05。G1〜G5 完了。QEMU 実起動で Debian 無改変の
> `gtk3-demo` が ImplusOS 上に**ウィンドウを描画し、ポインタ入力にも応答する**。
> 到達までにカーネル/ランタイム側のバグを 12 件修正した（§4）。
> 残るは W6（WM を落として panel バックエンドへ昇格する経路）のみ。**
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
| `Userland/Application/com.ImplusOS.waylandcompositor/{Compositor.h,Wayland.c,Compositor.c}` | 変更/新規（G3） | WM 非依存の Wayland 表示サーバ本体 |
| `Userland/Application/com.ImplusOS.waylandcompositor/Tests/` | 新規（G3） | ホスト側ハーネス（実 GTK3 バイナリで回帰） |
| `Kernel/Core/syscall/{Syscall_Main.h,Syscall_Dispatch.c,Syscall_File.c,Syscall_File.h}` | 変更（G3） | `SYSCALL_MEMFD_FROM_SHM`(273)：共有メモリ → memfd（`wl_keyboard.keymap` の fd 送出） |
| `Userland/{Source/Syscalls.c,API/Source/Memory.h}`, `libc/I_libc/.../sys/syscalls.h` | 変更（G3） | `os_memfd_from_shm()` ラッパ |
| `Userland/Application/com.ImplusOS.gtk3demo/Start.c` | 変更（G3） | ソケット待ち合わせ、compositor の再利用、クライアント／モードを引数で選択 |
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

### G3 — Wayland コンポジタ 【完了 2026-09-04】

**目的の変更**: 当初は「WM のウィンドウ 1 枚に GTK の描画をブリッジする」だけ
だったが、それでは GTK3 が `com.ImplusOS.windowmanager` に依存してしまう。
compositor を**それ自体で完結した表示サーバ**に作り替え、WM を「2 つある
出力バックエンドの片方」に格下げした。WM は前提条件ではなくなった。

| バックエンド | 条件 | 出力 | 入力 |
|---|---|---|---|
| **panel** | WM 不在（または引数 `panel`） | `window_register_service()` で入力オーナー権を取得 → `sys_get_display_framebuffer()` へ直接スキャンアウト | `input_read_keyboard/mouse`（生 HID、相対デルタ） |
| **hosted** | WM 稼働中（既定） | WM ウィンドウ 1 枚の backing store に合成 | `window_input_*_poll`（WM のルーティング、ウィンドウ座標） |

Wayland 側のコードは両者で完全に同一。起動時に自動選択し、
`process_spawn_with_arg` の引数 `panel` / `hosted` で強制もできる。

**WM が死んでも生き延びる**: hosted 動作中に `window_get_wm_pid()` が負に
なったら panel へ自動昇格する。全サーフェスのピクセルは commit 時に
compositor 側へコピー済みなので、クライアントを繋いだまま画面だけ引き継げる。

**ファイル構成**（`Userland/Application/com.ImplusOS.waylandcompositor/`）:

| ファイル | 内容 |
|---|---|
| `Compositor.h` | 共有型（クライアント／オブジェクト表／シーン／出力） |
| `Wayland.c` | ワイヤプロトコル一式。`wl_display`/`wl_registry`/`wl_callback`/`wl_compositor`/`wl_region`/`wl_shm`(+pool,+buffer)/`wl_surface`/`wl_seat`(+pointer,+keyboard)/`wl_output`/`wl_subcompositor`/`wl_data_device_manager`/`xdg_wm_base`/`xdg_positioner`/`xdg_surface`/`xdg_toplevel`/`xdg_popup` |
| `Compositor.c` | シーン（スタック順・フォーカス・配置）、合成、出力／入力バックエンド、メインループ |
| `Tests/` | **ホスト側ハーネス**（下記） |

**G3 第1弾からの主な変更点**:
- **複数クライアント**（最大 4）。オブジェクト表をクライアント毎に持つ。
- **複数トップレベル + `xdg_popup`**。自前のスタック順・クリックフォーカス・
  カスケード配置。ポップアップは `xdg_positioner` のアンカー矩形＋オフセットで配置。
- **入力を実装**（U3 消化）。`wl_seat` は version 5 で pointer+keyboard を公開し、
  `wl_pointer` の enter/leave/motion/button/axis/**frame**、`wl_keyboard` の
  keymap/enter/leave/key/modifiers/repeat_info を送る。
  set-1 スキャンコード → evdev キーコードは 0x01–0x58 が恒等、0xE0 系は表引き。
- **`wl_keyboard.keymap` の fd 送信**（旧「未了」）。ネイティブプロセスには
  `memfd_create` が無いので、**カーネルに `SYSCALL_MEMFD_FROM_SHM`(273) を新設**し
  （`SYSCALL_MEMFD_SHM_HANDLE` の逆）、共有メモリを memfd に包んで SCM_RIGHTS で渡す。
  keymap 本体は include 4 行だけの約 200 バイトで、クライアント側の libxkbcommon が
  同梱 `/usr/share/X11/xkb` から 34 KiB へ展開する。
- **`wl_shm` プールの寿命管理**。GTK は buffer を切り出した直後に pool を destroy
  するので、マッピングをオブジェクトとは別に参照カウントする。受け取った memfd は
  map 後に close（カーネルの共有オブジェクトは全体で 256 個しかない）。
- **サーフェスのピクセルは commit 時にコピー**して即 `wl_buffer.release`。
  これで任意のタイミングで再合成でき、クライアント終了後も画が残る。
- **frame コールバックは合成後にまとめて返す**（約 60 Hz）。即返しだと
  クライアントが全力で描き続ける。
- **部分送信の握り潰しを排除**。イベントが途中で切れるとクライアントの
  パーサが恒久的にずれるため、時間予算付きで送り切り、駄目なら接続を落とす。
- `wl_display.delete_id` を destroy 要求すべてに対して返す。
- ポインタ・スプライトを自前で描く（panel には WM のカーソルが無い）。

### G3.5 — ホスト側ハーネス（`Tests/`）【新規 2026-09-04】

`Compositor.c` / `Wayland.c` は ImplusOS の syscall ラッパ越しにしか外界に
触らないので、その面だけ差し替えれば **Linux ビルドホストでもそのままコンパイル
できる**。`Tests/HostShim.c` が AF_UNIX を実ソケットに、共有メモリハンドルを
`memfd_create` に、出力を PPM ダンプに繋ぐ。

これで**イメージに同梱するのと同一の Debian `gtk3-demo` バイナリ**を、
本物の `libwayland-client` 経由で、本物の compositor コードに接続して
動かせる。オペコード違いや引数長違いは QEMU 起動 1 回ではなく `gcc` 1 回で分かる。

```
make linux_runtime_stage                 # 一度だけ
Userland/Application/com.ImplusOS.waylandcompositor/Tests/run.sh
```

`WLC_TRACE=1` でリクエスト毎の 1 行トレース（`-DWLC_PROTOCOL_TRACE`）、
`WLC_INPUT=<file>` でポインタ／キーボードのスクリプト入力。

ハーネスで**カバーできない**もの: hosted バックエンド、WM 消滅時の panel 昇格、
そしてカーネル自身の AF_UNIX / SCM_RIGHTS / 共有メモリ実装。これらは QEMU が要る。

### G4 — GTK 実行時データの補完

- [x] gdk-pixbuf のローダ群 + `loaders.cache`（2026-09-05）。追加パッケージは
      不要だった: `gdk-pixbuf-query-loaders` は `libgdk-pixbuf-2.0-0` の deb に
      同梱されている。`stage-gtkdata.sh` がローダ 11 本を
      `/usr/lib/x86_64-linux-gnu/gdk-pixbuf-2.0/2.10.0/loaders/` へ置き、
      同梱の ld.so 越しに同梱の query-loaders を走らせてキャッシュを生成し、
      STAGE_DIR 接頭辞を落としてゲスト上のパスに直す。
      `glibc_envp` に `GDK_PIXBUF_MODULE_FILE` / `GDK_PIXBUF_MODULEDIR` も追加。
      **注**: png / jpeg は Debian のビルドでは libgdk_pixbuf 本体に組み込み
      （loaders/ に無いのはそのため）なので、§5 の PNG 問題はこのキャッシュ
      不在が原因ではない（ホストでキャッシュを外しても gtk3-demo は描画できる）。
- [x] `xkeyboard-config`（`/usr/share/X11/xkb`）: Xorg 側の staging で既に入っていた。
- [x] `adwaita-icon-theme`（2026-09-05）: **カーソルテーマは必須**だった。GDK は
      GSettings の `cursor-theme`（既定 "Adwaita"）を libwayland-cursor に渡し、
      読めなければ `cursor_theme_name` を NULL のままにしておいて、後から
      `_gdk_wayland_display_get_scaled_cursor_theme()` でそれを `g_assert` する。
      deb は 500 KiB で、アイコンも一緒に入る。
- [x] `shared-mime-info` の `mime.cache`（2026-09-05）: gdk-pixbuf は画像形式の
      判定を自前の署名比較ではなく GIO の `g_content_type_guess()` で行うので、
      MIME データベースが無いと**ローダも画像データも正しいのに**
      "Unrecognized image file format" になる（§5）。
- [ ] `/usr/share/gtk-3.0/settings.ini`（`gtk-font-name` を DejaVu に）。実害は
      出ていないので保留。

### G5 — 起動検証マイルストーン（QEMU、ユーザー側）

- [x] **W1**（2026-08-29 セッション19）: `gtk3-demo` が `cannot open display` で
      正常終了。glibc + GTK3 スタック 約60本のロード・再配置・初期化、
      GSettings/フォント/locale 読み込み、pthread 生成、GLib メインループ、
      `gtk_init` まで in-OS で通ることを確認。
- [x] **W2〜W5 相当をホストで確認（2026-09-04、`Tests/run.sh`）**。イメージに
      同梱するのと同じ Debian バイナリを、本物の `libwayland-client` 経由で
      本物の compositor コードに繋いだ結果:
      - `gtk3-demo` がレジストリ交換 → `xdg_toplevel` → `wl_shm` バッファ commit まで
        通り、ヘッダバー・サイドバー・ノートブック・フォントまで完全に描画（W3/W4）。
      - ポインタ移動＋クリックでリスト選択が変わり、タイトルと右ペインが追従（W5）。
      - Down キー 2 回で選択が Assistant → Benchmark → Builder に移動。
        keymap fd（SCM_RIGHTS）→ libxkbcommon → evdev マッピングが通っている証拠（W5）。
      - `gtk3-widget-factory` が全ウィジェットを描画し、コンボボックスのクリックで
        `xdg_popup` が親の真下に出る。
      - `gtk3-demo` と `gtk3-icon-browser` の同時接続でカスケード配置される。
- [x] **W2'〜W5'**（2026-09-05、セッション21–22）: QEMU 実起動で
      `gtk3-demo` がウィンドウを描画し、ポインタ入力に応答する。
      ヘッダバー・デモ一覧・ノートブック・本文まで、ホスト・ハーネスと同じ絵が
      hosted バックエンド（WM のウィンドウ 1 枚）の中に出る。リストをクリック
      すると選択とタイトルが追従する。到達までに潰したバグは §4。
- [ ] **W6**: WM を落として panel バックエンドへの昇格を確認、
      および WM 不在ブートでの `panel` 直起動。

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

- ~~**`SCM_RIGHTS` が未実装**~~ → K2（セッション20）で受信側、
  `SYSCALL_MEMFD_FROM_SHM`（G3 完了時）で送信側が揃った。
- ~~**共有メモリのクロスプロセス・コヒーレンシ**~~ → 実証済み。クライアントが
  描いた `wl_shm` プールがそのまま compositor 側で読めている（§4 #9〜#12）。
- **`wl_shm` プールの拡張は予約分まで**。昇格時に 2 冪（最低 1 MiB）で確保し、
  その範囲でしか伸ばせない（共有オブジェクトは真のリサイズができない）。
  超えると `[memfd] grow past reservation` を出して失敗する。真のリサイズには
  既存マッピングの張り替えが要る。
- **単一 ISO サイズ**。GTK3 閉包 + MIME DB + Adwaita でステージは約 **310 MiB**。
  `INSTALL_DISK_IMAGE_SIZE_MB` = 2560 の余裕内。
- **QEMU 実起動は TCG（この環境に `/dev/kvm` の権限が無い）**。起動からデスクトップ
  まで約 90 秒、`gtk3-demo` のウィンドウが出るまで更に数分かかる。KVM が使える
  環境なら桁で速くなるはず。

---

## 4. 修正したカーネル/ランタイム側のバグ（セッション21–22、2026-09-05）

QEMU 実起動で `gtk3-demo` を起動し、詰まるたびに原因を特定して潰した記録。
どれも GTK3 固有ではなく、外来 Linux ABI 全体に効く。

| # | 症状（クライアント側から見えたもの） | 原因 | 修正 |
|---|---|---|---|
| 1 | レジストリ 264 バイトを完全に受信した直後に恒久停止。要求を一切送らない | `recvmsg(2)` が `flags` を無視。libwayland は **`MSG_DONTWAIT`** で読むのに、EAGAIN が「ブロッキング再試行」に化けてイベントキューが空になった瞬間に固まる | `Syscall_LinuxCompat.c`: `recvmsg` / `recvfrom` で `MSG_DONTWAIT` を尊重 |
| 2 | クライアントが子プロセスを spawn した直後に compositor のソケットが `rx-BADF` に | AF_UNIX の fd が**全プロセス共通の番号空間**なのに `close()` に所有者チェックが無く、`posix_spawn` の子が glibc の closefrom フォールバックで他プロセスのソケットまで閉じていた | `UnixSocket.c`: `unix_socket_close()` は `owner_pid` が呼び出し元のときだけ閉じる |
| 3 | `GLib-CRITICAL: Failed to get RW lock: Resource deadlock avoided` が出続け、GSettings 初期化が無限ループ | `LINUX_CLONE_PARENT_SETTID` が **`0x00008000`（= `CLONE_PARENT`）**、正しくは `0x00100000`。新規スレッドの glibc `pd->tid` が 0 のままになり、rwlock が「未保持ロックの `__cur_writer`(=0) == 自分の tid(=0)」を自己デッドロックと誤検知 | 定数を修正 |
| 4 | fontconfig がキャッシュを `/tmp` に書く所でカーネルが `free()` 内で #PF | カーネル `realloc()` のその場拡張が隣の空きブロックを吸収するとき `block->next` は書くのに**新しい後続ブロックの `prev` を直していない** | `Memory_Main.c`: splice の両端を直し、`heap_search_hint` も追従 |
| 5 | （潜在）ライブラリのデータがゼロで読める | ファイルマッピング表が全プロセス共有 256 件、GTK3 クライアント 1 つで **291 件**登録して静かに溢れる | `FileMap.c`: 1024 件へ。溢れとショートリードを可視化 |
| 6 | `gtk_init` の途中で無反応（syscall も出さない） | D-Bus が無いのに `DBUS_SESSION_BUS_ADDRESS` も無く、GDBus が `dbus-launch` を `posix_spawn` して待ち続ける | `glibc_envp` に `DBUS_SESSION_BUS_ADDRESS` / `NO_AT_BRIDGE` |
| 7 | ローダも画像データも正しいのに `Unrecognized image file format` で `g_assert` 死 | gdk-pixbuf は形式判定に GIO の `g_content_type_guess()` を使う。**shared-mime-info の `mime.cache` が無いと何も名乗り出ない** | `stage-gtkdata.sh` が同梱の `update-mime-database` で生成 |
| 8 | `Failed to load cursor theme Adwaita` → `cursor_theme_name` の `g_assert` 死 | カーソルテーマが 1 つも無い | `adwaita-icon-theme` を vendoring |
| 9 | `wl_shm` バッファが 1 つも作れない（=カーソルテーマも読めない） | `fallocate(2)` が ENOSYS。libwayland は `memfd_create` + `posix_fallocate` でプールを作るので、glibc の手書きフォールバックがゼロ長 memfd に書けず失敗する | `Syscall_LinuxCompat.c`: `fallocate` を実装 |
| 10 | プールの**拡張**が `ftruncate grow on shm-backed memfd unsupported` で失敗 | 共有メモリオブジェクトはリサイズ不可 | 昇格時に 2 冪（最低 1 MiB）で予約し、その範囲内の拡張を許可。compositor 側に `wl_shm_pool.resize` を実装（新設の `SYSCALL_SHARED_MEMORY_SIZE` で予約量を確認してから受理） |
| 11 | `SCM_RIGHTS: shared_memory_grant failed` | 共有メモリの所有者判定が**スレッド単位の pid**。GTK は memfd を作ったスレッドと fd を送るスレッドが違う | `SharedMemory.c`: 同一性をアドレス空間（`process_memory_owner_pid_of`）で判定 |
| 12 | `[wl] create_pool without an fd`（fd の値が **0** で届く） | libwayland は fd を送る前に `fcntl(fd, F_DUPFD_CLOEXEC, 0)` で複製する。ImplusOS はファイル表に 0/1/2 を持たないので**空きに見えて fd 0 を返し**、しかも memfd の複製が `g_memfds[]` を引き継がず shm backing を失っていた | `Syscall_File.c`: dup は 3 番から探す。memfd の複製は memfd 状態を複写して共有オブジェクトの参照を取る |

**持ち込んだ計測手段**（今後の bring-up 用に残してある）:

- `Wayland.c`: `-DWLC_PROTOCOL_TRACE` のリクエスト・トレースに上限
  （`WLC_TRACE_MAX`、既定 600 行）。COM1 は 1 バイト 1 syscall なので、
  上限が無いとトレース自体が測りたいタイミングを変えてしまう。
  bind したインタフェース名は常時ログ。
  compositor の Makefile に `WLC_EXTRA_CFLAGS` フック。
- `UnixSocket.c`: `rx-EAGAIN` はトレースしない（ポーリングで即座に上限を
  食い潰し、肝心の tx/rx が見えなくなっていた）。上限 512。
- `Syscall_LinuxCompat.c`: `-DLINUX_SYSCALL_TRACE` は **AF_UNIX を触るまで
  発火しない**（ld.so の 4 万 syscall を飛ばす）。上限 `LINUX_TRACE_MAX` 行。
- `IDT_Main.c`: #PF ダンプに `pid=` と、KASLR スライドを割り出すための
  既知シンボルの実行時アドレスを追加。
- `Kernel/Source/Makefile`: strip 前の ELF を `Kernel_Main.ELF.sym` として残す
  （スライドを引けば `nm` / `objdump` でフォルト RIP が関数名になる）。
- `Tests/run.sh`: フォント / GSettings / gdk-pixbuf の探索先を全部ステージ側に
  向ける。以前はビルドホストの GTK を拾っていて、イメージに無いデータでも
  ハーネスが通ってしまっていた。

---

## 5. 追試の作法: ゲスト内プローブ

§4 の #7 は「ローダはある・データも正しい・それでも判定が失敗する」という
外から見分けのつかない状態で、カーネル側をいくら覗いても分からなかった。
決め手はゲスト内でライブラリに直接問い合わせる小さな Linux バイナリで、
ビルドホストの gcc で `-ldl` だけリンクして `/usr/bin` に置き、
`com.ImplusOS.gtk3demo` の起動先をそれに差し替えて動かした:

```c
void *h = dlopen("libgdk_pixbuf-2.0.so.0", RTLD_NOW | RTLD_GLOBAL);
GSList *(*get_formats)(void) = dlsym(h, "gdk_pixbuf_get_formats");
/* 登録済み 13 形式（png/jpeg 含む）を列挙 → ローダ不在ではない  */
/* gdk_pixbuf_loader_new_with_type("png") は成功 → デコーダも健全  */
/* gdk_pixbuf_new_from_file() だけ失敗    → 判定（sniffing）だけが壊れている */
```

ここまで絞れて初めて `g_content_type_guess()` と MIME データベースに辿り着けた。
同種の「ライブラリの内部状態を知りたい」場面ではこの手が一番速い。

---

## 6. 参照

- glibc 動的リンク基盤: [`TODO_glibc_Port.md`](TODO_glibc_Port.md)（実装ログ §11 セッション1–15）
- カーネル Linux ABI 拡充: [`TODO_Chromium_LinuxABI.md`](TODO_Chromium_LinuxABI.md)
- vendoring 機構: `Vendor/LinuxRuntime/README.md`, `resolve.sh`, `stage.sh`, `stage-gtkdata.sh`
- 外来 ABI 既定 envp: `Kernel/Core/process/ProcessManager_Create.c`（`glibc_envp`）
- ランチャ雛形: `Userland/Application/{Doom,Chromium}/Start.c`
- ネイティブ・コンポジタ（ブリッジ先）: `Userland/Application/com.ImplusOS.windowmanager/`
- AF_UNIX: `Kernel/IPC/UnixSocket.c`
