---
title: Redis使用
date: 2021-2-25
tags:
	- Redis
categories:
	- Spring
toc: true
---

> redis是一个key-value型存储系统，支持的value类型比较丰富，支持String、List、集合set（没有顺序，元素唯一）、有序集合Zset，都支持push、pop、add、remove、交并集等操作，这些操作都是原子性的，调用redis的操作不用考虑多进程之间的并发问题。在此基础上，redis支持各种方式的排序，为了保证效率，数据都是缓存在内存中的，区别在于：redis会周期性的把更新的数据写到磁盘或者修改操作追加的记录文件中，前者就是默认的RDD模式，就是把数据写到临时文件中，持久化结束后，把临时文件替换为上次持久化的文件，达到数据恢复的目的，优点在于：使用单独子进程进行持久化，主进程不会进行任何IO操作，保证了redis的高效性能，缺点在于RDD是间隔一段时间进行持久化，如果持久化间隔期间，redis发生故障，那么这个持久化就不会更新到磁盘中，就会发生数据的丢失。
>
> <!--more-->对于数据不那么严谨的应用，可以使用RDD模式。其存储机制默认设置为如果更改了一个key，则间隔900秒之后进行一次持久化存储，如果更改了十个key，则间隔300秒进行一次持久化存储，如果更改了一万个key，则1分钟之后进行持久化，持久化之后就会把最新的临时文件替换掉旧的RDD文件，使用RDD恢复数据，实际就是重启redis，redis会从rdd文件恢复数据；第二种就是AOF操作，就是把执行过的指令记录下来，恢复时就是把这些操作重新回放一遍，优点就是可以保持更高的数据完整性。如果追加file的时间是1s，一旦redis发生故障那么最多丢失数据的时间是1s，这一秒的数据丢失，且如果日志写入不完整，可以使用redis-check-aof对日志文件进行修复，aof文件没有被write之前，可以删除一些其中操作，缺点就是文件比RDD文件大，恢复的时间比较久，其实AOF和MYSQL中的binlog差不多，都是记录操作。

使用意图：就是将一些不经常更改的数据写入到redis中，例如店铺类别、区域信息、头条信息等，这些数据都是读取比较多，写比较少。

* 客户端读取数据：首先把请求发送给redis，如果redis有请求的数据那么就把数据直接返回给客户端，避免使用数据库，如果redis没有请求的数据，就把请求发给数据库，数据库获取到数据后，返回给客户端，同时把redis中的数据进行更新，保证redis始终返回的是新的数据。

* 客户端写入/修改数据：当数据发生变化或者修改，要同步的更新redis中的数据，简单点的操作就是数据发生修改时，就将redis相应的key的数据清空，如果客户端请求最新数据，因为redis没有，所以请求发给了DB，DB返回数据给客户端，同时把最新数据插入到了redis，保证了redis是最新的数据。

  <div align='center'><img src="1612949134565-6f08df50-431b-40ba-ba8e-eac9104e8f9d-20210225113349251.png" alt="image-20210129220709957" style="zoom:50%;" /></div>

### 业务逻辑

首先先看一下业务逻辑读和写时如何加入Redis缓存，对于读多写少数据，使用Redis缓存时明智的例如滚动展示的头条。采用Redis缓存，需要先定义存放的key，对于不同条件查询对象配置不同的key

当遇到请求，先检查Redis是否存在该key，否则从数据库取出然后放入到Redis中，注意不同的查询条件对应的数据key不可相同，否则就乱套了

```java
package com.imooc.o2o.service.impl;

import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.JavaType;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.imooc.o2o.cache.JedisUtil;
import com.imooc.o2o.dao.ShopCategoryDao;
import com.imooc.o2o.entity.ShopCategory;
import com.imooc.o2o.exceptions.ShopOperationException;
import com.imooc.o2o.service.ShopCategoryService;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

import java.io.IOException;
import java.util.ArrayList;
import java.util.List;

@Service
public class ShopCategoryServiceImpl implements ShopCategoryService {
    @Autowired
    private ShopCategoryDao shopCategoryDao;
    @Autowired
    private JedisUtil.Keys keys;
    @Autowired
    private JedisUtil.Strings strings;

    private Logger logger = LoggerFactory.getLogger(ShopCategoryServiceImpl.class);

    @Override
    public List<ShopCategory> queryShopCategory(ShopCategory shopCategoryCondition) {
        String key = SHOP_CATEGORY_KEY;
        List<ShopCategory> list = null;
        ObjectMapper mapper = new ObjectMapper();
        if(shopCategoryCondition == null){
          // 查询一级店铺类别
            key = SHOP_CATEGORY_KEY + "_parent_null"; 
        }else if(shopCategoryCondition.getParent() != null && shopCategoryCondition.getParent().getShopCategoryId() != null){
          // 查询对应以及类别下的二级类别
            key = SHOP_CATEGORY_KEY + "_parent_" + shopCategoryCondition.getParent().getShopCategoryId();
        }else{
          // 查询所有二级店铺类别
            key = SHOP_CATEGORY_KEY + "_parent_not_null_all"; 
        }

      	// Reids不存在则从数据库取出，取出成功后放入到缓存中
        if(!keys.exists(key)){
            list = shopCategoryDao.queryShopCategory(shopCategoryCondition);
            try {
              	// 因为是对象，所以这里采用转换为JSON字符串转入
                String jsonString = mapper.writeValueAsString(list);
                strings.set(key, jsonString);
            } catch (JsonProcessingException e) {
                logger.error("queryShopCategory error:" + e.getMessage());
                e.printStackTrace();
                throw new ShopOperationException("queryShopCategory error:" + e.getMessage());
            }
        }else{
          	// 从Redis缓存中读出字符串，经过JSON转换为对象
             String jsonString = strings.get(key);
             JavaType javaType = mapper.getTypeFactory().constructParametricType(ArrayList.class, ShopCategory.class);
            try {
                list = mapper.readValue(jsonString, javaType);
            } catch (IOException e) {
                logger.error("queryShopCategory error:" + e.getMessage());
                e.printStackTrace();
                throw new ShopOperationException("queryShopCategory error:" + e.getMessage());
            }
        }
        return list;
    }
    
    // 插入数据需要清除所有相关Redis缓存
    public ShopCategoryExecution insertShopCategory(ShopCategory shopCategory){
        try{
            int effectNum = shopCategoryDao.insertShopCategory(shopCategory);
            // 插入成功则需要清除Redis缓存
            if(effectNum > 0){
                // 将所有key为SHOP_CATEGORY_KEY打头的数据全部从Redis缓存中清除
                cacheService.removeKeys(SHOP_CATEGORY_KEY);
                return ShopCategoryExecution(ShopCategoryStateEnum.SCCESS);
            }else{
                return ShopCategoryExecution(ShopCategoryStateEnum.FAIL);
            }
        }catch(Exception e){
            logger.error("插入失败：" + e.toString());
            throw new ShopCategoryException(e.toString());
        }
    }
}
```



在添加数据时注意清除缓存，下次取就会直接去数据库进行取然后放入缓存中，保证缓存中始终是最新的数据

```java
package com.imooc.o2o.service.impl;//package com.imooc.o2o.service.impl;

import com.imooc.o2o.cache.JedisUtil;
import com.imooc.o2o.service.CacheService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

import java.util.Set;

@Service
public class CacheServiceImpl implements CacheService {
    @Autowired
    private JedisUtil.Keys keys;

    @Override
    public void removeKeys(String prefix) {
        // 首先找到所有以prefix为前缀的key集合
        Set<String> set = keys.keys(prefix + "*");
        for(String key : set){
            keys.del(key);
        }
    }
}
```



### 实现Redis配置

配置Redis，首先pom中引入Redis客户端，与远程Redis进行通信

```xml
        <!--redis客户端:Jedis，与redis服务器进行通信-->
        <dependency>
            <groupId>redis.clients</groupId>
            <artifactId>jedis</artifactId>
            <version>2.9.0</version>
        </dependency>
```

application.properties引入redis基础配置，这些配置后续会通过编码读入

```properties
# reids配置
redis.hostname=127.0.0.1
redis.port=6379
redis.database=0
redis.pool.maxActive=600
redis.pool.maxIdle=300
redis.pool.maxWait=3000
redis.pool.testOnBorrow=true
```

**连接池JedisPool**

读取Redis配置生成Redis连接池JedisPool

```java
package com.imooc.o2o.cache;

import org.springframework.context.annotation.Bean;
import redis.clients.jedis.JedisPool;
import redis.clients.jedis.JedisPoolConfig;

/**
 * 强指定redis的JedisPool接口构造函数，这样才能在centos中成功创建jedispool
 */
public class JedisPoolWriper {
    private JedisPool jedisPool;

    public JedisPoolWriper() {
    }

    public JedisPoolWriper(final JedisPoolConfig poolConfig, final String host, final int port) {
        try {
            jedisPool = new JedisPool(poolConfig, host, port);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    public JedisPool getJedisPool() {
        return jedisPool;
    }

    public void setJedisPool(JedisPool jedisPool) {
        this.jedisPool = jedisPool;
    }
}
```



**注入Redis配置到容器**

包括Redis连接池配置JedisPoolConfig，JedisPool连接，Jedis基本数据常见API

其中可以看到添加@Configuration注解，就是读取相应application.properties配置

JedisPoolConfig、JedisPool都添加了@Bean注解，需要在项目启动时注入到容器中

```java
package com.imooc.o2o.config.redis;

import com.imooc.o2o.cache.JedisPoolWriper;
import com.imooc.o2o.cache.JedisUtil;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import redis.clients.jedis.JedisPoolConfig;

@Configuration
public class RedisConfiguration {
    @Value("${redis.pool.maxActive}")
    private int maxActive;
    @Value("${redis.hostname}")
    private String hostname;
    @Value("${redis.port}")
    private int port;
    @Value("${redis.pool.maxIdle}")
    private int maxIdle;
    @Value("${redis.pool.maxWait}")
    private int maxWait;
    @Value("${redis.pool.testOnBorrow}")
    private boolean testOnBorrow;

    @Autowired
    private JedisPoolConfig jedisPoolConfig;
    @Autowired
    private JedisPoolWriper jedisWritePool;
    @Autowired
    private JedisUtil jedisUtil;

    /**
     * redis连接池配置
     * @return
     */
    @Bean("jedisPoolConfig")
    public JedisPoolConfig createJedisPoolConfig() {
        JedisPoolConfig jedisPoolConfig = new JedisPoolConfig();
        // 控制一个pool可以分配多少个jedis实例
        jedisPoolConfig.setMaxTotal(maxActive);
        // 连接池最多空闲多少个连接
        // 表示解释没有数据库连接时依然保持的链接数目，而不被清除，随时处于待命状态
        jedisPoolConfig.setMaxIdle(maxIdle);
        // 最大等待时间：但没有可用连接时，连接池等待连接被归还的最大时间，超出时间则抛出异常
        jedisPoolConfig.setMaxWaitMillis(maxWait);
        // 在获取连接时检查有效性
        jedisPoolConfig.setTestOnBorrow(testOnBorrow);
        return jedisPoolConfig;
    }

    /**
     * 创建Redis连接池，并做相关配置
     * @return
     */
    @Bean("jedisWritePool")
    public JedisPoolWriper createJedisPoolWriper(){
        JedisPoolWriper jedisPoolWriper = new JedisPoolWriper(jedisPoolConfig, hostname, port);
        return jedisPoolWriper;
    }

    /**
     * 创建Redis工具类，封装好redis的连接已进行相关操作
     * @return
     */
    @Bean("jedisUtil")
    public JedisUtil createJedisUtil(){
        JedisUtil jedisUtil = new JedisUtil();
        jedisUtil.setJedisPool(jedisWritePool);
        return jedisUtil;
    }

    /**
     * jedis的key操作
     * @return
     */
    @Bean("jedisKeys")
    public JedisUtil.Keys createKeys(){
        // 注意内部类的new方式
        JedisUtil.Keys keys = jedisUtil.new Keys();
        return keys;
    }

    /**
     * jedis的Strings操作
     * @return
     */
    @Bean("jedisStrings")
    public JedisUtil.Strings createStrings(){
        JedisUtil.Strings strings = jedisUtil.new Strings();
        return strings;
    }
}
```

**编写Jedis工具类**

将Jedis常用数据对象浅浅的封装一层，封装一层的目的在于：使用第三方组件时，浅浅封装一层避免直接调用API，可以在后续项目时随时更改，而项目其他地方不用更改，例如后续项目需要监控Redis取缓存时间，那么朱旭在封装前后，统计一下时间即可

```java
        /**
         * 根据key获取记录
         *
         * @param key
         * @return 值
         */
        public String get(String key) {
            // ShardedJedis sjedis = getShardedJedis();
            Jedis sjedis = getJedis();
            String value = sjedis.get(key);
            sjedis.close();
            return value;
        }
```

项目需要统计取key消耗时间

```java
        /**
         * 根据key获取记录，统计操作时间
         *
         * @param key
         * @return 值
         */
        public String get(String key) {
            // ShardedJedis sjedis = getShardedJedis();
            long startTime = System.currentTimeMillis();
            System.get
            Jedis sjedis = getJedis();
            String value = sjedis.get(key);
                long endTime = System.currentTimeMillis();
                logger.info(String.format("get Strings key:%d cost %d ms"), key, endTime - startTime);
            sjedis.close();
            return value;
        }
```



**JedisUtil工具类**

JedisUtil浅浅封装一层Redis常见数据对象的方法

```java
package com.imooc.o2o.cache;

import redis.clients.jedis.BinaryClient;
import redis.clients.jedis.Jedis;
import redis.clients.jedis.JedisPool;
import redis.clients.jedis.SortingParams;
import redis.clients.util.SafeEncoder;

import java.util.List;
import java.util.Map;
import java.util.Set;

public class JedisUtil {
    /**
     * 缓存生存时间
     */
    private final int expire = 60000;
    /**
     * 操作Key的方法
     */
    public Keys KEYS;
    /**
     * 对存储结构为String类型的操作
     */
    public Strings STRINGS;
    /**
     * 对存储结构为List类型的操作
     */
    public Lists LISTS;
    /**
     * 对存储结构为Set类型的操作
     */
    public Sets SETS;
    /**
     * 对存储结构为HashMap类型的操作
     */
    public Hash HASH;

    private JedisPool jedisPool;

    public JedisPool getJedisPool() {
        return jedisPool;
    }

    public void setJedisPool(JedisPoolWriper jedisPoolWriper) {
        this.jedisPool = jedisPoolWriper.getJedisPool();
    }

    public JedisPool getPool() {
        return jedisPool;
    }

    /**
     * 从jedis连接池中获取获取jedis对象
     *
     * @return
     */
    public Jedis getJedis() {
        return jedisPool.getResource();
    }

    /**
     * 设置过期时间
     *
     * @param @param
     * @author ruan 2013-4-11
     */
    public void expire(String key, int seconds) {
        if (seconds <= 0) {
            return;
        }
        Jedis jedis = getJedis();
        jedis.expire(key, seconds);
        jedis.close();
    }

    /**
     * 设置默认过期时间
     *
     * @param
     * @author ruan 2013-4-11
     */
    public void expire(String key) {
        expire(key, expire);
    }

    // *******************************************Keys*******************************************//
    public class Keys {

        /**
         * 清空所有key
         */
        public String flushAll() {
            Jedis jedis = getJedis();
            String stata = jedis.flushAll();
            jedis.close();
            return stata;
        }

        /**
         * 更改key
         *
         * @param oldkey
         * @param newkey
         * @return 状态码
         */
        public String rename(String oldkey, String newkey) {
            return rename(SafeEncoder.encode(oldkey),
                    SafeEncoder.encode(newkey));
        }

        /**
         * 更改key,仅当新key不存在时才执行
         *
         * @param oldkey
         * @param newkey
         * @return 状态码
         */
        public long renamenx(String oldkey, String newkey) {
            Jedis jedis = getJedis();
            long status = jedis.renamenx(oldkey, newkey);
            jedis.close();
            return status;
        }

        /**
         * 更改key
         *
         * @param oldkey
         * @param newkey
         * @return 状态码
         */
        public String rename(byte[] oldkey, byte[] newkey) {
            Jedis jedis = getJedis();
            String status = jedis.rename(oldkey, newkey);
            jedis.close();
            return status;
        }

        /**
         * 设置key的过期时间，以秒为单位
         *
         * @param key
         * @param seconds 已秒为单位
         * @return 影响的记录数
         */
        public long expired(String key, int seconds) {
            Jedis jedis = getJedis();
            long count = jedis.expire(key, seconds);
            jedis.close();
            return count;
        }

        /**
         * 设置key的过期时间,它是距历元（即格林威治标准时间 1970 年 1 月 1 日的 00:00:00，格里高利历）的偏移量。
         *
         * @param key
         * @param timestamp 已秒为单位
         * @return 影响的记录数
         */
        public long expireAt(String key, long timestamp) {
            Jedis jedis = getJedis();
            long count = jedis.expireAt(key, timestamp);
            jedis.close();
            return count;
        }

        /**
         * 查询key的过期时间
         *
         * @param key
         * @return 以秒为单位的时间表示
         */
        public long ttl(String key) {
            // ShardedJedis sjedis = getShardedJedis();
            Jedis sjedis = getJedis();
            long len = sjedis.ttl(key);
            sjedis.close();
            return len;
        }

        /**
         * 取消对key过期时间的设置
         *
         * @param key
         * @return 影响的记录数
         */
        public long persist(String key) {
            Jedis jedis = getJedis();
            long count = jedis.persist(key);
            jedis.close();
            return count;
        }

        /**
         * 删除keys对应的记录,可以是多个key
         *
         * @param keys
         * @return 删除的记录数
         */
        public long del(String... keys) {
            Jedis jedis = getJedis();
            long count = jedis.del(keys);
            jedis.close();
            return count;
        }

        /**
         * 删除keys对应的记录,可以是多个key
         *
         * @param keys
         * @return 删除的记录数
         */
        public long del(byte[]... keys) {
            Jedis jedis = getJedis();
            long count = jedis.del(keys);
            jedis.close();
            return count;
        }

        /**
         * 判断key是否存在
         *
         * @param key
         * @return boolean
         */
        public boolean exists(String key) {
            // ShardedJedis sjedis = getShardedJedis();
            Jedis sjedis = getJedis();
            boolean exis = sjedis.exists(key);
            sjedis.close();
            return exis;
        }

        /**
         * 对List,Set,SortSet进行排序,如果集合数据较大应避免使用这个方法
         *
         * @param key
         * @return List<String> 集合的全部记录
         **/
        public List<String> sort(String key) {
            // ShardedJedis sjedis = getShardedJedis();
            Jedis sjedis = getJedis();
            List<String> list = sjedis.sort(key);
            sjedis.close();
            return list;
        }

        /**
         * 对List,Set,SortSet进行排序或limit
         *
         * @param key
         * @param parame 定义排序类型或limit的起止位置.
         * @return List<String> 全部或部分记录
         **/
        public List<String> sort(String key, SortingParams parame) {
            // ShardedJedis sjedis = getShardedJedis();
            Jedis sjedis = getJedis();
            List<String> list = sjedis.sort(key, parame);
            sjedis.close();
            return list;
        }

        /**
         * 返回指定key存储的类型
         *
         * @param key
         * @return String string|list|set|zset|hash
         **/
        public String type(String key) {
            // ShardedJedis sjedis = getShardedJedis();
            Jedis sjedis = getJedis();
            String type = sjedis.type(key);
            sjedis.close();
            return type;
        }

        /**
         * 查找所有匹配给定的模式的键
         *
         * @param pattern key的表达式,*表示多个，？表示一个
         */
        public Set<String> keys(String pattern) {
            Jedis jedis = getJedis();
            Set<String> set = jedis.keys(pattern);
            jedis.close();
            return set;
        }
    }

    // *******************************************Sets*******************************************//
    public class Sets {

        /**
         * 向Set添加一条记录，如果member已存在返回0,否则返回1
         *
         * @param key
         * @param member
         * @return 操作码, 0或1
         */
        public long sadd(String key, String member) {
            Jedis jedis = getJedis();
            long s = jedis.sadd(key, member);
            jedis.close();
            return s;
        }

        public long sadd(byte[] key, byte[] member) {
            Jedis jedis = getJedis();
            long s = jedis.sadd(key, member);
            jedis.close();
            return s;
        }

        /**
         * 获取给定key中元素个数
         *
         * @param key
         * @return 元素个数
         */
        public long scard(String key) {
            // ShardedJedis sjedis = getShardedJedis();
            Jedis sjedis = getJedis();
            long len = sjedis.scard(key);
            sjedis.close();
            return len;
        }

        /**
         * 返回从第一组和所有的给定集合之间的差异的成员
         *
         * @param keys
         * @return 差异的成员集合
         */
        public Set<String> sdiff(String... keys) {
            Jedis jedis = getJedis();
            Set<String> set = jedis.sdiff(keys);
            jedis.close();
            return set;
        }

        /**
         * 这个命令等于sdiff,但返回的不是结果集,而是将结果集存储在新的集合中，如果目标已存在，则覆盖。
         *
         * @param newkey 新结果集的key
         * @param keys   比较的集合
         * @return 新集合中的记录数
         **/
        public long sdiffstore(String newkey, String... keys) {
            Jedis jedis = getJedis();
            long s = jedis.sdiffstore(newkey, keys);
            jedis.close();
            return s;
        }

        /**
         * 返回给定集合交集的成员,如果其中一个集合为不存在或为空，则返回空Set
         *
         * @param keys
         * @return 交集成员的集合
         **/
        public Set<String> sinter(String... keys) {
            Jedis jedis = getJedis();
            Set<String> set = jedis.sinter(keys);
            jedis.close();
            return set;
        }

        /**
         * 这个命令等于sinter,但返回的不是结果集,而是将结果集存储在新的集合中，如果目标已存在，则覆盖。
         *
         * @param newkey 新结果集的key
         * @param keys   比较的集合
         * @return 新集合中的记录数
         **/
        public long sinterstore(String newkey, String... keys) {
            Jedis jedis = getJedis();
            long s = jedis.sinterstore(newkey, keys);
            jedis.close();
            return s;
        }

        /**
         * 确定一个给定的值是否存在
         *
         * @param key
         * @param member 要判断的值
         * @return 存在返回1，不存在返回0
         **/
        public boolean sismember(String key, String member) {
            // ShardedJedis sjedis = getShardedJedis();
            Jedis sjedis = getJedis();
            boolean s = sjedis.sismember(key, member);
            sjedis.close();
            return s;
        }

        /**
         * 返回集合中的所有成员
         *
         * @param key
         * @return 成员集合
         */
        public Set<String> smembers(String key) {
            // ShardedJedis sjedis = getShardedJedis();
            Jedis sjedis = getJedis();
            Set<String> set = sjedis.smembers(key);
            sjedis.close();
            return set;
        }

        public Set<byte[]> smembers(byte[] key) {
            // ShardedJedis sjedis = getShardedJedis();
            Jedis sjedis = getJedis();
            Set<byte[]> set = sjedis.smembers(key);
            sjedis.close();
            return set;
        }

        /**
         * 将成员从源集合移出放入目标集合 <br/>
         * 如果源集合不存在或不包哈指定成员，不进行任何操作，返回0<br/>
         * 否则该成员从源集合上删除，并添加到目标集合，如果目标集合中成员已存在，则只在源集合进行删除
         *
         * @param srckey 源集合
         * @param dstkey 目标集合
         * @param member 源集合中的成员
         * @return 状态码，1成功，0失败
         */
        public long smove(String srckey, String dstkey, String member) {
            Jedis jedis = getJedis();
            long s = jedis.smove(srckey, dstkey, member);
            jedis.close();
            return s;
        }

        /**
         * 从集合中删除成员
         *
         * @param key
         * @return 被删除的成员
         */
        public String spop(String key) {
            Jedis jedis = getJedis();
            String s = jedis.spop(key);
            jedis.close();
            return s;
        }

        /**
         * 从集合中删除指定成员
         *
         * @param key
         * @param member 要删除的成员
         * @return 状态码，成功返回1，成员不存在返回0
         */
        public long srem(String key, String member) {
            Jedis jedis = getJedis();
            long s = jedis.srem(key, member);
            jedis.close();
            return s;
        }

        /**
         * 合并多个集合并返回合并后的结果，合并后的结果集合并不保存<br/>
         *
         * @param keys
         * @return 合并后的结果集合
         */
        public Set<String> sunion(String... keys) {
            Jedis jedis = getJedis();
            Set<String> set = jedis.sunion(keys);
            jedis.close();
            return set;
        }

        /**
         * 合并多个集合并将合并后的结果集保存在指定的新集合中，如果新集合已经存在则覆盖
         *
         * @param newkey 新集合的key
         * @param keys   要合并的集合
         **/
        public long sunionstore(String newkey, String... keys) {
            Jedis jedis = getJedis();
            long s = jedis.sunionstore(newkey, keys);
            jedis.close();
            return s;
        }
    }

    // *******************************************Hash*******************************************//
    public class Hash {

        /**
         * 从hash中删除指定的存储
         *
         * @param key
         * @param fieid 存储的名字
         * @return 状态码，1成功，0失败
         */
        public long hdel(String key, String fieid) {
            Jedis jedis = getJedis();
            long s = jedis.hdel(key, fieid);
            jedis.close();
            return s;
        }

        public long hdel(String key) {
            Jedis jedis = getJedis();
            long s = jedis.del(key);
            jedis.close();
            return s;
        }

        /**
         * 测试hash中指定的存储是否存在
         *
         * @param key
         * @param fieid 存储的名字
         * @return 1存在，0不存在
         */
        public boolean hexists(String key, String fieid) {
            // ShardedJedis sjedis = getShardedJedis();
            Jedis sjedis = getJedis();
            boolean s = sjedis.hexists(key, fieid);
            sjedis.close();
            return s;
        }

        /**
         * 返回hash中指定存储位置的值
         *
         * @param key
         * @param fieid 存储的名字
         * @return 存储对应的值
         */
        public String hget(String key, String fieid) {
            // ShardedJedis sjedis = getShardedJedis();
            Jedis sjedis = getJedis();
            String s = sjedis.hget(key, fieid);
            sjedis.close();
            return s;
        }

        public byte[] hget(byte[] key, byte[] fieid) {
            // ShardedJedis sjedis = getShardedJedis();
            Jedis sjedis = getJedis();
            byte[] s = sjedis.hget(key, fieid);
            sjedis.close();
            return s;
        }

        /**
         * 以Map的形式返回hash中的存储和值
         *
         * @param key
         * @return Map<Strinig, String>
         */
        public Map<String, String> hgetAll(String key) {
            // ShardedJedis sjedis = getShardedJedis();
            Jedis sjedis = getJedis();
            Map<String, String> map = sjedis.hgetAll(key);
            sjedis.close();
            return map;
        }

        /**
         * 添加一个对应关系
         *
         * @param key
         * @param fieid
         * @param value
         * @return 状态码 1成功，0失败，fieid已存在将更新，也返回0
         **/
        public long hset(String key, String fieid, String value) {
            Jedis jedis = getJedis();
            long s = jedis.hset(key, fieid, value);
            jedis.close();
            return s;
        }

        public long hset(String key, String fieid, byte[] value) {
            Jedis jedis = getJedis();
            long s = jedis.hset(key.getBytes(), fieid.getBytes(), value);
            jedis.close();
            return s;
        }

        /**
         * 添加对应关系，只有在fieid不存在时才执行
         *
         * @param key
         * @param fieid
         * @param value
         * @return 状态码 1成功，0失败fieid已存
         **/
        public long hsetnx(String key, String fieid, String value) {
            Jedis jedis = getJedis();
            long s = jedis.hsetnx(key, fieid, value);
            jedis.close();
            return s;
        }

        /**
         * 获取hash中value的集合
         *
         * @param key
         * @return List<String>
         */
        public List<String> hvals(String key) {
            // ShardedJedis sjedis = getShardedJedis();
            Jedis sjedis = getJedis();
            List<String> list = sjedis.hvals(key);
            sjedis.close();
            return list;
        }

        /**
         * 在指定的存储位置加上指定的数字，存储位置的值必须可转为数字类型
         *
         * @param key
         * @param fieid 存储位置
         * @param value 要增加的值,可以是负数
         * @return 增加指定数字后，存储位置的值
         */
        public long hincrby(String key, String fieid, long value) {
            Jedis jedis = getJedis();
            long s = jedis.hincrBy(key, fieid, value);
            jedis.close();
            return s;
        }

        /**
         * 返回指定hash中的所有存储名字,类似Map中的keySet方法
         *
         * @param key
         * @return Set<String> 存储名称的集合
         */
        public Set<String> hkeys(String key) {
            // ShardedJedis sjedis = getShardedJedis();
            Jedis sjedis = getJedis();
            Set<String> set = sjedis.hkeys(key);
            sjedis.close();
            return set;
        }

        /**
         * 获取hash中存储的个数，类似Map中size方法
         *
         * @param key
         * @return long 存储的个数
         */
        public long hlen(String key) {
            // ShardedJedis sjedis = getShardedJedis();
            Jedis sjedis = getJedis();
            long len = sjedis.hlen(key);
            sjedis.close();
            return len;
        }

        /**
         * 根据多个key，获取对应的value，返回List,如果指定的key不存在,List对应位置为null
         *
         * @param key
         * @param fieids 存储位置
         * @return List<String>
         */
        public List<String> hmget(String key, String... fieids) {
            // ShardedJedis sjedis = getShardedJedis();
            Jedis sjedis = getJedis();
            List<String> list = sjedis.hmget(key, fieids);
            sjedis.close();
            return list;
        }

        public List<byte[]> hmget(byte[] key, byte[]... fieids) {
            // ShardedJedis sjedis = getShardedJedis();
            Jedis sjedis = getJedis();
            List<byte[]> list = sjedis.hmget(key, fieids);
            sjedis.close();
            return list;
        }

        /**
         * 添加对应关系，如果对应关系已存在，则覆盖
         *
         * @param key
         * @param map 对应关系
         * @return 状态，成功返回OK
         */
        public String hmset(String key, Map<String, String> map) {
            Jedis jedis = getJedis();
            String s = jedis.hmset(key, map);
            jedis.close();
            return s;
        }

        /**
         * 添加对应关系，如果对应关系已存在，则覆盖
         *
         * @param key
         * @param map 对应关系
         * @return 状态，成功返回OK
         */
        public String hmset(byte[] key, Map<byte[], byte[]> map) {
            Jedis jedis = getJedis();
            String s = jedis.hmset(key, map);
            jedis.close();
            return s;
        }

    }

    // *******************************************Strings*******************************************//
    public class Strings {

        /**
         * 根据key获取记录
         *
         * @param key
         * @return 值
         */
        public String get(String key) {
            // ShardedJedis sjedis = getShardedJedis();
            Jedis sjedis = getJedis();
            String value = sjedis.get(key);
            sjedis.close();
            return value;
        }

        /**
         * 根据key获取记录
         *
         * @param key
         * @return 值
         */
        public byte[] get(byte[] key) {
            // ShardedJedis sjedis = getShardedJedis();
            Jedis sjedis = getJedis();
            byte[] value = sjedis.get(key);
            sjedis.close();
            return value;
        }

        /**
         * 添加有过期时间的记录
         *
         * @param key
         * @param seconds 过期时间，以秒为单位
         * @param value
         * @return String 操作状态
         */
        public String setEx(String key, int seconds, String value) {
            Jedis jedis = getJedis();
            String str = jedis.setex(key, seconds, value);
            jedis.close();
            return str;
        }

        /**
         * 添加有过期时间的记录
         *
         * @param key
         * @param seconds 过期时间，以秒为单位
         * @param value
         * @return String 操作状态
         */
        public String setEx(byte[] key, int seconds, byte[] value) {
            Jedis jedis = getJedis();
            String str = jedis.setex(key, seconds, value);
            jedis.close();
            return str;
        }

        /**
         * 添加一条记录，仅当给定的key不存在时才插入
         *
         * @param key
         * @param value
         * @return long 状态码，1插入成功且key不存在，0未插入，key存在
         */
        public long setnx(String key, String value) {
            Jedis jedis = getJedis();
            long str = jedis.setnx(key, value);
            jedis.close();
            return str;
        }

        /**
         * 添加记录,如果记录已存在将覆盖原有的value
         *
         * @param key
         * @param value
         * @return 状态码
         */
        public String set(String key, String value) {
            return set(SafeEncoder.encode(key), SafeEncoder.encode(value));
        }

        /**
         * 添加记录,如果记录已存在将覆盖原有的value
         *
         * @param key
         * @param value
         * @return 状态码
         */
        public String set(String key, byte[] value) {
            return set(SafeEncoder.encode(key), value);
        }

        /**
         * 添加记录,如果记录已存在将覆盖原有的value
         *
         * @param key
         * @param value
         * @return 状态码
         */
        public String set(byte[] key, byte[] value) {
            Jedis jedis = getJedis();
            String status = jedis.set(key, value);
            jedis.close();
            return status;
        }

        /**
         * 从指定位置开始插入数据，插入的数据会覆盖指定位置以后的数据<br/>
         * 例:String str1="123456789";<br/>
         * 对str1操作后setRange(key,4,0000)，str1="123400009";
         *
         * @param key
         * @param offset
         * @param value
         * @return long value的长度
         */
        public long setRange(String key, long offset, String value) {
            Jedis jedis = getJedis();
            long len = jedis.setrange(key, offset, value);
            jedis.close();
            return len;
        }

        /**
         * 在指定的key中追加value
         *
         * @param key
         * @param value
         * @return long 追加后value的长度
         **/
        public long append(String key, String value) {
            Jedis jedis = getJedis();
            long len = jedis.append(key, value);
            jedis.close();
            return len;
        }

        /**
         * 将key对应的value减去指定的值，只有value可以转为数字时该方法才可用
         *
         * @param key
         * @param number 要减去的值
         * @return long 减指定值后的值
         */
        public long decrBy(String key, long number) {
            Jedis jedis = getJedis();
            long len = jedis.decrBy(key, number);
            jedis.close();
            return len;
        }

        /**
         * <b>可以作为获取唯一id的方法</b><br/>
         * 将key对应的value加上指定的值，只有value可以转为数字时该方法才可用
         *
         * @param key
         * @param number 要减去的值
         * @return long 相加后的值
         */
        public long incrBy(String key, long number) {
            Jedis jedis = getJedis();
            long len = jedis.incrBy(key, number);
            jedis.close();
            return len;
        }

        /**
         * 对指定key对应的value进行截取
         *
         * @param key
         * @param startOffset 开始位置(包含)
         * @param endOffset   结束位置(包含)
         * @return String 截取的值
         */
        public String getrange(String key, long startOffset, long endOffset) {
            // ShardedJedis sjedis = getShardedJedis();
            Jedis sjedis = getJedis();
            String value = sjedis.getrange(key, startOffset, endOffset);
            sjedis.close();
            return value;
        }

        /**
         * 获取并设置指定key对应的value<br/>
         * 如果key存在返回之前的value,否则返回null
         *
         * @param key
         * @param value
         * @return String 原始value或null
         */
        public String getSet(String key, String value) {
            Jedis jedis = getJedis();
            String str = jedis.getSet(key, value);
            jedis.close();
            return str;
        }

        /**
         * 批量获取记录,如果指定的key不存在返回List的对应位置将是null
         *
         * @param keys
         * @return List<String> 值得集合
         */
        public List<String> mget(String... keys) {
            Jedis jedis = getJedis();
            List<String> str = jedis.mget(keys);
            jedis.close();
            return str;
        }

        /**
         * 批量存储记录
         *
         * @param keysvalues 例:keysvalues="key1","value1","key2","value2";
         * @return String 状态码
         */
        public String mset(String... keysvalues) {
            Jedis jedis = getJedis();
            String str = jedis.mset(keysvalues);
            jedis.close();
            return str;
        }

        /**
         * 获取key对应的值的长度
         *
         * @param key
         * @return value值得长度
         */
        public long strlen(String key) {
            Jedis jedis = getJedis();
            long len = jedis.strlen(key);
            jedis.close();
            return len;
        }
    }

    // *******************************************Lists*******************************************//
    public class Lists {

        /**
         * List长度
         *
         * @param key
         * @return 长度
         */
        public long llen(String key) {
            return llen(SafeEncoder.encode(key));
        }

        /**
         * List长度
         *
         * @param key
         * @return 长度
         */
        public long llen(byte[] key) {
            // ShardedJedis sjedis = getShardedJedis();
            Jedis sjedis = getJedis();
            long count = sjedis.llen(key);
            sjedis.close();
            return count;
        }

        /**
         * 覆盖操作,将覆盖List中指定位置的值
         *
         * @param key
         * @param index 位置
         * @param value 值
         * @return 状态码
         */
        public String lset(byte[] key, int index, byte[] value) {
            Jedis jedis = getJedis();
            String status = jedis.lset(key, index, value);
            jedis.close();
            return status;
        }

        /**
         * 覆盖操作,将覆盖List中指定位置的值
         *
         * @param @param index 位置
         * @param value  值
         * @return 状态码
         */
        public String lset(String key, int index, String value) {
            return lset(SafeEncoder.encode(key), index,
                    SafeEncoder.encode(value));
        }

        /**
         * 在value的相对位置插入记录
         *
         * @param @param 前面插入或后面插入
         * @param pivot  相对位置的内容
         * @param value  插入的内容
         * @return 记录总数
         */
        public long linsert(String key, BinaryClient.LIST_POSITION where, String pivot,
                            String value) {
            return linsert(SafeEncoder.encode(key), where,
                    SafeEncoder.encode(pivot), SafeEncoder.encode(value));
        }

        /**
         * 在指定位置插入记录
         *
         * @param key
         * @param where 前面插入或后面插入
         * @param pivot 相对位置的内容
         * @param value 插入的内容
         * @return 记录总数
         */
        public long linsert(byte[] key, BinaryClient.LIST_POSITION where, byte[] pivot,
                            byte[] value) {
            Jedis jedis = getJedis();
            long count = jedis.linsert(key, where, pivot, value);
            jedis.close();
            return count;
        }

        /**
         * 获取List中指定位置的值
         *
         * @param key
         * @param index 位置
         * @return 值
         **/
        public String lindex(String key, int index) {
            return SafeEncoder.encode(lindex(SafeEncoder.encode(key), index));
        }

        /**
         * 获取List中指定位置的值
         *
         * @param key
         * @param index 位置
         * @return 值
         **/
        public byte[] lindex(byte[] key, int index) {
            // ShardedJedis sjedis = getShardedJedis();
            Jedis sjedis = getJedis();
            byte[] value = sjedis.lindex(key, index);
            sjedis.close();
            return value;
        }

        /**
         * 将List中的第一条记录移出List
         *
         * @param key
         * @return 移出的记录
         */
        public String lpop(String key) {
            return SafeEncoder.encode(lpop(SafeEncoder.encode(key)));
        }

        /**
         * 将List中的第一条记录移出List
         *
         * @param key
         * @return 移出的记录
         */
        public byte[] lpop(byte[] key) {
            Jedis jedis = getJedis();
            byte[] value = jedis.lpop(key);
            jedis.close();
            return value;
        }

        /**
         * 将List中最后第一条记录移出List
         *
         * @param key
         * @return 移出的记录
         */
        public String rpop(String key) {
            Jedis jedis = getJedis();
            String value = jedis.rpop(key);
            jedis.close();
            return value;
        }

        /**
         * 向List尾部追加记录
         *
         * @param key
         * @param value
         * @return 记录总数
         */
        public long lpush(String key, String value) {
            return lpush(SafeEncoder.encode(key), SafeEncoder.encode(value));
        }

        /**
         * 向List头部追加记录
         *
         * @param key
         * @param value
         * @return 记录总数
         */
        public long rpush(String key, String value) {
            Jedis jedis = getJedis();
            long count = jedis.rpush(key, value);
            jedis.close();
            return count;
        }

        /**
         * 向List头部追加记录
         *
         * @param key
         * @param value
         * @return 记录总数
         */
        public long rpush(byte[] key, byte[] value) {
            Jedis jedis = getJedis();
            long count = jedis.rpush(key, value);
            jedis.close();
            return count;
        }

        /**
         * 向List中追加记录
         *
         * @param key
         * @param value
         * @return 记录总数
         */
        public long lpush(byte[] key, byte[] value) {
            Jedis jedis = getJedis();
            long count = jedis.lpush(key, value);
            jedis.close();
            return count;
        }

        /**
         * 获取指定范围的记录，可以做为分页使用
         *
         * @param key
         * @param start
         * @param end
         * @return List
         */
        public List<String> lrange(String key, long start, long end) {
            // ShardedJedis sjedis = getShardedJedis();
            Jedis sjedis = getJedis();
            List<String> list = sjedis.lrange(key, start, end);
            sjedis.close();
            return list;
        }

        /**
         * 获取指定范围的记录，可以做为分页使用
         *
         * @param key
         * @param start
         * @param end   如果为负数，则尾部开始计算
         * @return List
         */
        public List<byte[]> lrange(byte[] key, int start, int end) {
            // ShardedJedis sjedis = getShardedJedis();
            Jedis sjedis = getJedis();
            List<byte[]> list = sjedis.lrange(key, start, end);
            sjedis.close();
            return list;
        }

        /**
         * 删除List中c条记录，被删除的记录值为value
         *
         * @param key
         * @param c     要删除的数量，如果为负数则从List的尾部检查并删除符合的记录
         * @param value 要匹配的值
         * @return 删除后的List中的记录数
         */
        public long lrem(byte[] key, int c, byte[] value) {
            Jedis jedis = getJedis();
            long count = jedis.lrem(key, c, value);
            jedis.close();
            return count;
        }

        /**
         * 删除List中c条记录，被删除的记录值为value
         *
         * @param key
         * @param c     要删除的数量，如果为负数则从List的尾部检查并删除符合的记录
         * @param value 要匹配的值
         * @return 删除后的List中的记录数
         */
        public long lrem(String key, int c, String value) {
            return lrem(SafeEncoder.encode(key), c, SafeEncoder.encode(value));
        }

        /**
         * 算是删除吧，只保留start与end之间的记录
         *
         * @param key
         * @param start 记录的开始位置(0表示第一条记录)
         * @param end   记录的结束位置（如果为-1则表示最后一个，-2，-3以此类推）
         * @return 执行状态码
         */
        public String ltrim(byte[] key, int start, int end) {
            Jedis jedis = getJedis();
            String str = jedis.ltrim(key, start, end);
            jedis.close();
            return str;
        }

        /**
         * 算是删除吧，只保留start与end之间的记录
         *
         * @param key
         * @param start 记录的开始位置(0表示第一条记录)
         * @param end   记录的结束位置（如果为-1则表示最后一个，-2，-3以此类推）
         * @return 执行状态码
         */
        public String ltrim(String key, int start, int end) {
            return ltrim(SafeEncoder.encode(key), start, end);
        }
    }

}
```