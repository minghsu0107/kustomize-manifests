# Kafka and Zookeeper
- [Broker Configs](https://kafka.apache.org/documentation/#brokerconfigs)
- [Confluent Platform and Apache Kafka Compatibility](https://docs.confluent.io/platform/current/installation/versions-interoperability.html)
## Usage
```bash
kustomize build . | kubectl apply -f -
```
This will create a cluster with 3 controllers (`kafka-0.kafka-hs:9093`, `kafka-1.kafka-hs:9093`, `kafka-2.kafka-hs:9093`) and 2 brokers (`kafka-3.kafka-hs:9092`, `kafka-4.kafka-hs:9092`).
## Test
Test broker and topic operations:
```bash
# create a topic
kubectl -n kafka exec -ti testclient -- ./bin/kafka-topics.sh --bootstrap-server kafka-3.kafka-hs.kafka.svc.cluster.local:9092 --topic messages --create --partitions 1 --replication-factor 2 --config retention.ms=86400001 --config retention.bytes=274877906943
# describe dynamic configs of a topic
kubectl -n kafka exec -ti testclient -- ./bin/kafka-configs.sh --bootstrap-server kafka-3.kafka-hs.kafka.svc.cluster.local:9092 --entity-type topics --entity-name messages --describe
# alter topic configs
kubectl -n kafka exec -ti testclient -- ./bin/kafka-configs.sh --bootstrap-server kafka-3.kafka-hs.kafka.svc.cluster.local:9092 --alter --entity-type topics --entity-name messages --add-config retention.bytes=274877906944
# list topics, should have "messages"
kubectl -n kafka exec -ti testclient -- ./bin/kafka-topics.sh --list --bootstrap-server kafka-3.kafka-hs.kafka.svc.cluster.local:9092
# describe a topic
kubectl -n kafka exec -ti testclient -- ./bin/kafka-topics.sh --topic messages --describe --bootstrap-server kafka-3.kafka-hs.kafka.svc.cluster.local:9092
# delete a topic (marked for deletion)
kubectl -n kafka exec -ti testclient -- ./bin/kafka-topics.sh --delete --topic messages --bootstrap-server kafka-3.kafka-hs.kafka.svc.cluster.local:9092
```
Test consumer and producer functionalities:
```bash
# start consumer
kubectl -n kafka exec -ti testclient -- ./bin/kafka-console-consumer.sh --bootstrap-server kafka-3.kafka-hs.kafka.svc.cluster.local:9092,kafka-4.kafka-hs.kafka.svc.cluster.local:9092 --topic messages --from-beginning
# start producer
kubectl -n kafka exec -ti testclient -- ./bin/kafka-console-producer.sh --broker-list kafka-3.kafka-hs.kafka.svc.cluster.local:9092,kafka-4.kafka-hs.kafka.svc.cluster.local:9092 --topic messages
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
- Grafana dashboard for Confluent Kafka: https://github.com/confluentinc/cp-helm-charts/blob/master/grafana-dashboard/confluent-open-source-grafana-dashboard.json
- To filter out all Kafka-related logs on Loki:
```bash
{app="kafka"} != "SocketServer" != "InvalidReceiveException" != "org.apache.kafka.common.network" != "Thread.java" != "kafka_exporter.go"
```

