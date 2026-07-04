# KeymapEditor互換性のためのレイヤー番号リテラル化

> 作成: 2026-07-05
> 参考: [zmk-keyboard-torabo-tsuki-lp/memos/keymap-editor-compatibility.md](https://github.com/tkgitpocket/zmk-keyboard-torabo-tsuki-lp/blob/master/memos/keymap-editor-compatibility.md)

---

## 背景

[KeymapEditor](https://nickcoutsos.github.io/keymap-editor/)（`nickcoutsos/keymap-editor`）はCプリプロセッサや `dtc` を通さず、`&lt`/`&mo`/`&to` の引数を正規表現でそのまま静的辞書と突き合わせて表示する。そのため `#define DEFAULT_L 0` のような独自マクロは「未知のトークン」として `⊘` 表示になり、KeymapEditor上でレイヤー参照が正しく見えなくなる。

ファームウェアのビルド自体（`west build` 時にCプリプロセッサが実際に走る）には影響しない、あくまで表示上の問題。

## 変更内容

`config/roBa.keymap` 内の `&lt`/`&mo`/`&to` の第1引数（レイヤー番号）を、`DEFAULT_L`・`SYMBOL_L` などのマクロから数値リテラル直書きに変更した。あわせて `#define DEFAULT_L 0` 等のレイヤー番号マクロは削除し、ファイル冒頭にレイヤー番号と名前の対応をコメントとして残した。

| 番号 | 旧マクロ名 | 用途 |
| --- | --- | --- |
| 0 | `DEFAULT_L` | ベースQWERTYレイヤー |
| 1 | `SYMBOL_L` | 記号・カッコ |
| 2 | `MOUSE_CTRL_L` | マウスボタン・矢印・クリップボード |
| 3 | `NUM_FUNCTION_L` | 数字・ファンクションキー |
| 4 | `MOUSE_L` | マウスボタンのみ |
| 5 | `SCROLL_L` | トラックボールスクロールモード |
| 6 | `BT_L` | Bluetoothプロファイル選択 |

なお `boards/shields/roBa/roBa_R.overlay` 側（`scroll-layers = <5>` 等）はもともとレイヤー番号を数値リテラルで直書きしており、`roBa.keymap` のマクロを参照する構造ではなかったため、今回のマクロ削除による影響はない。レイヤー構成・番号自体の変更は行っていない。

## 今後レイヤーを追加・並び替えする場合の注意

マクロが無いため、レイヤーを追加/削除/並び替えする際は `roBa.keymap` 内の `&lt N`/`&mo N`/`&to N` の数値と `roBa_R.overlay` の `scroll-layers`/`automouse-layer`/`zip_temp_layer` の引数の両方を手動で整合させる必要がある。

## 関連

- [zmk-layout-shift-v2-migration.md](zmk-layout-shift-v2-migration.md)
