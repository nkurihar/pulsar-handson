# 0. 準備
1. お使いのPCに[DockerComunityEdition](https://www.docker.com/community-edition)をインストールしてください。ただしOSが対応していないなどで上記がインストールできない場合は代わりに[DockerToolBox](https://docs.docker.com/toolbox/overview/)と[DockerCompose](https://docs.docker.com/compose/install/)をインストールしてください。

2. [DockerHub](https://hub.docker.com/)でDocker IDを作成しておいてください(持っていない方)

3. `apachepulsar/pulsar`, `apachepulsar/pulsar-dashboard`をpullしておいてください:
```
# 2で作成したIDでログインします
$ docker login

Login with your Docker ID to push and pull images from Docker Hub. If you don't have a Docker ID, head over to https://hub.docker.com to create one.
Username (nkurihar@apache.org): nkurihar
Password: 
Login Succeeded

# apachepulsar/pulsarのイメージをダウンロードします
$ docker pull apachepulsar/pulsar

Using default tag: latest
latest: Pulling from apachepulsar/pulsar
c75480ad9aaf: Pull complete 
18d67befbc4e: Pull complete 
1f5d2d0853c7: Pull complete 
5de358416a75: Pull complete 
4049b231edea: Pull complete 
6617c62c7c10: Pull complete 
aa26fbcddb08: Pull complete 
d5b28339f9cb: Pull complete 
8d7b25fab67a: Pull complete 
32be1da32711: Pull complete 
bbbe7896b1d7: Pull complete 
37651c1d6628: Pull complete 
35a0c05fe222: Pull complete 
c336c4b6f88a: Pull complete 
29baf1639e9c: Pull complete 
Digest: sha256:f2647b61b7e31896204cee7ce9edcd35c1fd5bf2aece65feade2abe1ceeb8d80
Status: Downloaded newer image for apachepulsar/pulsar:latest

# 同様にapachepulsar/pulsar-dashboardのイメージをダウンロードしてください
$ docker pull apachepulsar/pulsar-dashboard
```
4. [nkurihar/pulsar-handson](https://github.com/nkurihar/pulsar-handson)をcloneしておいてください(要git)
```
$ git clone https://github.com/nkurihar/pulsar-handson.git
$ cd pulsar-handson
```

# 1. standalone
## コンテナを起動する
```bash
# pulsar-handson/standaloneに移動してください
$ cd ${your_work_directory}/pulsar-handson/standalone

# 起動
$ docker-compose up -d

Creating network "standalone_default" with the default driver
Creating standalone_dashboard_1  ... done
Creating standalone_standalone_1 ... done

# 確認
$ docker-compose ps

         Name                      Command            State         Ports       
--------------------------------------------------------------------------------
standalone_dashboard_1    supervisord -n              Up      80/tcp            
standalone_standalone_1   /bin/bash -c bin/apply-     Up      6650/tcp, 8080/tcp
                          con ... 

# 参考: 終了したいときは下記を実行してください
$ docker-compose down

Stopping standalone_dashboard_1  ... done
Stopping standalone_standalone_1 ... done
Removing standalone_dashboard_1  ... done
Removing standalone_standalone_1 ... done
Removing network standalone_default
```
[Dashboard](https://pulsar.incubator.apache.org/docs/latest/admin/Dashboard/)も同時に起動しており、ブラウザなどでコンテナの親ホスト:80(例. http://localhost:80 )にアクセスすると各トピックのstats情報を見ることができます。

## standaloneコンテナの中に入る
以降特に指定がなければstandaloneコンテナの中に入って各コマンドを実行してください
```bash
# standaloneコンテナの中に入る
$ docker exec -it standalone_standalone_1 /bin/bash
```
## メッセージの送信/受信を試す
```bash
# Consumerを起動
# -n で受信するメッセージの数を指定できます。0にするとずっと受信し続けます。
$ bin/pulsar-client consume -s sub -n 0 persistent://sample/standalone/ns1/topic1

# Producerからメッセージを送信（別ターミナルで）
# -m で送信するメッセージを指定できます。カンマ区切りで複数送信することが可能です。
$ bin/pulsar-client produce -m 'hoge,fuga,bar' persistent://sample/standalone/ns1/topic1

# Consumerがメッセージを受信
----- got message -----
hoge
----- got message -----
fuga
----- got message -----
bar
```
## パフォーマンス測定ツールを試す
```bash
# Producer
# 終了するときはCtrl+C
$ bin/pulsar-perf produce persistent://sample/standalone/ns1/topic1

<中略>
2018-02-16 10:43:51,830 - INFO  - [main:PerformanceProducer@401] - Throughput produced:    100.0  msg/s ---      0.8 Mbit/s --- Latency: mean:   4.714 ms - med:   3.303 - 95pct:   4.563 - 99pct:  64.946 - 99.9pct: 153.931 - 99.99pct: 162.850 - Max: 162.850
...

# Consumer
# 終了するときはCtrl+C
$ bin/pulsar-perf consume persistent://sample/standalone/ns1/topic1

<中略>
2018-02-16 10:49:22,136 - INFO  - [main:PerformanceConsumer@313] - Throughput received: 99.926  msg/s -- 0.781 Mbit/s --- Latency: mean: 7.856 ms - med: 8.000 - 95pct: 12.000 - 99pct: 13.000 - 99.9pct: 13.000 - 99.99pct: 13.000 - Max: 13.000
...

```
# 2. トピック / サブスクリプション
## プロパティを作成する
```bash
$ bin/pulsar-admin properties create -c standalone -r 'my-role' my-prop

# 作成されたことを確認
$ bin/pulsar-admin properties get my-prop

{
  "adminRoles" : [ "my-role" ],
  "allowedClusters" : [ "standalone" ]
}
```
## ネームスペースを作成する
```bash
$ bin/pulsar-admin namespaces create my-prop/standalone/my-ns

# 作成されたことを確認
$ bin/pulsar-admin namespaces list my-prop

my-prop/standalone/my-ns
```
## メッセージの送信/受信を試す(persistent)
```bash
# Consumerを起動
$ bin/pulsar-client consume -s sub -n 0 persistent://my-prop/standalone/my-ns/topic1

# Producerからメッセージを送信（別ターミナルで）
$ bin/pulsar-client produce -m 'hoge,fuga,bar' persistent://my-prop/standalone/my-ns/topic1

# Consumerがメッセージを受信
----- got message -----
hoge
----- got message -----
fuga
----- got message -----
bar

# perf producer
$ bin/pulsar-perf produce persistent://my-prop/standalone/my-ns/topic1

# perf consumer
$ bin/pulsar-perf consume persistent://my-prop/standalone/my-ns/topic1
```
## メッセージの送信/受信を試す(non-persistent)
```bash
# Consumerを起動
$ bin/pulsar-client consume -s sub -n 0 non-persistent://my-prop/standalone/my-ns/topic1

# Producerからメッセージを送信（別ターミナルで）
$ bin/pulsar-client produce -m 'hoge,fuga,bar' non-persistent://my-prop/standalone/my-ns/topic1

# Consumerがメッセージを受信
----- got message -----
hoge
----- got message -----
fuga
----- got message -----
bar

# perf producer
$ bin/pulsar-perf produce non-persistent://my-prop/standalone/my-ns/topic1

# perf consumer
$ bin/pulsar-perf consume non-persistent://my-prop/standalone/my-ns/topic1
```
## サブスクリプション
### Exclusive
```bash
# Consumerを起動
$ bin/pulsar-client consume -s sub persistent://my-prop/standalone/my-ns/topic1

# 同じサブスクリプション名でConsumerの起動しようとすると失敗する（別ターミナルで）
$ bin/pulsar-client consume -s sub persistent://my-prop/standalone/my-ns/topic1

ERROR Error while consuming messages
ERROR Exclusive consumer is already connected
org.apache.pulsar.client.api.PulsarClientException$ConsumerBusyException: Exclusive consumer is already connected
...

# 違うサブスクリプション名であれば接続できる（別ターミナルで）
$ bin/pulsar-client consume -s sub2 persistent://my-prop/standalone/my-ns/topic1
```
### Shared
```bash
# ConsumerをSharedで起動
$ bin/pulsar-client consume -s sub -t Shared -n 0 persistent://my-prop/standalone/my-ns/topic1

# 同じサブスクリプション名で別のConsumerを起動（別ターミナルで）
$ bin/pulsar-client consume -s sub -t Shared -n 0 persistent://my-prop/standalone/my-ns/topic1

# Producerからメッセージを5個送信（別ターミナルで）
$ bin/pulsar-client produce -m 1,2,3,4,5 persistent://my-prop/standalone/my-ns/topic1

# 各Consumerにメッセージがラウンドロビン（ただし厳密なラウンドロビンではなく偏る）で配られる
```
### Failover
```bash
# ConsumerをFailoverで起動
$ bin/pulsar-client consume -s sub -t Failover -n 0 persistent://my-prop/standalone/my-ns/topic1

# 同じサブスクリプション名で別のConsumerを起動（別ターミナルで）
$ bin/pulsar-client consume -s sub -t Failover -n 0 persistent://my-prop/standalone/my-ns/topic1

# Producerからメッセージを5個送信（別ターミナルで）
$ bin/pulsar-client produce -m 1,2,3,4,5 persistent://my-prop/standalone/my-ns/topic1

# メッセージを受信した方のConsumerの接続を切ってから再度送信
$ bin/pulsar-client produce -m 1,2,3,4,5 persistent://my-prop/standalone/my-ns/topic1

# もう1台のConsumerがメッセージを受け取る
```
# 3. Backlog / Retention
## Backlogが溜まる様子を確認
```bash
# Producerからメッセージを10個送信
$ bin/pulsar-client produce -m 'HelloPulsar' -n 10 persistent://my-prop/standalone/my-ns/topic1

# statsを確認(msgBacklog=10になっている)
$ bin/pulsar-admin persistent stats persistent://my-prop/standalone/my-ns/topic1

{
  "msgRateIn" : 0.0,
  "msgThroughputIn" : 0.0,
  "msgRateOut" : 0.0,
  "msgThroughputOut" : 0.0,
  "averageMsgSize" : 0.0,
  "storageSize" : 414,
  "publishers" : [ ],
  "subscriptions" : {
    "sub" : {
      "msgRateOut" : 0.0,
      "msgThroughputOut" : 0.0,
      "msgRateRedeliver" : 0.0,
      "msgBacklog" : 10,
      "blockedSubscriptionOnUnackedMsgs" : false,
      "unackedMessages" : 0,
      "type" : "Shared",
      "msgRateExpired" : 0.0,
      "consumers" : [ ]
    }
  },
  "replication" : { },
  "deduplicationStatus" : "Disabled"
}

# Backlogを消化
$ bin/pulsar-admin persistent skip-all -s sub persistent://my-prop/standalone/my-ns/topic1

# statsを確認(msgBacklog=0になっている)
$ bin/pulsar-admin persistent stats persistent://my-prop/standalone/my-ns/topic1
```
## Retention
```bash
# Retentionを設定
$ bin/pulsar-admin namespaces set-retention -s 1M -t 1h my-prop/standalone/my-ns

# 設定されたことを確認
$ bin/pulsar-admin namespaces get-retention my-prop/standalone/my-ns
{
  "retentionTimeInMinutes" : 60,
  "retentionSizeInMB" : 1
}

# Cursorを戻す
$ bin/pulsar-admin persistent reset-cursor -s sub -t 1h persistent://my-prop/standalone/my-ns/topic1

# statsを確認(msgBacklog=10に戻っている)
$ bin/pulsar-admin persistent stats persistent://my-prop/standalone/my-ns/topic1
```
# 4. GeoReplication
## standaloneは終了しておく
```bash
# ポートが競合してしまうため終了しておいてください
$ docker-compose down
```
## east / west clusterを起動
```bash
# pulsar-handson/georeplicationに移動してください
$ cd ${your_work_directory}/pulsar-handson/georeplication

# 起動
$ docker-compose up -d

# 確認
$ docker-compose ps

                      Name                                    Command               State          Ports       
---------------------------------------------------------------------------------------------------------------
pulsarhandson_dashboard_1                          supervisord -n                   Up       0.0.0.0:80->80/tcp
pulsarhandson_east-bookie_1                        /bin/bash -c bin/apply-con ...   Up                         
pulsarhandson_east-broker_1                        /bin/bash -c bin/apply-con ...   Up       6650/tcp, 8080/tcp
pulsarhandson_east-initialize-cluster-metadata_1   bin/pulsar initialize-clus ...   Exit 0                     
pulsarhandson_east-zookeeper_1                     bin/pulsar zookeeper             Up       2181/tcp          
pulsarhandson_global-zookeeper_1                   bin/pulsar global-zookeeper      Up       2184/tcp          
pulsarhandson_namespace-setup_1                    /bin/bash -c bin/apply-con ...   Exit 0                     
pulsarhandson_west-bookie_1                        /bin/bash -c bin/apply-con ...   Up                         
pulsarhandson_west-broker_1                        /bin/bash -c bin/apply-con ...   Up       6650/tcp, 8080/tcp
pulsarhandson_west-initialize-cluster-metadata_1   bin/pulsar initialize-clus ...   Exit 0                     
pulsarhandson_west-zookeeper_1                     bin/pulsar zookeeper             Up       2181/tcp

# 参考
# コンテナを終了したいときは下記コマンドを実行してください
$ docker-compose down
```
## メッセージの送受信(west)
```bash
# Consumer
$ docker exec -it pulsarhandson_west-broker_1 \
bin/pulsar-client consume -s sub \
persistent://my-prop/west/my-ns/topic1

# Producer（別ターミナル）
$ docker exec -it pulsarhandson_west-broker_1 \
bin/pulsar-client produce -m 'from west' \
persistent://my-prop/west/my-ns/topic1

# Consumerがメッセージを受信
----- got message -----
from west
```
## メッセージの送受信(global)
```bash
# west側 Consumer
$ docker exec -it pulsarhandson_west-broker_1 \
bin/pulsar-client consume -s sub -n 0 \
persistent://my-prop/global/my-ns/topic1

# east側 Consumer（別ターミナル）
$ docker exec -it pulsarhandson_east-broker_1 \
bin/pulsar-client consume -s sub -n 0 \
persistent://my-prop/global/my-ns/topic1

# west側 Producer（別ターミナル）
$ docker exec -it pulsarhandson_west-broker_1 \
bin/pulsar-client produce -m 'from west' \
persistent://my-prop/global/my-ns/topic1

# 各Consumerにwestからのメッセージが届く
----- got message -----
from west

# 同様にeast側からもメッセージを送信してみる
$ docker exec -it pulsarhandson_east-broker_1 \
bin/pulsar-client produce -m 'from east' \
persistent://my-prop/global/my-ns/topic1

# 各Consumerにeastからのメッセージが届く
----- got message -----
from east
```
