---
title: activeMQ学习指南 - Java基础调用
tags:
  - MQ
  - java
categories: 后台
abbrlink: 9471
date: 2020-10-01 08:00:00
---

# activeMQ学习指南 - Java基础调用

## 安装和运行

https://www.yangzhenyu.com.cn/19336/

## Java 基础例子

使用idea建立一个Maven项目

## Maven 代码

pom.xml

```xml
<dependency>
   <groupId>org.apache.activemq</groupId>
   <artifactId>activemq-all</artifactId>
   <version>5.9.0</version>
</dependency>

```

## 常用变量

Consumer.java

```java
package cn.com.yangzhenyu;

public class Constants {

    public static final String MQ_NAME = "mq";

    public static final String MQ_PASSWORD = "mq_password";

    public static final String MQ_BROKET_URL = "tcp://127.0.0.1:61616";

    public static final String QUEUE_NAME = "yzy_queue";
}

```

## 生产者

Producer.java

```java
package cn.com.yangzhenyu;

import org.apache.activemq.ActiveMQConnectionFactory;

import javax.jms.*;

public class Producer {
    public static void main(String[] args) {
        // 连接工厂
        ConnectionFactory factory;
        // 连接实例
        Connection connection = null;
        // 收发的线程实例
        Session session;
        // 消息发送目标地址
        Destination destination;
        // 消息创建者
        MessageProducer messageProducer;
        try {
            factory = new ActiveMQConnectionFactory(Constants.MQ_NAME, Constants.MQ_PASSWORD,
                    Constants.MQ_BROKET_URL);
            // 获取连接实例
            connection = factory.createConnection();
            // 启动连接
            connection.start();
            // 创建接收或发送的线程实例（创建session的时候定义是否要启用事务，且事务类型是Auto_ACKNOWLEDGE也就是消费者成功在Listern中获得消息返回时，会话自动确定用户收到消息）
            session = connection.createSession(Boolean.TRUE, Session.AUTO_ACKNOWLEDGE);
            // 创建队列（返回一个消息目的地）
            destination = session.createQueue(Constants.QUEUE_NAME);
            // 创建消息生产者
            messageProducer = session.createProducer(destination);
            // 创建TextMessage消息实体
            for (int i = 0; i < 5; i++) {
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }

                TextMessage message = session.createTextMessage("我是yzy,这是我的第" + (i + 1) + "个消息！");
                messageProducer.send(message);
                session.commit();

            }
//            session.commit();

        } catch (JMSException e) {
            e.printStackTrace();
        } finally {
            if (connection != null) {
                try {
                    connection.close();
                } catch (JMSException e) {
                    e.printStackTrace();
                }
            }
        }
    }
}
```

## 消费者

Consumer.java

```java
package cn.com.yangzhenyu;

import org.apache.activemq.ActiveMQConnectionFactory;

import javax.jms.*;

public class Consumer {
    public static void main(String[] args) {
        // 连接工厂
        ConnectionFactory connectionFactory;
        // 连接实例
        Connection connection = null;
        // 收发的线程实例
        Session session;
        // 消息发送目标地址
        Destination destination;
        try {
            // 实例化连接工厂
            connectionFactory = new ActiveMQConnectionFactory(Constants.MQ_NAME, Constants.MQ_PASSWORD, Constants.MQ_BROKET_URL);
            // 获取连接实例
            connection = connectionFactory.createConnection();
            // 启动连接
            connection.start();
            // 创建接收或发送的线程实例（消费者就不需要开启事务了）
            session = connection.createSession(Boolean.FALSE, Session.AUTO_ACKNOWLEDGE);
            // 创建队列（返回一个消息目的地）
            destination = session.createQueue(Constants.QUEUE_NAME);
            // 创建消息消费者
            MessageConsumer consumer = session.createConsumer(destination);
            //注册消息监听
            consumer.setMessageListener(new MessageListener() {
                @Override
                public void onMessage(Message message) {
                    try {
                        if (message instanceof TextMessage) {
                            System.out.println(((TextMessage) message).getText());
                        }
                    } catch (JMSException e) {
                        e.printStackTrace();
                    }
                }
            });
        } catch (JMSException e) {
            e.printStackTrace();
        }
    }
}
```

## 运行

- 先运行Consumer.main();
- 再运行Producer.main();

运行结果如下:

```
Consumer 收到消息:

我是yzy,这是我的第1个消息！
我是yzy,这是我的第2个消息！
我是yzy,这是我的第3个消息！
我是yzy,这是我的第4个消息！
我是yzy,这是我的第5个消息！
```