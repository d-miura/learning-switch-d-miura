#課題内容 (マルチプルテーブルを読む)
>OpenFlow1.3 版スイッチの動作を説明しよう。
>
>スイッチ動作の各ステップについて、trema dump_flows の出力 (マルチプルテーブルの内容) を混じえながら動作を説明すること。
>
>## 実行のしかた

>以下のように `--openflow13` オプションが必要です。

>``` shellsession
>$ bundle exec trema run lib/learning_switch13.rb --openflow13 -c trema.conf
>```

---

#解答
##スイッチの構成
以下の構成を考える．
```
vswitch('lsw') {
  datapath_id 0xabc
}

vhost ('host1') {
  ip '192.168.0.1'
}

vhost ('host2') {
  ip '192.168.0.2'
}

link 'lsw', 'host1'
link 'lsw', 'host2'
```

##Step.1 初期状態
tremaを起動し，switch_readyハンドラが呼ばれた直後のフローテーブルは以下のようになっている．
```
$ ./bin/trema dump_flows lsw
cookie=0x0, duration=464.323s, table=0, n_packets=0, n_bytes=0, priority=2,dl_dst=01:00:00:00:00:00/ff:00:00:00:00:00 actions=drop
cookie=0x0, duration=464.307s, table=0, n_packets=184, n_bytes=31205, priority=2,dl_dst=33:33:00:00:00:00/ff:ff:00:00:00:00 actions=drop
cookie=0x0, duration=464.307s, table=0, n_packets=44, n_bytes=15048, priority=1 actions=goto_table:1
cookie=0x0, duration=464.307s, table=1, n_packets=44, n_bytes=15048, priority=3,dl_dst=ff:ff:ff:ff:ff:ff actions=FLOOD
cookie=0x0, duration=464.307s, table=1, n_packets=0, n_bytes=0, priority=1 actions=CONTROLLER:65535
```
フローテーブルの内容は以下の通り
table_id = 0 (Filtering Table)
|優先度|処理内容|
|:----|:------|
|2|宛先MACアドレスがマルチキャストアドレスの場合，パケットをdropする|
