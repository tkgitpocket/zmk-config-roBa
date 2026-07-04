# CLAUDE.md

このファイルは、Claude Code (claude.ai/code) がこのリポジトリで作業する際のガイドです。

## 概要

これは自作分割キーボード **roBa** 用のZMKファームウェア設定である。右手側にPMW3610トラックボールセンサーを搭載し、左右とも Seeeduino XIAO BLE を使用する。ローカルのビルド環境は無く、ファームウェアは全て GitHub Actions 経由でビルドされる。

roBa標準（[kumamuk-git/zmk-config-roBa](https://github.com/kumamuk-git/zmk-config-roBa)）からのカスタマイズの詳細・経緯は [memos/](memos/) フォルダを参照。

## ファームウェアのビルド

ファームウェアはGitHub Actions上のZMKホスト型ワークフローを使い、push/PR時に自動でビルドされる。

```yaml
# .github/workflows/build.yml
uses: zmkfirmware/zmk/.github/workflows/build-user-config.yml@v0.3-branch
```

[build.yaml](build.yaml) のビルドマトリクスは3つの成果物を生成する:

- `seeeduino_xiao_ble` + `roBa_R`（`studio-rpc-usb-uart` スニペット付き）— 右手側（central）
- `seeeduino_xiao_ble` + `roBa_L` — 左手側（peripheral）
- `seeeduino_xiao_ble` + `settings_reset` — BLEボンド情報リセット用

キーマップ可視化SVGを再生成するには、GitHub Actionsで **Draw Keymap** ワークフローを手動実行する。`config/*.keymap` を読み込み、`keymap-drawer/` に出力する。

## アーキテクチャ

### 主要ファイル

| ファイル | 役割 |
|------|---------|
| [config/roBa.keymap](config/roBa.keymap) | キーマップ全体: レイヤー・コンボ・ビヘイビア・マクロ |
| [config/west.yml](config/west.yml) | ZMK依存関係マニフェスト（zmk v0.3-branch、pmw3610ドライバ、zmk-layout-shift v2） |
| [boards/shields/roBa/roBa.dtsi](boards/shields/roBa/roBa.dtsi) | 共通ハードウェア定義: kscanマトリクス（4行×11列）、エンコーダー、トラックボールリスナー |
| [boards/shields/roBa/roBa_L.overlay](boards/shields/roBa/roBa_L.overlay) | 左手側: col-gpios 6本、エンコーダー有効 |
| [boards/shields/roBa/roBa_R.overlay](boards/shields/roBa/roBa_R.overlay) | 右手側: col-gpios 5本、col-offset=6、SPI経由のPMW3610トラックボール |

### スプリット構成

- **右手側 = central**（`ZMK_SPLIT_ROLE_CENTRAL`）: USB接続、トラックボール、ZMK Studio RPC
- **左手側 = peripheral**: エンコーダーのみ
- マトリクスは全体で4行×11列。右手側オーバーレイは `col-offset = <6>` で列6〜10を担当

### レイヤー

[KeymapEditor](https://nickcoutsos.github.io/keymap-editor/) 互換のため、`config/roBa.keymap` 内の `&lt`/`&mo`/`&to` のレイヤー引数はマクロではなく数値リテラルを直書きしている（経緯は [memos/keymap-editor-compatibility.md](memos/keymap-editor-compatibility.md) 参照）。レイヤー番号と用途の対応は以下の通り（ファイル冒頭のコメントにも同じ表がある）。

| 番号 | 名称 | 用途 |
|--------|--------|---------|
| 0 | DEFAULT_L | ベースQWERTYレイヤー（Windows想定） |
| 1 | SYMBOL_L | 記号・カッコ |
| 2 | MOUSE_CTRL_L | マウスボタン・矢印・クリップボード（Windows想定、Ctrl系ショートカット） |
| 3 | NUM_FUNCTION_L | 数字（最上段）・ファンクションキー |
| 4 | MOUSE_L | マウスボタンのみ |
| 5 | SCROLL_L | トラックボールスクロールモード・標準速度（PMW3610の `scroll-layers` で発火） |
| 6 | BT_L | Bluetoothプロファイル選択・クリア |
| 7 | APPLE_L | Apple（Mac/iOS）用デフォルトレイヤー |
| 8 | CENTER2_L | マウスボタン・矢印・クリップボード（Apple想定、Cmd系ショートカット） |
| 9 | SLOWTRACKBALL_L | トラックボールカーソル低速モード（移行キー未割り当て） |
| 10 | SLOWSCROLL_L | トラックボールスクロール低速モード（移行キー未割り当て） |
| 11 | FASTRSCROLL_L | トラックボールスクロール高速モード（移行キー未割り当て） |

BTプロファイル選択マクロ（`out_bt_0`〜`out_bt_4`）は、プロファイル0-1（Windows想定）で `&to 0`、プロファイル2-4（Apple想定）で `&to 7` を実行し、デフォルトレイヤーを自動的に切り替える。詳細は [memos/apple-layers-and-speed-layers.md](memos/apple-layers-and-speed-layers.md) 参照。

レイヤーを追加・並び替える場合は、`roBa.keymap` 側の数値と `roBa_R.overlay` の `scroll-layers`/`zip_temp_layer`/`trackball_listener` 内の `layers = <N>` を手動で整合させること。

### 日本語配列対応（zmk-layout-shift）

記号キーのJIS配列変換には [zmk-layout-shift](https://github.com/kot149/zmk-layout-shift) v2 を使用している（旧 `JP_*` 独自マクロ方式から移行済み。経緯は [memos/zmk-layout-shift-v2-migration.md](memos/zmk-layout-shift-v2-migration.md) 参照）。

- `&kp` は `layout_shift_map_us_to_jis` マップに従ってUS配列キーコードをJIS配列OS向けに自動変換する（例: `&kp AT` → JIS OSでは `@`）。日本語配列の記号を追加する場合は、US配列における対応キーコードをそのまま `&kp` に渡せばよい。
- 変換の有効/無効はBTレイヤー（レイヤー6、`&out_bt_4` の直下のキー）の `&tog_ls_on` / `&tog_ls_off` で切り替える。初回フラッシュ後は一度 `&tog_ls_on` を押すこと（設定は不揮発領域に保存され、以後の再起動・再フラッシュでも保持される）。
- KeymapEditor対策として、`&kp` の実体（`zmk,behavior-layout-shift-key-press` への上書き）は `behaviors {}` 内に直接定義してある。includeの並び順の変化に依存しないための措置。

### カスタムビヘイビア

- `lt_to_layer_0` — ホールドでレイヤー起動、タップでレイヤー0に戻るhold-tap
- `long_hold_mod_tap`（350ms、tap-preferred）— shift/Z、shift/slash用
- `medium_hold_mod_tap`（250ms、balanced）— 汎用mod-tap
- `scroll_up_down` / `mouse_scroll` — エンコーダーのスクロールホイール割り当て
- `mo2`–`moI` — シフト時に記号が変化するmod-morphビヘイビア群
- `out_bt_0`–`out_bt_4` — BTプロファイル選択時に他プロファイルを切断・接続安定化し、あわせてOSに応じたデフォルトレイヤーへ切り替えるマクロ（[memos/bt-profile-auto-disconnect.md](memos/bt-profile-auto-disconnect.md) 参照）
