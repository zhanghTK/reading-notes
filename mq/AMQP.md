服务模型：投递服务模型，而非 CS 服务模型。
  在程序角度：当程序连接到 RabbitMQ 时必须决定自己是发送者还是接受者
  在 AMQP 角度：AMQP 是生产者还是消费者
  
宏观
消息发送：由生产者创建，包括标签和内容

路由：根据消息标签把消息发给接收方，路由过程中标签没有被传递；消息发后即忘

消息接受：从订阅队列接收消息，无法感知发送方

信道：建立在 TCP 连接内的虚拟连接，具有唯一 ID，避免 TCP 开销



微观
消息路由：交换器、队列、绑定

生产者把消息发布到交换器，绑定决定消息如何从交换器路由到特定队列

订阅队列中的消息：
  - basic.consume：
  - basic.get
  
消息投递：
  - 当队列拥有多个消费者时，每个消息只发送给一个订阅消费者
  - 投递过程：
    - 消息到达队列
    - MQ 将消息发送给订阅的消费者
    - 消费者确认接收到消息：basic.ack
    - MQ 删除队列中的消息
  - MQ 未收到消费者确认将不再向该消费者投递消息  
  - 拒绝消息
    - 连接断开
    - basic.reject
      - true：MQ 将消息发送给其他订阅消费者
      - false：MQ 将消息从队列移除，并加入 dead letter 队列
- 队列
  - 创建:
    - 生产者
    - 消费者：统一信道上订阅了另一个队列的话，将无法再声明队列，必须先取消订阅并将信道设置为传输模式
    - 由谁负责创建：生产消费者都尝试创建
    - 创建命令：queue.declare
  - 订阅&绑定：根据名称
  - 参数：
    - exclusive：私有队列
    - auto-delete：当没有消费者订阅队列时，队列自动移除
  - 特点：
    - 消息暂存，等待被消费
    - 负载均衡
    - MQ 中的终点
    
交换器 & 绑定
路由键：决定 MQ 将消息从交换器投递到哪个队列，绑定队列到交换器
交换器：
  - 有多种类型，每种类型对应不同路由算法，使用 exchange.declare 命令设置
  - 类型
    - direct：如果路由键匹配。消息被投递到对应队列。
      - 服务器必须实现默认交换器（名字为空）
      - 声明一个队列时会自动绑定到默认交换器
      - 消息发送：$channel->basic_publish($msg, '', 'queue-name');
        - $msg：消息
        - ''：指定默认交换器
        - ''：路由器键
      - 可以多重绑定：一个交换器与多个队列使用相同的路由键绑定，类似广播
    - fanout：将收到的消息广播到绑定的队列，交换器接收的消息转发给绑定的所有队列
      - 不需要路由键
      - 需要提前将交换器和队列绑定，二者是多对多关系
    - topic：来自不同源头的消息能够到达同一个队列
      - 需要路由键
      - 需要提前绑定交换器和队列，绑定时用到路由键
      - 一个消息满足从交换器到同一个队列的多个匹配只投递一次
      - * 将 . 视为分隔符，# 将任意字符均视为关键字的匹配
    - headers：匹配 header 而非路由键，性能差，不使用
    
多租户模式
vhost：由 RabbitMQ 服务器创建的虚拟消息服务器（mini 版的 RabbitMQ），拥有单独的队列，交换器，绑定，权限
vhost 之间绝对隔离，优点：安全，便于移植
当在 vhost 集群上创建 vhost 时，整个集群都会创建该 vhost
创建：rabbitmqctl add_vhost [vhost_name]
删除：rabbitmqctl delete_vhost [vhost_name]
查看 rabbitmqctl list_vhosts

持久化
默认情况下：队列和交换器及里面的消息会随服务器重启删除
持久化的要求：
- 消息的投递模式设置为 2（持久）
- 发送到持久化的交换器
- 到达持久化的队列
持久化的影响：消息吞吐量极大下降（10倍都不少见），集群环境下工作的不好

事务
AMQP 事务：设置信道为事务模式，发送消息，提交；问题：性能，迫使生产者与 MQ 产生同步
RabbitMQ：发送方确认，信道设置成 confirm 模式，即可发送消息。信道为消息指派唯一ID，消息发服务端处理完成后**异步**发送一个确认给生产者


erlang

节点：虚拟机实例
程序：多个程序可以运行在同一个节点上
节点之间可以进行本地通信，而不管它们是否真的在同一台服务器上（这是什么黑科技）
当应用程序崩溃时，节点没有崩溃，会自动尝试重启应用程序

命令
节点：
启动：rabbitmq-server -server 
守护进程方式启动：rabbitmq-server -server -detached
关闭：rabbitmqctl stop

程序：
停止：rabbitmqctl stop_app

配置
配置文件：/etc/rabbitmq/rabbitmq.config
配置结构：[{app_name,[{config_name, config_value}]}]

Mnesia：存储队列，交换器，绑定等元数据的None SQL DB。存储过程：先写入日志文件，在定期转储到 Mnesia 数据库文件
  dump_log_write_threshold：转储频度
Rabbit：RabbitMQ 特定的配置选项
  tcp_listeners
  ssl_listeners
  ssl_options
  vm_memory_high_watermark：允许消耗的内存，默认0.4
  msg_store_file_size_limit：垃圾收集存储内容之前消息存储数据库的最大大小
  queue_index_max_journal_entries：在转储到消息存储数据库并提交之前消息存储日志里的最大条目数
  
权限
用户是访问控制的基本单元
管理用户
创建：rabbitmqctl add_user USER_NAME PASSWORD
删除：rabbitmqctl delete_user USER_NAME
查看所有用户：rabbitmqctl list_users
修改密码：rabbitmqctl change_password USER_NAME NEW_PWD
授权：rabbitmqctl set_permissions -p VHOST USER_NAME 配置权限 写权限 读权限
  权限写法示例：
    ".*" ：匹配任何队列和交换器
    "checks-.*" ：只匹配名字以“checks-”开头的队列和交换器
    "" ：不匹配队列和交换器
查看授权：rabbitmqctl list_permissions -p VHOST
清除授权：rabbitmqctl clear_permissions -p VHOST USER_NAME
修改授权：rabbitmqctl set_permissions

-p：指定虚拟主机或者路径信息

检查
检查所有队列：rabbitmqctl list_queue
检查交换器：rabbitmqctl list_exchanges
检查绑定：rabbitmqctl list_bindings

日志
日志位置：/var/log/rabbitmq
日志文件：
  - RABBITMQ_NODENAME_sasl.log：erlang 程序日志
  - RABBITMQ_NODENAME.log：服务器运行信息
轮换日志：rabbitmqctl rotate_logs SUFFIX
通过 AMQP 访问日志：amq.rabbitmq.log 的 topic 交换器，绑定的路由键：error，warning， info

集群
单一节点 Rabbit MQ 内部维护的元数据：
- 队列元数据
- 交换器元数据
- 绑定元数据
- vhost 元数据
以上信息默认维护在内存

集群节点 Rabbit MQ 内部维护的元数据：上面的再加上集群节点位置、节点与已记录的其他类型元数据的关系

集群队列
不是每个节点都有所有队列的完全拷贝，集群在单个节点而不是所有节点上创建完整的队列信息（元数据，状态，内容），这么做的原因
  - 为了存储空间
  - 性能，网络、磁盘
只有队列所有者知道队列信息，其他节点只知道队列的元数据和指向该队列所在节点的指针
如果挂掉的队列声明了持久化，其他节点不能重新声明，反之可以

分布交换器
交换器实质是一张查询表，路由是由信道完成的。
集群中每个节点都拥有每个交换器（查询表）的所有信息。
如果消息发送到了信道，但在路由完成之前节点发生故障，消息可能丢失，解决：事务或 confirm

内存节点与磁盘节点
单节点系统只允许使用磁盘类型的节点
集群系统必须保证至少有一个磁盘节点（不要求所有节点都是磁盘节点是为了性能，想象一下 RPC 场景）
如果集中唯一的磁盘节点挂了，集群可以路由消息，但不能创建队列/交换器/绑定/用户，更改权限，添加删除集群节点
内存节点重启后从磁盘节点获取元数据