# prometheus monitoring



{% raw %}
# Prometheus Monitoring


## 部署 Prometheus 


### 镜像下载

```
jicki/prometheus-operator:v0.16.1
jicki/configmap-reload:v0.0.1
jicki/kube-rbac-proxy:v0.2.0
jicki/node-exporter:v0.15.2
jicki/kube-state-metrics:v1.2.0
jicki/addon-resizer:1.0
jicki/monitoring-grafana:4.6.3
jicki/grafana-watcher:v0.0.8
quay.io/prometheus/prometheus:v2.1.0
quay.io/coreos/prometheus-config-reloader:v0.0.2
```


### 创建 namespaces

```
# vi namespace.yaml

apiVersion: v1
kind: Namespace
metadata:
  name: monitoring

```


```
# 导入 yaml

[root@kubernetes-64 monitoring]# kubectl apply -f namespace.yaml
namespace "monitoring" created

```


### prometheus-operator


```
# vi prometheus-operator.yaml


---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: prometheus-operator
  namespace: monitoring
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: prometheus-operator
subjects:
- kind: ServiceAccount
  name: prometheus-operator
  namespace: monitoring
  
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  name: prometheus-operator
  namespace: monitoring
rules:
- apiGroups:
  - extensions
  resources:
  - thirdpartyresources
  verbs:
  - "*"
- apiGroups:
  - apiextensions.k8s.io
  resources:
  - customresourcedefinitions
  verbs:
  - "*"
- apiGroups:
  - monitoring.coreos.com
  resources:
  - alertmanagers
  - prometheuses
  - servicemonitors
  verbs:
  - "*"
- apiGroups:
  - apps
  resources:
  - statefulsets
  verbs: ["*"]
- apiGroups: [""]
  resources:
  - configmaps
  - secrets
  verbs: ["*"]
- apiGroups: [""]
  resources:
  - pods
  verbs: ["list", "delete"]
- apiGroups: [""]
  resources:
  - services
  - endpoints
  verbs: ["get", "create", "update"]
- apiGroups: [""]
  resources:
  - nodes
  verbs: ["list", "watch"]
- apiGroups: [""]
  resources:
  - namespaces
  verbs: ["list"]
  
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: prometheus-operator
  namespace: monitoring
  
---

apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  labels:
    k8s-app: prometheus-operator
  name: prometheus-operator
  namespace: monitoring
spec:
  replicas: 1
  template:
    metadata:
      labels:
        k8s-app: prometheus-operator
    spec:
      containers:
      - args:
        - --kubelet-service=kube-system/kubelet
        - --config-reloader-image=jicki/configmap-reload:v0.0.1
        image: jicki/prometheus-operator:v0.16.1
        name: prometheus-operator
        ports:
        - containerPort: 8080
          name: http
        resources:
          limits:
            cpu: 200m
            memory: 100Mi
          requests:
            cpu: 100m
            memory: 50Mi
      securityContext:
        runAsNonRoot: true
        runAsUser: 65534
      serviceAccountName: prometheus-operator
    
---
apiVersion: v1
kind: Service
metadata:
  name: prometheus-operator
  namespace: monitoring
  labels:
    k8s-app: prometheus-operator
spec:
  type: ClusterIP
  ports:
  - name: http
    port: 8080
    targetPort: http
    protocol: TCP
  selector:
    k8s-app: prometheus-operator


```


```
# 导入 yaml 文件

[root@kubernetes-64 monitoring]# kubectl apply -f prometheus-operator.yaml 
clusterrolebinding "prometheus-operator" created
clusterrole "prometheus-operator" created
serviceaccount "prometheus-operator" created
deployment "prometheus-operator" created
service "prometheus-operator" created


[root@kubernetes-64 monitoring]# kubectl get pod -n monitoring
NAME                                READY     STATUS    RESTARTS   AGE
prometheus-operator-c544bf4-v6z5z   1/1       Running   0          11s

```


###  node-exporter


```
vi node-exporter.yaml


---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: node-exporter
  namespace: monitoring
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: node-exporter
subjects:
- kind: ServiceAccount
  name: node-exporter
  namespace: monitoring
  
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-exporter
  namespace: monitoring
rules:
- apiGroups: ["authentication.k8s.io"]
  resources:
  - tokenreviews
  verbs: ["create"]
- apiGroups: ["authorization.k8s.io"]
  resources:
  - subjectaccessreviews
  verbs: ["create"]
  
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: node-exporter
  namespace: monitoring
  
---
apiVersion: extensions/v1beta1
kind: DaemonSet
metadata:
  name: node-exporter
  namespace: monitoring
spec:
  updateStrategy:
    rollingUpdate:
      maxUnavailable: 1
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: node-exporter
      name: node-exporter
    spec:
      serviceAccountName: node-exporter
      securityContext:
        runAsNonRoot: true
        runAsUser: 65534
      hostNetwork: true
      hostPID: true
      containers:
      - image: jicki/node-exporter:v0.15.2
        args:
        - "--web.listen-address=127.0.0.1:9101"
        - "--path.procfs=/host/proc"
        - "--path.sysfs=/host/sys"
        name: node-exporter
        resources:
          requests:
            memory: 30Mi
            cpu: 100m
          limits:
            memory: 50Mi
            cpu: 200m
        volumeMounts:
        - name: proc
          readOnly:  true
          mountPath: /host/proc
        - name: sys
          readOnly: true
          mountPath: /host/sys
      - name: kube-rbac-proxy
        image: jicki/kube-rbac-proxy:v0.2.0
        args:
        - "--secure-listen-address=:9100"
        - "--upstream=http://127.0.0.1:9101/"
        ports:
        - containerPort: 9100
          hostPort: 9100
          name: https
        resources:
          requests:
            memory: 20Mi
            cpu: 10m
          limits:
            memory: 40Mi
            cpu: 20m
      tolerations:
        - effect: NoSchedule
          operator: Exists
      volumes:
      - name: proc
        hostPath:
          path: /proc
      - name: sys
        hostPath:
          path: /sys
          
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: node-exporter
    k8s-app: node-exporter
  name: node-exporter
  namespace: monitoring
spec:
  type: ClusterIP
  clusterIP: None
  ports:
  - name: https
    port: 9100
    protocol: TCP
  selector:
    app: node-exporter


```


```
# 导入 yaml 文件

[root@kubernetes-64 monitoring]# kubectl apply -f node-exporter.yaml 
clusterrolebinding "node-exporter" created
clusterrole "node-exporter" created
serviceaccount "node-exporter" created
daemonset "node-exporter" created
service "node-exporter" created


[root@kubernetes-64 monitoring]# kubectl get pods -n monitoring -o wide
NAME                  READY     STATUS    RESTARTS   AGE       IP            NODE
node-exporter-2299v   2/2       Running   0          2m        172.16.1.66   kubernetes-66
node-exporter-2lsfc   2/2       Running   0          2m        172.16.1.64   kubernetes-64
node-exporter-mstjl   2/2       Running   0          2m        172.16.1.65   kubernetes-65


```



### kube-state-metrics


```
vi kube-state-metrics.yaml


---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: kube-state-metrics
  namespace: monitoring
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: kube-state-metrics
subjects:
- kind: ServiceAccount
  name: kube-state-metrics
  namespace: monitoring
  
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  name: kube-state-metrics
  namespace: monitoring
rules:
- apiGroups: [""]
  resources:
  - nodes
  - pods
  - services
  - resourcequotas
  - replicationcontrollers
  - limitranges
  - persistentvolumeclaims
  - persistentvolumes
  - namespaces
  - endpoints
  verbs: ["list", "watch"]
- apiGroups: ["extensions"]
  resources:
  - daemonsets
  - deployments
  - replicasets
  verbs: ["list", "watch"]
- apiGroups: ["apps"]
  resources:
  - statefulsets
  verbs: ["list", "watch"]
- apiGroups: ["batch"]
  resources:
  - cronjobs
  - jobs
  verbs: ["list", "watch"]
- apiGroups: ["autoscaling"]
  resources:
  - horizontalpodautoscalers
  verbs: ["list", "watch"]
- apiGroups: ["authentication.k8s.io"]
  resources:
  - tokenreviews
  verbs: ["create"]
- apiGroups: ["authorization.k8s.io"]
  resources:
  - subjectaccessreviews
  verbs: ["create"]
  
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: RoleBinding
metadata:
  name: kube-state-metrics
  namespace: monitoring
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: kube-state-metrics-resizer
subjects:
- kind: ServiceAccount
  name: kube-state-metrics
  
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: Role
metadata:
  name: kube-state-metrics-resizer
  namespace: monitoring
rules:
- apiGroups: [""]
  resources:
  - pods
  verbs: ["get"]
- apiGroups: ["extensions"]
  resources:
  - deployments
  resourceNames: ["kube-state-metrics"]
  verbs: ["get", "update"]
  
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: kube-state-metrics
  namespace: monitoring
  
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: kube-state-metrics
  namespace: monitoring
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: kube-state-metrics
    spec:
      serviceAccountName: kube-state-metrics
      securityContext:
        runAsNonRoot: true
        runAsUser: 65534
      containers:
      - name: kube-rbac-proxy-main
        image: jicki/kube-rbac-proxy:v0.2.0
        args:
        - "--secure-listen-address=:8443"
        - "--upstream=http://127.0.0.1:8081/"
        ports:
        - name: https-main
          containerPort: 8443
        resources:
          requests:
            memory: 20Mi
            cpu: 10m
          limits:
            memory: 40Mi
            cpu: 20m
      - name: kube-rbac-proxy-self
        image: jicki/kube-rbac-proxy:v0.2.0
        args:
        - "--secure-listen-address=:9443"
        - "--upstream=http://127.0.0.1:8082/"
        ports:
        - name: https-self
          containerPort: 9443
        resources:
          requests:
            memory: 20Mi
            cpu: 10m
          limits:
            memory: 40Mi
            cpu: 20m
      - name: kube-state-metrics
        image: jicki/kube-state-metrics:v1.2.0
        args:
        - "--host=127.0.0.1"
        - "--port=8081"
        - "--telemetry-host=127.0.0.1"
        - "--telemetry-port=8082"
      - name: addon-resizer
        image: jicki/addon-resizer:1.0
        resources:
          limits:
            cpu: 100m
            memory: 30Mi
          requests:
            cpu: 100m
            memory: 30Mi
        env:
          - name: MY_POD_NAME
            valueFrom:
              fieldRef:
                fieldPath: metadata.name
          - name: MY_POD_NAMESPACE
            valueFrom:
              fieldRef:
                fieldPath: metadata.namespace
        command:
          - /pod_nanny
          - --container=kube-state-metrics
          - --cpu=100m
          - --extra-cpu=2m
          - --memory=150Mi
          - --extra-memory=30Mi
          - --threshold=5
          - --deployment=kube-state-metrics
          
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: kube-state-metrics
    k8s-app: kube-state-metrics
  name: kube-state-metrics
  namespace: monitoring
spec:
  clusterIP: None
  ports:
  - name: https-main
    port: 8443
    targetPort: https-main
    protocol: TCP
  - name: https-self
    port: 9443
    targetPort: https-self
    protocol: TCP
  selector:
    app: kube-state-metrics
    

```


```
# 导入 yaml 文件

[root@kubernetes-64 monitoring]# kubectl apply -f kube-state-metrics.yaml 
clusterrolebinding "kube-state-metrics" created
clusterrole "kube-state-metrics" created
rolebinding "kube-state-metrics" created
role "kube-state-metrics-resizer" created
serviceaccount "kube-state-metrics" created
deployment "kube-state-metrics" created
service "kube-state-metrics" created


[root@kubernetes-64 monitoring]# kubectl get pods -n monitoring
NAME                                  READY     STATUS    RESTARTS   AGE
kube-state-metrics-578778548f-8gf8d   4/4       Running   0          1m
node-exporter-9pgx5                   2/2       Running   0          2m
node-exporter-dz9kp                   2/2       Running   0          2m
node-exporter-mcd9m                   2/2       Running   0          2m
prometheus-operator-9bf5574-s7k76     1/1       Running   0          2m


```

### prometheus

```shell
vi prometheus-k8s.yaml

---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: RoleBinding
metadata:
  name: prometheus-k8s
  namespace: monitoring
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: prometheus-k8s
subjects:
- kind: ServiceAccount
  name: prometheus-k8s
  namespace: monitoring
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: RoleBinding
metadata:
  name: prometheus-k8s
  namespace: kube-system
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: prometheus-k8s
subjects:
- kind: ServiceAccount
  name: prometheus-k8s
  namespace: monitoring
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: RoleBinding
metadata:
  name: prometheus-k8s
  namespace: default
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: prometheus-k8s
subjects:
- kind: ServiceAccount
  name: prometheus-k8s
  namespace: monitoring
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: prometheus-k8s
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: prometheus-k8s
subjects:
- kind: ServiceAccount
  name: prometheus-k8s
  namespace: monitoring

---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: Role
metadata:
  name: prometheus-k8s
  namespace: monitoring
rules:
- apiGroups: [""]
  resources:
  - nodes
  - services
  - endpoints
  - pods
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources:
  - configmaps
  verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: Role
metadata:
  name: prometheus-k8s
  namespace: kube-system
rules:
- apiGroups: [""]
  resources:
  - services
  - endpoints
  - pods
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: Role
metadata:
  name: prometheus-k8s
  namespace: default
rules:
- apiGroups: [""]
  resources:
  - services
  - endpoints
  - pods
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  name: prometheus-k8s
rules:
- apiGroups: [""]
  resources:
  - nodes/metrics
  verbs: ["get"]
- nonResourceURLs: ["/metrics"]
  verbs: ["get"]
  
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-k8s-rules
  namespace: monitoring
  labels:
    role: prometheus-rulefiles
    prometheus: k8s
data:
  alertmanager.rules.yaml: |+
    groups:
    - name: alertmanager.rules
      rules:
      - alert: AlertmanagerConfigInconsistent
        expr: count_values("config_hash", alertmanager_config_hash) BY (service) / ON(service)
          GROUP_LEFT() label_replace(prometheus_operator_alertmanager_spec_replicas, "service",
          "alertmanager-$1", "alertmanager", "(.*)") != 1
        for: 5m
        labels:
          severity: critical
        annotations:
          description: The configuration of the instances of the Alertmanager cluster
            `{{$labels.service}}` are out of sync.
      - alert: AlertmanagerDownOrMissing
        expr: label_replace(prometheus_operator_alertmanager_spec_replicas, "job", "alertmanager-$1",
          "alertmanager", "(.*)") / ON(job) GROUP_RIGHT() sum(up) BY (job) != 1
        for: 5m
        labels:
          severity: warning
        annotations:
          description: An unexpected number of Alertmanagers are scraped or Alertmanagers
            disappeared from discovery.
      - alert: AlertmanagerFailedReload
        expr: alertmanager_config_last_reload_successful == 0
        for: 10m
        labels:
          severity: warning
        annotations:
          description: Reloading Alertmanager's configuration has failed for {{ $labels.namespace
            }}/{{ $labels.pod}}.
  general.rules.yaml: |+
    groups:
    - name: general.rules
      rules:
      - alert: TargetDown
        expr: 100 * (count(up == 0) BY (job) / count(up) BY (job)) > 10
        for: 10m
        labels:
          severity: warning
        annotations:
          description: '{{ $value }}% of {{ $labels.job }} targets are down.'
          summary: Targets are down
      - alert: DeadMansSwitch
        expr: vector(1)
        labels:
          severity: none
        annotations:
          description: This is a DeadMansSwitch meant to ensure that the entire Alerting
            pipeline is functional.
          summary: Alerting DeadMansSwitch
      - record: fd_utilization
        expr: process_open_fds / process_max_fds
      - alert: FdExhaustionClose
        expr: predict_linear(fd_utilization[1h], 3600 * 4) > 1
        for: 10m
        labels:
          severity: warning
        annotations:
          description: '{{ $labels.job }}: {{ $labels.namespace }}/{{ $labels.pod }} instance
            will exhaust in file/socket descriptors within the next 4 hours'
          summary: file descriptors soon exhausted
      - alert: FdExhaustionClose
        expr: predict_linear(fd_utilization[10m], 3600) > 1
        for: 10m
        labels:
          severity: critical
        annotations:
          description: '{{ $labels.job }}: {{ $labels.namespace }}/{{ $labels.pod }} instance
            will exhaust in file/socket descriptors within the next hour'
          summary: file descriptors soon exhausted
  kube-state-metrics.rules.yaml: |+
    groups:
    - name: kube-state-metrics.rules
      rules:
      - alert: DeploymentGenerationMismatch
        expr: kube_deployment_status_observed_generation != kube_deployment_metadata_generation
        for: 15m
        labels:
          severity: warning
        annotations:
          description: Observed deployment generation does not match expected one for
            deployment {{$labels.namespaces}}/{{$labels.deployment}}
          summary: Deployment is outdated
      - alert: DeploymentReplicasNotUpdated
        expr: ((kube_deployment_status_replicas_updated != kube_deployment_spec_replicas)
          or (kube_deployment_status_replicas_available != kube_deployment_spec_replicas))
          unless (kube_deployment_spec_paused == 1)
        for: 15m
        labels:
          severity: warning
        annotations:
          description: Replicas are not updated and available for deployment {{$labels.namespaces}}/{{$labels.deployment}}
          summary: Deployment replicas are outdated
      - alert: DaemonSetRolloutStuck
        expr: kube_daemonset_status_number_ready / kube_daemonset_status_desired_number_scheduled
          * 100 < 100
        for: 15m
        labels:
          severity: warning
        annotations:
          description: Only {{$value}}% of desired pods scheduled and ready for daemon
            set {{$labels.namespaces}}/{{$labels.daemonset}}
          summary: DaemonSet is missing pods
      - alert: K8SDaemonSetsNotScheduled
        expr: kube_daemonset_status_desired_number_scheduled - kube_daemonset_status_current_number_scheduled
          > 0
        for: 10m
        labels:
          severity: warning
        annotations:
          description: A number of daemonsets are not scheduled.
          summary: Daemonsets are not scheduled correctly
      - alert: DaemonSetsMissScheduled
        expr: kube_daemonset_status_number_misscheduled > 0
        for: 10m
        labels:
          severity: warning
        annotations:
          description: A number of daemonsets are running where they are not supposed
            to run.
          summary: Daemonsets are not scheduled correctly
      - alert: PodFrequentlyRestarting
        expr: increase(kube_pod_container_status_restarts_total[1h]) > 5
        for: 10m
        labels:
          severity: warning
        annotations:
          description: Pod {{$labels.namespaces}}/{{$labels.pod}} is was restarted {{$value}}
            times within the last hour
          summary: Pod is restarting frequently
  kubelet.rules.yaml: |+
    groups:
    - name: kubelet.rules
      rules:
      - alert: K8SNodeNotReady
        expr: kube_node_status_condition{condition="Ready",status="true"} == 0
        for: 1h
        labels:
          severity: warning
        annotations:
          description: The Kubelet on {{ $labels.node }} has not checked in with the API,
            or has set itself to NotReady, for more than an hour
          summary: Node status is NotReady
      - alert: K8SManyNodesNotReady
        expr: count(kube_node_status_condition{condition="Ready",status="true"} == 0)
          > 1 and (count(kube_node_status_condition{condition="Ready",status="true"} ==
          0) / count(kube_node_status_condition{condition="Ready",status="true"})) > 0.2
        for: 1m
        labels:
          severity: critical
        annotations:
          description: '{{ $value }}% of Kubernetes nodes are not ready'
      - alert: K8SKubeletDown
        expr: count(up{job="kubelet"} == 0) / count(up{job="kubelet"}) * 100 > 3
        for: 1h
        labels:
          severity: warning
        annotations:
          description: Prometheus failed to scrape {{ $value }}% of kubelets.
      - alert: K8SKubeletDown
        expr: (absent(up{job="kubelet"} == 1) or count(up{job="kubelet"} == 0) / count(up{job="kubelet"}))
          * 100 > 1
        for: 1h
        labels:
          severity: critical
        annotations:
          description: Prometheus failed to scrape {{ $value }}% of kubelets, or all Kubelets
            have disappeared from service discovery.
          summary: Many Kubelets cannot be scraped
      - alert: K8SKubeletTooManyPods
        expr: kubelet_running_pod_count > 100
        for: 10m
        labels:
          severity: warning
        annotations:
          description: Kubelet {{$labels.instance}} is running {{$value}} pods, close
            to the limit of 110
          summary: Kubelet is close to pod limit
  kubernetes.rules.yaml: |+
    groups:
    - name: kubernetes.rules
      rules:
      - record: pod_name:container_memory_usage_bytes:sum
        expr: sum(container_memory_usage_bytes{container_name!="POD",pod_name!=""}) BY
          (pod_name)
      - record: pod_name:container_spec_cpu_shares:sum
        expr: sum(container_spec_cpu_shares{container_name!="POD",pod_name!=""}) BY (pod_name)
      - record: pod_name:container_cpu_usage:sum
        expr: sum(rate(container_cpu_usage_seconds_total{container_name!="POD",pod_name!=""}[5m]))
          BY (pod_name)
      - record: pod_name:container_fs_usage_bytes:sum
        expr: sum(container_fs_usage_bytes{container_name!="POD",pod_name!=""}) BY (pod_name)
      - record: namespace:container_memory_usage_bytes:sum
        expr: sum(container_memory_usage_bytes{container_name!=""}) BY (namespace)
      - record: namespace:container_spec_cpu_shares:sum
        expr: sum(container_spec_cpu_shares{container_name!=""}) BY (namespace)
      - record: namespace:container_cpu_usage:sum
        expr: sum(rate(container_cpu_usage_seconds_total{container_name!="POD"}[5m]))
          BY (namespace)
      - record: cluster:memory_usage:ratio
        expr: sum(container_memory_usage_bytes{container_name!="POD",pod_name!=""}) BY
          (cluster) / sum(machine_memory_bytes) BY (cluster)
      - record: cluster:container_spec_cpu_shares:ratio
        expr: sum(container_spec_cpu_shares{container_name!="POD",pod_name!=""}) / 1000
          / sum(machine_cpu_cores)
      - record: cluster:container_cpu_usage:ratio
        expr: sum(rate(container_cpu_usage_seconds_total{container_name!="POD",pod_name!=""}[5m]))
          / sum(machine_cpu_cores)
      - record: apiserver_latency_seconds:quantile
        expr: histogram_quantile(0.99, rate(apiserver_request_latencies_bucket[5m])) /
          1e+06
        labels:
          quantile: "0.99"
      - record: apiserver_latency:quantile_seconds
        expr: histogram_quantile(0.9, rate(apiserver_request_latencies_bucket[5m])) /
          1e+06
        labels:
          quantile: "0.9"
      - record: apiserver_latency_seconds:quantile
        expr: histogram_quantile(0.5, rate(apiserver_request_latencies_bucket[5m])) /
          1e+06
        labels:
          quantile: "0.5"
      - alert: APIServerLatencyHigh
        expr: apiserver_latency_seconds:quantile{quantile="0.99",subresource!="log",verb!~"^(?:WATCH|WATCHLIST|PROXY|CONNECT)$"}
          > 1
        for: 10m
        labels:
          severity: warning
        annotations:
          description: the API server has a 99th percentile latency of {{ $value }} seconds
            for {{$labels.verb}} {{$labels.resource}}
      - alert: APIServerLatencyHigh
        expr: apiserver_latency_seconds:quantile{quantile="0.99",subresource!="log",verb!~"^(?:WATCH|WATCHLIST|PROXY|CONNECT)$"}
          > 4
        for: 10m
        labels:
          severity: critical
        annotations:
          description: the API server has a 99th percentile latency of {{ $value }} seconds
            for {{$labels.verb}} {{$labels.resource}}
      - alert: APIServerErrorsHigh
        expr: rate(apiserver_request_count{code=~"^(?:5..)$"}[5m]) / rate(apiserver_request_count[5m])
          * 100 > 2
        for: 10m
        labels:
          severity: warning
        annotations:
          description: API server returns errors for {{ $value }}% of requests
      - alert: APIServerErrorsHigh
        expr: rate(apiserver_request_count{code=~"^(?:5..)$"}[5m]) / rate(apiserver_request_count[5m])
          * 100 > 5
        for: 10m
        labels:
          severity: critical
        annotations:
          description: API server returns errors for {{ $value }}% of requests
      - alert: K8SApiserverDown
        expr: absent(up{job="apiserver"} == 1)
        for: 20m
        labels:
          severity: critical
        annotations:
          description: No API servers are reachable or all have disappeared from service
            discovery
    
      - alert: K8sCertificateExpirationNotice
        labels:
          severity: warning
        annotations:
          description: Kubernetes API Certificate is expiring soon (less than 7 days)
        expr: sum(apiserver_client_certificate_expiration_seconds_bucket{le="604800"}) > 0
    
      - alert: K8sCertificateExpirationNotice
        labels:
          severity: critical
        annotations:
          description: Kubernetes API Certificate is expiring in less than 1 day
        expr: sum(apiserver_client_certificate_expiration_seconds_bucket{le="86400"}) > 0
  node.rules.yaml: |+
    groups:
    - name: node.rules
      rules:
      - record: instance:node_cpu:rate:sum
        expr: sum(rate(node_cpu{mode!="idle",mode!="iowait",mode!~"^(?:guest.*)$"}[3m]))
          BY (instance)
      - record: instance:node_filesystem_usage:sum
        expr: sum((node_filesystem_size{mountpoint="/"} - node_filesystem_free{mountpoint="/"}))
          BY (instance)
      - record: instance:node_network_receive_bytes:rate:sum
        expr: sum(rate(node_network_receive_bytes[3m])) BY (instance)
      - record: instance:node_network_transmit_bytes:rate:sum
        expr: sum(rate(node_network_transmit_bytes[3m])) BY (instance)
      - record: instance:node_cpu:ratio
        expr: sum(rate(node_cpu{mode!="idle"}[5m])) WITHOUT (cpu, mode) / ON(instance)
          GROUP_LEFT() count(sum(node_cpu) BY (instance, cpu)) BY (instance)
      - record: cluster:node_cpu:sum_rate5m
        expr: sum(rate(node_cpu{mode!="idle"}[5m]))
      - record: cluster:node_cpu:ratio
        expr: cluster:node_cpu:rate5m / count(sum(node_cpu) BY (instance, cpu))
      - alert: NodeExporterDown
        expr: absent(up{job="node-exporter"} == 1)
        for: 10m
        labels:
          severity: warning
        annotations:
          description: Prometheus could not scrape a node-exporter for more than 10m,
            or node-exporters have disappeared from discovery
      - alert: NodeDiskRunningFull
        expr: predict_linear(node_filesystem_free[6h], 3600 * 24) < 0
        for: 30m
        labels:
          severity: warning
        annotations:
          description: device {{$labels.device}} on node {{$labels.instance}} is running
            full within the next 24 hours (mounted at {{$labels.mountpoint}})
      - alert: NodeDiskRunningFull
        expr: predict_linear(node_filesystem_free[30m], 3600 * 2) < 0
        for: 10m
        labels:
          severity: critical
        annotations:
          description: device {{$labels.device}} on node {{$labels.instance}} is running
            full within the next 2 hours (mounted at {{$labels.mountpoint}})
  prometheus.rules.yaml: |+
    groups:
    - name: prometheus.rules
      rules:
      - alert: PrometheusConfigReloadFailed
        expr: prometheus_config_last_reload_successful == 0
        for: 10m
        labels:
          severity: warning
        annotations:
          description: Reloading Prometheus' configuration has failed for {{$labels.namespace}}/{{$labels.pod}}
      - alert: PrometheusNotificationQueueRunningFull
        expr: predict_linear(prometheus_notifications_queue_length[5m], 60 * 30) > prometheus_notifications_queue_capacity
        for: 10m
        labels:
          severity: warning
        annotations:
          description: Prometheus' alert notification queue is running full for {{$labels.namespace}}/{{
            $labels.pod}}
      - alert: PrometheusErrorSendingAlerts
        expr: rate(prometheus_notifications_errors_total[5m]) / rate(prometheus_notifications_sent_total[5m])
          > 0.01
        for: 10m
        labels:
          severity: warning
        annotations:
          description: Errors while sending alerts from Prometheus {{$labels.namespace}}/{{
            $labels.pod}} to Alertmanager {{$labels.Alertmanager}}
      - alert: PrometheusErrorSendingAlerts
        expr: rate(prometheus_notifications_errors_total[5m]) / rate(prometheus_notifications_sent_total[5m])
          > 0.03
        for: 10m
        labels:
          severity: critical
        annotations:
          description: Errors while sending alerts from Prometheus {{$labels.namespace}}/{{
            $labels.pod}} to Alertmanager {{$labels.Alertmanager}}
      - alert: PrometheusNotConnectedToAlertmanagers
        expr: prometheus_notifications_alertmanagers_discovered < 1
        for: 10m
        labels:
          severity: warning
        annotations:
          description: Prometheus {{ $labels.namespace }}/{{ $labels.pod}} is not connected
            to any Alertmanagers
      - alert: PrometheusTSDBReloadsFailing
        expr: increase(prometheus_tsdb_reloads_failures_total[2h]) > 0
        for: 12h
        labels:
          severity: warning
        annotations:
          description: '{{$labels.job}} at {{$labels.instance}} had {{$value | humanize}}
            reload failures over the last four hours.'
          summary: Prometheus has issues reloading data blocks from disk
      - alert: PrometheusTSDBCompactionsFailing
        expr: increase(prometheus_tsdb_compactions_failed_total[2h]) > 0
        for: 12h
        labels:
          severity: warning
        annotations:
          description: '{{$labels.job}} at {{$labels.instance}} had {{$value | humanize}}
            compaction failures over the last four hours.'
          summary: Prometheus has issues compacting sample blocks
      - alert: PrometheusTSDBWALCorruptions
        expr: tsdb_wal_corruptions_total > 0
        for: 4h
        labels:
          severity: warning
        annotations:
          description: '{{$labels.job}} at {{$labels.instance}} has a corrupted write-ahead
            log (WAL).'
          summary: Prometheus write-ahead log is corrupted


---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: prometheus-k8s
  namespace: monitoring
  
---
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  name: k8s
  namespace: monitoring
  labels:
    prometheus: k8s
spec:
  replicas: 2
  version: v2.1.0
  serviceAccountName: prometheus-k8s
  serviceMonitorSelector:
    matchExpressions:
    - {key: k8s-app, operator: Exists}
  ruleSelector:
    matchLabels:
      role: prometheus-rulefiles
      prometheus: k8s
  resources:
    requests:
      memory: 1024Mi
  alerting:
    alertmanagers:
    - namespace: monitoring
      name: alertmanager-main
      port: web
      
---
apiVersion: v1
kind: Service
metadata:
  labels:
    prometheus: k8s
  name: prometheus-k8s
  namespace: monitoring
spec:
  ports:
  - name: web
    port: 9090
    protocol: TCP
    targetPort: web
  selector:
    prometheus: k8s
  
---
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
# 导入 yaml 文件

[root@kubernetes-64 monitoring]# kubectl apply -f prometheus-k8s.yaml 
rolebinding "prometheus-k8s" created
rolebinding "prometheus-k8s" created
rolebinding "prometheus-k8s" created
clusterrolebinding "prometheus-k8s" created
role "prometheus-k8s" created
role "prometheus-k8s" created
role "prometheus-k8s" created
clusterrole "prometheus-k8s" created
configmap "prometheus-k8s-rules" created
serviceaccount "prometheus-k8s" created
prometheus "k8s" created
service "prometheus-k8s" created
ingress "prometheus-ingress" created


```


```
# 查看服务

[root@kubernetes-64 monitoring]# kubectl get pods -n monitoring
NAME                                  READY     STATUS    RESTARTS   AGE
grafana-f8db675bc-5qg9j               2/2       Running   0          2h
kube-state-metrics-578778548f-8gf8d   4/4       Running   0          3h
node-exporter-9pgx5                   2/2       Running   0          3h
node-exporter-dz9kp                   2/2       Running   0          3h
node-exporter-mcd9m                   2/2       Running   0          3h
prometheus-k8s-0                      2/2       Running   0          9m
prometheus-k8s-1                      2/2       Running   0          9m
prometheus-operator-9bf5574-s7k76     1/1       Running   0          3h


```



```
# 添加 监控 设置

vi  prometheus-k8s-service-monitor.yaml

---
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: kube-state-metrics
  namespace: monitoring
  labels:
    k8s-app: kube-state-metrics
spec:
  jobLabel: k8s-app
  selector:
    matchLabels:
      k8s-app: kube-state-metrics
  namespaceSelector:
    matchNames:
    - monitoring
  endpoints:
  - port: https-main
    scheme: https
    interval: 30s
    honorLabels: true
    bearerTokenFile: /var/run/secrets/kubernetes.io/serviceaccount/token
    tlsConfig:
      insecureSkipVerify: true
  - port: https-self
    scheme: https
    interval: 30s
    bearerTokenFile: /var/run/secrets/kubernetes.io/serviceaccount/token
    tlsConfig:
      insecureSkipVerify: true

---
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: node-exporter
  namespace: monitoring
  labels:
    k8s-app: node-exporter
spec:
  jobLabel: k8s-app
  selector:
    matchLabels:
      k8s-app: node-exporter
  namespaceSelector:
    matchNames:
    - monitoring
  endpoints:
  - port: https
    scheme: https
    interval: 30s
    bearerTokenFile: /var/run/secrets/kubernetes.io/serviceaccount/token
    tlsConfig:
      insecureSkipVerify: true
      
---
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: prometheus-operator
  namespace: monitoring
  labels:
    k8s-app: prometheus-operator
spec:
  endpoints:
  - port: http
  selector:
    matchLabels:
      k8s-app: prometheus-operator
      
---
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: prometheus
  namespace: monitoring
  labels:
    k8s-app: prometheus
spec:
  selector:
    matchLabels:
      prometheus: k8s
  namespaceSelector:
    matchNames:
    - monitoring
  endpoints:
  - port: web
    interval: 30s
    
    
---
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: alertmanager
  namespace: monitoring
  labels:
    k8s-app: alertmanager
spec:
  selector:
    matchLabels:
      alertmanager: main
  namespaceSelector:
    matchNames:
    - monitoring
  endpoints:
  - port: web
    interval: 30s

---
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: kube-apiserver
  namespace: monitoring
  labels:
    k8s-app: apiserver
spec:
  jobLabel: component
  selector:
    matchLabels:
      component: apiserver
      provider: kubernetes
  namespaceSelector:
    matchNames:
    - default
  endpoints:
  - port: https
    interval: 30s
    scheme: https
    tlsConfig:
      caFile: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
      serverName: kubernetes
    bearerTokenFile: /var/run/secrets/kubernetes.io/serviceaccount/token
    
---
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: kubelet
  namespace: monitoring
  labels:
    k8s-app: kubelet
spec:
  jobLabel: k8s-app
  endpoints:
  - port: https-metrics
    scheme: https
    interval: 30s
    tlsConfig:
      insecureSkipVerify: true
    bearerTokenFile: /var/run/secrets/kubernetes.io/serviceaccount/token
  - port: https-metrics
    scheme: https
    path: /metrics/cadvisor
    interval: 30s
    honorLabels: true
    tlsConfig:
      insecureSkipVerify: true
    bearerTokenFile: /var/run/secrets/kubernetes.io/serviceaccount/token
  selector:
    matchLabels:
      k8s-app: kubelet
  namespaceSelector:
    matchNames:
    - kube-system
    

```

```
# 导入 yaml 文件

[root@kubernetes-64 monitoring]# kubectl apply -f prometheus-k8s-service-monitor.yaml 
servicemonitor "kube-state-metrics" created
servicemonitor "node-exporter" created
servicemonitor "prometheus-operator" created
servicemonitor "prometheus" created
servicemonitor "alertmanager" created
servicemonitor "kube-apiserver" created
servicemonitor "kubelet" created



# 查看服务

[root@kubernetes-64 monitoring]# kubectl get servicemonitor -n monitoring
NAME                      AGE
alertmanager              23s
kube-apiserver            23s
kube-controller-manager   23s
kubelet                   23s
node-exporter             23s
prometheus                23s
prometheus-operator       23s

```

### alertmanager

```
# 配置认证

vi alertmanager-secret.yaml


apiVersion: v1
kind: Secret
metadata:
  name: alertmanager-main
  namespace: monitoring
data:
  alertmanager.yaml: Z2xvYmFsOgogIHJlc29sdmVfdGltZW91dDogNW0Kcm91dGU6CiAgZ3JvdXBfYnk6IFsnam9iJ10KICBncm91cF93YWl0OiAzMHMKICBncm91cF9pbnRlcnZhbDogNW0KICByZXBlYXRfaW50ZXJ2YWw6IDEyaAogIHJlY2VpdmVyOiAnbnVsbCcKICByb3V0ZXM6CiAgLSBtYXRjaDoKICAgICAgYWxlcnRuYW1lOiBEZWFkTWFuc1N3aXRjaAogICAgcmVjZWl2ZXI6ICdudWxsJwpyZWNlaXZlcnM6Ci0gbmFtZTogJ251bGwnCg==

```

```
# 导入 yaml 文件

[root@kubernetes-64 monitoring]# kubectl apply -f alertmanager-secret.yaml 
secret "alertmanager-main" created


# 查看

[root@kubernetes-64 monitoring]# kubectl get secret -n monitoring
NAME                              TYPE                                  DATA      AGE
alertmanager-main                 Opaque                                1         28s
default-token-t6cdn               kubernetes.io/service-account-token   3         4h
grafana-credentials               Opaque                                2         3h
kube-state-metrics-token-ctnl6    kubernetes.io/service-account-token   3         3h
node-exporter-token-nflwn         kubernetes.io/service-account-token   3         3h
prometheus-k8s                    Opaque                                2         21m
prometheus-k8s-token-2dt66        kubernetes.io/service-account-token   3         21m
prometheus-operator-token-k5m47   kubernetes.io/service-account-token   3         3h

```


```
vi alertmanager.yaml


---
apiVersion: monitoring.coreos.com/v1
kind: Alertmanager
metadata:
  name: main
  namespace: monitoring
  labels:
    alertmanager: main
spec:
  replicas: 3
  version: v0.13.0
  
---
apiVersion: v1
kind: Service
metadata:
  labels:
    alertmanager: main
  name: alertmanager-main
  namespace: monitoring
spec:
  ports:
  - name: web
    port: 9093
    protocol: TCP
    targetPort: web
  selector:
    alertmanager: main

---
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


```
# 导入 yaml 文件

[root@kubernetes-64 monitoring]# kubectl apply -f alertmanager.yaml 
alertmanager "main" created
service "alertmanager-main" created
ingress "alertmanager-ingress" created



# 查看服务

[root@kubernetes-64 monitoring]# kubectl get pods -n monitoring
NAME                                  READY     STATUS    RESTARTS   AGE
alertmanager-main-0                   2/2       Running   0          2m
alertmanager-main-1                   2/2       Running   0          1m
alertmanager-main-2                   2/2       Running   0          11s
grafana-f8db675bc-5qg9j               2/2       Running   0          3h
kube-state-metrics-578778548f-8gf8d   4/4       Running   0          3h
node-exporter-9pgx5                   2/2       Running   0          3h
node-exporter-dz9kp                   2/2       Running   0          3h
node-exporter-mcd9m                   2/2       Running   0          3h
prometheus-k8s-0                      2/2       Running   0          27m
prometheus-k8s-1                      2/2       Running   0          27m
prometheus-operator-9bf5574-s7k76     1/1       Running   0          3h


```






### grafana


```
# 创建 认证 (加密方式为 base, YWRtaW4= 等于 admin)

vi grafana-credentials.yaml

apiVersion: v1
kind: Secret
metadata:
  name: grafana-credentials
  namespace: monitoring
data:
  user: YWRtaW4=
  password: YWRtaW4=

```


```
# 导入 yaml 文件

[root@kubernetes-64 monitoring]# kubectl apply -f grafana-credentials.yaml 
secret "grafana-credentials" created

```



```
# 下载 dashboard yaml 文件

wget https://raw.githubusercontent.com/jicki/kuberneres/master/grafana-dashboards.yaml



# 导入 yaml 文件

[root@kubernetes-64 monitoring]# kubectl apply -f grafana-dashboards.yaml 
configmap "grafana-dashboards-0" created


```




```
# vi grafana.yaml

---
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: grafana
  namespace: monitoring
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: grafana
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 65534
      containers:
      - name: grafana
        image: jicki/monitoring-grafana:4.6.3-non-root
        env:
        - name: GF_AUTH_BASIC_ENABLED
          value: "true"
        - name: GF_AUTH_ANONYMOUS_ENABLED
          value: "true"
        - name: GF_SECURITY_ADMIN_USER
          valueFrom:
            secretKeyRef:
              name: grafana-credentials
              key: user
        - name: GF_SECURITY_ADMIN_PASSWORD
          valueFrom:
            secretKeyRef:
              name: grafana-credentials
              key: password
        volumeMounts:
        - name: grafana-storage
          mountPath: /data
        ports:
        - name: web
          containerPort: 3000
        resources:
          requests:
            memory: 100Mi
            cpu: 100m
          limits:
            memory: 200Mi
            cpu: 200m
      - name: grafana-watcher
        image: jicki/grafana-watcher:v0.0.8
        args:
          - '--watch-dir=/var/grafana-dashboards-0'
          - '--grafana-url=http://localhost:3000'
        env:
        - name: GRAFANA_USER
          valueFrom:
            secretKeyRef:
              name: grafana-credentials
              key: user
        - name: GRAFANA_PASSWORD
          valueFrom:
            secretKeyRef:
              name: grafana-credentials
              key: password
        resources:
          requests:
            memory: "16Mi"
            cpu: "50m"
          limits:
            memory: "32Mi"
            cpu: "100m"
        volumeMounts:
        - name: grafana-dashboards-0
          mountPath: /var/grafana-dashboards-0
      volumes:
      - name: grafana-storage
        emptyDir: {}
      - name: grafana-dashboards-0
        configMap:
          name: grafana-dashboards-0
          
---
apiVersion: v1
kind: Service
metadata:
  name: grafana
  namespace: monitoring
  labels:
    app: grafana
spec:
  ports:
  - port: 3000
    protocol: TCP
    targetPort: web
  selector:
    app: grafana
    
---
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
# 导入 yaml 文件

[root@kubernetes-64 monitoring]# kubectl apply -f grafana.yaml 
deployment "grafana" created
service "grafana" created
ingress "grafana-ingress" created






# 查看

[root@kubernetes-64 monitoring]# kubectl get pods -n monitoring
NAME                                  READY     STATUS    RESTARTS   AGE
grafana-f8db675bc-5qg9j               2/2       Running   0          2h
kube-state-metrics-578778548f-8gf8d   4/4       Running   0          2h
node-exporter-9pgx5                   2/2       Running   0          2h
node-exporter-dz9kp                   2/2       Running   0          2h
node-exporter-mcd9m                   2/2       Running   0          2h
prometheus-operator-9bf5574-s7k76     1/1       Running   0          2h


```

## 页面展示

![1][1]
![2][2]
![3][3]
![4][4]
![5][5]
{% endraw %}

  [1]: https://jicki.cn/img/posts/grafana/grafana1.png
  [2]: https://jicki.cn/img/posts/grafana/grafana2.png
  [3]: https://jicki.cn/img/posts/grafana/grafana3.png
  [4]: https://jicki.cn/img/posts/grafana/grafana4.png
  [5]: https://jicki.cn/img/posts/grafana/grafana5.png

