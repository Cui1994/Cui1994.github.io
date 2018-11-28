---
layout: post
category: "Python"
title:  "你真的了解SQLAlchemy吗"
tags: [Python, SQLAlchemy, MySQL]
---

* content
{:toc}

![](https://picsum.photos/800/300/?random)

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

SQAlchemy通过`create_engine`创建`Engine`对象来实现数据库连接.
```python
# DB_CONNECT_STRING = 'mysql+pymysql://root:1234@127.0.0.1/test?charset=utf8'
engine = create_engine(DB_CONNECT_STRING, pool_size=5, max_overflow=2, pool_recycle=60, echo=True)
```
这里设置`echo=True`来显示所执行的SQL日志。

### 延时加载

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
而`flush`则会发送请求到数据库，而此时的变更属于未提交变更，对其他事务是否可见取决于数据库的隔离机制。

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

## 实战Q&A

有了上面的理论基础，下面从实际应用出发，思考一些项目中的常见问题。

### Q1.是否应该启用连接池，如何配置连接池

首先，MySQL建立连接是非常复杂的过程，我们需要启用连接池来保持与MySQL的长连接，以减少建立连接的次数。

那连接池是否越大越好呢，甚至所有连接都使用长连接？答案也是否定的，因为MySQL在执行过程中使用的临时内存是存放在连接对象中的，如果保持大量长连接会导致MySQL的内存快速增长，以致于被系统强行杀掉（OOM）异常重启。因此需要结合业务的TPS来配置连接池大小，来保证与MySQL的活跃连接维持在`pool_size`+`max_overflow`内。

至于连接池的连接回收时间，先来看一个使用MySQL时的常见报错，在查询时与MySQL断开了连接：

```python
OperationalError: (OperationalError) (2013, "Lost connection to MySQL server during query (error(54, 'Connection reset by peer'))") 'SELECT student.id AS student_id, student.name AS student_name \nFROM student \nWHERE student.id = %s \n LIMIT %s' (1, 1)
```

这个错误是由MySQL连接器抛出的，原因一般有两个：
- 所连接的MySQL服务真的发生了重启操作，这种问题无法避免。
- MySQL连接器会主动关闭空闲（Sleep）时间达到`wait_timeout`时间的连接，这个超时时间的默认值为`8小时`。

SQLAlchemy在捕获到MySQL连接器的抛错后会主动清空连接池里的连接，如果需要进行查询需要重新建立连接。因此，你的连接池`pool_recycle`参数至少要小于MySQL的`wait_timeout`时间，以避免连接池中存在被MySQL连接器主动断开的连接。当然也不能太小，使连接池失去意义。一般的建议值为30分钟之内。

### Q2.如何在应用中管理Session

关于这点，[SQLAlchemy官方文档](https://docs.sqlalchemy.org/en/latest/orm/session_basics.html#when-do-i-construct-a-session-when-do-i-commit-it-and-when-do-i-close-it)中给出了合理的方案，但需要注意一些细节。

> 1. As a general rule, keep the lifecycle of the session separate and external from functions and objects that access and/or manipulate database data. This will greatly help with achieving a predictable and consistent transactional scope.
> 2. Make sure you have a clear notion of where transactions begin and end, and keep transactions short, meaning, they end at the series of a sequence of operations, instead of being held open indefinitely.

一般来说，你需要在访问数据库的时候创建`Session`，开启`Transaction`同时执行查询语句，在所有查询语句执行结束之后关闭`Transaction`和`Session`。任何时候，只要一个`Session`被创建并建立数据库连接，则它必须被关闭从而将数据库连接释放到连接池中，否则连接将永远不会释放直到达到数据库连接的超时时间。

对于Web应用来说，由于一个请求的处理时间足够短，可以更简单地将`Session`的生命周期同请求的生命周期保持一致，即如果一个请求需要访问数据库，则创建`Session`，处理完这个请求则关闭`Session`。

由于`Session`是必须被关闭的，为了方便的管理`Session`，保证一个请求内只有一个`Session`是更好的选择，你可以将创建的`Session`作为全局变量使用，或者更方便地，使用`scoped_session`，保证同一个线程内实例化的`Session`都为同一个对象。

```python
session_factory = sessionmaker(bind=engine, expire_on_commit=True)
DBSession = scoped_session(session_factory)
```

这样，在请求结束时，你只需要关闭一个`Session`即可。例如你可以在`Flask`应用中的`after_request`函数中加入一行代码调用`Session Factory`的`remove`方法，就可以保证请求中的`Session`对象被关闭，它会调用`Session`的`close`方法，将数据库连接放回连接池并将未`commit`的`Transaction`回滚。
```python
DBSession.remove()
```
> The scoped_session.remove() method first calls Session.close() on the current Session, which has the effect of releasing any connection/transactional resources owned by the Session first, then discarding the Session itself. “Releasing” here means that connections are returned to their connection pool and any transactional state is rolled back, ultimately using the rollback() method of the underlying DBAPI connection.

需要注意的是，`scoped_session`对在同一个线程的判断方式为`thrending.local()`，当使用多线程处理某个查询语句时，正确的使用方法是在当前线程中实例化一个`Session`并将其传入多线程函数中，否则其他线程中的`Session`将无法被关闭，数据库连接和`Transaction`也将一直不会被释放。

不同于`close`方法，写请求的`commit`则不是一定要执行的，个人不推荐和`remove`方法一样放在请求结束，而是自定义上下文或用装饰器来进行`commit`，保证需要执行`commit`的查询被`commit`。下面的代码是我个人推荐的写法：
```python
class RoutingSession(Session):

    def __init__(self, *args, **kw):
        super(RoutingSession, self).__init__(*args, **kw)

    def __enter__(self):
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        try:
            if exc_val is None:
                self.flush()
                self.commit()
            elif isinstance(exc_val, SQLAlchemyError):
                self.rollback()
        except SQLAlchemyError as e:
            self.rollback()
        finally:
            self.close()

def gen_commit_deco(session_factory):
    def decorated(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            try:
                with session_factory():
                    return func(*args, **kwargs)
            except SQLAlchemyError as e:
                raise
        return wrapper
    return decorated

session_factory = sessionmaker(class_=RoutingSession, bind=engine, expire_on_commit=False)
auto_commit = gen_commit_deco(DBSession)
```

### Q3.Session与Transaction是一回事吗

回到开篇的问题，`Session`与数据库的`Transaction`能画等号吗？

答案是否定的，只有`transactional`状态下的`Session`对象才相当于数据库的`Transaction`，除此之外，当`Session`执行`commit`或`rollback`方法后，与其对应的数据库`Transaction`也就结束了，之后这个`Session`再次执行`query`或`flush`方法，又会有一个新的数据库`Transaction`与其绑定，所以一个`Session`是可以对应不同的数据库`Transaction`的。

对于每一个`Transaction`，数据库都会记录它的回滚日志`undo log`来记录当前事务内数据库的某条记录的值`read-view`，`undo log`会一直保留，直到数据库中没有比它更老版本的`read-view`，因此`Transaction`的生命周期不能太长，否则数据库会浪费大量空间储存回滚日志。对于普通Web请求，可以用`Q2`中的方法来管理`Session`，但如果你在耗时较大的脚本或定时任务中用到`Session`，推荐将`Session`的生命周期控制在尽可能少的函数块内，以数据库避免长事务的产生。

可以用下面的命令判断数据库中存在的长事务：
```sql
select * from information_schema.innodb_trx where TIME_TO_SEC(timediff(now(),trx_started))>60
```

## 写在后面

SQLAlchemy对于每一位Python开发者来说都是宝藏，不但在于其强大且实用，其源码所蕴含的`unit work`思想和各类设计模式更是绝佳的教材。这里给自己挖个坑，下一篇SQLAlchemy博客将会从源码角度进行架构剖析。

(End)

