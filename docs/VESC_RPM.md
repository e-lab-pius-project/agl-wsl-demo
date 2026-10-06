# VESCのCAN回転数をAGLのメーターへ表示する

[起動ガイド](../README.md) · [USB-CAN接続](USB_CAN.md) · [メーターパネル開発](METER_DEVELOPMENT.md)

対象は配布イメージ `agl-20261001`、VESCのClassical CAN、モーター1台です。VESCが送る **電気的回転数（ERPM）を極対数で割り、モーター軸のRPMとして表示**します。

データの経路は `VESC → USB-CAN → WSL → cannelloni → AGL can0 → kuksa-can-provider → KUKSA → Flutter` です。まずVESCの代わりにWSLの仮想CANで試し、受信と換算を確認してから実機へ切り替えます。

配布イメージにはCAN受信・KUKSAの部品があります。ただし、VESC用のDBC・対応設定と、電動モーターのRPMを読む画面変更は必要です。現在のIVIホーム画面に、この設定だけで回転計が現れるわけではありません。

**検証状況（2026-10-06）:** 本書の設定例を配布版のQEMU上で実行し、WSLの仮想CANからKUKSAまで、0 / 3000 / -3000 / 約0.142857 rpmの更新を確認しました。USB-CAN・VESC実機の受信とFlutter画面の変更・ビルド・表示は未検証です。

## 1. 実機の値を記録する

| 項目 | 本書の模擬試験例 | 実機で確認する内容 |
|---|---|---|
| VESC ID | 1 | VESC Toolで設定した送信元ID |
| CANモード | VESC | 他のCANプロトコルではないこと |
| CANビットレート | 500 kbit/s | バス上の機器で一致させる。vcanには設定不要 |
| STATUS送信周期 | 10 Hz（100 ms） | STATUSを含む送信グループの設定 |
| モーターの磁極数 | 14極 | モーターの仕様・設定で確認 |
| 極対数 | 7 | 磁極数 ÷ 2。PIUSの確定値ではない |
| 表示対象 | モーター軸RPM | 車輪RPM・車速とは区別する |

VESC Toolとファームウェアの版も記録します。実機に合わせてID・極対数を決めるまで、本書の例のまま車両へ適用しないでください。

## 2. VESCが送るフレームを理解する

VESCの `CAN_PACKET_STATUS` はコマンド番号9です。29ビット拡張IDは `(9 << 8) | VESC_ID` で、ID 1の例は **`00000901`** です。STATUSでは下位8ビットが送信元VESCを表します。

| バイト（0始まり） | 内容 | 読み方 |
|---|---|---|
| 0〜3 | ERPM | 符号付き32ビット、ビッグエンディアン |
| 4〜5 | モーター電流 | 符号付き16ビット、ビッグエンディアン、値 ÷ 10 A |
| 6〜7 | Duty比 | 符号付き16ビット、ビッグエンディアン、値 ÷ 1000 |

本書は最初の4バイトだけを表示に使います。例えば `00 00 52 08 00 00 00 00` の先頭4バイトは21000です。

```text
モーター軸RPM = ERPM / 極対数
21000 / 7 = 3000 rpm
```

負のERPMは負のRPMとして保持します。正転・逆転と車両の進行方向の対応は実機で確認します。極対数7の場合にERPMをそのままRPMとして表示すると、表示が7倍になります。

STATUSの受信に `CAN_PACKET_SET_RPM` は使いません。SET_RPMは回転指令です。STATUSの周期送信をVESC側で有効にして受信します。仕様の出典は [VESC CAN通信仕様](https://github.com/vedderb/bldc/blob/master/documentation/comm_can.md) と [送受信の実装](https://github.com/vedderb/bldc/blob/master/comm/comm_can.c) です。

## 3. AGLと仮想CANを準備する

[起動ガイド](../README.md)でAGLを起動し、[仮想CAN手順](CAN.md)の1〜3節を実施します。車速試験の `PASS` を確認したら、同じトンネルでVESC形式のフレームを試します。

**この段階ではUSB-CANを接続せず、物理CANへのトンネルを停止してください。** cannelloniは双方向なので、模擬データを実際の車両バスへ流さない構成で作業します。

**WSL、リポジトリ内:**

```bash
bash scripts/ssh.sh
```

以降、5節までのコマンドは **SSHで入ったAGL内** で実行します。

```bash
systemctl is-active kuksa-databroker kuksa-can-provider
ip -details link show can0
python3 -c 'import cantools; print(cantools.__version__)'
```

両サービスがactive、can0がvcanであることを確認します。配布イメージにはPythonとcantoolsも含まれています。

## 4. VESC用のDBCとVSS対応を作る

このイメージの `/usr/share/vss/vss.json` は `vss_6.0-agl.json` へのシンボリックリンクです。次の信号が既に定義されています。

| パス | 型 | 単位 |
|---|---|---|
| `Vehicle.Powertrain.ElectricMotor.Speed` | float | rpm |

既存ファイルをコピーし、この信号にCANからの入力対応を追加します。KUKSAへの新しい信号登録や独自受信プログラムは、この1項目には不要です。

**AGL:** 以下はID 1、極対数7の模擬試験用です。実機へ切り替える際は冒頭の2変数を実機に合わせて再実行します。

```bash
python3 - <<'PY'
import json
from pathlib import Path
import cantools

vesc_id = 1
pole_pairs = 7
assert isinstance(vesc_id, int) and 0 <= vesc_id < 255
assert isinstance(pole_pairs, int) and pole_pairs > 0

base = Path('/etc/kuksa-can-provider')
frame_id = (9 << 8) | vesc_id
# DBCは最上位ビットで拡張IDを表す。CANバス上のID自体はframe_id。
dbc_id = 0x80000000 | frame_id
factor = 1 / pole_pairs
addition = f'''
BO_ {dbc_id} VESC_Status: 8 VESC
 SG_ VESC_MotorRPM : 7|32@0- ({factor:.17g},0) [{-2147483648 * factor:.17g}|{2147483647 * factor:.17g}] "rpm" AGL
'''
dbc_text = Path('/usr/share/dbc/agl-vcar.dbc').read_text() + addition
db = cantools.database.load_string(dbc_text, database_format='dbc')
message = db.get_message_by_name('VESC_Status')
assert message.is_extended_frame and message.frame_id == frame_id
for erpm in (0, 21000, -21000, 1):
    payload = erpm.to_bytes(4, 'big', signed=True) + bytes(4)
    rpm = message.decode(payload)['VESC_MotorRPM']
    assert abs(rpm - erpm / pole_pairs) < 0.000001

tree = json.loads(Path('/usr/share/vss/vss.json').read_text())
node = tree['Vehicle']
for name in ('Powertrain', 'ElectricMotor', 'Speed'):
    node = node['children'][name]
assert node['datatype'] == 'float' and node['unit'] == 'rpm'
node['dbc2vss'] = {'signal': 'VESC_MotorRPM', 'interval_ms': 100}
node.pop('vss2dbc', None)  # RPMからCANへの逆方向マッピングは作らない。

(base / 'vesc.dbc').write_text(dbc_text)
(base / 'vesc-vss.json').write_text(json.dumps(tree, indent=2) + '\n')
print(f'PASS: extended ID={frame_id:08X}, pole pairs={pole_pairs}')
PY
```

DBCの `7|32@0-` は、先頭バイトの最上位ビットから始まる32ビット・ビッグエンディアン・符号付き信号です。倍率 `1 / pole_pairs` によって、CANプロバイダーがERPMをRPMへ換算します。Flutterで再び極対数で割らないでください。

元のAGLデモDBCも残すので、既存の車速試験を続けられます。この例は特定のVESC IDを1台だけ読みます。複数台を扱う場合は信号名・VSSパス・表示対象を台ごとに決め、同じパスへ複数台の値を上書きしない構成にします。

## 5. CANプロバイダーへ設定を反映する

**AGL:** 元の設定を一度だけ退避し、DBCと対応表のパスを変更します。接続先、TLS、トークンなどは保持します。

```bash
python3 - <<'PY'
import configparser
import shutil
from pathlib import Path

path = Path('/etc/kuksa-can-provider/config.ini')
backup = path.with_name('config.ini.before-vesc')
if not backup.exists():
    shutil.copy2(path, backup)
config = configparser.ConfigParser()
config.read(path)
config['general']['mapping'] = '/etc/kuksa-can-provider/vesc-vss.json'
config['can']['dbcfile'] = '/etc/kuksa-can-provider/vesc.dbc'
with path.open('w') as output:
    config.write(output)
PY
systemctl restart kuksa-can-provider
systemctl is-active kuksa-can-provider
journalctl -u kuksa-can-provider -n 40 --no-pager
```

activeを確認し、DBCの読み込み、信号名、認証にエラーがないことを確認します。ブローカーの元のVSS定義は変更していないため、この手順では `kuksa-databroker` の再起動は不要です。

## 6. 仮想CANから既知の回転数を流す

**WSL、別のターミナル:** 約5秒間、10 Hzで21000 ERPMを送ります。

```bash
ip -details link show vcan_agl | grep -qw vcan &&
  cangen vcan_agl -e -g 100 -n 50 -I 00000901 -L 8 -D 0000520800000000
```

`-e` は拡張ID指定です。VESC IDを変えた場合は送信IDも変更します。

**AGL:** 次のコマンドでKUKSAを読みます。SSHの対話ターミナルで実行してください。

```bash
TERM=xterm databroker-cli --server https://localhost:55555 \
  --ca-cert /etc/kuksa-val/CA.pem \
  --token-file /etc/kuksa-can-provider/can-provider.token \
  get Vehicle.Powertrain.ElectricMotor.Speed
```

期待値は **3000 rpm** です。同じ方法で次の値を試します。表のデータを上の `-D` に指定します。

| 送信データ（16進、8バイト） | ERPM | 極対数7の期待値 |
|---|---:|---:|
| `0000000000000000` | 0 | 0 rpm |
| `0000520800000000` | 21000 | 3000 rpm |
| `FFFFADF800000000` | -21000 | -3000 rpm |
| `0000000100000000` | 1 | 約0.142857 rpm |

データが届かないときは、AGLで `candump can0` を実行して `00000901` と8バイトのデータを確認します。終了はCtrl+Cです。KUKSAの値が読み出せても、最後の受信値が残っているだけの場合があります。異なる値を順番に送って更新を確かめます。

## 7. Flutterの購読と表示を変更する

ここからは [メーターパネル開発手順](METER_DEVELOPMENT.md)の5〜7節を使い、公式 `flutter-instrument-cluster` の固定コミット `6c204692c9ee5ff2c1f20134d55223d2312c9be1` を変更・ビルド・配置します。**画面アプリのビルドと表示試験は本書作成時点では未実施です。**

### 7.1 購読先を電動モーターへ変更する

`lib/vehicle-signals/vss_path.dart` の `vehicleEngineRPM` を変更します。

```dart
static const String vehicleEngineRPM =
    'Vehicle.Powertrain.ElectricMotor.Speed';
```

元の購読先は `Vehicle.Powertrain.CombustionEngine.Speed` です。変数名を残すことで既存の購読リストと回転計への接続を利用します。

### 7.2 受信データ型も変更する

`lib/vehicle-signals/vss_provider.dart` の該当caseを次へ置き換えます。元の処理は `uint32` を読みますが、電動モーターのSpeedは **float** です。パスだけを変えても表示は更新されません。

```dart
case VSSPath.vehicleEngineRPM:
  final value = update.entry.value;
  if (value.hasFloat() && value.float.isFinite) {
    vehicleStatus.update(rpm: value.float.round());
  }
  break;
```

既存の画面状態はintなので、この最小変更では表示用に四捨五入します。KUKSAには小数を保持します。小数表示が必要なら状態のrpmもdoubleへ変更します。

### 7.3 数値・針・未受信表示を調整する

まず受信した符号付きRPMを数値で表示し、0 / 3000 / -3000を確認します。針の目盛りが正の範囲だけなら、針は絶対値で描き、方向を別表示するなど仕様を決めます。値を針の範囲へ収めても、元の数値の符号を失わないようにします。ラベルはモーター回転数とrpmにし、車速のkm/hと取り違えないでください。

公式デモの `vehicle_status_provider.dart` には初期値 `rpm: 7000` があります。実機向けの表示では、受信前に7000や0を実測値として表示せず、未受信状態を用意して `--` などを表示します。

CAN停止時の処理は別途実装が必要です。**このDBC追加だけでは、切断時に最後のRPMが残ります。** CAN受信時刻・有効性をプロバイダーから画面へ渡し、例えば10 Hz送信に対し1秒受信なしで「通信断」とする仕様を決めます。一定RPMでは値変更通知が来ない場合があるため、Flutterの購読イベントが来ないことだけをタイムアウト条件にしないでください。

TLS・CA・アプリ用トークンは既存の設定を使い、電動モーターSpeedの読み取り権限を確認します。CANプロバイダーの書き込み権限を持つトークンを、そのまま画面アプリへ流用しません。

変更後はメーターパネル開発手順のbuild → deploy → サービス再起動を行います。6節の模擬データで、KUKSA値と画面表示が一致することを確認します。

## 8. 実際のVESCへ切り替える

1. 仮想CANの送信を終了し、[CAN.mdの停止手順](CAN.md#4-停止)で両側のトンネルを止めます。
2. VESC Toolで既存設定を読み出して保存し、App Settings → GeneralでCAN Mode、VESC ID、ビットレートを確認します。必要な変更だけを保存します。
3. CAN Status Messagesの送信グループに **STATUS（番号9）** を含め、送信周期を設定します。項目名は版により異なります。本書の例は10 Hzです。
4. [USB-CAN接続手順](USB_CAN.md)で配線・終端・usbipd・SocketCANを準備します。listen-onlyではACKを返さないため、VESCとアダプターだけの場合のACK不足にも注意します。
5. WSLの物理CANで下の受信確認を行います。その後USB-CAN手順に従い、物理インターフェースをcannelloniへ接続します。
6. 4節の `vesc_id` と `pole_pairs` を実機に合わせて設定を作り直し、5節でCANプロバイダーを再起動します。
7. AGLのcandump、KUKSA、Flutterの順に同じ値が届くことを確認します。VESC Toolの表示がERPMか機械RPMかも確認して比較します。

**WSL、物理CANの受信確認:**

```bash
CAN_IF=can0  # USB-CAN手順で確認したWSL側の名前
ip -details -statistics link show "$CAN_IF"
candump -tz "$CAN_IF"
```

ID 1の例では `00000901 [8] ...` が周期的に見えることを確認します。このコマンドは受信だけです。**6節のcangenを物理CANへ向けて実行しないでください。** 初回は停止状態で受信を確認し、回転試験は車両・モーターの既定の試験手順で行います。

モーター軸RPMから車速を作る場合は減速比と車輪周長も必要です。減速比を「モーターRPM ÷ 車輪RPM」と定義すると、`車速[km/h] = モーターRPM / 減速比 × 車輪周長[m] × 60 / 1000` です。RPMをそのまま `Vehicle.Speed` へ書き込まないでください。

## 9. 切り分けと元に戻す方法

| 症状 | 確認箇所 |
|---|---|
| WSLでSTATUSが見えない | VESCモード、STATUS送信設定、ID、ビットレート、配線、終端、ACK |
| WSLで見えるがAGLで見えない | cannelloniの使用インターフェース、TCP接続 |
| AGLで見えるがKUKSAが変わらない | 拡張ID、VESC ID、DBC、mapping設定、プロバイダーのログ |
| KUKSAが7倍になる | 極対数7の場合の換算漏れ。実際の極対数で確認 |
| マイナスが巨大な正数になる | 符号付き32ビット・ビッグエンディアンとして読んでいるか |
| KUKSAは更新するが画面が変わらない | ElectricMotor.Speedの購読、floatの取得、読み取り権限 |
| 停止・切断しても表示が残る | 受信時刻・有効性の監視。0と未受信は別状態 |

**AGL、CANプロバイダーを元へ戻す場合:**

```bash
cp -p /etc/kuksa-can-provider/config.ini.before-vesc \
  /etc/kuksa-can-provider/config.ini
systemctl restart kuksa-can-provider
systemctl is-active kuksa-can-provider
```

Flutterの変更は別に戻して再ビルド・配置します。トンネルの停止・USBの切断は [USB-CAN手順](USB_CAN.md)を参照してください。元に戻した後、仮想CAN構成で `bash scripts/check-can.sh` が通ることを確認します。

## 10. 完了条件と参照資料

- 模擬データの0 / 21000 / -21000 ERPMが、極対数7で0 / 3000 / -3000 rpmになる。
- 実機のID・極対数・ビットレートで同じ換算が成り立つ。
- KUKSA値と画面表示が一致し、逆転時も数値を誤表示しない。
- 起動直後、CAN切断、再接続、一定RPMの連続受信をそれぞれ確認する。
- 使用イメージ、VESCファームウェア、設定値、DBC・VSS対応、画面の変更、試験結果を保存する。トークンの内容はGitへ保存しない。

参照資料（確認日: 2026-10-06）:

- [VESC CAN通信仕様](https://github.com/vedderb/bldc/blob/master/documentation/comm_can.md)
- [VESC CAN送受信ソース](https://github.com/vedderb/bldc/blob/master/comm/comm_can.c)
- [KUKSA CAN Provider](https://github.com/eclipse-kuksa/kuksa-can-provider)
- [cantoolsのDBC処理](https://cantools.readthedocs.io/en/latest/)
- [固定版のFlutterメーター](https://git.automotivelinux.org/apps/flutter-instrument-cluster/tree/?id=6c204692c9ee5ff2c1f20134d55223d2312c9be1)
- [配布イメージと元の設定・ソース情報](../images/README.md)
