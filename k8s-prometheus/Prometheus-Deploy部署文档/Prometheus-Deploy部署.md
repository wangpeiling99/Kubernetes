# k8s实战之部署Prometheus+Grafana可视化监控告警平台

## 1 Prometheus架构

### Prometheus 是什么

Prometheus（普罗米修斯）是一个最初在SoundCloud上构建的监控系统。自2012年成为社区开源项目，拥有非常活跃的开发人员和用户社区。为强调开源及独立维护，Prometheus于2016年加入云原生云计算基金会（CNCF），成为继Kubernetes之后的第二个托管项目。
官网地址:
https://prometheus.io
https://github.com/prometheus

### Prometheus 组成及架构

![微信图片_20230317204914](\images\微信图片_20230317204914.png)

- Prometheus Server：收集指标和存储时间序列数据，并提供查询接口
- ClientLibrary：客户端库
- Push Gateway：短期存储指标数据。主要用于临时性的任务
- Exporters：采集已有的第三方服务监控指标并暴露metrics
- Alertmanager：告警
- Web UI：简单的Web控制台



### 数据模型

Prometheus将所有数据存储为时间序列；具有相同度量名称以及标签属于同一个指标。
每个时间序列都由度量标准名称和一组键值对（也成为标签）唯一标识。
时间序列格式：

{=, …}
示例：api_http_requests_total{method=”POST”, handler=”/messages”}

### 作业和实例

实例：可以抓取的目标称为实例（Instances）
作业：具有相同目标的实例集合称为作业（Job）
scrape_configs:
-job_name: ‘prometheus’
static_configs:
-targets: [‘localhost:9090’]
-job_name: ‘node’
static_configs:
-targets: [‘192.168.1.10:9090’]

## 2 K8S监控指标及实现思路

### k8S监控指标

**Kubernetes本身监控**

- Node资源利用率
- Node数量
- Pods数量（Node）
- 资源对象状态

**Pod监控**

- Pod数量（项目）
- 容器资源利用率
- 应用程序

### Prometheus监控K8S实现的架构

![微信图片_20230317205422](\images\微信图片_20230317205422.png)

| 监控指标    |         具体实现         |          举例          |
| :---------- | :----------------------: | :--------------------: |
| Pod性能     |         cAdvisor         |        容器CPU         |
| Node性能    | node-exporter	节点CPU |       内存利用率       |
| K8S资源对象 |    kube-state-metrics    | Pod/Deployment/Service |

服务发现：
https://prometheus.io/docs/prometheus/latest/configuration/configuration/#kubernetes_sd_config

## 3 在K8S平台部署Prometheus

### 3.1 项目地址

可以直接选择git clone项目代码，也可以后面自己编写，后面的操作也会有实现代码

##### 选择k8s-prometheus目录使用，k8s-prom-iKubernetes目录没用过！

```shell
[root@master01 ~]# git clone https://github.com/wangpeiling99/Kubernetes.git

选择k8s-prometheus目录使用，k8s-prom-iKubernetes目录没用过！

[root@master01 k8s-prometheus]# ls
alertmanager  k8s-prometheus-adapter  metrics-Server  node_exporter  prometheus                      storageClass
grafana       kube-state-metrics      namespace.yaml  OWNERS         static-pv-prometheus-stafulSet

storageClass是PV等的配置
static-pv-prometheus-stafulSet是静态PV的Prometheus部署
metrics-Server是k8s本地监控（kubectl top）


```

### 3.2 使用RBAC进行授权

RBAC（Role-Based Access Control，基于角色的访问控制）：负责完成授权（Authorization）工作。
编写授权yaml

```shell
[root@master01 ~]# cd k8s-prometheus/
[root@master01 k8s-prometheus]# vim prometheus/prometheus-rbac.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: prometheus
  namespace: kube-system
  labels:
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
---
#apiVersion: rbac.authorization.k8s.io/v1beta1
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: prometheus
  labels:
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
rules:
  - apiGroups:
      - ""
    resources:
      - nodes
      - nodes/metrics
      - services
      - endpoints
      - pods
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - ""
    resources:
      - configmaps
    verbs:
      - get
  - nonResourceURLs:
      - "/metrics"
    verbs:
      - get
---
#apiVersion: rbac.authorization.k8s.io/v1beta1
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: prometheus
  labels:
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: prometheus
subjects:
- kind: ServiceAccount
  name: prometheus
  namespace: kube-system
```

创建

```
[root@k8s-master prometheus-k8s]# kubectl apply -f prometheus/prometheus-rbac.yaml
```

### 3.3配置管理

使用Configmap保存不需要加密配置信息
**其中需要把nodes中ip地址根据自己的地址进行修改，其他不需要改动**

```shell
[root@master01 k8s-prometheus]# vim prometheus/prometheus-configmap.yaml
# Prometheus configuration format https://prometheus.io/docs/prometheus/latest/configuration/configuration/
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
  namespace: kube-system
  labels:
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: EnsureExists
data:
  prometheus.yml: |
    rule_files:
    - /etc/config/rules/*.rules

    scrape_configs:
    - job_name: prometheus
      static_configs:
      - targets:
        - localhost:9090

    - job_name: kubernetes-nodes
      scrape_interval: 30s
      static_configs:
      - targets:
        - 192.168.10.110:9100
        - 192.168.10.120:9100

    - job_name: kubernetes-apiservers
      kubernetes_sd_configs:
      - role: endpoints
      relabel_configs:
      - action: keep
        regex: default;kubernetes;https
        source_labels:
        - __meta_kubernetes_namespace
        - __meta_kubernetes_service_name
        - __meta_kubernetes_endpoint_port_name
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        insecure_skip_verify: true
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token

    - job_name: kubernetes-nodes-kubelet
      kubernetes_sd_configs:
      - role: node
      relabel_configs:
      - action: labelmap
        regex: __meta_kubernetes_node_label_(.+)
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        insecure_skip_verify: true
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token

    - job_name: kubernetes-nodes-cadvisor
      kubernetes_sd_configs:
      - role: node
      relabel_configs:
      - action: labelmap
        regex: __meta_kubernetes_node_label_(.+)
      - target_label: __metrics_path__
        replacement: /metrics/cadvisor
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        insecure_skip_verify: true
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token

    - job_name: kubernetes-service-endpoints
      kubernetes_sd_configs:
      - role: endpoints
      relabel_configs:
      - action: keep
        regex: true
        source_labels:
        - __meta_kubernetes_service_annotation_prometheus_io_scrape
      - action: replace
        regex: (https?)
        source_labels:
        - __meta_kubernetes_service_annotation_prometheus_io_scheme
        target_label: __scheme__
      - action: replace
        regex: (.+)
        source_labels:
        - __meta_kubernetes_service_annotation_prometheus_io_path
        target_label: __metrics_path__
      - action: replace
        regex: ([^:]+)(?::\d+)?;(\d+)
        replacement: $1:$2
        source_labels:
        - __address__
        - __meta_kubernetes_service_annotation_prometheus_io_port
        target_label: __address__
      - action: labelmap
        regex: __meta_kubernetes_service_label_(.+)
      - action: replace
        source_labels:
        - __meta_kubernetes_namespace
        target_label: kubernetes_namespace
      - action: replace
        source_labels:
        - __meta_kubernetes_service_name
        target_label: kubernetes_name

    - job_name: kubernetes-services
      kubernetes_sd_configs:
      - role: service
      metrics_path: /probe
      params:
        module:
        - http_2xx
      relabel_configs:
      - action: keep
        regex: true
        source_labels:
        - __meta_kubernetes_service_annotation_prometheus_io_probe
      - source_labels:
        - __address__
        target_label: __param_target
      - replacement: blackbox
        target_label: __address__
      - source_labels:
        - __param_target
        target_label: instance
      - action: labelmap
        regex: __meta_kubernetes_service_label_(.+)
      - source_labels:
        - __meta_kubernetes_namespace
        target_label: kubernetes_namespace
      - source_labels:
        - __meta_kubernetes_service_name
        target_label: kubernetes_name

    - job_name: kubernetes-pods
      kubernetes_sd_configs:
      - role: pod
      relabel_configs:
      - action: keep
        regex: true
        source_labels:
        - __meta_kubernetes_pod_annotation_prometheus_io_scrape
      - action: replace
        regex: (.+)
        source_labels:
        - __meta_kubernetes_pod_annotation_prometheus_io_path
        target_label: __metrics_path__
      - action: replace
        regex: ([^:]+)(?::\d+)?;(\d+)
        replacement: $1:$2
        source_labels:
        - __address__
        - __meta_kubernetes_pod_annotation_prometheus_io_port
        target_label: __address__
      - action: labelmap
        regex: __meta_kubernetes_pod_label_(.+)
      - action: replace
        source_labels:
        - __meta_kubernetes_namespace
        target_label: kubernetes_namespace
      - action: replace
        source_labels:
        - __meta_kubernetes_pod_name
        target_label: kubernetes_pod_name
    alerting:
      alertmanagers:
      - static_configs:
          - targets: ["alertmanager:80"]
                                                                                
```

创建

```
[root@k8s-master prometheus-k8s]# kubectl apply -f prometheus/prometheus-configmap.yaml
```

### 3.4 有状态部署prometheus

这里使用storageclass进行动态供给，给prometheus的数据进行持久化

```shell
[root@master01 k8s-prometheus]# vim prometheus/prometheus-statefulset.yaml 
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: prometheus
  namespace: kube-system
  labels:
    k8s-app: prometheus
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
    version: v2.2.1
spec:
  serviceName: "prometheus"
  replicas: 1
  podManagementPolicy: "Parallel"
  updateStrategy:
   type: "RollingUpdate"
  selector:
    matchLabels:
      k8s-app: prometheus
  template:
    metadata:
      labels:
        k8s-app: prometheus
      annotations:
        scheduler.alpha.kubernetes.io/critical-pod: ''
    spec:
      priorityClassName: system-cluster-critical
      serviceAccountName: prometheus
      initContainers:
      - name: "init-chown-data"
        image: "busybox:latest"
        imagePullPolicy: "IfNotPresent"
        command: ["chown", "-R", "65534:65534", "/data"]
        volumeMounts:
        - name: prometheus-data
          mountPath: /data
          subPath: ""
      containers:
        - name: prometheus-server-configmap-reload
          image: "jimmidyson/configmap-reload:v0.1"
          imagePullPolicy: "IfNotPresent"
          args:
            - --volume-dir=/etc/config
            - --webhook-url=http://localhost:9090/-/reload
          volumeMounts:
            - name: config-volume
              mountPath: /etc/config
              readOnly: true
          resources:
            limits:
              cpu: 10m
              memory: 10Mi
            requests:
              cpu: 10m
              memory: 10Mi

        - name: prometheus-server
          image: "prom/prometheus:v2.2.1"
          imagePullPolicy: "IfNotPresent"
          args:
            - --config.file=/etc/config/prometheus.yml
            - --storage.tsdb.path=/data
            - --web.console.libraries=/etc/prometheus/console_libraries
            - --web.console.templates=/etc/prometheus/consoles
            - --web.enable-lifecycle
          ports:
            - containerPort: 9090
          readinessProbe:
            httpGet:
              path: /-/ready
              port: 9090
            initialDelaySeconds: 30
            timeoutSeconds: 30
          livenessProbe:
            httpGet:
              path: /-/healthy
              port: 9090
            initialDelaySeconds: 30
            timeoutSeconds: 30
          # based on 10 running nodes with 30 pods each
          resources:
            limits:
              cpu: 200m
              memory: 1000Mi
            requests:
              cpu: 200m
              memory: 1000Mi

          volumeMounts:
            - name: config-volume
              mountPath: /etc/config
            - name: prometheus-data
              mountPath: /data
              subPath: ""
            - name: prometheus-rules
              mountPath: /etc/config/rules

      terminationGracePeriodSeconds: 300
      volumes:
        - name: config-volume
          configMap:
            name: prometheus-config
        - name: prometheus-rules
          configMap:
            name: prometheus-rules

  volumeClaimTemplates:
  - metadata:
      name: prometheus-data
    spec:
      storageClassName: nfs-storage     #存储类根据自己的存储类名字修改
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: "2Gi"

```

创建

```
[root@k8s-master prometheus-k8s]# kubectl apply -f prometheus/prometheus-statefulset.yaml
```

检查状态

```
[root@k8s-master prometheus-k8s]# kubectl get pod -n kube-system
NAME                                    READY   STATUS    RESTARTS   AGE
coredns-5bd5f9dbd9-wv45t                1/1     Running   1          8d
kubernetes-dashboard-7d77666777-d5ng4   1/1     Running   5          14d
prometheus-0                            2/2     Running   6          14d
```

可以看到一个prometheus-0的pod，这就刚才使用statefulset控制器进行的有状态部署，两个容器的状态为Runing则是正常，如果不为Runing可以使用kubectl describe pod  prometheus-0 -n kube-system查看报错详情

### 3.5 创建service暴露访问端口

此处使用nodePort固定一个访问端口，不适用随机端口，便于访问

```shell
[root@master01 k8s-prometheus]# vim prometheus/prometheus-service.yaml
kind: Service
apiVersion: v1
metadata:
  name: prometheus
  namespace: kube-system
  labels:
    kubernetes.io/name: "Prometheus"
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
spec:
  type: NodePort
  ports:
    - name: http
      port: 9090
      protocol: TCP
      targetPort: 9090
      nodePort: 30090     #固定的对外访问的端口
  selector:
    k8s-app: prometheus

```

### 3.6 web访问

使用任意一个NodeIP加端口进行访问，访问地址：http://NodeIP:Port ,此例就是：http://192.168.10.1120:30090
访问成功的界面如图所示：

![微信图片_20230317213024](\images\微信图片_20230317213024.png)

## 4 在K8S平台部署Grafana

通过上面的web访问，可以看出prometheus自带的UI界面是没有多少功能的，可视化展示的功能不完善，不能满足日常的监控所需，因此常常我们需要再结合Prometheus+Grafana的方式来进行可视化的数据展示
官网地址：
https://grafana.com/grafana/download
刚才下载的项目中已经写好了Grafana的yaml，根据自己的环境进行修改

##### 注意：grafana在部署过程中需要给PVC的挂载目录下的grafana目录授权472

```shell
[root@master01 k8s-prometheus]# vim grafana/grafana.yaml 
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: grafana
  namespace: kube-system
spec:
  serviceName: "grafana"
  replicas: 1
  selector:
    matchLabels:
      app: grafana
  template:
    metadata:
      labels:
        app: grafana
    spec:
      containers:
      - name: grafana
        image: grafana/grafana:9.1.0
        ports:
          - containerPort: 3000
            name: http-grafana
            protocol: TCP
        resources:
          limits:
            cpu: 100m
            memory: 256Mi
          requests:
            cpu: 100m
            memory: 256Mi
        volumeMounts:
          - name: grafana-data
            mountPath: /var/lib/grafana
            subPath: grafana
      securityContext:
        fsGroup: 472
        runAsUser: 472
#  volumeClaimTemplates:
#  - metadata:
#      name: grafana-data
#    spec:
#      storageClassName: nfs-storage #和prometheus使用同一个存储类
#      accessModes:
#        - ReadWriteOnce
#      resources:
#        requests:
#          storage: "1Gi"
      volumes:
        - name: grafana-data
          persistentVolumeClaim:
            claimName: grafana-data
---

apiVersion: v1
kind: Service
metadata:
  name: grafana
  namespace: kube-system
spec:
  type: NodePort
  ports:
  - port : 80
    targetPort: 3000
    nodePort: 30091
  selector:
    app: grafana

```

```shell
[root@master01 k8s-prometheus]# vim grafana/grafana.yaml 
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: grafana-data
  namespace: kube-system
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  selector:
    matchLabels:
      pv: grafa
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv0002
  namespace: kube-system
  labels:
    pv: grafa
spec:
  capacity:
    storage: 2Gi
  accessModes:
    - ReadWriteOnce
  nfs:
    path: /data/nfs/
    server: 192.168.10.110

```



### 4.2 Grafana的web访问

使用任意一个NodeIP加端口进行访问，访问地址：http://NodeIP:Port ,此例就是：http://192.168.10.120:30091
成功访问界面如下，会需要进行账号密码登陆，默认账号密码都为admin，登陆之后会让修改密码

![微信图片_20230317214257](\images\微信图片_20230317214257.png)

登陆之后的界面如下

![微信图片_20230317214339](\images\微信图片_20230317214339.png)

第一步需要进行数据源添加，点击create your first data source数据库图标，根据下图所示进行添加即可

![微信图片_20230317214402](\images\微信图片_20230317214402.jpg)

第二步，添加完了之后点击底部的绿色的Save&Test，会成功提示Data sourse is working，则表示数据源添加成功

### 4.3 监控K8S集群中Pod、Node、资源对象数据的方法

##### 1）Pod

kubelet的节点使用cAdvisor提供的metrics接口获取该节点所有Pod和容器相关的性能指标数据，安装kubelet默认就开启了
暴露接口地址：
https://NodeIP:10255/metrics/cadvisor
https://NodeIP:10250/metrics/cadvisor

(这里声明1.18版本之前使用heapster监控数据，1.18.版本之后使用metrics)

##### 2）Node

需要使用node_exporter收集器采集节点资源利用率。
https://github.com/prometheus/node_exporter
使用文档：https://prometheus.io/docs/guides/node-exporter/

##### 部署node_exporter

```shell
#node_exporter的daemonset
[root@master01 k8s-prometheus]# vim node_exporter/node-exporter-daemonSet.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: prometheus-node-exporter
  namespace: kube-system
  labels:
    app: prometheus
    component: node-exporter
spec:
  selector:
    matchLabels:
      app: prometheus
      component: node-exporter
  template:
    metadata:
      name: prometheus-node-exporter
      labels:
        app: prometheus
        component: node-exporter
    spec:
      tolerations:
      - effect: NoSchedule
       # key: node-role.kubernetes.io/master
        key: node-role.kubernetes.io/control-plane
      containers:
      - image: prom/node-exporter:v1.5.0
        name: prometheus-node-exporter
        ports:
        - name: prom-node-exp
          containerPort: 9100
          hostPort: 9100
      hostNetwork: true
      hostPID: true

```

```shell
#node_exporter的svc
[root@master01 k8s-prometheus]# vim node_exporter/node-exporter-svc.yaml 
apiVersion: v1
kind: Service
metadata:
  annotations:
    prometheus.io/scrape: 'true'
  name: prometheus-node-exporter
  namespace: kube-system
  labels:
    app: prometheus
    component: node-exporter
spec:
  clusterIP: None
  ports:
    - name: prometheus-node-exporter
      port: 9100
      protocol: TCP
  selector:
    app: prometheus
    component: node-exporter
  type: ClusterIP
```



#### 资源对象

kube-state-metrics采集了k8s中各种资源对象的状态信息，只需要在master节点部署就行
https://github.com/kubernetes/kube-state-metrics

##### 1.创建rbac的yaml对metrics进行授权

```shell
[root@master01 k8s-prometheus]# vim kube-state-metrics/kube-state-metrics-rbac.yaml 
apiVersion: v1
kind: ServiceAccount
metadata:
  name: kube-state-metrics
  namespace: kube-system
  labels:
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
---
#apiVersion: rbac.authorization.k8s.io/v1beta1
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: kube-state-metrics
  labels:
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
rules:
- apiGroups: [""]
  resources:
  - configmaps
  - secrets
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
  - poddisruptionbudgets
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
---
#apiVersion: rbac.authorization.k8s.io/v1beta1
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: kube-state-metrics-resizer
  namespace: kube-system
  labels:
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
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
#apiVersion: rbac.authorization.k8s.io/v1beta1
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: kube-state-metrics
  labels:
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: kube-state-metrics
subjects:
- kind: ServiceAccount
  name: kube-state-metrics
  namespace: kube-system
---
#apiVersion: rbac.authorization.k8s.io/v1beta1
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: kube-state-metrics
  namespace: kube-system
  labels:
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: kube-state-metrics-resizer
subjects:
- kind: ServiceAccount
  name: kube-state-metrics
  namespace: kube-system




[root@k8s-master prometheus-k8s]# kubectl apply -f kube-state-metrics-rbac.yaml
```

#####   2.编写Service的yaml对metrics进行端口暴露

```shell
[root@master01 k8s-prometheus]# vim kube-state-metrics/kube-state-metrics-service.yaml 
apiVersion: v1
kind: Service
metadata:
  name: kube-state-metrics
  namespace: kube-system
  labels:
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
    kubernetes.io/name: "kube-state-metrics"
  annotations:
    prometheus.io/scrape: 'true'
spec:
  ports:
  - name: http-metrics
    port: 8080
    targetPort: http-metrics
    protocol: TCP
  - name: telemetry
    port: 8081
    targetPort: telemetry
    protocol: TCP
  selector:
    k8s-app: kube-state-metrics

```

#####  3.检查pod和svc的状态，可以看到正常运行了pod/kube-state-metrics-7c76bdbf68-kqqgd 和对外暴露了8080和8081端口

```shell
[root@k8s-master prometheus-k8s]# kubectl get pod,svc -n kube-system
```



## 5 使用Grafana可视化展示Prometheus监控数据

通常在使用Prometheus采集数据的时候我们需要监控K8S集群中Pod、Node、资源对象，因此我们需要安装对应的插件和资源采集器来提供api进行数据获取，在4.3中我们已经配置好，我们也可以使用Prometheus的UI界面中的Staus菜单下的Target中的各个采集器的状态情况，如图所示：

![微信图片_20230317220146](\images\微信图片_20230317220146.jpg)

只有当我们各个Target的状态都是UP状态时，我们可以使用自带的的界面去获取到某一监控项的相关的数据，如图所示：

![微信图片_20230317220251](\images\微信图片_20230317220251.jpg)

从上面的图中可以看出Prometheus的界面可视化展示的功能较单一，不能满足需求，因此我们需要结合Grafana来进行可视化展示Prometheus监控数据，在上一章节，已经成功部署了Granfana，因此需要在使用的时候添加dashboard和Panel来设计展示相关的监控项，但是实际上在Granfana社区里面有很多成熟的模板，我们可以直接使用，然后根据自己的环境修改Panel中的查询语句来获取数据
https://grafana.com/grafana/dashboards

![微信图片_20230317220409](\images\微信图片_20230317220409.jpg)

推荐模板：

- 集群资源监控的模板号：3119，如图所示进行添加

![微信图片_20230317220557](\images\微信图片_20230317220557.jpg)

![微信图片_20230317220700](\images\微信图片_20230317220700.png)

![微信图片_20230317220728](\images\微信图片_20230317220728.jpg)

当模板添加之后如果某一个Panel不显示数据，可以点击Panel上的编辑，查询PromQL语句，然后去Prometheus自己的界面上进行调试PromQL语句是否可以获取到值，最后调整之后的监控界面如图所示

![微信图片_20230317220838](\images\微信图片_20230317220838.jpg)

![微信图片_20230317220838](\images\微信图片_20230317220838.jpg)

资源状态监控：6417
同理，添加资源状态的监控模板，然后经过调整之后的监控界面如图所示，可以获取到k8s中各种资源状态的监控展示

![微信图片_20230317220919](\images\微信图片_20230317220919.jpg)



Node监控：9276
同理，添加资源状态的监控模板，然后经过调整之后的监控界面如图所示，可以获取到各个node上的基本情况

![微信图片_20230317220925](\images\微信图片_20230317220925.jpg)





## 6 在K8S中部署Alertmanager

### 6.1 部署Alertmanager的实现步骤

![微信图片_20230317221100](\images\微信图片_20230317221100.png)

### 6.2 部署告警

我们以Email来进行实现告警信息的发送

1. ##### 首先需要准备一个发件邮箱，开启stmp发送功能

2. ##### 使用configmap存储告警规则，编写报警规则的yaml文件，可根据自己的实际情况进行修改和添加报警的规则，prometheus比zabbix就麻烦在这里，所有的告警规则需要自己去定义

```shell
[root@master01 k8s-prometheus]# vim prometheus/prometheus-rules.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-rules
  namespace: kube-system
data:
  general.rules: |
    groups:
    - name: general.rules
      rules:
      - alert: InstanceDown
        expr: up == 0
        for: 1m
        labels:
          severity: error
        annotations:
          summary: "Instance {{ $labels.instance }} 停止工作"
          description: "{{ $labels.instance }} job {{ $labels.job }} 已经停止5分钟以上."
  node.rules: |
    groups:
    - name: node.rules
      rules:
      - alert: NodeFilesystemUsage
        expr: 100 - (node_filesystem_free_bytes{fstype=~"ext4|xfs"} / node_filesystem_size_bytes{fstype=~"ext4|xfs"} * 100) > 80
        for: 1m
        labels:
          severity: warning
        annotations:
          summary: "Instance {{ $labels.instance }} : {{ $labels.mountpoint }} 分区使用率过高"
          description: "{{ $labels.instance }}: {{ $labels.mountpoint }} 分区使用大于80% (当前值: {{ $value }})"

      - alert: NodeMemoryUsage
        expr: 100 - (node_memory_MemFree_bytes+node_memory_Cached_bytes+node_memory_Buffers_bytes) / node_memory_MemTotal_bytes * 100 > 80
        for: 1m
        labels:
          severity: warning
        annotations:
          summary: "Instance {{ $labels.instance }} 内存使用率过高"
          description: "{{ $labels.instance }}内存使用大于80% (当前值: {{ $value }})"

      - alert: NodeCPUUsage
        expr: 100 - (avg(irate(node_cpu_seconds_total{mode="idle"}[5m])) by (instance) * 100) > 60
        for: 1m
        labels:
          severity: warning
        annotations:
          summary: "Instance {{ $labels.instance }} CPU使用率过高"
          description: "{{ $labels.instance }}CPU使用大于60% (当前值: {{ $value }})"




[root@k8s-master prometheus-k8s]# kubectl apply -f prometheus-rules.yaml
```

#####  3.编写告警configmap的yaml文件部署，增加alertmanager告警配置，进行配置邮箱发送地址

```shell
[root@master01 k8s-prometheus]# vim alertmanager/alertmanager-configmap.yaml 

apiVersion: v1
kind: ConfigMap
metadata:
  name: alertmanager-config
  namespace: kube-system
  labels:
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: EnsureExists
data:
  alertmanager.yml: |
    global:
      resolve_timeout: 5m
      smtp_smarthost: 'smtp.qq.com:465'   #邮件服务器地址
      smtp_from: '496774695@qq.com'      #发送邮件的优先（自定义）
      smtp_auth_username: '496774695@qq.com'  #自己的邮箱
      smtp_auth_password: 'lrvjqumucalfbgjd'  #登录用的密码（开启服务生成）
      smtp_require_tls: false           #不启用证书验证
    receivers:
    - name: default-receiver
      email_configs:
      - to: '15047811859@163.com'
        send_resolved: true      #服务恢复后返回消息

    route:
      group_interval: 1m
      group_wait: 10s
      receiver: default-receiver
      repeat_interval: 1m
      
      
      
      
[root@k8s-master prometheus-k8s]# kubectl apply -f alertmanager-configmap.yaml
```

#####  

##### 4.创建PVC进行数据持久化，我这个yaml文件使用的跟Prometheus安装时用的存储类来进行自动供给，需要根据自己的实际情况修改

```shell
[root@master01 k8s-prometheus]# vim alertmanager/alertmanager-pvc.yaml 
#apiVersion: v1
#kind: PersistentVolumeClaim
#metadata:
#  name: alertmanager
#  namespace: kube-system
#  labels:
#    kubernetes.io/cluster-service: "true"
#    addonmanager.kubernetes.io/mode: EnsureExists
#spec:
#  storageClassName: managed-nfs-storage 
#  accessModes:
#    - ReadWriteOnce
#  resources:
#    requests:
#      storage: "2Gi"
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: alertmanager
  namespace: kube-system
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  selector:
    matchLabels:
      pv: alert
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv0003
  namespace: kube-system
  labels:
    pv: alert
spec:
  capacity:
    storage: 2Gi
  accessModes:
    - ReadWriteOnce
  nfs:
    path: /data/nfs/
    server: 192.168.10.110

```

#####  5.编写deployment的yaml来部署alertmanager的pod

```shell
[root@master01 k8s-prometheus]# vim alertmanager/alertmanager-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: alertmanager
  namespace: kube-system
  labels:
    k8s-app: alertmanager
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
    version: v0.14.0
spec:
  replicas: 1
  selector:
    matchLabels:
      k8s-app: alertmanager
      version: v0.14.0
  template:
    metadata:
      labels:
        k8s-app: alertmanager
        version: v0.14.0
      annotations:
        scheduler.alpha.kubernetes.io/critical-pod: ''
    spec:
      priorityClassName: system-cluster-critical
      containers:
        - name: prometheus-alertmanager
          image: "prom/alertmanager:v0.14.0"
          imagePullPolicy: "IfNotPresent"
          args:
            - --config.file=/etc/config/alertmanager.yml
            - --storage.path=/data
            - --web.external-url=/
          ports:
            - containerPort: 9093
          readinessProbe:
            httpGet:
              path: /#/status
              port: 9093
            initialDelaySeconds: 30
            timeoutSeconds: 30
          volumeMounts:
            - name: config-volume
              mountPath: /etc/config
            - name: storage-volume
              mountPath: "/data"
              subPath: ""
          resources:
            limits:
              cpu: 10m
              memory: 50Mi
            requests:
              cpu: 10m
              memory: 50Mi
        - name: prometheus-alertmanager-configmap-reload
          image: "jimmidyson/configmap-reload:v0.1"
          imagePullPolicy: "IfNotPresent"
          args:
            - --volume-dir=/etc/config
            - --webhook-url=http://localhost:9093/-/reload
          volumeMounts:
            - name: config-volume
              mountPath: /etc/config
              readOnly: true
          resources:
            limits:
              cpu: 10m
              memory: 10Mi
            requests:
              cpu: 10m
              memory: 10Mi
      volumes:
        - name: config-volume
          configMap:
            name: alertmanager-config
        - name: storage-volume
          persistentVolumeClaim:
            claimName: alertmanager

```

##### 6.创建 alertmanager的service对外暴露的端口

```shell
[root@master01 k8s-prometheus]# vim alertmanager/alertmanager-service.yaml 
apiVersion: v1
kind: Service
metadata:
  name: alertmanager
  namespace: kube-system
  labels:
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
    kubernetes.io/name: "Alertmanager"
spec:
  ports:
    - name: http
      port: 80
      protocol: TCP
      targetPort: 9093
  selector:
    k8s-app: alertmanager
  type: "ClusterIP"
```

##### 7.检测部署状态，可以发现pod/alertmanager-5d75d5688f-fmlq6和service/alertmanager正常运行

```shell
[root@k8s-master prometheus-k8s]# kubectl get pod,svc -n kube-system -o wide
```

### 6.3 测试告警发送

登录prometheus默认的web界面，选择Alert菜单，则可以看到刚才我们使用prometheus-rules.yaml定义的的四个告警规则

![微信图片_20230317222203](\images\微信图片_20230317222203.jpg)

因为告警规则中定义了一个**InstanceDown** 的实例，所以我们可以停掉110服务器上的kubelet，来测试是否可以收到报警邮件发送

稍微等一会，我们再刷新刚才的web上的告警规则界面，可以发现**InstanceDown** 的实例颜色变成粉红色了，并且显示2 active

根据规则等待五分钟之后，我们去刷新刚才配置的告警收件箱，收到了一个封**InstanceDown** 的邮件提醒，邮件的发送间隔时间可以在alertmanager-configmap.yaml配置文件中进行设置，恢复刚才停止的kubelet，将不会收到告警邮件提醒

![微信图片_20230317222400](\images\微信图片_20230317222400.png)



##### -可以添加adapter控制pod的自动增减

##### -可以添加metrics-Server本地监控数据

##### -可以添加serviceMonitor规则监控自定义项目