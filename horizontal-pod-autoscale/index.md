# Kubernetes Horizontal Pod Autoscaler


# Horizontal Pod Autoscaler

> Pod 水平自动伸缩（Horizontal Pod Autoscaler） 简称 HPA

* `HPA`&emsp; 可以基于CPU利用率自动伸缩 replication controller、deployment和 replica set 中的 pod 数量,（除了 CPU 利用率）也可以 基于其他应程序提供的度量指标 `custom metrics`。

* `Pod` 水平自动伸缩特性由 `Kubernetes API` 资源和控制器实现。资源决定了控制器的行为。 控制器会周期性的获取平均 `CPU` 利用率，并与目标值相比较后来调整 `replication controller` 或 `deployment` 中的副本数量。



## HPA 原理

* Pod 水平自动伸缩的实现是一个控制循环, 由 `controller manager` 的 `--horizontal-pod-autoscaler-sync-period` 参数 指定周期（默认值为15秒）。 每个周期内，`controller manager` 根据每个 `HorizontalPodAutoscaler` 定义中指定的指标查询资源利用率。 `controller manager` 可以从 `resource metrics API`（每个pod 资源指标）和 `custom metrics API`（其他指标）获取指标。

  * 对于每个 pod 的资源指标（如 CPU），控制器从资源指标 API 中获取每一个 `HorizontalPodAutoscaler` 指定 的 pod 的指标，然后，如果设置了目标使用率，控制器获取每个 pod 中的容器资源使用情况，并计算资源使用率。 如果使用原始值，将直接使用原始数据（不再计算百分比）。 然后，控制器根据平均的资源使用率或原始值计算出缩放的比例，进而计算出目标副本数。
    * 注: &emsp; 如果 pod 某些容器不支持资源采集，那么控制器将不会使用该 pod 的 CPU 使用率。 

  * 如果 pod 使用自定义指示，控制器机制与资源指标类似，区别在于自定义指标只使用原始值，而不是使用率。

  * 如果 pod 使用对象指标和外部指标（每个指标描述一个对象信息）。 这个指标将直接跟据目标设定值相比较，并生成一个上面提到的缩放比例。在 autoscaling/v2beta2 版本API中， 这个指标也可以根据 pod 数量平分后再计算。

  * 通常情况下，控制器将从一系列的聚合 API `（metrics.k8s.io、custom.metrics.k8s.io和external.metrics.k8s.io）` 中获取指标数据。 `metrics.k8s.io API` 通常由 `metrics-server`（需要额外启动）提供。

    * 注: &emsp; 自 `Kubernetes 1.11` 起, 从 `Heapster` 获取指标特性已废弃。



## HPA 算法细节

* 从最基本的角度来看，pod 水平自动缩放控制器跟据当前指标和期望指标来计算缩放比例。

  * `期望副本数` = `ceil[当前副本数 * ( 当前指标 / 期望指标 )]`

    * 如: &emsp; 当前指标为200m, 目标设定值为100m, 那么由于`200.0 / 100.0 == 2.0`,  副本数量将会翻倍。
    * 如: &emsp; 当前指标为50m, 副本数量将会减半，因为`50.0 / 100.0 == 0.5`。 
    * 如: &emsp; 计算出的缩放比例接近1.0（跟据`--horizontal-pod-autoscaler-tolerance` 参数全局配置的容忍值，默认为0.1）, 将会放弃本次缩放。




## 创建 HPA

* 如下使用 deployment 进行 HPA 的测试

  * HPA API `v1` 版本只支持`CPU` , `v2beta2` 版本支持多 `metrics(CPU，memory)`以及自定义`metrics`。

```
# 创建 deployment 与 service

[root@k8s-node-1 hpa]# cat nginx-deploy.yaml 
apiVersion: apps/v1
kind: Deployment 
metadata: 
  name: nginx
  labels:
    app: nginx
spec: 
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template: 
    metadata: 
      labels: 
        app: nginx 
        version: v1.0.0
    spec: 
      containers: 
        - name: nginx 
          image: nginx:alpine 
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 80
              name: http
          # 资源的限制
          resources:
            limits:
              cpu: 1000m
              memory: 500Mi
            requests:
              # 1 cpu = 1000m
              cpu: 0.1
              memory: 250Mi
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc
  labels:
    app: nginx
spec:
  ports:
    - port: 80
      name: http
      targetPort: 80
      protocol: TCP
  selector:
    app: nginx

```
 


* 创建 hpa 

  * 通过命令 `kubectl autoscale deployment nginx --min=1 --max=5 --cpu-percent=50` 默认使用 `api v1` 。

```
[root@k8s-node-1 hpa]# cat nginx-hpa.yaml 
apiVersion: autoscaling/v2beta2
kind: HorizontalPodAutoscaler
metadata:
  name: nginx-hpa
  labels:
    app: nginx-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx
  # 最少pod数量
  minReplicas: 1
  # 最多pods数量
  maxReplicas: 5
  metrics:
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 50
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
```



* 查看 hpa 服务

```
[root@k8s-node-1 hpa]# kubectl get hpa
NAME        REFERENCE          TARGETS          MINPODS   MAXPODS   REPLICAS   AGE
nginx-hpa   Deployment/nginx   1%/50%, 0%/50%   1         5         1          81m
```


* 测试一下

```
# 使用 wrk 进行压测
# 因为需要在 docker中执行,所以就直接对 svc 的IP进行测试
docker run --rm williamyeh/wrk -c100 -t5 -d300s http://10.254.60.87

```


```
# 查看 hpa 的情况 (感觉时间有点长, 特别是缩容的时候)

[root@k8s-node-1 hpa]# kubectl get hpa -w
NAME        REFERENCE          TARGETS          MINPODS   MAXPODS   REPLICAS   AGE
nginx-hpa   Deployment/nginx   1%/50%, 0%/50%     1         5         1          82m
nginx-hpa   Deployment/nginx   1%/50%, 999%/50%   1         5         1          83m
nginx-hpa   Deployment/nginx   1%/50%, 999%/50%   1         5         4          83m
nginx-hpa   Deployment/nginx   1%/50%, 999%/50%   1         5         5          83m
nginx-hpa   Deployment/nginx   1%/50%, 265%/50%   1         5         5          84m
nginx-hpa   Deployment/nginx   1%/50%, 231%/50%   1         5         5          85m
nginx-hpa   Deployment/nginx   1%/50%, 245%/50%   1         5         5          85m
nginx-hpa   Deployment/nginx   1%/50%, 255%/50%   1         5         5          86m
nginx-hpa   Deployment/nginx   1%/50%, 0%/50%     1         5         5          87m
nginx-hpa   Deployment/nginx   1%/50%, 0%/50%     1         5         5          90m
nginx-hpa   Deployment/nginx   1%/50%, 0%/50%     1         5         5          92m
nginx-hpa   Deployment/nginx   1%/50%, 0%/50%     1         5         1          93m

```



