---
layout:     post
title:      分布式自增ID算法snowflake
subtitle:   分布式自增ID算法snowflake算法java实现
date:       2019-12-12
author:     景山
header-img: img/post-bg-hacker.jpg
catalog: true
tags:
    - snowflake
    - id
    - java
---

&emsp;我们在设计数据库的时候，总会去考虑主键如何设计，是用auto_increment还是用UUID。根据不同的业务场景，每个人都有自己的看法，也有不同的选择。这里介绍Twitter公司的分布式自增ID算法snowflake，中国人叫雪花算法，该算法兼具int/long和UUID的优点。

## 主键策略对比
- auto_increment: 性能高、占用存储空间少、可读性高、存在ID碰撞（分表、分库、分布式），设置自增初始值+步长对ID分段，可解决分布式ID碰撞，适合中等规模的分布式场景。
- uuid: 简单、唯一、性能差（无序）、可读性差、占用存储空间大，适合小规模的分布式场景。
- snowflake：性能高、占用存储空间少、可读性高、唯一，适合大规模的分布式场景。

## snowflake算法背景
Twitter公司把存储系统从MySQL迁移到Cassandra，因为Cassandra没有顺序ID生成机制，而有希望能使用一种简单ID，且ID能够按照时间有序生成，于是snowflake顺势诞生。

## snowflake数据结构
snowflake的数据结构为64位long型。  
0 - 0000000000 0000000000 0000000000 0000000000 0 - 00000 - 00000 - 000000000000
- 第1位位符号位，未使用，ID不可能为负数。
- 第2-42位为毫秒级时间戳，41位的长度可以使用69年。
- 第43-52位为集群ID和服务器ID，10位的长度最多支持部署1024个节点。
- 第53-64位为毫秒内的计数，12位的计数顺序号支持每个节点每毫秒产生4096个ID序号。

## snowflake 实现
```
package com.up.snowflake;

/**
 * @author YuanJingshan
 * @version 1.0
 * @description 雪花算法工具，实现主键ID的生成，拒绝ID碰撞
 * @date 2019/12/11
 */
public class Snowflake {

    /**
     * 服务器id所占的位数
     */
    private final long workerIdBits = 5L;

    /**
     * 数据标识id所占的位数
     */
    private final long dataCenterIdBits = 5L;

    /**
     * 支持的最大服务器id，结果是31 (这个移位算法可以很快的计算出几位二进制数所能表示的最大十进制数)
     */
    private final long maxWorkerId = -1L ^ (-1L << workerIdBits);

    /**
     * 支持的最大数据标识id，结果是31
     */
    private final long maxDataCenterId = -1L ^ (-1L << dataCenterIdBits);

    /**
     * 序列在id中占的位数
     */
    private final long sequenceBits = 12L;

    /**
     * 服务器ID向左移12位
     */
    private final long workerIdShift = sequenceBits;

    /**
     * 数据标识id向左移17位(12+5)
     */
    private final long dataCenterIdShift = sequenceBits + workerIdBits;

    /**
     * 时间截向左移22位(5+5+12)
     */
    private final long timestampLeftShift = sequenceBits + workerIdBits + dataCenterIdBits;

    /**
     * 生成序列的掩码，这里为4095 (0b111111111111=0xfff=4095)
     */
    private final long sequenceMask = -1L ^ (-1L << sequenceBits);

    /**
     * 开始时间截
     */
    private long startTimestamp;

    /**
     * 服务器ID(0~31)
     */
    private long workerId;

    /**
     * 数据中心ID(0~31)
     */
    private long dataCenterId;

    /**
     * 毫秒内序列(0~4095)
     */
    private long sequence = 0L;

    /**
     * 上次生成ID的时间截
     */
    private long lastTimestamp = -1L;

    /**
     * 实例
     */
    private static Snowflake instance;

    /**
     * 获取实例
     *
     * @param startTimestamp 开始时间戳
     * @param workerId       服务器ID
     * @param dataCenterId   数据中心ID
     * @return
     */
    public static synchronized Snowflake getInstance(long startTimestamp, long workerId, long dataCenterId) {
        if (instance == null) {
            instance = new Snowflake(startTimestamp, workerId, dataCenterId);
        }
        return instance;
    }

    /**
     * 构造函数
     *
     * @param workerId     服务器ID (0~31)
     * @param dataCenterId 数据中心ID (0~31)
     */
    private Snowflake(long startTimestamp, long workerId, long dataCenterId) {
        if (workerId > maxWorkerId || workerId < 0) {
            throw new IllegalArgumentException(String.format("服务器的ID只能在0到%d之间！", maxWorkerId));
        }
        if (dataCenterId > maxDataCenterId || dataCenterId < 0) {
            throw new IllegalArgumentException(String.format("数据中心ID只能在0到%d之间！", maxDataCenterId));
        }
        this.workerId = workerId;
        this.dataCenterId = dataCenterId;
        this.startTimestamp = startTimestamp;
    }

    /**
     * 获得下一个ID (该方法是线程安全的)
     *
     * @return SnowflakeId
     */
    public synchronized long nextId() {
        long timestamp = timeGen();
        //如果当前时间小于上一次ID生成的时间戳，说明系统时钟回退过这个时候应当抛出异常
        if (timestamp < lastTimestamp) {
            throw new RuntimeException(String.format("系统时钟被回退%d秒，拒绝生成ID！", lastTimestamp - timestamp));
        }
        //如果是同一时间生成的，则进行毫秒内序列
        if (lastTimestamp == timestamp) {
            sequence = (sequence + 1) & sequenceMask;
            //毫秒内序列溢出
            if (sequence == 0) {
                //阻塞到下一个毫秒,获得新的时间戳
                timestamp = tilNextMillis(lastTimestamp);
            }
        } else {
            //时间戳改变，毫秒内序列重置
            sequence = 0L;
        }
        //上次生成ID的时间截
        lastTimestamp = timestamp;
        //移位并通过或运算拼到一起组成64位的ID
        return ((timestamp - startTimestamp) << timestampLeftShift)
                | (dataCenterId << dataCenterIdShift)
                | (workerId << workerIdShift)
                | sequence;
    }

    /**
     * 阻塞到下一个毫秒，直到获得新的时间戳
     *
     * @param lastTimestamp 上次生成ID的时间截
     * @return 当前时间戳
     */
    protected long tilNextMillis(long lastTimestamp) {
        long timestamp = timeGen();
        while (timestamp <= lastTimestamp) {
            timestamp = timeGen();
        }
        return timestamp;
    }

    /**
     * 返回以毫秒为单位的当前时间
     *
     * @return 当前时间(毫秒)
     */
    protected long timeGen() {
        return System.currentTimeMillis();
    }
}
```
