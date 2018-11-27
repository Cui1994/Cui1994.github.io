---
layout: post
category: "Python"
title:  "你真的了解SQLAlchemy吗"
tags: [Python, SQLAlchemy, MySQL]
---

* content
{:toc}




SQLAlchemy作为一款强大而又实用的ORM框架，越来越频繁的出现在各类Python项目中。如果你是一名Python开发工程师，肯定可以熟练的写出SQLAlchemy的查询语句来满足自己的业务需求。然而，SQLAlchemy背后的知识你又了解多少呢？它是什么时候与数据库建立连接的呢？又是怎样的连接方式呢？SQLAlchemy中的`Session`与数据库的`Transaction`是一回事吗？如何优雅地管理SQLAlchemy的连接与事务呢？本文将结合[SQLAlchemy文档](https://docs.sqlalchemy.org/en/latest/orm/tutorial.html)解答这些问题。

为了方便展示，我在本地MySQL中创建了一张`student`表，同时打开MySQL的`general-log`来追踪MySQL的行为。
```sql
CREATE TABLE `student` (
    id BIGINT NOT NULL AUTO_INCREMENT COMMENT '自增主键',
    name STRING NOT NULL DEFAULT '' COMMENT '姓名',
    primary key(id)
)ENGINE=InnoDB DEFAULT CHARSET=utf8 COMMENT '学生表';

set global general_log=1;
```

同时在iPython中创建我们的ORM对象。

```python
DeclarativeBase = declarative_base()

class StudentModel(DeclarativeBase):

    __tablename__ = 'student'

    id = Column(BigInteger, primary_key=True)
    name = Column(String, default=u'')
```


## 连接

### 延时加载

SQAlchemy通过`create_engine`创建`Engine`对象来实现数据库连接.
```python
# DB_CONNECT_STRING = 'mysql+pymysql://root:1234@127.0.0.1/test?charset=utf8'
engine = create_engine(DB_CONNECT_STRING, pool_size=5, max_overflow=2, pool_recycle=60, echo=True)
```
这里设置`echo=True`来显示所执行的SQL日志。

执行`create_engine`后`general-log`和`stdout`并没有显示任何信息，因为SQLAlchemy到数据库的连接是延时加载的，只有真正需要建立连接时才会创建。

>The Engine, when first returned by create_engine(), has not actually tried to connect to the database yet; that happens only the first time it is asked to perform a task against the database.

调用`connect`方法建立连接：
```python
connection = engine.connect()
```

此时`general-log`显示如下信息，表示真正建立了到数据库的连接，连接线程ID为`22`，除此之外，SQAlchemy会默认将数据库的`autocommit`设置为`0`，在查询时会隐式开始`Transaction`。
```
2018-11-27T13:52:48.695968Z    22 Connect   root@localhost on test using TCP/IP
2018-11-27T13:52:48.701437Z    22 Query     SET AUTOCOMMIT = 0
```

### 连接池

`Engine`对象维护着一个连接池`pool`，如果没有通过`poolclass=NullPool`来禁用连接池，在连接关闭后会被暂存在`pool`中，下次创建连接时会直接从`pool`中获取连接，如果从连接池中拿到的连接距离创建时间超过`pool_recycle`，`Engine`将会将此连接释放并创建新的连接。

在`pool_recycle`时间内调用`close`关闭连接，同时创建一个新连接：
```python
connection.close()
connection2 = engine.connect()
```

`general-log`并没有新日志显示，查看MySQL的连接发现并没有新连接产生。
```
mysql> show processlist;
+----+------+-----------------+------+---------+------+----------+------------------+
| Id | User | Host            | db   | Command | Time | State    | Info             |
+----+------+-----------------+------+---------+------+----------+------------------+
| 12 | root | localhost       | NULL | Query   |    0 | starting | show processlist |
| 22 | root | localhost:64797 | test | Sleep   |   11 |          | NULL             |
+----+------+-----------------+------+---------+------+----------+------------------+
2 rows in set (0.00 sec)
```

而如果在`pool_recycle`之外执行上面的语句，`general-log`则会展示老连接释放，新连接创建的过程。

```
2018-11-27T13:53:14.045708Z    22 Query     ROLLBACK
2018-11-27T14:05:03.360576Z    22 Quit
2018-11-27T14:05:03.367828Z    23 Connect   root@localhost on test using TCP/IP
2018-11-27T14:05:03.372500Z    23 Query     SET AUTOCOMMIT = 0
```

## Session

调用`sessionmaker`方法绑定`Engine`对象创建`Session Factory`，实例化后产生`Session`对象。
```python
DBSession = sessionmaker(bind=engine, expire_on_commit=False)
session = DBSession()
```

### Session状态

`Session`有两个状态：`begin`和`transactional`。
区别在于`Session`有没有真正建立到数据库的连接，即绑定的`Engine`对象是否通过调用`connect`方法变为`Connection`对象。

- 如果`autocommit`为默认值`False`，`Session`在实例化之后即处于`begin`状态。如果`autocommit`值为`True`，则需要显式调用`session.begin()`方法来开启事务。
- 当调用`query`、`execute`、`flush`方法后，将创建到数据库的连接，`Engine`变为`Connection`对象，`Session`进入`transactional`状态。
- 之后继续调用`commit`或`rollback`方法后，连接关闭放入连接池，`Connection`对象变为`Engine`对象，`Session`也重新回到`begin`状态。

>When the transactional state is completed after a rollback or commit, the Session releases all Transaction and Connection resources, and goes back to the “begin” state, which will again invoke new Connection and Transaction objects as new requests to emit SQL statements are received.

实例化一个ORM对象，依次调用`add`、`flush`和`rollback`方法，`stdout`显示了隐式开启`Transaction`的步骤：
```python
a = StudentModel()
session.add(a)
session.flush()
# 2018-11-27 23:04:02,579 INFO sqlalchemy.engine.base.Engine BEGIN (implicit)
# 2018-11-27 23:04:02,579 INFO sqlalchemy.engine.base.Engine INSERT INTO student (id, name) VALUES (%s, %s)
# 2018-11-27 23:04:02,579 INFO sqlalchemy.engine.base.Engine (2, u'')

session.rollback()
# 2018-11-27 23:04:14,036 INFO sqlalchemy.engine.base.Engine ROLLBACK
```

`general-log`也显示在`flush`之后建立了连接并执行了查询语句，之后进行了回滚。
```
2018-11-27T15:04:02.573576Z    32 Connect   root@localhost on test using TCP/IP
2018-11-27T15:04:02.573872Z    32 Query     SET AUTOCOMMIT = 0
2018-11-27T15:04:02.580709Z    32 Query     INSERT INTO student (id, name) VALUES (2, '')
2018-11-27T15:04:14.036762Z    32 Query     ROLLBACK
2018-11-27T15:04:14.042584Z    32 Query     ROLLBACK
```

保留上面的`session`在1分钟后重复执行上面的步骤，`general-log`显示了老连接释放和新连接的建立过程。
```
2018-11-27T15:10:43.755010Z    32 Quit
2018-11-27T15:10:43.762052Z    33 Connect   root@localhost on test using TCP/IP
2018-11-27T15:10:43.763616Z    33 Query     SET AUTOCOMMIT = 0
2018-11-27T15:10:43.764588Z    33 Query     INSERT INTO student (id, name) VALUES (2, '')
2018-11-27T15:11:11.283859Z    33 Query     ROLLBACK
2018-11-27T15:11:11.289628Z    33 Query     ROLLBACK
```

### flush与add的区别

通过上面的展示也可以发现，`add`实际上不会真正发送请求到数据库，这里所做的任何修改其他事务都是不可见的。
而`flush`则会发送请求到数据库，而此时的变更对其他事务是否可见取决于数据库的隔离机制。

### expire_on_commit

`expire_on_commit`可以用来更改SQLAlchemy的对象刷新机制，默认值为`True`即在`session`调用`commit`之后会主动将同一个`session`在`commit`之前查询得到的ORM对象的`_sa_instance_state.expire`属性设置为`Flase`，再次读取该对象属性时将重载这个对象，方法是重新调用之前的查询语句。

将`expire_on_commit`设置为`True`后重新执行代码，发现在获取b属性时又执行了一次查询。
```python
b = session.query(StudentModel).filter(StudentModel.id==2).first()
# 2018-11-27 23:52:03,584 INFO sqlalchemy.engine.base.Engine SELECT student.id AS student_id, student.name AS student_name
# FROM student
# WHERE student.id = %s
#  LIMIT %s
# 2018-11-27 23:52:03,585 INFO sqlalchemy.engine.base.Engine (2, 1)

b.name = u'test'
session.add(b)
session.flush()
# 2018-11-27 23:52:31,789 INFO sqlalchemy.engine.base.Engine UPDATE student SET name=%s WHERE student.id = %s
# 2018-11-27 23:52:31,789 INFO sqlalchemy.engine.base.Engine (u'test', 2)

session.commit()
# 2018-11-27 23:53:06,424 INFO sqlalchemy.engine.base.Engine COMMIT

b._sa_instance_state.__dict__
# {'_instance_dict': <weakref at 0x1028c6f70; to 'WeakInstanceDict' at 0x1036b6050>,
#  '_strong_obj': None,
#  'callables': {'id': <sqlalchemy.orm.state.InstanceState at 0x103743990>,
#   'name': <sqlalchemy.orm.state.InstanceState at 0x103743990>},
#  'class_': __main__.StudentModel,
#  'committed_state': {},
#  'expired': True,
#  'key': (__main__.StudentModel, (2,)),
#  'manager': <ClassManager of <class '__main__.StudentModel'> at 1036b64f0>,
#  'modified': False,
#  'obj': <weakref at 0x1037016d8; to 'StudentModel' at 0x103743850>,
#  'runid': 2,
#  'session_id': 1}

b.name
# 2018-11-27 23:54:34,431 INFO sqlalchemy.engine.base.Engine BEGIN (implicit)
# 2018-11-27 23:54:34,431 INFO sqlalchemy.engine.base.Engine SELECT student.id AS student_id, student.name AS student_name
# FROM student
# WHERE student.id = %s
# 2018-11-27 23:54:34,431 INFO sqlalchemy.engine.base.Engine (2,)
```
虽然这样可以保持数据同步，但实际应用中一般将这个值设置为`Flase`，避免内存中的数据属性被更改。
