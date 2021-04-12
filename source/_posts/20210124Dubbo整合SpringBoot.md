---
title: Dubbo整合SpringBoot
tags:
  - java
  - 分布式
  - dubbo
categories: 后台
abbrlink: 57357
date: 2021-01-24 20:00:00
---

## Dubbo整合SpringBoot

### 简介

[Dubbo](http://dubbo.io/) （开源分布式服务框架）

Dubbo(读音[ˈdʌbəʊ])是阿里巴巴公司开源的一个高性能优秀的[服务框架](https://baike.baidu.com/item/服务框架)，使得应用可通过高性能的 RPC 实现服务的输出和输入功能，可以和 [1] [Spring](https://baike.baidu.com/item/Spring)框架无缝集成。

Dubbo是一款高性能、轻量级的开源Java RPC框架，它提供了三大核心能力：面向接口的远程方法调用，智能容错和负载均衡，以及服务自动注册和发现。

### 开启zookeeper

启动开启zookeeper做注册中心 (记住IP和端口)

1. [linux](https://www.yangzhenyu.com.cn/1/)
2. [windows](https://www.yangzhenyu.com.cn/10442/#zookeeper-%E7%8E%AF%E5%A2%83%E6%90%AD%E5%BB%BA)

### 创建一个根项目

#### 创建一个maven项目作为根项目

pom.xml

```xml
<groupId>cn.com.yangzhenyu</groupId>
<artifactId>dubbo-boot</artifactId>
<packaging>pom</packaging>
<version>1</version>
<modules>
    <module>dubbo-api</module>
    <module>dubbo-consumer</module>
    <module>dubbo-provider</module>
</modules>
```

#### 创建module (dubbo-api)

dubbo 放着实体类和对应的接口是公共的类

##### UserVo.java

```java
package cn.com.yangzhenyu.bean;

import lombok.Data;

import java.io.Serializable;

@Data
public class UserVo implements Serializable {
    private int id;
    private String name;
    private String address;
}
```

> 一定要实现 Serializable 接口 序列化

##### IUserService

```java
public interface IUserService {

    UserVo findUser();

}
```

#### 创建module (dubbo-provider) 接口提供者

基于SpringBoot 2.4.3版本和Dubbo2.7.8实现

##### pom.xml 

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.4.3</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>
    <groupId>cn.com.yangzhenyu</groupId>
    <artifactId>dubbo-provider</artifactId>
    <version>1</version>
    <name>dubbo-provider</name>
    <description>Dubbo Provider project for Spring Boot</description>
    <properties>
        <java.version>1.8</java.version>
        <dubbo.version>2.7.8</dubbo.version>
    </properties>
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter</artifactId>
        </dependency>

        <!-- dubbo的依赖 -->
        <dependency>
            <groupId>org.apache.dubbo</groupId>
            <artifactId>dubbo-spring-boot-starter</artifactId>
            <version>${dubbo.version}</version>
        </dependency>

        <!-- zookeeper 需要的依赖-->
        <dependency>
            <groupId>org.apache.dubbo</groupId>
            <artifactId>dubbo-dependencies-zookeeper</artifactId>
            <version>${dubbo.version}</version>
            <type>pom</type>
            <exclusions>
                <exclusion>
                    <groupId>org.slf4j</groupId>
                    <artifactId>slf4j-log4j12</artifactId>
                </exclusion>
            </exclusions>
        </dependency>

        <dependency>
            <groupId>com.101tec</groupId>
            <artifactId>zkclient</artifactId>
            <version>0.11</version>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-devtools</artifactId>
            <scope>runtime</scope>
            <optional>true</optional>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>

        <dependency>
            <groupId>cn.com.yangzhenyu</groupId>
            <artifactId>dubbo-api</artifactId>
            <version>1</version>
            <scope>compile</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>

</project>
```

##### application.properties

```properties
dubbo.registry.address=zookeeper://127.0.0.1:2181

#当前服务/应用的名字
dubbo.application.name=dubbo-provider

#通信规则（通信协议和接口）
dubbo.protocol.name=dubbo
dubbo.protocol.port=100

#连接监控中心
dubbo.monitor.protocol=registry
```

##### DubboProviderApplication

```java
//开启包扫描
@EnableDubbo
@SpringBootApplication
public class DubboProviderApplication {

    public static void main(String[] args) {
        SpringApplication.run(DubboProviderApplication.class, args);
    }

}
```

##### UserServiceImpl

```
@DubboService
public class UserServiceImpl implements IUserService {
    @Override
    public UserVo findUser() {
        UserVo userVo = new UserVo();
        userVo.setId(1);
        userVo.setAddress("shanghai");
        userVo.setName("yzy");
        return userVo;
    }
}
```

> 旧版本中 DubboService 注解是Service //com.alibaba.dubbo.config.annotatio

##### 运行

直接启动SpringBoot项目 出现以下图 说明启动成功

![image-20210304164431566](http://img.yzy.ink/image-20210304164431566.png)

#### 创建module (dubbo-consumer) 接口消费者

基于SpringBoot 2.4.3版本和Dubbo2.7.8实现

##### pom.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.4.3</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>
    <groupId>cn.com.yangzhenyu</groupId>
    <artifactId>dubbo-consumer</artifactId>
    <version>1</version>
    <name>dubbo-consumer</name>
    <description>Dubbo Consumer project for Spring Boot</description>
    <properties>
        <java.version>1.8</java.version>
        <dubbo.version>2.7.8</dubbo.version>
    </properties>
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-devtools</artifactId>
            <scope>runtime</scope>
            <optional>true</optional>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>

        <dependency>
            <groupId>org.apache.dubbo</groupId>
            <artifactId>dubbo-spring-boot-starter</artifactId>
            <version>${dubbo.version}</version>
        </dependency>

        <dependency>
            <groupId>org.apache.dubbo</groupId>
            <artifactId>dubbo-dependencies-zookeeper</artifactId>
            <version>${dubbo.version}</version>
            <type>pom</type>
            <exclusions>
                <exclusion>
                    <groupId>org.slf4j</groupId>
                    <artifactId>slf4j-log4j12</artifactId>
                </exclusion>
            </exclusions>
        </dependency>

        <dependency>
            <groupId>com.101tec</groupId>
            <artifactId>zkclient</artifactId>
            <version>0.11</version>
        </dependency>

        <dependency>
            <groupId>cn.com.yangzhenyu</groupId>
            <artifactId>dubbo-api</artifactId>
            <version>1</version>
            <scope>compile</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>

</project>	
```

##### application.properties

```properties
server.port=9999

dubbo.application.name=dubbo-consumer
dubbo.registry.address=zookeeper://127.0.0.1:2181
dubbo.monitor.protocol=registry
```

##### DubboConsumerApplication

```java
//开启包扫描
@EnableDubbo
@SpringBootApplication
public class DubboConsumerApplication {

    public static void main(String[] args) {
        SpringApplication.run(DubboConsumerApplication.class, args);
    }

}	
```

##### TestController

```java
@RestController
public class TestController {

    @DubboReference
    private IUserService userService;

    @RequestMapping("test")
    public UserVo test(){
        return userService.findUser();
    }
}
```

> DubboReference注解在旧的版本里面是Reference //com.alibaba.dubbo.config.annotatio

##### 运行

![image-20210304165112405](http://img.yzy.ink/image-20210304165112405.png)

### 测试

访问 [http://localhost:9999/test](http://localhost:9999/test)

![image-20210304165152474](http://img.yzy.ink/image-20210304165152474.png)

> 成功远程调用接口成功

程序源码: [https://github.com/yttrium2016/dubbo-boot](https://github.com/yttrium2016/dubbo-boot)

### 其他

1. #### 官方管理 : [https://github.com/apache/dubbo-admin](https://github.com/apache/dubbo-admin)

2. #### 官方的例子 : [https://github.com/apache/dubbo-spring-boot-project](https://github.com/apache/dubbo-spring-boot-project)