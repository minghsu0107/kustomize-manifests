# Thanos Usage
1. Create buckets `prometheus-long-term` and `thanos-ruler` on your object storage. The default object storage is [Minio](https://github.com/minghsu0107/kustomize-manifests/tree/main/app/minio)
2. Execute the following command:

```bash
kustomize build . | kubectl apply -f -
```
