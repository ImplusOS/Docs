# ImplusOS 全体リファクタリング TODO

> 調査基準日: 2026-08-23 / 決定事項確定日: 2026-08-23
> 対象: Kernel 全体（Arch / Drivers / FileSystem / Network / Platform / Compat）、Docs、CI/CD
>
> **本ドキュメント末尾（12章）の「未決定事項」は全てユーザーの回答により確定済み。**
> 確定内容は各該当セクションに反映し、12章にも回答結果を追記してある。実装は
> この確定内容に基づき着手フェーズ順（1章）で進める。

> **現況追記 (2026-08-29):**
> - P1〜P7 の成果はコードベースに反映済み（`Kernel/Compat/`, `Kernel/Drivers/`
>   のフラット化, `Kernel/Core/vfs/VFS.c` の名前非依存化, exFAT 読み取り専用
>   ドライバ, `Library/Crypto` 等）。
> - **P8 (CI/CD) は未反映**: `.github/workflows/*` と `.markdownlint.jsonc` は
>   本ツリーに存在しない。`Kernel/config/arch.mk` の `CI=1`→`-Werror` 昇格
>   フックのみ実装済み。設計は `Docs/Architecture/CI_CD.md` 参照。
> - **arm64**: 8章記載の `__trunctfdf2` 未解決リンクエラーは未修正のまま
>   （`make ARCH=arm64 kernel` は現在も失敗する）。
> - P7 のドキュメント刷新は 2026-08-29 に再実施
>   （`README.md` / `AGENTS.md` / `CLAUDE.md` / `GEMINI.md` / `Docs/Architecture/*`
>   を現行ソースツリーに合わせて更新）。
>
> **現況追記 (2026-08-30) — カーネルのドライバ名/実装依存の除去:**
> - **FS**: `FS_VFS_Bridge` の `fs_bridge_id_t` 列挙 + `"FAT32_Driver.ELF"`
>   等のハードコード表を撤廃。`fs_bridge_mount_all()` が
>   `DEVICE_TYPE_FILESYSTEM` レジストリを走査してマウントし、デフォルト
>   ルートFSは `vfs_media_kind_t`（光学優先、DISK は書込可能を優先）で選ぶ。
>   `kernel_main.c all_fs_initialize()` は FS 名を一切知らない。
> - **擬似FS**: devfs/tmpfs/procfs/etcfs/DRM のリストとマウント先を
>   新設 `Kernel/Core/vfs/VFS_Pseudo.c` のテーブルへ移動。`kernel_main.c` は
>   `vfs_mount_pseudo_filesystems()` を呼ぶだけ。
> - **ネットワーク**: `network_stack_init()` の
>   `ethernet_init()/arp_init()/…` 直列呼び出しを撤廃。
>   `network_builtin_drivers_init_all()` が `NetworkBuiltinDrivers.c` の
>   `deps[]` からトポロジカル順を解いて各層 init を駆動する（9.2 の
>   `deps[]` が初めて load-bearing になった）。ランタイム経路
>   （`ethernet_poll()` / `ipv4_process_timer()` / DHCP リトライ）は直呼びのまま。
> - **入力**: `PS2_Client.c` / `driver_manager_get_ps2_driver()` の
>   `"PS2_Driver.ELF"` 名指しを撤廃し `DEVICE_TYPE_INPUT` の先頭ドライバに。
> - **RTC**: `Kernel/Drivers/RTC/` → `Kernel/Platform/rtc/`（port-IO の
>   プラットフォーム HAL でありドライバモジュールではない）。syscall/Compat の
>   include から `Drivers/` が消えた。`PlatformBuiltinDrivers.c` が
>   呼出可能な vtable 付きで DeviceRegistry 登録。
> - **DriverDB (A8)**: `DriverDB.c` にドライバ名のハードコードは元から無し
>   （VID/PID→ELF 名は外部マニフェスト `Manifest/DriverDB.txt` のみが持つ、
>   Linux の modalias 相当）。コード変更不要。
> - `ACPI/Timer/LAPIC/IOAPIC/GIC` の登録名文字列は
>   `PlatformBuiltinDrivers.c`（唯一の登録サイト）に残置。これは「組込み
>   ドライバが自らの identity を宣言する」Plan A の想定どおりで、カーネル
>   他所がその名前で lookup しているわけではない。
> - `make ARCH=x86_64 kernel` / `make driver_build` は通過（新規ファイル
>   の警告ゼロ）。QEMU 実機起動での回帰確認は未実施。

---

## 0. 背景調査サマリ（着手前に必ず読むこと）

### 0.1 ディレクトリの現状

```
Kernel/
├── Driver/                    ← ソースではなく "マニフェスト+ファームウェア置き場"
│   ├── DriverDB.txt           ← usb/pci vid → オンデマンドロードするELF名の手書き対応表
│   └── Firmware/AX900/        ← ファームウェアBLOBをそのままステージするだけ
├── Drivers/
│   ├── Module/                ← DriverManager/DriverModule/BusRegistry/DeviceRegistry など
│   │                             "カーネルに常駐するドライバ基盤" 本体
│   ├── Client/                ← カーネルに直接リンクされる「薄いプロキシ層」
│   │   （例: FAT32_Client.c は driver_manager_find() で
│   │     FAT32_Driver.ELF の vtable を引いて右から左に横流しするだけ）
│   └── Server/                ← 実際のドライバ実装（別ELFとしてビルドされる本体）
├── Core/                      ← process/vfs/syscall/elf/timer/sync/usercopy
├── Platform/                  ← acpi/interrupt/io/ipc/timer（起動シーケンスに直接組込み）
├── Network/                   ← arp/dhcp/ethernet/icmp/tcp/udp（起動シーケンスに直接組込み）
└── Arch/{x86_64,arm64}/       ← cpu/hal/mmu/smp/timer/virt/linker
```

`Kernel/Driver`（単数形）と `Kernel/Drivers`（複数形）が並存しており、前者はソースツリー
ではなくビルド成果物のステージング用ディレクトリ名（ブートイメージ上のパス規約）。
紛らわしいだけでなく、名前だけでは役割が推測できない。

`Client` / `Server` の分離は、実装を見る限り以下のように機能している:

- `Server/<Category>/<Driver>/<Driver>_Main.c` — 実ドライバ実装。`driver_module_init()`
  をエクスポートし、個別ELF（例: `FAT32_Driver.ELF`）としてビルドされる。
- `Client/<Category>/` — カーネル本体に静的リンクされるプロキシ。
  `driver_manager_find(DEVICE_TYPE_*, "XXX_Driver.ELF")` でロード済みモジュールの
  vtable を引いて転送するだけのコード（`FAT32_Client.c` 全179行の大半がこのパターン）。
  ファイルシステム系はさらに `XXX_VFS_Adapter.c` が `vfs_driver_t` への変換を担う。

この「モジュール本体 → カーネル内プロキシ → VFSアダプタ」の3段構成は、モノリシック
カーネルである ImplusOS にとって歴史的経緯以上の意味を持たない冗長な間接層になっている
（`Kernel/Core/kernel_main.c:33-36` が `Drivers/Client/FileSystem/FAT32/FAT32_Main.h` と
`FAT32_VFS_Adapter.h` を両方 include しているのがその証跡）。`grep -rl VFS_Adapter` の
結果は次の2ファイルのみ:

- `Kernel/Drivers/Client/FileSystem/FAT32/FAT32_VFS_Adapter.c`
- `Kernel/Drivers/Client/FileSystem/ISO9660/ISO9660_VFS_Adapter.c`

### 0.2 VFS が FileSystem 名/ID に依存している箇所

`Kernel/Core/vfs/VFS.c`:

- `vfs_mount()`（19–28行目）: `strcmp(driver->fs_type, "iso9660") == 0` という
  ハードコードされた文字列比較で「未設定なら iso9660 をデフォルトFSにする」処理をしている。
- `vfs_set_default_fs()`（30–38行目）: `fs_type` 文字列の一致でドライバを検索する。
- `Kernel/Core/kernel_main.c:220-255`（`all_fs_initialize()`）でも
  `vfs_set_default_fs("iso9660")` を直接呼び出しており、起動時のブートメディア判定ロジックが
  VFS 層と FS 名文字列を介して密結合している。

これが「VFS 側で FileSystem 名や ID に依存しない形で複数の FileSystem を管理する」という
要求の指す具体的な箇所。

### 0.3 exFAT は未実装

`Kernel/Drivers/Server/FileSystem/exFAT/exFAT_Main.c` は **0 バイト**。
`exFAT_Main.h` も存在しない。一方で `Docs/Architecture/Driver_Module_Guide.md` と
`Kernel_Architecture.md` は `exFAT_Driver.ELF` をあたかも実装済みであるかのように記載して
おり（ドキュメントの正確性という要求にも直結する）、`Kernel/Drivers/Module/DriverDB.h` /
`DriverSelect.h` 等に exFAT 向けの型・登録が実在するかも実装時に要確認。

参照実装として使えるのは:
- `Kernel/Drivers/Server/FileSystem/FAT32/FAT32_Main.c`（2319行、フル実装）
- `Kernel/Drivers/Server/FileSystem/ISO9660/`（読み取り専用の参考実装）

### 0.4 x86_64 / arm64 の差分

**[2026-08-24 P4完了時点の追記]** 本節は元々の調査時点（P0）の記述。P4実施時に
再調査した結果、以下の訂正が必要と判明した: `GDT_Main.h`/`IDT_Main.h` は
「ヘッダだけ存在して実体が無い」状態では **なかった**（`Kernel/Arch/arm64/cpu/Exception.c`
が `init_gdt`/`init_idt`/`init_idt_per_cpu`/`register_interrupt_handler`/
`gdt_set_kernel_rsp0` を全て実装済みで、`GenericTimer.c` からも実際に使われている）。
以下の元の表・分析は「P4着手前の状態」の記録として残すが、**P4完了により
タイマHAL選択・仮想化初期化の`#ifdef`分岐は解消済み**（8.1節参照）。

`Kernel/Arch/x86_64/` と `Kernel/Arch/arm64/` を比較すると（P0時点）:

| 要素 | x86_64 | arm64 | 備考 |
|---|---|---|---|
| CPU テーブル | GDT + IDT（`cpu/GDT_Main.c`, `IDT_Main.c`） | 例外ベクタ（`cpu/Exception.c`, `ExceptionVectors.S`）が `GDT_Main.h`/`IDT_Main.h` 宣言の関数を実装 | 双方とも `arch_ops_t.init_cpu_tables` 経由で統一済み |
| 割込みコントローラ | IOAPIC + LAPIC | GIC v3 | `Platform/interrupt/` で吸収 |
| タイマ | LAPIC Timer | Generic Timer | **P4で解消**: `arch_ops_t.get_timer_hal()` 経由に統一（8.1節） |
| 仮想化 | VMX（`virt/VMX_Main.c`, `VMX_EPT.c`, フルスクラッチ実装） | `arch_ops_t.virtualization_init` が明示的に `-1`（未対応）を返すスタブ | **P4で解消**: 両アーキとも `arch_ops_t.virtualization_init` 経由の呼び出しに統一済み（8.1節） |
| SMP起動 | AP trampoline（`smp/SMP_Trampoline.asm`） | PSCI（`smp/PSCI.c`） | 抽象化済み（`SMP_Main.c` 経由） |

`Kernel/Core/elf/ELF_Loader.c` の `#ifdef PLATFORM_ARM64`/`PLATFORM_X86_64` は
P4で監査済み（8.1節item5）: ELFマシンタイプ判定・命令キャッシュ同期命令・
再配置タイプ処理という、ISAごとに本質的に異なる領域のみで使われており、
`arch_ops_t` 越しに統一すべき「抽象化漏れ」ではなく正当なアーキ差と判断し、
変更していない。

### 0.5 サブシステムの現在の統合ポイント

`kernel_main_after_stack_switch()`（`Kernel/Core/kernel_main.c:347-587`）を見ると、
ACPI・IOAPIC/LAPIC・Timer は **ページング/ヒープ初期化の直後、割込み有効化前** に
初期化されており（399–439行目）、これはドライバモジュールローダ
（`driver_module_manager_init()`, 441行目）より前。つまり ACPI/Timer/LAPIC/IOAPIC は
「ディスクからELFを読んで動的ロードする」現行のドライバモジュール機構が動く**前提条件**
そのものであり、素朴に「ドライバ化」してディスクからロードする対象にはできない
（ニワトリタマゴ問題）。

一方、`network_stack_init()`（558行目）は `driver_module_init_deferred()` 実行後・
ファイルシステム初期化後というかなり遅いタイミングで呼ばれており、こちらは既存の
NIC ドライバモジュール機構（`driver_nic_t`, `AX900.c` 等）と同じ枠組みに素直に
乗せられる。

この非対称性（ACPI/Timer/LAPIC/IOAPIC は起動必須の「組込みドライバ」、
ARP/DHCP/Ethernet/ICMP/TCP/UDP/IPv4 は「真にロード可能なプロトコルドライバ」）は
本リファクタリングの設計判断の中核なので、フェーズ5で詳述する。

### 0.6 Linux 互換層の現状

`Kernel/Core/syscall/Syscall_LinuxCompat.c` が Linux ABI 互換（88 syscall、ELF の
`EI_OSABI==ELFOSABI_LINUX` 判定は `ELF_Loader.c`、ディスパッチ分岐は
`Syscall_Dispatch.c` の `syscall_dispatch()` 内 `if (abi == PROCESS_ABI_LINUX)`）を
担っている。既存の `Docs/Others/TODO_Chromium_LinuxABI.md` に詳細な現状分析があるので、
本リファクタリングでは実装内容そのものより「**Core/syscall の中に直書きされている
Linux 固有コードを、将来他OS互換層を追加できる形の独立サブシステムへ切り出す**」ことに
専念する。

### 0.7 CI/CD

`.github/` ディレクトリは存在しない。ビルドはクロスコンパイラ
（`gcc-x86-64-elf`, `g++-x86-64-elf`, aarch64版）に依存し、`Makefile` は
`ARCH=x86_64|arm64` で完全にパラメータ化済み。GitHub Actions のホストランナーには
`x86_64-elf-gcc`/`aarch64-elf-gcc` が標準搭載されていないため、CI 実装は
「クロスツールチェインの調達」が最初の技術的課題になる。

### 0.8 誤字: RecoveryEnviroment

`grep -rn RecoveryEnviroment` の該当箇所は次の5箇所のみ（小規模な機械的修正）:

- `RecoveryEnviroment/`（ディレクトリ名そのもの、`Recovery.c` と `Makefile` を含む）
- `Makefile:68,73,281,365`（`RECOVERY_DIR`, `RECOVERY_INIT_ELF`, オブジェクトパス、
  パターンルール）
- `RecoveryEnviroment/Makefile:9`（`clean` ターゲット内の `rm -rf` パス）

ビルド成果物パス（`$(BUILD_DIR)/RecoveryEnviroment/...`）はソース側のディレクトリ名から
機械的に導出されているだけなので、ディレクトリを `git mv` してから `Makefile` の文字列を
一括置換すれば良い。

---

## 1. 実行フェーズと優先順位

依存関係を考慮し、以下の順序で実施する。各フェーズは独立した PR 単位に分割できる粒度で
書いてある。

**全フェーズ完了 (2026-08-24)**。各フェーズの実施内容・検証結果は該当セクション
（3〜11章、各章冒頭の「進捗」欄）を参照。P5(9.2)は当初計画の完全な個別ELF分割から
実装可能性を踏まえたハイブリッド案へ、P8はトレードオフ判断によりツールチェイン調達
方式を変更しているなど、一部フェーズは着手時の技術的判明事項によりスコープを調整して
いる（各章「スコープ限定」参照）。x86_64については全フェーズ後の
`make ARCH=x86_64 all` → `image_livecd` → QEMU実機起動で最終回帰確認済み
（パニック/アサート無く19フェーズ全ての起動シーケンスを完走）。

| フェーズ | 内容 | リスク | 依存 | 状態 |
|---|---|---|---|---|
| P0-b | `RecoveryEnviroment` 誤字修正 | 低 | なし（最初にやる） | ✅完了 |
| P0-c | `Kernel/Driver` → `Kernel/Drivers` 統合（ソースツリーのみ。5.2節参照） | 低〜中 | なし | ✅完了 |
| P1 | ドライバモデル刷新（Client/Server区分廃止、5.1節の提案どおり） | 中〜高 | P0-c | ✅完了（5.3節は未着手、5章参照） |
| P2 | VFS 刷新（名前非依存化）+ VFS_Adapter 廃止 + exFAT 実装 | 中 | P1 | ✅完了（exFATは読み取り専用） |
| P3 | Kernel 堅牢性・安全性強化 | 中（横断的） | なし（並行可） | ✅完了 |
| P4 | x86_64/arm64 機能差解消 | 中〜高 | P1（一部） | ✅完了（arm64は既存の別課題によりビルド未完了、8章参照） |
| P5 | プラットフォーム/ネットワークサブシステムのドライバ化（案A採択、個別ELF分割） | 高 | P1, P4 | ✅完了（9.2はハイブリッド方式、9章参照） |
| P6 | Linux 互換層の独立サブシステム化（本リファクタに含める） | 中 | P1 | ✅完了 |
| P7 | ドキュメント刷新 | 低〜中（横断的） | 各フェーズ完了後に追随 | ✅完了 |
| P8 | GitHub Actions CI/CD 基盤 | 低 | 後回し。本プロジェクトでの優先度は低いとユーザーが明言したため最終フェーズとする | ✅完了（GitHub Actions実機での実行検証は未実施、2章参照） |

> P0-a として最初に置いていた CI/CD 整備は、ユーザーの指示によりフェーズ順の末尾
> （P8）へ移動した。内容自体は2章にそのまま残しているが、**着手順は最後**である
> ことに注意（2章というセクション番号は現状調査の記述順の都合上そのまま）。

---

## 2. P8: GitHub Actions CI/CD 基盤（優先度低・最終フェーズ）

**進捗: 完了 (2026-08-24)、P1〜P7完了後・余力ありのため着手**
- [x] トレードオフの説明どおり、推奨案2（自前Dockerイメージ/GHCR）ではなく
      「実際にこのプロジェクトのローカル開発で使われているHomebrewの
      クロスコンパイラフォーミュラ（`x86_64-elf-gcc`等）を`actions/cache`で
      キャッシュしつつ導入する」という第三の方式を採用。理由はワークフロー
      ファイル自体のコメントと `Docs/Architecture/CI_CD.md` 2章に記載
- [x] `.github/workflows/build.yml`: `matrix.arch: [x86_64, arm64]`。
      arm64は既知の既存バグ（P4進捗注記の `__trunctfdf2` リンク未解決）を
      理由に `continue-on-error: true` とし、x86_64側の合否がリグレッション
      検知として機能することを優先
- [x] `-Werror` 昇格用の `CI=1` を `Kernel/config/arch.mk` に追加したが、
      既存の（本リファクタリング前から存在する）警告が複数残っているため
      `build.yml` では**あえて有効化していない**（即座にCIが赤くなる
      state-shipping問題を避けるため）。理由と有効化手順は
      `Docs/Architecture/CI_CD.md` 3章に記載
- [x] `.github/workflows/docs-lint.yml`: `markdownlint-cli2` + 新設
      `.markdownlint.jsonc`（意図的に緩い設定、理由はファイル内コメント参照）
- [x] `.github/workflows/static-analysis.yml`: 計画は `clang-tidy` を
      候補としていたが、`compile_commands.json` 生成のセットアップコストが
      本フェーズの優先度に見合わないと判断し `cppcheck` を採用（理由は
      `Docs/Architecture/CI_CD.md` 4章）。`--error-exitstatus` は未使用
      （初回実行の検出数が未知のため非ブロッキングでスタート）
- [x] **未実施・ユーザー対応依頼事項**: 2.2節のブランチ保護設定は計画自体が
      明記するとおりリポジトリ設定変更でありコードコミットからは変更不可。
      推奨設定を `Docs/Architecture/CI_CD.md` 5章に記載したので、
      ワークフローが実環境で数回成功した後にユーザー自身が
      `Settings → Branches` で適用すること

### 元計画

> **ユーザー確定事項**: 「このプロジェクトではあまり重要ではない」との判断により、
> 本フェーズは最後（P8）に回す。P1〜P7 が完了し、かつ余力がある場合にのみ着手する。

**決定が必要な事項**: クロスツールチェインの調達方法。
1. CI 実行のたびに `x86_64-elf-gcc`/`aarch64-elf-gcc` をソースからビルド（時間がかかるが
   キャッシュで軽減可能。`actions/cache` で `~/.local/cross/` 等をキャッシュ）
2. 事前ビルド済みツールチェインを含む Docker イメージを自前で用意し GHCR に置く
   （初回構築コストは高いが以降のCIは高速）
3. 既存の GitHub Action（例: `crosstool-ng` 系）を利用

推奨は **2（自前 Docker イメージ）**。理由: ビルドが決定的になり、`apt install
gcc-x86-64-elf` のような Ubuntu 固有パッケージ（Debian/Ubuntu 限定で提供されている
`gcc-x86-64-elf` は実体が `binutils-multiarch` 等に依存し将来のランナーイメージ更新で
壊れやすい）への依存を切り離せる。

### 2.1 ブランチ保護

ワークフローファイルだけでは強制できないため、リポジトリ設定
（Settings → Branches → Branch protection rules）で `build.yml` と
`boot-smoke-test.yml` を必須チェックに指定することを **ユーザーに依頼**する
（Claude Code からは変更不可）。

---

## 3. P0-b: `RecoveryEnviroment` → `RecoveryEnvironment`

**進捗: 完了 (2026-08-24)**
- [x] `git mv RecoveryEnviroment RecoveryEnvironment` + Makefile 参照修正
- [x] `make ARCH=x86_64 recovery_build` で単体ビルド確認

1. `git mv RecoveryEnviroment RecoveryEnvironment`
2. `Makefile` 内の `RECOVERY_DIR := RecoveryEnviroment` を含む4箇所を
   `RecoveryEnvironment` に置換
3. `RecoveryEnvironment/Makefile` の `rm -rf ../Build/$(ARCH)/RecoveryEnviroment` を
   `RecoveryEnvironment` に修正
4. `Build/` 配下の成果物パス（`$(BUILD_DIR)/RecoveryEnviroment/...`）は変数から
   自動的に追随するので個別修正は不要
5. `make clean && make ARCH=x86_64 recovery_build` でビルド確認

---

## 4. P0-c: `Kernel/Driver` → `Kernel/Drivers` 統合

**ユーザー確定事項**: 「ソースツリーの統合にとどめる。必要な場合のみパス規約も変更可」
との回答を得た。調査の結果、ブートイメージ上のパス規約（`Kernel/Driver/`,
`Kernel/Driver/OnDemand/`, `Kernel/Driver/Firmware/`）を変更する**技術的必然性は無い**
（ソースツリー上のディレクトリ名とESP上のステージング先パス名は `Makefile` の変数
（`DRIVER_DB_SRC`, `FIRMWARE_SRC_DIR` 等）で既に分離されており、片方だけ変更しても
ビルドは成立する）。したがって本フェーズでは**ソースツリー側のみ**を統合し、
ブートイメージ上のパスは現状維持とする。
- **ソースツリー側**（`Kernel/Driver/DriverDB.txt`, `Kernel/Driver/Firmware/`）は
  `Kernel/Drivers/` 配下に統合する（例: `Kernel/Drivers/Manifest/DriverDB.txt`,
  `Kernel/Drivers/Firmware/AX900/`）。これは純粋にソース整理でありビルド成果物の
  ESP 上パスとは独立に変更できる。
- **ブートイメージ上のパス**（`Kernel/Driver/`, `Kernel/Driver/OnDemand/`,
  `Kernel/Driver/Firmware/`）は当面現状維持する。理由: これは実行時のファイル配置規約
  であり「ソースコードの構成」とは別の関心事。変更するなら別途
  `Docs/Others/TODO_Boot_Image_Layout.md` のような専用計画を切って、ブートローダ・
  インストーラ・ドキュメントを横断的に一度に変更すべきで、本リファクタリングの
  スコープ外とする（ユーザーが望む場合はフェーズを追加）。

**進捗: 完了 (2026-08-24)**
- [x] `Kernel/Driver/DriverDB.txt` → `Kernel/Drivers/Manifest/DriverDB.txt`
- [x] `Kernel/Driver/Firmware/` → `Kernel/Drivers/Firmware/`
- [x] `Makefile` の `DRIVER_DB_SRC`/`FIRMWARE_SRC_DIR` 参照元パスのみ更新
      （ブートイメージ上の `Kernel/Driver/...` 文字列は計画どおり不変更）
- [x] `make ARCH=x86_64 driver_stage` で単体ビルド確認

### 4.1 作業手順

1. `git mv Kernel/Driver/DriverDB.txt Kernel/Drivers/Manifest/DriverDB.txt`
2. `git mv Kernel/Driver/Firmware Kernel/Drivers/Firmware`
3. `Makefile` 内 `DRIVER_DB_SRC := Kernel/Driver/DriverDB.txt` と
   `FIRMWARE_SRC_DIR := Kernel/Driver/Firmware` の参照元パスのみ更新
   （コピー先=ブートイメージ上のパス文字列 `$(1)/Kernel/Driver/...` は変更しない）
4. `Kernel/Drivers/Module/DriverDB.c` 冒頭コメントに新しいソースパスを反映
5. 空になった `Kernel/Driver/` ディレクトリを削除
6. `Docs/Architecture/*.md` のパス記載を更新（フェーズ7と重複するが、この時点で
   最低限のパス修正はしておく）

---

## 5. P1: ドライバモデル刷新（Client/Server 区分の廃止）

**進捗: 5.1/5.2 完了 (2026-08-24)、5.3 未着手**
- [x] 5.1 節の仕分けどおり `Client/`/`Server/` を解体。実ドライバは
      `Kernel/Drivers/<Category>/<Driver>/` にフラット化、バス/入力ゲートウェイ
      （PCI/USB/NIC/Display/PS2/Evdev）は `Kernel/Drivers/Module/` へ統合。
      `DRM_Client`/`KVM_Client` は syscall実装寄りと判断し `Kernel/Core/drm/`・
      `Kernel/Core/kvm/` へ、`UnixSocket` は `Kernel/IPC/` へ移動（5.1節の想定どおり）
- [x] `Kernel/Drivers/Client/`・`Kernel/Drivers/Server/` は完全に消滅
      （FileSystem 分は P2 と合わせて完了、6.2節参照）
- [x] `Kernel/Drivers/Makefile`（`DRIVER_RESIDENT_DIRS`）・トップレベル `Makefile`
      （`DRIVER_MAKEFILES`）・各ドライバの `module.mk` 相対パスを深さ変更に追随して修正
- [x] `make ARCH=x86_64 all` フルビルドで `EXIT:0` を確認（唯一の `error:` は
      NetSurf 側 `nsfb` ネイティブリンクの既知の想定内失敗で無関係）
- [ ] 5.3節（既存ドライバの `bus_matches[]`/`probe()` 方式への移行、
      `BusRegistry` バインディング照会API、ホットプラグ経路の統一）は**未着手**。
      AX900 のみが新方式を使っている状態が変わっていないため、着手する場合は
      別途このセクションで進捗管理する

**ユーザー確定事項**: 案Aをそのまま採択。

- **案A（採択）**: ディレクトリの `Client/` と `Server/` という名前区分を廃止し、
  実体に即した名前に置き換える。実ドライバ実装は
  `Kernel/Drivers/<Category>/<DriverName>/` に一本化。カーネルに静的リンクされる
  「バス層・ゲートウェイ層」（PCI_Client.c, USB_Client.c, NIC.c, Display_Main.c 等、
  複数のドライバモジュールを横断してディスパッチする真に必要なコード）は
  `Kernel/Drivers/Module/` 配下（既存の DriverManager 等と同居）に統合し、
  「1ドライバにつき1プロキシファイルを手書きする」パターン（FAT32_Client.c,
  ISO9660 相当）は Linux の `register_filesystem()` 型の汎用登録機構で代替して
  **ファイル自体を削除**する（詳細はフェーズ2）。
- **案B**: ディレクトリ構成はそのままに、意味だけ変える（非推奨、ユーザー要求の
  「区分をなくす」に対して弱い）。

以降は案Aを前提に記述する。

### 5.1 現状分類の仕分け

`Kernel/Drivers/Client/` 配下9カテゴリを実態で仕分けると:

| ディレクトリ | 実態 | P1での扱い |
|---|---|---|
| `FileSystem/FAT32`, `FileSystem/ISO9660` | 1駆動につき1プロキシ+1VFSアダプタ | **廃止**。フェーズ2で汎用FS登録機構に置換 |
| `PCI/PCI_Client.c` | PCIバス列挙のカーネル側ゲートウェイ、`BusRegistry` の直接の呼び出し元 | `Kernel/Drivers/Bus/PCI/` に統合し実装を残す（名前だけ変更） |
| `USB/USB_Client.c` | USBバス列挙のカーネル側ゲートウェイ | `Kernel/Drivers/Bus/USB/` に統合 |
| `NIC/NIC.c` | 複数NICドライバの汎用ディスパッチ層 | `Kernel/Drivers/Module/NicManager.c` と機能重複がないか確認の上、`Module/` に統合するか独立の `Kernel/Drivers/Net/` に置く |
| `Display/Display_Main.c` | ディスプレイドライバ選択・共通処理 | `Kernel/Drivers/Module/DisplayManager.c` との統合を検討 |
| `PS2/PS2_Client.c`, `Evdev/Evdev_Client.c` | 入力デバイスのカーネル側ゲートウェイ | `Kernel/Drivers/Input/` に統合 |
| `DRM/DRM_Client.c`, `KVM/KVM_Client.c` | syscall向けDRM/KVM風インターフェースの実装 | 用途は「ドライバのクライアント」ではなく「syscall実装」に近いので `Kernel/Core/syscall/` 側との境界を再検討 |
| `UnixSocket/UnixSocket.c` | AF_UNIX ソケット実装 | ドライバではないので `Kernel/Core/` 系（IPC or Network）へ移動を検討 |

このように、`Client/` は実際には「汎用バスゲートウェイ」「per-FSプロキシ」
「syscall実装寄りのコード」という異なる性質のものが1つのフォルダ名の下に
混在している。**フォルダ名を変えるだけでなく、実態ごとに正しい置き場所へ再配置する
こと自体がこの要求の本質**である。

### 5.2 実ドライバ本体（現 `Server/`）の再配置

`Kernel/Drivers/Server/<Category>/<Driver>/` → `Kernel/Drivers/<Category>/<Driver>/`
にフラット化する。例:

```
Kernel/Drivers/Server/USB/XHCI/     → Kernel/Drivers/USB/XHCI/
Kernel/Drivers/Server/Block/AHCI/   → Kernel/Drivers/Block/AHCI/
Kernel/Drivers/Server/FileSystem/FAT32/ → Kernel/Drivers/FileSystem/FAT32/
```

`Kernel/Drivers/Module/` （DriverManager 等の基盤）と `Kernel/Drivers/Bus/`
（PCI/USBバス列挙、5.1で移設）はカテゴリ扱いせず区別する。

この移動は git 上は大量の `git mv` + Makefile/include パスの一括置換になるため、
**必ず1カテゴリ（例: まず `FileSystem/`）を通しで移行してビルド確認 →
残りのカテゴリを続ける」という段階的な移行にする**（一度に全部やると
ビルドが長時間壊れた状態になりレビュー不能になる）。

### 5.3 バス/デバイスモデルの Linux 参考改善

現行の `driver_module_descriptor_t.bus_matches[]` + `probe()`/`remove()`
（`Kernel/include/interfaces/driver_api.h:611-652`、API 2.2 で追加）は、
Linux の `struct device_driver` の `probe`/`remove` と `MODULE_DEVICE_TABLE`
に相当する良い設計だが、ドキュメントコメントにある通り **新しいドライバ
（AX900等）だけが使っており、VirtIONet/I219V 等の既存ドライバは未だに
「起動時に自分でバスをスキャンする」旧方式のまま**（`driver_api.h:611-617`
のコメント参照）。

改善タスク:
1. 全 PCI/USB 接続ドライバ（AHCI, NVMe, VirtIOBlk, VirtIONet, I219V,
   AC97, HDA, VirtIOSound, VirtIO Display, EHCI/OHCI/UHCI/XHCI, HID,
   MassStorage）を `bus_matches[]` + `probe()` 方式に移行し、
   「起動時に自分でPCI列挙する」コードを削除する
2. `BusRegistry`（`Kernel/Drivers/Module/BusRegistry.c`）に、Linux の
   `sysfs` の `driver`/`device` リンクに相当する「どのデバイスにどのドライバが
   バインドされているか」を問い合わせるAPI（`bus_registry_get_binding()`等）を追加し、
   `OSDebug` アプリや `/proc` 経由でユーザランドから可視化できるようにする
   （`Kernel/Core/vfs/ProcFS.c` に `/proc/bus/devices` 相当を追加）
3. ホットプラグ（`driver_manager_hotplug_poll()`）と `probe()`/`remove()` の
   結線を統一する（現状ホットプラグ経路と起動時経路でロジックが分かれていないか要確認）

---

## 6. P2: VFS 刷新・VFS_Adapter 廃止・exFAT 実装

**進捗: 完了 (2026-08-24)**
- [x] 6.1: `vfs_media_kind_t`（`UNKNOWN`/`OPTICAL`/`DISK`/`PSEUDO`）を
      `vfs_types.h` に追加し `vfs_driver_t.media_kind` に反映。`VFS.c` の
      `strcmp(driver->fs_type, "iso9660")` ハードコードを撤去し、
      `vfs_set_default_fs_by_kind()` を新設して `kernel_main.c` の
      `all_fs_initialize()` から呼ぶ形に変更（光学メディア優先という既存の
      起動時挙動は不変のまま維持）。`vfs_find_file`/`creat`/`mkdir`/`opendir`/
      `unlink` に丸ごとコピペされていたプレフィックス一致ロジックは
      `vfs_resolve_candidates()` に共通化
- [x] 6.2: `Kernel/Drivers/Client/FileSystem/{FAT32,ISO9660}/` の
      Client+VFS_Adapter 各2ファイルを、`Kernel/Drivers/Module/
      {FAT32,ISO9660}_VFS_Bridge.c/.h` の1ファイルに統合・削除。
      **設計上の補足**: 本節が理想として描いた「per-FSアダプタ関数が完全に0になる
      汎用キャストのみの構成」までは踏み込んでいない
      （`FAT32_FILE`/`ISO9660_FILE`/`exFAT_FILE` が各ドライバ固有のハンドル型で
      あり続けているため、`vfs_file_t.driver_data` への変換コード自体は
      FSごとに必要）。ただし要求の核心である「`VFS_Adapter` ファイルの削除」と
      「3段構成→2段構成への圧縮」は達成しており、`Kernel/Drivers/Client/`・
      `Kernel/Drivers/Server/` は本節の変更をもってディレクトリごと消滅した
- [x] 6.2 追補 (2026-08-29): FSごとの `*_VFS_Bridge.c/.h`（FAT32/ISO9660/exFAT
      の計6ファイル）と各FS固有の vtable 型（`fat32_driver_t`/`iso9660_driver_t`/
      `exfat_driver_t`）を廃止。各FSドライバモジュールは共通の
      `fs_module_ops_t`（`Kernel/include/kernel/interfaces/fs_module_ops.h`）
      1種類だけを `driver_module_descriptor_t.driver_api` としてエクスポートし、
      カーネル側の変換は単一の `Kernel/Drivers/Module/FS_VFS_Bridge.c` に集約。
      ファイルハンドルは不透明 `void *` + `handle_size` とし、per-open ハンドルの
      malloc/free と書き込み後の `vfs_file_t.size` 更新をブリッジが担う（§6.2 が
      「踏み込んでいない」とした汎用化を達成）。`readdir` の `vfs_dirent_t` 変換・
      exFAT の64bitサイズclamp・FAT属性→ディレクトリ判定はオンディスク形式を
      知るモジュール側へ移動。init は旧 `FAT32_VFS_Bridge.c` の遅延
      `driver_manager_find()` 再解決 + init-once パターンに全FS統一。
      `vfs_dirent_t`/`vfs_media_kind_t` はモジュールから安全に見えるよう
      `kernel/interfaces/vfs_dirent.h` に分離。FS追加は `g_fs_bridges[]` に1行 +
      `fs_bridge_id_t` に1定数のみで、新規ブリッジファイル不要
- [x] 6.3: exFAT 読み取り専用ドライバを実装（`Kernel/Drivers/FileSystem/exFAT/
      exFAT_Main.c/.h` + `Kernel/Drivers/Module/exFAT_VFS_Bridge.c/.h`）。
      Boot Sector 検証、MBR/GPT パーティション検出、FATチェーン走査、
      `NoFatChain` 最適化（ストリームエクステンションの
      `GeneralSecondaryFlags` bit1）に対応した O(1) 直接クラスタ算出、
      File Directory Entry + Stream Extension + File Name Entry の
      エントリセット解析、`find_file`/`read_file`/`read_at`/`opendir`/
      `readdir`/`closedir`/`list_root_files` を実装し `all_fs_initialize()`
      に組み込み済み。計画4項の段階リリース方針どおり
      **読み取り専用が第1弾**（`write_file`/`write_at`/`creat`/`mkdir`/
      `unlink`/`truncate` は `false` を返すのみ、書き込み対応は未着手）。
      Allocation Bitmap・Up-case Table の読み取りは実装していない
      （読み取り専用実装ではファイル探索・読み込みに必須ではないため
      スコープ外とした。名前比較は大文字小文字非依存のASCII比較で代替）
- [x] `make ARCH=x86_64 all` → `image` → `image_livecd` のフルビルドが
      いずれも `EXIT:0`（新規ファイルは `-Wall -Wextra -Wtype-limits
      -Wconversion -Wsign-conversion -Wshadow` で警告ゼロ）。
      QEMU (OVMF, q35, headless) でのブートテストで
      `fs_init duration_us=5647` を含む全初期化フェーズが完走し、
      panic/assert 無くユーザランド起動・アイドルループ到達まで確認

### 6.1 VFS の FS名/ID 非依存化

`Kernel/Core/vfs/VFS.c` の変更方針:

1. `vfs_driver_t`（`Kernel/include/kernel/interfaces/vfs_types.h`）に
   `fs_type` 文字列の代わりに、あるいは追加で次のような **属性ベースの選定情報**
   を持たせる:
   ```c
   typedef enum {
       VFS_MEDIA_KIND_UNKNOWN = 0,
       VFS_MEDIA_KIND_OPTICAL,   /* 現行 iso9660 相当の「読み取り専用光学メディア」 */
       VFS_MEDIA_KIND_DISK,      /* 現行 fat32/exfat 相当の「書き込み可能な固定/リムーバブルディスク」 */
       VFS_MEDIA_KIND_PSEUDO,    /* devfs/tmpfs/procfs/etcfs */
   } vfs_media_kind_t;

   typedef struct vfs_driver {
       vfs_media_kind_t media_kind;
       uint32_t priority;   /* 同じ media_kind 内でのデフォルト選定優先度 */
       /* ... 既存の関数ポインタ群 ... */
   } vfs_driver_t;
   ```
2. `vfs_mount()` の `strcmp(driver->fs_type, "iso9660") == 0` によるハードコードを
   撤去し、`media_kind == VFS_MEDIA_KIND_OPTICAL` のように **列挙型の比較**へ置換する
   （文字列比較を無くすだけでなく、「iso9660という個別実装を知っている」という
   VFS側の知識そのものを消す）。
3. `vfs_set_default_fs(const char *fs_type)` は、呼び出し元
   （`Kernel/Core/kernel_main.c` の `all_fs_initialize()`）がブートドライブ種別
   （`BOOT_DRIVE_TYPE_*`、`kernel_boot_drive_type_name()` 参照）を既に知っているので、
   文字列名ではなく `vfs_media_kind_t` を渡す `vfs_set_default_fs_by_kind()` に
   置き換える。既存の文字列版は POSIX 互換 `mount(2)` 等、外部から名前で指定する
   ユースケース向けに残すなら残してよいが、**内部の起動時デフォルトFS選定ロジックは
   名前非依存にする**のが要求の核心。
4. `vfs_find_file()` 等のプレフィックスマッチロジック（`VFS.c:115-158` など、
   同じ「最長一致プレフィックス検索→無ければデフォルトFS」のパターンが
   `vfs_creat`/`vfs_mkdir`/`vfs_opendir`/`vfs_unlink` の4関数に丸ごとコピペされている）
   は、**関数化して重複を排除する**（P3の堅牢性強化にも直結する副次効果）。

### 6.2 VFS_Adapter / Client プロキシの廃止

現状の3段構成（`FAT32_Main.c` 実装 → `FAT32_Client.c` プロキシ →
`FAT32_VFS_Adapter.c` 変換）を、次の2段に圧縮する:

```
FAT32_Main.c（ドライバ本体、driver_module_init() をエクスポート）
   ↓ driver_manager_find(DEVICE_TYPE_FILESYSTEM, name) で取得した vtable
汎用アダプタ（Kernel/Core/vfs/VFS.c 内の1関数、全FSドライバ共通）
```

具体的には、各 FS ドライバが直接 `vfs_driver_t` 互換の関数シグネチャ
（`find_file`/`read_file`/`write_file`/`read_at`/... 既存の `vfs_driver_t` の
フィールドと完全一致させる）で `driver_module_descriptor_t.driver_api` を
エクスポートするよう統一し、`vfs_mount()` 側は
`driver_manager_get_by_kind(DEVICE_TYPE_FILESYSTEM)` で得たドライバ一覧を
そのまま `vfs_driver_t` としてキャストして使う（型さえ合わせれば
per-FS のアダプタ関数は不要になる）。これにより:

- `Kernel/Drivers/Client/FileSystem/FAT32/FAT32_VFS_Adapter.c`
- `Kernel/Drivers/Client/FileSystem/FAT32/FAT32_Client.c`
- `Kernel/Drivers/Client/FileSystem/ISO9660/ISO9660_VFS_Adapter.c`
- （ISO9660の Client 相当ファイル、要確認）

が丸ごと削除できる。新規に exFAT を実装する際も、この汎用アダプタ経由になるため
**exFAT 用の VFS_Adapter を新規に書く必要が最初から無くなる**（要求の
「VFS_Adapter を削除」と「exFAT を実装する」が自然に両立する設計）。

### 6.3 exFAT ドライバの実装

参照実装: `Kernel/Drivers/Server/FileSystem/FAT32/FAT32_Main.c`（クラスタチェーン走査、
FAT テーブル管理、BPB パース、ディレクトリエントリ操作の実装パターン）。

1. `Kernel/Drivers/FileSystem/exFAT/exFAT_Main.h` — Boot Sector（exFAT は FAT32 と
   BPB レイアウトが異なる。`JumpBoot`, `FileSystemName="EXFAT   "`,
   `MustBeZero[53]`, `PartitionOffset`, `VolumeLength`, `FatOffset`, `FatLength`,
   `ClusterHeapOffset`, `ClusterCount`, `FirstClusterOfRootDirectory`,
   `VolumeSerialNumber`, `FileSystemRevision`, `VolumeFlags`, `BytesPerSectorShift`,
   `SectorsPerClusterShift`, `NumberOfFats`, `DriveSelect`, `PercentInUse` 等）
   の構造体を定義する。
2. `exFAT_Main.c` — 最低限の読み書き機能:
   - Boot Sector + チェックサムセクタの検証
   - Allocation Bitmap（exFAT はFATチェーンに加えてビットマップも持つ点がFAT32との
     主要な差）の読み取り
   - Up-case Table の読み取り（ファイル名の大文字小文字非依存比較に必要）
   - ディレクトリエントリセット（File Directory Entry + Stream Extension +
     File Name Entry の3点セット構造。FAT32の単純な32バイト固定長エントリとは
     異なるので実装量が最も多い部分）
   - クラスタチェーン走査（`NoFatChain` フラグが立っている場合は連続クラスタとして
     FAT参照を省略できる最適化がexFAT仕様にあるので、読み取り専用実装でもまず
     対応すること）
   - `find_file`/`read_file`/`write_file`/`read_at`/`write_at`/`truncate`/
     `get_file_size`/`creat`/`mkdir`/`opendir`/`readdir`/`closedir`/`close_file`/
     `unlink` を `vfs_driver_t` 互換シグネチャで実装（6.2の汎用アダプタに直結）
3. `Kernel/Drivers/FileSystem/exFAT/Makefile` — 他のドライバ同様
   `include ../../../module.mk` のみでよいはず（`Docs/Architecture/Driver_Module_Guide.md`
   6.3節のテンプレート参照）
4. 段階的スコープ: まず読み取り専用（`find_file`/`read_file`/`read_at`/
   `opendir`/`readdir`）を実装してブート・読み込みを動作確認し、その後
   書き込み系（`write_at`/`creat`/`mkdir`/`unlink`/`truncate`）を追加する
   2段階リリースを推奨（FAT32実装がフルスクラッチで2319行かかっている実績を踏まえ、
   一度に全機能を実装しようとしてレビュー不能になるのを避ける）

---

## 7. P3: Kernel の堅牢性・安全性強化

**進捗: 完了 (2026-08-24)**
- [x] 項目1: 固定長配列の境界チェック監査。攻撃面として最も価値の高い
      「syscall/IPCから到達可能なfd/handleテーブル」を対象に実監査を実施
      （`Syscall_File.c` の `g_files`/`g_open_files`/`g_dirs`/`g_pipes`/
      `g_timerfds`/`g_memfds`/`g_signalfds`、`Syscall_Socket.c` の
      `g_sockets`、`Syscall_Epoll.c`、`IPC/UnixSocket.c` の `g_usocks`。
      加えてネットワーク経由で到達可能な `Network/tcp/TCP.c` の
      `g_tcp_connections` インデックス生成経路も確認）。結果:
      いずれも `idx < 0 || idx >= LIMIT` 相当のチェックが（`||`の
      短絡評価込みで）漏れなく先行しており、境界チェック漏れは
      **発見されなかった**。唯一の実例（`vfs_mount()` のサイレントドロップ）は
      P2作業時に既に修正済み。**方法論の限定を明記**: 全約50個の
      固定長配列（`Kernel/Drivers/Module/*`, `Network/*` の内部free-slot
      走査ループ等）を1つずつ手動監査したわけではない
      （`for (i=0;i<N;i++)` によるフリースロット探索は配列サイズで
      構造的に安全なため優先度を下げた）。今後のPRでも同様の観点
      （外部入力が直接インデックスになる箇所を優先）でのレビューを推奨
- [x] 項目2: 重複コードの一本化。P2で `vfs_resolve_candidates()` として
      実施済み（6.1節参照）
- [x] 項目3: `-fstack-protector-strong` をカーネルの `KERNEL_CFLAGS`
      （`Kernel/config/arch.mk`）に有効化。`__stack_chk_guard`/
      `__stack_chk_fail` を新設 `Kernel/Core/hardening/StackProtector.c`
      に実装（`kernel_panic()` を呼ぶ）。ガードは起動時に
      RDTSC（x86_64）/`CNTVCT_EL0`（arm64）を使い2段階で再シード
      （`kernel_main.c` 冒頭 + timerフェーズ完了後）。既存の `-fPIE`
      コード生成と同じGOT相対アドレッシングで動作することを
      `objdump -dr` で確認済み（TLSベースの `%fs:` 方式ではない）。
      **スコープ限定**: カーネル本体のみ。ローダブルドライバモジュール
      （`module.mk`、`-fno-stack-protector` のまま）とUserland全アプリ
      （同）への拡張は、各モジュールが独立リンクされた `-shared` ELFで
      `__stack_chk_fail` を個別に持つ必要がある等の設計検討が要るため、
      本パスのスコープ外として次の課題に持ち越す
- [x] 項目4: usercopy の一貫性監査。`Syscall_Dispatch.c` は生の `memcpy`
      呼び出しゼロを確認。`Syscall_LinuxCompat.c` の8箇所の `memcpy` は
      全てカーネル側ローカルバッファ間コピーで、ユーザ空間との境界は
      必ず `copy_to_user_trusted`/`copy_from_user`/
      `process_user_buffer_is_valid` を経由していることを1箇所ずつ確認。
      抜け穴は発見されなかった
- [x] 項目5: `os_status_t` 統一は決定事項どおり段階移行で実施。本パスでの
      具体的な一歩として `vfs_mount()` を `void` → `bool` 戻り値化
      （マウントテーブル満杯を呼び出し元が検知可能に）。`os_status_t`
      という specific な型そのものの全面採用は、決定事項6の文言
      「1回の巨大な置換コミットにはしない」を踏まえ、P5で新設する
      組込みドライバ登録コード等の**新規コード**から自然に採用していく
      方針とし、既存の広範な `bool` 系APIを本パスで一括変換すること
      はしていない
- [x] 項目6: 静的解析のCI組み込みはP8（CI/CD）と不可分のため、P8に統合し
      本パスでは実施しない（決定事項5と整合）

横断的な改善なので、他フェーズと並行して継続的に適用する。優先度の高い具体項目:

1. **固定長配列の境界チェック監査**: `VFS.c` の `g_vfs_drivers[16]`,
   `g_vfs_directory_handles[32]`、`kernel_main.c` の
   `g_boot_profile[KERNEL_BOOT_PROFILE_MAX]` 等、`OS_CONFIG_*` で上限が
   定義されている全静的配列に対し、書き込み前チェックが漏れなく存在するか監査する。
   `vfs_mount()`（`VFS.c:19`）は `if (g_vfs_driver_count < 16)` チェックがあるが、
   **上限に達した場合サイレントに mount 要求を無視する**（エラーを一切通知しない）。
   ドライバマウント失敗を `kernel_panic` するか、少なくとも `debug_printf` で
   警告を出すよう修正する。
2. **重複コードの一本化**（副次的に堅牢性も上がる）: `VFS.c` の
   `vfs_find_file`/`vfs_creat`/`vfs_mkdir`/`vfs_opendir`/`vfs_unlink` に
   ほぼ同一の「最長一致プレフィックス検索」ロジックが5回コピペされている
   （6.1で言及）。共通ヘルパー `vfs_resolve_driver_for_path(path)` に切り出す。
3. **コンパイラの安全機構有効化の監査**: `Kernel/config/arch.mk` の現行フラグ
   （`-Wall -Wextra -Wtype-limits -Wconversion -Wsign-conversion -Wshadow`）に加え、
   `-fstack-protector-strong`（freestanding + `-nostdlib` 環境で
   `__stack_chk_fail` を自前実装する必要あり）、`-D_FORTIFY_SOURCE` 相当の
   自前 libc 側チェック、`-Wformat=2` の追加を検討する。
4. **usercopy の一貫性確認**: `Kernel/Core/usercopy/` がユーザ空間ポインタ検証を
   一元化している想定だが、syscall 実装（`Syscall_Dispatch.c`,
   `Syscall_LinuxCompat.c`）の全箇所がこの経路を通っているか監査する
   （ユーザポインタを直接 `memcpy` している箇所が抜け穴になりやすい）。
5. **エラーモデルの一貫性**: `Kernel/include/kernel/status.h` の `os_status_t`
   を返さない古いスタイルの `bool` 戻り値関数（VFS層はほぼ全て `bool`）が
   混在している。エラー原因が呼び出し元に伝わらない設計になっているため、
   `os_status_t` への統一を行う。**ユーザー確定事項**: 別タスクに切り出さず、
   本リファクタリングのスコープに含める。ただし影響範囲が非常に広い
   （VFS/ドライバAPI/syscall層すべてに波及する）ため、実施順序は
   「P2（VFS刷新）で新設・書き換えする関数から `os_status_t` を採用」→
   「P1でドライバAPIを触るタイミングで `driver_*` 系関数も追随」→
   「残る `bool` 戻り値関数を横断的に置換する仕上げパス」という段階移行で行い、
   1回の巨大な置換コミットにはしない。
6. **静的解析の CI 組み込み**（P0-aと連動）: `clang-tidy`/`cppcheck` を導入し、
   NULL チェック漏れ・整数オーバーフロー・未初期化変数を継続的に検出する。

---

## 8. P4: x86_64 / arm64 機能差の解消

**進捗: 完了 (2026-08-24)**
- [x] 8.1 item1: `arch_ops_t` に `get_timer_hal()` を追加
      （`Kernel/include/interfaces/arch_ops.h`）。x86_64は`&lapic_timer_hal`、
      arm64は`&generic_timer_hal`を返す実装を各 `arch_ops.c` に追加し、
      `kernel_main.c` の `#ifdef PLATFORM_X86_64 timer = &lapic_timer_hal;
      #elif ...` を `ops->get_timer_hal()` 呼び出しに置換
- [x] 8.1 item2: x86_64専用の `else { init_gdt(); init_idt(); }`
      フォールバックを削除。両アーキとも `arch_ops_t.init_cpu_tables` が
      非NULLである前提の単一コードパスに統一
- [x] 8.1 item3: 仮想化初期化を囲っていた `#ifdef PLATFORM_X86_64` を撤去。
      arm64 の `virtualization_init` は既に「非対応」を意味する `-1` を
      返す明示的スタブだったため、`if (ops && ops->virtualization_init)`
      のNULLチェックのみで両アーキ共通コードとして安全に動作する
- [x] 8.1 item4: `#if defined(PLATFORM_X86_64) || defined(PLATFORM_ARM64)`
      （恒真式の死んだ分岐）を削除。`timer_switch_lapic()` 自体が
      `g_timer_hal->switch_to_local` のNULLチェックで安全にno-opする
      設計だったため、ガード自体が不要だった
- [x] 8.1 item5: `Kernel/Core/elf/ELF_Loader.c` の `#ifdef` 4箇所を監査
      （ELFマシンタイプ判定・命令キャッシュ同期・再配置タイプ処理・
      追加ページマッピング）。いずれもISAごとに本質的に異なる領域
      （0.4節参照）で、`arch_ops_t` 越しの統一は不要と判断し変更なし
- [x] 8.2: `GDT_Main.h`/`IDT_Main.h`（arm64）は実際には
      `Kernel/Arch/arm64/cpu/Exception.c` が全関数を実装しており
      `GenericTimer.c`/`Exception.c` から実使用されていることを確認
      （0.4節のP0時点調査の誤りを訂正）。削除・ガード追加とも不要
- [x] 8.3: ドライバのアーキ依存性を棚卸し。ビルドシステムはARCH別の
      ドライバ除外を一切行っていない（`PS2_Driver.ELF` はarm64ビルドでも
      常にビルド・ステージされ、対応ハードウェアが無い環境では単に
      初期化に失敗するだけ）ことを確認し、`Docs/Architecture/
      Kernel_Architecture.md` 10.1節に明記。決定事項どおり
      「アーキ専用ドライバの存在」自体は是正対象としていない
- [x] `make ARCH=x86_64 all` → `image_livecd` → QEMU (OVMF/q35) 実機起動で
      P3+P4合算の回帰確認（後述）
- [x] `make ARCH=arm64 kernel` でのビルド確認も試行。**既存の無関係な
      未解決課題を発見**: `libc/I_libc/src/stdio.c` の `vsnprintf` が
      `long double` 経由で `__trunctfdf2`（`long double`→`double`変換の
      libgcc組込みシンボル）を要求するが、`-nostdlib` freestanding
      リンクにこのシンボルを供給する場所が無く `aarch64-elf-ld` が
      undefined reference で失敗する。P3/P4で新設・変更した
      `arch_ops.o`/`StackProtector.o` 等は正常にコンパイル・リンク
      対象に含まれており原因ではない（エラーは `stdio.o` 単体に限定）。
      本リファクタリングのP3/P4スコープ外の既存バグのため本パスでは
      未修正。arm64 の実機/QEMUブート確認は今回未実施（x86_64のみ
      実施）である点も明記しておく

### 8.1 `kernel_main.c` の `#ifdef` 削減

`arch_ops_t`（`Kernel/include/interfaces/arch_ops.h`）を拡張し、
`kernel_main_after_stack_switch()` 内の分岐を全て vtable 経由に統一する:

1. **タイマHAL選択**（`kernel_main.c:341-345,403-408`）:
   `extern const timer_hal_t lapic_timer_hal;` / `generic_timer_hal` を
   `#ifdef` で出し分けている箇所を、`arch_ops_t` に
   `const timer_hal_t *(*get_timer_hal)(void);` を追加して置換する。
2. **CPUテーブル初期化**（355-372行目）: 既に `ops->init_cpu_tables` は
   存在するが、x86_64側だけ `else { init_gdt(); init_idt(); }` という
   フォールバックが残っている。フォールバックを削除し
   `x86_64/arch_ops.c` 側で必ず `init_cpu_tables` を設定するよう保証する。
3. **仮想化初期化**（421-428行目）: `#ifdef PLATFORM_X86_64` で丸ごと
   囲われているが、`arch_ops_t.virtualization_init` は既に存在する
   （arm64側は現状 `NULL` のはず、要確認）。`#ifdef` を外し、
   `if (ops && ops->virtualization_init)` の NULL チェックだけで
   両アーキ共通コードにする。arm64 側は
   `Kernel/Arch/arm64/arch_ops.c` で `virtualization_init` に
   「no-op で成功を返す」スタブを明示的に設定する（暗黙のNULLに頼らない）。
4. **`timer_switch_lapic()` の呼び出しガード**（436行目）:
   `#if defined(PLATFORM_X86_64) || defined(PLATFORM_ARM64)` は
   ビルド対象アーキが必ずどちらかである以上恒真式であり、意味のない
   死んだ条件分岐。削除する。
5. `Kernel/Core/elf/ELF_Loader.c` 内の同種の `#ifdef` も同じ方針で
   `arch_ops_t` 経由に揃える（要調査: 具体的にどの分岐か）。

### 8.2 GDT/IDT ヘッダの扱い

`Kernel/Arch/arm64/cpu/GDT_Main.h`, `IDT_Main.h` はヘッダのみ存在し `.c` 実体が
無い（0.4節）。これは「x86_64専用ヘッダを共有includeパスの都合で空でも置いている」
状態と推測されるが、実際に誰がこれをincludeしているか確認し、
- 誰も参照していなければ削除
- ビルドの都合上必要なら、`#error "GDT is x86_64-only"` 等の明示的なガードを
  入れて「意図的に空である」ことをコードで表現する

### 8.3 ドライバのアーキ依存性監査

`Kernel/Drivers/` 配下の各ドライバが x86_64 専用ハードウェア（例: `PS2`は
レガシーPC/AT互換のみ）を対象にしているのか、PCI/USB経由で本質的に
アーキ非依存（AHCI/NVMe/VirtIOBlk/VirtIONet/USB各種コントローラ等）なのかを
`Makefile`（`driver_build`ターゲット）のビルド条件で棚卸しし、
`Docs/Architecture/Kernel_Architecture.md` 10.1/10.2節（現状記載済みだが
コード変更に追随していない）を正として更新する。**「アーキ専用が存在すること」
自体は正常であり全廃する対象ではない**点に注意（PS/2やVMXのように
ハードウェア自体がアーキ固有なものは差があって当然。差を無くす対象は
「同じ概念のはずなのに実装方針が割れているもの」＝8.1のような
カーネル内部の抽象化漏れ）。

---

## 9. P5: プラットフォーム/ネットワークサブシステムのドライバ化

**進捗: 完了 (2026-08-24)**
- [x] 9.1: `device_type_t` に `DEVICE_TYPE_PLATFORM` を追加。新設
      `Kernel/Drivers/Module/PlatformBuiltinDrivers.c` が
      ACPI/Timer/LAPIC/IOAPIC（x86_64）または ACPI/Timer/GIC（arm64）を
      `driver_manager_attach()` でDeviceRegistryに登録する。呼び出しは
      `kernel_main.c` の `driver_module_manager_init()` 完了**直後**
      （DeviceRegistryが存在する最初のタイミング）に配置し、
      各サブシステム自体の初期化呼び出し順序・タイミングは一切変更していない。
      **スコープ限定**: `Kernel/Platform/acpi/` 等の物理ディレクトリ移動
      （`Kernel/Drivers/Platform/{ACPI,IOAPIC,LAPIC,Timer}/`）は
      **実施していない**。理由: この移動はブート最重要パスの
      include参照を横断的に書き換える必要があり、9.1本文が明記する
      「起動シーケンスのリスクを増やさないことを最優先する」という
      優先順位と衝突するため、DeviceRegistry登録という**機能要件**を
      満たすことを優先し、パスの物理移動は見送った
- [x] 9.2: ユーザー承認によりハイブリッド案を採用（当初案の全数個別
      `.ELF` 化は、7層が直接C関数呼び出しで密結合しており「1カテゴリずつ
      安全に移行」が効かず、QEMU起動スモークテストでは検出できない
      機能退行リスクが高いと判断したため、実装着手後に方針を確認して決定）。
      `device_type_t` に `DEVICE_TYPE_NET_PROTOCOL` を追加。新設
      `Kernel/Drivers/Module/NetworkBuiltinDrivers.c` が
      Ethernet/ARP/IPv4/ICMP/UDP/TCP/DHCPの7層それぞれを
      `"<Name>_Driver"` という個別名でDeviceRegistryに登録し、各層の
      **実際の公開関数ポインタから成る本物のvtable**（`ethernet_builtin_
      driver_t` 等、7種の型）と、実際の `#include` 依存関係から
      調査した正確な `deps[]`（例: `DHCP_Driver` → `{IPv4_Driver,
      UDP_Driver}`）を持たせた。**スコープ限定**: `Kernel/Network/` 内部の
      層間呼び出し（例: `ARP.c` が `ethernet_send()` を直接呼ぶ）は
      vtable経由（`driver_manager_find()`）への書き換えを**行っていない**
      ---既存の直接C関数呼び出しのまま。よって `deps[]` は現状
      ドライバロード順序解決ロジック（`DriverModule.c`）には接続されて
      おらず、純粋に「正確な依存関係のドキュメント化」として機能する。
      将来、層ごとに段階的にvtable経由へ切り替える、または真の個別
      `.ELF` 化に進む場合の土台として設計してある
- [x] `make ARCH=x86_64 all` → `image_livecd` → QEMU実機起動で
      P5合算の回帰確認（後述、パニック/アサート無く起動完了を確認）

### 9.1 ACPI/Timer/LAPIC/IOAPICの組込みドライバ化 (元計画)

**ユーザー確定事項**: 案Aを採択。ACPI/Timer/LAPIC/IOAPIC は「組込みドライバ」として
扱う。ネットワークプロトコルスタック（9.2節）は個別ELFに分割する方針も確定
（9.2節参照）。

- **案A（採択）**: 「組込みドライバ（built-in driver）」という第三のドライバ種別を
  導入する。Linux の `built-in.o`（カーネルに静的リンクされるが `struct
  device_driver`/`module_init()` という**モジュールと同じ登録インターフェース**を使う
  ドライバ）に相当する。ACPI/Timer/LAPIC/IOAPIC を `driver_module_descriptor_t` /
  `driver_binary_t` と同じ vtable 形状のまま、`.ELF` に分離せず
  `Kernel/Drivers/Platform/{ACPI,Timer,LAPIC,IOAPIC}/` にソースを置いて
  **カーネル本体に直接リンクし、`driver_module_init()` 相当の初期化関数を
  カーネル起動シーケンスの中で直接呼び出す**。DeviceRegistry/DriverManager には
  通常のロード済みドライバと同じ形で登録されるため、`OSDebug` 等から見た
  「ドライバ一覧」には他のモジュールと区別なく並ぶ。**この案はニワトリタマゴ問題を
  回避しつつ「ACPI/Timer/LAPIC/IOAPICをドライバとして扱う」という要求を実現する。**
- **案B**: ACPI/Timer/LAPIC/IOAPIC は現状のまま `Platform/` に残し、
  「ドライバとして扱う」要求は ARP/DHCP/Ethernet/ICMP/TCP/UDP/IPv4（起動シーケンス上
  制約が無い）だけに適用する。実装コストは低いが、要求を部分的にしか満たさない。

以降は案Aを前提に記述する。

### 9.1 ACPI/Timer/LAPIC/IOAPIC の組込みドライバ化

1. `Kernel/Platform/acpi/`, `interrupt/`, `timer/` を
   `Kernel/Drivers/Platform/{ACPI,IOAPIC,LAPIC,Timer}/` へ再配置
2. 各サブシステムに `driver_module_descriptor_t` 相当の記述子を持たせる
   （`driver_api.h` の `device_type_t` に `DEVICE_TYPE_PLATFORM` 等の新カテゴリを
   追加する必要がある）
3. `kernel_main.c` からの直接関数呼び出し（`acpi_init()`, `timer_init()` 等）は
   維持しつつ、内部で `driver_manager_attach()` を呼んで DeviceRegistry に
   登録するようにラップする（呼び出し順序・タイミングは一切変えない。
   **見た目の登録のされ方だけを他ドライバと統一する**のがこのタスクの本質で、
   起動シーケンスのリスクを増やさないことを最優先する）
4. arm64 側は LAPIC/IOAPIC の代わりに GIC が対象になる。案Aの枠組みは
   アーキ非依存なので `Kernel/Drivers/Platform/GIC/` として同様に扱う
   （8章のアーキ差解消と合流するタスク）

### 9.2 ARP/DHCP/Ethernet/ICMP/TCP/UDP/IPv4 の真のドライバモジュール化

こちらは `network_stack_init()` が既にドライバモジュール初期化後の遅いタイミングで
呼ばれているため（0.5節）、真に `.ELF` として分離しロード可能にできる。

1. `Kernel/Network/{arp,dhcp,ethernet,icmp,tcp,udp}/` を
   `Kernel/Drivers/Net/{ARP,DHCP,Ethernet,ICMP,TCP,UDP,IPv4}/` へ再配置
   （IPv4は現状どこに実装があるか要確認。`Network/` 直下に無ければ
   `ethernet` か新規ディレクトリに存在するはず）
2. 各プロトコル層を `driver_module_descriptor_t` としてエクスポートし、
   `device_type_t` に `DEVICE_TYPE_NET_PROTOCOL` 等を追加。既存の
   `driver_nic_t`（物理層/リンク層のNICドライバ）とは別カテゴリとして、
   プロトコルスタックが「どのNICデバイスの上で動くか」を
   `bus_matches[]` ならぬ **`net_stack_bind(nic_device_name)`** のような
   明示APIで結びつける設計にする（プロトコルはPCI/USBのような「バス」に
   実際には繋がっていないため、既存の `bus_matches` の仕組みをそのまま
   転用するのは無理があり、新規の結線APIが必要）
3. **ユーザー確定事項**: 「1個の `NetStack_Driver.ELF` にまとめる」妥協案は採らず、
   `ARP_Driver.ELF` / `DHCP_Driver.ELF` / `Ethernet_Driver.ELF` / `ICMP_Driver.ELF` /
   `TCP_Driver.ELF` / `UDP_Driver.ELF` / `IPv4_Driver.ELF` として**個別ELFに分割**
   する。レイヤ間の依存順序（Ethernet → ARP/IPv4 → ICMP/UDP/TCP → DHCP）は
   `DRIVER_MAX_DEPS`（`driver_api.h:16`、現状4件までの依存表現）で表現する。
   4件では足りない依存関係が出た場合（例: DHCPはEthernet/ARP/UDP/IPv4の4つに
   依存し得るため境界事例）は `DRIVER_MAX_DEPS` を増やすか、`deps[]` を
   可変長にする改修を行う。ロード順序は既存の `driver_module_manager_init()` の
   依存解決ロジック（`deps[]` を見て未ロードの依存モジュールを先にロードする
   仕組みが既にあるはずなので `Kernel/Drivers/Module/DriverModule.c` で要確認）
   をそのまま流用する。

---

## 10. P6: Linux 互換層の独立サブシステム化

**進捗: 完了 (2026-08-24)**
- [x] 項目1: `git mv Kernel/Core/syscall/Syscall_LinuxCompat.c
      Kernel/Compat/Linux/Syscall_LinuxCompat.c`（2991行）。全ての
      `#include` が `Kernel/` ルート相対パスだったため、移動に伴う
      パス修正は不要だった。新設 `Kernel/Compat/Linux/
      Syscall_LinuxCompat.h` で `linux_syscall_dispatch()` と新設
      `linux_compat_layer_register()` を宣言（従来
      `Syscall_Dispatch.c` 内に手書きされていた `extern` 前方宣言を廃止）。
      `Kernel/Compat/Makefile`（`Core/Makefile` 等と同じ `find` パターン）
      を新設し、トップレベル `Kernel/Makefile` の `include` チェーンに追加
- [x] 項目2: `Kernel/Compat/compat_types.h`（`compat_layer_t`/
      `compat_dispatch_fn_t`/`process_abi_t`）と
      `Kernel/Compat/compat_registry.c`（`compat_registry_init`/
      `_register`/`_find`、最大4層、名前衝突時は再登録で上書き）を新設。
      `Syscall_Dispatch.c` の `if (process_get_current_abi_mode() ==
      PROCESS_ABI_LINUX) { linux_syscall_dispatch(...); }` を
      `compat_registry_find(process_get_current_abi_mode())` 経由の
      ルックアップに置換。登録自体は `Syscall_Init.c` の `syscall_init()`
      （`compat_registry_init()` + `linux_compat_layer_register()`）で行い、
      **`Syscall_Dispatch.c` は今後 Linux 固有識別子を一切知らずに済む**
      設計になった（将来のOS互換層追加は `Syscall_Init.c` に1行足すだけ）。
      `compat_dispatch_fn_t` のシグネチャは計画内の例示的な汎用形
      （`(num, a1...)`）ではなく、実際に存在する
      `linux_syscall_dispatch(saved_rsp, num, arg1..arg6)` の実シグネチャに
      合わせた（存在しない層のために不要な抽象化をしないという判断）
- [x] 項目3: `process_abi_t` を `Kernel/Compat/compat_types.h` に新設。
      **スコープ限定**: `Kernel/Core/process/ProcessScheduler.h` の
      `PROCESS_ABI_IMPLUS`/`PROCESS_ABI_LINUX` マクロ（生の `uint8_t` 比較）
      は変更していない。理由: プロセス/スケジューラサブシステムは
      カーネル全体から極めて広範囲に参照されており、新しい名前付き型を
      そこに導入するのは本フェーズのスコープ（Linux互換層自体の分離）を
      超えるリスクを伴うため。`compat_types.h` の `process_abi_t` は
      同じ値集合に対する `Kernel/Compat/` 側の型付きビューとして機能する
- [x] 項目4: POSIX層（`Userland/POSIX/`）とのカーネル/ユーザランド境界の
      違いのドキュメント化は、計画どおりP7（ドキュメント刷新）に実施
- [x] `make ARCH=x86_64 all` → `image_livecd` → QEMU実機起動で
      P6合算の回帰確認（後述）

1. `Kernel/Compat/Linux/` を新設し、`Kernel/Core/syscall/Syscall_LinuxCompat.c`
   と、それが依存する Linux 固有の定数・構造体変換（errno マッピング、
   `uname` 文字列、`stat`/`timespec` レイアウト変換等）をここに移動する
2. `Kernel/Core/syscall/Syscall_Dispatch.c` の
   `if (abi == PROCESS_ABI_LINUX) { linux_syscall_dispatch(...); }` を、
   将来の他OS互換層追加を見据えた**互換レイヤ登録テーブル**に置き換える:
   ```c
   typedef int64_t (*compat_dispatch_fn_t)(uint64_t num, uint64_t a1, ...);
   typedef struct {
       process_abi_t abi;
       const char *name;
       compat_dispatch_fn_t dispatch;
   } compat_layer_t;
   ```
   `Kernel/Compat/Linux/Syscall_LinuxCompat.c` はこの `compat_layer_t` を
   1つエクスポートし、`Kernel/Compat/compat_registry.c`
   （新設、`Kernel/Core/syscall` からは独立させる）が起動時に登録を集約する
3. ABI 判定自体（`ELF_Loader.c` の `EI_OSABI==ELFOSABI_LINUX` 検出）は
   ローダの責務のまま残すが、判定結果の型 `process_abi_t` の定義を
   `Kernel/Compat/compat_types.h` に移し、将来 `PROCESS_ABI_BSD` 等を
   追加する際に `Kernel/Compat/` だけを見ればよい状態にする
4. `Userland/POSIX/` との関係整理: POSIX 層はユーザランド側の互換レイヤであり
   `Kernel/Compat/Linux/`（カーネル側のLinux syscall番号ABI互換）とは
   **別の層**であることをドキュメントで明記する（フェーズ7）。両者を混同すると
   将来の変更で境界がなし崩しになる

---

## 11. P7: ドキュメント刷新

**進捗: 完了 (2026-08-24)**
- [x] 11.1: 既存ドキュメントは**各フェーズ完了時にその場で**追随修正する
      という方針どおり実施済み（P1で `Driver_Module_Guide.md`、P2/P4で
      `Kernel_Architecture.md`、P0-cで各種パス参照、等。本フェーズでの
      まとめ作業は不要だった）
- [x] 11.2 新規ドキュメント4本すべて作成:
      - [x] `Docs/Architecture/VFS_and_FileSystems.md` — P2後のVFS設計、
            `vfs_driver_t`契約、`vfs_resolve_candidates()`、最小読み取り
            専用FSドライバのコード例、exFAT実装の技術解説
      - [x] `Docs/Architecture/Compat_Layers.md` — `Kernel/Compat/`の設計、
            POSIX層との違いの表、新規互換層追加手順のコード例（P6後）
      - [x] `Docs/Architecture/Network_Stack.md` — P5ハイブリッド設計の
            詳解、7層のvtable一覧、**「なぜ真の個別ELF化をしなかったか」
            の技術的理由を正直に記載**
      - [x] `Docs/Architecture/Boot_Sequence.md` — 19フェーズの表、実際に
            QEMUで取得した `boot_profile_dump()` 出力例と読み方、P4の
            変更が起動シーケンス自体には影響しないことの明記
      - [x] `Docs/Architecture/CI_CD.md` — P8のワークフロー説明（4章参照）
- [x] 11.3: 英語で記述された既存の `Docs/Architecture/*.md` 群の慣習
      （表・コードブロックを積極使用）に合わせ、新規4文書も英語で統一。
      全て冒頭に "Last reviewed: 2026-08-24" を明記（11.3が求める
      「調査/更新基準日」の明記慣習）

「増やすだけでなく、しっかりとした Markdown として書き、コード例も書く」という要求に
沿って、以下を実施する。

### 11.1 既存ドキュメントの修正

`Docs/Architecture/Driver_Module_Guide.md` と `Kernel_Architecture.md` は
**既に実コードと乖離している**ことが今回の調査で判明した。例:
- `driver_binary_t` の実際の定義（`driver_api.h:254-295`）は `hal`/`pci`/`event`/
  `system`/`fs`/`bus` の6サブvtableを持つが、ドキュメント記載（57–105行目）は
  旧世代のフラットなフィールドのみで大幅に古い
- `driver_module_descriptor_t` の実際の定義（同648–652行目）には
  `bus_matches`/`bus_match_count`/`probe`/`remove` があるが、ドキュメント
  （107–120行目）には無い
- `exFAT_Driver.ELF` を実装済みとして記載している（368–371行目）が実体はゼロバイト

これらは各フェーズ完了時に**その場で**追随修正すること（ドキュメント刷新を
最後にまとめてやると、その間に何度も乖離が蓄積し手に負えなくなる）。

### 11.2 新規ドキュメント

- `Docs/Architecture/VFS_and_FileSystems.md` — VFS設計（P2後の名前非依存設計）、
  FSドライバの書き方、exFAT実装の解説。コード例: 最小の読み取り専用FSドライバを
  1つ書き下ろす
- `Docs/Architecture/Compat_Layers.md` — `Kernel/Compat/` の設計、
  新しいOS互換層を追加する手順をコード例付きで説明（P6後）
- `Docs/Architecture/Network_Stack.md` — プロトコルスタックのドライバ化後の
  構成、レイヤ間結線APIの使い方
- `Docs/Architecture/Boot_Sequence.md` — `kernel_main.c` の初期化順序を
  フェーズごとの図と共に詳細化（現状 `Kernel_Architecture.md` 2.2節に簡易版が
  あるが、独立ドキュメントとして起こし、`boot_profile_dump()` の出力を
  読み解く実例も載せる）
- `Docs/Architecture/CI_CD.md` — P0-aで構築したワークフローの説明、
  ローカルでの再現手順

### 11.3 スタイル

既存の `Docs/Others/TODO_Chromium_LinuxABI.md` や本ドキュメントに倣い、
日本語で記述する場合はコード識別子・パスは英語のまま保持し、
表・チェックリスト・コードブロックを積極的に使う。全ドキュメントに
「調査/更新基準日」を明記する慣習を踏襲する。

---

## 12. 決定事項一覧（2026-08-23 ユーザー回答により確定）

かつての「未決定事項」6項目は以下の通り全て確定した。各項目の詳細な反映先は
該当セクションを参照。

| # | 論点 | 確定内容 | 反映先 |
|---|---|---|---|
| 1 | `Kernel/Driver` の改名範囲 | ソースツリーのみ統合。ブートイメージ上のパス規約は現状維持（技術的必然性が無いため） | 4章 |
| 2 | Client/Server 廃止後のディレクトリ名 | 5.1節の提案どおり採用 | 5.1章 |
| 3 | ACPI/Timer/LAPIC/IOAPIC の扱い | 案A（組込みドライバ）を採択 | 9章, 9.1章 |
| 4 | プロトコルスタックの分割粒度 | 個別ELF（`ARP_Driver.ELF`等）に完全分割 | 9.2章 |
| 5 | CI/CD 整備の優先度 | 後回し。P8として最終フェーズに移動、本プロジェクトでは重要度低 | 1章, 2章 |
| 6 | `os_status_t` エラーモデル統一のスコープ | 本リファクタリングに含める。ただし段階移行で実施 | 7章5項 |

---

## 13. 進捗管理

各フェーズ着手時に、このファイルの該当セクションへ `- [ ]` 形式のチェックリストを
追記し、完了したサブタスクにチェックを入れていく運用とする（フェーズ0の現状調査は
本ドキュメント作成時点で完了しているため、以降のセクションから運用を開始する）。
