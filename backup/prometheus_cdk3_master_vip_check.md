# Master节点VIP漂移告警排查与处理手册

本文档旨在指导运维人员排查因Master节点VIP漂移导致的告警，并提供标准的恢复流程。

## 一、告警详细信息输出

1. 登录工单管理系统，点击 **工单管理**。
2. 在下拉列表中选择 **告警名称**，查看并输出告警详细信息。
3. 从告警详情中定位报错节点的IP地址，记录为 `$.{IP}`。

## 二、Master VIP地址MAC地址确认

1. **登录节点**：使用 `$.{IP}` 登录对应节点。
2. **获取VIP**：执行以下命令，查找当前对应的VIP地址：
   ```bash
   ps -ef | grep kube-proxy
   ```
3. **获取MAC地址**：在CDK页面或对应环境中执行以下命令，查找HWaddress对应的MAC地址：
   ```bash
   arp -n | grep $.{masterVIP}
   ```
4. **初步确认**：对比查找到的MAC地址与当前集群记录的MAC地址是否一致。

## 三、登录Master节点

### 1. 通过告警信息直接定位
通过告警详情中的 `component` 和 `region` 字段信息进行定位：
- `component` 字段通常包含挂载集群的Node节点信息，去掉后缀 `node` 即为集群名称。
- 当前集群名称记为 `$.{cluster}`。
- 当前集群(K8s)版本记为 `$.{k8s_version}`。

### 2. 通过CDAS平台间接定位
若上述信息无法直接找到集群，请登录 **CDAS平台** 进行查询：

1. **进入视图**：点击 **操作平台** -> **操作管理**，点击 **region视图**，选择当前的 `$.{region}`。
2. **选择服务**：点击 **CloudCDK** -> **CloudAutoDeploy-CDK** (或 CloudAutoDeploy-CDK-CFEDB-GaussDB)，选择对应的 `$.{schema}` 信息。
3. **执行SQL查询**：
   - 查询节点信息：
     ```sql
     select * from "node" where "private_ip" = '$.{告警nodeIP}'
     ```
   - 查询集群信息：
     ```sql
     select "uid","name","region" from "cluster" where "uid" = '$.{node表查找到的cluster_id}'
     ```
4. **登录Master**：根据查询结果中的 `region` 和 `name` 找到对应的集群，并登录该集群的3台Master节点。

## 四、查找Master节点的VIP地址对应的MAC地址

1. **保存MAC地址**：记录第二步中获取的MAC地址，记为 `$.{mac}`。
2. **定位漂移节点**：登录Master节点，执行以下命令，查找VIP漂移所在的Master节点：
   ```bash
   ip a
   ```
   *备注：如果 `eth0` 对应的地址有两个IP，说明当前节点持有VIP地址。*
3. **验证MAC地址归属**：执行以下命令统计匹配次数：
   ```bash
   ip a | grep $.{mac} | wc -l
   ```
   - **输出为1**：说明MAC地址正常归属当前节点，此时若仍存在问题，请联系 **CloudCDK当天的主OnCall** 进行排查。
   - **输出非1**：说明MAC地址漂移或异常，需要重启当前Master节点的Keepalived服务。

## 五、Keepalived重启操作

根据集群版本的不同，执行相应的重启操作。

### 1. 1.15集群版本

1. **获取容器ID**：在Master节点执行以下命令，找到Keepalived容器的ID：
   ```bash
   docker ps | grep keepalived
   ```
2. **登录CAC平台**：进入 **脚本管理**，搜索脚本 `sre_restart_container`，点击 **执行**。
3. **输入参数并提交**：
   - 参数1: `$.{containerID}` (上一步获取的ID)
   - 参数2: `$.{action}` (值为 restart)
   - 点击 **提交**，等待重启成功。

### 2. 1.9集群版本

1. **登录CAC节点**：执行以下脚本命令：
   ```bash
   sre_restart_node_component
   ```
2. **输入组件名**：在提示中输入 `keepalived`。
3. **确认执行**：执行成功后，Keepalived服务将重启。

## 六、确认MAC地址是否一致

完成重启操作后，需进行验证以确保问题已解决：

1. **再次确认MAC地址**：重复执行 **第二步**，确认Node节点的MAC地址信息。
2. **再次验证漂移情况**：重复执行 **第四步第3小步**，确认VIP漂移的Master节点地址和输出结果：
   ```bash
   ip a | grep $.{mac} | wc -l
   ```
   - **结果为1**：说明MAC地址已恢复正常归属，告警应消除。
   - **结果非1**：说明问题未解决，请立即联系 **CloudCDK OnCall** 进行进一步查看和处理。