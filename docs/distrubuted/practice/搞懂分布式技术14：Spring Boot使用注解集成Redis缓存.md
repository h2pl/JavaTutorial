# Table of Contents

* [Redis配置](#redis配置)
* [连接超时时间（毫秒）](#连接超时时间（毫秒）)
    * [一、创建 Caching 配置类](#一、创建-caching-配置类)
    * [二、创建需要缓存数据的类](#二、创建需要缓存数据的类)
    * [三、测试方法](#三、测试方法)


本文转载自 linkedkeeper.com

Spring Boot 熟悉后，集成一个外部扩展是一件很容易的事，集成Redis也很简单，看下面步骤配置：

一、添加pom依赖

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-redis</artifactId>
        </dependency>

二、创建 RedisClient.java
注意该类存放的package

    package org.springframework.data.redis.connection.jedis;
    
    import java.io.ByteArrayInputStream;
    import java.io.ByteArrayOutputStream;
    import java.io.IOException;
    import java.io.ObjectInputStream;
    import java.io.ObjectOutputStream;
    import java.io.UnsupportedEncodingException;
    
    import org.apache.commons.lang3.StringUtils;
    import org.slf4j.Logger;
    import org.slf4j.LoggerFactory;
    
    import redis.clients.jedis.Jedis;
    import redis.clients.jedis.Protocol;
    import redis.clients.jedis.exceptions.JedisException;
    
    /**
     * 工具类 RedisClient
     * 因为本类中获取JedisPool调用的是JedisConnectionFactory中protected修饰的方法fetchJedisConnector()
     * 所以该类需要与JedisConnectionFactory在同一个package中
     *
     * @author 单红宇(CSDN CATOOP)
     * @create 2017年4月9日
     */
    public class RedisClient {
    
        private static Logger logger = LoggerFactory.getLogger(RedisClient.class);
    
        private JedisConnectionFactory factory;
    
        public RedisClient(JedisConnectionFactory factory) {
            super();
            this.factory = factory;
        }
    
        /**
         * put操作（存储序列化对象）+ 生效时间
         * 
         * @param key
         * @param value
         * @return
         */
        public void putObject(final String key, final Object value, final int cacheSeconds) {
            if (StringUtils.isNotBlank(key)) {
                redisTemplete(key, new RedisExecute<Object>() {
                    @Override
                    public Object doInvoker(Jedis jedis) {
                        try {
                            jedis.setex(key.getBytes(Protocol.CHARSET), cacheSeconds, serialize(value));
                        } catch (UnsupportedEncodingException e) {
                        }
    
                        return null;
                    }
                });
            }
        }
    
        /**
         * get操作（获取序列化对象）
         * 
         * @param key
         * @return
         */
        public Object getObject(final String key) {
            return redisTemplete(key, new RedisExecute<Object>() {
                @Override
                public Object doInvoker(Jedis jedis) {
                    try {
                        byte[] byteKey = key.getBytes(Protocol.CHARSET);
                        byte[] byteValue = jedis.get(byteKey);
                        if (byteValue != null) {
                            return deserialize(byteValue);
                        }
                    } catch (UnsupportedEncodingException e) {
                        return null;
                    }
                    return null;
                }
            });
        }
    
        /**
         * setex操作
         * 
         * @param key
         *            键
         * @param value
         *            值
         * @param cacheSeconds
         *            超时时间，0为不超时
         * @return
         */
        public String set(final String key, final String value, final int cacheSeconds) {
            return redisTemplete(key, new RedisExecute<String>() {
                @Override
                public String doInvoker(Jedis jedis) {
                    if (cacheSeconds == 0) {
                        return jedis.set(key, value);
                    }
                    return jedis.setex(key, cacheSeconds, value);
                }
            });
        }
    
        /**
         * get操作
         * 
         * @param key
         *            键
         * @return 值
         */
        public String get(final String key) {
            return redisTemplete(key, new RedisExecute<String>() {
                @Override
                public String doInvoker(Jedis jedis) {
                    String value = jedis.get(key);
                    return StringUtils.isNotBlank(value) && !"nil".equalsIgnoreCase(value) ? value : null;
                }
            });
        }
    
        /**
         * del操作
         * 
         * @param key
         *            键
         * @return
         */
        public long del(final String key) {
            return redisTemplete(key, new RedisExecute<Long>() {
                @Override
                public Long doInvoker(Jedis jedis) {
                    return jedis.del(key);
                }
            });
        }
    
        /**
         * 获取资源
         * 
         * @return
         * @throws JedisException
         */
        public Jedis getResource() throws JedisException {
            Jedis jedis = null;
            try {
                jedis = factory.fetchJedisConnector();
            } catch (JedisException e) {
                logger.error("getResource.", e);
                returnBrokenResource(jedis);
                throw e;
            }
            return jedis;
        }
    
        /**
         * 获取资源
         * 
         * @return
         * @throws JedisException
         */
        public Jedis getJedis() throws JedisException {
            return getResource();
        }
    
        /**
         * 归还资源
         * 
         * @param jedis
         * @param isBroken
         */
        public void returnBrokenResource(Jedis jedis) {
            if (jedis != null) {
                jedis.close();
            }
        }
    
        /**
         * 释放资源
         * 
         * @param jedis
         * @param isBroken
         */
        public void returnResource(Jedis jedis) {
            if (jedis != null) {
                jedis.close();
            }
        }
    
        /**
         * 操作jedis客户端模板
         * 
         * @param key
         * @param execute
         * @return
         */
        public <R> R redisTemplete(String key, RedisExecute<R> execute) {
            Jedis jedis = null;
            try {
                jedis = getResource();
                if (jedis == null) {
                    return null;
                }
    
                return execute.doInvoker(jedis);
            } catch (Exception e) {
                logger.error("operator redis api fail,{}", key, e);
            } finally {
                returnResource(jedis);
            }
            return null;
        }
    
        /**
         * 功能简述: 对实体Bean进行序列化操作.
         * 
         * @param source
         *            待转换的实体
         * @return 转换之后的字节数组
         * @throws Exception
         */
        public static byte[] serialize(Object source) {
            ByteArrayOutputStream byteOut = null;
            ObjectOutputStream ObjOut = null;
            try {
                byteOut = new ByteArrayOutputStream();
                ObjOut = new ObjectOutputStream(byteOut);
                ObjOut.writeObject(source);
                ObjOut.flush();
            } catch (IOException e) {
                e.printStackTrace();
            } finally {
                try {
                    if (null != ObjOut) {
                        ObjOut.close();
                    }
                } catch (IOException e) {
                    ObjOut = null;
                }
            }
            return byteOut.toByteArray();
        }
    
        /**
         * 功能简述: 将字节数组反序列化为实体Bean.
         * 
         * @param source
         *            需要进行反序列化的字节数组
         * @return 反序列化后的实体Bean
         * @throws Exception
         */
        public static Object deserialize(byte[] source) {
            ObjectInputStream ObjIn = null;
            Object retVal = null;
            try {
                ByteArrayInputStream byteIn = new ByteArrayInputStream(source);
                ObjIn = new ObjectInputStream(byteIn);
                retVal = ObjIn.readObject();
            } catch (Exception e) {
                e.printStackTrace();
            } finally {
                try {
                    if (null != ObjIn) {
                        ObjIn.close();
                    }
                } catch (IOException e) {
                    ObjIn = null;
                }
            }
            return retVal;
        }
    
        interface RedisExecute<T> {
            T doInvoker(Jedis jedis);
        }
    }

三、创建Redis配置类
    RedisConfig.java
    
    package com.shanhy.example.redis;
    
    import org.springframework.context.annotation.Bean;
    import org.springframework.context.annotation.Configuration;
    import org.springframework.data.redis.connection.jedis.JedisConnectionFactory;
    import org.springframework.data.redis.connection.jedis.RedisClient;
    import org.springframework.data.redis.core.RedisTemplate;
    import org.springframework.data.redis.serializer.StringRedisSerializer;
    
    /**
     * Redis配置
     * 
     * @author 单红宇(CSDN catoop)
     * @create 2016年9月12日
     */
    @Configuration
    public class RedisConfig {
    
        @Bean
        public RedisTemplate<String, Object> redisTemplate(JedisConnectionFactory factory) {
            RedisTemplate<String, Object> template = new RedisTemplate<String, Object>();
            template.setConnectionFactory(factory);
            template.setKeySerializer(new StringRedisSerializer());
            template.setValueSerializer(new RedisObjectSerializer());
            template.afterPropertiesSet();
            return template;
        }
    
        @Bean
        public RedisClient redisClient(JedisConnectionFactory factory){
            return new RedisClient(factory);
        }
    }

    RedisObjectSerializer.java
    
    package com.shanhy.example.redis;
    
    import org.springframework.core.convert.converter.Converter;
    import org.springframework.core.serializer.support.DeserializingConverter;
    import org.springframework.core.serializer.support.SerializingConverter;
    import org.springframework.data.redis.serializer.RedisSerializer;
    import org.springframework.data.redis.serializer.SerializationException;

    /**
     * 实现对象的序列化接口
     * @author   单红宇(365384722)
     * @myblog  http://blog.csdn.net/catoop/
     * @create    2017年4月9日
     */
    public class RedisObjectSerializer implements RedisSerializer<Object> {
    
        private Converter<Object, byte[]> serializer = new SerializingConverter();
        private Converter<byte[], Object> deserializer = new DeserializingConverter();
    
        static final byte[] EMPTY_ARRAY = new byte[0];
    
        @Override
        public Object deserialize(byte[] bytes) {
            if (isEmpty(bytes)) {
                return null;
            }
    
            try {
                return deserializer.convert(bytes);
            } catch (Exception ex) {
                throw new SerializationException("Cannot deserialize", ex);
            }
        }
    
        @Override
        public byte[] serialize(Object object) {
            if (object == null) {
                return EMPTY_ARRAY;
            }
    
            try {
                return serializer.convert(object);
            } catch (Exception ex) {
                return EMPTY_ARRAY;
            }
        }
    
        private boolean isEmpty(byte[] data) {
            return (data == null || data.length == 0);
        }
    
    }

四、创建测试方法
下面代码随便放一个Controller里

    @Autowired
    private RedisTemplate<String, Object> redisTemplate;

    /**
     * 缓存测试
     *
     * @return
     * @author  SHANHY
     * @create  2016年9月12日
     */
    @RequestMapping("/redisTest")
    public String redisTest() {
        try {
            redisTemplate.opsForValue().set("test-key", "redis测试内容", 2, TimeUnit.SECONDS);// 缓存有效期2秒

            logger.info("从Redis中读取数据：" + redisTemplate.opsForValue().get("test-key").toString());

            TimeUnit.SECONDS.sleep(3);

            logger.info("等待3秒后尝试读取过期的数据：" + redisTemplate.opsForValue().get("test-key"));
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        return "OK";
    }

五、配置文件配置Redis
    application.yml
    
    spring:
      # Redis配置
      redis:
        host: 192.168.1.101
        port: 6379
        password:
        # 连接超时时间（毫秒）
        timeout: 10000
        pool:
          max-idle: 20
          min-idle: 5
          max-active: 20
          max-wait: 2

这样就完成了Redis的配置，可以正常使用 redisTemplate 了。

atoop/article/details/71275331

### 一、创建 Caching 配置类

RedisKeys.java

```
package com.shanhy.example.redis;

import java.util.HashMap;
import java.util.Map;

import javax.annotation.PostConstruct;

import org.springframework.stereotype.Component;

/**
 * 方法缓存key常量
 * 
 * @author SHANHY
 */
@Component
public class RedisKeys {

    // 测试 begin
    public static final String _CACHE_TEST = "_cache_test";// 缓存key
    public static final Long _CACHE_TEST_SECOND = 20L;// 缓存时间
    // 测试 end

    // 根据key设定具体的缓存时间
    private Map<String, Long> expiresMap = null;

    @PostConstruct
    public void init(){
        expiresMap = new HashMap<>();
        expiresMap.put(_CACHE_TEST, _CACHE_TEST_SECOND);
    }

    public Map<String, Long> getExpiresMap(){
        return this.expiresMap;
    }
}

```

CachingConfig.java

```
package com.shanhy.example.redis;

import java.lang.reflect.Method;
import java.util.ArrayList;
import java.util.List;

import org.springframework.cache.CacheManager;
import org.springframework.cache.annotation.CachingConfigurerSupport;
import org.springframework.cache.annotation.EnableCaching;
import org.springframework.cache.interceptor.KeyGenerator;
import org.springframework.cache.interceptor.SimpleKeyGenerator;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.cache.RedisCacheManager;
import org.springframework.data.redis.core.RedisTemplate;

/**
 * 注解式环境管理
 * 
 * @author 单红宇(CSDN catoop)
 * @create 2016年9月12日
 */
@Configuration
@EnableCaching
public class CachingConfig extends CachingConfigurerSupport {

    /**
     * 在使用@Cacheable时，如果不指定key，则使用找个默认的key生成器生成的key
     *
     * @return
     * 
     * @author 单红宇(CSDN CATOOP)
     * @create 2017年3月11日
     */
    @Override
    public KeyGenerator keyGenerator() {
        return new SimpleKeyGenerator() {

            /**
             * 对参数进行拼接后MD5
             */
            @Override
            public Object generate(Object target, Method method, Object... params) {
                StringBuilder sb = new StringBuilder();
                sb.append(target.getClass().getName());
                sb.append(".").append(method.getName());

                StringBuilder paramsSb = new StringBuilder();
                for (Object param : params) {
                    // 如果不指定，默认生成包含到键值中
                    if (param != null) {
                        paramsSb.append(param.toString());
                    }
                }

                if (paramsSb.length() > 0) {
                    sb.append("_").append(paramsSb);
                }
                return sb.toString();
            }

        };

    }

    /**
     * 管理缓存
     *
     * @param redisTemplate
     * @return
     */
    @Bean
    public CacheManager cacheManager(RedisTemplate<String, Object> redisTemplate, RedisKeys redisKeys) {
        RedisCacheManager rcm = new RedisCacheManager(redisTemplate);
        // 设置缓存默认过期时间（全局的）
        rcm.setDefaultExpiration(1800);// 30分钟

        // 根据key设定具体的缓存时间，key统一放在常量类RedisKeys中
        rcm.setExpires(redisKeys.getExpiresMap());

        List<String> cacheNames = new ArrayList<String>(redisKeys.getExpiresMap().keySet());
        rcm.setCacheNames(cacheNames);

        return rcm;
    }

}

```

### 二、创建需要缓存数据的类

TestService.java

```
package com.shanhy.example.service;

import org.apache.commons.lang3.RandomStringUtils;
import org.springframework.cache.annotation.Cacheable;
import org.springframework.stereotype.Service;

import com.shanhy.example.redis.RedisKeys;

@Service
public class TestService {

    /**
     * 固定key
     *
     * @return
     * @author SHANHY
     * @create  2017年4月9日
     */
    @Cacheable(value = RedisKeys._CACHE_TEST, key = "'" + RedisKeys._CACHE_TEST + "'")
    public String testCache() {
        return RandomStringUtils.randomNumeric(4);
    }

    /**
     * 存储在Redis中的key自动生成，生成规则详见CachingConfig.keyGenerator()方法
     *
     * @param str1
     * @param str2
     * @return
     * @author SHANHY
     * @create  2017年4月9日
     */
    @Cacheable(value = RedisKeys._CACHE_TEST)
    public String testCache2(String str1, String str2) {
        return RandomStringUtils.randomNumeric(4);
    }
}

```

说明一下，其中 @Cacheable 中的 value 值是在 CachingConfig的cacheManager 中配置的，那里是为了配置我们的缓存有效时间。其中 methodKeyGenerator 为 CachingConfig 中声明的 KeyGenerator。
另外，Cache 相关的注解还有几个，大家可以了解下，不过我们常用的就是 @Cacheable，一般情况也可以满足我们的大部分需求了。还有 @Cacheable 也可以配置表达式根据我们传递的参数值判断是否需要缓存。
注： TestService 中 testCache 中的 mapper.get 大家不用关心，这里面我只是访问了一下数据库而已，你只需要在这里做自己的业务代码即可。

### 三、测试方法

下面代码，随便放一个 Controller 中

```
package com.shanhy.example.controller;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.redis.connection.jedis.RedisClient;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import com.shanhy.example.service.TestService;

/**
 * 测试Controller
 * 
 * @author 单红宇(365384722)
 * @myblog http://blog.csdn.net/catoop/
 * @create 2017年4月9日
 */
@RestController
@RequestMapping("/test")
public class TestController {

    private static final Logger LOG = LoggerFactory.getLogger(TestController.class);

    @Autowired
    private RedisClient redisClient;

    @Autowired
    private TestService testService;

    @GetMapping("/redisCache")
    public String redisCache() {
        redisClient.set("shanhy", "hello,shanhy", 100);
        LOG.info("getRedisValue = {}", redisClient.get("shanhy"));
        testService.testCache2("aaa", "bbb");
        return testService.testCache();
    }
}

```

至此完毕！

最后说一下，这个 @Cacheable 基本是可以放在所有方法上的，Controller 的方法上也是可以的（这个我没有测试 ^_^）。
