# Redis Cluster
[reference](https://rancher.com/blog/2019/deploying-redis-cluster)
## Usage
```bash
kustomize build . | kubectl apply -f -
```
## Join Redis Nodes
Join all nodes:
```bash
kubectl exec -it redis-cluster-0 -n infra -- redis-cli --cluster create --cluster-replicas 1 $(kubectl get pods -n infra -o jsonpath='{range.items[*]}{.status.podIP}:6379 ' | rev | cut -c 7- | rev)
```
Check the cluster info:
```bash
kubectl exec -it redis-cluster-0 -n infra -- redis-cli cluster info
```
```bash
for x in $(seq 0 5); do echo "redis-cluster-$x"; kubectl exec redis-cluster-$x -n infra -- redis-cli role; echo; done
```
Benchmark (should be executed on master node):
```
kubectl exec -it redis-cluster-0 -n infra -- redis-benchmark -h localhost -p 6379
```
## Note
- [Rewriting/compacting append-only files](https://redislabs.com/ebook/part-2-core-concepts/chapter-4-keeping-data-safe-and-ensuring-performance/4-1-persistence-options/4-1-3-rewritingcompacting-append-only-files/)
- [Grafana Dashboard for Redis Exporter](https://grafana.com/grafana/dashboards/763/revisions)