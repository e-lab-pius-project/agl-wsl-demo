# UbuntuとQEMU内のAGLでCANの双方向通信を確認する

[起動ガイド](../README.md) · [USB-CANとcannelloni](USB_CAN.md) · [VESC回転数表示](VESC_RPM.md)

USB-CANアダプター2台をCANケーブルで接続し、片方をWSLのUbuntu、もう片方をQEMU内のAGLから操作します。**実際のCAN配線を通した通信試験**です。アダプター内部のループバック機能やcannelloniは使いません。

## 1. 構成と検証結果

```text
Ubuntu（WSL）                        QEMU内のAGL
SocketCAN / gs_usb                  SocketCAN / gs_usb
       ↓                                  ↓
アダプターB ←──── CAN_H / CAN_L ────→ アダプターA
```

両方のUSB機器をWindowsからWSLへ渡した後、AだけをQEMUへUSBパススルーします。UbuntuとAGLは別々のカーネルなので、`gs_usb`も共有しません。AはUbuntuの `lsusb` にも残りますが、パススルー中はUbuntuのSocketCANから操作できません。

2026-10-06に確認した構成:

| 項目 | 検証時の値 |
|---|---|
| Windows / WSL | Windows 11 / WSL2、Ubuntu 24.04（環境名 `wsl-agl`） |
| QEMU / AGL | QEMU 8.2.2 / 配布版 `agl-20261001`、AGL 21.93.0 |
| アダプター | USB2CANFD 2台。USB名はCANnectivity USB to CAN adapter |
| VID:PID / ドライバー | `1209:ca01` / `gs_usb` |
| A：AGL用 | Windows BUSID `6-3`、WSL USB Bus 001 Device 002、AGL `can1` |
| B：Ubuntu用 | Windows BUSID `9-1`、WSL USB Bus 001 Device 003、Ubuntu `can0` |
| AGLの既存 `can0` | 仮想CAN。今回の物理通信には使わない |
| 通信方式 | Classical CAN、500 kbit/s、8バイト |
| 試験結果 | 各方向6フレーム。標準・拡張IDと全データが一致、通信エラー0 |

番号は接続順や再起動で変わるため毎回確認します。**CAN FDの通信・高負荷・遅延の測定は含みません。**

## 2. 配線と準備

- 2台のCAN_H同士、CAN_L同士を接続します。基準GNDの接続は製品仕様に従います。
- バスの両端を120 Ωで終端します。内蔵終端が有効なら重複して追加しません。
- 試験対象は2台だけとし、VESCや車両など他の機器は接続しません。
- [起動ガイド](../README.md)と [USB-CAN手順](USB_CAN.md)のusbipd導入を済ませます。

| 表記 | 実行場所 |
|---|---|
| PowerShell | Windows。bindのみ管理者権限が必要 |
| Ubuntu | WSLのUbuntuターミナル |
| QEMUモニター | QEMUを起動したターミナルの `(qemu)` プロンプト |
| AGL | `bash scripts/ssh.sh` で接続したSSHターミナル |

## 3. 2台をWindowsからWSLへ渡す

**PowerShell:** Ubuntuを開いたままにします。環境名は `wsl -l -v` で確認します。

```powershell
wsl -d wsl-agl
```

**別の管理者PowerShell:** 同じ機器名が2台あるため、必要なら1台ずつ挿してA・Bを区別し、ラベルを付けます。

```powershell
usbipd list
$canA = Read-Host 'AGL用アダプターAのBUSID'
$canB = Read-Host 'Ubuntu用アダプターBのBUSID'
usbipd bind --busid $canA
usbipd bind --busid $canB
usbipd list
```

Sharedなら共有登録済みです。登録済みの機器はbind不要です。検証時には `edevmon` の警告が出ましたが通常のbindで接続できました。警告だけを理由に `--force` は付けません。

**PowerShell:** まずAだけをattachします。別のPowerShellを使う場合は `$canA` と `$canB` を再入力します。attachには管理者権限は不要です。

```powershell
usbipd attach --wsl wsl-agl --busid $canA
```

**Ubuntu:** AのUSBバス番号・デバイス番号を記録します。

```bash
lsusb
lsusb -t
```

検証例は `Bus 001 Device 002: ID 1209:ca01 ...` でした。次に **PowerShell** でBをattachし、Ubuntuの `lsusb` で追加された機器を確認します。

```powershell
usbipd attach --wsl wsl-agl --busid $canB
```

既に両方Attachedで区別できない場合は、QEMUが停止している状態でAを `usbipd detach --busid $canA` し、消えた行を確認してから再attachします。再attach後の番号を採用します。WindowsのBUSID `6-3` とUbuntuのUSB番号 `001/002` は別の識別子です。

## 4. QEMUをモニター付きで起動する

既にAGLが起動中なら作業を保存し、Ubuntuのリポジトリ内から `bash scripts/ssh.sh poweroff` で終了します。QEMUが終了してから起動し直します。

**Ubuntu:** 通常ユーザーがAへアクセスできるよう、対象1台にだけ権限を付けます。`001/002` は3節で記録した値へ置き換えます。

```bash
sudo apt install -y acl can-utils usbutils
ls -l /dev/bus/usb/001/002
sudo setfacl -m "u:$(id -un):rw" /dev/bus/usb/001/002
cd ~/agl-wsl-demo
bash scripts/start.sh -monitor stdio
```

`start.sh` は追加引数をQEMUへ渡します。`-monitor stdio` により、このターミナルがQEMUモニターになります。QEMU全体をsudoで起動する必要はありません。USBを再接続したら番号と権限を再確認します。

AGLの起動を待ち、**別のUbuntuターミナル** からSSHで入ります。

```bash
cd ~/agl-wsl-demo
bash scripts/ssh.sh
```

**AGL:**

```bash
ip -details link show can0
```

この配布版では起動時に仮想 `can0` が作られます。`vcan` と表示されることを確認してからUSBを追加します。GUIのContinueは、SSHやCAN認識の確認には不要です。

## 5. AだけをQEMUへ渡す

**QEMUモニター `(qemu)` :** Bus 001 Device 002のAを渡す例です。Bを選ばないよう、実際の番号に合わせます。

```text
info usbhost
device_add usb-host,hostbus=1,hostaddr=2,id=usbcan
info usb
```

2台は同じVID:PIDなので、`vendorid=0x1209,productid=0xca01` だけで選択せず、バス番号・デバイス番号で指定します。

**AGL:**

```bash
lsusb
ip -details link show type can
readlink -f /sys/class/net/can1/device/driver
```

検証時はAが `can1`、ドライバーが `gs_usb` でした。実際の名前を記録します。

**Ubuntu:**

```bash
lsusb -t
ip -details link show type can
```

AはQEMUが使用し、`lsusb -t` では `usbfs` と表示されることがあります。Bは `gs_usb` のまま残ります。UbuntuでBが `can1` などになっていても異常ではありません。必要なら `readlink -f /sys/class/net/can1/device` でUSB機器との対応を確認します。

以降はUbuntuのBを `HOST_CAN`、AGLのAを `GUEST_CAN` に設定します。以下は検証時の名前の例です。変数はそれぞれのターミナルで設定します。

## 6. 両方を500 kbit/sで有効化する

**AGL:**

```bash
GUEST_CAN=can1
ip link set "$GUEST_CAN" down
ip link set "$GUEST_CAN" type can bitrate 500000 fd off listen-only off loopback off
ip link set "$GUEST_CAN" up
ip -details -statistics link show "$GUEST_CAN"
```

**Ubuntu:**

```bash
HOST_CAN=can0
sudo ip link set "$HOST_CAN" down
sudo ip link set "$HOST_CAN" type can bitrate 500000 fd off listen-only off loopback off
sudo ip link set "$HOST_CAN" up
ip -details -statistics link show "$HOST_CAN"
```

両方のbitrateが500000で、インターフェースがUPになったことを確認します。`ERROR-ACTIVE` は正常に通信できるCANの状態名で、それだけで通信エラーを意味しません。

この2台だけの試験では双方がACKを返す必要があるため、listen-onlyを無効にします。内部ループバックも無効にします。

## 7. Ubuntu → AGLを試す

**先にAGL** で受信を開始します。15秒以内にUbuntu側の送信を実行してください。

```bash
timeout 15s candump -L -n 6 "$GUEST_CAN"
```

**Ubuntu:** 標準ID3フレーム、拡張ID3フレームを送信します。

```bash
for frame in \
  5A0#0001020304050607 \
  5A0#1122334455667788 \
  5A0#FFEEDDCCBBAA9988 \
  18FF0101#0123456789ABCDEF \
  18FF0101#8000000000000001 \
  18FF0101#0000000000000000
do
  cansend "$HOST_CAN" "$frame"
  sleep 0.1
done
```

AGLで6行表示され、candumpが終了することを確認します。時刻を除いたID・データ・順序が一致すれば成功です。cansendは3桁のIDを標準ID、8桁を拡張IDとして扱います。送信コマンドの成功だけでは受信成功とは判定しません。

```text
(時刻) can1 5A0#0001020304050607
(時刻) can1 5A0#1122334455667788
(時刻) can1 5A0#FFEEDDCCBBAA9988
(時刻) can1 18FF0101#0123456789ABCDEF
(時刻) can1 18FF0101#8000000000000001
(時刻) can1 18FF0101#0000000000000000
```

## 8. AGL → Ubuntuを試す

今度は **先にUbuntu** で受信を開始します。

```bash
timeout 15s candump -L -n 6 "$HOST_CAN"
```

**AGL:**

```bash
for frame in \
  5A0#0001020304050607 \
  5A0#1122334455667788 \
  5A0#FFEEDDCCBBAA9988 \
  18FF0101#0123456789ABCDEF \
  18FF0101#8000000000000001 \
  18FF0101#0000000000000000
do
  cansend "$GUEST_CAN" "$frame"
  sleep 0.1
done
```

Ubuntuで同じ6個のID・データが得られることを確認します。送信側自身で見えるSocketCANのローカルエコーではなく、**反対側のOSで受信すること**が判定条件です。

## 9. カウンターと結果を記録する

**AGL:**

```bash
ip -details -statistics link show "$GUEST_CAN"
```

**Ubuntu:**

```bash
ip -details -statistics link show "$HOST_CAN"
```

双方でTXが6フレーム増え、RXが増え、`bus-errors` / `error-pass` / `bus-off` とRX・TXのerrorsが増えていないことを確認します。ドライバーの送信エコーがRX統計に含まれる場合があるため、RX統計だけで相手からの受信数を判断しません。

検証時は各方向6フレームが完全一致し、両方のTXは6、RX統計は12、通信エラーと送受信エラーカウンターは0でした。CANコントローラーのエラーカウンターも `tx 0 rx 0` でした。

| 症状 | 確認事項 |
|---|---|
| QEMUへの追加時にPermission denied | 対象 `/dev/bus/usb/...` の番号・アクセス権 |
| AがUbuntuのCANに残る | QEMUモニターのエラー、指定機器、他プロセスによる使用 |
| AGLのlsusbには出るが物理CANがない | dmesg、対応ドライバー。仮想can0と区別 |
| Network is down | 送受信する両方をUPにしたか |
| 受信0件 / タイムアウト | 配線、終端、速度、受信開始の順序、インターフェース名 |
| ERROR-PASSIVE / BUS-OFF | 配線・速度・終端・ACK不足。原因を直してからdown/up |
| 同じVID:PIDの違う機器を渡した | USBバス番号・デバイス番号を再確認 |

日時、A/Bの識別、両OSのインターフェース名、ビットレート、受信した6行、試験前後のエラー数を記録します。

## 10. 終了と次回の再開

**AGL:**

```bash
ip link set "$GUEST_CAN" down
```

**Ubuntu:**

```bash
sudo ip link set "$HOST_CAN" down
```

AをUbuntuへ戻す場合は **QEMUモニター** で実行します。

```text
device_del usbcan
```

Ubuntuの `lsusb -t` と `ip -details link show type can` で再認識を確認します。名前が変わる場合があります。QEMUを終了する場合はUbuntuのリポジトリ内で `bash scripts/ssh.sh poweroff` を使います。モニターの `quit` はゲストの正常終了を待たないため、通常の終了には使いません。

WindowsへUSBを戻す場合は、QEMUから解放した後、PowerShellでA・Bそれぞれに `usbipd detach --busid 対象BUSID` を実行します。

USBパススルー追加とCANのUP設定は次回起動時に自動再現されません。再起動・再接続後は番号を確認して手順をやり直します。QEMUモニターへの入力はUbuntuのシェルコマンドとは別です。

## 参照先

- [QEMU：USBパススルーとdevice_add](https://www.qemu.org/docs/master/system/devices/usb.html)
- [Microsoft：USBデバイスをWSLへ接続](https://learn.microsoft.com/ja-jp/windows/wsl/connect-usb)
- [Linux：SocketCAN](https://docs.kernel.org/networking/can.html)
- [can-utils：candump / cansend](https://github.com/linux-can/can-utils)

この試験の到達点はUbuntuとAGLのSocketCAN間です。KUKSAやメーター表示へ進む際は、CANプロバイダーの接続先をAGLの実機CAN名へ合わせ、[VESC回転数表示手順](VESC_RPM.md)のDBC・信号対応を設定します。
