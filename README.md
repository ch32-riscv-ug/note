# CH32開発ツール調査メモ

このリポジトリは、CH32 RISC-V向けArduino環境で利用できる書き込み・デバッグ・bootloader経路の現状をまとめる調査メモです。

## 調査資料

- [書き込み・デバッグ・bootloader経路と独自プローブの現状調査レポート](research/programming-tools-and-probes.ja.md)
- [WCH-LinkE経由の書き込みアプリ、OS・機能・ライセンスと理想形](research/wch-linke-host-apps.ja.md)

## 調査範囲

- WCH-Link系、factory USB/UART ISP、独自probe、custom bootloader、IAP/OTA
- host OS、対応MCU、書き込み・デバッグ機能、配布形態、license
- CH32V003の1線SWIOと、CH32V2xx/V3xx等の2線RVSWDの第三者実装
- software USB、hardware USB DFU/UF2、UART/RS-485、外部媒体、offline writer
