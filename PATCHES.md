# rtl-sdr-rs への変更点

元は crates.io の `rtl-sdr-rs` 0.4.2（https://github.com/ccostes/rtl-sdr-rs 、MPL-2.0）。
`examples/` と `.github/` は載せていない。
`Cargo.toml` には、周りのワークスペースから切り離す `[workspace]` と、元からある未使用フィールドの警告を止める `[lints.rust]` を足している。
ソースで元と違うのは次の 1 件。

## Windows で RTL-SDR Blog V4 が判定されない

- 対象：`src/device/device_handle.rs`
- 症状：Windows で V4 が通常の R828D（基準クロック 16 MHz）として初期化され、PLL がロックせず受信できない。
- 原因：V4 の判定は USB の製造者名 `RTLSDRBlog` と製品名 `Blog V4` の一致で行う。
  シリアル／インデックス指定で開く経路は `nusb::DeviceInfo` の文字列を使うが、nusb は Windows では製造者名を常に `None` で返す。
- 変更：デバイスを開いたあと、列挙情報で取れなかった文字列だけをディスクリプタから読んで補う（`fill_missing_strings`）。
  3 つとも取れている環境（macOS / Linux）では何も読まず、挙動は変わらない。

上流が同じ問題を直したら、この vendor とルートの `[patch.crates-io]` を外す。
