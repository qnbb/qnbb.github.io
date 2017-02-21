---
layout: post
title: Redis封装Map，JSON实现String、Map相互转换问题
categories: [Redis, JSON]
description: Redis封装Map，JSON实现String、Map相互转换问题
keywords: Redis, JSON
---


Redis 作为大多数公司的缓存机制和分布式锁服务框架，基于Spring的org.springframework.data.redis 更是被广泛使用。

#### 事件起因
将Map对象封装进Redis，习惯性使用 `RedisService<String, String>`的方式，代码如下：
```
@Resource
private RedisService<String, String> redisService;
@Override
public ResultDto<Map<Integer, MerchantAgreementDto>> getMerchantAgreement(Long masterMerchantId) {
...
try {
//1.1 取数据时，先尝试取出redis (String)value
String jsonResult = redisService.get(key);
 //1.2 将String类型的value专为Map
 merchantAgreementDtos = JSON.parseObject(jsonResult, HashMap.class);
...
//2.1 先将map变为String
String jsonString = JSON.toJSONString(merchantAgreementDtos);
//2.2 String封装进redis
redisService.set(MERCHANT_AGREEMENT_KEY + masterMerchantId,jsonString, TIMEOUT);
...
```
#### 问题定位
redis 序列化，Map和String互转之后，并没有成功转回Map，而是转为 JSONObject。其它业务方调用改方法时，取出的数据，不是Map类型，而是JSONObject类型，不符合业务要求。

首先查看下一开使用的 Redis 配置，

```
 <bean id="jedisPoolConfig" class="redis.clients.jedis.JedisPoolConfig"
p:maxIdle="${redis.pool.maxIdle}" p:maxTotal="${redis.pool.maxTotal}"
p:maxWaitMillis="${redis.pool.maxWait}" p:testOnBorrow="${redis.pool.testOnBorrow}" />
<bean id="jedisConnFactory"
class="org.springframework.data.redis.connection.jedis.JedisConnectionFactory"
p:hostName="${redis.hostname}" p:port="${redis.port}" p:usePool="true"
p:poolConfig-ref="jedisPoolConfig" />
<bean id="redisTemplate" class="org.springframework.data.redis.core.RedisTemplate"
p:connectionFactory-ref="jedisConnFactory">
<property name="keySerializer">
<bean
class="org.springframework.data.redis.serializer.StringRedisSerializer" />
</property>
<property name="valueSerializer">
<bean
class="org.springframework.data.redis.serializer.StringRedisSerializer" />
</property>
<property name="hashKeySerializer">
<bean
class="org.springframework.data.redis.serializer.StringRedisSerializer" />
</property>
<property name="hashValueSerializer">
<bean
class="org.springframework.data.redis.serializer.StringRedisSerializer" />
</property>
</bean>
```
RedisTemplate中需要声明4种serializer，默认为“JdkSerializationRedisSerializer”：
1. keySerializer ：对于普通K-V操作时，key采取的序列化策略
2. valueSerializer：value采取的序列化策略
3. hashKeySerializer： 在hash数据结构中，hash-key的序列化策略
4. hashValueSerializer：hash-value的序列化策略

其中JdkSerializationRedisSerializer和StringRedisSerializer是最基础的序列化策略，其中“JacksonJsonRedisSerializer”与“OxmSerializer”都是基于stirng存储，因此它们是较为“高级”的序列化(最终还是使用string解析以及构建java对象)。

可以看出 valueSerializer 的序列化方式设置的是 StringRedisSerializer。在业务代码中，存取Redis缓存的代码如下：
```
//1.1 取出redis (String)value
String jsonResult = redisService.get(key);
 //1.2 将String类型的value专为Map
 merchantAgreementDtos = JSON.parseObject(jsonResult, HashMap.class);
...
//2.1 先将map变为String
String jsonString = JSON.toJSONString(merchantAgreementDtos);
//2.2 String封装进redis
redisService.set(MERCHANT_AGREEMENT_KEY + masterMerchantId,jsonString, TIMEOUT);
```
后台dubbo调用接口，显示结果如下：
```
invoke com.baidu.bainuo.finance.settlement.agent.MerchantMarketingAgent. getMerchantAgreement (1416362209360176)
{"result":{"2201":{"customPercentage":100000,"createTime":"2016-01-17 16:42:07","createUser":1416362209360176,"masterMerchantId":1416362209360176,"updateTime":"2016-01-17 16:42:07","equivalentType":2201,"status":1,"platformPercentage":0,"updateUser":1416362209360176},"2501":{"customPercentage":100000,"createTime":"2016-01-21 16:10:24","createUser":1416362209360176,"masterMerchantId":1416362209360176,"updateTime":"2016-01-21 16:10:24","equivalentType":2501,"status":1,"platformPercentage":0,"updateUser":1416362209360176}},"code":0,"msg":null}
```
总结：Redis通过String去封装<Key, Value>，在业务代码中String 转换回Map的时候，没有成功转换回 Map，而是转为 JSONObject。

#### 问题解决
现有两种解决方案，一种是 在业务代码中，String 转 Map 添加泛型 <T>，另外一种则是，直接Redis序列化 Map 类型。

- String 转 Map 添加泛型 < T >

```
merchantAgreementDtos = JSON.parseObject(jsonResult, HashMap.class);
```
改成
```
merchantAgreementDtos = JSON.parseObject(jsonResult, , new TypeReference<Map<Integer, MerchantAgreementDto>>() {});
```

- Redis序列化 Map 类型
在业务代码中，
```
  @Resource
  private RedisService<String, String> redisService;
```
Redis 作如下修改，valueSerializer 配置成 KryoSerializer，或者这部分不作配置，使用默认的 JdkSerializationRedisSerializer。

```
  @Resource
  private RedisService<String, Map<Integer, MerchantAgreementDto>> redisService;
```
```
 <!-- redis配置 -->
  <bean id="poolConfig" class="redis.clients.jedis.JedisPoolConfig">
  <property name="maxIdle" value="${redis.maxIdle}" />
  <property name="maxTotal" value="${redis.maxTotal}" />
  <property name="maxWaitMillis" value="${redis.maxWait}" />
  <property name="testOnBorrow" value="${redis.testOnBorrow}" />
  </bean>

  <bean id="connectionFactory" class="org.springframework.data.redis.connection.jedis.JedisConnectionFactory">
  <property name="hostName" value="${redis.host}"/>
  <property name="port" value="${redis.port}"/>
  <property name="poolConfig" ref="poolConfig"/>
  </bean>

  <bean id="redisTemplate" class="org.springframework.data.redis.core.RedisTemplate">
  <property name="connectionFactory" ref="connectionFactory"/>
  <property name="keySerializer">
  <bean class="org.springframework.data.redis.serializer.StringRedisSerializer" />
  </property>
  <property name="valueSerializer">
  <bean class="com.baidu.bainuo.common.cache.serializer.KryoSerializer" />
  </property>
  </bean>
```
KryoSerializer的序列化将比 默认的 JdkSerializationRedisSerializer快很多，代码如下：
```
import com.esotericsoftware.kryo.Kryo;
import com.esotericsoftware.kryo.io.Input;
import com.esotericsoftware.kryo.io.Output;
import org.springframework.data.redis.serializer.RedisSerializer;
import org.springframework.data.redis.serializer.SerializationException;

/**
 * 利用Kryo序列化对象
 * Created by qiaodi on 16/8/25.
 */
public class KryoSerializer<T> implements RedisSerializer<T> {

  @Override
  public byte[] serialize(Object o) throws SerializationException {
  Kryo kryo = new Kryo();
  Output output = new Output(1, 4096);
  kryo.writeClassAndObject(output, o);
  byte[] bytes = output.getBuffer();
  output.flush();
  output.close();
  return bytes;
  }

  @Override
  public T deserialize(byte[] bytes) throws SerializationException {
  Kryo kryo = new Kryo();
  Input input = new Input(bytes);
  T object = (T)kryo.readClassAndObject(input);
  input.close();
  return object;
  }
}
```
在 pom.xml 添加以下配置：
```
  <dependency>
  <groupId>com.esotericsoftware</groupId>
  <artifactId>kryo</artifactId>
  <version>4.0.0</version>
  </dependency>
```
这种方式，可以直接序列化 Map 类型，dubbo 测试接口 效果如下:
```
dubbo>{"result":{"2201":{"customPercentage":100000,"createTime":"2016-01-17 16:42:07","createUser":1416362209360176,"masterMerchantId":1416362209360176,"updateTime":"2016-01-17 16:42:07","equivalentType":2201,"status":1,"platformPercentage":0,"updateUser":1416362209360176},"2501":{"customPercentage":100000,"createTime":"2016-01-21 16:10:24","createUser":1416362209360176,"masterMerchantId":1416362209360176,"updateTime":"2016-01-21 16:10:24","equivalentType":2501,"status":1,"platformPercentage":0,"updateUser":1416362209360176}},"code":0,"msg":null}{"result":{"2201":{"customPercentage":100000,"createTime":"2016-01-17 16:42:07","createUser":1416362209360176,"masterMerchantId":1416362209360176,"updateTime":"2016-01-17 16:42:07","equivalentType":2201,"status":1,"platformPercentage":0,"updateUser":1416362209360176},"2501":{"customPercentage":100000,"createTime":"2016-01-21 16:10:24","createUser":1416362209360176,"masterMerchantId":1416362209360176,"updateTime":"2016-01-21 16:10:24","equivalentType":2501,"status":1,"platformPercentage":0,"updateUser":1416362209360176}},"code":0,"msg":null}
elapsed: 1296 ms.Unsupported command: {"result":{"2201":{"customPercentage":100000,"createTime":"2016-01-17
```
#### 问题总结
1. Redis 序列化Map类型，可以通过默认的 JdkSerializationRedisSerializer，也可以用速度更快的 KryoSerializer
2. 利用FastJson 的泛型特性 做String Map之间相互转换，可参考 [FastJson 泛型转换踩坑](http://blog.csdn.net/ykdsg/article/details/50432494)。