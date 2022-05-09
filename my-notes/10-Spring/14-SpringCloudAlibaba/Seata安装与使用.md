## 一、单机版

### 1. registry.conf

#### 1.1 修改配置中心和注册中心 

> registry {
>
> ​    type = "nacos"
>
> ​    loadBalance = "RandomLoadBalance"
>
> ​    loadBalanceVirtualNodes = 10
>
> ​    nacos {
> ​        application = "seata-server"
> ​        serverAddr = "127.0.0.1:8848"
> ​        group = "SEATA_GROUP"
> ​        namespace = ""
> ​        cluster = "default"
> ​        username = ""
> ​        password = ""
>
> ​    }
>
> ​    config {
> ​        type = "nacos"
> ​        nacos {
> ​        serverAddr = "127.0.0.1:8848"
> ​        namespace = ""
> ​        group = "SEATA_GROUP"
> ​        username = "nacos"
> ​        password = "nacos"
> ​    }
> }

### 2. file.conf

> store {
>
> ​    mode = "db"
>
> ​    db {
>
> ​        datasource = "druid"
> ​        dbType = "mysql"
> ​        driverClassName = "com.mysql.jdbc.Driver"
> ​        url = "jdbc:mysql://127.0.0.1:3306/seata?rewriteBatchedStatements=true"
> ​        user = "mysql"
> ​        password = "mysql"
> ​        minConn = 5
> ​        maxConn = 100
> ​        globalTable = "global_table"
> ​        branchTable = "branch_table"
> ​        lockTable = "lock_table"
> ​        queryLimit = 100
> ​        maxWait = 5000
>
> }

### 3. 创建数据库

>
> drop table if exists global_table;
> create table global_table (
>  xid varchar(128) not null,
>  transaction_id bigint,
>  status tinyint not null,
>  application_id varchar(32),
>  transaction_service_group varchar(32),
>  transaction_name varchar(128),
>  timeout int,
>  begin_time bigint,
>  application_data varchar(2000),
>  gmt_create datetime,
>  gmt_modified datetime,
>  primary key (xid),
>  key idx_gmt_modified_status (gmt_modified, status),
>  key idx_transaction_id (transaction_id)
> );
>
> -- the table to store BranchSession data
> drop table if exists branch_table;
> create table branch_table (
>  branch_id bigint not null,
>  xid varchar(128) not null,
>  transaction_id bigint ,
>  resource_group_id varchar(32),
>  resource_id varchar(256) ,
>  lock_key varchar(128) ,
>  branch_type varchar(8) ,
>  status tinyint,
>  client_id varchar(64),
>  application_data varchar(2000),
>  gmt_create datetime,
>  gmt_modified datetime,
>  primary key (branch_id),
>  key idx_xid (xid)
> );
>
> -- the table to store lock data
> drop table if exists lock_table;
> create table lock_table (
>  row_key varchar(128) not null,
>  xid varchar(96),
>  transaction_id long ,
>  branch_id long,
>  resource_id varchar(256) ,
>  table_name varchar(32) ,
>  pk varchar(36) ,
>  gmt_create datetime ,
>  gmt_modified datetime,
>  primary key(row_key)
> );
>
> -- the table to store seata xid data
> -- 0.7.0+ add context
> -- you must to init this sql for you business databese. the seata server not need it.
> -- 此脚本必须初始化在你当前的业务数据库中，用于AT 模式XID记录。与server端无关（注：业务数据库）
> -- 注意此处0.3.0+ 增加唯一索引 ux_undo_log
> drop table undo_log;
> CREATE TABLE undo_log (
>  id bigint(20) NOT NULL AUTO_INCREMENT,
>  branch_id bigint(20) NOT NULL,
>  xid varchar(100) NOT NULL,
>  context varchar(128) NOT NULL,
>  rollback_info longblob NOT NULL,
>  log_status int(11) NOT NULL,
>  log_created datetime NOT NULL,
>  log_modified datetime NOT NULL,
>  ext varchar(100) DEFAULT NULL,
>  PRIMARY KEY (id),
>  UNIQUE KEY ux_undo_log (xid,branch_id)
> ) ENGINE=InnoDB AUTO_INCREMENT=1 DEFAULT CHARSET=utf8;