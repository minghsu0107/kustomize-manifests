# Kustomize Manifests
## Overview
This repository provides one-shot deployment for ingress controller, logging stack, observability platforms, and some common applications on Kubernetes, including:
- Ingress controller (Traefik)
  - With distributed tracing enabled using Zipkin protocol
  - Send spans to Opencensus span collector: `http://oc-collector.tracing:9411/api/v2/spans`
- Logging stack
  - `Loki`: log aggregation system
    - Save logs for 168 hours
  - `Promtail`: daemonset that tails logs from stdout and stderr of all pods
  - `Cassandra`: long-term storage backend
- Monitoring stack
  - `Kubelet Cadvisor`: exposes container metrics
  - `Prometheus node exporter`: scraps metrics from application endpoints and sends to Prometheus server
  - `Prometheus blackbox exporter`: allows blackbox probing of endpoints over HTTP, HTTPS, DNS, TCP and ICMP. It is used to monitor K8s services here.
  - `Kube-state-metrics`: server that listens to the Kubernetes API server and generates metrics about the state of Kubernetes components, such as number of running jobs, available replicas, and number of running/stopped/terminated pods, by polling Kubernetes API
  - `Node-directory-size-metrics`: daemonset that provides metrics in Prometheus format about disk usage on the nodes
  - `Prometheus`: metrics server that collects metrics from Cadvisor, Prometheus node exporter, Kube-state-metrics server and Kubelet metrics
      - Save metrics for 7 days
  - `Grafana`: web UI that visualizes collected metrics
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
  - `Opencensus collector`: Opencensus span collector
  - `Jaeger collector`: Jaeger span collector
  - `Jaeger query`: Jaeger web UI
  - `Elasticsearch`: storage backend of Jaeger
  - `Elasticsearch exporter`: server that exports metrics of the Elasticsearch cluster
  - `Elasticsearch index cleaner`: cronjob that delete any indices older than 1 day

In addition, some common application deployment templates are provided:
- [Kafka + Zookeeper](app/kafka)
  - Save logs for 72 hours
- [Minio](app/minio)
  - 8 replica servers, each with 10Gi storage
- [NATS Streaming](app/nats-streaming)
  - Save logs for 3 hours
- [Redis cluster](app/redis-cluster)
## Prerequisites
1. A Kubernetes cluster with version `1.18+`
2. Slack webhook URL

You need to replace `slack_api_url` with your webhook URL in `monitoring/all-in-one.yaml` (line 262). You could also set up emails to which Alertmanager sends notification. See details in `monitoring/all-in-one.yaml`.

3. Dynamic volume provisoner (optional)

The default storageclass is `local-path` in this template. You need to change it to the storageclass of your provisioner in `patch.yaml`. However, K3s has a local path provisioner out-of-the-box. So if you are using K3s, you don't have to do any modification.

4. Service load balancer (optional)

The service load balancer will create daemon pods listening on port 80 on all nodes and proxy external traffic to the ingress service (Traefik). If you are using K3s, you will already have a service LB out-of-the-box. However, service load balancer is optional. You could add your external IPs to service declaration in `ingress/traefik.yaml`. This way, your Traefik instance can receive external traffics without service LB.

## Notes for K3s Configurations
- You need to disable Traefik, which is deployed by default.
```bash
kubectl -n kube-system delete helmcharts.helm.cattle.io traefik
sudo systemctl stop k3s
sudo vim /etc/systemd/system/k3s.service # add '--no-deploy traefik' to ExecStart
sudo rm /var/lib/rancher/k3s/server/manifests/traefik.yaml
sudo systemctl daemon-reload
sudo systemctl restart k3s
```
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
```bash
kustomize build . | kubectl apply -f -
```
Alternatively, you could deploy them separately:
```bash
kustomize build ingress | kubectl apply -f -
kustomize build logging | kubectl apply -f -

kustomize build monitoring | kubectl apply -f -
# you could also deploy prometheus with thanos
# thanos ues s3 as storage backend
# kustomize build app/minio | kubectl apply -f -
# kustomize build monitoring-thanos | kubectl apply -f -

kustomize build tracing | kubectl apply -f -
```
Build apps:
```bash
kustomize build app/kafka | kubectl apply -f -
kustomize build app/minio | kubectl apply -f -
kustomize build app/nats-streaming | kubectl apply -f -
kustomize build app/redis-cluster | kubectl apply -f -
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
- Jaeger: `http://jaeger-query.tracing:16686`
- Loki: `http://loki-headless.logging:3100`
- Prometheus: `http://prometheus:9090`
- Thanos (optional): `http://thanos-querier:10902`

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

Note that Prometheus data source name should be set to `prometheus` for all templates provided in this repository.
