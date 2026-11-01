# Intel Gen9.5 / i915 相当ドライバ実装仕様

## 目的と対象

本仕様は ImplusOS の `Kernel/Source/Drivers/Display/IntelGen9/IntelGen9.c` を、Linux `i915` の機能範囲に近づけるための実装設計である。既存ソースに記載された対象機種に合わせ、第一段階では Coffee Lake / Whiskey Lake / Comet Lake の Gen9.5 を対象とする。Skylake / Kaby Lake / Cannon Lake 等の追加は、platform-specific register、power well、display PLL、DDI PHY、GT workaround を個別に確認してから行う。

**この文書は完成ドライバではなく、実装・検証可能な単位へ分割した仕様である。** このソースダンプだけから未検証の表示 PLL / DDI / AUX / GT command encoding を推測して動かすパッチを作ることは、画面消失や GPU hang の危険があるため行わない。

## Linux i915 相当の完了定義

最低限、次を満たしたものを「i915 相当の基本機能」とする。

1. 対応 PCI ID / stepping / platform 情報の検出、BAR / OpRegion / VBT の取得、初期化失敗からの安全な復帰。
2. Display KMS: connector / encoder / CRTC / plane の列挙、EDID とモード検出、HDMI / DP 出力、Atomic check / commit、page flip、hot-plug。
3. Memory: GGTT と per-process PPGTT、GPU virtual address 管理、GEM 相当の buffer object、pin/unpin、cache coherency、fence / sync object。
4. GT: 少なくとも RCS と BCS の engine discovery、context、command submission、scheduler、interrupt/fence completion、hang detection、reset/recovery。
5. Firmware: Gen9 で必要な DMC、任意機能を含む GuC / HuC の正しい firmware version 検証とロード・失敗処理。
6. Power: forcewake の参照カウント、display power well、runtime suspend/resume、system suspend/resume。
7. User ABI: GPU object の作成・破棄・map、job submit、sync、mode enumeration/modeset をユーザープロセスへ安全に公開する API。
8. 実機検証: 複数の Gen9.5 SKU、HDMI/DP モニタ、複数解像度、長時間 GPU 負荷、ハング復旧、再起動・サスペンド復帰。

Video decode/encode (VCS)、VECS、audio、PSR、FBC、HDCP、MST、debug/perf API は上記基本機能の後に対応する拡張領域とする。Linux i915 の機能をすべて含む完全互換を名乗るには、これらも別途実装・検証が必要。

## 現在の ImplusOS ソースから確認できたこと

### 既存実装

- `IntelGen9.c` は独立した `driver_display_t` モジュールで、ファームウェアが設定済みの linear framebuffer を使い、BCS で back buffer を転送する設計。表示 pipe / plane / PLL / port を設定する実装ではない。
- 対応 PCI ID は同ファイルの allowlist に限定される。QEMU など Intel GPU のない環境では probe が false となり、generic framebuffer driver を使う。
- `driver_display_t` が公開するのは draw/fill/present、幅・高さ、monitor/mode 情報、`set_mode` 等の簡易 callback であり、connector / encoder / CRTC / plane / atomic state の object model はない。
- `DriverBinary.h` API v2.3 には PCI config / BAR / MSI-X / IRQ handler 登録、イベント、VFS read、DMA allocation、MMIO/WC mapping がある。これは低レベル bring-up の土台にはなる。
- DMA API は `dma_alloc[_ex]` / `dma_free` / `virt_to_phys` が中心で、scatter-gather table、IOMMU map/unmap、range 単位の `dma_sync_for_device/cpu`、GPU VM の page pinning/eviction API は公開されていない。
- ACPI 公開 API は現在 `acpi_info_t` を返す構造で、一般の ACPI table (例: OpRegion/VBT に関係する情報) を検索・取得する API は確認できない。
- `Display_Main.c` は選択済み display callback を呼び出す層であり、DRM device node、GEM ioctl、execbuffer UAPI、syncobj、atomic KMS UAPI は確認できない。

### この差が意味すること

現状の IntelGen9 は「ファームウェアがすでに点灯させた画面の framebuffer を高速転送する表示ドライバ」であり、GPU をユーザー空間の 3D / compute / media から利用させる i915 型 driver ではない。既存モジュールの数十〜数百行を追加するだけでは、この境界を越えられない。

## 推奨ソース構成

既存の Display module へ機能を追加し続けず、PCI GPU parent module と各 subsystem を分離する。

```text
Kernel/Source/Drivers/GPU/IntelGen9/
  IntelGen9.c                  # PCI parent module / lifecycle
  ig9_device.[ch]              # PCI IDs, platform/stepping capabilities
  ig9_uncore.[ch]              # MMIO, forcewake, power-domain access
  ig9_irq.[ch]                 # MSI-X/interrupt status/worker dispatch
  ig9_firmware.[ch]            # OpRegion, VBT, DMC/GuC/HuC image loader
  ig9_gtt.[ch]                 # GGTT allocator and PTE/TLB invalidation
  ig9_ppgtt.[ch]               # per-process GPU page tables / VM
  ig9_gem.[ch]                 # buffer objects, pinning, mappings
  ig9_engine.[ch]              # RCS/BCS/VCS/VECS discovery and engine state
  ig9_context.[ch]             # HW contexts and context switching
  ig9_submit.[ch]              # validated command submission and fences
  ig9_sched.[ch]               # queues, priorities, preemption policy
  ig9_reset.[ch]                # hang detection, engine/GT reset, recovery
  ig9_power.[ch]                # runtime PM, RC6, suspend/resume
  display/
    ig9_kms.[ch]                # atomic state/check/commit
    ig9_connector.[ch]          # connector/encoder inventory
    ig9_edid.[ch]                # EDID parsing and mode validation
    ig9_ddc.[ch]                 # DDC/I2C transport
    ig9_dp_aux.[ch]              # DP AUX and link training
    ig9_hdmi.[ch]                # HDMI port and mode programming
    ig9_pll.[ch]                 # platform clock/PLL calculations
    ig9_pipe.[ch]                # CRTC/timing generator
    ig9_plane.[ch]               # primary/sprite/cursor planes
    ig9_hpd.[ch]                 # hot-plug detection and debounce
  tests/
    ig9_mmio_test.c
    ig9_gtt_test.c
    ig9_edid_test.c
    ig9_atomic_test.c
    ig9_submit_test.c

Kernel/Source/Graphics/GPU/
  gpu_device.[ch]                # common GPU device/object interface
  gpu_vm.[ch]                    # process address spaces
  gpu_sync.[ch]                  # fences and synchronization objects
  gpu_submit.[ch]                # kernel job submission interface
  gpu_uapi.[ch]                  # syscall/ioctl-like user ABI

Kernel/Source/Graphics/Display/
  kms_core.[ch]                  # connector/CRTC/plane objects and atomic commit
```

`GPU` 共通層は Intel 専用レジスタを持たず、Intel Gen9 実装は backend として接続する。VirtIO GPU 等との共通化は、GPU object/sync/submit の実際の契約が固まった後に行う。

## 必要なカーネル基盤の追加

### 1. DMA / memory

- scatter-gather page list (physical page array + lifetime/refcount)。
- DMA map/unmap と方向指定の cache sync API。
- GPU VM 用 page pin/unpin、物理ページ参照の寿命保証。
- CPU mapping と GPU mapping の cache attribute / access policy。
- IOMMU がない環境でも動く direct DMA backend と、将来の IOMMU backend を分離。
- GPU fault / object eviction を実装する前に、少なくとも pinned object と確実な cleanup を提供する。

### 2. 同期と実行コンテキスト

- 割り込みハンドラから重い処理をしないための deferred work / worker queue。
- wait queue / completion / timeout / cancel の明確な契約。
- engine reset と device removal の間に job / DMA memory を解放しない lifetime 管理。
- scheduler と per-process GPU context を保護する lock / reference primitives。

### 3. ACPI / firmware

- ACPI table lookup API と table length/checksum validation。
- Intel OpRegion の安全な検出・map・検証。
- VBT parser は version / section bounds / checksum を検証し、未知の block は無視する。
- VFS firmware loader は path・最大サイズ・CSS header・firmware version・checksum を検証する。
- DMC/GuC/HuC firmware が存在しない場合の機能縮退を定義し、必須機能と optional 機能を区別する。

### 4. Display API

現在の `driver_display_t` に単純な `set_mode` callback を増やすのではなく、次の object model を追加する。

- connector: HDMI/DP/eDP、接続状態、EDID、対応モード。
- encoder: 出力 port / PHY / protocol capability。
- CRTC: timing、clock、active state。
- plane: primary / cursor / sprite、format、stride、source/destination rect。
- atomic state: 変更前の check と commit を分離し、commit 失敗時は旧状態を維持。
- page-flip completion event と vblank sequence。

### 5. User ABI

Linux Mesa を使いたい場合は i915 UAPI 互換性を目標にする必要がある。OS 独自 API にする場合でも、buffer object、VM、context、exec/job submission、sync、display modeset の機能契約は必要。安全なユーザー memory copy、プロセスごとの handle namespace、権限検査なしにユーザー batch buffer を GPU に渡してはいけない。

## 実装フェーズと exit criteria

### Phase A — parent/core と安全な起動

- PCI probe を allowlist + platform capability table に分離。
- BAR0 / OpRegion / MMIO range を検証し、所有権と解放規則を確立。
- forcewake を単純な set/clear から参照カウント・domain 管理へ変更。
- IRQ/MSI-X、event、worker queue、device removal の寿命管理。
- generic framebuffer fallback を常に維持。

**Exit:** 対応/非対応 GPU で probe が正しく分岐し、初期化失敗時に画面が維持される。BAR/MMIO と IRQ の leak がない。

### Phase B — display discovery と KMS

- OpRegion/VBT parse、DDC/EDID、DP AUX、HDMI/DP port discovery。
- connector/encoder/CRTC/plane object model。
- platform-specific PLL、DDI PHY、transcoder、pipe/plane programming。
- atomic check/commit、vblank/page flip、HPD interrupt。

**Exit:** 実機で HDMI と DP の双方について EDID 読み出し、複数モードの modeset、画面復帰、抜き差しができる。失敗時に firmware mode へ戻せる。

### Phase C — Memory/GEM/VM

- GGTT allocator、PPGTT page-table levels、PTE cache/TLB invalidation。
- GEM-like object、handle、pinning、mmap、CPU/GPU coherency。
- per-process GPU VM と object lifetime。

**Exit:** 複数プロセスに分離した GPU VA space ができ、allocate/map/unmap/release を反復しても PTE leak / use-after-free がない。

### Phase D — GT engine / submit

- RCS/BCS engine discovery、ring/context、completion fence。
- command parser/validator、job queue、scheduler、timeout。
- interrupt-driven completion、hangcheck、engine reset/GT reset。

**Exit:** trusted self-test だけでなくユーザージョブを分離して実行でき、timeout/hang 後に他プロセスと display を破壊せず復旧する。

### Phase E — GuC/HuC/DMC と media

- firmware version matrix、WOPCM layout、load/auth sequence。
- GuC submission は fallback scheduler と並行して導入。
- VCS decode/encode、VECS の codec/format capability を SKU ごとに公開。

**Exit:** firmware なし/不一致/ロード失敗の各状態で起動が安全に縮退し、対応 codec のテストを完了。

### Phase F — power / reliability / tuning

- RC6 / frequency / runtime PM / display power wells。
- suspend/resume、reset state restore、error state capture、telemetry。
- long-running stress、race tests、hotplug/modeset stress。

**Exit:** 24時間 GPU workload、複数回 suspend/resume、hotplug/modeset stress で kernel hang / memory leak / display corruption がない。

## 実装上のガードレール

- 実機検証がない register sequence / PLL / AUX training は有効化しない。
- まず read-only discovery を完成させ、write path は機能ごとに feature flag で段階解放する。
- GPU command submission は kernel-generated commands から始め、ユーザー command buffer は validator と isolation が完成するまで禁止。
- reset/timeout 時は fence を error-complete し、DMA 完了を確定できないメモリは解放しない。
- firmware mode を最後の fallback として保持する。
- `Gen9` 全般を一括で扱わず、PCI ID / revision / platform family ごとの capability table と workaround table を持つ。

## 検証の前提

ソースダンプだけでは module 全体のビルドやリンクは再現できない。完成判定には、完全なリポジトリ（トップレベル Makefile、include、libc、bootloader、kernel）、対象実機の PCI vendor/device/revision、firmware/VBT、起動ログ、HDMI/DP の接続条件が必要。さらに Intel hardware PRM / platform datasheet を参照し、各レジスタ sequence を SKU ごとに検証する。

## 参考資料

- Linux kernel: `Documentation/gpu/i915.rst` / https://docs.kernel.org/gpu/i915.html
- Intel Display Driver documentation: https://drm.pages.freedesktop.org/intel-docs/ (KMS, DDI, PLL, AUX, VBT, HPD, PSR, FBC)
- Intel Graphics for Linux PRM portal: https://www.intel.com/content/www/us/en/developer/articles/technical/intelr-graphics-for-linuxr.html

