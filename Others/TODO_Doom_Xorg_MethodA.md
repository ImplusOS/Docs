# ImplusOS — Doom 実行プラン（方法A: 無改変 Xorg + ソフト GL）

> **ステータス: 2026-08-29。**
> **P1（ランタイム取得・ステージ・ランチャ配線・envp/cwd）完了・検証済み。**
> **M1〜M5（カーネル: /tmp tmpfs・/dev/dri/card0 char device・Linux DRM/KMS
> エミュレーション・デバイス fd mmap・/dev/input/event* + evdev ioctl）
> 実装済み・`make kernel` 通過。ただし QEMU 未ブートのため動作は未検証。**
> **M6（Xorg 立ち上げ反復 + 実描画確認）未了 — QEMU ブートループが必須で、
> 本セッション環境（カーネルビルドのみ可・ブート不可）では到達できない。**

調査基準日: 2026-08-29
対象バイナリ: `Userland/Application/Doom/Resource/linuxxdoom-x86_64`
（[EVV1E/DOOM](https://github.com/EVV1E/DOOM) の無改変ビルド、GPLv2）

関連: [`TODO_glibc_Port.md`](TODO_glibc_Port.md)（ランタイム同梱機構）、
[`TODO_Chromium_LinuxABI.md`](TODO_Chromium_LinuxABI.md)（Linux syscall 互換層）、
[`TODO_GTK3_Wayland_LinuxABI.md`](TODO_GTK3_Wayland_LinuxABI.md)（同型の外来 GUI 実行）。

---

## 0. 制約と到達点の定義

**制約（ユーザ指定）**: Doom バイナリは無改変。外部ソースコード追加なし。
追加できるのは「ビルド済みバイナリ／データファイル」と ImplusOS 側の配線のみ。

`linuxxdoom-x86_64` は無改変だと必ず **X11 + GLX クライアント**として動く
（`DT_NEEDED`: `libX11` `libm` `libopenal.so.1` `libGL.so.1` `libGLEW.so.2.2`
`libfluidsynth.so.3` `libasound` `libc`）。よって実画面到達には
(1) X サーバ本体 と (2) GLX が通る GL ドライバ の両方が要り、少なくとも一方が
Linux デバイスノード経由で ImplusOS のフレームバッファへ書く必要がある。

**方法A**: Debian trixie の **Xorg（modesetting DDX 内蔵・無改変）** ＋
**Mesa ソフト GL（llvmpipe / kms_swrast）** をビルド済みのまま同梱し、カーネルの
KMS shim を `/dev/dri/card0` として Linux ioctl/mmap ABI で見せて接続する。
Doom はただの GLX クライアントとして起動する。

| マイルストン | 到達点 |
|---|---|
| **P1（済）** | ランタイム全依存 `.so` 閉包 + Xorg/Mesa/xkb/フォント/`xorg.conf` を OS イメージへステージ。ランチャが Xorg→Doom の順に spawn。ホストで `ld.so --library-path <stage> linuxxdoom` が依存解決まで進む。 |
| **P2** | ImplusOS 上で `Xorg :0` が `/dev/dri/card0` を開き、`GETRESOURCES`→`SETCRTC` までログが進む。 |
| **P3** | `linuxxdoom` が `XOpenDisplay(":0")` 成功、`glXCreateContext`（llvmpipe）成功、`glewInit()` 通過。 |
| **P4** | Doom タイトル画面が ImplusOS のコンソールに描画される（`PAGE_FLIP`→`display_present()`）。 |
| **P5** | `/dev/input/event*` 経由でキーボード入力が Xorg→Doom に届き、操作可能。 |

---

## 1. P1 でやったこと（このコミット）

| パス | 種別 | 内容 |
|---|---|---|
| `Vendor/LinuxRuntime/resolve.sh` | 変更 | 閉包起点を `CHROME_BIN` 単体 → `FOREIGN_BINS`（空白区切り）に一般化。`SONAME2PKG_OVERRIDE` 追加（`libSDL2-2.0.so.0` を実体入り `libsdl2-2.0-0` に固定：`libfluidsynth.so.3` が直接 NEEDED、`libsdl2-compat-shim` は `.so` 無し）。 |
| `Vendor/LinuxRuntime/Makefile` | 変更 | `DOOM_BIN` / `FOREIGN_BINS` 追加。`xorgdata` ターゲット追加。 |
| `Vendor/LinuxRuntime/resolve.sh` | 変更（再掲） | seed パッケージの ELF 走査をパス不問（`file` マジック判定）に拡大。`/usr/lib/xorg/Xorg` のような bin/ にも `.so` 名でもない実体の NEEDED（`libunwind.so.8` 等）を取りこぼさないため。 |
| `Vendor/LinuxRuntime/packages.seed.txt` | 変更 | Doom 直接依存（`libgl1` `libglx0` `libglvnd0` `libglew2.2` `libopenal1` `libfluidsynth3` `libsdl2-2.0-0`）＋ X サーバスタック（`xserver-xorg-core` `xserver-xorg-legacy` `libgl1-mesa-dri` `libglx-mesa0` `libxcvt0` `libxfont2` `xkb-data` `x11-xkb-utils` `xfonts-base` `xfonts-encodings` `xfonts-utils`）を seed に追加。`libunwind8` `libaudit1` `libcap-ng0` は Xorg 実体から推移的に取得。 |
| `Vendor/LinuxRuntime/packages.lock` | 再生成 | 119 → **170** パッケージ（trixie / snapshot `20250901T000000Z` ピン、sha256 込み）。 |
| `Vendor/LinuxRuntime/closure.txt` | 再生成 | 133 → **177** soname、未解決 **0**。 |
| `Vendor/LinuxRuntime/stage-xorg.sh` | 新規 | `.so` 閉包以外の X 実行時データを配置：`/usr/bin/Xorg`、`/usr/lib/xorg/modules/**`（`modesetting_drv.so` / `extensions/libglx.so` ほか）、`/usr/lib/x86_64-linux-gnu/dri/**`（`swrast_dri.so` / `kms_swrast_dri.so`）、`/usr/share/X11/xkb/**` + `xkbcomp`、`/usr/share/fonts/X11/{misc,encodings}/**`、`/etc/X11/xorg.conf`。 |
| `Makefile` | 変更（1 行） | `linux_runtime_stage` の `-C` 呼び出しに `xorgdata` を追加。 |
| `Userland/Application/Doom/Start.c` | 変更 | `BUSYBOX_PATH`→`DOOM_PATH` 改名。`/usr/bin/Xorg :0 -config /etc/X11/xorg.conf -nolisten tcp -novtswitch -keeptty +iglx -logfile /dev/shm/Xorg.0.log` を spawn → `sleep_ms(4000)` → Doom spawn → Doom 終了で Xorg を kill。 |
| `Userland/Application/Doom/Resource/fetch-wad.sh` | 新規 | DOOM シェアウェア IWAD を取得（md5/size/マジック検証）。 |
| `Userland/Application/Doom/Makefile` | 変更 | `all:` に `wad` ターゲット追加（初回のみ自動取得、失敗しても継続）。 |
| `Userland/Application/Doom/.gitignore` | 新規 | `doom1.wad` / `soundfont.sf2` を非コミット。 |
| `Kernel/Core/process/ProcessManager_Create.c` | 変更 | (a) 外来 ELF spawn 時、子プロセスの cwd を実行体のあるディレクトリに設定（`IdentifyVersion()` の `access("./doom1.wad")` 対策）。(b) `glibc_envp` に `LIBGL_ALWAYS_SOFTWARE=1` `GALLIUM_DRIVER=llvmpipe` `XKB_CONFIG_ROOT=/usr/share/X11/xkb` `DOOMWADDIR=/Userland/Doom/Resource` `SOUNDFONT=/Userland/Doom/Resource/soundfont.sf2` を追加。 |
| `Kernel/IPC/UnixSocket.c` | 変更（1 行） | 先行の未コミット編集に `EPOLL*/POLL*` というコメント内 `*/` があり早期クローズしてビルド不能だったのを修正（`EPOLL / POLL`）。方法A とは独立の既存バグ。 |

検証:
```
make -C Vendor/LinuxRuntime resolve   # 177 sonames, 0 unresolved
make -C Vendor/LinuxRuntime stage     # placed=177 missing=0
make -C Vendor/LinuxRuntime xorgdata  # Xorg(ELF実体) + 11 modules + DRI + xkb + fonts + xorg.conf
make -C Vendor/LinuxRuntime verify    # closure satisfied
make kernel                           # OK（cwd + envp 変更込み）
make -C Userland/Application/Doom      # Doom.ELF OK
```
ステージ済みツリーへの素振り（ホスト x86_64 Linux、実体のみ使用）:
```
S=Build/x86_64/LinuxRuntime/stage
$S/lib64/ld-linux-x86-64.so.2 --library-path $S/usr/lib/x86_64-linux-gnu $S/usr/bin/Xorg -version
  -> "X.Org X Server 1.21.1.16" が表示（Xorg の .so 閉包充足）
(cd Userland/Application/Doom/Resource && .../ld-linux-x86-64.so.2 --library-path .../x86_64-linux-gnu ./linuxxdoom-x86_64)
  -> "W_InitFiles: no files found"（WAD 未配置。全 .so は解決済み・実行到達）
```
Doom + Xorg + 全 xorg モジュール + 全ステージ `.so` の DT_NEEDED 再帰チェック → 未充足 **0**。

---

## 2. ランタイムのフレーム経路（P4 到達時）

```
linuxxdoom  --glXSwapBuffers-->  libGL(glvnd) --> libGLX_mesa --> libgallium(llvmpipe)
   |                                                                   |
   |  X11 protocol (unix:/tmp/.X11-unix/X0)              kms_swrast: CREATE_DUMB/MAP_DUMB
   v                                                                   v
 Xorg (modesetting DDX, ShadowFB)  --DRM ioctl(/dev/dri/card0)-->  カーネル DRM/KMS shim
   |  DamageReport -> shadow blit -> DRM_IOCTL_MODE_DIRTYFB / PAGE_FLIP
   v
 カーネル: dumb BO --memcpy--> display_get_framebuffer()  ; display_present()
   v
 ImplusOS コンソール画面
```

---

## 3. カーネル作業（M1〜M5 実装済み・`make kernel` 通過・QEMU 未検証 / M6 未了）

### 実装済みの変更（このコミット、B: ブラインド実装）

| ファイル | 変更 |
|---|---|
| `Kernel/Core/vfs/TmpFS.c` | フラット名前空間を `/dev/shm` に加え `/tmp` `/run` でも受理（`tmpfs_path_ok`）。`mkdir` は tmpfs 配下なら成功（ディレクトリは暗黙）。`opendir`/`readdir` を任意サブディレクトリ対応（直下の子だけ列挙）。`TMPFS_MAX_FILES` 64→256。 |
| `Kernel/Core/kernel_main.c` | `vfs_mount("/tmp",...)` / `vfs_mount("/run",...)`。`drm_kms_init()`。 |
| `Kernel/include/kernel/interfaces/vfs_types.h` | `vfs_driver_t` に任意フック `dev_ioctl` / `dev_read` / `dev_poll` / `dev_mmap` を追加（既存 FS 無影響）。 |
| `Kernel/Core/vfs/VFS.c` / `VFS.h` | `vfs_file_is_chardev` + `vfs_dev_{ioctl,read,poll,mmap}` ラッパ。 |
| `Kernel/Core/vfs/DevFS.c` | ノード追加: `/dev/dri/card0`・`/dev/dri/renderD128`・`/dev/input/event0`・`/dev/input/event1`。`dev_*` フック実装（DRM は `drm_kms_*`、evdev は `evdev_*` に委譲、read は user バウンス）。close で `drm_kms_close()`。 |
| `Kernel/Core/drm/DRM_Kms.c` / `.h`（新規） | 実 Linux DRM/KMS。`_IOC` デコード + `drm_mode.h` レイアウト。VERSION / GET_CAP / SET_CLIENT_CAP / GETRESOURCES(2パス) / GETCONNECTOR(+modeinfo) / GETENCODER / GET・SETCRTC / CREATE・MAP・DESTROY_DUMB / ADDFB・ADDFB2・RMFB・GETFB / PAGE_FLIP(+flip-complete イベント) / DIRTYFB / GETPLANERES・GETPLANE / OBJ_GETPROPERTIES / ATOMIC→`-EOPNOTSUPP`（legacy 強制）/ GEM_CLOSE。dumb BO は `pmm_alloc_pages`、`PAGE_FLIP`/`DIRTYFB`/`SETCRTC` で `display_get_framebuffer()` へブリット + `display_present()`。read で `drm_event_vblank` を排出。 |
| `Kernel/Core/syscall/Syscall_File.c` / `.h` | `used==1` fd で driver が `dev_read`/`dev_poll` を持てばそちらへ。`syscall_file_is_chardev` / `syscall_file_ioctl` / `syscall_file_dev_mmap` を追加。 |
| `Kernel/Compat/Linux/Syscall_LinuxCompat.c` | `syscall_ioctl_ex`: chardev fd を `syscall_file_ioctl` へ（`arg==0` ガードより前）。`linux_mmap`: chardev fd を `syscall_file_dev_mmap` へ。 |
| `Kernel/Core/process/ProcessManager_Create.c` | 外来 ELF spawn 時の子 cwd = 実行体ディレクトリ。`glibc_envp` に `LIBGL_ALWAYS_SOFTWARE` / `GALLIUM_DRIVER=llvmpipe` / `XKB_CONFIG_ROOT` / `DOOMWADDIR` / `SOUNDFONT`。 |
| `Kernel/Drivers/Module/Evdev_Client.c` | `evdev_ioctl` を stub から実装（`EVIOCGVERSION`/`EVIOCGID`/`EVIOCGNAME`/`EVIOCGPHYS`/`EVIOCGPROP`/`EVIOCGBIT(0/KEY/REL)`/`EVIOCGKEY・LED・SW`/`EVIOCGABS`/`EVIOCGRAB`/`EVIOCSCLOCKID`）。dev0=キーボード, dev1=相対ポインタ。 |
| `Vendor/LinuxRuntime/packages.seed.txt` ほか | `xserver-xorg-input-evdev` 追加（lock 173 pkg / closure 179 soname / 未解決 0）。`stage-xorg.sh` が `evdev_drv.so` 同梱 + `xorg.conf` に InputDevice(kbd0/mouse0→/dev/input/event0,1) 追加。 |

**未検証の既知リスク（QEMU で潰す）**: `drm_kms_mmap` の `process_user_reserve`+`paging_map_user_page` による BO マップ実効性 / dumb BO を `process_user_alloc` でなく物理ページにしたことと Xorg 単一プロセス前提の整合 / `drm_kms_read` が user ポインタ前提（`linux_read` は user ポインタを直接渡す = OK のはず）/ TmpFS フラット名前空間で X のロックファイル生成 / evdev bitmap の網羅性 / llvmpipe JIT の `PROT_EXEC`（`syscall_vm_mprotect` は EXEC 対応済み）。

### M1. `/tmp` と `/run` を tmpfs に  — 実装済み

**現状**: `TmpFS` は `/dev/shm` のみ（`Kernel/Core/vfs/TmpFS.c` `TMPFS_PREFIX`）。
`/tmp` はどの FS にもマウントされていない。`glibc_envp` は `HOME=/tmp`
`XDG_RUNTIME_DIR=/tmp` を指しているが実体が無い。
**X の UNIX ソケットパス `/tmp/.X11-unix/X0` は libX11/Xserver にコンパイル時固定**で、
無改変では変えられない。

**作業**:
- `TmpFS` を複数プレフィックスに対応させる（`/dev/shm` `/tmp` `/run`）か、`/tmp`
  専用にもう 1 インスタンス mount する（`kernel_main.c` の vfs_mount 群）。
- `TmpFS` にサブディレクトリ作成を実装（`mkdir("/tmp/.X11-unix")`、`mkdir("/tmp/.cache")` …）。
  現状 `tmpfs_vfs_mkdir` の有無・ネストパス対応を確認。上限（`TMPFS_MAX_FILES 64` /
  `TMPFS_PATH_MAX 256`）を X 用に引き上げ。
- AF_UNIX 抽象名前空間（`@/tmp/.X11-unix/X0`、先頭 `\0`）対応があれば socket ファイル
  実体無しでも可。`Kernel/IPC/UnixSocket.c` の bind/connect はパス文字列比較なので
  抽象アドレス（NUL 始まり、長さ指定）を扱えるよう小改修すれば M1 の FS 部分を回避可。

**確認**: `Xorg` 起動ログに `unix:/tmp/.X11-unix/X0` の bind 成功、`ls /tmp/.X11-unix`。

### M2. `/dev/dri/card0` キャラクタデバイス + fd ルーティング

**現状**: `Kernel/Core/drm/DRM_Client.c` は `SYSCALL_DRM_*`（206–210）という
**ImplusOS ネイティブ syscall 専用**で、fd base は偽の `0x6000`。`open("/dev/dri/card0")`
からは到達不能。`DevFS`（`Kernel/Core/vfs/DevFS.c`）は `null/zero/full/urandom/random/tty`
のみ。`vfs_driver_t` に `ioctl`/`mmap` フックが無い。Linux 互換層
（`Syscall_LinuxCompat.c` `syscall_ioctl_ex`）は TTY ioctl だけ。

**作業**:
1. `DevFS` に `/dev/dri/card0`（と将来用 `/dev/dri/renderD128`、`/dev/fb0`）を追加。
   `devfs_entry_t` に `kind = DEVFS_KIND_DRM_CARD` を新設。
2. デバイス fd バックエンドの概念を fd レイヤに導入。最小案:
   - `file_t`（`vfs_types.h`）にデバイス種別タグを持たせる、または
   - `Syscall_File.c` / `Syscall_LinuxCompat.c` の `read`/`write`/`ioctl`/`mmap`/`close`/`poll`
     ハンドラで「この fd が DevFS の DRM ノードなら `drm_kms_*()` に委譲」する分岐を追加。
   - `vfs_driver_t` に任意フック `int64_t (*ioctl)(vfs_file_t*, uint64_t req, uint64_t arg)` /
     `int64_t (*mmap)(vfs_file_t*, uint64_t off, uint64_t len, uint32_t prot, void** out)` /
     `int64_t (*poll)(vfs_file_t*, uint32_t events)` を足すのが素直（NULL 可・既存 FS 無影響）。
3. Linux 互換の `open` は既に VFS 経由で `/dev/*` を引ける（`/dev/null` 等が動いている）
   ので、`/dev/dri/card0` の `find_file` を返せば `open` は通る。`O_CLOEXEC`/`O_NONBLOCK`
   フラグ保持を確認。

**確認**: Linux バイナリから `open("/dev/dri/card0", O_RDWR)` が >= 0 を返す。

### M3. 実 Linux DRM/KMS ioctl エミュレーション（`DRM_Client.c` を置換）

**現状の `drm_client_ioctl` は使い物にならない**:
- 要求番号が偽物（`DRM_IOCTL_VERSION 0xC0` 等）。実際の Linux は `_IOC` エンコード
  （例 `DRM_IOWR(0x00, struct drm_version)` = `0xC0186400`、
  `DRM_IOWR(0xA0, struct drm_mode_card_res)` = `0xC04064A0`）。
- 構造体レイアウトが実物と不一致（`drm_mode_get_connector` / `drm_mode_modeinfo` /
  `drm_mode_card_res` …）。

**作業**: `Kernel/Core/drm/DRM_Kms.c`（新規、`DRM_Client.c` は削除か native 専用に隔離）。
`<drm/drm.h>` `<drm/drm_mode.h>` のレイアウトに正確に合わせる。`_IOC` の
dir/type/nr/size を自前デコード。既存の `display_get_framebuffer()` へのブリットは流用。

必要な ioctl（modesetting DDX + kms_swrast + libdrm が投げるもの）:

| ioctl | 構造体 | 実装要点 |
|---|---|---|
| `DRM_IOCTL_VERSION` | `drm_version` | name=`"implusdrm"` 相当、`*_len` 2 パス（長さ問い合わせ→本体）対応必須 |
| `DRM_IOCTL_GET_CAP` | `drm_get_cap` | `DRM_CAP_DUMB_BUFFER=1`、`DRM_CAP_CRTC_IN_VBLANK_EVENT=1`、`PRIME=0`、`ADDFB2_MODIFIERS=0` |
| `DRM_IOCTL_SET_CLIENT_CAP` | `drm_set_client_cap` | `UNIVERSAL_PLANES` / `ATOMIC` を受理（0 でも可、`Atomic"false"` なので） |
| `DRM_IOCTL_SET_MASTER` / `DROP_MASTER` | — | 0 |
| `DRM_IOCTL_MODE_GETRESOURCES` | `drm_mode_card_res` | crtc/connector/encoder 各 1、fb 0。2 パス（count 問い合わせ→ID 配列書き込み）対応必須 |
| `DRM_IOCTL_MODE_GETCONNECTOR` | `drm_mode_get_connector` | connected、`count_modes=1`（`display_width/height()` から 1 モード）、`count_encoders=1`、`count_props` は 0 でも DDX は動く。2 パス対応必須。`drm_mode_modeinfo` を 1 個返す |
| `DRM_IOCTL_MODE_GETENCODER` | `drm_mode_get_encoder` | `encoder_id=1`、`crtc_id=1`、`possible_crtcs=1` |
| `DRM_IOCTL_MODE_GETCRTC` | `drm_mode_crtc` | 初期は `mode_valid=0` |
| `DRM_IOCTL_MODE_SETCRTC` | `drm_mode_crtc` | `fb_id`/`mode`/`set_connectors_ptr` を記録。以降の `PAGE_FLIP`/`DIRTYFB` の宛先 |
| `DRM_IOCTL_MODE_GETPLANERESOURCES` / `GETPLANE` | `drm_mode_get_plane_res` / `drm_mode_get_plane` | primary plane 1 個（`UNIVERSAL_PLANES` を返した場合） |
| `DRM_IOCTL_MODE_OBJ_GETPROPERTIES` / `GETPROPERTY` | — | 空（`count=0`）で可（legacy パス） |
| `DRM_IOCTL_MODE_CREATE_DUMB` | `drm_mode_create_dumb` | `process_user_alloc`（既存ロジック流用）。pitch=`width*bpp/8` |
| `DRM_IOCTL_MODE_MAP_DUMB` | `drm_mode_map_dumb` | `offset` に「mmap 用の擬似オフセットトークン」を返す（M4 が解決）。**現状の「カーネルポインタをそのまま返す」実装は Linux プロセスでは無効**なので置換 |
| `DRM_IOCTL_MODE_DESTROY_DUMB` | `drm_mode_destroy_dumb` | 解放 |
| `DRM_IOCTL_MODE_ADDFB` / `ADDFB2` | `drm_mode_fb_cmd` / `_cmd2` | handle→fb_id。`_cmd2` は plane[0]/pitch[0] を見る |
| `DRM_IOCTL_MODE_RMFB` | uint32 | 解放 |
| `DRM_IOCTL_MODE_PAGE_FLIP` | `drm_mode_crtc_page_flip` | dumb BO を `display_get_framebuffer()` へブリット→`display_present()`。`DRM_MODE_PAGE_FLIP_EVENT` 指定時は完了 `drm_event_vblank` を fd の read キューへ積む |
| `DRM_IOCTL_MODE_DIRTYFB` | `drm_mode_fb_dirty_cmd` | ShadowFB 経路の実描画。矩形リストぶんだけ部分ブリット |
| `DRM_IOCTL_MODE_ATOMIC` | `drm_mode_atomic` | `Atomic"false"` 前提なので `-EOPNOTSUPP`（`-95`）で可 |
| `DRM_IOCTL_GEM_CLOSE` | `drm_gem_close` | handle 解放 |
| `DRM_IOCTL_MODE_CREATE_LEASE` 等 | — | `-EINVAL` |

fd の `read()`（`drmHandleEvent` が `PAGE_FLIP_EVENT` 完了を待つ）: `drm_event_vblank`
（`struct drm_event` ヘッダ + `user_data`）をノンブロッキング/ブロッキングで返す。
`poll()`: 完了イベントがキューにあれば `POLLIN`。

### M4. DRM fd の `mmap`（dumb BO）+ llvmpipe 用 `PROT_EXEC` mmap

**作業**:
- `MAP_DUMB` が返したオフセットトークンで `mmap(fd=/dev/dri/card0, offset=token)` が来たら、
  対応する dumb BO の物理ページを **呼び出しプロセスのユーザ空間**に実マップする
  （`Syscall_LinuxCompat.c` `linux_mmap` にデバイス fd 分岐を追加。`process_user_*` /
  ページテーブル API を使用）。現在の `drm_client_mmap` は `return (void*)offset` で
  カーネルポインタを渡すだけ＝Linux プロセスからは触れない。要置換。
- **llvmpipe は LLVM MCJIT で実行可能メモリを要求する**。`mmap(PROT_EXEC)` /
  `mprotect(...PROT_EXEC)` がユーザ空間で機能すること（W^X を保つなら
  `PROT_WRITE`→`PROT_EXEC` の遷移を許可）。`Syscall_VM.c` / `linux_mmap` / `linux_mprotect`
  を確認・対応。ここが無いと `glXCreateContext` 前後で SIGSEGV/`llvm::...` abort。
  代替: `GALLIUM_DRIVER=softpipe`（JIT 無し・激遅だが可）を envp で切替可能にしておく。

### M5.（P5 用）`/dev/input/event*` を Xorg に

`Kernel/Drivers/Module/Evdev_Client.c` に evdev リングは既にある（`SYSCALL_EVDEV_*`、
fd base `0x7000`、キーボード=dev0 / マウス=dev1）。M2 と同じ要領で
`/dev/input/event0` `/dev/input/event1` を `DevFS` に追加し `read`/`ioctl`（`EVIOCGBIT`
`EVIOCGABS` `EVIOCGNAME` `EVIOCGID` 等）を `evdev_ioctl` に委譲。
`xorg.conf` に `InputDevice`（`Driver "libinput"` は udev 依存なので `"evdev"` を明示、
無ければ `xserver-xorg-input-evdev` を seed 追加）または `-config` で
`Option "Device" "/dev/input/event0"`。`AutoAddDevices"false"` のままなら明示セクション必須。
P4 までは入力不要（Doom はタイトル画面まで進む）。

### M6. Xorg 立ち上げ反復 — QEMU 実測ログ（2026-08-29 1回目）

初回ブートで判明・対処済み:

| 症状 | 原因 | 対処（このコミット） |
|---|---|---|
| `Linking lock file (/tmp/.X0-lock) in place failed: Function not implemented` で X が即 fatal | `link(2)`/`linkat(2)` が `-ENOSYS`。X の `LockServer()` は `/tmp/.tXn-lock` を `link()` で `/tmp/.Xn-lock` へ置く | `syscall_file_link()` 新規（pseudo-fs 限定・内容コピーで link を模倣）＋ `LINUX_SYS_LINK`/`LINKAT` dispatch |
| Doom が `I_Init` 付近で `libpulse` の `pa_make_fd_cloexec()` assert → `Aborting.` | `fcntl(sockfd, F_GETFD)` がソケット fd で負値。libopenal がロードした PipeWire/Pulse バックエンドが接続ソケットに対して呼ぶ | `syscall_fcntl_ex` の `F_GETFD`/`F_SETFD` をソケット fd 対応（0 を返す/受理）＋ `glibc_envp` に `ALSOFT_DRIVERS=null` `SDL_AUDIODRIVER=dummy` `PULSE_SERVER=none`（音は無音で継続） |
| `Failed to create secure directory (/tmp/pulse)` | TmpFS の `mkdir` は成功を返すが実ディレクトリを `stat` できずパーミッション検査に落ちる | 上記の `ALSOFT_DRIVERS=null` で Pulse 経路自体を無効化して回避 |

**2回目ブート**（audio は `ALSOFT_DRIVERS=null` で通過、Doom は `ST_Init` まで到達し
`Couldn't connect to display!` → X 未起動のため。X は下記で fatal）:

| 症状 | 原因 | 対処（このコミット） |
|---|---|---|
| `_XSERVTransSocketOpenCOTSServer: Unable to open socket for local` / `unix` → `Cannot establish any listening sockets` で X fatal | `socket(AF_UNIX, SOCK_STREAM)` が `-ENOMEM`。グローバル `g_usocks` テーブル（`UNIX_SOCK_MAX=16`）が枯渇。**プロセス終了時に AF_UNIX ソケットを解放していなかった**（`syscall_file_close_all_for_pid` / `syscall_socket_close_all_for_pid` はあるが unix 版が無い）ため、アプリ起動のたびにスロットがリーク | `unix_socket_close_all_for_pid()` 新規＋プロセス終了処理から呼び出し。`UNIX_SOCK_MAX` 16→64。 |

**3回目ブート**（`Cannot establish any listening sockets` が再発）:

| 症状 | 原因 | 対処 |
|---|---|---|
| socket 枯渇を直しても `Unable to open socket for local/unix` | **`UNIX_SOCK_FD_BASE = 0x8000`**。Xtrans は `socket()` の戻り fd が `sysconf(_SC_OPEN_MAX)` 以上だと無条件で拒否する。`_SC_OPEN_MAX` = `RLIMIT_NOFILE` = `OS_CONFIG_FILE_MAX_FD` = 256 だったので、32768 起点の AF_UNIX fd は全て門前払い | `UNIX_SOCK_FD_BASE` を **256** に（fd 配置: ファイル 0–255 / AF_UNIX 256–319 / inet 512–575、全て FD_SETSIZE 未満）。`RLIMIT_NOFILE` を **1024** に。あわせて `read`/`write`/`readv`/`writev`/`fcntl(F_GETFL/F_SETFL)` を AF_UNIX fd 対応（X は素の `read`/`writev` を使う。従来は `recvmsg`/`sendmsg` しか通っていなかった） |

**4回目ブート**（X が `InitOutput` まで到達。fatal は `no screens found`）:

| 症状 | 原因 | 対処 |
|---|---|---|
| `(EE) Unable to locate/open config file: "/etc/X11/xorg.conf"` → 内蔵デフォルト設定で起動 | **2つの独立バグ**: ①トップ `Makefile` が `$(LINUX_RUNTIME_STAGE)` から `lib64` と `usr` しかイメージへコピーしておらず `etc` が入っていなかった ②`vfs_resolve_candidates()` が最長プレフィックス一致で **早期 return** するため、`/etc` にマウントされた EtcFS が同じ接頭辞のディスク上の実ファイルを完全に隠していた | ①`image_livecd` / `install_payload` に `cp -a …/etc` を追加 ②早期 return を削除し、プレフィックス一致ドライバを**先に**試したうえで default/catch-all を後段に積む（疑似 FS が実ファイルを隠さない） |
| `Failed to load module "glx"/"modesetting"/"fbdev"/"vesa" (module does not exist)` → `(EE) No drivers available` → `no screens found` | Xorg の `FindModuleInSubdir()` は `stat("<dir>/<name>/")` の **末尾スラッシュ付きパス**が `S_ISDIR` を返すことでサブディレクトリへ再帰する。ImplusOS のパス走査は末尾 `/` で最終コンポーネント処理後にループを抜けて「not found」を返していたため、`modules/drivers`・`modules/extensions`・`modules/input` が一度も探索されなかった（トップ直下の `libexa.so` 等だけ見えていた） | `vfs_normalize_path()` を新設し `vfs_find_file`/`vfs_opendir`/`vfs_creat`/`vfs_mkdir` の入口で末尾 `/` と連続 `//` を正規化。あわせて `iso9660_lookup_path()` 自体も末尾/重複スラッシュを許容 |
| （予防）`drmSetClientCap` に成功を返していた | UNIVERSAL_PLANES / ATOMIC に yes と答えると DDX が plane / atomic commit 経路に進んでしまう（shim 未実装） | `DRM_IOCTL_SET_CLIENT_CAP` を一律 `-EOPNOTSUPP` にしてレガシー modeset 経路へ固定 |

**5回目ブート**（4回目の修正後もモジュールが全滅：`glx`/`modesetting`/`evdev` すべて
`module does not exist, 0` → `No drivers available` → `no screens found`）:

| 症状 | 原因 | 対処 |
|---|---|---|
| `LoadModule` が **どのモジュールも**開けない（`FindModuleInSubdir()` が空を返す）。ステージ済み `.so` は ISO 上に正しい小文字パス（Rock Ridge / Joliet）で存在し、`ld.so` 経由の `.so` ロード（Doom エンジン初期化）は成功している | glibc の `opendir()` は **開いた fd に対し `fstat()` して `S_ISDIR` を確認し**、ディレクトリでなければ `NULL` を返す。ImplusOS の Linux 互換 `linux_stat_fd()`（`FSTAT` / `statx`+`AT_EMPTY_PATH`）は `syscall_file_get_file_info()` が `FILE_USED_DIR` fd を弾く → ソケットでもない → **`-EBADF`** を返していた。よって外来バイナリの `opendir()` が全て失敗し、Xorg のモジュール探索（`modules/{drivers,extensions,input}` を `opendir`/`readdir` で走査）が一つも見つけられなかった。`ld.so` は `open`+`mmap` だけで `opendir` を使わないため無傷だった | `syscall_file_is_dir(fd)` を新設（`Syscall_File.c/.h`）。`linux_stat_fd()` の中身を `linux_fill_stat_for_fd()` に切り出し、ディレクトリ fd を `S_IFDIR\|0755` / `st_blksize=4096` で報告。`linux_statx()` の `AT_EMPTY_PATH` 枝も同ヘルパへ集約（同じ穴があった）。`make kernel` 通過 |

（Doom 側の `CR2=0x968` ページフォルトは派生現象：X 未起動 → `XOpenDisplay` 失敗 → 無改変 `linuxxdoom` の `I_Error()` → `I_ShutdownGraphics()` が `XCloseDisplay(NULL)` を叩く。X が上がれば通らない。）

**6回目ブート**（`fstat` 修正が効き、`glx`/`modesetting`/`evdev` の3モジュールとも
`Loading /usr/lib/xorg/modules/.../*.so` → `Module ... vendor="X.Org Foundation"` で
ロード成功。`modeset(0)` が `/dev/dri/card0` を開き、コネクタ `DVI-I-1` connected /
`1280x800` を検出、ShadowFB 有効、拡張機能を順に初期化…）:

| 症状 | 状態 |
|---|---|
| `(II) Initializing extension GLX` の直後に `Xorg: symbol lookup error: /usr/lib/xorg/modules/extensions/libglx.so: undefined symbol: <名前は Doom のバナー出力に潰されて不明>` で X が即終了 | **調査中**。ステージ済みツリーには `libglx.so` の全 UND シンボル（サーバ提供の `serverClient`/`dixLookupWindow`/… は `Xorg` 実体が 2492 個エクスポート、`gl*` は glvnd `libGL.so.1` が 3470 個）が揃っており、NEEDED 閉包も欠けなし。よってファイル欠落ではなく **ld.so の実行時シンボル解決スコープの問題**の可能性が高い（`dlopen` したモジュールからメイン実行体のエクスポートが引けていない等）。 |

**7回目ブート**（`LD_BIND_NOW=1` を入れたら真因が露出）:

| 症状 | 原因 | 対処 |
|---|---|---|
| `(EE) Failed to load .../modesetting_drv.so: undefined symbol: gbm_bo_get_plane_count` → `Failed to load module "modesetting" (loader failed)` → `No drivers available` | `modesetting_drv.so` は **`gbm_*` を 12 個 UND 参照するのに `DT_NEEDED` に `libgbm.so.1` を持たない**（Debian 実物も同じ）。ストック環境では `AccelMethod "glamor"` が `libglamoregl.so`→`libgbm` を引き込むので解決する。本 xorg.conf は GPU 無しのため `AccelMethod "none"` 固定 → `libgbm` を誰もロードしない → リロケーション失敗。※6回目の遅延バインドでは `gbm_*` を呼ぶ経路（glamor/GPU BO）に入らず生存し、`Initializing extension GLX` まで到達していた（`libglx.so: undefined symbol` の正体は未確定だが、同じ「スコープに無いシンボル」系の可能性大）。 | `glibc_envp` に `LD_PRELOAD=libgbm.so.1` を追加（`libgbm` を `modesetting_drv.so` の `dlopen` 前にグローバルスコープへ載せる）。`libgbm.so.1` は closure 済み・NEEDED 閉包もステージ済み。`LD_BIND_NOW=1` は撤去（`modesetting_drv.so` の未使用 `gbm_*` PLT を誤って致命化するため）。`LD_WARN=1` と Doom settle 20000ms は診断のため残置。 |

**8回目ブート**（`LD_PRELOAD=libgbm.so.1` 追加後）:

| 症状 | 分析 |
|---|---|
| `modesetting_drv.so` ロード成功 → `modeset(0): using /dev/dri/card0` → コネクタ `DVI-I-1`/`1280x800` 検出 → ShadowFB 有効 → 全拡張機能初期化 → **`(II) Initializing extension GLX` 直後にまた `libglx.so: undefined symbol: <名前は今回も Doom バナーに潰された>`** で即終了 | `libgbm` プリロードで modesetting は完走。残る唯一の壁が libglx.so の GLX 拡張初期化。**7回目（`LD_BIND_NOW=1`）では libglx.so は eager バインドで完全解決してロード成功していた**のに、8回目（遅延バインド）は `GlxExtensionInit()` 呼び出し時点で未解決になる → 「ロード時スコープにはあるが呼び出し時スコープから消える」= **`dlclose` 等でスコープが縮む系**の疑い。`libglx.so` の全 UND シンボル（サーバ系は `Xorg` 実体が全エクスポート、`gl*` は glvnd `libGL.so.1`、`glxServer`/`lastGLContext`/`enableIndirectGLX` も `Xorg` の OBJECT）はツリー内に存在確認済み・TLS シンボルも無し。→ **シンボル名の確定が必須**。 |

**9〜10回目ブート**（`LD_BIND_NOW=1` を追加 → 真の失敗に到達）:

| 症状 | 分析 |
|---|---|
| 10回目: `(II) Initializing extension GLX` → `(II) AIGLX: Screen 0 is not DRI2 capable` → 直後に `[OS] [PF] CR2:0x38 RIP=libglx.so` の **SIGSEGV**（`Caught signal 11 ... Server aborting`、backtrace: `libglx.so` → libc シグナルフレーム）。9回目の `undefined symbol: <空>` は「glibc がシンボル名文字列を読む前に fault」= 遅延 PLT fixup 経路自体が壊れていた。`LD_BIND_NOW=1` で eager 解決に切替えたら **libglx.so は綺麗にロードし、GLX 拡張初期化まで進んだ**（＝遅延経路のバグを回避）。 | `NULL+0x38` は `__DRIswrastExtension::createNewScreen2`（オフセット 0x38）を **`screen->swrast==NULL`** で呼んだ形。`+iglx` の GLX swrast プロバイダ経路: `glxProbeDriver("swrast")` → `dlopen("swrast_dri.so")`（trixie では **`libdril_dri.so`** = 薄い DRI ローダ shim へのシンボリックリンク。全 `*_dri.so` が同一）→ `libdril_dri.so` は実体の `libgallium-25.0.7-2.so` と、**サーフェスレス EGL 経由でスクリーンを作るため `libEGL.so.1` を実行時 `dlopen`** する。**`libEGL.so.1` がステージに無い**（`libgl1`/`libglvnd0`/`libglx0` はあるが `libegl1`/`libegl-mesa0` が seed に無く、`dlopen` 依存なので DT_NEEDED 走査の closure にも入っていない）→ dril の EGL 初期化失敗 → `createNewScreen2` が NULL → 0x38 で死ぬ。※`libgbm` と同型の「`dlopen` 依存はリゾルバが拾わない」問題。 |

**現在地**: Xorg はモジュールロード〜`/dev/dri/card0` オープン〜コネクタ/モード検出〜ShadowFB〜**全拡張機能初期化**まで到達。落ちるのは GLX 拡張初期化だけ。

---

## M7. libglx.so の SIGSEGV = メイン実行体のデータシンボルが dlopen モジュールから引けない

**11回目ブート**（EGL 追加後も 10回目と1バイト違わぬ同一クラッシュ）:

```
(II) AIGLX: Screen 0 is not DRI2 capable
[OS] [PF] CR2:0x38  RIP:0x4100993411  Access: read, mode: user
Backtrace: 0:Xorg  1:libc.so.6(signal)  2:libglx.so
Segmentation fault at address 0x38 → Caught signal 11 → Server aborting
```

**逆アセンブルで確定した faulting 命令**（`libglx.so`, `glxProbeDriver`/GLX ディスパッチ近傍）:

```
2a1a7: mov 0x1cdf2(%rip),%r14   # 46fa0 → R_X86_64_GLOB_DAT  glxServer
2a1b4: mov $0xb,%ebp
2a1b9: call *0x38(%r14)          ← ここ。r14 = *(GOT[glxServer])
```

`glxServer` は **Xorg 実行体（PIE）だけが定義・エクスポートする 128B の OBJECT シンボル**
（`readelf`: `1790: 2756e0 128 OBJECT GLOBAL DEFAULT 22 glxServer`、バージョン無し。
どのステージ `.so` にも無い）。libglx.so はこれを `UND` で参照し `GLOB_DAT` で GOT へ解決する。
**その GOT スロットが 0** → `r14=0` → `call *0x38(0)` が `[0x38]` を read → `CR2=0x38`。

対して **FUNC シンボル `LegalNewID`（同じく Xorg 実行体のみ）は解決できている**
（直前の `2a19e: call LegalNewID@plt` を通り、戻り値非ゼロで `0x2a1a7` へフォールスルーして
いる＝クラッシュ到達自体が LegalNewID 解決の証拠。未解決なら PLT 経由で addr 0 へ jmp して
RIP=0 で死ぬ）。

**12回目ブート**（`LD_WARN=1` 撤去）: **`undefined symbol` は一切出ず**、クラッシュは
10〜11回目と完全同一（`CR2:0x38 RIP:0x4100993411`）。`LD_BIND_NOW=1` の eager 解決でも
`dlopen("libglx.so")` は成功しており（`Module glx: vendor=...` が出る）、
→ **当初の「`glxServer` GLOB_DAT が未解決で 0」説は否定された。libglx.so のシンボルは
全て解決できている。** これは reloc の穴ではなく**実行時の真の NULL 参照**。

**訂正した診断**: `+iglx` の GLX **swrast プロバイダ**経路
（`glxdriswrast.c:__glXDRIscreenProbe`）で:
```
screen->driver = glxProbeDriver(..., &screen->core, __DRI_CORE,
                                     &screen->swrast, __DRI_SWRAST);
...
(*screen->swrast->createNewScreen2)(...)   ← createNewScreen2 は __DRIswrastExtension の +0x38
```
`screen->swrast`（`__DRI_SWRAST` 拡張ポインタ）が **NULL** のまま `->createNewScreen2` を
呼んで `[NULL+0x38]` を read → `CR2=0x38`。かつ `glxProbeDriver` の
「does not export required extensions」ログは出ていない＝ core だけ検証して
swrast 未検証で返しているか、`__driDriverGetExtensions_swrast()` が `DRI_SWRast` を
含まないリストを返している。

**根本**: trixie の `swrast_dri.so` は **Mesa 25 の `libdril_dri.so`（薄い DRI ローダ shim）**
への symlink。この shim が xserver 21.1 の `+iglx` swrast プロバイダが要求する
**レガシー `__DRI_SWRAST` 拡張を(実質)提供していない**可能性が高い
（Mesa 24+ で古典 `swrast_dri.so` は廃止、`dril` は主に DRI2/DRI3 バックエンド互換用）。
= **Mesa 25 `dril` shim と xserver の `+iglx` ソフト GLX が非互換**というディストリ側の問題。
libEGL 追加が無影響だったのも、EGL 到達前にここで死んでいるため。

> **訂正（13回目ブート後）**: 上の「`glxServer` GLOB_DAT 未解決」も、その次の
> 「`screen->swrast` が NULL」も **どちらも確証が無い**。`LD_WARN=1` を外した 12 回目でも
> `undefined symbol` は出ず（＝未解決シンボル説と矛盾）、一方で faulting 命令の特定も
> 失敗している: RIP `0x4100993411` からページ整列した base を仮定すると候補は
> `.text+0x13411/0x20411/0x2b411/0x30411` の 4 つだが、`0x2b411` は `lea`（fault しない）、
> `0x30411` は書き込み（PF は read）、`0x20411` は同一構造体の `+0x4` 側が先に fault する
> はずで矛盾、`0x13411` は GL dispatch（`glGetColorTableParameteriv` 直前）で拡張初期化中には
> 到達しない上に PF が報告する RBP（`0x4100968380`）と必要値（`0x30`）が合わない。
> **GLX の原因は未特定。** 下の A〜D は依然として妥当な選択肢だが、根拠は
> 「`+iglx` の GLX プロバイダ初期化中に NULL 近傍アドレスを読んで落ちる」までである。

**現状の結論**: Xorg 本体は `/dev/dri/card0`〜モード検出〜ShadowFB〜全拡張初期化まで
健全に到達。残るのは「GPU 無し環境での動くソフト GL 経路」で、これはクイックフィックスでは
なく方法A の中核未解決部（TODO §M6 の既知リスク）。選択肢:

| 案 | 内容 | コスト |
|---|---|---|
| A | `+iglx` を捨て、Doom を **クライアント側 direct sw rendering（DRI3 + llvmpipe）** に。カーネル `/dev/dri/card0` shim に DRI3 + dma-buf(fd 受け渡し) を実装 | 大（カーネル） |
| B | 古い Mesa（`swrast_dri.so` が実体だった版, ~Mesa 23）を別途ステージして `+iglx` を成立させる | 中（パッケージング。snapshot から旧 Mesa を pin） |
| C | `xserver` を `--enable-glx --disable-dri*` かつ **`GLX_swrast`（`swrastwrap`）** 経由の純ソフト間接 GLX に寄せる。無改変 deb では不可＝ビルドが要る | 大 |
| D | GLX 無効化（`-extension GLX`）で X を起動状態まで持って行き、**DRM shim + ShadowFB + Xtrans + evdev の経路を単体で検証**（Doom は GL 初期化で失敗するが、X が画面を present できることの確認になる） | 小（診断） |

**次アクション（推奨: まず D で足場を固める）**:
- `Userland/Application/Doom/Start.c` の `XORG_ARGS` に `-extension GLX` を足し、Doom spawn は残す。
  X が `screen 0: 1280x800` で accept を回し、Doom が `XOpenDisplay(":0")` に成功する所まで
  行けば、方法A の非 GL 部分（M1〜M5＋Xtrans）は全て健全と確定する。そのうえで A か B を選ぶ。
- 参考: この経路とは別に判明した ImplusOS 側の実バグ = 「dlopen したモジュールの**遅延 PLT
  fixup が壊れていて未解決シンボル名すら出力前に落ちる**」（6〜9回目）。`LD_BIND_NOW=1` で
  回避中だが、`Kernel/Core/elf` / interp 受け渡し / `Syscall_LinuxCompat` の
  `mmap`/`mprotect`(GOT の RELRO 化含む) 周辺として別途要調査。

**対処（このコミットで適用済み・イメージ再ビルド済み）**:
1. `Vendor/LinuxRuntime/packages.seed.txt` に `libegl1` `libegl-mesa0` を追加。
2. `make -C Vendor/LinuxRuntime resolve fetch stage xorgdata` 実行 → `packages.lock` 173→175 pkg、
   `closure.txt` 177→179 soname / 未解決 0。両 deb を cache へ取得。
3. `stage-xorg.sh` に §2b「EGL (glvnd + Mesa vendor)」節を追加:
   `libEGL.so.1`（libegl1）/ `libEGL_mesa.so.0`（libegl-mesa0）/
   `/usr/share/glvnd/egl_vendor.d/50_mesa.json` を明示配置。
4. `make image_livecd` 実行済み → `Image/ImplusOS-x86_64-LiveCD.iso` に反映確認済み
   （`libEGL*.so*` + `egl_vendor.d/50_mesa.json` を ISO 内で確認）。
   `libEGL_mesa.so.0` の NEEDED 閉包（`libgallium` / `libgbm` / `libwayland-*` / `libxcb-*` …）は
   ステージ内で全解決（0 missing）。
5. → `make run_uefi_cdrom` で 11回目ブート待ち。
6. その先の想定壁（TODO §M6, §6）: dril のサーフェスレス EGL が `/dev/dri/renderD128` に
   一部 ioctl を要求する可能性 / llvmpipe JIT の `mmap(PROT_EXEC)` / `libLLVM.so.19.1`
   （~120MB）の eager リロケーション時間 / glvnd の EGL vendor 探索パス。

**残置している診断/暫定変更**:
- `glibc_envp`: `LD_PRELOAD=libgbm.so.1`（恒久修正）、`LD_BIND_NOW=1`（libglx.so の壊れた遅延 fixup 経路を回避。恒久化するか、遅延 fixup 側のカーネル/ld.so バグを別途調べるか要判断）、`LD_WARN=1`（診断、無害）。
- `Userland/Application/Doom/Start.c`: `DOOM_DIAGNOSTIC_XORG_ONLY 0`（Doom spawn 復活済み）。`XORG_SETTLE_MS 20000`（X が画面に出たら 4000 に戻す）。

既知の非致命警告: `/usr/lib/xorg/protocol.txt` は deb に同梱されていない。X コアフォントは
`fonts.dir` が deb に無く（Debian は postinst の `update-fonts-dir` で生成、ホストに `mkfontdir` 無し）
`FontPath` は `built-ins` にフォールバックする — `fixed`/`cursor` は組み込みで賄われ、Doom は X フォントを使わない。

次に想定される壁: `modesetting` の `/dev/dri/card0` ハンドシェイク（`drmGetVersion`/`GETRESOURCES`/`GETCONNECTOR` の 2 パス）、`xkbcomp` の fork/exec、Mesa llvmpipe の実行可能 `mmap`。

### M6（旧・想定リスト）

`/dev/shm/Xorg.0.log` を QEMU 経由で回収し、以下を潰す:
- ミッシングモジュール（`Failed to load module`）→ `stage-xorg.sh` に追加。
- `xkbcomp` 失敗 → `XKB_CONFIG_ROOT` / `xkbcomp` パス。`-xkbdir` 明示も可。
- フォントパス（`could not open default font 'fixed'`）→ `xfonts-base` の `fonts.dir` 同梱確認、
  `-fp /usr/share/fonts/X11/misc` 明示。
- GLX 不成立（`GLX: Initialized DRISWRAST` が出ない）→ `+iglx` は指定済み。
  `libglx.so` ロード確認、`Section "Module" Load "glx"`。indirect GLX で
  `linuxxdoom` の `glXChooseVisual` が通ればよい（直接不可でも可）。
- `Atomic` / `PageFlip` は `xorg.conf` で無効化済み（legacy `SETCRTC`+`DIRTYFB` 経路）。

---

## 4. 検証手順

**ホスト素振り（P1 相当・カーネル不要）**:
```
S=Build/x86_64/LinuxRuntime/stage
$S/lib64/ld-linux-x86-64.so.2 --library-path \
  $S/usr/lib/x86_64-linux-gnu $S/usr/bin/Xorg -version
$S/lib64/ld-linux-x86-64.so.2 --library-path \
  $S/usr/lib/x86_64-linux-gnu Userland/Application/Doom/Resource/linuxxdoom-x86_64 -?
```

**QEMU（P2〜P4）**: `make image_livecd && make run_uefi_cdrom`。COM1 シリアルに
`Xorg.0.log` を tee するか `/dev/shm/Xorg.0.log` を Doom 終了後にシリアルダンプ。
チェックポイント: `card0 opened` → `GETRESOURCES` → `SETCRTC` → `X: display :0 ready`
→ `linuxxdoom: XOpenDisplay ok` → `glewInit ok` → 画面。

---

## 5. データファイル（バイナリ／データ追加のみ・ソース不要）

| ファイル | 置き場所 | 状態 |
|---|---|---|
| `doom1.wad` | `Userland/Application/Doom/Resource/doom1.wad` | **配線済み**。`Resource/fetch-wad.sh` が DOOM シェアウェア IWAD v1.9（size 4196020 / md5 `f0cefca49926d00903cf57551d901abe`、`IWAD` マジック検証、自由再配布可）を取得。Doom の `Makefile` の `all:` が `wad` ターゲット経由で初回のみ自動取得（オフライン時は警告して続行）。`.gitignore` 済み（`Vendor/LinuxRuntime/cache` と同じ非コミット運用）。`DOOMWADDIR=/Userland/Doom/Resource`（envp）＋ 子 cwd がこのディレクトリを指す。 |
| `soundfont.sf2` | `Userland/Application/Doom/Resource/soundfont.sf2` | 任意。FluidSynth 音楽用。無くても `i_sound.c` は継続（致命的でない）。`SOUNDFONT` envp は設定済み。 |

**ホスト素振り結果（doom1.wad 配置後）**: `linuxxdoom` は
`W_Init`（`adding ./doom1.wad`）→ `R_Init`（textures/flats/sprites/colormaps）→
`P_Init` → `I_Init` → sfx ロード → `D_CheckNetGame` → `S_Init` → `HU_Init` →
`ST_Init` まで完走し、`I_InitGraphics` の `XOpenDisplay` で
**"Couldn't connect to display!"**（＝ホストに X が無いため）。
エンジン初期化は全て通り、残るは X サーバ（P2〜）だけであることを確認。

---

## 6. リスクと代替

- **llvmpipe JIT が W^X で詰まる** → `GALLIUM_DRIVER=softpipe`（JIT 不要）に envp で退避。激遅。
- **Xorg が logind/seat を要求** → `xserver-xorg-legacy` 同梱 + `-keeptty -novtswitch`、
  単一プロセス＝root 相当で回避。だめなら `Xorg.wrap` を介さず直接起動（済）。
- **`/tmp` FS が重い** → AF_UNIX 抽象アドレス対応（M1 代替）で socket 実体を不要に。
- **modesetting が DRI3/present を要求** → `xorg.conf` で `PageFlip"false"` `Atomic"false"`
  `AccelMethod"none"` `ShadowFB"true"`、GLX は `+iglx`（indirect）。それでも不可なら
  Xorg 側は `fbdev` DDX ＋ カーネル `/dev/fb0`（`FBIOGET_VSCREENINFO`/`mmap`）に切替
  （M3 を fbdev ioctl セットに縮小できるが GLX 経路は別途 `kms_swrast` 用に card0 が要る）。

---

## M8. GLX を外したら X は ScreenInit を完走 — 次は XKB / xkbcomp

**13回目ブート**（`XORG_ARGS` に `-extension GLX`、`+iglx` 撤去）:

DRM/KMS shim が**実動作**した。GLX 以外は初めて全部通っている:

```
(II) Initializing extension DRI2
(II) modeset(0): Damage tracking initialized
(II) modeset(0): Setting screen physical size to 338 x 211
```

＝ `ScreenInit` 完走。M1〜M4（tmpfs / `/dev/dri/card0` char device / DRM ioctl 2 パス /
dumb BO mmap）と ShadowFB + Damage が実機で成立したことの証明。

新しい fatal:
```
(EE) XKB: Could not invoke xkbcomp
(EE) XKB: Couldn't compile keymap
(EE) XKB: Failed to load keymap. Loading default keymap instead.
(EE) XKB: Could not invoke xkbcomp        ← 2回目（fallback RMLVO）も同じ
Keyboard initialization failed. ...
(EE) Fatal server error: Failed to activate virtual core keyboard: 2
```

X は仮想コアキーボードを**必ず**作り、その keymap は `xkbcomp` を実行して
`.xkm` を得る以外に方法が無い（`XkbCompileKeymap` → `XkbDDXLoadKeymapByNames` →
`RunXkbComp`）。`-extension GLX` のように回避できる場所ではない。
`Could not invoke xkbcomp`（"not enough memory" 無し）は **`Popen()` が NULL を返した**時のみ
出るメッセージで、`Popen()` は `pipe()` → `fork()` → 子で `dup2()` →
`execl("/bin/sh", "sh", "-c", cmd)`。

### 調査で判明した事実

| 項目 | 状態 |
|---|---|
| `fork` / `execve` / `pipe` / `pipe2` / `dup2` / `wait4` | Linux 互換層に実装済み |
| `setitimer` / `getitimer` | 0 を返す（Popen の SIGALRM 無効化を塞がない） |
| **`/bin/sh`** | **イメージに存在しない**（`/bin` 自体が一度もステージされていなかった） |
| BusyBox | `Userland/Application/BusyBox/Resource/busybox` に **musl 静的リンク**の Linux ELF。ld.so も `.so` 閉包も不要 |
| `xkbcomp` | `/usr/bin/xkbcomp` にステージ済み。NEEDED（libX11 / libxkbfile / libc）も全て閉包内 |
| xkm 出力先 | Xorg に `/var/lib/xkb/` がコンパイル時固定。`OutputDirectory()` は `access(dir, W_OK\|X_OK)` で採否を決める |
| **`/var`** | **イメージに存在せず、どの FS にもマウントされていない** |
| **`syscall_access()`** | **ディレクトリへの `W_OK` を無条件で `-13 (EACCES)` にしていた**（"directories are read-only here"）。tmpfs 上の `/tmp` `/run` すら書込不可と報告していた |

### 対処（このコミット。`make kernel` + `make image_livecd` 通過・ISO 反映確認済み）

| ファイル | 変更 |
|---|---|
| `Kernel/Core/vfs/TmpFS.c` | `TMPFS_PREFIXES` に `/var` を追加。ディスクレス Linux と同じく read-only ISO 上で書込可能な `/var` を用意（ディレクトリはフラット名前空間なので `mkdir` 不要） |
| `Kernel/Core/vfs/VFS_Pseudo.c` | 疑似 FS テーブルに `{ "/var", tmpfs }` を追加 |
| `Kernel/Core/vfs/VFS.c` / `.h` | `vfs_dir_is_writable(path)` を新設。`vfs_opendir()` と同じ候補順で解決し、そのドライバが `creat` を持つかで判定 |
| `Kernel/Compat/Linux/Syscall_LinuxCompat.c` | `syscall_access()`: ディレクトリの `W_OK` を `vfs_dir_is_writable()` の結果で答える（一律 EACCES を撤廃） |
| `Makefile` | `STAGE_POSIX_BIN` マクロを新設し `image_livecd` / `install_payload` に適用 → `/bin/busybox` と **`/bin/sh`**（BusyBox 実体コピー、ash は `argv[0]=="sh"` で起動） |
| `Kernel/Compat/Linux/Syscall_LinuxCompat.c` | `[lxproc]` トレース: `pipe`/`pipe2`/`clone`/`fork`/`vfork`/`execve`/`dup2`/`dup3`/`setitimer`/`wait4` が**失敗した時だけ**シリアルに syscall 番号と戻り値を出す。`Popen()` が NULL を返す原因を推測でなく特定するため（成功時は無音） |

ISO 検証済み: `/bin/sh` `/bin/busybox`（各 1131168 B）、`/usr/bin/xkbcomp`、
`/usr/share/X11/xkb/{rules,keycodes,types,compat,symbols,geometry}`。

### 次ブートで見るもの

1. `XKB: Could not invoke xkbcomp` が消え、`(II) modeset(0)` の後に X が
   `/tmp/.X11-unix/X0` で accept を回し始める → Doom が `XOpenDisplay(":0")` に成功するか。
2. まだ出る場合、直前に `[lxproc] syscall <nr> failed -> <rc>` が必ず出る。
   nr: 22=pipe / 33=dup2 / 56=clone / 57=fork / 58=vfork / 59=execve /
   61=wait4 / 292=dup3 / 293=pipe2 / 38=setitimer。これで原因が一意に決まる。
3. Doom は GLX 無効なので `glXChooseVisual` 相当で失敗する見込み。X が画面を出す所までを
   まず確定させ、その後 M7 の A〜D で GL 経路を選ぶ。

---

## M9. `fork()` が EINVAL — glibc の fork は `clone(stack=0)` で来る

**14回目ブート**（`/bin/sh` + `/var` tmpfs + `access()` 修正 + `[lxproc]` トレース投入）:

```
(II) modeset(0): Setting screen physical size to 338 x 211
[lxproc] syscall 0x0000000000000038 failed -> 0xFFFFFFFFFFFFFFEA
(EE) XKB: Could not invoke xkbcomp
```

トレースが一発で特定した: **syscall 0x38 = 56 = `clone`、戻り値 -22 = `EINVAL`**。
（`/bin/sh` も `/var` も `access()` も、ここまで到達すらしていなかった。）

**原因**: `linux_clone()` は先頭で `if (stack == 0u) return LINUX_EINVAL;` としており、
**pthread_create 経路しか実装していなかった**。ところが glibc の `fork()` は
`SYS_fork` ではなく `clone` で来る:

```c
/* glibc sysdeps/unix/sysv/linux/arch-fork.h */
const int flags = CLONE_CHILD_SETTID | CLONE_CHILD_CLEARTID | SIGCHLD;
INLINE_SYSCALL_CALL (clone, flags, 0 /* stack */, NULL, ctid, 0);
```

`CLONE_VM` 無し・`stack == 0`（子は親のスタックをそのまま使う）。よって
**外来 Linux バイナリからの `fork()` は例外なく EINVAL** だった。
`Popen()` = `pipe()` + `fork()` + `execl("/bin/sh", ...)` なので、X は
「`XKB: Could not invoke xkbcomp`」という情報量ゼロのメッセージだけ残して落ちていた。

**対処（このコミット。`make kernel` / `make image_livecd` 通過）**:

| ファイル | 変更 |
|---|---|
| `Kernel/Compat/Linux/Syscall_LinuxCompat.c` | `LINUX_CLONE_VM` を定義。`linux_clone()` の先頭で `(flags & (CLONE_VM\|CLONE_THREAD)) == 0` なら **fork 経路**として `process_fork()` へ。`CLONE_PARENT_SETTID` は親のアドレス空間なので実施。`stack == 0 → EINVAL` はスレッド経路専用に降格 |
| 同上 | `linux_clone()` に `should_switch` 出力引数を追加し、`case LINUX_SYS_CLONE` が fork 成功時に `request_switch = 1`（`SYS_fork` 経路と同じ挙動。ファイル内の `linux_epoll_wait` 等と同じパターン） |

`process_fork()` 自体は完全実装だった（`process_clone_address_space()` でアドレス空間を
クローンし、子のカーネルスタック上で `child_kstack[SYSCALL_FRAME_RAX] = 0` を立てて
子が 0 を返すようにしている）。繋がっていなかっただけ。

**未実装として明示した点**: `CLONE_CHILD_SETTID` / `CLONE_CHILD_CLEARTID` は
`child_tid` を**子のアドレス空間**へ書くもので、`process_fork()` から戻った時点では
まだ親として動いているため、ここで書くと親自身の TCB を壊す。glibc はこれを
`THREAD_SELF->tid` の更新にしか使っておらず、子は exec 前にそれを参照しない
（Linux でも `execve` が両方クリアする）。正しく実装するにはクロスアドレス空間書き込みが要る。

### 次に予測される壁: fork 後の fd 継承（未着手）

`Kernel/Core/syscall/Syscall_File.c` の fd テーブルは
**`g_files[FILE_MAX_FD]` というシステム全体で 1 本のグローバル配列**で、各スロットが
`owner_pid` を 1 つだけ持つ（`fd_is_owned_by_current_process()`）。
つまり **fd 番号はシステム全体で一意**であり、`process_fork()` は fd テーブルを
複製していない。POSIX の fork は「子が同じ fd 番号で同じ open file description を見る」
ことを要求するので、現状では:

- `Popen()` の子が `dup2(pdes[0], 0)` → 子は `pdes[0]` を所有していないので `EBADF`
- 続く `execl` 後、`xkbcomp` の stdin がパイプにならない

`[lxproc]` トレースは `dup2`(33) / `dup3`(292) / `execve`(59) も拾うので、
**次のブートでこの予測が当たっているかがそのまま出る**。

対処の選択肢（着手前）:

| 案 | 内容 | 規模 |
|---|---|---|
| **A（推奨）** | `kernel_file_t` に追加所有者ビットマップ（`uint64_t extra_owners[4]`、256 プロセス分・全体で 8 KB）を持たせ、fork 時に親の全 fd に子のビットを立てる。`fd_is_owned_by_current_process()` に OR 条件を1つ足すだけで、**ビットマップが空＝現状の全 fd は挙動が一切変わらない**（純粋な加算的変更）。close / プロセス終了は「自分のビットを落とし、所有者が居なくなった時だけ実体を解放」。fd 番号が同一スロットなのでオフセット共有も POSIX 通り | 中（`Syscall_File.c` の `owner_pid` 参照 27 箇所のうち close 系数箇所 + 継承関数） |
| B | プロセス毎 fd テーブル（本来の POSIX 形）。fd 番号のグローバル一意性という現在の制約自体が消える | 大（コア再設計） |
| C | 回避不可 — X は `xkbcomp` を `/bin/sh` 経由で起動する以外に keymap を得る手段が無く、`-extension GLX` のようには外せない |

---

## M10. X サーバ起動成功 — 残るは DISPLAY と GLX

**15回目ブート**（`linux_clone()` の fork 経路投入後）。**方法A の非 GL 部分が全部通った**:

- `[lxproc]` トレースが**完全に沈黙** → `clone`(fork) / `execve("/bin/sh")` / `dup2` すべて成功
- **`XKB: Could not invoke xkbcomp` が消滅** → `Popen()` → `/bin/sh` → `xkbcomp` が実行され、
  keymap がコンパイルされ、`/var` tmpfs に `.xkm` が書けた
- **X が fatal を出さずに起動状態を維持**（`Setting screen physical size to 338 x 211` 以降エラー無し）
- Doom が `V_Init` → `W_Init`(doom1.wad) → `R_Init` → `P_Init` → `I_Init` → sfx 108 個 →
  `D_CheckNetGame` → `S_Init` → `HU_Init` → `ST_Init` まで完走

つまり **M1〜M5 ＋ Xtrans(AF_UNIX) ＋ fork/exec/pipe ＋ XKB が実機で成立**した。

**予測が外れた点（記録）**: M9 末尾で「次は fork 後の fd 継承が壁」と予測したが**外れ**。
`[lxproc]` は `dup2`/`dup3`/`execve` の失敗も拾う設定だったのに一度も発火せず、
xkbcomp は実際にパイプ経由で keymap を受け取れている。グローバル fd テーブルは
この経路では問題にならなかった（`syscall_pipe_read`/`write` が `g_pipes` を
`open_index` で引くため所有者チェックを通らない等が効いている可能性）。
**案A の fd 所有者ビットマップは着手しない。** 実測で不要と判明した。

### 残る失敗

```
ST_Init: Init status bar.
Couldn't connect to display!
Reading screen...
[OS] [PF] CR2: 0x968  (= I_Error -> I_ShutdownGraphics -> XCloseDisplay(NULL))
```

**原因: `DISPLAY` が環境変数に一度も設定されていなかった。**
`Userland/Application/Doom/Start.c` のコメントは「DISPLAY 未設定なら Xlib は :0 へ
自動接続する」としていたが、これは**誤り**。`XOpenDisplay(NULL)` は `getenv("DISPLAY")`
が NULL だとその NULL をそのまま `xcb_connect()` に渡し、xcb は最初からエラー状態の
接続を返す。フォールバックは無い。

**対処（このコミット。`make kernel` / Doom.ELF / `make image_livecd` 通過）**:

| ファイル | 変更 |
|---|---|
| `Kernel/Core/process/ProcessManager_Create.c` | `glibc_envp` に **`DISPLAY=:0`** を追加（＋なぜ必須かをコメント化） |
| `Userland/Application/Doom/Start.c` | 誤ったコメントを訂正（envp はカーネル側 `glibc_envp` が組み立てる旨も明記） |

### 次ブートで見るもの

1. Doom の `XOpenDisplay(":0")` が成功するか（= Xtrans/AF_UNIX のクライアント側が通るか）。
   成功すれば方法A の X11 接続経路は完全に確定する。
2. その直後、**Doom は GLX を要求する**（`libGL` + `libGLEW` リンク）。今は
   `-extension GLX` で GLX を無効化しているので `glXChooseVisual` 相当で失敗する見込み。
   ここまで来れば残る未解決は **§M7 の GLX 問題 1 点のみ**になる。
3. 画面表示について: X は起動しているがルートウィンドウしか無く、ImplusOS コンソールの
   描画と競合している可能性がある。Doom が接続してウィンドウを作った時に
   ShadowFB→`DIRTYFB`→`display_present()` が実際に出るかを併せて確認する。

---

## M11. `DISPLAY` を設定しても接続不可 — evdev の poll/read 意味論バグ

**16回目ブート**（`DISPLAY=:0` 投入後）: 症状は変わらず `Couldn't connect to display!`。
`DISPLAY=:0` がカーネル ELF と ISO の両方に入っていることは確認済みなので、
環境変数の問題ではない。

**ログの決定的な点**: `(II) modeset(0): Setting screen physical size to 338 x 211`
（= `ScreenInit` 完了）**以降、X が一行も出力していない**。`-verbose 3 -logverbose 3`
を指定しているので、正常なら次に `InitInput()` が
`(II) XINPUT: Adding extended input device "kbd0" ...` /
`(**) evdev: kbd0: Device: "/dev/input/event0"` を必ず出す。それが無い。
X は fatal も出していない（`(EE)` も backtrace も無し）。
→ **X は `Dispatch()` に到達しておらず、`InitCoreDevices()`〜`InitInput()` 付近で
止まっている**。到達していなければ `accept()` を回さないので、クライアントは
ハンドシェイクを完了できず「接続できない」ように見える。

### 見つかった 2 つの実バグ（evdev 経路）

| 箇所 | 症状 |
|---|---|
| `Kernel/Drivers/Module/Evdev_Client.c` `evdev_read()` | リングが空のとき **`0` を返していた**。キャラクタデバイスの `read()` が返す 0 は **EOF** であり、`xf86-input-evdev` はこれを「デバイスが抜かれた」と解釈してデバイスを無効化する。Linux の evdev は空なら `-EAGAIN`（またはブロック）。 |
| `Kernel/Core/vfs/DevFS.c` `devfs_vfs_dev_poll()` | `/dev/input/event*` に対し **無条件で POLLIN を返していた**（コメント自身が「evdev に poll 入口が無いので readable と報告して read に丸投げ」と認めている）。上のバグと組み合わさると「readable と言われる → 読む → EOF」を延々繰り返す。X の `WaitForSomething` と入力スレッドはどちらもこの形の select/poll ループ。 |

### 対処（このコミット。`make kernel` / `make image_livecd` 通過）

| ファイル | 変更 |
|---|---|
| `Evdev_Client.c` / `.h` | `evdev_read()` は空リングで **`-11 (EAGAIN)`** を返す。`evdev_has_events(fd)` を新設（非破壊的な「読めるものがあるか」問い合わせ） |
| `DevFS.c` | `devfs_vfs_dev_poll()` の input 分岐を `evdev_has_events()` の結果で答えるよう変更 |
| `IPC/UnixSocket.c` | `[usock]` トレース: `listen` 成功時にバインドパスを 1 行、`connect` **失敗時のみ** パスを 1 行。X クライアントから見た `Couldn't connect to display!` の裏に ECONNREFUSED があるのかを一意に判定するため（通常運用では計 1〜2 行） |

### 次ブートで見るもの

1. `[usock] listen '/tmp/.X11-unix/X0'` が出るか（＝ X が実際に bind/listen したか）。
2. `(II) XINPUT: Adding extended input device "kbd0"` 系のログが出て、X が `InitInput()` を
   抜けるか。抜ければ `Dispatch()` に入り accept が回る。
3. Doom 側で `[usock] connect FAILED '...'` が出れば、パス不一致（抽象名前空間 vs
   ファイルシステムパス）が原因と確定する。出なければ接続自体は成立しており、
   失敗はハンドシェイク以降。
4. 接続が通れば、次は **§M7 の GLX 問題 1 点のみ**（Doom は `libGL`/`libGLEW` を
   リンクしており、現在 `-extension GLX` で GLX を無効化しているため
   `glXChooseVisual` 相当で失敗する見込み）。

---

## M12. 【M10 の記述を訂正】XKB は成功していなかった — `fork()` が 281 秒かけて失敗

**17回目ブート**（evdev poll/read 修正 + `[usock]` トレース投入）。
**M10 で「XKB が成功した」と書いたのは誤りだった。** 15/16 回目のログは、
失敗が現れる前に打ち切られていただけ。今回タイムスタンプ付きで最後まで出たことで判明:

```
[   100.156] (II) modeset(0): Setting screen physical size to 338 x 211
[lxproc] syscall 0x0000000000000038 failed -> 0xFFFFFFFFFFFFFFF5     ← clone, -11 EAGAIN
[   381.044] (EE) XKB: Could not invoke xkbcomp
[lxproc] syscall 0x0000000000000038 failed -> 0xFFFFFFFFFFFFFFF5
[   658.428] (EE) XKB: Could not invoke xkbcomp
...
(EE) Fatal server error: Failed to activate virtual core keyboard: 2
```

**`fork()` 1 回に約 281 秒かかった末に失敗している**（t=100→381、t=381→658）。
`linux_clone()` の fork 経路は繋がった（M9）が、`process_fork()` が
`process_clone_address_space()` で Xorg の巨大なアドレス空間を
**eager に全ページコピー**しようとしている。`KERNEL_COW_FORK` は既定 0
（config.h の注記: 2026-08-29 に 1 にしたらブート不安定になり同日 revert、
専用の QEMU ブート回帰パスが必要）。

Xorg がマップする実体の規模: `libLLVM.so.19.1` 129 MB / `libgallium` 42 MB /
`libz3` 27 MB ほか。ただし **281 秒がコピー量由来なのか、ページ単位の
病的な遅さ由来なのかは外から判別できない**（`copy_present_page()` 自体は
確保＋4 KiB memcpy＋マップで妥当に見え、PMM も回転ヒント＋ワード走査で高速）。
→ **推測せず計測する。**

### あわせて見つかった実バグ: AF_UNIX 抽象アドレスが全て "" に潰れる

`[usock]` トレースが示したもの:

```
[usock] listen ''                       ← Xtrans の抽象ソケット（名前が消えている）
[usock] listen '/tmp/.X11-unix/X0'      ← Xtrans のファイルシステムソケット
[usock] connect FAILED '/run/dbus/system_bus_socket'   ← dbus 不在。無害
```

Doom 側に `connect FAILED '/tmp/.X11-unix/X0'` は**出ていない**。
原因は `linux_copy_sockaddr_un()` が `strlen(addr.sun_path)` を使っていたこと。
Linux の抽象アドレスは `sun_path[0] == '\0'` で名前が `addr_len` まで続くため、
**すべての抽象アドレスが空文字列に潰れて互いにエイリアスする**。
libxcb は抽象ソケットを先に試すので、Doom の connect は「空パスで listen している
最初のソケット」にマッチして**成功してしまい**、応答の来ないハンドシェイクで
止まっていた（だから `connect FAILED` が出なかった）。

### 対処（このコミット。`make kernel` / `make image_livecd` 通過）

| ファイル | 変更 |
|---|---|
| `Kernel/Compat/Linux/Syscall_LinuxCompat.c` | `linux_copy_sockaddr_un()` に `addr_len` を渡し、抽象アドレスを Linux 自身の表示規約（`ss` / `/proc/net/unix`）と同じ **`@<name>`** に符号化。無名アドレス（autobind）は `EINVAL` |
| `Kernel/IPC/UnixSocket.c` | `unix_socket_connect()` は**空名を一切マッチさせない**（bind 失敗のまま listen したソケットが catch-all になるのを防ぐ） |
| `Kernel/Core/process/ProcessManager_Create.c` | `[fork]` 計測: `process_clone_address_space()` の **rc / 所要 ms / 空き物理メモリ** を fork ごとに 1 行 |

### 次ブートで確定すること

`[fork] clone_address_space rc=<n> elapsed_ms=<n> free_bytes=<n>` が出る。

- `elapsed_ms` が大きく `free_bytes` が枯渇 → **コピー量が原因**。対処は
  (a) `KERNEL_COW_FORK=1` を検証する / (b) 読み取り専用ページは共有し書込可能ページのみ
  コピーする（PTE の OS 予約ビットで「teardown 時に解放しない」印を付ける。CoW フォルト
  経路も refcount も不要だが、親が先に終了した場合の解放ハザードが残る）/ (c) ゲスト RAM を増やす。
- `elapsed_ms` が大きいのに `free_bytes` に余裕あり → **ページ単位の病的遅さ**。
  `copy_present_page` / `paging_map_user_page` 側を最適化すれば eager のままで足りる。

どちらかが確定してから着手する（コア MM の変更なので、当てずっぽうでは触らない）。

---

## M13. 真因は mmap の eager 全コピー — ファイルマッピングを遅延ページ化

**18回目ブート**の計測が決着をつけた:

```
[fork] clone_address_space rc=-1 elapsed_ms=0x668BC (=419,516ms) free_bytes=0x5A5B1D0 (=94MB)
```

**約 420 秒かけて物理メモリを食い尽くし（残り 94 MB / ゲスト 8 GiB）、ページ確保失敗で -1。**
＝ ページ単位の病的な遅さではなく **コピー量**が原因。

### 真因

`linux_mmap()` のファイルバック経路は**完全な eager スナップショットコピー**だった
（先頭で `(void)prot;` と prot を捨てている）。glibc の `ld.so` は各共有オブジェクトを

1. イメージ全体をまたぐマッピング 1 本
2. その上に各 PT_LOAD セグメントを `MAP_FIXED` で重ねる

という手順でロードする。よって 1 ライブラリにつきほぼ 2 重に物理ページを確保し、
しかも `paging_map_user_range_alloc()` は重ね書き時に旧ページを解放しない。
`libLLVM.so.19.1` 129 MB / `libgallium` 42 MB / `libz3` 27 MB を含む約 180 個の `.so` で
これが積み上がり、**Xorg の常駐が数 GB** になっていた（実 Linux の Xorg RSS は数十 MB）。
fork が遅いのではなく、コピーすべき対象が本来の数十倍あった。

**方針はユーザ判断で「mmap を遅延ページ化」を選択。**

### 実装（このコミット。`make kernel` / `make image_livecd` 通過）

| ファイル | 変更 |
|---|---|
| `Kernel/Core/memory/FileMap.c` / `.h`（新規） | 遅延ファイルマッピングの登録表（システム全体で 256 件）。`filemap_register` / `filemap_handle_fault` / `filemap_unregister_range` / `filemap_release_pid` / `filemap_clone_for_fork` |
| `Kernel/Core/syscall/Syscall_File.c` / `.h` | **オープンファイル記述への参照** API: `syscall_file_mmap_acquire` / `_reacquire` / `_read` / `_release`。POSIX 通り、マッピングは fd が閉じられても生き残る（`ld.so` はマップ直後に fd を閉じる）。既存の `kernel_open_file_t.refcount` をそのまま利用 |
| `Kernel/Compat/Linux/Syscall_LinuxCompat.c` | ファイルバック mmap が `LINUX_MMAP_LAZY_MIN_BYTES`（1 MiB）以上なら、物理ページを確保せず VA を予約して `filemap_register`。閾値未満は実績のある eager 経路のまま（表の件数も抑えられる）。表が満杯／VA 不足なら eager にフォールバック。`munmap` で `filemap_unregister_range` |
| `Kernel/Arch/x86_64/cpu/IDT_Main.c` | `#PF` ハンドラで `paging_handle_swap_fault()` の**前**に `filemap_handle_fault()` を呼ぶ。swap ハンドラは未マップのユーザページすべてにゼロページを与える catch-all なので、後だと共有オブジェクトの .text が白紙で埋まる |
| `Kernel/Core/process/ProcessManager_Create.c` | `execve` / プロセス終了で `filemap_release_pid`、`fork` で `filemap_clone_for_fork` |

**フォルト文脈での I/O の妥当性**（実装前に検証）: `#PF` は割り込みゲート（`0x8E`）＝
IF クリアで入る。ATA / AHCI プロトコルには `sleep` / `wait_queue` / `schedule` 呼び出しが
一つも無く**純ポーリング**なので、割り込み禁止のまま読める。読み先はカーネルバッファ
（ユーザページのフォルト再入無し）。既存の `#PF` 再入ガードがバグを可視化する。

**重なりの解決**: 記録に `seq`（登録順）を持たせ、フォルト時は**最後に登録された記録が勝つ**。
`ld.so` は全 span マッピングの上にセグメントを別のファイルオフセットで重ねるため、
先勝ちにすると各セグメントに span 側のバイト列が入ってしまう。`fork` の複製も
親の `seq` 順に行い、子でも同じ解決順序を保つ。

### 次ブートで見るもの

1. `[fork] clone_address_space rc=0 elapsed_ms=<小さい値>` になるか。
   なれば `xkbcomp` が走り、X が `Dispatch()` に入って accept を回し始める。
2. Doom の `XOpenDisplay(":0")` が成功するか（抽象アドレス修正 M12 と併せて）。
3. 成功すれば残る未解決は **§M7 の GLX 1 点**（Doom は `libGL`/`libGLEW` をリンクしており、
   現在 `-extension GLX` で無効化中）。
4. 新経路の失敗が出るなら、共有オブジェクトの .text が白紙化して即クラッシュする形で
   現れるはず（重なり解決か、ページ内のファイルオフセット計算の誤り）。

---

## M13a. 遅延 mmap の初回投入で Xorg が即クラッシュ — `.bss` の乗っ取り

**19回目ブート**: Xorg が起動直後に落ちる（ログはこの #PF 一つだけ）:

```
[OS] [PF] CR2: 0x28  RIP: 0x0000010000191A10  RBP/rdi/rsi: 0x0000010000339000
[PFDBG] name=Xorg  kf_rax=0x0000004078004C80 (ld.so 内)
```

RIP が `0x10000000000` 台＝**USER_MMAP アリーナ内**。つまり遅延マッピングは
フォルトインして**実行までは到達している**。落ちているのは `[0x28]` の読み＝
本来ポインタが入っているはずの場所が 0 だった、という形。

### 原因（4件。主因は1番目）

1. **`.bss` の乗っ取り（主因）**。`ld.so` は共有オブジェクトを
   ①イメージ全体をファイルマップ → ②各セグメントを `MAP_FIXED` → ③`.bss` を
   **`MAP_ANONYMOUS|MAP_FIXED`** で重ねる、という順でロードする。
   遅延化でライブラリが `process_user_reserve()` の **USER_MMAP アリーナ**へ移った結果、
   ③は `linux_mmap` の「アリーナ内の匿名 MAP_FIXED は何もコミットせず demand-zero」枝に入る。
   ところが**①の span 記録がそのフォルトを横取りしてファイル内容を書き込んでいた**。
   `.bss` がゼロでなくファイルのバイト列になれば、ゼロ初期化前提のポインタがガベージになり
   `[0x28]` の NULL 参照になる。
2. **`MAP_SHARED` の書き戻し登録を飛ばしていた**。遅延経路が早期 return するため
   `linux_mshared_register()` に到達しない。
3. **TLB を無効化していなかった**（demand-zero 経路は `invlpg` している）。
4. **fork のコピー範囲がアリーナを含まない**。従来 `0x1000..USER_STACK_BASE` のみで、
   アリーナ（`0x10000000000`〜）もユーザスタックも範囲外。ライブラリがアリーナへ
   移ったので、子はライブラリの可変データを持たないまま動き出す。

### 対処（このコミット。`make kernel` / `make image_livecd` 通過）

| ファイル | 変更 |
|---|---|
| `Core/memory/FileMap.c` / `.h` | `filemap_register_zero()` を新設。`file_handle == -1` の**ゼロ穴**記録を、既存のファイルマッピングと重なる時だけ登録する。フォルト時は `seq` で最後の登録が勝つので、ゼロ穴が span を上書きし demand-zero へ落ちる。`filemap_release_pid` は `-1` を「記録終端」と誤認しないよう `found` フラグで判定、`filemap_clone_for_fork` はゼロ穴を参照取得なしで複製 |
| `Compat/Linux/Syscall_LinuxCompat.c` | アリーナ内の匿名 `MAP_FIXED` で `filemap_register_zero()` を呼ぶ。遅延経路から **`MAP_SHARED` を除外**（書き戻し登録を壊さない） |
| `Core/memory/FileMap.c` | ページ投入後に `hal_mmu_invalidate_tlb()` |
| `Core/process/ProcessManager_Create.c` | Linux fork のコピー範囲を `0x1000..USER_STACK_TOP` に広げ、さらに `USER_MMAP_BASE..USER_MMAP_LIMIT` を第2パスとして追加。走査は非在レベルをスキップするので、ほぼ空のアリーナを covering しても安い |

### 次ブートで見るもの

- Xorg が起動を続けられるか。落ちるなら、共有オブジェクトの内容が壊れている形
  （即 #PF / 不正命令）で出るので、重なり解決かページ内オフセット計算を疑う。
- 通れば `[fork] clone_address_space rc=0 elapsed_ms=<小>` が出て `xkbcomp` が走り、
  X が `Dispatch()` に入る。

---

## M13b. 遅延 mmap 成功 — 先読みを追加

**20回目ブート**: `.bss` 乗っ取りの修正（M13a）が効き、**遅延ファイルマッピングが実働**した。

- Xorg が `libglx` / `modesetting` / `evdev` / `libshadow` を全てロードし、
  `/dev/dri/card0` オープン → コネクタ検出 → ShadowFB → 全拡張初期化 →
  `Damage tracking initialized` → `Setting screen physical size` まで完走
- Doom もエンジン初期化を完走（`Device: 0x4100da46f0`、RIP `0x1000004695D`）
- **両プロセスとも全ライブラリを USER_MMAP アリーナ（`0x10000000000` 台）から実行**
  ＝ eager 全コピーは完全に廃止できた

`[usock] listen '@/tmp/.X11-unix/X0'` も出ており、抽象アドレスの符号化（M12）も正しい。

### 未確定

ログが **`[fork] clone_address_space` 行の前で終わっている**ため、
fork が現実的な時間で成功するようになったかは**まだ不明**。X は相変わらず
`Setting screen physical size` の後で沈黙しており（`InitCoreDevices` → XKB → fork 待ち）、
Doom は X が accept していないので `Couldn't connect to display!` のまま。

### このコミットの追加: フォルト先読み

需要ページングは 1 フォルト = ISO への 1 コマンドになる。`ISO9660_read_at()` は
連続セクタを 1 回の `disk_read()` にまとめるので、**まとめて読むほど安い**。

`filemap_handle_fault()` を、フォルトページから最大 16 ページ（64 KiB）を
`pmm_alloc_pages()` の連続フレームへ 1 回で読み込み、まとめてマップするよう変更。
先読み窓は次で打ち切る:

- その記録の終端
- **自分より新しい `seq` の記録の開始位置**（セグメントや `.bss` ゼロ穴を跨がない）
- 既にマップ済みのページ

連続フレームが取れなければ 1 ページに縮退。途中でマップに失敗した場合は、
使わなかった残りのフレームだけを返却する。

### 次ブートで見るもの

**`[fork] clone_address_space rc=? elapsed_ms=? free_bytes=?` が出るまでログを取ること。**
これが今の唯一の未知数。

- `rc=0` かつ `elapsed_ms` が小さい → `xkbcomp` が走り X が `Dispatch()` に入る。
  Doom の `XOpenDisplay(":0")` が通れば残るは §M7 の GLX 1 点。
- まだ大きい／失敗する → 常駐がまだ大きいということ。次の手は
  §M12 の案（COW 有効化 / 読取専用ページ共有）に戻る。

---

## M14. fork の真因 — `paging_copy_present_user_range()` が `PAGE_USER` を見ていなかった

**21回目ブート**でついに `[fork]` の数字が採れた:

```
[fork] clone_address_space rc=-1 elapsed_ms=0x54545 (=345,413ms) free_bytes=0x5A32480 (=94.4MB)
```

前回（eager mmap 時）が **420 秒 / 94.5 MB**。遅延ページ化で Xorg の常駐は激減したはずなのに
**`free_bytes` がほぼ同一**で、やはりメモリを尽きるまでコピーしている。
＝ コピー量が **Xorg のサイズと無関係**。

### 真因

`paging_copy_present_user_range()` は **PML4 / PDPT / PD / PT のどの階層でも
`PAGE_USER` を検査していなかった**（`PAGE_PRESENT` のみ）。
一方 `paging_destroy_process_space()` は最初から `PAGE_USER` を検査している。

プロセスのページテーブルにはカーネル自身のマッピングも入っており、カーネルは
**`PAGING_BOOT_IDENTITY_GB = 64` GB を低位 VA に identity map** している。
これは fork が歩く範囲 `0x1000..0x4800000000`（288 GB）に**丸ごと含まれる**。

→ **fork は毎回カーネルの identity マップを歩き、物理メモリが尽きるまで
カーネルページを子へコピーしていた。** 所要時間が搭載 RAM に比例し、
`free_bytes` がプロセスに関係なく同じ値で終わる観測と完全に一致する。

これは最初から存在していたバグで、M13 の遅延 mmap は**別の実バグ**（Xorg の常駐が
数 GB になる eager 全コピー）を直したものの、fork のブロッカーそのものではなかった。
M13 は無駄ではない（実際に数 GB の浪費を除去した）が、fork を止めていたのはこちら。

### 対処（このコミット。`make kernel` / `make image_livecd` 通過）

| ファイル | 変更 |
|---|---|
| `Kernel/Arch/x86_64/mmu/Paging_Main.c` | `paging_copy_present_user_range()` の 4 階層すべてを `(PAGE_PRESENT \| PAGE_USER)` で絞る。理由をコメント化 |
| 同上 | `paging_cow_clone_user_range()`（`KERNEL_COW_FORK` 既定 0 で未使用だが同じ潜在バグ）の 3 階層にも同じ絞りを適用 |

### 次ブートで見るもの

`[fork] clone_address_space rc=0 elapsed_ms=<小さい値>` になるはず。
そうなれば `/bin/sh` → `xkbcomp` が走って keymap がコンパイルされ、X が
`InitInput()` を抜けて `Dispatch()` に入り、`/tmp/.X11-unix/X0` で accept を回し始める。
Doom の `XOpenDisplay(":0")` が通れば、残る未解決は **§M7 の GLX 1 点**。

---

## M15. fork が 827 ms で成功 — 次は fd 継承と execve のシグナルリセット

**22回目ブート**。M14 の `PAGE_USER` 修正が的中:

```
[fork] clone_address_space rc=0 elapsed_ms=0x33B (=827ms) free_bytes=0x5A89EF0
```

**fork が「345,000 ms かけて失敗」→「827 ms で成功」**。
`/bin/sh` の `execve` も通り、`[PFDBG]` のプロセス名が `sh` になっている。

続けて 2 つの新しい壁が露出した。

### (1) fork 後の子が親の fd を所有していない

```
[lxproc] syscall 0x21 failed -> 0xFFFFFFFFFFFFFFF2
```
`0x21` = 33 = **`dup2`**、戻り値 **-14 = `OS_STATUS_FAULT`**。
`Popen()` の子は `dup2(pdes[0], 0)` でパイプを stdin に付け替えるが、これが失敗するので
`xkbcomp` は keymap を stdin から受け取れず、Xorg は
`Couldn't open compiled keymap file /var/lib/xkb/server-0.xkm` で落ちる。

原因は M9 末尾で予測したとおり: `g_files[]` は**システム全体で 1 本**の配列で
各スロットが `owner_pid` を 1 つだけ持ち、`process_fork()` は fd を複製していなかった。
当時「トレースが沈黙しているので予測は外れ」と判断したが、それは **fork 自体が
一度も成功していなかった**ため。fork が通った今、予測どおりに現れた。

### (2) `execve` がシグナル配置をリセットしていない

`sh` が NULL 参照で落ちた直後から、**RIP `0x40001E1C40`（Xorg のテキスト）**で
同じフォルトが延々と繰り返される。fork した子は Xorg の SIGSEGV ハンドラを
継承しており、`execve("/bin/sh")` 後もそれが残っていたため、自分のものでない
アドレスのハンドラが走って再びフォルト、を無限に繰り返していた。
POSIX では `execve` は捕捉中のシグナルをすべて `SIG_DFL` に戻す。

### 対処（このコミット。`make kernel` / `make image_livecd` 通過）

| ファイル | 変更 |
|---|---|
| `Kernel/Core/syscall/Syscall_File.c` / `.h` | `kernel_file_t` に**追加所有者ビットマップ** `extra_owners[]`（`OS_CONFIG_PROCESS_MAX_COUNT` ビット = 全体で 8 KB）。`fd_is_owned_by_current_process()` は `owner_pid` 一致に加えビットも見る。`release_fd_locked_for(fd, pid)` を新設し、**最後の所有者が離したときだけ**実体を解放（`owner_pid` が抜ける場合はビットマップ中の 1 つへ委譲）。`close(2)` / `close_all_for_pid` / CLOEXEC 掃除をすべてこの経路へ。`syscall_file_fork_inherit(parent, child)` を新設 |
| `Kernel/Core/process/ProcessManager_Create.c` | `process_fork()` から `syscall_file_fork_inherit()` を呼ぶ |
| 同上 | `process_execve()` でユーザハンドラ（`signal_handlers[] > 1`）を `SIG_DFL` に戻し、`signal_flags`/`sa_mask`/`restorer`/`pending_signals`/altstack をクリア。`SIG_IGN` とブロックマスクは POSIX 通り継承 |

ビットマップは**継承されなかった fd では常に空**なので、fork しない既存の全経路は
挙動が一切変わらない（純粋な加算的変更）。fd 番号が同一スロットなので、
親子がオフセットを共有するのも POSIX の規定どおり。

### 次ブートで見るもの

1. `[lxproc] syscall 0x21 failed` が消え、`xkbcomp` が keymap を出力するか。
   `/var/lib/xkb/server-0.xkm` が開ければ X は `InitInput()` を抜けて `Dispatch()` へ。
2. `sh` の NULL フォルトが残る場合、Xorg のハンドラ継承は消えているので
   **1 回だけ**出るはずで、busybox 側の実問題として切り分けられる。
3. `[lxproc] syscall 0x3B failed -> -1`（`execve`）は `sh` が `xkbcomp` を起動できて
   いないことを示す。fd 継承後も残るなら次はこれ。

---

## M16. `execve` の真因 — 自分自身を exec するプロセスの状態 `RUNNING` を拒否していた

**23回目ブート**（fd 継承 + execve シグナルリセット投入後）。前回の 2 件は両方解決:

- `[lxproc] syscall 0x21 failed`（`dup2`）が**消滅** → fd 継承が機能
- Xorg のハンドラ継承によるフォルト嵐が消え、**`sh` の fault が 1 回だけ**に

残る `[lxproc] syscall 0x3B failed -> -1`（`execve`）を追い込んだ。

### 計測で否定できたもの

`[fork]` トレースに空き**物理**ページを追加した結果:

```
[fork] clone_address_space rc=0 elapsed_ms=0x2BB free_heap=0x5A89EF0 free_pages=0x1EC922
```

`free_pages = 0x1EC922` = 2,017,570 ページ = **約 7.7 GB 空き**。物理メモリ枯渇ではない。
`[mem] OOM` も出ない（※そもそも `memory_report_oom()` は引数を捨てるだけで**何も出力していなかった**。
これも直した。「OOM ログが無い＝OOM でない」と判断していた過去の推論は根拠を欠いていた）。

さらに **`[execve-fail]` が 1 行も出ない**。つまりタグを付けた 4 箇所
（`paging_create_process_space` / `thread-reserve` / `initial-stack-map` / ELF ロード）は
どれも通っていない。`process_execve` 内の `return -1` を数え直すと **6 箇所**あり、
ELF ロード後の 2 つが未タグだった。

### 真因

`process_execve()` の最終段:

```c
if (proc->state != PROCESS_STATE_INIT &&
    proc->state != PROCESS_STATE_READY) {
    return -1;
}
```

`ProcessScheduler.c:203` はディスパッチしたプロセスに **`PROCESS_STATE_RUNNING`(=2)** を設定する。
ログ中の全 `[PFDBG]` も `state=0x2`。**自分自身を `execve` するプロセスは定義上 RUNNING** なので、
このチェックが必ず弾いていた。

しかも復帰用 RIP（`kstack[SYSCALL_FRAME_RCX] = image_info.entry`）とプロセス名は
**このチェックより前**に設定される。したがって:

1. 新イメージ（busybox）はロード済み・プロセス名は `sh`
2. しかし `proc->entry` / `user_stack_exchanged` は未設定のまま `-1` で返る
3. プロセスは新エントリへ復帰するが**引き渡されていない古いスタック**で走り出す
4. 最初の argv 参照で NULL 死（`RIP=0x404303`, `rdi=0x4EB1E5` = busybox のテキスト/文字列）

`fork` + 自己 `exec` は外来 Linux バイナリで初めて使われた経路なので、これまで露見しなかった。

### 対処（このコミット。`make kernel` / `make image_livecd` 通過）

| ファイル | 変更 |
|---|---|
| `Kernel/Core/process/ProcessManager_Create.c` | 状態チェックに **`PROCESS_STATE_RUNNING` を許可**。このチェックの目的は「裏で片付けられたプロセス」の検出なので、その状態だけを弾くようにした |
| 同上 | 残っていた `user-stack-init` の `-1` にもタグを付け、復帰不能点以降の失敗は全て `process_exit_current_with_status(127)` でプロセスを終了（半端に exec されたプロセスをユーザモードへ戻さない） |
| `Kernel/MemoryManagement/Memory_Main.c` / `.h` | `memory_report_oom()` が**何も出力していなかった**のを修正（先頭 8 回のみ site / 要求サイズ / 空きページ数を出力）。`memory_free_pages()` を新設し `[fork]` と `[execve-fail]` の各トレースに空き物理ページを表示 |
| `Userland/Application/Doom/Start.c` | Doom の起動を **userland 起動 15 秒後**に（`XORG_SETTLE_MS` 20000 → 15000）※ユーザ指示 |

### 反復環境

この環境で QEMU を直接起動できることを確認した（`qemu-system-x86_64` 10.2.1 + 同梱 `OVMF_CODE_4M.fd`）。
`/dev/kvm` は権限が無く TCG のみなので実時間は遅い（`-smp 16` は TCG で極端に遅いため 2 に落とす）。
ヘッドレス起動スクリプトでシリアルをファイルへ落として反復する。

---

## M17. XKB が落ちる真因は 4 つ — `wait4` / `dup2` / パイプ write / `O_TRUNC`

**24回目ブート**。M16 の `execve` 修正が効き、`[lxproc] syscall 0x3B failed`（execve）は
消滅。X は `ScreenInit` を完走し、`InitCoreDevices` → XKB まで到達した。
そこで `xkbcomp` を 2 回起動し、2 回とも失敗している:

```
[fork] clone_address_space rc=0x0 elapsed_ms=0x386 free_heap=0x5A89EF0 free_pages=0x1EC922
(EE) Couldn't open compiled keymap file /var/lib/xkb/server-0.xkm      ← 1回目
(EE) XKB: Failed to load keymap. Loading default keymap instead.
[fork] clone_address_space rc=0x0 elapsed_ms=0x556 free_heap=0x5A611D0 free_pages=0x1EC6E5
[lxproc] syscall 0x0000000000000021 failed -> 0xFFFFFFFFFFFFFFF3       ← 2回目: dup2 = -13
(EE) Couldn't open compiled keymap file /var/lib/xkb/server-0.xkm
XKB: Failed to compile keymap
(EE) Fatal server error: Failed to activate virtual core keyboard: 2
...
syntax error: line 1 of stdin                    ← xkbcomp が別の stdin を読んでいる
Errors encountered in stdin; not compiled.
```

`fork` も `execve` も通っているのに `.xkm` が出てこない。ログの後半に現れる
`syntax error: line 1 of stdin` は **xkbcomp が実際に走った**証拠なので、
「起動できない」ではなく「起動したものが正しい入力を得られていない／
親がその完了を待っていない」形である。

### 見つかった 4 件（すべて実バグ）

| # | 箇所 | 症状 | 根拠 |
|---|---|---|---|
| 1 | `Syscall_LinuxCompat.c` `linux_wait4()` | **ブロッキング `wait4` が「子はまだ終了していない」を `0` で返していた**。Linux の `0` は `WNOHANG` 専用の戻り値。X の `Pclose()` は `do { pid = waitpid(cur->pid,&pstat,0); } while (pid == -1 && errno == EINTR); return pid == -1 ? -1 : pstat;` なので、`0` はループを 1 周で抜け、**一度も書かれていない `pstat`** をそのまま返す。`XkbDDXCompileKeymapByNames()` は `Pclose(out) == 0` を成功と解釈し、まだ書かれていない `.xkm` を開きにいく → `Couldn't open compiled keymap file`。ログに `Error compiling keymap` も `Could not invoke xkbcomp` も出ていないことが「`Pclose` が 0 を返した」ことの裏付け | 1回目の失敗そのもの |
| 2 | `Syscall_File.c` `syscall_file_dup2()` | 差し替え先 fd の所有者判定が **`owner_pid` の一致のみ**で、fork で継承した fd（`extra_owners` ビット）を弾いて `-13 (EACCES)` を返していた。`Popen()` の子がやる `dup2(pipe, 0)` はまさにこの形。加えて `release_fd_locked()` を無条件に呼ぶため、共有中のスロットの実体を他の所有者ごと落としていた | `[lxproc] 0x21 -> -13` は `OS_STATUS_ACCESS_DENIED` を返すこの 1 箇所しかない |
| 3 | `Syscall_File.c` `syscall_pipe_write()` | パイプが満杯のとき **`0` を返していた**。ブロッキング `write(2)` は非空要求に対して 0 を返さない。glibc の `_IO_new_file_write` は `while (to_do > 0)` で回すので、`0` は `to_do` を減らさず**呼び出し側を永久にスピンさせる**。X は数十 KB の keymap を 4 KiB のパイプへ流し込むので必ずここを通る | コード上明白。SMP で相手が同時に排出していると「たまたま」進むため今まで表面化しなかった |
| 4 | `Syscall_File.c` `syscall_file_open()` | **`O_TRUNC` を受け取って無視していた**。`fopen(path,"w")` が既存ファイルを切り詰めないので、新しい内容より長い旧内容の尾が残る。X は keymap のコンパイルを 2 回試すため、2 回目の `xkbcomp` が書く `/var/lib/xkb/server-0.xkm` に 1 回目の残骸が付く | コード上明白。`vfs_truncate()` は存在するのに `open` から呼ばれていなかった |

### 対処（このコミット。`make kernel` / `make image_livecd` 通過）

| ファイル | 変更 |
|---|---|
| `Kernel/Compat/Linux/Syscall_LinuxCompat.c` | `linux_syscall_restart()` を新設: `rax` に syscall 番号を書き戻し `rcx`（復帰 RIP）を 2 バイト巻き戻して、次にスケジュールされた時に `syscall` 命令を再実行させる（Linux の `ERESTARTSYS` と同じ手口）。`syscall_entry.asm` は全引数レジスタをこのフレームから復元するので、再実行は同じ引数で走る |
| 同上 | `linux_wait4()`: 子が未終了なら 1 スライス眠ってから**同じ呼び出しをやり直す**（`0` を返さない）。`process_waitpid_ex()` の `-1`（該当する子が無い）は `ECHILD` に写像（従来は `-1` = `EPERM` に化けていた） |
| 同上 | `linux_waitid()` も同型（`siginfo_t` を書かずに `0` を返していた）。`request_restart` をディスパッチャの末尾で処理 |
| `Kernel/Core/syscall/Syscall_File.c` | `syscall_file_dup2()`: 差し替え先の所有者判定を `fd_is_owned_by_current_process()` に。スロットの旧所有者集合を保存して新しいバインドへ引き継ぐ（参照カウントはスロット単位なので勘定は正確なまま）。所有していない fd への差し替えは従来どおり拒否 |
| 同上 | `syscall_pipe_write()`: 満杯なら `syscall_pipe_write_wait()` で読み手が空けるのを待つ（`syscall_pipe_read()` と対称）。`O_NONBLOCK` は `EAGAIN`、読み手全滅は `SIGPIPE`/`EPIPE` |
| 同上 | `syscall_file_open()`: `O_TRUNC` を実装（`FILE_O_TRUNC 0x0200`、書き込み可かつ既存サイズ != 0 のときだけ `syscall_file_truncate(fd,0)`）|
| `Kernel/Core/vfs/VFS.c` | `vfs_truncate()` が `fs_driver->truncate` の NULL を見ていなかったのを修正 |

### 補足: 1回目と2回目で症状が違った理由

`syscall_file_open()` / `syscall_file_pipe()` は **fd 3 から**割り当てる（0/1/2 は
コンソール扱いで予約）。したがって `dup2(pipe, 0)` の差し替え先スロット 0 は
**明示的な `dup2` でしか埋まらない**。1回目は空スロットなので `dup2` が通り
（`[lxproc]` も沈黙）、2回目は 1回目の子がまだ生きたままスロット 0 を握っていた
（#1 のせいで親が待っていない）ので `owner_pid` 不一致 → `-13`。
つまり #2 は #1 の二次症状として現れていた。

### 派生していた 2 つの #PF（どちらもこの修正で消えるはず）

- `Xorg`: `CR2=0x0 RIP=0x0 Error=0x15`（present + user + 命令フェッチ）。
  `Failed to activate virtual core keyboard` の後始末で NULL 関数ポインタを
  呼んでいる X 自身の経路。`present` なのは、カーネルが低位 VA に
  identity map を張っており（`PAGING_BOOT_IDENTITY_GB`）アドレス 0 が
  「存在するがユーザからは触れない」ページだから。
- `linuxxdoom`: `CR2=0x968`。X が上がっていない → `XOpenDisplay` 失敗 →
  `I_Error()` → `I_ShutdownGraphics()` が `XCloseDisplay(NULL)`。M10 と同じ。

---

## M18. XKB 突破を実測で確認 — 次は evdev キーボードと X の入力スレッド

**25回目ブート**（M17 の 4 件投入後）。**XKB は完全に通った**:

```
[fork] clone_address_space rc=0x0 elapsed_ms=0x1AE      ← fork 1 回だけ
(II) Using input driver 'evdev' for 'kbd0'
(II) XINPUT: Adding extended input device "mouse0" (type: MOUSE, id 6)
```

- `XKB:` で始まる行が**一行も出ない**（従来は必ず `Could not invoke xkbcomp` か
  `Couldn't open compiled keymap file` が出て fatal だった）
- `[lxproc]` も沈黙（`dup2` の `-13` が消滅）
- fork は 1 回で済んでいる（＝1 回目の `xkbcomp` が成功し、
  「デフォルト keymap で再試行」に落ちていない）
- **X は `InitCoreDevices` を抜け `InitInput()` に入った**。ここまで到達したのは初めて

つまり M17 の 4 件が XKB のブロッカーそのものだった。

### 新たに露出した 2 件

| 症状 | 原因 | 対処 |
|---|---|---|
| `(EE) evdev: kbd0: Unable to query fd: Inappropriate ioctl for device` → `(EE) PreInit returned 2 for "kbd0"`（mouse0 は正常） | `evdev_drv.so` を逆アセンブルして特定: このメッセージは **`libevdev_set_fd()` が負の errno を返した**ときの出力（`6900: call libevdev_set_fd; 6906: lea "Unable to query fd: %s"`）。`libevdev_set_fd()` は `EVIOCGBIT(0)` が **`EV_REP` を立てているデバイスにだけ `EVIOCGREP` を投げる**。つまりキーボードだけ。`evdev_ioctl()` は nr=0x03 を持っておらず `-25 (ENOTTY)` を返していた | `Evdev_Client.c` に `EVIOCGREP`/`EVIOCSREP`（nr 0x03）を実装（Linux 既定の delay 250ms / period 33ms を返す。set は受理）。あわせて `EVIOCGKEY/LED/SW` のバッファを 64→128 バイトに（libevdev は KEY_CNT ビット = 96 バイトを要求するのに 64 しか埋めていなかった）。**`EV_LED` は逆に申告をやめた** — LED の実体が無いのに申告すると、サーバがキーボード LED 制御経路に入りデバイスノードへ `input_event` を書き戻す（この shim に無い機能）ため |
| `[OS] [#GP] user-mode #GP -> terminating pid=5 name=InputThread rip=0x6E672D78756E696C err=0` で **X ごと死ぬ** | `rip` は ASCII（リトルエンディアンで `6C 69 6E 75 78 2D 67 6E` = `"linux-gn"`）＝ X の入力スレッドが**文字列データへ復帰**している。非カノニカルアドレスなので #GP。カーネル側のスレッド周りのバグ（詳細未特定） | **未解決**。X 側は `-dumbSched`（入力をメインループで回し、スレッド化入力とスマートスケジューラを止める公式オプション）で回避。`Start.c` の `XORG_ARGS` に追加。あわせて #GP ハンドラに `cs/rflags/user_rsp/user_ss` とユーザスタック 8 ワードのダンプを追加した |

**26回目ブート**（`EVIOCGREP` + `-dumbSched` 投入後）:

- **`kbd0` が通った**: `(--) evdev: kbd0: Found keys` → `(II) evdev: kbd0: Configuring as keyboard`
  → `(II) XINPUT: Adding extended input device "kbd0" (type: KEYBOARD, id 6)`
- `InputThread` の #GP は消滅（`-dumbSched` が効いた）
- 新しい壁: `kbd0` を追加した直後に
  `[OS] [PF] CR2:0x1000 RIP:0x410087B004`（読み取り、ユーザモード）。
  X のバックトレースは `2: /usr/lib/xorg/modules/input/evdev_drv.so [0x410086d650]`。
  `EvdevInit()` → `EvdevAddKeyClass()` 付近。`[fork]` が出ていないので
  `InitKeyboardDeviceStruct()` の `XkbCompileKeymap` にはまだ達していない

### このコミットで追加した診断: モジュールロードマップ

外来バイナリのクラッシュは生の RIP しか出ないのに、その `.so` を配置したのは
こちらのカーネルである。`[lxmap] <name> base=<addr> len=<n> off=<n>` を
`.so` の file-backed mmap ごとに 1 行出すようにした（`LINUX_MODULE_MAP_TRACE`）。
遅延経路と eager 経路の両方に入れてある — X の入力/映像ドライバは 1 MiB 未満で
必ず eager 側を通るため。これで `RIP → <module>+<offset>` が確定でき、
以後この種のクラッシュを推測でなく逆アセンブルで詰められる。

### 現在の到達点

| 項目 | 状態 |
|---|---|
| tmpfs (`/tmp` `/run` `/var`) / AF_UNIX / Xtrans | 済 |
| `/dev/dri/card0` DRM/KMS shim・ShadowFB・Damage・モード検出 | 済 |
| 遅延ファイルマッピング・fork・execve | 済 |
| **XKB / xkbcomp** | **済（M17）** |
| **evdev mouse0 / kbd0** | **済（M18）** |
| X が `Dispatch()` に入り accept を回す | 未（evdev キーボード初期化の #PF が残る） |
| Doom の `XOpenDisplay(":0")` | 未（X が上がりきらないため） |
| GLX（Doom は `glXCreateContext`/`glXMakeCurrent`/`glXSwapBuffers`/`glewInit` を UND 参照）| 未着手。現在 `-extension GLX` で無効化中。§M7 の A〜D は依然有効だが、`libdril_dri.so` を実際に読むと `__DRI_SWRAST` v5 の関数ポインタは**全部埋まっている**（+0x10〜+0x38 が非 NULL）ので、M7 の「dril が swrast を提供していない」という推定は**誤り**。GLX 再挑戦時はこの前提から始めないこと |

---

## M19. evdev キーボードで X が落ちる — `xkbInfo->kbdProc` が壊れた値になる

**27〜29回目ブート**。`EVIOCGREP` でキーボードが通るようになった直後に、
毎回**まったく同じアドレス**で落ちる（＝決定的、ASLR 無しで再現性 100%）:

```
(II) XINPUT: Adding extended input device "kbd0" (type: KEYBOARD, id 6)
[fork] clone_address_space rc=0 elapsed_ms=0x2F3          ← kbd0 用の xkbcomp。成功
[OS] [PF] CR2:0x1000  RIP:0x410087B004  Access: read, mode: user
[OS] [PF] user stack: 0x400016175B 0x410069E9F0 ... 0x410086A090
```

### モジュールマップで確定させた事実

新しい `[lxmap]` トレース（M18）で全モジュールのロードベースが分かるので、
生アドレスを `<module>+<offset>` に落とせる:

| 生アドレス | 解決結果 |
|---|---|
| `RIP 0x410087B004` | `libmtdev.so.1` base `0x4100877000` + **`0x4004`** = **`.rodata` の先頭+4**（LOAD3 は `R` のみ。実行可能ですらない） |
| user stack[0] `0x400016175B` | `Xorg` base `0x4000000000` + `0x16175B` = **`XkbDDXKeybdCtrlProc+0x4B`**（＝戻りアドレス。呼び出し元） |
| user stack[7] `0x410086A090` | `evdev_drv.so` + `0x5090` = **`EvdevKbdCtrl`**（＝本来呼ばれるべき関数） |

`Xorg` の該当箇所を逆アセンブルすると:

```
161735: mov 0xb8(%rbx),%rax     # dev->key
161741: mov 0x68(%rax),%rax     # ->xkbInfo
16174a: mov 0x38(%rax),%rax     # ->kbdProc
161753: mov %rbp,%rsi
161756: mov %rbx,%rdi
161759: call *%rax              ← ここ。rax = 0x410087B004（libmtdev の .rodata）
16175b: mov %r12d,0x10(%rbp)    ← 戻りアドレス = user stack[0]
```

そして `evdev_drv.so` 側は正しい値を渡している:

```
89dc: lea -0x3953(%rip),%rcx    # 5090 = EvdevKbdCtrl   ← 第4引数 ctrl_func
89e8: call InitKeyboardDeviceStruct@plt
```

`xkbInfo->kbdProc = ctrl_func` は `InitKeyboardDeviceStructInternal()` が行う。
**渡した値（`evdev+0x5090`）と、後で読み出した値（`libmtdev+0x4004`）が違う。**
どちらも非 NULL なので Xorg の 3 段の NULL チェックは素通りする。

### 分かっていること / 潰した仮説

- `EvdevKbdCtrl` は `evdev_drv.so` 内の static 関数で、アドレスは RIP 相対 `lea`
  で計算される。**動的リロケーションは介在しない**ので「シンボル解決ミス」ではない
- `LD_BIND_NOW=1` は既に入っているので遅延 PLT 経路でもない
- `libmtdev` は本来この経路では使われない（`mtdev_new_open` は
  `libevdev_has_event_code(EV_ABS, ABS_MT_POSITION_X/Y)` の内側にあり、
  この shim は EV_ABS を申告していない）。**アドレスがたまたま libmtdev の
  .rodata に落ちているだけ**で、libmtdev 自体は無関係
- ページのゼロ化は疑ったが白: `paging_map_user_range_alloc()`・
  デマンドゼロフォルト経路とも `memset(phys_page, 0, PAGE_SIZE)` 済み。
  `brk` 伸長（`process_set_heap_cursor` → `process_user_alloc`）も同じ経路
- kbd0 用の 2 回目の `xkbcomp` は成功している（`[fork] rc=0`、XKB エラー無し）

→ **Xorg のヒープ上で `XkbSrvInfoRec` が書き換わっている**線が濃厚。同種の症状が
`InputThread` の #GP（M18: 復帰先が文字列データ）でも出ており、
**「本来書いた値と違うポインタが読み出される」**という同じ形をしている。
共通原因があるなら、そこが本丸。

### 「kbd0 を外せば起動する」は実測で否定された

一度 `xorg.conf` から `kbd0` を外して**30回目ブート**を回した。X は mouse0 の
追加まで進むが、**その直後に別の場所で同じ形のクラッシュをする**:

```
(II) XINPUT: Adding extended input device "mouse0" (type: MOUSE, id 6)
[OS] [PF] CR2:0x1  RIP:0x410084FD90
```
`[lxmap]` で解決 → `modesetting_drv.so + 0x6d90` = **`drmModeConnectorSetProperty@plt`**
（`udev_*@plt` の並びの中）。つまりキーボード固有の問題ではなく、
**「本来有効なはずのポインタが 1 や 0x1000 のような小さいゴミになっている」**
という同じ症状が、デバイス有効化まわりの別経路でも出る。

したがって `kbd0` の除去は**何も買わない**ので元に戻した（キーボード側の
カーネル実装は正しく、外す理由が無い）。M19 は「evdev キーボードのバグ」ではなく
**Xorg プロセス内でポインタが壊れる系統的なバグ**として追うべきである。
確認された 3 例:

| # | 症状 | 壊れていたポインタ |
|---|---|---|
| 1 | `InputThread` #GP、`rip` が ASCII `"linux-gn"` | スレッドの復帰先 |
| 2 | `CR2:0x1000` `RIP=libmtdev+0x4004` | `dev->key->xkbInfo->kbdProc`（正しくは `evdev_drv.so+0x5090`）|
| 3 | `CR2:0x1` `RIP=modesetting_drv.so+0x6d90` | `drmModeConnectorSetProperty` の呼び出し経路 |

3 例とも「**書いたはずの値と違う、小さい/無関係な値が読み出される**」形。
ページのゼロ化・遅延/eager mmap の配置・シンボル解決は個別に潰してあるので
（M19 の「潰した仮説」参照）、次は**同じ物理ページが二重に使われていないか**
（`process_user_mmap` / heap / アリーナの重なり、`filemap` の seq 解決）を疑うのが筋。

### 次に見るべきもの

1. `XkbSrvInfoRec`（`dev->key->xkbInfo`）の確保直後と `XkbDDXKeybdCtrlProc`
   到達時で `+0x38` の値を比べる。X 側は無改変なので、カーネルから
   その物理ページを監視するか、`0x410069E9F0`（stack[1]、`dev` らしきヒープ
   ポインタ）を起点に追う
2. `InputThread` の #GP（M18）と同一原因かどうか。両方とも
   「書いたはずの値と違う値が読める」形なので、まずこの 2 つを結ぶ仮説を立てる
3. `[lxmap]` + `[OS] [PF] user stack:` は常設の診断として残してある。
   外来バイナリのクラッシュはこの 2 つで `<module>+<offset>` と呼び出し元まで
   機械的に落とせる（推測でアドレスを当てにいかないこと）

---

## M20. 系統的な原因を特定 — `brk` と mmap が同じアロケータを奪い合っていた

M18/M19 の 3 例（`InputThread` の #GP、`xkbInfo->kbdProc`、`modesetting_drv.so` の PLT）は
**すべて同じ 1 つのバグ**だった。

### 追い方

`[lxmap]`（M18）で全モジュールのロードベースが分かるので、生アドレスを
`<module>+<offset>` に落とせる。クラッシュは 100% 決定的（ASLR 無し）:

```
[OS] [PF] CR2:0x1000 RIP:0x410087B004
[OS] [PF] user stack: 0x400016175B 0x410069E9F0 ... 0x410086A090
```
- `RIP` → `libmtdev.so.1 + 0x4004` = **`.rodata`**（実行可能ですらない LOAD）
- `stack[0]` → `Xorg + 0x16175B` = **`XkbDDXKeybdCtrlProc+0x4B`**（戻りアドレス）
- `stack[7]` → `evdev_drv.so + 0x5090` = **`EvdevKbdCtrl`**（本来の正しい値）

Xorg 側:
```
161735: mov 0xb8(%rbx),%rax   # dev->key
161741: mov 0x68(%rax),%rax   # ->xkbInfo
16174a: mov 0x38(%rax),%rax   # ->kbdProc     (XkbSrvInfoRec の +0x38 に一致)
161759: call *%rax            ← rax がゴミ
```
evdev 側は正しい値を渡している（`lea -0x3953(%rip),%rcx  # 5090`）。
**書いた値と読んだ値が違う。**

### 真因

```c
void *process_user_mmap(uint64_t length, uint64_t flags)
{
    ...
    return process_user_alloc(aligned_len);   /* ← brk と同じアロケータ */
}

int process_set_heap_cursor(uint64_t addr)   /* = brk(2) */
{
    if (addr > proc->user_heap_cursor) grow = addr - proc->user_heap_cursor;
    ...
    if (grow) process_user_alloc((uint32_t)grow);
}
```

`process_user_alloc()` は **バンプ + フリーリスト**アロケータで、
`process_user_mmap()`（＝ 1 MiB 未満の file-backed mmap = **X のドライバモジュール全部**）と
`brk()`（＝ glibc の malloc アリーナ）が**同じ 1 本のカーソルを共有していた**。

glibc は「break が返した値から連続した領域を自分が排他所有している」前提で
チャンクアドレスを計算し、以後 break を読み直さない。したがって:

1. `brk()` が**フリーリストから満たされる**と、`process_user_alloc()` は
   再利用アドレスを返し **`user_heap_cursor` を進めない**。それでも
   `linux_brk()` は要求された `addr` を成功として返す
2. glibc は `[旧 break, addr)` を自分のヒープだと信じる
3. 次の `dlopen()` → `process_user_mmap()` → バンプ経路 → **旧 break のアドレスを返す**
4. 共有オブジェクトが **malloc が既に配ったメモリの上にマップされる**

`XkbSrvInfoRec` は `InitKbdFeedbackClassDeviceStruct()` の直前に `calloc` される。
その領域がモジュールのマッピングと重なれば、`kbdProc` は
「そのモジュールが自分の再配置で書いた値」に化ける — 観測した
`libmtdev + 0x4004` はまさにその形。`InputThread` が文字列データへ復帰したのも、
`modesetting_drv.so` の PLT で落ちたのも、すべて同じ「重なり」の別の当たり方。

遅くならないと出ないのは、フリーリストにスロットが溜まってからでないと (1) が
起きないため。M13 の遅延 mmap 以降に露出したのも、ライブラリの配置が変わって
重なり方が変わったから。

### 対処（このコミット）

| ファイル | 変更 |
|---|---|
| `ProcessScheduler.h` | `process_t` に `user_brk_base` / `user_brk_cursor` / `user_brk_limit` |
| `ProcessManager.h` | `USER_BRK_WINDOW_SIZE`（8 GiB）。`process_get_brk()` / `process_set_brk()` |
| `ProcessManager_Create.c` | プロセス生成と `execve` の両方で、`user_heap_alloc_limit` の直下に **brk 専用ウィンドウ**を切り出し、バンプアロケータの上限をその下端まで下げる（両者が絶対に交わらない）。`process_set_brk()` は**要求されたアドレスにちょうどページをマップ**する（アロケータ経由にしない）。縮小はカーソルを戻すだけ（マップは残す）。fork / スレッド生成で継承 |
| `Syscall_LinuxCompat.c` | `linux_brk()` を `process_get_brk()` / `process_set_brk()` へ |

ヒープ領域は約 28 GiB、スレッドスタック予約が約 2 GiB なので、
brk に 8 GiB を割いてもバンプ側に約 18 GiB 残る。

**あわせて直った以前の未修正点**: `process_set_heap_cursor()` の
`process_user_alloc((uint32_t)grow)` は `uint64_t` を **`uint32_t` に切り詰めて**いた。
brk 経路がこの関数を通らなくなったので影響しない。

---

## M21. `Dispatch()` 到達 — 次は `struct epoll_event` の ABI

**31回目ブート**（M20 の brk 分離を投入）。**M18/M19 の 3 例が全部消えた**:

- `kbd0` を追加しても落ちない（M19 のクラッシュ消滅）
- `mouse0` も追加され（id 7）、`InitInput()` を完走
- **X が `Dispatch()` に入った** — バックトレースに `WaitForSomething` が出ている

＝ M20 の「brk と mmap が同じアロケータを共有」が、3 つのクラッシュすべての真因だったことが実測で裏付けられた。

新しい fatal（今度はメインループの中）:

```
[OS] [PF] CR2:0x18  RIP:0x40001E26FD
Backtrace: Xorg[0x1e26fd] <- Xorg[0x1daf0b] <- Xorg[0x649c7] <- Xorg[0x68cd4]
           <- libc <- libc <- Xorg[_start+0x21]
```
シンボル解決（Debian の Xorg はローカルシンボルが落ちているので近傍名）:
`0x1daf0b` = **`WaitForSomething+0x18b`**。つまり
`_start → __libc_start_main → dix_main → Dispatch → WaitForSomething → ospoll_wait` で
NULL+0x18 を読んでいる。

### 真因

`ospoll_wait()`（`os/ospoll.c`, epoll バックエンド）は

```c
nready = epoll_wait(ospoll->epoll_fd, ospoll->events, MAX_EVENTS, timeout);
for (i = 0; i < nready; i++) {
    struct ospollfd *osfd = ospoll->events[i].data.ptr;   /* ← NULL だった */
    ...
}
```

**Linux の x86_64 `struct epoll_event` は `EPOLL_PACKED`（`__attribute__((packed))`）で
12 バイト・`data` はオフセット 4。** ImplusOS の定義は非 packed の 16 バイト・
オフセット 8 だった（カーネル `Syscall_Epoll.h` の `epoll_event_t`、
freestanding libc の `sys/epoll.h` の両方）。

したがって `epoll_wait()` が返す `data` は userland から見ると **4 バイトずれ**、
配列のストライドも 12 対 16 でずれる。`data.ptr` がゴミ（0）になり、
`osfd->` の +0x18 を読んで死ぬ。**`CR2=0x18` と完全に一致。**

### 対処（このコミット）

| ファイル | 変更 |
|---|---|
| `Kernel/Core/syscall/Syscall_Epoll.h` | `epoll_event_t` を x86_64 で `__attribute__((packed))` に（Linux と同一レイアウト）。他アーキは従来どおり（Linux も x86_64 だけ packed） |
| `libc/I_libc/include/sys/epoll.h` | `struct epoll_event` に同じ処置。カーネルとネイティブ userland でレイアウトを一致させる |

`sizeof` が 16→12 になるので、`epoll_ctl`/`epoll_wait` のユーザバッファ検証
（`maxevents * sizeof(epoll_event_t)`）も自動的に正しい値になる。

---

## M22. X が完全起動 — 残るはクライアントのハンドシェイク 1 点

**32〜39回目ブート**。M20（brk 分離）と M21（epoll ABI）の後、**X サーバは
最後まで起動し、メインループで生存し続ける**ようになった:

- `kbd0`(id 6) / `mouse0`(id 7) とも追加、`InitInput()` 完走
- `Dispatch()` に入り、`fatal` も `#PF` も出ない
- Doom もエンジン初期化を完走（`W_Init` → `I_Init` → sfx 108 個 →
  `D_CheckNetGame` → `S_Init` → `HU_Init` → `ST_Init`）

### この区間で見つけて直したもの

| # | 症状 | 真因 | 対処 |
|---|---|---|---|
| 1 | X がメインループで `CR2:0x18`、`WaitForSomething+0x18b` | **`struct epoll_event` の ABI 不一致**（M21）| x86_64 で `packed` に |
| 2 | X が `Dispatch()` に入った途端、2 CPU が張り付いて何も進まない | **`epoll_wait` が無限タイムアウトでも 0 を返す**。`WaitForSomething()` はタイマが無ければ `-1` を渡すので、X は userland で回り続けて CPU を食い潰す（TCG では Doom が餓死する）| 無限タイムアウトのとき `epoll_wait` / `poll` / `ppoll` は 0 を返さず、1 スライス眠って**同じ syscall をやり直す**（M17 の再実行機構）|
| 3 | Doom が接続はできるのに X が accept 直後に閉じ、X はクライアント 0 になってサーバリセット | **xserver の `AllocNewConnection()` は fd >= `lastfdesc` のクライアントを捨てる**。`lastfdesc` はコンパイル時 `MAXCLIENTS`(256) に張り付く（`-maxclients` では動かない。実測済み）。ImplusOS の AF_UNIX fd は 256 起点だったので**全 X クライアントが必ず捨てられていた** | fd レンジを再配置: ファイル `0..191`（`OS_CONFIG_FILE_MAX_FD` 256→192）、**AF_UNIX `192..255`**（`UNIX_SOCK_FD_BASE` 256→192）。以後 X は setup request を実際に読めるようになった（`[usock] rx fd=0xC3 n=0xC`）|
| 4 | ブロッキングソケットの `recv` が `EAGAIN` を返す | `unix_sock_t` に O_NONBLOCK の概念が無く、`F_GETFL` は常に `O_NONBLOCK` を詐称し `F_SETFL` は捨てていた | `nonblock` フラグを追加。`F_GETFL`/`F_SETFL` を実装。ブロッキングなら EAGAIN を返さず 1 スライス眠って syscall をやり直す |
| 5 | `select` / `pselect6` が **未実装（ENOSYS）** | Xlib / libxcb はサーバの応答をこれで待つ | `linux_select_common()` を実装（`poll` と同じ per-fd 判定を使用。NULL タイムアウトは再実行）|

### 残っている 1 点

```
[usock] conn-ok  fd=0xC2 n=0xC0      Doom が接続
[usock] tx       fd=0xC2 n=0xC       12 バイトの setup request 送信
[usock] accept   fd=0xC0 n=0xC3      X が accept
[usock] rx       fd=0xC3 n=0xC       X が 12 バイトを読む      ← M22-3 で到達
[usock] rx-EAGAIN fd=0xC2 n=8        Doom が 8 バイト読もうとして EAGAIN
Couldn't connect to display!         ← ここで諦める
```

X は setup request を読んでいるが**応答を書いていない**（`tx fd=0xC3` が出ない）。
そして Doom は最初の `EAGAIN` 1 回で諦めている（再試行の `rx` が続かない）。
#4/#5 を入れても挙動が変わらないので、**Doom 側の待ちが
`read`/`recvfrom`/`recvmsg`/`select`/`poll` のどれを通っているのか**が
まだ特定できていない。次にやるべきは `linux_unix_block_retry()` に判定理由
（`is_nonblock` の値）を出させて、どの分岐で抜けているかを 1 ブートで確定させること。

**方法A の残作業は 2 つだけ**: (a) このハンドシェイク、(b) §M7 の GLX
（Doom は `glXCreateContext`/`glXMakeCurrent`/`glXSwapBuffers`/`glewInit` を
UND 参照。現在 `-extension GLX` で無効化中）。

---

## M23. X11 接続が成立 — Doom がウィンドウと GL コンテキストを得るところまで到達

**40〜44回目ブート**。ここで方法A の X11 部分が**全部通った**。

### 直した 4 件

| # | 症状 | 真因 | 対処 |
|---|---|---|---|
| 1 | X が `kbd0` 追加後に沈黙し `Dispatch()` に入らない（`accept` が消えた）| M22 で入れた `select` が、NULL タイムアウトのときカーネル内で syscall を再実行し続け、X の入力デバイス初期化中に**ユーザ空間へ戻らなくなっていた** | `select`/`pselect6` は `epoll_wait` と違い**再実行しない**（1 スライス眠って 0 を返す）。呼び出し側が再試行できる spurious wakeup は無害だが、戻らない待ちはデッドロック |
| 2 | **`Couldn't connect to display!`（本命）** | `linux_socket_recvfrom()` が **AF_UNIX の全エラーを `EBUSY` に潰していた**（`if (got < 0) return LINUX_EBUSY;`）。libxcb の `read_block()` は `EAGAIN` なら poll して再試行、それ以外は接続を捨てる。したがって X がまだ応答を書いている最中の 1 回目の `recv()` で**必ず**接続が死んでいた | 本当の errno をそのまま返す。`sendto` 側の同じ潰し込みも修正 |
| 3 | `glXCreateContext()` が NULL（`context: (nil)` → `glXMakeCurrent` で GLX BadMatch）| **`libGLX_mesa.so.0` が未ステージ**。glvnd の `libGLX.so.0` は実装を `libGLX_<vendor>.so.0` として **dlopen** するので DT_NEEDED 閉包に入らない（`libgbm` / `libEGL` と同じ穴）。vendor 0 個だと `glXCreateContext` は黙って NULL を返す | `stage-xorg.sh` に §2c を追加し `libglx-mesa0` から明示配置（NEEDED 閉包は 15/15 充足済み）|
| 4 | GLX 自体が無効化されたままだった | §M7 の `CR2=0x38` クラッシュを理由に `-extension GLX` にしていたが、あれは **M20 の brk/mmap 破壊**が原因だった | `+iglx` に戻した。**`(II) IGLX: Loaded and initialized swrast` / `(II) GLX: Initialized DRISWRAST GL provider for screen 0` が出て、M7 のクラッシュは完全に消滅** |

### 実測できた到達点

```
[usock] conn-ok fd=0xC2 n=0xC0     Doom が接続
[usock] accept  fd=0xC0 n=0xC3     X が accept
[usock] tx 0xC2 n=0xC / rx 0xC3 n=0xC / tx 0xC3 n=0x20 / rx 0xC2 n=0x20
                                   ← X11 ハンドシェイク成立
Got display!                       ← XOpenDisplay 成功
Mapping window...                  ← ウィンドウ生成・マップ
visual_info: 0x457e1d1e60          ← glXChooseVisual 成功
context: 0x457e246980              ← glXCreateContext 成功
```

X サーバは fatal なし・#PF なしで起動し、GLX 拡張まで初期化される。
Doom はディスプレイに接続し、ウィンドウを作り、GL コンテキストを得るところまで動く。

### 残っている 1 点

コンテキストを current にする段で **X サーバが NULL 関数ポインタを呼んで落ちる**:

```
[OS] [PF] CR2:0x0 RIP:0x0 Error:0x15 (present|user|命令フェッチ)
[OS] [PF] user stack: 0x41006425B3 ...
```
`[lxmap]` で解決 → `libglx.so + 0x255B3`。逆アセンブルすると `DoMakeCurrent`:

```
25541: cmpb $0x0,0x61(%r15)     旧コンテキスト
25557: call *0x10(%r15)         loseCurrent(old)
25584: mov 0x20(%rsp),%rdx      新コンテキスト
2558e: cmpb $0x0,0x61(%rdx)     isDirect なら飛ばす
2559b: mov %r12,0xb8(%rdx)      drawPriv
255a5: mov %r13,0xc0(%rdx)      readPriv
255ac: mov %rdx,0x0(%rbp)       lastGLContext = cx
255b0: call *0x8(%rdx)          ← cx->makeCurrent が NULL
```

`__GLXcontext` の vtable（+0x0 destroy / +0x8 makeCurrent / +0x10 loseCurrent）が
埋まっていない。`glxdriswrast.c` の `__glXDRIscreenCreateContext()` は calloc 直後に
これらを必ず設定し、`swrast->createNewContext` が失敗したら context ごと free して
NULL を返す（＝クライアントにエラーが返るだけでクラッシュしない）ので、
**この構造体がどこで作られたのかが次の調査点**。`0x61`（isDirect）も 0 に読めている
＝構造体が丸ごとゼロの可能性がある。

次にやること:
1. `DoCreateContext` 側（`libglx.so` 内）を逆アセンブルし、どの `createContext`
   実装が呼ばれたかを確定する。Mesa 25 の `libdril_dri.so` が
   `__DRI_SWRAST` の `createNewContext` に何を返すかも併せて確認する
2. M20 と同型の「ゼロで返ってくるはずのない構造体がゼロ」パターンなので、
   残存するメモリ破壊の可能性も一応疑う（brk/mmap は分離済みだが、
   `process_user_alloc` のフリーリスト再利用など未検証の経路はまだある）
