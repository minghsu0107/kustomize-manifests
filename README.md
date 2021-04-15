# Kustomize Manifests
## Overview
This repository provides one-shot deployment for ingress controller, logging stack, and observability platforms on Kubernetes, including:
- Ingress controller (Traefik)
- Logging stack
  - `Loki`: log aggregation system
  - `Promtail`: daemonset that tails logs from stdout and stderr of all pods
  - `Cassandra`: long-term storage backend
- Monitoring stack
  - `Cadvisor`: exposes container metrics
  - `Prometheus node exporter`: scraps metrics from application endpoints and sends to Prometheus server
  - `Kube-state-metrics`: server that listens to the Kubernetes API server and generates metrics about the state of Kubernetes components, such as number of running jobs, available replicas, and number of running/stopped/terminated pods, by polling Kubernetes API
  - `Node-directory-size-metrics`: daemonset that provides metrics in Prometheus format about disk usage on the nodes
  - `Prometheus`: metrics server that collects metrics from Cadvisor, Prometheus node exporter, Kube-state-metrics server and Kubelet metrics
  - `Grafana`: web UI that visualizes collected metrics
  - `Alertmanager`: server that sends alert to sysadmins when alert conditions are met, based on Prometheus metrics
- Tracing components
  - `Opencensus collector`: Opencensus span collector
  - `Jaeger collector`: Jaeger span collector
  - `Jaeger query`: Jaeger web UI
  - `Elasticsearch`: storage backend of Jaeger
  - `Elasticsearch exporter`: server that exports metrics of the Elasticsearch cluster
## Prerequisites
1. A Kubernetes cluster with version `1.18+`
2. Slack webhook URL

You need to replace `slack_api_url` with your webhook URL in `monitoring/all-in-one.yaml` (line 262). You could also set up emails to which Alertmanager sends notification. See details in `monitoring/all-in-one.yaml`.

3. Dynamic volume provisoner (optional)

The default storageclass is `local-path` in this template. You need to change it to the storageclass of your provisioner in `patch.yaml`. However, K3s has a local path provisioner out-of-the-box. So if you are using K3s, you don't have to do any modification.

4. Service load balancer (optional)

The service load balancer will create daemon pods listening on port 80 on all nodes, and proxy external traffic to the ingress service (Traefik). If you are using K3s, you will already have a service LB out-of-the-box. However, service load balancer is optional. You could add your external IPs to service declaration in `ingress/traefik.yaml`. This way, your Traefik instance can receive external traffics without service LB.

Notes for K3s configurations:
- You need to disable Traefik, which is deployed by default.
```bash
kubectl -n kube-system delete helmcharts.helm.cattle.io traefik
sudo systemctl stop k3s
sudo vim /etc/systemd/system/k3s.service # add '--no-deploy traefik' to ExecStart
sudo rm /var/lib/rancher/k3s/server/manifests/traefik.yaml
sudo systemctl daemon-reload
sudo systemctl restart k3s
```
- You need to enable Kubelet metrics on read-only port `10255`.
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
kustomize build tracing | kubectl apply -f -
```
- Alertmanager web will listen on node port `30615`
- Grafana web will listen on node port `31565`
- Prometheus web will listen on node port `30830`
### Grafana Integration
Default account: `admin`.

Default password: `admin`.

Add the following data sources in Grafana:
- Jaeger: `http://jaeger-query.tracing:16686`
- Loki: `http://loki-headless.logging:3100`
- Prometheus: `http://prometheus:9090`

Dashboards:
- [Elasticsearch exporter](https://grafana.com/grafana/dashboards/2322)
- [Cassandra Metrics](https://grafana.com/grafana/dashboards/6258)