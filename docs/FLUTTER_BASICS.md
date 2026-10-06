# Flutter入門：コマンドを実行すると何が起きるのか

[起動ガイド](../README.md) · [メーターパネル開発手順](METER_DEVELOPMENT.md) · [技術用語集](GLOSSARY.md)

Flutterを初めて使い、メーター開発の手順書を読む学生向けの資料です。プログラムを編集することと、ビルドすること、実行先へ配置することの違いから説明します。FlutterやDart VMを知っている前提ではありません。

**メーターの見た目はDartコードで決め、そのコードを実行環境に合う形へ変換して動かします。** 開発中はコードの一部を実行中のアプリへ反映でき、配布時にはAGL向けの成果物を用意します。

初回は1〜7節を順に読み、AGLへの配置に進む前に8〜9節を参照してください。本文はWSLのLinuxアプリと今回のAGLを対象にします。ブラウザー向けFlutterは実行方式が異なるため、ここでは扱いません。

確認日: 2026-10-06。仕組みは公式資料に基づく説明です。Flutterの画面試作・AGL向けビルド・Hot Reloadは、本リポジトリの環境ではまだ実行検証していません。CANの疎通確認済みという記録とは分けて読んでください。

## 1. まず、どのコンピューターでアプリを動かすかを決める

この開発環境には、Windows、WSLのUbuntu、QEMU内のAGLがあります。Ubuntuでコマンドを打ったからといって、自動的にAGLへアプリが入るわけではありません。

| 作業 | 主に使う場所 | 結果が現れる場所 |
|---|---|---|
| ソースコードを編集 | エディターでWSL内のプロジェクトを開く | `.dart` ファイル |
| `flutter run -d linux` | WSLのUbuntu | Ubuntuでアプリが動き、WSLg経由でWindowsにウィンドウが出る |
| AGL向けにビルド | 対応するYoctoビルド環境など | AGLに配置するアプリや設定の成果物 |
| AGLへ配置・起動 | SSH接続先のAGL | QEMUの画面内にアプリが出る |

最初はUbuntuで数字や針の表示を作ります。この段階ではQEMUとCANを止めていても開発できます。具体的なSDK導入は [開発手順の4節](METER_DEVELOPMENT.md#4-wslでflutterの画面を試作する) を参照してください。

## 2. doctorは環境確認、createは雛形作成、runは起動

以下はFlutter SDKとLinux開発ツールの導入後に、Ubuntuのターミナルで実行する例です。Dartのソースコードとは別の、シェルに入力するコマンドです。

```bash
flutter doctor -v
flutter create --platforms=linux pius_meter
cd pius_meter
flutter run -d linux
```

**`flutter doctor -v` は開発環境の診断です。** Flutterの版、コンパイラーなどのツール、実行対象を確認し、不足を報告します。`-v` は詳細表示を意味します。メーターの作成・ビルド・起動をするコマンドではありません。今回見るのは特にFlutterとLinux toolchainで、Androidを使わないならAndroid SDKの警告は切り分けて扱います。

`flutter create --platforms=linux pius_meter` は、`pius_meter` フォルダーにプロジェクトの雛形を作ります。`--platforms=linux` はLinux向けの起動部分を生成する指定です。AGLやKUKSAへの接続設定は、このコマンドだけでは作られません。

| 生成される主な場所 | 役割 |
|---|---|
| `lib/main.dart` | アプリの入口と最初の画面。主な編集対象 |
| `pubspec.yaml` | アプリ情報、利用パッケージ、画像などの素材の設定 |
| `linux/` | Linuxでアプリを起動し、OSとつなぐ部分 |
| `test/` | 動作を検査するテストコード |
| `build/` | ビルド時に生成される成果物。通常は直接編集しない |

雛形には動作するサンプルコードが入ります。標準テンプレートは、中央の数字を右下の「＋」ボタンで増やすカウンターアプリです。何も編集せずに起動しても画面が出ます。テンプレートの色や細部はSDKの版で変わります。

`flutter run -d linux` は、現在のプロジェクトをLinux向けにビルドして起動します。`-d` は実行対象を選ぶ指定で、ここではUbuntu上のLinuxデスクトップです。通常は開発用のDebugモードで動き、コマンドも起動したままアプリへ接続し続けます。

createは通常、プロジェクトの最初の1回だけです。その後は同じフォルダーで編集とrunを繰り返します。参照: [Flutterコマンド一覧](https://docs.flutter.dev/reference/flutter-cli)、[新規アプリの作成](https://docs.flutter.dev/reference/create-new-app)。

## 3. Dartで画面の部品を組み合わせる

Flutterは画面を作るための仕組みで、アプリを書く言語がDartです。画面の構成を記述する部品を **Widget（ウィジェット）** と呼びます。

次は画面の一部だけを示す例です。この断片だけを `main.dart` 全体として保存するものではありません。実行できる数値メーターの全体例は [開発手順の4.2節](METER_DEVELOPMENT.md#42-小さな画面から作る) にあります。

```dart
const rpm = 3000;

// build()の中で返す画面部品の例。
return Center(
  child: Text(
    '$rpm rpm',
    style: const TextStyle(fontSize: 64),
  ),
);
```

`Center` は中身を中央へ配置し、`Text` は文字を表示します。`'$rpm rpm'` の `$rpm` は、変数rpmの値を文字列へ埋め込む書き方です。この場合は「3000 rpm」になります。KUKSAから読み出す特別な記法ではありません。

| コードに出てくる名前 | 役割 |
|---|---|
| `main()` | プログラムを開始するときに呼ぶ入口 |
| `runApp()` | Flutterへ最初のWidgetを渡し、画面を動かし始める |
| `MaterialApp` / `Scaffold` | アプリ全体の設定 / 画面の土台 |
| `Row` / `Column` / `Stack` | 横に並べる / 縦に並べる / 重ねる |
| `build()` | 現在のデータに応じたWidgetの構成を返す処理 |
| State | 現在のRPMなど、変化していく画面の状態 |
| `setState()` | 状態を変更し、その変更を画面へ反映するようFlutterへ知らせる |

Widgetの `build()` と、`flutter build` というビルドコマンドも区別します。前者は実行中に画面の構成を作る処理、後者は実行・配布用の成果物を作る操作です。参照: [Flutter UIの基本](https://docs.flutter.dev/ui)。

## 4. 円と針は描画し、RPMの値は外から渡す

数字はTextで表示できます。円形の目盛りや針を自分で描く場合は、描画領域を用意する `CustomPaint` と、描き方を定義する `CustomPainter` を使います。

`CustomPainter` の `paint(Canvas canvas, Size size)` 内で、円を `canvas.drawCircle()`、針を `canvas.drawLine()` で描きます。RPMから針の角度を計算し、中心から針の先端まで線を引きます。[CustomPainterの公式説明](https://api.flutter.dev/flutter/rendering/CustomPainter-class.html)

例えば0〜6000 rpmを270度の範囲へ割り当てるなら、割合は「RPM ÷ 6000」です。左下を開始点にすると、0 rpmで左下、3000 rpmで真上、6000 rpmで右下を向きます。Flutterの座標では右がxの正方向、下がyの正方向です。

```text
角度 = 135度 + 270度 × 割合
先端x = 中心x + 針の長さ × cos(角度)
先端y = 中心y + 針の長さ × sin(角度)
```

Dartのsin・cosへ渡す角度はラジアンなので、「度 × π ÷ 180」で変換します。6000 rpmという上限はこの例の設定です。実際のモーターに合わせて決め、範囲を超える値や逆転時の表示も設計します。

最初は固定値やSliderでrpmを変えます。KUKSAへ接続するときは、購読処理が受け取った値で状態を更新します。次はStateクラス内に置く処理の断片です。

```dart
double? rpm; // nullは未受信。0 rpmとは区別する。

void onRpmReceived(double value) {
  if (!mounted || !value.isFinite) return;
  setState(() {
    rpm = value;
  });
}
```

この関数を書くだけでKUKSAへ接続されるわけではありません。別途、接続先・TLS・トークンを設定し、受信時にこの関数を呼ぶ購読処理が必要です。未受信なら `--` などを表示し、有効な値があるときに描画部品へ渡します。

今回の信号は `Vehicle.Powertrain.ElectricMotor.Speed` で、型はfloat、単位はrpmです。VESCのERPMからRPMへの換算はCANプロバイダーで行い、画面側で二重に割り算しません。[VESCから画面までの設定手順](VESC_RPM.md)で具体的に扱います。

**値が変わって針が動くことは、Hot Reloadとは別です。** KUKSA受信はデータの更新、Hot Reloadは開発中のプログラムの更新です。配布用のReleaseアプリでも、受信値に合わせて針は動きます。

## 5. Flutter Engineはアプリ内で動く実行・描画の基盤

ファイルとして保存されたプログラムを起動すると、OSは実行中のプログラムを **プロセス** として管理します。別のプログラムから使う機能をまとめたものが **ライブラリー** です。

Flutter Engineは、通常、アプリのプロセス内で使われるライブラリーです。Dartコードを実行する環境や、画面を描画する機能を提供します。OSの表示・入力機能とEngineをつなぐ部分を **Embedder（エンベダー）** と呼びます。[Flutterの構造](https://docs.flutter.dev/resources/architectural-overview)

今回のAGLでの担当は次のとおりです。

| 部品 | 何を担当するか |
|---|---|
| 自作のDartコードとFlutter Framework | どの数字・円・針を、どこへ表示するかを記述する |
| Dartの実行環境 | Dartの処理を実行し、メモリなどを管理する |
| Flutter Engine | Dartの実行環境を含み、描画などの機能を提供する |
| `flutter-auto` | Engineを組み込み、AGLの表示・入力環境につなぐ実行プログラム |
| `agl-compositor` | アプリが作った表示面を画面へ配置・合成する |
| systemd | アプリや表示サービスを起動・監視する |

バックグラウンドで継続して動くプロセスをデーモンと呼び、通信相手の要求を受ける役割をサーバーと呼びます。今回のAGLではsystemdがFlutterアプリを起動・監視します。Engine自体が全アプリ共通の描画サーバーになるわけではありません。

一方、Debugモードのアプリには、開発ツールが接続する **Dart VM Service** という窓口があります。Hot Reloadやデバッグのために利用する通信機能です。「サーバー機能がある」のはこの部分であり、画面を配信するWebサーバーとは役割が違います。

## 6. Dartコードは中間表現を経て、CPUで実行される

人が書く `.dart` が **ソースコード** です。CPUが直接実行する命令が **機械語** で、ソースを実行に適した形式へ変換する作業を **コンパイル** と呼びます。ビルドは、コンパイルに加えて素材やライブラリーをまとめる処理なども含みます。

変換の途中で使う形式が **中間表現（IR: Intermediate Representation）** です。DartではKernelという中間表現を使います。ここでのKernelは、WSLやAGLのLinuxカーネルとは別のものです。DLLや画面の画像データでもありません。

Debugモードでは、この中間表現をDart VMが読み込み、必要なコードを実行時に機械語へ変換します。この方式を **JIT（Just-in-Time）コンパイル** と呼びます。[Dartの実行方式](https://dart.dev/overview#dart-native-machine-code-jit-and-aot)

| 段階 | メーターの例 |
|---|---|
| Dartソース | RPMを角度へ変換し、針を描く処理を書く |
| Kernel中間表現 | コンパイラーが処理しやすい形式へ変換する |
| JITコンパイル | 実行時に、その処理をCPUの機械語へ変換する |
| 実行 | RPMから角度を計算し、Engineの描画機能を呼ぶ |

**Dart VMは、単にソースを一行ずつ解釈するインタープリターではありません。** JITコンパイラーやメモリ管理などを備える実行環境です。「中間表現を読み込むこと」と「実行時に機械語へ変換すること」は両立します。

VMはVirtual Machineの略ですが、ここでのDart VMはDartプログラムの実行環境を指します。QEMUのように仮想CPUや仮想ディスクを用意してLinuxを起動するものではありません。

配布用では、実行前に機械語へ変換する **AOT（Ahead-of-Time）コンパイル** を使います。AOTでもメモリ管理などを担うDartの実行環境は必要です。

| モード | Linux向けDartコードの実行 | 用途 | Hot Reload |
|---|---|---|---|
| Debug | JIT | コードを変えながら開発 | 対応 |
| Profile | AOT | 性能を測定 | 非対応 |
| Release | AOT | 配布・通常運用 | 非対応 |

描画機能のC/C++部分まで毎回JITで作る、という意味ではありません。この説明の対象はDartアプリ側です。参照: [Flutterのビルドモード](https://docs.flutter.dev/testing/build-modes)。

## 7. Hot Reloadは、実行中のDartコードを更新する

Ubuntuで `flutter run -d linux` を実行したまま、針の色を赤から青へ変更したとします。ターミナルで `r` を押すと、次の処理が行われます。

1. 開発ツールが変更したDartコードと関連する部分を差分コンパイルする。
2. 生成したKernel中間表現を実行中のアプリへ送る。
3. Dart VMが変更を読み込み、以後の実行に反映する。
4. FlutterがWidgetの構築・配置・描画をやり直す。

プロセスを終了して更新版を起動する操作ではありません。例えばStateに保持した3000 rpmを残し、針の色だけを変更できます。[Hot Reloadの仕組み](https://docs.flutter.dev/tools/hot-reload#how-it-works)

**描画DLLの差し替えでもありません。** Flutter Engine本体を読み込み直すのではなく、Dartコードを更新します。描画に限らず計算やイベント処理も対象です。C/C++のライブラリーやEngine自体を変えた場合には再ビルド・再起動が必要です。

| 操作 | 起きること | StateのRPM | `main()` |
|---|---|---|---|
| Hot Reload：`r` | 対応するDartコード変更を反映する | 基本的に保持 | 再実行しない |
| Hot Restart：`R` | Dart側を初めから実行する | 初期化される | 再実行する |
| 終了して再起動 | プロセスを終了し、起動し直す | 初期化される | 再実行する |

Hot Restartは、通常、アプリのプロセス全体を終了する操作とは異なります。Hot Reloadには変更内容の制約もあり、何でも反映できるわけではありません。`main()`や `initState()`、保持される状態の初期値を変えた場合などは、Hot Restartが必要になります。

### カウンターアプリで状態の保持を確かめる

SDK導入後、2節の雛形を起動して次を試せます。これは学習用の確認手順で、本資料作成時点では未実行です。

1. 「＋」で表示を3にする。
2. `lib/main.dart` の `build()` 内にある説明文を別の文字列へ変更して保存する。`main()`内の初期値は変更しない。
3. run中のターミナルで `r` を押す。説明文が変わり、数字は3のままになることを確認する。
4. `R` を押す。Dart側が再起動し、数字が初期値の0へ戻ることを確認する。
5. `q` で実行を終了する。

エディターからの保存と連動するHot Reloadは、対応する拡張機能・設定がある場合に使えます。通常のターミナルでrunしただけなら、ファイルを保存した後に `r` を押します。

Hot Reloadで変更した元のソースファイルはディスクに残りますが、実行中のアプリへの反映は配布物の更新ではありません。配布先でも使うには、変更を含む成果物をビルドして配置します。

## 8. 普段の画面開発と、AGLへ配置する作業を分ける

通常のFlutter開発では、プロジェクトを作り、runで起動して編集し、テストを通して配布用にビルドします。Flutter SDKは、この一連の作業に使う道具です。

| 作業 | Ubuntuでのコマンド例 | 何が変わるか |
|---|---|---|
| 診断 | `flutter doctor -v` | 環境の不足を確認する |
| プロジェクト作成 | `flutter create --platforms=linux pius_meter` | ソースと設定の雛形ができる |
| 編集・実行 | `flutter run -d linux` | Ubuntuで開発用アプリが動く |
| コード検査 | `flutter analyze` | 型やコード上の問題を調べる |
| テスト | `flutter test` | 用意したテストで処理や表示を確認する |
| 配布用ビルド | `flutter build linux --release` | Linuxデスクトップ向けの成果物を作る |

雛形のテストもカウンターアプリ用です。画面をメーターへ変更したら、期待する表示に合わせてテストを更新します。参照: [Flutterのテスト](https://docs.flutter.dev/testing/overview)。

AGLでも、DartやCustomPainterの書き方は同じです。ただし、最終的な実行先はAGLです。UbuntuとCPUが同じでも、Engine、ライブラリー、画面の表示方式まで一致するとは限りません。Ubuntu用に作った実行ファイルをそのままコピーする手順には進みません。

| 項目 | Ubuntuでの試作 | 今回のAGLでの実行 |
|---|---|---|
| アプリの起動部分 | Flutter標準のLinuxランナー | `flutter-auto` |
| ビルド | Flutter SDKのコマンド | 対応するAGLレシピ・ツールチェーン |
| 配置 | そのUbuntuで実行 | SSH経由のdevtoolなどでAGLへ転送 |
| 自動起動 | 通常のデスクトップアプリとして起動 | systemdに登録 |
| 現在のHot Reload | Debugで利用できる | 配布イメージはReleaseなので使えない |

AGL向けには、ソースやビルド方法を記録する **レシピ** と、対象環境向けのコンパイラーなどを組み合わせます。今回の手順書で使う **devtool** はYoctoの開発支援ツールです。Flutter SDKを入れただけでは利用できません。

AGLのビルド環境を準備した後の `devtool modify` はレシピのソースを編集対象として展開し、`devtool build` はAGL向けにビルドし、`devtool deploy-target` は成果物をAGLへ配置します。配置先の実行時依存パッケージは別途確認が必要です。[devtool公式資料](https://docs.yoctoproject.org/ref-manual/devtool-reference.html)

最初は公式 `flutter-instrument-cluster` のKUKSA通信処理を使い、画面部分へ自作メーターを組み込む方針です。アプリのソースリポジトリ名と、レシピ・サービス名の `flutter-cluster-dashboard` は異なります。実際のビルド・配置は [開発手順の5〜7節](METER_DEVELOPMENT.md#5-公式メーターのソースを読む) を参照してください。この工程は文書化済みですが未実行です。

### AGLでもHot Reloadしたい場合

AGL向けのDebug対応Engine・アプリと、それらに合う開発ツールを用意し、Dart VM Serviceへ接続できる構成が必要です。QEMUならその通信経路も設定します。対応状況はEmbedderや使用版に依存します。

現在のイメージは `/etc/default/flutter` で3.38.3 / releaseを指定しています。文字列をdebugへ変えるだけでは、Debug用の成果物や接続環境は用意されません。このリポジトリではAGL上のHot Reloadをまだ構築・検証していません。

## 9. 保存・配置・自動起動は、それぞれ別に行う

| 操作 | 保存・変更する対象 |
|---|---|
| エディターで保存、Gitへコミット | 人が編集したソースコード |
| ビルド | 対象環境で動く成果物 |
| デプロイ（配置） | 実行先AGLのファイル |
| `systemctl start` | 今動いているサービスの状態 |
| `systemctl enable` | 次回起動時の自動起動設定。今すぐ起動はしない |
| Yoctoレイヤーへの記録 | 他のPCや新しいイメージでも再現するための定義 |

今回のAGL内で配置・編集したファイルは、作業用 `agl-work.qcow2` に保存されます。元のrawイメージも必要です。アプリの自動起動、USBパススルー、CANの有効化はそれぞれ設定し、再起動後に通して確認します。

普段の針の色・文字サイズの調整はUbuntuで行い、AGLではKUKSAとの接続、表示、起動を確認します。画面の変更のたびにOS全体を作り直す必要はありません。

## 10. 次に読む資料と、迷ったときの確認先

| 知りたいこと | 参照先 |
|---|---|
| SDK導入と最初の数値表示 | [メーターパネル開発手順](METER_DEVELOPMENT.md)の4節 |
| ERPMを受信して針の入力にする | [VESC回転数表示](VESC_RPM.md) |
| UbuntuとAGLでUSB-CANを試す | [CAN疎通確認](USB_CAN_LOOPBACK.md) |
| 単語の意味を引く | [技術用語集](GLOSSARY.md#ui)、特にDart VM、IR、JIT、AOT |
| 全体構成を確認する | [構成要素の概要](COMPONENTS.md) |
| 公式ソースと対象バージョンを調べる | [公式リンク一覧](REFERENCES.md) |

手順と結果が合わない場合は、実行場所（UbuntuかAGLか）、実行したコマンド、エラー、Flutterの版を記録してください。教材の不明点は授業担当者または [リポジトリのIssues](https://github.com/e-lab-pius-project/agl-wsl-demo/issues) へ共有できます。認証トークンの内容は載せないでください。
