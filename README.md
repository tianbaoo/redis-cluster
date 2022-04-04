## redis-cluster集群搭建
### redis主从复制
* 主从复制即将master中的数据即时、有效的复制到slave中
* 一个master可以拥有多个slave，一个slave只对应一个master
* master:写数据；执行写操作时，将出现变化的数据自动同步到slave
* slave:读数据；写数据（禁止）
* 读写分离：主节点写，从节点读，提高服务器的读写负载能力
* 数据冗余︰主从复制实现了数据的热备份，是持久化之外的一种数据冗余方式
* 故障恢复︰当主节点出现问题时，可以由从节点提供服务，实现快速的故障恢复 ; 实际上是一种服务的冗余
* 负载均衡︰在主从复制的基础上，配合读写分离，可以由主节点提供写服务，由从节点提供读服务（即写Redis数据时应用连接主节点，读Redis数据时应用连接从节点），分担服务器负载 ; 尤其是在写少读多的场景下，通过多个从节点分担读负载，可以大大提高Redis服务器的并发量
* 主从切换技术的方法是：当主服务器宕机后，需要手动把一台从服务器切换为主服务器，这就需要人工干预，费事费力，还会造成一段时间内服务不可用
* 优缺点：数据备份、读写分离，提高服务器性能；不能自动故障恢复,RedisHA系统（需要开发）、无法实现动态扩容
### redis哨兵模式
* 哨兵是一个独立的进程，作为进程，它会独立运行。其原理是哨兵通过发送命令，等待Redis服务器响应，从而监控运行的多个Redis实例
* 当哨兵监测到master宕机，会自动将slave切换成master，然后通过发布订阅模式通知其他的从服务器，修改配置文件，让它们切换主机
* 故障切换（failover）过程：假设主服务器宕机，哨兵1先检测到这个结果，系统并不会马上进行failover过程，仅仅是哨兵1主观的认为主服务器不可用，这个现象成为主观下线。当后面的哨兵也检测到主服务器不可用，并且数量达到一定值时，那么哨兵之间就会进行一次投票，投票的结果由一个哨兵发起，进行failover操作。切换成功后，就会通过发布订阅模式，让各个哨兵把自己监控的从服务器实现切换主机，这个过程称为客观下线。这样对于客户端而言，一切都是透明的
* 在实际生产情况中，Redis Sentinel（哨兵） 是集群的高可用的保障，为避免 Sentinel 发生意外，它一般是由 3～5 个节点组成，这样就算挂了个别节点，该集群仍然可以正常运转
* Redis Sentinel的节点数量要满足2n+1（n>=1）的奇数个
* 优缺点：自动化故障恢复；Redis 数据节点中 slave 节点作为备份节点不提供服务、无法实现动态扩容
### redis-cluster集群
* Redis Cluster是社区版推出的Redis分布式集群解决方案，主要解决Redis分布式方面的需求，比如，当遇到单机内存，并发和流量等瓶颈的时候，Redis Cluster能起到很好的负载均衡的目的
* 群集至少需要3主3从，且每个实例使用不同的配置文件
* 在cluster架构下，默认的，一般redis-master用于接收读写，而redis-slave则用于备份，当有请求是在向slave发起时，会直接重定向到对应key所在的master来处理，但如果不介意读取的是redis-cluster中有可能过期的数据并且对写请求不感兴趣时，则亦可通过readonly命令，将slave设置成可读，然后通过slave获取相关的key，达到读写分离。具体可以参阅redis官方文档等相关内容
* 优缺点：解决分布式负载均衡的问题。具体解决方案是分片/虚拟槽slot、可实现动态扩容、P2P模式，无中心化；为了性能提升，客户端需要缓存路由表信息、Slave在集群中充当“冷备”，不能缓解读压力
### 目录结构
```go
.
├── README.md
├── docker-compose.yml
├── redis-1
│   ├── data
│   │   ├── appendonly.aof
│   │   ├── dump.rdb
│   │   ├── nodes.conf
│   │   └── redis.log
│   └── redis.conf
├── redis-2
│   ├── data
│   │   ├── appendonly.aof
│   │   ├── dump.rdb
│   │   ├── nodes.conf
│   │   └── redis.log
│   └── redis.conf
├── redis-3
│   ├── data
│   │   ├── appendonly.aof
│   │   ├── dump.rdb
│   │   ├── nodes.conf
│   │   └── redis.log
│   └── redis.conf
├── redis-4
│   ├── data
│   │   ├── appendonly.aof
│   │   ├── dump.rdb
│   │   ├── nodes.conf
│   │   └── redis.log
│   └── redis.conf
├── redis-5
│   ├── data
│   │   ├── appendonly.aof
│   │   ├── dump.rdb
│   │   ├── nodes.conf
│   │   └── redis.log
│   └── redis.conf
└── redis-6
    ├── data
    │   ├── appendonly.aof
    │   ├── dump.rdb
    │   ├── nodes.conf
    │   └── redis.log
    └── redis.conf
```
### 修改IP配置
* ifconfig查看本机IP是多少，我这里为192.168.1.6  
* 修改redis-1到redis-6中的redis.conf中的cluster-announce-ip 192.168.1.6字段为你的IP，替换192.168.1.6
* 在redis-cluster目录下运行
```bash
sudo docker-compose up -d
```
### 创建集群关联
**cluster-replicas 表示 一个主节点有几个从节点**
```bash
sudo docker exec -it redis-6381 redis-cli -p 6381 -a 123456 --cluster create 192.168.1.6:6381 192.168.1.6:6382 192.168.1.6:6383 192.168.1.6:6384 192.168.1.6:6385 192.168.1.6:6386 --cluster-replicas 1
```
按要求输入"yes"然后按回车  
### 集群测试
进入一个节点后进行设置key value  
```bash
sudo docker exec -it redis-6381 redis-cli -h 192.168.1.6 -p 6383 -a 123456

Warning: Using a password with '-a' or '-u' option on the command line interface may not be safe.
192.168.1.6:6383> set name admin
(error) MOVED 5798 192.168.1.6:6382
```
由于Redis Cluster会根据key进行hash运算，然后将key分散到不同slots，name的hash运算结果在redis-6382节点上的slots中。所以我们操作redis-6383写操作会自动路由到redis-6382。然而error提示无法路由  
这是因为差一个 -c 参数，因为这是cluster集群，需要加-c参数  
```bash
sudo docker exec -it redis-6381 redis-cli -h 192.168.1.6 -p 6383 -a 123456 -c

Warning: Using a password with '-a' or '-u' option on the command line interface may not be safe.
192.168.1.6:6383> set name admin
-> Redirected to slot [5798] located at 192.168.1.6:6382
OK
192.168.1.6:6382> get name
"admin"
192.168.1.6:6382>
```
### 查看集群信息
进入其中一个节点后
```bash
cluster nodes
```
```bash
cluster slots
```
```bash
cluster info
```
### 手动扩容
#### 创建新节点
redis-6381到redis-6386所在机器的IP为192.168.1.6  
此时我们要在192.168.1.9的机器上新增redis-6387、redis-6388节点并加入刚才在192.168.1.6创建的集群中  
首先需要在192.168.1.9机器中创建
```go
.
├── README.md
├── docker-compose.yml
├── redis-7
│   ├── data
│   │   ├── appendonly.aof
│   │   ├── dump.rdb
│   │   ├── nodes.conf
│   │   └── redis.log
│   └── redis.conf
├── redis-8
│   ├── data
│   │   ├── appendonly.aof
│   │   ├── dump.rdb
│   │   ├── nodes.conf
│   │   └── redis.log
│   └── redis.conf
```
将redis-7、redis-8目录下的redis.conf中的cluster-announce-ip 192.168.1.6 ，替换192.168.1.9  
其次docker-compose.yml内容修改为
```bash
# 描述 Compose 文件的版本信息
version: "3.8"

# 定义服务，可以多个
services:
  redis-6387: # 服务名称
    image: redis:6 # 创建容器时所需的镜像
    container_name: redis-6387 # 容器名称
    restart: always # 容器总是重新启动
    # network_mode: "host" # host 网络模式
    ports:
      - "6387:6387"
      - "16387:16387"    
    volumes: # 数据卷，目录挂载
      - ./redis-7/redis.conf:/usr/local/etc/redis/redis.conf
      - ./redis-7/data:/data
    command: ["redis-server", "/usr/local/etc/redis/redis.conf"] # 覆盖容器启动后默认执行的命令
    environment:
      # 设置时区为上海，否则时间会有问题
      - TZ=Asia/Shanghai
    logging:
      options:
        max-size: '100m'
        max-file: '10'

  redis-6388:
    image: redis:6
    container_name: redis-6388
    # network_mode: "host" # host 网络模式
    ports:
      - "6388:6388"
      - "16388:16388"
    volumes:
      - ./redis-8/redis.conf:/usr/local/etc/redis/redis.conf
      - ./redis-8/data:/data
    command: ["redis-server", "/usr/local/etc/redis/redis.conf"]
    environment:
      # 设置时区为上海，否则时间会有问题
      - TZ=Asia/Shanghai
    logging:
      options:
        max-size: '100m'
        max-file: '10'
```
运行
```bash
sudo docker-compose up -d
```
#### 添加主节点
```bash
# 进入任意节点
sudo docker exet -it redis-6381 /bin/sh

# 添加主节点(192.168.1.6:6381 -a 123456这个可以是任何已存在的节点，主要用于获取集群信息)
redis-cli --cluster add-node 192.168.1.9:6387 192.168.1.6:6381 -a 123456
```
进入一个节点，查看刚刚添加的的主节点ID是多少  
```bash
cluster nodes
```
记录刚刚添加的主节点的ID  
#### 添加从节点
```bash
# 进入任意节点
docker exet -it redis-6381 /bin/sh

# 添加从节点(192.168.1.6:6381 -a 123456  这个可以是任何已存在的节点，主要用于获取集群信息)(t60c7d0ae166717d645e87u48b72b9706c7c75f2  为刚加的主节点 id)
redis-cli --cluster add-node --cluster-slave --cluster-master-id t60c7d0ae166717d645e87u48b72b9706c7c75f2 192.168.1.9:6388 192.168.1.6:6381 -a 123456
```
#### 分配插槽
刚加入的主从节点还不能使用，因为还没有分配插槽  
```bash
-- 分配插槽(192.168.1.6:6381 -a 123456  这个可以是任何已存在的节点，主要用于获取集群信息)
redis-cli --cluster rebalance --cluster-threshold 1 --cluster-use-empty-masters 192.168.1.6:6381 -a 123456
```
**rebalance平衡集群节点slot数量**  
**语法**
```bash
rebalance host:port
--weight <arg>
--auto-weights
--threshold <arg>
--use-empty-masters
--timeout <arg>
--simulate
--pipeline <arg>

host:port：这个是必传参数，用来从一个节点获取整个集群信息，相当于获取集群信息的入口。
--weight <arg>：节点的权重，格式为node_id=weight，如果需要为多个节点分配权重的话，需要添加多个--weight <arg>参数，即--weight b31e3a2e=5 --weight 60b8e3a1=5，node_id可为节点名称的前缀，只要保证前缀位数能唯一区分该节点即可。没有传递–weight的节点的权重默认为1。
--auto-weights：这个参数在rebalance流程中并未用到。
--threshold <arg>：只有节点需要迁移的slot阈值超过threshold，才会执行rebalance操作。具体计算方法可以参考下面的rebalance命令流程的第四步。
--use-empty-masters：rebalance是否考虑没有节点的master，默认没有分配slot节点的master是不参与rebalance的，设置--use-empty-masters可以让没有分配slot的节点参与rebalance。
--timeout <arg>：设置migrate命令的超时时间。
--simulate：设置该参数，可以模拟rebalance操作，提示用户会迁移哪些slots，而不会真正执行迁移操作。
--pipeline <arg>：与reshar的pipeline参数一样，定义cluster getkeysinslot命令一次取出的key数量，不传的话使用默认值为10。
```