```
CREATE DATABASE IF NOT EXISTS test_01 DEFAULT CHARACTER SET utf8mb4 DEFAULT COLLATE utf8mb4_general_ci;

USE test_01;

CREATE TABLE IF NOT EXISTS `goods_info`  (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `goods_id` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `goods_code` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `goods_name` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `goods_info` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  PRIMARY KEY (`id`) USING BTREE
) ENGINE = InnoDB AUTO_INCREMENT = 1 CHARACTER SET = utf8mb4 COLLATE = utf8mb4_general_ci ROW_FORMAT = Compact;

-- 
INSERT INTO `goods_info` VALUES (1, '000001', '000001', 'a', '1');
INSERT INTO `goods_info` VALUES (2, '000002', '000002', 'ab', '12');
INSERT INTO `goods_info` VALUES (3, '000003', '000003', 'abc', '123');
INSERT INTO `goods_info` VALUES (4, '000004', '000004', 'abcd', '1234');
INSERT INTO `goods_info` VALUES (5, '000005', '000005', 'abcde', '12345');
INSERT INTO `goods_info` VALUES (6, '000006', '000006', 'abcdef', '123456');
```

```
## 创建一个同名表结构
[root@node2] mysql> CREATE TABLE `xxx`();
## 使用备份的frm文件替代生成的frm文件, 重启实例
[root@node1] scp -r goods_info.frm root@node2:$PWD

[root@node2] chown -R mysql:mysql goods_info.frm

[root@node2] systemctl restart mariadb

## 删除当前的ibd文件
[root@node2] mysql> alter table goods_info discard tablespace;
## 使用备份的ibd文件, 恢复表数据
[root@node1] scp -r goods_info.ibd root@node2:$PWD

[root@node2] chown -R mysql:mysql goods_info.ibd
## 将ibd文件重新加载进来
[root@node2] mysql> alter table goods_info import tablespace;
```



## 主-从

## Master
[root@node2] vim /etc/my.cnf
```
server-id=130
log-bin=mysql-bin
### 忽略表
replicate-wild-ignore-table=mysql.*
replicate-wild-ignore-table=sys.*
```

### 创建用户、授权
[root@node1] mysql> CREATE USER 'rep'@'node2' IDENTIFIED BY '!qA@wS3Ed$rF';
[root@node1] mysql> grant replication slave on *.* to 'rep'@'node2' identified by '!qA@wS3Ed$rF';
[root@node1] mysql> FLUSH PRIVILEGES;

### 刷新表、并加锁，阻止写操作
[root@node1] mysql> flush tables with read lock;

### 获取二进制日志信息
[root@node1] mysql> show master status;
```
+-------------------+----------+--------------+------------------+
| File              | Position | Binlog_Do_DB | Binlog_Ignore_DB |
+-------------------+----------+--------------+------------------+
| mysql-bin.000006 |      245 |              |                  |
+-------------------+----------+--------------+------------------+
```

### 数据库解锁
[root@node1] mysql> unlock tables;


## Selve

[root@node2] vim /etc/my.cnf
```
server-id=131
log-bin=mysql-bin
### 忽略表
replicate-wild-ignore-table=mysql.*
replicate-wild-ignore-table=sys.*
```

### 配置同步参数
[root@node2] mysql> CHANGE MASTER TO 
				-> MASTER_HOST='<master_host>', 
				-> MASTER_PORT='<master_port>', 
				-> MASTER_USER='<replication_user_name>', 
				-> MASTER_PASSWORD='<replication_password>', 
				-> MASTER_LOG_FILE='<recorded_log_file_name>',
				-> MASTER_LOG_POS='<recorded_log_position>';

### 启动主从同步进程
[root@node2] mysql> start slave;

### 检查状态
[root@node2] mysql> show slave status \G;
```
*************************** 1. row ***************************
   ......
slave_io_running: yes
slave_sql_running: yes
   ......
```
