# BTプロファイル選択時の自動切断マクロ

> 作成: 2026-07-05
> 参考: [zmk-keyboard-torabo-tsuki-lp/memos/layer-restructure-apple-windows.md](https://github.com/tkgitpocket/zmk-keyboard-torabo-tsuki-lp/blob/master/memos/layer-restructure-apple-windows.md)（BTプロファイル別デフォルトレイヤー切り替えの節）

---

## 背景

torabo-tsuki-lp側では、BTプロファイル選択マクロ `out_bt_0`〜`out_bt_4` に「選択したプロファイル以外を明示的に切断する」処理を追加し、接続の安定性を上げていた（ZMKは切断直後に選択中プロファイルへ自動的に再接続するため、実質的に「そのプロファイルへ強制的に繋ぎ直す」動作になる）。

torabo側はこのマクロで併せてMac/Windows別のデフォルトレイヤーへ `&to` で切り替えている。導入当初のroBaにはOS別のデフォルトレイヤーが存在しなかったため、その部分は見送り「自動切断による接続安定化」のみを取り込んでいたが、その後 [apple-layers-and-speed-layers.md](apple-layers-and-speed-layers.md) でApple用デフォルトレイヤー（レイヤー7）を追加したのに合わせて、`&to` によるデフォルトレイヤー自動切替も追加した。

## 変更内容

`config/roBa.keymap` のBTレイヤー（レイヤー6）で、素の `&bt BT_SEL n` を使っていた箇所を `out_bt_0`〜`out_bt_4` マクロに置き換えた。

```
out_bt_0: out_bt_0 {
    compatible = "zmk,behavior-macro";
    #binding-cells = <0>;
    wait-ms = <50>;
    bindings = <&out OUT_BLE &bt BT_SEL 0 &bt BT_DISC 1 &bt BT_DISC 2 &bt BT_DISC 3 &bt BT_DISC 4 &to 0>;
};
```

（`out_bt_1`〜`out_bt_4` も同様に、選択したプロファイル以外の4つを `&bt BT_DISC` で切断したうえで、プロファイル0-1は `&to 0`（Windows想定、DEFAULT_L）、プロファイル2-4は `&to 7`（Apple想定、APPLE_L）でデフォルトレイヤーを切り替える。）

## 未検証・要フォローアップ

- 実機での動作確認は未実施。プロファイル切替時に一瞬切断が入るため、体感の切替速度が変わる可能性がある。
- プロファイル2〜4を必ずApple機器に割り当てる運用を前提にしている。Windows機器をプロファイル2〜4に割り当てる場合は `&to` の割り当てを変更すること。

## 関連

- [zmk-layout-shift-v2-migration.md](zmk-layout-shift-v2-migration.md)
