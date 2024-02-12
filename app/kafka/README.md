# Kafka and Zookeeper
- [Broker Configs](https://kafka.apache.org/documentation/#brokerconfigs)
- [Helm chart for cp-kafka](https://github.com/confluentinc/cp-helm-charts/tree/master/charts/cp-kafka)
- [Confluent Platform and Apache Kafka Compatibility](https://docs.confluent.io/platform/current/installation/versions-interoperability.html)
## Usage
```bash
kustomize build . | kubectl apply -f -
```
## Test
Test zookeeper functionalities:
```bash
# show zookeeper pods
for i in 0 1 2; do kubectl exec zk-$i -n kafka -- hostname -f; done
# The servers in a ZooKeeper ensemble use natural numbers as unique identifiers
# Store each server's identifier in a file called myid in the server's data directory
for i in 0 1 2; do echo "myid zk-$i";kubectl exec zk-$i -n kafka -- cat /var/lib/zookeeper/data/myid; done
# show config
kubectl exec zk-0 -n kafka -c zookeeper -- cat /opt/zookeeper/conf/zoo.cfg
# write world to the path /hello on the zk-0 Pod in the ensemble
kubectl exec zk-0 -n kafka -c zookeeper -- zkCli.sh create /hello world
# get the data from the zk-1 Pod
kubectl exec zk-1 -n kafka -c zookeeper -- zkCli.sh get /hello
```
Test broker and topic operations:
```bash
# create a topic
kubectl -n kafka exec -ti testclient -- ./bin/kafka-topics.sh --bootstrap-server kafka-0.kafka-hs.kafka.svc.cluster.local:9092 --topic messages --create --partitions 1 --replication-factor 3 --config retention.ms=86400001 --config retention.bytes=274877906943
# describe dynamic configs of a topic
kubectl -n kafka exec -ti testclient -- ./bin/kafka-configs.sh -bootstrap-server kafka-0.kafka-hs.kafka.svc.cluster.local:9092 --entity-type topics --entity-name messages --describe
# alter topic configs
kubectl -n kafka exec -ti testclient -- ./bin/kafka-configs.sh --bootstrap-server kafka-0.kafka-hs.kafka.svc.cluster.local:9092 --alter --entity-type topics --entity-name messages --add-config retention.bytes=274877906944
# list topics, should have "messages"
kubectl -n kafka exec -ti testclient -- ./bin/kafka-topics.sh --list --bootstrap-server kafka-0.kafka-hs.kafka.svc.cluster.local:9092
# list all topics using zookeeper shell
kubectl -n kafka exec -ti testclient -- ./bin/zookeeper-shell.sh zk-cs.kafka.svc.cluster.local:2181 ls /brokers/topics
# describe a topic
kubectl -n kafka exec -ti testclient -- ./bin/kafka-topics.sh --topic messages --describe --bootstrap-server kafka-0.kafka-hs.kafka.svc.cluster.local:9092
# delete a topic (marked for deletion)
kubectl -n kafka exec -ti testclient -- ./bin/kafka-topics.sh --delete --topic messages --bootstrap-server kafka-0.kafka-hs.kafka.svc.cluster.local:9092
# list topics that are marked deleted using zookeeper shell
kubectl -n kafka exec -ti testclient -- ./bin/zookeeper-shell.sh zk-cs.kafka.svc.cluster.local:2181 ls /admin/delete_topics
# delete a topic using zookeeper shell
kubectl -n kafka exec -ti testclient -- ./bin/zookeeper-shell.sh zk-cs.kafka.svc.cluster.local:2181 deleteall /brokers/topics/messages
# list broker ids using zookeeper shell
kubectl -n kafka exec -ti testclient -- ./bin/zookeeper-shell.sh zk-cs.kafka.svc.cluster.local:2181 ls /brokers/ids
# describe a broker using zookeeper shell
kubectl -n kafka exec -ti testclient -- ./bin/zookeeper-shell.sh zk-cs.kafka.svc.cluster.local:2181 get /brokers/ids/1001
```
Test consumer and producer functionalities:
```bash
# start consumer
kubectl -n kafka exec -ti testclient -- ./bin/kafka-console-consumer.sh --bootstrap-server kafka-0.kafka-hs.kafka.svc.cluster.local:9092 --topic messages --from-beginning
# start producer
kubectl -n kafka exec -ti testclient -- ./bin/kafka-console-producer.sh --broker-list kafka-0.kafka-hs.kafka.svc.cluster.local:9092,kafka-1.kafka-hs.kafka.svc.cluster.local:9092,kafka-2.kafka-hs.kafka.svc.cluster.local:9092 --topic messages
```
Send messages in producer:
```
>hello
>world
```
You should receive messages in consumer:
```
hello
world
```
## Final Notes
- After deleting a topic using the above command, one should also delete the topic directory on each broker (as defined in the logs.dirs and log.dir properties) with `rm -rf` command.
- Check the Confluent platform and Apache Kafka compatibility [here](https://docs.confluent.io/platform/current/installation/versions-interoperability.html). For example, we use Confluent platform version 7.6.0, which maps to Kafka version 3.6.0.
- One can also test the Kafka cluster using [this helper](https://github.com/rmoff/kafka-listeners/tree/master/golang).
- Kafka Exporter
    - [Github](https://github.com/danielqsj/kafka_exporter)
    - [Dockerhub](https://hub.docker.com/r/danielqsj/kafka-exporter)
    - Grafana dashboard ID: 7589
        1. Import dashboard
        2. Go to Kafka Exporter Overview / Settings
        3. Go to variable `instance` and set query as `label_values(kafka_brokers, instance)`
            - the value of variable `instance` is set to the value of label `instance` in metric `kafka_brokers`
        4. Update the dashboard settings and save the variable
- Zookeeper Exporter
    - [Github](https://github.com/dabealu/zookeeper-exporter)
    - [Dockerhub](https://hub.docker.com/r/bitnami/zookeeper-exporter)
    - Grafana dashboard ID: 11442
        1. Import dashboard
        2. Go to Zookeeper Exporter (dabealu) / Settings
        3. Go to variable `instance` and set query as `label_values(zk_up,instance)`
        4. Go to variable `job` and set query as `label_values(zk_up,job)`
        5. Go to variable `version` and set query as `label_values(zk_version{job=~"$job", instance=~"$instance"}, version)`
        6. Update the dashboard settings and save the variable
- To filter out all Kafka-related logs on Loki:
```bash
{app="kafka"} != "SocketServer" != "InvalidReceiveException" != "org.apache.kafka.common.network" != "Thread.java" != "kafka_exporter.go"
```

