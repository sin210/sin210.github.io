# 一、架构（3 Master 高可用）
- **k8s-master-1：192.168.1.10**
- **k8s-master-2：192.168.1.11**
- **k8s-master-3：192.168.1.12**
- （可选）Worker 节点随便加

**前提：**
- 3 台都是 Master
- 必须有一个 **VIP/负载均衡（LB）** 地址：`192.168.1.100`（用来统一接入 apiserver）

---

# 二、所有节点统一初始化（3 Master + 所有 Worker 都执行）
```bash
# 1. 关闭防火墙
systemctl stop firewalld
systemctl disable firewalld

# 2. 关闭 SELinux
setenforce 0
sed -i 's/^SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config

# 3. 关闭 swap（必须关）
swapoff -a
sed -i '/swap/s/^/#/' /etc/fstab

# 4. 内核参数（桥接、流量转发）
cat <<EOF > /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
sysctl --system

# 5. 安装 docker/containerd（这里用 containerd）
yum install -y containerd.io
systemctl enable --now containerd

# 6. 添加 k8s 源
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF

# 7. 安装 kubeadm kubelet kubectl
yum install -y kubelet kubeadm kubectl
systemctl enable --now kubelet
```

---

# 三、在 **第一台 Master（k8s-master-1）** 执行：初始化集群
```bash
kubeadm init \
  --apiserver-advertise-address=192.168.1.10 \
  --control-plane-endpoint=192.168.1.100 \  # 这里是 LB/VIP
  --kubernetes-version=v1.28.2 \
  --pod-network-cidr=10.244.0.0/16 \
  --upload-certs
```

执行完会输出**两条关键命令**，你要保存好：
1. **加入 Master 节点的命令**（带 --control-plane）
2. **加入 Worker 节点的命令**

---

# 四、配置 kubectl（所有 Master 都要做）
```bash
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
```

---

# 五、安装网络插件（Calico/Flannel）
```bash
kubectl apply -f https://docs.projectcalico.org/v3.26/manifests/calico.yaml
```

---

# 六、把 **Master2、Master3** 加入集群（高可用）
在 **master2、master3** 上执行 init 生成的 **join 命令**，类似：
```bash
kubeadm join 192.168.1.100:6443 \
  --token xxx \
  --discovery-token-ca-cert-hash sha256:xxx \
  --control-plane \
  --certificate-key xxx
```
只要带 `--control-plane`，就是**加入 Master 节点**。

---

# 七、验证 3 Master 高可用集群
```bash
kubectl get nodes
```
你会看到：
```
k8s-master-1   Ready    control-plane,master
k8s-master-2   Ready    control-plane,master
k8s-master-3   Ready    control-plane,master
```

查看 etcd 集群状态：
```bash
kubectl exec -n kube-system etcd-k8s-master-1 -- etcdctl endpoint status --cluster
```

---

# 八、面试必背 4 句话（直接背）
1. **3 Master K8s 高可用 = 3 个控制平面节点 + 3 节点 etcd 集群 + 前端负载均衡**。
2. **kubeadm init 必须用 --control-plane-endpoint 指定 LB 地址**。
3. **新 Master 加入要加 --control-plane 参数**。
4. **只要 > 一半 Master 存活，集群就可用，3 台允许挂 1 台**。

---

我给你一套**阿里云ECS + 3 Master 高可用K8s集群**的**最简可直接照做流程**，专门适配阿里云环境，**面试/实操都能用**。

---

# 一、阿里云 3 Master 高可用 K8s 架构（必懂）
- 3 台 ECS（都做 Master）
- 阿里云 **SLB（负载均衡）** 转发 6443 端口给 3 台 Master
- 3 台 Master 自带 3 节点 etcd 集群
- 高可用：挂任意 1 台 Master，集群正常

---

# 二、阿里云前期准备（一步都不能少）
## 1. 买 3 台 ECS（配置建议）
- 系统：CentOS 7.9 / Ubuntu 20.04
- 配置：2核4G 以上
- 网络：**同一VPC、同一可用区**
- 安全组 **必须开放**（入方向）：
  - 6443（kube-apiserver）
  - 2379/2380（etcd）
  - 10250、10257、10259（kubelet、controller-manager）
  - 443、80、8080 等业务端口

## 2. 创建阿里云 SLB（关键！高可用必须）
1. 控制台 → 负载均衡 SLB → 创建 **内网SLB**
2. 监听 **TCP:6443**
3. 后端服务器：添加 3 台 Master ECS，端口 **6443**
4. 健康检查：TCP 6443
5. 拿到 SLB 内网地址：
   示例：`k8s-lb-xxxxxxx.cn-beijing.alb.aliyuncs.com`
   或内网IP：`192.168.x.x`

> 这个 SLB 就是你的 `control-plane-endpoint`

---

# 三、所有3台Master节点执行：初始化环境
```bash
# 1. 关闭防火墙
systemctl stop firewalld
systemctl disable firewalld

# 2. 关闭selinux
setenforce 0
sed -i 's/^SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config

# 3. 关闭swap（必须）
swapoff -a
sed -i '/swap/s/^/#/' /etc/fstab

# 4. 桥接网络
modprobe br_netfilter
cat <<EOF > /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
net.ipv4.ip_forward=1
EOF
sysctl --system
```

## 安装 containerd（阿里云推荐）
```bash
yum install -y yum-utils device-mapper-persistent-data lvm2
yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
yum install -y containerd.io

containerd config default > /etc/containerd/config.toml

# 修改镜像源为阿里云（必须）
sed -i 's|sandbox_image.*|sandbox_image = "registry.aliyuncs.com/google_containers/pause:3.9"|' /etc/containerd/config.toml

systemctl enable --now containerd
```

## 安装 kubeadm、kubelet、kubectl（阿里云源）
```bash
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF

yum install -y kubelet kubeadm kubectl
systemctl enable --now kubelet
```

---

# 四、在 Master-1 初始化高可用集群（核心命令）
把 **SLB地址** 替换成你的：
```bash
kubeadm init \
  --apiserver-advertise-address=0.0.0.0 \
  --control-plane-endpoint=你的SLB地址:6443 \
  --kubernetes-version=v1.29.0 \
  --pod-network-cidr=10.244.0.0/16 \
  --image-repository=registry.aliyuncs.com/google_containers \
  --upload-certs
```

执行后会输出 **两条命令**，你必须保存：
1. **添加 Master 节点的命令**（带 --control-plane）
2. **添加 Worker 节点的命令**

---

# 五、配置 kubectl（3台Master都执行）
```bash
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
```

---

# 六、安装网络插件 Calico
```bash
kubectl apply -f https://docs.projectcalico.org/v3.27/manifests/calico.yaml
```

---

# 七、把 Master2、Master3 加入集群
在 **Master2、Master3** 执行上面生成的 **join 命令**，示例：
```bash
kubeadm join SLB地址:6443 \
  --token xxx \
  --discovery-token-ca-cert-hash sha256:xxx \
  --control-plane \
  --certificate-key xxx
```

---

# 八、验证集群
```bash
kubectl get nodes

# 看到 3 个 Ready control-plane 就成功
```

查看 etcd 集群：
```bash
kubectl exec -n kube-system etcd-节点名 -- etcdctl endpoint status --cluster
```

---

# 九、阿里云特别注意（面试必说）
1. **必须用 SLB 做 apiserver 负载均衡**
2. **安全组必须放开6443、2379、2380**
3. **镜像仓库必须用阿里云：registry.aliyuncs.com/google_containers**
4. 3 Master 高可用，**允许挂1台**
5. etcd 自动是 3 节点集群，无需手动搭建

---

# 十、你只要背这 4 句（面试满分）
1. 阿里云 3 Master 高可用 K8s = **3台ECS + SLB负载均衡 + 3节点etcd**。
2. 初始化用 `--control-plane-endpoint` 指向SLB。
3. 镜像仓库必须换成阿里云。
4. 3 Master 挂 1 台仍正常工作。

---

我给你一份**面试能直接说、特别清晰**的  
**SLB（阿里云负载均衡） vs HAProxy 功能对比**，不绕弯、直接干货。

---

# 一句话先记住
- **SLB = 阿里云官方云产品负载均衡**（托管、不用管运维）
- **HAProxy = 开源软件负载均衡**（自己装、自己运维、自己扛高可用）

---

# 一、核心功能对比（最清晰版）

## 1. 身份与部署
- **SLB**
  - 阿里云**托管服务**
  - 点开控制台创建，几分钟可用
  - 不用自己装、不用管机器、不用管宕机

- **HAProxy**
  - **开源软件**
  - 必须自己买 ECS，自己安装、配置、启动
  - 要自己做高可用（keepalived 双机热备）

## 2. 高可用谁来做
- **SLB**：阿里云自带集群高可用，**你完全不用管**
- **HAProxy**：你自己做高可用，至少 **2 台 ECS + keepalived**

## 3. 支持协议
两者基本一样：
- TCP（4层）
- HTTP/HTTPS（7层）
- SSL 证书卸载

## 4. 负载均衡策略
- **SLB**
  - 轮询
  - 加权轮询
  - 最小连接
  - IP 哈希
- **HAProxy**
  - 上面所有都支持
  - 还支持更多：URL 哈希、Header 路由、动态权重、复杂规则

## 5. 健康检查
- **SLB**：TCP/HTTP/HTTPS 检查，简单配置，阿里云托管
- **HAProxy**：更灵活、自定义检查脚本、自定义判断逻辑

## 6. 会话保持
- **SLB**：支持
- **HAProxy**：支持，且更灵活（cookie、源IP等）

## 7. 性能与并发
- **SLB**：阿里云弹性扩容，扛不住自动扩，不用你管
- **HAProxy**：性能很强，但受限于你 ECS 配置，要自己监控扩容

## 8. 费用
- **SLB**：实例费 + 流量费
- **HAProxy**：只收你 ECS 费，软件免费

## 9. 运维成本（重点！）
- **SLB**：极低，配置完就不用管
- **HAProxy**：高，要管配置、重启、日志、高可用、性能调优

---

# 二、最简单总结（面试直接背）
1. **SLB 是云厂商托管负载均衡，省心、高可用自带、不用运维。**
2. **HAProxy 是开源软件，功能更强更灵活，但要自己装、自己维护高可用。**
3. 阿里云上 **K8s 3 Master 高可用，推荐用 SLB，稳定、简单、不出坑**。
4. 自建机房/不想用云产品 → 用 HAProxy。

---

# 三、你在阿里云搭 3 Master K8s 应该怎么选？
**标准答案：用 SLB**
- 不用自己搭 HAProxy 双机
- 不用配置 keepalived
- 不用担心中间层挂掉
- 阿里云保证 SLB 高可用
- `kubeadm init` 直接指向 SLB 地址即可