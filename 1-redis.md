第一章 REDIS

一 redis配置daemon进程并开机自启动，适用生产环境
①redis utils目录下有个redis_init_script脚本
②将redis_init_script拷贝到/etc/init.d/目录下并重命名为redis_6379
③修改redis_6379中的REDISPORT为6379
④创建两个目录 /etc/redis 存放redis配置文件redis.conf,/var/redis/6379 存放redis持久化文件
⑤修改redis配置文件redis.conf并放到/etc/redis下
 daemon yes 后台进程
 pidfile /var/run/redis_6379.pid
 port 6379
 dir /var/redis/6379 #持久化文件存放路径
 ⑥启动redis /etc/init.d/redis_6379
 ⑦开机自启动 chkconfig redis_6379 on
 
二 redis持久化机制
RDB  持久化机制，对redis中的数据执行周期性的持久化
AOF  机制对每条写入命令作为日志，以append-only的模式写入一个日志文件中，在redis重启的时候，可以通过回放AOF日志中的写入指令来重新构建整个数据集
如果我们想要redis仅仅作为纯内存的缓存来用，那么可以禁止RDB和AOF所有的持久化机制
通过RDB或AOF，都可以将redis内存中的数据给持久化到磁盘上面来，然后可以将这些数据备份到别的地方去，比如说阿里云，云服务
如果redis挂了，服务器上的内存和磁盘上的数据都丢了，可以从云服务上拷贝回来之前的数据，放到指定的目录中，然后重新启动redis，redis就会自动根据持久化数据文件中的数据，去恢复内存中的数据，继续对外提供服务
如果同时使用RDB和AOF两种持久化机制，那么在redis重启的时候，会使用AOF来重新构建数据，因为AOF中的数据更加完整

优劣对比：
RDB持久化机制的优点
（1）RDB会生成多个数据文件，每个数据文件都代表了某一个时刻中redis的数据，这种多个数据文件的方式，非常适合做冷备，可以将这种完整的数据文件发送到一些远程的安全存储上去，比如说Amazon的S3云服务上去，在国内可以是阿里云的ODPS分布式存储上，以预定好的备份策略来定期备份redis中的数据
（2）RDB对redis对外提供的读写服务，影响非常小，可以让redis保持高性能，因为redis主进程只需要fork一个子进程，让子进程执行磁盘IO操作来进行RDB持久化即可
（3）相对于AOF持久化机制来说，直接基于RDB数据文件来重启和恢复redis进程，更加快速
RDB持久化机制的缺点
（1）如果想要在redis故障时，尽可能少的丢失数据，那么RDB没有AOF好。一般来说，RDB数据快照文件，都是每隔5分钟，或者更长时间生成一次，这个时候就得接受一旦redis进程宕机，那么会丢失最近5分钟的数据
（2）RDB每次在fork子进程来执行RDB快照数据文件生成的时候，如果数据文件特别大，可能会导致对客户端提供的服务暂停数毫秒，或者甚至数
AOF持久化机制的优点
（1）AOF可以更好的保护数据不丢失，一般AOF会每隔1秒，通过一个后台线程执行一次fsync操作，最多丢失1秒钟的数据
（2）AOF日志文件以append-only模式写入，所以没有任何磁盘寻址的开销，写入性能非常高，而且文件不容易破损，即使文件尾部破损，也很容易修复
（3）AOF日志文件即使过大的时候，出现后台重写操作，也不会影响客户端的读写。因为在rewrite log的时候，会对其中的指导进行压缩，创建出一份需要恢复数据的最小日志出来。再创建新日志文件的时候，老的日志文件还是照常写入。当新的merge后的日志文件ready的时候，再交换新老日志文件即可。
（4）AOF日志文件的命令通过非常可读的方式进行记录，这个特性非常适合做灾难性的误删除的紧急恢复。比如某人不小心用flushall命令清空了所有数据，只要这个时候后台rewrite还没有发生，那么就可以立即拷贝AOF文件，将最后一条flushall命令给删了，然后再将该AOF文件放回去，就可以通过恢复机制，自动恢复所有数据
AOF持久化机制的缺点
（1）对于同一份数据来说，AOF日志文件通常比RDB数据快照文件更大
（2）AOF开启后，支持的写QPS会比RDB支持的写QPS低，因为AOF一般会配置成每秒fsync一次日志文件，当然，每秒一次fsync，性能也还是很高的
（3）以前AOF发生过bug，就是通过AOF记录的日志，进行数据恢复的时候，没有恢复一模一样的数据出来。所以说，类似AOF这种较为复杂的基于命令日志/merge/回放的方式，比基于RDB每次持久化一份完整的数据快照文件的方式，更加脆弱一些，容易有bug。不过AOF就是为了避免rewrite过程导致的bug，
    因此每次rewrite并不是基于旧的指令日志进行merge的，而是基于当时内存中的数据进行指令的重新构建，这样健壮性会好很多。

1、如何配置RDB持久化机制
redis.conf文件，也就是/etc/redis/6379.conf，去配置持久化
save 60 1000
每隔60s，如果有超过1000个key发生了变更，那么就生成一个新的dump.rdb文件，就是当前redis内存中完整的数据快照，这个操作也被称之为snapshotting，快照
也可以手动调用save或者bgsave命令，同步或异步执行rdb快照生成
save可以设置多个，就是多个snapshotting检查点，每到一个检查点，就会去check一下，是否有指定的key数量发生了变更，如果有，就生成一个新的dump.rdb文件
2、RDB持久化机制的工作流程
（1）redis根据配置自己尝试去生成rdb快照文件
（2）fork一个子进程出来
（3）子进程尝试将数据dump到临时的rdb快照文件中
（4）完成rdb快照文件的生成之后，就替换之前的旧的快照文件
dump.rdb，每次生成一个新的快照，都会覆盖之前的老快照

3、基于RDB持久化机制的数据恢复实验
（1）在redis中保存几条数据，立即停掉redis进程，然后重启redis，看看刚才插入的数据还在不在
数据还在，为什么？
带出来一个知识点，通过redis-cli SHUTDOWN这种方式去停掉redis，其实是一种安全退出的模式，redis在退出的时候会将内存中的数据立即生成一份完整的rdb快照
/var/redis/6379/dump.rdb
（2）在redis中再保存几条新的数据，用kill -9粗暴杀死redis进程，模拟redis故障异常退出，导致内存数据丢失的场景
这次就发现，redis进程异常被杀掉，数据没有进dump文件，几条最新的数据就丢失了
（2）手动设置一个save检查点，save 5 1
（3）写入几条数据，等待5秒钟，会发现自动进行了一次dump rdb快照，在dump.rdb中发现了数据
（4）异常停掉redis进程，再重新启动redis，看刚才插入的数据还在
rdb的手动配置检查点，以及rdb快照的生成，包括数据的丢失和恢复，全都演示过了

1、AOF持久化的配置
AOF持久化，默认是关闭的，默认是打开RDB持久化
appendonly yes，可以打开AOF持久化机制，在生产环境里面，一般来说AOF都是要打开的，除非你说随便丢个几分钟的数据也无所谓
打开AOF持久化机制之后，redis每次接收到一条写命令，就会写入日志文件中，当然是先写入os cache的，然后每隔一定时间再fsync一下
而且即使AOF和RDB都开启了，redis重启的时候，也是优先通过AOF进行数据恢复的，因为aof数据比较完整
可以配置AOF的fsync策略，有三种策略可以选择，一种是每次写入一条数据就执行一次fsync; 一种是每隔一秒执行一次fsync; 一种是不主动执行fsync
always: 每次写入一条数据，立即将这个数据对应的写日志fsync到磁盘上去，性能非常非常差，吞吐量很低; 确保说redis里的数据一条都不丢，那就只能这样了
mysql -> 内存策略，大量磁盘，QPS到多少，一两k。QPS，每秒钟的请求数量
redis -> 内存，磁盘持久化，QPS到多少，单机，一般来说，上万QPS没问题
everysec: 每秒将os cache中的数据fsync到磁盘，这个最常用的，生产环境一般都这么配置，性能很高，QPS还是可以上万的
no: 仅仅redis负责将数据写入os cache就撒手不管了，然后后面os自己会时不时有自己的策略将数据刷入磁盘，不可控了
2、AOF持久化的数据恢复实验
（1）先仅仅打开RDB，写入一些数据，然后kill -9杀掉redis进程，接着重启redis，发现数据没了，因为RDB快照还没生成
（2）打开AOF的开关，启用AOF持久化
（3）写入一些数据，观察AOF文件中的日志内容
其实你在appendonly.aof文件中，可以看到刚写的日志，它们其实就是先写入os cache的，然后1秒后才fsync到磁盘中，只有fsync到磁盘中了，才是安全的，要不然光是在os cache中，机器只要重启，就什么都没了
（4）kill -9杀掉redis进程，重新启动redis进程，发现数据被恢复回来了，就是从AOF文件中恢复回来的
redis进程启动的时候，直接就会从appendonly.aof中加载所有的日志，把内存中的数据恢复回来
3、AOF rewrite
redis中的数据其实有限的，很多数据可能会自动过期，可能会被用户删除，可能会被redis用缓存清除的算法清理掉
redis中的数据会不断淘汰掉旧的，就一部分常用的数据会被自动保留在redis内存中
所以可能很多之前的已经被清理掉的数据，对应的写日志还停留在AOF中，AOF日志文件就一个，会不断的膨胀，到很大很大
所以AOF会自动在后台每隔一定时间做rewrite操作，比如日志里已经存放了针对100w数据的写日志了; redis内存只剩下10万; 基于内存中当前的10万数据构建一套最新的日志，到AOF中; 覆盖之前的老日志; 
确保AOF日志文件不会过大，保持跟redis内存数据量一致
redis 2.4之前，还需要手动，开发一些脚本，crontab，通过BGREWRITEAOF命令去执行AOF rewrite，但是redis 2.4之后，会自动进行rewrite操作
在redis.conf中，可以配置rewrite策略
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
比如说上一次AOF rewrite之后，是128mb
然后就会接着128mb继续写AOF的日志，如果发现增长的比例，超过了之前的100%，256mb，就可能会去触发一次rewrite
但是此时还要去跟min-size，64mb去比较，256mb > 64mb，才会去触发rewrite
（1）redis fork一个子进程
（2）子进程基于当前内存中的数据，构建日志，开始往一个新的临时的AOF文件中写入日志
（3）redis主进程，接收到client新的写操作之后，在内存中写入日志，同时新的日志也继续写入旧的AOF文件
（4）子进程写完新的日志文件之后，redis主进程将内存中的新日志再次追加到新的AOF文件中
（5）用新的日志文件替换掉旧的日志文件
4、AOF破损文件的修复
如果redis在append数据到AOF文件时，机器宕机了，可能会导致AOF文件破损
用redis-check-aof --fix命令来修复破损的AOF文件
5、AOF和RDB同时工作
（1）如果RDB在执行snapshotting操作，那么redis不会执行AOF rewrite; 如果redis再执行AOF rewrite，那么就不会执行RDB snapshotting
（2）如果RDB在执行snapshotting，此时用户执行BGREWRITEAOF命令，那么等RDB快照生成之后，才会去执行AOF rewrite
（3）同时有RDB snapshot文件和AOF日志文件，那么redis重启的时候，会优先使用AOF进行数据恢复，因为其中的日志更完整
6、最后一个小实验，让大家对redis的数据恢复有更加深刻的体会
（1）在有rdb的dump和aof的appendonly的同时，rdb里也有部分数据，aof里也有部分数据，这个时候其实会发现，rdb的数据不会恢复到内存中
（2）我们模拟让aof破损，然后fix，有一条数据会被fix删除
（3）再次用fix得aof文件去重启redis，发现数据只剩下一条了
数据恢复完全是依赖于底层的磁盘的持久化的，主要rdb和aof上都没有数据，那就没了