## 1. traefik部署方式

```yaml
kubectl apply -f traefik-crd.yaml
kubectl apply -f crd.yaml
kubectl apply -f rbac.yaml -n kube-system
kubectl apply -f config.yaml -n kube-system
kubectl apply -f deploy.yaml -n kube-system
kubectl apply -f dashboard-route.yaml -n kube-system
```

## 2. ingress nginx 
宿主机网络暴露：deployment .spec.template.spec.hostNetwork: true 使用宿主机网络，端口暴露在宿主机上

## 3. kubernetes dashboard
  ### 1. 使用以下命令暴露
  kubectl port-forward --namespace kubernetes-dashboard --address 0.0.0.0 service/kubernetes-dashboard 443

  ### 2. 使用以下（master节点ip）URL登录
  https://10.70.128.50

  ### 3. 使用以下命令获取token
  kubectl describe secret/kubernetes-dashboard-token-fxfxp -n kubernetes-dashboard

  ### 4. kubernetes-dashboard.yaml中已做注释，默认的clusterrole权限不够，使用超级使用户cluster-admin，其拥有访问kube-apiserver的所有权限
