+++
title = "ca65 のソースコードを直接実行する"
date = 2023-07-23

[taxonomies]
tags = ["NES", "ca65"]
+++

ca65 はコメント開始文字が ';' なので、シェルから直接実行可能なソースコードが書ける。

たとえば、以下のソースコードを `foo.s` として保存し、`sh foo.s` とすると直接 FCEUX 上で実行できる
(事前に `cc65`, `fceux` パッケージをインストールしておく必要がある)。

```ca65
_DUMMY=0; ca65 -o foo.o "$0" && ld65 --target nes -o foo.nes foo.o && fceux foo.nes; exit

; 矩形波を再生するだけ。

.segment "HEADER"

        .byte   "NES", $1A      ; iNES magic
        .byte   2               ; 16K PRG count
        .byte   1               ; 8K CHR Count
        .byte   0               ; Mapper 0, Horizontal mirroring

.segment "CODE"

Reset:
        lda     #$0F
        sta     $4015

        lda     #(1 << 5) | (1 << 4) | 15
        sta     $4000
        lda     #<253
        sta     $4002
        lda     #>253
        sta     $4003

@loop_forever:
        jmp     @loop_forever

Nmi:
Irq:
        rti

.segment "VECTORS"

        .addr   Nmi
        .addr   Reset
        .addr   Irq
```

サウンドのみを扱うコードを手軽にテストしたい場合に役立つかも。

なお、セグメントの割り当てについては cc65 の [nes.cfg](https://github.com/cc65/cc65/blob/ce3bcadd24d3af05dc30f542722f6efb66382e82/cfg/nes.cfg) を参照。

この記事は [ソースコード直接実行のテクニック - 何とは言わない天然水飲みたさ](https://blog.cardina1.red/2017/04/02/direct-source-execution/) からヒントを得て作成した。
