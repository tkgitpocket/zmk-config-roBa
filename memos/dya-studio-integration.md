# DYA Studio 対応（ZMK 4系移行）の変更内容

> ブランチ: `dya-studio-level2`
> 対応レベル: DYA Studio Level 2（マクロ/コンボ編集・トラックボール設定・BLEプロファイル管理）
> 参考: [DYA Studio 開発者ガイド](https://studio.dya.cormoran.works/developer-guide)（SPAのため取得できず）、
> [zmk-keyboard-dya-dash](https://github.com/cormoran/zmk-keyboard-dya-dash)、torabo-tsuki-lp の `dya-studio-level2` ブランチ。

キーマップ（レイヤー構成・ビヘイビア）は変更していない。**静的コンボのみ削除**（後述）。

## 1. ZMK本体 / ビルド基盤

| 項目 | 変更前 | 変更後 |
|---|---|---|
| ZMK | `zmkfirmware/zmk` `v0.3-branch`（Zephyr 3.5） | `cormoran/zmk` `main+dya`（ZMK 4系 / Zephyr 4.1） |
| ボード指定（`build.yaml`） | `seeeduino_xiao_ble` | `xiao_ble//zmk`（Zephyr 4.1 のZMKボードバリアント） |
| ワークフロー（`.github/workflows/build.yml`） | `build-user-config.yml@v0.3-branch` | `build-user-config.yml@main` |

- v0.3-branch+dya でも一度ビルド成功させたが、マクロ/コンボ画面でDYA Studioがフリーズしたため4系に移行した。
- Zephyr 4.1 にはネイティブの `pixart,pmw3610` ドライバがあり、旧ドライバ（kumamuk-git）とバインディングが衝突するため、ドライバも差し替えた（§2）。

## 2. トラックボールドライバ

- `kumamuk-git/zmk-pmw3610-driver` → `cormoran/zmk-driver-pmw3610-with-custom-studio-rpc`（`config/west.yml`）。
- `roBa_R.overlay`: `compatible = "cormoran,pmw3610"`、`cpi = <400>`、`evt-type` / `x-input-code` / `y-input-code`、`settings-id = "trackball"` を追加。`scroll-layers` は新ドライバに無いため削除。
- `roBa_R.conf`: 旧ドライバ専用のKconfig（`CPI_DIVIDOR` / `ORIENTATION_*` / `SCROLL_TICK` / `INVERT_SCROLL_X` / `POLLING_RATE_125_SW` / `AUTOMOUSE_TIMEOUT_MS` / `MOVEMENT_THRESHOLD` ほか）を削除し、新ドライバ向けに以下を設定。
  - `PMW3610_INVERT_X=y`（実機でXY両方反転していたためINVERT_Yから変更）、`PMW3610_REPORT_INTERVAL_MIN=12`、`PMW3610_INIT_POWER_UP_EXTRA_DELAY_MS=200`
  - `PMW3610_RUN_DOWNSHIFT_TIME_MS=3264`、`PMW3610_REST1_SAMPLE_TIME_MS=20`、`PMW3610_SMART_ALGORITHM=y`
  - `ZMK_PMW3610_CUSTOM_SETTINGS=y`（DYA Studioからセンサー設定を変更可能）
  - `ZMK_PMW3610_STUDIO_RPC=y` と `ZMK_PMW3610_SPLIT_RPC_RELAY=y`（§8のバッファ拡張とセット。単独だとTXバッファ不足でビルドエラーになる）

### スクロールの実装変更（要実機確認）

旧ドライバはセンサー側でスクロール値を出していた（`scroll-layers = <5 10 11>`、`SCROLL_TICK=16`）。新ドライバは常にXY移動を出すので、`roBa_R.overlay` の `trackball_listener` で `zip_xy_to_scroll_mapper` によりXY→スクロールに変換する（マクロ `SCROLL_CHAIN`）。

| レイヤー | 処理 |
|---|---|
| 5 SCROLL_L | mapper → Y反転 → `zip_scroll_scaler 1 16`（旧TICK=16相当） |
| 10 SLOWSCROLL_L | 同上、`1 32`（標準の1/2） |
| 11 FASTRSCROLL_L | 同上、`3 8`（標準の6倍） |

スクロール方向・感度、カーソルの向き（`INVERT_Y`）は実機で確認し、必要なら `INPUT_TRANSFORM_*` / `INVERT_*` で調整する。

## 3. DYA Studio用モジュール（`config/west.yml`）

すべて `cormoran` リモートの `main`。

- `zmk-module-ble-management`（BLEプロファイル管理）
- `zmk-module-settings-rpc`（設定同期）
- `zmk-module-runtime-input-processor`（トラックボール速度のランタイム調整）
- `zmk-feature-custom-settings` / `zmk-feature-runtime-combo` / `zmk-feature-runtime-macro`
- `zmk-feature-input-stream`（入力ストリーム表示）/ `zmk-feature-device-info`（機体情報）
- `zmk-module-devtool`（診断：Studioロック状態の確認/切替など）

## 4. Kconfig

`boards/shields/roBa/roBa_R.conf`（central）に追加:
`BT_MAX_CONN=5` / `BT_MAX_PAIRED=5`、`ZMK_BLE_MANAGEMENT(_STUDIO_RPC)`、`ZMK_SETTINGS_RPC(_STUDIO)`、`ZMK_SPLIT_RELAY_EVENT`、`ZMK_SPLIT_BLE_CENTRAL_SPLIT_RUN_STACK_SIZE=768`、`ZMK_SETTINGS_SAVE_DEBOUNCE=10000`、`ZMK_RUNTIME_INPUT_PROCESSOR(_STUDIO_RPC)`、`ZMK_RUNTIME_COMBO(_STUDIO_RPC)`、`ZMK_RUNTIME_MACRO(_STUDIO_RPC)`、`ZMK_CUSTOM_SETTINGS_SPLIT_RPC_RELAY`、`ZMK_CUSTOM_SETTINGS_LARGE_VALUE_MAX_SIZE=256`、`ZMK_INPUT_STREAM_FEATURE(_STUDIO_RPC)`、`ZMK_DEVICE_INFO(_STUDIO_RPC)`、`ZMK_DEVTOOL(_STUDIO_RPC)`。

`roBa_L.conf`（peripheral）に追加: `BT_MAX_CONN=5` / `BT_MAX_PAIRED=5`、`ZMK_SETTINGS_RPC`、`ZMK_SPLIT_RELAY_EVENT`、`ZMK_SPLIT_BLE_CENTRAL_SPLIT_RUN_STACK_SIZE=768`、`ZMK_SETTINGS_SAVE_DEBOUNCE=10000`。

**`CONFIG_ZMK_STUDIO_LOCKING=n` → `y`**: DYA StudioがBLEでデバイスを検出する仕組みがロック解除の動作をトリガーにしているため。BTレイヤー（レイヤー6）の `&studio_unlock` を押してから接続する。

## 5. トラックボール速度のランタイム調整（`roBa_R.overlay`）

`zmk,input-processor-runtime` ノードを2つ追加し、`trackball_listener` の各processorリストの末尾に連結。既定は1/1で、DYA Studioから変更するまで既存の速度は変わらない。

- `trackball_speed_rip`（ラベル `tball`）: カーソル移動用
- `trackball_scroll_rip`（ラベル `tscroll`）: スクロール用

## 6. 静的コンボの削除（`config/roBa.keymap`）

DYA Studioでランタイム定義できるため、`combos { ... }` ノードを削除した。**DYA Studio側で再登録が必要**（位置はキー位置番号）。

| 名前 | 出力 | キー位置 |
|---|---|---|
| tab | `&kp TAB` | 11 12 |
| shift_tab | `&kp LS(TAB)` | 12 13 |
| mb4 | `&mkp MB4` | 6 7 |
| mb5 | `&mkp MB5` | 7 8 |
| semicolon | `&kp SEMICOLON` | 18 19 |
| colon | `&kp COLON` | 19 20 |
| underscore | `&kp UNDERSCORE` | 20 21 |
| comma | `&kp COMMA` | 30 31 |
| dot | `&kp DOT` | 31 32 |
| asterisk | `&kp ASTERISK` | 17 18 |
| plus | `&kp PLUS` | 29 30 |
| mb3 | `&mkp MCLK` | 6 8 7 |
| esc | `&kt ESC` | 0 1 |
| lctrl | `&kt LCTRL` | 10 11 12 |

（旧定義はgit履歴の `config/roBa.keymap` で確認できる）

## 7. 既知の注意点

- 4系はZMK本家の `zmk-layout-shift v2` などの外部モジュールとの互換が未確認の箇所がある（ビルドは成功）。実機でJIS変換が効くか確認すること。
- AML（`zip_temp_layer`）やBTプロファイル自動レイヤー切替は4系でもビルドは通っているが、実機での挙動は要確認。

## 8. 安定化設定（dya-dash に合わせた追加）

トラボ設定画面・マクロ/コンボ画面でフリーズしたため、安定動作している [zmk-keyboard-dya-dash](https://github.com/cormoran/zmk-keyboard-dya-dash) の `dya_dash_right.conf` / `dya_dash_left.conf` に合わせた。

- バッファ/スタック拡張（right）: `ZMK_STUDIO_RPC_RX/TX_BUF_SIZE=256`、`..._CUSTOM_SUBSYSTEM_REQUEST_PAYLOAD_MAX_BYTES=256`、`ZMK_SPLIT_RELAY_EVENT_DATA_LEN=240`、`SYSTEM_WORKQUEUE_STACK_SIZE=4096`、`ZMK_STUDIO_RPC_THREAD_STACK_SIZE=6000`、`ZMK_LOW_PRIORITY_THREAD_STACK_SIZE=4096`（left は 2048）
- `ZMK_CUSTOM_SETTINGS(_STUDIO_RPC)`、`ZMK_BEHAVIOR_LOCAL_ID_TYPE_CRC16` / `ZMK_BEHAVIOR_LOCAL_IDS_IN_BINDINGS`（ランタイムmacro/comboのビヘイビア参照用）
- 追加モジュール: `zmk-feature-fast-keymap`、`zmk-feature-watchdog`（フリーズ時の自動復帰）、`zmk-feature-module-physical-layout`
- `CONFIG_CONSOLE=n`、`BOARD_SERIAL_BACKEND_CDC_ACM=n`

## 9. レイヤー6（studio_unlock）に入れない問題への対処

レイヤー6へは、レイヤー2（またはCENTER2）の `&lt 6 SPACE` をホールドして入る。標準の `&lt` は tap-preferred のため、ホールド確定前（200ms以内）に別キーを押すとレイヤー6にならず、レイヤー2のキー（Zの位置=Ctrl+Z）が出てしまう。

対処: `lt_hp`（hold-preferred、200ms）を追加し、レイヤー2・8の `&lt 6 SPACE` だけをこれに置換。他キーを押した時点で即ホールド確定になる。

使い方: Space位置（`&lt 2 SPACE`）を押したまま、レイヤー2の `&lt_hp 6 SPACE`（デフォルトレイヤーのRの位置）を押したままにし、Z位置の `&studio_unlock` を押す。

### 追記: レイヤー6を経由しない `&studio_unlock`

1秒以上ホールドしてもレイヤー6に入れず Ctrl+Z になる報告があったため、レイヤー2（MOUSE_CTRL_L）とレイヤー8（CENTER2_L）の最下段左から3番目（デフォルトレイヤーのLeft Alt位置、元は `&trans`）にも `&studio_unlock` を追加した。Spaceホールド（レイヤー2）の状態でそのキーを押せばアンロックできる。
