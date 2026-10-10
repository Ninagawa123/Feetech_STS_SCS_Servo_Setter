# Feetech STS SCS Servo Setter

ESP32 と半二重通信回路を使って、Feetech のシリアルバスサーボ（**STS / SCS 系列**）を
シリアルモニターから対話的に設定・確認・動作テストするためのツールです。

- サーボの検索（STS / SCS の自動判定つき）
- ID の変更、通信速度（ボーレート）の変更
- 位置・電圧・バージョン（型番）の読み取り
- 任意レジスタの読み書き
- 複数サーボの一斉制御（SyncWrite）、トルク ON / OFF

---

## 目次

1. [必要なもの](#必要なもの)
2. [配線](#配線)
3. [ビルドと書き込み](#ビルドと書き込み)
4. [はじめの使い方（5ステップ）](#はじめの使い方5ステップ)
5. [コマンド一覧](#コマンド一覧)
6. [各コマンドの使用例](#各コマンドの使用例)
7. [STS と SCS の違い](#sts-と-scs-の違い)
8. [注意点](#注意点)
9. [トラブルシューティング](#トラブルシューティング)
10. [技術メモ](#技術メモ)

---

## 必要なもの

| 項目 | 内容 |
|---|---|
| マイコン | ESP32（PlatformIO の `esp32dev`） |
| 回路 | 半二重通信回路（TX/RX を 1 本の信号線にまとめる回路） |
| サーボ | Feetech STS 系列（STS3215 など）/ SCS 系列（SCS0009 など） |
| 電源 | サーボの定格電圧に合った電源 |
| 開発環境 | [PlatformIO](https://platformio.org/)（VSCode 拡張または CLI） |

[Meridian Board Lite](https://github.com/Ninagawa123/Meridian_LITE/blob/main/README.md)があれば、ESP32と半二重回路のセットとして使えます。
その場合、ボード上のLeft系列にサーボを接続してください。
それ以外の場合は、下記のように配線してください。

## 配線

| 信号 | ESP32 ピン | 設定場所 |
|---|---|---|
| TX | GPIO 27 | `platformio.ini` の `-D TX1=27` |
| RX | GPIO 32 | `platformio.ini` の `-D RX1=32` |
| EN（送受信切替） | GPIO 33 | `src/main.cpp` の `PIN_EN` |
| GND | GND | サーボ電源と ESP32 の GND を共通にする |

- EN = HIGH で送信、LOW で受信です。
- PC とのシリアルモニター速度は **115200 bps** です。

## ビルドと書き込み

`Feetech_STS_SCS_Servo_Setter/` フォルダ（`platformio.ini` がある場所）で実行します。

```bash
pio run              # ビルド
pio run -t upload    # ESP32 に書き込み
pio device monitor   # シリアルモニターを開く（115200 bps）
```

> シリアルモニターは **改行コードを送る設定**（LF / CR / CRLF のいずれか）にしてください。
> 「改行なし」だと空欄リターンが送られず、入力待ちのまま止まります。

## はじめの使い方（5ステップ）

1. **書き込んでモニターを開く**
   起動するとメニューが表示されます。先頭行に現在の設定が出ます。
   ```
   Current target ID: 1, Sync IDs: 1,2,3,4,5,6, Baud rate: 1000000 bps, Series: STS
   ```
2. **`f` でサーボを探す**
   空欄のままリターンすると ID1〜20 を検索します。
   見つかったサーボの系列（STS / SCS）が自動判定され、**系列・ターゲット ID・一斉制御 ID リストが自動で設定**されます。
3. **`m` で型番・バージョンを確認する**
4. **必要なら `i` で ID を変更する**（新品のサーボは ID1 なので、**1 台ずつ** つないで変更）
5. **`e` → `x` で動作テスト**
   トルクを ON にして、角度を一斉送信します。

> 何も入力せずにリターンすると、いつでもメニューを再表示できます。

## コマンド一覧

コマンドは 1 文字を入力してリターンします。

### 検索・設定

| コマンド | 機能 | 説明 |
|---|---|---|
| `f` | Find servos | サーボを検索。STS / SCS を自動判定し、系列・ターゲット ID・一斉制御 ID リストを自動設定 |
| `z` | Set target ID | 個別コマンド（`r` `v` `p` `s` `i` `m` など）の対象 ID を設定 |
| `n` | Set sync servo IDs | 一斉制御する ID リストを設定（例：`1,3,7`、`1-4,7`） |
| `t` | Servo series | サーボ系列を手動で STS / SCS に切り替え |
| `b` | Set baud rate | 通信速度を変更（ESP32 のみ / サーボ＋ESP32） |
| `i` | Change servo ID | ターゲットサーボの ID を書き換え（EEPROM に保存） |

### 読み取り

| コマンド | 機能 | 説明 |
|---|---|---|
| `m` | Read version | 系列・ファームウェア版・型番を表示 |
| `r` | Read position | ターゲットサーボの現在位置と角度 |
| `v` | Read voltage | ターゲットサーボの電圧（範囲外なら警告） |
| `a` | Read all positions | リストの全サーボ の位置を一括表示 |
| `p` | Read register | 任意のアドレスから 1〜4 バイト読み取り |

### 動作・書き込み

| コマンド | 機能 | 説明 |
|---|---|---|
| `e` | All torque ON | リストの全サーボ のトルクを ON |
| `d` | All torque OFF | リストの全サーボ のトルクを OFF |
| `x` | Send angle to all | リストの全サーボ に同じ角度を一斉送信 |
| `c` | All to center | リストの全サーボ を中央（0°）へ |
| `s` | Write register | 任意のアドレスに 1〜2 バイト書き込み（書き込み後に読み返して確認） |

> 「リスト」は `n` コマンドまたは `f` で設定される一斉制御 ID リスト（初期値 `1,2,3,4,5,6`）です。
> ID がとびとびでも使えます。

## 各コマンドの使用例

### `f` サーボ検索

```
f
Enter maximum ID to scan (1-253, empty = 20):        ← 空欄リターンで ID20 まで
Found servo ID: 1 (STS)
Found servo ID: 2 (STS)
=== Servo Detection Results ===
Found 2 servo(s): STS=2, SCS=0
Connected IDs: 1(STS), 2(STS)
Servo series auto-set to: STS
Target ID auto-set to: 1
Sync servo IDs auto-set to: 1,2
```

- 系列はサーボのアドレス 2（END）の値で判定します（0 = STS、1 = SCS）。
- 複数見つかった場合、ターゲット ID は「設定された系列と同じサーボのうち最小の ID」になります。
- 一斉制御 ID リストには、見つかったサーボのうち設定された系列と同じものが入ります。
- STS と SCS が混在している場合は、台数の多い方に設定され、警告が表示されます。

### `m` バージョン・型番の読み取り

```
m
Servo series    : STS (END=0)
Firmware version: 3.10
Servo version   : 9.3
Model number    : 777 (0x0309) -> STS3215
```
（数値は例です）

型番名が分かっているのは次の 4 種類です。それ以外は `Unknown` と表示されますが、
数値は表示されるので、`src/main.cpp` の `STS_MODELS[]` に 1 行追加すれば表示できます。

| 型番コード | 型番 |
|---|---|
| 777 | STS3215 |
| 2825 | STS3250 |
| 11272 | SM8512BL |
| 1284 | SCS0009 |

### `i` ID の変更

```
i
Current target servo ID: 1
Enter new servo ID (1-253): 5
Changing servo ID 1 -> 5
Servo ID changed successfully [OK]. Target ID is now 5
```

- 変更元が応答しない場合、変更先の ID がすでに使われている場合は中止します。
- **同じ ID のサーボが複数つながっていると全部が書き換わります。1 台ずつ** 行ってください。

### `b` 通信速度の変更

```
b
Current ESP32 baud rate: 1000000 bps
  [0] 1000000 bps
  [1] 500000 bps
  [2] 250000 bps
  [3] 128000 bps
  [4] 115200 bps
  [5] 76800 bps
  [6] 57600 bps
  [7] 38400 bps
Enter baud rate number (0-7): 4
Apply to: [1] ESP32 only
          [2] Servo ID 1 (EEPROM) + ESP32
2
Verification: Baud rate register = 4 [OK]
```

| 選択 | 動作 | 使いどころ |
|---|---|---|
| `[1]` ESP32 only | ESP32 側だけ速度を変える | すでに別の速度に設定されたサーボと通信したいとき |
| `[2]` Servo + ESP32 | サーボの設定を書き換え、ESP32 側も合わせる | サーボの通信速度を変更したいとき |

> ESP32 は起動時に常に 1 Mbps で始まります。サーボの速度を変えた後は、起動のたびに
> `b` → `[1]` で ESP32 側を合わせてください。

### `n` 一斉制御 ID リストの設定

```
n
Current sync servo IDs: 1,2,3,4,5,6
Enter sync servo IDs (e.g. 1,3,7 or 1-4,7): 1-3,7,12
Sync servo IDs changed to: 1,2,3,7,12 (5 servos)
```

- カンマ区切りで ID を、`-` で範囲を指定できます。重複は除かれ、小さい順に並びます。
- 1 台だけにしたいときは `5` のように 1 つだけ入力します。
- 1 パケットに入る台数（STS 31 台 / SCS 35 台）を超える場合は、自動で分割送信します。

### `t` サーボ系列の切り替え

```
t
Current servo series: STS
Select servo series: [1] STS  [2] SCS
2
Servo series changed to: SCS (position 0-1023, 300 deg)
```

通常は `f` で自動設定されるので、手動で切り替える必要はありません。

### `x` 角度の一斉送信

```
x
Enter angle (-180 to 180 degrees): 90        ← SCS の場合は -150 to 150
Converting 90.00 degrees to position 3072
Sending angle command to 6 servos (IDs: 1,2,3,4,5,6)...
```

### `p` / `s` レジスタの読み書き

```
p
Enter register address (0-255): 40
Enter data length to read (1-4 bytes): 1
Register 40 (0x28): Dec=1, Hex=0x01, Bin=1

s
Enter register address (0-255): 40
Enter data length to write (1-2 bytes): 1
Enter decimal value to write (0-255): 1
Writing 1 (0x01) to address 40 of servo ID 1
Verification: Read value = 1 (0x01) [OK]
```

2 バイトの値は、現在の系列に合わせたバイト順で読み書きされます。

## STS と SCS の違い

このツールは系列の設定（`f` で自動 / `t` で手動）に合わせて、以下を自動で切り替えます。

| 項目 | STS | SCS |
|---|---|---|
| 2 バイト値のバイト順 | 下位バイトが先（リトルエンディアン） | 上位バイトが先（ビッグエンディアン） |
| EEPROM ロックのアドレス | 55 | 48 |
| 一斉送信の形式 | アドレス 41 から 7 バイト（加速度あり） | アドレス 42 から 6 バイト（加速度なし） |
| 位置の範囲 | 0〜4095（中央 2048） | 0〜1023（中央 512） |
| 角度の範囲 | ±180°（360°） | ±150°（300°） |

共通のもの：ID（5）、ボーレート（6）、トルク（40）、現在位置（56）、電圧（62）、
バージョン情報（0〜4）、通信パケットの形式。

> SCS の角度範囲は SCS0009 などを想定して 300° に固定しています。
> 機種によって異なるため、合わない場合は `src/main.cpp` の `SCS_ANGLE_RANGE` を変更してください。

## 注意点

- **起動時の動作**：電源を入れると、ID1〜6 のうち最初に応答したサーボで系列（STS / SCS）を自動判定し、
  ID1・ID2 のトルクを ON、ID1〜6 を中央位置へ動かします（1 Mbps）。サーボが急に動くことがあるので注意してください。
- ID1〜6 のどれも応答しない場合は STS として扱います。その場合は `f` か `t` で系列を設定してください。
- `i` と `b` の `[2]` は EEPROM に書き込みます。実行前に系列が正しく設定されているか確認してください。

## トラブルシューティング

| 症状 | 考えられる原因 | 対処 |
|---|---|---|
| `f` で何も見つからない | 配線・電源・通信速度の不一致 | 配線（EN/TX/RX/GND）と電源を確認。`b` → `[1]` で ESP32 側の速度を変えて再検索 |
| 入力しても反応しない | モニターが改行を送っていない | シリアルモニターの改行設定を LF / CR / CRLF にする |
| 位置の値がおかしい | 系列の設定が違う | `m` で系列を確認し、`t` で合わせる |
| `i` / `b` の変更が電源 OFF で戻る | 系列が違い、EEPROM のロック解除に失敗 | 系列を合わせてから再実行 |
| `b` の後に応答しない | サーボと ESP32 の速度がずれた | `b` → `[1]` で他の速度を順に試す |
| 電圧に Warning が出る | 電源電圧が 5.0〜14.0 V の範囲外 | 電源を確認 |

## 技術メモ

### 主なレジスタアドレス

| アドレス | 内容 | 備考 |
|---|---|---|
| 0 / 1 | ファームウェア版（メジャー / マイナー） | 読み取り専用 |
| 2 | END（0 = STS、1 = SCS） | 系列判定に使用 |
| 3 / 4 | 型番コード | 読み取り専用 |
| 5 | ID | EEPROM |
| 6 | ボーレート（0〜7） | EEPROM |
| 40 | トルク ON / OFF | |
| 41〜 / 42〜 | 一斉送信の書き込み先（STS / SCS） | |
| 55 / 48 | EEPROM ロック（STS / SCS） | 0 = 書き込み可、1 = ロック |
| 56 | 現在位置 | 2 バイト |
| 62 | 電圧 | 単位 0.1 V |

### 主な設定値（`src/main.cpp`）

| 名前 | 初期値 | 内容 |
|---|---|---|
| `targetServoID` | 1 | 個別コマンドの対象 ID |
| `syncServoIDs` | 1,2,3,4,5,6 | 一斉制御 ID リスト（`syncServoCount` は台数） |
| `servoBaudIndex` | 0（1 Mbps） | ESP32 側の通信速度 |
| `servoType` | STS | サーボ系列 |
| `DEFAULT_SCAN_MAX_ID` | 20 | `f` で空欄リターン時の最大 ID |
| `SCS_ANGLE_RANGE` | 300.0 | SCS の可動角度 |

## 免責事項

このコードはテスト・学習目的で提供しています。実際に使う前に十分に動作確認してください。
サーボの損傷や予期しない動作について、作者は責任を負いません。

## ライセンス

MIT License

## 関連リンク

- [Feetech 公式サイト](https://www.feetechrc.com/)
- [FEETECH STS メモリテーブル解説](https://aifitlab-wiki.super.site/feetech-servo-motor-docs/feetech-sts-servo-memory-table-analysis)
- [FEETECH SCSCL メモリテーブル解説](https://aifitlab-wiki.super.site/feetech-servo-motor-docs/feetech-scscl-servo-memory-table-analysis)
- [STS3215 仕様（Waveshare Wiki）](https://www.waveshare.com/wiki/ST3215_Servo)
- [PlatformIO](https://platformio.org/)
