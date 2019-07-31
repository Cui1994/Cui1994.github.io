---
layout: post
category: "问题排查"
title:  "MySQL localhost 无法访问问题"
tags: [MySQL]
---

* content
{:toc}

![622-800x300.jpg](https://i.loli.net/2019/07/31/5d412043e82c417014.jpg)

线下开发时需要在一台机器上部署 MySQL，并创建不同权限的 user 给其他机器使用，操作通常很简单，在部署机上使用 root 用户登录，创建用户并授予访问权限，方式如下：
```sql
CREATE USER 'test'@'%' IDENTIFIED BY 'abcd1234';
GRANT ALL PRIVILEGES ON * . * TO 'test'@'%';
FLUSH PRIVILEGES;
```
这里 host 用通配符 `%` 表示该用户可以从任意主机访问数据库。操作执行之后从其他主机就可以访问当前数据库了，看起来没问题，直到我在本机使用 localhost 访问数据库报错：
```
ERROR 1045 (28000): Access denied for user 'test'@'localhost' (using password: YES)
```




这个问题是由于 MySQL（5.6）在使用 localhost 和 ip 进行访问时，其通信方式其实是不同的，用 localhost 采用 Unix domain socket 进行通信，而 ip 则是采用 Internet domain socket，因此 localhost 是不能被 `%` 通配符解析，从而命中上面创建的用户和权限的。那解决方式也很简单了，再单独为 localhost 进行授权即可。
```sql
CREATE USER 'test'@'localhost' IDENTIFIED BY 'abcd1234';
GRANT ALL PRIVILEGES ON * . * TO 'test'@'localhost';
FLUSH PRIVILEGES;
```

除此之外，有的公司在内部机房访问时会采用内部 DNS 解析，可以用机器名直接访问 MySQL，这时在本机用 host 访问也会产生上述问题，解决思路都是一样的，这里就不再赘述了。

（ End ）