# 構成要素の概要

[起動ガイド](../README.md) · [メーターパネル開発手順](METER_DEVELOPMENT.md) · [公式リンク一覧](REFERENCES.md)

言葉の意味を調べる場合は [技術用語集](GLOSSARY.md) を参照してください。本書は構成全体の関係を説明します。

Flutterのコマンドから実行の仕組みまで順に学ぶ場合は [Flutter入門](FLUTTER_BASICS.md) を参照してください。

対象は本リポジトリで配布する AGL 21.93.0、2026-10-01 の qemux86-64 イメージです。調査日: 2026-10-06。

## 全体の関係

開発時の実行環境は **Windows → WSL2のUbuntu → QEMU → AGL → メーターアプリ** です。画面を試作する段階では、WSLのUbuntuでFlutterアプリを直接実行することもできます。

車速の入力は **CAN → SocketCAN → CANプロバイダー → KUKSA → Flutterの画面** と進みます。WSLとQEMU内のCANを接続する箇所には、cannelloniを使います。

OSを作るときに使うのが **Yocto Project / OpenEmbedded / BitBake** です。表示色や配置を変えるたびに、OS全体を作り直す必要はありません。

## 用語と担当範囲

| 構成要素 | 何をするものか | 今回の役割 |
|---|---|---|
| AGL (Automotive Grade Linux) | 自動車向けのオープンソースソフトウェア基盤 | メーターアプリ、車両データサービス、表示環境が動くゲストOS |
| Yocto Project | 組み込み向けLinuxを構成・ビルドするためのプロジェクトとツール群 | AGLのイメージや開発用SDKを作る基盤 |
| OpenEmbedded / BitBake | ビルド定義と、その定義を実行する仕組み | ソース取得、コンパイル、パッケージ化、イメージ生成 |
| レシピ / レイヤー | ソフトウェアの作り方を記述した `.bb` と、それらをまとめたもの | メーターアプリをAGLに組み込む |
| Linuxカーネル / ドライバー | CPU、メモリ、画面、ネットワーク、CANなどを扱う | QEMU内の仮想ハードウェアや、将来のRaspberry Piを制御 |
| Flutter | ウィジェットを組み合わせて画面を作るUIフレームワーク | 速度、警告、電池残量などを描画 |
| Dart | Flutterアプリを記述するプログラミング言語 | 画面の構造、表示状態、データ更新処理を記述 |
| Flutter Engine / Embedder | Dartの実行・描画とOS側の画面機能をつなぐ | AGLでは `flutter-auto` がアプリを実行 |
| Wayland / agl-compositor | アプリの表示面を画面へ配置する仕組み | メーターを表示する画面、向き、背景などを管理 |
| systemd | サービスの起動・停止・依存関係を管理 | メーター、KUKSA、コンポジターの起動管理 |
| KUKSA Databroker | 車両信号の値を受け取り、アプリへ配信するサービス | `Vehicle.Speed` などを取得・購読する窓口 |
| VSS (Vehicle Signal Specification) | 車両信号の名前・型・単位を整理する仕様 | 車速を `Vehicle.Speed` としてアプリへ渡す |
| gRPC / Protocol Buffers | API通信とメッセージ形式 | FlutterとKUKSAの通信。この版の公式アプリは `kuksa.val.v1` を利用 |
| CAN / SocketCAN | 車載通信と、LinuxでCANを扱うAPI | CANフレームを受信する。仮想インターフェース `vcan` で模擬可能 |
| DBC | CAN ID、ビット位置、倍率などを記述するファイル形式 | バイト列を車速などの数値へ変換する定義 |
| kuksa-can-provider | CANフレームを信号に変換してKUKSAへ渡すサービス | DBCと対応付けを使い、CAN入力を `Vehicle.Speed` にする |
| cannelloni | CANフレームをネットワーク越しに運ぶツール | WSLの `vcan_agl` とAGLの `can0` をTCP接続 |
| WSL2 / Ubuntu | Windows内でLinux環境を利用する仕組み / ディストリビューション | QEMU、編集、転送、仮想CAN試験のホスト |
| WSLg | WSL内のGUIアプリをWindows上に表示する機能 | QEMUのウィンドウや、試作Flutterアプリを表示 |
| QEMU / KVM | 仮想マシン / Linuxの仮想化支援機能 | AGLのカーネルとルートファイルシステムを起動 |
| SSH / SCP | リモート操作 / ファイル転送 | WSLからAGLへ接続、アプリや設定を配置 |
| Git / GitHub | 変更履歴管理 / 共有サービス | ソース、手順書、検証版イメージを共有 |

個々の定義と実装の参照先は [公式リンク一覧](REFERENCES.md) にまとめています。

## AGL と Yocto の違い

Yoctoは、**用途別のLinuxを構成・ビルドするための仕組み**です。

この構成で起動するOSはAGLです。Ubuntuはその外側でQEMUを動かしています。AGLのルートファイルシステムにUbuntuの `apt install` 手順をそのまま適用することはできません。

変更するものによって、必要なビルド範囲が異なります。

| 変更内容 | 主な作業 |
|---|---|
| 数字の色・大きさ・配置 | Dartや画像を変更し、アプリをビルド |
| 受信する信号 | KUKSA購読・状態更新・必要な信号定義を変更 |
| CANのビット位置や倍率 | DBC、CANプロバイダーの対応付けを変更 |
| アプリを起動時に立ち上げる | systemdサービスと表示設定を変更 |
| ドライバーを追加する | カーネル設定・レシピを変更してビルド |
| x86-64からRaspberry Piへ移す | 対象ハードウェアの設定でOSとアプリを再ビルド |

Yoctoでいう「レシピ」は料理の手順のように「何を取得してどう作るか」を表します。レイヤーはその手順を関連する単位でまとめたものです。AGLの既存レシピを使うと、Flutter Engineの版や配置場所も合わせられます。

## IVI とメーターパネルの違い

**IVI (In-Vehicle Infotainment)** はナビ、音楽、空調操作などの情報端末です。**Instrument Cluster (IC)** は運転者の前にある速度計・警告灯などの計器盤です。

現在配布中の `agl-ivi-demo-flutter` はIVIデモです。そこからメーター開発へ進む際の公式部品は以下です。

| 名前 | 種類 | 用途 |
|---|---|---|
| `flutter-ics-homescreen` | 現在動くアプリ | IVIのホーム画面 |
| `flutter-instrument-cluster` | ソースリポジトリ | 公式FlutterメーターのDartソース |
| `flutter-cluster-dashboard` | レシピ・パッケージ・サービス名 | 上記ソースをAGL向けにビルド・起動 |
| `agl-cluster-demo-flutter` | イメージのビルドターゲット | メーターを主画面にしたAGLイメージ |

名前が似ていても役割は異なります。配布中のIVIイメージにメーターアプリが既に入っていると仮定せず、開発手順に従って追加します。

## Flutter SDK と AGLの実行環境

Flutter SDKは開発用ツール、Engineはアプリを動かす実行基盤です。今回のAGLイメージの `/etc/default/flutter` では **FLUTTER_VERSION=3.38.3、FLUTTER_RUNTIME=release** を確認しています。

WSLで `flutter run -d linux` を実行すると、Ubuntu向けのデスクトップアプリとして動きます。AGLでは `flutter-auto`、対応するEngine、アプリの成果物、表示設定、サービスを組み合わせて動かします。Ubuntu用の実行ファイルだけをAGLへコピーすればよいとは限りません。

通常のFlutter SDK、AGL向けSDK、Yoctoの標準SDK、eSDKは同じものではありません。`devtool` はYoctoビルド環境やeSDKで使います。任意の標準SDKを入れるだけで、AGLの全レシピを `devtool` でビルドできるとは限りません。

## CANの値が画面へ届くまで

例えば車速42 km/hでは、現在の試験用DBCに合わせた `0x3E9` のフレームを送ります。CANプロバイダーが数値へ変換し、KUKSAに `Vehicle.Speed = 42` を渡します。メーターアプリはその信号を購読し、受け取った値で画面を更新します。

**CANフレームが届くこと、KUKSAの値が変わること、画面が変わることは別の確認項目です。** 本リポジトリの既存試験はKUKSAまでを確認しています。

DBCのIDと倍率は車両ごとに異なります。AGLデモのDBCがPIUSの通信仕様と一致するわけではありません。実車へ進むときはPIUSの定義を確認し、VSSとの対応を作ります。
