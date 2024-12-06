+++
title = "6502 の除算ルーチンに時々見られる誤りについて"
date = 2024-12-06

[taxonomies]
tags = ["NES", "6502"]
+++

6502 で、`u16` 値を `u8` 値で割った商 (`u16`) と余り (`u8`) を求めるルーチンを考える。

以下、全ての除算ルーチンは以下のインターフェースを持つものとする:

```ca65
; 引数
;       lhs_quot        左辺 (u16le)
;       rhs             右辺 (u8)
;
; 戻り値
;       lhs_quot        商   (u16le)
;       rem             余り (u8)
;
; 左辺と商については同一のメモリ領域を使い回している。

lhs_quot := $00
rhs      := $02
rem      := $03
```

正しい実装の一例を示すと以下のようになる (大まかには 2 進数で筆算と同様の操作を行っている)。
これは `0x8000 / 0x81` に対して正しい答え (商 0xFE, 余り 2) を返す:

```ca65
;;; 正しい u16 / u8 除算。
; 開始時は lhs_quot に左辺が入っており、lhs_quot の下位から商がシフトインしていく。
;
; NOTE: rhs == 0 の場合、quot = 0xFFFF, rem = lhs & 0xFF を返す。
DivCorrect:
        lda     #0
        sta     rem

        ldx     #16
        asl     lhs_quot
        rol     lhs_quot+1
@loop:
        ; rem がオーバーフローしたら rhs を減算。
        rol     rem
        bcs     @subtract

        ; rem >= rhs ならば rhs を減算。
        lda     rem
        cmp     rhs
        bcc     @next
@subtract:
        ; ここでは常にキャリーフラグが真。
        lda     rem
        sbc     rhs
        sta     rem
        sec             ; rem がオーバーフローした場合、これが必要。

@next:
        rol     lhs_quot
        rol     lhs_quot+1

        dex
        bne     @loop

        rts
```

しかし、実際に使われている除算ルーチンの中には、右辺の値が大きいときに正しく動かないものがある。

たとえば、以下のコード (パターン A と呼称する) は計算途中での rem のオーバーフローを考慮していない。
これは `0x8000 / 0x81` に対して誤った答え (商 0, 余り 0) を返す:

```ca65
;;; 正しくない u16 / u8 除算 (余りのオーバーフローを考慮していない)。
DivWrongA:
        lda     #0
        sta     rem

        ldx     #16
        asl     lhs_quot
        rol     lhs_quot+1
@loop:
        ; XXX: ここでのオーバーフローを考慮していない!
        rol     rem

        lda     rem
        cmp     rhs
        bcc     @next
@subtract:
        ; ここでは常にキャリーフラグが真。
        lda     rem
        sbc     rhs
        sta     rem

@next:
        rol     lhs_quot
        rol     lhs_quot+1

        dex
        bne     @loop

        rts
```

また、以下のコード (パターン B と呼称する) は rem のオーバーフローは考慮しているが、減算後のキャリーフラグが常に真であると誤って仮定している。
これは `0x8000 / 0x81` に対して誤った答え (商 0x7E, 余り 2) を返す:

```ca65
;;; 正しくない u16 / u8 除算 (余りのオーバーフロー時、減算後にキャリーフラグをセットしていない)。
DivWrongB:
        lda     #0
        sta     rem

        ldx     #16
        asl     lhs_quot
        rol     lhs_quot+1
@loop:
        rol     rem
        bcs     @subtract

        lda     rem
        cmp     rhs
        bcc     @next
@subtract:
        ; ここでは常にキャリーフラグが真。
        lda     rem
        sbc     rhs
        sta     rem
        ; XXX: sbc で必ずキャリーフラグが真になると仮定しているが、rem がオーバーフローした場合これは成り立たない!

@next:
        rol     lhs_quot
        rol     lhs_quot+1

        dex
        bne     @loop

        rts
```

NES/SNES/PCE の場合、いくつかのゲームで `u16 / u8` などの除算ルーチンが パターン A の誤った実装になっていることを確認している。
しかし、パターン A, B とも右辺が 0x80 を超えないならば問題は起こらないはずなので、それも考慮すると TAS 的な意味でこれが役立つケースはあまり多くないと思われる。

本記事中のコードを NES 向けにまとめたものを[置いておく](https://gist.github.com/taotao54321/22fe89edf7539032e98a4a3b0037dd88)。
NES エミュレータでステップ実行すると各ルーチンで答えが違うことがわかる。
