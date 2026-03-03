### 最终方案：纯手动创建 YAML + 部署监控（无任何外部依赖）
#### 步骤 1：创建 Grafana 部署文件（复制完整内容）
执行 `vi /root/grafana.yaml`，按 `i` 进入编辑模式，粘贴以下完整内容：
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: monitoring
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: grafana
  namespace: monitoring
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana
  namespace: monitoring
  labels:
    app: grafana
spec:
  replicas: 1
  selector:
    matchLabels:
      app: grafana
  template:
    metadata:
      labels:
        app: grafana
    spec:
      serviceAccountName: grafana
      containers:
      - name: grafana
        image: grafana/grafana:9.5.2
        ports:
        - containerPort: 3000
          protocol: TCP
        env:
        - name: GF_SECURITY_ADMIN_USER
          value: "admin"
        - name: GF_SECURITY_ADMIN_PASSWORD
          value: "admin123"
        - name: GF_USERS_ALLOW_SIGN_UP
          value: "false"
        volumeMounts:
        - name: grafana-data
          mountPath: /var/lib/grafana
        resources:
          limits:
            cpu: 500m
            memory: 512Mi
          requests:
            cpu: 100m
            memory: 128Mi
      volumes:
      - name: grafana-data
        emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: grafana
  namespace: monitoring
  labels:
    app: grafana
spec:
  type: NodePort
  ports:
  - port: 80
    targetPort: 3000
    nodePort: 30001  # 固定端口，方便访问
  selector:
    app: grafana
```
粘贴完成后，按 `Esc` → 输入 `:wq` 保存退出。

#### 步骤 2：创建 Prometheus 部署文件（复制完整内容）
执行 `vi /root/prometheus.yaml`，按 `i` 进入编辑模式，粘贴以下完整内容：
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: prometheus
  namespace: monitoring
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: prometheus
rules:
- apiGroups: [""]
  resources:
  - nodes
  - nodes/metrics
  - services
  - endpoints
  - pods
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources:
  - configmaps
  verbs: ["get"]
- apiGroups:
  - networking.k8s.io
  resources:
  - ingresses
  verbs: ["get", "list", "watch"]
- nonResourceURLs: ["/metrics"]
  verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: prometheus
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: prometheus
subjects:
- kind: ServiceAccount
  name: prometheus
  namespace: monitoring
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus
  namespace: monitoring
  labels:
    app: prometheus
spec:
  replicas: 1
  selector:
    matchLabels:
      app: prometheus
  template:
    metadata:
      labels:
        app: prometheus
    spec:
      serviceAccountName: prometheus
      containers:
      - name: prometheus
        image: prom/prometheus:v2.45.0
        args:
        - "--config.file=/etc/prometheus/prometheus.yml"
        - "--storage.tsdb.path=/prometheus/"
        ports:
        - containerPort: 9090
        volumeMounts:
        - name: prometheus-config
          mountPath: /etc/prometheus/
        - name: prometheus-storage
          mountPath: /prometheus/
        resources:
          limits:
            cpu: 500m
            memory: 512Mi
          requests:
            cpu: 100m
            memory: 128Mi
      volumes:
      - name: prometheus-config
        configMap:
          name: prometheus-config
      - name: prometheus-storage
        emptyDir: {}
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
  namespace: monitoring
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s
      evaluation_interval: 15s
    scrape_configs:
    - job_name: 'prometheus'
      static_configs:
      - targets: ['localhost:9090']
    - job_name: 'kubernetes-nodes'
      kubernetes_sd_configs:
      - role: node
      relabel_configs:
      - source_labels: [__address__]
        regex: '(.*):10250'
        replacement: '${1}:9100'
        target_label: __address__
      - action: labelmap
        regex: __meta_kubernetes_node_label_(.+)
    - job_name: 'kubernetes-pods'
      kubernetes_sd_configs:
      - role: pod
      relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)
      - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
        action: replace
        regex: ([^:]+)(?::\d+)?;(\d+)
        replacement: $1:$2
        target_label: __address__
      - action: labelmap
        regex: __meta_kubernetes_pod_label_(.+)
---
apiVersion: v1
kind: Service
metadata:
  name: prometheus
  namespace: monitoring
  labels:
    app: prometheus
spec:
  type: NodePort
  ports:
  - port: 9090
    targetPort: 9090
    nodePort: 30002  # 固定端口
  selector:
    app: prometheus
```
粘贴完成后，按 `Esc` → 输入 `:wq` 保存退出。

#### 步骤 3：部署 Node Exporter（采集节点数据）
执行 `vi /root/node-exporter.yaml`，按 `i` 进入编辑模式，粘贴以下内容：
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
  namespace: monitoring
  labels:
    app: node-exporter
spec:
  selector:
    matchLabels:
      app: node-exporter
  template:
    metadata:
      labels:
        app: node-exporter
    spec:
      hostNetwork: true
      hostPID: true
      containers:
      - name: node-exporter
        image: prom/node-exporter:v1.6.1
        args:
        - "--path.procfs=/host/proc"
        - "--path.sysfs=/host/sys"
        - "--collector.filesystem.ignored-mount-points=^/(sys|proc|dev|host|etc)($|/)"
        ports:
        - containerPort: 9100
        volumeMounts:
        - name: proc
          mountPath: /host/proc
          readOnly: true
        - name: sys
          mountPath: /host/sys
          readOnly: true
        - name: rootfs
          mountPath: /rootfs
          readOnly: true
      volumes:
      - name: proc
        hostPath:
          path: /proc
      - name: sys
        hostPath:
          path: /sys
      - name: rootfs
        hostPath:
          path: /
---
apiVersion: v1
kind: Service
metadata:
  name: node-exporter
  namespace: monitoring
  labels:
    app: node-exporter
spec:
  type: ClusterIP
  ports:
  - port: 9100
    targetPort: 9100
  selector:
    app: node-exporter
```
粘贴完成后，按 `Esc` → 输入 `:wq` 保存退出。

#### 步骤 4：一键部署所有监控组件
```bash
# 部署 Grafana
kubectl apply -f /root/grafana.yaml

# 部署 Prometheus
kubectl apply -f /root/prometheus.yaml

# 部署 Node Exporter（所有节点都会运行）
kubectl apply -f /root/node-exporter.yaml

# 查看所有监控 Pod 状态（等待 30 秒）
sleep 30 && kubectl get pods -n monitoring
```

### 步骤 5：访问监控面板
1. **Grafana 访问地址**：`http://你的节点IP:30001`（比如 192.168.225.157:30001）
2. **登录账号**：`admin`，密码：`admin123`
3. **添加 Prometheus 数据源**：
   - 登录 Grafana → 左侧「Configuration」→「Data sources」→「Add data source」→ 选择「Prometheus」
   - URL 填写：`http://prometheus.monitoring:9090` → 点击「Save & test」
4. **导入节点监控面板**：
   - 左侧「Dashboards」→「Import」→ 输入面板 ID：`1860` → 选择刚添加的 Prometheus 数据源 → 「Import」
   - 即可看到 2 主 1 从所有节点的 CPU、内存、磁盘、网络监控。

### 总结
1. **核心问题**：所有外部镜像/下载地址均失效，无法通过网络获取部署文件；
2. **解决关键**：纯手动创建完整的监控 YAML 文件，无任何外部依赖；
3. **验证标准**：
   - `kubectl get pods -n monitoring` 显示所有 Pod 为 `Running`（Node Exporter 会在 3 个节点各运行一个）；
   - 浏览器访问 `http://节点IP:30001` 能登录 Grafana，且看到所有节点的监控数据。