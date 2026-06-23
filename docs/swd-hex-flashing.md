# ZMK firmware を SWD 経由で HEX 書き込みする手順

このメモは、ZMK firmware を UF2 のドラッグアンドドロップではなく、SWD 経由で `*.hex` として書き込むための手順です。

ここでは次の流れを前提にします。

- WSL 上で ZMK firmware をビルドする
- 生成された HEX を Windows の Downloads フォルダへコピーする
- Windows 上の MSYS UCRT64 から OpenOCD と telnet を実行する
- ST-Link などの SWD プローブで nRF52840 に書き込む

## 前提

- MCU: nRF52840
- 書き込み方法: SWD
- 書き込み環境: Windows 上の MSYS UCRT64
- Debug probe: ST-Link など
- OpenOCD telnet port: `4444`
- OpenOCD target: nRF52 / nRF52840
- ZMK firmware HEX: `<board>.hex`
- 起動領域用 HEX: `s140_nrf52_7.3.0_softdevice.hex`

起動領域用 HEX には、Adafruit nRF52 Bootloader repository 内の次のファイルを使います。

```text
Adafruit_nRF52_Bootloader/lib/softdevice/s140_nrf52_7.3.0/s140_nrf52_7.3.0_softdevice.hex
```

この記事の例では、ZMK firmware を `0x1000` から配置する構成を扱います。`s140_nrf52_7.3.0_softdevice.hex` は、実質的には先頭の MBR/起動領域を用意するために使います。ZMK/Zephyr のアプリケーション HEX を `0x1000` から書くため、SoftDevice 本体領域の多くは ZMK firmware で上書きされます。

## flash partition の例

ZMK firmware を `0x1000` から配置する場合、DTS の flash partition はたとえば次のようになります。

```dts
sd_partition: partition@0 {
    reg = <0x00000000 0x00001000>;
};

code_partition: partition@1000 {
    reg = <0x00001000 0x000d3000>;
};

storage_partition: partition@d4000 {
    reg = <0x000d4000 0x00020000>;
};

boot_partition: partition@f4000 {
    reg = <0x000f4000 0x0000c000>;
};
```

ビルドログでは次のように `start address: 0x1000` になれば、この構成どおりです。

```text
Converted to uf2, output size: ..., start address: 0x1000
```

`0x26000` 開始の partition は、Adafruit bootloader などがアプリケーションへジャンプする前提の構成で使われます。bootloader を書かずに SWD でアプリだけを直書きする場合は、アプリの開始アドレスと実際の起動経路が一致しているか確認してください。

## ビルド

WSL 側の workspace root でビルドします。

```sh
cd /home/takec/zmk-workspace
nix develop -c just build <board> -p
```

ビルド後、次の HEX が生成されます。

```text
/home/takec/zmk-workspace/firmware/<board>.hex
```

たとえば board 名が `ms88sf21` なら、生成される HEX は次のようになります。

```text
/home/takec/zmk-workspace/firmware/ms88sf21.hex
```

## HEX を Windows 側にコピーする

OpenOCD は Windows 上の MSYS UCRT64 で実行するため、WSL 上で生成した ZMK firmware HEX を Windows の Downloads フォルダへコピーします。

WSL からコピーする場合:

```sh
cp /home/takec/zmk-workspace/firmware/<board>.hex /mnt/c/Users/<WindowsUser>/Downloads/<board>.hex
```

起動領域用 HEX も Downloads にコピーしておきます。

```sh
cp /path/to/Adafruit_nRF52_Bootloader/lib/softdevice/s140_nrf52_7.3.0/s140_nrf52_7.3.0_softdevice.hex /mnt/c/Users/<WindowsUser>/Downloads/s140_nrf52_7.3.0_softdevice.hex
```

`<WindowsUser>` は Windows のユーザー名に置き換えます。

エクスプローラーから WSL のファイルを開いて、Downloads にコピーしても構いません。

```text
\\wsl.localhost\<DistroName>\home\<LinuxUser>\zmk-workspace\firmware\<board>.hex
```

## OpenOCD を起動する

MSYS UCRT64 を開き、Windows 側で OpenOCD を起動します。ST-Link を使う場合の例です。

```sh
openocd -f interface/stlink.cfg -f target/nrf52.cfg
```

OpenOCD は起動したままにしておきます。別の MSYS UCRT64 ターミナルを開いて telnet します。

```sh
telnet 127.0.0.1 4444
```

## 初回書き込み: mass erase あり

初回や状態を完全にリセットしたい場合は、先に `nrf5 mass_erase` を実行します。

```text
reset halt
nrf5 mass_erase
program C:/Users/<WindowsUser>/Downloads/s140_nrf52_7.3.0_softdevice.hex verify
program C:/Users/<WindowsUser>/Downloads/<board>.hex verify reset
exit
```

MSYS style のパスで指定する場合:

```text
reset halt
nrf5 mass_erase
program /c/Users/<WindowsUser>/Downloads/s140_nrf52_7.3.0_softdevice.hex verify
program /c/Users/<WindowsUser>/Downloads/<board>.hex verify reset
exit
```

`nrf5 mass_erase` を実行すると、BLE のペアリング情報や ZMK settings も消えます。書き込み後は PC 側で古い BLE デバイスを削除し、再ペアリングしてください。

## 初回書き込み: mass erase なし

すでに必要な起動領域が入っていて、全消去したくない場合は `nrf5 mass_erase` を省略します。

```text
reset halt
program C:/Users/<WindowsUser>/Downloads/s140_nrf52_7.3.0_softdevice.hex verify
program C:/Users/<WindowsUser>/Downloads/<board>.hex verify reset
exit
```

ただし、以前に別の partition 構成や別 firmware を書いていた場合は、古いデータが残って挙動が不安定になることがあります。その場合は `mass erase あり` の手順で一度初期化した方が切り分けやすくなります。

## 通常の firmware 更新

一度起動領域を書いて動作確認できたあとは、通常は ZMK firmware だけを書き込めば十分です。

```text
reset halt
program C:/Users/<WindowsUser>/Downloads/<board>.hex verify reset
exit
```

MSYS style のパスで指定する場合:

```text
reset halt
program /c/Users/<WindowsUser>/Downloads/<board>.hex verify reset
exit
```

通常の firmware 更新では `nrf5 mass_erase` を使わない方が、BLE ペアリング情報を維持しやすくなります。

## BLE ペアリングが消える条件

ZMK の BLE bond 情報は `storage_partition` に保存されます。

次の場合は PC 側のペアリングを削除して再ペアリングが必要になることがあります。

- `nrf5 mass_erase`
- `erase_all`
- `recover`
- OpenOCD や probe tool 側で chip erase を実行した
- `storage_partition` の位置やサイズを変更した
- firmware 側の BLE identity や device name を変更した

通常の firmware 更新では、全消去を避けて ZMK firmware HEX だけを書き込みます。

## 回路情報の整理

キーマトリクスの入力が動かない場合、firmware 書き込みではなく kscan 設定や回路側の問題であることも多いです。記事やメモに残す場合は、次の情報を整理しておくと切り分けしやすくなります。

```text
MCU:
Board/module:
Rows:
Columns:
Diode direction:
Row GPIOs:
Column GPIOs:
Encoder:
Pointing device:
Battery measurement:
Bootloader:
Application start address:
Storage partition:
SWD probe:
OpenOCD command:
```

## matrix/kscan の一般的な注意点

ZMK の `zmk,kscan-gpio-matrix` では、回路上のダイオード向きと `diode-direction`、row/column GPIO の設定を揃える必要があります。

たとえば `diode-direction = "col2row"` の場合、一般的には columns がスキャン出力、rows が入力になります。

```dts
kscan0: kscan_0 {
    compatible = "zmk,kscan-gpio-matrix";
    row-gpios = <
        &gpio0 <ROW0_PIN> (GPIO_ACTIVE_HIGH | GPIO_PULL_DOWN)
        &gpio0 <ROW1_PIN> (GPIO_ACTIVE_HIGH | GPIO_PULL_DOWN)
    >;
    col-gpios = <
        &gpio0 <COL0_PIN> GPIO_ACTIVE_HIGH
        &gpio0 <COL1_PIN> GPIO_ACTIVE_HIGH
    >;
    diode-direction = "col2row";
};
```

実際の GPIO flags は、使っている matrix driver、ダイオードの向き、外付けプルアップ/プルダウンの有無に合わせて確認してください。

## トラブルシュートの順番

1. ビルドログで application start address を確認する
2. OpenOCD で `reset halt` できるか確認する
3. 初回は `nrf5 mass_erase` ありで起動領域と ZMK HEX を書く
4. PC 側で古い BLE デバイスを削除する
5. BLE scan にデバイス名が出るか確認する
6. BLE に出ない場合は起動領域、partition、application start address を疑う
7. BLE に出るがキー入力がない場合は kscan の row/col/diode direction を疑う
8. 入力が暴れる場合は pull-up/pull-down、未はんだスイッチ、導通、未接続ピンを疑う

BLE scan に出ているなら、少なくとも firmware は起動しています。そこから先は flash 書き込みよりも、matrix や keymap の切り分けに進む方が近道です。
