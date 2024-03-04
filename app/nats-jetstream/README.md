# NATS JetStream
## Usage
```bash
kustomize build . | kubectl apply -f -
```

- https://grafana.com/grafana/dashboards/14725-nats-jetstream/

## Test

Start a subscription:
```bash
kubectl -n infra exec -ti nats-box -- nats sub -s nats://o7N95haCEk5kmoae@jetstream:4222 test.demo
```

Publish 5 messages and check whether we could receive them:
```bash
kubectl -n infra exec -ti nats-box -- nats pub -s nats://o7N95haCEk5kmoae@jetstream:4222 test.demo "message {{.Count}} @ {{.TimeStamp}}" --count=5
```
