### 一、Redis 是什么？
Redis（Remote Dictionary Server）是一款**开源的、高性能的键值对（Key-Value）内存数据库**，同时支持数据持久化（把内存数据写到硬盘），常用来做缓存、分布式锁、消息队列、排行榜等场景。

核心特点：
- 基于内存操作，速度极快（读写性能可达 10 万+/秒）；
- 支持多种数据类型（字符串、哈希、列表、集合、有序集合等）；
- 支持持久化（RDB/AOF 两种方式）；
- 支持主从复制、集群，易扩展；
- 单线程模型（避免线程切换开销），但通过 I/O 多路复用保证高性能。

### 二、快速安装（以 Linux 为例，Windows 建议用 WSL 或 Redis 桌面版）
#### 1. 安装步骤
```bash
# 1. 安装依赖
yum install -y gcc gcc-c++ make

# 2. 下载并解压 Redis（以 7.2.4 版本为例）
wget https://download.redis.io/releases/redis-7.2.4.tar.gz
tar -zxvf redis-7.2.4.tar.gz
cd redis-7.2.4

# 3. 编译安装
make
make install PREFIX=/usr/local/redis  # 指定安装目录

# 4. 复制配置文件（方便修改）
cp redis.conf /usr/local/redis/conf/

# 5. 启动 Redis（先修改配置，允许远程访问+后台运行）
vim /usr/local/redis/conf/redis.conf
# 修改以下配置：
# bind 127.0.0.1 → 注释掉（允许所有IP访问）
# protected-mode yes → no（关闭保护模式）
# daemonize no → yes（后台运行）

# 6. 启动 Redis 服务
/usr/local/redis/bin/redis-server /usr/local/redis/conf/redis.conf

# 7. 进入 Redis 客户端
/usr/local/redis/bin/redis-cli
```

#### 2. 验证安装
客户端输入 `ping`，返回 `PONG` 说明启动成功：
```bash
127.0.0.1:6379> ping
PONG
```

### 三、核心基础：数据类型与常用命令
Redis 的核心是**键（Key）** + **值（Value）**，Value 支持多种数据类型，先掌握最常用的 5 种：

| 数据类型 | 作用                 | 典型场景               |
|----------|----------------------|------------------------|
| String   | 字符串/数字          | 缓存、计数器、分布式ID |
| Hash     | 键值对集合           | 缓存对象（用户、商品） |
| List     | 有序列表（双向链表） | 消息队列、最新列表     |
| Set      | 无序不重复集合       | 去重、交集/并集        |
| ZSet     | 有序不重复集合       | 排行榜、计分系统       |

#### 1. 通用命令（所有数据类型都能用）
```bash
# 查看所有键（支持模糊匹配，如 keys user*）
keys *
# 判断键是否存在（返回 1 存在，0 不存在）
exists name
# 删除键（返回删除的数量）
del name
# 设置键的过期时间（单位：秒，expireat 用时间戳）
expire name 60
# 查看键的剩余过期时间（-1 永不过期，-2 已过期）
ttl name
# 查看值的类型（string/hash/list/set/zset）
type name
```

#### 2. 常用数据类型命令示例
##### （1）String（最基础）
```bash
# 设置值（setnx：不存在才设置，分布式锁常用）
set name "zhangsan"
# 获取值
get name  # 返回 "zhangsan"
# 数值递增（incrby 指定步长，decr/decrby 递减）
set age 18
incr age  # age 变为 19
incrby age 5  # age 变为 24
# 追加字符串
append name " - 北京"  # name 变为 "zhangsan - 北京"
# 获取字符串长度
strlen name  # 返回 13
```

##### （2）Hash（适合存对象）
```bash
# 设置单个字段
hset user:1 name "lisi" age 20
# 设置多个字段
hmset user:1 gender "male" city "Shanghai"
# 获取单个字段
hget user:1 name  # 返回 "lisi"
# 获取多个字段
hmget user:1 name age  # 返回 ["lisi", "20"]
# 获取所有字段和值
hgetall user:1  # 返回 ["name","lisi","age","20",...]
# 获取所有字段名
hkeys user:1  # 返回 ["name","age","gender","city"]
# 获取所有值
hvals user:1  # 返回 ["lisi","20","male","Shanghai"]
# 删除字段
hdel user:1 city
```

##### （3）List（有序、可重复）
```bash
# 从列表左侧/右侧添加元素
lpush list1 a b c  # 列表：c b a
rpush list1 d e    # 列表：c b a d e
# 从左侧/右侧弹出元素（弹出后元素从列表删除）
lpop list1  # 返回 c，列表变为：b a d e
rpop list1  # 返回 e，列表变为：b a d
# 获取列表长度
llen list1  # 返回 3
# 获取指定范围元素（0 是第一个，-1 是最后一个）
lrange list1 0 -1  # 返回 ["b","a","d"]
```

##### （4）Set（无序、不重复）
```bash
# 添加元素
sadd set1 1 2 3 3  # 只添加 1、2、3（去重）
# 获取所有元素
smembers set1  # 返回 [1,2,3]
# 判断元素是否在集合中
sismember set1 2  # 返回 1
# 求交集（共同元素）
sadd set2 2 3 4
sinter set1 set2  # 返回 [2,3]
# 求并集
sunion set1 set2  # 返回 [1,2,3,4]
# 删除元素
srem set1 1  # 集合变为 [2,3]
```

##### （5）ZSet（有序、不重复，通过分数排序）
```bash
# 添加元素（格式：zadd 键 分数 元素）
zadd rank 100 "zhangsan" 90 "lisi" 95 "wangwu"
# 获取元素分数
zscore rank "lisi"  # 返回 90
# 按分数升序/降序取元素（withscores 显示分数）
zrange rank 0 -1 withscores  # 升序：lisi(90)、wangwu(95)、zhangsan(100)
zrevrange rank 0 -1 withscores  # 降序：zhangsan(100)、wangwu(95)、lisi(90)
# 按分数范围取元素（90<=分数<=95）
zrangebyscore rank 90 95 withscores  # 返回 lisi(90)、wangwu(95)
```

### 四、简单实战：用 Python 操作 Redis
学习 Redis 最终要结合编程语言使用，这里以 Python 为例（需安装 `redis` 库）：

#### 1. 安装依赖
```bash
pip install redis
```

#### 2. 示例代码
```python
import redis

# 连接 Redis（host 填你的 Redis 地址，port 默认为 6379，无密码则不加 password）
r = redis.Redis(
    host='127.0.0.1',
    port=6379,
    db=0,  # 使用第 0 个数据库（Redis 默认有 16 个，0-15）
    decode_responses=True  # 自动将返回值从 bytes 转为字符串
)

# 1. 操作 String
r.set("username", "python_user")
print("String 获取：", r.get("username"))  # 输出：python_user
r.incr("visit_count")  # 计数器+1
print("计数器：", r.get("visit_count"))  # 输出：1

# 2. 操作 Hash
r.hset("user:2", mapping={"name": "redis_user", "age": 25})
print("Hash 获取：", r.hgetall("user:2"))  # 输出：{'name': 'redis_user', 'age': '25'}

# 3. 操作 ZSet（排行榜）
r.zadd("score_rank", {"小明": 88, "小红": 95, "小刚": 90})
# 获取前三名（降序）
top3 = r.zrevrange("score_rank", 0, 2, withscores=True)
print("排行榜前三名：", top3)  # 输出：[('小红', 95.0), ('小刚', 90.0), ('小明', 88.0)]
```

### 五、Redis 持久化（入门级理解）
Redis 数据默认存在内存中，重启后会丢失，持久化就是把数据存到硬盘：
- **RDB**：定时快照（比如每 5 分钟/改了 1000 次），生成二进制文件，恢复快，适合备份；
- **AOF**：记录所有写命令（比如 set、hset），重启时重新执行命令恢复，数据更完整，适合高可用。

新手默认开启 RDB 即可，无需手动修改配置。

---

### 总结
1. Redis 是高性能内存键值数据库，核心优势是**快**，支持多种数据类型和持久化；
2. 入门先掌握**通用命令** + **5 种核心数据类型**的常用操作，理解每种类型的适用场景；
3. 结合编程语言（如 Python）实操，能快速把 Redis 用在实际项目中（比如缓存、排行榜）。