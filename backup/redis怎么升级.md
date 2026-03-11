# Redis 升级超详细笔记（可直接抄到笔记软件）
## 一、升级前必须搞懂的基础
1. **Redis 版本规则**
    - 小版本：如 6.2.5 → 6.2.10，**兼容、无风险**
    - 大版本：如 6.x → 7.x → 8.x，**可能不兼容**，必须先测
2. **升级核心原则**
    - 先备份、再测试、后上线
    - 生产环境：**滚动升级，不停服**
    - 单机：允许短停机
3. **必须确认的信息**
    - 当前版本：`redis-cli INFO server | grep redis_version`
    - 数据目录：`redis-cli CONFIG GET dir`
    - 配置文件：`redis-cli CONFIG GET config-file`
    - 是否开启 AOF：`redis-cli CONFIG GET appendonly`
    - 是否主从/集群/Sentinel

---

## 二、升级前必做准备（任何场景都要）
### 1. 备份数据（强制）
```bash
# 手动触发 RDB 快照
redis-cli SAVE

# 备份数据目录（把 dir 结果填进去）
cp -r /var/lib/redis /var/lib/redis.bak.$(date +%Y%m%d)

# 备份配置文件
cp /etc/redis/redis.conf /etc/redis/redis.conf.bak.$(date +%Y%m%d)
```

### 2. 检查数据完整性
```bash
# 检查 RDB
redis-check-rdb /var/lib/redis/dump.rdb

# 检查 AOF（如果开启）
redis-check-aof /var/lib/redis/appendonly.aof
```

### 3. 停写、观察业务（可选但推荐）
- 观察 QPS、内存、命中率
- 可临时做**流量降级/只读**，避免升级时数据不一致

---

## 三、单机 Redis 升级（最简单，允许停机）
### 步骤
1. **停止旧版 Redis**
    ```bash
    redis-cli SHUTDOWN
    # 或 systemctl stop redis
    ```
2. **安装新版本（源码示例）**
    ```bash
    wget https://download.redis.io/releases/redis-7.2.5.tar.gz
    tar zxf redis-7.2.5.tar.gz
    cd redis-7.2.5
    make -j$(nproc)
    sudo make install
    ```
3. **用旧配置启动**
    ```bash
    redis-server /etc/redis/redis.conf
    # 或 systemctl start redis
    ```
4. **验证**
    ```bash
    redis-cli PING          # 返回 PONG
    redis-cli INFO server | grep redis_version
    redis-cli DBSIZE        # 看数据条数
    ```

---

## 四、主从 + Sentinel 高可用升级（生产最常用）
### 目标：**零停机滚动升级**
### 顺序：
**从节点 → 全部升级 → 切换主节点 → 升级原主节点**

### 1. 升级所有从节点（逐个）
```bash
# 登陆从节点
redis-cli -h 从IP SHUTDOWN

# 安装新版本（同上）

# 启动新版
redis-server /etc/redis/redis.conf

# 检查主从状态
redis-cli -h 从IP INFO replication
# 看到 master_link_status:up 说明同步正常
```

### 2. 手动触发主从切换（Sentinel）
```bash
redis-cli -p 26379 SENTINEL failover 主节点名称
```
- 等待：新主升主，旧主变从
- 检查：`INFO replication`

### 3. 升级原来的主节点
现在它已经是从节点，**直接按从节点步骤升级**。

### 4. 最终检查
- 所有节点版本一致
- 主从正常
- Sentinel 状态正常
- 业务无报错

---

## 五、Redis Cluster 集群升级（零停机）
### 顺序：
**所有从节点 → 升级 → 主节点逐个故障转移 → 再升级**

### 步骤
1. 逐个升级**所有从节点**（停机→更新→启动→验证同步）
2. 对**主节点**执行故障转移：
    ```bash
    redis-cli -h 主IP CLUSTER FAILOVER
    ```
3. 原主变成从，再升级
4. 逐个做完所有主节点
5. 最终检查：
    ```bash
    redis-cli CLUSTER NODES
    redis-cli CLUSTER INFO
    ```

---

## 六、Docker 版 Redis 升级
1. **备份数据卷**
    ```bash
    docker cp 容器名:/data /backup/redis_data_$(date +%Y%m%d)
    ```
2. 停止并删除旧容器
    ```bash
    docker stop redis
    docker rm redis
    ```
3. 运行新版镜像（挂载同数据卷）
    ```bash
    docker run -d \
      --name redis \
      -p 6379:6379 \
      -v /host/redis.conf:/usr/local/etc/redis/redis.conf \
      -v /host/data:/data \
      redis:7.2.5
    ```
4. 验证
    ```bash
    docker exec -it redis redis-cli INFO server
    ```

---

## 七、跨大版本必须注意（重点笔记）
1. **命令改名**
    - `slaveof` → `replicaof`
    - `slave-read-only` → `replica-read-only`
2. **配置变化**
    - 大版本建议用新配置，不要直接全量覆盖
    - 对比新旧 `redis.conf` 差异
3. **客户端兼容**
    - Jedis、Lettuce、StackExchange.Redis 等要支持新版
4. **模块不兼容**
    - RedisJSON、RedisSearch、RedisBloom 等必须对应版本

---

## 八、升级后必做验证清单
- [ ] PING 正常
- [ ] 版本正确
- [ ] 数据条数一致
- [ ] 主从/集群状态正常
- [ ] 读写正常
- [ ] 持久化正常
- [ ] 业务无报错
- [ ] 监控无异常

---

## 九、回滚方案（出问题立刻执行）
1. 停止新版 Redis
2. 恢复备份数据与配置
    ```bash
    cp -r /var/lib/redis.bak.xxx/* /var/lib/redis/
    cp /etc/redis/redis.conf.bak.xxx /etc/redis/redis.conf
    ```
3. 安装旧版本
4. 启动旧版
5. 检查业务恢复