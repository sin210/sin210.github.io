Pod 是 Kubernetes（K8s）中最小的部署单元，其生命周期是从**创建到终止**的完整过程，核心包含「阶段划分」「关键钩子」「终止流程」三大核心部分，以下是适合笔记的精简版总结：

## 一、核心阶段（Phase）
Pod 的生命周期会表现为不同阶段，仅反映宏观状态，不代表容器具体状态：
| 阶段         | 含义                                                                 |
|--------------|----------------------------------------------------------------------|
| Pending      | Pod 已被 K8s 接受，但容器尚未创建（或正在创建），如拉取镜像、调度中   |
| Running      | Pod 已调度到节点，所有容器已创建，至少一个容器处于运行/启动/重启状态   |
| Succeeded    | Pod 中所有容器都已正常终止，且不会重启（如一次性任务 Job 类型 Pod）   |
| Failed       | Pod 中所有容器已终止，至少一个容器异常退出（退出码非 0）               |
| Unknown      | K8s 无法获取 Pod 状态（如节点失联）                                 |

## 二、关键生命周期钩子（Hook）
K8s 提供钩子让用户在 Pod 容器生命周期的关键节点执行自定义逻辑，是核心扩展能力：
### 1. 启动钩子（PostStart）
- 触发时机：容器创建完成后（立即触发，不保证在容器进程启动前执行）
- 用途：初始化配置、注册服务、加载数据等
- 示例（YAML 片段）：
  ```yaml
  spec:
    containers:
    - name: nginx
      image: nginx
      lifecycle:
        postStart:
          exec:
            command: ["/bin/sh", "-c", "echo '容器启动完成' > /tmp/start.log"]
  ```

### 2. 终止钩子（PreStop）
- 触发时机：容器终止前（默认有30s宽限期，超时强制杀死）
- 用途：优雅关闭服务、保存数据、通知上下游
- 示例（YAML 片段）：
  ```yaml
  spec:
    containers:
    - name: nginx
      image: nginx
      lifecycle:
        preStop:
          exec:
            command: ["/bin/sh", "-c", "nginx -s quit"] # 优雅关闭nginx
  ```

## 三、Pod 终止流程（优雅退出）
Pod 不会被直接杀死，K8s 会按固定流程终止，保证服务无感知：
1. K8s 向 Pod 发送终止信号（TERM），并标记 Pod 为 Terminating 状态；
2. Pod 从服务端点（Service）中移除，不再接收新请求；
3. 触发 PreStop 钩子（如果配置）；
4. 等待宽限期（默认30s，可通过 `terminationGracePeriodSeconds` 修改）；
5. 宽限期内容器自行退出，超时则发送 SIGKILL 强制杀死容器；
6. K8s 清理 Pod 资源，删除 Pod 对象。

## 四、特殊状态：容器重启策略
Pod 的重启策略（restartPolicy）决定容器异常时的行为，仅作用于 Running 阶段的 Pod：
- Always：容器退出就重启（默认，适合长期运行的服务，如Web服务）；
- OnFailure：容器异常退出（退出码非0）时重启（适合一次性任务）；
- Never：容器退出永不重启（适合测试场景）。

---

### 总结（笔记核心要点）
1. Pod 生命周期核心分 5 个阶段（Pending/Running/Succeeded/Failed/Unknown），仅反映宏观状态；
2. 两大核心钩子：PostStart（容器启动后）、PreStop（容器终止前），用于自定义操作；
3. Pod 终止有优雅流程（宽限期+PreStop），重启策略（Always/OnFailure/Never）决定容器异常后的行为。