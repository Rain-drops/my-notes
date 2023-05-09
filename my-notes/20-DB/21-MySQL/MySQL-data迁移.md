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
