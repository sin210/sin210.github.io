### 一、日常巡检/状态查看（最核心）
这类命令用于快速了解 Redis 实例的运行状态、资源占用、连接情况，是运维的基础操作。

#### 1. `info`：查看 Redis 核心状态信息（重中之重）
这是运维最常用的命令，能输出几乎所有关键指标，支持按模块过滤，避免信息过载。
```bash
# 查看所有信息（内容较多，建议结合 grep 过滤）
info
# 查看指定模块的信息（常用模块如下）
info server       # 服务器信息（版本、启动时间、运行ID等）
info clients      # 客户端连接信息（连接数、阻塞客户端数等）
info memory       # 内存使用信息（已用内存、内存碎片率、最大内存等）
info stats        # 统计信息（命中/未命中数、命令执行数、过期key数等）
info persistence  # 持久化信息（RDB/AOF状态、最后持久化时间等）
info replication  # 主从复制信息（角色、主库地址、复制状态等）
```
**实用示例**（结合 Linux 命令过滤）：
```bash
# 查看已用内存
redis-cli info memory | grep used_memory_human
# 查看连接数
redis-cli info clients | grep connected_clients
# 查看缓存命中率（命中率 = keyspace_hits / (keyspace_hits + keyspace_misses)）
redis-cli info stats | grep -E "keyspace_hits|keyspace_misses"
```

#### 2. `config get`：查看配置参数
用于确认 Redis 配置是否符合预期，无需去翻配置文件。
```bash
# 查看单个配置
config get maxmemory          # 查看最大内存限制
config get requirepass       # 查看是否设置密码（返回 "" 表示无密码）
config get daemonize         # 查看是否后台运行
config get save              # 查看RDB持久化策略
config get appendonly        # 查看AOF是否开启
# 模糊匹配配置（* 通配符）
config get "*memory*"        # 查看所有和内存相关的配置
```

#### 3. `client list`：查看客户端连接详情
排查**连接数过高、慢连接、异常客户端**的核心命令。
```bash
client list  # 输出所有客户端连接，每行一个客户端，包含地址、空闲时间、发送/接收字节数等
# 实用过滤（找空闲时间超过3600秒的客户端）
redis-cli client list | grep -E "idle=([3-9][0-9]{3}|[1-9][0-9]{4,})"
```

### 二、性能监控/问题排查
用于定位 Redis 性能瓶颈、慢查询、阻塞等问题。

#### 1. `slowlog`：查看慢查询日志
Redis 会记录执行时间超过 `slowlog-log-slower-than`（默认10000微秒，即10ms）的命令，是排查慢命令的关键。
```bash
slowlog get 10    # 获取最近10条慢查询日志
slowlog len       # 查看慢查询日志总数
slowlog reset     # 清空慢查询日志
# 先调整慢查询阈值（临时生效，重启失效，永久生效需改配置）
config set slowlog-log-slower-than 5000  # 记录超过5ms的命令
```

#### 2. `monitor`：实时监控所有执行的命令
用于临时排查“谁在执行什么命令”，**注意：生产环境慎用，会增加Redis负载**，建议短时间开启。
```bash
monitor  # 实时输出所有客户端执行的命令（按Ctrl+C停止）
```

#### 3. `keys`：慎用！模糊查询Key（生产环境禁用）
`keys *` 会遍历所有Key，Redis是单线程，遍历期间会阻塞所有请求，**生产环境严禁使用**！
替代方案：`scan`（渐进式遍历，不阻塞）
```bash
# scan 语法：scan 游标 [MATCH 匹配规则] [COUNT 每次遍历数量]
scan 0 MATCH user* COUNT 100  # 从游标0开始，匹配user开头的Key，每次遍历100个
# 示例输出：1) "10086" 2) 1) "user:1" 2) "user:2"（游标返回0表示遍历完成）
```

### 三、数据管理/实例维护
用于管理数据、调整实例状态，保障数据安全和服务稳定。

#### 1. 数据备份/恢复
```bash
# 手动触发RDB持久化（生成快照文件，运维常用作手动备份）
bgsave  # 后台执行（不阻塞），推荐使用
save    # 前台执行（阻塞Redis，生产环境禁用）

# 数据恢复：无需命令，将RDB/AOF文件放到Redis数据目录，重启Redis即可
```

#### 2. 内存管理
```bash
# 手动清理过期Key（Redis会自动清理，但可手动触发）
expire key 0      # 手动让指定Key过期
flushdb           # 清空当前数据库（默认db0）
flushall          # 清空所有数据库（生产环境慎用！）

# 内存淘汰策略（当内存达到maxmemory时触发）
config set maxmemory-policy allkeys-lru  # 淘汰最少使用的Key（最常用）
# 其他策略：volatile-lru（只淘汰带过期时间的）、allkeys-random（随机淘汰）等
```

#### 3. 客户端管理
```bash
# 查看当前客户端连接数
info clients | grep connected_clients
# 关闭指定客户端连接（解决异常连接）
client kill 192.168.1.100:54321  # 格式：IP:端口
# 关闭所有空闲超过N秒的客户端（批量清理空闲连接）
client kill IDLE 3600
```

#### 4. 主从复制运维
```bash
# 查看主从状态
info replication
# 从库执行：手动切换为主库（故障转移时用）
replicaof no one  # Redis5.0+用replicaof，旧版本用slaveof
# 从库执行：指定主库地址（搭建主从时用）
replicaof 192.168.1.200 6379
# 设置主库密码（从库连接主库时需要）
config set masterauth 123456
```

#### 5. 密码/权限管理
```bash
# 设置访问密码（临时生效，永久生效需改redis.conf）
config set requirepass 123456
# 验证密码（连接后执行）
auth 123456
# 清除密码（测试环境用，生产环境禁止）
config set requirepass ""
```

### 四、应急操作
用于处理Redis卡死、阻塞等紧急情况。
```bash
# 查看当前正在执行的命令（排查阻塞）
redis-cli info stats | grep latest_fork_usec  # 查看fork操作耗时（bgsave/AOF重写会fork）
# 强制关闭Redis（紧急情况，优先用redis-cli shutdown）
redis-cli shutdown  # 优雅关闭（会触发持久化，推荐）
kill -9 [Redis进程ID]  # 强制杀死（可能丢失数据，仅应急用）
```

### 总结
1. 运维Redis的核心命令围绕**状态查看（info）、配置确认（config）、慢查询排查（slowlog）、连接管理（client）** 展开，其中`info`是最基础也最重要的命令；
2. 生产环境严禁使用`keys`、`save`、`flushall`等阻塞/高危命令，优先用`scan`、`bgsave`替代；
3. 慢查询（slowlog）和客户端连接（client list）是排查性能问题的核心工具，需熟练掌握过滤和分析方法。