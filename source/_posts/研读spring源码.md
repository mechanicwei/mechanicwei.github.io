---
title: 研读spring源码
date: 2016.04.03
tags:
---

> [spring-1.6.4](https://github.com/rails/spring)

## 原理

创建一个进程，提前`eager_load`，通过执行`spring`+**command**，把command传给进程，进程收到后，用`fork`创建子进程处理，这样不用每次执行命令都要`load`代码

## 执行入口

- 项目中`bin/spring`的`require "spring/binstub"`
- 在`binstub`文件load(执行)gem中的`bin/spring`
- `Spring::Client.run(ARGV)`

## Client

### #boot_server

用`Process.spawn`（创建一个进程来执行命令）启动server  
```ruby
pid = Process.spawn(
  gem_env,
  "ruby",
  "-e", "gem 'spring', '#{Spring::VERSION}'; require 'spring/server'; Spring::Server.boot"
)
```

### #run

`application, client = UNIXSocket.pair` 用于两个进程通信

- 把client给server  
    `server.send_io client`
- application写入命令信息  
    `send_json application, "args" => args, "env" => ENV.to_hash`
- server通过client读取信息  
    `serve server.accept`  
    `app_client = client.recv_io` #这里的app_client就是上面的client

`send_json server, "args" => args, "default_rails_env" => default_rails_env`  
这里同时把参数传给了server，是为了在server里使用

## Server

### #boot

### #serve

- 接收到Client传过来的client(io)，然后交给`ApplicationManager`处理 –run返回pid，client写入流中  
    `client.puts @applications[rails_env_for(args, default_rails_env)].run(app_client)`

## ApplicationManager

### #run

`@child, child_socket = UNIXSocket.pair`

- 用`Process.spawn`执行`require 'spring/application/boot'`，`boot`中创建`Spring::Application`

## Application

### #initialize

manger: 上面的child_socket

### #run

循环的从manger获取client（包含命令参数）  
`serve manager.recv_io(UNIXSocket)`

### #serve

- 从client读取命令参数
- 用`fork`创建一个新的进程来执行命令

## 修改项目文件

### 属于Rails自动导入的文件

`Application#serve`  

```ruby
if Rails.application.reloaders.any?(&:updated?)
  ActionDispatch::Reloader.cleanup!
  ActionDispatch::Reloader.prepare!
end
```

### 其他文件

关闭server，下次执行命令重启server  
`Spring::Watcher::Polling`-`#start`创建一个线程不断的检查监测文件是否改变：取所有文件最后改变时间的最大值，与之前存下的值进行比较

## Awesome

- ljust `"hello".ljust(20) #=> "hello "`
- 实现两个进程之间的通信：`application, client = UNIXSocket.pair`