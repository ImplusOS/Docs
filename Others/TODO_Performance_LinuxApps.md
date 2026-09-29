# ImplusOS — Linux アプリが遅い件の原因と対策

> **ステータス: 2026-09-06。原因 7 件を特定・修正、QEMU/KVM で計測済み。**
> デスクトップ到達までのブートが **207.3 秒 → 11.0 秒**（同一ホスト・KVM 有効・
> 同一 QEMU 構成、ブロックキャッシュの有無だけを変えた比較）。
> §4 の起床機構は初版に 3 つのバグがあり（うち 1 つは §4 とは別の
> `process_sleep_current_ms()` 側）、Xorg/xterm の実ログとブート停止で発覚。§9 参照。

調査基準日: 2026-09-05（§9 は 2026-09-06 追記）

関連: [`TODO_Doom_Xorg_MethodA.md`](TODO_Doom_Xorg_MethodA.md)（X サーバ経路）、
[`TODO_Terminal_xterm.md`](TODO_Terminal_xterm.md)（擬似端末）、
[`TODO_Chromium_LinuxABI.md`](TODO_Chromium_LinuxABI.md)（Linux syscall 互換層）。

---

## 0. 症状と、そこから絞れること

> 「KVM を有効化していても、実機で確認しても、Linux アプリの起動・動作が遅い」

この一文が原因をかなり絞る。**KVM の有無で変わらず、実機でも変わらない**なら、
遅さは「命令を実行している時間」ではない。CPU を速くしても縮まないコストは
この OS では 3 種類しかない:

1. **ボーレート** — COM1 は 115200 bps。1 バイト ≒ 87 µs の実時間で、
   `x86_uart_write_char()` はその間ビジーウェイトする。速い CPU ほど無駄に回る。
2. **メディアのコマンド往復** — 光学メディア／USB BOT の 1 コマンドの
   ターンアラウンド。転送量ではなく回数で効く。
3. **固定のスリープ量子** — 「イベントを待つ」実装が実際には「n ms 寝て、
   起きて、もう一度見る」になっている箇所。

実際、原因は 3 つとも該当した。以下、効いた順。

---

## 1. ブロックキャッシュが無かった（最大の原因）

`disk_read()` とメディアの間に何のキャッシュも無く、**ISO9660/UDF のパス解決が
毎回ディレクトリを 2 KiB セクタ単位でメディアから読み直していた**。

glibc の `ld.so` は 1 本の共有ライブラリを解決するのに `/usr` → `lib` →
`x86_64-linux-gnu` と歩き、各ディレクトリを先頭から名前が一致するまで走査する。
Linux プログラムが最初の 1 命令を実行するまでにこれを数百回やる。つまり
**同じ数百 KiB のディレクトリデータを何万回もメディアから取り直していた**。
1 回が実コマンドである光学/USB では、これがそのまま実時間になる。

### 対策

`Kernel/Source/Platform/io/Block_Cache.[ch]`（新規）を `disk_read()` の下に置く。

| 設計 | 値・理由 |
|---|---|
| ライン | 4 KiB（1 ページ、512 B セクタ 8 個） |
| 連想度 | 4 way・セット内ラウンドロビン |
| 容量 | 物理メモリの 1/64、2〜32 MiB にクランプ。4 GiB 機で 16 MiB |
| ページ確保 | ライン初回使用時に 1 ページずつ。アイドル時のコストは 0 |
| 書き込み | キャッシュせず、該当ラインを無効化して素通し |
| バイパス | 128 KiB 以上の要求（ELF 本体・ファームウェア等の流し読み）はキャッシュを汚さない |
| 先読み | ミス時に最大 8 ライン (32 KiB) を **1 コマンド**で充填。直前の充填の続きなら要求長を超えても伸ばす（シーケンシャル検出） |

先読み幅 32 KiB は AHCI の DMA 窓（`AHCI_DMA_SECTORS`）と ATAPI 16 ブロックに
一致させてあり、ここに挙げたどの経路でも 1 コマンドで収まる。

デバイスキーは `block_device_t*` とユニット番号の合成。ユニット番号だけでは
**AHCI unit 0 と USB unit 0 が衝突**し、片方のセクタをもう片方に返す
＝無言のファイルシステム破壊になる。

`OS_CONFIG_BLOCK_CACHE=0` で丸ごと無効化できる（下の計測に使った）。

### ホスト側ハーネス

```bash
./Kernel/Source/Platform/io/Tests/run.sh
```

実物の `Block_Cache.c` をホスト `cc` でスタブと一緒にビルドして走らせる。
16 ケース。内容が LBA の既知関数である合成ディスクを使うので、**バイト値まで**
検証する（件数一致では通らない）。境界・非整列・デバイス独立・書き込み無効化・
バイパス・メディア末尾・エラー伝播・ページ枯渇時の正当性を含む。

最後の 1 ケースは実ワークロードの再現で、3 ディレクトリに対する 200 回の
パス解決 ≒ 11,600 リクエストが、**メディアコマンド 14 回**に畳まれることを
確認している。

---

## 2. ATAPI が 1 コマンド 1 ブロック（2048 B）だった

`Protocol_AHCI.c` の `atapi_read_one()` は READ(10) を常に `blocks=1` で発行し、
`ahci_read()` はそれを 512 B セクタごとに呼んでいた。デマンドページングの
先読み（`FileMap.c` の 64 KiB）が **SCSI コマンド 32 回**、各回の前にタスク
ファイルと CI レジスタのビジーウェイト 2 本、という形になっていた。転送量は
最初から問題ではなく、往復回数が全てだった。

`atapi_read_blocks(lba, blocks)` に置き換え、DMA 窓いっぱい（16 ブロック =
32 KiB）まで 1 コマンドで読む。`ahci_read()` は要求をブロック span に丸めて
チャンクごとに発行し、必要部分だけコピーする。

※ USB BOT (`MassStorage.c`) は元から複数ブロック転送だったので変更不要。

---

## 3. `OS_CONFIG_FOREIGN_TRACE` が「既定 off」と書いてあって 1 だった

`Kernel/Source/include/kernel/config.h` のコメントは

> Off by default; set to 1 for a bring-up boot.

と書いてありながら `#define OS_CONFIG_FOREIGN_TRACE 1`。全 mmap・全 AF_UNIX
パケット・全シグナル/exec/exit・5 秒ごとのハートビートが COM1 へ 1 バイトずつ
出ていた。コメント自身が「X サーバと GL クライアントは起動までに ~400 セグメントを
mmap し、数千の小パケットを交換するので、起動時間の大半を
`serial_write_string()` の中で過ごす」と説明している通り。

コメント通り **0** にした。診断が要るブートでは
`make kernel EXTRA_KERNEL_CFLAGS=-DOS_CONFIG_FOREIGN_TRACE=1`。

代わりに、常時 on の軽い版 `OS_CONFIG_FOREIGN_LAUNCH_LOG` を追加した（§6）。

---

## 4. poll/select/epoll_wait の 8 ms が「上限」でなく「固定費」だった

このカーネルは syscall の途中でブロックできない（`process_schedule_on_syscall()`
は `syscall` 命令の直後にしか復帰させられない）。そのため poll 系は
「8 ms 寝て 0 を返し、呼び直させる」実装になっていた（`Syscall_Epoll.c` の
冒頭コメントに経緯がある）。

問題は、**その 8 ms を早めに切り上げる手段が無かった**こと。X11 の 1 往復は
この待ちを 2 回跨ぐので、**機械がどれだけ速くても 1 リクエスト ≒ 16 ms**。
Xorg + クライアントの起動は往復数千回なので、これだけで数十秒が「純粋に寝ている
時間」として乗る。KVM で変わらなかった理由がここにある。

### 対策

`Kernel/Source/Core/syscall/Poll_Wait.[ch]`（新規）。

- パークした側は `poll_wait_park(generation, ms)` で寝る
- fd を readable/writable にした側が `poll_wait_notify()` を呼び、全パーク中
  プロセスを即座に起こす
- 起床は **fd 単位ではなく全体**。空振りは「もう一度スキャンする」だけで、
  この API の呼び出し側は元々それを許容している。まだ対応していないイベント源が
  あっても従来の時限ポーリングに劣化するだけで、ハングしない

ロストウェイクアップは 2 箇所あり、別々に塞いだ:

1. **fd を見てから park するまで**に来たイベント → 世代カウンタ。
   呼び出し側はスキャン前に世代を読み、`poll_wait_park()` に渡す。動いていたら
   寝ない。
2. **park 登録から実際に BLOCKED になるまで** → プロセスはまだ runnable なので
   `process_wake_pid()` が起こす相手がいない。ここは既存の `wake_pending`
   クレジット機構がちょうど該当するが、`process_block_current()` しか消費して
   いなかった。`process_sleep_current_ms()` にも同じ消費を入れ、消費した場合は
   **1**（＝寝ていない）を返すようにした。0（＝BLOCKED）で CPU を手放す
   呼び出し側は、そのまま走り続けて再スキャンする。

`OS_CONFIG_POLL_WAIT_WAKEUPS=0` でこの機構ごと無効化できる（park は素の
`process_sleep_current_ms()` に、notify は no-op になる）。§9 のとおり、この
帳簿付けは間違えると CPU スピンになる種類のものなので、**1 ビルドで切り分け
られること**自体に価値がある。ブロックキャッシュ・ATAPI・トレース・UART の
修正はこのスイッチと独立なので、切っても失われない。

`poll_wait_notify()` を呼ぶ箇所: AF_UNIX の send/recv/connect（X11 の伝送路）、
pty の read/write、パイプの read/write、eventfd write、TCP/UDP の受信、evdev。

pty の read/write 側の 1〜10 ms バックオフも `poll_wait_park()` に置き換えた。
バックオフは上限としてだけ残り、キー入力の遅延がアイドル時間とともに伸びない。

---

## 5. UART に存在確認が無く、1 バイトあたり 100 万回スピンしうる

`x86_uart_write_char()` は LSR の THRE を待つループの上限が `0x100000`
（1,048,576 回）で、**1 バイトごと**。実機で 0x3F8 をデコードするが UART が
いない構成（今どきの機械ではむしろ普通）では LSR が 0 を返し続け、
1 文字ごとに数百 ms〜1 秒のポート I/O を焼く。「実機だと遅い」の一因になりうる。

- スクラッチレジスタ（オフセット 7）の読み戻しで存在確認。Linux の 8250
  ドライバと同じ判定。無ければ以降の書き込みは no-op。
  `OS_CONFIG_SERIAL_ASSUME_PRESENT` で probe を飛ばせる
- タイムアウトを 20000 回へ。115200 で 1 文字 87 µs なので桁で足りる
- FIFO が有効なら THRE 1 回のポーリングにつき 16 バイト投入する。
  1 バイトごとに FIFO を空にするまで待つと、文字と文字の間で回線がアイドルする

---

## 6. Linux アプリ起動時のログを端末に出す

`OS_CONFIG_FOREIGN_TRACE` を落とした代わりに、**常時 on の 1 行ログ**を入れた
（`OS_CONFIG_FOREIGN_LAUNCH_LOG`、既定 1）。プログラムあたり数行なので
リリースブートで点けたままにできる。

```
[app] spawn pid=7 abi=linux path=/usr/bin/xterm args='-fn fixed ...' at=41230ms
[app] exec  pid=7 abi=linux path=/bin/sh at=41880ms
[app] exit  pid=7 status=0 lived=9310ms
```

### 端末（xterm）で読めるようにする — `/dev/kmsg` と `/dev/applog`

シリアルケーブルの無い実機では COM1 に出しても見えない。カーネルログの
in-RAM リング（`Debug/serial/Serial.c`）を `/dev/kmsg` として userland へ
出した。

- open ごとに独立した読み出し位置。遅い読み手はリングの後端から押し出された
  分を失うだけで、化けた内容は返らない
- open 時は **末尾 4 KiB から**（tail 相当）。端末はアプリを起動する「前」に
  開かれるので、64 KiB のブートログを全部流すと肝心の行が画面外へ流れる
- read は tail -f 形状: 新しい分を返し、無ければ EOF ではなく待つ。
  待ちは 20 ms の素のスリープで、**意図的に `Poll_Wait.h` に繋いでいない**。
  `serial_write_string()` は割り込みハンドラの中や
  プロセステーブルロックの下からも走るので、そこからスケジュール可能な仕事を
  起こすとロック順が逆転する
- `dev_poll` は未読があるときだけ POLLIN

**`/dev/applog`（launch ログ専用の小さいリング、4 KiB）を別に用意した。**
端末が流すのはこちら。カーネルログ全体を窓に流すと **自分自身にフィードバック
する**: 1 行描画する = X トラフィック、X トラフィックはログに出る、そのログを
描画する…… の閉ループになる（初版は実際にこれを踏み、`[usock]` トレースの
512 行キャップに当たって初めて止まった）。`[app]` 行は exec と exit でしか
出ないので、端末が何をしてもループは閉じない。全体が見たいときは手で
`cat /dev/kmsg` すればよい。

あわせて、**リングはバックエンドの有無に関わらず常に埋める**ようにした。
シリアルドライバモジュールが `serial_register_backend()` で伝送路を奪うと、
それまでのコードはカーネル側リングへの追記ごと止めていた。`/dev/kmsg`・
`serial_copy_log()`・パニック画面は全てこのリングを読むので、ドライバが
ロードされただけで真っ白になってはいけない。

また `serial_enable_file_logging()` が `g_log_buf_active = false` でリングを
止めていたのをやめた（64 KiB 固定なので維持コストは無い）。

### 端末側

トップ Makefile の `STAGE_POSIX_SHELL` が `/bin/imsession` を生成する:

```sh
#!/bin/sh
# ImplusOS terminal session: app launch log + shell.
/bin/busybox cat /dev/applog &
exec /bin/sh
```

`Terminal` アプリの xterm 引数を `-e /bin/sh /bin/imsession` にした。Linux の
コンソールと同じで、使えるプロンプトの上にカーネルメッセージが流れてくる。

引数ではなくスクリプトファイルにしてあるのは、`process_spawn_with_arg()` が
起動引数を空白で分割し、クオートを解釈しないため（`sh -c '...'` が渡せない）。

### ホスト側ハーネス

```bash
./Kernel/Source/Debug/serial/Tests/run.sh
```

`serial_read_log()` のカーソル算術を 16 ケースで検証。リング境界の巻き戻り、
リングを 3 周以上またぐ追従、読み手が後端から脱落したときの先頭スナップ、
複数読み手の独立性、短いバッファでの部分読み、退化引数。

---

## 7. 計測

同一ホスト・同一 ISO・QEMU q35 + KVM + 4 vCPU + 8 GiB、LiveCD を
IDE-CD (AHCI) からブート。QEMU 起動からシリアルに `[loginui] wm_pid=` が
出るまでの実時間。

| 構成 | デスクトップ到達 |
|---|---|
| 本修正一式（`Build/` を消しての全ビルド） | **11.0 秒** |
| 同上・別ラン | 13.4 秒 / 15.5 秒 |
| ブロックキャッシュのみ無効（`OS_CONFIG_BLOCK_CACHE=0`） | **207.3 秒** |

ばらつきはホスト側の負荷（並行ビルド）由来。

無効側にも §2 の ATAPI 複数ブロック化と §3 のトレース停止は入っているので、
修正前の実際の状態はこれより悪い。207.3 秒側が下限の見積もり。

**注意**: この計測はブート（＝ネイティブアプリのみ）であり、Linux アプリの
起動そのものではない。ただしブートが叩く経路（ISO のパス解決とデマンド
ページング）は `ld.so` が叩く経路と同じで、支配的なのはそこ。§4 の poll 待ちと
§5 の UART は、この計測にはほとんど現れない（ブートは X11 往復をしないため）。
Linux アプリ側の実測は QEMU 上での対話操作が要るので未実施。

---

## 8. 残件

- **Linux アプリ起動の実測**。WM のランチャからの起動が要るので headless では
  取れていない。`[app]` 行の `at=` / `lived=` がそのまま使える
- **`/dev/urandom` などの単純ノードの read が -EFAULT を返す**。DevFS は
  ドライバテーブルに `dev_read` を持つため `Syscall_File.c` が全ノードを
  `dev_read` に流すが、`devfs_vfs_dev_read()` の `default:` は `-14`。
  HEAD 時点から存在する既存の問題で、本件では触っていない
- **シリアルドライバモジュールのバックエンドが再帰する**。
  `module_backend_write_char()` はカーネルの `serial_write_char` を
  未定義シンボルとして取り込んでおり（`nm -D` で `U serial_write_char`）、
  カーネル側はそれを `g_backend->write_char` へ振る。ロードされれば無限再帰。
  現状ロードされていないので顕在化していない
- **evdev の `poll_wait_notify()` は割り込み文脈になりうる**。
  `evdev_push_*` は現時点で呼び出し元が無い（`TODO_Doom_Xorg_MethodA.md` の
  残件）。入力を繋ぐときは、既に `syscall_futex_on_timer_tick()` が
  タイマ割り込みから `process_wake_pid()` を呼んでいるのと同じ扱いでよい
  （プロセステーブルロックは常に IRQ マスク下で取られる）
- **poll 待ちのフォールバック 8 ms** → **上げた（2026-09-28、§10）**。
  `EPOLL_POLL_SLICE_MS` / `LX_WAIT_SLICE_MS` / `LINUX_POLL_SLICE_MS` を
  すべて 16 ms に。Linux ABI の syscall/s は 8,946 → 1,598（5.6 倍減）、
  うち待機系は 8,266 → 1,069/s（7.7 倍減）。`notify` を呼ばないイベント源の
  遅延上限も 1 → 16 ms に伸びるが、`[sysprof]` に timerfd / signalfd が一度も
  出ない（Chromium・Xorg・WM とも未使用）ので、この 2 つの notify 追加は
  保留のまま。シグナルは `process_signal_deliver{,_group}` の unlock 後に
  notify を入れた — §4 の一覧には無かったが、入れないとスライス延長がそのまま
  EINTR / SIGCHLD の遅延上限になる。`wait4` は子の終了時に独立に入っている
  `poll_wait_notify()` で起きるので、この追加が無いと困る経路では無い

---

## 9. 実ログでの検証と、§4 に残っていた 2 つのバグ（2026-09-06）

Terminal 起動時の実ログ（Xorg + xterm、上記修正入りのビルド）で、**58.8 秒
以降まったく進まなくなる**のが観測された。以降に出るのは Xorg の dbus 再接続
だけで、それがちょうど 10.02 秒周期＝Xorg の `dbus-core` の再接続タイマそのもの。
つまり Xorg のメインループは生きているのに、**クライアントからの要求が一切
届いていない**状態だった。

ログから読める内訳（`[    38.7]` 等は Xorg 自身のタイムスタンプ）:

| 区間 | 時間 | 内容 |
|---|---|---|
| 40.16 → 40.72 | 0.56 秒 | `dlopen` libglx.so |
| 41.87 → 44.70 | **2.83 秒** | IGLX swrast（libgallium）ロード |
| 44.75 → 46.42 | 1.67 秒 | xkbcomp 1 回目（`lived=1189ms`）|
| 46.49 → 48.43 | 1.94 秒 | xkbcomp 2 回目（`lived=2127ms`）|
| 48.8 → 58.8 | 10 秒 | xterm ↔ Xorg のハンドシェイク |
| 58.8 → 139+ | **停止** | dbus 再接続以外なにも起きない |

### 9.1 バグ 1 — タイマ sleep が futex のクレジットを食っていた（真因）

こちらが本命で、**ブート自体を間欠的に止めた**（`[UDF] mounted` の後で停止、
QEMU が 200〜370% CPU）。1 回目のブートは通って 2 回目で出る、という再現性。

原因は §4 で `process_sleep_current_ms()` に入れた「`wake_pending` クレジットを
消費して寝ない」処理そのものだった。このクレジットは元々
`process_block_current()`（＝futex 待ち）専用のもので:

- futex 待ちは「futex ワードを確認 → 変化なし → `process_block_current()` へ
  向かう」の途中で `FUTEX_WAKE` が来ると起こす相手がいないので、
  `process_wake_pid()` がクレジットを置いていく
- glibc の pthread 実装は futex を絶えず叩く

そこにタイマ sleep 側も同じクレジットを消費するようにしたので、**無関係の
sleep が futex トラフィックで片っ端から短絡された**。pipe / pty / poll の
待ちループは毎回「もう起こされている」と言われて一度も寝ず、CPU を焼き続けた。
逆方向の害もあり、futex 待ちが待っていた起床をタイマ sleep が横取りしていた。

切り分けは `OS_CONFIG_POLL_WAIT_WAKEUPS=0`（§4 の機構を丸ごと無効化）を
足して行った。**それでもハングしたので、Poll_Wait の帳簿ではなく
`process_sleep_current_ms()` 側だと確定した。**

対処:

- `process_sleep_current_ms()` は **クレジットに一切触らない**（元の挙動へ戻す）。
  タイマ sleep には自分の締め切りがあるので、そもそもクレジットは要らない
- §4 が塞ぎたかった「登録してから BLOCKED になるまで」の窓は、**Poll_Wait の
  中だけで閉じる**: sleep から戻った直後に自分の parked ビットがまだ立って
  いるか見て、消えていたら（＝その隙間で notify が走った）`process_wake_pid()`
  で自分を起こし直す。他人のクレジットに触らずに済む
- `process_wake_pid()` のクレジットは **カウンタではなくフラグ（1 で飽和）** に
  した。元のコメント自身が「呼び出し側は『起床が 1 回以上あった』しか必要と
  しない」と書いており、取り残しがあっても暴走しなくなる

### 9.2 バグ 2 — パーク登録の取り残し

9.1 に至る過程で見つけたもので、単独でも害がある。

パークは 2 通りで終わる: `poll_wait_notify()` に起こされる（ビットが消える）か、
**単に締め切りが来る（誰もビットを消さない）**か。後者で取り残されたビットは、
以後の notify が毎回「ブロックしていないプロセス」に `process_wake_pid()` を
呼ぶ原因になる。9.1 のクレジット問題と噛み合って悪化していた。

対処: `poll_wait_park()` は **先頭で自分のビットを落とす**。取り残しは
「起きてから次のパークまで」に限定される。

### 9.3 バグ 3 — 「寝ないで済ませる」に上限が無かった

世代カウンタが動いていると `poll_wait_park()` は寝ずに戻る設計だが、世代は
**グローバル**なので、誰かが fd を ready にし続ければ常に成立する。
`linux_syscall_restart()` は RIP を 2 戻してユーザ空間から syscall をやり直させる
ので、**寝もせず譲りもしない** = 100% CPU のスピンになる。端末がカーネルログを
流し始めた瞬間、pty への書き込みが毎行世代を進め、Xorg と xterm の両方がこれに
落ちた（実ログの「58.8 秒以降なにも進まない」）。

対処: 寝なかった場合も **1（＝スケジューラに譲れ）を返す**。さらに連続で寝ないのを
`POLL_WAIT_MAX_DECLINES`（既定 1）回までに制限し、それを超えたら世代が動いて
いても寝る。

### 9.4 ついでに直したもの

| 箇所 | 内容 |
|---|---|
| `IPC/UnixSocket.c` | `[usock]` トレースを `OS_CONFIG_FOREIGN_TRACE` 配下へ。AF_UNIX は X11 の伝送路なので、パケット 1 個 1 行はハンドシェイクの真ん中に数百行＝数秒の 115200 baud を差し込んでいた。`connect FAILED` は 8 件でキャップ（Xorg の dbus 再試行は永久に 10 秒ごとに来る）|
| `Core/syscall/Syscall_File.c` | パイプの read/write 待ちを `poll_wait_park()` へ。Xorg は xkbcomp へキーマップ数十 KB を 4 KiB のパイプで送るので、1 ms 固定待ちだと往復ごとに 1 ms 積む |
| `Core/process/ProcessManager_Create.c` | プロセス終了時に `poll_wait_notify()`。`wait4()` で待っている親（Xorg の `Pclose()`）を 10 ms スライスの満了まで待たせない |
| 同上 | `[app]` 行を **1 行組み立てて 1 回で出力**。CPU ごとの `serial_write_string()` が文字単位で混ざり、実ログでは Xorg の出力と `[app]` 行が互いに割り込んで判読不能になっていた |
| `Core/syscall/Poll_Wait.c` | `g_poll_lock` の遅延 `spinlock_init()` を廃止（静的初期化で十分）。先着 2 CPU が両方「未初期化」を見て両方ロックを初期化＝両方が同時に critical section に入りうる |
| `Arch/x86_64/hal/uart_hal.c` | 存在確認 probe を **`x86_uart_init()` からのみ**呼ぶようにした。probe はスクラッチレジスタに書いて読み返す方式なので、2 CPU が同時に走ると互いの値を読んで両方「UART 無し」と判断し、以後カーネルのシリアル出力が全部消える（＝ハングと見分けがつかない）|

### 9.5 検証

- ブート: **10.4 / 10.8 / 10.8 / 10.8 秒**（連続 4 回、ビルドを挟まず、
  スクリーンショットで desktop 到達を確認）。別セッションでも
  10.6 / 10.7 / 11.1 秒
- ホスト側ハーネス 3 本すべてパス（block cache 16 / serial log 20 / pty 12 ケース）
- フルビルド警告 0 件（新規ファイル分）

#### 間欠的な「ブート停止」について（重要）

修正の途中、`[UDF] mounted` の後で止まり QEMU が 200〜370% CPU、という現象を
何度か踏んだ。これを追って 9.1〜9.3 のバグを見つけたので追跡自体は無駄では
なかったが、**最終的に、観測された停止はすべて「1.1 GB の ISO を書いた直後の
1 本目のブート」で起きていた**。ビルドを挟まずに連続でブートすると
4 回連続で 10.4〜10.8 秒に収まり、一度も再現しない。

ホストは 4 コア / 15 GB に対して QEMU へ 4 vCPU / 8 GB を渡しているので、
ISO 書き込み直後は writeback とページキャッシュの奪い合いが起きる。
`timeboot.sh` の上限は 240 秒で、その状況では超えうる。

つまり **この現象がカーネル側のバグだという確証はない**。テストハーネス由来の
可能性が高い。ただし 9.1〜9.3 は実在のバグなので修正は残してある。

万一実機で挙動が疑わしいときのために、§4 の機構は
`OS_CONFIG_POLL_WAIT_WAKEUPS=0` で丸ごと落とせる（park は素の
`process_sleep_current_ms()` に戻り、notify は no-op）。ブロックキャッシュ・
ATAPI・トレース・UART の修正はこのスイッチと独立。

#### まだ確認できていないこと

- **Terminal を実際に開いてテキストが描画されるところ**は未確認。QMP の合成
  マウスでスタートメニューを開こうとしたが、ゲストの PS/2 は syscall 経路
  からのポーリングでイベントを取りこぼし、カーソルが目標座標まで届かなかった
  （画面隅へ overshoot して座標を確定する方式まで作ったが、WM がカーソルを
  中央へ戻す動きと噛み合わず断念）。`[app]` 行の `at=` / `lived=` が
  そのまま計測に使えるので、実機での確認をお願いしたい
- 上記のとおり、§0 のログにあった **58.8 秒以降の停止が直ったかどうかも未確認**。
  原因（9.3 のスピン）は特定できているが、直ったことの確認は Terminal を
  開くまで待つ必要がある

---

## 10. 計測ベースの軽量化（2026-09-28）

「起動時間」「CPU 負荷」「メモリ使用量」の 3 点を、**まず QEMU で実測して
から**支配的要素を直した。以下はすべて同一条件の連続実行どうしの比較で、
数値は `run3`（変更前ベースライン）と `run6`（変更後）の実測値。

### 10.1 計測ハーネス

- `/tmp/opencode/bench.py <dir> <秒> <間隔>` — LiveCD を KVM・ヘッドレスで
  ブートし、①シリアル全バイト（ホスト壁時刻を小数 3 桁で前置）、
  ②QEMU vCPU のホスト側 CPU 消費（`cpu-s/秒` の差分）、③定期スクリーン
  ショットを `<dir>` へ
- `make image_livecd AUTOSTART="/Userland/PerfProbe/PerfProbe.ELF
  /Userland/Chromium/Chromium.ELF"
  EXTRA_KERNEL_CFLAGS=-DLINUX_SYSCALL_PROFILE=1` で、計測プローブと
  Chromium を同時に自動起動し、Linux ABI のヒストグラム `[sysprof]`
  （100,000 件ごと）も取る
- **音声は `-audiodev none,id=audio0` + `hda-duplex` が必須**。QEMU の既定は
  PulseAudio へ接続を試みて 30 秒で抜け、**その間 vCPU が 1 も走らない**
  （ホスト CPU 0.00 のままシリアルも無出力）。ベースラインの
  「desktop up = 43.4 秒」はこの擬似遅れで、ハーネス修正後は 14.4 秒になる。
  **起動時刻は必ずゲストの `at=` で比較すること**
- 解析: `compare.py`（横並び表）/ `sysprof.py` / `steady.py` / `report.py`
- `PerfProbe` の `mem=` を物理（PMM bitmap）、`heap=` をカーネルヒープに分離。
  それまでの `mem=` は `get_used_memory()` をそのまま出していて、realloc の
  縮小時に `used_memory` を引いていなかったため **15,076 MB** と出ていた
  （`/proc/meminfo`・`sysinfo` も同様に物理量へ修正）

### 10.2 結果

| 指標 | run3（前） | run6（後） |
|---|---|---|
| Linux ABI syscall/s（定常 t=100–160 s） | 8,946 | **1,598**（5.6 倍減） |
| うち待機系 epoll_wait/ppoll/poll | 8,266/s（92.4%） | **1,069/s**（7.7 倍減） |
| ゲスト busy（同窓 mean / median） | 5.9 / 5.8 % | **5.6 / 5.4 %** |
| ホスト CPU（同窓 mean / median） | 0.31 / 0.31 cpu-s/s | **0.30 / 0.30** |
| chrome 起動（`[app] spawn`, guest ms） | 26,178 | **17,360** |
| 最後の子プロセス exec（guest ms） | 55,607 | **35,760**（36%短縮） |
| `[elfload]` 4 本の合計 | 56.0 s（run4） | **14.0 s**（75%短縮） |
| 1 本あたり最大 | 20.6 s | **3.6 s** |
| カーネルヒープ | 15,076 MB（誤計上） | **79 MB** |
| 物理メモリ使用 | 計測不能 | **4,753 / 9,216 MB** |

busy の 20 秒窓別プロファイルは run3 とおなじ平坦さ（4.8–6.9%）に戻った。
run4 / run5 で見えていた t=160–200 s の山は、**ELF ロードが遅くて 4 本目の
子プロセス exec が guest 70 秒までずれ込み、Chromium の起動処理そのものが
その窓に入っていた**ことによるもので、スライス引き上げの副作用では無い
（run6 でロードが 35.8 秒で終わると山ごと消える）。

### 10.3 支配的コスト 1 — ATAPI の DMA 窓（解消済み）

`[elfload]` は read / copy / map の 3 分解を出す。カウンタは**ロードごとの
構造体**にした — ファイルスコープだと並行ロードで噛み合い、
`read_ms=36306` が `total_ms=20590` の中に収まるという行が出た
（`mb=` が 62 / 630 / 652 / 333 と散らばるのも同じ理由）。

読み込みが支配的だった。`ahci_read()` の 1 コマンドは `AHCI_DMA_SECTORS`
（64 セクタ = 32 KiB）で、ローダは 256 KiB チャンクで読むので
**1 チャンク = 8 個の ATAPI READ(10)** になっていた。333 MB の chrome を
1 回ロードするだけで **10,656 コマンド**、1 コマンドあたり約 682 µs、
実効 46 MB/s。§2 と同じ「往復回数が全て」の構図が、もう一段粗い粒度で
残っていた。

`AHCI_DMA_SECTORS` を 1024（512 KiB）へ。ローダの 256 KiB チャンクが
1 コマンドで完結する（`chunks=1336` がそのままコマンド数になる）。

| 333 MB の 1 ロード | 前 | 後 |
|---|---|---|
| read | 7.3 s（46 MB/s） | **1.5 s（220 MB/s）** |
| copy | 0.19 s | 0.11 s |
| map | 2.2 s | 1.9 s |
| 合計（並行 2 ロード込み） | 9.7–20.6 s | **3.3–3.6 s** |

並行していた子プロセスの exec が互いのディスクロック待ちで 20 秒まで
伸びていた分も、読みが速いだけで消えた。

### 10.4 支配的コスト 2 — ページ割当（残）

read が速くなった結果、`map` がロードの **半分**（333 MB = 85,248 ページで
1.6–2.2 s ＝ **19–26 µs/ページ**）を占めるようになった。うち約半分は
§10.7 のスキャンが占めていたことが分かった。残りの `paging_map_user_range_alloc()`
は 1 ページごとに `alloc_page()`（IRQ マスク + `page_lock`）＋ 4 KiB の
`memset()` ＋ 5 階層のテーブル探索をやっている。削る順:

1. `p_filesz` の分は zero-fill をやめて、読み込み先を直接マップ済み物理ページに
   （現状は staging へ読んでから zero + copy している）
2. チャンク境界ごとに分かれる `alloc_page()` のロックとテーブル探索を
   セグメント単位にまとめる
3. さらに先は PT_LOAD の demand paging（`filemap_handle_fault()`）で、
   exec を O(bytes) から O(触ったページ数) へ。同じ chrome を 1 回の spawn と
   3 回の子 exec で 4 回読み直しているのも、ここでファイル側に cache が
   できると消える

### 10.5 測定上の限界

- `[sysprof]` は 100,000 件ごとの窓なので、syscall が減ると窓が伸びる。
  レートは後半の dump 間隔から出す（前半は起動で詰まっている）
- guest busy はホストの割り込みで振れる。同じバイナリの run4 / run6 の
  同窓 mean が 6.8 / 5.6、run5 が 6.8 だったので ±15% 程度は再現性がある。
  これ未満の差を 1 回の実行で主張しない
- `[elfload]` の read/copy/map はロードが並行している間は互いを含む。
  `total_ms` と `mb` / `chunks` だけがロード単位で正確

### 10.6 物理メモリの「総量」に PCI ホールが混入していた（解消済み）

`get_total_memory_pages()` は `g_max_pages` を返していたが、これは**アドレスの
上限**であって RAM の量ではない — 全 E820 ディスクリプタの最大終端アドレスを
採るので、pc 機械の below-4G / above-4G の間に広がる **1,024 MiB の PCI ホール**
がまくりこまれる。結果、`-m 8192` のマシンが「9,216 MB」になり、
`total - free` で出す「使用量」には hole がまるごと含まれていた。

- `g_usable_pages`（`is_usable_memory_type()` で PMM に解放した領域の合計）を
  追加し、`get_total_memory_pages()` はそれを返す
- 「ここまで物理アドレスが伸びるか」だけが答えの場合（Paging_Main の
  identity map）は `get_physical_address_pages()`（旧値＝`g_max_pages`）を使う
- ついでに `sysinfo` / `/proc/meminfo` の used・free をヒープ基準から物理
  （PMM bitmap）基準へ。ヒープの `get_free_memory()` は「malloc 領域の空き」
  であり `total_bytes` と比較できない量だった
- 併せて `realloc()` の縮小時に `used_memory` がデビットされないバグを修正
  （tail を free list に返しているのに計上はそのまま）。カーネルヒープ表示が
  8 GB のマシンで 15,076 MB になっていたのはこの累積

検証（run7、数値部分は汚染の影響を受けない）: **total 8,120 MB /
used 3,658 MB / heap 79 MB**。9,216 − 8,120 = 1,096 MB がちょうど hole 分。

### 10.7 支配的コスト 3 — 未実装スワップのための O(4096) 走査（対処済み）

`swap_track_page()` / `swap_find_track()` は `g_swap_tracks[4096]`（≈96 KB）を
**線形走査**する。しかも呼び出し箇所が hot なところに全部ある:

- マップ: `paging_map_user_page()` の fast / slow 両パス
- demand fault: `paging_handle_swap_fault()`
- unmap: 1 ページごと
- **アドレス空間破棄: PTE ごと**（Chromium の子プロセス exec / exit のたびに
  そのプロセスが持つ全ページ分）

追跡先は**永続に空**である。`swap_alloc_slot()` は呼び出し元が無く（ビルドの
`defined but not used` 警告と一致）、`PAGE_SWAP` を PTE に立てる箇所も
`swapped = 1` を書く箇所も存在せず、`paging_swap_reclaim_one_page()` は 0 を返す
= **スワップアウトがそもそも未実装**。つまりこの表は何かを答えないのに、
1 ページあたり「ミス走査（4096 全体）＋挿入走査（4096 全体）」の 2 回を
払い続けていた。1 往復 ≒ 10–20 µs で、`map` が 19–26 µs/ページだった内訳と
ほぼ一致し、ロードの map 相（1.6–2.2 s / 3.6 s）の**約半分**を占める。

`g_swap_track_active = 0`（既定）で `swap_find_track()` / `swap_track_page()` を
即 return にした。挙動は不変（追跡先は常に空なので、読む側も書き込む側も
何も得ていない）、スワップアウトを実装したときに 1 へ flip するだけ。
効果は map 相だけでなく、Chromium が取り続ける demand fault と、子プロセスの
teardown にも及ぶはず（run9 で計測、数字は後追記）。

### 10.8 計測汚染の判別と run7 の棄却

run7（21:24:25–21:28:25）は**棄却**。同じホストで別ビルドが走っており、
窓内に書き込まれたビルド成果物が **2,282 ファイル**あった
（run3 / run4 / run5 / run6 はいずれも **0 件**）。

vCPU がホストから予定を奪われると、次の 2 つが同時に起きる:

1. **tick 喪失** — `get_uptime_ms()` は tick 基準（`XSession.c` のコメント
   「interrupts are held off のときに遅れる」と同じ現象）。run7 の
   `[probe] t=` は host timestamp に対し **25% 時間遅れ**（run6 は 1.2%）、
   しかも t=40 s 以降ずっと一定レートで失われ続けた
2. **busy 膨張** — `busy = 1 − idle/wall` は wall（TSC 基準）が進むのに
   idle が積まれないので、runnable で走れない時間が busy になる
   （5.6% → 25.1%）。hostcpu 0.30 → 0.99、`[elfload] read_ms` 1.5 → 5.9 s も
   同じ理由（TSC 基準なので実時間を正しく測っていた＝本当に飢餓状態）

対照として **syscall 数・page fault 数・画面は run6 とほぼ同一**だった
（chrome の `sys=` 260,106 vs 263,850、`flt=` 29,706 vs 29,775、screendump の
差分 194 px / 1M px）。つまり「アプリが 4 倍働いた」のではなく
**「同じ仕事を 4 倍の時間でやった」**。メモリ表示（MemTotal / used / heap）は
時間に依存しないので run7 の値は使える。

**毎 run 確認する 3 点**（以後の run に適用）:

```sh
# 1. 窓内にビルドが無かったこと
find Build Image -newermt "@$start" ! -newermt "@$end" | wc -l   # 0 であること
# 2. tick が失われていないこと（[probe] t= の host に対する傾き ≈ 1.0）
# 3. 別の qemu-system が同時に無かったこと（4 コアホストなので飽和する）
```

ハーネス側の対応: `bench.py` は ISO を `/tmp/opencode/bench.iso` にコピーして
使う（`make run_uefi_usb` など他セッションが ISO を書ロックしていると、
QEMU が `Failed to get shared "write" lock` で起動できない）。
