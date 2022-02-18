# Kafka and Zookeeper
- [reference 1](https://kow3ns.github.io/kubernetes-zookeeper/manifests/)
- [reference 2](https://kubernetes.io/blog/2017/09/kubernetes-statefulsets-daemonsets/)
- [cp-kafka](https://github.com/confluentinc/cp-helm-charts/tree/master/charts/cp-kafka)
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
kubectl exec zk-0 -n kafka -- cat /opt/zookeeper/conf/zoo.cfg
# write world to the path /hello on the zk-0 Pod in the ensemble
kubectl exec zk-0 -n kafka -- zkCli.sh create /hello world
# get the data from the zk-1 Pod
kubectl exec zk-1 -n kafka -- zkCli.sh get /hello
```
Test broker and topic operations:
```bash
# create a topic
kubectl -n kafka exec -ti testclient -- ./bin/kafka-topics.sh --zookeeper zk-cs.kafka.svc.cluster.local:2181 --topic messages --create --partitions 1 --replication-factor 3
# list topics, should have __consumer_offsets and messages etc.
kubectl -n kafka exec -ti testclient -- ./bin/kafka-topics.sh --list --zookeeper zk-cs.kafka.svc.cluster.local:2181
# list all topics using zookeeper shell
kubectl -n kafka exec -ti testclient -- ./bin/zookeeper-shell.sh zk-cs.kafka.svc.cluster.local:2181 ls /brokers/topics
# describe a topic
kubectl -n kafka exec -ti testclient -- ./bin/kafka-topics.sh --topic messages --describe --zookeeper zk-cs.kafka.svc.cluster.local:2181
# delete a topic (marked for deletion)
kubectl -n kafka exec -ti testclient -- ./bin/kafka-topics.sh --delete --topic messages  --zookeeper zk-cs.kafka.svc.cluster.local:2181
# list topics that are marked deleted
kubectl -n kafka exec -ti testclient -- ./bin/zookeeper-shell.sh zk-cs.kafka.svc.cluster.local:2181 ls /admin/delete_topics
# delete a topic using zookeeper shell
kubectl -n kafka exec -ti testclient -- ./bin/zookeeper-shell.sh zk-cs.kafka.svc.cluster.local:2181 rmr /brokers/topics/messages
# list broker ids using zookeeper shell
kubectl -n kafka exec -ti testclient -- ./bin/zookeeper-shell.sh zk-cs.kafka.svc.cluster.local:2181 ls /brokers/ids
# describe a broker using zookeeper shell
kubectl -n kafka exec -ti testclient -- ./bin/zookeeper-shell.sh zk-cs.kafka.svc.cluster.local:2181 get /brokers/ids/2
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
- Check the Confluent platform and Apache Kafka compatibility [here](https://docs.confluent.io/platform/current/installation/versions-interoperability.html). For example, we use Confluent platform version 6.0.1, which maps to Kafka version 2.6.1.
- One can also test the Kafka cluster using [this helper](https://github.com/rmoff/kafka-listeners/tree/master/golang).
- Kafka Exporter
    - [Github](https://github.com/danielqsj/kafka_exporter)
    - [Dockerhub](https://hub.docker.com/r/danielqsj/kafka-exporter)
    - Grafana dashboard ID: 7589
        1. Import dashboard
        2. Go to Kafka Exporter Overview / Settings
        3. Go to variable `instance` and set query as `label_values(kafka_brokers, instance)`
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

