---
layout: post
category: "问题排查"
title:  "MySQL 主键 auto_increment 机制"
tags: [MySQL]
---

![image.png](https://i.loli.net/2020/07/28/L5JYE4pTbq1eOZc.png)


前些日子对接数仓服务，在某天写入元信息时突然报了一波失败，失败信息为主键重复。
```shell
Duplicate entry '67' for key 'PRIMARY' (MySQL error.)
```
由于数仓服务是为我们业务部门独立搭建的服务，定位为"联合运维"，迫不得已共同排查问题。过程中了解到数仓的设计，也让我大跌眼镜。






DB 报错，自然从 DB 入手，拉出 MySQL 相关的 log 发现 insert 的时候确实写入了重复的主键。

我们通常写入 DB 都是不指定主键的，但这个数仓的存储设计可以说是奇葩，在存储分区 Partition 信息时，将信息分成了多个表存储，用 partition_id 进行关联。

这个本来没什么问题，但这个 partition_id 是用另一张 PART_ID 表主键自增值 auto_increment 来实现的，而且并未持久化。 在出问题的当天，DBA 对线上服务器进行了重启，auto_increment 重置。

## MySQL auto_increment
MySQL auto_increment 值 MySQL 维护在内存中（5.7之前版本，MySQL 8 中的自增值会维护在 redo log 中），服务器重启后，MySQL 会在下一次插入数据前首先执行下面的语句将正确的计数赋予自增计数器。
```
SELECT MAX(auto_increment_field) FROM table_name FOR UPDATE;
```
因此，只要将用于生成 partition_id 的 PART_ID 表的每一条记录都进行持久化就可以解决。

问题很简单，而且由于服务处于验证阶段，数据清空后重新导入，问题解决。

不过，有几个问题还是挺有意思的。

### 1、MySQL 自增值是否可以用来做 ID 生成器？
用 MySQL 自增值进行 ID 生成看起来确实不优雅，比如每次获取自增值都需要触发一次写入，ID 位数不固定，有顺序等，完全被市面上成熟的分布式 ID 生成器爆成渣。

不过，话说回来，在 MySQL 里的很多主键 id 关联场景不都是用自增值做了 ID 生成器吗。所以，方案还是要看场景，如果是简单的 id 关联场景，MySQL 足够了。

这次的问提是由于生成唯一 id 时没有实际进行数据写入，而非 MySQL 本身机制出了问题。

### 2、如何不写入数据触发 auto_increment 计数器自增？
auto_increment 在事务中的行为注意下面几点：
1、InnoDB 在事务中进行写入会立即触发 auto_increment 自增，写入的记录自增列也会被赋予最新值。
2、若事务回滚，auto_increment 不会跟着回滚，后续插入只会继续递增。

在事务中，auto_increment 的自增发生在插入时而不是提交时，在事务中写入后回滚就可以出发计数器自增而不写入数据。

当然，这已经被证明是一个失败的设计，不但不应用这种方式生成唯一 id，在开发中，也不应该完全依赖自增值的连贯顺序。

### 3、其他扩展
如果 MySQL 是多主模型，如何避免主键冲突？
为每一个 master 设置不同的 auto_increment 初始值和步进值即可。（实际上之前饿了么的异地多活多主复制就是这么干的）

(End)


