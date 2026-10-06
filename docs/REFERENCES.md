# 公式リンク・ソース一覧

[起動ガイド](../README.md) · [構成要素](COMPONENTS.md) · [メーターパネル開発手順](METER_DEVELOPMENT.md)

略語・専門用語の意味は [技術用語集](GLOSSARY.md) にまとめています。

確認日: 2026-10-06。まず「読む順序」の資料を読み、その後は作業に必要な項目を参照してください。`master` や `main` の資料は更新されます。配布イメージとの照合には「固定したソース」を使います。

## 読む順序

1. [AGL公式サイト](https://www.automotivelinux.org/) — プロジェクトの目的と対象。
2. [Yocto Project概要](https://docs.yoctoproject.org/overview-manual/yp-intro.html) — OSを作る仕組み。
3. [Flutter UIの基本](https://docs.flutter.dev/ui) — 画面を構成するウィジェット。
4. [KUKSA利用ガイド](https://github.com/eclipse-kuksa/kuksa-databroker/blob/main/doc/user_guide.md) — 車両データの取得と購読。
5. [本リポジトリの開発手順](METER_DEVELOPMENT.md) — 上記をつないだ作業手順。

## 分野別リンク

| 分野 | 参照先 | 読む目的 |
|---|---|---|
| AGL | [公式ドキュメント](https://docs.automotivelinux.org/en/master/) | 全体の入口 |
| 配布イメージ | [Ready Made Images](https://docs.automotivelinux.org/en/master/01_Getting_Started/01_Quickstart/01_Using_Ready_Made_Images/) | 公式イメージの起動方法 |
| イメージの種類 | [Available Demo Images](https://docs.automotivelinux.org/en/master/01_Getting_Started/02_Building_AGL_Image/07_Available_Demo_Images/) | IVI / Clusterのビルドターゲット |
| AGLビルド | [ビルドホスト準備](https://docs.automotivelinux.org/en/master/01_Getting_Started/02_Building_AGL_Image/02_Preparing_Your_Build_Host/) | ホストOSと必要パッケージ |
| AGLビルド | [環境初期化](https://docs.automotivelinux.org/en/master/01_Getting_Started/02_Building_AGL_Image/04_Initializing_Your_Build_Environment/) | aglsetup.shと機能選択 |
| Yocto | [プロジェクト概要](https://docs.yoctoproject.org/overview-manual/yp-intro.html) | レイヤー・BitBake・OS生成 |
| Yocto | [システム要件](https://docs.yoctoproject.org/ref-manual/system-requirements.html) | フルビルド前のホスト要件確認 |
| Yocto | [devtoolのリファレンス](https://docs.yoctoproject.org/ref-manual/devtool-reference.html) | modify / build / deploy-target / finish |
| Yocto SDK | [Extensible SDK](https://docs.yoctoproject.org/sdk-manual/extensible.html) | SDKとeSDKの開発手順 |
| repo | [repo initマニュアル](https://gerrit.googlesource.com/git-repo/+/HEAD/man/repo-init.1) | 固定manifestの読み込み |
| Flutter | [SDK導入](https://docs.flutter.dev/install/manual) | Linux向けSDKとPATH設定 |
| Flutter | [SDK archive](https://docs.flutter.dev/install/archive) | 必要なFlutterの版を選ぶ |
| Flutter | [Linux開発環境](https://docs.flutter.dev/platform-integration/linux/setup) | コンパイラー・GTK・doctor |
| Flutter | [UIの基本](https://docs.flutter.dev/ui) | レイアウト・描画 |
| Flutter | [テスト](https://docs.flutter.dev/testing/overview) | Widgetテストと結合テスト |
| Dart | [言語ドキュメント](https://dart.dev/docs) | 型、非同期処理、Stream |
| KUKSA | [Databroker利用ガイド](https://github.com/eclipse-kuksa/kuksa-databroker/blob/main/doc/user_guide.md) | 接続、TLS、認証、API |
| VSS | [COVESA VSS](https://covesa.github.io/vehicle_signal_specification/) | 信号名・型・単位 |
| SocketCAN | [Linuxカーネル文書](https://docs.kernel.org/networking/can.html) | CANインターフェースとソケット |
| CAN試験 | [linux-can/can-utils](https://github.com/linux-can/can-utils) | candump / cansend / cangen |
| CAN転送 | [cannelloni](https://github.com/mguentner/cannelloni) | ネットワーク越しのCAN接続 |
| AGLのCAN入力 | [Demo Control Panel（salmon版）](https://docs.automotivelinux.org/en/salmon/06_Component_Documentation/09_AGL_Demo_Control_Panel/) | 公式デモにおけるCAN転送例。版の違いに注意 |
| WSL | [導入](https://learn.microsoft.com/ja-jp/windows/wsl/install) / [GUIアプリ](https://learn.microsoft.com/ja-jp/windows/wsl/tutorials/gui-apps) | Windows側の環境準備 |
| USB/IP | [MicrosoftのUSB接続手順](https://learn.microsoft.com/ja-jp/windows/wsl/connect-usb) / [usbipd-win](https://github.com/dorssel/usbipd-win/wiki/WSL-support) | USB-CANをWindowsからWSLへ接続 |
| USB-CAN | [CANnectivity](https://github.com/CANnectivity/cannectivity) / [本リポジトリの接続手順](USB_CAN.md) | gs_usb方式とAGLまでの接続 |
| QEMU | [起動オプション](https://www.qemu.org/docs/master/system/invocation.html) | 仮想ディスク・ネットワーク・共有 |
| systemd | [サービス定義の公式マニュアルソース](https://github.com/systemd/systemd/blob/main/man/systemd.service.xml) | アプリの起動・再起動 |
| Wayland | [公式概要](https://wayland.freedesktop.org/) | 表示の仕組み |

## 配布イメージに合わせて固定したソース

対象ビルド: **2026-10-01-b3881**。以下のリンクはコミットを指定しています。

| 対象 | 固定ソース | 主な確認箇所 |
|---|---|---|
| メーターアプリ | [flutter-instrument-cluster](https://git.automotivelinux.org/apps/flutter-instrument-cluster/tree/?id=6c204692c9ee5ff2c1f20134d55223d2312c9be1) | lib/、assets/、pubspec.yaml |
| ビルド定義 | [flutter-cluster-dashboard_git.bb](https://git.automotivelinux.org/AGL/meta-agl-demo/tree/recipes-demo/flutter-cluster-dashboard/flutter-cluster-dashboard_git.bb?id=b442b90ae8be41d438b6b01f6e17afb8eeab1a78) | SRCREV、flutter-app、配置と依存関係 |
| 起動と設定 | [flutter-cluster-dashboard/files](https://git.automotivelinux.org/AGL/meta-agl-demo/tree/recipes-demo/flutter-cluster-dashboard/files?id=b442b90ae8be41d438b6b01f6e17afb8eeab1a78) | service、表示設定、KUKSA設定 |
| メーター用OS | [agl-cluster-demo-flutter.bb](https://git.automotivelinux.org/AGL/meta-agl-demo/tree/recipes-platform/images/agl-cluster-demo-flutter.bb?id=b442b90ae8be41d438b6b01f6e17afb8eeab1a78) | OSに含めるパッケージ |
| 現在のIVIアプリ | [flutter-ics-homescreenのレシピ](https://git.automotivelinux.org/AGL/meta-agl-demo/tree/recipes-demo/flutter-ics-homescreen/flutter-ics-homescreen_git.bb?id=b442b90ae8be41d438b6b01f6e17afb8eeab1a78) | 現在のホーム画面との違い |
| 全レイヤーの版 | [配布Release](https://github.com/e-lab-pius-project/agl-wsl-demo/releases/tag/agl-20261001) | repo_manifest.xml、local.conf、SPDX |
| 元のビルド | [AGL公式配布ディレクトリ](https://download.automotivelinux.org/AGL/snapshots/master/2026-10-01-b3881/qemux86-64/) | 公式成果物とビルド情報 |

## 過去の資料を読むとき

[trout版のFlutter Instrument Cluster手順](https://docs.automotivelinux.org/en/trout/01_Getting_Started/03_Build_and_Boot_guide_Profile/03_Flutter_Instrument_Cluster_%28qemu-x86%29/)も公式資料ですが、今回のmasterスナップショットとは異なります。古いイメージ名、機能指定、FlutterやKUKSAのAPIをそのまま混ぜず、上の固定レシピと照合してください。
