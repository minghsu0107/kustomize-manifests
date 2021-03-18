# Kustomize Manifests
## Overview
This repository provides one-shot deployment for ingress controller, logging stack, and observability platforms on Kubernetes, including:
- Ingress controller (Traefik)
- Logging stack (Loki + Promtail)
- Monitoring stack (Prometheus + Grafana + Alertmanager)
- Tracing components (Opencensus Collector + Jaeger + Elasticsearch + Elasticsearch Exporter)
## Prerequisites
1. A Kubernetes cluster
2. Slack webhook URL

You need to replace `slack_api_url` with your webhook URL in `monitoring/all-in-one.yaml` (line 262).

3. Dynamic volume provisoner (optional)

The default storageclass is `local-path` in this template. You need to change it to the storageclass of your provisioner in `patch.yaml`. However, K3s has a local path provisioner out-of-the-box. So if you are using K3s, you don't have to do any modification.

4. Service load balancer (optional)

The service load balancer will create daemon pods listening on port 80 on all nodes, and proxy external traffic to the ingress service (Traefik). If you are using K3s, you will already have a service LB out-of-the-box. However, service load balancer is optional. You could add your external IPs to service declaration in `ingress/traefik.yaml`. This way, your Traefik instance can receive external traffics without service LB.

Note that if you are using K3s, you need to disable Traefik, which is deployed by default.
```bash
kubectl -n kube-system delete helmcharts.helm.cattle.io traefik
sudo systemctl stop k3s
sudo vim /etc/systemd/system/k3s.service # add '--no-deploy traefik' to ExecStart
sudo rm /var/lib/rancher/k3s/server/manifests/traefik.yaml
sudo systemctl daemon-reload
sudo service k3s restart
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
kustomize build ingress | kubectl apply -f -
kustomize build logging | kubectl apply -f -
kustomize build monitoring | kubectl apply -f -
kustomize build tracing | kubectl apply -f -
```
- Alertmanager web will listen to node port `30615`
- Grafana web will listen to node port `31565`
- Prometheus web will listen to node port `30830`
### Grafana Integration
Add the following data sources in Grafana:
- Jaeger: `http://jaeger.tracing:16686`
- Loki: `http://loki-headless.logging:3100`
- Prometheus: `http://prometheus:9090`

In addition, to view metrics collected from the elasticsearch exporter, you could import the [this dashboard](https://grafana.com/grafana/dashboards/2322) in Grafana.