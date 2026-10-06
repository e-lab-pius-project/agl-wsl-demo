# AGL・WSL・メーターパネル開発の用語集

[起動ガイド](../README.md) · [構成要素の概要](COMPONENTS.md) · [開発手順](METER_DEVELOPMENT.md) · [公式リンク一覧](REFERENCES.md)

対象は、このリポジトリの **Windows → WSL2のUbuntu → QEMU → AGL** という構成です。作成日: 2026-10-06。

用語の意味を調べるときに使う資料です。構成全体を順に学ぶ場合は [構成要素の概要](COMPONENTS.md)、実際に操作する場合は各手順書を参照してください。ページ内検索（Ctrl+F）で英語名・略語を検索できます。

## 分野別目次

- [車載システム](#automotive): AGL、IVI、Instrument Cluster、HMI
- [OSと操作環境](#os): Yocto、Ubuntu、Linux、WSL、BusyBox、systemd
- [仮想マシンとディスク](#virtualization): QEMU、KVM、qcow2、ext4、イメージ
- [画面とFlutter](#ui): Dart、Widget、Engine、Hot Reload、Wayland
- [ビルドと配置](#build): SDK、BitBake、レシピ、BSP、devtool
- [CAN通信](#can): SocketCAN、DBC、CAN ID、ビットレート、listen-only
- [車両データと通信経路](#data): KUKSA、VSS、cannelloni、gRPC、USB/IP
- [接続・認証・共有](#access): SSH、SCP、TLS、Git、SHA-256
- [混同しやすい言葉](#differences)
- [実際の名前の読み方](#examples)

<a id="automotive"></a>

## 1. 車載システム

| 用語 | 意味 | このデモでの使い方 |
|---|---|---|
| **AGL — Automotive Grade Linux** | 自動車向けのオープンソースソフトウェア基盤。 | QEMUで起動するLinuxシステム。Flutterアプリや車両データサービスが動く。 |
| **IVI — In-Vehicle Infotainment** | ナビ、音楽、電話連携などを扱う車載情報・娯楽システム。 | 現在配布している画面はFlutterのIVIデモ。 |
| **Instrument Cluster / IC** | 車速・警告灯などを表示する運転者向けの計器盤。メーターパネル。 | PIUS用として開発したい画面。ここでのICは「集積回路」の意味ではない。 |
| **HMI — Human-Machine Interface** | 人と機械の間で、情報を表示したり操作を受け付けたりする仕組み。 | メーターの数字・警告表示や、操作画面。 |
| **UCB — Unified Code Base** | AGLが共有するソフトウェア基盤を指す名称。 | AGLの紹介やリリース資料で出てくる。 |
| **デモアプリ / デモイメージ** | 機能や構成を示すためのアプリ / 起動用システム一式。 | デモのCAN定義や表示内容が、そのままPIUSの仕様になるわけではない。 |

参照: [AGL公式](https://www.automotivelinux.org/)、[AGLのイメージ一覧](https://docs.automotivelinux.org/en/master/01_Getting_Started/02_Building_AGL_Image/07_Available_Demo_Images/)。

<a id="os"></a>

## 2. OSと操作環境

| 用語 | 意味 | このデモでの使い方 |
|---|---|---|
| **OS — Operating System** | ハードウェアやアプリの実行を管理する基本ソフトウェア。 | Windows、WSL内のUbuntu、QEMU内のAGLを区別する。 |
| **Linuxカーネル** | CPU、メモリ、デバイスなどを管理するLinuxの中心部分。 | WSLのカーネルと、QEMU内で動くAGLのカーネルは別。 |
| **ディストリビューション** | カーネル、コマンド、ライブラリー、管理機能などをまとめたLinuxシステム。 | UbuntuとAGLは別のディストリビューション。 |
| **Ubuntu** | Linuxディストリビューションの一つ。 | WSL内でQEMU、開発ツール、CAN転送を動かす環境。 |
| **Yocto Project** | 組み込み機器向けLinuxを構成・ビルドするためのプロジェクトとツール群。 | AGLのOSイメージを作る基盤。「Yacto」ではなく「Yocto」。 |
| **WSL — Windows Subsystem for Linux** | Windows上でLinux環境を利用する仕組み。 | 開発用UbuntuをWindows内で利用する。 |
| **WSL2** | Linuxカーネルを軽量な仮想マシン内で動かすWSLの方式。 | このデモの前提。`wsl -l -v` のVERSIONで確認する。 |
| **WSLg** | WSLのLinux GUIアプリをWindowsに表示する機能。 | QEMUのウィンドウやUbuntu上のFlutter試作画面を表示する。 |
| **ドライバー** | OSが特定のデバイスを扱うためのソフトウェア。 | WSLの `gs_usb` がUSB-CANを認識する。 |
| **カーネルモジュール** | カーネルに追加して読み込める機能。ドライバーなどを含む。 | `modinfo` で情報を調べ、`modprobe` で読み込む。 |
| **BusyBox** | 複数の基本コマンドを一つの実行ファイルにまとめたツール。 | AGLに含まれるが、AGL全体やカーネルの名前ではない。 |
| **systemd / サービス** | Linuxの起動や常駐プログラムを管理する仕組み / 管理される処理。 | `systemctl` でKUKSAやメーターを起動・停止し、`journalctl` でログを見る。 |
| **ターミナル / シェル** | 文字で操作する窓口 / 入力されたコマンドを解釈するプログラム。 | PowerShell、WSLのBash、SSH接続先のAGLを区別する。 |
| **root / sudo** | Linuxの管理者ユーザー / 別ユーザーの権限でコマンドを実行する仕組み。 | AGLへの接続はデモのroot。WSLでは必要な管理操作にsudoを使う。 |

参照: [Yoctoの概要](https://docs.yoctoproject.org/overview-manual/yp-intro.html)、[WSLの導入](https://learn.microsoft.com/ja-jp/windows/wsl/install)。実環境の版は [起動ガイド](../README.md) に記録しています。

<a id="virtualization"></a>

## 3. 仮想マシンとディスク

| 用語 | 意味 | このデモでの使い方 |
|---|---|---|
| **仮想マシン / VM** | ソフトウェアで用意したコンピューター環境。 | 仮想CPU・メモリ・ディスクを使ってAGLを起動する。 |
| **QEMU** | 仮想ハードウェアを用意し、OSを動かすソフトウェア。 | AGLのカーネルとディスクを読み込む。 |
| **KVM — Kernel-based Virtual Machine** | Linuxの仮想化支援機能。CPUの機能を使って仮想マシンを実行する。 | QEMUが `/dev/kvm` を利用する。 |
| **ホスト / ゲスト** | 仮想マシンを実行する側 / 仮想マシン内で動く側。 | QEMUから見たホストはWSLのUbuntu、ゲストはAGL。 |
| **アーキテクチャ** | CPUの命令体系などの分類。 | 配布版はx86-64。Raspberry Piへ移す際はARM向けの成果物が必要。 |
| **virtio** | 仮想マシン向けに設計されたデバイスの仕組み。 | 仮想ディスク、ネットワーク、画面などに使う。 |
| **ディスクイメージ** | ディスクやファイルシステムの内容を保存したファイル。 | 起動用AGLのデータ。スクリーンショット画像とは別の意味。 |
| **raw形式** | ディスク等の内容を、仮想ディスク専用の管理ヘッダーなしで保存する形式。 | 元の `.ext4` ファイルをQEMUがrawとして扱う。 |
| **ext4** | Linuxのファイルシステム形式。ファイルやディレクトリを保存する規則。 | 元のAGLイメージ内部の形式。`.ext4` はその内容を持つファイルの拡張子。 |
| **rootfs — Root Filesystem** | Linuxの `/` 以下の土台になるファイルシステム。 | AGLのコマンド、設定、ライブラリー、アプリなどが入る。 |
| **qcow2** | QEMUの仮想ディスク形式。必要に応じて領域を確保し、差分ディスクにも使える。 | `agl-work.qcow2` にAGLでの変更を保存する。qcow2自体が常に差分専用という意味ではない。 |
| **Copy-on-Write / オーバーレイ** | 元データを残し、書き換えた部分を別の場所へ記録する仕組み / 上に重ねる変更データ。 | 設定変更や追加アプリを作業用qcow2に保存する。 |
| **backing file** | 差分ディスクが参照する元のイメージ。 | 今回は元の `.ext4`。作業用qcow2だけでは動かず、元ファイルも必要。 |
| **スナップショット** | ある時点の状態を保存したもの。対象によって保存範囲が異なる。 | AGLの「master snapshot」は開発版の配布物。VMの実行状態保存とは別の意味。 |
| **xz / bzImage** | 圧縮形式 / この構成で使うLinuxカーネルイメージのファイル名。 | `.ext4.xz` を展開し、`bzImage` と組み合わせて起動する。 |
| **マウント** | ファイルシステムを特定のディレクトリから利用できるようにすること。 | 共有フォルダーや外部ディスクをLinux内で扱う際に使う。 |

今回のqcow2は元イメージを絶対パスで参照します。元ファイルを移動・削除すると参照できなくなります。参照: [QEMUのディスク形式](https://www.qemu.org/docs/master/system/images.html)。

<a id="ui"></a>

## 4. 画面とFlutter

| 用語 | 意味 | このデモでの使い方 |
|---|---|---|
| **UI — User Interface** | 利用者が見る表示や操作部分。 | 車速の数字、単位、アイコン、警告など。 |
| **Flutter** | 画面をウィジェットの組合せで作るUIフレームワーク。 | メーターパネルを実装する。 |
| **Dart** | Flutterアプリを記述するプログラミング言語。 | `.dart` ファイルに画面や処理を書く。 |
| **Widget（ウィジェット）** | Flutterで画面の構成を記述する基本単位。 | Text、Columnなどを組み合わせる。 |
| **状態 / State** | 画面表示を決める、その時点のデータ。 | 現在の車速、警告の有無、接続状態など。 |
| **Riverpod / Provider** | Dart・Flutterの状態や依存関係を管理するライブラリー / その仕組み。 | 公式メーターで、受信値を画面へ伝える処理に登場する。 |
| **Flutter Engine** | 描画やDartコードの実行などを担うFlutterの実行基盤。 | アプリのビルド成果物と対応する版が必要。 |
| **Embedder（エンベダー）** | Flutter EngineとOS側の画面・入力などをつなぐ部分。 | AGLでは `flutter-auto` を使ってFlutterアプリを動かす。 |
| **flutter-auto** | AGLで使われるFlutterの実行プログラム。 | systemdサービスからメーターの成果物を指定して起動する。 |
| **Hot Reload** | 実行中の状態を保ちながら、対応するコード変更を反映する開発機能。 | WSLで色やレイアウトを調整する。すべての変更が反映できるわけではない。 |
| **Hot Restart** | アプリのDart側を再起動し、初期状態から実行し直す開発機能。 | mainの変更や状態の初期化を反映するときに使う。 |
| **Wayland / Compositor** | Linuxの表示システムで使う通信方式 / アプリの表示面を画面へ配置するプログラム。 | AGLでは `agl-compositor` が表示を管理する。 |
| **アセット / assets** | アプリに含める画像、フォントなどの素材。 | メーターのアイコンや背景画像。`pubspec.yaml` で指定する。 |
| **Widgetテスト** | ウィジェットを動かし、文字表示や操作結果などを検査するテスト。 | 速度42の入力に対し「42」「km/h」が表示されるかを確認する。 |

参照: [Flutter UI](https://docs.flutter.dev/ui)、[Hot Reload](https://docs.flutter.dev/tools/hot-reload)。公式メーターの実装は [固定ソース一覧](REFERENCES.md) を参照してください。

<a id="build"></a>

## 5. ビルドと配置

| 用語 | 意味 | このデモでの使い方 |
|---|---|---|
| **ソースコード / ビルド** | 人が編集するプログラム / ソースや素材から実行・配布用の成果物を作る処理。 | Dartを編集してアプリをビルドする。OS全体のビルドとは規模が異なる。 |
| **SDK — Software Development Kit** | 開発に必要なツールやライブラリーのまとまり。 | Flutter SDKとAGL向けSDKは用途が異なる。 |
| **ツールチェーン** | コンパイラーやリンカーなど、プログラムを作るための道具一式。 | 対象CPUとAGLの環境に合わせて選ぶ。 |
| **クロスコンパイル** | 開発機とは異なる対象環境向けにプログラムを作ること。 | 例: x86-64の開発PCでRaspberry PiのARM向けに作る。 |
| **AOT / JIT** | Ahead-of-Time: 実行前にコンパイル / Just-in-Time: 実行時にコンパイル。 | Flutterのリリース用成果物と開発中の実行方式を理解する際に出てくる。 |
| **OpenEmbedded** | 組み込みLinuxのビルドに使うメタデータや仕組みを開発するプロジェクト。 | Yocto・AGLが利用するビルド基盤。 |
| **BitBake** | 依存関係と定義に従ってビルドの各作業を実行するツール。 | `bitbake flutter-cluster-dashboard` などで指定した対象を作る。 |
| **レシピ / .bb** | ソースの取得先、ビルド方法、配置先などを書いた定義。 | メーターアプリの作り方を指定する。 |
| **レイヤー / .bbappend** | 関連するビルド定義のまとまり / 既存レシピに設定や処理を追加するファイル。 | PIUS用の変更を独自レイヤーに残す。 |
| **BSP — Board Support Package** | 特定のボード向けの設定、ドライバー、ビルド定義など。 | Raspberry Pi向けのOSを作る際に使う。 |
| **eSDK / devtool** | 拡張可能なYocto SDK / アプリの変更・ビルド・配置を支援するツール。 | `devtool modify`、`build`、`deploy-target` を使う。 |
| **パッケージ / 依存関係** | インストールする単位 / 実行やビルドに必要な別のソフトウェア。 | メーター本体に加えて、Engineやフォントなどが必要。 |
| **デプロイ / Deploy** | 作った成果物を実行先へ配置すること。 | ビルドしたメーターをQEMU内のAGLへ配置する。配置だけでは動作確認完了ではない。 |

参照: [Yoctoの概要](https://docs.yoctoproject.org/overview-manual/yp-intro.html)、[devtool](https://docs.yoctoproject.org/ref-manual/devtool-reference.html)、[SDK](https://docs.yoctoproject.org/sdk-manual/extensible.html)。

<a id="can"></a>

## 6. CAN通信

| 用語 | 意味 | このデモでの使い方 |
|---|---|---|
| **CAN — Controller Area Network** | 複数の機器がバスを共有してデータを交換する通信方式。 | 車速や状態のデータを受け取る。 |
| **Classical CAN / CAN FD** | 従来のCAN / データ長拡張やデータ区間の高速化に対応した方式。 | 基本試験はClassical CAN。FDは機器・速度設定・処理側の対応確認が必要。 |
| **フレーム / ペイロード** | CANで送る一まとまりの情報 / その中のデータ部分。 | CAN IDとデータバイト列を観測する。 |
| **CAN ID / 標準ID / 拡張ID** | フレームの識別子 / 11ビット / 29ビットのID形式。 | デモの `0x3E9` は標準ID。送信機器の住所そのものではない。 |
| **DLC — Data Length Code** | フレームのデータ長を表すコード。 | Classical CANの通常のデータ長は最大8バイト。CAN FDは最大64バイトで、DLC値とバイト数が常に一致するわけではない。 |
| **ビットレート / bitrate** | 単位時間に送るビット数。 | 例: 500000 bit/s = 500 kbit/s。相手のバスに合わせる。 |
| **DBC** | CANのどのビットが何の信号か、倍率・単位などを定義するファイル形式。 | 生のバイト列を車速などの値へ変換するために使う。 |
| **エンディアン / 符号 / スケーリング** | データの並び方 / 負値の扱い / 倍率・オフセットによる換算。 | CANの生値を正しい物理量へ変換する条件。 |
| **SocketCAN** | CANをソケットやネットワークインターフェースとして扱うLinuxの仕組み。 | USB-CANを `can0` などの名前で利用する。 |
| **vcan / can0** | 物理機器を使わない仮想CAN / インターフェース名の一例。 | 名前がcan0でも物理CANとは限らない。今回のAGL側can0はvcan。 |
| **USB-CAN / CANnectivity / gs_usb** | USB接続のCANアダプター / 対応ファームウェア / Linux側の対応ドライバー。 | WindowsからWSLへ渡したUSB-CANを認識する。 |
| **CAN_H・CAN_L / 終端抵抗** | CANの差動信号線 / バス端の信号反射を抑える抵抗。 | ソフトウェア設定とは別に配線と終端を確認する。 |
| **ACK / listen-only** | フレームを正常受信したことをバス上で示す応答 / 送信やACKを行わない観測モード。 | 受信専用のアダプターは相手のACK役にならない。 |
| **BUS-OFF** | エラーが重なり、CANコントローラーが通信から離脱した状態。 | ビットレート、配線、終端、ACK不足などを調べる。 |
| **can-utils** | SocketCANの観測・送信などを行うコマンド集。 | `candump` は観測、`cansend` は指定フレーム送信、`cangen` は連続的なフレーム生成。 |

参照: [Linux SocketCAN](https://docs.kernel.org/networking/can.html)、[CANnectivity](https://github.com/CANnectivity/cannectivity)。実際の操作は [仮想CAN](CAN.md) / [USB-CAN](USB_CAN.md) の手順を使います。

<a id="data"></a>

## 7. 車両データと通信経路

| 用語 | 意味 | このデモでの使い方 |
|---|---|---|
| **信号 / Signal** | 車速、電池残量など、意味・型・単位を持つ個々のデータ。 | 一つのCANフレームに複数の信号が入ることがある。 |
| **VSS — Vehicle Signal Specification** | 車両の信号名・階層・型・単位などを定める仕様。 | 車速を `Vehicle.Speed` という名前で扱う。CANのビット位置を決めるDBCとは役割が異なる。 |
| **KUKSA Databroker** | 車両データを受け取り、アプリに取得・購読用APIを提供するサービス。 | CANの変換結果をFlutterへ渡す窓口。 |
| **kuksa-can-provider** | CANフレームから信号を取り出してKUKSAへ渡すプログラム。 | DBCと対応付けを使ってデータを変換する。 |
| **購読 / Subscribe** | データ配信を登録して、更新を継続して受け取ること。 | メーターが車速の更新を受け取る。変化がないことだけで通信断とは判断できない。 |
| **API / gRPC / Protocol Buffers** | ソフトウェア間の呼出し口 / 通信の仕組み / メッセージ定義とシリアライズの仕組み。 | 公式メーターとKUKSAの通信に使う。この版のアプリは `kuksa.val.v1` を利用。 |
| **cannelloni** | CANフレームをネットワーク経由で運ぶツール。 | WSLとAGLのCANをTCPで接続する。信号の意味を変換するものではなく、転送は双方向。 |
| **TCP / クライアント・サーバー** | 接続を確立して順序のあるデータ列を届ける通信方式 / 接続する側・待ち受ける側。 | WSL側のcannelloniがサーバー、AGL側がクライアント。 |
| **localhost / 127.0.0.1** | そのネットワーク環境の自分自身を指す名前・IPv4アドレス。 | AGL内のlocalhostと、WSL内のlocalhostを区別する。 |
| **ポート / ポート転送** | 通信先のサービスを区別する番号 / 指定した通信を別の宛先へ届ける設定。 | WSLの2222番からAGLのSSHの22番へ転送する。 |
| **USB/IP / usbipd-win** | USB通信をIPネットワーク越しに転送する仕組み / Windows側の実装。 | Windowsに挿したUSB-CANをWSLで使う。 |
| **BUSID / bind / attach / detach** | USB機器を識別する番号 / 共有登録 / 接続 / 切り離し。 | usbipdの操作。bindしただけではWSLへのattachは完了していない。 |

参照: [KUKSA](https://github.com/eclipse-kuksa/kuksa-databroker/blob/main/doc/user_guide.md)、[VSS](https://covesa.github.io/vehicle_signal_specification/)、[cannelloni](https://github.com/mguentner/cannelloni)、[WSLのUSB接続](https://learn.microsoft.com/ja-jp/windows/wsl/connect-usb)。

<a id="access"></a>

## 8. 接続・認証・共有

| 用語 | 意味 | このデモでの使い方 |
|---|---|---|
| **SSH — Secure Shell** | 暗号化した接続で別の環境を操作する仕組み。 | WSLからAGLのシェルに入る。SSHで入った後のコマンドはAGL内で実行される。 |
| **SCP / SFTP** | SSHを利用するファイル転送手段。 | 文書の `scp -O` は従来のSCP方式を明示。SFTPは接続先のSFTP機能が必要。 |
| **SSHFS / 共有フォルダー** | SSH経由で遠隔ファイルをマウントする仕組み / 複数環境から利用するフォルダー。 | 9pやvirtiofsもQEMUの共有手段。現構成ではこれらの共有は未設定。 |
| **ホスト鍵 / known_hosts** | SSHサーバーを識別する鍵 / 接続したサーバーの鍵を記録するファイル。 | AGLの接続先が変わっていないか検査する。 |
| **TLS / CA証明書** | 通信の暗号化と相手確認の仕組み / 信頼する認証局の証明書。 | KUKSAの接続先を確認するために使う。 |
| **認証トークン / JWT** | APIへのアクセス等に使う情報 / トークンを表現する形式の一つ。 | KUKSAの読み取り権限などに関係する。トークンを持つことと、全信号を操作できることは同じではない。 |
| **Git / リポジトリ** | ファイルの変更履歴を管理する仕組み / ソースや履歴の保管単位。 | 手順、設定、アプリの変更を追跡する。 |
| **commit / branch / tag** | 変更の記録 / 開発の分岐 / 特定のコミットを示す名前。 | 変更や配布対象の版を特定する。 |
| **clone / push** | リポジトリを手元へ複製 / ローカルのコミットをリモートへ送る操作。 | 文書を編集しただけではGitHubへ反映されない。 |
| **GitHub / Releases** | Gitリポジトリなどの共有サービス / 版ごとの配布ページ。 | 文書はリポジトリ、約852 MiBの起動イメージはReleasesに置く。 |
| **SHA-256 / チェックサム** | ファイル等から照合用の値を計算する方式 / 照合値。 | ダウンロードしたイメージが期待したファイルと一致するか確認する。署名とは別。 |
| **manifest / SPDX / SBOM** | 構成一覧を表す一般的な呼び名 / ソフトウェア情報の交換仕様 / ソフトウェア部品表。 | repo_manifest.xmlはソースの版、パッケージmanifestは収録一覧、SPDX文書は部品・ライセンス等の追跡に使う。 |

このデモの接続例は [起動ガイド](../README.md)、配布情報は [イメージの説明](../images/README.md)、仕様の参照先は [公式リンク一覧](REFERENCES.md) にまとめています。

<a id="differences"></a>

## 9. 混同しやすい言葉

| 組合せ | 違い |
|---|---|
| **IVI と Instrument Cluster** | IVIはナビ・音楽など、Clusterは車速・警告など。配布中はIVIデモ。 |
| **AGL と Yocto** | AGLは今回動かすシステム。Yoctoは、そのようなシステムを作るための仕組み。 |
| **Flutter と Dart** | FlutterはUIフレームワーク、Dartはコードを書く言語。 |
| **Flutter SDK と Engine** | SDKは開発用、Engineは実行・描画用。版や実行方式を合わせる必要がある。 |
| **WSL と QEMU** | WSLはWindows内のLinux環境。今回のQEMUは、そのLinux内から別のOSであるAGLを動かす。 |
| **WSLアプリの3.x と WSL2** | 前者はアプリのバージョン、後者はLinux環境の実行方式。数字の意味が異なる。 |
| **raw・qcow2 と ext4** | 前者はQEMUから見たイメージ形式、後者は中に保存するファイルシステム形式。 |
| **qcow2 と差分ディスク** | qcow2は形式。今回は元イメージを参照する差分ディスクとして使う。単独のqcow2も作れる。 |
| **WSLのcan0 と AGLのcan0** | 別OS内の別インターフェース。名前が同じだけで自動接続されない。 |
| **CAN ID と VSSパス** | CAN IDはフレームの識別子。VSSパスは車速などの信号の意味を表す名前。 |
| **DBC と VSS** | DBCはCANのビット配置や換算。VSSはアプリで扱う信号名・型・単位。 |
| **CAN受信成功 と メーター動作成功** | フレーム到着、KUKSAの値更新、画面更新を別々に確認する。 |
| **ファイル転送 と デプロイ完了** | コピーできても、依存関係・権限・起動設定・動作確認が必要。 |

<a id="examples"></a>

## 10. 実際の名前の読み方

| 文書に出てくる名前 | 読み解き方 |
|---|---|
| `agl-ivi-demo-flutter-qemux86-64-20261001034413.ext4.xz` | AGLのIVIデモ、Flutter版、QEMU x86-64用、生成時刻を含む名前、ext4内容をxz圧縮したファイル。 |
| `agl-work.qcow2` | このPCでの変更を保存する作業用ディスク。元のext4も必要。 |
| `flutter-ics-homescreen` | 現在のIVIホーム画面のアプリ・サービス名。 |
| `flutter-instrument-cluster` | 公式Flutterメーターのソースリポジトリ名。 |
| `flutter-cluster-dashboard` | 上記ソースを作るレシピ・パッケージ・サービスの名前。 |
| `Vehicle.Speed` | KUKSAで扱うVSSの車速パス。今回のデモではkm/hで確認する。 |
| `127.0.0.1:2222` | WSL内から接続する、AGLのSSHへの転送入口。 |
| `10.0.2.2:20000` | 今回のQEMUのネットワーク構成で、AGLからホスト側のCAN転送サーバーへ接続する宛先。 |
| `localhost:55555` | AGL内から見たKUKSAの接続先。認証とTLSの設定も必要。 |

数字や名前を覚えるよりも、**どのOSで実行しているか、どこへ接続しているか、何を保存しているか**を確認しながら手順を読むと理解しやすくなります。
