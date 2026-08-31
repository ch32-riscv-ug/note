# CH32向け新規書き込み・デバッグツールの設計案

- 起案日: 2026-08-31
- 状態: 提案。実装未着手。名前は未確定
- 前提資料: [host application比較](wch-linke-host-apps.ja.md)、[全経路調査](programming-tools-and-probes.ja.md)、[probe・USB経路調査](programming-probes-and-usb-paths.ja.md)
- 想定言語: Rust

## 1. なぜ新規に作るか

### 1.1 既存toolが不足している点

`../ArduinoCore-CH32`の実測と上記調査で確認済みの不足を、推測と分けて並べる。

| tool | 不足 | 根拠 |
|---|---|---|
| probe-rs | CH32特化ではないため、LinkE固有機能(power-off/NRST erase、SDI print、power制御、mode切替、firmware版取得)が構造的に入らない | [host app調査 5.2](wch-linke-host-apps.ja.md)、[probe-rs #3023](https://github.com/probe-rs/probe-rs/issues/3023) |
| probe-rs | target定義に穴がある(V205/V407/V467/X315/M030)。hand-maintainedなYAMLのため、新SKUのたびにupstream待ちになる | ADR-0008、boards.txtの`[compile only]`群 |
| probe-rs | debug/resetの不具合。LinkE firmware 2.11では`download --reset`がrc=0で成功表示のままtargetが走らない。`disable_debug_module`が`DtmOperationFailed`を出すが失敗にならない | 2026-08-22実測(upload-and-fixture) |
| minichlink | C実装。GDB stubにstack buffer overflowがあり修正PRを出したが取り込みが遅い | `pr/ch32fun` `feb06f4` |
| minichlink | versioned releaseが無く、配布物は自前buildが前提。Board Manager配布済みの古いbuildは`-l`(serial選択)を持たない | 2026-08-19実測 |
| wlink | probe選択が`--device INDEX`のみでserialを選択子にできない。verify commandが無い。GDB/step/breakpointが無い。README自身がproduction-readyでないとする | [host app調査 5.2](wch-linke-host-apps.ja.md) |
| WCH OpenOCD | 対応sourceが非公開でGPLのbinary再配布ができない。attach時にflash先頭を書き換えて戻さない | `../ArduinoCore-CH32/docs/debugger.ja.md` |
| すべてのOSS tool | LinkE firmwareの更新経路が無い。IAP mode(`4348:55e0`)の入口は既知なのに、書き込みを実装したものが存在しない | 2026-08-22実測 |

つまり不足は「機能が少し足りない」ではなく、**責任範囲の切り方がCH32に合っていない**ことに起因する。probe-rsは汎用probe抽象、minichlinkは実験環境、wlinkはprotocol実証、WCH公式は非公開binary、という別の目的で作られている。

### 1.2 「一つのtoolでは足りない」の内訳

現状、一つの作業を通すのに複数のtoolを混ぜている。

| やりたいこと | 今の手段 |
|---|---|
| 書き込み | probe-rs |
| 書き込み後に走らせる(FW 2.11) | wlink reset |
| probeの種別とfirmware版を知る | wlink status |
| SDI printを有効化して受ける | wlink |
| GDB debug | MounRiver同梱のWCH OpenOCD(PATH頼み) |
| V003のGDB debug | minichlink |
| LinkE firmware更新 | WCH-LinkUtility(Windows) |
| USB/UART ISP | wchisp(GPL-2.0) |
| 互換probe | minichlink |

**不足しているのはbinaryの数ではなく機能とcontractである**。したがって新toolは「複数のtoolを作る」のではなく、**1つのCLI + 再利用可能なlibrary crate群**にし、Arduino統合に必要な別実行形式(pluggable discovery/monitor)も同じbinaryのsubcommandで賄う。

## 2. 作らないもの

範囲を先に閉じる。ここを開けると probe-rs の二番煎じになる。

- ARM Cortex-M、CMSIS-DAP、JTAG、他vendorのMCU。**WCH RISC-V(1線SWIO / 2線RVSWD)だけを見る**
- IDE、GUI(第1版では。GUIを作るならCLI/JSON APIの薄いclientとする)
- WCH firmware blobの同梱・再配布(LinkE firmware imageはuser-suppliedとする)
- 独自probe firmware(第1版では。`Q-045`は別題)
- 汎用のflash algorithm frameworkやprobe plugin機構

## 3. 出発点の選択

判断基準を「工数」ではなく「長期の保守コストとlicenseの説明可能性」に置く(ArduinoCore-CH32のREADMEが掲げる`Prefer low long-term maintenance cost over the shortest initial implementation`と同じ基準)。

| 案 | 内容 | 評価 |
|---|---|---|
| **A. 新規実装 + 既存実装を仕様corpusとして読む(推奨)** | codeは持ち込まず、wlink / minichlink / RINS / probe-rsを**仕様書として**読み、実機で検証して自分のprotocol.mdを書く | 第三者noticeも継承した設計負債も持たない。Arduino Board Manager再配布時のlicense inventoryが自prjのdependencyだけで閉じる。層を最初から切れる |
| B. wlink取り込み | wlink(MIT OR Apache-2.0)の実装を取り込んで育てる | 立ち上がりは速いが、wlinkのCLIとprotocolの結合、chip enum table、`--device INDEX`前提の設計を引き継ぐ。取り込み範囲の追跡とnotice管理が永続的に残る |
| C. probe-rsのplugin | upstreamに寄せる | probe固有機能の受け皿が無く、target gapもupstream合意待ちになる。今回の動機と正面から衝突する |

**Aを採る。ただし「cleanに書く」と「先行実装を読まない」は別である。**

希少なのはcodeではなく**経験的なprotocol知識**(firmware版ごとの応答差、attach手順、erase後の待ち、固まる条件)であり、これは何年もかけて他プロジェクトが見つけたものである。したがって:

- wlinkの`protocol.md`と実装、minichlinkの`pgm-wch-linke.c` / `chips.c` / GDB stub、[RINS](https://perigoso.github.io/rins/)、probe-rsのWCH-Link backendは**一次参照として読む**
- 読んだ内容は**自分の実機capture(USB record)で裏を取ってから**実装する。裏が取れないものは「未検証」として記録し、実装しない
- 成果物として**自分のprotocol.mdを持つ**。firmware版ごとの差分表を含める。これが無いと、cleanに書いた意味が実装者の頭の中に閉じる
- `../pr/ch32fun`のminichlink修正は継続する。**C実装のbugは移植せず、bugが生まれた構造(固定長bufferとlength未検証)を再現しないことを設計で保証する**

probe-rsへのtarget定義貢献は、こちらのtarget DBが実機で正しいと確認できた後の副産物として出せる。

## 4. 機能セット

優先度は P0=第1版に必須、P1=第2版、P2=以降。

### 4.0 共通契約(全commandに効く。ここが既存toolとの最大の差)

| 項目 | 仕様 | 優先 |
|---|---|---|
| machine-readable出力 | 全commandに`--json`。進捗はNDJSONで別stream | P0 |
| 安定したexit code | 後述の表。textを解析させない | P0 |
| probe識別子 | `VID:PID:SERIAL`をcanonical IDにする。indexは補助扱い | P0 |
| target識別 | 曖昧ならfail-closed。候補と根拠(chip ID)を出す | P0 |
| capability照会 | **probe型番 × probe firmware × target family × operation**で可否を返す。`tool supports LinkE`のbooleanにしない | P0 |
| 排他制御 | USB serial単位のlock、timeout、cancel、安全なdetach | P0 |
| errorの分類 | 権限/driver、probe firmware、target未特定、protected、配線、transport timeoutを別codeにする | P0 |
| 再現性 | `version --json`でtool版、source revision、target DB revision、flash algorithm hashを出す | P1 |
| log | backend生logをartifactとして保存できる | P1 |

exit code案:

| code | 意味 |
|---|---|
| 0 | 成功 |
| 2 | 引数・使い方 |
| 10 | probeが見つからない |
| 11 | probeを開けない(権限、driver binding) |
| 12 | probe firmwareが要求を満たさない |
| 20 | targetを特定できない |
| 21 | targetがprotectedで、明示のunprotectが必要 |
| 22 | attach失敗(配線、電源、BOOT) |
| 30 | verify不一致 |
| 40 | transport timeout |
| 50 | 書けたがtargetが走っていない |

### 4.1 probe層

| 機能 | 内容 | 優先 |
|---|---|---|
| list / info | 型番、HW、firmware版、mode、serial、interface構成、使用中か。Windowsでは各interfaceにbindされているdriver名も出す | P0 |
| 複数probe | serial指定で確実に1台だけopen。同時接続を前提にする | P0 |
| power | 3.3 V / 5 V出力、power cycle | P0 |
| mode | RISC-V / DAP / IAPの検出と切替 | P1 |
| firmware版の解釈 | 生byte・tool表示・WCH表示(2.12 / v32)を同じ値として扱わない。**既知不良版(2.11のreset問題)を検出して警告する** | P0 |
| firmware更新 | IAP modeへ入り、user-suppliedなimageを書く。**imageは同梱しない**(license)。`wlink-iap`が先行実装 | P1 |
| 対応probe | LinkE、LinkW、初代Link(CH549)を型番として区別し、非対応operationはcapabilityで弾く | P0 |

### 4.2 target層

| 機能 | 内容 | 優先 |
|---|---|---|
| 自動検出 | chip ID → family/SKU。曖昧なら候補を出して停止 | P0 |
| **device DBの生成** | `../ch32-data` / `../ch32-device-data`から生成する。**Arduino coreと同じdataを唯一のsourceにする**。手書きYAMLを持たない | P0 |
| DB内容 | flash/RAM容量、page/sector、debug IF(1線/2線)、option byte layout、system flash領域、flash algorithm | P0 |
| 情報取得 | UID、flash size、protection状態、option bytes | P0 |
| option bytes | 構造化read/write(`--set rdp=off boot=flash`)。生値の読み書きも残す | P1 |
| system flash | 対応family(V003/V00X/CH641/M007等)で書込先を選べる | P1 |
| 未対応SKU | 「未対応」と「DBに無い」を区別して報告する | P0 |

probe-rsのtarget gapを埋めるのが目的ではなく、**gapが構造的に生まれない置き方にする**のが目的である。

### 4.3 書き込み層

| 機能 | 内容 | 優先 |
|---|---|---|
| 入力形式 | ELF / IHEX / BIN(+offset) / UF2 | P0 |
| erase policy | auto / sector / chip / none を明示指定 | P0 |
| verify | readbackまたはCRCで**標準経路にする**。`--verify none`は明示的に選ぶもの | P0 |
| reset policy | run / halt / none | P0 |
| **走ったことの確認** | `--confirm-run`。resetのあとPCがflash領域にあるかを見て、走っていなければexit 50。「書けるのに動かない」をtoolが検出する | P0 |
| 読み出し | 範囲指定でfileへdump、blank check | P1 |
| 大image | chunk・timeout・retry方針を明示。**16.7 KBのsketchでprobeが固まる事例がある**ため、retry回数と再試行した事実を必ず表示する | P0 |
| 進捗 | NDJSON。IDE/CIで進捗bar自作が可能 | P1 |

`--confirm-run`は既存toolに無く、今回いちばん痛い障害(「Finished表示だがtargetは停止」)への直接の答えになる。

### 4.4 復旧・診断層

| 機能 | 内容 | 優先 |
|---|---|---|
| recovery | power-off erase / NRST erase / connect-under-reset / mass erase / unprotect を**別operation**にする | P0 |
| doctor | 権限(udev)、Windows driver binding、LinkE firmware、targetの電源・BOOT・配線を切り分け、次の一手を出す | P0 |
| udev rule出力 | `doctor --emit-udev`でruleを生成 | P1 |
| probeが固まった時 | USB再接続が必要なことを検出して手順を出す。`USBDEVFS_RESET`は使わない | P1 |

### 4.5 実行時I/O層(monitor)

| 機能 | 内容 | 優先 |
|---|---|---|
| 3経路を別名で | 物理UART bridge / SDI print / RTT。同じCOMに見えても別物として扱う | P0 |
| port発見 | VID/PID/serial/interfaceからportを決める。COM番号・`/dev/ttyACM*`の番号に依存しない | P0 |
| 再enumeration追従 | upload後にportを閉じて開き直す。**LinkEのUART bridgeはflash直後に配送が止まることがあり、再openで直る**ことを実測済み | P0 |
| SDI | enable/disableとwatch。firmware要件(2.10以降)をcapabilityで判定 | P1 |
| log | timestamp付き保存 | P2 |

### 4.6 debug層

| 機能 | 内容 | 優先 |
|---|---|---|
| 実行制御 | halt / run / reset / step、hw/sw breakpoint、memory・register R/W | P1 |
| GDB server | `gdbstub` crateで実装。**attach時にflashを書き換えない**(WCH OpenOCDの挙動を再現しない)ため`load`必須にならない | P1 |
| 速度 | WCH OpenOCDの`load`は約1 KB/s。ここは明確に上を狙う | P1 |
| DAP | VS Code直結。probe-rsに委譲する選択肢も残す | P2 |
| semihosting / RTT | `run`でexit codeまで拾えるとHILが楽になる | P2 |

### 4.7 ISP / bootloader層

| 機能 | 内容 | 優先 |
|---|---|---|
| USB/UART factory ISP | protocolを自前実装する(wchispはGPL-2.0のため取り込まない) | P2 |
| 自動entry | X03x/X315/H417のCDC 1200 bps touch + `SystemReset_StartMode()` | P2 |
| custom bootloader | DFU / UF2 / rv003usb HIDへの橋渡し | P2 |

### 4.8 互換probe backend

| 機能 | 内容 | 優先 |
|---|---|---|
| minichlink互換HID probe | ESP32-S2 funprog、rvswdio、NHC-Link042 | P2 |
| capability | backendごとの非対応operationを**実行前に**返す | P2 |

### 4.9 Arduino統合

| 機能 | 内容 | 優先 |
|---|---|---|
| pluggable discovery | probeをportとしてIDEに出す。同じbinaryのsubcommandで実装 | P1 |
| pluggable monitor | UART/SDIをSerial Monitorに繋ぐ | P1 |
| recipe | `platform.txt`から呼びやすいCLI。**Arduino専用にしない**(ch32fun、CI、単体利用から同じCLIを使う) | P0 |

## 5. MVPの決め方

「今ArduinoCore-CH32を止めているもの」の順に並べる。これが第1版の範囲になる。

1. 書けるのに走らない(firmware版依存) → `flash --confirm-run` + probe firmware検出
2. target gap(V205/V407/V467/X315/M030) → ch32-dataからのDB生成
3. 複数probe・HILの安定性 → serial選択、lock、retryの明示、doctor
4. monitor統合(UART / SDI) → port発見と再open
5. GDB(WCH OpenOCDのsource非公開とflash書き換え) → GDB server

1〜4までで「日常の書き込みとCIが既存tool無しで回る」状態になる。5でMounRiver依存が切れる。

## 6. CLI案

```text
<tool> probe list [--json]
<tool> probe info    --probe <id> [--json]
<tool> probe power   <on|off|3v3|5v>
<tool> probe mode    <riscv|dap>
<tool> probe firmware check|update --image <file>

<tool> target info   [--json]
<tool> target option get [--json]
<tool> target option set rdp=off boot=flash

<tool> flash <file> [--chip <SKU>] [--offset <addr>]
       [--erase auto|sector|chip|none] [--verify crc|readback|none]
       [--reset run|halt|none] [--confirm-run]
<tool> verify <file>
<tool> read   --range 0x08000000+16k -o dump.bin
<tool> erase  [--mode sector|chip]
<tool> reset  [--halt]
<tool> recover --method power-off|nrst|unprotect

<tool> monitor --source uart|sdi|rtt [--baud 115200]
<tool> gdb     [--port 3333]
<tool> run     <elf>

<tool> capabilities [--json]
<tool> doctor [--json] [--emit-udev]
<tool> version --json
```

全commandが`--probe VID:PID:SERIAL`を受け、環境変数でも指定できるようにする。

## 7. アーキテクチャ

crate分割案(prefixは名前決定後に置換)。

| crate | 責務 | 単体で使えるか |
|---|---|---|
| `*-usb` | nusbによるdevice列挙・open・lock | ○ |
| `*-linkproto` | WCH-Link bulk protocol(`0x81 cmd len ...`)。**protocol.mdをrepositoryの一級成果物にする** | ○ |
| `*-dmi` | RISC-V debug module / DMI操作。1線・2線の差を吸収 | ○ |
| `*-target` | device DB。ch32-dataからbuild時生成 | ○ |
| `*-flash` | erase/program/verify、flash algorithm | ○ |
| `*-debug` | 実行制御、breakpoint、GDB server | ○ |
| `*-isp` | factory ISP(P2) | ○ |
| `*`(bin) | CLI。上を組むだけ | - |

依存の選択:

| 用途 | crate | 理由 |
|---|---|---|
| USB | **nusb** | pure Rust。libusbの同梱・build依存が消え、配布が単純になる。wlinkが既定で採用済み |
| GDB | gdbstub | protocol実装を書かない |
| ELF | object / goblin | |
| serial | serialport | monitor用 |
| CLI | clap | |
| JSON | serde | |

Rustを選ぶ理由は、memory safety(minichlinkのbuffer overrunへの回答)だけでなく、**nusbで配布が単純になること**と**cargo-distで5 platform分のbinaryとattestationが出せること**が大きい。逆にRustで自動的には解決しない部分(protocol知識、target DB、実機CI)を過小評価しないこと。

### 7.1 「cleanな実装」として具体的に担保するもの

工数を制約にしない前提で、**後から入れられないもの**を最初に入れる。

| 項目 | 内容 | なぜ後から入らないか |
|---|---|---|
| **USB record/replay harness** | 実機のbulk転送を記録し、CIで再生してprotocol regression testにする。firmware 2.11 / 2.12 / LinkE 2.15の記録をfixtureとして持つ | 実機無しCIの可否がtest設計を決める。**既存4toolのどれも持っていない**。「FW 2.11では走らない」のような事象をtestとして固定できる |
| **RISC-V Debug Spec準拠層** | DMI/debug moduleを仕様(0.13.2 / 1.0)に沿って実装し、**WCH固有の差分はquirk層に隔離する** | 層を後で切り直すのは事実上の書き直し。ここが分かれていれば新family追加が安くなる。minichlinkとの構造的な差はここ |
| spec-first | protocol.mdとJSON contract(schema)をcodeより先に書き、contractをCLIとは独立にversioningする | 出力仕様を後から安定させるのは互換性破壊になる |
| `#![forbid(unsafe_code)]` | USB境界crate以外で徹底 | - |
| library層でpanicしない | `unwrap`/`expect`を禁止し、typed errorで返す。unknown responseは黙って無視せずerrorにする | 「成功表示なのに失敗している」を構造的に防ぐ。今回の動機そのもの |
| protocol decoderのfuzz | `cargo-fuzz` | - |
| flash stubをsourceから | 事前buildのblobを持たず、in-repoのsourceからbuildしてhashを`version --json`に出す | 再現性の主張が後付けできない |
| target DBの`verified`区別 | ch32-data由来の生成物に、実機確認済みかのflagを持たせて出力に出す | 「実装済み」と「実機確認済み」を混ぜたのが既存toolの読みにくさの原因 |
| dependency方針 | `cargo-deny`、MSRV固定、unmaintained crateを入れない | - |
| 実機CI | LinkE + 各family。HILが無い変更でもunit/replay testは通る構成にする | - |

## 8. 名前

### 8.1 評価軸

1. **射程の正確さ**: 対応するものを名乗り、対応しないものを名乗らないこと
2. **検索性**: `CH32`が入ると発見されやすい
3. **既存との非衝突**: `wlink` / `wchisp` / `minichlink` / `probe-rs`と混同しない
4. **打鍵**: recipe・CI・毎日のCLIで打つ
5. **suite性**: crate prefixとして自然か(`*-linkproto`等)

### 8.2 `CH32`だけでは広すぎる

**`CH32`はRISC-Vだけの名前ではない。CH32F103 / F203 / F208はArm Cortex-M3である。** 本toolのnon-goals(§2)はArm・CMSIS-DAP・SWDを明示的に外しているので、`ch32tool`は**対応しないものを名乗る**ことになる。

これは語感の問題ではなく、具体的なsupport負荷につながる。

- 同じWCH-LinkEに**ARM/DAP mode(PID `8012`)がある**。`ch32tool`という名前は「`mode dap`にして CH32F103 を書けるはず」という期待を作る
- ArduinoCore-CH32側は既に`platform.txt`の`name=CH32 RISC-V`、FQBN vendor `ch32-riscv-ug`で**RISC-Vを明示している**。tool名だけ広いのは一貫しない

したがって**`RV`(または`RISC-V`)を入れる側に寄せる**。「無い方がシンプル」は正しいが、ここで削るのは装飾ではなく射程の定義である。

### 8.3 候補

空き状況は2026-08-31にcrates.io APIとGitHub repository検索で確認した(crates.io 404 = 未登録)。

| 候補 | 射程 | 検索性 | 打鍵 | crates.io | GitHub同名 | 所見 |
|---|---|---|---|---|---|---|
| **`ch32rv`** | ◎ | ◎ | 6字 | 空き | 0件 | RISC-Vだけを名乗り、CH32Fを名乗らない。coreの`CH32 RISC-V`表記と一貫。発音できる |
| `ch32rvtool` | ◎ | ◎ | 10字 | 空き | 0件 | genre markerが明示的。`minichlink`と同じ長さなので許容範囲。`rvtool`の並びがやや詰まる |
| `wchrv` | ◎(CH5xx含む) | ○ | 5字 | 空き | `kaidegit/wchrv-toolchain`(0★) | WCH RISC-V全体を正確に指す。ただし**会社名を冠する**ため非公式prjとしては`CH32`(device family名)より踏み込む。子音5連で発音しにくい |
| `ch32tool` | **×** | ◎ | 8字 | 空き | 0件 | Armを含むCH32全体を名乗る。§8.2により不採用 |
| `ch32ctl` | × | ◎ | 7字 | 空き | 0件 | 同上。射程の問題は`tool`/`ctl`の差では解けない |
| `ch32v` / `ch32vtool` | × | ○ | 5/9字 | 空き | - | Arduino architecture idと同じだが、**CH32Vはfamily名でもある**ためX03x/L103/M030を含まないと読める。不採用 |
| `rvlink` | △ | △ | 6字 | 空き | 無関係な小repoのみ | link=probe経路に読め、ISP/bootloader経路を含めると名前負け |
| `qingke*` | ◎ | × | - | 空き | - | RISC-V coreだけを正確に指すが、**WCHのcore brand名**であり、device family名より商標的な判断が要る。読みも割れる |

`wchisp`が会社名を使っている前例はあるが、ArduinoCore-CH32のREADMEは`The name CH32 identifies the target device family and does not imply endorsement`という立て方をしている。**device family名(CH32)+ 命令セット(RV)** はこの立て方をそのまま延長できる。

### 8.4 推奨

**`ch32rv`を第一候補とする。**

- 対応するもの(CH32のRISC-V系: V00x/V1/V2/V3/X03x/X3xx/L103/M0xx/CH641)を過不足なく名乗り、CH32F(Arm)を名乗らない
- `wlink` / `wchisp` / `openocd` / `edbg` / `minichlink` / `dfu-util`と同じく、`tool`のような genre marker は必須ではない
- crate prefixとして`ch32rv-linkproto` / `ch32rv-target` / `ch32rv-flash`と自然に伸びる
- crates.io・GitHubともに衝突なし

判断が割れる点は一つだけで、**CH5xx(CH570/572/584/585/59x)を将来射程に入れるか**である。あれらは同じSWIO/RVSWDでminichlinkが既に扱っており、`ch32fun`も対応する。

| CH5xxの扱い | 名前 |
|---|---|
| 射程外(推奨。§2のnon-goals維持) | **`ch32rv`** |
| 射程内にする | `wchrv`(会社名を冠する判断が要る) |

genre markerを明示したい場合の対抗は`ch32rvtool`。ただし`ch32rv flash blink.elf`は既に十分tool名として読める。

再利用価値の高い下位crateは、prefixを揃えずに説明的な名前で独立publishする案も残す(`wch-link`、`ch32-target`はいずれも空き)。名前が決まったらcrates.ioにplaceholderをpublishし、GitHub organization/repository名も同時に押さえる。

## 9. 配布

| 項目 | 方針 |
|---|---|
| license | MIT OR Apache-2.0(probe-rs / wlinkと同じ。Arduino packageへの同梱が容易) |
| binary | cargo-dist等でWin x64/arm64、Linux x64/arm64、macOS x64/arm64。checksumとartifact attestation付き |
| USB権限 | Linuxはudev ruleを同梱し`doctor --emit-udev`でも出す。**WindowsはnusbでもvendorインターフェースのWinUSB bindingが要る**ため、driver bindingの検出と手順表示を機能として持つ |
| Arduino | Board Manager package(ADR-0014の自前build配布に載せる) |
| vendor blob | 同梱しない。LinkE firmware imageはuser-supplied |
| SBOM | releaseに添付 |

## 10. 段階計画と判断点

| 段階 | 内容 | 完了条件 |
|---|---|---|
| M0 | 名前確定、repo作成、wlink資産の取り込み方針決定、protocol.mdの整備 | placeholder publish |
| M1 | probe list/info、target info、flash/verify/reset、`--confirm-run`、JSON contract、doctor | 手元の全familyでprobe-rsを置き換えられる |
| M2 | ch32-data由来のtarget DB、option/system flash、recovery、monitor(UART/SDI) | `[compile only]`のSKUが書けるようになる |
| M3 | GDB server、Arduino pluggable discovery/monitor、firmware IAP | MounRiver依存が切れる |
| M4 | ISP、互換probe backend | - |

工数を制約にしない前提なので、**判断点は「速いか」ではなく「cleanな実装を保てているか」**に置く。M1終了時に次を確認する。

- protocol.mdが実装から独立して読めるか(実装を読まないと分からない挙動が残っていないか)
- replay testだけでprotocol層のregressionを検出できるか
- target DBが生成物のままか(手書きの例外が入り込んでいないか)
- non-goals(§2)を守れているか

これらが崩れているなら、機能追加ではなく層の切り直しを先に行う。**upstreamへの貢献(probe-rsのtarget定義、minichlinkの修正)は撤退先ではなく並行して続ける**。

## 11. リスク

| リスク | 対応 |
|---|---|
| **新規実装ゆえに経験的知識が抜ける** | 他prjが何年もかけて見つけたfirmware版依存の癖を、実装だけ新しくして取りこぼす。§3のとおり先行実装を仕様として読み、実機captureで裏を取り、未検証は未実装として明示する |
| LinkE protocolに公開一次仕様が無い | wlink実装・RINS・minichlinkを一次参照とし、**自分のprotocol.mdを成果物として残す**。firmware版ごとの差分を記録する |
| probe firmware依存 | capabilityをfirmware版込みで返す。既知不良版を検出する |
| target DBの正しさ | ch32-data由来にし、実機で確認したSKUと未確認SKUを出力で区別する |
| 実機CIの維持コスト | HILは段階的に。record/replay + unit + fuzzはHIL無しで回す(§7.1) |
| 一人で保守する範囲の拡大 | non-goalsを守る。ARM対応・GUI・独自probe firmwareは第1版に入れない |
| 既存toolとの重複 | 価値の所在は「contract、生成されたtarget DB、confirm-run、doctor、firmware更新、replay test、protocol.md」だと説明できる状態を保つ。**protocol codeは書き直すが、protocolの再解析はしない**(§3) |

## 12. 参照

- [WCH-LinkE host application調査](wch-linke-host-apps.ja.md)
- [全経路調査](programming-tools-and-probes.ja.md)
- [probe・USB経路調査](programming-probes-and-usb-paths.ja.md)
- `../ArduinoCore-CH32/docs/upload-and-fixture.ja.md`(実測ログ)
- `../ArduinoCore-CH32/docs/adr/0008-upload-strategy.ja.md`
- [wlink protocol notes](https://github.com/ch32-rs/wlink/blob/main/protocol.md)
- [RINS](https://perigoso.github.io/rins/)
- [wlink-iap](https://github.com/cjacker/wlink-iap)
- [nusb](https://github.com/kevinmehall/nusb)
- [gdbstub](https://github.com/daniel5151/gdbstub)
