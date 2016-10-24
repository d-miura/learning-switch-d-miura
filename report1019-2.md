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
##スイッチのステップごとの動作説明
###Step.1 初期状態
tremaを起動し，switch_readyハンドラが呼ばれた直後のフローテーブルは以下のようになっている．
```
$ ./bin/trema dump_flows lsw
cookie=0x0, duration=38.139s, table=0, n_packets=0, n_bytes=0, priority=2,dl_dst=01:00:00:00:00:00/ff:00:00:00:00:00 actions=drop
cookie=0x0, duration=38.097s, table=0, n_packets=0, n_bytes=0, priority=2,dl_dst=33:33:00:00:00:00/ff:ff:00:00:00:00 actions=drop
cookie=0x0, duration=38.097s, table=0, n_packets=0, n_bytes=0, priority=1 actions=goto_table:1
cookie=0x0, duration=38.097s, table=1, n_packets=0, n_bytes=0, priority=3,dl_dst=ff:ff:ff:ff:ff:ff actions=FLOOD
cookie=0x0, duration=38.097s, table=1, n_packets=0, n_bytes=0, priority=1 actions=CONTROLLER:65535
ensy
```
フローテーブルの内容は以下の通り

table_id = 0 (フィルタリングテーブル)

|優先度|処理内容                                      |
|:--|:------------------------------------------------------------|
|2  |宛先MACアドレスがマルチキャストアドレスの場合，パケットをdropする|
|2  |宛先MACアドレスがipv6マルチキャストアドレスの場合，パケットをdropする|
|1  |GoTo Forwarding Table|

table_id = 1 (転送テーブル)

|優先度|処理内容                                      |
|:--|:---------------------------------------------------|
|3  |宛先MACアドレスがブロードキャストアドレスであればパケットをフラッディング|
|1  |コントローラにpacketIn|

###Step.2 host1からhost2へパケットを送信
host1からhost2へパケットを送信した直後のフローテーブルは以下のようになっている．
```
$ ./bin/trema send_packets --source host1 --dest host2
$ ./bin/trema dump_flows lsw
cookie=0x0, duration=78.998s, table=0, n_packets=0, n_bytes=0, priority=2,dl_dst=01:00:00:00:00:00/ff:00:00:00:00:00 actions=drop
cookie=0x0, duration=78.956s, table=0, n_packets=0, n_bytes=0, priority=2,dl_dst=33:33:00:00:00:00/ff:ff:00:00:00:00 actions=drop
cookie=0x0, duration=78.956s, table=0, n_packets=1, n_bytes=42, priority=1 actions=goto_table:1
cookie=0x0, duration=78.956s, table=1, n_packets=0, n_bytes=0, priority=3,dl_dst=ff:ff:ff:ff:ff:ff actions=FLOOD
cookie=0x0, duration=78.956s, table=1, n_packets=1, n_bytes=42, priority=1 actions=CONTROLLER:65535
```
ここでは，まずフィルタリングテーブルに記載の処理が行われる．しかし，host1からhost2へのパケットは宛先MACアドレスがマルチキャストアドレスでも，ipv6マルチキャストアドレスでもないので,転送テーブルに処理が移行される．
転送テーブルでは宛先MACアドレスがブロードキャストアドレスではないので，packetInが明示的になされる．
ここで，コントローラはFDBにhost1のMACアドレスと送信元ポート番号の対応を登録し，受信したパケットをフラッディングするのでhost2はhost１からのパケットを受信する．
この処理では，フローテーブルに変化は発生しない．

###Step.3 host2からhost1へパケットを送信
host1からhost2へパケットを送信した直後のフローテーブルは以下のようになっている．
```
$ ./bin/trema send_packets --source host2 --dest host1
$ ./bin/trema dump_flows lswcookie=0x0, duration=101.128s, table=0, n_packets=0, n_bytes=0, priority=2,dl_dst=01:00:00:00:00:00/ff:00:00:00:00:00 actions=drop
cookie=0x0, duration=101.086s, table=0, n_packets=0, n_bytes=0, priority=2,dl_dst=33:33:00:00:00:00/ff:ff:00:00:00:00 actions=drop
cookie=0x0, duration=101.086s, table=0, n_packets=2, n_bytes=84, priority=1 actions=goto_table:1
cookie=0x0, duration=101.086s, table=1, n_packets=0, n_bytes=0, priority=3,dl_dst=ff:ff:ff:ff:ff:ff actions=FLOOD
cookie=0x0, duration=3.826s, table=1, n_packets=0, n_bytes=0, idle_timeout=180, priority=2,in_port=2,dl_src=f7:7e:c9:dc:a0:82,dl_dst=3a:ec:21:0e:0d:8b actions=output:1
cookie=0x0, duration=101.086s, table=1, n_packets=2, n_bytes=84, priority=1 actions=CONTROLLER:65535
```
ここで，転送テーブルが以下の内容に更新される
table_id = 1 (転送テーブル)

|優先度|処理内容                                      |
|:--|:---------------------------------------------------|
|3  |宛先MACアドレスがブロードキャストアドレスであればパケットをフラッディング|
|1  |コントローラにpacketIn|
|2  |送信元portが2,送信元MACアドレスがf7:7e:c9:dc:a0:82,送信先MACアドレスが3a:ec:21:0e:0d:8bのパケットをポート１へ転送|

この処理はコントローラがFDBを参照することで，新たにhost2からのパケットをhost1に転送するエントリがフローテーブルに追加される．
