+++
title = "バードウィーク (FC) のブート処理は実はバグっている"
date = 2024-07-29

[taxonomies]
tags = ["NES"]
+++

FC版バードウィークは、ハードリセット時に以下のような奇妙な処理を行う (Mesen の場合、`brk` 命令実行時にブレークするよう設定すると容易に確認できる):

* RESET ハンドラ (`$FD27`) へジャンプする。
* 雑多な初期化処理の後、RAM 初期化ルーチン (`$FE2A`) を呼ぶ。
* RAM 初期化ルーチン内でソフトリセット判定用バイト列の初期化などを行う。
* バグにより RAM 初期化ルーチンからの戻りアドレスが破壊され、`rts` したときプログラムカウンタが `$0001` へ飛ぶ。
* `$0001` 上の `brk` 命令を実行する。本作では RESET と IRQ のハンドラが共通なので、再度 RESET ハンドラへジャンプする。
* 最初から同様の処理を行うが、今度はソフトリセットと判定されるため、RAM 初期化ルーチンの戻りアドレスの破壊が起こらず正常に処理が進む。

これは、RAM 初期化ルーチン内の「ハードリセットならばスタックページをゼロクリアする」というロジックが原因。
本作ではスコア/ハイスコアがスタックページに置かれているので、これらをまとめてクリアしようとしたのだろうが、結果として戻りアドレス領域 `$01FE-$01FF` まで破壊してしまっている。結果、`rts` 実行時にプログラムカウンタは `$0001` へ飛ぶ。

しかし、RAM 初期化ルーチンは常にゼロページ全体をゼロクリアするため、`$0001` の値は 0 となっている。これは `brk` 命令であり、本作は IRQ (BRK) 発生時は RESET ハンドラへ飛ぶようになっているため、結果的には前述のように問題なく動くことになる。
