# moNa2（右手トラックボール）DYA Studio 対応ファームウェア

[DYA Studio](https://studio.dya.cormoran.works/) の全タブ（キーマップ / マクロ&コンボ / トラックボール / 接続 / 設定 / 診断）に対応させた moNa2 用 ZMK config です。

- ZMK: `cormoran/zmk` の `main+dya`（Zephyr 4.1）
- ベース: 実機で使っていた `soraclock/DYA` の c5a8298（2026-04-18）のビルド
- キーマップ・エンコーダー・トラックボールの値は、2026-10-12 に DYA Studio で実機から読み出した値と同じにしてある（Studio 上で変更・保存していた分も含む）

## 旧ファームからの変更点

| 項目 | 旧（c5a8298） | 新 |
| --- | --- | --- |
| ZMK | `cormoran/zmk v0.3-branch+dya`（Zephyr 3.5） | `cormoran/zmk main+dya`（Zephyr 4.1） |
| ボード名 | `seeeduino_xiao_ble` | `xiao_ble//zmk` |
| NFC ピンの GPIO 化 | `CONFIG_NFCT_PINS_AS_GPIOS` | `&uicr { nfct-pins-as-gpios; }`（Zephyr 4.1 で Kconfig が廃止） |
| PMW3610 ドライバー | `badjeff/zmk-pmw3610-driver` | `cormoran/zmk-driver-pmw3610-with-custom-studio-rpc`（既定値は同じ。CPI 600） |
| コンボ | `zmk,combos` | 同じ `zmk,combos`（Studio のコンボ一覧には出ない。Studio で追加したコンボは runtime combo として別枠で動く） |
| Battery History | 有効 | 無効（DYA Studio に画面がなく、DYA2 でも保存が遅い問題のため無効化されている） |
| 追加モジュール | — | custom-settings, fast-keymap, runtime-macro, runtime-combo, input-stream, physical-layout, default-layer, os-detection, device-info, watchdog, kscan-diagnostics |

### Studio で変更されていた値を keymap / overlay に固定したもの

- Layer 0 / 3 のエンコーダー: スクロール量 ±100（他のレイヤーは ±30）
- Layer 8（mouse_mac）の位置 1: `&kp TAB`
- Layer 9（mouse_win）の位置 1: `&kp LG(TAB)`、最下段の `LC(Z)` / `LC(C)` / `LC(V)` を 1 つ右へ
- スクロール用プロセッサー（Layer 10）: 倍率 1/21、X/Y 反転

### Win / Mac の自動切替

旧ファームは `BT_WIN` / `BT_MAC` マクロ（プロファイル選択＋ `&to 0` / `&tog 3`）で手動切替していたが、接続先の OS を検出して自動で切り替える方式に変えた。

- マクロはプロファイルの選択だけを行う（Studio 上の表示名は `BT_0` / `BT_1`。Studio に保存済みのキーマップと互換を保つため、ノード名は `bt_win` / `bt_mac` のまま）
- `CONFIG_ZMK_DEFAULT_LAYER_MIN_INDEX` / `MAX_INDEX` を 3 にし、Mac 以外へ切り替えたときに Layer 3（MAC_BASE）が外れるようにした
- 割り当ては Studio の「接続」タブで設定し、キーボード側に保存される（`settings_reset` を書くと消えるので、その場合は設定し直す）
  - 接続先（USB、プロファイル 0〜3）のデフォルトレイヤー: すべて「OS 検出に従う」
  - OS ごとのデフォルトレイヤー: macOS → Layer 3、それ以外は未設定（Layer 0）
- BLE の OS 検出は推定なので、誤判定したプロファイルは「接続」タブの OS 欄で手動指定する

### 意図して入れていない・使わないもの

- **コンボの runtime combo 化**: `cormoran,runtime-combo-defaults` に移すと Studio で編集できるが、別コンボの途中で押したキーをコンボの先頭として再判定しない（ZMK 標準は再判定する）ため、旧ファームと挙動が変わる。そのため標準の `zmk,combos` のままにしている。
- **zmk-module-devtool**（診断タブの「スタック使用量」）: Studio のロック解除・再起動を認証なしの RPC で実行できるため、作者が製品ファームでは無効にするよう明記している。このため「スタック使用量」だけは未対応表示のまま。

## ビルド

GitHub Actions（`.github/workflows/build.yml`）で `build.yaml` の 3 つを生成します。

| ファイル | 書き込み先 |
| --- | --- |
| `mona2_r` | 右手（central・PC と接続する側） |
| `mona2_l` | 左手（peripheral） |
| `settings_reset` | 左右どちらにも使う設定消去用 |

ローカルでは Docker（`zmkfirmware/zmk-build-arm:stable`）で次のようにビルドできます。

```bash
west init -l config && west update --narrow && west zephyr-export
west build -s zmk/app -b xiao_ble//zmk -S studio-rpc-usb-uart -- \
  -DSHIELD="mona2_r rgbled_adapter" -DZMK_CONFIG=$PWD/config -DZMK_EXTRA_MODULES=$PWD
```

## 書き込み手順

ZMK のメジャー更新（Zephyr 3.5 → 4.1）と、設定の保存形式が変わるモジュール更新を含むため、**左右とも設定を消去してから**書き込んでください。キーマップ等はファームに焼き込み済みなので、消去しても内容は変わりません。

1. 右手の XIAO をリセット 2 回押しでブートローダーにし、`settings_reset.uf2` をコピーする
2. 左手も同じく `settings_reset.uf2` をコピーする
3. 右手に `mona2_r.uf2`、左手に `mona2_l.uf2` をコピーする
4. 左右を同時にリセットして左右をペアリングし直す
5. PC / Mac の Bluetooth 設定から旧 `mona2` を削除し、ペアリングし直す（どのプロファイルでもよい）
6. DYA Studio で接続し、各タブが「未対応」と出ないことを確認する
7. 「接続」タブで Win / Mac の自動切替を設定する（上の「Win / Mac の自動切替」参照）
