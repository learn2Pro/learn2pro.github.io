---
title: 详解雪花算法
date: 2020-07-09 09:37:20
tags:
- Algorithm
categories:
- Technology
---

## 背景
雪花算法解决的问题是，如何在分布式环境下产生一个唯一的id，而这些唯一id在业务或者技术上都会有广泛使用，
比如订单号，一次请求的requestId，traceId等；现有的一些算法比如微软的UUID，他能保证唯一，但是雪花算法
相较于他更有优势的一点，它能保证整体递增，这对我们业务类的id很有用；取名的出处是，`世界上找不到两片相同的
雪花`,在我们rpc的框架中，也用到了雪花算法来生成request-id;

## 算法
雪花算法的ID占用64bit，其中分别有不同的含义，第一位bit是符号位，因为我们时间都是正的，所以为0；后续41位为
时间，接着5位datacenterId+5位workerId,最后12bit位sequence表示单机在同一毫秒可以产生的id数量,所以能到单机`400w`的并发
![](/image/snow/snow_flake_id.svg)

## 实现
1. 代码实现非常简单，而且单机非常高效
```java
//用户自己定义的业务起始值，相当于magic num的概念
private final long twepoch = 1288834974657L;
private long nextId() {
    long timestamp = timeGen();

    if (timestamp < lastTimestamp) {
        LOGGER.error("clock is moving backwards.  Rejecting requests until {}.", lastTimestamp);
        throw new RuntimeException(
                "Clock moved backwards.  Refusing to generate id for " + (lastTimestamp - timestamp) +
                        " milliseconds");
    }

    if (lastTimestamp == timestamp) {
        sequence = (sequence + 1) & sequenceMask;
        if (sequence == 0) {
            timestamp = tilNextMillis(lastTimestamp);
        }
    } else {
        sequence = 0;
    }

    lastTimestamp = timestamp;
    return ((timestamp - twepoch) << timestampLeftShift) | (datacenterId << datacenterIdShift) |
            (workerId << workerIdShift) | sequence;
}
```
## 总结
雪花算法被广泛的应用，当然工业级别的id生成算法还有很多，包括美团的[leaf](https://github.com/Meituan-Dianping/Leaf)
百度的[UidGenerator](https://github.com/baidu/uid-generator/blob/master/README.zh_cn.md)
百度使用未来时间，存放到RingBuffer中，再次提升雪花算法的并发度，能到单机`600w`