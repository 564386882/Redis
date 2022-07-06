# SpringBoot集成Redis

添加依赖，使用lettuce作为客户端

```xml
		<!-- spring boot redis 缓存引入 -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-redis</artifactId>
        </dependency>
        <!-- redis lettuce pool 需要这个依赖 -->
        <dependency>
            <groupId>org.apache.commons</groupId>
            <artifactId>commons-pool2</artifactId>
        </dependency>
```

添加key序列化（可配置前缀）

```java
import com.alibaba.fastjson.JSON;
import com.alibaba.fastjson.parser.ParserConfig;
import com.alibaba.fastjson.serializer.SerializerFeature;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.redis.serializer.RedisSerializer;
import org.springframework.data.redis.serializer.SerializationException;

import java.nio.charset.Charset;
import java.nio.charset.StandardCharsets;

/**
 * 为redis key 统一加上前缀
 */
@Slf4j
public class RedisKeySerializer implements RedisSerializer<String> {
	public static final Charset DEFAULT_CHARSET = StandardCharsets.UTF_8;

	static {
		ParserConfig.getGlobalInstance().setAutoTypeSupport(true);
	}

	private String prefix;

	public RedisKeySerializer(String prefix) {
		super();
		this.prefix = prefix;
	}

	/**
	 * 序列化
	 */
	@Override
	public byte[] serialize(String key) throws SerializationException {
		if (null == key) {
			return new byte[0];
		}
		return JSON.toJSONString(prefix + key, SerializerFeature.WriteClassName).getBytes(DEFAULT_CHARSET);
	}

	/**
	 * 反序列化
	 */
	@Override
	public String deserialize(byte[] bytes) throws SerializationException {
		if (null == bytes || bytes.length <= 0) {
			return null;
		}
		String text = new String(bytes, DEFAULT_CHARSET);
		return JSON.parseObject(prefix + text, String.class);
	}
}
```

添加value序列化

```java
public class FastJson2JsonRedisSerializer<T> implements RedisSerializer<T> {

    public static final Charset DEFAULT_CHARSET = StandardCharsets.UTF_8;

    static {
        ParserConfig.getGlobalInstance().setAutoTypeSupport(true);
    }

    private final Class<T> clazz;

    public FastJson2JsonRedisSerializer(Class<T> clazz) {
        super();
        this.clazz = clazz;
    }

    /**
     * 序列化
     */
    @Override
    public byte[] serialize(T t) throws SerializationException {
        if (null == t) {
            return new byte[0];
        }
       return JSON.toJSONString(t, SerializerFeature.WriteClassName).getBytes(DEFAULT_CHARSET);
    }

    /**
     * 反序列化
     */
    @Override
    public T deserialize(byte[] bytes) throws SerializationException {
        if (null == bytes || bytes.length <= 0) {
            return null;
        }
        return (T) JSON.parseObject(new String(bytes, DEFAULT_CHARSET), clazz);
    }
}
```

配置redis序列化，采用自定义的序列化配置

```java
/**
 * @description Redis配置类：Lettuce客户端
 */
@Configuration
@Slf4j
public class LettuceRedisConfig {

   @Value("${redis.key-prefix}")
   private String prefix;

   @Bean(name = "l2RedisTemplate")
   public RedisTemplate<String, Object> l2RedisTemplate(LettuceConnectionFactory lettuceConnectionFactory) {
      // 配置redisTemplate
      RedisTemplate<String, Object> redisTemplate = new RedisTemplate<>();
      redisTemplate.setConnectionFactory(lettuceConnectionFactory);
      // 设置序列化
      RedisKeySerializer redisKeySerializer = new RedisKeySerializer(prefix);
      FastJson2JsonRedisSerializer<Object> valueSerializer = new FastJson2JsonRedisSerializer<>(Object.class);
      // key序列化，使用自定义序列化方式
      redisTemplate.setKeySerializer(redisKeySerializer);
      // value序列化
      redisTemplate.setValueSerializer(valueSerializer);
      // Hash key序列化，使用自定义序列化方式
      redisTemplate.setHashKeySerializer(redisKeySerializer);
      // Hash value序列化
      redisTemplate.setHashValueSerializer(valueSerializer);
      redisTemplate.afterPropertiesSet();
      return redisTemplate;
   }

   @Bean(name = "l2StringRedisTemplate")
   public RedisTemplate<String, String> l2StringRedisTemplate(LettuceConnectionFactory lettuceConnectionFactory) {
      // 配置redisTemplate
      RedisTemplate<String, String> redisTemplate = new RedisTemplate<>();
      redisTemplate.setConnectionFactory(lettuceConnectionFactory);
      // 设置序列化
      RedisKeySerializer redisKeySerializer = new RedisKeySerializer(prefix);
      log.info("redis-prefix:{}", prefix);
      FastJson2JsonRedisSerializer<Object> valueSerializer = new FastJson2JsonRedisSerializer<>(Object.class);
      // key序列化，使用自定义序列化方式
      redisTemplate.setKeySerializer(redisKeySerializer);
      // value序列化
      redisTemplate.setValueSerializer(valueSerializer);
      // Hash key序列化，使用自定义序列化方式
      redisTemplate.setHashKeySerializer(redisKeySerializer);
      // Hash value序列化
      redisTemplate.setHashValueSerializer(valueSerializer);
      redisTemplate.afterPropertiesSet();
      return redisTemplate;
   }
}
```
