# Kustomize Manifests
## Overview
This repository provides one-shot deployment for ingress controller, logging stack, observability platforms, and some common applications on Kubernetes, including:
- Ingress controller (`Traefik`)
  - With distributed tracing enabled using Jaeger
  - Send spans to Jaeger collector: `http://jaeger-collector.tracing:14268/api/traces?format=jaeger.thrift`
- Logging stack
  - `Loki`: log aggregation system
    - Save logs on S3 for 72 hours
    - Components
      - Core: Distributor, Ingester, Ruler, Table manager, Querier, Querier frontend
      - Compactor: Dedup the index on S3 and merging all the files to a single file per table every 5 minutes
  - `Promtail`: daemonset that tails logs from stdout and stderr of all pods
- Monitoring stack
  - `Kubelet Cadvisor`: exposes container metrics
  - `Prometheus node exporter`: scraps metrics from application endpoints and sends to Prometheus server
  - `Prometheus blackbox exporter`: allows blackbox probing of endpoints over HTTP, HTTPS, DNS, TCP and ICMP. It is used to monitor K8s services here.
  - `Kube-state-metrics`: server that listens to the Kubernetes API server and generates metrics about the state of Kubernetes components, such as number of running jobs, available replicas, and number of running/stopped/terminated pods, by polling Kubernetes API
  - `Prometheus`: metrics server that collects metrics from Cadvisor, Prometheus node exporter, Kube-state-metrics server and Kubelet metrics
      - Save metrics for 7 days
  - `Grafana`: web UI that visualizes collected metrics (version 8.3.4)
  - `Alertmanager`: server that sends alert to sysadmins when alert conditions are met, based on Prometheus metrics
  - `Thanos components` (optional): a set of components that can be composed into a highly available metric system with unlimited storage capacity
    - Sidecar
      - The main component that runs along Prometheus
      - Reads and archives data on the object store
      - Manages Prometheus’s configuration and lifecycle
      - Injects external labels into the Prometheus configuration to distinguish each Prometheus instance
      - Listens in on Thanos gRPC protocol and translates queries between gRPC and REST
    - Ruler
      - Basically does the same thing as the querier but for Prometheus’ rules. The only difference is that it can communicate with Thanos components.
      - Rule results are written back to disk in the Prometheus 2.0 storage format.
      - Rule nodes at the same time participate in the system as source store nodes, which means that they expose StoreAPI and upload their generated TSDB blocks to an object store.
    - Store
      - Implements the Store API on top of historical data in an object storage bucket
      - Acts primarily as an API gateway and therefore does not need significant amounts of local disk space
      - Joins a Thanos cluster on startup and advertises the data it can access 
      - Keeps a small amount of information about all remote blocks on a local disk in sync with the bucket
      - This data is generally safe to delete across restarts at the cost of increased startup times
    - Querier
      - Listens in on HTTP and translates queries to Thanos gRPC format
      - Aggregates and deduplicate the query result from different sources
      - Can read data from Sidecar, Ruler, and Store
    - Compactor 
      - applies the compaction procedure of the Prometheus 2.0 storage engine to block data stored in object storage
      - Generally not concurrent with safe semantics and must be deployed as a singleton against a bucket
      - Responsible for downsampling data: 5 minute downsampling after 40 hours and 1 hour downsampling after 10 days
    - Two buckets: `prometheus-long-term` and `thanos-ruler` on S3
- Tracing components
  - `Jaeger`: distributed tracing system for monitoring and troubleshooting microservices-based distributed systems
    - Collector: tracing span collector
    - Query: web UI
        - To connect to web UI from `localhost:30188`, run `kubectl -n tracing port-forward deploy/jaeger-query 30188:16686 --address 0.0.0.0`
    - `Elasticsearch`: storage backend of Jaeger
    - `Elasticsearch exporter`: server that exports metrics of the Elasticsearch cluster
    - `Elasticsearch index cleaner`: cronjob that delete any indices older than 1 day
  - `Tempo`: high-volume distributed tracing system that takes advantage of 100% sampling, and only requires an object storage backend
    - Distributor
      - Accepts spans in multiple formats including Jaeger, OpenTelemetry, Zipkin
      - Routes spans to ingesters by hashing the traceID and using a distributed consistent hash ring
    - Ingester
      - Batches trace into blocks, creates bloom filters and indexes, and then flushes it all to the backend
    - Querier
      - Responsible for finding the requested trace id in either the ingesters or the backend storage
      - Depending on parameters it will query both the ingesters and pull bloom/indexes from the backend to search blocks in object storage
    - Query Frontend
      - Queries should be sent to the Query Frontend; responsible for sharding the search space for an incoming query
      - Internally, the Query Frontend splits the blockID space into a configurable number of shards and queues these requests; queriers connect to the Query Frontend via a streaming gRPC connection to process these sharded queries
    - Compactor
      - Streams blocks to and from the backend storage to reduce the total number of blocks
    - Vulture
      - Monitors Grafana Tempo's performance; it pushes traces, queries Tempo, and metrics 404s and traces with missing spans

In addition, some common application deployment templates are provided:
- [Kafka + Zookeeper](app/kafka)
  - Log retention for 72 hours
- [Minio](app/minio)
  - 8 replica servers, each with 10Gi storage
- [NATS Streaming](app/nats-streaming)
  - Log retention for 3 hours
- [Redis cluster](app/redis-cluster)
- [Cassandra cluster](app/cassandra)
- [Kibana](app/kibana)
  - Can be used to visualize data in `elasticsearch.tracing`
  - To connect to web UI from `localhost:30100`, run `kubectl -n kibana port-forward deploy/kibana 30100:5601 --address 0.0.0.0`
## Prerequisites
1. A Kubernetes cluster with version `1.18+`
2. Slack webhook URL

Take `monitoring-thanos` for example. You need to replace `slack_api_url` with your webhook URL in `monitoring-thanos/alertmanager-configmap.yaml`. You could also set up emails to which Alertmanager sends notification.

3. Dynamic volume provisoner (optional)

The default storageclass is `local-path` in this template. You need to change it to the storageclass of your provisioner in `patch.yaml`. However, K3s has a local path provisioner out-of-the-box. So if you are using K3s, you don't have to do any modification.

4. Service load balancer (optional)

The service load balancer will create a daemonset with each pod listening on node ports specified by sevices of type `LoadBalancer` (eg. Traefik service). Those daemon pods will proxy external traffic to these services. If you are using K3s, you will have a service LB out-of-the-box. However, service load balancer is optional. You could add your external IPs to Traefik service declaration (in `ingress/traefik.yaml`). This way, your Traefik instance can receive external traffics without service LB.

## Notes for K3s Configurations
You need to disable Traefik, which is deployed by default.
```bash
kubectl -n kube-system delete helmcharts.helm.cattle.io traefik
sudo systemctl stop k3s
sudo vim /etc/systemd/system/k3s.service # add '--no-deploy traefik' to ExecStart
sudo rm /var/lib/rancher/k3s/server/manifests/traefik.yaml
sudo systemctl daemon-reload
sudo systemctl restart k3s
```
Or you can start K3s server without deploying Traefik by add `--no-deploy traefik`.
- (Optional) You can enable/disable the read-only port for the Kubelet to serve on with no authentication/authorization (set to `0` to disable). Default: 10255.
```bash
# K3s server
server --kubelet-arg "read-only-port=10255"

# K3s agent
agent --kubelet-arg "read-only-port=10255"
```
## Usage
### Network Inspection
Check whether coreDNS works:
```bash
kubectl apply -f testing/dnstest.sh
./dnstest.sh
```
Check whether cluster nodes can ping each other:
```bash
kubectl apply -f testing/overlaytest.sh
./pingtest.sh
```
### One-Shot Deployment
You could adjust the resources of each container as well as storage class and mounted volume size by simply modifying `patch.yaml`.

```bash
kustomize build ingress | kubectl apply -f -

# logging and thanos need s3 as storage backend
kustomize build app/minio | kubectl apply -f -

kustomize build logging | kubectl apply -f -

# remember to create needed buckets in minio beforehand
kustomize build monitoring-thanos | kubectl apply -f -

# use jaeger as tracing platform
kustomize build tracing | kubectl apply -f -

# or use tempo as tracing platform
kustomize build tracing-tempo | kubectl apply -f -
```
Build apps:
```bash
kustomize build app/kafka | kubectl apply -f -
kustomize build app/minio | kubectl apply -f -
kustomize build app/nats-streaming | kubectl apply -f -
kustomize build app/redis-cluster | kubectl apply -f -
kustomize build app/cassandra | kubectl apply -f -
kustomize build app/kibana | kubectl apply -f -
```
- Traefik admin web will listen on port `8082`
- Alertmanager web will listen on node port `30615`
- Grafana web will listen on node port `31565`
- Prometheus web will listen on node port `30830`
- Thanos querier web will listen on node port `30831`
- Thanos ruler web will listen on node port `30832`
- Minio web will listen on node port `30120`
### Grafana Integration
Default account: `admin`.

Default password: `admin`.

Add the following data sources in Grafana:
- Monitoring
  - Prometheus (Thanos): `http://thanos-querier:10902` (datasource name: `prometheus`)
      - Thanos does deduplication and uses external labels for identifying Prometheus replicas.
      - If you use `http://prometheus:9090` as data source, then metrics may duplicate and external labels will become invisible since external labels are added to time series or alerts only when communicating with external systems, such as federation, remote storage, and Alertmanager.
- Tracing
  - Jaeger: `http://jaeger-query.tracing:16686`
  - Tempo
    - Grafana version 7.5.x or higher: `http://query-frontend.tracing:3200` (which is the case in this repository)
    - Grafana version 7.4.x or lower: `http://query-frontend.tracing:16686` (need `tempo-query` as adapter)
- Logging
  - Loki: `http://loki-headless.logging:3100`

Thanos and Tempo data sources are added by default. If you are using different data sources, such as Jaeger for tracing, you can modify [Grafana data sources configuration](monitoring-thanos/configs/grafana-datasources.yaml).

To obtain correlation between logs and traces, one can use [Loki derived fields](https://grafana.com/docs/grafana/latest/datasources/loki/#derived-fields) to parse traceID from logs and link to the tracing web UI. For example, we can link to Jaeger Explore in Grafana:

![](https://i.imgur.com/FasRgsr.png)

Dashboards:
- [Elasticsearch Exporter](https://grafana.com/grafana/dashboards/2322)
- [MinIO Overview](https://grafana.com/grafana/dashboards/13502)
- [Cassandra Metrics](dashboard/cassandra.json)
- [Kafka Exporter](dashboard/kafka.json)
- [NATS Exporter](dashboard/nats.json)
- [Redis Exporter](dashboard/redis.json)
- [Zookeeper Exporter](dashboard/zookeeper.json)
- [Container Metrics](dashboard/container.json)
- [Prometheus Stats](dashboard/prometheus.json)
- [Kube-State Metrics](dashboard/kube-state-metrics.json)
- [Node Exporter Full](https://grafana.com/grafana/dashboards/1860)
- [Jaeger](https://grafana.com/grafana/dashboards/10001)

Note that Prometheus data source name should be set to `prometheus` for all templates provided in this repository.
