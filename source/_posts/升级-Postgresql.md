---
title: 升级 Postgresql
date: 2016-07-19
tags:
---

> 使用`pg_upgrade`升级Posgresql，9.3 -> 9.5

## 升级过程

### 安装 9.5
参考 [How to Install PostgreSQL 9.5 on Ubuntu (12.04 - 15.10)](https://www.howtoforge.com/tutorial/how-to-install-postgresql-95-on-ubuntu-12_04-15_10/)

### 关闭数据库9.3
`sudo service postgresql stop`  
如果用monit设置了自启动，同时关闭Monit，`sudo service monit stop`

### 升级
**切换到postgres用户**  
`sudo su postgres`

**upgrade命令**  
`/usr/lib/postgresql/9.5/bin/pg_upgrade -b /usr/lib/postgresql/9.3/bin/ -B /usr/lib/postgresql/9.5/bin/ -d /var/lib/postgresql/9.3/main/ -o "-D /etc/postgresql/9.3/main" -O "-D /etc/postgresql/9.5/main" -D /var/lib/postgresql/9.5/main/ -p 5431 -P 5432 -U postgres`

执行命令时，出现问题：

> waiting for server to start….postgres cannot access the server configuration file “/var/lib/postgresql/9.5/main/postgresql.conf”: No such file or directory

解决方案：增加`-o "-D /etc/postgresql/9.3/main" -O "-D /etc/postgresql/9.5/main"`参数，设置配置文件目录

### 移除9.3
`sudo apt-get purge postgresql-9.3`

### 启动9.5
`sudo service postgresql start`  
如果设置了monit，`sudo service monit start`

## 参考资料
- [PostgreSQL数据库的升级](http://www.sijitao.net/1597.html)
- [How to Install PostgreSQL 9.5 on Ubuntu (12.04 - 15.10)](https://www.howtoforge.com/tutorial/how-to-install-postgresql-95-on-ubuntu-12_04-15_10/)
- [pg_upgrade](https://www.postgresql.org/docs/9.4/static/pgupgrade.html)


