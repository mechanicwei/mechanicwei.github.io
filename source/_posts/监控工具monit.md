---
title: 监控工具monit
date: 2016.05.12
tags:
---

> [monit](https://mmonit.com/monit)

参考[使用 Monit＋Mina 监控服务器](https://ruby-china.org/topics/23176)

## 代码

### Monitrc

```
set daemon 20
set logfile /var/log/monit.log
set idfile /var/lib/monit/id
set statefile /var/lib/monit/state

set alert <%= mail_alert %>
set mailserver <%= mail_server %> port <%= mail_port %>
  username <%= mail_username %> password <%= mail_password %>
  using SSLv3
  with timeout 30 seconds

set mail-format {
  from: <%= mail_username %>
  subject: $SERVICE $EVENT at $DATE
  message: Monit $ACTION $SERVICE
          at $DATE
          on $HOST
          $DESCRIPTION.
          Yours sincerely,
          monit
}

set httpd port 2812 and
  use address localhost
  allow localhost
  allow admin:monit

check system localhost
  if loadavg (1min) > 4 then alert
  if loadavg (5min) > 2 then alert
  if memory usage > 90% then alert
  if cpu usage (user) > 70% for 3 cycles then alert
  if cpu usage (system) > 30% then alert
  if cpu usage (wait) > 20% then alert

include /etc/monit/conf.d/*
```

### Nginx
```
check process nginx with pidfile /run/nginx.pid
  start program = "/etc/init.d/nginx start" as uid <%= user %> and gid <%= user %>
  stop  program = "/etc/init.d/nginx stop" as uid <%= user %> and gid <%= user %>

  if loadavg(5min) greater than 10 for 8 cycles then stop
  if 3 restarts within 5 cycles then timeout
  if failed host 127.0.0.1 port 80 then restart
  # if cpu is greater than 40% for 2 cycles then alert
  # if cpu > 60% for 5 cycles then restart
```




### Memcached

```
check process memcached with pidfile /var/run/memcached.pid
  start program = "/usr/sbin/service memcached start"
  stop program = "/usr/sbin/service memcached stop"
  if failed host 127.0.0.1 port 11211 then restart
  if 2 restarts within 3 cycles then timeout
```


### [](#Postgres "Postgres")Postgres

```
check process postgres with pidfile <%= postgres_pidfile %>
   group database
   start program = "/etc/init.d/postgresql start"
   stop  program = "/etc/init.d/postgresql stop"
   if failed unixsocket /var/run/postgresql/.s.PGSQL.5432 protocol pgsql
      then restart
   if failed host 127.0.0.1 port 5432 protocol pgsql then restart
```

```
check process puma
  with pidfile <%= deploy_to %>/shared/tmp/pids/puma.pid
  start program = "/bin/bash -l -c 'cd <%= deploy_to %>/<%= current_path %> && PATH=<%= home_dir %>/.rbenv/shims:<%= home_dir %>/.rbenv/bin:$PATH <%= puma_cmd %> -q -d -e <%= puma_env %> -C <%= puma_config %>'"
  stop program = "/bin/bash -l -c 'cd <%= deploy_to %>/<%= current_path %> && PATH=<%= home_dir %>/.rbenv/shims:<%= home_dir %>/.rbenv/bin:$PATH <%= pumactl_cmd %> -F <%= puma_config %> stop'"
  group puma

# check process puma_worker_0
#   with pidfile <%= deploy_to %>/shared/tmp/pids/puma_worker_0.pid
#   if totalmem is greater than 700 MB for 2 cycles then exec "/bin/bash -c 'kill -s QUIT `cat <%= deploy_to %>/shared/tmp/pids/puma_worker_0.pid`'"

# check process puma_worker_1
#   with pidfile <%= deploy_to %>/shared/tmp/pids/puma_worker_1.pid
#   if totalmem is greater than 700 MB for 2 cycles then exec "/bin/bash -c 'kill -s QUIT `cat <%= deploy_to %>/shared/tmp/pids/puma_worker_1.pid`'"
```


### Redis

```
check process redis with pidfile /var/run/redis/redis-server.pid
  start program = "/etc/init.d/redis-server start"
  stop program = "/etc/init.d/redis-server stop"
  group redis

  if failed host 127.0.0.1 port 6379 then restart
  if 5 restarts within 5 cycles then timeout
```

### Sidekiq

```
check process sidekiq
  with pidfile <%= deploy_to %>/shared/tmp/pids/sidekiq.pid
  start program = "/bin/bash -c 'cd <%= deploy_to %>/current; PATH=<%= home_dir %>/.rbenv/shims:<%= home_dir %>/.rbenv/bin:$PATH <%= sidekiq %> -d -e <%= rails_env %> -C <%= sidekiq_config %> -P <%= sidekiq_pid %> -L <%= sidekiq_log %>'" as uid <%= user %> and gid <%= user %>
  stop program  = "/bin/bash -c 'cd <%= deploy_to %>/current; PATH=<%= home_dir %>/.rbenv/shims:<%= home_dir %>/.rbenv/bin:$PATH <%= sidekiqctl %> stop <%= sidekiq_pid %> <%= sidekiq_timeout %>'"
  group sidekiq
```

## 遇到的问题

### 1、puma重启失败

> failed to start

[debugging-monit](http://stackoverflow.com/questions/3356476/debugging-monit)，跟着这篇帖子调试，发现是PATH的原因，不能找到`bundle`  
`PATH=<%= home_dir %>/.rbenv/shims:<%= home_dir %>/.rbenv/bin:$PATH`  
rvm对应的[设置](https://noobsnippets.wordpress.com/2015/05/01/set-up-monit-for-sidekiq-on-rails-server/)

### 2、没有监控sidekiq

monit monitor sidekiq

### 3、发送邮件失败

> mail from address must be same as authorization user

得设置`mail-format`