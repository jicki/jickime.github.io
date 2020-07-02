# Coreos kube-prometheus 监控


# kube-prometheus

> kube-prometheus 既 Prometheus Operator
>
> kube-prometheus creates/configures/manages Prometheus clusters atop Kubernetes


**Components included in this package:**

* The Prometheus Operator
* Highly available Prometheus
* Highly available Alertmanager
* Prometheus node-exporter
* Prometheus Adapter for Kubernetes Metrics APIs
* kube-state-metrics
* Grafana


```
# 要求

Kubernetes cluster of version >=1.8.0

```

```
# 说明

我们一般监控系统基本上来说都是读取系统上面的 /proc 和 /sys 的信息

prometheus 的 node-exporter 服务配置选项 --path.procfs 和 --path.sysfs 指定从这俩选项的值的proc和sys读取,容器跑node-exporter 只需要挂载宿主机的 /proc 和 /sys 到容器fs的某个路径挂载属性设置为readonly 后用这两个选项指定即可读取到.

```

```
Prometheus 与 Prometheus-operator 的区别
    
    Prometheus  -  是主动去拉取数据的, 所以在k8s里pod因为调度的原因导致pod的ip会发生变化,人工不可能去维持.

    Prometheus-operator  -  是自主定义的CRD资源以及Controller的实现，Prometheus Operator这个controller有RBAC权限下去负责监听 这些自定义资源的变化，并且根据这些资源的定义自动化的完成如Prometheus Server自身以及配置的自动化管理工作 在Kubernetes中我们使用Deployment、DamenSet，StatefulSet来管理应用Workload，使用Service，Ingress来管理应用的访问方式，使用ConfigMap和Secret来管理应用配置。我们在集群中对这些资源的创建，更新，删除的动作都会被转换为事件(Event)，Kubernetes的Controller Manager负责监听这些事件并触发相应的任务来满足用户的期望。这种方式我们成为声明式，用户只需要关心应用程序的最终状态，其它的都通过Kubernetes来帮助我们完成，通过这种方式可以大大简化应用的配置管理复杂度。


    Prometheus Operator 架构图如下:
```


![图13][13]



## CustomResourceDefinitions


   `Prometheus`  定义了所需要的Prometheus部署, Kind: Prometheus 类型中, 所描述部署的 Prometheus Server 集群.

   `ServiceMonitor`  以声明方式指定应如何监控服务组数据, yaml中通过Selector来依据 Labels 选取对应的Service的 endpoints，并让 Prometheus Server 通过 Service 进行拉取（拉）指标资料(也就是metrics信息), metrics信息要在http的url输出符合metrics格式的信息.

   `PrometheusRule`  它定义了一个所需的Prometheus规则文件，该文件可以由包含Prometheus警报和记录规则的Prometheus实例加载。

   `Alertmanager`  通过 `kind: Alertmanager` 自定义资源来描述信息，再由 Operator 依据描述内容部署 Alertmanager 集群。



## 部署 kube-Prometheus


> 官方 github https://github.com/coreos/kube-prometheus



**下载官方的yaml**

按照以下顺序导入:


`namespaces`

```
wget https://raw.githubusercontent.com/coreos/kube-prometheus/master/manifests/00namespace-namespace.yaml

```


`prometheus-operator`


```
wget https://raw.githubusercontent.com/coreos/kube-prometheus/master/manifests/0prometheus-operator-serviceAccount.yaml


wget https://raw.githubusercontent.com/coreos/kube-prometheus/master/manifests/0prometheus-operator-0alertmanagerCustomResourceDefinition.yaml


wget https://raw.githubusercontent.com/coreos/kube-prometheus/master/manifests/0prometheus-operator-0podmonitorCustomResourceDefinition.yaml


wget https://raw.githubusercontent.com/coreos/kube-prometheus/master/manifests/0prometheus-operator-0prometheusCustomResourceDefinition.yaml


wget https://raw.githubusercontent.com/coreos/kube-prometheus/master/manifests/0prometheus-operator-0prometheusruleCustomResourceDefinition.yaml


wget https://raw.githubusercontent.com/coreos/kube-prometheus/master/manifests/0prometheus-operator-0servicemonitorCustomResourceDefinition.yaml


wget https://raw.githubusercontent.com/coreos/kube-prometheus/master/manifests/0prometheus-operator-clusterRole.yaml


wget https://raw.githubusercontent.com/coreos/kube-prometheus/master/manifests/0prometheus-operator-clusterRoleBinding.yaml


wget https://raw.githubusercontent.com/coreos/kube-prometheus/master/manifests/0prometheus-operator-deployment.yaml


wget https://raw.githubusercontent.com/coreos/kube-prometheus/master/manifests/0prometheus-operator-service.yaml


wget https://raw.githubusercontent.com/coreos/kube-prometheus/master/manifests/0prometheus-operator-serviceMonitor.yaml

```


`prometheus-adapter`


```

wget https://raw.githubusercontent.com/coreos/kube-prometheus/master/manifests/prometheus-adapter-serviceAccount.yaml


wget https://raw.githubusercontent.com/coreos/kube-prometheus/master/manifests/prometheus-adapter-roleBindingAuthReader.yaml


wget https://raw.githubusercontent.com/coreos/kube-prometheus/master/manifests/prometheus-adapter-apiService.yaml


wget https://raw.githubusercontent.com/coreos/kube-prometheus/master/manifests/prometheus-adapter-clusterRole.yaml


wget https://raw.githubusercontent.com/coreos/kube-prometheus/master/manifests/prometheus-adapter-clusterRoleAggregatedMetricsReader.yaml


wget https://raw.githubusercontent.com/coreos/kube-prometheus/master/manifests/prometheus-adapter-clusterRoleBinding.yaml


wget https://raw.githubusercontent.com/coreos/kube-prometheus/master/manifests/prometheus-adapter-clusterRoleBindingDelegator.yaml


wget https://raw.githubusercontent.com/coreos/kube-prometheus/master/manifests/prometheus-adapter-clusterRoleServerResources.yaml


wget https://raw.githubusercontent.com/coreos/kube-prometheus/master/manifests/prometheus-adapter-configMap.yaml


wget https://raw.githubusercontent.com/coreos/kube-prometheus/master/manifests/prometheus-adapter-deployment.yaml


wget https://raw.githubusercontent.com/coreos/kube-prometheus/master/manifests/prometheus-adapter-service.yaml


```



`node-exporter`

```
wget https://raw.githubusercontent.com/coreos/kube-prometheus/master/manifests/node-exporter-serviceAccount.yaml


wget https://raw.githubusercontent.com/coreos/kube-prometheus/master/manifests/node-exporter-clusterRole.yaml


wget https://raw.githubusercontent.com/coreos/kube-prometheus/master/manifests/node-exporter-clusterRoleBinding.yaml


wget https://raw.githubusercontent.com/coreos/kube-prometheus/master/manifests/node-exporter-daemonset.yaml


wget https://raw.githubusercontent.com/coreos/kube-prometheus/master/manifests/node-exporter-service.yaml


wget https://raw.githubusercontent.com/coreos/kube-prometheus/master/manifests/node-exporter-serviceMonitor.yaml

```


`kube-state-metrics`

```
wget https://raw.githubusercontent.com/coreos/kube-prometheus/master/manifests/kube-state-metrics-serviceAccount.yaml


wget https://raw.githubusercontent.com/coreos/kube-prometheus/master/manifests/kube-state-metrics-role.yaml


wget https://raw.githubusercontent.com/coreos/kube-prometheus/master/manifests/kube-state-metrics-clusterRole.yaml


wget https://raw.githubusercontent.com/coreos/kube-prometheus/master/manifests/kube-state-metrics-clusterRoleBinding.yaml


wget https://raw.githubusercontent.com/coreos/kube-prometheus/master/manifests/kube-state-metrics-roleBinding.yaml


wget https://raw.githubusercontent.com/coreos/kube-prometheus/master/manifests/kube-state-metrics-deployment.yaml


wget https://raw.githubusercontent.com/coreos/kube-prometheus/master/manifests/kube-state-metrics-service.yaml


wget https://raw.githubusercontent.com/coreos/kube-prometheus/master/manifests/kube-state-metrics-serviceMonitor.yaml


```

`prometheus`

```
wget https://raw.githubusercontent.com/coreos/kube-prometheus/master/manifests/prometheus-serviceAccount.yaml


wget https://raw.githubusercontent.com/coreos/kube-prometheus/master/manifests/prometheus-clusterRole.yaml


wget https://raw.githubusercontent.com/coreos/kube-prometheus/master/manifests/prometheus-clusterRoleBinding.yaml


wget https://raw.githubusercontent.com/coreos/kube-prometheus/master/manifests/prometheus-roleBindingConfig.yaml


wget https://raw.githubusercontent.com/coreos/kube-prometheus/master/manifests/prometheus-roleBindingSpecificNamespaces.yaml


wget https://raw.githubusercontent.com/coreos/kube-prometheus/master/manifests/prometheus-roleConfig.yaml


wget https://raw.githubusercontent.com/coreos/kube-prometheus/master/manifests/prometheus-roleSpecificNamespaces.yaml


wget https://raw.githubusercontent.com/coreos/kube-prometheus/master/manifests/prometheus-rules.yaml


wget https://raw.githubusercontent.com/coreos/kube-prometheus/master/manifests/prometheus-prometheus.yaml


wget https://raw.githubusercontent.com/coreos/kube-prometheus/master/manifests/prometheus-service.yaml


wget https://raw.githubusercontent.com/coreos/kube-prometheus/master/manifests/prometheus-serviceMonitor.yaml


wget https://raw.githubusercontent.com/coreos/kube-prometheus/master/manifests/prometheus-serviceMonitorApiserver.yaml


wget https://raw.githubusercontent.com/coreos/kube-prometheus/master/manifests/prometheus-serviceMonitorCoreDNS.yaml


wget https://raw.githubusercontent.com/coreos/kube-prometheus/master/manifests/prometheus-serviceMonitorKubeControllerManager.yaml


wget https://raw.githubusercontent.com/coreos/kube-prometheus/master/manifests/prometheus-serviceMonitorKubeScheduler.yaml


wget https://raw.githubusercontent.com/coreos/kube-prometheus/master/manifests/prometheus-serviceMonitorKubelet.yaml
```

`alertmanager`


```
wget https://raw.githubusercontent.com/coreos/kube-prometheus/master/manifests/alertmanager-serviceAccount.yaml


wget https://raw.githubusercontent.com/coreos/kube-prometheus/master/manifests/alertmanager-secret.yaml



wget https://raw.githubusercontent.com/coreos/kube-prometheus/master/manifests/alertmanager-alertmanager.yaml


wget https://raw.githubusercontent.com/coreos/kube-prometheus/master/manifests/alertmanager-service.yaml


wget https://raw.githubusercontent.com/coreos/kube-prometheus/master/manifests/alertmanager-serviceMonitor.yaml
```


`grafana`


```
wget https://raw.githubusercontent.com/coreos/kube-prometheus/master/manifests/grafana-serviceAccount.yaml


wget https://raw.githubusercontent.com/coreos/kube-prometheus/master/manifests/grafana-dashboardDefinitions.yaml


wget https://raw.githubusercontent.com/coreos/kube-prometheus/master/manifests/grafana-dashboardDatasources.yaml


wget https://raw.githubusercontent.com/coreos/kube-prometheus/master/manifests/grafana-dashboardSources.yaml


wget https://raw.githubusercontent.com/coreos/kube-prometheus/master/manifests/grafana-deployment.yaml


wget https://raw.githubusercontent.com/coreos/kube-prometheus/master/manifests/grafana-service.yaml


wget https://raw.githubusercontent.com/coreos/kube-prometheus/master/manifests/grafana-serviceMonitor.yaml

```



### 导入服务


```
# 这里有一个 k8s.gcr.io 的镜像,修改一下


sed -i 's/k8s\.gcr\.io/registry\.cn-hangzhou\.aliyuncs\.com\/google_containers/g' kube-state-metrics-deployment.yaml


kubectl apply -f .

```


### 查看服务


```


[root@kubernetes-1 prometheus]# kubectl get all -n monitoring
NAME                                       READY   STATUS    RESTARTS   AGE
pod/alertmanager-main-0                    2/2     Running   0          26m
pod/alertmanager-main-1                    2/2     Running   0          25m
pod/alertmanager-main-2                    2/2     Running   0          16m
pod/grafana-558647b59-k88zr                1/1     Running   0          26m
pod/kube-state-metrics-b96d568bf-2tdds     4/4     Running   0          10m
pod/node-exporter-fqn64                    2/2     Running   0          26m
pod/node-exporter-gp5wr                    2/2     Running   0          26m
pod/node-exporter-m7kkx                    2/2     Running   0          26m
pod/node-exporter-mlzp2                    2/2     Running   0          26m
pod/node-exporter-pfnfz                    2/2     Running   0          26m
pod/prometheus-adapter-74fc6495d7-m5fqr    1/1     Running   0          26m
pod/prometheus-k8s-0                       3/3     Running   1          27m
pod/prometheus-k8s-1                       3/3     Running   1          27m
pod/prometheus-operator-69bd579bf9-94qw4   1/1     Running   0          28m

NAME                            TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)             AGE
service/alertmanager-main       ClusterIP   10.254.26.190   <none>        9093/TCP            26m
service/alertmanager-operated   ClusterIP   None            <none>        9093/TCP,6783/TCP   26m
service/grafana                 ClusterIP   10.254.63.91    <none>        3000/TCP            26m
service/kube-state-metrics      ClusterIP   None            <none>        8443/TCP,9443/TCP   26m
service/node-exporter           ClusterIP   None            <none>        9100/TCP            26m
service/prometheus-adapter      ClusterIP   10.254.40.87    <none>        443/TCP             26m
service/prometheus-k8s          ClusterIP   10.254.60.94    <none>        9090/TCP            27m
service/prometheus-operated     ClusterIP   None            <none>        9090/TCP            27m
service/prometheus-operator     ClusterIP   None            <none>        8080/TCP            28m

NAME                           DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
daemonset.apps/node-exporter   5         5         5       5            5           kubernetes.io/os=linux   26m

NAME                                  READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/grafana               1/1     1            1           26m
deployment.apps/kube-state-metrics    1/1     1            1           26m
deployment.apps/prometheus-adapter    1/1     1            1           26m
deployment.apps/prometheus-operator   1/1     1            1           28m

NAME                                             DESIRED   CURRENT   READY   AGE
replicaset.apps/grafana-558647b59                1         1         1       26m
replicaset.apps/kube-state-metrics-54c8c98cb5    0         0         0       11m
replicaset.apps/kube-state-metrics-b96d568bf     1         1         1       10m
replicaset.apps/kube-state-metrics-f7bd7b6cd     0         0         0       26m
replicaset.apps/prometheus-adapter-74fc6495d7    1         1         1       26m
replicaset.apps/prometheus-operator-69bd579bf9   1         1         1       28m

NAME                                 READY   AGE
statefulset.apps/alertmanager-main   3/3     26m
statefulset.apps/prometheus-k8s      2/2     27m

```


### 配置web入口

```
# 查看 svc

[root@kubernetes-1 prometheus]# kubectl get svc -n monitoring
NAME                    TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)             AGE
alertmanager-main       ClusterIP   10.254.26.190   <none>        9093/TCP            44m
alertmanager-operated   ClusterIP   None            <none>        9093/TCP,6783/TCP   44m
grafana                 ClusterIP   10.254.63.91    <none>        3000/TCP            44m
kube-state-metrics      ClusterIP   None            <none>        8443/TCP,9443/TCP   45m
node-exporter           ClusterIP   None            <none>        9100/TCP            45m
prometheus-adapter      ClusterIP   10.254.40.87    <none>        443/TCP             45m
prometheus-k8s          ClusterIP   10.254.60.94    <none>        9090/TCP            45m
prometheus-operated     ClusterIP   None            <none>        9090/TCP            45m
prometheus-operator     ClusterIP   None            <none>        8080/TCP            47m

```


```
# 首先配置 prometheus 的 webui 配置一个 ingress

vi prometheus-ingress.yaml


apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: prometheus-ingress
  namespace: monitoring
spec:
  rules:
  - host: prometheus.jicki.cn
    http:
      paths:
      - backend:
          serviceName: prometheus-k8s
          servicePort: 9090
```


```
# 配置 grafana 的 ingress


vi grafana-ingress.yaml

apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: grafana-ingress
  namespace: monitoring
spec:
  rules:
  - host: grafana.jicki.cn
    http:
      paths:
      - backend:
          serviceName: grafana
          servicePort: 3000

```


```
# 配置 alertmanager 的 ingress

vi alertmanager-ingress.yaml


apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: alertmanager-ingress
  namespace: monitoring
spec:
  rules:
  - host: alertmanager.jicki.cn
    http:
      paths:
      - backend:
          serviceName: alertmanager-main
          servicePort: 9093

```



### 查看 Prometheus


![图1][1]

![图2][2]

![图3][3]

![图4][4]

![图5][5]

![图6][6]


### 查看 grafana

> 初始账号密码都是 admin , 初始登录以后会让你修改密码


![图7][7]


![图8][8]


![图9][9]


![图10][10]

![图11][11]


### 查看 alertmanager


![图12][12]


  [1]: https://jicki.cn/img/posts/kube-prometheus/1.png
  [2]: https://jicki.cn/img/posts/kube-prometheus/2.png
  [3]: https://jicki.cn/img/posts/kube-prometheus/3.png 
  [4]: https://jicki.cn/img/posts/kube-prometheus/4.png 
  [5]: https://jicki.cn/img/posts/kube-prometheus/5.png 
  [6]: https://jicki.cn/img/posts/kube-prometheus/6.png 
  [7]: https://jicki.cn/img/posts/kube-prometheus/7.png 
  [8]: https://jicki.cn/img/posts/kube-prometheus/8.png 
  [9]: https://jicki.cn/img/posts/kube-prometheus/9.png 
  [10]: https://jicki.cn/img/posts/kube-prometheus/10.png 
  [11]: https://jicki.cn/img/posts/kube-prometheus/11.png 
  [12]: https://jicki.cn/img/posts/kube-prometheus/12.png 
  [13]: https://coreos.com/sites/default/files/inline-images/p1.png

