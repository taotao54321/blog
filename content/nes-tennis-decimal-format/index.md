+++
title = "テニス (FC/NES) の賢い 10 進フォーマット"
date = 2023-08-03

[taxonomies]
tags = ["NES"]
+++

NES ゲームで数値を 10 進フォーマットする場合、`10^i` での除算を繰り返す方法がよく使われている。

しかし、結果が 2 桁に収まるとわかっている場合、二分探索の考え方を用いたより高速な方法がある。
アルゴリズムは以下の通り (フォーマット対象の数値を `n`, 十の位を格納する変数を `tens` とする):

* `tens = 0` と初期化する。
* `tens` を 1 回左シフトする。`n >= 80` ならば `n -= 80, tens |= 1` とする。
* `tens` を 1 回左シフトする。`n >= 40` ならば `n -= 40, tens |= 1` とする。
* `tens` を 1 回左シフトする。`n >= 20` ならば `n -= 20, tens |= 1` とする。
* `tens` を 1 回左シフトする。`n >= 10` ならば `n -= 10, tens |= 1` とする。
* `tens` に十の位、`n` に一の位が求まっている。

閾値 40, 20, 10 は 80 を右シフトすることで生成できるので、コードサイズも短くて済む。

Rust で書くと以下の通り ([playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&gist=42a8f1c23b4bd43b641ee9e216ff29ab)):

```rust
fn main() {
    for n in 0..=99 {
        let (tens, ones) = fmt_decimal(n);
        println!("{n}\t{tens}\t{ones}");
        assert_eq!(10 * tens + ones, n);
    }
}

/// `0..=99` の整数を 2 桁 10 進フォーマットし、(十の位, 一の位) を返す。
fn fmt_decimal(mut n: u8) -> (u8, u8) {
    let mut tens = 0; // 十の位
    let mut thresh = 80;
    for _ in 0..4 {
        tens <<= 1;
        if n >= thresh {
            tens |= 1;
            n -= thresh;
        }
        thresh >>= 1;
    }

    (tens, n)
}
```

これは任天堂『テニス』でタイブレーク時のポイント表示に使われている (該当コード: `$E934`)。
