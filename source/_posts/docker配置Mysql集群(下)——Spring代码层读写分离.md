---
title: docker配置Mysql集群(下)——Spring代码层读写分离
date: 2021-1-29
tags:
	- Docker
	- MYSQL集群
categories:
	- Spring
toc:
    true
---
上节介绍了利用Docker部署MYSQL集群，下面介绍如何利用部署好的master和slave实现Spring代码层的读写分离，保证发生读操作时访问slave结点，发生写操作时访问master结点。

### <span style="color:orange;">1. 实现基础</span>

**a.spring有关数据源路由的源码**

<!--more-->

```java
package org.springframework.jdbc.datasource.lookup;

import java.sql.Connection;
import java.sql.SQLException;
import java.util.HashMap;
import java.util.Iterator;
import java.util.Map;
import java.util.Map.Entry;
import javax.sql.DataSource;
import org.springframework.beans.factory.InitializingBean;
import org.springframework.jdbc.datasource.AbstractDataSource;
import org.springframework.util.Assert;

public abstract class AbstractRoutingDataSource extends AbstractDataSource implements InitializingBean {
    private Map<Object, Object> targetDataSources;
    private Object defaultTargetDataSource;
    private boolean lenientFallback = true;
    private DataSourceLookup dataSourceLookup = new JndiDataSourceLookup();
    private Map<Object, DataSource> resolvedDataSources;
    private DataSource resolvedDefaultDataSource;
    
    protected DataSource determineTargetDataSource() {
        Assert.notNull(this.resolvedDataSources, "DataSource router not initialized");
        Object lookupKey = this.determineCurrentLookupKey();
        DataSource dataSource = (DataSource)this.resolvedDataSources.get(lookupKey);
        if (dataSource == null && (this.lenientFallback || lookupKey == null)) {
            dataSource = this.resolvedDefaultDataSource;
        }

        if (dataSource == null) {
            throw new IllegalStateException("Cannot determine target DataSource for lookup key [" + lookupKey + "]");
        } else {
            return dataSource;
        }
    }

    protected abstract Object determineCurrentLookupKey();
    
    ...
}
```
首先看一下在`springframework.jdbc`包下的源码，类名`AbstractRoutingDataSource`（抽象路由数据源），这个虚拟类就是spring提供的提供路由导向的数据库类

- 通过变量Map存储不同方法的数据源
- `determineTargetDataSource`方法通过调用`determineCurrentLookupKey`来指明使用何种数据库，然后在Map中寻找相应的datasource

我们可以通过实现上面`determineCurrentLookupKey`抽象方法，通过建立一个Map存储master和slave的数据源，通过SQL对应是update还是insert操作，来决定使用何种key，进而调用哪个数据源



**b.Mybatis提供Interceptor**

```java
package org.apache.ibatis.plugin;

import java.util.Properties;

public interface Interceptor {
    Object intercept(Invocation var1) throws Throwable;

    Object plugin(Object var1);

    void setProperties(Properties var1);
}
```
`Interceptor`是mybatis提供一个接口，是拦截类，可以拦截mybatis中的操作

- `plugin`方法：定义拦截何种类型的类，以决定是否织入代理
- `intercept`方法：定义对拦截下来的类织入的逻辑

通过自定义实现接口中的两个方法，我们可以使用plugin方法仅拦截属于Mybatis底层SQL执行器的类，`interceptor`方法可以编写对应拦截到执行器的执行SQL方法





**c.如何将`Interceptor`方法和`determineCurrentLookupKey`方法以及具体不同数据库操作的线程统筹**

这里用到了一个中间人——**ThreadLocal**，通过ThreadLocal来隔绝不同数据库线程的要选择的key。
具体而言为：Interceptor通过拦截执行器对应的数据库操作，查看是属于读还是写操作，来向ThreadLocal中放入对应线程选择数据源的名字（master or slave），然后通过实现AbstractRoutingDataSource中的determineCurrentLookupKey方法去从ThreadLocal中取对应线程放入的key，然后根据对应的key在Map型变量targetDataSources选择相应的数据源

### <span style="color:orange;">2.实现方式</span>
**a.DynamicDataSourceHolder类**

主要定义中间人池子ThreadLocal来隔离不同线程放入的key，其中可选择的key定义为静态变量"master"和"slave"
```java
package com.imooc.o2o.dao.split;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

/**
 * 池子，放入决定数据源key的池子，其中池子利用ThreadLocal设置为线程安全
 */
public class DynamicDataSourceHolder {
    private static Logger logger = LoggerFactory.getLogger(DynamicDataSourceHolder.class);
    // 池子，利用ThreadLocal保证线程安全，线程安全是访问同一个变量时容器出现问题，特别多个线程对一个变量进行写入的时候
    // ThreadLocal是除了加锁之外的另外一种保证多线程访问变量的线程安全方法，即每个线程对变量的访问都是基于线程自己的变量
    // 也就是说中央变量只有一个，但是每个线程都有一个中央变量的独立拷贝，每个线程只访问修改自己独立拷贝的变量
    private static ThreadLocal<String> contextHolder = new ThreadLocal<String>();
    public static String DB_MASTER = "master";
    public static String DB_SLAVE = "slave";

    public static String getDbType(){
        String db = contextHolder.get();
        if(db == null){
            db = DB_MASTER;
        }
        return db;
    }

    /**
     * 设置线程的dbType
     * @param str
     */
    public static void setDbType(String str){
        logger.debug("所使用的数据源为：" + str);
        contextHolder.set(str);
    }

    /**
     * 清理连接类型
     */
    public static void clearDbType(){
        contextHolder.remove();
    }
}
```





**b.DynamicDataSourceInterceptor类**

通过实现Mybatis提供的Interceptor接口来拦截Mybatis执行器Executor，intercept方法定义织入的规则，通过invocation.getArgs()来获取对应线程执行的数据库操作是读操作还是写操作，然后向DynamicDataSourceHolder中的ThreadLocal放入对应线程选择的key，是选择master还是slave



**读写分离逻辑**

这里首先先判断是否为事务操作

+ 如果是事务操作，会包含一系列增删改查的操作，所以使用master数据库

- 如果不是事务操作，然后再看对应增删改查是读取还是写入操作，从而实现读写分离的逻辑
```java
package com.imooc.o2o.dao.split;

import org.apache.ibatis.executor.Executor;
import org.apache.ibatis.executor.keygen.SelectKeyGenerator;
import org.apache.ibatis.mapping.BoundSql;
import org.apache.ibatis.mapping.MappedStatement;
import org.apache.ibatis.mapping.SqlCommandType;
import org.apache.ibatis.plugin.*;
import org.apache.ibatis.session.ResultHandler;
import org.apache.ibatis.session.RowBounds;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.transaction.support.TransactionSynchronizationManager;

import java.util.Locale;
import java.util.Properties;

/**
 * 拦截器拦截mybatis传递进来的SQL信息，根据SQL信息来使用相应的读写分离的数据源
 */
// mybatis会把增删改的方法封装在update这个method中
@Intercepts({@Signature(type=Executor.class, method="update", args={MappedStatement.class, Object.class}),
@Signature(type=Executor.class, method="query", args={MappedStatement.class, Object.class, RowBounds.class, ResultHandler.class})})
public class DynamicDataSourceInterceptor implements Interceptor {
    private static Logger logger = LoggerFactory.getLogger(DynamicDataSourceInterceptor.class);
    private static final String REGEX = ".*insert\\u0020.*|.*update\\u0020.*|.*delete\\u0020.*";

    /**
     * 织入的规则
     * @param invocation
     * @return
     * @throws Throwable
     */
    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        // 利用spring提供的，判断当前是不是事务已表明是不是数据库操作
        boolean synchronizationActive = TransactionSynchronizationManager.isActualTransactionActive();
        Object[] objects = invocation.getArgs();
        MappedStatement ms = (MappedStatement)objects[0]; // 参数第一个是解释什么数据库操作
        String lookupKey = DynamicDataSourceHolder.DB_MASTER;
        if(synchronizationActive != true){
            // 读方法，
            if(ms.getSqlCommandType().equals(SqlCommandType.SELECT)){
                // selectKey 为自增id查询主键（SELECT LAST_INSERT_ID())方法，使用主库，通常使用自增主键插入数据库操作
                if(ms.getId().contains(SelectKeyGenerator.SELECT_KEY_SUFFIX)){
                    lookupKey = DynamicDataSourceHolder.DB_MASTER;
                }else{
                    BoundSql boundSql = ms.getSqlSource().getBoundSql(objects[1]);
                    String sql = boundSql.getSql().toLowerCase(Locale.CHINA).replaceAll("[\\t\\n\\r]", "");
                    if(sql.matches(REGEX)){
                        lookupKey = DynamicDataSourceHolder.DB_MASTER;
                    }else{
                        lookupKey = DynamicDataSourceHolder.DB_SLAVE;
                    }
                }
            }
        }else{
            // 因为一般事务操作可能包含增删改查混合操作，所以用master
            lookupKey = DynamicDataSourceHolder.DB_MASTER;
        }
        logger.debug("设置方法[{}] use[{}] Strategy,SqlCommandType[{}]...", ms.getId(), lookupKey, ms.getSqlCommandType().name());
        DynamicDataSourceHolder.setDbType(lookupKey);
        return invocation.proceed();
    }

    /**
     * 返回封装好的代理对象，决定返回的是本体还是编织好的代理
     * @param target
     * @return
     */
    @Override
    public Object plugin(Object target) {
        // 这里的Executor是mybatis支持一系列增删改查操作的执行器
        if(target instanceof Executor){
            // 如果是mybatis的Executor类型，那么就把编入逻辑织入，不是返回本体
            return Plugin.wrap(target, this);
        }
        return target;
    }

    /**
     * 非必要
     * @param properties
     */
    @Override
    public void setProperties(Properties properties) {

    }
}
```




**c.DynamicDataSource类**

继承AbstractRoutingDataSource，实现determineCurrentLookupKey方法，从池子中获取对应线程应该选择的数据源名称key，然后在targetDataSource中获取对应的数据源连接给线程（targetDataSource是个Map变量，在spring-dao.xml配置文件中配置）
```java
package com.imooc.o2o.dao.split;

import org.springframework.jdbc.datasource.lookup.AbstractRoutingDataSource;

/**
 * 从池子中取key，然后从HashMap<String,DataSource>中用key找到对应的DataSource
 */
public class DynamicDataSource extends AbstractRoutingDataSource {
    @Override
    protected Object determineCurrentLookupKey() {
        return DynamicDataSourceHolder.getDbType();
    }
}

```


### <span style="color:orange;">3.装配</span>

1. mybatis-config中装配拦截器

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration
        PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>
    <!-- 配置全局属性 -->
    <settings>
        <!-- 使用jdbc的getGeneratedKeys获取数据库自增主键值 -->
        <setting name="useGeneratedKeys" value="true"/>

        <!-- 使用列标签（数据库列名）替换列名（查询语句名字） 默认:true -->
        <setting name="useColumnLabel" value="true"/>

        <!-- 开启驼峰命名转换:Table{create_time} -> Entity{createTime} -->
        <setting name="mapUnderscoreToCamelCase" value="true"/>
    </settings>

    <!--自定义读写分离拦截器-->
    <plugins>
        <plugin interceptor="com.imooc.o2o.dao.split.DynamicDataSourceInterceptor"/>
    </plugins>
</configuration>
```

2. spring-dao.xml配置路由数据源

* 设置abstract为true的连接池，定义为abstract是为了后边继承通用的连接池属性

- 定义好master和slave的数据源连接配置，指明parent为上面abstract的连接池
- 将继承实现AbstractRoutingDataSourc    e的类，配置好Map变量targetDataSource
- 定义为懒加载，因为只在SQL语句正式执行的时候才指定出来dataSource

jdbc.properties文件：

	jdbc.master.driver=com.mysql.cj.jdbc.Driver
	jdbc.master.url=jdbc:mysql://localhost:10001/o2o?useUnicode=true&characterEncoding=utf8
	jdbc.master.username=root
	jdbc.master.password=make1234
	jdbc.slave.driver=com.mysql.cj.jdbc.Driver
	jdbc.slave.url=jdbc:mysql://localhost:10002/o2o?useUnicode=true&characterEncoding=utf8
	jdbc.slave.username=root
	jdbc.slave.password=make1234
spring-dao.xml文件：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:p="http://www.springframework.org/schema/p" xmlns:bean="http://www.springframework.org/schema/context"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd
       http://www.springframework.org/schema/context
       http://www.springframework.org/schema/context/spring-context.xsd">

    <context:component-scan base-package="com.imooc.o2o.dao"/>

    <!--配置数据库properties的数据-->
    <context:property-placeholder location="classpath:jdbc.properties"/>

    <!--1.数据库连接池，设置为abstract，便于后面继承属性配置主从库-->
    <bean id="abstractDataSource" class="com.mchange.v2.c3p0.ComboPooledDataSource"
        abstract="true" destroy-method="close">
        <!--c3p0连接池的私有属性-->
        <property name="maxPoolSize" value="30"/>
        <property name="minPoolSize" value="10"/>
        <!--关闭连接后不自动commit-->
        <property name="autoCommitOnClose" value="false"/>
        <!--获取连接的超时时间-->
        <property name="checkoutTimeout" value="10000"/>
        <!--当获取连接失败重试次数-->
        <property name="acquireRetryAttempts" value="10"/>
    </bean>

    <!--2.主库配置-->
    <bean id="master" parent="abstractDataSource">
        <!--配置连接池属性-->
        <property name="driverClass" value="${jdbc.master.driver}"/>
        <property name="jdbcUrl" value="${jdbc.master.url}"/>
        <property name="user" value="${jdbc.master.username}"/>
        <property name="password" value="${jdbc.master.password}"/>
    </bean>

    <!--3.从库配置-->
    <bean id="slave" parent="abstractDataSource">
        <!--配置连接池属性-->
        <property name="driverClass" value="${jdbc.slave.driver}"/>
        <property name="jdbcUrl" value="${jdbc.slave.url}"/>
        <property name="user" value="${jdbc.slave.username}"/>
        <property name="password" value="${jdbc.slave.password}"/>
    </bean>

    <!--4.配置动态数据源，这儿的targetDataSource就是路由数据源所对应的名称-->
    <bean id="dynamicDataSource" class="com.imooc.o2o.dao.split.DynamicDataSource">
        <!--将数据源放入到map中，这里的key要与com.imooc.o2o.dao.split.DynamicDataSourceHolder中的key相同-->
        <property name="targetDataSources">
            <map>
                <entry value-ref="master" key="master"></entry>
                <entry value-ref="slave" key="slave"></entry>
            </map>
        </property>
    </bean>

    <!--5.懒加载，需要在SQL语句正式执行的时候才指定出来dataSource-->
    <bean id="dataSource" class="org.springframework.jdbc.datasource.LazyConnectionDataSourceProxy">
        <property name="targetDataSource">
            <ref bean="dynamicDataSource"></ref>
        </property>
    </bean>

    <!--配置SqlSessionFactory对象-->
    <!--MyBatis全局配置文件-->
    <!--扫描entity包，将取出结果包装成entity对象-->
    <!--扫描Sql配置文件-->
    <bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean"
          p:dataSource-ref="dataSource"
          p:configLocation="classpath:mybatis-config.xml"
          p:typeAliasesPackage="com.imooc.o2o.entity"
          p:mapperLocations="classpath:mapper/*.xml"/>

    <!--配置扫描Dao接口包，动态实现Dao接口，注入到Spring容器中-->
    <bean class="org.mybatis.spring.mapper.MapperScannerConfigurer">
        <!--注入SqlSessionFactory-->
        <property name="sqlSessionFactoryBeanName" value="sqlSessionFactory"/>
        <!--给出需要扫描Dao接口包-->
        <property name="basePackage" value="com.imooc.o2o.dao"/>
    </bean>
</beans>
```

