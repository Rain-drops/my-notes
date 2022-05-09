一、单例

> [root@master ~]# tar -zxvf redis-6.2.4.tar.gz 
>
> [root@master ~]# cd redis
>
> [root@master ~]# make
>
> [root@master ~]# cd src
>
> [root@master ~]# make install
>
> [root@master ~]# mkdir -p /usr/local/redis/bin /usr/local/redis/etc
>
> [root@master ~]# mv mkreleasehdr.sh redis-benchmark redis-check-aof redis-check-rdb redis-cli redis-server /usr/local/redis/bin/
>
> [root@master ~]# mv ../redis.conf ../sentinel.conf /usr/local/redis/etc/
>
> #守护进程
>
> [root@master ~]# sed -i '/^daemonize/cdaemonize yes' /usr/local/redis/etc/redis.conf
>
> [root@master ~]# sed -i '/^bind/cprotected-mode yes' /usr/local/redis/etc/redis.conf
>
> [root@master ~]# sed -i '/^protected-mode/cprotected-mode yes' /usr/local/redis/etc/redis.conf
>
> [root@master ~]# touch /etc/profile.d/redis.sh 
>
> [root@master ~]# echo 'export PATH=$PATH:/usr/local/redis/bin' > /etc/profile.d/redis.sh 
>
> [root@master ~]# source /etc/profile
>
> [root@master ~]# redis-server /usr/local/redis/etc/redis.conf

二、主从

> #旧版本
>
> [root@slave ~]# echo 'slaveof master 6379' > /usr/local/redis/tec/redis.conf
>
> #新版本
>
> 127.0.0.1:6379> replicaof master 6379

三、哨兵

> [root@sentinel ~]# vim /usr/local/redis/etc/sentinel.conf
>
> > protected-mode no
> >
> > daemonize yes
> >
> > port 26379
> >
> > #后边的1代表的：如果有俩个哨兵判断这个主节点挂了那这个主节点就挂了，通常设置为哨兵个数一半加一
> >
> > sentinel monitor mymaster 192.168.196.140 6379 1
> >
> > #sentinel auth-pass mymaster 123456
> >
> > #在故障转移时，最多有多少从节点对新的主节点进行同步。
> >
> > #这个值越小完成故障转移的时间就越长，这个值越大就意味着越多的从节点因为同步数据而暂时阻塞不可用
> >
> > sentinel parallel-syncs mymaster 1
> >
> > #5秒内master 没有响应，就认为DOWN
> >
> > sentinel down-after-milliseconds mymaster 5000 
> >
> > sentinel failover-timeout mymaster 15000
> >
> > logfile /var/log/redis/sentinel.log
> >
> > pidfile /var/run/sentinel.pid
>
> [root@sentinel ~]# redis-sentinel /usr/local/redis/tec/sentinel.conf

四、集群（三主三从）

> [root@master ~]# yum install ruby rubygems -y
>
> [root@master ~]# vim /usr/local/redis/etc/redis.conf
>
> > #开启集群
> >
> > cluster-enabled yes
> >
> > #集群节点配置文件，这个文件是不能手动编辑的。确保每一个集群节点的配置文件不同
> >
> > cluster-config-file nodes.conf
> >
> > #集群节点的超时时间，单位：ms，超时后集群会认为该节点失败
> >
> > cluster-node-timeout 15000
> >
> > #持久化
> >
> > appendonly yes
> >
> > #数据存储目录，目录要提前创建好
> >
> > dir /data/redis/
>
> [root@master ~]# redis-cli --cluster create 192.168.196.001:6379 192.168.196.002:6379 192.168.196.003:6379 192.168.196.004:6379 192.168.196.005:6379 192.168.196.006:6379 --cluster-replicas 1