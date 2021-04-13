## 1. traefik部署
yaml文件见本[GitHub](https://github.com/NoviceZeng/k8s/tree/main/trafik_deployment)

```yaml
kubectl apply -f traefik-crd.yaml
kubectl apply -f crd.yaml
kubectl apply -f rbac.yaml -n kube-system
kubectl apply -f config.yaml -n kube-system
kubectl apply -f deploy.yaml -n kube-system
kubectl apply -f dashboard-route.yaml -n kube-system
```

## 2. ingress nginx 
### 2.1. 部署
[Ingress Github官网](https://github.com/kubernetes/ingress-nginx)
部署：kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/nginx-0.30.0/deploy/static/mandatory.yaml

### 2.2. 宿主机网络暴露
```deployment .spec.template.spec.hostNetwork: true ``` # 使用宿主机网络，端口暴露在宿主机上
> * 每个node需要启动一个ingress-controller？
> * 每个node都会占用80和443端口？
### 2.3. 使用service的nodeport方式

## 3. [kubernetes dashboard](https://github.com/kubernetes/dashboard)部署以及常见问题
### 3.1. 使用以下命令暴露
kubectl port-forward --namespace kubernetes-dashboard --address 0.0.0.0 service/kubernetes-dashboard 443

### 3.2. 使用以下（master节点ip）URL登录
https://10.70.128.50

### 3.3. 使用以下命令获取token
kubectl describe secret/kubernetes-dashboard-token-fxfxp -n kubernetes-dashboard

### 3.4. 权限报错处理
kubernetes-dashboard.yaml中已做注释，默认的clusterrole权限不够，使用超级使用户cluster-admin，其拥有访问kube-apiserver的所有权限
