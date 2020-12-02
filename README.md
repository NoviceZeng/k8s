# k8s
traefik部署方式

```yaml
kubectl apply -f traefik-crd.yaml
kubectl apply -f crd.yaml
kubectl apply -f rbac.yaml -n kube-system
kubectl apply -f config.yaml -n kube-system
kubectl apply -f deploy.yaml -n kube-system
kubectl apply -f dashboard-route.yaml -n kube-system
```
