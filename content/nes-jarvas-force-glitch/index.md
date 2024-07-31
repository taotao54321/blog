+++
title = "未来神話ジャーヴァス (FC) バトル敗北時に兵力が激増するバグの詳細"
date = 2024-08-01

[taxonomies]
tags = ["NES"]
+++

FC版未来神話ジャーヴァスは、ギルド内のバトルで負けると名声値が 1 下がり(ただし負にはならない)、さらに自軍の兵力が以下のように変化する:

* 敗北後の名声値が 0 の場合、全ユニットの兵力が 0 になる。
* 敗北後の名声値が 0 でない場合、志願兵、傭兵、グルカ兵の兵力が変動する。

後者のケースは本来「志願兵、傭兵、グルカ兵の兵力がそれぞれ 3/4 になる」仕様だったと思われるが、バグにより実際にはこれらの兵力が激増することがある。
なお、この現象は 2019 年には既に[発見されていた](https://x.com/oresika2010/status/1130521162797047809)模様。

このバグには以下の 2 つの誤りが関係している:

* 兵力減少ルーチン内の計算ミス
* 兵力減少ルーチンを呼び出す際の引数渡しのミス

まず兵力減少ルーチン (`F1DD`) について述べる。このルーチンはバトルで負けた際に志願兵/傭兵/グルカ兵それぞれについて呼び出されている。コードを以下に示す:

```ca65
;;; バトルに負けた際の兵力「減少」の計算。
; 引数
;       A       元の兵力上位
;       X       元の兵力下位
;
; 戻り値
;       A       結果の兵力上位
;       X       結果の兵力下位
L_F1DD:
        ; quarter = (元の値) / 4
        ; half    = (元の値) / 2
@quarter := $F0
@half := $F2
        stx     @quarter
        sta     @quarter+1
        stx     @half
        sta     @half+1
        .repeat 2
                lsr     @quarter+1
                ror     @quarter
        .endrepeat
        lsr     @half+1
        ror     @half

        ; XXX: ここで本来は quarter + half を返したかったのだろうが、実際には全く違う計算になっている。
        ; 戻り値下位は add((元の兵力上位), quarter & 0xFF) となる。
        ; 戻り値上位は adc(X, quarter >> 8) となる。
        clc
        adc     @quarter
        sta     @half
        tax
        adc     @quarter+1
        sta     @half+1

        rts
```

`XXX` コメントにある通り、このコードは元の兵力の 3/4 を返さない。たとえば元の兵力が 255 なら結果の兵力は 16191 となる。

`XXX` の箇所の計算式は、元の兵力を {{ katex(body="f") }} とおくと以下のように整理できる:

{% katex(block=true) %}257 \cdot (\lfloor \frac{f}{256} \rfloor + \lfloor \frac{f}{4} \rfloor) \mod 65536{% end %}

たとえば Rust では以下のように書ける:

```rust
fn decrease_force(f: u16) -> u16 {
    (f / 256 + f / 4).wrapping_mul(257)
}
```

兵力減少ルーチンについては以上だが、話はこれで終わりではなく、実際に志願兵 0 人、傭兵 255 人の状態でこのバグを試すと傭兵は 0 人になる。これは上記ルーチンの呼び出し側の引数渡しにもミスがあることが原因。呼び出し側のコード (`$F175`) を以下に示す:

```ca65
force_irregular := $0411 ; 志願兵の兵力 (u16le)
force_mercenary := $0413 ; 傭兵の兵力 (u16le)
force_gurkha    := $0415 ; グルカ兵の兵力 (u16le)

L_F175:
        ; 志願兵の兵力減少処理。
        lda     force_irregular+1
        ldx     force_irregular
        jsr     L_F1DD
        sta     force_irregular+1
        stx     force_irregular

        ; 傭兵の兵力減少処理。
        lda     force_mercenary+1
        stx     force_mercenary ; XXX: stx は ldx の typo!
        jsr     L_F1DD
        sta     force_mercenary+1
        stx     force_mercenary

        ; グルカ兵の兵力減少処理。
        lda     force_gurkha+1
        ldx     force_gurkha
        jsr     L_F1DD
        sta     force_gurkha+1
        stx     force_gurkha
```

`XXX` コメントにある通り、傭兵については `ldx` とすべきところを `stx` と typo している。
つまり、傭兵の「元の兵力」は上位バイトのみが正しく、下位バイトは志願兵の変化後の兵力の下位バイトがそのまま引き継がれる。これによる兵力の変化例をいくつか示す:

* 志願兵 0 人、傭兵 255 人なら志願兵は 0 人のままで、傭兵の「元の兵力」は 0 となり、傭兵は 0 人に変化する。
* 志願兵 0 人、傭兵 256 人以上 511 人以下なら志願兵は 0 人のままで、傭兵の「元の兵力」は 256 となり、傭兵は 16705 人に変化する。
* 志願兵 255 人、傭兵 0 人なら志願兵は 16191 人に変化し、これにより傭兵の「元の兵力」は 63 となり、傭兵は 3855 人に変化する。
