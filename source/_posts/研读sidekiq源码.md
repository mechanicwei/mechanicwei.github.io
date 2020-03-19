---
title: 研读sidekiq源码
date: 2016.03.26
tags:
---

> [sidekiq-4.1.1](https://github.com/mperham/sidekiq)

# Client

## 入队

队列分为2种

- SortedSet（有序集合）：用于存以后执行的任务  
    放`perform_in`和`perform_at`创建的任务，score 是任务执行时间  
    `{"class"=>"TestWorker", "args"=>[], "at"=>1458998746.806636, "retry"=>true, "queue"=>"default", "jid"=>"99993d7d1adf0e023c82dd1e", "created_at"=>1458998566.806747}`  
    相比于下面多个一个`at`

- List（列表）：用于马上执行的任务  
    放`perform_async`创建的任务  
    `{"class"=>"TestWorker", "args"=>["haha"], "retry"=>true, "queue"=>"default", "jid"=>"4dd735256dbd510e6dd76169", "created_at"=>1458998304.433952}`

入队代码  
    ```ruby
        def atomic_push(conn, payloads)
            if payloads.first['at']
                conn.zadd('schedule'.freeze, payloads.map do |hash|
                    at = hash.delete('at'.freeze).to_s
                    [at, Sidekiq.dump_json(hash)]
                end)
            else
                q = payloads.first['queue']
                now = Time.now.to_f
                to_push = payloads.map do |entry|
                    entry['enqueued_at'.freeze] = now
                    Sidekiq.dump_json(entry)
                end
                conn.sadd('queues'.freeze, q)
                conn.lpush("queue:#{q}", to_push)
            end
        end
    ```


### Client Middleware

在执行入队操作时，会先执行中间件  
```ruby
    def invoke(*args)
        chain = retrieve.dup
        traverse_chain = lambda do
            if chain.empty?
                yield
            else
                chain.shift.call(*args, &traverse_chain)
            end
        end
        traverse_chain.call
    end
```


`chain = retrieve.dup`获取中间件的实例  
居然用block的形式，依次执行chain里面的中间件。

# Server

## Sidekiq::Scheduled

轮询`SortedSet`  

```ruby
    def start
        @thread ||= safe_thread("scheduler") do
            initial_wait

            while !@done
                enqueue
                wait
            end
            Sidekiq.logger.info("Scheduler exiting...")
        end
    end
```

找出一个执行时间已到的任务，然后放入`List`去执行  

```ruby
while job = conn.zrangebyscore(sorted_set, '-inf'.freeze, now, :limit => [0, 1]).first do
  if conn.zrem(sorted_set, job)
    Sidekiq::Client.push(Sidekiq.load_json(job))
    Sidekiq::Logging.logger.debug { "enqueued #{sorted_set}: #{job}" }
  end
end
```


## Sidekiq::Processor

执行任务  
从队列找出一个job，执行  

```ruby
def process_one
  @job = fetch
  process(@job) if @job
  @job = nil
end
```


## Server Middleware

### RetryJobs

捕获执行任务出的错误，在job中记录错误和retry信息  

```ruby
def call(worker, msg, queue)
  yield
rescue Sidekiq::Shutdown
  # ignore, will be pushed back onto queue during hard_shutdown
  raise
rescue Exception => e
  # ignore, will be pushed back onto queue during hard_shutdown
  raise Sidekiq::Shutdown if exception_caused_by_shutdown?(e)

  raise e unless msg['retry']
  attempt_retry(worker, msg, queue, e)
end
```

然后放入**retry**(`SortedSet`)队列中  

```ruby
Sidekiq.redis do |conn|
  conn.zadd('retry', retry_at.to_s, payload)
end
```



## Web

### 存状态信息

用[concurrent-ruby](https://github.com/ruby-concurrency/concurrent-ruby)的`Map`(job状态)和`AtomicFixnum`(完成，失败数量)  
创建一个线程，每隔5s的读取`Map`和`AtomicFixnum`中更新的数据，然后写入Redis  

```ruby
while true
  heartbeat(k, data, json)
  sleep 5
end
```

### 读状态信息

[Sinatra](https://github.com/sinatra/sinatra)

## delay

为所有class定义delay方法：

1.  定义了`sidekiq_delay`，同名为`delay`

```ruby
def sidekiq_delay(options={})
  Proxy.new(DelayedClass, self, options)
end
alias_method :delay, :sidekiq_delay
```

2.  把方法扩展到`Module`中，这样所有的类都拥有了这个方法
```ruby
Module.__send__(:include, Sidekiq::Extensions::Klass) unless defined?(::Rails)
```

3.  Proxy定义了一个幽灵方法(method_missing)，获取方法名称，然后入列

## sidekiq-limit_fetch 插件

[sidekiq-limit_fetch-3.1.0](https://github.com/brainopia/sidekiq-limit_fetch)，复写sidekiq的fetch strategy  
`Sidekiq::Manager` prepend了一个module，覆写了`initialize`和`start`方法

```ruby
class Sidekiq::Manager
  module InitLimitFetch
    def initialize(options={})
      options[:fetch] = Sidekiq::LimitFetch
      super
    end

    def start
      Sidekiq::LimitFetch::Queues.start options
      Sidekiq::LimitFetch::Global::Monitor.start!
      super
    end
  end

  prepend InitLimitFetch
end
```

### fetch strategy

limit例子：  
`"#{PREFIX}:probed:#@name"`：(List，@name是队列名称) 存执行的job  
写了一个redis的脚本筛去在limit限制外的队列

## 关闭sidekiq

接收`Ctri + C`发出的**INT**信号，raise和rescue`Interrupt`

*   不再从List队列中拿任务
*   不再轮询`SortedSet`
*   对于已经在执行的任务：最多等一个timeout的时间，如果还没结束，强制退出，重新入队

# Awesome

`x, Thread.current[:sidekiq_worker_set] = Thread.current[:sidekiq_worker_set], nil`

`__send__`不能被复写 `send`可以被复写