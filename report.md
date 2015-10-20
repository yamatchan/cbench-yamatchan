#cbenchレポート課題
Rubyプロファイラ(ruby-prof)により分析したスレッド呼び出し時間の上位10件を以下に示す．  
これより，**Kernel**や**BinData::Base**に処理時間を要していることが分かる．  
従って，これらのボトルネックを解消する必要がある．  

```
%self   total     self    wait      child    calls   name
 2.55   3.066    2.924       0      0.142   285778   BinData::Base#get_parameter
 2.49   2.924    2.848       0      0.076  1054515   Kernel#respond_to?
 2.48   5.446    2.847       0      2.599   219936   Kernel#clone
 2.37  24.051    2.72        0     21.331   219936   BinData::Base#new
 2.25   2.575    2.575       0      0       260082   BasicObject#!
 2.13  20.918    2.436       0     18.482   633145  *BinData::BasePrimitive#_value
 2.03  26.385    2.325       0      24.06   219936   BinData::SanitizedPrototype#instantiate
 1.97  24.343    2.256       0     22.087   224788   BinData::Struct#instantiate_obj_at
 1.53   1.754    1.754       0      0       238316   Symbol#to_s
 1.48   7.848    1.7         0      6.148    38900   Hash#each
```

対象コードが./lib/cbench.rbなので，packet_inハンドラからボトルネックを探す．  
packet_inで呼び出されているsend_flow_mod_addメソッドは，
PacketInメッセージのポート番号に+1したポートへ転送するといったFlowModを送っている．  
send_flow_mod_addで指定されているマッチングルールを確認してみると，ExactMatch#newでマッチングルールが生成されている．  
ExactMatch#newは12種類の条件を引数で指定したメッセージと等しいマッチングルールを生成するので，このマッチングルールはそれら12種類の条件が**全て等しいメッセージにしか**適用されない．  
しかし，今回は受信したメッセージのポート番号に+1したポートへ転送するものなので，マッチングルールは**ポート番号のみ**で良い．  
よって，ExactMatch.new(message)をMatch.new(in_port: message.in_port)と変更することで，**未知パケットが到着する確率が低く**なりpacket_inハンドラが呼び出される頻度が少なくなる．   
結果，Match#newやSendOutPort#newの呼び出し回数が減少し，内部で呼びだされているBinData::Base#newの呼び出し回数も減少するのでボトルネックの解消が可能となる．  

コード変更前と変更後で以下のコマンドを実行し比較した．

```
$ ./bin/cbench --port 6633 --switches 1 --loops 10 --ms-per-test 10000 --delay 1000 --throughput
```

実行結果を以下に示す．  
結果，1秒あたりのレスポンスがコード変更前と比較して**約22.7%**上昇したので，ボトルネックの改善ができたといえる．

```
【コード変更前】
RESULT: 1 switches 9 tests min/max/avg/stdev = 176.17/200.90/187.46/10.70 responses/s
```

```
【コード変更後】
RESULT: 1 switches 9 tests min/max/avg/stdev = 211.17/247.84/229.99/14.79 responses/s
```
