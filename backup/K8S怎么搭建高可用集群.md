### 一、先明确：K8s 高可用集群核心架构（3/5节点，推荐3节点）
高可用的核心是**控制平面（Master）多副本** + **etcd 集群高可用**（etcd 是 K8s 核心数据库，需至少3节点），避免单点故障。

| 节点角色       | 数量 | 硬件/系统要求（最低）| 核心组件                          |
|----------------|------|-----------------------------|-----------------------------------|
| 控制平面（Master） | 3台  | 2核4G/统信UOS/CentOS 7+      | kube-apiserver、kube-controller-manager、kube-scheduler、etcd |
| 工作节点（Node）| 2+台 | 4核8G/统信UOS/CentOS 7+      | kubelet、kube-proxy、containerd    |
| 负载均衡器（LB）| 1台  | 1核2G/任意Linux              | nginx/haproxy（转发apiserver请求） |

> 注：新手可先用 **3节点混合部署**（每台既做Master又做Node），减少机器数量，生产环境建议分离部署。

### 二、环境准备（所有节点执行）
#### 1. 系统基础配置（关闭防火墙/SELinux/swap）
```bash
# 1. 关闭防火墙
systemctl stop firewalld && systemctl disable firewalld

# 2. 关闭SELinux（统信UOS/CentOS通用）
setenforce 0
sed -i 's/^SELINUX=enforcing$/SELINUX=disabled/' /etc/selinux/config

# 3. 关闭swap（K8s要求必须关闭）
swapoff -a
sed -i '/swap/s/^/#/' /etc/fstab

# 4. 配置内核参数（开启IPVS/网桥转发）
cat > /etc/sysctl.d/k8s.conf << EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF
sysctl --system

# 5. 配置主机名和hosts（示例：3台Master节点，IP和主机名对应）
# 以Master1为例，其他节点修改hostname即可
hostnamectl set-hostname k8s-master01
cat >> /etc/hosts << EOF
192.168.1.10 k8s-master01
192.168.1.11 k8s-master02
192.168.1.12 k8s-master03
192.168.1.9  k8s-lb  # 负载均衡器IP
EOF

# 6. 时间同步（高可用核心，所有节点时间必须一致）
yum install -y chrony
systemctl start chronyd && systemctl enable chronyd
chronyc sync
```

#### 2. 安装容器运行时（containerd，替代docker）
```bash
# 1. 安装依赖
yum install -y yum-utils device-mapper-persistent-data lvm2

# 2. 添加阿里云镜像源
yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

# 3. 安装containerd
yum install -y containerd.io

# 4. 配置containerd（生成默认配置并修改）
containerd config default > /etc/containerd/config.toml
# 修改sandbox镜像为国内源（关键，避免拉取失败）
sed -i 's#k8s.gcr.io/pause#registry.aliyuncs.com/google_containers/pause#g' /etc/containerd/config.toml
# 启用SystemdCgroup（K8s要求）
sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml

# 5. 重启并开机自启
systemctl restart containerd && systemctl enable containerd
```

#### 3. 安装kubeadm/kubelet/kubectl（K8s核心工具）
```bash
# 1. 添加阿里云K8s源
cat > /etc/yum.repos.d/kubernetes.repo << EOF
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF

# 2. 安装指定版本（推荐1.28.x，稳定版）
yum install -y kubelet-1.28.3 kubeadm-1.28.3 kubectl-1.28.3

# 3. 开机自启kubelet（暂不启动，初始化后自动启动）
systemctl enable kubelet
```

### 三、搭建负载均衡器（LB节点）
用于转发所有对 apiserver 的请求到3个Master节点，避免单Master入口故障。这里用 **haproxy** 为例：
```bash
# 1. 安装haproxy
yum install -y haproxy

# 2. 修改配置文件 /etc/haproxy/haproxy.cfg
cat > /etc/haproxy/haproxy.cfg << EOF
global
    log         127.0.0.1 local2
    chroot      /var/lib/haproxy
    pidfile     /var/run/haproxy.pid
    maxconn     4000
    user        haproxy
    group       haproxy
    daemon

defaults
    mode                    http
    log                     global
    option                  httplog
    option                  dontlognull
    option http-server-close
    option forwardfor       except 127.0.0.0/8
    option                  redispatch
    retries                 3
    timeout http-request    10s
    timeout queue           1m
    timeout connect         10s
    timeout client          1m
    timeout server          1m
    timeout http-keep-alive 10s
    timeout check           10s
    maxconn                 3000

# 核心配置：转发8443端口（apiserver端口）到3个Master节点
frontend k8s-apiserver
    bind *:8443
    mode tcp
    option tcplog
    default_backend k8s-apiserver-backend

backend k8s-apiserver-backend
    mode tcp
    option tcp-check
    balance roundrobin
    server k8s-master01 192.168.1.10:6443 check inter 2000 fall 2 rise 2
    server k8s-master02 192.168.1.11:6443 check inter 2000 fall 2 rise 2
    server k8s-master03 192.168.1.12:6443 check inter 2000 fall 2 rise 2
EOF

# 3. 启动并开机自启
systemctl start haproxy && systemctl enable haproxy
# 验证端口监听
ss -tnlp | grep 8443
```

### 四、初始化第一个Master节点（k8s-master01）
```bash
# 1. 初始化集群（指定LB地址、pod网段、service网段）
kubeadm init \
  --control-plane-endpoint "k8s-lb:8443" \  # LB的地址和端口
  --image-repository registry.aliyuncs.com/google_containers \  # 国内镜像源
  --kubernetes-version v1.28.3 \
  --pod-network-cidr=10.244.0.0/16 \  # flannel网络网段
  --service-cidr=10.96.0.0/12 \
  --upload-certs  # 生成证书，用于其他Master节点加入

# 2. 配置kubectl（普通用户权限，可选）
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config

# 3. 安装网络插件（flannel，必须安装，否则Pod不通）
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/v0.22.0/Documentation/kube-flannel.yml
# 国内源替代（避免访问失败）
kubectl apply -f https://mirror.ghproxy.com/https://raw.githubusercontent.com/flannel-io/flannel/v0.22.0/Documentation/kube-flannel.yml

# 4. 记录加入命令（关键！其他Master/Node节点需要）
# 初始化成功后会输出：
# 1) 加入Master节点的命令（含--control-plane和证书）
# 2) 加入Node节点的命令
# 示例：
# kubeadm join k8s-lb:8443 --token xxx --discovery-token-ca-cert-hash sha256:xxx --control-plane --certificate-key xxx
# kubeadm join k8s-lb:8443 --token xxx --discovery-token-ca-cert-hash sha256:xxx
```

### 五、加入其他Master节点（k8s-master02/03）
在master02和master03上执行第一步初始化成功后输出的**Master加入命令**：
```bash
# 示例命令（替换为你自己的token和certificate-key）
kubeadm join k8s-lb:8443 \
  --token abcdef.0123456789abcdef \
  --discovery-token-ca-cert-hash sha256:1234567890abcdef1234567890abcdef1234567890abcdef1234567890abcdef \
  --control-plane \
  --certificate-key 1234567890abcdef1234567890abcdef1234567890abcdef1234567890abcdef
```

### 六、加入Node节点（工作节点）
在所有Node节点上执行初始化成功后输出的**Node加入命令**：
```bash
# 示例命令
kubeadm join k8s-lb:8443 \
  --token abcdef.0123456789abcdef \
  --discovery-token-ca-cert-hash sha256:1234567890abcdef1234567890abcdef1234567890abcdef1234567890abcdef
```

### 七、验证高可用集群
在任意Master节点执行以下命令，确认集群状态：
```bash
# 1. 查看节点状态（所有节点应为Ready）
kubectl get nodes
# 输出示例：
# NAME           STATUS   ROLES           AGE   VERSION
# k8s-master01   Ready    control-plane   10m   v1.28.3
# k8s-master02   Ready    control-plane   5m    v1.28.3
# k8s-master03   Ready    control-plane   3m    v1.28.3
# k8s-node01     Ready    <none>          1m    v1.28.3

# 2. 查看控制平面组件（所有pod应为Running）
kubectl get pods -n kube-system
# 重点看：kube-apiserver、kube-controller-manager、kube-scheduler 各3个副本，etcd 3个副本

# 3. 验证高可用：停止一个Master的apiserver，集群仍可用
systemctl stop kubelet containerd -n k8s-master01
# 再次执行kubectl get nodes，集群仍能正常响应，说明高可用生效
```

### 八、运维关键注意事项
1. **证书有效期**：K8s证书默认1年，需定期更新（`kubeadm certs renew all`）；
2. **etcd备份**：高可用核心是etcd，需定时备份（`etcdctl snapshot save`）；
3. **LB高可用**：生产环境LB也需做双机热备（keepalived），避免LB单点故障；
4. **统信UOS适配**：若用统信UOS，需替换yum源为统信官方源，containerd配置一致；
5. **网络插件**：除了flannel，也可选用calico（更适合大规模集群）。

### 总结
1. K8s高可用集群核心是**3节点控制平面+etcd集群+LB负载均衡**，避免单点故障；
2. 搭建流程：环境准备 → LB部署 → 第一个Master初始化 → 其他Master/Node加入 → 验证；
3. 运维重点关注**证书更新、etcd备份、LB高可用**，确保集群长期稳定运行。