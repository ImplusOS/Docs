# ImplusOS glibc 移植プラン（Chromium ヘッドレス起動のための動的リンク基盤）

> **ステータス: §7 の決定を 2026-08-29 に確定。実装フェーズ着手（G0/G1/G2 完了、§11 参照）。**
>
> 調査基準日: 2026-08-29
> 対象: 外来 Linux ABI バイナリ（当面は `Userland/Application/Chromium/Resource/chrome`）を
> **動的リンクのまま multiprocess で起動**できるようにするための、Linux ランタイム
> （glibc + 依存共有ライブラリ + ロケール）同梱基盤の新設。
> カーネル側 Linux syscall 互換レイヤーの拡充は既存
> [`TODO_Chromium_LinuxABI.md`](TODO_Chromium_LinuxABI.md) が担当。本書はそれと重複しない。

---

## 0. なぜ必要か（現状の失敗）

Userland の初期化で Chromium ランチャ（`Userland/Application/Chromium/Start.c`）が

```c
process_spawn("/Userland/Chromium/Resource/chrome");
```

を呼ぶが、次で停止する:

```
[spawn-fail] pid=0x…5 path=/Userland/Chromium/Resource/chrome
             cr3=0x…D59C000 elf_err=interpreter load failed
```

`elf_err` の発生源は `Kernel/Core/elf/ELF_Loader.c:1377`
（`elf_load_image_biased(... interp_path ...)` の失敗）。

`Resource/chrome` は **upstream の Linux デスクトップ版 Chromium 実バイナリ**:

| 属性 | 値 |
|---|---|
| サイズ | 518,687,224 バイト（約 495 MiB） |
| 種別 | `ET_DYN`（PIE）、`EI_OSABI=SYSV`、エントリ `0x5309000` → ローダは `linux_abi=1` と判定 |
| `PT_INTERP` | `/lib64/ld-linux-x86-64.so.2` |
| `DT_NEEDED`（直接） | 32 本（`libc.so.6` `libm.so.6` `libdl.so.2` `libpthread.so.0` `libgcc_s.so.1`、`libglib-2.0` `libgobject-2.0` `libgio-2.0` `libnspr4` `libnss3` `libnssutil3` `libsmime3` `libdbus-1` `libcups` `libexpat` `libxcb` `libxkbcommon` `libasound` `libgbm` `libX11` `libXext` `libXcomposite` `libXdamage` `libXfixes` `libXrandr` `libatk-1.0` `libatk-bridge-2.0` `libatspi` `libcairo` `libpango-1.0` `libudev`） |
| 推移的閉包（実測見込み） | 60〜90 本（`libxcb` → `libXau`/`libXdmcp`、`libcairo` → `pixman`/`fontconfig`/`freetype`/`libpng`/`zlib`、`libpango` → `harfbuzz`/`fribidi`、`libglib` → `libpcre2`/`libffi`、`libnss3` → `libnspr4`、`libcups` → `libgnutls`/`libavahi` …） |

> `chrome` は `libstdc++` を必要としない（Chromium は自前の `libc++` を静的リンク）。
> `libgcc_s.so.1` は必要。

現在のツリーには次が **すべて無い**:

- `/lib64/ld-linux-x86-64.so.2`（動的リンカ本体）＝ **今回の直接原因**
- glibc ランタイム（`libc.so.6` ほか）
- 上記 `DT_NEEDED` 閉包の共有ライブラリ本体
- Linux ランタイムをイメージへ取り込む仕組み（vendoring / ビルド / ステージング）

ImplusOS の Linux ABI 経路は現状 **静的リンクバイナリ専用**（BusyBox が
`1.35.0-x86_64-linux-musl` の静的ビルドで動いているのが唯一の実績）。
`ELF_Loader.c` に `PT_INTERP` を読む処理自体はあるが、指定パスの実ファイルが
VFS 上に存在しなければ即失敗する。

---

## 1. 現況インベントリ（2026-08-29 実測）

### 1.1 カーネル側（概ね準備済み）

| 項目 | 状態 | 参照 |
|---|---|---|
| `Kernel/Compat/Linux/Syscall_LinuxCompat.c` | 実装済み・245 `case` | ABI 判定は `ELF_Loader.c`、分岐は `compat_registry` 経由 |
| DevFS `/dev/{null,zero,full,urandom,random,tty}` | 実装済み | `Kernel/Core/vfs/DevFS.c` |
| TmpFS `/dev/shm` | 実装済み（サブディレクトリ非対応） | `Kernel/Core/vfs/TmpFS.c` |
| ProcFS `/proc/self/{maps,status,stat,cmdline,exe,fd/N}` ほか | 実装済み（自プロセスのみ、open 時生成） | `Kernel/Core/vfs/ProcFS.c` |
| EtcFS `/etc/{nsswitch.conf,ld.so.conf,passwd,group,host.conf,gai.conf,hosts,resolv.conf,localtime,os-release}` | 実装済み | `Kernel/Core/vfs/EtcFS.c` |
| `PT_INTERP` 読み取り＋ロードバイアス（メイン=`0x40_0000_0000` / interp=`USER_CODE_LIMIT-128MiB`） | 実装済み | `ELF_Loader.c:1150-1399` |
| auxv | `AT_PHDR/PHENT/PHNUM/PAGESZ/BASE/ENTRY/HWCAP/CLKTCK/UID/EUID/GID/EGID/SECURE/RANDOM/EXECFN`。**`AT_SYSINFO_EHDR`（vDSO）無し** | `ProcessManager_Create.c:1354-1369` |
| `PROCESS_ELF_MAX_SIZE` | 768 MiB（`chrome` 495 MiB は収まる） | `ProcessManager_Create.c:49` |
| セグメントのストリーミング読み込み（256 KiB ステージング） | 実装済み | `ELF_Loader.c:1257-1289` |
| COW fork | 実装済みだが `KERNEL_COW_FORK=0`（既定無効） → **G7 で既定有効化** | `Kernel/include/kernel/config.h:33` |
| `OS_CONFIG_FILE_MAX_FD` / `OS_CONFIG_PROCESS_MAX_COUNT` | 256 / 256 | `config.h` |
| デマンドページング（ELF セグメント） | 無し（全 `PT_LOAD` を eager map） | `chrome` の静的画像 ~333 MB 常駐 |

### 1.2 未整備（本プランで新設）

- Linux ランタイム（glibc + 依存 `.so` 閉包 + `C.UTF-8` locale-archive）
- それを取得・ステージするビルド機構（`libc/` と `Vendor/` 配下、**各ディレクトリ自前 Makefile**）
- イメージへの **常時** 配線（トグル不可、単一 ISO）
- 外来 Linux ABI プロセスの既定 envp 補正、multiprocess fork 既定有効化
- OSS ライセンス表記（`Docs/OSS_License/`）

---

## 2. 方針（2026-08-29 確定）

1. **Linux ランタイムはビルド済みバイナリを vendoring する。**
   第一経路は Debian 13 "trixie"（glibc 2.41 系）の公式 `.deb` から
   `.so` を抽出して同梱。ImplusOS 固有の `sysdeps` port も、依存ライブラリの
   自前クロスビルドも **行わない**。
   （glibc をソースから作る経路は §G6 に「フォールバック」として残すが既定では使わない。）
2. **第三者コード／バイナリには一切変更を加えない。** パッチ・リンク時差し替え・
   バイナリ書き換えall禁止。取得した `.so` はそのままステージする。互換の
   ずれはすべてカーネルの Linux syscall 互換層側で吸収する（`ENOSYS` に集約）。
3. **glibc（Linux ランタイム）同梱は既定かつ変更不可。** `WITH_GLIBC` の
   ような opt-in フラグは設けない。ISO は分割しない（単一の
   `Image/ImplusOS-x86_64-*.iso` に常に入る）。`INSTALL_DISK_IMAGE_SIZE_MB` を
   恒久的に拡張する（§G5）。
4. `libc/I_libc`（最小 freestanding libc）は **カーネル／ネイティブ userland 用として不変**。
   Linux ランタイムは **外来 Linux バイナリ専用**で、別ツリー・別ステージ・別リンク。
5. **ビルドルールをトップ Makefile にべた書きしない。** vendoring / glibc /
   locale はそれぞれのディレクトリの Makefile（`libc/Makefile`,
   `Vendor/LinuxRuntime/Makefile` 等）に閉じ込め、トップ Makefile は
   `$(MAKE) -C <dir>` で委譲するだけにする。既存の `Vendor/Library/Makefile`
   （libpng/zlib/…）と同じ構造。
6. **multiprocess Chromium を目標にする。** `--single-process` に逃げない。
   `KERNEL_COW_FORK` を既定有効化し、zygote → renderer/gpu/utility の多重
   fork が成立することを目標とする（`TODO_Chromium_LinuxABI.md` §3.2 と協調）。
7. **`C.UTF-8` locale-archive を同梱する。** ホストの `localedef` で生成
   （glibc ソースの改変ではなく、データ生成）。`LANG=C.UTF-8` を既定にする。
8. glibc バージョンは **2.41 系に固定**（trixie の版に追従、`readelf -V
   Resource/chrome` の `GLIBC_2.xx` 要求がすべて満たされることを確認）。
9. **OSS ライセンス順守。** 同梱する各ライブラリのライセンス（LGPL-2.1 の
   glibc、MIT/X11 の X libs、MPL-2.0 の NSS、LGPL-2.1 の GTK/glib 系 等）を
   `Docs/OSS_License/` に列挙し、再配布条件（LGPL: 動的リンクなので条件は緩い
   が、ライブラリの入手先とライセンス全文の同梱が必要）を満たす。
10. 動作確認は **QEMU（q35/OVMF, 4 GiB, `-serial stdio`）実起動**でのみ成立
    とみなす。コンパイル通過・`-Werror` 通過は前提条件であって完了条件ではない。

---

## 3. アーキテクチャ

```
外来 Linux ELF（chrome, PIE, glibc 依存）
  │  execve / process_spawn
  ▼
ELF_Loader.c        … PT_INTERP="/lib64/ld-linux-x86-64.so.2" を VFS から読む
  │                    interp を USER_CODE_LIMIT-128MiB にバイアス配置、
  │                    AT_BASE / AT_ENTRY / AT_PHDR を auxv に積む
  ▼
ld-linux-x86-64.so.2（Debian trixie の glibc 2.41、ユーザー空間で実行、無改変）
  │  openat()/mmap(MAP_FIXED)/mprotect()/close() を発行
  │  /etc/ld.so.conf + DT_RUNPATH + LD_LIBRARY_PATH=/lib64:/usr/lib で .so を検索
  ▼
Kernel/Compat/Linux/Syscall_LinuxCompat.c … 上記 syscall を VFS/VM へ委譲
  ▼
libc.so.6 → (chrome) 依存 .so 閉包 60〜90 本 → chrome _start → __libc_start_main → main
  │
  ├─ zygote: clone/clone3 → COW fork（KERNEL_COW_FORK=1）
  └─ renderer/gpu/utility 子プロセス（multiprocess）
```

**イメージ上のファイル配置（常時）**

```
::/lib64/ld-linux-x86-64.so.2              ← glibc 動的リンカ（trixie, 無改変）
::/lib64/libc.so.6 libm.so.6 …             ← glibc ランタイム（実体コピー、symlink 解決）
::/usr/lib/x86_64-linux-gnu/…              ← 依存 .so 閉包（trixie のパス構造を踏襲）
::/usr/lib/locale/locale-archive           ← C.UTF-8（localedef 生成）
::/etc/ld.so.conf                          ← EtcFS が供給（イメージには入れない）
::/Userland/Chromium/Resource/chrome …     ← 既存（ランチャ Start.c が spawn）
```

---

## 4. Linux ランタイム vendoring 機構

### 4.1 ディレクトリ構成（新設）

```
Vendor/LinuxRuntime/
├── Makefile               … fetch → extract → stage を実行（トップから -C 委譲）
├── fetch.sh               … .deb を snapshot.debian.org から取得（URL+sha256 ピン）
├── packages.lock          … {パッケージ名, バージョン, .deb URL, sha256} の固定表
├── closure.txt            … Resource/chrome から解決した soname → パッケージ 対応（生成物、コミットする）
├── stage.sh               … .deb 展開 → /lib64・/usr/lib へ symlink 解決コピー
├── cache/                 … 取得した .deb（.gitignore、ビルド時生成）
├── LICENSES/              … 各パッケージの copyright ファイル（stage 時に収集）
└── README.md
```

- `cache/` は `.gitignore`。初回ビルドのみネットワークが要る。以後はオフラインで
  再現ビルド可能（`packages.lock` と sha256 で固定）。
- 巨大バイナリ（合計 150〜250 MiB）を **リポジトリにはコミットしない**。
  `packages.lock`（テキスト）と `closure.txt`（テキスト）だけをコミットし、
  実体は決定論的に再取得する。
  → **確認事項**: この方針でよいか（vendoring = 実バイナリのコミット、を
  期待している場合は `cache/` をコミット対象に変える。§7-6）。

### 4.2 依存閉包の解決（`closure.txt` 生成手順）

1. `readelf -d Resource/chrome` で直接 `DT_NEEDED` を列挙。
2. 各 soname を Debian trixie の `Contents-amd64` インデックスで
   パッケージへ逆引き（`libc6`, `libx11-6`, `libnss3`, `libglib2.0-0t64`, …）。
3. 取得した `.deb` を展開し、含まれる `.so` の `DT_NEEDED` を再帰的に辿る。
4. glibc が供給する soname（`libc.so.6` `libm.so.6` `libpthread.so.0`
   `libdl.so.2` `librt.so.1` `libresolv.so.2` `ld-linux-x86-64.so.2`）で
   打ち切り。ImplusOS が自前提供するもの（無し）で打ち切り。
5. 収束した集合を `closure.txt` に固定（`soname  package  path` の3列）。
6. `packages.lock` に各パッケージの
   `snapshot.debian.org/archive/debian/<date>/pool/.../<pkg>_<ver>_amd64.deb`
   URL と sha256 を記録。

### 4.3 `Vendor/LinuxRuntime/Makefile` ターゲット

| ターゲット | 動作 |
|---|---|
| `fetch` | `packages.lock` の全 `.deb` を `cache/` へ取得し sha256 照合 |
| `stage STAGE_DIR=<d>` | `cache/*.deb` を展開、`<d>/lib64` `<d>/usr/lib/x86_64-linux-gnu` へ実体コピー（symlink 解決）、`LICENSES/` に copyright 収集 |
| `closure` | `Resource/chrome` から `closure.txt` を再生成（開発時のみ） |
| `clean` | `cache/` と展開ディレクトリを削除 |

トップ `Makefile` からは `linux_runtime_stage:` が `$(MAKE) -C
Vendor/LinuxRuntime stage STAGE_DIR=…` を呼ぶだけ。

---

## 5. フェーズ分割

### G0 — vendoring 機構の骨組み

- [ ] `Vendor/LinuxRuntime/` を新設（§4.1 の構成）。`.gitignore` に `cache/`。
- [ ] `Vendor/LinuxRuntime/Makefile`（`fetch` / `stage` / `closure` / `clean`）。
      ロジックは `fetch.sh` / `stage.sh` に置き、Makefile は薄く。
- [ ] トップ `Makefile` に `linux_runtime_stage` を追加し `-C` 委譲のみ。
      **既存ターゲットのレシピ本体は書き換えない**（`install_payload` からの
      呼び出し 1 行追加に留める）。
- [ ] `Vendor/LinuxRuntime/README.md`（入手元・ピン方針・再現手順）。

### G1 — Debian trixie ランタイムの取得と閉包固定

- [ ] `readelf -V Userland/Application/Chromium/Resource/chrome` で
      `GLIBC_2.xx` 最大要求を確認 → trixie の glibc 2.41-x が満たすことを確認。
- [ ] §4.2 の手順で `closure.txt` を生成（初版）。
- [ ] `packages.lock` を作成（snapshot.debian.org の 1 スナップショット日付に統一、
      sha256 ピン）。
- [ ] `make -C Vendor/LinuxRuntime fetch` が sha256 照合込みで完走。
- [ ] `make -C Vendor/LinuxRuntime stage STAGE_DIR=/tmp/lr` が
      `/tmp/lr/lib64/ld-linux-x86-64.so.2` と `libc.so.6`、および閉包の全 `.so`
      を配置し、`LICENSES/` に各 copyright を集めることを確認。
- [ ] 欠落 soname（`stage` 後に `closure.txt` の全 soname が揃うか照合する
      自己検査を `stage.sh` 末尾に入れる）ゼロを確認。

### G2 — ロケール（`C.UTF-8`）

- [ ] `Vendor/LinuxRuntime/Makefile` に `locale` ターゲット:
      ホスト `localedef -i C -f UTF-8 --prefix=<stage> C.UTF-8` で
      `<stage>/usr/lib/locale/locale-archive` を生成（trixie の
      `/usr/share/i18n` を使う。ホストに無ければ `locales` パッケージも
      `packages.lock` に加えて `charmaps`/`i18n` データを展開して使用）。
- [ ] `stage` が `locale-archive` も配置することを確認。

### G3 — ELF ローダ / auxv / envp（カーネル小改修）

- [ ] `ELF_Loader.c` の `PT_INTERP` 解決が `/lib64/ld-linux-x86-64.so.2` を
      絶対パスで VFS 参照することを確認。ld.so 自身は `PT_INTERP` を持たない
      ので「nested interpreter not supported」には当たらない。
- [ ] `chrome` メイン画像の最大バイアス済みアドレスが interp 窓
      （`USER_CODE_LIMIT - 128MiB`）に食い込まないことを実測。食い込むなら
      `ELF_INTERP_BIAS_RESERVE` 見直し。
- [ ] **外来 Linux ABI プロセスの既定 envp を補正**
      （`ProcessManager_Create.c:2310` `linux_envp[]`）。現状は ImplusOS 独自
      ldso 向けの `LD_LIBRARY_PATH=/Userland/Service/com.ImplusOS.dynmain/lib`
      /`LD_PRELOAD=libpreload.so` が入っており glibc の ld.so が誤動作する。
      → `PATH=/bin:/usr/bin` / `LD_LIBRARY_PATH=/lib64:/usr/lib/x86_64-linux-gnu`
      / `LANG=C.UTF-8` / `LC_ALL=C.UTF-8` / `TZ=UTC` のみ、`LD_PRELOAD` は外す。
      （`abi_mode == PROCESS_ABI_LINUX` のときだけ切り替え。ネイティブ経路は不変。）
- [ ] `AT_SYSINFO_EHDR` は積まない（確定）。glibc は vDSO 不在で実 syscall に
      フォールバック。
- [ ] `AT_RANDOM`(16B) / `AT_EXECFN` の妥当性を再確認。

### G4 — `/etc` 補助ファイル（EtcFS、確認と補充）

- [ ] `/etc/ld.so.conf` に `/lib64` `/usr/lib/x86_64-linux-gnu` `/usr/lib` を含める
      （trixie のマルチアーチパスを追加）。
- [ ] `/etc/nsswitch.conf` `hosts: files dns` / `passwd: files`。
- [ ] `/etc/passwd` `/etc/group` に root/nobody。
- [ ] `/etc/resolv.conf`（動的）`nameserver 10.0.2.3`。
- [ ] `/etc/localtime` 妥当な `Etc/UTC` TZif。
- [ ] `/etc/machine-id`（dbus/glib が参照）を静的生成（16 バイト hex 固定でよい）。

### G5 — イメージ配線（常時・トグル不可・単一 ISO）

- [ ] `install_payload`（`Makefile:398`）に `linux_runtime_stage` を **無条件**
      依存追加。`$(INSTALL_PAYLOAD_ROOT)/{lib64,usr/lib,usr/lib/locale}` に配置。
- [ ] パーティションイメージ作成部（`Makefile:437` 付近）に
      `mmd ::/lib64 ::/usr ::/usr/lib ::/usr/lib/x86_64-linux-gnu
      ::/usr/lib/locale` と対応する `mcopy -s` を **無条件** 追加。
- [ ] `image_livecd` 経路にも同じ配置を無条件追加。
- [ ] `INSTALL_DISK_IMAGE_SIZE_MB` を 256 → **2560** に恒久変更
      （`chrome` リソース 661 MiB + ランタイム閉包 ~250 MiB + locale ~30 MiB +
      既存ペイロード + FAT オーバヘッド + 余裕）。実測後に最小化。
- [ ] `WITH_GLIBC` 等の分岐は **入れない**。

### G6 —（フォールバックのみ）glibc ソースビルド

> 既定では使わない。trixie バイナリが版ズレ等で使えないと判明した場合の保険。

- [ ] `.gitmodules` に `libc/glibc`（`release/2.41/master`、タグ `glibc-2.41`、無改変）。
- [ ] `libc/Makefile` + `libc/glibc.mk`（`configure`/`build`/`stage`/`clean`）。
      `--host=x86_64-linux-gnu`、`--enable-kernel=5.15.0`、`libc_cv_slibdir=/lib64`、
      `--disable-nscd --without-selinux`、`MAKEINFO=:`。out-of-tree
      （`Build/glibc/obj`）、DESTDIR（`Build/glibc/sysroot`）。作業ツリー無改変。
- [ ] トップ `Makefile` は `glibc:` で `$(MAKE) -C libc glibc` を呼ぶだけ。

### G7 — multiprocess fork（COW 既定有効化、`TODO_Chromium_LinuxABI.md` と協調）

- [~] `KERNEL_COW_FORK` 既定 `1` 化は **一旦保留**（2026-08-29 セッション3 で
      `0` に差し戻し。起動不安定＝§11 参照）。まず COW 単体の QEMU 回帰パスが要る。
      それまで multiprocess は eager copy（ゲスト RAM 増）で回す。
- [ ] `process_clone_address_space` の Linux ABI 経路が COW clone を使い、
      失敗時 eager copy にフォールバックすることを確認（既存実装）。
- [ ] PF ハンドラの COW フォールト処理（`PAGE_COW` bit11、物理ページ参照
      カウント）を SMP で確認。TLB シュートダウン経路のレビュー。
- [ ] `clone`/`clone3` の `CLONE_VM|CLONE_FS|CLONE_FILES|CLONE_THREAD|
      CLONE_SETTLS|CLONE_PARENT_SETTID|CLONE_CHILD_CLEARTID` の各フラグを
      Chromium/glibc-NPTL が使う組合せで検証（`Syscall_LinuxCompat.c`）。
- [ ] zygote は `--no-sandbox` 前提（seccomp/ユーザ名前空間は非対象）。
      `--no-zygote` は **付けない**（multiprocess 目標のため）。GPU は
      `--disable-gpu --use-gl=swiftshader`、当面 `--headless=new`。
- [ ] 子プロセス多重時の `OS_CONFIG_PROCESS_MAX_COUNT`(256) / メモリ余力を実測。
      `chrome` の ~333 MB 静的画像は COW 共有されるので子ごとの増分は小さい想定。
      崩れる場合は ELF セグメントのデマンドページングを別 TODO 起票。

### G8 — 起動検証マイルストーン（QEMU）

- [ ] **M2**: ホスト `x86_64-linux-gnu-gcc` で作った動的 Hello World を
      `/bin/hello` に入れ、`process_spawn` で stdout が出て正常終了。
      （ld.so → `libc.so.6` mmap → シンボル解決 → `main` → `write` →
      `exit_group` の全経路）。
- [ ] **M3**: trixie の動的 `dash`（`packages.lock` に追加）を起動しプロンプト。
- [ ] `-DLINUX_SYSCALL_TRACE`（`TODO_Chromium_LinuxABI.md` §6）でトレースし
      `ENOSYS` を §G9 に記録。
- [ ] **M4**: `chrome --headless=new --no-sandbox --disable-gpu
      --use-gl=swiftshader --dump-dom about:blank` が DOM を出力。
- [ ] **M5**: 同上で multiprocess（zygote あり）で renderer が起動し
      `--dump-dom https://example.com` 相当が取得できる。

### G9 — syscall ギャップ埋め（運用フェーズ）

トレースで顕在化した `ENOSYS` を潰す（`TODO_Chromium_LinuxABI.md` の未実装表と
重複するものは向こうで管理）。想定候補: `statx` 完全性 / `clone3` /
`rseq`（`-ENOSYS` 明示で可）/ `set_robust_list` / `membarrier` /
`sched_getaffinity` / `prlimit64` の各 `RLIMIT_*` / `getrandom(GRND_INSECURE)` /
`faccessat2` / `close_range` / `pidfd_*`。

### G10 — OSS ライセンス表記

- [ ] `Docs/OSS_License/LinuxRuntime/` に、同梱する全パッケージの
      `usr/share/doc/<pkg>/copyright` を収集（`stage.sh` が
      `Vendor/LinuxRuntime/LICENSES/` に集めたものを配置）。
- [ ] `Docs/OSS_License/README.md`（or 既存の索引）に
      「Linux runtime libraries (Debian trixie, unmodified): glibc (LGPL-2.1+),
      libX11 他 X libraries (MIT), NSS (MPL-2.0), GLib/GTK stack (LGPL-2.1+),
      ALSA lib (LGPL-2.1+), … 入手元 snapshot.debian.org、`packages.lock` 参照」
      を追記。
- [ ] LGPL 順守メモ: 動的リンクのみ・ライブラリ無改変・入手元明示・
      ライセンス全文同梱で条件を満たす旨を記載。Chromium バイナリ自体の
      再配布可否（BSD-3-Clause + 多数の同梱 OSS）は
      `Userland/Application/Chromium/` 側の別 TODO とする。

---

## 6. 生成物・変更一覧

| パス | 種別 | 内容 |
|---|---|---|
| `Vendor/LinuxRuntime/` | 新規 | vendoring 機構一式（Makefile/スクリプト/lock/closure/README） |
| `Vendor/LinuxRuntime/cache/` | 非コミット | 取得 `.deb`（`.gitignore`） |
| `.gitignore`（追記） | 変更 | `Vendor/LinuxRuntime/cache/` |
| `Makefile`（最小追記） | 変更 | `linux_runtime_stage` 委譲 1 ターゲット、`install_payload`/`image`/`image_livecd` に無条件 1 行、`INSTALL_DISK_IMAGE_SIZE_MB` 変更 |
| `libc/Makefile` `libc/glibc.mk` | 新規 | glibc ソースビルド（フォールバック、既定未使用） |
| `.gitmodules`（追記） | 変更 | `libc/glibc`（フォールバック用、無改変 submodule） |
| `Kernel/Core/process/ProcessManager_Create.c` | 変更 | 外来 Linux ABI の既定 envp 補正（`abi_mode==LINUX` 時のみ） |
| `Kernel/Core/vfs/EtcFS.c` | 変更 | `ld.so.conf` マルチアーチパス追加、`/etc/machine-id` 追加 |
| `Kernel/include/kernel/config.h` | 変更 | `KERNEL_COW_FORK` 既定 `1` |
| `Docs/OSS_License/LinuxRuntime/` | 新規 | 同梱ライブラリの copyright 収集＋索引追記 |

**トップ Makefile への“べた書き”は避け、各機構を自ディレクトリの Makefile に閉じる**（方針 §2-5）。
**第三者コード・バイナリは無改変**（方針 §2-2）。

---

## 7. 決定事項（2026-08-29 確定）

| # | 論点 | 決定 |
|---|---|---|
| 7-1 | 非 glibc `DT_NEEDED` 閉包 | **ビルド済みバイナリを入手してそのまま流用**。Debian trixie の公式 `.deb`（glibc 2.41 系）から `.so` 閉包を抽出し無改変で同梱。個別クロスビルド（旧案 C）はフォールバック位置付けに後退。 |
| 7-2 | glibc バージョン | **2.41 系に固定**（trixie 追従）。`readelf -V Resource/chrome` の要求を満たすこと。 |
| 7-3 | ISO / 同梱可否 | **ISO は分割しない。Linux ランタイム同梱は既定かつ変更不可。** `WITH_GLIBC` 等のトグルは設けず、`INSTALL_DISK_IMAGE_SIZE_MB` を恒久拡張。 |
| 7-4 | ロケール | **`C.UTF-8` locale-archive を同梱**（`localedef` 生成）。`LANG=LC_ALL=C.UTF-8` 既定。 |
| 7-5 | multiprocess / COW | **multiprocess で fork する**。`KERNEL_COW_FORK` 既定有効化、`--single-process`/`--no-zygote` に逃げない。`TODO_Chromium_LinuxABI.md` と並行実施。 |
| 7-6 | vendoring 実体のコミット | **未確定**。テキストのピン（`packages.lock`+`closure.txt`）のみコミットし `.deb`/`.so` 実体はビルド時取得（`cache/` は `.gitignore`）を既定案とする。実バイナリのリポジトリ同梱を望む場合は指示があれば `cache/` をコミット対象へ変更。 |
| 7-7 | 追加要望 | ① ビルドルールは各ライブラリ/`libc` ごとの Makefile に分離（トップにべた書き禁止）。② 第三者コード・ソース・バイナリは一切無改変。③ 同梱ライブラリを `Docs/OSS_License/` に追記、ライセンス順守。 |

---

## 8. 受け入れ基準（マイルストーン）

| # | 条件 | 状態 |
|---|---|---|
| M0 | `make -C Vendor/LinuxRuntime fetch` が sha256 照合込みで完走 | ☐ |
| M1 | `make image`（**引数なし**）が単一 ISO に `/lib64/ld-linux-x86-64.so.2` + 閉包全 `.so` + `locale-archive` を含めて完走 | ☐ |
| M2 | 動的リンクの Hello World が QEMU 上で stdout を出して正常終了 | ☐ |
| M3 | 動的リンクの `dash` が対話プロンプトを出す | ☐ |
| M4 | `chrome --headless=new --no-sandbox --disable-gpu --dump-dom about:blank` が DOM 出力（単プロセス可） | ☐ |
| M5 | 同上が **multiprocess（zygote あり）** で renderer 起動まで到達 | ☐ |
| M6 | `Docs/OSS_License/` に同梱ライブラリ全件の copyright とライセンス索引が入っている | ☐ |
| M7 | 第三者ツリー（`Vendor/LinuxRuntime/cache` 展開物、`libc/glibc`）に差分パッチが 1 件も無い | ☐ |

---

## 9. リスク / 未解決

- **依存閉包の規模**: `chrome` フル版は 60〜90 本の `.so` を引く。trixie の
  マルチアーチ配置をそのまま持ち込むのが最も破綻が少ないが、`fontconfig` の
  キャッシュ、`gio` モジュール、`dbus` デーモン不在時の挙動など、ファイル配置
  以外の実行時前提が残る。M4 到達前に潰しきれない可能性。
- **QEMU 実起動がこの環境で不可**: `TODO_Chromium_LinuxABI.md` と同じ制約。
  M2〜M5 はユーザーの実機/QEMU 検証待ちになる。実装はコンパイル通過＋
  `-Werror` 通過＋トレース設計までを各セッションの完了線とする。
- **eager map の物理メモリ**: COW で子プロセス増分は抑えられるが、親の
  ~333 MB は常駐。4 GiB QEMU で複数 renderer + GPU + V8 ヒープが乗るかは実測。
- **snapshot.debian.org の可用性**: ピン先が消えることは基本ないが、初回
  ビルドはネット必須。`cache/` コミット（7-6）で回避可能。
- **ライセンス**: 同梱ライブラリは動的リンク・無改変なので LGPL/MIT/MPL いずれも
  再配布可能。ただし Chromium バイナリ自体の再配布条件は別途要確認（本プランの
  スコープ外、`Userland/Application/Chromium/` 側 TODO）。

---

## 10. 参照

- 直接原因: `Kernel/Core/elf/ELF_Loader.c:1364-1387`（`interpreter load failed`）
- ロードバイアス／`PT_INTERP`: `ELF_Loader.c:1150-1399`
- auxv 構築: `Kernel/Core/process/ProcessManager_Create.c:1290-1400`
- 外来 ABI 既定 envp: `ProcessManager_Create.c:2300-2320`
- Linux syscall 互換層: `Kernel/Compat/Linux/Syscall_LinuxCompat.c`
- COW fork: `Kernel/include/kernel/config.h:33`、`Kernel/Arch/x86_64/mmu/`、`ProcessManager` clone 経路
- カーネル側 ABI 拡充の既存プラン: [`TODO_Chromium_LinuxABI.md`](TODO_Chromium_LinuxABI.md)
- イメージ生成: `Makefile:398-465`（`install_payload` / `image`）、vendor 委譲の先例 `Vendor/Library/Makefile`
- ランチャ: `Userland/Application/Chromium/{Start.c,Makefile}`

---

## 11. 実装ログ

### 2026-08-29 セッション1（G0/G1/G2 + カーネル小改修）

**新規 `Vendor/LinuxRuntime/`**（vendoring 機構、`Vendor/Library` と同じ委譲構造）
- `resolve.sh` — `Resource/chrome` の `DT_NEEDED` 推移的閉包を Debian trixie の
  `Contents-amd64` / `Packages` 索引から解決。**閉包は「必要な soname に一致する
  `.so` 1 個」だけを辿る**（パッケージ内の兄弟 `.so`＝DRI ドライバや gdb ヘルパを
  辿ると `libgbm → Mesa → libLLVM` で閉包が数百 MB 膨張するため）。`-dbg`/`-dev`/
  `/debug/`/`.build-id` は soname マップから除外。
- `fetch.sh` — `packages.lock` の `.deb` を `cache/` へ取得し sha256 照合。
  `--verify-only` で閉包充足チェックのみ。
- `stage.sh` — `.deb` 展開 → `<STAGE>/lib64`（ld-linux）・`/usr/lib/x86_64-linux-gnu`
  へ**実体解決コピー**、`SONAME` 別名も張る、`copyright` を `LICENSES/` へ収集、
  `closure.txt` の全 soname が揃うか自己検査。
- `Makefile` — `resolve` / `fetch` / `stage` / `locale` / `verify` / `clean`。
- `packages.lock`（**コミット**）: **93 パッケージ**、`.deb` 合計 63 MiB（圧縮）。
  suite=trixie / snapshot=`20250901T000000Z` にピン。
- `closure.txt`（**コミット**）: **111 soname、未解決 0**。
- 実測（`make -C Vendor/LinuxRuntime stage`＋`locale`、キャッシュからオフライン）:
  ステージ合計 **232 MiB**、うち `libLLVM.so.19.1` 124 MiB ＋
  `libgallium-25.0.7-2.so` 41 MiB ＋ `libz3.so.4` 27 MiB ＝ **192 MiB が Mesa/LLVM
  由来**（`chrome` の `DT_NEEDED libgbm.so.1` → Debian `libgbm1` → `libgallium`
  → `libLLVM` の連鎖）。`ld-linux-x86-64.so.2`（225 KiB, glibc 2.41-12）と
  `chrome` の直接 `DT_NEEDED` 32 本はすべてステージ済み。`locale-archive` 生成 OK。

**§7-1 具体化 → (a) で確定（ユーザー承認 2026-08-29）**: 上記 192 MiB の LLVM
スタックは `chrome` の `libgbm.so.1` 依存に引きずられて **ld.so がプロセス起動時に
必ずロードする**（`--headless --disable-gpu` でも `DT_NEEDED` は消えない）。
(b) 最小 `libgbm.so.1` / (c) `headless_shell` 自前ビルド も選択肢だったが、
**このまま同梱**（イメージ +230 MiB を許容）で `packages.lock` を確定。

**トップ `Makefile`**（レシピ本体は書き換えず、委譲＋無条件フックのみ）
- `linux_runtime_stage` ターゲット追加（x86_64 のみ、arm64 は no-op）。
- `install_payload` / `image_livecd` の依存に `linux_runtime_stage` を無条件追加。
- `install_payload`: `$(LINUX_RUNTIME_STAGE)/{lib64,usr}` を payload root へコピー、
  FAT パーティションへ `mcopy -s ... ::/`。`image_livecd`: `IMAGE_STAGE_DIR` へ
  `cp -a`（xorriso が拾う）。トグルは無し（§7-3）。
- `INSTALL_DISK_IMAGE_SIZE_MB` 256 → **2560**。
- `.gitignore`: `Vendor/LinuxRuntime/{cache,.work,LICENSES}/`。

**カーネル小改修**（`make kernel` 通過、`-Werror` は未確認）
- `ProcessManager_Create.c`: 外来 Linux ABI プロセスの既定 envp を
  `PT_INTERP` パスで分岐（`.../ld-linux-...` なら glibc 用
  `LD_LIBRARY_PATH=/lib64:/usr/lib/x86_64-linux-gnu:/usr/lib` ＋ `LANG=LC_ALL=C.UTF-8`
  ＋ `TZ=UTC`、`LD_PRELOAD` 撤去。in-tree テスト ld.so 経路は従来値を維持）。
- `Kernel/include/kernel/config.h`: `KERNEL_COW_FORK` 既定 `0` → **`1`**
  （multiprocess chrome のため。`-DKERNEL_COW_FORK=0` で従来の eager copy に戻せる。
  **QEMU 実起動での回帰確認は未**）。
- `EtcFS.c`: `/etc/ld.so.conf` に `/usr/lib/x86_64-linux-gnu` を追加、
  `/etc/machine-id` を静的追加。

**未着手**: G3 の interp 窓オーバーラップ実測、G4 の残り、G8 起動検証（QEMU 不可）、
G9 syscall ギャップ、G10 ライセンス索引の `Docs/OSS_License/` への配置、
`image`（FAT）フルビルドの通し確認、`-Werror` ビルド。

### 2026-08-29 セッション2（初回 QEMU 起動 → 初期スタック 16B アラインメント修正）

ユーザーが QEMU 起動。`chrome`（pid 4、親=Chromium ランチャ pid 3）が
**#GP（vector 0x0D, error 0）** で panic。RIP `0x000000407801CFE7` は
**glibc 動的リンカ内**（`interp_base 0x40_7800_0000` + `0x1CFE7`）。フォルト命令
バイト列 `... 0F 29 55 80 ...` ＝ `movaps %xmm2, -0x80(%rbp)`、`%rbp=0x47FFFFFDD8`
→ `-0x80` が `0x47FFFFFD58`（`&0xF==8`）で 16B 非整列 → `movaps` が #GP。

**原因**: `initialize_elf_user_stack_ex`（`ProcessManager_Create.c:1375-1378`）の
初期スタック構築が、SysV x86-64 ABI が要求する「エントリ時 `%rsp`（argc を指す）
16B 整列」を満たしていなかった。`sp -= 8ULL` の固定補正が、積むワード数の
パリティが特定の値である前提で、`argc + envc` が偶数のとき（Chromium は
`argc=1` ＋ env 5 個）に `%rsp ≡ 8 (mod 16)` を生む。glibc の `_start` /
ld.so `RTLD_START` は `%rsp` を再整列せず即 xmm を `-0x80(%rbp)` にスピルする
ため、動的リンカが最初の `movaps` で落ちる（静的 musl BusyBox が耐えていたのは
musl `_start` が自前で `and $-16,%rsp` するため）。

**修正**: `sp -= 8ULL` を廃止し、実ブロックサイズ
（`1 + argc + 1 + envc + 1 + auxv_words + 1` ワード）を引いた後に
`sp &= ~0xFULL` で 16B に切り下げ。`make kernel` 通過（新規警告なし）。
次の QEMU 起動で ld.so → `libc.so.6` → `chrome _start` → `main` へ進めるはず。

**方針確認（§7-1）**: LLVM スタック 192 MiB を含めて **このまま同梱で確定**
（ユーザー承認 2026-08-29）。`packages.lock` / `closure.txt` は現状のまま。

### 2026-08-29 セッション3（セッション2 修正の 2 件の回帰を是正）

ユーザー報告: Userland 到達前の強制再起動、kernel 初期化中の
`__stack_chk_fail` BSOD、「サービスを開始中です」画面で **`com.ImplusOS.windowmanager`
（ネイティブ ELF、pid 2）が #GP**。#GP の RIP `0x0000004000_047A9`
（`USER_CODE_BASE + 0x47A9`）、フォルト命令 `66 0F 6F 84 24 …` ＝
`movdqa [rsp+0x130], xmm0`、`User RSP=0x47FFFFF9C8`（`&0xF==8` で 16B 非整列）。

**回帰 1 — 初期スタックアラインmeントを Linux/native で分けていなかった。**
セッション2 で `initialize_elf_user_stack_ex` を「常に 16B 整列」にしたが、
ネイティブ ImplusOS アプリの `_start` は asm crt の無い素の
`void _start(void)` で、GCC は「CALL された」前提＝**エントリ `%rsp % 16 == 8`**
を仮定してプロローグを吐く。16B 整列（`==0`）で渡すとフレームが 8 ズレ、
ネイティブアプリが最初の整列 SSE アクセス（`movdqa`/`movaps`）で #GP。
一方 glibc の `_start`/ld.so `RTLD_START` は `%rsp % 16 == 0` を要求する
（両者は排他）。
→ **修正**: ブロック確保後 `sp &= ~0xF` し、`image_info->linux_abi` が偽の
ときだけ `sp -= 8`。Linux ABI は `==0`、ネイティブは `==8` で渡す。歴史的な
無条件 `sp -= 8` はネイティブには正しく Linux には誤りだった。

**回帰 2 — `KERNEL_COW_FORK` 既定 `1` が起動を不安定化。** 既定を **`0` へ戻した**
（`config.h`）。Userland 到達前の断続的トリプルフォルト再起動と kernel 初期化中の
`__stack_chk_fail` は COW（PTE エイリアス＋SMP TLB＋物理アロケータ同時変更、
QEMU 未検証）の症状。multiprocess Chromium は COW の専用 QEMU 回帰パス後に
再度有効化するか、ゲスト RAM を増やして eager copy で回す（G7 を更新）。

`make kernel` 再ビルド通過。要再起動確認。

### 2026-08-29 セッション4（`libdl.so.2: Error 95` → `newfstatat(AT_EMPTY_PATH)` 実装）

セッション3後の再起動でアラインメント #GP は解消。ld.so が実行を開始し、
`chrome: error while loading shared libraries: libdl.so.2: cannot open shared
object file: Error 95` で停止。Error 95 ＝ `LINUX_ENOTSUP`（`OS_STATUS_NOT_SUPPORTED`）。

**原因**: glibc の動的リンカは開いた共有オブジェクトを
`fstatat(fd, "", &st, AT_EMPTY_PATH)`（`newfstatat`、syscall 262）で fstat する。
ImplusOS の `newfstatat` ハンドラ（`Syscall_LinuxCompat.c:3202`）は
`dirfd != AT_FDCWD` を無条件で `LINUX_ENOTSUP` にしていたため、**あらゆる `.so`
のロードが失敗**（`libdl.so.2` が最初に踏むだけ）。`statx`(291) 側は既に
`AT_EMPTY_PATH` を処理していたが、rtld は `newfstatat` 経路を使っていた。

**修正**（`Syscall_LinuxCompat.c`）:
- `LINUX_SYS_NEWFSTATAT`: `flags & AT_EMPTY_PATH` かつ空パスなら
  `linux_stat_fd((int32_t)dirfd, statbuf)` に委譲（`LINUX_SYS_FSTAT` と同じ経路）。
  絶対パスなら dirfd を無視して解決。相対パス＋非 CWD dirfd のみ従来どおり `ENOTSUP`。
- `LINUX_SYS_OPENAT`: 同様に、**絶対パスなら dirfd を無視**（POSIX 準拠）。
  相対パス＋非 AT_FDCWD のみ `ENOTSUP`。

`make kernel` 通過。次の再起動で ld.so → 依存 `.so` 閉包 → `chrome main` へ。
以降の `ENOTSUP`/`ENOSYS` は `-DLINUX_SYSCALL_TRACE` でトレースして G9 で潰す。

### 2026-08-29 セッション5（`failed to map segment` → トレース配線）

セッション4後の再起動で `libdl.so.2` の fstat は通過し、次は
`chrome: error while loading shared libraries: libdl.so.2: failed to map
segment from shared object`（glibc `_dl_map_segments` の per-segment
`mmap(MAP_FIXED|MAP_COPY|MAP_FILE)` が `MAP_FAILED`）。

**トレース配線を修正**: `make ... EXTRA_CFLAGS=-DLINUX_SYSCALL_TRACE` は
トップ Makefile の `kernel:` が値を下へ渡しておらず、かつ `arch.mk` が
参照していなかったため **無効だった**。
- `Kernel/config/arch.mk`: `KERNEL_CFLAGS += $(EXTRA_KERNEL_CFLAGS)`。
- トップ `Makefile` `kernel:`: `EXTRA_KERNEL_CFLAGS="..."` を `-C Kernel` へ伝播。
- 使い方: `rm -rf Build/x86_64/Kernel && make image_livecd
  EXTRA_KERNEL_CFLAGS=-DLINUX_SYSCALL_TRACE && make run_uefi_usb`。
  シリアルに `[lx] #<syscall> (a1,..,a6)` と `[lx] #<syscall> = <ret>`
  （`<ret>` が巨大な符号なし値なら負 errno）。`mmap` は `#9`。

**mmap 失敗の第一候補（要トレース確認）**: `process_user_alloc`
（`process_user_mmap` の実体、`ProcessManager_Create.c:1548`）は返り値を
**16 バイト境界**にしか整列しない。Linux の `mmap` は常にページ境界を返す
契約で、glibc は `l->l_addr = map_start - loadcmds[0].mapstart` を計算して
以降のセグメントを `mmap(l->l_addr + c->mapstart, …, MAP_FIXED, …)` で貼る。
非ページ境界の `map_start` だと後続 FIXED アドレスも非整列になり失敗し得る
（`linux_mmap` の FIXED 分岐のアドレス検査は
`length > 0x4000000000 - addr` が addr>コード基底のとき符号なしアンダー
フローして素通りするので、非整列を弾けていない点も別途要修正）。
トレースで `#9` の引数（特に非 FIXED 初回 mmap の返りアドレスと、続く
FIXED mmap のアドレス／`= <ret>`）を確認して確定する。

### 2026-08-29 セッション6（`failed to map segment` の真因＝syscall 第5引数マーシャリングバグ）

トレース取得（`EXTRA_KERNEL_CFLAGS=-DLINUX_SYSCALL_TRACE`）で確定:

```
#257 openat(AT_FDCWD, ".../libdl.so.2", O_CLOEXEC) = 3
#0   read(3, buf, 0x340) = 0x340
#5   fstat(3, ...) = 0
#9   mmap(NULL, 0x2000, RW,  MAP_PRIVATE|ANON,   fd=0,    0) = 0x4100000000   ← glibc は fd=-1 を渡している
#9   mmap(NULL, 0x4010, R,   MAP_PRIVATE|DENYWR, fd=0x24, 0) = -9 (EBADF)      ← glibc は fd=3 を渡している
#3   close(3)
#20  writev(2, iov, 10)   ← "failed to map segment" をstderrへ
```

`mmap` の**第5引数 (fd) がカーネルに化けて届いていた**（anon 版で `0`＝
本来 `-1`、file 版で `0x24`＝本来 `3`）。→ `syscall_file_get_file_info(化けたfd)`
が失敗し `EBADF` → glibc が "failed to map segment"。

**真因**: `Kernel/Arch/x86_64/cpu/Syscall_Entry.asm`。第5・第6引数 (a5,a6) を
スタック渡しで C の `syscall_dispatch(...,a5,a6)` に渡す際、**16B アラインの
`sub rsp, 8` を 2 つの `push` の"後"に置いていた**ため、スタック引数が 8B
ずれて、`syscall_dispatch` は a5 をパディング領域（ゴミ）から、a6 を a5 の
スロットから読んでいた。ネイティブ ABI 経路は `syscall_dispatch` が
`(void)arg5;` で a5 を実質無視していたため露見せず（ただし
`process_create_thread` の第5引数もゴミだった）。Linux `mmap`/`pread` が
a5 を必要とする初の呼び出し元。

**修正**: `sub rsp, 8`（パッド）を 2 つの `push` の"前"へ移動し、
`push a6` → `push a5` の順にして、呼び出し先が a5 を `[rsp+8]`、a6 を
`[rsp+16]` で正しく読めるようにした。a1..a4（レジスタ渡し）は不変。
`nasm` アセンブル＋リンク通過。

次の再起動で `mmap(fd=3)` が通り、`libdl.so.2` → 依存 `.so` 閉包 →
`chrome` 本体エントリへ進むはず。

### 2026-08-29 セッション7（`mmap` 修正で ld.so 前進、次は「全ライブラリが libdl に解決される」）

セッション6 の `Syscall_Entry.asm` 修正で `mmap(fd=3)` が実アドレスを返し、
**`libdl.so.2` は 4 セグメントとも正しくマップされた**（`l_addr=0x4100002000`、
FIXED セグメントが `+0x1000/+0x2000/+0x3000`、`pread64` も動作）。

次の症状: `chrome` の他の全 `DT_NEEDED`（`libasound`/`libnss3`/`libdbus-1`/
`libudev`/`libgcc_s`/`libc` …）の **バージョンが「`libdl.so.2` が提供元」**として
"not found" になる（`ALSA_0.9` `NSS_3.2` `LIBDBUS_1_3` `GLIBC_2.36`
`GLIBC_ABI_DT_RELR` 等、40 件以上）。これは glibc の
`_dl_check_map_versions` が **chrome の各 `Verneed` 名（"libasound.so.2" 等）を
すべて `libdl.so.2` の link_map に解決している**＝`openat("…/libXXX.so")` で
`libdl.so.2` の中身が返ってきて `open_verify` が soname "libdl.so.2" を読み、
既ロード分に dedup している、という状態。

**イメージ（ISO）は正常**: ホストで `xorriso -R -J` 生成 ISO を作り直して検証
→ `/usr/lib/x86_64-linux-gnu/` に 111 ファイル、名前・サイズ・SONAME すべて正
（`libpthread.so.0`=14408B, `libc.so.6`=2003408B, `libasound.so.2`=1178192B）。
→ **バグは ImplusOS の ISO9660 読み取り（大ディレクトリのルックアップ or
ファイル内容の読み）側**。`libdl.so.2` は小さく単純なので通り、他が化ける。

**デバッグ追加**（`-DLINUX_SYSCALL_TRACE` 時のみ）: `linux_open_path` が
解決後のパス文字列・返り fd・先頭 8 バイト（ELF マジック確認用）を
`[lx] open '<path>' -> <fd> hdr[n]=7F 45 4C 46 …` の形でシリアル出力。
次回ログでどのパスがどのファイル（マジック `7F 45 4C 46`＝ELF か、化けてるか）
を開いているか確定する。`libc.so.6` を開いた行の hdr を特に確認。

### 2026-08-29 セッション8（真因＝`stat.st_ino` が全ファイル固定値 → glibc の inode dedup 誤爆）

パスログで確定: **全ライブラリが正しいパス・正しい ELF マジック
（`7F 45 4C 46 …`）で open されている**（ISO 読みは正常）。しかし各ライブラリの
syscall 列は `open → read(ヘッダ) → fstat → close` で **`mmap` が無い**
＝実際にはロードされていない。マップされたのは `libdl.so.2` だけ。

**真因**: `linux_stat_fill_common`（`Syscall_LinuxCompat.c`）が **全ファイルに
`st_ino = 1`（定数）** を返していた。glibc の ld.so は
`_dl_map_object_from_fd` で「既ロードの共有オブジェクトか」を
**`(st_dev, st_ino)` の一致**で判定する。全ファイルが `(0x8200, 1)` になるため、
2 本目以降の `.so` はすべて「`libdl.so.2` と同一 inode」＝既ロード扱いで
fd を閉じて `libdl` の link_map を再利用 → chrome の全 `Verneed` 名が
`libdl.so.2` に解決 → 全バージョン not found。

**修正**:
- `Kernel/Drivers/Module/ISO9660_VFS_Bridge.c`: `vfs_file_t.internal_id` を
  malloc ポインタ → **ISO9660 の extent LBA**（ファイルごとに一意・open を
  跨いで安定）に変更。`driver_data` は従来どおり構造体ポインタ。
- `Kernel/Compat/Linux/Syscall_LinuxCompat.c`: `linux_stat_fill_common_ino(st,
  size, ino)` を新設し `st_ino = ino ? ino : 1`。`fstat`/`stat`/`statx`/
  `newfstatat` の実ファイル経路（`vfs_file_t` を持つ 4 箇所）で
  `vf.internal_id` を渡す。ディレクトリ／ソケット経路は従来の 1。

`make kernel` 通過。次の再起動で 2 本目以降の `.so` も `mmap` され、
`chrome` 本体エントリへ進むはず。

### 2026-08-29 セッション9（st_ino 修正で全 .so ロード成功 → 次は chrome の CHECK 失敗 = 疑似的な #GP）

セッション8 の `st_ino` 修正が的中。**`chrome` の `DT_NEEDED` 閉包（約 35 本）が
すべて個別に `mmap` され**、バージョンエラーは消滅。ld.so を完全に抜けて
**chrome 自身のコード**（`0x400C3xxxxx`＝`USER_CODE_BASE + chrome .text`）へ到達。

次のパニック: **#GP vector 0x0D, error `0x1A`**。`0x1A = (3<<3)|2` は
「CPL 3 から IDT vector 3 を参照」＝**userland が `int3` を実行**したことを示す。
`rip bytes` が `… C3 CC 0F 0B`（`ret; int3; ud2`）＝ Chromium の
`IMMEDIATE_CRASH()` / 失敗した `CHECK()`。ImplusOS の IDT は全ゲート DPL=0
なので userland `int3` が `#GP` になり、**プロセスではなくカーネルごと panic**
していた。

**修正**（`Kernel/Arch/x86_64/cpu/IDT_Main.c` の `general_protection_fault_handler`）:
`rsp` が指す例外フレームの CS（`frame[2]`）を見て CPL 3 由来の #GP なら、
`process_debug_dump_pid` → `process_exit_current_signaled(SIGILL=4)` →
`process_run_next_on_current_cpu()` ループで **その userland プロセスだけを
終了**しカーネルは継続（PF ハンドラの user 経路と同じ作法）。カーネル由来の
#GP は従来どおり `panic_exception`。

これで chrome の CHECK 失敗はカーネルを巻き込まず、chrome が `int3` 直前に
stderr へ出す `[FATAL:…] Check failed: …` がシリアルに残るはず。次ログで
その FATAL 行を確認 → 原因（sandbox / `/proc` / 未実装 syscall 等）を特定して
G9 で潰す。`make kernel` 通過。

### 2026-08-29 セッション10（`[FATAL:partition_address_space.cc(82)]` → 遅延コミット mmap VM 改修）

セッション9 の #GP 生存化で FATAL が読めた:
`[FATAL:partition_address_space.cc(82)] Check failed: false.`
直前トレース: PartitionAlloc が `mmap(hint, 0x1000F0000≈4GiB, PROT_NONE, ANON)`
と `mmap(hint, 0x800000000=32GiB, PROT_NONE, ANON)` を投げて両方 `ENOMEM`。
ImplusOS の mmap は (a) `process_user_alloc(uint32_t)` で **4 GiB 上限**、
(b) 匿名 mmap も **全ページを即座に物理割当＋ゼロ埋め**（PROT_NONE 予約が不可）、
(c) ユーザ空間が固定 32 GiB 窓で PartitionAlloc のプール（計 ~48 GiB）が入らない。

**独立作業 1/2/3 として実装（`make kernel` 通過）**:

**(2) 4 GiB 上限撤廃**: `user_alloc_t.size` `uint32_t`→`uint64_t`、
`process_user_alloc(uint32_t)`→`(uint64_t)`、`aligned_len > UINT32_MAX`
チェック削除、関連キャスト整理（`ProcessScheduler.h` / `ProcessManager_Create.c`）。

**(3) 大きな mmap アリーナ**: `USER_MMAP_BASE=0x100_0000_0000`(1 TiB)〜
`USER_MMAP_LIMIT=0x1000_0000_0000`(16 TiB) を新設（`ProcessManager.h`）。
既存の code/heap/stack は PML4[0]（<512 GiB）、アリーナは PML4[2..31] で非衝突。
`process_t.user_mmap_cursor`（bump、fork でコピー、thread は owner 共有）。
`is_user_virtual_address()` にアリーナを追加（`Paging_Main.c`）。

**(1) 遅延コミット匿名 mmap**:
- `process_user_reserve(uint64_t length)` 新設 — アリーナから bump で
  ページ整列アドレスを返すが **物理割当は一切しない**（`ProcessManager_Create.c`）。
- `linux_mmap` の匿名経路: 非 FIXED は `process_user_reserve`、FIXED も
  アドレスがアリーナ内なら即受理してマップしない（`Syscall_LinuxCompat.c`）。
- `page_fault_handler`: 従来 `PF_USER` のフォルトのみ demand-zero していたのを、
  **ユーザアドレスへのカーネルモードフォルトも** `paging_handle_swap_fault`
  に通す（`copy_to/from_user` の `memcpy` が遅延ページを踏んでも panic せず
  ゼロページを供給）。`paging_handle_swap_fault` は非ユーザアドレスで 0 を返す
  ので本物のカーネル野良ポインタは従来どおり panic（`IDT_Main.c`）。
- `process_user_buffer_is_valid`: アリーナ範囲内なら**常在チェック無しで有効**
  （遅延ページは #PF で埋まる）。
- `paging_protect_user_range`（=`mprotect`）: ユーザアドレスなら不在ページを
  エラーにせず読み飛ばす（PROT_NONE 予約→`mprotect(RW)` commit を許容）。
  不在の PML4/PDPT/PD は 512 GiB/1 GiB/2 MiB 単位でスキップ。
- `paging_unmap_range` / `mremap`: 未マップ 1 GiB/2 MiB を一括スキップ、
  `mremap` の移動先は `process_user_reserve`、移動元は `process_user_munmap`
  ＋`process_user_free` で物理解放。

**既知の割り切り**: アリーナは bump のみ（アドレス空間は再利用せず、16 TiB
枠で十分の想定）。VMA 追跡が無いためアリーナ内の不正ポインタは EFAULT に
ならずゼロページが供給される（真の修正には VMA 木が必要）。demand-zero ページは
RWX（V8 JIT は動くが W^X ではない）。`/proc/self/maps` はアリーナ領域を列挙
しない。

次の再起動で PartitionAlloc のプール予約が通り、chrome 起動がさらに前進する
見込み。

### 2026-08-29 セッション11（VM 改修後の前進確認 → chrome サブプロセスの `abort()` で特権 `hlt` → SIG_DFL 致命シグナル自己終了修正 + QEMU/KVM 起動高速化）

セッション10 の VM 改修後トレース: PartitionAlloc のプール `mmap` が全て成功し、
chrome が Chromium スレッドプール起動まで到達（FATAL ログ無し）。その先で
`#GP err=0x0`（セレクタ非依存）@ `libc.so.6 + 0x28507`、プロセス名は
（fork 子ゆえ）`Userland.ELF` 表示。

**原因**: staged `libc.so.6` を逆アセンブルすると `libc+0x28507` は `abort()`
内の特権 `hlt`。chrome が `abort()` を呼ぶ → `raise(SIGABRT)` → ImplusOS の
シグナル配送は SIG_DFL の致命シグナルでプロセスを終了させていなかったため
`abort()` が生き残り、glibc の最終手段 `hlt`（特権命令）に落ちて #GP。
セッション9 の `general_protection_fault_handler` がそれを user-mode #GP として
回収し、プロセスは死ぬが「chrome が自分で終了した」形にならない。

**修正（`make kernel` 通過、400712 バイト）**:
- `process_signal_default_terminates(signum)` / `process_signal_maybe_self_terminate(signum)`
  を新設（`ProcessManager_Create.c`、`ProcessManager.h` に宣言）。
  致命シグナル既定集合（SIGHUP/INT/QUIT/ILL/TRAP/ABRT/BUS/FPE/KILL/SEGV/PIPE/
  ALRM/TERM/XCPU/XFSZ/VTALRM/PROF/SYS 等）で、対象プロセスにユーザ
  ハンドラが未登録なら `process_exit_current_signaled(signum)` を呼ぶ。
- `Syscall_LinuxCompat.c` の KILL / TKILL / TGKILL 経路: 配送成功かつ
  対象が自分自身なら `process_signal_maybe_self_terminate()` → `request_switch = 1`。
  これで `abort()` の `tgkill(self, SIGABRT)` が `hlt` に到達する前に
  プロセスが正規終了する。

**起動高速化（`Makefile`）**:
- `QEMU_ACCEL` を新設。ホストに使用可能な `/dev/kvm` があれば自動で
  `-enable-kvm` を付与（`QEMU_ACCEL=` で無効化可）。`QEMU_COMMON` に配線。
- CPU モデルは QEMU 既定（`qemu64`: SSE2 のみ、AVX / XSAVE / FSGSBASE 無し）
  のまま。カーネルが `CR4.OSXSAVE` / `CR4.FSGSBASE` を立てていないため
  `-cpu host` は付けない（glibc が未対応命令パスを選ぶと #GP になる）。
- 併せて運用メモ: `-DLINUX_SYSCALL_TRACE` はシリアル 115200 baud に
  syscall 毎に出力するため chrome では致命的に遅い。原因調査が済んだら
  外してビルドすること（`rm -rf Build/x86_64/Kernel` が必要）。

**次の一手**: `-DLINUX_SYSCALL_TRACE` 無しのシリアルログを取得し、chrome が
`abort()` を呼ぶ直前の `[FATAL:...]` / `chrome: ...` 行を確認する。
SIG_DFL 修正はビルド済み・未起動テスト。

### 2026-08-29 セッション12（Kernel 初期化後の「再起動ループ」＝カーネルスタック溢れによるヒープ破壊の緩和）

症状: Kernel 初期化完了後、たまに再起動を繰り返すループに入る（加えて過去に
Kernel 初期化中の `__stack_chk_fail` BSOD も）。`kernel_panic` /
`panic_exception` はどちらも `while(1) hlt` で止まるだけ（再起動しない）ので、
「再起動」＝**トリプルフォルト**（フォルト配送中の二重フォルト中にさらに
フォルト → CPU リセット、QEMU 既定で VM リセット）。

**根本原因（推定）**: プロセスごとのカーネルスタックは `malloc(32 KiB)` で
**ガードページ無し**。Linux 互換層＋Chromium の深くネストした glibc syscall、
さらにセッション9/10 でフォルトハンドラがこのスタック上でスケジューラ処理・
遅延ページ処理を走らせるようになり 32 KiB を超過。溢れても隣接ヒープは
マップ済みのため即フォルトせず**サイレントにヒープを破壊** → 後で
ページテーブル/関数ポインタ/IST 近傍を踏んで #GP や トリプルフォルト。
「たまに・症状がまちまち」に合致。

**修正**:
- `PROCESS_KERNEL_STACK_SIZE` 32 KiB → **128 KiB**（`ProcessScheduler.h`）。
  AP スタックは 256 KiB、syscall スタックは 64 KiB。実プロセス数は数個なので
  実コストは ~1 MiB。
- **カーネルスタック・カナリア**（`ProcessManager_Create.c`）:
  `process_kstack_arm()` が確保直後にスタック最下位 16 バイトへ番兵
  (`PROCESS_KSTACK_CANARY`) を書き込む。全 4 確保サイト（main / thread /
  fork child、`initialize_process_memory`）に配線。
  `process_kstack_canary_check()` を `activate_process_context()`
  （唯一のコンテキストスイッチ経路、毎回通る）で実行。番兵が壊れていたら
  pid/name を出して `kernel_panic("KSTACK", ...)` で**即・確定的に停止**
  （サイレント破壊 → ランダム再起動を、明確な診断に変換）。
- **#PF ハンドラの再入ガード**（`IDT_Main.c`）: CPU ごとに `g_pf_depth[]`。
  フォルト処理中にさらに #PF が来たら（＝遅延ページ処理自体がフォルト＝
  カーネルバグ）demand-paging を試みずパニック経路へ。デマンドページ試行
  ブロックを抜けたら即カウンタを戻す（プロセス終了でスタックを捨てる経路
  でもカウンタが残らないように）。
- **パニック経路をフォルトしないように**（`IDT_Main.c`）:
  `panic_dump_stack_words` / `panic_dump_stack_trace` が
  `rsp` / フレームポインタ鎖を辿る際、`panic_addr_readable()`
  （`paging_virt_to_phys` でアクティブ CR3 を歩く）で毎回マップ確認してから
  デリファレンス。未マップなら停止。これでパニック中の入れ子フォルト
  （＝二重→三重フォルト＝再起動）を封じる。
- **`Makefile`**: `QEMU_NO_REBOOT ?= -no-reboot` を新設し `QEMU_COMMON` に
  配線。トリプルフォルトで VM がリセット（＝ループ・最後のログが流れる）
  代わりに**停止**して最後のシリアル/画面を残す。`QEMU_NO_REBOOT=` で解除。

`make kernel` 通過（404808 バイト、警告は既存のもののみ）。

**割り切り**: 128 KiB でもなお溢れる深い経路が残る可能性はあるが、その場合は
カナリアが `[OS] [KSTACK] kernel stack overflow pid=... name=...` を出して
確定停止するので、どの経路が深すぎるか特定できる。真の恒久対策は
カーネルスタックを専用アロケータでページ整列＋下にガードページ。

### 2026-08-29 セッション13（ICU FATAL の真因＝`/proc/self/exe` 誤り + Chromium 単一プロセス化 + シリアルログ Viewer アプリ）

ユーザ報告の `[FATAL:base/i18n/icu_util.cc:310] Check failed: result` /
`[ERROR:...icu_util.cc:237] Invalid file descriptor to ICU data received` は
これまで追っていた `abort()` の正体。

**真因**: `/proc/self/exe` の `readlink` が `procfs_readlink()` で
`process_copy_launch_argument()`（＝起動引数、chrome には無かったので空）を
返し、空なら `/Userland/Userland.ELF` にフォールバックしていた。Chromium /
glibc は `readlink("/proc/self/exe")` で自分の**アセットディレクトリ**を
決めるので、`DIR_ASSETS` が `/Userland` になり `icudtl.dat` を
`/Userland/icudtl.dat` に探して見つからず fd=-1 → CHECK → abort。
（実ファイルは `/Userland/Chromium/Resource/icudtl.dat` に同梱済み。）

**修正**:
- `process_t.exe_path[256]` を新設（`ProcessScheduler.h`）。exec 元の絶対パスを
  保持。`process_spawn_user_elf_with_arg` / `process_execve` で設定、fork で
  コピー、`reset_process_slot` でクリア（`ProcessManager_Create.c`）。
- `process_copy_exe_path()` を追加（`ProcessManager.h` に宣言）。
- `procfs_readlink()` の `"exe"` 経路を exe_path 優先に変更（無ければ
  launch_argument → `/Userland/Userland.ELF` の順にフォールバック）。
- **Linux-ABI の argv 生成を空白分割対応に**（`ProcessManager_Create.c`）:
  `launch_argument` を空白で `argv[1..]`（最大 32）へトークン化。空白が
  無ければ従来どおり単一 `argv[1]`。これで Linux バイナリに複数フラグを
  渡せる。
- **`Userland/Application/Chromium/Start.c`**: `process_spawn_with_arg` で
  `--single-process --no-zygote --no-sandbox --disable-gpu
   --disable-dev-shm-usage --no-first-run --disable-features=Vulkan
   --user-data-dir=/tmp/chromium --headless=new --dump-dom about:blank`
  を渡す。単一プロセス化で (a) zygote/子プロセスへの fd 受け渡し
  （ImplusOS 未実装、ICU fd=-1 のもう一つの原因）を回避、
  (b) WSL2/TCG での fork ストーム（glibc 子 5 本が各々 GB 単位を
  demand-page ＋動的リンクやり直し）を解消＝**起動高速化**。

**シリアルログ Viewer アプリ**（ユーザ要望）:
- カーネル: `serial_copy_log()` を包む `SYSCALL_READ_KERNEL_LOG (268)` を新設
  （`Syscall_Main.h` / `libc/.../sys/syscalls.h` / `Syscall_Dispatch.c`）。
  8 KiB の静的ステージバッファ＋spinlock 経由で `copy_to_user`。
- Userland: `read_kernel_log(char*, uint32_t)` を `OSDebug.h` / `Syscalls.c` に。
- 新アプリ `Userland/Application/com.ImplusOS.serialmonitor/`（`SerialMonitor.c`
  ＋ sysnotif 流の Makefile）。ウインドウを作り ~150ms 毎にカーネルログ末尾を
  取得して描画。`f`=末尾追従 / `j``k`=行スクロール / `n``p`=ページ /
  `g`=先頭 / `q`or`Esc`=終了 / マウスホイール対応。PANIC/#GP/#PF/FATAL 行は
  赤、ERROR/WARN/spawn- は黄。
- `apps.list` に
  `Serial Monitor|/Userland/com.ImplusOS.serialmonitor/com.ImplusOS.serialmonitor.ELF|SM`
  を追加。APP_DIRS 自動 glob ＋ image ステージ自動なので Makefile 改修不要。

`make kernel` / `make app_build` 通過（警告は既存のもののみ）。
serialmonitor.ELF 246880 バイト、Chromium.ELF 再リンク済み。

### 2026-08-29 セッション14（ICU 突破後：crashpad の `setsockopt(SO_PASSCRED)` FATAL）

セッション13 の `/proc/self/exe` 修正で ICU を突破。次のブロッカーは crashpad:
`[ERROR:...crashpad/util/linux/socket.cc:45] setsockopt: Protocol not available (92)`
→ `[FATAL:...crashpad_linux.cc:245] Check failed: client.StartHandler(...)`。
ImplusOS の `AF_UNIX` に `SO_PASSCRED` / `SCM_CREDENTIALS` が無く、
`linux_socket_setsockopt` が未知オプションを ENOPROTOOPT で弾いていた。

**修正**:
- `Start.c` に **`--disable-crashpad-for-testing --disable-breakpad`** を追加。
  この switch はこのビルドにコンパイル済み（`crashpad.cc: IsCrashpadEnabled()`
  が参照）。クラッシュハンドラ初期化ごとスキップされ `StartHandler` に到達しない。
- 併せて `linux_socket_setsockopt` / `getsockopt` を寛容化
  （`Syscall_LinuxCompat.c`）:
  - setsockopt: `SO_PASSCRED/PASSSEC/SNDBUF/RCVBUF/BROADCAST/REUSEPORT/LINGER/
    SNDTIMEO/RCVTIMEO` と非 `SOL_SOCKET` レベルを **no-op で 0 返し**。
    未知オプションも ENOPROTOOPT ではなく accept-and-ignore。
  - getsockopt: `SO_PEERCRED`→`ucred{0,0,0}`、`SO_TYPE`→1(SOCK_STREAM)、
    `SO_ERROR`→0、`SO_SNDBUF/RCVBUF`→212992、`SO_PROTOCOL/PASSCRED`→0。
  実 Linux ソフトが setsockopt 失敗を致命扱いするのを避ける一般的措置。

`make kernel` / `make app_build` 通過。次は再起動して Serial Monitor で
crashpad 後の次のログを確認する。

### 2026-08-29 セッション15（`pthread_create: Operation not permitted` ＝ スレッドエントリ検証が狭すぎ）

crashpad / sandbox host を突破。次:
`[ERROR:...platform_thread_posix.cc:162] pthread_create: Operation not permitted (1)`
→ `[FATAL:...simple_thread.cc:59] Check failed: success`。

**真因**: `is_valid_user_entry()`（`ProcessManager_Create.c`）が
`entry < USER_CODE_LIMIT`（0x40_80000000）しか許可していなかった。glibc の
`pthread_create` は `clone(fn=start_thread)` を投げ、`start_thread` は
**libc.so 内**＝ ld.so が mmap アリーナ（`USER_MMAP_BASE` ≒ 1 TiB 以上）へ
マップした領域にある。エントリがそこを指すため
`process_create_thread_ex` が即 `-1` を返し、`linux_clone` がそれを
`-1`（＝ `-EPERM`）として glibc へ返却 → 全 `pthread_create` が EPERM。

**修正**:
- `is_valid_user_entry()`: `[0x1000, USER_CODE_LIMIT)` に加え
  `[USER_MMAP_BASE, USER_MMAP_LIMIT)` も許可。
- `linux_clone`: スレッド作成失敗時の戻り値を `-1`(EPERM) →
  `LINUX_EAGAIN`(-11)。glibc/Chromium が期待する「スレッドが作れない」errno。
  失敗時に `[lx] clone/thread create failed` をシリアル出力（Serial Monitor で
  次の詰まりが見えるように）。

`make kernel` 通過（新規警告なし）。

### 2026-08-29 セッション16（GTK3 / Wayland 外来バイナリの vendoring ＋ テストアプリ配線）

Chromium トラックと同じ基盤の上で **GTK3 / Wayland バイナリ**を動かす作業に着手。
新規詳細は [`TODO_GTK3_Wayland_LinuxABI.md`](TODO_GTK3_Wayland_LinuxABI.md)。要点のみ:

- `Vendor/LinuxRuntime` の閉包を拡張: `packages.seed.txt` に GTK3/Wayland 21 件を
  追加（起点 `gtk-3-examples` の `/usr/bin/gtk3-demo`）。`resolve` 再実行で
  `packages.lock` 93→**119**、`closure.txt` 114→**133 soname・未解決 0**。
  追加された主なもの: `libgtk-3` `libgdk-3` `libgdk_pixbuf-2.0` `libepoxy`
  `libharfbuzz` `libcairo-gobject` `libpangocairo`/`libpangoft2`
  `libwayland-client`/`-cursor`/`-egl` `libXi`/`libXcursor`/`libXinerama`/`libXrender`。
  （glib/gobject/gio・pango・cairo・libX11・libxkbcommon は Chromium 閉包で既存）。
- `stage-gtkdata.sh` 新設（`gtkdata` ターゲット、トップ Makefile から無条件委譲）:
  `.so` 以外の GTK 実行時データを配置 — gtk3-demo 実行体を `/usr/bin` へ、
  GSettings スキーマ 38 本をホスト `glib-compile-schemas` で `gschemas.compiled`
  に、DejaVu フォント + `/etc/fonts/fonts.conf`。gdk-pixbuf `loaders.cache` は
  ホストにツールが無く未生成（G4 で対応、画像は出ないがウィンドウは出る想定）。
- `Kernel/Core/process/ProcessManager_Create.c` の `glibc_envp` に
  `HOME` / `XDG_RUNTIME_DIR=/tmp` ほか XDG / `GDK_BACKEND=wayland,x11` /
  `GSETTINGS_SCHEMA_DIR` / `GSETTINGS_BACKEND=memory` / `FONTCONFIG_*` を追加。
  ネイティブ経路不変、Chromium にも無害。
- ランチャ `Userland/Application/com.ImplusOS.gtk3demo/`（`/usr/bin/gtk3-demo` を
  `process_spawn`）。`apps.list` に `GTK3 Demo`。
- `make kernel` / `make app_build` 通過。`make image_livecd
  EXTRA_KERNEL_CFLAGS=-DLINUX_SYSCALL_TRACE` 完走（ISO 内 `/usr/bin/gtk3-demo`
  確認済み、ステージ 232→254 MiB）。

**検証（この環境でできる範囲）**: ステージした**実体のみ**（同梱 ld.so +
`--library-path` に同梱 `.so` 閉包）で `gtk3-demo` をホスト実行 →
推移的 NEEDED 63 本すべて解決し、`gtk_init` を通って
`Gtk-WARNING: cannot open display:` で終了コード 1。`gtk3-demo` の最大
glibc 要求 `GLIBC_2.38` ≤ 同梱 `2.41-12`。**＝ glibc 動的リンク + GTK3
ランタイム同梱は成立**。実描画は Wayland コンポジタ未実装のため次フェーズ
（`TODO_GTK3_Wayland_LinuxABI.md` G3）。

**未着手（ユーザー QEMU 検証待ち）**: ランチャから `gtk3-demo` を spawn し
`-DLINUX_SYSCALL_TRACE` のシリアル/Serial Monitor で、`gdk_display_open`
失敗まで到達するか手前で `ENOSYS`/`ENOTSUP` が出るかを確認。出た syscall を
`TODO_GTK3_Wayland_LinuxABI.md` G6 に記録。

### 2026-08-29 セッション17（gtk3-demo 初回 QEMU 起動 → スレッド生成が全滅する `is_valid_user_entry` の範囲バグを修正）

ユーザーが QEMU 起動。トレース取得（`-DLINUX_SYSCALL_TRACE`）:

- **GTK3/Wayland/X の `.so` 閉包 約60本すべてが `/usr/lib/x86_64-linux-gnu/` から
  open→mmap 成功**（`libgtk-3`/`libgdk-3`/`libwayland-client`/`-cursor`/`-egl`/
  `libepoxy`/`libharfbuzz`/`libpangocairo`/`libpangoft2`/`libcairo-gobject`/
  `libgdk_pixbuf`/… 全部）。`locale-archive` も mmap 成功。動的リンク・再配置・
  glibc 初期化を完走。ホスト素振り（`cannot open display`）より深く、GLib の
  スレッドプール起動まで到達。
- そこで `GLib-ERROR: creating thread 'pool-spawner': Error creating thread:
  Resource temporarily unavailable` → `g_error()` → `abort()` →
  user-mode #GP（`err=0x1A`、rip は libglib 内）。カーネルは
  セッション9 の #GP 生存化でプロセスのみ終了、巻き込まれず。
- 直前トレース: `#435 clone3 = -38(ENOSYS)`（glibc は `clone` にフォールバック、
  正常）→ `#56 clone = -11(EAGAIN)` ＋ カーネルが `[lx] clone/thread create failed`。

**真因**: `is_valid_user_entry()`（`ProcessManager_Create.c`）が
スレッドエントリを `[0x1000, USER_CODE_LIMIT)`（`0x40_8000_0000` 未満）または
mmap アリーナ `[USER_MMAP_BASE, USER_MMAP_LIMIT)`（≥1 TiB）しか許可していなかった。
**セッション15 は「ld.so が libc をアリーナへ mmap する」前提だったが、実際の
gtk3-demo では `linux_mmap` のファイル mmap 経路が `process_user_mmap` を使い、
`.so` 閉包を全部ヒープ窓（`0x41_xxxx_xxxx`）へ置く**。`libc.so.6` が
`0x4101220000` にマップされ、glibc の `start_thread` エントリがそこを指すため、
どちらの許可レンジにも入らず `process_create_thread_ex` が即 `-1`、
`linux_clone` が `-EAGAIN` を返す。→ GLib の必須 pool-spawner スレッドが
作れず致命化。

**修正**: `is_valid_user_entry()` の低位レンジを
`[0x1000, USER_CODE_LIMIT)` → **`[0x1000, USER_STACK_BASE)`**（`0x47E0_0000_00`
未満、＝コード＋ヒープ窓を全部カバー、スタック域とカーネル空間は除外）に拡大。
アリーナ側 `[USER_MMAP_BASE, USER_MMAP_LIMIT)` は据え置き。

`make kernel` 通過（新規警告なし、404352 バイト）。`make image_livecd
EXTRA_KERNEL_CFLAGS=-DLINUX_SYSCALL_TRACE` 再ビルド。次の起動で pool-spawner
スレッドが立ち、`gtk_init` → `gdk_display_open` 失敗（`cannot open display`）
まで進むはず。次のブロッカー候補（G6）: `rt_sigaction(sig 32/33)` が
`signum >= 32` で `EINVAL`（glibc NPTL の SIGCANCEL/SIGSETXID。今回は glibc が
許容して継続。恒久対応は `OS_CONFIG_SIGNAL_HANDLER_MAX_PER_PROCESS` を
33+ へ拡張してから受理）、`clone3` 実装、`/usr/share/zoneinfo/UTC` 不在。

### 2026-08-29 セッション18（`is_valid_user_entry` 修正で前進 → `ppoll` 未実装で GLib メインループが暴走。poll/ppoll 実装）

セッション17 の修正で `clone`（スレッド生成）が tid を返すようになり、
GLib の pool-spawner スレッドが立って先へ進行。次の症状は **「重い・遅い、
ただしクラッシュ無し」**。トレースで確定:

```
#271 ppoll(...) = -38 (ENOSYS)
(gtk3-demo): GLib-WARNING **: poll(2) failed due to: Function not implemented.
```

`ppoll`(271) と `poll`(7) が未実装で `ENOSYS` を返していた。GLib の
`GMainContext` はイベントループを `ppoll` で回すので、`ENOSYS` を食うと
**ブロックできずビジーループ**に落ち、毎周回この警告を 115200 baud の
シリアルへ吐く。この出力自体が「遅さ」の正体（プロセスは正常動作）。

**修正**:
- `Kernel/Core/syscall/Syscall_Epoll.c` / `.h`: 単一 fd の準備状態を返す
  公開ヘルパ `syscall_poll_one_fd(fd, events)` を追加（内部 `epoll_poll_fd`
  をそのまま公開。regular/pipe/timerfd/memfd/signalfd/socket/eventfd を
  網羅。EPOLL* ビット＝POLL* ビット）。
- `Kernel/Compat/Linux/Syscall_LinuxCompat.c`: `linux_poll_common` /
  `linux_ppoll` を実装し `LINUX_SYS_POLL`(7) / `LINUX_SYS_PPOLL`(271) を
  dispatch に配線。pollfd 集合を 1 回だけノンブロッキング走査 → ready が
  あれば `revents` を書き戻して件数を返す。無くて timeout≠0 なら
  `process_sleep_current_ms(8ms)` で 1 スライスだけ park して 0 を返す
  （`syscall_epoll_wait_ex` と同じ「ブロックは timed poll へ縮退」方式。
  この経路は内部で本当のブロックができないため）。上限 256 fd。

これで GLib メインループは `ENOSYS` 警告を出さず、アイドル時は ~8ms 間隔で
tick する（100% スピンしない）。

**あわせて `-DLINUX_SYSCALL_TRACE` を外してイメージを再ビルド**。`ppoll` が
機能すると GLib の tick 毎に数十 syscall が走り、トレース出力（syscall 毎に
シリアル出力）が今度は律速になるため。デバッグ時のみ
`rm -rf Build/x86_64/Kernel && make image_livecd
EXTRA_KERNEL_CFLAGS=-DLINUX_SYSCALL_TRACE` で復活。

`make kernel` 通過（404352 バイト、新規警告なし）。次の起動で GLib が
静かに回り、`gtk_init` → `gdk_display_open` 失敗（`cannot open display`、
コンポジタ未実装）まで到達するはず。

### 2026-08-29 セッション19（スレッド生成の SMP レース＝子スレッドが RIP=0 へ飛ぶ #PF を修正）

セッション18 後の起動で `poll`/`ppoll` は機能し警告スパムは消滅。次は
**間欠的な #PF**:

```
[PF] CR2: 0x0  RIP: 0x0  RBP: 0x0  Error: 0x15 (read, user, exec)
[PF] Terminating process pid=5 (gtk3-demo)   ← kf_rax=0xCA(futex) の直後
```

`RIP=0 / RBP=0 / CR2=0` かつ命令フェッチ＝**関数ポインタ 0 経由の call**。
スレッド（`UserRSP` が thread-stack 領域）が glibc `__clone` の子
トランポリンで `movq 0(%rsp),%rax; call *%rax` を実行し `%rax` が 0。

**真因（SMP TOCTOU レース）**: `linux_clone` は
`process_create_thread_ex()` でスレッドを **READY にした後** に
`process_set_thread_user_rsp(tid, stack)` で子 RSP を
glibc 用意のスタックへ差し替えていた。`process_create_thread_ex` 内では
先に `initialize_raw_user_stack()` がカーネル選定の別スタックを
`saved_user_rsp` に入れる。READY 公開から RSP 差し替えまでの窓で
**別 CPU のスケジューラが子を走らせる**と、glibc の子は
カーネル選定スタック上で `0(%rsp)` を読み（＝ゴミ/0）→ `call *0` → RIP=0。
`futex` を通せた子＝レースに勝った子、落ちた子＝負けた子、で間欠性に一致。

**修正**:
- `process_create_thread_ex()` に `uint64_t user_stack` 引数を追加
  （`ProcessManager.h` / 定義 / ネイティブ用ラッパ `process_create_thread`
  は `0` を渡す）。非 0 なら `initialize_raw_user_stack()` を**呼ばず**
  `thread->saved_user_rsp = user_stack` を **READY 公開前**に設定
  （glibc が fn/arg を積んだスタックなので中身は触らない）。
- `linux_clone`: `stack` を新引数として渡し、後段の
  `process_set_thread_user_rsp(tid, stack)` を削除。

`make kernel` 通過（404352 バイト、新規警告なし）。次の起動で全
pthread が正しいスタックで立ち、GLib → `gtk_init` →
`gdk_display_open` 失敗まで到達するはず。

### 2026-08-29 セッション20（Wayland コンポジタ第1弾：カーネル共有メモリ配線 K1–K5 ＋ ハンドロール compositor）

W1（`cannot open display`）到達を受けて、**実ウィンドウ表示**（`TODO_GTK3_Wayland_LinuxABI.md` G3）に着手。

**カーネル（K1–K5、`make kernel` 通過）**
- **K1 memfd を共有メモリで裏打ち**（`Syscall_File.c` / `Syscall_LinuxCompat.c`）:
  `kernel_memfd_t.shm_handle` 追加。最初の非 0 `ftruncate()` で
  `shared_memory_create(size)` に昇格（`malloc` 裏打ちは撤去）。
  `linux_mmap` の memfd 経路を新設し `shared_memory_map()` で
  **実共有ページ**をマップ（従来のファイル経路は一度きりのスナップショット
  コピーで shm には無意味だった）。dup で `shared_memory_addref`、close で
  `shared_memory_release`、grow は非対応（クライアントがプール再作成）。
- **K2 SCM_RIGHTS 実装**（`UnixSocket.c`）: `sendmsg` が渡された fd を
  `syscall_memfd_shm_handle()` でハンドル化 → `shared_memory_grant(h,
  peer->owner_pid)` ＋ `addref` → キュー。`recvmsg` が
  `syscall_memfd_install_shm()` で受信側に新しい memfd fd を作って
  in-flight 参照を引き渡す。従来は生の fd 整数をコピーするだけで無意味だった。
- **K3 AF_UNIX 強化**（`UnixSocket.c` / `.h`）: リングバッファ 1 KiB→**256 KiB**
  （BSS ではなく malloc）。`unix_sock_t.owner_pid` 追加。`recv` は空＋peer 生存で
  `-EAGAIN`、peer 消滅で `0`（libwayland が 0 を切断と誤認するため）。
  `iovlen`/`controllen` に上限。
- **K4**: ネイティブ AF_UNIX syscall（220–229）は既存。`SYSCALL_MEMFD_SHM_HANDLE`
  (269) を新設 — ネイティブプロセスが SCM_RIGHTS で受け取った memfd を
  `os_shared_memory_map()` するためのハンドル取得。`os_memfd_shm_handle()`
  ラッパを `Userland/API/Memory.h` に追加。
- **K5 poll が AF_UNIX を認識**（`Syscall_Epoll.c` / `UnixSocket.c`）:
  `epoll_poll_fd` の fd 種別スイッチが `0x8000+` を拾わず `EPOLLERR` を
  返していた → `unix_socket_poll()` を新設し配線。これが無いと
  libwayland がディスプレイ fd を死んだ接続と見なして即終了する。
- `SharedMemory.c/.h`: `shared_memory_addref` / `shared_memory_release`
  （非オーナー decref）/ `shared_memory_size` を追加。

**Userland**
- `Userland/Application/com.ImplusOS.waylandcompositor/`（新規、ネイティブ ELF）:
  libwayland 非依存の **ハンドロール Wayland ワイヤプロトコル**。`/tmp/wayland-0`
  で listen（ネイティブ AF_UNIX syscall）、`Window.h` の backing store に
  クライアントの `wl_shm` バッファを blit。対応: `wl_display`(sync/get_registry)、
  `wl_registry`(bind)、`wl_callback`、`wl_compositor`、`wl_shm`(+pool+buffer、
  create_pool の fd を SCM_RIGHTS で受信)、`wl_surface`(attach/frame/commit)、
  `xdg_wm_base`/`xdg_surface`/`xdg_toplevel`（初回 configure＋ack）、
  スタブ `wl_seat`(caps=0)/`wl_output`/`wl_subcompositor`/`wl_data_device_manager`。
  入力転送は未実装（U3）。
- `com.ImplusOS.gtk3demo/Start.c`: compositor を先に spawn → `sleep_ms(1500)` →
  `gtk3-demo`。`apps.list` に「Wayland Compositor」も追加。
- `make kernel` / `make app_build` 通過（compositor 260 KB、警告なし）。

**次の起動で見るもの**（シリアル）: `[wl] compositor starting` →
`[wl] listening on /tmp/wayland-0` → GTK 接続後 `[wl] client connected` →
`[wl] registry sent` → `[wl] pool mapped` → `[wl] xdg toplevel configured`。
ここまで出れば shm 共有と wl プロトコルが機能。ウィンドウが WM 上に出れば G3 成立。
出ない場合、止まった `[wl]` 行の直前が次のギャップ。

### 2026-08-29 セッション21（compositor 起動 → `socket setup failed` ＝ K3 の 256 KiB malloc 失敗を修正 ＋ accept のバッファ移送）

ユーザー起動: `[wl] compositor starting` → **`[wl] socket setup failed`**。

**原因**: セッション20 の K3 で `unix_socket_create()` を
`malloc(UNIX_SOCK_BUF_SIZE=256 KiB)` にしたが、カーネルヒープが
256 KiB の連続確保に失敗して `NULL` → `unix_socket_create` が `-12` を返し、
compositor の `u_socket()` が負値 → 起動失敗。

**修正**:
- `UNIX_SOCK_BUF_SIZE` 256 KiB → **32 KiB**、リングバッファを malloc から
  **inline BSS 配列**へ戻す（32 KiB × 16 = 512 KiB BSS）。実際の Wayland
  ソケット往来はピクセルが共有メモリを通るため小さく、32 KiB で十分。
  `!s->buf` チェック等を除去。
- `unix_socket_accept()`: **listener のリングに溜まったバイトと SCM_RIGHTS
  ハンドルを accept 時に新ソケットへ移送**。クライアントは connect 後
  `peer_fd` が listener を指している間に最初のバイト
  （`get_registry`/`sync`）を送るため、移送しないと初回ハンドシェイクが
  失われる。
- compositor: `u_socket`/`u_bind`/`u_listen` を個別に dbg 出力
  （次に詰まったら箇所が分かる）。

`make kernel` / `make app_build` 通過。次の起動で
`[wl] u_socket ok` → `[wl] u_bind ok` → `[wl] u_listen ok` →
`[wl] listening on /tmp/wayland-0` → `[wl] client connected` → … と進むはず。

### 2026-08-29 セッション22（`u_bind FAILED` ＝ `SYSCALL_UNIX_BIND`/`CONNECT` の成功判定が反転していた既存バグ）

`[wl] u_socket ok` → **`[wl] u_bind FAILED`**。

**原因（既存バグ、パス指定 AF_UNIX を誰も使っていなかったため未発覚）**:
`Syscall_Dispatch.c` の `SYSCALL_UNIX_BIND` / `SYSCALL_UNIX_CONNECT` が
`if (!copy_user_cstring(kpath, ...)) fail;` としていた。`copy_user_cstring`
（→ `copy_user_cstring_s`）は**成功で 0・失敗で負**を返すので、
`!0` が真＝**成功時に fail** していた（失敗時は負値で `!(-1)==0` となり
ゴミパスで処理続行）。Chromium は `socketpair` を使うためこの経路は
初めて踏まれた。

**修正**: 両ケースを `... != 0` 判定に変更。

`make kernel` 通過。次の起動で `[wl] u_bind ok` → `[wl] u_listen ok` →
`[wl] listening ...` → GTK 接続で `[wl] client connected` → `registry sent`
→ … と進むはず。

### 2026-08-29 セッション23（Wayland ハンドシェイク成功 → libxkbcommon が xkb データ dir を「実行不可」で拒否 → #PF）

大きく前進。シリアル:
```
[wl] client connected
[wl] registry sent          ← Wayland レジストリ交換まで成功
xkbcommon: ERROR: failed to add default include path /usr/share/X11/xkb
GLib: Cannot convert message: ... to "e" is not supported   ← 別件・非致命（未調査）
Gdk-WARNING: Failed to load cursor theme Adwaita             ← 非致命（未同梱）
[PF] RIP=0x41016B4174 (libxkbcommon 内) CR2=0 rdi=0 → pid 5 (gtk3-demo) 終了
```

**真因**: `syscall_access()`（`Syscall_LinuxCompat.c`、`access`/`faccessat`
共通）が **`X_OK`（mode & 1）を無条件で `-EACCES`** にしていた。
libxkbcommon は include dir を `access(dir, R_OK|X_OK)` で検査し、EACCES を
「無い」と扱ったうえで**その後 NULL 参照でクラッシュ**（xkbcommon 側の
堅牢性バグ、無改変方針なので触れない）。ISO には
`/usr/share/X11/xkb/{rules/evdev,keycodes,symbols,...}` は同梱済み
（ユーザーの `stage-xorg.sh` / `xorgdata` による）。

**修正**: `syscall_access()` を書き直し。
- 実ファイルが存在 → `X_OK`/`R_OK`/`F_OK` は常に成功（実行権限モデルが無い）。
  `W_OK` は read-only fs 判定を維持。
- `vfs_find_file` が外れたら `vfs_opendir(path)` でディレクトリ存在を確認 →
  ディレクトリなら `R_OK`/`X_OK`/`F_OK` 成功、`W_OK` は EACCES。
- どちらでもなければ `-ENOENT`。

`make kernel` 通過。次の起動で xkbcommon がキーマップをコンパイルできて
クラッシュしないはず。`gdk_display_open` を抜けてウィンドウ表示へ。
残: `GLib: ... to "e"` の調査、カーソルテーマ、実描画確認。
