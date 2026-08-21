```
kubectl create ns podinfo

helm repo add podinfo https://stefanprodan.github.io/podinfo
helm repo update

helm install podinfo podinfo/podinfo -n podinfo

kubectl -n podinfo get pods 
```

**Ожидаемый результат:** под `podinfo-xxx` в статусе `Running`.

Прежде чем настраивать PodMonitor, убедитесь, что приложение действительно отдаёт метрики:

```
kubectl -n podinfo port-forward svc/podinfo 9898:9898 &

curl localhost:9898/metrics | head -30 
```
