---
title: 基于Zookeeper的分布式锁
tags:
  - java
  - zookeeper
categories: 后台
abbrlink: 10442
date: 2021-01-11 20:00:00
---

## 基于Zookeeper的分布式锁

基于 SpringBoot 项目 实现锁的操作

### zookeeper 环境搭建

1. [linux安装教程](https://www.yangzhenyu.com.cn/1/)
2. Windwos安装教程
   1. [下载](https://zookeeper.apache.org/releases.html)
   2. 解压
   3. 进入conf目录 zoo_sample.cfg 复制为 zoo.cfg
   4. 修改配置
   5. 进入bin目录
   6. 双击 zkServer.cmd

### 引入Jar包依赖

在SpringBoot 2.4.3上加入Jar包出现了一个小插曲

```xml
 <dependency>
            <groupId>org.apache.zookeeper</groupId>
            <artifactId>zookeeper</artifactId>
            <version>3.6.2</version>
 </dependency>
```

在SpringBoot项目 只加入以上jar包 运行项目

![image-20210304143226020](http://img.yzy.ink/image-20210304143226020.png)

> 出现了 LoggerFactory is not a Logback LoggerContext but Logback is on the classpath 的错误
>
> slf4j jar包冲突

最后找到了一个idea的jar包冲突管理神器插件

![image-20210304143823163](http://img.yzy.ink/image-20210304143823163.png)

使用方法

 1. 在idea插件里面安装

 2. 打开pom.xml的文件 左下角选择Dependency Analyzer

    ![image-20210304144449647](http://img.yzy.ink/image-20210304144449647.png)

 3. 右键 exclusion 去除 最终的引入依赖

    ```xml
    <dependency>
        <groupId>org.apache.zookeeper</groupId>
        <artifactId>zookeeper</artifactId>
        <version>3.6.2</version>
        <exclusions>
            <exclusion>
                <artifactId>slf4j-log4j12</artifactId>
                <groupId>org.slf4j</groupId>
            </exclusion>
            <exclusion>
                <artifactId>log4j</artifactId>
                <groupId>log4j</groupId>
            </exclusion>
        </exclusions>
    </dependency>
    ```
### Config类

```java
@Configuration
public class ZooKeeperConfig {

    @Value("${zookeeper.address}") // 127.0.0.1:2181
    private String connectString;
    @Value("${zookeeper.timeout}") // 60000
    private int timeout;

    @Bean(name = "zooKeeper")
    public ZooKeeper zooKeeper() {
        ZooKeeper zooKeeper = null;
        try {
            CountDownLatch countDownLatch = new CountDownLatch(1);
            //连接成功后，会回调watcher监听，此连接操作是异步的，执行完new语句后，直接调用后续代码
            //  可指定多台服务地址 127.0.0.1:2181,127.0.0.1:2182,127.0.0.1:2183
            zooKeeper = new ZooKeeper(connectString, timeout, event -> {
                System.out.println("MyWatcher.process(接收到的事件：)" + event);
                if (Watcher.Event.KeeperState.SyncConnected == event.getState()) {
                    //如果收到了服务端的响应事件,连接成功
                    countDownLatch.countDown();
                    System.out.println("OK");
                }
            });
            countDownLatch.await();
        } catch (Exception e) {
            e.printStackTrace();
        }
        return zooKeeper;
    }
}
```

### Lock接口

```java
public interface Lock {

    void lock();

    void unLock();
}
```

### AbsLock类

```java
public abstract class AbsLock implements Lock {

    public static String lockName = "/myLock";

	// 为了让他重试 获取锁
    @Override
    public void lock() {
        if (!tryLock()) {
            waitLock();
            lock();
        }
    }

    public abstract void waitLock(); // 阻塞

    public abstract boolean tryLock(); // 真的加锁逻辑

    public abstract void unLock(); // 解锁
}
```

### SimpleZkLock类

```java
public class SimpleZkLock extends AbsLock {

    private ZooKeeper zooKeeper;

    private CountDownLatch waitLatch = null;

    public SimpleZkLock(ZooKeeper zooKeeper) {
        this.zooKeeper = zooKeeper;
    }

    @Override
    public void waitLock() {
        try {
            waitLatch = new CountDownLatch(1);
            Stat stat = zooKeeper.exists(lockName, event -> {
                if (event.getType() == Watcher.Event.EventType.NodeDeleted && event.getPath().equals(lockName)) {
                    System.out.println("删除了:"+lockName);
                    waitLatch.countDown();
                }
            });
            if (stat != null) {
                waitLatch.await();
            }
        } catch (KeeperException | InterruptedException e) {
            e.printStackTrace();
        }
    }

    @Override
    public boolean tryLock() {
        try {
            zooKeeper.create(lockName, lockName.getBytes(), ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.EPHEMERAL);
            return true;
        } catch (Exception e) {
            return false;
        }
    }

    @Override
    public void unLock() {
        try {
            zooKeeper.delete(lockName, -1);
        } catch (InterruptedException | KeeperException e) {
            e.printStackTrace();
        }
    }
}
```

### 测试方法

```java
@RequestMapping("get2")
public Object get2() throws InterruptedException {
    sum = 0;
    long start = System.currentTimeMillis();
    CountDownLatch countDownLatch = new CountDownLatch(300);

    for (int i = 0; i < 300; i++) {
        new Thread(() -> {
            SimpleZkLock zookeeperLock = new SimpleZkLock(zooKeeper);
            zookeeperLock.lock();
            for (int j = 0; j < 100; j++) {
                sum++;
            }
            countDownLatch.countDown();
            zookeeperLock.unLock();
        }).start();
    }

    countDownLatch.await();
    System.out.println(sum);
    return "结果:" + sum + ",时间:" + (System.currentTimeMillis() - start);
}
```

![image-20210304150901669](http://img.yzy.ink/image-20210304150901669.png)

> 简单的实现了锁的逻辑 原理是通过创建一个临时节点,因为节点的唯一性,只有一个线程可以新建成功,保证了唯一性,其他线程通过监听节点状态来阻塞,删除后再去争夺这把锁
>
> 缺点: 羊群效应 所有其他节点都监听一个节点 切无先后顺序,性能在并发量非常大的时候会造成流量堆积

### 优化方案

```java
public class ZookeeperLock implements Lock {

    public static String root = "/lockRoot";

    protected ZooKeeper zooKeeper;

    private CountDownLatch countDownLatch = null;

    private String nodeKey = "";

    public ZookeeperLock(ZooKeeper zooKeeper) {
        this.zooKeeper = zooKeeper;
    }

    @Override
    public void lock() {
        try {
            Stat stat = zooKeeper.exists(root, false);
            if (stat == null){
                // 如果没有节点 创建根节点
                // 根节点如果是临时节点 无法创建子节点的
                zooKeeper.create(root, null, ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT);
            }

            // 创建子节点
            String lockTemp = root + "/temp";
            String myLockName = zooKeeper.create(lockTemp, null, ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.EPHEMERAL_SEQUENTIAL);
            nodeKey = myLockName;
            myLockName = myLockName.substring(root.length() + 1);

            List<String> list = zooKeeper.getChildren(root, false);
            Collections.sort(list);

            int i = list.indexOf(myLockName);
            if (i != 0 && list.size() > 1) {
                countDownLatch = new CountDownLatch(1);
                Stat stat1 = zooKeeper.exists(root + "/" + list.get(i - 1), event -> {
                    if (event.getType().equals(Watcher.Event.EventType.NodeDeleted) && event.getPath().equals(root + "/" + list.get(i - 1))) {
                        System.out.println("删除了:" + root + "/" + list.get(i - 1));
                        countDownLatch.countDown();
                    }
                });
                if (stat1 != null) {
                    countDownLatch.await();
                }
            }

        } catch (KeeperException | InterruptedException e) {
            e.printStackTrace();
        }
    }

    @Override
    public void unLock() {
        try {
            zooKeeper.delete(nodeKey, -1);
        } catch (InterruptedException | KeeperException e) {
            e.printStackTrace();
        }
    }

}
```

测试代码

```java
@RequestMapping("get")
public Object get() throws InterruptedException {
    sum = 0;
    long start = System.currentTimeMillis();
    CountDownLatch countDownLatch = new CountDownLatch(300);

    for (int i = 0; i < 300; i++) {
        new Thread(() -> {
            ZookeeperLock zookeeperLock = new ZookeeperLock(zooKeeper);
            zookeeperLock.lock();
            for (int j = 0; j < 100; j++) {
                sum++;
            }
            countDownLatch.countDown();
            zookeeperLock.unLock();
        }).start();
    }

    countDownLatch.await();
    System.out.println(sum);
    return "结果:" + sum + ",时间:" + (System.currentTimeMillis() - start);
}
```

![image-20210304151947676](http://img.yzy.ink/image-20210304151947676.png)

> 在一个目录下面每次都创建一个有序节点,去监听它前一个节点,每次获取Lock去判断是否是最小的节点,是的话就放过.其他线程阻塞监听另一个节点

项目源码: [https://github.com/yttrium2016/zookeeper-boot](https://github.com/yttrium2016/zookeeper-boot)