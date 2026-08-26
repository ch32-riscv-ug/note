# WCH-LinkE経由の書き込みアプリ調査

調査日: 2026-08-26

## 1. 範囲と前提

本稿は、WCH-LinkEをUSBでhost PCへ接続し、CH32/CH5xx系targetへ書き込み・デバッグするときのhost applicationを比較する。独自probe firmwareやcustom bootloaderそのものは[全経路の調査レポート](programming-tools-and-probes.ja.md)を参照する。

「対応OS」は次の三つを区別する。

- **公式prebuilt**: projectまたはvendorが、そのOS向けbinaryを継続的に配布している。
- **source build**: sourceと依存libraryからbuild可能だが、利用者側でtoolchainを用意する。
- **原理上動く**: Python/libusb等が対応していても、その組合せのCI・release・実機確認が明示されていない。

また、LinkEの二つのserial用途は別物として扱う。

- **物理UART bridge**: LinkEのTX/RX pinとUSB virtual COMを結ぶ。最大921600 baud。一般のserial terminalでも使用できる。
- **SDI virtual serial**: targetのdebug wireと専用`printf`実装を使い、LinkEのCOM portへ出力する。WCH-LinkE firmwareとMCU系列に制約がある。

記号は、○=直接対応、△=一部対応・別tool併用・source build、×=非対応または目的外、?=公開資料から確定できない、である。

## 2. 公式アプリで可能な操作

[WCH-Link manual V2.4](https://www.wch.cn/uploads/file/20250124/1737704462135866.pdf)がWCH-LinkUtilityについて明示する操作は次のとおりである。

- Erase All、Program、Verify、Reset and run
- MCU UID、flash size、read-protect状態、Link firmware versionの取得
- read protectionの設定・解除、target flashの読み出し
- 通常接続不能時のpower-off eraseおよびNRST erase
- LinkEの3.3 V/5 V出力制御
- 接続時の自動連続download、複数LinkからのGUI選択
- 二線debug interfaceの無効化、user option byteの設定
- V003/V00X/CH641/M007ではprogram flashとsystem flashの書込先選択
- SDI printfの有効化とLinkE COM portからの受信
- WCH-Link firmwareのonline update、別LinkEを使うoffline recovery update
- RISC-V modeとARM/DAP modeの切替

ここでの「全部の機能」は、単なるflash書き込みではなく、上記のtarget管理、LinkE本体管理、debug sessionまで含むものとする。WCH-LinkUtility単体にはsource-level debuggerはなく、公式構成ではMounRiver StudioとWCH OpenOCD/GDBがその層を担当する。

## 3. OSと配布形態

### 3.1 実行可能性

| tool | Windows | Linux | macOS | 配布上の実態 |
|---|---|---|---|---|
| WCH-LinkUtility | ○ 公式GUI | × standalone版を確認できず | × standalone版を確認できず | 公式downloadはWindows application。MRS同梱物からも利用される |
| MounRiver Studio + WCH OpenOCD | ○ | ○ x64 | ○ | IDEとvendor OpenOCDの組。OSごとに別package |
| [probe-rs 0.32.0](https://github.com/probe-rs/probe-rs/releases/tag/v0.32.0) | ○ x64 | ○ x64/ARM64 | ○ Intel/Apple Silicon | 署名・checksum付きrelease assetとinstallerあり |
| [wlink 0.1.2](https://github.com/ch32-rs/wlink/releases/tag/v0.1.2) | ○ x64/x86 | ○ x64 | ○ Apple Silicon | release/CIのprebuilt。Linux ARM64とIntel Macはsource build候補 |
| [minichlink](https://github.com/cnlohr/ch32fun/tree/master/minichlink) | ○ repo同梱exe、source build可 | ○ source build | ○ source build | versioned release assetはない。libusb/HIDとOS設定が必要 |
| [rvprog.py](https://github.com/wagiminator/MCU-Flash-Tools/blob/main/rvprog.py) | △ Python/PyUSB | ○ Python/PyUSB | △ Python/PyUSB | 単一script。WindowsのUSB driverとmacOS実機確認は利用環境依存 |
| [wlink-iap](https://github.com/cjacker/wlink-iap) | △ portが必要 | ○ source build | △ source build候補 | POSIX Makefile + libusb。LinkE firmware更新専用 |

WCH-LinkUtilityのLinux/macOS欄は「Windows版をWine等で動かせない」という意味ではなく、公式のnative standalone packageを確認できなかった、という意味である。MounRiver Studio自体と、その中のWCH OpenOCDは別扱いである。

### 3.2 USB driverと権限

| host | 主な注意点 |
|---|---|
| Windows | 公式driverを保ったまま使えるtoolと、programming functionをWinUSBへ差し替えるtoolがある。ローカルINFのhardware IDでは`MI_00`（minichlink READMEの表記はinterface 1）。`wlink`のx86 buildは公式driver backendを持つ。nusb/libusb版、probe-rs、minichlinkではdriver bindingを事前に確認する必要がある |
| Linux | 通常はlibusb/nusbから直接開く。非root利用にはudev ruleが必要。LinkEのCDC serialは別device nodeになる |
| macOS | libusb/nusbからuser spaceで開けるが、Intel/Apple Siliconのarchitecture、署名・notarization、serial port名の変化が配布課題になる |

Arduino packageへ同梱する場合、「sourceからbuildできる」だけでは不足し、各architectureのbinary、USB権限設定、driverの競合、署名、更新時の互換性まで配布物の一部になる。

## 4. 書き込み・復旧機能

| tool | 入力形式 | program/verify | flash read | erase | 接続不能時復旧 | protection/option | system flash |
|---|---|---:|---:|---:|---:|---:|---:|
| WCH-LinkUtility | BIN/HEX等 | ○/○ | ○ | ○ | ○ power-off/NRST | ○/○ | ○ 対応系列 |
| WCH OpenOCD | ELF/BIN等 | ○/○ | ○ | ○ | ○ vendor command | ○/○ target依存 | △ config/command依存 |
| probe-rs | ELF/BIN/IHEX/UF2 | ○/○ | ○ memory read | ○ | △ 通常erase/unprotect。LinkE power-off/NRST専用eraseなし | ○ unprotect、optionはmemory model依存 | △ target definition依存 |
| wlink | ELF/IHEX/BIN | ○/△ | ○ dump | ○ | ○ power-off/pin-RST | ○ protect/unprotect | × 専用CLIなし |
| minichlink | BIN中心 | ○/○ | ○ | ○ | ○ power-cycle系 | ○、option領域の直接read/write | ○ named regionで扱えるtargetあり |
| rvprog.py | BIN | ○/○ | × | ○ | ○ power-cycle unbrick | ○ lock/unlock、V00X NRST option | × |

`wlink`のprogram処理には転送時の応答確認があるが、CLIには独立したfull-image verify commandがないため△とした。`probe-rs`の「復旧」は一般的なerase/unprotectであり、WCH公式protocolのpower-off eraseやpin-NRST eraseと同一ではない。

## 5. デバッグ、LinkE固有機能、運用機能

### 5.1 デバッグ機能

| tool | memory/register | halt/run/reset | step/breakpoint | GDB | DAP/IDE | symbol/source debug |
|---|---:|---:|---:|---:|---:|---:|
| MRS + WCH OpenOCD | ○ | ○ | ○ | ○ | ○ MRS | ○ |
| probe-rs | ○ | ○ | ○ | ○ | ○ VS Code DAP | ○ DWARF |
| wlink | ○ | ○ | × | × | × | × |
| minichlink | ○ | ○ | △ | ○ 限定実装 | △ VS Code/GDB経由 | △。step-over等の既知制限あり |
| WCH-LinkUtility | flash readのみ | reset-run | × | × | × | × |
| rvprog.py | × | reset only | × | × | × | × |

### 5.2 LinkE本体と開発時I/O

| tool | SDI printf | 物理UART bridge | 3.3/5 V | RV/DAP切替 | 複数Link指定 | Link firmware更新 |
|---|---:|---:|---:|---:|---:|---:|
| WCH-LinkUtility | ○ enable + COM受信 | ○ COMとして利用 | ○ | ○ | ○ GUI list | ○ online/offline |
| MRS + WCH OpenOCD | △ MRS機能 | ○ OS COM併用 | ○ vendor command | △ | ○/△ adapter設定 | ○ MRS prompt |
| probe-rs | × | △ 外部terminalのみ | × | × | ○ USB serial | × |
| wlink | ○ enable/watch | △ watchはprint-only | ○ | ○ experimental | △ USB index | × |
| minichlink | △ debug-wire terminal。公式SDI COMとは別実装 | △ 外部terminalのみ | ○ LinkE backend | ○ ARM検出時にRVへ切替 | ○ USB serial filter | × |
| rvprog.py | × | △ 外部terminalのみ | unbrick時のみ | ○ | × 最初のdevice | × |
| wlink-iap | × | × | × | IAP entry/exit | × | ○ IAP image書込 |

SDI printfと物理UART bridgeは、同じCOM portに見える場合があってもtarget側のdata pathが異なる。Arduino Serial Monitorとの統合では、upload用bulk interface、SDI用COM、target UART用COM、upload後の再enumerationを別々に識別する必要がある。

`wlink` 0.1.2はprobe一覧にUSB serialを表示する一方、selectorは`--device INDEX`である。接続順が変わる環境ではstable IDにならない。probe-rsと現行minichlinkはUSB serialによる指定が可能である。

### 5.3 LinkE firmwareとの組合せ

- probe-rs 0.32.0のWCH-Link backendはfirmware 2.7未満を拒否する。firmware自体の更新機能は持たない。
- wlinkのSDI printはfirmware 2.10以降を要求し、READMEで確認されているcurrent firmwareは2.15（WCH表示上のv35）である。
- WCH manual V2.4はSDI機能をLinkE firmware V1.80以降としている。tool間で`2.10`と`v30`のようにversion表示法が異なるため、生byte、tool表示、marketing表示を同じ値として比較できない。
- target自動検出や新MCU対応はhost toolだけでなくLinkE firmwareの応答にも依存する。Arduino packageがhost binaryだけを更新しても、旧LinkEが同じ対応範囲になるとは限らない。

## 6. MCU対応の読み方

LinkEのUSB protocol対応と、MCU対応は別層である。host toolが新しいMCUを扱うには、少なくともchip ID/family分類、memory map、flash容量・page/sector、flash algorithm、protection/option byte、debug core差分が必要になる。

| tool | target情報の置き場所 | 対応範囲の特徴 |
|---|---|---|
| WCH-LinkUtility / WCH OpenOCD | vendor binary/config | 公開OSSより新旧WCH系列を広く覆うが、差分追跡と再現buildが難しい |
| probe-rs | target YAML + flash algorithm + WCH family table | V003/V00X/V103/V20x/V30x/V317/X03x/H417等。0.32.0でMITのch32-data系metadataからのtarget生成と自動検出が更新されたが、V205/V407/V467/X315/M030等にはgapが残る |
| wlink | Rust enum/table + protocol operation | 主要系列を広く扱うが、V205/V407/V467/X315/M030等にgap。READMEは実機tested listと実装済みlistを厳密には分けていない |
| minichlink | `chips.c`とbackend | V003から複数CH32/CH5xx、WCH-Link以外の自作probeまで広い。系列ごとのdebug品質は均一ではない |
| rvprog.py | script内tableとstub | V00X/V103/V203/V208/V30x/X03x/L103、CH57x/58x/59x。小さく追いやすいがdebuggerではない |

詳細な系列表は[全経路の調査レポート](programming-tools-and-probes.ja.md#4-mcu系列別の現状)に置いている。

## 7. ライセンスと再配布の評価

「ライセンス的にきれい」は、license名だけでなく、対応sourceを入手できること、binaryを再現できること、target定義やfirmware blobの由来を説明できること、Arduino Board Manager packageへ再配布できることを含めて評価する。

| tool | license | source/binaryの対応 | Arduino packageへの組込み評価 |
|---|---|---|---|
| probe-rs | MIT OR Apache-2.0 | source、release、checksum、artifact attestationあり | **良好**。依存crateの通常のlicense inventoryは別途必要 |
| wlink | MIT OR Apache-2.0 | sourceとCI/releaseあり | **良好**。defaultのnusb経路が扱いやすい。Windows公式driver用x86 backendを同梱する場合はvendor DLL条件を分離して確認 |
| minichlink | MIT | sourceあり、repo内にWindows binaryとlibusb DLLも含む | **概ね良好**。同梱libusb/HIDの個別notice、versioned release・reproducible buildを整備する必要あり |
| rvprog.py | MIT | 小さい単一source | **良好**。Python/PyUSB/libusbをBoard Managerでどう配るかは別問題 |
| wlink-iap | MIT | updater sourceあり | **host codeは良好**。firmware imageはWCH binaryであり、host codeのMIT licenseだけではblobの再配布条件を解決しない |
| WCH OpenOCD | GPL系 | binaryはMRSにあるが、同じWCH改変版の対応source・build手順の追跡が容易でない | **要整理**。GPLだから悪いのではなく、配布binaryに対応するsource提供、notice、再現性を満たせるかが問題 |
| WCH-LinkUtility / MRS | proprietary | vendor binary | **vendor条件依存**。改変・自前配布・headless統合の自由度はOSSより低い |

### 7.1 「全部入りでlicenseもきれい」な既存toolはあるか

2026-08-26時点では、次の四条件を一つで満たす既存toolは確認できない。

1. WCH-LinkUtility相当のflash、復旧、protection、option/system flash、給電、mode切替、firmware更新
2. OpenOCD/probe-rs相当のstep、breakpoint、GDB/DAP、source debug
3. Windows/Linux/macOSの公式prebuiltとstableな複数probe指定
4. host source、target data、同梱artifactをOSS条件だけで説明・再配布可能

近さは次のように分かれる。

- **probe-rs**: 三OS配布、library API、debug、licenseは最も整っているが、LinkE固有管理機能が不足する。
- **wlink**: LinkE固有のpower、復旧erase、SDI、mode切替をcleanなRust sourceで持つが、GDB/DAPとstepがなく、production-readyではないとの自己表記がある。
- **minichlink**: licenseが単純で、LinkEと自作probeの双方を広く扱うが、debug品質、CLI契約、release engineeringは統合製品向けに揃っていない。
- **WCH公式組合せ**: 公開機能は最も広いが、単一cross-platform CLI、再配布、source追跡という条件から外れる。

したがって、既存の一製品をそのまま採用して全条件を満たす状態ではなく、cleanな既存OSSを核に不足するLinkE serviceを追加するか、複数toolを共通front-endで合成する形になる。

## 8. 理想のtool像

ここでは実装順ではなく、完成形の異なる構成を列挙する。

### A. probe-rs統合型

一つのRust library/CLIにWCH-Link transport、target DB、flash/debug、LinkE管理APIを収める。

- `flash`、`verify`、`erase`、`read`、`debug`、GDB、DAPはprobe-rsを利用
- LinkE extensionとしてpower、power-off/NRST recovery、SDI、RV/DAP切替、firmware version、option/system flashを追加
- probe selectionはVID:PID:serialをcanonical IDにする
- human CLIと同じoperationをJSON/structured APIでも公開する
- firmware更新は、license不明blobをcore releaseから切り離し、user-supplied imageまたはvendor downloaderへ委譲する選択肢を持つ

長所はdebug stackとcross-platform releaseを再利用できること、短所はWCH固有機能を汎用probe abstractionへどう収めるかの設計負荷である。

### B. core + LinkE管理service分離型

probe-rsはflash/debugだけに保ち、別の小さい`linkectl` library/CLIがLinkE本体を管理する。Arduino側は一つのwrapperから両者を呼ぶ。

- `probe-rs`: target検出、flash、verify、debug、GDB/DAP
- `linkectl`: list/info、serial指定、power、recovery erase、SDI session、UART discovery、mode、firmware IAP
- 共通device lockを持ち、二processが同じUSB interfaceを同時openしない
- firmware artifactの配布policyをhost codeから独立させる

長所はupstreamとの境界とlicense inventoryが明瞭なこと、短所はatomicなupload操作、error model、USB再enumerationをwrapperで調停する必要があることだ。

### C. wlink完全化型

WCH専用suiteとして`wlink`を拡張し、公式機能の再現を優先する。

- 現在のflash、memory/register、protection、power、recovery、SDI、mode切替を基礎にする
- stable serial selector、explicit verify、option/system flash、Link firmware IAPを追加
- GDB serverまたはDAP adapter、step/breakpoint、DWARF layerを追加
- chip tableを生成dataに替え、Arduino coreと同じdevice databaseから生成する

長所はWCH protocolに自然なCLIを作れること、短所はprobe-rs/OpenOCDが既に持つdebugger機能を大きく再実装することになる。

### D. minichlink universal programmer型

LinkEだけでなくESP32-S2、RP2040、STM32、AVR-Arduino等の互換probeも同じhost APIで扱う。過去機材と自作probeの互換性を重視した姿である。

- backend capabilityを列挙し、unsupported operationを実行前に返す
- 現行の単文字CLIに加えてstable long optionとJSON resultを提供
- flash format parser、target database、transport backendを分離
- Windows/Linux/macOSのversioned binary、udev/driver metadata、SBOMを配布
- GDB機能は限定範囲を明示するか、debugだけprobe-rs bridgeへ委譲する

長所はWCH-LinkE以外の資産を一つの入口で使えること、短所はbackendごとの機能差が大きく、「全deviceで全機能」を保証できないことだ。

### E. Arduino upload broker型

下位toolを置き換えず、Arduino IDEから見える契約だけを統一する常駐serviceまたは単発brokerである。

- Board Manager packageは`wch-upload`だけを直接呼ぶ
- backendはprobe-rs、wlink、minichlink、公式OpenOCDを選択可能
- `discover --json`、`capabilities --json`、`upload --probe SERIAL`、`recover`、`monitor`を共通化
- upload portとmonitor portを別IDとして追跡し、再enumeration後に再接続
- GUIはbroker APIの薄いclientとし、CLI/headless/CIと同じ実装を使う

長所は既存機材とvendor fallbackを保ちやすいこと、短所はbackend差を隠し過ぎると、失敗理由や利用可能機能が不透明になることだ。capability negotiationが必須になる。

## 9. 理想形に共通する外部仕様

構成A〜Eのどれでも、次のcontractがあるとArduino、CLI、CI、GUIを同じ実装へ接続できる。

| 項目 | 外部仕様の例 |
|---|---|
| probe discovery | model、USB VID/PID/interface、serial、firmware、mode、利用中状態をJSONで返す |
| target discovery | detected family、exact/ambiguous、chip ID、flash size、protectionを根拠付きで返す |
| capability | flash/read/debug/power/SDI/UART/mode/IAPをprobe・firmware・targetの組ごとに返す |
| upload | input format、address、erase policy、verify policy、reset policyを明示する |
| recovery | power-off、NRST、connect-under-reset、mass eraseを別operationにする |
| monitor | physical UART、SDI、RTTを別transport名にし、portとbaudを返す |
| concurrency | USB serial単位のlock、timeout、cancel、安全なdetachを定義する |
| error | driver/permission、probe firmware、unsupported target、protected flash、wiringを別codeにする |
| reproducibility | version、source revision、target DB revision、flash algorithm hash、SBOMを出力する |

特に重要なのは、`tool supports LinkE`という一つのbooleanではなく、**probe model × probe firmware × target family × operation**のcapabilityを返すことだ。これにより、旧LinkE、複数probe、自作互換probe、新旧MCUが混在しても、Arduino側が誤ったmenuやoperationを提示しにくくなる。

## 10. 参照資料

- [WCH-Link User Manual download page](https://www.wch-ic.com/downloads/WCH-LinkUserManual_PDF.html)
- [WCH-Link manual V2.4](https://www.wch.cn/uploads/file/20250124/1737704462135866.pdf)
- [probe-rs 0.32.0 release](https://github.com/probe-rs/probe-rs/releases/tag/v0.32.0)
- [probe-rs repository](https://github.com/probe-rs/probe-rs)
- [probe-rs WCH SDI issue #3023](https://github.com/probe-rs/probe-rs/issues/3023)
- [wlink repository](https://github.com/ch32-rs/wlink)
- [wlink protocol notes](https://github.com/ch32-rs/wlink/blob/main/protocol.md)
- [wlink release workflow](https://github.com/ch32-rs/wlink/blob/main/.github/workflows/ci.yml)
- [ch32fun/minichlink](https://github.com/cnlohr/ch32fun/tree/master/minichlink)
- [rvprog.py](https://github.com/wagiminator/MCU-Flash-Tools/blob/main/rvprog.py)
- [wlink-iap](https://github.com/cjacker/wlink-iap)
- [ch32-rs toolchain overview](https://github.com/ch32-rs/ch32-rs)

### 調査したローカル版

- `../probe-rs`: `40388f2b` (2026-08-24)、release 0.32.0
- `../ch32fun`: `6c4dd53` (2026-08-24)
- `../tools/wlink/0.1.2`: Linux x64 release binary
- `../tools/MounRiverStudio_Linux_X64_V2.5.0`: Linux版MRSとWCH OpenOCD同梱を確認
- 一時cloneしたwlink: `249f2c1` (2026-05-01)、release 0.1.2

release、nightly、WCH-Link firmware、MCU target tableは更新されるため、表は上記調査日時点のsnapshotである。
