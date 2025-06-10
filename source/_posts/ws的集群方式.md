---
title: websocket的集群方式
date: 2025.02.21
tags:
---

## 方式一
将连接信息存入db(如：redis、mysql)中，需要推送的时候，从**db**中查出连接信息，把消息推送给对应有连接的节点
- redis：设置key的ttl
- mysql: 记录最后活跃时间

如果最后活跃时间超过一定时间，就认为连接失效
https://github.com/Terry-Mao/goim 采用的就是该方式

## 方式二
pub-sub（如：kafka、redis）
将要推送的消息发布出去，各节点收到后判断自身有没有连接

## 方式三
每建立、断开一个连接，需要将连接信息广播到其他所有节点，每个节点在内存中都存了所有的连接信息
每个节点需要监听集群里其他节点，如果某个节点down了，需要从内存中删除该节点的连接信息
需要推送的时候，从**内存**中查出连接信息，把消息推送给对应有连接的节点

该方式很适合Erlang，自带分布式，以下是对[simple_global](https://github.com/suexcxine/simple_global)的研究
Erlang的ets只是local的，simple_global用gen_server实现全局的ets（每个node的ets都包含所有的数据）

新node启动的时候，向集群里的其他节点发送`sync_req`
```erlang
init([]) ->
    process_flag(trap_exit, true),
    ok = net_kernel:monitor_nodes(true),
    broadcast([{?SERVER, Node} || Node <- nodes()], {sync_req, self()}),
    ets:new(?ETS, [ordered_set, named_table, public, {keypos, 1}, {read_concurrency, true}]),
    {ok, #{peers => #{}}}.
```

其他节点收到`sync_req`后，把自己的ets数据同步给新node
```erlang
handle_info({sync_req, Peer}, #{peers := Peers} = State) ->
    gen_server:cast(Peer, {sync_resp, self(), local_registered_info()}),
    ...
```

新node收到`sync_resp`后，把数据写入本地的ets
```erlang
handle_cast({sync_resp, Peer, Regs}, #{peers := Peers} = State) ->
    lists:foreach(fun
        ({Name, Pid}) ->
            on_remote_reg_notify(Name, Pid);
        ({Name, Pid, Meta}) ->
            on_remote_reg_notify(Name, Pid, Meta)
    end, Regs),
    ...
```

收到新数据后，写入当前node的ets，并向集群里的其他节点发送`register_notify`
```erlang
handle_call({register, Name, Pid}, _From, #{peers := Peers} = State) ->
    ...
    MRef = erlang:monitor(process, Pid),
    ets:insert(?ETS, {Name, Pid, local, MRef, #{}}),
    ets:insert(?ETS, {{ref, MRef}, Name}),
    broadcast(maps:keys(Peers), {register_notify, self(), Name, Pid}),
    ...
```
同理，删除数据时，删除本地的ets数据，并向集群里的其他节点发送`unregister_notify`

**问：当一个node突然断电、kill、网络断开了，其他node如何处理？**
新node收到`sync_resp`后，除了写入ets，还会监听该node
```erlang
MRef = erlang:monitor(process, Peer),
```
即使当一个node突然断电、kill、网络断开了，其他node也会收到`DOWN`事件，删除ets中属于该node的数据
- Erlang VM 的分布式心跳机制会检测连接状态
- 所以通常在1-5 秒内会检测到连接中断，发送 DOWN 消息, Reasion为noconnection

```erlang
handle_info({'DOWN', MRef, process, Peer, _Reason}, #{peers := Peers} = State) ->
    case maps:take(Pid, Peers) of
        {_, LeftPeers} ->
            Node = node(Pid),
            Ms = [{{'_', '_', '$1', '_', '_'}, [{'=:=', '$1', Node}], [true]}],
            ets:select_delete(?ETS, Ms),
            % we don't need to clear ref records
            % since only local registration have those ref records
            {noreply, State#{peers => LeftPeers}};
        _ ->
            {noreply, State}
    end;
```
