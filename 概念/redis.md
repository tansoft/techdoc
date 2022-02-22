
## 特性

* 常用命令复杂度O(1)
* 

## 常用结构

* Hash：Key-HashMap结构，可只修改map里某个属性互不冲突。
* List：双向链表，支持双向Pop/Push。
* Set：去重，底层是hash table。
* Sorted Set：有序集，通过分数排序。实现是hash table(去重)加skip list(score->element,排序)，skip list像平衡二叉树，不同范围score分成多层，每层是按score排序链表。

## 常用命令

* Incr/IncrBy/IncrByFloat/Decr/DecrBy：计数器场景，key不存在会创建并设原值为0。IncrByFloat专门针对float，decr用负数。

* SetNx：仅当key不存在时才Set。用于选主或分布式锁：所有Client不断抢注Master，成功后不断使用Expire刷新过期时间。如果Master掉了key就会失效，剩下的节点发生新一轮抢夺。

* LPush/RPop/RPush/LPop/BLPop/BRPop：常用LPush/RPop，可用阻塞版本BLPop/BRPop一直等消息。
* RPopLPush/BRPopLPush：弹出来同时推入另一个list，做状态机。
* LLen获取列表的长度。

* ZAdd/ZRem是O(log(N))，ZRangeByScore/ZRemRangeByScore是O(log(N)+M)，N是Set大小，M是结果/操作元素的个数。N 1000万复杂度才几十，但是M不能太多影响性能。

## Lua Script

* 避免Eval每次传输完整Script，先用Script Load载入script得到哈希值后用EvalHash
* Lua中使用CJSON处理json。

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
 

在Redis使用过程中，Lua脚本的支持无疑给开发者提供一个非常友好的开发环境，从而大幅度解放用户的创造力。如果使用得当，Lua脚本可以给性能和资源消耗带来非常大的改善。取代将数据传送给CPU，脚本允许你在最接近数据的地方执行逻辑，从而减少网络延时和数据的冗余传输。

在Redis中，Lua一个非常经典的用例就是数据过滤或者将数据聚合到应用程序。通过将处理工作流封装到一个脚本中，你只需要调用它就可以在更短的时间内使用很少的资源来获取一个更小的答案。

提示：Lua确实非常棒，但是同样也存在一些问题，比如很难进行错误报告和处理。一个明智的方法就是使用Redis的Pub/Sub功能，并且让脚本通过专用信道来推送日志消息。然后建立一个订阅者进程，并进行相应的处理。

1、使用hash取代将数据存储为数千（或者数百万）独立的字符串。哈希表是非常有效率的，并且可以减少你的内存使用（因为小Hashes会被编码成一个非常小的空间）；同时，哈希还更有益于细节抽象和代码可读。

2、合适时候，使用list代替set。如果你不需要使用set特性，List在使用更少内存的情况下可以提供比set更快的速度。

3、Sorted sets是最昂贵的数据结构，不管是内存消耗还是基本操作的复杂性。如果你只是需要一个查询记录的途径，并不在意排序这样的属性，那么轻建议使用哈希表。

4、Redis中一个经常被忽视的功能就是bitmaps或者bitsets（V2.2之后）。Bitsets允许你在Redis值上执行多个bit-level操作，比如一些轻量级的分析。

用Multi(Start Transaction)、Exec(Commit)、Discard(Rollback)实现。 在事务提交前，不会执行任何指令，只会把它们存到一个队列里，不影响其他客户端的操作。在事务提交时，批量执行所有指令。《Redis设计与实现》中的详述。

注意，Redis里的事务，与我们平时的事务概念很不一样：

它仅仅是保证事务里的操作会被连续独占的执行。因为是单线程架构，在执行完事务内所有指令前是不可能再去同时执行其他客户端的请求的。
它没有隔离级别的概念，因为事务提交前任何指令都不会被实际执行，也就不存在”事务内的查询要看到事务里的更新，在事务外查询不能看到”这个让人万分头痛的问题。
它不保证原子性——所有指令同时成功或同时失败，只有决定是否开始执行全部指令的能力，没有执行到一半进行回滚的能力。在redis里失败分两种，一种是明显的指令错误，比如指令名拼错，指令参数个数不对，在2.6版中全部指令都不会执行。另一种是隐含的，比如在事务里，第一句是SET foo bar， 第二句是LLEN foo，对第一句产生的String类型的key执行LLEN会失败，但这种错误只有在指令运行后才能发现，这时候第一句成功，第二句失败。还有，如果事务执行到一半redis被KILL，已经执行的指令同样也不会被回滚。
Watch指令，类似乐观锁，事务提交时，如果Key的值已被别的客户端改变，比如某个list已被别的客户端push/pop过了，整个事务队列都不会被执行。
————————————————
版权声明：本文为CSDN博主「hguisu」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/hguisu/article/details/90748695
