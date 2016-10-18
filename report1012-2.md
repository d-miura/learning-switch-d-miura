#課題内容
>複数スイッチに対応したラーニングスイッチ (multi_learning_switch.rb) の動作を説明しよう。
>
>* 複数スイッチの FDB をどのように実現しているか、コードと動作を解説する
>* 動作の様子やフローテーブルの内容もステップごとに確認すること
>* 必要に応じて図解すること
>
>![](https://github.com/handai-trema/deck/blob/develop/week2/multi_learning_switch.jpeg)

#解答
##コードの解説
multi_learning_switch.rbの各ハンドラの動作について解説する

-start
```ruby
def start(_argv)
  @fdbs = {}
  logger.info "#{name} started."
end
```
FDBをコントローラに接続されるスイッチごとに管理するため，インスタンス変数fdbsをハッシュとして宣言している．その後，起動メッセージの表示を行っている．

-switch_ready
```ruby
def switch_ready(datapath_id)
  @fdbs[datapath_id] = FDB.new
end
```
コントローラにスイッチが接続された際の処理．コントローラではスイッチごとにFDBを管理するため，スイッチとのデータパスidをキーとして，ハッシュにFDBを追加している．

-packet_in
```ruby
def packet_in(datapath_id, packet_in)
  return if packet_in.destination_mac.reserved?
  @fdbs.fetch(datapath_id).learn(packet_in.source_mac, packet_in.in_port)
  flow_mod_and_packet_out packet_in
end
```
packet_inが発生した際，はじめにコントローラに宛先MACアドレスが登録されているか確認する．登録されている場合，スイッチのフローテーブルを更新するだけなので，ハンドラの処理を終了する．
もし，パケットの宛先MACアドレスがFDBに登録さていない場合，「パケットの送信元MACアドレスと，パケットを受けたスイッチのポート番号の対応関係」を（データパスidでスイッチを特定し）パケットを受けたスイッチのFDBに学習させる．その上で，flow_mod_and_packet_outハンドラを呼び出す．

```ruby
def age_fdbs
  @fdbs.each_value(&:age)
end
```
fdbsのエージングを管理？


-flow_mod_and_packet_out
```ruby
def flow_mod_and_packet_out(packet_in)
  port_no = @fdbs.fetch(packet_in.dpid).lookup(packet_in.destination_mac)
  flow_mod(packet_in, port_no) if port_no
  packet_out(packet_in, port_no || :flood)
end
```
packet_inのあったスイッチに対応するFDBを参照し，宛先MACアドレスに対応するポート番号が登録されているか調べる．対応するポート番号が存在する場合は，flow_modメッセージを作成してフローテーブルにエントリを登録し，以降はスイッチに処理を任せ，そのポート番号にpacket_outする．対応するポート番号が存在しない場合，フラッディングを行う．


-flow_mod
-packet_out
複数スイッチ対応による変更はなし

##動作の解説
