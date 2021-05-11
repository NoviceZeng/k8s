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

hostNetwork类型的pod也存在多方面的缺点：
> * 假如pod重建，pod可能会调度到其他node上，这个时候pod的ip就改变了，那么客户端的配置很可能需要更改
> * 因为与宿主机共享网络协议栈，所以需要考虑端口冲突的问题。如果hostNetwork类型的pod太多，端口冲突将会非常常见，运维成本也会变高
> * 不能使用HostAliases特性，因为Kubelet只管理非hostNetwork类型Pod的hosts文件!

### 2.3. 使用service的nodeport方式
如下面Jenkins的service配置所示：
```
apiVersion: v1
kind: Service
metadata:
  name: jenkins
spec:
  selector:
    name: jenkins
  type: NodePort
  ports:
    - name: http
      port: 80
      targetPort: 8080
      protocol: TCP
      nodePort: 30006
    - name: agent
      port: 50000
      protocol: TCP
```

## 3. [kubernetes dashboard](https://github.com/kubernetes/dashboard)部署以及常见问题
### 3.1. 使用以下命令暴露
kubectl port-forward --namespace kubernetes-dashboard --address 0.0.0.0 service/kubernetes-dashboard 443

### 3.2. 使用以下（master节点ip）URL登录
https://10.70.128.50

### 3.3. 使用以下命令获取token
kubectl describe secret/kubernetes-dashboard-token-fxfxp -n kubernetes-dashboard

### 3.4. 权限报错处理
kubernetes-dashboard.yaml中已做注释，默认的clusterrole权限不够，使用超级使用户cluster-admin，其拥有访问kube-apiserver的所有权限
