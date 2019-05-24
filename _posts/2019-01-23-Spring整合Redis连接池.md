---
layout: post
title:  "Spring整合Redis连接池"
date:   2019-01-23 18:00:01 +0800
categories: Spring
tag: Redis
---

* content
{:toc}



架构整合				
======


Spring Redis
------



1.在pom.xml中添加redis客户端jedis依赖
```bash
<dependency>
<groupId>redis.clients</groupId>
<artifactId>jedis</artifactId>
<version>2.6.0</version>
</dependency>
```

2.redis linux下安装
```bash
1.下载、上传、解压
redis-4.0.2.tar.gz
2.安装C语言编译环境
yum install -y gcc-c++
3.编译安装
编译：进入Redis解压目录执行make命令
安装：make install
4.创建Redis专属目录
mkdir /usr/local/redis
5.将redis.conf复制到专属目录并修改
35 # By default Redis does not run as a daemon. Use 'yes' if you need it.
36 # Note that Redis will write a pid file in /var/run/redis.pid when daemonized.
37 daemonize yes
6.启动Redis
/usr/local/bin/redis-server /usr/local/redis/redis.conf
查看6379端口监听情况
临时指定端口号的启动方式如下：
/usr/local/bin/redis-server /usr/local/redis/redis.conf --port 7000
如果不指定配置文件位置则按默认配置启动。
7.压力测试
/usr/local/bin/redis-benchmark
Redis每秒80000次写操作，110000次读操作。
8.通过Redis客户端登录Redis服务器
[root@right bin]# /usr/local/bin/redis-cli [-p 6379]
127.0.0.1:6379> ping
PONG
127.0.0.1:6379> exit
[root@right bin]#
9.停止Redis服务器
按默认6379端口号停止：/usr/local/bin/redis-cli shutdown
停止指定服务器：/usr/local/bin/redis-cli -h 127.0.0.1 -p 6379 shutdown
在客户端登录状态下停止：127.0.0.1:6379> shutdown
```

3.新建spring配置文件 spring-redis.xml
```bash
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xmlns:context="http://www.springframework.org/schema/context" xsi:schemaLocation="
		http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-4.1.xsd
		http://www.springframework.org/schema/context  http://www.springframework.org/schema/context/spring-context-4.1.xsd"
	default-lazy-init="true">

	<description>Jedis Configuration</description>

    <!-- 加载配置属性文件 -->
	<context:property-placeholder ignore-unresolvable="true" location="classpath:system.properties" />
	
    <!-- jedis pool配置 -->
    <bean id="jedisPoolConfig" class="redis.clients.jedis.JedisPoolConfig">
        <property name="maxTotal" value="${redis.maxTotal}" />
        <property name="maxIdle" value="${redis.maxIdle}" />
        <property name="maxWaitMillis" value="${redis.maxWaitMillis}" />
        <property name="testOnBorrow" value="${redis.testOnBorrow}" />
    </bean>

    <!-- spring data redis -->
    <bean id="marketJedisConnectionFactory"
        class="org.springframework.data.redis.connection.jedis.JedisConnectionFactory">
        <property name="usePool" value="true"></property>
        <property name="hostName" value="${market.redis.host}" />
        <property name="port" value="${market.redis.port}" />
        <property name="password" value="${market.redis.pass}" />
        <property name="timeout" value="${market.redis.timeout}" />
        <property name="database" value="${market.redis.default.db}"></property>
        <constructor-arg index="0" ref="jedisPoolConfig" />
    </bean>

    <bean id="marketRedisTemplate" class="org.springframework.data.redis.core.RedisTemplate">
        <!-- <property name="defaultSerializer"> -->
        <!-- <bean -->
        <!-- class="org.springframework.data.redis.serializer.Jackson2JsonRedisSerializer"> -->
        <!-- <constructor-arg index="0" -->
        <!-- value="java.util.Map"></constructor-arg> -->
        <!-- </bean> -->
        <!-- </property> -->
        <property name="keySerializer">
            <bean
                class="org.springframework.data.redis.serializer.StringRedisSerializer" />
        </property>
        <property name="hashKeySerializer">
            <bean
                class="org.springframework.data.redis.serializer.StringRedisSerializer" />
        </property>
  <!--       <property name="hashValueSerializer">
             <bean
                class="org.springframework.data.redis.serializer.Jackson2JsonRedisSerializer">
                <constructor-arg index="0" value="java.util.Map"></constructor-arg>
            </bean> 
        </property>-->
        <property name="connectionFactory" ref="marketJedisConnectionFactory" />
    </bean>
	
</beans>
```

4.system.properties中配置
```bash
#redis settings
redis.keyPrefix=xxx
#Fri Jan 16 17:12:50 CST 2015
redis.timeout=90000
redis.testOnBorrow=true
redis.default.db=0
redis.maxTotal=100
redis.maxIdle=50
redis.pass=xxx
redis.host=0.0.0.0
redis.maxWaitMillis=1000
redis.port=6379

market.redis.timeout=90000
market.redis.testOnBorrow=true
market.redis.default.db=0
market.redis.maxTotal=100
market.redis.maxIdle=50
market.redis.pass=xxx
market.redis.host=0.0.0.0
market.redis.maxWaitMillis=1000
market.redis.port=6379
```

5.配置说明
```bash
#ip地址
redis.hostName=127.0.0.1
#端口号
redis.port=6379
#如果有密码
redis.password=
#客户端超时时间单位是毫秒 默认是2000
redis.timeout=10000
#最大空闲数
redis.maxIdle=300
#连接池的最大数据库连接数。设为0表示无限制,如果是jedis 2.4以后用redis.maxTotal
#redis.maxActive=600
#控制一个pool可分配多少个jedis实例,用来替换上面的redis.maxActive,如果是jedis 2.4以后用该属性
redis.maxTotal=1000
#最大建立连接等待时间。如果超过此时间将接到异常。设为-1表示无限制。
redis.maxWaitMillis=1000
#连接的最小空闲时间 默认1800000毫秒(30分钟)
redis.minEvictableIdleTimeMillis=300000
#每次释放连接的最大数目,默认3
redis.numTestsPerEvictionRun=1024
#逐出扫描的时间间隔(毫秒) 如果为负数,则不运行逐出线程, 默认-1
redis.timeBetweenEvictionRunsMillis=30000
#是否在从池中取出连接前进行检验,如果检验失败,则从池中去除连接并尝试取出另一个,数据量大的时候建议关闭
redis.testOnBorrow=true
#在空闲时检查有效性, 默认false
redis.testWhileIdle=true
```

6.编写redis通用工具类
```bash
import java.util.List;
import java.util.concurrent.TimeUnit;

import org.springframework.data.redis.core.RedisTemplate;

import com.jniu.bas.spring.SpringContextHolder;

public class RedisUtils {
	private static RedisUtils redisUtils;
	
	@SuppressWarnings("rawtypes")
	private RedisTemplate marketRedisTemplate;
	
	@SuppressWarnings("rawtypes")
	private RedisUtils(){
		marketRedisTemplate =   (RedisTemplate) SpringContextHolder.getBean("marketRedisTemplate");
	}
	
	public static RedisUtils getInstance() {
		if (redisUtils == null) {
			synchronized (RedisUtils.class) {
				if (redisUtils == null) {
					redisUtils = new RedisUtils();
					return redisUtils;
				}else{
					return redisUtils;
				}
			}
		}else{
			return redisUtils;
		}
	}
	
	/**
	 * 获取value  适用于配置文件缓存
	 * @param key
	 * @return
	 */
	public String getValue(String key){
		return  (String)marketRedisTemplate.opsForValue().get(key);
	}
	
	/**
	 * 设置带有过期时间的key
	 * @param key
	 * @param value
	 * @param time
	 */
	@SuppressWarnings("unchecked")
	public void setValue(String key,String value,long time){
		  marketRedisTemplate.opsForValue().set(key, value);
		  marketRedisTemplate.expire(key,time,TimeUnit.SECONDS);
	}
	
	/**
	 * 设置key value  适用于配置文件缓存
	 * @param key
	 * @return
	 */
	@SuppressWarnings("unchecked")
	public void setValue(String key,String value){
		  marketRedisTemplate.opsForValue().set(key, value);
	}
	/**
	 * 删除KEY 
	 * @param key
	 * @return
	 */
	@SuppressWarnings("unchecked")
	public void deleteKey(String key){
		marketRedisTemplate.delete(key);
	}
	
	/**
	 * 根据表名与主键ID获取实体
	 * @param key
	 * @param id
	 * @return
	 */
	@SuppressWarnings("unchecked")
	public Object getObject(String key,String id){
		return marketRedisTemplate.opsForHash().get(key, id);
	}
	/**
	 * 设置表名与主键ID 
	 * @param key
	 * @param id
	 * @return
	 */
	@SuppressWarnings("unchecked")
	public void setObject(String key,String id,Object obj){
		 marketRedisTemplate.opsForHash().put(key, id, obj);
	}
	
	/**
	 * 根据ID 删除
	 * @param key
	 * @param id
	 * @return
	 */
	@SuppressWarnings("unchecked")
	public void deleteObjectByid(String key,String id){
		 marketRedisTemplate.opsForHash().delete(key, id);
	}
	/**
	 * 根据key 删除
	 * @param key
	 * @param id
	 * @return
	 */
	@SuppressWarnings("unchecked")
	public void deleteObjectByKey(String key){
		 marketRedisTemplate.delete(key);
	}
	/**
	 * 单条set list 数据
	 * @param key
	 * @param obj
	 * @return
	 */
	@SuppressWarnings("unchecked")
	public long  addListByKey(String key,Object obj){
		return marketRedisTemplate.opsForList().rightPush(key, obj);
	}
	
	/**
	 * 获取list的大小
	 * @param key
	 * @return
	 */
	@SuppressWarnings("unchecked")
	public long getListSize(String key){
		return marketRedisTemplate.opsForList().size(key);
	}
	
	/**
	 * 根据index下标 获取数据
	 * @param key
	 * @param index
	 * @return
	 */
	@SuppressWarnings("unchecked")
	public Object getListDataByIndex(String key,long index){
		return marketRedisTemplate.opsForList().index(key, index);  
	}
	/**
	 * 从list里移除
	 * @param key
	 * @param index
	 * @param value
	 */
	@SuppressWarnings("unchecked")
	public void removeListByKeyAndIndex(String key,long index,Object value){
		marketRedisTemplate.opsForList().remove(key, index, value);
	}
	
	
}

```

7.redis使用范例
```bash
RedisUtils resdis = RedisUtils.getInstance();
//存Redis
String classJson = "";//实体类json
resdis.addListByKey("key",classJson);***存放json***
//移除Redis
String key ="key";
long length = resdis.getListSize(key);
for (int i=0;i<length;i++) {
String jsonStr = (String)resdis.getListDataByIndex(key, i);
	//json转对应实体
Class class = JsonUtil.fromJson(jsonStr, Class.class);
if (Class.getId().equals("当前业务id一致")) {
	resdis.removeListByKeyAndIndex(key, i, jsonStr);***移除也传入json***
	break;
	}
}
坑记录：移除不成功
***
传入json，移除也必须是json
***
```


（完）
======

