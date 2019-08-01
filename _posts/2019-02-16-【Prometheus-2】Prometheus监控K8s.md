---
layout:  post  # 使用的布局（不需要改）
catalog: true  # 是否归档
author: 陈国林 # 作者
tags:          #标签
    - Prometheus
---

# 一. 简介
`Prometheus`和`Kubernets`都属于CNCF成员项目，推荐使用`Prometheus`来监控`Kubernets`集群。
传统的监控系统需要集群所有服务器将监控数据发往监控系统，`Prometheus`则可以通过`服务发现`来发现K8s集群内部已经暴露的监控点，然后主动拉取数据。

因此，我们只需要在`K8s`集群中部署一份`Prometheus`实例，它就可以通过向`Apiserver`查询集群状态，然后向所有已经支持Prometheus metrics的kubelet 获取所有Pod的运行数据。
这种动态发现的架构，非常适合服务器和程序都不固定的`K8s`集群环境。

以下部署都是基于`Mac Docker、K8s`单机环境，和集群部署会有一些区别。

# 二. Prometheus监控
`Prometheus`和`K8s`集成目前支持5种服务发现的模式，分别是`Node`，`Service`，`Pod`，`Endpoints`，`Ingress`。

1. Node
    * 从K8s集群各节点kubelet组件，获取节点`kubelet`的基本运行状态的监控指标
    * 从K8s集群各节点kubelet内置的cAdvisor，获取节点中运行的`容器`的监控指标
    * 从K8s集群各节点Node Exporter，获取`节点`运行资源相关的监控指标
2. Pod
    * 从Pod实例中采集业务应用自定义监控指标
3. Endpoints
    * 从K8s集群API Server组件，获取K8s集群相关的运行监控指标
4. Service
    * 从K8s集群Service的访问地址，通过Blackbox Exporter获取网络探测指标
5. Ingress
    * 从K8s集群Ingress的访问信息，通过Blackbox Exporter获取网络探测指标

## ① prometheus-configmap.yml
`Prometheus`服务启动需要配置prometheus.yml文件，在K8s集群中我们将配置文件存为`configmap`命名为`prometheus-configmap-v1.0.0.yml`，初始版本为v1.0.0。  
可以参考[prometheus-kubernetes.yml](https://github.com/prometheus/prometheus/blob/master/documentation/examples/prometheus-kubernetes.yml)

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config-v1.0.0
  namespace: kube-prometheus
  labels:
    k8s-app: prometheus-config
data:
  prometheus.yml: |
    global:
      # 数据拉取时间间隔，默认15s
      scrape_interval: 15s
      # 数据拉取超时时间，默认15s
      scrape_timeout: 15s

    scrape_configs:
    # Scrape config for prometheus
    - job_name: 'prometheus'
      static_configs:
        - targets: ['localhost:9090']

    # Scrape config for API servers
    - job_name: 'kubernetes-apiservers'
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      kubernetes_sd_configs:
      - role: endpoints
      relabel_configs:
      - source_labels: [__meta_kubernetes_namespace, __meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
        action: keep
        regex: default;kubernetes;https

    # Scrape config for nodes (kubelet)
    - job_name: 'kubernetes-nodes'
      scrape_interval: 30s
      scrape_timeout: 10s
      metrics_path: /metrics
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      kubernetes_sd_configs:
      - api_server: https://localhost:6443
        role: node
        tls_config:
          cert_file: /etc/kubernetes/ssl/xxxx.pem   #根据实际做替换
          key_file: /etc/kubernetes/ssl/xxxx.pem    #根据实际做替换
          insecure_skip_verify: true
      relabel_configs:
      - separator: ;
        regex: __meta_kubernetes_node_label_(.+)
        replacement: $1
        action: labelmap
      - separator: ;
        regex: (.*)
        target_label: __address__
        replacement: pre.kubernetes.m.com:6443
        action: replace
      - source_labels: [__meta_kubernetes_node_name]
        separator: ;
        regex: (.+)
        target_label: __metrics_path__
        replacement: /api/v1/nodes/${1}/proxy/metrics
        action: replace

    # Scrape config for Kubelet cAdvisor
    - job_name: kubernetes-cadvisor
      scrape_interval: 30s
      scrape_timeout: 10s
      metrics_path: /metrics
      scheme: http
      kubernetes_sd_configs:
      - api_server: https://localhost:6443
        role: node
        tls_config:
          cert_file: /etc/kubernetes/ssl/xxxx.pem     #根据实际做替换
          key_file: /etc/kubernetes/ssl/xxxx-key.pem  #根据实际做替换
          insecure_skip_verify: true
      tls_config:
        cert_file: /etc/kubernetes/ssl/xxxx.pem   #根据实际做替换
        key_file: /etc/kubernetes/ssl/xxxx.pem    #根据实际做替换
        insecure_skip_verify: true
      relabel_configs:
      - source_labels: [__address__]
        separator: ;
        regex: (.*):10250
        target_label: __address__
        replacement: ${1}:4194
        action: replace
      - source_labels: [__meta_kubernetes_node_address_InternalIP]
        separator: ;
        regex: (.*)
        target_label: ip
        replacement: $1
        action: replace

    # Scrape config for service endpoints
    - job_name: kubernetes-service-endpoints
      scrape_interval: 30s
      scrape_timeout: 10s
      metrics_path: /metrics
      scheme: http
      kubernetes_sd_configs:
      - api_server: https://localhost:6443
        role: endpoints
        tls_config:
          cert_file: /etc/kubernetes/ssl/xxxx.pem     #根据实际做替换
          key_file: /etc/kubernetes/ssl/xxxx.pem      #根据实际做替换
          insecure_skip_verify: true
      tls_config:
        cert_file: /etc/kubernetes/ssl/xxxx.pem     #根据实际做替换
        key_file: /etc/kubernetes/ssl/xxxx.pem      #根据实际做替换
        insecure_skip_verify: true
      relabel_configs:
      - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scrape]
        separator: ;
        regex: "true"
        replacement: $1
        action: keep
      - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scheme]
        separator: ;
        regex: (https?)
        target_label: __scheme__
        replacement: $1
        action: replace
      - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_path]
        separator: ;
        regex: (.+)
        target_label: __metrics_path__
        replacement: $1
        action: replace
      - source_labels: [__address__, __meta_kubernetes_service_annotation_prometheus_io_port]
        separator: ;
        regex: ([^:]+)(?::\d+)?;(\d+)
        target_label: __address__
        replacement: $1:$2
        action: replace
      - separator: ;
        regex: __meta_kubernetes_service_label_prometheus_(.+)
        replacement: $1
        action: labelmap
      - separator: ;
        regex: prometheus_(.+)
        replacement: $1
        action: labelmap
      - source_labels: [__meta_kubernetes_namespace]
        separator: ;
        regex: (.*)
        target_label: namespace
        replacement: $1
        action: replace

    # Scrape config for probing services via the Blackbox Exporter
    - job_name: 'kubernetes-services'
      metrics_path: /probe
      params:
        module: [http_2xx]
      kubernetes_sd_configs:
      - role: service
      relabel_configs:
      - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_probe]
        action: keep
        regex: true
      - source_labels: [__address__]
        target_label: __param_target
      - target_label: __address__
        replacement: blackbox
      - source_labels: [__param_target]
        target_label: instance
      - action: labelmap
        regex: __meta_kubernetes_service_label_(.+)
      - source_labels: [__meta_kubernetes_service_namespace]
        target_label: kubernetes_namespace
      - source_labels: [__meta_kubernetes_service_name]
        target_label: kubernetes_name

    # Scrape config for Pod
    - job_name: kubernetes-pods
      scrape_interval: 30s
      scrape_timeout: 10s
      metrics_path: /metrics
      scheme: http
      kubernetes_sd_configs:
      - api_server: https://localhost:6443
        role: pod
        tls_config:
          cert_file: /etc/kubernetes/ssl/xxxx.pem
          key_file: /etc/kubernetes/ssl/xxxx.pem
          insecure_skip_verify: true
      tls_config:
        cert_file: /etc/kubernetes/ssl/xxxx.pem
        key_file: /etc/kubernetes/ssl/xxxx.pem
        insecure_skip_verify: true
      relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        separator: ;
        regex: "true"
        replacement: $1
        action: keep
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scheme]
        separator: ;
        regex: (https?)
        target_label: __scheme__
        replacement: $1
        action: replace
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        separator: ;
        regex: (.+)
        target_label: __metrics_path__
        replacement: $1
        action: replace
      - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
        separator: ;
        regex: ([^:]+)(?::\d+)?;(\d+)
        target_label: __address__
        replacement: $1:$2
        action: replace
      - separator: ;
        regex: __meta_kubernetes_pod_label_(.+)
        replacement: $1
        action: labelmap
      - source_labels: [__meta_kubernetes_namespace]
        separator: ;
        regex: (.*)
        target_label: kubernetes_namespace
        replacement: $1
        action: replace
      - source_labels: [__meta_kubernetes_pod_name]
        separator: ;
        regex: (.*)
        target_label: kubernetes_pod_name
        replacement: $1
        action: replace
      - separator: ;
        regex: prometheus_(.+)
        replacement: $1
        action: labelmap
```

1. 配置文件详解
    * tls_config: SSL/TLS配置
        * ca_file: CA证书路径
    * bearer_token_file: token文件路径
    * kubernetes_sd_configs: 服务发现配置
        * role: 服务发现的模式
    * relabel_configs: labels重写配置，比较重要的配置，具体可以参考[relabel_config](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#relabel_config)
        * `__address__` label将会被设置为`instance`
        * `__meta__`开头的label是由Prometheus服务发现设置的，在relabel过程都可以使用
        * relabel结束后`__`开头的所有label都会被删除
2. 部署K8s prometheus configmap
    * 创建kube-prometheus ns: `$ kubectl create namespace kube-prometheus`
    * 创建configmap: `$ kubectl apply -f prometheus-configmap-v1.0.0.yml`
    * 查看configmap: `$ kubectl get cm -n kube-prometheus`

## ② prometheus deployment
使用`Deployment`部署`Prometheus`服务，保证任何时候都有Pod在运行，`prometheus-deployment-v1.0.0.yml`初始版本为v1.0.0。

```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: prometheus-deployment
  namespace: kube-prometheus
  labels:
    k8s-app: prometheus-deployment
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: prometheus-deployment
    spec:
      containers:
      - image: prom/prometheus:v2.6.0
        name: prometheus
        command:
        - "/bin/prometheus"
        args:
        - "--config.file=/etc/prometheus/prometheus.yml"
        - "--storage.tsdb.path=/prometheus"
        - "--storage.tsdb.retention=24h"
        ports:
        - containerPort: 9090
          protocol: TCP
        volumeMounts:
        - mountPath: "/prometheus"
          name: data
        - mountPath: "/etc/prometheus"
          name: config-volume
        resources:
          requests:
            cpu: 100m
            memory: 100Mi
          limits:
            cpu: 500m
            memory: 2500Mi
      volumes:
      - emptyDir: {}
        name: data
      - configMap:
          name: prometheus-config-v1.0.0
        name: config-volume
```

1. 配置文件详解
    * metadata: Deployment相关的元信息
    * spec.template.spec.containers: Pod容器相关配置
    * spec.template.spec.volumes: 卷相关的配置，配置从configmap读取配置文件
2. 部署prometheus deployment
    * 启动Prometheus服务: `$ kubectl apply -f prometheus-deployment-v1.0.0.yml`
    * 查看Prometheus服务是否启动成功: `$ kubectl get all -n kube-prometheus`
    * 查看Prometheus服务日志: `$ kubectl logs -f pod/prometheus-6786ddbfb8-wxxhn -n kube-prometheus`

## ③ prometheus service
把Prometheus Deployment相关的Pod抽象成Service，方便其它Pod访问，同时为了外网也能够访问Prometheus服务我们在Service之上加一层Ingress。

`prometheus-service-v1.0.0.yml`初始版本为v1.0.0。
```
apiVersion: v1
kind: Service
metadata:
  name: prometheus-service
  namespace: kube-prometheus
  labels:
    k8s-app: prometheus-service
spec:
  selector:
    app: prometheus-deployment
  ports:
    - protocol: TCP
      port: 9090
      targetPort: 9090

---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: prometheus-ingress
  namespace: kube-prometheus
  labels:
    k8s-app: prometheus-ingress
spec:
  rules:
  - host: k8s.prometheus.com
    http:
      paths:
      - path: /
        backend:
          serviceName: prometheus-service
          servicePort: 9090
```

1. Service配置文件详解
    * metadata.name: service名称为prometheus-service
    * spec.selector: 选择label为app=prometheus-deployment相关的Pod
    * spec.ports.port: 开放9090端口，默认和targetPort一样
    * spec.ports.targetPort: 后端Prometheus Pod对应的端口为9090
2. Ingress配置文件详解    
    * metadata.name: ingress名称为prometheus-ingress
    * spec.rules.host: k8s.prometheus.com 配置访问域名 (需要本地配置hosts绑定到nginx-ingress-controller任一节点上)
    * spec.rules.http.paths.backend.serviceName: 后端对应的service名称
    * spec.rules.http.paths.backend.servicePort: 后端对应的service端口
3. 部署Service和Ingress
    * kubectl apply -f prometheus-service-v1.0.0.yml
4. 直接访问`http://k8s.prometheus.com/graph`即可访问Prometheus web UI界面
   ![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/prometheus-localhost.png?raw=true) 
    
`注意: K8s上相关的资源例如Pod如果希望被Prometheus采集，需要设置annotations prometheus_io_scrape: "true"，如果没有设置prometheus_io_port则prometheus默认会拉取容器暴露的所有端口（适合用于一个Pod内有多个想被采集的端口）`    
    
# 三. Grafana展示
`Prometheus`自带的web UI功能比较简单，业界的做法是使用`Grafana`来做监控报表展示。`Grafana`是一个开源的时序统计和监控平台，支持`Elasticsearch`、`Graphite`、`Influxdb`、`Prometheus`等众多的数据源，并以功能强大的界面编辑器著称。

## ① Grafana Deployment
我们也把`Grafana`部署在K8s集群中，`grafana-deployment-v1.0.0.yml`配置文件如下，初始版本为1.0.0。

```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: grafana-deployment
  namespace: kube-grafana
  labels:
    app: grafana-deployment
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: grafana-deployment
    spec:
      containers:
      - name: grafana
        image: grafana/grafana:5.0.0
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 3000
          name: grafana
        env:
        - name: GF_SECURITY_ADMIN_USER
          value: admin
        - name: GF_SECURITY_ADMIN_PASSWORD
          value: admin
        resources:
          limits:
            cpu: 100m
            memory: 256Mi
          requests:
            cpu: 100m
            memory: 256Mi
        volumeMounts:
        - name: storage
          mountPath: /var/lib/grafana
      volumes:
      - name: storage
        hostPath:
          path: /www/grafana
```

1. 配置文件详解
    * spec.template.spec.containers.image: 使用grafana/grafana:5.0.0版本镜像
    * spec.template.spec.containers.env: 设置grafana环境变量，管理员默认账户名和密码
    * spec.template.spec.containers.resources: 资源申请
    * spec.template.spec.volumes: 卷设置，由于Grafana需要持久化存储数据所以我们需要使用hostPath把Pod数据挂载到宿主机上
2. 部署Grafana
    * 创建namespace: `$ kubectl  create namespace kube-grafana`
    * 部署Deployment: `$ kubectl apply -f grafana-deployment-v1.0.0.yml`
    * 查看是否部署成功: `$ kubectl get all -n kube-grafana`

## ② Grafana service
把Grafana Deployment相关的Pod抽象成Service，方便其它Pod访问，同时为了外网也能够访问Grafana服务我们在Service之上加一层Ingress。

grafana-service-v1.0.0.yml
```
apiVersion: v1
kind: Service
metadata:
  name: grafana-service
  namespace: kube-grafana
  labels:
    k8s-app: grafana-service
spec:
  selector:
    app: grafana-deployment
  ports:
    - protocol: TCP
      port: 3000
      targetPort: 3000

---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: grafana-ingress
  namespace: kube-grafana
  labels:
    k8s-app: grafana-ingress
spec:
  rules:
  - host: k8s.grafana.com
    http:
      paths:
      - path: /
        backend:
          serviceName: grafana-service
          servicePort: 3000
```

1. Service配置文件详解
    * metadata.name: service名称为grafana-service
    * spec.selector: 选择label为app=grafana-deployment相关的Pod
    * spec.ports.port: 开放3000端口，默认和targetPort一样
    * spec.ports.targetPort: 后端grafana Pod对应的端口为3000
2. Ingress配置文件详解    
    * metadata.name: ingress名称为grafana-ingress
    * spec.rules.host: k8s.grafana.com 配置访问域名 (需要本地配置hosts绑定到nginx-ingress-controller任一节点上)
    * spec.rules.http.paths.backend.serviceName: 后端对应的service名称
    * spec.rules.http.paths.backend.servicePort: 后端对应的service端口
3. 部署Service和Ingress
    * kubectl apply -f grafana-service-v1.0.0.yml
4. 直接访问`http://k8s.grafana.com/login`即可访问Grafana界面，默认账号为admin:admin
   ![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/grafana-login.png?raw=true) 
    
## ③ Grafana 报表
![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/grafana-login.png?raw=true)

![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/grafana-datasource.png?raw=true)

![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/grafana-prometheus.png?raw=true)

![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/grafana-k8s-monitor.png?raw=true)

## ④ 开源Grafana报表
Grafana官方提供了一些开源的dashboard，关于Kubernets的监控有以下一些报表可供参考
https://grafana.com/dashboards?dataSource=prometheus&search=Kubernetes

1. Kubernetes Cluster monitor dashboard
    * https://grafana.com/api/dashboards/7249/revisions/1/download
    * https://grafana.com/api/dashboards/8685/revisions/1/download
    * https://grafana.com/api/dashboards/162/revisions/1/download
    * https://grafana.com/api/dashboards/6417/revisions/1/download
2. Kubernetes Node monitor dashboard
    * https://grafana.com/api/dashboards/941/revisions/1/download
    * https://grafana.com/api/dashboards/3140/revisions/1/download
3. Kubernets Pod monitor dashboard
    * https://grafana.com/api/dashboards/6336/revisions/1/download
    * https://grafana.com/api/dashboards/7670/revisions/1/download
    * https://grafana.com/api/dashboards/747/revisions/2/download

