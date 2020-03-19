---
title: Postgres Advisory Locks
date: 2018.01.03
tags:
---

### 问题

Postgres的`select ... for update`可以锁住一行，但是想找一个可以带上purpose的锁住一行，purpose不同可以获得不同的锁。

### 方案

然而并没有，但找到一个[AdvisoryLocks](https://www.postgresql.org/docs/9.4/static/explicit-locking.html)。  
AdvisoryLocks可以根据不同的key，获得不同的锁。但是AdvisoryLocks的key只能是一个`bigint`或者两个`int`，不能传入字符串，就不能很明确的锁住一行。  
Google了一下，[这篇文章](https://stackoverflow.com/questions/29353845/how-do-i-use-string-as-a-key-to-postgresql-advisory-lock)提供了一种解决方案`select pg_advisory_lock( hashtext('fredfred') );`

### 相关

Ruby还有一个相关的gem [with_advisory_lock](https://github.com/ClosureTree/with_advisory_lock)，该gem也支持字符串作为key，查看了一下源码：  
`Zlib.crc32(input.to_s)`  
使用crc将字符串转换成了数字。

### 总结

不管哪种方法，都有一个缺陷，不能做到string映射int的唯一性