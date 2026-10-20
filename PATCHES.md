# rtl-sdr-rs への変更点

元は crates.io の `rtl-sdr-rs` 0.4.2（https://github.com/ccostes/rtl-sdr-rs 、MPL-2.0）。
`examples/` と `.github/` は載せていない。
`Cargo.toml` には、周りのワークスペースから切り離す `[workspace]` と、元からある未使用フィールドの警告を止める `[lints.rust]` を足している。
ソースで元と違うのは次の 3 件。

## Windows で RTL-SDR Blog V4 が判定されない

- 対象：`src/device/device_handle.rs`
- 症状：Windows で V4 が通常の R828D（基準クロック 16 MHz）として初期化され、PLL がロックせず受信できない。
- 原因：V4 の判定は USB の製造者名 `RTLSDRBlog` と製品名 `Blog V4` の一致で行う。
  シリアル／インデックス指定で開く経路は `nusb::DeviceInfo` の文字列を使うが、nusb は Windows では製造者名を常に `None` で返す。
- 変更：デバイスを開いたあと、列挙情報で取れなかった文字列だけをディスクリプタから読んで補う（`fill_missing_strings`）。
  3 つとも取れている環境（macOS / Linux）では何も読まず、挙動は変わらない。

## PPM 補正の中間計算のオーバーフロー

- 対象：`src/rtlsdr.rs`
- 症状：PPM 補正の `-ppm × 2^24` が i32 の範囲を超えると、デバッグビルドでは panic し、リリースビルドでは誤った補正値になる（例：-128 ppm、±200 ppm）。
- 変更：乗算と除算を i64 で行い、最後にレジスタ用の値へ変換する。
  正負の値とアプリの入力上限 ±1000 ppm までの計算を、USB を使わない回帰テストで確認する。
  レジスタの有効範囲や実機の補正精度を拡張する変更ではない。

## PPM 補正時の IF/NCO の基準クロック

- 対象：`src/rtlsdr.rs`
- 症状：`set_if_freq` が固定の 28.8 MHz を使い、PPM 補正を無視していた。
  ダイレクトサンプリングでは同調周波数の補正が効かず、通常の R82xx でもチューナーと復調器の IF の間にずれが残る。
- 変更：実際の基準クロックと PPM から IF/NCO の値を計算する。
  通常の R82xx では、PPM を変えたときに IF も設定し直す。
  チューナーと復調器のクロック計算を f64 で統一し、正負の PPM を与える計算テストを追加した。

上流がこれらの問題を直したら、この vendor とルートの `[patch.crates-io]` を外す。
