> ### 阿良老师微信：init1024

# K8s管理平台开发

## 1、Kubernetes API使用

### 1.1 API是什么？

API（Application Programming Interface，应用程序接口）： 是一些预先定义的接口（如函数、HTTP接口），或指软件系统不同组成部分衔接的约定。 用来提供应用程序与开发人员基于某软件或硬件得以访问的一组例程，而又无需访问源码，或理解内部工作机制的细节。 

K8s也提供API接口，提供这个接口的是管理节点的apiserver组件，apiserver服务负责提供HTTP API，以便用户、其他组件相互通信。  

有两种方式可以操作K8s中的资源：

- HTTP API：https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.19/  
- 客户端库： https://kubernetes.io/zh/docs/reference/using-api/client-libraries/  

### 1.2 K8s认证方式

K8s支持三种客户端身份认证：

- HTTPS 证书认证：基于CA证书签名的数字证书认证

- HTTP Token认证：通过一个Token来识别用户

- HTTP Base认证：用户名+密码的方式认证

HTTPS证书认证（kubeconfig）：

```
import os
from kubernetes import client, config
config.load_kube_config(file_path)  # 指定kubeconfig配置文件
apps_api = client.AppsV1Api()  # 资源接口类实例化

for dp in apps_api.list_deployment_for_all_namespaces().items:
    print(dp)
```

HTTP Token认证（ServiceAccount）：

```
from kubernetes import client, config
configuration = client.Configuration()
configuration.host = "https://192.168.31.61:6443"  # APISERVER地址
configuration.ssl_ca_cert="ca.crt"  # CA证书 
configuration.verify_ssl = True   # 启用证书验证
configuration.api_key = {"authorization": "Bearer " + token}  # 指定Token字符串
client.Configuration.set_default(configuration)
apps_api = client.AppsV1Api() 
```

获取Token字符串：创建service account并绑定默认cluster-admin管理员集群角色：

```
# 创建用户
$ kubectl create serviceaccount dashboard-admin -n kube-system
# 用户授权
$ kubectl create clusterrolebinding dashboard-admin --clusterrole=cluster-admin --serviceaccount=kube-system:dashboard-admin
# 获取用户Token
$ kubectl describe secrets -n kube-system $(kubectl -n kube-system get secret | awk '/dashboard-admin/{print $1}')
```

其他常用资源接口类实例化：

```
core_api = client.CoreV1Api()  # namespace,pod,service,pv,pvc
apps_api = client.AppsV1Api()  # deployment
networking_api = client.NetworkingV1beta1Api()  # ingress
storage_api = client.StorageV1Api()  # storage_class
```

### 1.3 示例

#### 客户端库

Deployment操作：

```
# 查询
for dp in apps_api.list_deployment_for_all_namespaces().items:
    print(dp.metadata.name)

# 创建
namespace = "default"
name = "api-test"
replicas = 3
labels = {'a':'1', 'b':'2'}  # 不区分数据类型，都要加引号
image = "nginx"
body = client.V1Deployment(
            api_version="apps/v1",
            kind="Deployment",
            metadata=client.V1ObjectMeta(name=name),
            spec=client.V1DeploymentSpec(
                replicas=replicas,
                selector={'matchLabels': labels},
                template=client.V1PodTemplateSpec(
                    metadata=client.V1ObjectMeta(labels=labels),
                    spec=client.V1PodSpec(
                        containers=[client.V1Container(
                            name="web",
                            image=image
                        )]
                    )
                ),
            )
        )
try:
    apps_api.create_namespaced_deployment(namespace=namespace, body=body)
except Exception as e:
    status = getattr(e, "status")
    if status == 400:
        print(e)
        print("格式错误")
    elif status == 403:
        print("没权限")
# 删除
namespace = "default"
name = "api-test"
apps_api.delete_namespaced_deployment(namespace=namespace, name=name)
```

 Service操作：

```
# 查询
for svc in core_api.list_namespaced_service(namespace="default").items:
    print(svc.metadata.name)

# 创建
namespace = "default"
name = "api-test"
selector = {'a':'1', 'b':'2'}  # 不区分数据类型，都要加引号
port = 80
target_port = 80
type = "NodePort"
body = client.V1Service(
    api_version="v1",
    kind="Service",
    metadata=client.V1ObjectMeta(
        name=name
    ),
    spec=client.V1ServiceSpec(
        selector=selector,
        ports=[client.V1ServicePort(
            port=port,
            target_port=target_port
        )],
        type=type
    )
)
core_api.create_namespaced_service(namespace=namespace, body=body)

# 删除
core_api.delete_namespaced_service(namespace=namespace, name=name)
```

#### HTTP API

```
token="eyJhbGciOiJSUzI1NiIsI..."
curl --cacert /etc/kubernetes/pki/ca.crt -H "Authorization: Bearer $token"  https://192.168.31.61:6443/api/v1/namespaces/default/pods
```

```
curl https://192.168.31.61:6443/api/v1/nodes \
--cacert /etc/kubernetes/pki/ca.crt \
--cert /etc/kubernetes/pki/apiserver-kubelet-client.crt \
--key /etc/kubernetes/pki/apiserver-kubelet-client.key   
```

## 2、 网站布局

### 2.1 开发登录页面

python manage.py startapp dashboard

视图：index、login

### 2.2 登录认证（集成RBAC）

![](https://k8s-1252881505.cos.ap-beijing.myqcloud.com/web-dev/k8s-auth.jpg)

devops/devops/k8s.py 单独存放常用的功能函数。

登录认证检查：

```
from kubernetes import client, config
from django.shortcuts import redirect
import os

def auth_check(auth_type, str):
    if auth_type == "token":
        token = str
        configuration = client.Configuration()
        configuration.host = "https://192.168.31.61:6443"  # APISERVER地址
        configuration.ssl_ca_cert = r"C:\Users\zhenl\Desktop\devops\ca.crt"  # CA证书
        configuration.verify_ssl = True  # 启用证书验证
        configuration.api_key = {"authorization": "Bearer " + token}  # 指定Token字符串
        client.Configuration.set_default(configuration)
        try:
            core_api = client.CoreApi()
            core_api.get_api_versions()  # 查询资源测试
            return True
        except Exception:
            return False
    elif auth_type == "kubeconfig":
        random_str = str
        file_path = os.path.join('kubeconfig', random_str)
        config.load_kube_config(r"%s" % file_path)
        try:
            core_api = client.CoreApi()
            core_api.get_api_versions()  # 查询资源测试
            return True
        except Exception:
            return False
```

登录认证装饰器：

```
def self_login_required(func):
    def inner(request, *args, **kwargs):
        is_login = request.session.get('is_login', False)
        if is_login:
            return func(request, *args, **kwargs)
        else:
            return redirect("/login")
    return inner
```

视图：

```
from django.shortcuts import render
from django.http import JsonResponse
from kubernetes import client, config
import os,hashlib,random
from devops import k8s
# Create your views here.

@k8s.self_login_required
def index(request):
    return  render(request, 'index.html')

def login(request):
    if request.method == "GET":
        return  render(request, 'login.html')
    elif request.method == "POST":
        token = request.POST.get("token", None)
        if token:
            if k8s.auth_check('token', token):
                request.session['is_login'] = True
                request.session['auth_type'] = 'token'
                request.session['token'] = token
                code = 0
                msg = "认证成功"
            else:
                code = 1
                msg = "Token无效！"
        else:
            file_obj = request.FILES.get("file")
            random_str = hashlib.md5(str(random.random()).encode()).hexdigest()
            file_path = os.path.join('kubeconfig', random_str)
            try:
                with open(file_path, 'w', encoding='utf8') as f:
                    data = file_obj.read().decode()  # bytes转str
                    f.write(data)
            except Exception:
                code = 1
                msg = "文件类型错误！"
            if k8s.auth_check('kubeconfig', random_str):
                request.session['is_login'] = True
                request.session['auth_type'] = 'token'
                request.session['token'] = random_str
                code = 0
                msg = "认证成功"
            else:
                code = 1
                msg = "认证文件无效！"
        res = {'code': code, 'msg': msg}
        return JsonResponse(res)
```



### 2.3 管理平台页面布局

根据布局创建对象的django 应用。

- dashboard
- k8s
- workload
- loadbalancer
- storage



导航栏显示：

```
layui-nav-itemed  # 展开的样式类
layui-this  # 子菜单选中背景样式类

<li class="layui-nav-item {% block nav-item-1 %}{% endblock %}">
    <a href="javascript:;">Kubernetes</a>
    <dl class="layui-nav-child">
    <dd><a href="{% url 'node' %}" class="{% block nav-this-1-1 %}{% endblock %}">Nodes</a></dd>
    <dd><a href="{% url 'namespace' %}" class="{% block nav-this-1-2 %}{% endblock %}">Namespaces</a></dd>
    <dd><a href="javascript:;" class="{% block nav-this-1-3 %}{% endblock %}">PersistentVolumes</a></dd>
    </dl>
</li>
```

### 2.4 命名空间选择

思路：登录成功跳转到首页->ajax GET请求namespace接口获取所有命名空间并追究到select列表项，再设置default为默认命名空间->再将当前命名空间存储到本地session，实现其他页面能获取，携带命名空间请求接口。

## 3、数据表格展示K8s常见资源

大致思路：

1. 使用Layui从接口获取JSON数据，动态渲染表格

2. Django准备接口，以JSON格式返回

   资源增删改查采用不同HTTP方法：

   | HTTP方法 | **数据处理** | **说明**     |
   | -------- | ------------ | ------------ |
   | POST     | 新增         | 新增一个资源 |
   | GET      | 获取         | 取得一个资源 |
   | PUT      | 更新         | 更新一个资源 |
   | DELETE   | 删除         | 删除一个资源 |

3. 接口类实例化，遍历获取接口数据，取对应字段值，组成一个字典

### 3.1 Namespace

查询：

```
for ns in core_api.list_namespace().items:
    name = ns.metadata.name
    labels = ns.metadata.labels
    create_time = ns.metadata.creation_timestamp
    namespace = {"name": name, "labels": labels, "create_time": create_time}
```

> 字段：名称、标签、创建时间

删除：

```
core_api.delete_namespace(name=name)
```

创建：

```
body = client.V1Namespace(
    api_version="v1",
    kind="Namespace",
    metadata=client.V1ObjectMeta(
    name=ns_name
    )
)
core_api.create_namespace(body=body)
```

数据表格：

```
      table.render({
        elem: '#test'
        ,url:'{% url 'namespace_api' %}'
        ,toolbar: '#toolbarDemo' //开启头部工具栏，并为其绑定左侧模板
        ,defaultToolbar: ['filter', 'exports', 'print', { //自定义头部工具栏右侧图标。如无需自定义，去除该参数即可
          title: '提示'
          ,layEvent: 'LAYTABLE_TIPS'
          ,icon: 'layui-icon-tips'
        }]
        ,cols: [[
          {field: 'name', title: '名称', sort: true}
          ,{field: 'labels', title: '标签',templet: labelsFormat}
          ,{field: 'create_time', title: '创建时间'}
          ,{fixed: 'right', title:'操作', toolbar: '#barDemo', width:150}
        ]]
        ,page: true
      });
      // 标签格式化，是一个对象
      function labelsFormat(d){
          result = "";
          if (d.labels == null){
              return "None"
          } else {
              for (let key in d.labels) {
                  result += '<span style="border: 1px solid #d6e5ec;border-radius: 8px">' +
                      key + ':' + d.labels[key] +
                      '</span><br>'
              }
              return result
          }
      }
```

### 3.2 Node

查询：

```
for node in core_api.list_node_with_http_info()[0].items:
    name = node.metadata.name
    labels = node.metadata.labels
    status = node.status.conditions[-1].status
    scheduler = ("是" if node.spec.unschedulable is None else "否")
    cpu = node.status.capacity['cpu']
    memory = node.status.capacity['memory']
    kebelet_version = node.status.node_info.kubelet_version
    cri_version = node.status.node_info.container_runtime_version
    create_time = node.metadata.creation_timestamp
    node = {"name": name, "labels": labels, "status":status,
                 "scheduler":scheduler , "cpu":cpu, "memory":memory,
                 "kebelet_version":kebelet_version, "cri_version":cri_version,
                "create_time": create_time}
```

> 字段：名称、标签、准备就绪、可调度、CPU、内存、kubelet版本、CRI版本、创建时间

数据表格：

```
  table.render({
    elem: '#test'
    ,url:'{% url 'node_api' %}'
    ,toolbar: '#toolbarDemo' //开启头部工具栏，并为其绑定左侧模板
    ,defaultToolbar: ['filter', 'exports', 'print', { //自定义头部工具栏右侧图标。如无需自定义，去除该参数即可
      title: '提示'
      ,layEvent: 'LAYTABLE_TIPS'
      ,icon: 'layui-icon-tips'
    }]
    ,cols: [[
      {field: 'name', title: '名称', sort: true}
      ,{field: 'labels', title: '标签',templet: labelsFormat}
      ,{field: 'status', title: '准备就绪'}
      ,{field: 'scheduler', title: '可调度'}
      ,{field: 'cpu', title: 'CPU'}
      ,{field: 'memory', title: '内存'}
      ,{field: 'kebelet_version', title: 'kubelet版本'}
      ,{field: 'cri_version', title: 'CRI版本'}
      ,{field: 'create_time', title: '创建时间'}
      ,{fixed: 'right', title:'操作', toolbar: '#barDemo', width:150}
    ]]
    ,page: true
  });
  // 标签格式化，是一个对象
      function labelsFormat(d){
          result = "";
          if (d.labels == null){
              return "None"
          } else {
              for (let key in d.labels) {
                  result += '<span style="border: 1px solid #d6e5ec;border-radius: 8px">' +
                      key + ':' + d.labels[key] +
                      '</span><br>'
              }
              return result
          }
   }
```

### 3.3 PV

查询：

```
            for pv in core_api.list_persistent_volume().items:
                name = pv.metadata.name
                capacity = pv.spec.capacity["storage"]
                access_modes = pv.spec.access_modes
                reclaim_policy = pv.spec.persistent_volume_reclaim_policy
                status = pv.status.phase
                if pv.spec.claim_ref is not None:
                    pvc_ns = pv.spec.claim_ref.namespace
                    pvc_name = pv.spec.claim_ref.name
                    pvc = "%s / %s" % (pvc_ns, pvc_name)
                else:
                    pvc = "未绑定"
                storage_class = pv.spec.storage_class_name
                create_time = pv.metadata.creation_timestamp
                pv = {"name": name, "capacity": capacity, "access_modes":access_modes,
                             "reclaim_policy":reclaim_policy , "status":status, "pvc":pvc,
                            "storage_class":storage_class,"create_time": create_time}
```

> 字段：名称、容量、访问模式、回收策略、状态、卷申请（PVC）/命名空间、存储类、创建时间

创建：

```
        name = request.POST.get("name", None)
        capacity = request.POST.get("capacity", None)
        access_mode = request.POST.get("access_mode", None)
        storage_type = request.POST.get("storage_type", None)
        server_ip = request.POST.get("server_ip", None)
        mount_path = request.POST.get("mount_path", None)

		body = client.V1PersistentVolume(
            api_version="v1",
            kind="PersistentVolume",
            metadata=client.V1ObjectMeta(name=name),
            spec=client.V1PersistentVolumeSpec(
                capacity={'storage':capacity},
                access_modes=[access_mode],
                nfs=client.V1NFSVolumeSource(
                    server=server_ip,
                    path="/ifs/kubernetes/%s" %mount_path
                )
            )
        )
        core_api.create_persistent_volume(body=body)
```

删除：

```
core_api.delete_persistent_volume(name=name)
```

数据表格：

```
table.render({
  elem: '#test'
  ,url:'{% url 'pv_api' %}'
  ,toolbar: '#toolbarDemo' //开启头部工具栏，并为其绑定左侧模板
  ,defaultToolbar: ['filter', 'exports', 'print', { //自定义头部工具栏右侧图标。如无需自定义，去除该参数即可
    title: '提示'
    ,layEvent: 'LAYTABLE_TIPS'
    ,icon: 'layui-icon-tips'
  }]
  ,cols: [[
    {field: 'name', title: '名称', sort: true}
    ,{field: 'capacity', title: '容量'}
    ,{field: 'access_modes', title: '访问模式'}
    ,{field: 'reclaim_policy', title: '回收策略'}
    ,{field: 'status', title: '状态'}
    ,{field: 'pvc', title: 'PVC(命名空间/名称)'}
    ,{field: 'storage_class', title: '存储类'}
    ,{field: 'create_time', title: '创建时间'}
    ,{fixed: 'right', title:'操作', toolbar: '#barDemo', width:150}
  ]]
  ,page: true
  ,id: 'pvtb'
});
```

### 3.4 Deployment

查询：

```
for dp in apps_api.list_namespaced_deployment(namespace).items:
    name = dp.metadata.name
    namespace = dp.metadata.namespace
    replicas = dp.spec.replicas
    available_replicas = ( 0 if dp.status.available_replicas is None else dp.status.available_replicas)
    labels = dp.metadata.labels
    selector = dp.spec.selector.match_labels
    containers = {}
    for c in dp.spec.template.spec.containers:
        containers[c.name] = c.image
    create_time = dp.metadata.creation_timestamp
    dp = {"name": name, "namespace": namespace, "replicas":replicas,
                 "available_replicas":available_replicas , "labels":labels, "selector":selector,
                 "containers":containers, "create_time": create_time}
```

> 字段：名称、命名空间、预期副本数、可用副本数、Pod标签选择器、镜像/状态、创建时间

创建：

```
        name = request.POST.get("name",None)
        namespace = request.POST.get("namespace",None)
        image = request.POST.get("image",None)
        replicas = int(request.POST.get("replicas",None))
        # 处理标签
        labels = {}
        try:
            for l in request.POST.get("labels",None).split(","):
                k = l.split("=")[0]
                v = l.split("=")[1]
                labels[k] = v
        except Exception as e:
            res = {"code": 1, "msg": "标签格式错误！"}
            return JsonResponse(res)
        resources = request.POST.get("resources",None)
        health_liveness = request.POST.get("health[liveness]",None)  # {'health[liveness]': ['on'], 'health[readiness]': ['on']}
        health_readiness = request.POST.get("health[readiness]",None)

        if resources == "1c2g":
            resources = client.V1ResourceRequirements(limits={"cpu":"1","memory":"1Gi"},
                                                      requests={"cpu":"0.9","memory":"0.9Gi"})
        elif resources == "2c4g":
            resources = client.V1ResourceRequirements(limits={"cpu": "2", "memory": "4Gi"},
                                                      requests={"cpu": "1.9", "memory": "3.9Gi"})
        elif resources == "4c8g":
            resources = client.V1ResourceRequirements(limits={"cpu": "4", "memory": "8Gi"},
                                                      requests={"cpu": "3.9", "memory": "7.9Gi"})
        else:
            resources = client.V1ResourceRequirements(limits={"cpu":"500m","memory":"1Gi"},
                                                      requests={"cpu":"450m","memory":"900Mi"})
        liveness_probe = ""
        if health_liveness == "on":
            liveness_probe = client.V1Probe(http_get="/",timeout_seconds=30,initial_delay_seconds=30)
        readiness_probe = ""
        if health_readiness == "on":
            readiness_probe = client.V1Probe(http_get="/",timeout_seconds=30,initial_delay_seconds=30)

        for dp in apps_api.list_namespaced_deployment(namespace=namespace).items:
            if name == dp.metadata.name:
                res = {"code": 1, "msg": "Deployment已经存在！"}
                return JsonResponse(res)

        body = client.V1Deployment(
            api_version="apps/v1",
            kind="Deployment",
            metadata=client.V1ObjectMeta(name=name),
            spec=client.V1DeploymentSpec(
                replicas=replicas,
                selector={'matchLabels': labels},
                template=client.V1PodTemplateSpec(
                    metadata=client.V1ObjectMeta(labels=labels),
                    spec=client.V1PodSpec(
                        containers=[client.V1Container(   # https://github.com/kubernetes-client/python/blob/master/kubernetes/docs/V1Container.md
                            name="web",
                            image=image,
                            env=[{"name": "TEST", "value": "123"}, {"name": "DEV", "value": "456"}],
                            ports=[client.V1ContainerPort(container_port=80)],
                            # liveness_probe=liveness_probe, 
                            # readiness_probe=readiness_probe,
                            resources=resources,
                        )]
                    )
                ),
            )
        )

   apps_api.create_namespaced_deployment(namespace=namespace, body=body)
```

删除：

```
apps_api.delete_namespaced_deployment(namespace=namespace, name=name)
```

数据表格：

```
      table.render({
        elem: '#test'
        ,url:'{% url 'deployment_api' %}?namespace=' + namespace
        ,toolbar: '#toolbarDemo' //开启头部工具栏，并为其绑定左侧模板
        ,defaultToolbar: ['filter', 'exports', 'print', { //自定义头部工具栏右侧图标。如无需自定义，去除该参数即可
          title: '提示'
          ,layEvent: 'LAYTABLE_TIPS'
          ,icon: 'layui-icon-tips'
        }]
        ,cols: [[
          {field: 'name', title: '名称', sort: true}
          ,{field: 'namespace', title: '命名空间'}
          ,{field: 'replicas', title: '预期副本数'}
          ,{field: 'available_replicas', title: '可用副本数'}
          ,{field: 'labels', title: '标签',templet: labelsFormat}
          ,{field: 'selector', title: 'Pod标签选择器',templet: selecotrFormat}
          ,{field: 'containers', title: '容器', templet: containersFormat}
          ,{field: 'create_time', title: '创建时间'}
          ,{fixed: 'right', title:'操作', toolbar: '#barDemo', width:150}
        ]]
        ,page: true
          ,id: 'dptb'
      });
      // 标签格式化，是一个对象
      function labelsFormat(d){
          result = "";
          if(d.labels == null){
              return "None"
          } else {
              for (let key in d.labels) {
                  result += '<span style="border: 1px solid #d6e5ec;border-radius: 8px">' +
                      key + ':' + d.labels[key] +
                      '</span><br>'
              }
              return result
          }
      }
      function selecotrFormat(d){
          result = "";
          for(let key in d.selector) {
             result += '<span style="border: 1px solid #d6e5ec;border-radius: 8px">' +
                  key + ':' + d.selector[key] +
                     '</span><br>'
          }
          return result
      }
      function containersFormat(d) {
          result = "";
          for(let key in d.containers) {
              result += key + '=' + d.containers[key] + '<br>'
          }
          return result
      }
```



### 3.5 DaemonSet

查询：

```
for ds in apps_api.list_namespaced_daemon_set(namespace).items:
    name = ds.metadata.name
    namespace = ds.metadata.namespace
    desired_number = ds.status.desired_number_scheduled
    available_number = ds.status.number_available
    labels = ds.metadata.labels
    selector = ds.spec.selector.match_labels
	containers = {}
	for c in ds.spec.template.spec.containers:
		containers[c.name] = c.image    
	create_time = ds.metadata.creation_timestamp
 
    ds = {"name": name, "namespace": namespace, "labels": labels, "desired_number": desired_number,
            "available_number": available_number,
            "selector": selector, "containers": containers, "create_time": create_time}
```

> 字段：名称、命名空间、预期节点数、可用节点数、Pod标签选择器、镜像、创建时间

创建：

        name = request.POST.get("name",None)
        namespace = request.POST.get("namespace",None)
        image = request.POST.get("image",None)
        # 处理标签
        labels = {}
        try:
            for l in request.POST.get("labels",None).split(","):
                k = l.split("=")[0]
                v = l.split("=")[1]
                labels[k] = v
        except Exception as e:
            res = {"code": 1, "msg": "标签格式错误！"}
            return JsonResponse(res)
        resources = request.POST.get("resources",None)
        health_liveness = request.POST.get("health[liveness]",None)  # {'health[liveness]': ['on'], 'health[readiness]': ['on']}
        health_readiness = request.POST.get("health[readiness]",None)
    
        if resources == "1c2g":
            resources = client.V1ResourceRequirements(limits={"cpu":"1","memory":"1Gi"},
                                                      requests={"cpu":"0.9","memory":"0.9Gi"})
        elif resources == "2c4g":
            resources = client.V1ResourceRequirements(limits={"cpu": "2", "memory": "4Gi"},
                                                      requests={"cpu": "1.9", "memory": "3.9Gi"})
        elif resources == "4c8g":
            resources = client.V1ResourceRequirements(limits={"cpu": "4", "memory": "8Gi"},
                                                      requests={"cpu": "3.9", "memory": "7.9Gi"})
        else:
            resources = client.V1ResourceRequirements(limits={"cpu":"500m","memory":"1Gi"},
                                                      requests={"cpu":"450m","memory":"900Mi"})
        liveness_probe = ""
        if health_liveness == "on":
            liveness_probe = client.V1Probe(http_get="/",timeout_seconds=30,initial_delay_seconds=30)
        readiness_probe = ""
        if health_readiness == "on":
            readiness_probe = client.V1Probe(http_get="/",timeout_seconds=30,initial_delay_seconds=30)
    
        for dp in apps_api.list_namespaced_daemon_set(namespace=namespace).items:
            if name == dp.metadata.name:
                res = {"code": 1, "msg": "DaemonSet已经存在！"}
                return JsonResponse(res)
    
        body = client.V1DaemonSet(
            api_version="apps/v1",
            kind="DaemonSet",
            metadata=client.V1ObjectMeta(name=name),
            spec=client.V1DeploymentSpec(
                selector={'matchLabels': labels},
                template=client.V1PodTemplateSpec(
                    metadata=client.V1ObjectMeta(labels=labels),
                    spec=client.V1PodSpec(
                        containers=[client.V1Container(   # https://github.com/kubernetes-client/python/blob/master/kubernetes/docs/V1Container.md
                            name="web",
                            image=image,
                            env=[{"name": "TEST", "value": "123"}, {"name": "DEV", "value": "456"}],
                            ports=[client.V1ContainerPort(container_port=80)],
                            # liveness_probe=liveness_probe, 
                            # readiness_probe=readiness_probe,
                            resources=resources,
                        )]
                    )
                ),
            )
        )
删除：

```
apps_api.delete_namespaced_daemon_set(namespace=namespace, name=name)
```

数据表格：

```
      table.render({
        elem: '#test'
        ,url:'{% url 'daemonset_api' %}?namespace=' + namespace
        ,toolbar: '#toolbarDemo' //开启头部工具栏，并为其绑定左侧模板
        ,defaultToolbar: ['filter', 'exports', 'print', { //自定义头部工具栏右侧图标。如无需自定义，去除该参数即可
          title: '提示'
          ,layEvent: 'LAYTABLE_TIPS'
          ,icon: 'layui-icon-tips'
        }]
        ,cols: [[
          {field: 'name', title: '名称', sort: true}
          ,{field: 'namespace', title: '命名空间',sort: true}
          ,{field: 'desired_number', title: '预期节点数',width: 100}
          ,{field: 'available_number', title: '可用节点数',width: 100}
          ,{field: 'labels', title: '标签',templet: labelsFormat}
          ,{field: 'selector', title: 'Pod 标签选择器',templet: selecotrFormat}
          ,{field: 'containers', title: '容器', templet: containersFormat}
          ,{field: 'create_time', title: '创建时间',width: 200}
          ,{fixed: 'right', title:'操作', toolbar: '#barDemo',width: 150}
        ]]
        ,page: true
          ,id: 'dstb'
      });
      // 标签格式化，是一个对象
      function labelsFormat(d){
          result = "";
          if(d.labels == null){
              return "None"
          } else {
              for (let key in d.labels) {
                  result += '<span style="border: 1px solid #d6e5ec;border-radius: 8px">' +
                      key + ':' + d.labels[key] +
                      '</span><br>'
              }
              return result
          }
      }
      function selecotrFormat(d){
          result = "";
          for(let key in d.selector) {
             result += '<span style="border: 1px solid #d6e5ec;border-radius: 8px">' +
                  key + ':' + d.selector[key] +
                     '</span><br>'
          }
          return result
      }
      function containersFormat(d) {
          result = "";
          for(let key in d.containers) {
              result += key + '=' + d.containers[key] + '<br>'
          }
          return result
      }
```

### 3.6 StatefulSet

查询：

```
for sts in apps_api.list_namespaced_stateful_set(namespace).items:
    name = sts.metadata.name
    namespace = sts.metadata.namespace
    labels = sts.metadata.labels
    selector = sts.spec.selector.match_labels
    replicas = sts.spec.replicas
    ready_replicas = ("0" if sts.status.ready_replicas is None else sts.status.ready_replicas)
    #current_replicas = sts.status.current_replicas
    service_name = sts.spec.service_name
	containers = {}
	for c in sts.spec.template.spec.containers:
		containers[c.name] = c.image    
    create_time = k8s.timestamp_format(sts.metadata.creation_timestamp)
 
    ds = {"name": name, "namespace": namespace, "labels": labels, "replicas": replicas,
            "ready_replicas": ready_replicas, "service_name": service_name,
            "selector": selector, "containers": containers, "create_time": create_time}
```

> 字段：名称、命名空间、Service名称、预期副本数、可用副本数、Pod标签选择器、镜像、创建时间

删除：

```
apps_api.delete_namespaced_stateful_set(namespace=namespace, name=name)
```

数据表格：

```
      table.render({
        elem: '#test'
        ,url:'{% url 'statefulset_api' %}?namespace=' + namespace
        ,toolbar: '#toolbarDemo' //开启头部工具栏，并为其绑定左侧模板
        ,defaultToolbar: ['filter', 'exports', 'print', { //自定义头部工具栏右侧图标。如无需自定义，去除该参数即可
          title: '提示'
          ,layEvent: 'LAYTABLE_TIPS'
          ,icon: 'layui-icon-tips'
        }]
        ,cols: [[
          {field: 'name', title: '名称', sort: true}
          ,{field: 'namespace', title: '命名空间',sort: true}
          ,{field: 'service_name', title: 'Service名称'}
          ,{field: 'replicas', title: '预期副本数',width: 100}
          ,{field: 'ready_replicas', title: '可用副本数',width: 100}
          ,{field: 'labels', title: '标签',templet: labelsFormat}
          ,{field: 'selector', title: 'Pod 标签选择器',templet: selecotrFormat}
          ,{field: 'containers', title: '容器', templet: containersFormat}
          ,{field: 'create_time', title: '创建时间',width: 200}
          ,{fixed: 'right', title:'操作', toolbar: '#barDemo',width: 150}
        ]]
        ,page: true
          ,id: 'ststb'
      });
      // 标签格式化，是一个对象
      function labelsFormat(d){
          result = "";
          if(d.labels == null){
              return "None"
          } else {
              for (let key in d.labels) {
                  result += '<span style="border: 1px solid #d6e5ec;border-radius: 8px">' +
                      key + ':' + d.labels[key] +
                      '</span><br>'
              }
              return result
          }
      }
      function selecotrFormat(d){
          result = "";
          for(let key in d.selector) {
             result += '<span style="border: 1px solid #d6e5ec;border-radius: 8px">' +
                  key + ':' + d.selector[key] +
                     '</span><br>'
          }
          return result
      }
      function containersFormat(d) {
          result = "";
          for(let key in d.containers) {
              result += key + '=' + d.containers[key] + '<br>'
          }
          return result
      }
```



### 3.7 Pod

查询：

```
for po in core_api.list_namespaced_pod(namespace).items:
    name = po.metadata.name
    namespace = po.metadata.namespace
    labels = po.metadata.labels
    pod_ip = po.status.pod_ip

    containers = []  # [{},{},{}]
    status = "None"
    # 只为None说明Pod没有创建（不能调度或者正在下载镜像）
    if po.status.container_statuses is None:
        status = po.status.conditions[-1].reason
    else:
        for c in po.status.container_statuses:
            c_name = c.name
            c_image = c.image

            # 获取重启次数
            restart_count = c.restart_count

            # 获取容器状态
            c_status = "None"
            if c.ready is True:
                c_status = "Running"
            elif c.ready is False:
                if c.state.waiting is not None:
                    c_status = c.state.waiting.reason
                elif c.state.terminated is not None:
                    c_status = c.state.terminated.reason
                elif c.state.last_state.terminated is not None:
                    c_status = c.last_state.terminated.reason

            c = {'c_name': c_name,'c_image':c_image ,'restart_count': restart_count, 'c_status': c_status}
            containers.append(c)

    create_time = k8s.timestamp_format(po.metadata.creation_timestamp)

    po = {"name": name, "namespace": namespace, "pod_ip": pod_ip,
            "labels": labels, "containers": containers, "status": status,
            "create_time": create_time}
```

> 字段：名称、命名空间、IP地址、标签、容器组、状态、创建时间

删除：

```
core_api.delete_namespaced_pod(namespace=namespace, name=name)
```

数据表格：

```
table.render({
  elem: '#test'
  ,url:'{% url 'pod_api' %}?namespace=' + namespace
  ,toolbar: '#toolbarDemo' //开启头部工具栏，并为其绑定左侧模板
  ,defaultToolbar: ['filter', 'exports', 'print', { //自定义头部工具栏右侧图标。如无需自定义，去除该参数即可
    title: '提示'
    ,layEvent: 'LAYTABLE_TIPS'
    ,icon: 'layui-icon-tips'
  }]
  ,cols: [[
    {field: 'name', title: '名称', sort: true}
    ,{field: 'namespace', title: '命名空间',sort: true}
    ,{field: 'pod_ip', title: 'IP地址'}
    ,{field: 'labels', title: '标签', templet: labelsFormat}
    ,{field: 'containers', title: '容器组', templet: containersFormat}
    ,{field: 'status', title: '状态',sort: true, templet: statusFormat}
    ,{field: 'create_time', title: '创建时间'}
    ,{fixed: 'right', title:'操作', toolbar: '#barDemo',width: 250}
  ]]
  ,page: true
  ,id: 'potb'
});
// 标签格式化，是一个对象
function labelsFormat(d){
    result = "";
    if(d.labels == null){
        return "None"
    } else {
        for (let key in d.labels) {
            result += '<span style="border: 1px solid #d6e5ec;border-radius: 8px">' +
                key + ':' + d.labels[key] +
                '</span><br>'
        }
        return result
    }
}
function containersFormat(d) {
    if (d.containers) {
        for(let key in d.containers) {
            data = d.containers[key];
            result += key + ':' + data.c_name  + '=' + data.c_image + '<br>' +
                      '重启次数:' + data.restart_count  + '<br>' +
                      '状态:' + data.c_status + '<br>'
        }
        return result
    } else {
        return "None"
    }
}
// 如果status为None，使用容器状态显示
function statusFormat(d){
    if(d.status == "None"){
        for(let key in d.containers) {
            result += d.containers[key].c_status + '<br>'
        }
        return result
    } else {
        return d.status
    }
}
```

### 3.8 Service

查询：

```
for svc in core_api.list_namespaced_service(namespace=namespace).items:
    name = svc.metadata.name
    namespace = svc.metadata.namespace
    labels = svc.metadata.labels
    type = svc.spec.type
    cluster_ip = svc.spec.cluster_ip
    ports = []
    for p in svc.spec.ports:  # 不是序列，不能直接返回
        port_name = p.name
        port = p.port
        target_port = p.target_port
        protocol = p.protocol
        node_port = ""
        if type == "NodePort":
            node_port = " <br> NodePort: %s" % p.node_port

        port = {'port_name': port_name, 'port': port, 'protocol': protocol, 'target_port':target_port, 'node_port': node_port}
        ports.append(port)

    selector = svc.spec.selector
    create_time = svc.metadata.creation_timestamp

    # 确认是否关联Pod
    endpoint = ""
    for ep in core_api.list_namespaced_endpoints(namespace=namespace).items:
        if ep.metadata.name == name and ep.subsets is None:
            endpoint = "未关联"
        else:
            endpoint = "已关联"

    svc = {"name": name, "namespace": namespace, "type": type,
            "cluster_ip": cluster_ip, "ports": ports, "labels": labels,
            "selector": selector, "endpoint": endpoint, "create_time": create_time}
```

> 字段：名称、命名空间、类型、集群IP、端口信息、Pod标签选择器、后端Pod、创建时间

创建：

```
        name = request.POST.get("name",None)
        namespace = request.POST.get("namespace",None)
        port = int(request.POST.get("port",None))
        target_port = int(request.POST.get("target-port",None))
        labels = {}
        try:
            for l in request.POST.get("labels",None).split(","):
                k = l.split("=")[0]
                v = l.split("=")[1]
                labels[k] = v
        except Exception as e:
            res = {"code": 1, "msg": "标签格式错误！"}
            return JsonResponse(res)
        type = request.POST.get("type","")

        body = client.V1Service(
            api_version="v1",
            kind="Service",
            metadata=client.V1ObjectMeta(
                name=name
            ),
            spec=client.V1ServiceSpec(
                selector=labels,
                ports=[client.V1ServicePort(
                    port=port,
                    target_port=target_port,

                )],
                type=type
            )
        )
       core_api.create_namespaced_service(namespace=namespace, body=body)
```

删除：

```
core_api.delete_namespaced_service(namespace=namespace, name=name)
```

数据表格：

```
table.render({
  elem: '#test'
  ,url:'{% url 'service_api' %}?namespace=' + namespace
  ,toolbar: '#toolbarDemo' //开启头部工具栏，并为其绑定左侧模板
  ,defaultToolbar: ['filter', 'exports', 'print', { //自定义头部工具栏右侧图标。如无需自定义，去除该参数即可
    title: '提示'
    ,layEvent: 'LAYTABLE_TIPS'
    ,icon: 'layui-icon-tips'
  }]
  ,cols: [[
    {field: 'name', title: '名称', sort: true, width: 300}
    ,{field: 'namespace', title: '命名空间',width: 200, sort: true}
    ,{field: 'type', title: '类型',width: 120, sort: true}
    ,{field: 'cluster_ip', title: '集群IP',width: 150}
    ,{field: 'ports', title: '端口信息',templet: portsFormat}
    ,{field: 'labels', title: '标签', templet: labelsFormat}
    ,{field: 'selector', title: 'Pod 标签选择器', templet: selecotrFormat}
    ,{field: 'endpoint', title: '后端 Pod'}
    ,{field: 'create_time', title: '创建时间',width: 200}
    ,{fixed: 'right', title:'操作', toolbar: '#barDemo',width: 150}
  ]]
  ,page: true
    ,id: 'svctb'
});
// 标签格式化，是一个对象
function labelsFormat(d){
    result = "";
    if(d.labels == null){
        return "None"
    } else {
        for (let key in d.labels) {
            result += '<span style="border: 1px solid #d6e5ec;border-radius: 8px">' +
                key + ':' + d.labels[key] +
                '</span><br>'
        }
        return result
    }
}
function selecotrFormat(d){
    result = "";
    for(let key in d.selector) {
       result += '<span style="border: 1px solid #d6e5ec;border-radius: 8px">' +
            key + ':' + d.selector[key] +
               '</span><br>'
    }
    return result
}
function portsFormat(d) {
    result = "";
    for(let key in d.ports) {
        data = d.ports[key];
        result += '名称: ' + data.port_name + '<br>' +
                '端口: ' + data.port + '<br>' +
                '协议: ' + data.protocol + '<br>' +
                '容器端口: ' + data.target_port + '<br>'
    }
    return result
}
```

### 3.9 Ingress

查询：

```
for ing in networking_api.list_namespaced_ingress(namespace=namespace).items:
    name = ing.metadata.name
    namespace = ing.metadata.namespace
    labels = ing.metadata.labels
    service = "None"
    http_hosts = "None"
    for h in ing.spec.rules:
        host = h.host
        path = ("/" if h.http.paths[0].path is None else h.http.paths[0].path)
        service_name = h.http.paths[0].backend.service_name
        service_port = h.http.paths[0].backend.service_port
        http_hosts = {'host': host, 'path': path, 'service_name': service_name, 'service_port': service_port}

    https_hosts = "None"
    if ing.spec.tls is None:
        https_hosts = ing.spec.tls
    else:
        for tls in ing.spec.tls:
            host = tls.hosts[0]
            secret_name = tls.secret_name
            https_hosts = {'host': host, 'secret_name': secret_name}

    create_time = k8s.timestamp_format(ing.metadata.creation_timestamp)

    ing = {"name": name, "namespace": namespace,"labels": labels ,"http_hosts": http_hosts,
            "https_hosts": https_hosts, "service": service, "create_time": create_time

```

> 字段：名称、命名空间、HTTP、HTTPS、关联Service、创建时间

创建：

```
        name = request.POST.get("name",None)
        namespace = request.POST.get("namespace",None)
        host = request.POST.get("host",None)
        path = request.POST.get("path","/")
        svc_name = request.POST.get("svc_name",None)
        svc_port = int(request.POST.get("svc_port",None))

        body = client.NetworkingV1beta1Ingress(
            api_version="networking.k8s.io/v1beta1",
            kind="Ingress",
            metadata=client.V1ObjectMeta(name=name, annotations={
                "nginx.ingress.kubernetes.io/rewrite-target": "/"
            }),
            spec=client.NetworkingV1beta1IngressSpec(
                rules=[client.NetworkingV1beta1IngressRule(
                    host=host,
                    http=client.NetworkingV1beta1HTTPIngressRuleValue(
                        paths=[client.NetworkingV1beta1HTTPIngressPath(
                            path=path,
                            backend=client.NetworkingV1beta1IngressBackend(
                                service_port=svc_port,
                                service_name=svc_name)

                        )]
                    )
                )
                ]
            )
        )
    networking_api.create_namespaced_ingress(namespace=namespace, body=body)
```

删除：

```
networking_api.delete_namespaced_ingress(namespace=namespace, name=name)
```

数据表格：

```
table.render({
  elem: '#test'
  ,url:'{% url 'ingress_api' %}?namespace=' + namespace
  ,toolbar: '#toolbarDemo' //开启头部工具栏，并为其绑定左侧模板
  ,defaultToolbar: ['filter', 'exports', 'print', { //自定义头部工具栏右侧图标。如无需自定义，去除该参数即可
    title: '提示'
    ,layEvent: 'LAYTABLE_TIPS'
    ,icon: 'layui-icon-tips'
  }]
  ,cols: [[
    {field: 'name', title: '名称', sort: true, width: 300}
    ,{field: 'namespace', title: '命名空间',width: 200, sort: true}
    ,{field: 'http_hosts', title: 'HTTP',templet: httpFormat}
    ,{field: 'https_hosts', title: 'HTTPS',templet: httpsFormat}
    ,{field: 'service', title: '关联 Service', templet: serviceFormat}
    ,{field: 'create_time', title: '创建时间',width: 200}
    ,{fixed: 'right', title:'操作', toolbar: '#barDemo',width: 150}
  ]]
  ,page: true
    ,id: 'ingtb'
});
// 标签格式化，是一个对象
function httpFormat(d){
    return "域名: " + d.http_hosts.host + '<br>' + "路径: " + d.http_hosts.path + '<br>'
}
function httpsFormat(d){
    if(d.https_hosts != null){
        return "域名: " + d.https_hosts.host + '<br>' + "证书Secret名称: " + d.https_hosts.secret_name + '<br>';
    } else {
        return "None"
    }
}
function serviceFormat(d) {
    return "名称: " + d.http_hosts.service_name + '<br>' + "端口: " + d.http_hosts.service_port + '<br>';
}
```

### 3.10 PVC

查询：

```
for pvc in core_api.list_namespaced_persistent_volume_claim(namespace=namespace).items:
    name = pvc.metadata.name
    namespace = pvc.metadata.namespace
    labels = pvc.metadata.labels
    storage_class_name = pvc.spec.storage_class_name
    access_modes = pvc.spec.access_modes
    capacity = (pvc.status.capacity if pvc.status.capacity is None else pvc.status.capacity["storage"])
    volume_name = pvc.spec.volume_name
    status = pvc.status.phase
    create_time = pvc.metadata.creation_timestamp

    pvc = {"name": name, "namespace": namespace, "lables": labels,
            "storage_class_name": storage_class_name, "access_modes": access_modes, "capacity": capacity,
            "volume_name": volume_name, "status": status, "create_time": create_time}
```

> 字段：名称、命名空间、状态、卷名称、容量、访问模式、存储类、创建时间

创建：

```
        name = request.POST.get("name", None)
        namespace = request.POST.get("namespace", None)
        storage_class = request.POST.get("storage_class", None)
        access_mode = request.POST.get("access_mode", None)
        capacity = request.POST.get("capacity", None)
        body = client.V1PersistentVolumeClaim(
                api_version="v1",
                kind="PersistentVolumeClaim",
                metadata=client.V1ObjectMeta(name=name,namespace=namespace),
                spec=client.V1PersistentVolumeClaimSpec(
                    storage_class_name=storage_class,   # 使用存储类创建PV，如果不用可去掉
                    access_modes=[access_mode],
                    resources=client.V1ResourceRequirements(
                      requests={"storage" : capacity}
                    )
                )
            )
         core_api.create_namespaced_persistent_volume_claim(namespace=namespace, body=body)
```

删除：

```
core_api.delete_namespaced_persistent_volume_claim(namespace=namespace, name=name)
```

数据表格：

```
layui.use('table', function(){
  var table = layui.table;
  var $ = layui.jquery;

  table.render({
    elem: '#test'
    ,url:'{% url 'pvc_api' %}?namespace=' + namespace
    ,toolbar: '#toolbarDemo' //开启头部工具栏，并为其绑定左侧模板
    ,defaultToolbar: ['filter', 'exports', 'print', { //自定义头部工具栏右侧图标。如无需自定义，去除该参数即可
      title: '提示'
      ,layEvent: 'LAYTABLE_TIPS'
      ,icon: 'layui-icon-tips'
    }]
    ,cols: [[
      {field: 'name', title: '名称', sort: true}
      ,{field: 'namespace', title: '命名空间',sort: true}
      ,{field: 'labels', title: '标签',templet: labelsFormat}
      ,{field: 'status', title: '状态',width: 130}
      ,{field: 'volume_name', title: '卷名称'}
      ,{field: 'capacity', title: '容量',width: 130}
      ,{field: 'access_modes', title: '访问模式'}
      ,{field: 'storage_class_name', title: '存储类'}
      ,{field: 'create_time', title: '创建时间',width: 200}
      ,{fixed: 'right', title:'操作', toolbar: '#barDemo',width: 150}
    ]]
    ,page: true
      ,id: 'pvctb'
  });
  // 标签格式化，是一个对象
  function labelsFormat(d){
      result = "";
      if(d.labels == null){
          return "None"
      } else {
          for (let key in d.labels) {
              result += '<span style="border: 1px solid #d6e5ec;border-radius: 8px">' +
                  key + ':' + d.labels[key] +
                  '</span><br>'
          }
          return result
      }
  }
```

### 3.11 ConfigMap

查询：

```
for cm in core_api.list_namespaced_config_map(namespace=namespace).items:
    name = cm.metadata.name
    namespace = cm.metadata.namespace
    data_length = ("0" if cm.data is None else len(cm.data))
    create_time = cm.metadata.creation_timestamp

    cm = {"name": name, "namespace": namespace, "data_length": data_length, "create_time": create_time}
```

> 字段：名称、命名空间、数据数量、创建时间

删除：

```
core_api.delete_namespaced_config_map(name=name,namespace=namespace)
```

数据表格：

```
table.render({
  elem: '#test'
  ,url:'{% url 'configmap_api' %}?namespace=' + namespace
  ,toolbar: '#toolbarDemo' //开启头部工具栏，并为其绑定左侧模板
  ,defaultToolbar: ['filter', 'exports', 'print', { //自定义头部工具栏右侧图标。如无需自定义，去除该参数即可
    title: '提示'
    ,layEvent: 'LAYTABLE_TIPS'
    ,icon: 'layui-icon-tips'
  }]
  ,cols: [[
    {field: 'name', title: '名称', sort: true}
    ,{field: 'namespace', title: '命名空间',sort: true}
    ,{field: 'data_length', title: '数据数量'}
    ,{field: 'create_time', title: '创建时间'}
    ,{fixed: 'right', title:'操作', toolbar: '#barDemo',width: 150}
  ]]
  ,page: true
    ,id: 'cmtb'
});
// 标签格式化，是一个对象
function labelsFormat(d){
    result = "";
    if(d.labels == null){
        return "None"
    } else {
        for (let key in d.labels) {
            result += '<span style="border: 1px solid #d6e5ec;border-radius: 8px">' +
                key + ':' + d.labels[key] +
                '</span><br>'
        }
        return result
    }
}
```

### 3.12 Secret

查询：

```
for secret in core_api.list_namespaced_secret(namespace=namespace).items:
    name = secret.metadata.name
    namespace = secret.metadata.namespace
    data_length = ("空" if secret.data is None else len(secret.data))
    create_time = secret.metadata.creation_timestamp

    se = {"name": name, "namespace": namespace, "data_length": data_length, "create_time": create_time}
```

> 字段：名称、命名空间、数据数量、创建时间

删除：

```
core_api.delete_namespaced_secret(namespace=namespace, name=name)
```

数据表格：

```
table.render({
  elem: '#test'
  ,url:'{% url 'secret_api' %}?namespace=' + namespace
  ,toolbar: '#toolbarDemo' //开启头部工具栏，并为其绑定左侧模板
  ,defaultToolbar: ['filter', 'exports', 'print', { //自定义头部工具栏右侧图标。如无需自定义，去除该参数即可
    title: '提示'
    ,layEvent: 'LAYTABLE_TIPS'
    ,icon: 'layui-icon-tips'
  }]
  ,cols: [[
    {field: 'name', title: '名称', sort: true}
    ,{field: 'namespace', title: '命名空间'}
    ,{field: 'data_length', title: '数据数量'}
    ,{field: 'create_time', title: '创建时间'}
    ,{fixed: 'right', title:'操作', toolbar: '#barDemo',width: 150}
  ]]
  ,page: true
    ,id: 'secrettb'
});
// 标签格式化，是一个对象
function labelsFormat(d){
    result = "";
    if(d.labels == null){
        return "None"
    } else {
        for (let key in d.labels) {
            result += '<span style="border: 1px solid #d6e5ec;border-radius: 8px">' +
                key + ':' + d.labels[key] +
                '</span><br>'
        }
        return result
    }
}
```

### 优化

1、减少视图函数认证代码

2、显示每个页面导航（面包屑）

3、每个k8s资源都有一个时间，默认是UTC，进行格式化中国时区

## 创建



## 查看YAML

ACE 是一个开源的、独立的、基于浏览器的代码编辑器，可以嵌入到任何web页面或JavaScript应用程序中。  

https://github.com/ajaxorg/ace

```
yum install epel-release -y
yum install npm git -y
git clone https://github.com/ajaxorg/ace.git
cd ace
npm install
node ./Makefile.dryice.js
```

使用步骤：

1、将ace包放到static目录下

2、准备导出yaml接口

3、准备ace editor编辑器页面

4、ajax调用导出的yaml接口并将yaml内容填充到档期打开的编辑器

