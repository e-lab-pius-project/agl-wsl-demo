# メーターパネル開発手順

[起動ガイド](../README.md) · [構成要素](COMPONENTS.md) · [公式リンク一覧](REFERENCES.md) · [CAN試験](CAN.md)

対象: Windows 11 / WSL2 / Ubuntu 24.04 / QEMU、配布済みの AGL 21.93.0 イメージ。調査日: 2026-10-06。

## 到達目標と確認状況

最初の目標は **仮想CANで車速を送り、AGL上のメーターに同じ数値を表示すること** です。その後にPIUS用のデザインと信号対応を追加します。

| 段階 | 完了の判断 | 本リポジトリでの状況 |
|---|---|---|
| AGLの起動 | ホーム画面とSSHが使える | 実機PC上のQEMUで確認済み |
| CAN入力 | KUKSAの車速が 0 → 20 → 42 → 0 になる | 確認済み |
| WSLでの画面試作 | Flutterの画面とWidgetテストが動く | 以下に手順・教材例を掲載。未実行 |
| AGL向けアプリ開発 | 公式メーターをビルド・配置し表示できる | 固定した公式ソースに基づく手順。ビルド・配置は未実行 |
| メーターとの結合 | CAN、KUKSA、画面の値が一致する | 未実施 |
| Raspberry Pi / 実車 | 実際の通信と表示を確認できる | 未実施 |

**手順書を公開したことと、全工程を実行検証済みであることは別です。** この文書ではコマンドの実行場所と、確認が必要な箇所を示します。

## 1. 作業場所を整理する

| 表記 | 実行場所 | 主な作業 |
|---|---|---|
| Windows | PowerShell / エディター | WSL起動、ファイル編集 |
| WSL | Ubuntuのターミナル | Flutterの試作、QEMU、SSH、CAN送信 |
| AGL | `bash scripts/ssh.sh` で入った先 | サービス操作、ログ、表示 |
| ビルドホスト | AGLのYoctoビルド環境を用意したLinux | AGL向けアプリのビルド、OSへの組込み |

RAM 8 GBの学生PCでは、まずWSLでの画面試作と配布イメージでのCAN試験を進めます。Yoctoの初回ビルドは別途用意した十分なメモリ・ディスクのあるPCで行う運用を想定します。[ホスト要件](https://docs.yoctoproject.org/ref-manual/system-requirements.html)を確認してください。8 GB機で全工程を快適に実行できるという検証はしていません。

以降の例はリポジトリを `~/agl-wsl-demo` に置いた場合です。別の場所にcloneした場合は読み替えます。

## 2. まずAGLと車速入力を確認する

**WSL:**

```bash
cd ~/agl-wsl-demo
bash scripts/start.sh
```

別のWSLターミナルで [CAN.md](CAN.md) に従ってトンネルを接続し、試験します。

```bash
cd ~/agl-wsl-demo
bash scripts/check-can.sh
```

`PASS` が出ることを確認します。この時点のIVIホーム画面に速度計がなくても、CAN試験の失敗ではありません。試験はKUKSAの値を検査しています。

## 3. メーターに表示する情報を決める

最初は車速1項目に絞り、次の表を作ってから実装します。

| 項目 | 初期仕様の例 | 決めること |
|---|---|---|
| 車速 | 数値と km/h | 小数桁、表示範囲、単位 |
| 電池残量 | % とバー | PIUS側のデータ定義、欠測時の表示 |
| 方向指示器 | 左右のアイコン | 信号の意味、点滅をどちら側で作るか |
| 警告 | アイコンと短い文言 | 発生・解除条件、優先順位 |
| データ状態 | 接続中 / 未受信 / 通信異常 | 起動直後、切断時、復帰時の動作 |

画面サイズの初期条件は現在のQEMUと同じ **1920×1080** にします。実機の画面サイズが決まったら、その寸法でも確認します。色だけに意味を持たせず、数字・形・文字を併用します。

受信していない値を、停止を意味する「0」として表示しない設計にします。CANが変化時だけ送られるのか周期送信なのかも確認してください。**値が変化しないことだけでは通信断と判定できません。** KUKSAの購読接続状態に加え、送信元の更新周期やハートビートを使って鮮度を判断します。

## 4. WSLでFlutterの画面を試作する

この段階はUbuntu上のデスクトップアプリです。AGL向けの配置は後で行います。QEMUを止めて作業しても構いません。

### 4.1 Flutter SDKを用意する

**WSL:** [FlutterのLinux向け導入手順](https://docs.flutter.dev/install/manual)と[Linux開発環境](https://docs.flutter.dev/platform-integration/linux/setup)に従います。

```bash
sudo apt update
sudo apt install -y curl git unzip xz-utils zip libglu1-mesa \
  clang cmake ninja-build pkg-config libgtk-3-dev libstdc++-12-dev
```

本イメージのEngineは **3.38.3** なので、まず [SDK archive](https://docs.flutter.dev/install/archive) から Linux x64 の同じ版を選び、`~/develop/flutter` に展開します。FlutterはWindows用ではなくWSL内のLinux用を使います。

```bash
export PATH="$HOME/develop/flutter/bin:$PATH"
flutter --version
flutter doctor -v
flutter devices
```

Flutterの版とLinux toolchainを確認します。`flutter devices` にLinuxが出ることが目安です。PATH設定は必要に応じて `~/.bashrc` に追加します。Android向けの警告とLinux向けのエラーは区別してください。

版をそろえるだけでAGLに配置できるわけではありません。AGLのEngineやEmbedderとの組合せは後段のAGLレシピで合わせます。

### 4.2 小さな画面から作る

**WSL:**

```bash
mkdir -p ~/work
cd ~/work
flutter create --platforms=linux pius_meter
cd pius_meter
flutter run -d linux
```

雛形が表示されたら終了し、`lib/main.dart` を次の教材例で置き換えます。

```dart
import 'package:flutter/material.dart';

void main() => runApp(const MaterialApp(home: SpeedPanel(speed: 42)));

class SpeedPanel extends StatelessWidget {
  const SpeedPanel({super.key, required this.speed});
  final double speed;

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      backgroundColor: const Color(0xff101828),
      body: Center(
        child: Column(
          mainAxisSize: MainAxisSize.min,
          children: [
            Text(speed.toStringAsFixed(0),
                style: const TextStyle(fontSize: 120, color: Colors.white)),
            const Text('km/h',
                style: TextStyle(fontSize: 32, color: Colors.white70)),
          ],
        ),
      ),
    );
  }
}
```

`flutter run -d linux` で42 km/hが表示されることを確認します。起動したままソースを変更し、ターミナルで `r` を押すとHot Reloadできます。`main()` 内の初期値変更など、反映されない変更には `R` のHot Restartを使います。

これはレイアウト練習用の固定値です。CANや通信断処理はまだありません。車速の引数を0、20、42に変え、配置・サイズ・色を調整します。円形の針などは、この表示ができてから追加します。

### 4.3 表示をテストする

雛形の `test/widget_test.dart` は元のカウンターアプリ用なので、次で置き換えます。

```dart
import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:pius_meter/main.dart';

void main() {
  testWidgets('車速と単位を表示できる', (tester) async {
    for (final speed in [0.0, 20.0, 42.0]) {
      await tester.pumpWidget(MaterialApp(home: SpeedPanel(speed: speed)));
      expect(find.text(speed.toStringAsFixed(0)), findsOneWidget);
      expect(find.text('km/h'), findsOneWidget);
    }
  });
}
```

**WSL、pius_meter内:**

```bash
dart format lib test
flutter analyze
flutter test
```

ここまでの成果は画面部品のソースです。`build/linux/...` の実行ファイルをそのままAGLへ転送する手順には進みません。

## 5. 公式メーターのソースを読む

既存の通信・起動構成を利用するため、まず公式メーターを変更なしでAGL上に導入し、その後にデザインを移します。

**WSL、ソース閲覧用の別フォルダー:**

```bash
cd ~/work
git clone https://gerrit.automotivelinux.org/gerrit/apps/flutter-instrument-cluster
cd flutter-instrument-cluster
git checkout 6c204692c9ee5ff2c1f20134d55223d2312c9be1
git switch -c meter-design
```

今回の配布イメージに対応するAGLレシピが指定しているコミットです。

| ファイル・フォルダー | 読む内容 |
|---|---|
| `lib/main.dart` | 起動処理とアプリ全体 |
| `lib/screen/home.dart` | 画面の配置と受信処理の開始 |
| `lib/screen/widgets/` | メーターやアイコン |
| `lib/screen/paints/` | 独自描画 |
| `lib/vehicle-signals/vss_client.dart` | KUKSAへのgRPC購読 |
| `lib/vehicle-signals/vehicle_status_provider.dart` | 受信値と画面状態の対応 |
| `lib/vehicle-signals/vss_path.dart` | 使用する信号名 |
| `lib/config.dart` | 接続先、TLS、トークン設定 |
| `pubspec.yaml` | パッケージ依存関係と画像素材 |

この版は `kuksa.val.v1` の生成コードを含みます。別の版のKUKSAチュートリアルからAPIを混ぜないでください。公式アプリのトップ画面は起動時にKUKSAへ接続するため、WSLで開くだけではAGLと同じデータを取得できません。

デザインを移すときは、公式アプリ側の車速の状態を `SpeedPanel` の引数へ渡します。公式ソースを丸ごと教材の `main.dart` で置き換えると、既存のKUKSA接続処理が失われます。

## 6. AGL向けにビルドする

**ここからはビルド担当者向けです。** AGLのフルビルド環境または対応eSDKが必要です。以下は固定した公式レシピに基づく手順例で、今回この工程のビルド実行はしていません。

### 6.1 配布版に合わせた環境を準備する

[AGLのビルドホスト準備](https://docs.automotivelinux.org/en/master/01_Getting_Started/02_Building_AGL_Image/02_Preparing_Your_Build_Host/)に従って必要パッケージと `repo` を導入します。通常ユーザーで、Linuxファイルシステム上に作業場所を置きます。

**ビルドホスト:**

```bash
mkdir -p ~/AGL/agl-20261001
cd ~/AGL/agl-20261001
repo init --standalone-manifest \
  -u https://github.com/e-lab-pius-project/agl-wsl-demo/releases/download/agl-20261001/repo_manifest.xml
repo sync -j2
source meta-agl/scripts/aglsetup.sh -m qemux86-64 -b build-meter \
  agl-demo agl-devel agl-flutter
```

`--standalone-manifest` は配布済みXMLの固定コミットを使う指定です。生成されたビルド環境で次を確認します。

```bash
bitbake-layers show-recipes flutter-cluster-dashboard
bitbake -e flutter-cluster-dashboard | grep -E '^(SRCREV|FLUTTER_VERSION|MACHINE)='
```

対象レシピの `SRCREV` が `6c204692c9ee5ff2c1f20134d55223d2312c9be1`、対象マシンが `qemux86-64` であることを確認します。公開した `local.conf` は公式CIの記録で、CI固有の絶対パスを含みます。そのまま自分の設定ファイルへ上書きしないでください。

### 6.2 公式アプリを変更なしでビルドする

**ビルド環境を読み込んだ同じターミナル:**

```bash
devtool modify flutter-cluster-dashboard
devtool status
devtool build flutter-cluster-dashboard
```

編集対象は `devtool status` に表示されたソースフォルダーです。先ほど閲覧用にcloneしたフォルダーと取り違えないでください。初回は依存するEngineやツールのビルドも必要になることがあります。

公式レシピはアプリ本体だけでなく、`flutter-cluster-dashboard.service`、表示設定、KUKSA設定なども配置します。まず無変更版を通してから、色やレイアウトを変更します。

### 6.3 パッケージまたはイメージとして配布する場合

アプリの変更をGitにコミットし、独自レイヤーへ残します。例では独自レイヤーをビルドディレクトリの隣へ作ります。

```bash
bitbake-layers create-layer ../meta-pius
bitbake-layers add-layer ../meta-pius
# devtoolのソース内で変更をコミットしてから実行
devtool finish flutter-cluster-dashboard ../meta-pius
bitbake flutter-cluster-dashboard
```

授業用に現在のIVI環境へ同梱する場合は、`conf/local.conf` に次を追加します。

```bitbake
IMAGE_INSTALL:append = " flutter-cluster-dashboard"
```

`bitbake agl-ivi-demo-flutter` で作成する場合は、IVIホーム画面とメーターのどちらを起動するかも設定します。同じ背景表示を要求するアプリを同時に起動しないようにします。

メーター専用の構成を作る場合は、公式ターゲット `bitbake agl-cluster-demo-flutter` を選べます。ただし、サービス構成やCAN入力設定はIVIイメージと完全に同じとは限りません。新しいイメージで [CAN試験](CAN.md) を再確認します。

本リポジトリの `prepare.sh` は公開済みIVIイメージ専用です。別イメージに差し替えるときは、対応するカーネル・ハッシュ・起動設定を一組として用意します。

## 7. QEMUのAGLへ配置して起動する

この節は **ビルド環境とQEMUが同じWSL内にある場合** の例です。別PCのビルドサーバーで `127.0.0.1:2222` を指定しても学生PCのQEMUには接続できません。別PC運用では、担当者が同一ビルド由来のパッケージ一式またはイメージを配布してください。

### 7.1 作業用ディスクを退避する

AGLを `poweroff` で終了してから、**WSL** で実行します。

```bash
cd ~/agl-wsl-demo
cp --reflink=auto --sparse=always \
  "$HOME/agl-qemu/agl-work.qcow2" \
  "$HOME/agl-qemu/agl-work.before-meter.$(date +%Y%m%d-%H%M%S).qcow2"
bash scripts/start.sh
```

`AGL_DIR` を変更した人は読み替えます。バックアップも元のrawディスクを参照するため、rawディスクを消さないでください。

### 7.2 依存関係を確認して配置する

**AGL:**

```bash
cat /etc/default/flutter
command -v flutter-auto
rpm -q flutter-auto agl-flutter-env liberation-fonts
systemctl is-active agl-compositor kuksa-databroker
```

`devtool deploy-target` はアプリのレシピ成果物を配置しますが、実行時の依存パッケージは自動では入れません。不足や版違いがあれば同じビルドのパッケージを用意するか、前節のイメージへ同梱します。

devtoolからも同じSSH設定を使うため、**ビルドホストの `~/.ssh/config` に以下を追記**します。既存設定は上書きしません。保存先を変更した場合は `UserKnownHostsFile` を調整します。

```sshconfig
Host agl-meter-demo
    HostName 127.0.0.1
    Port 2222
    User root
    UserKnownHostsFile ~/agl-qemu/known_hosts
    StrictHostKeyChecking accept-new
```

**ビルドホスト、devtool環境:**

```bash
ssh agl-meter-demo true
devtool deploy-target -n -P 2222 flutter-cluster-dashboard root@agl-meter-demo
devtool deploy-target -P 2222 flutter-cluster-dashboard root@agl-meter-demo
```

`-n` で配置予定を確認してから実配置します。この操作は `devtool modify/build` 後、`devtool finish` 前の作業環境で行います。finish済みの場合は再度modify/buildするか、パッケージで配置します。

### 7.3 IVIからメーターへ表示を切り替える

**AGL:**

```bash
systemctl daemon-reload
systemctl cat flutter-cluster-dashboard
systemctl stop flutter-ics-homescreen
systemctl start flutter-cluster-dashboard
systemctl status flutter-cluster-dashboard --no-pager
journalctl -u flutter-cluster-dashboard -n 80 --no-pager
```

公式サービスは `agl-driver` ユーザーで `flutter-auto` を実行します。SSHのrootシェルから単に `flutter-auto` を起動すると、Wayland接続先や権限が合わないことがあるため、まず同梱サービスを使います。

この固定レシピの表示設定は `/usr/share/flutter/flutter-cluster-dashboard.json` に配置されますが、元ファイルはTOML形式です。向きやサイズが合わない場合は、中身と `/etc/xdg/weston/weston.ini` を確認し、バックアップ後に1920×1080の横向きに合わせてサービスを再起動します。拡張子だけで形式を決めつけないでください。

元のIVI画面へ戻す場合:

```bash
systemctl stop flutter-cluster-dashboard
systemctl start flutter-ics-homescreen
```

必要なら **ビルドホスト** で `devtool undeploy-target -P 2222 flutter-cluster-dashboard root@agl-meter-demo` を使い、配置前のファイルを復元します。サービスを止めてから行い、AGLで `systemctl daemon-reload` を実行します。

## 8. CAN・KUKSA・表示をつなぐ

[CAN.md](CAN.md) のトンネルを開始します。メーターが読み出す接続設定は、AGL内の次のファイルです。

- 共通設定: `/etc/xdg/AGL/kuksa.toml`
- アプリ設定: `/etc/xdg/AGL/flutter-cluster-dashboard/kuksa.toml`
- CA証明書: `/etc/kuksa-val/CA.pem`
- アプリ用トークン: `/etc/xdg/AGL/flutter-cluster-dashboard/flutter-cluster-dashboard.token`

通常の接続先はAGL内の `localhost:55555` です。この配布構成はTLSを使います。ホスト名、CA、認証、読み取り許可を合わせます。認証エラーを解消するためにTLS検証を無効化したり、トークンの内容をGitに載せたりしないでください。

**WSL、リポジトリ内:**

```bash
bash scripts/check-can.sh
```

KUKSAで0、20、42、0が確認でき、同時にメーターの数値も一致することを確認します。表示をゆっくり確認したい場合は **仮想CANであることを確認したうえで** 次を送れます。

```bash
ip -details link show vcan_agl | grep -qw vcan &&
  cangen vcan_agl -g 100 -n 50 -I 3E9 -L 8 -D 1500000000000000
```

これは42 km/hを約5秒送ります。終了後は `bash scripts/check-can.sh` で最後を0に戻します。

| 観察 | 切り分ける場所 |
|---|---|
| WSLのcandumpに出ない | 送信側、インターフェース |
| AGLの `candump can0` に出ない | cannelloni、TCP接続 |
| CANは届くがKUKSAが変わらない | ID、DBC、対応付け、CANプロバイダー |
| KUKSAは変わるが画面が変わらない | TLS・認証、購読パス、状態更新、Widget |
| アプリ自体が起動しない | Engineの版、成果物、依存関係、サービス、表示権限 |

デザインを変えるときは、devtoolのソースを編集し、build → アプリ停止 → deploy-target → アプリ開始を繰り返します。AGL全体のイメージ作成は、配布版を作る段階で行います。

## 9. PIUS用の信号へ置き換える

USB-CANの接続、ビットレート設定、物理CANからAGLへの転送は [USB-CAN接続手順](USB_CAN.md) を参照してください。

公式デモのID `0x3E9` とPIUSのCAN IDは別の定義です。次の対応表を実際のPIUS仕様書に基づいて埋めます。

| CAN ID・形式 | 信号 | ビット位置・長さ | 符号・バイト順 | 倍率・オフセット | 単位 | VSSパス |
|---|---|---|---|---|---|---|
| 実仕様から転記 | 車速 | 実仕様から転記 | 実仕様から転記 | 実仕様から転記 | km/h等 | Vehicle.Speed等 |
| 実仕様から転記 | 電池残量 | 実仕様から転記 | 実仕様から転記 | 実仕様から転記 | %等 | 使用版のVSSで確認 |

標準IDと拡張IDを区別します。11ビット標準IDの最大値は `0x7FF` なので、例えば `0x2010` は標準IDとして扱えません。実際のフレーム形式、CAN FDの使用有無、データ長も確認します。

DBCとVSSの対応を作り、まず保存ログまたはvcanで既知の値を入力します。境界値・欠測・通信復帰を確認してから実際のCANインターフェースへ接続します。

## 10. 完了条件と成果物

- 車速0、20、42が入力から画面まで一致し、単位も一致する。
- 起動直後・未受信・切断・復帰時の表示が仕様どおりになる。
- 画面サイズを変えても数字や警告が欠けない。
- 表示の遅延とCPU・メモリ使用量を記録する。QEMUの測定値を実機性能とは扱わない。
- ソース、依存関係、使用イメージ、ビルド手順、設定差分、試験結果がGitで追跡できる。
- 起動失敗時にIVI画面または退避ディスクへ戻せる。

Raspberry Piへ移す段階では、対応AGLイメージ・BSPを選び、ARM向けにアプリを作り直します。画面、GPU、CANドライバー、起動時間を実機で再確認します。x86-64用の今回のディスクを、そのままPiのSDカードへ書くことはできません。
