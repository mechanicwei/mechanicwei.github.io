---
title: 问题记录之kafka生产乱序
date: 2024.08.21
tags:
---

## 背景
2个消息生产乱序了，在业务上必须按顺序消费，才能保证流程正确往下走
使用的kafka client为：https://github.com/IBM/sarama v1.45.1


## 如何确定生产乱序
找到2个消息的offset，进行比对，如果先publish的消息offset还要大，那么就生产乱序了
如何找到消息的offset?

方式一
使用的是AsyncProducer，需要开启Successes()
`kafkaConfig.Producer.Return.Successes = true`
```go
for {
	select {
	case msg := <-producer.Successes():
		fmt.Println(msg) // msg.Offset
	case e := <-producer.Errors():
	case <-p.exit:
		return
	}
}
```
方式二
1、获取partition
根据partitionKey和partition总数计算出partition
```go
	p := sarama.NewHashPartitioner(`topic`)
	partition, err := p.Partition(&sarama.ProducerMessage{
		Key: sarama.ByteEncoder(partitionKey),
	}, 12)
```
2、 根据大致的时间点，在阿里云控制台找到该partition 前几秒的offset

3、 根据offset， 去消费消息，找到想要的消息
/opt/kafka/bin/kafka-console-consumer.sh --topic topic --partition 8 --offset 1343996026 --max-messages 500 --bootstrap-server "$server" --property print.offset=true  --property print.timestamp=true

该topic有很多数据，业务上没有打印success msg，这里采用的是方式二

## 乱序原因
MaxOpenRequests默认为5：收到server响应之前，client最多可以发5个请求
> Throughput can improve but message ordering is not guaranteed if Producer.Idempotent is disabled

把MaxOpenRequests设置为1后，发现还是有乱序的情况，https://github.com/IBM/sarama/issues/2619 从该issue中可以看出，retry会导致乱序
查看sarama的日志，有一些写失败的报错，写失败就会导致重试，重试就有可能乱序
> [Sarama] client/coordinator request to broker x.x.x.x:9092 failed: write tcp 172.20.0.241:43284->x.x.x.x:9092: write: broken pipe

去服务器抓包，发现kafka server会主动断开连接，猜测是因为这个原因导致了写失败，找阿里云反馈，得到的回复是：Kafka服务端发现连接超过10分钟空闲，为了避免资源浪费，会主动断开连接；java客户端默认9分钟自动回收连接，推荐我们不要用sarama这个库。
![](/images/kafka抓包.jpg)


## 解决方案
方案一：换client sdk，用官方的，https://github.com/confluentinc/confluent-kafka-go
这个库用的cgo，本质是对[librdkafka](https://github.com/confluentinc/librdkafka)的封装，librdkafka支持配置`connections.max.idle.ms`，配置成`1000*60*9=5400000`，跟java的一样

考虑到切换成本，实际用了一个取巧的方案针对性解决该问题：因为AsyncProducer是异步的，导致了乱选，把第一次publish改成SyncProducer，第二次还是AsyncProducer，这样就能保证它们的顺序性了

## 其他知识点
- client如果不向connection读写数据，是无法感知connection已经关闭的
