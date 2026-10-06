# USB-CANアダプターをWSL・AGLへ接続する

[起動ガイド](../README.md) · [仮想CAN試験](CAN.md) · [メーター開発](METER_DEVELOPMENT.md)

対象: Windows 11 / WSL2 / Ubuntu 24.04 / 本リポジトリのQEMU版AGL。確認日: 2026-10-06。

## 1. 接続の構成と確認範囲

**CANバス → USB-CAN → WindowsのUSB/IP転送 → WSLのSocketCAN → cannelloni → QEMU内のAGL → KUKSA → メーター** の順につなぎます。

USBデバイスを受け取るのはWSLです。QEMUへUSBを直接割り当てる設定は不要です。QEMU内では既存の仮想CAN `can0` を使います。

| 場所 | この手順での名前 | 実体 |
|---|---|---|
| Windows | usbipdのBUSID | 接続したUSBデバイスを識別 |
| WSL | 例: `can0`。以下では変数 `CAN_IF` に設定 | USB-CANが作る物理CANインターフェース |
| AGL | `can0` | AGL内の仮想CAN。WSLのcan0とは別 |
| AGL | `Vehicle.Speed` など | KUKSAが配信する信号 |

**WSLのcan0にビットレートを設定します。AGLの仮想can0には設定しません。**

確認済みなのは、仮想CANからAGL・KUKSAまでの接続と、確認PCのWSLに `gs_usb` / `peak_usb` / `kvaser_usb` モジュールが存在することです。今回はアダプターが未接続のため、**この手順全体のUSB-CAN実機試験は未実施**です。

## 2. アダプターと配線を確認する

### 対応方式

この手順は、LinuxでSocketCANインターフェースを作れるアダプターを対象にしています。商品名だけでなく、型番とファームウェアの方式を確認します。

| アダプターの方式・例 | Linux側 | 注意点 |
|---|---|---|
| CANnectivity / gs_usb互換 / candleLight系 | `gs_usb` | 機器のファームウェアとドライバーの対応を確認 |
| LinuxでサポートされるPEAK USB製品 | `peak_usb` | 型番ごとの対応を確認 |
| LinuxでサポートされるKvaser USB製品 | `kvaser_usb` | 型番ごとの対応を確認 |
| SLCAN方式 | シリアルデバイス＋slcand等 | USB接続だけではcan0にならない。この文書の直接接続手順とは別 |
| 独自DLL・専用APIのみの製品 | メーカー依存 | SocketCAN対応を確認してから使用 |

CANnectivityはUSB-CAN用のファームウェアで、Linuxの `gs_usb` を利用できます。[CANnectivity公式](https://github.com/CANnectivity/cannectivity)

USB-UART変換器がCOMポートとして見えるだけでは、CANアダプターとは判断できません。例えばCH340という表示だけではCAN対応を確認できません。

### 接続前に決める値

- **ビットレート**: 相手の仕様から確認。この文書では Classical CAN の **500 kbit/s** を例にします。
- **CANの種類**: Classical CAN / CAN FD。FDの場合は仲裁側とデータ側の速度を確認します。
- **コネクターの端子**: メーカー資料でCAN_H、CAN_L、必要な基準GNDを確認します。端子番号は製品ごとに確認します。
- **終端**: 通常の高速CANバスは両端を120 Ωで終端します。既存バスに終端がある場合は、アダプターの内蔵終端を追加しないよう確認します。
- **相手ノード**: 受信専用の観測では、既存の通信が成立しているバスを使います。

配線・終端・ビットレートはUSB/IPやQEMUでは補えません。まず机上の試験バスで確認します。

## 3. WindowsからUSBをWSLへ渡す

[Microsoft公式手順](https://learn.microsoft.com/ja-jp/windows/wsl/connect-usb)に基づき、usbipd-win 5.x以降のコマンドを使います。確認PCでは5.3.0でした。

### 3.1 初回のインストール

**管理者PowerShell:**

```powershell
winget install --interactive --exact dorssel.usbipd-win
wsl --update
```

既に導入済みなら再インストールは不要です。必要ならPowerShellを開き直し、版を確認します。

```powershell
usbipd --version
wsl -l -v
```

対象UbuntuがWSL2であることを確認します。

### 3.2 Ubuntuを起動しておく

**別のPowerShell:**

```powershell
wsl -d Ubuntu-24.04
```

Ubuntuのターミナルは開いたままにします。既存環境の名前が `wsl-agl` なら読み替えてください。

### 3.3 USBデバイスを共有する

USB-CANを挿し、Windows側のCANアプリを閉じます。

**管理者PowerShell:**

```powershell
usbipd list
$canBusId = Read-Host 'USB-CANのBUSIDを入力（例: 4-4）'
usbipd bind --busid $canBusId
usbipd list
```

機器名とVID:PIDから対象を確認し、Sharedになったことを確認します。BUSIDはこのPCで表示された値を使います。過去の実験で使ったBUSIDをそのまま流用しません。

### 3.4 WSLへ接続する

**通常のPowerShell:**

```powershell
usbipd list
$canBusId = Read-Host '共有したUSB-CANのBUSIDを入力'
usbipd attach --wsl Ubuntu-24.04 --busid $canBusId
usbipd list
```

AttachedになればUSB/IP接続は完了です。この間、同じデバイスをWindowsのCANアプリから同時には使えません。ディストリビューション指定は接続に使用するWSL環境の選択であり、そのディストリビューションだけへのアクセス隔離ではありません。

USBを抜き差しした後やWSLを再起動した後は、BUSIDと状態を再確認し、必要ならattachをやり直します。

### bindがUSBフィルターとの競合で失敗する場合

まずWindows側の利用アプリを終了し、[usbipd-winのWSLサポート資料](https://github.com/dorssel/usbipd-win/wiki/WSL-support)を確認します。`--force` は通常手順には使いません。ツールがフィルター競合を報告し、対象USB-CANを確認できた場合の対処例は以下です。

**管理者PowerShell:**

```powershell
usbipd bind --busid $canBusId --force
```

force共有ではWindows側の利用が制限されます。終了後にWindowsへ戻すためのunbind手順は後述します。

## 4. WSLでドライバーとCANインターフェースを確認する

**WSLのUbuntu:**

```bash
sudo apt update
sudo apt install -y usbutils can-utils iproute2
uname -r
lsusb
sudo dmesg | tail -n 60
ip -brief link
```

CANnectivity / gs_usb互換機器の場合:

```bash
modinfo gs_usb
sudo modprobe gs_usb
lsusb -t
ip -details link show type can
```

別方式なら対応するドライバーへ読み替えます。`modinfo` 成功はドライバーの存在確認であって、実機の認識成功ではありません。

一覧からUSB-CANに対応するインターフェースを選びます。複数台・複数チャンネルの場合、can0とは限りません。

```bash
CAN_IF=can0
readlink -f "/sys/class/net/$CAN_IF/device"
readlink -f "/sys/class/net/$CAN_IF/device/driver"
ip -details link show "$CAN_IF"
```

`lsusb` に機器があるのにCANインターフェースがない場合は、`dmesg` とドライバーの対応を確認します。ドライバーがない場合はAGL側ではなく **WSLカーネル側** の問題です。まずWSLを更新して再確認します。

## 5. 受信専用で通信を確認する

### Classical CANの例

**WSL、同じターミナル:**

```bash
CAN_IF=can0
CAN_BITRATE=500000
sudo ip link set dev "$CAN_IF" down
sudo ip link set dev "$CAN_IF" type can bitrate "$CAN_BITRATE" listen-only on
```

上の設定が成功した場合だけ、インターフェースを有効にします。

```bash
sudo ip link set dev "$CAN_IF" up
ip -details -statistics link show "$CAN_IF"
candump -tz "$CAN_IF"
```

詳細表示の `LISTEN-ONLY` と `bitrate 500000` を確認します。`candump` は Ctrl+C で終了します。

**listen-onlyは機器・ドライバーが対応している場合に使えます。** `Operation not supported` 等で失敗した場合は、そのままupせず、対応機器を使うか、管理された試験バスで通常モードを使用します。受信専用モードではACKも返さないため、送信ノードとこのアダプターだけの2台構成ではACK不足になります。ACKを返す別の通常ノードが必要です。

### 通常モードを使う机上試験

相手ノードとの通信を成立させる必要があり、送信が許される試験バスでは、downの状態で次を設定します。

```bash
sudo ip link set dev "$CAN_IF" type can bitrate "$CAN_BITRATE" listen-only off
sudo ip link set dev "$CAN_IF" up
```

これは受信専用ではありません。後段のcannelloniを接続するとAGL側からのフレームも物理バスへ出せる構成になります。

### CAN FDの場合

この文書の基本例はClassical CANです。CAN FD対応アダプター・バスで、仲裁500 kbit/s、データ2 Mbit/sと確認できている場合の設定例:

```bash
sudo ip link set dev "$CAN_IF" down
sudo ip link set dev "$CAN_IF" type can bitrate 500000 dbitrate 2000000 fd on listen-only on
```

成功後にupし、詳細表示を確認します。速度は例であり、実バスに合わせます。FDフレームの転送とKUKSAでのデコードまでを、この環境で検証したわけではありません。Classical CANへ戻す場合はdownの後で `fd off` を明示して設定し直します。

## 6. WSLのUSB-CANをQEMU内のAGLへ接続する

### 6.1 前提と仮想CAN試験からの切り替え

[CAN.md](CAN.md) の「WSL側の準備」に従ってcannelloniを導入し、QEMUでAGLを起動しておきます。**今回はvcan_aglの代わりに物理CANを使います。**

既存のトンネルがあれば、**WSL、agl-wsl-demoのリポジトリ内**で停止します。

```bash
bash scripts/ssh.sh systemctl stop agl-can-guest
sudo systemctl stop agl-can-host
```

未作成のサービスに対する `Unit not loaded` は、停止対象がないことを示します。

**cannelloniは双方向です。** AGL側のcan0に送信されたフレームもWSLへ戻ります。物理バスを観測する構成では、前節のlisten-onlyを確認します。AGL内のシミュレーターやテスト送信プログラムも停止し、仮想試験用の `check-can.sh` をこの接続で実行しないでください。

### 6.2 WSL側のサーバーを開始する

**WSL:**

```bash
CAN_IF=can0
ip -details -statistics link show "$CAN_IF"
sudo systemd-run --unit=agl-can-host --collect \
  /usr/local/bin/cannelloni -C s -I "$CAN_IF" -L 127.0.0.1 -l 20000 -p
```

`CAN_IF` は確認したUSB-CANの名前です。127.0.0.1だけで待ち受けます。`-p` は今回の接続で必要だったpeerチェック解除です。

### 6.3 AGL側のクライアントを開始する

**WSLのリポジトリ内:**

```bash
bash scripts/ssh.sh
```

ここからは **AGL** です。

```bash
ip -details link show can0
systemd-run --unit=agl-can-guest --collect \
  /usr/bin/cannelloni -C c -I can0 -R 10.0.2.2 -r 20000
journalctl -u agl-can-guest -n 30 --no-pager
candump -tz can0
```

AGLの `can0` はvcanです。物理側のビットレートをこのcan0に設定しません。`10.0.2.2` はQEMUのユーザーネットワークから見たホストで、今回の起動スクリプトでは追加ポート転送は不要です。

WSLでも `ss -tn | grep ':20000'` と `journalctl -u agl-can-host` を確認します。物理送信元から出た同じCAN ID・データが、WSLとAGLの両方のcandumpに出れば、フレーム転送まで確認できています。

## 7. KUKSAとメーターまで確認する

**AGL:**

```bash
systemctl is-active kuksa-can-provider kuksa-databroker
journalctl -u kuksa-can-provider -n 60 --no-pager
TERM=xterm databroker-cli \
  --server https://localhost:55555 \
  --ca-cert /etc/kuksa-val/CA.pem \
  --token-file /etc/kuksa-can-provider/can-provider.token \
  get Vehicle.Speed
```

これはAGLにSSHで対話ログインしたターミナルから実行します。トークンの内容を表示する必要はありません。

**CANが届いても、DBCに定義のないフレームは自動で車速になりません。** 配布版はAGLデモ用のDBCです。PIUSでは次を合わせます。

- CAN ID、標準/拡張フレーム、DLC。
- ビット位置、符号、バイト順、倍率、オフセット。
- 変換後の単位とVSSパス。
- CANプロバイダーの設定と対応付け。

設定の場所は `/etc/kuksa-can-provider/config.ini`、`/usr/share/dbc/agl-vcar.dbc`、`/usr/share/vss/vss.json` です。[メーター開発手順](METER_DEVELOPMENT.md)も参照してください。

例えば標準ID `0x3E9` のデモ信号を出す机上の送信元なら、既存のデモ定義で確認できます。PIUS固有IDを同じ意味と仮定してはいけません。KUKSAの値が変わった後に、メーター画面の値が一致することを確認します。

## 8. 終了してWindowsへ戻す

まずcandumpを Ctrl+C で終了し、AGLのSSHを `exit` で抜けます。

**WSL、リポジトリ内:**

```bash
bash scripts/ssh.sh systemctl stop agl-can-guest
sudo systemctl stop agl-can-host
CAN_IF=can0
sudo ip link set dev "$CAN_IF" down
```

**PowerShell:**

```powershell
usbipd list
$canBusId = Read-Host '切り離すUSB-CANのBUSIDを入力'
usbipd detach --busid $canBusId
```

共有登録も解除する場合、特に `bind --force` を使った場合は **管理者PowerShell** で実行します。

```powershell
$canBusId = Read-Host '共有解除するUSB-CANのBUSIDを入力'
usbipd unbind --busid $canBusId
usbipd list
```

USBを抜いただけでは共有登録が残る場合があります。QEMUも終了するなら、WSLのリポジトリ内で `bash scripts/ssh.sh poweroff` を実行します。

仮想CAN試験へ戻すときは、物理CAN用のトンネルを停止した状態で [CAN.md](CAN.md) のvcan_agl接続を作り直します。

## 9. トラブルの切り分け

| 症状 | 確認すること |
|---|---|
| usbipdに機器が出ない | USBケーブル、給電、Windowsのデバイス認識 |
| Persistedにだけある | 以前の共有記録。現在Connectedにあることを確認 |
| attachが失敗する | Sharedか、WSL2が起動中か、Windowsのアプリが使用していないか |
| WSLのlsusbにはあるがcan0がない | 対応ドライバー、機種、ファームウェア、dmesg |
| modprobeがModule not found | WSLカーネルとそのモジュール。AGLのモジュールとは別 |
| Operation not supported | listen-onlyやCAN FDなど、その機器が未対応の設定 |
| candumpに何も出ない | 相手が送っているか、速度、配線、終端、CANの種類 |
| ERROR-PASSIVE / BUS-OFF | 速度・配線・終端・ACK不足。原因を直してからdown/up |
| WSLでは見えるがAGLでは見えない | トンネルの接続先、サービス、TCPの20000番 |
| AGLで見えるがKUKSAが変わらない | CANプロバイダー、DBC、VSS対応、標準/拡張ID |
| KUKSAは変わるが画面が変わらない | メーターアプリ、購読パス、TLS・認証 |
| 抜き差し後に止まった | attach、インターフェース名、ビットレート、トンネルを再確認 |

この構成は開発・表示試験用です。USB/IPとTCP転送の遅延が入るため、実機CANのタイミング保証を確認する試験とは分けて扱います。

## 10. 記録する項目と参照先

試験時は、アダプターの型番・ファームウェア、WSLカーネル、ドライバー、ビットレート、Classical/FD、listen-onlyの有無、使用DBC、観測したIDと値を記録します。

- [Microsoft: USBデバイスをWSLへ接続](https://learn.microsoft.com/ja-jp/windows/wsl/connect-usb)
- [usbipd-win: WSL support](https://github.com/dorssel/usbipd-win/wiki/WSL-support)
- [Linux: SocketCAN](https://docs.kernel.org/networking/can.html)
- [CANnectivity](https://github.com/CANnectivity/cannectivity)
- [cannelloni](https://github.com/mguentner/cannelloni)
