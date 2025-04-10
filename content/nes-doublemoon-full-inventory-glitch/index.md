+++
title = "ダブルムーン伝説 (FC) インベントリ満杯状態によるメモリ破壊バグ"
date = 2025-04-09

[taxonomies]
tags = ["NES"]
+++

FC版『ダブルムーン伝説』は、マップ上の宝箱や各種イベントによりアイテムを入手する際、PTのインベントリに空きがなければその中の非重要アイテムを 1 つ削除してからアイテムを入手する、という仕様になっている (該当コード: PRG 18 `$B58D`)。しかし、これの実装ロジックには問題があり、インベントリではないメモリ領域を破壊することができる。

上記の「非重要アイテム」かどうかの判定は、重要アイテムIDテーブル (PRG 18 `$B5F7`) を参照して行われる。しかし、このテーブルにはなぜか店売りの「つえ」(アイテムID 0x55) が含まれているため、予めPT全員のインベントリを杖で埋めておくことで、削除すべき非重要アイテムが見つからない状態を作り出せる。

このとき、非重要アイテムの探索はPT内で末尾から先頭 (1 人目) の順に行われるが、PT全員について調べても非重要アイテムが見つからないため、さらにPT内の 0 人目のメンバーのインベントリも調べる、という挙動になる。そして、0 人目のPTメンバーに対応する冒険者構造体ポインタの値は 0 (該当コード: PRG 31 `$FF12`) なので、ゼロページの先頭を冒険者構造体とみなし、その中のインベントリ領域を操作することになる。これにより、`$28-$36` あたりの領域に対して書き込みが行われる。

ただし、このメモリ領域はおそらく雑用変数の類なので、スピードラン的な意味でこのバグが有用かどうかはよくわかっていない。とりあえずニーグルで癒しの杖を受け取るイベントなどを利用して試したが、メモリ破壊が起こっても表面上の挙動に変化は見られなかった。

一応、非重要アイテムを消した際にそれが装備中だったならば装備の再評価が行われるので、雑用変数の値がいい感じになっていればこれによってさらなるメモリ破壊に繋がるケースはあるかもしれない (チートで軽く試したところ、装備再評価によってフリーズする例は確認した)。
