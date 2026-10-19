# RSH-A10 の Windows 11 (Home) における可用性調査

- 対象 Issue: [#1 Windows環境における可用性調査](https://github.com/bd8z/usbhubctl/issues/1)
- 対象機: [RSHTECH RSH-A10](https://de.rshtech.com/products/10-ports-aluminum-usb-30-data-hub-with-12v-3a-power-adapterrsh-a10)（10 ポート USB 3.0 データハブ、12V/3A 電源付き）
- 調査日: 2026-10-19
- 更新: 2026-10-20（WSL2 + usbipd-win による実質ルートを 3.5 に追加、チップ数の記述を修正（[7. 内部トポロジ](#7-内部トポロジ参考)）、「デバイス無効化」の記述を緩和（[3.2](#32-ポート毎の電源切り替えプログラム制御-uhubctl--usbhubctl)））
- 制約: **購入前の静的調査のみ**。デバイスはホスト PC に接続されていないため、動的検証は行っていない（到着後の検証チェックリストは [6. 到着後の検証チェックリスト](#6-到着後の検証チェックリスト) を参照）。

## 1. 結論（要約）

| 用途 | Windows 11 (Home) での可否 | 備考 |
| - | - | - |
| 10 ポート USB 3.0 ハブとしてのデータ用途（ポート拡張） | **使える** | ドライバ不要・PnP。自己給電のため PC 側の電力予算を食わない |
| 物理スイッチによるポート毎の電源切り替え | **使える** | OS 非依存（ハードウェアスイッチ）。ただし [4. 懸念](#4-懸念事項) の 3〜5 を確認すること |
| プログラムによるポート毎の電源切り替え（uhubctl / usbhubctl） | **ネイティブでは使えない** | uhubctl・本リポジトリの usbhubctl は Linux 専用。Windows に汎用ハブのポート毎電源制御を行う標準的な手段は存在しない |
| プログラムによるポート毎の電源切り替え（WSL2 と usbipd-win を経由） | **有望（到着時の確認が必要）** | WSL2 の Linux 上で usbhubctl がそのまま使える見込み。ただしデバイスが WSL2 と Windows で排他的になる等、運用上の注意点あり（[3.5](#35-wsl2-と-usbipd-win-を経由して-usbhubctl-を実行する場合)） |

**購入判断の指針**:

- 「USB 3.0 ポート拡張 + 物理スイッチでポート単位に電源を切り替える」のが目的なら → **購入して問題ない**（懸念を確認する）。
- 「このプロジェクト（usbhubctl）で Windows 11 のマシンからプラグ単位に電源を切りたい」のが目的なら → **ネイティブ（Windows 上で直接）は不可**だが、**WSL2 と usbipd-win の経路（[3.5](#35-wsl2-と-usbipd-win-を経由して-usbhubctl-を実行する場合)）を通せば利用可能になり得る**。この経路の許容度（デバイス排他・再起動毎の再アタッチ等）次第で判断すること。到着時の検証で実効性を確認すること。

## 2. デバイス概要（ベンダー情報より）

- ポート: USB 3.0 (SuperSpeed 5Gbps) × 10、USB 2.0/1.1 へ後方互換
- 外形: アルミ + ABS、17 × 4.6 × 2.5 cm
- 電源: 外付け AC アダプタ 12V 3A（36W、UL 認証）。ハブ本体は**自己給電（Self Powered）**
- ケーブル: USB 3.0 データケーブル 1m（取り外し可）
- スイッチ: **各ポートに独立した On/Off スイッチ + LED 表示**（ケーブル抜きなしでポートの電源を切れる）
- ベンダー記載の OS 互換: Windows 2003/XP/Vista/7/8/10、macOS、Linux（**Windows 11 の明記はない**。詳細は [3.1](#31-基本ハブ機能データ--ポート拡張)）
- 同梱: ハブ本体、1m ケーブル、12V/3A アダプタ、マニュアル
- 参考: 本リポジトリの [usb_hubs/README_RSHTECH_RSH-A10.md](../../usb_hubs/README_RSHTECH_RSH-A10.md) に同系ハブ（RSH-A10/RSH-A16）の実測評価がある

## 3. Windows 11 (Home) での可用性（用途別）

### 3.1 基本ハブ機能（データ / ポート拡張）

**結論: 使える。**

- RSH-A10 の内蔵ハブは Realtek 製（USB 3.2: `0bda:0411`、USB 2.1 対応ハブ: `0bda:5411`）の汎用ハブ。Windows 11 に同梱の xHCI ハブドライバ（`usbhub.sys` / `usbhub3.sys`）で認識され、**サードパーティードライバは不要**（ベンダーも「no drivers needed / Plug & play」としている）。
- ベンダーの互換リストに Windows 11 が無いのは、マーケティング情報の古い記述と判断してよい。Windows 11 の USB スタックは Windows 10 と同じ xHCI 方式であり、Win 10 で動作する汎用ハブが Win 11 で動かないことは原理として考えにくい（個別の不具合は排除できない）。
- ハブは自己給電（設定記述子 `bmAttributes = Self Powered`, `MaxPower = 0mA`）のため、PC 側のバス電力予算（USB ポートあたり 900mA 等）を消費せず、Windows の電力制限で動作が不安定になる要因もない。
- データ速度: PC 側のポートが SuperSpeed（USB 3.x/USB4）であれば 5Gbps。USB 2.0 ポートに差すと 480Mbps になる。
- 注意点: Windows 11 の USB セレクティブサスペンド（電源の省電力化）により、アイドル中の下流デバイスをサスペンドする設定が有効。このハブの**ポート電源自体**を切ることとは別物だが、接続デバイス（例: 外付けドライブ）の動作に影響し得る。常時動作が必須のデバイスには、デバイスのプロパティで「このデバイスをシャットダウンするためにコンピュータに許可をしないでください」のチェックを外す（= 許可しない）こと。

### 3.2 ポート毎の電源切り替え（プログラム制御: uhubctl / usbhubctl）

**結論: Windows では使えない。このプロジェクトの主要ユースケースが Windows では実現できない。**

理由（技術的根拠）:

1. **uhubctl は Linux 専用。** `/sys/bus/usb/devices/...-portN/disable` への書込みや、root 権限での生コントロールトランスファによる実装であり、Windows に相当する機構は存在しない。
2. **本リポジトリの usbhubctl も Linux 専用。** 実装の各バックエンドが Linux 前提:
   - 探索: `backend_query_lsusb.py` が `lsusb -tv` を実行してパース（Windows には `lsusb` がない）
   - 電源切替: `backend_power_sysfs.py` が `/sys/bus/usb/devices/.../disable` へ書込み（`Path.sysfs_path`, `src/usbhubctl/usbhubctl.py:284`）
   - 電源切替（旧）: `backend_power_uhubctl.py` が `uhubctl` バイナリを呼ぶ
   - `pyproject.toml` の classifier も `Operating System :: POSIX :: Linux` のみ
3. **Windows に「汎用ハブのポート毎電源切替」の標準的なユーザモード機構がない。** USB 仕様の `HUB_PORT_POWER`（`SET_FEATURE`）でポート電源を切り替えられるが、Windows ではハブデバイスをカーネルモードのハブドライバ（`usbhub3.sys` 等）が占有しており、ユーザモードからハブに対する生コントロールトランスファ（pyusb / WinUSB 経由）を送ることはサポートされていない。Linux と異なり `/dev` 経由の生アクセスが存在しない。
4. Device Manager の「デバイスの無効化」はデバイス／ドライバレベルの操作であり、ポートごとの制御を提供しない（ハブチップを無効化するとその下流全体が外された扱いになる）。ポートの給電が切られるかデータだけ止まるかはホストコントローラ・デバイス依存であり、usbhubctl のようなポート単位の確実な切替の代替にはならない。
5. 将来的な Windows サポートには、ハブへの生コントロールトランスファが可能なカーネルドライバ等の新規実装が必要で、現在の実装コスト・リスクは大きい（本 Issue の範囲外）。

### 3.3 ポート毎の電源切り替え（物理スイッチ）

**結論: 使える（OS 非依存）。ただし確認事項あり。**

- 各ポートの On/Off スイッチはハードウェア実装のため、Windows でも Linux と同じ挙動を期待できる。
- ただし、本リポジトリの同系ハブ（RSH-A10/RSH-A16）の評価（[usb_hubs/README_RSHTECH_RSH-A10.md](../../usb_hubs/README_RSHTECH_RSH-A10.md) の Assessment）に以下の報告がある:
  - 「Power on/off ONLY works when the button is in pressed state」— ボタンを押している間のみ ON/OFF が効く、という報告。個体差・機種差の可能性があり、**到着時に必ず実機確認**する。
  - LED が暗く見にくい（LED による状態確認が苦手な場合、デバイスの接続/切断で状態を見る）。
  - Linux 上で uhubctl による切替は約 2.5 秒と遅い（Linux 側の話。物理スイッチはこれとは別）。
- 切り替え時の挙動: ポート電源 OFF にすると下流デバイスは外された扱いになり（接続検知リセット）、ON にすると再接続扱いになる。Windows のデバイス再検出は自動だが、デバイス依存で再接続までに数秒かかる場合がある。

### 3.4 Windows 11 (Home) 固有の考慮事項

- **Home 版固有の制限は USB ハブにない。** Pro 版の Group Policy 等も USB ハブの動作には関係しない。
- **Fast Startup（高速スタートアップ）**: ハイブリッドシャットダウンではカーネルセッションの状態が保存され、USB 状態が前回から残る既知の怪異（ポート状態が想定と異なる等）が起き得る。異常が出たら一度「再起動」（シャットダウン + 電源投入ではない）で確認する。
- **スリープ / ハイバーレーション**: PC 休止で USB データリンクは切断される（12V アダプタは通電したまま）。復帰時に接続デバイスの再接続・初期化が必要になる。ハブ側のポート電源状態は機器仕様上どうなるか不明（実機確認）。
- **2.4GHz 無線干渉**: ベンダー自身が「USB 3.0 データ通信中の 2.4〜2.5GHz ノイズで無線マウス / キーボードの受信が不安定になり得る。受信機は USB 2.0 ポートに接続するか距離を置く」と案内している。OS に依存しない一般論だが、無線ペリフェラルを併用する場合は注意。
- **ポート番号の把握**: 物理ポートに番号表記がない（同系ハブの評価も同一）。Windows の Device Manager ではハブ内のポート番号が直接見えない（`USB3.2 Hub` インスタンスとして見えるだけ）。物理プラグ ↔ ポート番号のマッピングは、Device Manager でのデバイスの抜き差し + 第 3 者ツール（USBTreeView 等）での探索が必要。このマッピングの自動化が本プロジェクトの目的だが（[7. 内部トポロジ](#7-内部トポロジ参考)）、それは Linux 側でのみ機能する。
- **認証 / 正規品**: ベンダージャンルのページには「Package contents: ROSONWAY 10-Port USB 3.0 Hub」と別ブランド名（Rosonway）の記載が混在しており、同硬件が複数ブランドで販売されている（ホワイトラベル）ことを示唆する。AliExpress（USD35）から購入する場合、本リポジトリの実測データと完全同一ロットとは限らない点は留意すること。

### 3.5 WSL2 と usbipd-win を経由して usbhubctl を実行する場合

**結論: 本プロジェクトのユースケース（プログラムによるポート毎電源切替）を Windows 11 のマシンで実現する現実的なルート。**

仕組み:

- WSL2（Windows Subsystem for Linux）は完全な Linux カーネルを軽量な仮想環境で実行する。Microsoft の公式ドキュメントでは Windows 10 (Build 19041+) または Windows 11 での利用を前提としており、**エディション（Home / Pro）による制限の記載はない**（Windows 11 Home でも利用可能）。
- [usbipd-win](https://github.com/dorssel/usbipd-win)（GPL-3.0、`winget install usbipd` 等で導入）が、Windows 側で接続された USB デバイスを WSL2 の Linux 側に USB/IP プロトコルで共有する。Windows 10 x64/ARM64 (1809+) 以上で動作。
- WSL2 の中では、通常のリナックス USB スタック（`lsusb -tv`、`/sys/bus/usb/devices/.../disable` への書込み、root 権限）がそのまま利用可能になるため、**本リポジトリの usbhubctl は Linux 上で動作する前提どおり（`known_hubs.rsh_a10` を含め）そのまま使える見込み**。
- 使い方（要約）: 管理権限で `usbipd bind --busid=<BUSID>`（共有は再起動後も維持）し、WSL2 には `usbipd attach --wsl --busid=<BUSID>`（管理権限不要）でアタッチする。

注意点・制約:

- **デバイス排他**: アタッチ中は Windows 側からはそのデバイス（ハブおよび下流の接続デバイス）が見えなくなる。ハブを Windows 側でデータ用途として使うときと、usbhubctl で使うときとでは、アタッチ／デタッチの切替が必要になる（切替は即時だが、運用上の考慮事項）。
- **アタッチの非持続性**: 再起動・デバイスの抜き差し・リセット後は再アタッチが必要（bind は持続するが attach はしない）。
- **ハブ構成との相性**: RSH-A10 は 6 つのハブチップ（[7. 内部トポロジ](#7-内部トポロジ参考)）からなり、Windows 上では複数インスタンスとして見える。ルートハブのアタッチで下流のチップ・接続デバイスが WSL2 側に見えて、Linux 実機と同一の `lsusb -tv` トポロジになるかは、**到着時の確認が必要**（[6. 到着後の検証チェックリスト](#6-到着後の検証チェックリスト) に追加した項目を参照）。
- **WSL2 カーネルの USB サポート**: 公式 Wiki は「最新の WSL2 カーネルはほとんどの USB デバイスに対応」としている（`wsl --update` で更新）。Realtek の汎用ハブチップ（`0bda:0411` / `0bda:5411`）は Linux の標準ハブドライバで認識されるため、問題ない見込み。
- **サードパーティ製ソフトウェアの導入**: usbipd-win は Windows 側にサービスを追加する（TCP 3240 のファイアウォールルールも）。セキュリティポリシーが厳しい環境では要確認。

この経路の成立により、[1. 結論](#1-結論要約) の見立ては**「プログラムによるポート毎電源切替はネイティブ Windows では不可だが、WSL2 を経由すれば Windows 11 のマシンでも利用可能になり得る」**となる。

## 4. 懸念事項

| # | 懸念 | 影響 | 重さ | 対応策 |
| - | - | - | - | - |
| 1 | ベンダー互換リストに Windows 11 の明記がない（Win 10 まで） | 低。汎用ハブ・ドライバ不要のため Win 11 で動作する可能性は高いが、明示的検証事例が無い | 低 | 到着後に Device Manager で認識確認 |
| 2 | プログラムによるポート毎電源切替がネイティブ Windows では不可能（uhubctl / usbhubctl は Linux 専用、Windows には標準機構がない） | **高。ネイティブ運用では本プロジェクトの主要ユースケースが成立しない** | 高 | プログラム制御が必要なら WSL2 + usbipd-win 経路（[3.5](#35-wsl2-と-usbipd-win-を経由して-usbhubctl-を実行する場合)）または Linux ホスト（デュアルブート等）で運用。ネイティブのみが前提なら「物理スイッチ」に限定 |
| 3 | 同系ハブの実測で「ボタンを押している間のみ ON/OFF が効く」報告 | 中。物理スイッチ頼みの運用が前提なら重要 | 中 | 到着時に全ポートで実機確認 |
| 4 | ポートに番号表記がない。Windows では物理プラグ ↔ ポート番号のマッピングが手作業 | 中 | 中 | 到着時にマッピングを作成し、本リポジトリの `known_hubs` 定義と突合（Linux 側で検証） |
| 5 | LED が暗く見にくい（同系ハブの実測） | 低〜中 | 低 | 状態確認は接続デバイスの挙動で行う |
| 6 | Fast Startup / スリープ復帰時の USB 状態の怪異 | 低〜中 | 低 | 異常時に「再起動」で確認。必要なら Fast Startup を無効化 |
| 7 | USB 3.0 の 2.4GHz 無線干渉（ベンダー公認の挙動） | 低（無線ペリフェラル使用時のみ） | 低 | 無線受信機は USB 2.0 ポートに接続 / 距離を確保 |
| 8 | ホワイトラベルで複数ブランド販売（Rosonway 等の混載） | 低〜中。ロット差で挙動が異なる可能性 | 低 | 到着時に外観・シリアル・挙動を確認 |
| 9 | USB セレクティブサスペンドによる下流デバイスのサスペンド | 低（常時通信デバイス使用時のみ） | 低 | デバイスの電源管理で「シャットダウン許可」を切る |
| 10 | WSL2 + usbipd-win 経路の制約（デバイス排他・再起動毎の再アタッチ・サードパーティ製サービスの導入） | 中。プログラム制御を WSL2 経由で使う運用が前提なら重要 | 中 | 運用方法で許容度を判断。到着時に [6. チェックリスト](#6-到着後の検証チェックリスト) の WSL2 関連項目を実行 |

## 5. 推奨アクション

1. **購入判断**:
   - 物理スイッチでのポート電源切替 + USB 3.0 ポート拡張が目的 → **購入可**（懸念 3〜8 の確認を残条件とする）。
   - Windows 11 のマシンからプログラム制御することが目的 → ネイティブでは不可だが、**WSL2 + usbipd-win 経路（[3.5](#35-wsl2-と-usbipd-win-を経由して-usbhubctl-を実行する場合)）の利用許容度次第で購入可**。その経路を許容できない場合は、Linux ホストで運用するか、Windows ネイティブでのプログラム制御を実現する別手段（カーネルドライバ等）を別途検討する。
2. **到着後**: [6. 検証チェックリスト](#6-到着後の検証チェックリスト) を実行し、結果を本ドキュメントに追記する。
3. **長期検討（任意）**:
   - WSL2 + usbipd-win 経路を正式な使い方として確立する場合は、本リポジトリ側に「WSL2 環境での起動方法・注意点」ドキュメントを追加する。
   - Windows ネイティブ用の電源切替バックエンド（`BackendPowerABC` の実装）の feasibility を別途 Issue 化して評価する。現状の技術制約（[3.2](#32-ポート毎の電源切り替えプログラム制御-uhubctl--usbhubctl)）から、実装コストは大きいと考える。

## 6. 到着後の検証チェックリスト

（動的調査の計画。購入前に実行可能な項目はなし。）

- [ ] Device Manager: `USB3.2 Hub`（`0bda:0411`）× 3 と `USB2.1 Hub`（`0bda:5411`）× 3 の 6 インスタンスが認識されること（[7. 内部トポロジ](#7-内部トポロジ参考) の構成）
- [ ] ハブが「自己給電」で認識されること（アダプタ接続時）
- [ ] 全 10 ポートで: スイッチ ON → LED 点灯 + 接続デバイスの認識。スイッチ OFF → 接続デバイスが外された扱いになること
- [ ] 「ボタン長押しでないと効かない」挙動（懸念 3）の個体確認
- [ ] 物理プラグ ↔ ポート番号のマッピング作成（Device Manager + USB 探索ツール）
- [ ] スーパースピード接続時の転送速度確認（5Gbps クラス）
- [ ] スリープ復帰 / Fast Startup 前後のポート状態確認（懸念 6）
- [ ] （WSL2 経路）Windows 11 Home で `wsl --install` が成功し、WSL2（`wsl -l -v` で VERSION 2）が使えること
- [ ] （WSL2 経路）`winget install usbipd` 後に `usbipd list` でハブの各チップ（`0bda:0411` / `0bda:5411`）の busid を確認できること
- [ ] （WSL2 経路）ルートハブを `usbipd bind` + `usbipd attach --wsl` した状態で、WSL2 内で `lsusb -tv` が 6 チップ構成（[7. 内部トポロジ](#7-内部トポロジ参考)）を示すこと
- [ ] （WSL2 経路）WSL2 内で本リポジトリの `tests/test_live.py` の `test_toggle_rsh_a10` が全ポートの切替に成功すること
- [ ] （WSL2 経路）アタッチ中 / デタッチ時の Windows 側でのデータ用途との切り替わりが想定どおりであることの確認（懸念 10）
- [ ] （可能なら）Linux 実機で本リポジトリの `known_hubs.rsh_a10` トポロジ定義と実機が一致することの確認（`tests/test_live.py` の `test_toggle_rsh_a10`）

## 7. 内部トポロジ（参考）

本リポジトリの定義 [`src/usbhubctl/known_hubs/rsh_a10.py`](../../src/usbhubctl/known_hubs/rsh_a10.py) と、同系ハブの実測（[usb_hubs/README_RSHTECH_RSH-A10.md](../../usb_hubs/README_RSHTECH_RSH-A10.md) の Internals）より:

- ハブチップ 6 枚: USB 3.2 ハブ（`0bda:0411`, 4 ポート）× 3 と USB 2.1 対応ハブ（`0bda:5411`, 4 ポート）× 3（`usb_hubs/README_RSHTECH_RSH-A10.md` の `dmesg` 出力と一致）。1 枚のチップは 1 つの上流ポート + 最大 4 つの下流ポートを持つ。
- 各物理プラグ（計 10）は USB 3.2 経路および USB 2.1 経路の双方に接続されている（USB 3.x ハブの標準的な構成）。
- ルートチップの下流ポート割当（`rsh_a10.py`）:
  - ポート 1 → プラグ 1
  - ポート 2 → プラグ 2
  - ポート 3 → サブハブ（プラグ 7, 8, 9, 10）
  - ポート 4 → サブハブ（プラグ 3, 4, 5, 6）
- ハブ記述子: ポート毎電源スイッチ / ポート毎過電流保護あり（`wHubCharacteristic = 0x0009`）、USB 3.2 ハブは ContainerID を報告（Windows の USB 3.x デバイス管理に有利）。

## 8. 参照

- ベンダー製品ページ: <https://de.rshtech.com/products/10-ports-aluminum-usb-30-data-hub-with-12v-3a-power-adapterrsh-a10>
- AliExpress 掲載: <https://www.aliexpress.com/item/1005006336855327.html>（USD35）
- 同系ハブの実測（本リポジトリ）: [usb_hubs/README_RSHTECH_RSH-A10.md](../../usb_hubs/README_RSHTECH_RSH-A10.md) / [usb_hubs/README_RSHTECH_RSH-A16.md](../../usb_hubs/README_RSHTECH_RSH-A16.md)
- トポロジ定義: [src/usbhubctl/known_hubs/rsh_a10.py](../../src/usbhubctl/known_hubs/rsh_a10.py)
- 電源切替バックエンド（Linux 専用）: [src/usbhubctl/backend_power_sysfs.py](../../src/usbhubctl/backend_power_sysfs.py) / [src/usbhubctl/backend_power_uhubctl.py](../../src/usbhubctl/backend_power_uhubctl.py)
- uhubctl（Linux）: <https://github.com/mvp/uhubctl>
- WSL のインストール要件（Windows 10 2004+ / Windows 11、エディション制限の記載なし）: <https://learn.microsoft.com/en-us/windows/wsl/install>
- usbipd-win（Windows → WSL2 への USB シェア、GPL-3.0）: <https://github.com/dorssel/usbipd-win>
