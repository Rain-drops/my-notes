### DB

#### 1. 对索引的理解，组合索引，索引的最佳实践

。。。。。。

#### 2. 聚簇索引/非聚簇索引

。。。。。。

#### 3. mysql 索引底层实现

。。。。。。

#### 4. 为什么不用 B-tree，为什么不用 hash

。。。。。。

#### 5. 叶子结点存放的是数据还是指向数据的内存地址，使用索引需要注意的几个地方

。。。。。。



#### 8. 什么是红黑树，什么是 b-tree，为什么 hashMap 中用红黑树不用其他树

红黑树更适合于**插入修改密集型**任务

#### 9. 对 mysql 索引的理解，为什么 mysql 索引中用 b+tree，不用 b-tree 或者其他树，为什么不用 hash 索引

。。。。。。



#### 6. 事务的传播机制

> PROPAGATION_REQUIRED：如果当前没有事务，就新建一个事务，有则加入当前事务。
>
> PROPAGATION_SUPPORTS：如果当前没有事务，就以非事务方式运行
>
> PROPAGATION_MANDATORY：如果当前没有事务，则抛出异常
>
> PROPAGATION_REQUIRES_NEW：新建事务，如果当前有事务，则将当前事务挂起
>
> PROPAGATION_NOT_SUPPORTED：以非事务方式运行，如果当前有事务，则将当前事务挂起
>
> PROPAGATION_NEVER：以非事务方式运行，如果当前存在事务，则抛出异常
>
> PROPAGATION_NESTED：如果当前没有事务，就新建一个事务，有则嵌套事务

#### Mysql默认的事务隔离级别，幻读，脏读，mvcc，rr 怎么实现的，rc 如何实现的

**读未提交 Read uncommitted：**

**读已提交 Read committed：**

**可重复读 Repeatable read：**

**串行化     serializable：**

**[MVCC](https://baijiahao.baidu.com/s?id=1669272579360136533&wfr=spider&for=pc)：**多版本并发控制

#### Mysql [间隙锁](https://www.jianshu.com/p/32904ee07e56)

是 innodb 在**可重复读**提交下为了解决幻读问题时引入的锁机制。在可重复读隔离级别下，数据库通过行级锁和间隙锁共同组成（next-key lock）。

**幻读**问题的存在是因为**新增或更新**操作，这时如果进行范围查询的时候（加锁查询），会出现不一致问题。

> **加锁规则：**
>
> 1. 加锁的基本单位是 next-key lock，是前开后闭原则
> 2. 查询过程中访问的对象会增加锁
> 3. 索引上的等值查询，给唯一索引加锁的时候，next-key lock 会升级为行锁
> 4. 索引上的等值查询，向右遍历时，最后一个值不满足查询需求时，next-key lock 退化为间隙锁
> 5. 唯一索引上的范围查询，会访问到不满足条件的第一个值为止

**间隙锁简单案例：**

| 步骤 | 事务 A                                                | 事务 B                                            |
| ---- | ----------------------------------------------------- | ------------------------------------------------- |
| 1    | begin; <br/>select * from t where id = 11 for update; | -                                                 |
| 2    | -                                                     | insert into user value(12,12,12) <br/>**blocked** |
| 3    | commit;                                               | -                                                 |

当有事务 A 和事务 B 时，事务 A 会对数据库表增加 (10, 15] 这个区间锁，这时 `insert id = 12` 的数据的时候就会因为区间锁 (10, 15] 而被锁住无法执行。

**间隙锁死锁问题：**

| 步骤 | 事务 A                                              | 事务 B                                              |
| ---- | --------------------------------------------------- | --------------------------------------------------- |
| 1    | begin;<br/>select * from t where id = 9 for update; | -                                                   |
| 2    | -                                                   | begin;<br/>select * from t where id = 6 for update; |
| 3    | -                                                   | insert into user value(7,7,7)<br/>**blocked**       |
| 4    | insert into user value(7,7,7)<br/>**blocked**       |                                                     |

事务 A 获取到 (5, 10] 之间的间隙锁不允许其他的 DDL 操作，在事务提交，间隙锁释放之前，事务 B 也获取到了间隙锁 (5, 10]，这时两个事务就处于死锁状态。

#### Mysql 死锁



#### 写一段会造成死锁的 sql 语句，死锁发生了如何解决，mysql有没有提供什么机制去解决死锁

> show processlist; // 查看正在执行的 SQL 
>
> kill id 					 // 杀死 SQL 进程

#### 数据库的连接过程（Java代码层面）

> Class.forName()、DriverManager.getConnection()、conn.createStatement()、stmt.executeQuery()、conn.close()
>
> 连接器、缓存层、分析器、优化器、执行器



#### 有两个表，table a，table b，写 sql 查询出仅在 table a 中的数据、仅在 table b 中的数据、既在 table a 又在 table b 中的数据

。。。。。。

#### explain 可以看到哪些信息，什么信息说明什么，explain的结果列讲一下

| 列            | 描述                                                         |
| ------------- | ------------------------------------------------------------ |
| table         | 表名                                                         |
| type          | 连接类型。从最好到最差的连接类型为 **const、eq_reg、ref、range、indexhe 和 ALL** |
| possible_keys | 显示可能应用在这张表中的索引                                 |
| key           | 实际使用的索引                                               |
| key_len       | 使用的索引的长度。在不损失精确性的情况下，长度越短越好       |
| ref           | 显示索引的哪一列被使用了，如果可能的话，是一个常数           |
| rows          | MYSQL 认为必须检查的用来返回请求数据的行数                   |
| Extra         | 关于MYSQL如何解析查询的额外信息                              |

### 