# NATS Streaming
## Usage
```bash
kustomize build . | kubectl apply -f -
```
Check logs:
```bash
kubectl logs stan-0 -c stan -n infra
```
- [Grafana dashboard for NATS server](https://grafana.com/grafana/dashboards/2279)