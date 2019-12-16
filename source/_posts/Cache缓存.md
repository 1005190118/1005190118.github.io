---
title: Cacha缓存
---



> 这里是使用redis做Mybatis二级缓存

# 第一步添加依赖

  ```xml
  		<!-- cache缓存 -->
  		<dependency>
  			<groupId>org.springframework.boot</groupId>
  			<artifactId>spring-boot-starter-cache</artifactId>
  		</dependency>
  		<!--添加Redis依赖 -->
  		<dependency>
  			<groupId>org.springframework.boot</groupId>
  			<artifactId>spring-boot-starter-data-redis</artifactId>
  		</dependency>
  ```

# 第二步启动类上添加注解

  ```java
  @EnableCaching
  ```

  

# 第三步编写RedisConfig

  ```java
  @Configuration
  public class RedisConfig {
  
      @Bean
      public RedisTemplate<String,String> redisTemplate(RedisConnectionFactory factory){
          RedisTemplate<String,String> redisTemplate  = new RedisTemplate<>();
  
          redisTemplate.setConnectionFactory(factory);
          return redisTemplate;
      }
  
      /**
       * 生成自定义的key
       * @return
       */
      @Bean("myKeyGenerator")
      public KeyGenerator MyKeyGenerator() {
          return (o, method, objects) -> {
              StringBuilder stringBuilder = new StringBuilder();
              stringBuilder.append(o.getClass().getSimpleName());
              stringBuilder.append(".");
              stringBuilder.append(method.getName());
              stringBuilder.append("[");
              for (Object obj : objects) {
                  stringBuilder.append(obj.toString());
              }
              stringBuilder.append("]");
              return stringBuilder.toString();
          };
      }
  
      @Bean
      public CacheManager cacheManager(RedisConnectionFactory redisConnectionFactory) {
          return new RedisCacheManager(
                  RedisCacheWriter.nonLockingRedisCacheWriter(redisConnectionFactory),
                  /**
                   * 默认策略，未配置的 key 会使用这个
                   */
                  this.getRedisCacheConfigurationWithTtl(6000),
                  /**
                   * // 指定 key 策略
                   */
                  this.getRedisCacheConfigurationMap()
          );
      }
  
      private Map<String, RedisCacheConfiguration> getRedisCacheConfigurationMap() {
          Map<String, RedisCacheConfiguration> redisCacheConfigurationMap = new HashMap<>();
          redisCacheConfigurationMap.put("shmUserInfo", this.getRedisCacheConfigurationWithTtl(3000));
          redisCacheConfigurationMap.put("UserInfoListAnother", this.getRedisCacheConfigurationWithTtl(18000));
          return redisCacheConfigurationMap;
      }
  
      private RedisCacheConfiguration getRedisCacheConfigurationWithTtl(Integer seconds) {
          Jackson2JsonRedisSerializer<Object> jackson2JsonRedisSerializer = new Jackson2JsonRedisSerializer<>(Object.class);
          ObjectMapper om = new ObjectMapper();
          om.setVisibility(PropertyAccessor.ALL, JsonAutoDetect.Visibility.ANY);
          om.enableDefaultTyping(ObjectMapper.DefaultTyping.NON_FINAL);
          jackson2JsonRedisSerializer.setObjectMapper(om);
  
          RedisCacheConfiguration redisCacheConfiguration = RedisCacheConfiguration.defaultCacheConfig();
          redisCacheConfiguration = redisCacheConfiguration.serializeValuesWith(
                  RedisSerializationContext
                          .SerializationPair
                          .fromSerializer(jackson2JsonRedisSerializer)
          ).entryTtl(Duration.ofSeconds(seconds));
  
          return redisCacheConfiguration;
      }
  
  }
  
  ```

# 第四步在service上使用

  > **直接在service类上注解**@CacheConfig**指定cacheNames（缓存名称），keyGenerator（key的生成器），cacheManager（那个缓存器），就不用在下面的3中注解上指定了。**
  >
  > > **3种注解：**
  > >
  > > **1）：**@Cacheable **用于查找**
  > >
  > > **2）：**@CachePut **用于增加，更改**
  > >
  > > **3）：**@CacheEvict **用于删除**

  示例：

  ```java
  @Service
  @CacheConfig(cacheNames = "shmUserInfo")
  public class UserServiceImpl implements UserService {
  
      @Resource
      private UserMapper userMapper;
  
      /**
       * 保存用户信息
       * @param user
       * @return
       */
      @Override
      @CachePut(key = "'UserServiceImpl.findByPhone['+#result.phone+']'")
      public  int saveUser(User user) {
          return userMapper.saveUser(user);
      }
  
      /**
       * 通过手机号查找用户
       * @param phone
       * @return
       */
      @Override
      @Cacheable(keyGenerator = "myKeyGenerator")
      public User findByPhone(String phone) {
          return userMapper.findByPhone(phone);
      }
  
      @Override
      @Cacheable(keyGenerator = "myKeyGenerator")
      public User findById(Integer id) {
          return userMapper.findById(id);
      }
  
      /**
       * 更改用户信息
       * @param user
       * @return
       */
      @Override
      @CachePut(key = "'UserServiceImpl.findByPhone['+#result.phone+']'")
      public User updateUser(User user) {
          userMapper.updateUser(user);
          return  user;
      }
  
      @Override
      @Cacheable(keyGenerator = "myKeyGenerator")
      public List<User> listUser(){
          return userMapper.listUser();
      }
  }
  注解详情看B站springboot高级教程，前几章讲的是cache缓存，讲上面3个注解

  ```

  