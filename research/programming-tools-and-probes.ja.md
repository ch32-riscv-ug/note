# CH32 RISC-V書き込み・デバッグ・bootloader経路と独自プローブの現状調査レポート

- 調査基準日: 2026-08-26
- 目的: Arduino環境から利用できる書き込み・デバッグ・bootloader経路と、その実装・対応範囲・制約の現状を整理する
- 状態: 2026-08-26時点の資料・source調査。表の対応は、特記しない限り全組合せの実機認定を意味しない

## 1. 機能層と更新経路

各プロジェクトは同じ「書き込みツール」に見えても、担当する層が違う。

| 層 | 役割 | 既存例 | 主な実装箇所 |
|---|---|---|---|
| Arduino frontend | `platform.txt`、board選択、エラー表示、probe選択 | Arduino recipe、PlatformIO | `platform.txt`、`boards.txt`、discovery tool |
| image/package | link offset、最大sketch容量、header、CRC/署名、UF2変換 | linker script、`uf2conv.py`、DFU suffix | build recipe、linker、変換tool |
| ホストdebug/flash | ELF解析、target DB、flash algorithm、実行制御、GDB/DAP | probe-rs、OpenOCD、minichlink | host library/CLI |
| USB probe transport | 列挙、serial、command framing、batch転送 | WCH-Link bulk、funprog HID、Black Magic GDB serial | probe firmwareとhost backend |
| debug物理層 | 1線SWIOまたは2線RVSWDでDMIを転送 | LinkE、PicoRVD PIO、Swindle PIO | probe MCUのPIO/RMT/timer |
| target内boot/update | reset直後の判定、image受信、erase/write/verify、appへjump | factory ISP、rv003usb、tinyboot、DFU/UF2 | system flashまたはuser flashの予約領域 |
| target固有処理 | chip ID、memory map、option bytes、erase/write stub | probe-rs YAML、minichlink `chips.c`、WCH OpenOCD | host DBまたはprobe/bootloader firmware |

target固有情報の配置は実装ごとに異なる。probe-rsはchip ID表とflash algorithmをhost側に置き、PicoRVD/Swindle型はtarget処理の一部をprobe firmware側に置く。bootloader経路ではflash処理がtarget内にあり、tinybootはtransport、boot状態遷移、family別HALを分離している。

### 1.1 Arduinoから見た全書き込み経路

各経路は、Arduino frontendからtargetへ到達する別々の経路である。

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

経路ごとに用途が異なる。SWIO/RVSWDはblank chip、復旧、source debugを扱える。factory ISPとcustom bootloaderは主にflash更新を扱い、application IAPや外部媒体IAPは稼働中または現場での更新経路になる。WCH-MCU-DLはPCから切り離した量産書き込みを対象とする。

## 2. WCH側インターフェースの整理

### 2.1 debug interface

| 系列 | 主なcore | debug配線 | 備考 |
|---|---|---|---|
| CH32V003、CH641 | QingKe V2A | 1線SWIO/SDI | データのパルス幅で0/1を表す。外部pull-upが必要な自作例が多い |
| CH32V00X、CH32M007 | QingKe V2C | 1線SWIO/SDI | V003とはflash page・memory構成が異なる |
| CH32V103、CH56x/57x | QingKe V3系 | 2線RVSWD | 配線名はSWDIO/SWCLKだがArm SWDではない |
| CH32V20x/V30x/V317、X03x、L103、CH643等 | QingKe V4系 | 2線RVSWD | 既存OSS実装が最も多い2線系列 |
| CH32M030 | QingKe V2C系 | 1線/2線の両対応と公式資料に記載 | 工場ISPはなく、外部debugまたは導入済みcustom IAPを使用 |
| CH32H41x | dual core | 2線RVSWD | core選択、個別work area、dual-core resetを別途扱う |
| CH32V4x7、CH32X315 | 新しい2線系列 | 2線RVSWD | OSS target DBが追いついていない |

WCH自身もQingKeを「1-wire/2-wire DTM」と説明している。2線RVSWDの公開一次仕様は十分ではないが、[RINS](https://perigoso.github.io/rins/)は物理・論理層を第三者実装向けに整理しており、[WCH RVSWD protocolの初期解析](https://github-wiki-see.page/m/fxsheep/openocd_wchlink-rv/wiki/WCH-RVSWD-protocol)とも整合する。

### 2.2 WCH-Link USB host protocol

WCH-LinkのRISC-V modeはUSB bulk (`1a86:8010`)で、概ね次を扱う。

- probe型・firmware版取得
- target attachとchip family/ID取得
- debug速度設定
- DMI read/write/nop
- reset/detach
- flash protection、電源制御、特殊erase、SDI print等のvendor command

パケットは`0x81, command, length, payload...`、成功応答は`0x82,...`である。[wlinkのprotocol.md](https://github.com/ch32-rs/wlink/blob/main/protocol.md)と[RINSのWCH-Link資料](https://perigoso.github.io/rins/wch-link/index.html)に解析結果がある。

`probe-rs`のWCH-Link backendは、attach、速度設定、DMI、reset、flash protection解除という比較的小さなsubsetを使い、フラッシュ書き込み自体はtarget側flash algorithmをDMI経由で動かす。WCH-Link固有処理とtarget/debug処理は別moduleになっている。

### 2.3 factory ISPとcustom IAPの区別

WCHの資料ではISP/IAP/BOOTという語が混在する。本レポートでは次のように区別する。

| 種類 | 格納場所 | 誰が導入するか | 主目的 | 消去事故への強さ |
|---|---|---|---|---|
| factory ISP | system/BOOT領域 | 工場出荷時 | USB/UARTからcode flashを書込 | code flash消去後も残るが、option/entry条件に依存 |
| custom system bootloader | 書換可能なsystem/BOOT領域 | 最初にdebug probe等で導入 | factory ISP置換、user flashを全量確保 | system領域を書き損じるとprobe救出が必要 |
| custom user-flash bootloader | code flash先頭等 | 最初にdebug probe/factory ISPで導入 | DFU/UF2/UART/OTA | mass eraseで消える。appのlink offsetが必要 |
| application IAP | 通常application内 | appと同時 | 稼働中に受信して自己更新 | app破損時には入れない。小さいrecovery stub併用が望ましい |

CH32V003のfactory bootloaderは`0x1FFFF000`からの1,920 byte system領域にあり、UART 115200 bpsで動く。外部BOOT pinだけではなくapplicationが`START_MODE`を設定してresetする必要があるため、appが完全に壊れた場合の入口としては弱い。factory bootloaderの挙動とprotocolは[CH32V003 factory bootloader missing manual](https://github.com/basilhussain/ch32v003-bootloader-docs)に詳しい。V00Xではsystem領域の大きさや一部packageのUART remapが異なる。

USB内蔵系列のfactory ISPは`wchisp`/WCHISPToolから使えるが、「USB peripheralがある」と「そのSKUのfactory bootloaderがUSB経路を公開する」は同義ではない。正確な対応は**exact SKU、BOOT pin/option byte、bootloader version**で管理する。

### 2.4 WCH/OSSのcustom bootloader・IAP事例

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
| [PlumBL](https://github.com/HaiMianBBao/PlumBL) | CH32V30xほか、CherryUSB DFU/U2F | user flash予約 | `dfu-util`/U2F tool | multi-platform port例 |

これらの実装は、USB peripheralを持たないV003でのsoftware USB更新、USB内蔵品でのDFU/UF2、UART IAPの複数family展開、RS-485/1線UARTによる更新をそれぞれ実証または実装対象としている。

### 2.5 custom bootloader transport比較

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

Arduino IDEから扱う場合、software USB HID、DFU、UARTはいずれも対応するhost commandが必要になる。UF2は利用者からはfile copyに見えるが、Uploadボタンとの統合にはmount pointの検出、copy、完了監視が必要になる。

### 2.6 bootloader実装に共通して現れる仕様項目

- image headerにmagic、format version、target、load address、長さ、version、hashを持つ実装がある。
- CRCは転送・保存データの破損検出に使われ、署名はimageの作成元確認に使われる。両者の役割は異なる。
- 更新失敗への方式として、bootloader常駐、A/B slot、download slotからのcopy、trial boot、watchdog、applicationの`confirm()`がある。
- boot entryにはbutton、double reset、RAM magic、application command、無効image検出、BOOT pin/option byteが使われている。
- user-flash bootloaderでは、bootloader領域、metadata page、application offsetの分だけapplication容量が減る。
- appへのjump時にはvector table/interrupt、clock、USB pull-up、peripheral、stack、`.data/.bss`の状態が関係する。
- system/BOOT領域を書き換える実装と、code flash内だけを更新する実装がある。前者の破損時は外部debug probeが必要になる。
- USB経路ではVID/PID、serial、DFU alt setting、UF2 family ID、UART経路ではportとnode IDが識別情報になる。

### 2.7 Arduino統合時に経路ごと変化する項目

| 項目 | debug probe | factory ISP | custom bootloader |
|---|---|---|---|
| application origin | 通常はcode flash先頭 | 通常はcode flash先頭 | user-flash BLではoffsetあり |
| 最大sketch容量 | chip/option byte依存 | chip/option byte依存 | bootloader・metadata予約分を除く |
| 入力image | ELF/BIN/HEX | BIN/HEX等 | BIN、DFU、UF2、独自header付きimage等 |
| device discovery | probe VID/PID/serial | USB ISP VID/PID、TTY | HID/DFU VID/PID、TTY、MSC volume、network ID |
| entry操作 | attach、NRST、power cycle | BOOT pin、option、applicationからreset | button、double reset、RAM magic、1200-bps touch等 |
| upload後の変化 | target reset | ISPからapplicationへUSB/TTYが変化 | bootloaderとapplicationでVID/PID/portが変わる場合あり |

Arduino platformでは、これらは`platform.txt`のtool recipe、`boards.txt`の`upload.maximum_size`、linker script、`upload.use_1200bps_touch`、Pluggable Discovery等に対応する。現在の`../ArduinoCore-CH32`はprobe-rsのprogrammer経路を持つが、serial/factory ISP/custom bootloader経路はまだ統合されていない。

## 3. ホストツール比較

記号: ○=実装あり、△=制限・別操作・要確認、×=目的外、?=資料だけでは確定できない。

### 3.1 対応host・配布・開発基盤

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

### 3.2 機能マトリクス

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

## 4. MCU系列別の現状

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

### 4.1 family別のupload route

`factory`欄はfamily全体の保証ではない。USB pinがbond-outされないpackage、BOOT entry pin/option、出荷bootloader版で変わるので、最終DBはSKU/package単位にする。

| family | debug probe | factory ISP | 公開custom boot/IAPの根拠 |
|---|---|---|---|
| V003 | SWIO | UART、appからSW entry | rv003usb software USB、tinyboot UART/RS-485、WCH USART_IAP |
| V00X | SWIO | UART、appからSW entry、packageでpin差 | tinyboot、rv003usb V006、WCH USART_IAP |
| V103 | RVSWD | USB + USART1、BOOT0/1 | tinyboot、WCH UART_USB_IAP/HOST_IAP |
| V203/V208 | RVSWD | USB + USART1、BOOT0/1 | wch-uf2、WCH系USB/UART IAP |
| V205 | RVSWD | USB + USART2、BOOT0/1、小packageはISP不可あり | 今回の調査ではfamily固有の公開例を未確認 |
| V30x/V317 | RVSWD | USB + USART1、BOOT0/1、小package注意 | Swindle DFU、WCH USB/UART/USB-host/Ethernet IAP |
| V407/V467 | RVSWD | USB + USART1、BOOT0/1 | 手元WCH USB_UART IAP、USB-host IAP |
| X033/X035 | RVSWD | USB + UART、appからSW entry可 | 手元WCH USB_UART/HOST_IAP |
| X315 | RVSWD | USBFS + UART keyless、SW entry可 | 手元WCH USB_UART/HOST_IAP |
| L103 | RVSWD | USB + USART2、BOOT0/1 | 今回の調査ではfamily固有の公開例を未確認 |
| M030 | SWIO/RVSWD | factory ISPなし | 手元WCH UART_USB_IAP/HOST_IAP |
| H41x | RVSWD、dual | USBFS + UART、SW entry可、optionで経路制御 | 今回の調査ではcustom bootloader公開例を未確認 |

factory ISPの有無とcustom IAP sampleの有無を混同しない。たとえばM030に`UART_USB_IAP` sampleがあっても、blank chipにそのIAPが最初から入っていることにはならない。

### 4.2 probe-rsのtarget gap

`probe-rs`のWCH-Link backend自体はDMIを運ぶ汎用層である。未対応系列の多くは「物理的にLinkEで接続できない」のではなく、次が未登録である。

- WCH-Link attach応答のfamily/chip ID
- exact SKUとmemory map
- option byteによるCODE/RAM split
- flash loader/algorithmとwork RAM
- erase page、write size、system flash/option領域

V205/V407/X315/M030をprobe-rsで扱うには、これらのtarget情報を追加する必要がある。情報源としてWCH OpenOCD、minichlink、各reference manual、手元の`ch32-device-data`が存在する。

### 4.3 ローカルWCH OpenOCDの収録範囲と配布状態

手元の`../tools/MRS_Toolchain_Linux_X64_V240`を軽く調べた結果:

- `wch-riscv.cfg`と`wch-dual-core.cfg`がある。
- binary内にV003/V00X/V103/V205/V20x/V30x/V317/V407/V467/X03x/X315/L103/M030/H417が列挙される。
- 電源、reset、erase、flash protection、Link indexなどのvendor command文字列がある。
- READMEはbuild scriptの場所を示しつつ、sourceについてMounRiver supportへの問い合わせを案内する。
- 単体起動は同梱`libjaylink.so.0`等のlibrary pathが必要で、バイナリを1個置くだけでは動かない。

このbinaryはOSS toolで未定義のfamilyを含む一方、対応sourceと再現buildの入手性、runtime library依存がprobe-rs/minichlinkと異なる。

## 5. LinkE代替の既存実装

### 5.1 CH32V003の1線SWIO

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

### 5.2 2線RVSWDの第三者実装

2線RVSWDには、動作を主張する第三者実装が複数存在する。

1. [Swindle](https://github.com/mean00/swindle)はRP2040をstable probe hostとして掲げ、CH32V203/208/303/305/307をtarget sourceで識別する。RP2040 PIOの`rvswd.pio`がstart/stop、clock、turnaround、read/writeを生成し、Black Magic由来のtarget層がflash stubとGDB remoteを提供する。
2. `rv003usb/rvswdio_programmer`は`opmode=1`をSWIO、`opmode=2`をRVSWDとして自動判別し、READMEでV003/00x/20x/30x/X03x等の書き込み・semihost・basic GDBを掲げる。
3. ESP32-S2 funprogの現行sourceにもSWCLK pin、RVSWD read/write、family検出があり、README名より広い実装になっている。ただし非V003の検証範囲は明確でない。
4. RINSはRVSWDを「SWDではない」と明記し、第三者実装を目的に物理・論理層を文書化している。

各実装の対応範囲は、波形生成だけでなく、familyごとの初期化、Debug Module差、flash algorithm、option bytes、unbrick手順、実機確認状況によって異なる。

## 6. 独自probeのhost protocol方式比較

### 6.1 WCH-Link USB protocol互換

利点:

- probe-rs、wlink、WCH OpenOCDを比較的小さい変更で利用できる。
- probe側がDMIを提供すれば、probe-rsのtarget/debug層を再利用できる。

問題:

- 公式VID `0x1a86`を独自製品で名乗れない。独自VID/PIDをhost toolへ追加する必要がある。
- protocolは非公式解析で、firmware版による差がある。
- LinkE variant/versionを偽装すると、将来hostが未実装commandを送る危険がある。

### 6.2 minichlink funprog HID互換

利点:

- ESP32-S2とCH32V003 firmware、minichlink host側がすでにある。
- HID control transferなので一般OSでdriver不要にしやすい。
- low-level read/write、block write、power、terminal等のcommandがある。

問題:

- probe-rs/OpenOCDから直接使えない。
- minichlinkのtarget/debug層と強く結び付き、API/version管理が弱い。
- low-speed HID実装は帯域・latency上の上限がある。

### 6.3 probe上GDB server（PicoRVD/Swindle型）

利点:

- hostは標準GDBだけでよく、VS Code等にも接続しやすい。
- DMI往復の一部をprobe内で完結できる。

問題:

- ELF/Arduino uploader、machine-readableなprobe列挙、複数lane管理は別途必要。
- chip DBとflash algorithmがprobe firmwareへ入りやすい。
- Swindle/Black MagicをベースにするとGPL-3.0系になる。

## 7. 独自probeに使われているMCU

### 7.1 RP2040

実装例と特性:

- PIOによって1線パルス幅符号化と2線clocked protocolを決定論的に実装できる。
- PicoRVD（1線）とSwindle（2線）の実装資産が同じMCU上にある。
- USB device、固有ID、UF2 recovery、安価な既製boardがある。
- host側にWi-Fi/Bluetooth等の余計な状態を持ち込まない。

弱点:

- native high-speed USBではない。
- 安価なPico互換boardはlevel shifter、Vref sense、保護、target power switchを持たない。
- PIO実装を統合しても、target認定は別に必要。

### 7.2 RP2350

Swindleではexperimentalとされている。RP2040よりRAM/性能に余裕がある一方、確認できた実装例はRP2040より少ない。

### 7.3 ESP32-S2/S3

S2実装はSWIO/RVSWDコードを持ち、Wi-Fi/Web UIやfunprog HIDを利用する実装がある。S3の実装例はV003でflash/verifyを掲げている。ESP32-S2 funprogはtiming-sensitiveなGPIO操作とcritical sectionを含む。

### 7.4 CH32V003自身

`rvswdio_programmer`はCH32V003をprobe MCUとして使う。USBはlow-speed software実装で、RAM/flash容量が小さく、probe firmwareの初回書き込みには別のprogrammerが必要になる。

## 8. 主な資料とローカル調査対象

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

外部projectのmain/nightlyとWCH firmwareは変化する。本調査で確認したlocal commitは上記のとおりで、release版とmain/nightlyの対応範囲が異なる場合がある。
