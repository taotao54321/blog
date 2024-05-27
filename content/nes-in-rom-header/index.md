+++
title = "一部の商用 FC/NES ゲームに存在する ROM 内ヘッダについて"
date = 2024-05-27

[taxonomies]
tags = ["NES"]
+++

一部の商用 FC/NES ゲームは、PRG-ROM 末尾の割り込みベクタ手前に 26 バイトの ROM 内ヘッダらしきものを持っている (通常は論理アドレス `$FFE0-$FFF9` にマップされる。以下、このことを仮定する)。

この ROM 内ヘッダの内容については NesDev Forum 内の "[Headers at the end of commercial roms](https://forums.nesdev.org/viewtopic.php?t=6078)" スレッドである程度解明されている。以下に概要を示す:

| addr | type | description |
| --   | --   | --          |
| `$FFE0` | `u8[16]` | ゲーム名。16 バイトに満たない場合、先頭または末尾が 0x00 または 0x20 でパディングされる。 |
| `$FFF0` | `u16` | PRG チェックサム (チェックサム領域を除いた全バイトの総和)。<br>PRG 全体を対象とするものと、PRG 最終バンクのみを対象とするものがある。<br>大抵はビッグエンディアンだが、リトルエンディアンの場合もある。 |
| `$FFF2` | `u16` | CHR チェックサム (全バイトの総和) だが、大抵は 0 になっている。 |
| `$FFF4` | `u8` | ROM サイズ情報。<br>bit7-4: PRG サイズ (0:64KiB, 2:32KiB, 3:128KiB, 4:256KiB or 512KiB, 5:512KiB)<br>bit3-0: CHR サイズ (0:8KiB RAM, 1:16KiB, 2:32KiB, 3:128KiB, 4:256KiB, 8:8KiB RAM) |
| `$FFF5` | `u8` | VRAM ミラーリング (2:V, 4:マッパー制御, 0x81:H, 0x82:H) |
| `$FFF6` | | (不明。バージョン番号?) |
| `$FFF7` | | (不明。0x10 以下の値になる?) |
| `$FFF8` | `u8` | licensee code (ゲームボーイ ROM ヘッダ内の [Old licensee code](https://gbdev.io/pandocs/The_Cartridge_Header.html#014b--old-licensee-code) と同様と思われる) |
| `$FFF9` | | (不明。同一シリーズ作品では同じ値になる?) |

ROM 内ヘッダは通常は最終バンクの割り込みベクタ手前に 1 つだけ存在するが、全バンクに ROM 内ヘッダが置かれているゲームもある (『ドラゴンクエスト4』など)。

ヘッダ内の情報は必ずしも正しいとは限らない。たとえば、『アークティック』の ROM 内ヘッダでは VRAM ミラーリングが 4 (マッパー制御) となっているが、実際には H ミラー固定である。

この ROM 内ヘッダは 1987 年頃から出現しているとのこと。任天堂への提出用なのかもしれないが、詳細は不明。

thefox 氏が日本/USA/ヨーロッパの ROM セットに対する ROM 内ヘッダの調査結果を公開している。現在はリンク切れだが、Internet Archive から入手できる:

* [hdrinfos-japan.txt](https://web.archive.org/web/20140922133703/http://thefox.aspekt.fi/hdrinfos-japan.txt)
* [hdrinfos-usa.txt](https://web.archive.org/web/20140922133703/https://thefox.aspekt.fi/hdrinfos-usa.txt)
* [hdrinfos-europe.txt](https://web.archive.org/web/20140922133703/https://thefox.aspekt.fi/hdrinfos-europe.txt)
