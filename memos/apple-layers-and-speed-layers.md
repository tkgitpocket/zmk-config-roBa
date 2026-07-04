# Apple用レイヤー・center2・トラックボール/スクロールの速度レイヤー追加

> 作成: 2026-07-05
> 参考: [zmk-keyboard-torabo-tsuki-lp/memos/layer-restructure-apple-windows.md](https://github.com/tkgitpocket/zmk-keyboard-torabo-tsuki-lp/blob/master/memos/layer-restructure-apple-windows.md)

---

## 追加したレイヤー

| 番号 | 名称 | 内容 |
| --- | --- | --- |
| 7 | `apple_default_layer`（APPLE_L） | Mac/iOS用デフォルトレイヤー。DEFAULT_L(0)からの差分のみ上書き |
| 8 | `center2_layer`（CENTER2_L） | `mouse_arrow_ctrl`（MOUSE_CTRL_L, Windows想定）のApple向け変種。Ctrl系ショートカットをCmd系に置換 |
| 9 | `slow_trackball_layer`（SLOWTRACKBALL_L） | トラックボールカーソルの低速モード。中身は`MOUSE`レイヤーと同一、レイヤー番号のみ異なる |
| 10 | `slow_scroll_layer`（SLOWSCROLL_L） | スクロール低速モード。中身は`SCROLL`レイヤーと同一 |
| 11 | `fast_scroll_layer`（FASTRSCROLL_L） | スクロール高速モード。中身は`SCROLL`レイヤーと同一 |

## apple_default_layer の内容

`default_layer`（レイヤー0）に対する差分のみ:

- `LEFT_ALT` の位置 → `LEFT_GUI`（Cmd）に置き換え（Winキー位置はそのままなので、Mac上ではどちらもCmd相当になる）
- `&lt 1 INT_HENKAN`（変換） → `&lt 1 LANGUAGE_1`（かな相当）
- `&lt 3 INT_MUHENKAN`（無変換） → `&lt 3 LANGUAGE_2`（英数相当）
- SPACEホールド先を `&lt 2 SPACE`（MOUSE_CTRL_L）→ `&lt 8 SPACE`（CENTER2_L）に変更

## center2_layer の内容

`mouse_arrow_ctrl`（レイヤー2）に対する差分のみ、torabo-tsuki-lpのcenter_layer/center2_layer対応表に準拠:

| 用途 | mouse_arrow_ctrl（Windows想定） | center2_layer（Apple想定） |
| --- | --- | --- |
| 行頭移動 | `HOME` | `LG(LEFT)` |
| 行末移動 | `END` | `LG(RIGHT)` |
| redo | `RC(Y)` | `LG(LS(Z))` |
| 保存 | `LC(S)` | `LG(S)` |
| undo | `LC(Z)` | `LG(Z)` |
| 切り取り | `LC(X)` | `LG(X)` |
| コピー | `LC(C)` | `LG(C)` |
| 貼り付け | `LC(V)` | `LG(V)` |

矢印キー・PageUp/Down・Backspace/Delete・Enter・マウスボタン・`&lt 6 SPACE`（BT_Lへの入れ子アクセス）・`&mo 5`（SCROLL_L）はOS差が無いため元のまま維持。

## BTプロファイル選択との連動

`out_bt_0`/`out_bt_1`（プロファイル0・1、Windows想定）に `&to 0`、`out_bt_2`/`out_bt_3`/`out_bt_4`（プロファイル2〜4、Apple想定）に `&to 7` を追加し、プロファイル切替と同時にデフォルトレイヤーも自動的に切り替わるようにした。

## slow/fastのトラックボール・スクロールレイヤー

**現時点ではこれらのレイヤーへ切り替えるキーは未割り当て。** どの物理キーに割り当てるかは後日決める。レイヤー自体と速度設定（`boards/shields/roBa/roBa_R.overlay`）のみ先行して用意した。

### 速度の決め方

torabo-tsuki-lp（`zmk-driver-paw3222` + ZMK core標準の`zmk,input-processor-scaler`によるソフトウェアスケーリング方式）とroBa（`kumamuk-git/zmk-pmw3610-driver`のハードウェアCPI/スクロールモード方式）はセンサーもアーキテクチャも異なるため、絶対的なCPI値ではなく「標準速度に対する倍率」を揃えることにした。

- **トラックボールカーソル**: torabo-tsuki-lpは標準`zip_xy_scaler 1 2`に対し低速`zip_xy_scaler 1 6`（標準の1/3）。roBaの標準はMOUSE_L/MOUSE_CTRL_Lで使っている`zip_xy_scaler 1 1`なので、低速は同じ1/3倍の `zip_xy_scaler 1 3` とした。
- **スクロール**: torabo-tsuki-lpは標準`zip_scroll_scaler 1 4`に対し、低速`zip_scroll_scaler 1 8`（標準の1/2）、高速`zip_scroll_scaler 3 2`（標準の6倍）。roBaの標準スクロールはPMW3610の`CONFIG_PMW3610_SCROLL_TICK`によるハードウェアスクロールのみで、`zip_scroll_scaler`による追加スケーリングは今まで無かった(実質1倍)。同じ倍率を適用し、低速`zip_scroll_scaler 1 2`、高速`zip_scroll_scaler 6 1`とした。

高速スクロールの6倍という値はtorabo側の比率をそのまま踏襲したものであり、実機で試して速すぎる場合は`fast_scroll`ノードの`zip_scroll_scaler`の値を調整すること。

### `scroll-layers`プロパティの拡張

roBaのPMW3610ドライバは `scroll-layers = <N...>` に列挙したレイヤーが有効なときだけセンサー自体がスクロール値（ホイール）を報告するハードウェア機構になっている（`zip_scroll_scaler`はREL_WHEEL/REL_HWHEELのみをスケーリングするため、センサーがスクロールモードに入っていないと効果が無い）。そのため `scroll-layers = <5>` を `scroll-layers = <5 10 11>` に拡張し、SLOWSCROLL_L(10)・FASTRSCROLL_L(11)でもセンサーがスクロール報告モードに入るようにした。

## 未検証・要フォローアップ

- slow/fastレイヤーへの移行キー（どの物理キーに`&mo`/`&lt`を割り当てるか）は未定。
- apple_default_layer・center2_layerともに実機での動作確認は未実施。
- 高速スクロール(6倍)が実際の使用感としてどうかは要検証。

## 関連

- [bt-profile-auto-disconnect.md](bt-profile-auto-disconnect.md)
- [keymap-editor-compatibility.md](keymap-editor-compatibility.md)
