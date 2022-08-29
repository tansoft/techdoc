
## 特性

* 常用命令复杂度O(1)
* Redis 6 多线程（通过io-threads-do-reads和io-threads启用）只是用来处理网络数据读写和协议解析，执行命令仍然是单线程顺序执行。所以不需考虑控制 key、lua、事务，LPUSH/LPOP 等等并发及线程安全问题。

## 常用数据类型

* String：binary安全字符串。
* Hash：Key-HashMap结构（map<str,map<str,obj>>），可只修改map里某个属性互不冲突。大型对象非常推荐。
* List：双向链表，支持双向Pop/Push。栈：lpush+lpop，队列：lpush+rpop，有限集合：lpush+ltrim，消息队列：lpush+brpop。
* Set：去重，底层是hash table。无需去重使用list性能最高。求交集并集，如好友关系，共同好友，共同兴趣。
* Sorted Set(zSet)：有序集，通过分数排序。实现是hash table(去重)加skip list(score->element,排序)，skip list像平衡二叉树，不同范围score分成多层，每层是按score排序链表。zrangebyscore用于排行榜。

## 特殊类型

* bitmaps/bitsets：位。
* HyperLogLogs（基数统计）：Redis 2.8.9。求多个列表基数（不重复的元素），可多个合并（求并集总数），每个键12K大小，带有0.81% 标准错误。用于独立ip数统计，uv，共同好友数等。
* geospatial：Redis3.2。底层是zset实现。地理位置距离，附近。城市数据(GEO： http://www.jsons.cn/lngcode)！ 有效经度-180度到180度，有效纬度-85.05112878度到85.05112878度。
* Stream：Redis 5.0。消息队列（类kafka）。不同key对应一个队列，消息默认持久化，可多播（可以有多个消费组），组内维护消费记录下标。解决pub/sub不能持久化，LPUSH+BRPOP不能多播和分组问题。

## 常用命令

* Incr/IncrBy/IncrByFloat/Decr/DecrBy：计数器场景，短期多次访问场景。key不存在会创建并设原值为0。IncrByFloat专门针对float，decr用负数。

* mset：是原子性操作，多key同步写入。

* SetNx：仅当key不存在时才Set。用于选主或分布式锁：所有Client不断抢注Master，成功后不断使用Expire刷新过期时间。如果Master掉了key就会失效，剩下的节点发生新一轮抢夺。

* LPush/RPop/RPush/LPop/BLPop/BRPop：常用LPush/RPop，可用阻塞版本BLPop/BRPop一直等消息。
* RPopLPush/BRPopLPush：弹出来同时推入另一个list，做状态机。
* LLen获取列表的长度。

* ZAdd/ZRem是O(log(N))，ZRangeByScore/ZRemRangeByScore是O(log(N)+M)，N是Set大小，M是结果/操作元素的个数。N 1000万复杂度才几十，但是M不能太多影响性能。

* Sub/Pub，订阅，可用于通知，群发消息。结合keyspace notification可用于修改通知，也可以设置超时key实现延时通知，但是时间不会精准。

## 常用代码

* 独立用户量统计

```redis
pfadd key1 a b c d e f g h i
pfcount key1    # 9，key1数量
pfadd key2 c j k l m e g a
pfcount key2    # 8，key2数量
pfmerge allkey key1 key2
pfcount allkey  # => 13, 两key去重后数量，可能会有一点误差
```

* 每周打卡

```redis
setbit sign 2 1     #周二打卡
setbit sign 4 0     #周四没打
getbit sign 3       #0，查看周三是否打卡
bitcount sign       #1，打卡天数
```

* 订阅

```redis
#发布信息
publish channel:1 hi    #返回值是通知了多少个订阅者
#订阅信息
subscribe channel:1
#模式（通配符）订阅
#通配符中?表示1个占位符，*表示任意占位符(包括0)，?*表示1个以上占位符
#如多个命中会响应多条
psubscribe c? b* d?*
#退订模式订阅，需要和订阅严格匹配
punsubscribe c*
#退订订阅
unsubscribe channel:1
```

* 事务

```redis
multi：开启事务，后续命令逐个放入队列中
exec：执行事务中所有命令
discard：放弃执行事务
watch：监视一或多个key，如果key被修改，则事务中断
unwatch：取消WATCH对所有key的监视。
```

* 地理位置

两级无法直接添加，我们一般会下载

```redis
geoadd china:city 118.76 32.04 manjing 112.55 37.86 taiyuan 123.43 41.80 shenyang   #增加三个位置到china:city
geoadd china:city 144.05 22.52 shengzhen 120.16 30.24 hangzhou 108.96 34.26 xian       #可多次添加

#获取多个位置
geopos china:city taiyuan manjing
1) 1) "112.54999905824661255"
   1) "37.86000073876942196"
2) 1) "118.75999957323074341"
   1) "32.03999960287850968"

#获取距离，单位m：米，km：公里，mi：英里，ft：英尺
geodist china:city taiyuan shenyang m
"1026439.1070"
geodist china:city taiyuan shenyang km
"1026.4391"

#附近
#以 100,30 这个坐标为中心, 寻找半径为1000km的城市
georadius china:city 110 30 1000 km
1) "xian"
2) "hangzhou"
3) "manjing"
4) "taiyuan"
#返回距离
georadius china:city 110 30 500 km withdist
1) 1) "xian"
   2) "483.8340"
#返回坐标，距离，最近2个
georadius china:city 110 30 1000 km withcoord withdist count 2
1) 1) "xian"
   2) "483.8340"
   3) 1) "108.96000176668167114"
      2) "34.25999964418929977"
2) 1) "manjing"
   2) "864.9816"
   3) 1) "118.75999957323074341"
      2) "32.03999960287850968"

#返回城市附近的城市
georadiusbymember china:city taiyuan 1000 km withcoord withdist count 2
1) 1) "taiyuan"
   2) "0.0000"
   3) 1) "112.54999905824661255"
      2) "37.86000073876942196"
2) 1) "xian"
   2) "514.2264"
   3) 1) "108.96000176668167114"
      2) "34.25999964418929977"

#返回geohash
geohash china:city taiyuan shenyang
1) "ww8p3hhqmp0"
2) "wxrvb9qyxk0"
```

* 消息队列

```redis
# *号表示服务器自动生成ID，后面顺序跟着一堆key/value
# 名字叫laoqian，年龄30岁
xadd codehole * name laoqian age 30
1527849609889-0  # 生成的消息ID，时间戳+当前毫秒第几个消息
xadd codehole * name xiaoyu age 29
1527849629172-0
xlen codehole       # 2 队列长度

#列出一段范围的消息，-表示最小值, +表示最大值
xrange codehole - +
1) 1) 1527849609889-0
   1) 1) "name"
      1) "laoqian"
      2) "age"
      3) "30"
2) 1) 1527849629172-0
   1) 1) "name"
      1) "xiaoyu"
      2) "age"
      3) "29"
xrange codehole 1527849629172-0 +
xrange codehole - 1527849629172-0
xdel codehole 1527849609889-0
xlen codehole  # del后，长度不受影响，依然是2
xrange codehole - + #但是列举时消息不会再出现
del codehole    #删除队列

#单独消费，等于一个普通list
xread count 2 streams codehole 0-0  #从Stream头部读取两条消息
#从Stream尾部读取一条消息，在最后了所以不会返回任何消息
xread count 1 streams codehole $
#从尾部阻塞等待新消息，block 1000表示阻塞1s，0表示一直等待
xread block 0 count 1 streams codehole $

#组消费
xgroup create codehole cg1 0-0  #表示从头开始消费
xgroup create codehole cg2 $    #从尾部开始，只接受新消息
# >号表示从当前消费组last_delivered_id开始读
xreadgroup GROUP cg1 c1 count 1 streams codehole >
# 阻塞等读
xreadgroup GROUP cg1 c1 block 0 count 2 streams codehole >
# ack消息
xack codehole cg1 1527851486781-0

# 列出pending状态
xpending codehole cg1
# 可以列出具体信息
xpending codehole cg1 - + 10
1) 1) "1553585533795-1"
   2) "consumerA"
   3) (integer) 15907787    #过了15907787ms了
   4) (integer) 4           #消息被读取了4次
# 列出具体消费者pending的信息
xpending codehole cg1 - + 10 consumerA

# 转移超时消息
#超3600s的消息1553585533795-1到消费者B的Pending列表
#转移后的超时时间会被重置，读取次数会+1，以避免反复转移
xclaim codehole cg1 consumerB 3600000 1553585533795-1
#死信处理，可以判断消息被读取次数，过多就是无法处理

#获取流信息
xinfo stream codehole
 1) length          #流长度2
 2) (integer) 2
 3) radix-tree-keys
 4) (integer) 1
 5) radix-tree-nodes
 6) (integer) 2
 7) groups
 8) (integer) 2  # 两个消费组
 9) first-entry  # 第一个消息
10) 1) 1527851486781-0
    2) 1) "name"
       2) "laoqian"
       3) "age"
       4) "30"
11) last-entry  # 最后一个消息
12) 1) 1527851498956-0
    2) 1) "name"
       2) "xiaoqian"
       3) "age"
       4) "1"

# 获取消费组信息
xinfo groups codehole
1) 1) name
   2) "cg1"
   3) consumers
   4) (integer) 0  # 该消费组还没有消费者
   5) pending
   6) (integer) 0  # 该消费组没有正在处理的消息
2) 1) name
   2) "cg2"
   3) consumers  # 该消费组还没有消费者
   4) (integer) 0
   5) pending
   6) (integer) 0  # 该消费组没有正在处理的消息

#查看消费组中消费者信息
xinfo consumers codehole cg1

#查看内存信息
MEMORY USAGE <key> [SAMPLES <count>]
MEMORY STATS
#内存gc
MEMORY PURGE
#查看分配器状态
MEMORY MALLOC-STATS
```

## 事务

* 用Multi(Start Transaction)、Exec(Commit)、Discard(Rollback)实现。
* 其实只是批量执行一堆命令，保证执行过程不会有其他操作（单线程），无法保证所有成功执行（原子性），中途出错也无法回滚。
* Watch指令：类似乐观锁，事务提交时，如果Key的值已被别的客户端改变，比如某个list已被别的客户端push/pop过了，整个事务队列都不会被执行。
* pipeline也类似事务的效果，多个命令一起发送，一起查询返回结果。

## 过期key清除策略

注意每次清除的key是随机抽样的，例如每次找三个key，删最旧的key这样，all和volatile只是每次在不同地方找key来删除。

* noeviction：不淘汰。
* allkeys-*：所有key
* volatile-*：设置了过期时间的key
* allkeys-random/volatile-random：随机删除。
* allkeys-ttl/volatile-ttl：随机maxmemory-samples个按时间删除
* allkeys-lru/volatile-lru：随机maxmemory-samples个按最旧更新删除
* allkeys-lfu/volatile-lfu：随机maxmemory-samples个按最少访问删除，Redis4.0

Reids 默认采用 惰性删除 和 定时删除 的结合。设置了过期的key放到一个独立字典中，默认每100ms进行一次过期扫描：

* 随机抽取 20 个 key
* 删除这 20 个key中过期的key
* 如果过期的 key 比例超过 1/4，就重复步骤 1，继续删除
* 最多单次运行时间25ms
* 从库不单独扫描，直接同步del删除结果

## 持久化

Redis本身持久化机制无法避免少量数据丢失问题。

* RDB持久化：全量快照。save命令触发，阻塞服务器直到RDB完成线上环境不建议使用，bgsave命令触发：通过Fork子进程进行（过程中可保证数据不会修改copy-on-write），还有自动触发：
  * redis.conf中配置save m n，
  * 主从复制，从节点要从主节点进行全量复制时
  * 执行debug reload命令重新加载redis时
  * 默认情况下执行shutdown时，如没开启aof持久化，也会触发bgsave 
* AOF持久化：AppendOnlyFile，增量写操作记录保存到文件中，AOF重写用于刷新无效记录，相当于压缩aof文件。
* 虚拟内存（VM）方式：不建议使用。
* DISKSTORE：不建议使用。
* 主从备份：2.8之前是全量备份，2.8之后支持增量。
  * 如果主服务器没有持久化，应禁用实例自动重启，因为重启后空库会快速同步给从库。
  * 全量复制使用RDB效率更高，AOF平时也会耗费性能。
  * 2.8.18之后支持无磁盘复制，适合硬盘较慢场景，直接发送给从库，参考repl-diskless-sync

## 哨兵模式 Redis Sentinel

* 主要解决主从复制模式下故障转移问题
* 哨兵集群主要功能，监控主从节点是否正常，自动故障转移，提供配置（客户端初始化时，通过哨兵获得主节点），通知（通知故障转移结果给客户端）
* 建议哨兵是三机器，通过主集群的__sentinel__:hello频道订阅实现互相发现，通过Raft进行选举。
* 哨兵通过info得知集群状态，切换时选择salve-priority和同步状态最快的节点
* 主观下线：任何一个哨兵监控探测，并作出下线判断；客观下线：哨兵集群共同决定节点下线；

## 分片 Redis Cluster

* Redis3.0。最多1000节点。
* 集群中没有代理，节点间异步复制，没有归并操作
* 无法保证强一致（异步复制），failover时有少量数据丢失，脑裂时会造成少数方master写入数据丢失（数据以最后failover节点为准），因此不建议在多个机房部署redis集群，脑裂不好解决。
* 节点间使用tcp gossip协议通信，端口是对外端号+10000。
* 可用性：以下场景集群可用：
  * 大部分master可用，少部分不可用master至少有一个可用slave。
  * 更进一步，通过使用 replicas migration 技术，当前没有slave的master会从拥有多个slave的master接受新slave来确保可用性，集群有更高的健壮性。
* multi-key写入需要都在本slot下，因此建议程序使用hashtag定义key保证同一分片，key中的大括号对内容为hash内容，如：{user}.following。
* Redis集群没有使用一致性hash，而是引入哈希槽（Hash Slot）概念，共有16384个，key通过CRC16后确认放置槽位。
则可能存在的一种分配如下： 节点A包含0到5500号哈希槽； 节点B包含5501到11000号哈希槽； 节点C包含11001 到 16384号哈希槽。
* 管理员可在任意节点中声明新节点：CLUSTER MEET ip port
* 客户端访问数据不在本节点时，会返回重定向信息，客户端再发起正确连接。
  * MOVED重定向：告诉客户端需要重连节点信息
  * ASK重定向：集群伸缩时，告知客户端新节点信息，需要向新节点重新咨询
* 不建议在集群模式使用订阅，因为消息需要在所有节点间同步。
* 已经淘汰方案：
  * Redis Sentinel 集群 + Keepalived/Haproxy
  * Twemproxy
  * Codis


## Lua Script

* 避免Eval每次传输完整Script，先用Script Load载入script得到哈希值后用EvalHash
* Lua中使用CJSON处理json。
* 典型场景数据过滤和数据聚合。
* 很难进行错误报告和处理，使用Redis的Pub/Sub功能，推送日志再自行处理。

```lua
- 用Redis做定时任务，script定期执行，检查触发时间为score的sorted set中，取出到期任务放到list中给订阅Client blocking popup。
— KEYS: [1]job:sleeping, [2]job:ready
— ARGS: [1]currentTime
— Comments: result is the job id
local jobs=redis.call(‘zrangebyscore’, KEYS[1], ‘-inf’, ARGV[1])
local count = table.maxn(jobs)
 
if count>0  then
  — Comments: remove from Sleeping Job sorted set
  redis.call(‘zremrangebyscore’, KEYS[1], ‘-inf’, ARGV[1])
 
  — Comments: add to the Ready Job list
  — Comments: can optimize to use lpush id1,id2,… for better performance
  for i=1,count do
    redis.call(‘lpush’, KEYS[2], jobs[i])
  end
end
```

## 键空间通知 keyspace notification

* 通过redis.conf指定notify-keyspace-events类型，或CONFIG SET命令启用，可选参数如下，K或E是总开关，AKE为发送所有通知：
    * K：键空间通知，可只监听对应key，__keyspace@<db>__ 为前缀，如：PUBLISH __keyspace@0__:mykey del
    * E：键事件通知，可只监听对应事件，__keyevent@<db>__ 为前缀，如：PUBLISH __keyevent@0__:del mykey
    * g：DEL、EXPIRE、RENAME等类型无关的通用命令的通知
    * $：字符串命令的通知
    * l：列表命令的通知
    * s：集合命令的通知
    * h：哈希命令的通知
    * z：有序集合命令的通知
    * x：过期键被删除时发送，会可能延迟，触发时机：当键被访问且已过期；系统后台渐进查找删除过期键。
    * e：驱逐(evict)事件：因 maxmemory 政策被删除时发送
    * A：等于 g$lshzxe
* 键只有真正修改了才有通知，使用发送即忘（fire and forget）策略
* 订阅全部事件 psubscribe '__key*__:*'
* redis 2.8.0 后才有
* 参考 http://redisdoc.com/topic/notification.html

## 常用配置

```
daemonize no
pidfile /var/run/redis.pid
port 6379
bind 127.0.0.1
#客户端闲置多长时间后关闭连接，如果指定为0，表示关闭该功能
timeout 300
#日志级别debug、verbose、notice、warning，默认为verbose
loglevel verbose
#日志记录方式，默认为标准输出，如守护方式运行且该值为stdout，输出到/dev/null
logfile stdout
#数据库数量，默认数据库为0，可以使用SELECT <dbid>命令在连接上指定数据库id
databases 16

#rdb相关
#指定存盘动作，可多条
#60秒内10000个更改，300秒内10个更改，900秒（15分钟）内1个更改触发存盘
save 900 1
save 300 10
save 60 10000
#本地数据库是否压缩，默认yes，LZF压缩
rdbcompression yes
#本地数据库文件名，默认值为dump.rdb
dbfilename dump.rdb
#本地数据库存放目录
dir ./
#当bgsave无法成功时，是否造成写出错（让管理人员尽快发现）
stop-writes-on-bgsave-error no

#本机为slave时设置master端口信息，启动时会自动从master同步
slaveof <masterip> <masterport>
#master密码
masterauth <master-password>
#Redis连接密码，客户端连接Redis时需要AUTH <password>，默认关闭
requirepass foobared
#最大客户端连接数，默认无限制0。连接数限制时会关闭新的连接并向客户端返回max number of clients reached错误
maxclients 128

#aof相关
#是否每次更新操作后日志记录，默认异步写盘，如不开启可能会断电时数据丢失。默认为no
appendonly no
#指定更新日志文件名，默认为appendonly.aof
appendfilename appendonly.aof
#指定更新日志条件：
# no：操作系统数据缓存同步写盘（快） 
# always：每次更新后调用fsync()强制写盘（慢，安全） 
# everysec：每秒写盘（折衷，默认值）
appendfsync everysec
# aof重写期间是否同步
no-appendfsync-on-rewrite no
# aof重写触发配置
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
# 加载aof出错如何处理
aof-load-truncated yes
# 文件重写策略
aof-rewrite-incremental-fsync yes

#最大内存限制，数据达到后先尝试清除已到期或即将到期的Key，如仍到达最大内存设置，将无法写入，但可读取。Redis新vm机制会把Key放内存Value放swap区
maxmemory <bytes>
#是否启用虚拟内存机制，默认no，VM机制将数据分页存放，Redis将冷数据页swap到磁盘，热页由磁盘自动换出到内存中
vm-enabled no
#虚拟内存文件路径，默认值为/tmp/redis.swap，不可多Redis实例共享
vm-swap-file /tmp/redis.swap
#大于vm-max-memory数据存入虚拟内存，索引数据总在内存，vm-max-memory默认为0，即value都存磁盘。
vm-max-memory 0
#swap文件分很多page，一对象可存多个page上，一page不能被多对象共享，建议存储很多小对象，page大小设置为32或64bytes；如果存储很大对象，可以使用更大page，不确定就使用默认值
vm-page-size 32
#swap文件page数量，页表（bitmap）放内存，每8个pages消耗1byte内存
vm-pages 134217728
#访问swap文件线程,不建议超过机器核数,为0表示对swap文件操作都串行，可能造成较长延迟。默认值为4
vm-max-threads 4
#是否合并返回客户端的较小包，默认开启
glueoutputbuf yes
#超过一定数量或者最大元素超过临界值时，采用特殊哈希算法
hash-max-zipmap-entries 64
hash-max-zipmap-value 512
#是否激活重置哈希，默认开启
activerehashing yes
#指定包含其它的配置文件
include /path/to/local.conf

# 4.0新增，开启自动内存碎片整理(总开关)
activedefrag yes
# 当碎片达到 100mb 时，开启内存碎片整理
active-defrag-ignore-bytes 100mb
# 当碎片超过 10% 时，开启内存碎片整理
active-defrag-threshold-lower 10
# 内存碎片超过 100%，则尽最大努力整理
active-defrag-threshold-upper 100
# 内存自动整理占用资源最小百分比
active-defrag-cycle-min 25
# 内存自动整理占用资源最大百分比
active-defrag-cycle-max 75

```

## 参考链接：

* https://pdai.tech/md/db/nosql-redis/db-redis-introduce.html
