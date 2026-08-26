# CH32 RISC-V書き込み・デバッグ・bootloader経路と独自プローブの事前調査

- 調査基準日: 2026-08-26
- 目的: Arduino環境で採用する全書き込み経路と、独自ホストツール／bootloader／LinkE代替プローブを作る場合の境界・優先順位を決める
- 状態: 方向性決定用の事前調査。ここに書かれた対応は、実機認定済みを意味しない

## 1. 結論

### 1.1 先に進めるべきもの

1. **既存の`probe-rs`経路を捨てない。** 現在の0.32.0はWCH-Linkによるチップ自動認識、ELF/BIN/IHEX書き込み、メモリ・レジスタ操作、実行制御、GDB/DAPを一つのライブラリで扱える。配布もWindows、Linux x86-64/AArch64、macOS x86-64/Apple Siliconをカバーする。
2. **最初に作る独自ホストソフトは、デバッガ本体ではなく薄い`ch32-upload`が妥当。** Arduinoから安定したCLIを提供し、`probe-rs`、必要なら`wlink`、WCH OpenOCD、`wchisp`、独自bootloaderをtransport/backendとして隠す。
3. `probe-rs` 0.32.0で不足する主要系列は、手元の対象では **CH32V205 / CH32V407・V467 / CH32X315 / CH32M030**。一方、ローカルにあるMounRiver Studio 2.40付属WCH OpenOCDバイナリには、これらすべての文字列とCH32H417 dual-core設定が入っている。短期fallbackには使えるが、再現ビルドとライセンス対応が弱い。
4. **LinkE代替ハードは作れる。** CH32V003の1線SWIOは公式資料・公式CH32F103実装・PicoRVD・ESP32・CH32V003自身を使う実装がある。2線RVSWDもRINSの解析資料、`rv003usb/rvswdio_programmer`、ESP32-S2 funprog、2026年公開のRP2040向けSwindleが実装済みである。
5. **通常のArm SWDではない。** WCHの2線インターフェースは配線名がSWDIO/SWCLKでも、プロトコルはArm SWD非互換のRVSWD/2-wire SDIである。通常のRaspberry Pi Debug ProbeやCMSIS-DAP firmwareを焼くだけではCH32 RISC-Vを扱えない。
6. **カスタムbootloaderも第一級の経路にする。** V003/V00XではUARTまたはsoftware USB、USB内蔵品ではvendor USB ISP、DFU、UF2、UARTを選べる。さらにEthernet、無線、USBメモリ等からapplicationがimageを受け取り、IAPする経路もある。これはdebug probeの代替ではなく、通常更新を簡単にする別経路である。

### 1.2 推奨する開発順

- 短期: `probe-rs`の不足target追加、SDI print・電源・probe情報などの不足機能を上流へ追加する。
- 並行: backend差を吸収する`ch32-upload`の仕様を作る。曖昧なprobeを拒否し、JSON出力と安定した終了コードを持たせる。
- 並行: Arduino board定義に`upload transport`と`bootloader layout`を分けて持たせ、LinkE、factory ISP、custom bootloaderを同じfrontendから選べるようにする。
- 独自probe PoC: RP2040で1線SWIOと2線RVSWDのDMI read/writeを実証する。
- 独自probe MVP: V003/V00XとV20x/V30xを対象に、flash/verify/reset/unbrickと一意なUSB serialを実装する。
- 独自bootloader PoC: V003のsoftware USB HIDとUART、V203/V307のhardware USB DFUまたはUF2を別profileで実証する。
- その後: UART CDC、SDI print、GDB/DAP、V103/X035/L103、未対応系列、H417 dual-coreを追加する。

「全LinkE互換」を最初の完成条件にはしない。特にARM CMSIS-DAP/JTAG、無線LinkW互換、5 V給電、全WCH BLE/USB MCU対応は別スコープとする。

## 2. まず分けるべき7つの層と更新経路

各プロジェクトは同じ「書き込みツール」に見えても、担当する層が違う。

| 層 | 役割 | 既存例 | 自作時の置き場所 |
|---|---|---|---|
| Arduino frontend | `platform.txt`、board選択、エラー表示、probe選択 | Arduino recipe、PlatformIO | `ch32-upload` CLI |
| image/package | link offset、最大sketch容量、header、CRC/署名、UF2変換 | linker script、`uf2conv.py`、DFU suffix | board/upload profileから生成 |
| ホストdebug/flash | ELF解析、target DB、flash algorithm、実行制御、GDB/DAP | probe-rs、OpenOCD、minichlink | 原則は既存library再利用 |
| USB probe transport | 列挙、serial、command framing、batch転送 | WCH-Link bulk、funprog HID、Black Magic GDB serial | version付きopen protocol |
| debug物理層 | 1線SWIOまたは2線RVSWDでDMIを転送 | LinkE、PicoRVD PIO、Swindle PIO | probe MCUのPIO/RMT/timer |
| target内boot/update | reset直後の判定、image受信、erase/write/verify、appへjump | factory ISP、rv003usb、tinyboot、DFU/UF2 | system flashまたはuser flashの予約領域 |
| target固有処理 | chip ID、memory map、option bytes、erase/write stub | probe-rs YAML、minichlink `chips.c`、WCH OpenOCD | 更新しやすいホスト側DB |

重要なのは、**ターゲット固有の知識をprobe firmwareへ詰め込みすぎない**こと。probeはdebug物理層、DMI batch、NRST、電源、UARTに集中させる。chip ID表やフラッシュアルゴリズムをホスト側で更新できれば、新MCU追加のたびにprobe firmwareを配り直さずに済む。

bootloader経路では逆に、最低限のflash処理はtarget内に必要になる。ただし受信transport、image検証、boot状態遷移、family別flash HALを分離すれば、UART/USB/RS-485/ネットワークで同じ更新protocolを再利用できる。

### 2.1 Arduinoから見た全書き込み経路

各経路は直列の一枚岩ではなく、`ch32-upload`の下で分岐する。

| 経路 | targetへの最終入口 | 初回blank書込 | 壊れたappの救出 | source debug | 通常更新の使いやすさ | 互換対象 |
|---|---|---:|---:|---:|---:|---|
| 公式LinkE | SWIO/RVSWD | ○ | ○ | ○ | △ 外部配線 | 既存WCH機材、MRS、probe-rs/wlink |
| 独自debug probe | SWIO/RVSWD | ○ | ○ | ○ | △ 外部配線 | 将来の自作fixture/probe |
| WCH-MCU-DL/offline writer | SWIO/RVSWDまたはUSB/UART ISP | mode依存で○ | mode依存 | × | ○ PCなし量産 | 既存生産設備、rolling code、machine制御 |
| factory USB ISP | system/BOOT領域 + USB device | chip依存で○ | ○、BOOT entry可能時 | × | ○ | WCHISPTool、wchisp、既存USB board |
| factory UART ISP | system/BOOT領域 + UART | chip/entry条件依存 | △ app協調しか入口がない品種あり | × | △ USB-UARTが必要 | WCHISPTool、wchisp、既存fixture |
| custom software USB BL | GPIO bitbang USB LS | × 事前導入必要 | ○ BLが無傷なら | × | ○ driverless HID可 | V003/V00XのUSB端子付き独自board |
| custom hardware USB BL | USB peripheral | × 事前導入必要 | ○ BLが無傷なら | × | ○ DFU/UF2/HID | V103/V2xx/V3xx/V4x7/X03x/X315/M030等の該当SKU |
| custom UART/RS-485 BL | UART | × 事前導入必要 | ○ BLが無傷なら | × | ○ 製品配線を共有可 | USBなしboard、既設制御bus |
| application IAP/OTA | appが受信しflash APIを呼ぶ | × | △ appが動く間だけ | × | ○ 遠隔更新向け | Ethernet/Wi-Fi/BLE/CAN/LoRa等 |
| USB host/外部媒体 IAP | targetのUSB host、SD等 | × | △ boot stub次第 | × | ○ PC不要 | 保守員によるUSBメモリ更新等 |

`初回blank書込`、`開発時debug`、`通常の利用者更新`、`brick recovery`を一つの経路だけで満たそうとしない。推奨する役割分担は次である。

- 開発機の製造・救出: LinkEまたは独自SWIO/RVSWD probeを必ず残す。
- 量産: PC接続fixtureのほか、既存WCH-MCU-DLによるoffline書込も互換経路として残す。
- 開発: 同じdebug経路を使い、GDB/DAPとflashを統合する。
- 通常更新: boardに応じてfactory ISPまたはcustom bootloaderを使う。
- 現場更新: UART/RS-485、ネットワークIAP、USBメモリ等を製品要件に応じて追加する。
- bootloader自身の更新: 通常app更新とは分離し、原則debug probeだけに許す。自己更新するなら電源断から復帰できる二段構成が必要。

## 3. WCH側インターフェースの整理

### 3.1 debug interface

| 系列 | 主なcore | debug配線 | 備考 |
|---|---|---|---|
| CH32V003、CH641 | QingKe V2A | 1線SWIO/SDI | データのパルス幅で0/1を表す。外部pull-upが必要な自作例が多い |
| CH32V00X、CH32M007 | QingKe V2C | 1線SWIO/SDI | V003とはflash page・memory構成が異なる |
| CH32V103、CH56x/57x | QingKe V3系 | 2線RVSWD | 配線名はSWDIO/SWCLKだがArm SWDではない |
| CH32V20x/V30x/V317、X03x、L103、CH643等 | QingKe V4系 | 2線RVSWD | 既存OSS実装が最も多い2線系列 |
| CH32M030 | QingKe V2C系 | 1線/2線の両対応と公式資料に記載 | 工場ISPがなく、debug経路の重要度が高い |
| CH32H41x | dual core | 2線RVSWD | core選択、個別work area、dual-core resetを別途扱う |
| CH32V4x7、CH32X315 | 新しい2線系列 | 2線RVSWD | OSS target DBが追いついていない |

WCH自身もQingKeを「1-wire/2-wire DTM」と説明している。2線RVSWDの公開一次仕様は十分ではないが、[RINS](https://perigoso.github.io/rins/)は物理・論理層を第三者実装向けに整理しており、[WCH RVSWD protocolの初期解析](https://github-wiki-see.page/m/fxsheep/openocd_wchlink-rv/wiki/WCH-RVSWD-protocol)とも整合する。

### 3.2 WCH-Link USB host protocol

WCH-LinkのRISC-V modeはUSB bulk (`1a86:8010`)で、概ね次を扱う。

- probe型・firmware版取得
- target attachとchip family/ID取得
- debug速度設定
- DMI read/write/nop
- reset/detach
- flash protection、電源制御、特殊erase、SDI print等のvendor command

パケットは`0x81, command, length, payload...`、成功応答は`0x82,...`である。[wlinkのprotocol.md](https://github.com/ch32-rs/wlink/blob/main/protocol.md)と[RINSのWCH-Link資料](https://perigoso.github.io/rins/wch-link/index.html)に解析結果がある。

`probe-rs`のWCH-Link backendは、attach、速度設定、DMI、reset、flash protection解除という比較的小さなsubsetを使い、フラッシュ書き込み自体はtarget側flash algorithmをDMI経由で動かす。これは独自probe設計に好都合で、低レベルDMI transportを追加できれば上位機能を再利用できる。

### 3.3 factory ISPとcustom IAPを区別する

WCHの資料ではISP/IAP/BOOTという語が混在するため、設計上は次のように区別する。

| 種類 | 格納場所 | 誰が導入するか | 主目的 | 消去事故への強さ |
|---|---|---|---|---|
| factory ISP | system/BOOT領域 | 工場出荷時 | USB/UARTからcode flashを書込 | code flash消去後も残るが、option/entry条件に依存 |
| custom system bootloader | 書換可能なsystem/BOOT領域 | 最初にdebug probe等で導入 | factory ISP置換、user flashを全量確保 | system領域を書き損じるとprobe救出が必要 |
| custom user-flash bootloader | code flash先頭等 | 最初にdebug probe/factory ISPで導入 | DFU/UF2/UART/OTA | mass eraseで消える。appのlink offsetが必要 |
| application IAP | 通常application内 | appと同時 | 稼働中に受信して自己更新 | app破損時には入れない。小さいrecovery stub併用が望ましい |

CH32V003のfactory bootloaderは`0x1FFFF000`からの1,920 byte system領域にあり、UART 115200 bpsで動く。外部BOOT pinだけではなくapplicationが`START_MODE`を設定してresetする必要があるため、appが完全に壊れた場合の入口としては弱い。factory bootloaderの挙動とprotocolは[CH32V003 factory bootloader missing manual](https://github.com/basilhussain/ch32v003-bootloader-docs)に詳しい。V00Xではsystem領域の大きさや一部packageのUART remapが異なる。

USB内蔵系列のfactory ISPは`wchisp`/WCHISPToolから使えるが、「USB peripheralがある」と「そのSKUのfactory bootloaderがUSB経路を公開する」は同義ではない。正確な対応は**exact SKU、BOOT pin/option byte、bootloader version**で管理する。

### 3.4 WCH/OSSのcustom bootloader・IAP事例

| 実装 | target/transport | 配置・image条件 | host | 主な機能と注意 |
|---|---|---|---|---|
| WCH公式EVT `USART_IAP` | V003/V00X、UART | IAPとoffset付きAPPの組 | WCHMcuIAP sample | erase/program/verify/jump。factory BLとは別のsource公開sample |
| WCH公式EVT `UART_USB_IAP` / `USB_UART` | V103、V30x、V407、X035、X315、M030等 | user flash予約 + APP offset | WCHMcuIAP sample | UARTとhardware USB deviceの両入口。family別sourceが手元にある |
| WCH公式EVT `ETH_IAP` | V307 Ethernet | user flash予約 | sample protocol | network IAPの具体例。製品向けには認証・rollback追加が必要 |
| WCH公式EVT `HOST_IAP` | V103/V307/X035/M030等のUSB host | targetがUSBメモリを読む | PC不要 | 現場更新の例。媒体上imageの真正性確認が別途必要 |
| [rv003usb bootloader](https://github.com/cnlohr/rv003usb/tree/master/bootloader) | V003、GPIO software USB low-speed HID | 1,920 byte system領域 | minichlink | driver不要、約5秒timeout/button/host検出。bootloader自身は自己更新しない |
| `rv003usb/bootloader_v006` | V006系、software USB | V00X向け別実装 | minichlink | 旧V003版と十分異なり、現状README整備途上 |
| [tinyboot](https://github.com/OpenServoCore/tinyboot) | V003/V00X/V103、UART・1線UART・RS-485 | systemまたはuser flash mode | Rust `tinyboot` CLI | CRC16、info、retry、trial boot/confirm。USB/SPI/radioへtransport拡張可能 |
| [wch-uf2](https://github.com/ArcaneNibble/wch-uf2) | CH32V2xxのUSBD | 先頭4 KiB予約、APP `0x08001000` | OSのMSC + UF2 copy | double reset、flash/RAM download。V3xx非対応、hardcoded値のfamily化が必要 |
| [Swindle CH32V3x DFU BL](https://github.com/mean00/swindle_bootloader_ch32v3x) | CH32V3x hardware USB | 先頭16 KiB予約、実サイズ約6 KiB、APP `0x4000` | `dfu-util` | RAM marker/button/invalid CRCでDFU、12-byte header+CRC32 |
| [PlumBL](https://github.com/HaiMianBBao/PlumBL) | CH32V30xほか、CherryUSB DFU/U2F | user flash予約 | `dfu-util`/U2F tool | multi-platform port例。採用時は対応commitとlicenseを固定 |

これらから、V003のようなUSB peripheralなしの品種でもsoftware USB更新は可能であり、USB内蔵品では標準classを使うDFU/UF2が実用候補になる。またUART IAPはほぼ全familyへ移植しやすく、RS-485/1線UARTに拡張すれば製品の既存connectorをそのまま更新口にできる。

### 3.5 custom bootloader transportの選択

| transport/protocol | driver/host依存 | 速度目安 | MCU資源 | 長所 | 短所 |
|---|---|---:|---:|---|---|
| software USB HID | OS標準HID、専用CLI | USB low-speed | 極小だがcycle制約が厳しい | V003でもUSB端子だけで更新 | pin/clock/割込み制約、signal品質、USB認証試験 |
| hardware USB vendor/HID | HIDならdriver不要 | FS/HS | 中 | WCH sample/minichlink互換を作りやすい | 独自host protocol保守 |
| USB DFU | `dfu-util`等 | FS/HS | 中 | 標準protocol、CLIが既存 | Arduino側でDFU image/offset管理が必要 |
| USB UF2 MSC | OSのfile copy | FS | 大きめ | 利用者UXが最良、追加driver不要 | MSC emulationが複雑、copy完了判定・大image再起動に注意 |
| UART | USB-UART/serial API | 115.2 kbps～ | 小 | USBなしを含め移植しやすい | port選択、baud、reset/entry配線がboard依存 |
| 1線UART/RS-485 | transceiverまたはhalf-duplex adapter | 配線依存 | 小 | 長距離・multi-drop・既設bus | node address、衝突回避、DE timingが必要 |
| Ethernet/Wi-Fi/BLE等 | network stack/app | 可変 | 大 | 遠隔更新 | 認証、暗号、再送、rollbackが必須。Arduino uploadとは別serviceにもなる |
| USB host/SD/SPI flash | 外部媒体 | 可変 | host/filesystem分 | PCなしで更新、staged update | 媒体破損・image選択・電源断対策 |

Arduino開発用の既定は、V003/V00X boardなら`software USB HID`または`UART`、native USB boardなら`DFU`を第一候補とする。UF2は教育・配布boardで特に有力だが、IDEのUploadボタンでは結局mount point discovery/copy/completion監視が必要なので、`ch32-upload`で包む。

### 3.6 bootloader共通仕様で外せない項目

- image header: magic、format version、target family/exact board、load address、長さ、version、hash。
- integrityとauthenticityを分ける。CRCは転送事故検出であり、署名検証の代わりではない。
- anti-rollbackが必要かを製品profileで決める。開発boardでは解除可能なrecovery方法も要る。
- erase途中や電源断後に少なくともbootloaderへ戻れること。大容量品はA/B slotまたはdownload slot→copyを検討する。
- trial boot、watchdog、applicationからの`confirm()`、失敗回数上限を定義する。
- boot entryをbutton、double reset、RAM magic、application command、無効imageの論理和にする。
- bootloader領域、metadata page、application offset、最大sketch容量をlinkerとArduino metadataから同一生成する。
- vector table/interrupt、clock、USB pull-up、peripheral、stack、`.data/.bss`をappへjumpする前に規定状態へ戻す。
- bootloader updaterは通常firmware updaterと分離し、system/BOOT領域への書込みには明示flagとprobe recoveryを要求する。
- USB VID/PID、serial、DFU alt setting、UF2 family ID、UART node IDをboard identityとして一元管理する。

### 3.7 Arduino統合で必要なprofileモデル

「boardにupload toolを1個固定」では足りない。同じboardに複数の経路を定義する。

```text
board
  mcu: CH32V203C8T6
  upload_profiles:
    - debug-probe: probe-rs + LinkE/custom probe, ELF@0x00000000
    - factory-usb: wchisp, BIN@0x08000000
    - custom-dfu: dfu-util, BIN@0x08004000 + header
    - custom-uart: ch32-upload boot, BIN@profile.app_origin
  recovery_profile: debug-probe
  bootloader:
    kind / version / reserved_ranges / metadata_range / entry_methods
```

profileを選ぶと、少なくとも次を連動させる。

- linker scriptのapplication origin/length
- `upload.maximum_size`と予約領域
- ELF→BIN/HEX/UF2/DFU変換とheader/署名付加
- port discovery（probe serial、USB VID/PID+serial、TTY、mount point、network identity）
- 1200-bps touch、DTR/RTS、RAM magic、vendor command等のboot entry
- upload後にUSB portが別VID/PIDで再enumerateする場合の追跡
- bootloaderなしのboardへoffset付きappだけを書かないpreflight check

互換性のため、最初は`wch-link`、`factory-usb`、`factory-uart`という既存経路を安定IDで残す。独自実装は`custom-probe`、`custom-dfu`、`custom-uf2`、`custom-uart`等として追加し、内部backendを替えてもArduinoのprofile名とCLI終了コードを変えない。

## 4. ホストツール比較

記号: ○=実装あり、△=制限・別操作・要確認、×=目的外、?=資料だけでは確定できない。

### 4.1 対応host・配布・開発基盤

| tool | 主用途 | 実装 | prebuilt host | license/配布上の注意 |
|---|---|---|---|---|
| [probe-rs 0.32.0](https://github.com/probe-rs/probe-rs/releases/tag/v0.32.0) | probe統合、flash、debug | Rust library/CLI | Win x64、Linux x64/ARM64、macOS x64/ARM64 | MIT OR Apache-2.0。単一配布物を作りやすい |
| [wlink 0.1.2](https://github.com/ch32-rs/wlink) | WCH-Link専用CLI/library | Rust、nusb/libusb | Win x64/x86、Linux x64、macOS ARM64 | MIT OR Apache-2.0。README自身がproduction-readyではないとする |
| [minichlink](https://github.com/cnlohr/ch32fun/tree/master/minichlink) | WCH/互換probe/ISP統合 | C、libusb/HID/serial | source build: Win/Linux/macOS/FreeBSD/NetBSD | MIT。versioned release assetがなく自前buildが必要 |
| WCH OpenOCD | WCH-Link flash/debug | WCH fork of OpenOCD | MRS/公式coreにWin/Linux/macOS版 | GPL系だが対応sourceと再現buildの入手性が問題。binary fallback向け |
| WCH-LinkUtility | LinkE GUI、firmware管理 | proprietary GUI | 主にWindows | 公式。再配布条件・自動化・headless運用に弱い |
| [WCH-MCU-DL](https://www.wch.cn/uploads/file/20240821/1724227120114035.pdf) | offline/batch writer | proprietary hardware + 設定software | PCで事前設定後はstandalone | USB/UART/SWD mode、check、rolling code、machine I/O。量産互換用 |
| [wchisp 0.3.0/nightly](https://github.com/ch32-rs/wchisp) | USB/UART factory ISP | Rust、libusb/serial | Win x64、Linux x64/ARM64、macOS ARM64 | GPL-2.0。WCH-Link用ではない |
| [WCHISPTool_CMD](https://www.wch.cn/downloads/WCHISPTool_CMD_ZIP.html) | 公式USB/UART ISP CLI | proprietary CLI + library/sample | Win x86/x64、Linux x64、macOS x64/ARM64 | 設定INI生成にWindows GUIを要求する構成が使いにくい |
| [dfu-util](https://dfu-util.sourceforge.net/) | USB DFU custom BL | C/libusb | Win/Linux/macOS等 | GPL-2.0。標準DFUならMCU vendor非依存 |
| [tinyboot CLI](https://github.com/OpenServoCore/tinyboot) | UART/RS-485 custom BL | Rust/serial | source build、Cargo install | MIT OR Apache-2.0。V003/V00X/V103対応 |
| UF2 file copy | USB MSC custom BL | OS file API | MSCをmountできるhost | host tool不要に見えるがIDE統合にはvolume discoveryが必要 |
| [Swindle](https://github.com/mean00/swindle) | probe上GDB server | C/C++ + Rust + Black Magic | probe firmware。hostはGDBが動く環境 | GPL-3.0系。RP2040はstable表記、CH32はV2xx/V3xx |
| [PicoRVD](https://github.com/aappleby/picorvd) | V003用probe上GDB server | RP2040 SDK + PIO | probe firmware。hostはGDB | MIT。READMEでvery alpha |
| [rvprog.py](https://github.com/wagiminator/MCU-Flash-Tools/blob/main/rvprog.py) | 小さいWCH-Link flasher参考実装 | Python/USB | Python環境 | 対象・機能は限定。protocol理解用 |

`probe-rs`、`wlink`、`wchisp`のhost欄は2026-08-26時点のrelease/CI matrixであり、sourceからの追加build可能性とは分けている。

### 4.2 機能マトリクス

| tool | ELF/HEX/BIN flash | verify/readback | erase/unbrick | memory/register | halt/run/step | GDB/DAP | SDI print | power/NRST | 複数probe指定 |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| probe-rs | ○ | ○ | ○/△ | ○ | ○ | ○ | × | resetのみ | ○ serial |
| wlink | ○ | △ dump可 | ○ power-off/pin-RST | ○ | halt/run/reset、stepなし | × | ○ | ○ 3.3/5 V | △ indexのみ |
| minichlink | BIN中心 | ○ | ○ | ○ | ○/限定 | ○/限定 | ○ | backend依存 | ○ LinkE serial |
| WCH OpenOCD | ○ | ○ | ○ | ○ | ○ | ○ GDB | ? | ○ commandあり | △ index |
| WCH-LinkUtility | BIN/HEX等 | ○ | ○ power/NRST | flash read | debugはMRS側 | MRS側 | ○ | ○ | ○ GUI選択 |
| WCH-MCU-DL | BIN/HEX | ○ check | ○/mode依存 | × | × | × | × | ○ target power | 1台へ事前設定 |
| wchisp | ○ | ○ | ISP erase | config/EEPROM | × | × | × | × | △ index |
| WCHISPTool_CMD | ○ | ○ | △ | × | × | × | × | × | port/location指定 |
| Swindle | GDB `load` | GDB経由 | ○/target限定 | ○ | ○ | ○ GDB | RTT、非SDI | NRST | USB CDC path |
| PicoRVD | GDB `load` | △ | ○ V003 | ○ | ○ | ○ GDB | console別 | reset | USB CDC path |

補足:

- probe-rsの[公式機能](https://github.com/probe-rs/probe-rs)は、任意メモリread/write、halt/run/step/breakpoint、ELF/BIN/IHEX、CLI/VS Code DAP/GDBを含む。ただしWCH-LinkのSDI printは[未解決issue](https://github.com/probe-rs/probe-rs/issues/3023)である。
- wlinkはSDI printとLinkE UART watchを同一sessionで扱える点が強い。一方、0.1.2ではUSB serialを表示できてもselectorは`--device INDEX`だけである。
- WCH公式マニュアルは、LinkUtilityについてprogram/verify/reset-run、chip情報、flash read、read protection、power/NRST erase、3.3/5 V制御、複数Link選択、option bytes、system flash、SDI virtual serialを列挙している。[WCH-Link manual V2.4](https://www.wch.cn/uploads/file/20250124/1737704462135866.pdf)
- `wchisp`とWCHISPTool_CMDはfactory bootloader経路であり、ブランク時やBOOT entry条件を満たせない場合はLinkE代替にならない。

## 5. MCU系列別の現状

これはsource上のtarget定義・chip tableを比較した表であり、全セルの実機確認表ではない。

| MCU系列 | debug | probe-rs 0.32 | wlink 0.1.2 | minichlink 2026-08-24 | MRS 2.40 WCH OpenOCD | OSS自作probeの根拠 |
|---|---|---:|---:|---:|---:|---|
| V003 / CH641 | 1線 | ○ | ○ | ○ | ○ | PicoRVD、ESP32、rvswdio、公式1-line例 |
| V00X / M007 | 1線 | ○ | ○ | ○ | ○ | rvswdio、ESP32-S2コード |
| V103 | 2線 | ○ | ○ | ○ | ○ | generic RVSWD実装はあるがrvswdio READMEでは非対応 |
| V203 / V208 | 2線 | ○ | ○ | ○ | ○ | Swindle、rvswdio、ESP32-S2 |
| V205 | 2線 | **×** | **×** | ○ table | ○ | 物理層はV20x系。flash/ID認定が必要 |
| V303 / V305 / V307 | 2線 | ○ | ○ | ○ | ○ | Swindle、rvswdio、ESP32-S2 |
| V317 | 2線 | ○ | ○ | ○ | ○ | generic RVSWDコードはある。Swindle targetは未追加 |
| V407 / V467 | 2線 | **×** | **×** | **×** | ○ | protocolよりtarget固有flash/IDが不足 |
| X033 / X035 | 2線 | ○ | ○ | ○ | ○ | rvswdio、ESP32-S2 |
| X315 | 2線 | **×** | **×** | **×** | ○ | protocolよりtarget固有flash/IDが不足 |
| L103 | 2線 | ○ | ○ | ○ | ○ | generic RVSWD実装を移植可能 |
| M030 | 1/2線 | **×** | **×** | ○ table | ○ | rvswdio/ESP32-S2にfamily codeあり、実機認定要 |
| H415/H416/H417 | 2線、dual | ○ | ○ | ○ table | ○ dual config | 単一DMIだけでなくcore選択・flash配置の実装要 |

### 5.1 family別のupload route候補

`factory`欄はfamily全体の保証ではない。USB pinがbond-outされないpackage、BOOT entry pin/option、出荷bootloader版で変わるので、最終DBはSKU/package単位にする。

| family | debug probe | factory ISP候補 | 公開custom boot/IAPの根拠 | 推奨するArduino初期profile |
|---|---|---|---|---|
| V003 | SWIO | UART、appからSW entry | rv003usb software USB、tinyboot UART/RS-485、WCH USART_IAP | `wch-link` + `factory-uart`、USB boardだけ`softusb-hid` |
| V00X | SWIO | UART、appからSW entry、packageでpin差 | tinyboot、rv003usb V006、WCH USART_IAP | `wch-link` + `factory-uart` |
| V103 | RVSWD | USB + USART1、BOOT0/1 | tinyboot、WCH UART_USB_IAP/HOST_IAP | `wch-link` + `factory-usb`、後にcustom UART |
| V203/V208 | RVSWD | USB + USART1、BOOT0/1 | wch-uf2、WCH系USB/UART IAP | `wch-link` + `factory-usb`、DFU/UF2実験profile |
| V205 | RVSWD | USB + USART2、BOOT0/1、小packageはISP不可あり | generic USB/UART IAPを移植 | 当面WCH OpenOCD + factory ISP |
| V30x/V317 | RVSWD | USB + USART1、BOOT0/1、小package注意 | Swindle DFU、WCH USB/UART/USB-host/Ethernet IAP | `wch-link` + `factory-usb`、後にDFU |
| V407/V467 | RVSWD | USB + USART1、BOOT0/1 | 手元WCH USB_UART IAP、USB-host IAP | 当面WCH OpenOCD + factory ISP |
| X033/X035 | RVSWD | USB + UART、appからSW entry可 | 手元WCH USB_UART/HOST_IAP | `wch-link` + `factory-usb` |
| X315 | RVSWD | USBFS + UART keyless、SW entry可 | 手元WCH USB_UART/HOST_IAP | 当面WCH OpenOCD + factory ISP |
| L103 | RVSWD | USB + USART2、BOOT0/1 | generic UART BLは移植可能 | `wch-link` + `factory-usb` |
| M030 | SWIO/RVSWD | factory ISPなしとして扱う | 手元WCH UART_USB_IAP/HOST_IAP | WCH OpenOCDで初回導入 + custom USB/UART |
| H41x | RVSWD、dual | USBFS + UART、SW entry可、optionで経路制御 | family HAL/dual-core boot設計が必要 | `wch-link` + factory ISPから開始 |

factory ISPの有無とcustom IAP sampleの有無を混同しない。たとえばM030に`UART_USB_IAP` sampleがあっても、blank chipにそのIAPが最初から入っていることにはならない。

### 5.2 probe-rsのtarget gapの意味

`probe-rs`のWCH-Link backend自体はDMIを運ぶ汎用層である。未対応系列の多くは「物理的にLinkEで接続できない」のではなく、次が未登録である。

- WCH-Link attach応答のfamily/chip ID
- exact SKUとmemory map
- option byteによるCODE/RAM split
- flash loader/algorithmとwork RAM
- erase page、write size、system flash/option領域

したがってV205/V407/X315/M030の第一候補は、別CLIを増やすより**probe-rs target追加**である。WCH OpenOCDとminichlinkの実装、各RM、手元の`ch32-device-data`を相互参照できる。

### 5.3 WCH OpenOCDを「全対応の正本」にしない理由

手元の`../tools/MRS_Toolchain_Linux_X64_V240`を軽く調べた結果:

- `wch-riscv.cfg`と`wch-dual-core.cfg`がある。
- binary内にV003/V00X/V103/V205/V20x/V30x/V317/V407/V467/X03x/X315/L103/M030/H417が列挙される。
- 電源、reset、erase、flash protection、Link indexなどのvendor command文字列がある。
- READMEはbuild scriptの場所を示しつつ、sourceについてMounRiver supportへの問い合わせを案内する。
- 単体起動は同梱`libjaylink.so.0`等のlibrary pathが必要で、バイナリを1個置くだけでは動かない。

coverage確認とfallbackには価値があるが、独自toolの長期基盤としては、テスト可能なtarget DBと再現buildを持つprobe-rs/minichlink側へ知識を移す方がよい。

## 6. LinkE代替の既存実装

### 6.1 CH32V003の1線SWIO

| project | probe MCU | host interface | flash | debug | 成熟度・注意 |
|---|---|---|---:|---:|---|
| [WCH公式CH32F103 1Line例](https://github.com/openwch/ch32v003/tree/main/CH32V003_1Line_Base_on_CH32F103) | CH32F103 | 独自sample | ○ | 基礎DMI | 一次資料。移植の基準 |
| [PicoRVD](https://github.com/aappleby/picorvd) | RP2040 PIO | USB CDC GDB remote | ○ | ○ breakpoint/step | very alphaだが層分離が明快 |
| [ESP32-S2 funprog](https://github.com/cnlohr/esp32s2-cookbook/tree/master/ch32v003programmer) | ESP32-S2 bitbang | vendor HID、minichlink | ○ | minichlink側 | 現行codeはSWIOとRVSWDの両方を持つ |
| [rvswdio_programmer](https://github.com/cnlohr/rv003usb/tree/master/rvswdio_programmer) | CH32V003 | low-speed USB HID、minichlink | ○ | basic GDB/printf | experimental/RFCだが1/2線両方を実装 |
| [NHC-Link042](https://github.com/NgoHungCuong/NHC-Link042) | STM32F042 | USB、minichlink backend | ○ V003 | △ | MIT。既存STM32F042 boardの再利用例 |
| [Ardulink/zooswio](https://github.com/zoobab/zooswio) | AVR Arduino等 | UART、minichlink `-C ardulink` | △ V003 | × | WIP・不安定表記だがUno/Nano等の旧機材をbootstrapに使える |
| [WCH_WebLink](https://github.com/Subjective-Reality-Labs/WCH_WebLink) | ESP32/ESP32-C3 | Wi-Fi WebSocket/Web UI | ○ V003 | SWIO terminal | 読出し等は未実装、V003だけを検証 |
| [ESP32-S3 CH32V003 programmer](https://github.com/Ishu1519/esp32s3-ch32-programmer) | ESP32-S3 | USB serial + Python | ○+verify | × | 2026-08-21開始で非常に新しい |
| [Flipper Zero flasher](https://github.com/sukvojte/wch_swio_flasher) | Flipper Zero | NHC-Link042 emulation/minichlink | ○ V003 | △ | V003だけを実機確認 |

PicoRVDは、PIO物理層、RISC-V Debug Module、V003 flash、software breakpoint、GDB serverを分離しており、RP2040実装の読みやすい参照になる。V003はhardware breakpointを持たないため、PicoRVDはflash patchによるsoftware breakpointも実装している。

### 6.2 2線RVSWDは自作可能か

**可能。推測だけではなく、動作を主張する実装が複数ある。**

1. [Swindle](https://github.com/mean00/swindle)はRP2040をstable probe hostとして掲げ、CH32V203/208/303/305/307をtarget sourceで識別する。RP2040 PIOの`rvswd.pio`がstart/stop、clock、turnaround、read/writeを生成し、Black Magic由来のtarget層がflash stubとGDB remoteを提供する。
2. `rv003usb/rvswdio_programmer`は`opmode=1`をSWIO、`opmode=2`をRVSWDとして自動判別し、READMEでV003/00x/20x/30x/X03x等の書き込み・semihost・basic GDBを掲げる。
3. ESP32-S2 funprogの現行sourceにもSWCLK pin、RVSWD read/write、family検出があり、README名より広い実装になっている。ただし非V003の検証範囲は明確でない。
4. RINSはRVSWDを「SWDではない」と明記し、第三者実装を目的に物理・論理層を文書化している。

つまり課題は波形生成の可否ではなく、**familyごとの初期化、Debug Module差、flash algorithm、option bytes、unbrick手順、実機回帰試験**である。

## 7. 独自probeの方式比較

### 7.1 WCH-Link USB protocol互換

利点:

- probe-rs、wlink、WCH OpenOCDを比較的小さい変更で利用できる。
- probe側がDMIを提供すれば、probe-rsのtarget/debug層を再利用できる。

問題:

- 公式VID `0x1a86`を独自製品で名乗れない。独自VID/PIDをhost toolへ追加する必要がある。
- protocolは非公式解析で、firmware版による差がある。
- LinkE variant/versionを偽装すると、将来hostが未実装commandを送る危険がある。

結論: **完全な偽装ではなく、「WCH-Link command subsetに似たopen protocol」を定義し、probe-rsへ明示的な新probe backendを追加する**のが安全。短期PoCだけ互換subsetを使う案はある。

### 7.2 minichlink funprog HID互換

利点:

- ESP32-S2とCH32V003 firmware、minichlink host側がすでにある。
- HID control transferなので一般OSでdriver不要にしやすい。
- low-level read/write、block write、power、terminal等のcommandがある。

問題:

- probe-rs/OpenOCDから直接使えない。
- minichlinkのtarget/debug層と強く結び付き、API/version管理が弱い。
- low-speed HID実装は帯域・latency上の上限がある。

結論: 互換モードまたは初期PoCには良い。長期の唯一のhost protocolにはしない。

### 7.3 probe上GDB server（PicoRVD/Swindle型）

利点:

- hostは標準GDBだけでよく、VS Code等にも接続しやすい。
- DMI往復の一部をprobe内で完結できる。

問題:

- ELF/Arduino uploader、machine-readableなprobe列挙、複数lane管理は別途必要。
- chip DBとflash algorithmがprobe firmwareへ入りやすい。
- Swindle/Black MagicをベースにするとGPL-3.0系になる。

結論: デバッグ製品としては有力。Arduino用の共通upload frontendとは別層として統合する。

### 7.4 新しいversion付きopen protocol

推奨要件:

- USB composite: vendor bulkまたはHID + CDC UART + DFU/UF2
- 固有serial、protocol major/minor、capability bitset
- request ID、timeout、明示的error code、再同期command
- DMI single/batch read/write、SWIO/RVSWD選択、自動probe
- NRST assert/deassert、target Vref測定、power switch（搭載時だけcapabilityを立てる）
- raw memory/flashをprobeに固定せず、まずDMI batchを高速化
- firmware updateはROM boot/UF2等で必ず復旧可能にする
- 自前VIDまたはpid.codes等の正当なVID/PIDを使う

## 8. 推奨ハードウェア

### 8.1 第一候補: RP2040

理由:

- PIOによって1線パルス幅符号化と2線clocked protocolを決定論的に実装できる。
- PicoRVD（1線）とSwindle（2線）の実装資産が同じMCU上にある。
- USB device、固有ID、UF2 recovery、安価な既製boardがある。
- host側にWi-Fi/Bluetooth等の余計な状態を持ち込まない。

弱点:

- native high-speed USBではない。
- 安価なPico互換boardはlevel shifter、Vref sense、保護、target power switchを持たない。
- PIO実装を統合しても、target認定は別に必要。

### 8.2 RP2350

将来候補。Swindleではexperimental。RAM/性能の余裕はあるが、MVPの既知実装量はRP2040が上。

### 8.3 ESP32-S2/S3

Wi-Fi/Web UI、remote fixture、既存funprog互換が必要なら有力。S2実装はSWIO/RVSWDコードを持つ。S3の新規実装もV003でflash/verifyを実証した。ただし、割込み・cache・RTOS・無線によるtiming jitterを物理層から隔離する設計が要る。RMT/SPI/GDMA等を使うか、critical sectionでのbitbangを測定する。

### 8.4 CH32V003自身

非常に安く、`rvswdio_programmer`の実績がある。しかしlow-speed software USB、RAM/flash余裕、復旧性、開発時に別programmerが必要という循環がある。量産用の超低価格probeやboard内蔵recoveryには適するが、最初の汎用開発probeにはRP2040が扱いやすい。

## 9. 独自probeとbootloaderのMVP範囲

### 9.1 MVPで必須

| 項目 | 合格条件 |
|---|---|
| host | Windows x64、Linux x64/ARM64、macOS x64/ARM64で同じCLI semantics |
| identity | 固有USB serialを列挙し、0台/複数台/指定不一致を明確に失敗させる |
| protocols | 1線SWIOと2線RVSWD。速度を少なくともsafe/normalの2段階で指定 |
| target | V003、V006または他V00X、V203、V307の4本を最低代表にする |
| input | ELF、Intel HEX、raw BIN+address |
| flash | 必要sector erase、write、readback verify、reset-and-run |
| recovery | connect-under-resetまたはNRST/power-cycle erase。失敗してもprobe自身はbrickしない |
| diagnostics | probe FW/protocol版、target ID、選択probe、速度、erase/write/verify範囲、構造化error |
| automation | JSON出力、安定exit code、非対話、timeout、Ctrl-C後のclean detach |
| safety | Vref検出、未給電targetへ信号を押し込まない、power capabilityの明示 |

### 9.2 MVPから外す

- Arm SWD/CMSIS-DAP/JTAG
- LinkW wireless互換
- 5 V target給電（回路保護と電源設計を別レビューする）
- 全CH5xx/BLE MCU
- SDI printとUARTの同時利用
- H417 dual-core
- option byteの任意editor、read protection設定
- IDE内の完全なsource-level debug

ただし、protocolとUSB descriptorには将来capabilityを追加できるversioningを最初から入れる。

### 9.3 Phase 2

- V103、X035、L103を追加
- V205/M030を追加し、既存OSSのcoverage gapを埋める
- CDC UART bridge
- SDI print（session lifetimeとtarget非停止動作を試験）
- probe-rs backendを完成し、DAP/GDBを利用
- 3.3 V target power switch、NRST、Vref senseを備えた専用PCB

### 9.4 Phase 3

- V317、V407/V467、X315
- H415/H416/H417 dual-core
- production fixture向けmulti-lane、remote service、firmware署名/rollback
- option bytes、protection、system flashを安全な高レベルAPIで提供

### 9.5 custom bootloader MVP

probeとは別artifact/repositoryにし、まず2系統だけを認定する。

| profile | target | MVP | 非目標 |
|---|---|---|---|
| `softusb-hid` | V003F4P6 | rv003usb互換host、BIN flash、verify、button/RAM magic entry、固有serial | CDC、全pin配置、bootloader自己更新 |
| `custom-dfu` | V203C8T6またはV307VCT6 | USB DFU、CRC付きheader、button/RAM magic/invalid-app entry、offset link | UF2、署名、A/B、全USB peripheral variant |
| `custom-uart`（次点） | V003/V006 | UART full duplex、CRC、retry、info、appからboot entry | multi-drop/RS-485はPhase 2 |

MVPでも、bootloaderを含まないboardへの誤ったoffset image、target違い、範囲外erase/writeは必ず拒否する。LinkE/SWIO/RVSWDによるbootloader導入・復旧手順を同時に提供する。

## 10. `ch32-upload` frontend案

独自probe完成を待たず、公式LinkEでも価値がある。

```text
ch32-upload list --format json
ch32-upload info --probe <serial> --format json
ch32-upload flash firmware.elf --probe <serial> --chip auto --verify --reset run
ch32-upload flash firmware.bin --transport factory-usb --device <serial>
ch32-upload flash firmware.bin --transport custom-uart --port <tty> --enter auto
ch32-upload flash firmware.dfu --transport custom-dfu --device <serial>
ch32-upload flash firmware.uf2 --transport custom-uf2 --volume <identity>
ch32-upload erase --probe <serial> --scope code --connect-under-reset
ch32-upload doctor --probe <serial>
```

### 必須の設計原則

- `--probe`なしで複数見つかったら書かない。
- `--chip`指定と実chip IDが不一致なら書かない。
- `flash`成功はwrite完了ではなく、要求時はverifyとreset/runまで含めて定義する。
- 人向けstderrとmachine-readable stdoutを分離する。
- backend名はdiagnosticには出すがArduinoメニューには出さない。
- target DBを`ch32-device-data`等から生成し、toolごとの手書き表を増やさない。
- probe firmware更新を暗黙に行わない。認定済み版とrollback手順を記録する。
- transport選択前にbuild artifactのorigin/headerと、実boardのbootloader identity/versionを照合する。
- upload中にserial port、DFU device、通常applicationのVID/PIDが切り替わっても、USB serialまたは物理port pathで同一boardを追跡する。

### backend優先順

1. debug経路: probe-rs library。target未対応系列だけWCH OpenOCDまたはwlink/minichlinkへfallback
2. factory ISP経路: wchisp。未対応command/SKUだけ公式WCHISPTool_CMDを任意fallback
3. custom bootloader経路: 内蔵native client、またはdfu-util/minichlink/tinyboot CLI adapter
4. 独自probe経路: probe-rsの明示的な新backend

backendのprocess wrapperから始める場合も、終了コード、progress、error taxonomyをfrontendの公開仕様とする。将来in-process libraryへ替えてもArduino recipeを変えない。

## 11. 実機認定matrix

family名だけで「対応」としない。最低でも次のキーを記録する。

```text
host OS/arch
host tool + commit/version
probe model / hardware revision / firmware / USB serial
probe protocol version
target exact SKU / package / silicon revision
debug interface / speed / wiring length / target voltage
flash layout option bytes
upload transport / bootloader kind + version / app origin / reserved ranges
operation: identify, erase, flash, verify, reset-run, halt/resume, memory, recovery
artifact hash / elapsed time / retry count / raw log
```

代表targetの優先順:

1. CH32V003F4P6: 1線、64-byte page、software breakpoint、unbrick
2. CH32V006F8P6: 新V2C 1線、256-byte page
3. CH32V203C8T6: 主流2線、小容量V20x
4. CH32V307VCT6: 2線、大容量、CODE/RAM split
5. CH32X035C8T6: 2線、既存Arduino実機資産
6. CH32L103: low-power/reset差
7. CH32M030: ISPなし・既存tool gap
8. CH32H417: dual-core

各targetで、通常書き込みだけでなく次も試す。

- user firmwareがdebug pinをGPIO化した状態からの回復
- read protection有効/解除
- blank chipと不正option byte
- USB切断、target電源断、verify途中失敗
- 同型probe 2台同時接続とserial指定
- 通常app→bootloader→書込→新appというport再enumerationと自動追跡
- bootloader待機timeout、button、double reset、RAM magic、invalid-image各entry
- 転送、erase、write、first bootの各時点で電源断し、bootloaderへ復帰できること
- bootloaderなし/版違い/target違いへoffset imageを書かないこと
- 100回以上の連続flash、速度ごとのerror率
- 書き込み対象外sectorが保持されること

## 12. 直近の具体的作業

### P0: 既存toolを完成させる

- [ ] probe-rsへV205、M030、V407/V467、X315を追加する難易度を系列ごとに見積もる
- [ ] probe-rs WCH-Link backendへprobe model/FW表示、SDI print、power/NRST eraseを追加する設計を作る
- [ ] wlinkへUSB serial selectorとJSON出力を追加する小patchを検討する
- [ ] WCH OpenOCD 2.40で4つのgap系列のchip ID/flash情報を採取し、公開資料と照合する
- [ ] `ch32-upload` CLI contractとerror codeを仕様化する
- [ ] board/SKU DBにfactory ISPのUSB/UART有無、entry条件、VID/PID、bootloader版を追加する
- [ ] upload profileからlinker origin、最大size、image形式を生成するschemaを決める

### P1: 独自probe feasibility

- [ ] logic analyzerでLinkEのV003 SWIOとV203 RVSWDを採取する
- [ ] RP2040でPicoRVD SWIO PIOとSwindle RVSWD PIOを別々にbuild・実機確認する
- [ ] DMI read/write batchのopen USB protocolを小さく定義する
- [ ] V003とV203でchip ID、RAM read/write、halt/resumeを通す
- [ ] probe-rsに実験backendを追加し、既存target YAMLのflash algorithmで書けるか確認する

### P2a: 専用hardware

- [ ] Vref sense、level translation、NRST、3.3 V power switch、過電流保護、UARTを回路仕様化する
- [ ] UF2 recoveryと固有serialを含むfirmware更新仕様を作る
- [ ] 既製RP2040 board用配線adapterと専用PCBを分ける

### P2b: custom bootloader feasibility

- [ ] V003でrv003usb HID bootloaderを導入し、Linux/Windows/macOSからArduino uploadを通す
- [ ] V003/V006でWCH factory UART ISPとtinybootを比較し、entry、速度、復旧性、flash占有を測る
- [ ] V203またはV307でfactory USB ISP、UF2、DFUを同一boardで比較する
- [ ] WCH公式UART/USB IAP sampleのprotocolとlinker差をfamily別共通HALへ切り分ける
- [ ] `ch32-upload`でUSB再enumeration、TTY、MSC mountを同じdevice identityへ関連付ける
- [ ] 途中電源断と破損imageを含むboot state machine試験を自動化する

## 13. 採用判断

現時点では次の判断が妥当である。

- ArduinoCoreの最初の安定releaseを独自probe完成に依存させない。
- ただし独自probeを「P2の遠い実験」に留める必要もない。2線RVSWDの実装可能性はすでに確認できるため、RP2040 PoCを並行してよい。
- 先にprobe-rs target gapを埋めれば、公式LinkEと将来の独自probeの両方が同じtarget DB・flash algorithmを使える。
- LinkEの不満が主にUX、複数台、machine-readable出力、firmware差の可視化なら、hardwareより`ch32-upload`とprobe-rs改善が先に効く。
- 不満が電気的安定性、recoverability、fixture向け電源/NRST、open firmwareにあるなら、RP2040専用probeの価値がある。
- 利用者が普段LinkEを挿さずに更新できることが目的ならcustom bootloaderが効く。ただし初回導入・mass erase・system領域破損の救出用debug padはboardに残す。
- Arduino側はupload tool名を直接boardへ焼き込まず、`board × transport × bootloader layout` profileからlinker、最大sketch容量、image形式、entry方法を一体生成する。

## 14. 主な資料とローカル調査対象

### 公式・仕様

- [WCH-Link manual V2.4](https://www.wch.cn/uploads/file/20250124/1737704462135866.pdf)
- [WCH-MCU-DL offline programmer manual](https://www.wch.cn/uploads/file/20240821/1724227120114035.pdf)
- [QingKe V4 Processor Manual](https://www.wch-ic.com/downloads/QingKeV4_Processor_Manual_PDF.html)
- [WCH QingKe processor overview](https://www.wch-ic.com/products/QingKe.html)
- [CH32V003 1-line example](https://github.com/openwch/ch32v003/tree/main/CH32V003_1Line_Base_on_CH32F103)
- [WCHISPTool_CMD manual](https://device.report/m/dad92b59d9fb01614cedc4cadf480331454e10414fac45336f85a7ce6a71a202_optim.pdf)
- [RINS WCH custom DTM documentation](https://perigoso.github.io/rins/)

### tool/probe source

- [probe-rs](https://github.com/probe-rs/probe-rs)
- [wlink](https://github.com/ch32-rs/wlink)
- [wchisp](https://github.com/ch32-rs/wchisp)
- [ch32fun/minichlink](https://github.com/cnlohr/ch32fun/tree/master/minichlink)
- [PicoRVD](https://github.com/aappleby/picorvd)
- [Swindle](https://github.com/mean00/swindle)
- [ESP32-S2 funprog](https://github.com/cnlohr/esp32s2-cookbook/tree/master/ch32v003programmer)
- [rv003usb rvswdio programmer](https://github.com/cnlohr/rv003usb/tree/master/rvswdio_programmer)
- [NHC-Link042](https://github.com/NgoHungCuong/NHC-Link042)
- [Ardulink/zooswio](https://github.com/zoobab/zooswio)
- [wlink-iap](https://github.com/cjacker/wlink-iap)

### bootloader/IAP source

- [CH32V003 factory bootloader documentation](https://github.com/basilhussain/ch32v003-bootloader-docs)
- [rv003usb software USB bootloader](https://github.com/cnlohr/rv003usb/tree/master/bootloader)
- [tinyboot UART/RS-485 bootloader](https://github.com/OpenServoCore/tinyboot)
- [wch-uf2](https://github.com/ArcaneNibble/wch-uf2)
- [Swindle CH32V3x DFU bootloader](https://github.com/mean00/swindle_bootloader_ch32v3x)
- [PlumBL](https://github.com/HaiMianBBao/PlumBL)
- [WCH CH32V003 IAP introduction](https://raw.githubusercontent.com/openwch/ch32v003/main/CH32V003_IAP_Use_Introduction.pdf)

### 調査したローカル版

- `../probe-rs`: `40388f2b` (2026-08-24)、0.32.0 tagとも比較
- `../ch32fun`: `6c4dd53` (2026-08-24)
- `../ArduinoCore-CH32/docs/research/upload-programmers.ja.md`: 既存R-17調査
- `../ArduinoCore-CH32/docs/upload-and-fixture.ja.md`: LinkE実測、複数probe、firmware差
- `../tools/MRS_Toolchain_Linux_X64_V240`: WCH OpenOCD binary/configを軽く確認
- `../CH32V003`、`../CH32V006`、`../CH32V103`、`../CH32V307`、`../CH32V407`、`../CH32X035`、`../CH32X315`、`../CH32M030`: USART/USB/Ethernet/USB-host IAP例を確認
- 一時clone: wlink `249f2c1`、wchisp `cefd870`、Swindle `3435ffe`、rv003usb `80b1893`、PicoRVD `4310ade`、ESP32-S2 cookbook `ca21357`
- bootloader一時clone: tinyboot `40df7449`、wch-uf2 `81b5955a`、Swindle CH32V3x DFU `be67c072`

外部projectのmain/nightlyとWCH firmwareは変化する。仕様書化するときは、採用commit、配布artifact、checksum、license、実機結果を固定する。
