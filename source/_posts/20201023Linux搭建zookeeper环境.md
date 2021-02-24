---
abbrlink: 1
title: Linux搭建zookeeper环境
tags:
  - java
  - zookeeper
  - 分布式
categories: 后台
date: 2020-10-23 08:00:00
---


## Linux搭建zookeeper环境

### 简介

ZooKeeper是一个[分布式](https://baike.baidu.com/item/分布式/19276232)的，开放源码的[分布式应用程序](https://baike.baidu.com/item/分布式应用程序/9854429)协调服务，是[Google](https://baike.baidu.com/item/Google)的Chubby一个[开源](https://baike.baidu.com/item/开源/246339)的实现，是Hadoop和[Hbase](https://baike.baidu.com/item/Hbase/7670213)的重要组件。它是一个为分布式应用提供一致性服务的软件，提供的功能包括：配置维护、域名服务、分布式同步、组服务等。

ZooKeeper的目标就是封装好复杂易出错的关键服务，将简单易用的接口和性能高效、功能稳定的系统提供给用户。

ZooKeeper包含一个简单的原语集，提供Java和C的接口。

ZooKeeper代码版本中，提供了分布式独享锁、选举、队列的接口，代码在$zookeeper_home\src\recipes。其中分布锁和队列有[Java](https://baike.baidu.com/item/Java/85979)和C两个版本，选举只有Java版本。



### 安装

使用ssh连接上linux服务器

在指定目录下载 zookeeper [官方网址](https://zookeeper.apache.org/)

我在linux的/opt目录下载安装的

```sh
cd /opt
mkdir zookeeper
cd zookeeper
wget https://www.apache.org/dyn/closer.lua/zookeeper/zookeeper-3.6.2/apache-zookeeper-3.6.2-bin.tar.gz
# 解压
tar -xzvf apache-zookeeper-3.6.2-bin.tar.gz
```

> 运行zookeeper 需要Java8的环境 [安装指南](https://www.yangzhenyu.com.cn/605/#linux%E7%89%88%E6%9C%AC)



### 运行

进入解压目录的bin目录

![image-20210224165656322](http://img.yzy.ink/image-20210224165656322.png)

```sh
#使用一下命令查看帮助
./zkServer.sh
#Using config: /opt/apache-zookeeper-3.6.2/bin/../conf/zoo.cfg
#grep: /opt/apache-zookeeper-3.6.2/bin/../conf/zoo.cfg: No such file or directory
#grep: /opt/apache-zookeeper-3.6.2/bin/../conf/zoo.cfg: No such file or directory
#mkdir: cannot create directory ‘’: No such file or directory
#Usage: ./zkServer.sh [--config <conf-dir>] {start|start-#foreground|stop|version|restart|status|print-cmd}

#启动
./zkServer.sh start

#启动调试
./zkServer.sh start-foreground

#重启
./zkServer.shrestart

#状态
./zkServer status
```

默认启动会报错

![image-20210224170124632](http://img.yzy.ink/image-20210224170124632.png)

> 因为没有没有的配置文件../conf/zoo.cfg: No such file or directory

```sh
#进入配置文件目录 
cd conf
#拷贝默认配置(自行修改参数)
cp zoo_sample.cfg zoo.cfg
#重新运行
./zkServer.sh start
#启动完毕 默认端口2181
```

### Java 简单连通测试

maven配置 (最好和服务器版本一致)

```xml
<dependency>
    <groupId>org.apache.zookeeper</groupId>
    <artifactId>zookeeper</artifactId>
    <version>3.6.2</version>
</dependency>
```

Java代码 简单连接

```java
public class App {
    public static void main(String[] args) throws InterruptedException, IOException {
        final CountDownLatch countDownLatch = new CountDownLatch(1);
        //连接成功后，会回调watcher监听，此连接操作是异步的，执行完new语句后，直接调用后续代码
        //  可指定多台服务地址 127.0.0.1:2181,127.0.0.1:2182,127.0.0.1:2183
        ZooKeeper zooKeeper = new ZooKeeper("127.0.0.1:2181", 50000, event -> {
            if (Watcher.Event.KeeperState.SyncConnected == event.getState()) {
                //如果收到了服务端的响应事件,连接成功
                countDownLatch.countDown();
            }
        });
        countDownLatch.await();
        System.out.println(zooKeeper.getState());
    }
}
```

![image-20210224171515672](http://img.yzy.ink/image-20210224171515672.png)



> 输出结果: CONNECTED 说明连接成功