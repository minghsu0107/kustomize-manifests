# Tempo Usage
1. Create bucket `tempo` on your object storage. Default object storage is [Minio](https://github.com/minghsu0107/kustomize-manifests/tree/main/app/minio)
2. Execute the following command:

```bash
kustomize build . | kubectl apply -f -
```
