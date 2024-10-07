![image-20241006153455006](https://img2023.cnblogs.com/blog/3332572/202410/3332572-20241007204543097-212092999.png)
![image-20241006153710723](https://img2023.cnblogs.com/blog/3332572/202410/3332572-20241007204542079-238942752.png)
![image-20241007180724123](https://img2023.cnblogs.com/blog/3332572/202410/3332572-20241007204525712-663083160.png)
![image-20241007203427726](https://img2023.cnblogs.com/blog/3332572/202410/3332572-20241007204520040-311446757.png)

# 一、机器规划

|  角色  |   主机名    |    ip 地址     |
| :----: | :---------: | :------------: |
| master | k8s-master1 | 192.168.112.10 |
|  node  |  k8s-node1  | 192.168.112.20 |
|  node  |  k8s-node2  | 192.168.112.30 |

|     平台      |            VMware Workstation            |
| :-----------: | :--------------------------------------: |
| **操作系统**  | **CentOS Linux release 7.9.2009 (Core)** |
| **内存、CPU** |                 **4C4G**                 |
| **磁盘大小**  |               **20G SCSI**               |

# 二、部署安装 node-exporter、prometheus、Grafana、kube-state-metrics

## 1、创建 monitor-sa 命名空间

> master 节点操作

```bash
kubectl create ns monitor-sa
```

## 2、安装node-exporter组件

> master 节点操作

```yaml
cat >> node-export.yaml  <<EOF
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
  namespace: monitor-sa
  labels:
    name: node-exporter
spec:
  selector:
    matchLabels:
     name: node-exporter
  template:
    metadata:
      labels:
        name: node-exporter
    spec:
      hostPID: true
      hostIPC: true
      hostNetwork: true
      containers:
      - name: node-exporter
        image: prom/node-exporter:v0.16.0
        ports:
        - containerPort: 9100
        resources:
          requests:
            cpu: 0.15
        securityContext:
          privileged: true
        args:
        - --path.procfs
        - /host/proc
        - --path.sysfs
        - /host/sys
        - --collector.filesystem.ignored-mount-points
        - '"^/(sys|proc|dev|host|etc)($|/)"'
        volumeMounts:
        - name: dev
          mountPath: /host/dev
        - name: proc
          mountPath: /host/proc
        - name: sys
          mountPath: /host/sys
        - name: rootfs
          mountPath: /rootfs
      tolerations:
      - key: "node-role.kubernetes.io/master"
        operator: "Exists"
        effect: "NoSchedule"
      volumes:
        - name: proc
          hostPath:
            path: /proc
        - name: dev
          hostPath:
            path: /dev
        - name: sys
          hostPath:
            path: /sys
        - name: rootfs
          hostPath:
            path: /
EOF
```

### 2.1、说明

- **主机命名空间共享 (`hostPID`, `hostIPC`, `hostNetwork`)**
  - **`hostPID: true`**: 允许 Pod 使用主机的 PID 命名空间。Pod 可以看到主机上的所有进程
  - **`hostIPC: true`**: 允许 Pod 使用主机的 IPC 命名空间。Pod 可以与其他在主机上运行的进程共享 IPC 资源（如信号量、消息队列等）。
  - **`hostNetwork: true`**: 允许 Pod 使用主机的网络命名空间。Pod 将使用主机的网络接口
-  **命令行参数 (`args`)**
  - **`--path.procfs /host/proc`**: 指定 `node-exporter` 应该从 `/host/proc` 路径读取进程文件系统的数据。这使得 `node-exporter` 可以访问宿主机的进程信息。
  - **`--path.sysfs /host/sys`**: 指定 `node-exporter` 应该从 `/host/sys` 路径读取系统文件系统的数据。这使得 `node-exporter` 可以访问宿主机的系统信息。
  - **`--collector.filesystem.ignored-mount-points "^/(sys|proc|dev|host|etc)($|/)"`**: 指定哪些文件系统的挂载点应该被忽略，不被 `node-exporter` 收集。这里忽略了 `/sys`, `/proc`, `/dev`, `/host`, 和 `/etc` 这些挂载点，避免收集不必要的数据。
- **挂载点 (`volumeMounts` 和 `volumes`)**
  - `/proc` 挂载
    - **宿主机路径:** `/proc`
    - **容器内路径:** `/host/proc`
    - **作用:** 让 `node-exporter` 访问宿主机的进程文件系统。
  - `/dev` 挂载
    - **宿主机路径:** `/dev`
    - **容器内路径:** `/host/dev`
    - **作用:** 让 `node-exporter` 访问宿主机的设备文件。
  - `/sys` 挂载
    - **宿主机路径:** `/sys`
    - **容器内路径:** `/host/sys`
    - **作用:** 让 `node-exporter` 访问宿主机的系统文件系统。
  - `/` 挂载
    - **宿主机路径:** `/`
    - **容器内路径:** `/rootfs`
    - **作用:** 让 `node-exporter` 访问宿主机的根文件系统。
- **容忍度 (`tolerations`)**
  - **`key: "node-role.kubernetes.io/master"`**: 指定容忍的污点键。
  - **`operator: "Exists"`**: 表示只要存在该污点键，无论值是什么，都予以容忍。
  - **`effect: "NoSchedule"`**: 表示即使节点上有这种污点，也不会阻止 Pod 被调度到该节点上。

### 2.2、应用资源清单

```bash
kubectl apply -f node-export.yaml

kubectl get pods -n monitor-sa -l name=node-exporter
```

![image-20241005214533113](https://img2023.cnblogs.com/blog/3332572/202410/3332572-20241007204551383-460197980.png)

### 2.3、通过node-exporter采集数据

> node-export默认的监听端口是9100，可以看到当前主机获取到的所有监控数据

```bash
# curl http://<master-ip>:9100/metrics

curl http://192.168.112.10:9100/metrics
```

![image-20241005214626996](https://img2023.cnblogs.com/blog/3332572/202410/3332572-20241007204550978-1331719554.png)

## 3、k8s 集群中部署 prometheus

### 3.1、创建一个 sa 账号

```bash
kubectl create serviceaccount monitor -n monitor-sa
```

### 3.2、将 sa 账号 monitor 通过 clusterrolebing 绑定到 clusterrole 上

```bash
kubectl create clusterrolebinding monitor-clusterrolebinding -n monitor-sa --clusterrole=cluster-admin  --serviceaccount=monitor-sa:monitor
```

### 3.3、创建数据目录

> 所有 node 节点

```bash
mkdir /data && chmod 777 /data/
```

### 3.4、安装prometheus

> master 节点操作

#### 3.4.1、将 `prometheus.yml` 文件以 ConfigMap 的形式进行管理

```yaml
cat  >> prometheus-cfg.yaml << 'EOF'
---
kind: ConfigMap
apiVersion: v1
metadata:
  labels:
    app: prometheus
  name: prometheus-config
  namespace: monitor-sa
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s
      scrape_timeout: 10s
      evaluation_interval: 1m
    scrape_configs:
    - job_name: 'kubernetes-node'
      kubernetes_sd_configs:
      - role: node
      relabel_configs:
      - source_labels: [__address__]
        regex: '(.*):10250'
        replacement: '${1}:9100'
        target_label: __address__
        action: replace
      - action: labelmap
        regex: __meta_kubernetes_node_label_(.+)
    - job_name: 'kubernetes-node-cadvisor'
      kubernetes_sd_configs:
      - role:  node
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      relabel_configs:
      - action: labelmap
        regex: __meta_kubernetes_node_label_(.+)
      - target_label: __address__
        replacement: kubernetes.default.svc:443
      - source_labels: [__meta_kubernetes_node_name]
        regex: (.+)
        target_label: __metrics_path__
        replacement: '/api/v1/nodes/${1}/proxy/metrics/cadvisor'
    - job_name: 'kubernetes-apiserver'
      kubernetes_sd_configs:
      - role: endpoints
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      relabel_configs:
      - source_labels: [__meta_kubernetes_namespace, __meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
        action: keep
        regex: default;kubernetes;https
    - job_name: 'kubernetes-service-endpoints'
      kubernetes_sd_configs:
      - role: endpoints
      relabel_configs:
      - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scheme]
        action: replace
        target_label: __scheme__
        regex: (https?)
      - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)
      - source_labels: [__address__, __meta_kubernetes_service_annotation_prometheus_io_port]
        action: replace
        target_label: __address__
        regex: ([^:]+)(?::\d+)?;(\d+)
        replacement: '$1:$2'
      - action: labelmap
        regex: __meta_kubernetes_service_label_(.+)
      - source_labels: [__meta_kubernetes_namespace]
        action: replace
        target_label: kubernetes_namespace
      - source_labels: [__meta_kubernetes_service_name]
        action: replace
        target_label: kubernetes_name 
EOF
```

#### 3.4.2、应用 cm 资源清单

```bash
kubectl apply -f prometheus-cfg.yaml

kubectl get cm prometheus-config -n monitor-sa -o yaml
```

> 需要确保 cm 正确解析了变量 $1、$2
>
> 不然 prometheus 获取不到对应的 IP 地址会无法正常监控

![image-20241005215107744](https://img2023.cnblogs.com/blog/3332572/202410/3332572-20241007204550481-1166117457.png)

#### 3.4.3、通过 Deployment 部署 prometheus

```yaml
cat >> prometheus-deploy.yaml << EOF
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus-server
  namespace: monitor-sa
  labels:
    app: prometheus
spec:
  replicas: 2
  selector:
    matchLabels:
      app: prometheus
      component: server
    #matchExpressions:
    #- {key: app, operator: In, values: [prometheus]}
    #- {key: component, operator: In, values: [server]}
  template:
    metadata:
      labels:
        app: prometheus
        component: server
      annotations:
        prometheus.io/scrape: 'false'
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - prometheus
              - key: component
                operator: In
                values:
                - server
            topologyKey: kubernetes.io/hostname
      serviceAccountName: monitor
      containers:
      - name: prometheus
        image: quay.io/prometheus/prometheus:latest
        imagePullPolicy: IfNotPresent
        command:
          - prometheus
          - --config.file=/etc/prometheus/prometheus.yml
          - --storage.tsdb.path=/prometheus
          - --storage.tsdb.retention=720h
        ports:
        - containerPort: 9090
          protocol: TCP
        volumeMounts:
        - mountPath: /etc/prometheus/prometheus.yml
          name: prometheus-config
          subPath: prometheus.yml
        - mountPath: /prometheus/
          name: prometheus-storage-volume
      volumes:
        - name: prometheus-config
          configMap:
            name: prometheus-config
            items:
              - key: prometheus.yml
                path: prometheus.yml
                mode: 0644
        - name: prometheus-storage-volume
          hostPath:
           path: /data
           type: Directory
EOF
```

#### 3.4.4、应用 prometheus 资源清单

```bash
kubectl apply -f prometheus-deploy.yaml
```

![image-20241005215357542](https://img2023.cnblogs.com/blog/3332572/202410/3332572-20241007204550058-874773747.png)

#### 3.4.5、给 prometheus 的 pod 创建一个 svc

```yaml
cat  > prometheus-svc.yaml << EOF
---
apiVersion: v1
kind: Service
metadata:
  name: prometheus
  namespace: monitor-sa
  labels:
    app: prometheus
spec:
  type: NodePort
  ports:
    - port: 9090
      targetPort: 9090
      protocol: TCP
  selector:
    app: prometheus
    component: server
EOF
```

#### 3.4.6、应用 svc 资源清单

```bash
kubectl get svc -n monitor-sa -o wide
```

![image-20241005215425028](https://img2023.cnblogs.com/blog/3332572/202410/3332572-20241007204549736-1128295035.png)

通过上面可以看到service在宿主机上映射的端口是30172，这样我们访问k8s集群的k8s-master1节点的ip:30172，就可以访问到prometheus的web ui界面了

### 3.5、访问prometheus UI界面

```bash
# <k8s-master1 IP>:32032
192.168.112.10:32032
```

![image-20241005215529171](https://img2023.cnblogs.com/blog/3332572/202410/3332572-20241007204549384-2031711416.png)

### 3.6、查看配置的服务发现

点击页面的Status->Targets，可看到如下,说明我们配置的服务发现可以正常采集数据

![image-20241005221024862](https://img2023.cnblogs.com/blog/3332572/202410/3332572-20241007204549026-2147170613.png)

## 4、prometheus热更新

### 4.1、热加载 prometheus

\#为了每次修改配置文件可以热加载prometheus，也就是不停止prometheus，就可以使配置生效，如修改prometheus-cfg.yaml，想要使配置生效可用如下热加载命令：

```bash
curl -X POST http://<prometheus-pod-ip>:9090/-/reload
```

> <prometheus-pod-ip>

```bash
kubectl get pods -n monitor-sa -l app=prometheus -o wide
```

![image-20241005221822766](https://img2023.cnblogs.com/blog/3332572/202410/3332572-20241007204548674-703370492.png)

### 4.2、暴力重启 prometheus

> 热加载速度比较慢，可以暴力重启prometheus
>
> 如修改上面的prometheus-cfg.yaml文件之后，可执行如下强制删除

```bash
kubectl delete -f prometheus-cfg.yaml

kubectl delete -f prometheus-deploy.yaml

# 然后再通过apply更新

kubectl apply -f prometheus-cfg.yaml

kubectl apply -f prometheus-deploy.yaml
```

> 线上最好热加载，暴力删除可能造成监控数据的丢失

## 5、Grafana安装和配置

### 5.1、下载 Grafana 需要的镜像

```bash
链接：https://pan.baidu.com/s/1TmVGKxde_cEYrbjiETboEA 
提取码：052u
```

### 5.2、在 k8s 集群各个节点导入 Grafana 镜像

```bash
docker load -i heapster-grafana-amd64_v5_0_4.tar.gz

docker images | grep grafana
```

![image-20241005231752018](https://img2023.cnblogs.com/blog/3332572/202410/3332572-20241007204548322-259537964.png)

![image-20241005231829736](https://img2023.cnblogs.com/blog/3332572/202410/3332572-20241007204547856-1926697012.png)

![image-20241005231844131](https://img2023.cnblogs.com/blog/3332572/202410/3332572-20241007204547483-1589976776.png)

### 5.3、master 节点创建 grafana.yaml

```\yaml
cat >> grafana.yaml << EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: monitoring-grafana
  namespace: kube-system
spec:
  replicas: 1
  selector:
    matchLabels:
      task: monitoring
      k8s-app: grafana
  template:
    metadata:
      labels:
        task: monitoring
        k8s-app: grafana
    spec:
      containers:
      - name: grafana
        image: k8s.gcr.io/heapster-grafana-amd64:v5.0.4
        ports:
        - containerPort: 3000
          protocol: TCP
        volumeMounts:
        - mountPath: /etc/ssl/certs
          name: ca-certificates
          readOnly: true
        - mountPath: /var
          name: grafana-storage
        env:
        - name: INFLUXDB_HOST
          value: monitoring-influxdb
        - name: GF_SERVER_HTTP_PORT
          value: "3000"
          # The following env variables are required to make Grafana accessible via
          # the kubernetes api-server proxy. On production clusters, we recommend
          # removing these env variables, setup auth for grafana, and expose the grafana
          # service using a LoadBalancer or a public IP.
        - name: GF_AUTH_BASIC_ENABLED
          value: "false"
        - name: GF_AUTH_ANONYMOUS_ENABLED
          value: "true"
        - name: GF_AUTH_ANONYMOUS_ORG_ROLE
          value: Admin
        - name: GF_SERVER_ROOT_URL
          # If you're only using the API Server proxy, set this value instead:
          # value: /api/v1/namespaces/kube-system/services/monitoring-grafana/proxy
          value: /
      volumes:
      - name: ca-certificates
        hostPath:
          path: /etc/ssl/certs
      - name: grafana-storage
        emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  labels:
    # For use as a Cluster add-on (https://github.com/kubernetes/kubernetes/tree/master/cluster/addons)
    # If you are NOT using this as an addon, you should comment out this line.
    kubernetes.io/cluster-service: 'true'
    kubernetes.io/name: monitoring-grafana
  name: monitoring-grafana
  namespace: kube-system
spec:
  # In a production setup, we recommend accessing Grafana through an external Loadbalancer
  # or through a public IP.
  # type: LoadBalancer
  # You could also use NodePort to expose the service at a randomly-generated port
  # type: NodePort
  ports:
  - port: 80
    targetPort: 3000
  selector:
    k8s-app: grafana
  type: NodePort
EOF
```

### 5.4、查看 Grafana 的 pod 和 svc

![image-20241005232832195](https://img2023.cnblogs.com/blog/3332572/202410/3332572-20241007204547116-375263658.png)

### 5.5、查看 Grafana UI 界面

```bash
# <master-ip>:<grafana-svc-port>

192.168.112.10:31455
```

![image-20241006150242320](https://img2023.cnblogs.com/blog/3332572/202410/3332572-20241007204546772-152660888.png)

### 5.6、给 Grafana 接入 Prometheus 数据源

|              选择 Create your first data source              |
| :----------------------------------------------------------: |
| ![image-20241006150534861](https://img2023.cnblogs.com/blog/3332572/202410/3332572-20241007204546400-1569925420.png) |
| ![image-20241006150624325](https://img2023.cnblogs.com/blog/3332572/202410/3332572-20241007204546003-365105653.png) |
| **Name: Prometheus \|Type: Prometheus\|HTTP 处的URL写 如下：http://prometheus.monitor-sa.svc:9090** |
| ![image-20241006151022903](https://img2023.cnblogs.com/blog/3332572/202410/3332572-20241007204545578-1638824283.png) |
| **点击左下角 Save & Test，出现如下 Data source is working，说明 prometheus 数据源成功的被 grafana 接入了** |
| ![image-20241006151134680](https://img2023.cnblogs.com/blog/3332572/202410/3332572-20241007204545186-635302832.png) |
| ![image-20241006151148648](https://img2023.cnblogs.com/blog/3332572/202410/3332572-20241007204544808-1363511905.png) |

### 5.7、获取监控模板

- **可以在 Grafana Dashboard 官网搜索需要的**

[Grafana dashboards | Grafana Labs](https://grafana.com/grafana/dashboards/?dataSource=prometheus&search=kubernetes)

- **也可以直接克隆  Github 仓库，获取 node_exporter.json 、 docker_rev1.json 监控模板**

```bash
git clone git@github.com:misakivv/Grafana-Dashboard.git
```

### 5.8、导入监控模板

|              依次点击左侧栏的 + 号下方的 Import              |
| :----------------------------------------------------------: |
| ![image-20241006152716109](https://img2023.cnblogs.com/blog/3332572/202410/3332572-20241007204544413-1066650436.png) |
| **选择 Upload json file，选择一个本地的node_exporter.json 文件** |
| ![image-20241006153035574](https://img2023.cnblogs.com/blog/3332572/202410/3332572-20241007204544004-2134914446.png) |
| **导入后 Options 选项中会出现 Name 是自动生成的，Prometheus 是需要我们选择 Prometheus的** |
| ![image-20241006153231878](https://img2023.cnblogs.com/blog/3332572/202410/3332572-20241007204543561-429036717.png) |
|               **点击 Import 即可出现如下界面**               |
| ![image-20241006153455006](https://img2023.cnblogs.com/blog/3332572/202410/3332572-20241007204543097-212092999.png) |
|        **按照如上操作，导入docker_rev1.json监控模板**        |
| ![image-20241006153635176](https://img2023.cnblogs.com/blog/3332572/202410/3332572-20241007204542584-438538269.png) |
| ![image-20241006153710723](https://img2023.cnblogs.com/blog/3332572/202410/3332572-20241007204542079-238942752.png) |

## 6、安装配置 kube-state-metrics 组件

### 6.1、什么是 kube-state-metrics

kube-state-metrics通过监听API Server生成有关资源对象的状态指标，比如Deployment、Node、Pod，需要注意的是kube-state-metrics只是简单的提供一个metrics数据，并不会存储这些指标数据，所以我们可以使用Prometheus来抓取这些数据然后存储，主要关注的是业务相关的一些元数据，比如Deployment、Pod、副本状态等；调度了多少个replicas？现在可用的有几个？多少个Pod是running/stopped/terminated状态？Pod重启了多少次？有多少job在运行中。

### 6.2、创建 sa ，并进行授权

> k8s-master1 节点编写一个 kube-state-metrics-rbac.yaml 文件

```yaml
cat >> kube-state-metrics-rbac.yaml << EOF
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: kube-state-metrics
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: kube-state-metrics
rules:
- apiGroups: [""]
  resources: ["nodes", "pods", "services", "resourcequotas", "replicationcontrollers", "limitranges", "persistentvolumeclaims", "persistentvolumes", "namespaces", "endpoints"]
  verbs: ["list", "watch"]
- apiGroups: ["extensions"]
  resources: ["daemonsets", "deployments", "replicasets"]
  verbs: ["list", "watch"]
- apiGroups: ["apps"]
  resources: ["statefulsets"]
  verbs: ["list", "watch"]
- apiGroups: ["batch"]
  resources: ["cronjobs", "jobs"]
  verbs: ["list", "watch"]
- apiGroups: ["autoscaling"]
  resources: ["horizontalpodautoscalers"]
  verbs: ["list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: kube-state-metrics
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: kube-state-metrics
subjects:
- kind: ServiceAccount
  name: kube-state-metrics
  namespace: kube-system
EOF
```

```bash
kubectl get sa,clusterrole,clusterrolebinding -n kube-system | grep kube-state-metrics
```

![image-20241006155708266](https://img2023.cnblogs.com/blog/3332572/202410/3332572-20241007204541476-1484349009.png)

### 6.3、创建并应用 kube-state-metrics-deploy.yaml 文件

> k8s-master1 节点操作

```yaml
cat > kube-state-metrics-deploy.yaml <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kube-state-metrics
  namespace: kube-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kube-state-metrics
  template:
    metadata:
      labels:
        app: kube-state-metrics
    spec:
      serviceAccountName: kube-state-metrics
      containers:
      - name: kube-state-metrics
#        image: gcr.io/google_containers/kube-state-metrics-amd64:v1.3.1
        image: quay.io/coreos/kube-state-metrics:latest
        ports:
        - containerPort: 8080
EOF
```

```bash
kubectl apply -f kube-state-metrics-deploy.yaml

kubectl get pods -n kube-system -l app=kube-state-metrics -w
```

![image-20241006162620908](https://img2023.cnblogs.com/blog/3332572/202410/3332572-20241007204541119-818875188.png)

> 拉取 kube-state-metrics 指定镜像版本失败时可以选择在集群各个节点上
>
> docker pull quay.io/coreos/kube-state-metrics:latest
>
> 拉取最新 tag 版本

![image-20241006162304963](https://img2023.cnblogs.com/blog/3332572/202410/3332572-20241007204540691-1113213521.png)

### 6.4、创建并应用 kube-state-metrics-svc.yaml 文件

> k8s-master1 节点操作

```yaml
cat >> kube-state-metrics-svc.yaml <<EOF
apiVersion: v1
kind: Service
metadata:
  annotations:
    prometheus.io/scrape: 'true'
  name: kube-state-metrics
  namespace: kube-system
  labels:
    app: kube-state-metrics
spec:
  ports:
  - name: kube-state-metrics
    port: 8080
    protocol: TCP
  selector:
    app: kube-state-metrics
EOF
```

```bash
kubectl apply -f kube-state-metrics-svc.yaml

kubectl get svc -n kube-system -l app=kube-state-metrics
```

![image-20241006163135415](https://img2023.cnblogs.com/blog/3332572/202410/3332572-20241007204540359-1766067012.png)

### 6.5、获取 kube-state-metrics json 文件

```bash
git clone git@github.com:misakivv/Grafana-Dashboard.git
```

![image-20241006163929057](https://img2023.cnblogs.com/blog/3332572/202410/3332572-20241007204540038-267464911.png)

### 6.6、向 Grafana 导入 kube-state-metrics json 文件

|                   点击左侧栏 + 号的 Import                   |
| :----------------------------------------------------------: |
| ![image-20241006163710887](https://img2023.cnblogs.com/blog/3332572/202410/3332572-20241007204539644-852632905.png) |
| **点击 Upload .json File，上传 Kubernetes Cluster (Prometheus)-1577674936972.json** |
| ![image-20241006164143527](https://img2023.cnblogs.com/blog/3332572/202410/3332572-20241007204539174-747293929.png) |
| ![image-20241006164305915](https://img2023.cnblogs.com/blog/3332572/202410/3332572-20241007204538753-2001854212.png) |
|                           **查看**                           |
| ![image-20241006165042653](https://img2023.cnblogs.com/blog/3332572/202410/3332572-20241007204538349-1130291806.png) |
| **同样的导入 Kubernetes cluster monitoring (via Prometheus) (k8s 1.16)-1577691996738.json ** |
| ![](https://img2023.cnblogs.com/blog/3332572/202410/3332572-20241007204537783-2132707089.png) |
| ![image-20241006165253781](https://img2023.cnblogs.com/blog/3332572/202410/3332572-20241007204537334-639720976.png) |
| ![image-20241006165821679](https://img2023.cnblogs.com/blog/3332572/202410/3332572-20241007204536762-1205246428.png) |
| ![image-20241006165850539](https://img2023.cnblogs.com/blog/3332572/202410/3332572-20241007204536366-531136992.png) |
| ![image-20241006165912093](https://img2023.cnblogs.com/blog/3332572/202410/3332572-20241007204535940-1879411887.png) |
| ![image-20241006165931820](https://img2023.cnblogs.com/blog/3332572/202410/3332572-20241007204535558-599319438.png) |
| ![image-20241006165949255](https://img2023.cnblogs.com/blog/3332572/202410/3332572-20241007204535098-2035774107.png) |
| ![image-20241006170018099](https://img2023.cnblogs.com/blog/3332572/202410/3332572-20241007204534093-177247946.png) |
| ![image-20241006170044368](https://img2023.cnblogs.com/blog/3332572/202410/3332572-20241007204533503-974427855.png) |

# 三、安装和配置 Alertmanager -- 发送告警到 QQ 邮箱

## 1、将 alertmanager-cm.yaml 文件以 cm 形式进行管理

> k8s-master1 节点操作

```yaml
cat >> alertmanager-cm.yaml << EOF
kind: ConfigMap
apiVersion: v1
metadata:
  name: alertmanager
  namespace: monitor-sa
data:
  alertmanager.yml: |-
    global:
      resolve_timeout: 1m
      smtp_smarthost: 'smtp.qq.com:465'
      smtp_from: '2830909671@qq.com'
      smtp_auth_username: '2830909671@qq.com'
      smtp_auth_password: 'ajjgpgwwfkpcdgih'
      smtp_require_tls: false
    route:
      group_by: [alertname]
      group_wait: 5s
      group_interval: 5s
      repeat_interval: 5m
      receiver: default-receiver
    receivers:
    - name: 'default-receiver'
      email_configs:
      - to: 'misakikk@qq.com'
        send_resolved: true
EOF
```

```bash
kubectl apply -f alertmanager-cm.yaml

kubectl get cm alertmanager -n monitor-sa
```

![image-20241006174637564](https://img2023.cnblogs.com/blog/3332572/202410/3332572-20241007204532972-1230037288.png)

### 1.1、alertmanager配置文件说明

```bash
smtp_smarthost: 'smtp.qq.com:465'
#用于发送邮件的邮箱的SMTP服务器地址+端口。QQ 邮箱 SMTP 服务地址，官方地址为 smtp.qq.com 端口为 465 或 587，同时要设置开启 POP3/SMTP 服务。
smtp_from: '2830909671@qq.com'
#这是指定从哪个邮箱发送报警
smtp_auth_password: 'ajjgpgwwfkpcdgih'
#这是发送邮箱的授权码而不是登录密码
email_configs:
   - to: 'misakikk@qq.com'
#to后面指定发送到哪个邮箱
```

## 2、重新生成并应用 prometheus-cfg.yaml 文件

> k8s-master1 节点操作

```yaml
cat > prometheus-cfg.yaml << 'EOF'
kind: ConfigMap
apiVersion: v1
metadata:
  labels:
    app: prometheus
  name: prometheus-config
  namespace: monitor-sa
data:
  prometheus.yml: |
    rule_files:
    - /etc/prometheus/rules.yml
    alerting:
      alertmanagers:
      - static_configs:
        - targets: ["localhost:9093"]
    global:
      scrape_interval: 15s
      scrape_timeout: 10s
      evaluation_interval: 1m
    scrape_configs:
    - job_name: 'kubernetes-node'
      kubernetes_sd_configs:
      - role: node
      relabel_configs:
      - source_labels: [__address__]
        regex: '(.*):10250'
        replacement: '${1}:9100'
        target_label: __address__
        action: replace
      - action: labelmap
        regex: __meta_kubernetes_node_label_(.+)
    - job_name: 'kubernetes-node-cadvisor'
      kubernetes_sd_configs:
      - role:  node
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      relabel_configs:
      - action: labelmap
        regex: __meta_kubernetes_node_label_(.+)
      - target_label: __address__
        replacement: kubernetes.default.svc:443
      - source_labels: [__meta_kubernetes_node_name]
        regex: (.+)
        target_label: __metrics_path__
        replacement: '/api/v1/nodes/${1}/proxy/metrics/cadvisor'
    - job_name: 'kubernetes-apiserver'
      kubernetes_sd_configs:
      - role: endpoints
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      relabel_configs:
      - source_labels: [__meta_kubernetes_namespace, __meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
        action: keep
        regex: default;kubernetes;https
    - job_name: 'kubernetes-service-endpoints'
      kubernetes_sd_configs:
      - role: endpoints
      relabel_configs:
      - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scheme]
        action: replace
        target_label: __scheme__
        regex: (https?)
      - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)
      - source_labels: [__address__, __meta_kubernetes_service_annotation_prometheus_io_port]
        action: replace
        target_label: __address__
        regex: ([^:]+)(?::\d+)?;(\d+)
        replacement: '$1:$2'
      - action: labelmap
        regex: __meta_kubernetes_service_label_(.+)
      - source_labels: [__meta_kubernetes_namespace]
        action: replace
        target_label: kubernetes_namespace
      - source_labels: [__meta_kubernetes_service_name]
        action: replace
        target_label: kubernetes_name 
    - job_name: 'kubernetes-pods'
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
        replacement: '$1:$2'
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
    - job_name: 'kubernetes-schedule'
      scrape_interval: 5s
      static_configs:
      - targets: ['192.168.112.10:10259']
    - job_name: 'kubernetes-controller-manager'
      scrape_interval: 5s
      static_configs:
      - targets: ['192.168.112.10:10257']
    - job_name: 'kubernetes-kube-proxy'
      scrape_interval: 5s
      static_configs:
      - targets: ['192.168.112.10:10249','192.168.112.20:10249','192.168.112.30:10249']
    - job_name: 'kubernetes-etcd'
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/k8s-certs/etcd/ca.crt
        cert_file: /var/run/secrets/kubernetes.io/k8s-certs/etcd/server.crt
        key_file: /var/run/secrets/kubernetes.io/k8s-certs/etcd/server.key
      scrape_interval: 5s
      static_configs:
      - targets: ['192.168.112.10:2381']
  rules.yml: |
    groups:
    - name: example
      rules:
      - alert: kube-proxy的cpu使用率大于80%
        expr: rate(process_cpu_seconds_total{job=~"kubernetes-kube-proxy"}[1m]) * 100 > 80
        for: 2s
        labels:
          severity: warnning
        annotations:
          description: "{{$labels.instance}}的{{$labels.job}}组件的cpu使用率超过80%"
      - alert:  kube-proxy的cpu使用率大于90%
        expr: rate(process_cpu_seconds_total{job=~"kubernetes-kube-proxy"}[1m]) * 100 > 90
        for: 2s
        labels:
          severity: critical
        annotations:
          description: "{{$labels.instance}}的{{$labels.job}}组件的cpu使用率超过90%"
      - alert: scheduler的cpu使用率大于80%
        expr: rate(process_cpu_seconds_total{job=~"kubernetes-schedule"}[1m]) * 100 > 80
        for: 2s
        labels:
          severity: warnning
        annotations:
          description: "{{$labels.instance}}的{{$labels.job}}组件的cpu使用率超过80%"
      - alert:  scheduler的cpu使用率大于90%
        expr: rate(process_cpu_seconds_total{job=~"kubernetes-schedule"}[1m]) * 100 > 90
        for: 2s
        labels:
          severity: critical
        annotations:
          description: "{{$labels.instance}}的{{$labels.job}}组件的cpu使用率超过90%"
      - alert: controller-manager的cpu使用率大于80%
        expr: rate(process_cpu_seconds_total{job=~"kubernetes-controller-manager"}[1m]) * 100 > 80
        for: 2s
        labels:
          severity: warnning
        annotations:
          description: "{{$labels.instance}}的{{$labels.job}}组件的cpu使用率超过80%"
      - alert:  controller-manager的cpu使用率大于90%
        expr: rate(process_cpu_seconds_total{job=~"kubernetes-controller-manager"}[1m]) * 100 > 0
        for: 2s
        labels:
          severity: critical
        annotations:
          description: "{{$labels.instance}}的{{$labels.job}}组件的cpu使用率超过90%"
      - alert: apiserver的cpu使用率大于80%
        expr: rate(process_cpu_seconds_total{job=~"kubernetes-apiserver"}[1m]) * 100 > 80
        for: 2s
        labels:
          severity: warnning
        annotations:
          description: "{{$labels.instance}}的{{$labels.job}}组件的cpu使用率超过80%"
      - alert:  apiserver的cpu使用率大于90%
        expr: rate(process_cpu_seconds_total{job=~"kubernetes-apiserver"}[1m]) * 100 > 90
        for: 2s
        labels:
          severity: critical
        annotations:
          description: "{{$labels.instance}}的{{$labels.job}}组件的cpu使用率超过90%"
      - alert: etcd的cpu使用率大于80%
        expr: rate(process_cpu_seconds_total{job=~"kubernetes-etcd"}[1m]) * 100 > 80
        for: 2s
        labels:
          severity: warnning
        annotations:
          description: "{{$labels.instance}}的{{$labels.job}}组件的cpu使用率超过80%"
      - alert:  etcd的cpu使用率大于90%
        expr: rate(process_cpu_seconds_total{job=~"kubernetes-etcd"}[1m]) * 100 > 90
        for: 2s
        labels:
          severity: critical
        annotations:
          description: "{{$labels.instance}}的{{$labels.job}}组件的cpu使用率超过90%"
      - alert: kube-state-metrics的cpu使用率大于80%
        expr: rate(process_cpu_seconds_total{k8s_app=~"kube-state-metrics"}[1m]) * 100 > 80
        for: 2s
        labels:
          severity: warnning
        annotations:
          description: "{{$labels.instance}}的{{$labels.k8s_app}}组件的cpu使用率超过80%"
          value: "{{ $value }}%"
          threshold: "80%"      
      - alert: kube-state-metrics的cpu使用率大于90%
        expr: rate(process_cpu_seconds_total{k8s_app=~"kube-state-metrics"}[1m]) * 100 > 0
        for: 2s
        labels:
          severity: critical
        annotations:
          description: "{{$labels.instance}}的{{$labels.k8s_app}}组件的cpu使用率超过90%"
          value: "{{ $value }}%"
          threshold: "90%"      
      - alert: coredns的cpu使用率大于80%
        expr: rate(process_cpu_seconds_total{k8s_app=~"kube-dns"}[1m]) * 100 > 80
        for: 2s
        labels:
          severity: warnning
        annotations:
          description: "{{$labels.instance}}的{{$labels.k8s_app}}组件的cpu使用率超过80%"
          value: "{{ $value }}%"
          threshold: "80%"      
      - alert: coredns的cpu使用率大于90%
        expr: rate(process_cpu_seconds_total{k8s_app=~"kube-dns"}[1m]) * 100 > 90
        for: 2s
        labels:
          severity: critical
        annotations:
          description: "{{$labels.instance}}的{{$labels.k8s_app}}组件的cpu使用率超过90%"
          value: "{{ $value }}%"
          threshold: "90%"      
      - alert: kube-proxy打开句柄数>600
        expr: process_open_fds{job=~"kubernetes-kube-proxy"}  > 600
        for: 2s
        labels:
          severity: warnning
        annotations:
          description: "{{$labels.instance}}的{{$labels.job}}打开句柄数>600"
          value: "{{ $value }}"
      - alert: kube-proxy打开句柄数>1000
        expr: process_open_fds{job=~"kubernetes-kube-proxy"}  > 1000
        for: 2s
        labels:
          severity: critical
        annotations:
          description: "{{$labels.instance}}的{{$labels.job}}打开句柄数>1000"
          value: "{{ $value }}"
      - alert: kubernetes-schedule打开句柄数>600
        expr: process_open_fds{job=~"kubernetes-schedule"}  > 600
        for: 2s
        labels:
          severity: warnning
        annotations:
          description: "{{$labels.instance}}的{{$labels.job}}打开句柄数>600"
          value: "{{ $value }}"
      - alert: kubernetes-schedule打开句柄数>1000
        expr: process_open_fds{job=~"kubernetes-schedule"}  > 1000
        for: 2s
        labels:
          severity: critical
        annotations:
          description: "{{$labels.instance}}的{{$labels.job}}打开句柄数>1000"
          value: "{{ $value }}"
      - alert: kubernetes-controller-manager打开句柄数>600
        expr: process_open_fds{job=~"kubernetes-controller-manager"}  > 600
        for: 2s
        labels:
          severity: warnning
        annotations:
          description: "{{$labels.instance}}的{{$labels.job}}打开句柄数>600"
          value: "{{ $value }}"
      - alert: kubernetes-controller-manager打开句柄数>1000
        expr: process_open_fds{job=~"kubernetes-controller-manager"}  > 1000
        for: 2s
        labels:
          severity: critical
        annotations:
          description: "{{$labels.instance}}的{{$labels.job}}打开句柄数>1000"
          value: "{{ $value }}"
      - alert: kubernetes-apiserver打开句柄数>600
        expr: process_open_fds{job=~"kubernetes-apiserver"}  > 600
        for: 2s
        labels:
          severity: warnning
        annotations:
          description: "{{$labels.instance}}的{{$labels.job}}打开句柄数>600"
          value: "{{ $value }}"
      - alert: kubernetes-apiserver打开句柄数>1000
        expr: process_open_fds{job=~"kubernetes-apiserver"}  > 1000
        for: 2s
        labels:
          severity: critical
        annotations:
          description: "{{$labels.instance}}的{{$labels.job}}打开句柄数>1000"
          value: "{{ $value }}"
      - alert: kubernetes-etcd打开句柄数>600
        expr: process_open_fds{job=~"kubernetes-etcd"}  > 600
        for: 2s
        labels:
          severity: warnning
        annotations:
          description: "{{$labels.instance}}的{{$labels.job}}打开句柄数>600"
          value: "{{ $value }}"
      - alert: kubernetes-etcd打开句柄数>1000
        expr: process_open_fds{job=~"kubernetes-etcd"}  > 1000
        for: 2s
        labels:
          severity: critical
        annotations:
          description: "{{$labels.instance}}的{{$labels.job}}打开句柄数>1000"
          value: "{{ $value }}"
      - alert: coredns
        expr: process_open_fds{k8s_app=~"kube-dns"}  > 600
        for: 2s
        labels:
          severity: warnning 
        annotations:
          description: "插件{{$labels.k8s_app}}({{$labels.instance}}): 打开句柄数超过600"
          value: "{{ $value }}"
      - alert: coredns
        expr: process_open_fds{k8s_app=~"kube-dns"}  > 1000
        for: 2s
        labels:
          severity: critical
        annotations:
          description: "插件{{$labels.k8s_app}}({{$labels.instance}}): 打开句柄数超过1000"
          value: "{{ $value }}"
      - alert: kube-proxy
        expr: process_virtual_memory_bytes{job=~"kubernetes-kube-proxy"}  > 2000000000
        for: 2s
        labels:
          severity: warnning
        annotations:
          description: "组件{{$labels.job}}({{$labels.instance}}): 使用虚拟内存超过2G"
          value: "{{ $value }}"
      - alert: scheduler
        expr: process_virtual_memory_bytes{job=~"kubernetes-schedule"}  > 2000000000
        for: 2s
        labels:
          severity: warnning
        annotations:
          description: "组件{{$labels.job}}({{$labels.instance}}): 使用虚拟内存超过2G"
          value: "{{ $value }}"
      - alert: kubernetes-controller-manager
        expr: process_virtual_memory_bytes{job=~"kubernetes-controller-manager"}  > 2000000000
        for: 2s
        labels:
          severity: warnning
        annotations:
          description: "组件{{$labels.job}}({{$labels.instance}}): 使用虚拟内存超过2G"
          value: "{{ $value }}"
      - alert: kubernetes-apiserver
        expr: process_virtual_memory_bytes{job=~"kubernetes-apiserver"}  > 2000000000
        for: 2s
        labels:
          severity: warnning
        annotations:
          description: "组件{{$labels.job}}({{$labels.instance}}): 使用虚拟内存超过2G"
          value: "{{ $value }}"
      - alert: kubernetes-etcd
        expr: process_virtual_memory_bytes{job=~"kubernetes-etcd"}  > 2000000000
        for: 2s
        labels:
          severity: warnning
        annotations:
          description: "组件{{$labels.job}}({{$labels.instance}}): 使用虚拟内存超过2G"
          value: "{{ $value }}"
      - alert: kube-dns
        expr: process_virtual_memory_bytes{k8s_app=~"kube-dns"}  > 2000000000
        for: 2s
        labels:
          severity: warnning
        annotations:
          description: "插件{{$labels.k8s_app}}({{$labels.instance}}): 使用虚拟内存超过2G"
          value: "{{ $value }}"
      - alert: HttpRequestsAvg
        expr: sum(rate(rest_client_requests_total{job=~"kubernetes-kube-proxy|kubernetes-kubelet|kubernetes-schedule|kubernetes-control-manager|kubernetes-apiservers"}[1m]))  > 1000
        for: 2s
        labels:
          team: admin
        annotations:
          description: "组件{{$labels.job}}({{$labels.instance}}): TPS超过1000"
          value: "{{ $value }}"
          threshold: "1000"   
      - alert: Pod_restarts
        expr: kube_pod_container_status_restarts_total{namespace=~"kube-system|default|monitor-sa"} > 0
        for: 2s
        labels:
          severity: warnning
        annotations:
          description: "在{{$labels.namespace}}名称空间下发现{{$labels.pod}}这个pod下的容器{{$labels.container}}被重启,这个监控指标是由{{$labels.instance}}采集的"
          value: "{{ $value }}"
          threshold: "0"
      - alert: Pod_waiting
        expr: kube_pod_container_status_waiting_reason{namespace=~"kube-system|default"} == 1
        for: 2s
        labels:
          team: admin
        annotations:
          description: "空间{{$labels.namespace}}({{$labels.instance}}): 发现{{$labels.pod}}下的{{$labels.container}}启动异常等待中"
          value: "{{ $value }}"
          threshold: "1"   
      - alert: Pod_terminated
        expr: kube_pod_container_status_terminated_reason{namespace=~"kube-system|default|monitor-sa"} == 1
        for: 2s
        labels:
          team: admin
        annotations:
          description: "空间{{$labels.namespace}}({{$labels.instance}}): 发现{{$labels.pod}}下的{{$labels.container}}被删除"
          value: "{{ $value }}"
          threshold: "1"
      - alert: Etcd_leader
        expr: etcd_server_has_leader{job="kubernetes-etcd"} == 0
        for: 2s
        labels:
          team: admin
        annotations:
          description: "组件{{$labels.job}}({{$labels.instance}}): 当前没有leader"
          value: "{{ $value }}"
          threshold: "0"
      - alert: Etcd_leader_changes
        expr: rate(etcd_server_leader_changes_seen_total{job="kubernetes-etcd"}[1m]) > 0
        for: 2s
        labels:
          team: admin
        annotations:
          description: "组件{{$labels.job}}({{$labels.instance}}): 当前leader已发生改变"
          value: "{{ $value }}"
          threshold: "0"
      - alert: Etcd_failed
        expr: rate(etcd_server_proposals_failed_total{job="kubernetes-etcd"}[1m]) > 0
        for: 2s
        labels:
          team: admin
        annotations:
          description: "组件{{$labels.job}}({{$labels.instance}}): 服务失败"
          value: "{{ $value }}"
          threshold: "0"
      - alert: Etcd_db_total_size
        expr: etcd_debugging_mvcc_db_total_size_in_bytes{job="kubernetes-etcd"} > 10000000000
        for: 2s
        labels:
          team: admin
        annotations:
          description: "组件{{$labels.job}}({{$labels.instance}})：db空间超过10G"
          value: "{{ $value }}"
          threshold: "10G"
      - alert: Endpoint_ready
        expr: kube_endpoint_address_not_ready{namespace=~"kube-system|default"} == 1
        for: 2s
        labels:
          team: admin
        annotations:
          description: "空间{{$labels.namespace}}({{$labels.instance}}): 发现{{$labels.endpoint}}不可用"
          value: "{{ $value }}"
          threshold: "1"
    - name: 物理节点状态-监控告警
      rules:
      - alert: 物理节点cpu使用率
        expr: 100-avg(irate(node_cpu_seconds_total{mode="idle"}[5m])) by(instance)*100 > 90
        for: 2s
        labels:
          severity: ccritical
        annotations:
          summary: "{{ $labels.instance }}cpu使用率过高"
          description: "{{ $labels.instance }}的cpu使用率超过90%,当前使用率[{{ $value }}],需要排查处理" 
      - alert: 物理节点内存使用率
        expr: (node_memory_MemTotal_bytes - (node_memory_MemFree_bytes + node_memory_Buffers_bytes + node_memory_Cached_bytes)) / node_memory_MemTotal_bytes * 100 > 90
        for: 2s
        labels:
          severity: critical
        annotations:
          summary: "{{ $labels.instance }}内存使用率过高"
          description: "{{ $labels.instance }}的内存使用率超过90%,当前使用率[{{ $value }}],需要排查处理"
      - alert: InstanceDown
        expr: up == 0
        for: 2s
        labels:
          severity: critical
        annotations:   
          summary: "{{ $labels.instance }}: 服务器宕机"
          description: "{{ $labels.instance }}: 服务器延时超过2分钟"
      - alert: 物理节点磁盘的IO性能
        expr: 100-(avg(irate(node_disk_io_time_seconds_total[1m])) by(instance)* 100) < 60
        for: 2s
        labels:
          severity: critical
        annotations:
          summary: "{{$labels.mountpoint}} 流入磁盘IO使用率过高！"
          description: "{{$labels.mountpoint }} 流入磁盘IO大于60%(目前使用:{{$value}})"
      - alert: 入网流量带宽
        expr: ((sum(rate (node_network_receive_bytes_total{device!~'tap.*|veth.*|br.*|docker.*|virbr*|lo*'}[5m])) by (instance)) / 100) > 102400
        for: 2s
        labels:
          severity: critical
        annotations:
          summary: "{{$labels.mountpoint}} 流入网络带宽过高！"
          description: "{{$labels.mountpoint }}流入网络带宽持续5分钟高于100M. RX带宽使用率{{$value}}"
      - alert: 出网流量带宽
        expr: ((sum(rate (node_network_transmit_bytes_total{device!~'tap.*|veth.*|br.*|docker.*|virbr*|lo*'}[5m])) by (instance)) / 100) > 102400
        for: 2s
        labels:
          severity: critical
        annotations:
          summary: "{{$labels.mountpoint}} 流出网络带宽过高！"
          description: "{{$labels.mountpoint }}流出网络带宽持续5分钟高于100M. RX带宽使用率{{$value}}"
      - alert: TCP会话
        expr: node_netstat_Tcp_CurrEstab > 1000
        for: 2s
        labels:
          severity: critical
        annotations:
          summary: "{{$labels.mountpoint}} TCP_ESTABLISHED过高！"
          description: "{{$labels.mountpoint }} TCP_ESTABLISHED大于1000%(目前使用:{{$value}}%)"
      - alert: 磁盘容量
        expr: 100-(node_filesystem_free_bytes{fstype=~"ext4|xfs"}/node_filesystem_size_bytes {fstype=~"ext4|xfs"}*100) > 80
        for: 2s
        labels:
          severity: critical
        annotations:
          summary: "{{$labels.mountpoint}} 磁盘分区使用率过高！"
          description: "{{$labels.mountpoint }} 磁盘分区使用大于80%(目前使用:{{$value}}%)"
EOF
```

> **注意：**
>
> 除了`kube-proxy` 默认在每个节点的 `10249` 端口上暴露其指标
>
> 其余的 `kubernetes-schedule`、`kubernetes-controller-manager`、`kubernetes-etcd` 这些组件Pod 的容器需要根据自己的 k8s 集群情况进行修改

```bash
kubectl apply -f prometheus-cfg.yaml

kubectl get cm prometheus-config -n monitor-sa -o yaml
```

> 同样的还是需要检查 cm 文件中是否正确解析了 $1 $2

![image-20241006191724613](https://img2023.cnblogs.com/blog/3332572/202410/3332572-20241007204532513-1673644801.png)

## 3、重新生成 prometheus-deploy.yaml 文件

> k8s-master1 节点操作

```yaml
cat > prometheus-deploy.yaml << EOF
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus-server
  namespace: monitor-sa
  labels:
    app: prometheus
spec:
  replicas: 2
  selector:
    matchLabels:
      app: prometheus
      component: server
    #matchExpressions:
    #- {key: app, operator: In, values: [prometheus]}
    #- {key: component, operator: In, values: [server]}
  template:
    metadata:
      labels:
        app: prometheus
        component: server
      annotations:
        prometheus.io/scrape: 'false'
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - prometheus
              - key: component
                operator: In
                values:
                - server
            topologyKey: kubernetes.io/hostname
      serviceAccountName: monitor
      containers:
      - name: prometheus
        image: quay.io/prometheus/prometheus:latest
        imagePullPolicy: IfNotPresent
        command:
        - "/bin/prometheus"
        args:
        - "--config.file=/etc/prometheus/prometheus.yml"
        - "--storage.tsdb.path=/prometheus"
        - "--storage.tsdb.retention=24h"
        - "--web.enable-lifecycle"
        ports:
        - containerPort: 9090
          protocol: TCP
        volumeMounts:
        - mountPath: /etc/prometheus
          name: prometheus-config
        - mountPath: /prometheus/
          name: prometheus-storage-volume
        - name: k8s-certs
          mountPath: /var/run/secrets/kubernetes.io/k8s-certs/etcd/
      - name: alertmanager
        image: prom/alertmanager:latest
        imagePullPolicy: IfNotPresent
        args:
        - "--config.file=/etc/alertmanager/alertmanager.yml"
        - "--log.level=debug"
        ports:
        - containerPort: 9093
          protocol: TCP
          name: alertmanager
        volumeMounts:
        - name: alertmanager-config
          mountPath: /etc/alertmanager
        - name: alertmanager-storage
          mountPath: /alertmanager
        - name: localtime
          mountPath: /etc/localtime
      volumes:
        - name: prometheus-config
          configMap:
            name: prometheus-config
        - name: prometheus-storage-volume
          hostPath:
           path: /data
           type: Directory
        - name: k8s-certs
          secret:
           secretName: etcd-certs
        - name: alertmanager-config
          configMap:
            name: alertmanager
        - name: alertmanager-storage
          hostPath:
           path: /data/alertmanager
           type: DirectoryOrCreate
        - name: localtime
          hostPath:
           path: /usr/share/zoneinfo/Asia/Shanghai
EOF
```

### 3.1、创建一个名为 etcd-certs 的 Secret

```bash
kubectl -n monitor-sa create secret generic etcd-certs --from-file=/etc/kubernetes/pki/etcd/server.key  --from-file=/etc/kubernetes/pki/etcd/server.crt --from-file=/etc/kubernetes/pki/etcd/ca.crt
```

![image-20241006194149381](https://img2023.cnblogs.com/blog/3332572/202410/3332572-20241007204532113-564999659.png)

### 3.2、应用 prometheus-deploy.yaml 文件

```
kubectl apply -f prometheus-deploy.yaml

kubectl get pods -n monitor-sa
```

![image-20241006211236948](https://img2023.cnblogs.com/blog/3332572/202410/3332572-20241007204531740-716943841.png)

## 4、重新生成并创建 alertmanager-svc.yaml 文件

```yaml
cat >alertmanager-svc.yaml <<EOF
---
apiVersion: v1
kind: Service
metadata:
  labels:
    name: prometheus
    kubernetes.io/cluster-service: 'true'
  name: alertmanager
  namespace: monitor-sa
spec:
  ports:
  - name: alertmanager
    nodePort: 30066
    port: 9093
    protocol: TCP
    targetPort: 9093
  selector:
    app: prometheus
  sessionAffinity: None
  type: NodePort
EOF
```

```bash
kubectl apply -f alertmanager-svc.yaml

kubectl get svc alertmanager -n monitor-sa
```

![image-20241006211627124](https://img2023.cnblogs.com/blog/3332572/202410/3332572-20241007204531328-2138376274.png)

## 5、访问 prometheus UI 界面

![image-20241007102819517](https://img2023.cnblogs.com/blog/3332572/202410/3332572-20241007204530972-227738002.png)

### 5.1、【error】kube-controller-manager、etcd、kube-proxy、kube-scheduler 组件 connection refused

#### 5.1.1、kube-proxy

> 默认情况下，该服务监听端口只提供给127.0.0.1，需修改为0.0.0.0

```bash
 kubectl edit cm/kube-proxy -n kube-system
```

- **编辑文件，将文件修改允许0.0.0.0即可，保存**
  metricsBindAddress: 0.0.0.0:10249

![image-20241007121231980](https://img2023.cnblogs.com/blog/3332572/202410/3332572-20241007204530613-1039497578.png)

- **删除重建 kube-proxy 的 pod**

```bash
kubectl delete pod -l k8s-app=kube-proxy -n kube-system
```

![image-20241007121419636](https://img2023.cnblogs.com/blog/3332572/202410/3332572-20241007204530167-1779632151.png)

- **效果**

![image-20241007121111763](https://img2023.cnblogs.com/blog/3332572/202410/3332572-20241007204529790-1037740550.png)

#### 5.1.2、kube-controller-manager

> 事先说明：到这一步我试过网上很多方法都没有成功获取到数据，所以我重新创建了 sa 慎用，仅供参考

- **修改 kube-controller-manager 的 yaml 文件**

默认监听本地修改为 0.0.0.0

```yaml
- --bind-address=127.0.0.1
# 修改为
- --bind-address=0.0.0.0
```

- **创建ServiceAccount**

创建一个新的ServiceAccount，用于Prometheus访问 `kube-controller-manager`。

```yaml
cat > prom-sa << EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: prometheus-sa
  namespace: monitor-sa
EOF
```

- **创建ClusterRole**

创建一个ClusterRole，定义Prometheus所需的权限。

```bash
cat > porm-role << EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: prometheus-role
rules:
- nonResourceURLs:
  - "/metrics"
  verbs:
  - get
EOF
```

- **创建ClusterRoleBinding**

将ServiceAccount绑定到ClusterRole。

```yaml
cat > prom-bind.yaml << EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: prometheus-binding
subjects:
- kind: ServiceAccount
  name: prometheus-sa
  namespace: monitor-sa
roleRef:
  kind: ClusterRole
  name: prometheus-role
  apiGroup: rbac.authorization.k8s.io
EOF
```

- **获取ServiceAccount的Token**

获取ServiceAccount的Token，以便在Prometheus配置中使用。

```bash
TOKEN=$(kubectl get secret $(kubectl get sa prometheus-sa -n monitor-sa -o json | jq -r '.secrets[].name') -n monitor-sa -o json | jq -r '.data.token' | base64 --decode)
```

- **修改Prometheus配置文件（cm）**

```yaml
- job_name: 'kubernetes-controller-manager'
      scheme: https
      tls_config:
        insecure_skip_verify: true  # 禁用证书验证
      authorization:
        credentials: eyJhbGciOiJSUzI1NiIsImtpZCI6IkFEWVNqaWlueWVDMzBUcTZvQk9MRkpxQ0diLWRGWkNoaWlpZkgwR21NcEkifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJtb25pdG9yLXNhIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZWNyZXQubmFtZSI6InByb21ldGhldXMtc2EtdG9rZW4tbnQ5bm4iLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC5uYW1lIjoicHJvbWV0aGV1cy1zYSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50LnVpZCI6IjQ4YTA5NDExLTAwMmYtNDE0Ni05YzY4LTBiNmVjOWYzYWZlZCIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDptb25pdG9yLXNhOnByb21ldGhldXMtc2EifQ.DNgCjTVxsrGDltvQZG-x7qPQrh369SO_e0faGrrhjgkBLS4q2sh85wkaBNNZcIjxZcVk7ZU9gQmQkM3AIgGIcIURpQGDMgVVI_xF1JV8iQWe-nL1yHnQAXDjyMAd1826wVvMH8LSKqdKfPVaMHN8t0LScX5yHonSJUqoevxi7Mm7tiUd33IlMQ6xH6M8Tu8bsg-fOVmL6nnGpC1tPgaZy8M_GA_Kh9j8SwHXi4Yd9r75eOSa3J6N4KF6n-EPKxnGmXDooA60G94YptsDFCQMi1t4TLAFR1FKraycWHwPbIwviUZTvA1WXbkiHnh0R6q-y0hHJVbAi_ZXagVXKZFBaw  # 替换为实际的Token值
      scrape_interval: 5s
      static_configs:
      - targets: ['192.168.112.10:10257']
```

- **重启Prometheus**

更新配置后，重启Prometheus以应用新的配置。

```yaml
kubectl rollout restart deployment/prometheus-server -n monitor-sa
```

- **效果**

![image-20241007173313086](https://img2023.cnblogs.com/blog/3332572/202410/3332572-20241007204529465-993513554.png)

#### 5.1.3、kube-schedule

> 和 kube-controller-manager 操作一致

- **效果**

![image-20241007174612964](https://img2023.cnblogs.com/blog/3332572/202410/3332572-20241007204529045-1610933165.png)

#### 5.1.4、etcd

- **修改创建 etcd 的 yaml 文件**

添加 master 节点 ip + etcd port

```yaml
vim /etc/kubernetes/manifests/etcd.yaml

- --listen-metrics-urls=http://127.0.0.1:2381,http://192.168.112.10:2381
```

![image-20241007175408503](https://img2023.cnblogs.com/blog/3332572/202410/3332572-20241007204528681-880776064.png)

- **修改 prometheus.yaml 文件**

```bash
改为 http
```

![image-20241007175613604](https://img2023.cnblogs.com/blog/3332572/202410/3332572-20241007204528262-469610873.png)

- **效果**

![image-20241007175150365](https://img2023.cnblogs.com/blog/3332572/202410/3332572-20241007204527902-194238849.png)

## 6、点击Alerts，查看

![image-20241007175925889](https://img2023.cnblogs.com/blog/3332572/202410/3332572-20241007204527489-1085781472.png)

## 7、把controller-manager的cpu使用率大于90%展开

> FIRING表示prometheus已经将告警发给alertmanager
>
> 在Alertmanager 中可以看到有 alert。

![image-20241007180133147](https://img2023.cnblogs.com/blog/3332572/202410/3332572-20241007204526931-1961258602.png)

## 8、登录 alertmanager UI

```bash
<master-ip>:svc-alertmanager-port

192.168.112.10:30066
```

![image-20241007180525364](https://img2023.cnblogs.com/blog/3332572/202410/3332572-20241007204526529-1405273581.png)

![image-20241007180341995](https://img2023.cnblogs.com/blog/3332572/202410/3332572-20241007204526134-878257958.png)

## 9、登录 QQ 邮箱查看告警信息

![image-20241007180724123](https://img2023.cnblogs.com/blog/3332572/202410/3332572-20241007204525712-663083160.png)

# 四、配置 Alertmanager 报警 -- 发送告警到钉钉

## 1、手机端拉群

因为 PC 端不好操作

![IMG_20241007_185306](https://img2023.cnblogs.com/blog/3332572/202410/3332572-20241007204525131-1336039533.jpg)

## 2、创建自定义机器人

> [自定义机器人安全设置 - 钉钉开放平台 (dingtalk.com)](https://open.dingtalk.com/document/robots/customize-robot-security-settings)

|                            群设置                            |
| :----------------------------------------------------------: |
| ![image-20241007185813640](https://img2023.cnblogs.com/blog/3332572/202410/3332572-20241007204524752-1228794490.png) |
|                          **机器人**                          |
| ![image-20241007190002110](https://img2023.cnblogs.com/blog/3332572/202410/3332572-20241007204524390-1254187804.png) |
|                        **添加机器人**                        |
| ![image-20241007190053622](https://img2023.cnblogs.com/blog/3332572/202410/3332572-20241007204523963-603631619.png) |
|                          **自定义**                          |
| ![image-20241007190125011](https://img2023.cnblogs.com/blog/3332572/202410/3332572-20241007204523379-997757021.png) |
|                           **添加**                           |
| ![image-20241007190214813](https://img2023.cnblogs.com/blog/3332572/202410/3332572-20241007204523000-199116520.png) |
|                   **机器人名字、安全设置**                   |
| ![image-20241007190915612](https://img2023.cnblogs.com/blog/3332572/202410/3332572-20241007204522616-1668641645.png) |
|                      **保管好 Webhook**                      |
| ![image-20241007191221897](https://img2023.cnblogs.com/blog/3332572/202410/3332572-20241007204522011-686507178.png) |

## 3、获取钉钉的 Webhook 插件

> master 节点操作

```bash
git clone git@github.com:misakivv/prometheus-webhook-dingtalk.git

cd prometheus-webhook-dingtalk

tar zxvf prometheus-webhook-dingtalk-0.3.0.linux-amd64.tar.gz

cd prometheus-webhook-dingtalk-0.3.0.linux-amd64
```

![image-20241007192418023](https://img2023.cnblogs.com/blog/3332572/202410/3332572-20241007204521618-1824790674.png)

## 4、启动钉钉告警插件

```bash
nohup ./prometheus-webhook-dingtalk --web.listen-address="0.0.0.0:8060" --ding.profile="cluster1=https://oapi.dingtalk.com/robot/send?access_token=feb3df2c6a987c8c1466c16eb90f4c2d3817c481aacf15cecc46f588f2716f25" &
```

![image-20241007202305558](https://img2023.cnblogs.com/blog/3332572/202410/3332572-20241007204521240-1651136825.png)

## 5、对 alertmanager-cm.yaml 文件做备份

```bash
cp alertmanager-cm.yaml alertmanager-cm.yaml.bak
```

## 6、重新生成新的 alertmanager-cm.yaml 文件

```yaml
cat >alertmanager-cm.yaml <<EOF
kind: ConfigMap
apiVersion: v1
metadata:
  name: alertmanager
  namespace: monitor-sa
data:
  alertmanager.yml: |-
    global:
      resolve_timeout: 1m
      smtp_smarthost: 'smtp.qq.com:465'
      smtp_from: '2830909671@qq.com'
      smtp_auth_username: '2830909671@qq.com'
      smtp_auth_password: 'ajjgpgwwfkpcdgih'
      smtp_require_tls: false
    route:
      group_by: [alertname]
      group_wait: 10s
      group_interval: 10s
      repeat_interval: 10m
      receiver: cluster1
    receivers:
    - name: cluster1
      webhook_configs:
      - url: 'http://192.168.112.10:8060/dingtalk/cluster1/send'
        send_resolved: true
EOF
```

## 7、重建资源以生效

```bash
kubectl delete cm alertmanager -n monitor-sa

kubectl apply -f alertmanager-cm.yaml

kubectl delete -f prometheus-cfg.yaml

kubectl apply -f prometheus-cfg.yaml

kubectl delete -f prometheus-deploy.yaml

kubectl apply -f prometheus-deploy.yaml
```

![image-20241007203234415](https://img2023.cnblogs.com/blog/3332572/202410/3332572-20241007204520868-1991701076.png)

## 8、效果

| ![image-20241007203102485](https://img2023.cnblogs.com/blog/3332572/202410/3332572-20241007204520474-234949640.png) |
| :----------------------------------------------------------: |
| ![image-20241007203427726](https://img2023.cnblogs.com/blog/3332572/202410/3332572-20241007204520040-311446757.png) |
| ![image-20241007203454338](https://img2023.cnblogs.com/blog/3332572/202410/3332572-20241007204519562-918276789.png) |
| ![image-20241007203613020](https://img2023.cnblogs.com/blog/3332572/202410/3332572-20241007204519117-39562252.png) |
| ![image-20241007203905132](https://img2023.cnblogs.com/blog/3332572/202410/3332572-20241007204518541-4065032.png) |
| ![image-20241007203933639](https://img2023.cnblogs.com/blog/3332572/202410/3332572-20241007204517472-1199044124.png) |

> 暂时先这样，其实 alertmanager 还有静默、去重、抑制等功能，下一篇再共同学习
