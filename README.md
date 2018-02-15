# 0. Dockerの準備
お使いのPCに下記をインストールしてください:
1. [DockerComunityEdition](https://www.docker.com/community-edition)もしくは[DockerToolBox](https://docs.docker.com/toolbox/overview/)
2. [DockerCompose](https://docs.docker.com/compose/install/)

# 1. standalone
## standaloneコンテナを起動する
```bash
$ docker run --name standalone --hostname standalone -d \
-p 6650:6650 \
-p 8080:8080 \
-v $PWD/data:/pulsar/data \
apachepulsar/pulsar \
bin/pulsar standalone

# JVMのmemoryが足りない場合
$ docker run --name standalone --hostname standalone -d \
-p 6650:6650 \
-p 8080:8080 \
-v $PWD/data:/pulsar/data \
-e PULSAR_MEM=" -Xms512m -Xmx512m -XX:MaxDirectMemorySize=1g" \
apachepulsar/pulsar \
/bin/bash -c "bin/apply-config-from-env.py conf/standalone.conf && bin/pulsar standalone"

# 確認
$ docker ps -a

CONTAINER ID        IMAGE                        COMMAND                  CREATED             STATUS              PORTS                                            NAMES
1a99c999dd95        apachepulsar/pulsar:latest   "bin/pulsar standalon"   11 seconds ago      Up 10 seconds       0.0.0.0:6650->6650/tcp, 0.0.0.0:8080->8080/tcp   standalone

# 終了したいときは
$ docker stop standalone
$ docker rm standalone
```
## ダッシュボードを起動する
```bash
$ docker run -d --name dashboard --link standalone:standalone \
-p 80:80 \
-e SERVICE_URL=http://standalone:8080 \
apachepulsar/pulsar-dashboard
```
ブラウザなどでコンテナの親ホスト:80にアクセスすると各トピックのstats情報を見ることができます。
## standaloneコンテナの中に入る
以降特に指定がなければstandaloneコンテナの中に入って各コマンドを実行してください
```bash
# standaloneコンテナの中に入る
$ docker exec -it standalone /bin/bash
```
## メッセージの送信/受信を試す
```bash
# Consumerを起動
$ bin/pulsar-client consume -s sub persistent://sample/standalone/ns1/topic1

# Producerからメッセージを送信（別ターミナルで）
$ bin/pulsar-client produce -m 'HelloPulsar' persistent://sample/standalone/ns1/topic1

# Consumerがメッセージを受信
----- got message -----
HelloPulsar
```
## パフォーマンス測定ツールを試す
```bash
# Producer
$ bin/pulsar-perf produce persistent://sample/standalone/ns1/topic1

# Consumer
$ bin/pulsar-perf consume persistent://sample/standalone/ns1/topic1
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
## メッセージの送信/受信を試す
```bash
# Consumerを起動
$ bin/pulsar-client consume -s sub persistent://my-prop/standalone/my-ns/topic1

# Producerからメッセージを送信（別ターミナルで）
$ bin/pulsar-client produce -m 'HelloPulsar' persistent://my-prop/standalone/my-ns/topic1

# Consumerがメッセージを受信
----- got message -----
HelloPulsar
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
```
### Shared
```bash
# ConsumerをSharedで起動
$ bin/pulsar-client consume -s sub -t Shared -n 0 persistent://my-prop/standalone/my-ns/topic1

# 同じサブスクリプション名で別のConsumerを起動（別ターミナルで）
$ bin/pulsar-client consume -s sub -t Shared -n 0 persistent://my-prop/standalone/my-ns/topic1

# Producerからメッセージを5個送信（別ターミナルで）
$ bin/pulsar-client produce -m 1,2,3,4,5 persistent://my-prop/standalone/my-ns/topic1

# 各Consumerにメッセージがラウンドロビン（ただし厳密なラウンドロビンではなく多少偏る）で配られる
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
## east / west clusterを起動
```bash
$ git clone https://github.com/nkurihar/pulsar-handson.git
$ cd pulsar-handson
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

# 終了したいときは
$ docker-compose down
```
## メッセージの送受信(west)
```bash
# Consumer
$ docker exec -it pulsarhandson_east-broker_1 \
bin/pulsar-client consume -s sub-west \
persistent://my-prop/west/my-ns/topic1

# Producer（別ターミナル）
$ docker exec -it pulsarhandson_east-broker_1 \
bin/pulsar-client produce -m 'from west' \
persistent://my-prop/west/my-ns/topic1

# Consumerがメッセージを受信
----- got message -----
from west
```
## メッセージの送受信(global)
```bash
# west側 Consumer
$ docker exec -it pulsarhandson_east-broker_1 \
bin/pulsar-client consume -s sub-west \
persistent://my-prop/global/my-ns/topic1

# east側 Consumer（別ターミナル）
$ docker exec -it pulsarhandson_east-broker_1 \
bin/pulsar-client consume -s sub-east \
persistent://my-prop/global/my-ns/topic1

# west側 Producer（別ターミナル）
$ docker exec -it pulsarhandson_east-broker_1 \
bin/pulsar-client produce -m 'from west' \
persistent://my-prop/global/my-ns/topic1

# 各Consumerに各Producerからのメッセージが届く
----- got message -----
from west

# 同様にeast側からもメッセージを送信してみてください
```
