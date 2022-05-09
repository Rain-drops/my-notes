### 一、单机版

> [root@master ~]# cd  /nacos/bin
>
> [root@master ~]# vim /nacos/conf/application.properties
>
> > spring.datasource.platform=mysql
> > db.num=1
> > db.url.0=jdbc:mysql://localhost:3306/nacos?characterEncoding=utf8&connectTimeout=1000&socketTimeout=3000&autoReconnect=true
> > db.user=root
> > db.password=root
>
> 创建数据库 nacos，并设置字符集 utf8，并执行/nacos/conf/nacos-mysql.sql中的语句，创建表，并插入数据
>
> [root@master ~]# sh startup.sh -m standalone
>
> 访问：localhost:8848/nacos，输入默认账号密码：nacos，nacos



### 二、集群版

> [root@master ~]#  vim /nacos/conf/cluster.conf
>
> > 192.168.16.101:8847
> > 192.168.16.102:8847
> > 192.168.16.103:8847
>
> [root@master ~]# sh startup.sh

