##Redis默认不是以守护进程的方式运行，可以通过该配置项修改，使用yes启用守护进程
daemonize no

##当Redis以守护进程方式运行时，Redis默认会把pid写入/var/run/redis.pid文件，可以通过pidfile指定
pidfile /var/run/redis.pid

##指定Redis监听端口，默认端口为6379 

port 6379
#TCP接收队列长度，受/proc/sys/net/core/somaxconn和tcp_max_syn_backlog这两个内核参数的影响
tcp-backlog 511

##绑定的主机地址
bind 127.0.0.1

##当 客户端闲置多长时间后关闭连接，如果指定为0，表示关闭该功能
timeout 300

#如果非零，则设置SO_KEEPALIVE选项来向空闲连接的客户端发送ACK,用来定时向client发送tcp_ack包来探测client是否存活的
tcp-keepalive 60

##指定日志记录级别，Redis总共支持四个级别：debug、verbose、notice、warning，默认为verbose
##debug （大量信息，对开发/测试有用）
##verbose （很多精简的有用信息，但是不像debug等级那么多）
##notice （适量的信息，基本上是你生产环境中需要的）
##warning （只有很重要/严重的信息会记录下来）
loglevel verbose

##日志名
logfile "./redis7003.log"

##设置数据库的数量，可以使用SELECT <dbid>命令在连接上指定数据库id
databases 16

##持久化rdb文件，指定在多长时间内，有多少次更新操作，就将数据同步到数据文件，可以多个条件配合save <seconds> <changes>
#Redis默认配置文件中提供了三个条件：
#save 900 1
#save 300 10
#save 60 10000
#分别表示900秒（15分钟）内有1个更改，300秒（5分钟）内有10个更改以及60秒内有10000个更改。

#默认如果开启RDB快照(至少一条save指令)并且最新的后台保存失败，Redis将会停止接受写操作
#这将使用户知道数据没有正确的持久化到硬盘，否则可能没人注意到并且造成一些灾难
stop-writes-on-bgsave-error yes

##指定存储至本地数据库时是否压缩数据，默认为yes，Redis采用LZF压缩，如果为了节省CPU时间，可以关闭该选项，但会导致数据库文件变的巨大
rdbcompression yes

###指定本地数据库文件名，默认值为dump.rdb;除非非常要紧的数据否则尽量不要开启数据持久化
dbfilename dump.rdb

###指定本地数据库存放目录
dir ./

##设置当本机为slav服务时，设置master服务的IP地址及端口，在Redis启动时，它会自动从master进行数据同步
slaveof <masterip> <masterport>

###当master服务设置了密码保护时，slav服务连接master的密码
masterauth <master-password>

####设置Redis连接密码，如果配置了连接密码，客户端在连接Redis时需要通过AUTH <password>命令提供密码，默认关闭
requirepass foobared

##你可以配置salve实例是否接受写操作。可写的slave实例可能对存储临时数据比较有用(因为写入salve
##的数据在同master同步之后将很容易被删除
slave-read-only yes

#是否在slave套接字发送SYNC之后禁用 TCP_NODELAY？
#如果你选择“yes”Redis将使用更少的TCP包和带宽来向slaves发送数据。但是这将使数据传输到slave
#上有延迟，Linux内核的默认配置会达到40毫秒
#如果你选择了 "no" 数据传输到salve的延迟将会减少但要使用更多的带宽
repl-disable-tcp-nodelay no

#slave的优先级是一个整数展示在Redis的Info输出中。如果master不再正常工作了，哨兵将用它来
#选择一个slave提升=升为master。
#优先级数字小的salve会优先考虑提升为master，所以例如有三个slave优先级分别为10，100，25，
#哨兵将挑选优先级最小数字为10的slave。
#0作为一个特殊的优先级，标识这个slave不能作为master，所以一个优先级为0的slave永远不会被
#哨兵挑选提升为master
slave-priority 100

##设置同一时间最大客户端连接数，默认无限制，Redis可以同时打开的客户端连接数为Redis进程可以打开的最大文件描述符数，如果设置 maxclients 0，表示不作限制。当客户端连接数到达限制时，Redis会关闭新的连接并向客户端返回max number of clients reached错误信息
maxclients 128

##指定Redis最大内存限制，Redis在启动时会把数据加载到内存中，达到最大内存后，Redis会先尝试清除已到期或即将到期的Key，当此方法处理 后，仍然到达最大内存设置，将无法再进行写入操作，但仍然可以进行读取操作。Redis新的vm机制，会把Key存放内存，Value会存放在swap区
maxmemory <bytes>


##最大内存策略：如果达到内存限制了，Redis如何选择删除key。你可以在下面五个行为里选：
#volatile-lru -> 根据LRU算法删除带有过期时间的key。
#allkeys-lru -> 根据LRU算法删除任何key。
#volatile-random -> 根据过期设置来随机删除key, 具备过期时间的key。 
#allkeys->random -> 无差别随机删, 任何一个key。 
#volatile-ttl -> 根据最近过期时间来删除（辅以TTL）, 这是对于有过期时间的key 
#noeviction -> 谁也不删，直接在写操作时返回错误。
maxmemory-policy volatile-lru

##指定是否在每次更新操作后进行日志记录，Redis在默认情况下是异步的把数据写入磁盘，如果不开启，可能会在断电时导致一段时间内的数据丢失。因为 redis本身同步数据文件是按上面save条件来同步的，所以有的数据会在一段时间内只存在于内存中。默认为no
appendonly no

###指定操作日志文件名，目录为dir设置的目录，默认为appendonly.aof
appendfilename appendonly.aof

####指定更新日志条件，共有3个可选值： 
#no：表示等操作系统进行数据缓存同步到磁盘（快） 
#always：表示每次更新操作后手动调用fsync()将数据写到磁盘（慢，安全） 
#everysec：表示每秒同步一次（折衷，默认值）
appendfsync everysec

 

##指定是否启用虚拟内存机制，默认值为no，简单的介绍一下，VM机制将数据分页存放，由Redis将访问量较少的页即冷数据swap到磁盘上，访问多的页面由磁盘自动换出到内存中（在后面的文章我会仔细分析Redis的VM机制）
vm-enabled no

###虚拟内存文件路径，默认值为/tmp/redis.swap，不可多个Redis实例共享
vm-swap-file /tmp/redis.swap

#如果AOF的同步策略设置成 "always" 或者 "everysec"，并且后台的存储进程（后台存储或写入AOF
#日志）会产生很多磁盘I/O开销。某些Linux的配置下会使Redis因为 fsync()系统调用而阻塞很久。
#注意，目前对这个情况还没有完美修正，甚至不同线程的 fsync() 会阻塞我们同步的write(2)调用。
#为了缓解这个问题，可以用下面这个选项。它可以在 BGSAVE 或 BGREWRITEAOF 处理时阻止主进程进行fsync()。
#这就意味着如果有子进程在进行保存操作，那么Redis就处于"不可同步"的状态。
#这实际上是说，在最差的情况下可能会丢掉30秒钟的日志数据。（默认Linux设定）
#如果你有延时问题把这个设置成"yes"，否则就保持"no"，这是保存持久数据的最安全的方式。
no-appendfsync-on-rewrite yes

#自动重写AOF文件
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb

#AOF文件可能在尾部是不完整的（这跟system关闭有问题，尤其是mount ext4文件系统时
#没有加上data=ordered选项。只会发生在os死时，redis自己死不会不完整）。
#那redis重启时load进内存的时候就有问题了。
#发生的时候，可以选择redis启动报错，并且通知用户和写日志，或者load尽量多正常的数据。
#如果aof-load-truncated是yes，会自动发布一个log给客户端然后load（默认）。
#如果是no，用户必须手动redis-check-aof修复AOF文件才可以。
#注意，如果在读取的过程中，发现这个aof是损坏的，服务器也是会退出的，
#这个选项仅仅用于当服务器尝试读取更多的数据但又找不到相应的数据时。
aof-load-truncated yes

#Lua 脚本的最大执行时间，毫秒为单位
lua-time-limit 5000

#Redis慢查询日志可以记录超过指定时间的查询
slowlog-log-slower-than 10000

#这个长度没有限制。只是要主要会消耗内存。你可以通过 SLOWLOG RESET 来回收内存。
slowlog-max-len 128

#redis延时监控系统在运行时会采样一些操作，以便收集可能导致延时的数据根源。
#通过 LATENCY命令 可以打印一些图样和获取一些报告，方便监控
#这个系统仅仅记录那个执行时间大于或等于预定时间（毫秒）的操作, 
#这个预定时间是通过latency-monitor-threshold配置来指定的，
#当设置为0时，这个监控系统处于停止状态
latency-monitor-threshold 0

#Redis能通知 Pub/Sub 客户端关于键空间发生的事件，默认关闭
notify-keyspace-events ""

#当hash只有少量的entry时，并且最大的entry所占空间没有超过指定的限制时，会用一种节省内存的
#数据结构来编码。可以通过下面的指令来设定限制
hash-max-ziplist-entries 512
hash-max-ziplist-value 64

#与hash似，数据元素较少的list，可以用另一种方式来编码从而节省大量空间。
#这种特殊的方式只有在符合下面限制时才��以用
list-max-ziplist-entries 512
list-max-ziplist-value 64

#set有一种特殊编码的情况：当set数据全是十进制64位有符号整型数字构成的字符串时。
#下面这个配置项就是用来设置set使用这种编码来节省内存的最大长度。
set-max-intset-entries 512

#与hash和list相似，有序集合也可以用一种特别的编码方式来节省大量空间。
#这种编码只适合长度和元素都小于下面限制的有序集合
zset-max-ziplist-entries 128
zset-max-ziplist-value 64

#HyperLogLog稀疏结构表示字节的限制。该限制包括
#16个字节的头。当HyperLogLog使用稀疏结构表示
#这些限制，它会被转换成密度表示。
#值大于16000是完全没用的，因为在该点
#密集的表示是更多的内存效率。
#建议值是3000左右，以便具有的内存好处, 减少内存的消耗
hll-sparse-max-bytes 3000

#启用哈希刷新，每100个CPU毫秒会拿出1个毫秒来刷新Redis的主哈希表（顶级键值映射表）
activerehashing yes

#客户端的输出缓冲区的限制，可用于强制断开那些因为某种原因从服务器读取数据的速度不够快的客户端
client-output-buffer-limit normal 0 0 0
client-output-buffer-limit slave 256mb 64mb 60
client-output-buffer-limit pubsub 32mb 8mb 60

#默认情况下，“hz”的被设定为10。提高该值将在Redis空闲时使用更多的CPU时，但同时当有多个key
#同时到期会使Redis的反应更灵敏，以及超时可以更精确地处理
hz 10

#当一个子进程重写AOF文件时，如果启用下面的选项，则文件每生成32M数据会被同步
aof-rewrite-incremental-fsync yes


#将所有大于vm-max-memory的数据存入虚拟内存,无论vm-max-memory设置多小,所有索引数据都是内存存储的(Redis的索引数据 就是keys),也就是说,当vm-max-memory设置为0的时候,其实是所有value都存在于磁盘。默认值为0
vm-max-memory 0

#Redis swap文件分成了很多的page，一个对象可以保存在多个page上面，但一个page上不能被多个对象共享，vm-page-size是要根据存储的 数据大小来设定的，作者建议如果存储很多小对象，page大小最好设置为32或者64bytes；如果存储很大大对象，则可以使用更大的page，如果不 确定，就使用默认值
vm-page-size 32

#设置swap文件中的page数量，由于页表（一种表示页面空闲或使用的bitmap）是在放在内存中的，，在磁盘上每8个pages将消耗1byte的内存。
vm-pages 134217728

#设置访问swap文件的线程数,最好不要超过机器的核数,如果设置为0,那么所有对swap文件的操作都是串行的，可能会造成比较长时间的延迟。默认值为4
vm-max-threads 4

#设置在向客户端应答时，是否把较小的包合并为一个包发送，默认为开启
glueoutputbuf yes

#指定在超过一定的数量或者最大的元素超过某一临界值时，采用一种特殊的哈希算法
hash-max-zipmap-entries 64
hash-max-zipmap-value 512

#指定是否激活重置哈希，默认为开启（后面在介绍Redis的哈希算法时具体介绍）
activerehashing yes

#指定包含其它的配置文件，可以在同一主机上多个Redis实例之间使用同一份配置文件，而同时各个实例又拥有自己的特定配置文件
include /path/to/local.conf

#指定内存映射文件路径（windows版特有的一个文件，注意将该文件路径配置在一个足够空间的路径下）    
heapdir ./rdb/

#指定内存映射文件大小，如果设置了最大内存那么该文件的大小=1.5*最大内存大小
maxheap 1024000000