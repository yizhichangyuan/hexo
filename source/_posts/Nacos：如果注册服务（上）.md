---
title: Nacos：如何注册服务（上）
date: 2021-3-14
tags:
	- Nacos
categories:
	- SpringCloud
toc: true
---

SpringCloud-Alibaba-Nacos是阿里出品的一个服务发现与注册的组件，微服务需要调用其他微服务模块的服务或者微服务模块需要被其他模块调用，都需要到Nacos注册中心中注册该微服务模块

**如何微服务模块注册到注册中心，主要是以下几个步骤**

1.在微服务模块引入Nacos的依赖

2.启动Nacos服务器

3.微服务模块设置好注册中心的接口以及该微服务模块需要注册到服务中心的名称

4.在启动类加入 @EnableDiscoveryClient 注解表示该服务注册到Nacos中

<!--more-->

### 一、引入Nacos依赖

首先需要引入SpringCloud-Alibaba组件包，其中就有Nacos，由于我们每个微服务都需要将自身注册到注册中心以及发现要调用的服务地址，因此我们首先将SpringCloud-Alibaba组件包放置在Common通用模块下，然后Common模块引入Nacos依赖，然后下个每个微模块通过引用common模块来实现引入Common模块的Nacos依赖以及公共工具类、Bean等

Common模块--对应的Module为Maven

```xml
 <?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
        <parent>
        <artifactId>doermail</artifactId>
        <groupId>com.lookstarry.doermail</groupId>
        <version>0.0.1-SNAPSHOT</version>
    </parent>
    <modelVersion>4.0.0</modelVersion>

    <artifactId>doermail-common</artifactId> 

  <!--所有微服务共同的依赖，注意这里的dependency不能有scope标签，这样其他微服务模块好像无法引入该依赖，要去掉</scope>标签-->
  <dependencies>
    <!--Nacos微服务发现与注册-->
        <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
        </dependency> 
    </dependencies>

    <!--dependencyManagement含义表示依赖管理，
    以后再dependencies中引入spring-cloud-alibaba依赖不需要指定版本号-->
    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>com.alibaba.cloud</groupId>
                <artifactId>spring-cloud-alibaba-dependencies</artifactId>
                <version>2.1.0.RELEASE</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>
```

其他微服务模块如何引用Common模块下的依赖以及工具类呢？

通过在其他模块的pom.xml中dependencies中引入Common模块的信息

```xml
       <dependency>
            <groupId>com.lookstarry.doermail</groupId>
            <artifactId>doermail-common</artifactId>
            <version>0.0.1-SNAPSHOT</version>
        </dependency>
```

这样其他微服务模块都能引入Common模块下nacos



### 二、下载并启动 Nacos Server

1.首先需要获取 Nacos Server，支持直接下载和源码构建两种方式。

​	直接下载：[Nacos Server 下载页](https://github.com/alibaba/nacos/releases)

​    源码构建：进入 Nacos [Github 项目页面](https://github.com/alibaba/nacos)，将代码 git clone 到本地自行编译打包，[参考此文档](https://nacos.io/zh-cn/docs/quick-start.html)。**推荐使用源码构建方式以获取最新版本**

2.启动 Server，进入解压后文件夹或编译打包好的文件夹，找到如下相对文件夹 nacos/bin，并对照操作系统实际情况之下如下命令。

* Linux/Unix/Mac 操作系统，执行命令 `sh startup.sh -m standalone`

* Windows 操作系统，执行命令 `cmd startup.cmd`

<div align='center'><img src="1615625705374-a7db4ed1-7021-4e98-9eee-058ae34afd4d.png" alt="image.png" style="zoom:50%;" /></div>

### 三、微服务模块引入Nacos依赖

​	在Common模块修改 pom.xml 文件，引入 Nacos Discovery Starter，在第一步在Common模块已经引入，所以微服务模块不需要重复引入

```
<dependency>
     <groupId>com.alibaba.cloud</groupId>
     <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
 </dependency>
```



### 四、微服务模块配置注册中心Nacos地址以及该微服务注册到注册中心的地址

1.在微服务应用的 /src/main/resources/application.properties 配置文件中配置 Nacos Server 地址，

此外还需要配置注册服务的名称，否则无法注册服务

```properties
spring.cloud.nacos.discovery.server-addr=127.0.0.1:8848
spring.application.name=doermail-coupon
```

2.使用 @EnableDiscoveryClient 注解在微服务的启动类上开启服务注册与发现功能

```java
 @SpringBootApplication
 @EnableDiscoveryClient //本微服务开启服务注册与发现
 public class ProviderApplication {
    public static void main(String[] args) {
        SpringApplication.run(ProviderApplication.class, args);
    }
 }
```

3.启动该微服务，控制台就会出现如下，表示该微服务注册到了nacos中

<div align='center'><img src="1615627294785-e203d13a-6656-44b9-b5e2-69f5a32017d1.png" style="zoom:50%;" /></div>

4.浏览器访问nacos提供的可视化页面http://127.0.0.1:8848/nacos/,用户名和密码均为nacos

就可以看到刚刚注册的微服务doermail-coupon，名称为我们指定的application-name

<div align='center'><img src="1615627330848-e62fcb72-d405-48f1-8e27-0053b1fcf750.png" alt="截屏2021-03-13 17.22.05.png" style="zoom:50%;" /></div>