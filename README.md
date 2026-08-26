# CH32開発ツール調査メモ

このリポジトリは、CH32 RISC-V向けArduino環境と、その書き込み・デバッグ・bootloader基盤の方向性を決めるための調査メモです。実装は仕様確定後に個別リポジトリへ分離します。

## 調査資料

- [書き込み・デバッグ・bootloader経路と独自プローブの事前調査](research/programming-tools-and-probes.ja.md)

## 現時点の要約

- 当面のArduinoアップロードは`probe-rs`を主経路とし、未対応MCUはWCH OpenOCDまたは個別の上流追加で埋める。
- 独自ホストツールを作るなら、まず複数backendを隠す薄い`ch32-upload`から始め、デバッグエンジンを一から作らない。
- LinkE代替ハードは実現可能。CH32V003系の1線SWIOだけでなく、CH32V2xx/V3xxの2線RVSWDをRP2040で実装した実例がある。
- 独自プローブの第一候補はRP2040/RP2350。物理層をPIOへ閉じ込め、ターゲットDBとフラッシュ処理はホスト側へ置く。
- カスタムbootloaderも正式な経路とし、V003/V00Xのsoftware USB/UART、USB内蔵品のDFU/UF2、UART/RS-485、IAP/OTAをboard別profileで扱う。
- 最初から「全WCH-LinkE互換」を目標にせず、書き込み・verify・reset・一意なprobe選択をMVPにする。UART、SDI print、電源制御、GDB/DAP、全familyは段階追加する。
