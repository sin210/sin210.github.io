---

# Linux运维核心知识体系（从基础到进阶）
## 一、磁盘管理与LVM
### 1. 磁盘分区
磁盘分区是将物理硬盘划分为独立的逻辑区域，Linux中核心工具及适用场景：
- **fdisk**：MBR分区格式，支持2TB以下磁盘
- **parted**：GPT分区格式，支持大容量磁盘

#### 常用操作
```bash
# 查看磁盘整体情况
fdisk -l

# 进入分区交互操作（以/dev/sdb为例）
fdisk /dev/sdb
# 交互流程：
# n → 新建分区
# p → 选择主分区（扩展分区选e）
# 1 → 分区编号（自定义）
# 回车 → 使用默认起始扇区
# 回车 → 使用默认结束扇区（或指定大小如+100G）
# w → 保存分区表并退出
```

### 2. LVM逻辑卷管理
LVM（Logical Volume Manager）提供动态调整卷大小的灵活磁盘管理方案，核心概念：
| 组件 | 说明 |
|------|------|
| PV（物理卷） | 物理磁盘/分区（如/dev/sdb1） |
| VG（卷组） | 多个PV的集合，形成统一存储池 |
| LV（逻辑卷） | 从VG中划分的可动态调整的逻辑存储单元 |

#### 核心操作流程
```bash
# 1. 创建物理卷(PV)
pvcreate /dev/sdb1

# 2. 创建卷组(VG)，命名为vg01
vgcreate vg01 /dev/sdb1

# 3. 创建逻辑卷(LV)，大小10G，命名为lv01
lvcreate -L 10G -n lv01 vg01

# 4. 格式化并挂载LV
mkfs.ext4 /dev/vg01/lv01
mount /dev/vg01/lv01 /data

# 5. 扩容LV（增加5G）
lvextend -L +5G /dev/vg01/lv01  # 调整逻辑卷大小
resize2fs /dev/vg01/lv01        # 同步文件系统大小
```

## 二、网络管理
### 1. iptables filter表核心链区别
filter表是iptables最常用表，核心链功能：
- **INPUT**：处理进入本机的数据包
- **OUTPUT**：处理从本机发出的数据包
- **FORWARD**：处理经过本机转发的数据包（本机作为路由器时生效）

#### FORWARD链典型配置
```bash
# 启用IP转发（临时生效）
echo 1 > /proc/sys/net/ipv4/ip_forward

# 配置转发规则
iptables -A FORWARD -i eth0 -o eth1 -j ACCEPT  # 允许eth0→eth1转发
iptables -A FORWARD -i eth1 -o eth0 -m state --state ESTABLISHED,RELATED -j ACCEPT  # 允许回应包
```

### 2. IP与网关配置
#### 临时配置（重启网络失效）
```bash
# 添加IP地址
ip addr add 192.168.1.100/24 dev eth0

# 添加默认网关
ip route add default via 192.168.1.1
```

#### 永久配置（CentOS/RHEL）
编辑网卡配置文件 `/etc/sysconfig/network-scripts/ifcfg-eth0`：
```ini
BOOTPROTO=static       # 静态IP（dhcp为动态）
IPADDR=192.168.1.100   # 本机IP
NETMASK=255.255.255.0  # 子网掩码
GATEWAY=192.168.1.1    # 默认网关
ONBOOT=yes             # 开机自启
```

## 三、RAID技术对比
| RAID级别 | 优势 | 劣势 | 磁盘利用率 | 适用场景 |
|----------|------|------|------------|----------|
| RAID 0 | 读写速度最快，容量最大化 | 无冗余，单盘故障数据全丢 | 100% | 临时数据、缓存、非重要数据 |
| RAID 1 | 完全冗余，读取性能好 | 写入性能下降，成本高 | 50% | 系统盘、重要数据存储 |
| RAID 5 | 平衡性能和冗余，成本适中 | 写入性能一般，故障重建慢 | (n-1)/n（n为磁盘数） | 文件服务器、通用数据存储 |

#### RAID 5创建示例
```bash
# 创建RAID 5（3块磁盘）
mdadm --create /dev/md0 --level=5 --raid-devices=3 /dev/sdb /dev/sdc /dev/sdd

# 格式化并挂载
mkfs.ext4 /dev/md0
mount /dev/md0 /raid
```

## 四、OSI七层模型
| 层级 | 名称 | 核心协议/技术 |
|------|------|---------------|
| 7 | 应用层 | HTTP、FTP、SSH、DNS |
| 6 | 表示层 | SSL/TLS、JPEG、MPEG |
| 5 | 会话层 | RPC、NetBIOS |
| 4 | 传输层 | TCP、UDP |
| 3 | 网络层 | IP、ICMP、ARP |
| 2 | 数据链路层 | Ethernet、MAC地址 |
| 1 | 物理层 | 网线、光纤、电信号 |

## 五、系统优化要点
### 1. 内核参数优化
编辑 `/etc/sysctl.conf`，添加/修改以下关键参数：
```ini
# 重用TIME-WAIT状态的TCP连接
net.ipv4.tcp_tw_reuse = 1
# 扩大本地端口范围（解决端口耗尽）
net.ipv4.ip_local_port_range = 10000 65000
# 降低swap使用优先级（优先使用物理内存）
vm.swappiness = 10
# 增加系统最大文件描述符数
fs.file-max = 65535
```
生效配置：`sysctl -p`

### 2. 系统资源限制
编辑 `/etc/security/limits.conf`：
```ini
* soft nofile 65535  # 软限制：最大打开文件数
* hard nofile 65535  # 硬限制：最大打开文件数
* soft nproc 65535   # 软限制：最大进程数
* hard nproc 65535   # 硬限制：最大进程数
```

## 六、Dockerfile指令对比
| 指令 | 功能 | 特点 |
|------|------|------|
| COPY | 复制文件到镜像 | 简单直接，仅支持本地文件/目录 |
| ADD | 复制文件到镜像 | 支持URL下载、自动解压tar文件 |

#### 使用建议
- 普通文件复制优先用 `COPY`（更清晰、稳定）
- 需要解压压缩包/下载远程文件时用 `ADD`

## 七、虚拟机快照管理（KVM/QEMU）
```bash
# 创建快照
virsh snapshot-create-as vm_name snapshot_name

# 恢复快照
virsh snapshot-revert vm_name snapshot_name

# 查看快照列表
virsh snapshot-list vm_name

# 删除快照
virsh snapshot-delete vm_name snapshot_name

# 基础虚拟机管理
virsh start vm_name      # 开机
virsh shutdown vm_name   # 正常关机
virsh destroy vm_name    # 强制关机（类似拔电源）
```

## 八、Linux常用命令
### 1. 文件操作
```bash
ls -lah                  # 以人类可读格式详细列出所有文件
cd /path/to/directory   # 切换目录
pwd                      # 显示当前工作路径
cp -r source dest       # 递归复制目录/文件
mv source dest          # 移动/重命名文件
rm -rf file              # 强制递归删除（慎用）
find /path -name "*.log" # 按名称查找文件
```

### 2. 系统监控
```bash
top                      # 实时进程/资源监控
htop                     # 增强版top（需安装）
ps aux                   # 查看所有进程详情
df -h                    # 磁盘使用情况（人类可读）
free -h                  # 内存/交换分区使用情况
netstat -tlnp           # 查看监听的TCP端口及对应进程
```

### 3. 文本处理
```bash
grep "pattern" file      # 搜索文本内容
awk '{print $1}' file    # 提取文本第1列
sed 's/old/new/g' file   # 全局替换文本
cat file | head -n 10    # 查看文件前10行
```

## 九、Shell巡检脚本
```bash
#!/bin/bash
# 系统状态巡检脚本

echo "=== 系统巡检报告 $(date) ==="

# 1. CPU使用率
echo -e "\n1. CPU使用率："
top -bn1 | grep "Cpu(s)" | awk '{print "CPU使用率: " $2 "%"}'

# 2. 内存使用情况
echo -e "\n2. 内存使用情况："
free -h | awk '/Mem/{printf "总内存: %s, 已用: %s, 使用率: %.2f%%\n", $2,$3,$3/$2*100}'

# 3. 磁盘使用情况（根分区）
echo -e "\n3. 磁盘使用情况："
df -h | awk '$NF=="/"{printf "根分区: 总量 %s, 已用 %s, 使用率 %s\n", $2,$3,$5}'

# 4. IO性能
echo -e "\n4. IO性能："
iostat -x 1 2 | tail -n +4 | awk 'NR==1{print "设备", "利用率", "等待时间"} NR>1{print $1, $14"%", $10"ms"}'

# 5. 网络连接数（按状态统计）
echo -e "\n5. 网络连接数："
netstat -an | awk '/^tcp/{++S[$NF]} END {for(a in S) print a, S[a]}'
```

## 十、批量管理50台服务器
### 1. 批量修改主机名
#### 方法1：SSH循环脚本
```bash
#!/bin/bash
# 服务器列表
servers=(server1 server2 server3 ... server50)
new_hostname_prefix="web-server"

# 循环修改主机名
for i in "${!servers[@]}"; do
    ssh root@${servers[$i]} "hostnamectl set-hostname ${new_hostname_prefix}-$(($i+1))"
done
```

#### 方法2：Ansible Playbook（推荐）
```yaml
---
- hosts: all
  become: yes  # 提权执行
  tasks:
    - name: 批量设置主机名
      hostname:
        name: "web-server-{{ inventory_hostname.split('-')[-1] }}"
```

### 2. SSH免密配置（管理机执行）
```bash
# 1. 生成密钥对（无密码）
ssh-keygen -t rsa -P '' -f ~/.ssh/id_rsa

# 2. 分发公钥到所有服务器（需输入密码）
for server in server1 server2 ... server50; do
    ssh-copy-id root@$server
done

# 3. 验证免密登录
ssh root@server1 "hostname"
```

### 3. 批量部署脚本（读取服务器列表）
```bash
#!/bin/bash
# 从server_list.txt读取服务器列表
while read server; do
    echo "正在配置 $server ..."
    # 免密分发公钥（自动输入密码，需安装sshpass）
    sshpass -p "your_password" ssh-copy-id -o StrictHostKeyChecking=no root@$server
done < server_list.txt
```

---

## 学习建议
1. **循序渐进**：从基础命令入手，掌握后再学习服务配置、自动化工具
2. **实验环境**：用虚拟机（VMware/KVM）搭建测试环境，避免直接操作生产机
3. **文档记录**：整理个人操作手册，记录常用命令、配置和排错经验
4. **自动化思维**：优先掌握Shell脚本、Ansible等工具，提升运维效率

### 总结
1. Linux运维核心围绕**存储（磁盘/LVM/RAID）、网络（iptables/IP配置）、系统优化、自动化管理**四大核心方向展开；
2. 实操是掌握Linux的关键，需结合实验环境反复练习命令和配置；
3. 批量管理场景优先使用Ansible等自动化工具，替代手动SSH循环，提升效率和稳定性。