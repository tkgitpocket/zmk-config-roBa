# トラックボール周りのroBa標準からのカスタマイズ

> 作成: 2026-07-05
> `upstream/main`（[kumamuk-git/zmk-config-roBa](https://github.com/kumamuk-git/zmk-config-roBa)）との差分をもとに作成。

---

roBa標準（upstream）から `boards/shields/roBa/` 配下で変更している主なハードウェア/入力系の設定をまとめる。キーマップ本体（レイヤー構成・combo・mod-morph等）は独自設計のため対象外。

## 1. AML（Automatic Mouse Layer）を標準の `automouse-layer` から `zip_temp_layer` ベースに変更

upstream標準は `&trackball { automouse-layer = <4>; }` のようにPMW3610ドライバ組み込みのAML機能を使うだけだったが、roBaでは `input-processors` の `zip_temp_layer`（一時レイヤー起動）+ `zip_xy_scaler`（カーソル移動量のON/OFF）を組み合わせた方式に変更している（`boards/shields/roBa/roBa_R.overlay`）。

```
&trackball_listener {
    status = "okay";
    device = <&trackball>;

    // ベース: MOUSE_L(4) を自動起動(10秒間有効) → 未起動時はカーソル移動を抑制
    input-processors = <&zip_temp_layer 4 10000>, <&zip_xy_scaler 0 1>;

    // MOUSE_L が有効なとき: 通常のカーソル移動を許可（ベースをスキップ）
    mouse_active {
        layers = <4>;
        input-processors = <&zip_xy_scaler 1 1>;
    };

    // MOUSE_CTRL_L(2) が有効なとき: Spaceホールドでの使用のため移動を許可
    mouse_ctrl_active {
        layers = <2>;
        input-processors = <&zip_xy_scaler 1 1>;
    };
};
```

**理由**: トラックボールを触っていないときにカーソルが動いてしまう誤操作を防ぐため、「MOUSE_L/MOUSE_CTRL_Lが有効な間だけ移動量を通す」方式にし、それ以外は `zip_xy_scaler 0 1` でカーソル移動をゼロに抑制している。

## 2. トラックボールの「揺れ」（チャタリング的な誤起動）対策

タイピング直後にトラックボールが物理的に揺れてAMLが誤発動するのを防ぐため、`zip_temp_layer` に以下を設定している。

```
// タイピング後 300ms 以内は AML を起動しない
// クリックボタンを押しても AML を維持する位置: MB4(6) MB3(7) MB5(8) LCLK(18,19,31) RCLK(20,32)
&zip_temp_layer {
    require-prior-idle-ms = <300>;
    excluded-positions = <6 7 8 18 19 20 31 32>;
};
```

- `require-prior-idle-ms = <300>`: 直前のキー入力から300ms以内はAMLの自動起動をトリガーしない。
- `excluded-positions`: マウスクリック系のキー（MB4/MB3/MB5・左右クリック）を押してもAML起動のトリガーやタイマーリセットの対象に含めない（＝クリック操作自体でAMLが持続する位置）。

## 3. PMW3610のCPI分周・向き設定

`boards/shields/roBa/roBa_R.conf` で `CONFIG_PMW3610_ORIENTATION_180` を無効化し、`CONFIG_PMW3610_ORIENTATION_0` を有効化（コメントに `# for coropit` とあり、coropit対応ビルド向けにセンサーの向きを変更したもの）。

## 4. 左エンコーダーのsteps変更

`boards/shields/roBa/roBa.dtsi` の `left_encoder` の `steps` を upstream標準の `12` から `24` に変更（1回転あたりの分解能を上げている）。

## 関連

- [zmk-layout-shift-v2-migration.md](zmk-layout-shift-v2-migration.md)
- [keymap-editor-compatibility.md](keymap-editor-compatibility.md)
- [bt-profile-auto-disconnect.md](bt-profile-auto-disconnect.md)
