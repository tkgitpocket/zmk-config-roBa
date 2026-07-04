# JP_xxxマクロ → zmk-layout-shift v2 移行メモ

> 作成: 2026-07-05
> 参考: [zmk-keyboard-torabo-tsuki-lp/memos/layout-shift-v2-migration.md](https://github.com/tkgitpocket/zmk-keyboard-torabo-tsuki-lp/blob/master/memos/layout-shift-v2-migration.md)
> 状態: 移行完了・実機での動作確認は未実施（要: `&tog_ls_on` を一度押してJIS変換を有効化）

---

## 背景

roBaの `config/roBa.keymap` は、記号キーをJIS配列OS向けに出力するため `#define JP_xxx`（例: `JP_DQUOTE` = `AT`）という独自マクロで、変換後の物理キーコードを直接指定する方式になっていた。

- この方式はKeymapEditorが `JP_xxx` を認識できず記号キーの表示が崩れる問題があった。
- 同じ問題を先に踏んでいた `torabo-tsuki-lp` リポジトリで [zmk-layout-shift](https://github.com/kot149/zmk-layout-shift) v2への移行実績があり、そのまま流用できると判断した。

## 変更内容

- `config/west.yml`: `kot149/zmk-layout-shift` を `v2` で依存に追加。
- `config/roBa.keymap`:
  - `#define JP_xxx` を全廃止。
  - `#include <layout_shift.dtsi>` を追加（`behaviors.dtsi` より前に配置）。
  - `&tog_ls` / `&tog_ls_on` / `&tog_ls_off` に `layout-maps = <&layout_shift_map_us_to_jis>;` を設定。
  - KeymapEditor対策として、`&kp` を `zmk,behavior-layout-shift-key-press` で上書きする定義を `behaviors {}` 内に直接記述（`original_key_press` + `kp: key_press` オーバーライド）。includeの並び順をKeymapEditorが変更してしまうことがあるため、生成物に依存せずキーマップに直書きする方式を採用。
  - 記号キーの `&kp JP_xxx` を対応する標準キーコードに置換（下表）。旧 `JP_KANA`/`JP_EISU`/`JP_HANZEN` はどのバインディングからも未使用だったため削除。`JP_BSLH` はUS→JIS変換テーブルに存在しない値だったため `INT1` に置換（変換なしでそのまま素通し）。
  - `&tog_ls_on`（JIS変換の有効化トグル）は、BTレイヤー（レイヤー6）の2段目（ホームロー）右端キー（`&bt BT_SEL 4` の真下）に配置した。torabo-tsuki-lp側では一番右下のキーに置いていたが、roBaは同じ位置に既存の `&bt BT_CLR_ALL`（全プロファイルのボンド解除）が割り当て済みだったため、それを残しつつ空いていたキーに配置した。

## 旧マクロ → 新キーコード対応表

| 旧マクロ (旧値) | 新キーコード | 備考 |
| --- | --- | --- |
| `JP_DQUOTE` (`AT`) | `DOUBLE_QUOTES` | map: `DOUBLE_QUOTES → AT_SIGN` |
| `JP_AMPERSAND` (`CARET`) | `AMPERSAND` | map: `AMPERSAND → CARET` |
| `JP_QUOTE` (`AMPERSAND`) | `SINGLE_QUOTE` | map: `SINGLE_QUOTE → AMPERSAND` |
| `JP_EQUAL` (`UNDER`) | `EQUAL` | map: `EQUAL → UNDERSCORE` |
| `JP_CARET` (`EQUAL`) | `CARET` | map: `CARET → EQUAL` |
| `JP_YEN` (`0x89`) | `BACKSLASH` | map: `BACKSLASH → 0x89` |
| `JP_PLUS` (`COLON`) | `PLUS` | map: `PLUS → COLON` |
| `JP_TILDE` (`PLUS`) | `TILDE` | map: `TILDE → PLUS` |
| `JP_PIPE` (`LS(0x89)`) | `PIPE` | map: `PIPE → LS(0x89)` |
| `JP_AT` (`LEFT_BRACKET`) | `AT` | map: `AT_SIGN → LEFT_BRACKET` |
| `JP_COLON` (`SINGLE_QUOTE`) | `COLON` | map: `COLON → SINGLE_QUOTE` |
| `JP_ASTERISK` (`DOUBLE_QUOTES`) | `ASTERISK` | map: `ASTERISK → DOUBLE_QUOTES` |
| `JP_BACKQUOTE` (`LEFT_BRACE`) | `GRAVE` | map: `GRAVE → LEFT_BRACE` |
| `JP_UNDERSCORE` (`LS(0x87)`) | `UNDERSCORE` | map: `UNDERSCORE → LS(0x87)` |
| `JP_LBRACKET` (`RIGHT_BRACKET`) | `LEFT_BRACKET` | map: `LEFT_BRACKET → RIGHT_BRACKET` |
| `JP_RBRACKET` (`BACKSLASH`) | `RIGHT_BRACKET` | map: `RIGHT_BRACKET → BACKSLASH` |
| `JP_LPAREN` (`ASTERISK`) | `LEFT_PARENTHESIS` | map: `LEFT_PARENTHESIS → ASTERISK` |
| `JP_RPAREN` (`LEFT_PARENTHESIS`) | `RIGHT_PARENTHESIS` | map: `RIGHT_PARENTHESIS → LEFT_PARENTHESIS` |
| `JP_LBRACE` (`RIGHT_BRACE`) | `LEFT_BRACE` | map: `LEFT_BRACE → RIGHT_BRACE` |
| `JP_RBRACE` (`PIPE`) | `RIGHT_BRACE` | map: `RIGHT_BRACE → PIPE` |
| `JP_BSLH` (`INT1`) | `INT1`（変更なし） | US→JIS変換テーブルに存在しないため素通し |
| `JP_KANA` / `JP_EISU` / `JP_HANZEN` | 削除（未使用） | どのバインディングからも参照されていなかった |

## 未検証・要フォローアップ

- 実機での動作確認（記号キーが期待通りJIS配列で出力されるか）は未実施。フラッシュ直後は変換が無効なため、BTレイヤーの `&tog_ls_on` を一度押して有効化すること。
- `CONFIG_LAYOUT_SHIFT_PERSISTENT_STATE=y`（デフォルト有効）により、一度有効化すれば設定は不揮発領域に保存され、以後は再起動・再フラッシュ（全消去でない限り）でも保持される。

## 関連

- [keymap-editor-compatibility.md](keymap-editor-compatibility.md)
- [bt-profile-auto-disconnect.md](bt-profile-auto-disconnect.md)
