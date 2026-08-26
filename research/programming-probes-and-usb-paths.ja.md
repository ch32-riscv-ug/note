# CH32用書き込み装置・デバッグprobeとhost接続経路の調査

調査日: 2026-08-26

## 1. 調査範囲

本稿は、CH32 RISC-V MCUのdebug interfaceへ接続する書き込み装置を中心に、次を装置単位で比較する。

- target側が1線SWIO、2線RVSWD、または両方か
- flash書き込みだけか、halt/step/breakpoint/GDBまで扱うか
- WCH公式SDI printf、minichlink debug-wire terminal、RTT、UARTのどれを扱うか
- host側の書き込みapplicationと、PCからtargetまでのdata path
- Windowsでvendor driver、WinUSB binding、HID、CDC、Mass Storageのどれを使うか

host application自体の機能・OS・licenseは[WCH-LinkE host application調査](wch-linke-host-apps.ja.md)、bootloaderやfactory ISPを含む全経路は[全体調査](programming-tools-and-probes.ja.md)に分離している。

記号は、○=実装あり、△=限定・別方式・検証範囲不明、×=非対応、?=公開資料から確定できない、である。

## 2. 用語を分ける

### 2.1 target側debug wire

| 名称 | 信号 | 主なtarget | 備考 |
|---|---|---|---|
| 1線SWIO | data 1本、pull-up | CH32V003、CH641等 | pulse幅でbitを表す。ARM SWDとは異なる |
| 2線RVSWD | data + clock | CH32V1/2/3、X03x等 | WCH固有RISC-V debug transport。ARM SWDとpin名が似てもprotocolは異なる |
| 1/2線切替可能target | 1線またはdata + clock | V00X/M007、M030、CH564、CH584/585、CH570/572等 | option/configやfamily条件によりinterfaceが変わる |
| ARM SWD | SWDIO + SWCLK | CH32F、一般Cortex-M | CMSIS-DAP/OpenOCDで広く使う標準系。WCH RVSWDとは互換でない |

従って「CMSIS-DAP対応」「ARM SWD対応」だけでは、CH32 RISC-Vの2線RVSWD対応を意味しない。

### 2.2 print経路

| 名称 | targetからhostへの経路 | 必要なtarget code | 公式SDI互換か |
|---|---|---|---:|
| WCH SDI virtual serial | SWIO/RVSWD → LinkE firmware → USB COM | WCH EVTのSDI printf系 | ○ |
| minichlink terminal/semihost | debug register/memory handshake → probe protocol → minichlink `-T` | ch32fun系debug printf/semihost | ×。同じwireを使う別protocol |
| SEGGER RTT | target RAM ring buffer → debug memory read → probe | RTT control block/library | × |
| physical UART | target UART TX/RX → probe UART bridge → USB CDC | UART driver | ×。debug wireを使わない |
| probe log console | probe firmwareのlog → USB CDC/UART | target側不要 | ×。target printではない |

「SDI print対応」という表現は狭義にはWCH公式方式だけを指す。本稿では似た用途のterminalを別列にする。

## 3. 1線・2線・debug・printの装置比較

### 3.1 公式・市販系

| 装置 | probe MCU/形態 | 1線SWIO | 2線RVSWD | flash | 本格debug | 公式SDI | 物理UART | 状態 |
|---|---|---:|---:|---:|---:|---:|---:|---|
| **WCH-LinkE** | CH32V305系公式probe | ○ | ○ | ○ | ○ MRS/OpenOCD/probe-rs | **○** | ○ | 現行の基準装置 |
| WCH-LinkW | CH32V208系、wireless対応 | ○ | ○ | ○ | ○ | × manualではLinkE限定 | ○ | wireless/USB。host tool側の対応差あり |
| 旧WCH-Link | CH549系 | × V003/V00X不可 | ○ 対応系列 | ○ | ○ | × | ○、最大baud低い | 生産終了、公式保守のみ |
| WCH-DAPLink | WCH公式CMSIS-DAP系 | × CH32 RISC-V用ではない | × CH32 RVSWD用ではない | ARM SWD/JTAG用 | ○ ARM target | × | ○ | HID/WinUSB modeを持つがLinkE代替ではない |
| WCH-MCU-DL | offline/batch writer | mode/target依存 | ○ SWD mode | ○ | × | × | UART programming mode | PCで設定後standalone動作可能 |

[WCH-Link manual V2.4](https://www.wch.cn/uploads/file/20250124/1737704462135866.pdf)は、WCH-LinkEについてV003等の1線targetとV10x/V20x/V30x等の2線target、download/debug、SDI virtual serial、3.3/5 V、UART bridgeを記載している。公式SDI機能は同manualでWCH-LinkE限定とされる。

### 3.2 OSS・自作probe

| 装置/project | probe MCU | 1線 | 2線 | flash | GDB/debug | target print | 対象・成熟度 |
|---|---|---:|---:|---:|---:|---|---|
| [rvswdio_programmer](https://github.com/cnlohr/rv003usb/tree/master/rvswdio_programmer) | CH32V003 | **○** | **○** | ○ | ○ basic GDB | ○ minichlink semihost | V003/V00X/V20x/V30x/X03x/CH57x/585/59x等。experimental/RFC |
| [ESP32-S2 funprog](https://github.com/cnlohr/esp32s2-cookbook/tree/master/ch32v003programmer) | ESP32-S2 | **○** | **○ source** | ○ | ○ minichlink側 | ○ minichlink terminal | READMEの実機案内はV003中心。現行sourceにRVSWD/family detectionあり |
| [Swindle](https://github.com/mean00/swindle) | RP2040等 | × | ○ | ○ GDB `load` | ○ break/step/GDB | ○ RTT、UART。非SDI | V20x/V30x系。RP2040 stable、RP2350 experimental |
| [PicoRVD](https://github.com/aappleby/picorvd) | RP2040 PIO | ○ | × | ○ GDB `load` | ○ break/step/GDB | × target print。probe consoleは別 | V003専用、very alpha |
| [NHC-Link042](https://github.com/NgoHungCuong/NHC-Link042) | STM32F042 | ○ | × | ○ | △ minichlink generic GDB | △ minichlink terminal | V003、MIT。USB protocolはvendor bulk |
| [Flipper Zero flasher](https://github.com/sukvojte/wch_swio_flasher) | Flipper Zero | ○ | × | ○ | △ NHC emulation/minichlink | △ | V003で確認 |
| [Ardulink/zooswio](https://github.com/zoobab/zooswio) | AVR Arduino等 | ○ | × | △ | × | × | UART経由、WIP/不安定表記 |
| [WCH WebLink](https://github.com/Subjective-Reality-Labs/WCH_WebLink) | ESP32/ESP32-C3 | ○ | × | ○ | × source debugger | ○ SWIO terminalまたはUART | V003のみ。browser/WebSocket/HTTP |
| [ESP32-S3 programmer](https://github.com/Ishu1519/esp32s3-ch32-programmer) | ESP32-S3 | ○ | × | ○ verify | △ DMI操作、GDBなし | × | V003実機baseline、USB-UART + Python |
| [WCH公式1Line例](https://github.com/openwch/ch32v003/tree/main/CH32V003_1Line_Base_on_CH32F103) | CH32F103 | ○ | × | 基礎実装 | DMI primitive | × | 一次資料だが完成host productではない |

### 3.3 「1/2線、debug、SDI」の該当関係

- **公式SDIまで含めて全部該当する装置はWCH-LinkE**である。書き込みはWCH-LinkUtility/MRS/probe-rs/wlink/minichlink等、source debugはMRS + WCH OpenOCDまたはprobe-rs、SDI enable/watchはWCH-LinkUtilityまたはwlinkが扱う。
- `rvswdio_programmer`は1線と2線を自動判定し、flash、basic GDB、semihostを同じ小型probeで扱う。ただしprintはWCH公式SDI COM互換ではない。
- ESP32-S2 funprogの現行sourceにも1線/2線と`PollTerminal`がある。READMEはV003 programmerとして説明しており、2線各familyの実機認定表はない。
- Swindleは2線debugとRTT/UARTを持つが、1線と公式SDIは持たない。

## 4. 装置ごとのhost applicationと書き込み経路

| probe | host application | PC→probe | probe→target | flash処理の主な所在 |
|---|---|---|---|---|
| WCH-LinkE/LinkW/旧Link | WCH-LinkUtility、MRS/WCH OpenOCD、probe-rs、wlink、minichlink、rvprog.py | proprietary USB bulk、別CDC COM | 1線または2線 | hostのtarget DB/flash stub + Link firmware transport |
| rvswdio_programmer | minichlink、minichlink GDB server | low-speed USB HID feature report | 1線/2線 bitbang | minichlink command + probe内target operation |
| ESP32-S2 funprog | minichlink、minichlink GDB server、testapp | USB HID feature report | 1線/2線 GPIO bitbang | minichlink + probe firmware |
| Swindle | GDB/GDB frontend | USB CDC GDB remote、別CDC UART/RTT、またはTCP | 2線RVSWD PIO/GPIO | probe内Black Magic target/flash driver |
| PicoRVD | GDB/GDB frontend | USB CDC GDB remote | 1線SWIO PIO | probe内V003 flash/soft breakpoint |
| NHC-Link042 | NHC-Link042 CLIまたはminichlink | vendor-specific USB bulk | 1線SWIO | host/minichlink + probe primitive |
| Ardulink | minichlink `-C ardulink -c COMx` | UART/USB serial | 1線GPIO bitbang | minichlink + AVR firmware |
| WCH WebLink | browser、HTTP `curl`、WebSocket client | Wi-Fi HTTP/WebSocket | 1線GPIO bitbang | ESP32 firmware。binaryをWeb upload |
| ESP32-S3 programmer | `tools/flash_tool.py` | USB-UART 115200 | 1線GPIO bitbang | ESP32-S3 firmware。Pythonはcommand/data送信 |
| Flipper flasher | Flipper appまたはminichlink | standalone UI、またはUSB NHC emulation | 1線SWIO | Flipper firmware/minichlink |
| WCH-MCU-DL | 公式configuration software | USBでjob設定後standalone | SWD/USB/UART programming | offline writer内job/algorithm |

代表的な経路は次のようになる。

```text
Arduino IDE -> probe-rs/wlink -> WinUSB/libusb/nusb -> LinkE vendor bulk
            -> LinkE firmware -> SWIO or RVSWD -> DMI -> target flash algorithm

Arduino IDE -> minichlink -> hidapi -> funprog HID feature report
            -> ESP32-S2/CH32V003 probe -> SWIO or RVSWD -> target

GDB/VS Code -> COM port (CDC ACM) -> Swindle/PicoRVD GDB server
            -> probe-side target driver -> RVSWD or SWIO -> target

Browser/curl -> Wi-Fi -> WCH WebLink -> SWIO -> CH32V003
```

## 5. USB classとWindows driver

### 5.1 「driver不要」の意味

WindowsでUSB functionを使うには必ずkernel driverがある。「driver不要」は通常、**利用者がvendor独自driver packageをインストールしなくても、Windows同梱class driverが自動でbindする**という意味である。

| USB方式 | Windows側driver | 初回手動install | host API | probe例 |
|---|---|---:|---|---|
| HID | `hidusb.sys` + `hidclass.sys` | 通常不要 | hidapi/HidD | rvswdio、ESP32-S2 funprog |
| CDC ACM | `usbser.sys` | Win10/11でdescriptor適合なら通常不要。Win7はINF/CDC driverが必要になりやすい | COM port/serial API | PicoRVD、Swindle GDB/UART、LinkE UART/SDI COM |
| Mass Storage/UF2 | `usbstor.sys`等 | 通常不要 | file copy | RP2040 BOOTSELでprobe firmware導入 |
| WinUSB | `winusb.sys` | Microsoft OS descriptorで`WINUSB` compatible IDを出せばWin8+で自動化可能。無ければINF/Zadig等が必要 | WinUSB/libusb/nusb | LinkEをOSS host toolから開く構成 |
| vendor独自driver | WCH/CH375系等 | 必要 | vendor DLL/API | 公式LinkE stack、wlink x86 official-driver backend |
| vendor-specific bulk + libusb | WinUSB/libusbK等へbinding | 通常必要 | libusb | NHC-Link042、OS descriptorを持たない独自bulk device |
| USB-UART bridge | `usbser.sys`またはCH340/CP210x/FTDI driver | bridge chip次第 | COM port | Ardulink、ESP32-S3 boardのUART経路 |
| Wi-Fi/Ethernet | network driverはPC側既存 | probe専用USB driver不要 | HTTP/WebSocket/TCP | WebLink、network Swindle |

Microsoftの[HID architecture](https://learn.microsoft.com/en-us/windows-hardware/drivers/hid/hid-architecture)はsystem-supplied `hidclass.sys`を、[USB serial driver](https://learn.microsoft.com/en-us/windows-hardware/drivers/usbcon/usb-driver-installation-based-on-compatible-ids)はCDC class/subclass `02/02`に対する`usbser.sys`の自動loadを説明している。[WinUSB device documentation](https://learn.microsoft.com/en-us/windows-hardware/drivers/usbcon/automatic-installation-of-winusb)が示すとおり、WinUSBはOS同梱driverだが、自動bindにはMicrosoft OS descriptor/compatible IDが必要である。

### 5.2 WCH-LinkEのWindows binding

WCH-LinkE RISC-V modeはVID:PID `1A86:8010`の複合deviceで、program/debug用のvendor interfaceとCDC serial functionを持つ。

- WCH-LinkUtility/MRSをinstallすると公式WCH-Link driverが入る。manualはLinkUtility単体の場合に`WCHLinkDrv_WHQL_S.exe`の手動実行を案内する。
- `wlink` Windows x86 buildはWCHのCH375 DLL/backendを使えるため、公式driverを維持した構成を選べる。
- probe-rs、wlinkのnusb build、minichlink等から直接bulk transferする場合、programming functionをWinUSBへbindする構成がある。ローカル`../ch32fun/misc/drivers_for_WCH-LinkE`のINFは`VID_1A86&PID_8010&MI_00`をWinUSBへbindする。
- minichlink READMEには「interface 1」と書かれているが、Windows hardware ID上の対象は上記`MI_00`である。Device Manager/Zadigでは文字列ではなくVID/PID/MIを確認する必要がある。
- CDC serial functionは別interfaceなので、programming functionをWinUSBへ変更してもCOM側を残せる。ただしdevice全体や誤ったinterfaceへdriverをbindするとCOMや公式toolを壊す。
- 公式manualはWindows 7でCDC driverが必要と明記する。Windows 10/11では適合descriptorならinbox `usbser.sys`を利用できる。

ARM/DAP modeはPID `8012`、IAP modeは第三者実装で`4348:55E0`として検出される。mode変更でVID/PID/interface構成が変わるため、Arduino discoveryはRISC-V modeだけを見てはいけない。

### 5.3 HID probe

rvswdio programmerとESP32-S2 funprogはvendor-defined HID report/feature reportを使う。Windows標準HID stackへbindするため、通常はZadigやvendor kernel driverが不要である。

ただし次は別に必要になる。

- host application側のhidapi libraryまたは同等API
- 固有VID/PIDとUSB serialによるdevice discovery
- 複数probeを区別できる一意serial descriptor
- HID report size、timeout、protocol versionの互換管理

HIDはdriver配布を単純にできる一方、特にCH32V003 software low-speed USBではbulk/high-speed probeほどのbandwidthは得られない。

### 5.4 CDC/GDB probe

PicoRVDとSwindleはGDB remote protocolをUSB CDC COM portへ流すため、hostに専用flasher libraryを要求しない。GDBの`load`がflash書き込み、通常のGDB commandがhalt/step/breakpointを行う。

- Windows 10/11では適合CDC descriptorならCOM portとして自動認識できる。
- Linuxでは`/dev/ttyACM*`のgroup/udev権限、macOSでは`/dev/cu.usbmodem*`等のport discoveryが必要になる。
- COM番号/pathは再接続で変化し得るため、VID/PID/USB serial/interfaceからGDB portとUART portを対応付ける必要がある。
- Swindleは2または3個のCDC functionを持ち、GDB server、UART bridge、構成によってlog/RTTを分離する。COM番号だけでは役割を確定できない。

## 6. probe firmwareの導入・更新経路

| probe MCU/device | probe firmwareを書き込む経路 | Windows driver |
|---|---|---|
| 公式WCH-LinkE | WCH-LinkUtility/MRS online IAP、別LinkEによるoffline update、wlink-iap | 通常WCH driver/WinUSB。IAP modeは別VID/PID |
| RP2040 PicoRVD/Swindle | BOOTSEL UF2 file copy、別debug probe | UF2はMass Storage classで通常追加driver不要 |
| RP2350 Swindle | BOOTSEL/UF2系、project記載のboard手順 | Mass Storage系 |
| ESP32-S2 funprog | esptool/ESPUtilでESP32 imageを書込 | ROM download CDC/USB方式またはboard上USB-UART次第 |
| ESP32-S3 programmer | PlatformIO/esptool | native USB CDCまたはUSB-UART bridge次第 |
| CH32V003 rvswdio | 最初のWCH-LinkE/minichlink等が必要 | 初回probe書込経路に依存。その後host接続はHID |
| STM32F042 NHC-Link | ST-Link/SWD、DFU等board依存 | 初回書込器/DFU bindingに依存 |
| AVR Ardulink | Arduino bootloader/ISP | boardのUSB-UARTまたは16U2 CDC次第 |

「完成後のprobeがdriverless」であっても、そのprobe firmwareを最初に導入するbootstrap経路は別に必要である。

## 7. Arduino統合で見えるdevice

一つの物理probeが複数のlogical deviceを出すため、Arduinoのupload portとmonitor portを一つのCOM portとして扱えない場合がある。

| logical function | Arduinoからの用途 | discovery key |
|---|---|---|
| LinkE vendor bulk | upload/debug | VID/PID/interface/serial |
| LinkE CDC | physical UARTまたはSDI monitor | VID/PID/interface/serial + COM path |
| Swindle CDC 0 | GDB upload/debug | interface string/number + serial |
| Swindle CDC 1 | target UART | interface string/number + serial |
| Swindle CDC 2 | log/RTT | build variant + interface string/number |
| HID funprog | upload/debug/terminalをprotocol内multiplex | VID/PID/serial/report protocol version |
| UF2 boot volume | probe firmware recovery | volume label/board ID。target upload portではない |

Windowsではさらに、各interfaceに現在bindされているdriver名をdiagnostic情報として取得できると、`device found but cannot open`をpermission、WCH driver、WinUSB、CDCの違いに分けられる。

## 8. licenseと再利用範囲

| project | 主license | host protocol再利用上の注意 |
|---|---|---|
| WCH-LinkE firmware/protocol | proprietary firmware、USB protocolは第三者解析 | firmware cloneや公式VIDの流用は別問題。host側OSS実装はprobe-rs/wlink/minichlink等に存在 |
| rvswdio / rv003usb | MIT系 | low-speed HID、SWIO/RVSWD、terminalの小型参照実装 |
| ESP32-S2 cookbook | MIT | ch32fun由来部分やESP-IDF componentを含むため、再配布時はfile単位のnoticeも監査 |
| Swindle | GPL-3.0系 + component licenses | Black Magic由来。firmware派生物の配布条件を確認 |
| PicoRVD | MIT | V003専用だが層分離された参照実装 |
| NHC-Link042 | MIT | STM32 vendor library同梱部分は別noticeを確認 |
| WCH WebLink | MIT | minichlink/ESP32-S2由来部分を含む |

USB classをHIDやCDCにすること自体にはproject licenseの制約はないが、VID/PIDの割当、USB stack、取り込んだprobe/target codeのlicenseは別々に管理する必要がある。

## 9. 参照資料と調査版

### 公式・platform資料

- [WCH-Link manual V2.4](https://www.wch.cn/uploads/file/20250124/1737704462135866.pdf)
- [WCH-Link manual download page](https://www.wch-ic.com/downloads/WCH-LinkUserManual_PDF.html)
- [WCH official CH32V003 1-line example](https://github.com/openwch/ch32v003/tree/main/CH32V003_1Line_Base_on_CH32F103)
- [Microsoft HID architecture](https://learn.microsoft.com/en-us/windows-hardware/drivers/hid/hid-architecture)
- [Microsoft USB serial driver](https://learn.microsoft.com/en-us/windows-hardware/drivers/usbcon/usb-driver-installation-based-on-compatible-ids)
- [Microsoft automatic WinUSB installation](https://learn.microsoft.com/en-us/windows-hardware/drivers/usbcon/automatic-installation-of-winusb)

### probe source

- [rv003usb rvswdio programmer](https://github.com/cnlohr/rv003usb/tree/master/rvswdio_programmer)
- [ESP32-S2 CH32 programmer](https://github.com/cnlohr/esp32s2-cookbook/tree/master/ch32v003programmer)
- [Swindle](https://github.com/mean00/swindle)
- [PicoRVD](https://github.com/aappleby/picorvd)
- [NHC-Link042](https://github.com/NgoHungCuong/NHC-Link042)
- [WCH WebLink](https://github.com/Subjective-Reality-Labs/WCH_WebLink)
- [ESP32-S3 CH32V003 programmer](https://github.com/Ishu1519/esp32s3-ch32-programmer)
- [Flipper Zero SWIO flasher](https://github.com/sukvojte/wch_swio_flasher)
- [Ardulink/zooswio](https://github.com/zoobab/zooswio)

### 調査した版

- `../ch32fun`: `6c4dd53` (2026-08-24)、minichlink backend、Windows INF、udev rules
- 一時clone: rv003usb `80b1893`、ESP32-S2 cookbook `ca21357`、Swindle `3435ffe`、PicoRVD `4310ade`、NHC-Link042 `77c74f2`
- `../tools/MounRiverStudio_Linux_X64_V2.5.0`および`../tools/MRS_Toolchain_Linux_X64_V240`

probe firmwareとhost implementationは更新されるため、本表は上記日時点のsource snapshotである。実装済み、READMEで主張、実機確認済みは同義ではない。
