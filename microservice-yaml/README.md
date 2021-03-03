## 1. 所需环境
**k8s 1.20.1 containerd 1.4.3 centos8**
[kuboard](https://kuboard.cn/install/install-k8s.html) 安装环境

考虑到后续需要docker build、push镜像到仓库(ctr只是cri，不具备docker镜像打包、推送功能)，根据实际情况，一台机器安装docker。

后续Jenkins发布，使用到Jenkins-slave，slave里面也用到docker，所以master节点安装containerd的同时，又安装了docker

 ## 2. Harbor仓库、Helm相关配置 
### 2.1 启用Harbor的Chart仓库服务
harbor 安装helm charts的仓库 
  ``` 
  docker-compose stop
 ./install.sh --with-chartmuseum
 ```

### 2.2 安装push插件
helm plugin install https://github.com/chartmuseum/helm-push

### 2.3 添加repo仓库
```helm repo add  --username admin --password Harbor12345 myrepo http://10.70.128.51/chartrepo/microservice```

### 2.4 推送chart包
```helm push ms-0.1.0.tgz --username=admin --password=Harbor12345 http://10.70.128.51/chartrepo/microservice```
![image](https://user-images.githubusercontent.com/33800153/109740758-c31c6c80-7c06-11eb-9c2f-7471bb031fd8.png)

如果上面报错，直接用chart包名myrepo来push
``` helm push ms-0.1.0.tgz myrepo -u admin -p Harbor12345 ```


## 3. Jenkins安装部署后
### 1.5 Eureka注册中心为基础服务，先部署好，不需要放到Jenkins中进行发布

6. Master 1.20, kuboard安装后master节点自动设置污点
