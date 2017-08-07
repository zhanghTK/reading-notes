Sentinel 是运行在特殊模式下的 Redis 服务器，用于监视和管理主从服务器

## 初始化

与普通 Redis 服务器存在差异：![Sentinel 模式下 Redis 服务器主要功能](https://i.loli.net/2017/08/03/5982f922c3b60.png)

### 使用专用代码

- 宏定义：#deine REDIS_SERVERPORT 6379 -> #define REDIS_SENTINEL_PORT 26379
- 结构体：redis.c/redisCommandTable -> sentinel.c/sentinelcmds

### 创建实例

Sentinel 模式下服务器会创建一个 sentinel.c/sentinelState 结构的实例，而不同与普通服务器的 redisServer 结构实例

### 初始化

sentinel.c/sentinelRedisInstance 用于表示一个被 Sentinel 监视的 Redis 服务实例（可以是主从服务器或其他 Sentinel）

sentinelState 中定义：dict *masters 字典的值是一个指向 sentinelRedisInstance 结构的指针，用于记录所有被 Sentinel 监视的主服务相关信息

初始化最重要的内容：初始化 masters 字典

### 创建连向主服务的网络连接

- 命令连接：用于向主服务器发送命令，并接收命令回复
- 订阅连接：订阅主服务器的 __sentinel__:hello 频道

此时，Sentinel 是主服务的客户端

## 获取主服务器信息

Sentinel 默认每十秒向主服务器发送 INFO 命令，以获取：
- 主服务器信息：用于更新主服务实例
- 主服务下从服务器信息，用来更新/添加主服务实例的 slaves 字段，定义于 sentinelRedisInstance 的 dict *slaves，字典的值是从服务器对应的 sentinelRedisInstance 结构

## 获取从服务器信息

当发现新的从服务器会创建从服务实例，并创建从服务器的命令连接和订阅连接

Sentinel 默认每十秒向从服务器发送 INFO 命令，获取并更新从服务（以及从服务器与主服务器）信息

## 向主/从服务器发送信息

Sentinel 默认每两秒向服务的 __sentinel__:hello 频道发送消息：

PUBLISH __sentinel__:hello "<s_ip>,<s_port>,<s_runid>,<s_epoch>,<m_name>,<m_ip>,<m_port>,<m_epoch>"

## 接收主/从服务器的频道信息

Sentinel 既向 __sentinel__:hello 发送信息，又从 __sentinel__:hello 接收消息

当一个服务器被多个 Sentinel 监视时，每个 Sentinel 发送的信息除了服务器，彼此也会收到：
- 如果是自己发的，丢弃（根据 s_runid 判断）
- 如果是其他服务器发的，根据信息中的参数，更新主服务器实例结构


### 更新 sentinels 字段

Sentinel 状态的主服务器实例的 sentinels 字段保存除自身外，共同监视主服务的其他 Sentinel 资料

当接收到信息的处理：
1. 提取 sentinel 有关参数
2. 提取主服务器有关参数
3. 从 sentinel 的 masters 字典中查找主服务器实例结构，查找 sentinels 字段，根据结构创建/更新 sentinel 信息

### 创建连向其他 Sentinel 命令的连接

sentinel 通过频道发现新的 Sentinel 时，会保存其对应实例，并创建一个连向新 Sentinel 的命令连接。（彼此互联，但不创建订阅连接）

## 检测主观下线状态

Sentinel 默认每秒向所有创建了命令连接的实例（主从服务器，其他 sentinel）发送 PING 命令，以检测实例是否在线

可以在 Sentinel 配置文件中 down-after-milliseconds 指定 Sentinel，主从服务器的下线时间

## 检查客观下线状态

Sentinel 会根据对其他 Sentinel 询问的结果判断主服务器客观下线状态：

1. 发送 SENTINEL is-master-down-by-addr 命令
2. 接收 SENTINEL is-master-down-by-addr 命令：检查，发送 Multi Bulk 回复
3. 接收 SENTINEL is-master-down-by-addr 命令回复：Sentinel 会根据对其他 Sentinel 询问的结果判断主服务器客观下线状态

## 选举领头 Sentinel

当一个主服务器被判断为库管下线时，监视这个下线服务器的各个 Sentinel 会协商，选举一个领头 Sentinel，由领头 Sentinel 对下线主服务器执行故障转义。

选举算法：对 Raft 算法的实现

## 故障转移

1. 选出新的主服务器

2. 修改从服务器的复制目标

3. 将旧的主服务器变为从服务器