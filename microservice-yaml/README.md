 ## 1. Harbor仓库、Helm相关配置 
### 1.1 启用Harbor的Chart仓库服务
harbor 安装helm charts的仓库 
  ``` docker-compose stop
 ./install.sh --with-chartmuseum
 ```

2. 安装push插件
helm plugin install https://github.com/chartmuseum/helm-push

3. 添加repo仓库
helm repo add  --username admin --password Harbor12345 myrepo http://10.70.128.51/chartrepo/microservice

4. 推送与安装chart
helm push mysql-0.3.5.tgz --username=admin --password=Harbor12345 http://10.70.128.51/chartrepo/microservice
如果上面报错，就用第三部添加的仓库名myrepo来执行，helm push mysql-0.3.5.tgz myrepo -u admin -p Harbor12345
 helm install web --version 0.3.5 myrepo/demo


5. Eureka注册中心为基础服务，先部署好，不需要放到Jenkins中进行发布

6. Master 1.20, kuboard安装后master节点自动设置污点
