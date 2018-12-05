# redis
#### redis sentinel

##### 主从复制的问题
* 写能力和存储能力受限（只能在主节点写入，从节点只是主节点的副本）

####  安装配置
* 一主 二从配置
```
redis-7000.conf  master
port 7000
daemonize yes
pidfile /var/run/redis-7000.pid
logfile "7000.log"
dir "/root/redis/data"


redis-7001.conf   slave 1
port 7001
daemonize yes
pidfile /var/run/redis-7001.pid
logfile "7001.log"
dir "/root/redis/data"
slaveof 127.0.0.1 7000


 redis-7002.conf  slave 2 
port 7002
daemonize yes
pidfile /var/run/redis-7002.pid
logfile "7002.log"
dir "/root/redis/data"
slaveof 127.0.0.1 7000

```

* redis-sentinel 配置
```
redis-sentinel-26379.conf


port 26379
daemonize yes
dir "/root/redis-3.2.6/log"
logfile "26379.log"
#监控主节点的名字 ip 端口 2代表 至少有几个sentinel 发现主节点有问题了 故障发现
sentinel monitor mymaster 127.0.0.1 7000 2 
# 监听 3w毫秒 30s有问题之后开始转移
sentinel down-after-milliseconds mymaster 30000 
# 在发生failover主备切换时，这个选项指定了最多可以有多少个slave同时对新的master进行同步，这个数字越小，完成failover所需的时间就越长，但是如果这个数字越大，就意味着越多的slave因为replication而不可用。可以通过将这个值设为 1 来保证每次只有一个slave处于不能处理命令请求的状态。
sentinel parallel-syncs mymaster 1
#故障转移时间
sentinel failover-timeout mymaster 180000  

//生成 26380 和26381 的配置
sed "s/26379/26380/g" redis-sentinel-26379.conf > redis-sentinel-26380.conf
sed "s/26379/26381/g" redis-sentinel-26379.conf > redis-sentinel-26381.conf

//启动实例
src/redis-sentinel conf/redis-sentinel-26379.conf
src/redis-sentinel conf/redis-sentinel-26380.conf
src/redis-sentinel conf/redis-sentinel-26381.conf
```

##### 三个定时任务
* 每10s 每个sentinel 对master 和slave执行info
  * 发现slave 节点
  * 确认主从关系
* 每2s每个sentinel  通过master 节点的channel交换信息（pus/sub）
  * 通过_sentinel_:hello频道交互
  * 交互对节点的看法和自身信息
*  每1s每个sentinel对其他的sentinel和redis执行ping
   * 心跳检测，失败判断的依据  



######  主观下线 和 客观下线
* 主观下线:每个sentinel 节点对Redis 节点失败的偏见
* 客观下线:所有sentinel节点对Redis 节点失败达成共识

##### 领导者选举
* 只有一个sentinel节点完成故障转移
* 通过sentinel is-master-down-by-addr 命令都希望成为领导者
* 每个做主观下线的sentinel节点向其他sentinel节点发送命令，要求将他设置为领导者
* 收到命令的sentinel 节点如果没有同意通过其他sentinel节点发送的命令，那么将同意该请求，否则拒绝
* 如果该sentinel节点发现自己的票数已经超过sentinel集合半数且超过quorum，那么它将成为领导者
* 如果此过程有多个sentinel节点成为领导者，那么将等待一段时间重新进行选举



###### 故障转移（sentinel领导者节点决定）
* 从slave 节点中选出一个合适的节点作为新的master节点
* 对上面的slave节点执行slaveof no one 命令让其成为master
* 向剩余的slave节点发送命令，让它们成为新master节点的slave 节点 复制规则和parallel-syncs参数有关
* 更新对原来master节点配置slave ，并保持对齐关注当其恢复后命令它去复制新的master节点
* 

##### 选择合适的slave节点
* 选择slave-priority（slave节点优先级） 最高的slave节点，如果存在则返回，不存在则继续
* 选择复制偏移量最大的slave节点（复制的最完整），如果存在则返回，不存在则继续
* 选择runId 最小的slave节点
