+++
title = "NES ゲームにおける CPU サイクル依存乱数とその調整法"
date = 2023-12-20

[taxonomies]
tags = ["NES"]
+++

NES ゲームでは、乱数が CPU の消費サイクル数に依存していることがしばしばある(以下、この種の乱数を「サイクル依存乱数」と呼ぶ)。

大抵の場合、サイクル依存乱数が生じる原因は「メインスレッドが NMI 割り込みを待つ間に乱数を更新し続けている」ことである。
このようなコードは NMI スレッドのみでほぼ全てのゲームロジックを処理するタイプのゲームでよく見られる。どうせメインスレッドは NMI を待つだけなので、それなら乱数を回しておけば負荷分散になり、かつ CPU サイクルを乱数エントロピーとして使える、という発想だろう。

注: NES プログラムでは PPU へのアクセスは VBLANK 中に行わねばならないという制約があるため、必然的に(疑似)マルチスレッドとなる([NESdev Wiki - NMI thread](https://www.nesdev.org/wiki/NMI_thread) などを参照)。

サイクル依存乱数は現代的な乱数に比べて理詰めで調整するのは難しいが、ゲームロジック内の処理が少し変わるだけで変動するので、TAS 的には乱数調整に伴うフレームロスを抑えられる可能性がある。たとえば、NES の Wizardry シリーズの乱数はサイクル依存だが、入力処理ルーチンの処理量が入力に依存しているため、入力をいろいろ試すことでフレームロスなしに乱数調整ができることがある。

また、乱数がサイクル依存かつ DPCM が使われている場合、サブフレーム入力によって乱数調整ができる可能性がある。通常、DPCM を使うゲームは [DPCM glitch](https://www.nesdev.org/wiki/Controller_reading#DPCM_conflict) 対策として 1 フレームに入力を 2 回以上読み、入力に不一致があれば適当に補正するようになっている。よって、サブフレーム入力でわざと入力を不一致にすれば処理量が変化し、結果として乱数も変化するというわけである。[FC 版クォースの TAS](https://tasvideos.org/5692M) はこの方法で乱数調整を行っている。
