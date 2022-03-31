---
typora-copy-images-to: SpringBootNotesPictures
---

# 本地缓存组件 Caffeine

Redis 这种 NoSql 作为分布式缓存组件，能提供多个服务间的缓存，但是 Redis 需要网络开销，增加时耗。本地缓存是直接从本地内存中读取，没有网络开销，例如秒杀系统或者数据量小的缓存等，比远程缓存更合适。

在下面缓存组件中 Caffeine 性能是其中最好的：

![Caffeine性能](SpringBootNotesPictures/Caffeine性能.png)

## Caffeine 参数配置

| 参数              |   类型   | 描述                                                         |
| ----------------- | :------: | ------------------------------------------------------------ |
| initialCapacity   | integer  | 初始的缓存空间大小                                           |
| maximumSize       |   long   | 缓存的最大条数                                               |
| maximumWeight     |   long   | 缓存的最大权重                                               |
| expireAfterAccess | duration | 最后一次写入或访问后，指定经过多长的时间过期                 |
| expireAfterWrite  | duration | 最后一次写入后，指定经过多长的时间缓存过期                   |
| refreshAfterWrite | duration | 创建缓存或者最近一次更新缓存后，经过指定的时间间隔后刷新缓存 |
| weakKeys          | boolean  | 打开 key 的弱引用                                            |
| weakValues        | boolean  | 打开 value 的弱引用                                          |
| softValues        | boolean  | 打开 value 的软引用                                          |
| recordStats       |    -     | 开发统计功能                                                 |

**注意：**

- `weakValues` 和 `softValues` 不可以同时使用。
- `maximumSize` 和 `maximumWeight` 不可以同时使用。
- `expireAfterWrite` 和 `expireAfterAccess` 同时存在时，以 `expireAfterWrite` 为准。



## 软引用与弱引用

- **软引用：** 如果一个对象只具有软引用，且内存空间足够，垃圾回收器就不会回收它；如果内存空间不足了，就会回收这些对象的内存。
- **弱引用：** 弱引用的对象拥有更短暂的生命周期。在垃圾回收器线程扫描它所管辖的内存区域的过程中，一旦发现了只具有弱引用的对象，不管当前内存空间足够与否，都会回收它的内存

```java
// 软引用
Caffeine.newBuilder().softValues().build();

// 弱引用
Caffeine.newBuilder().weakKeys().weakValues().build();
```



# 使用 Caffeine 方法实现缓存

直接引入 Caffeine 依赖，然后使用 Caffeine 方法实现缓存。

## 实现步骤

### 依赖

```xml
<dependency>
    <groupId>com.github.ben-manes.caffeine</groupId>
    <artifactId>caffeine</artifactId>
</dependency>
```

完整依赖：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.3.7.RELEASE</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>

    <groupId>org.example</groupId>
    <artifactId>CaffineCache</artifactId>
    <version>1.0-SNAPSHOT</version>

    <properties>
        <java.version>1.8</java.version>
    </properties>
    
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <dependency>
            <groupId>com.github.ben-manes.caffeine</groupId>
            <artifactId>caffeine</artifactId>
        </dependency>

        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
        </dependency>

        <!-- https://mvnrepository.com/artifact/io.swagger.core.v3/swagger-models -->
        <dependency>
            <groupId>io.springfox</groupId>
            <artifactId>springfox-swagger2</artifactId>
            <version>2.9.2</version>
        </dependency>
        <!-- https://mvnrepository.com/artifact/io.springfox/springfox-swagger-ui -->
        <dependency>
            <groupId>io.springfox</groupId>
            <artifactId>springfox-swagger-ui</artifactId>
            <version>2.9.2</version>
        </dependency>
        <!-- https://mvnrepository.com/artifact/com.github.xiaoymin/knife4j-spring-boot-starter -->
        <dependency>
            <groupId>com.github.xiaoymin</groupId>
            <artifactId>knife4j-spring-boot-starter</artifactId>
            <version>2.0.9</version>
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

### 缓存配置类

可以直接在配置文件中进行配置，也可以使用`JavaConfig`的配置方式来代替配置文件：

```java
package com.example.config;

import com.github.benmanes.caffeine.cache.Cache;
import com.github.benmanes.caffeine.cache.Caffeine;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.util.concurrent.TimeUnit;

@Configuration
public class CaffeineCacheConfig {
    @Bean
    public Cache<String, Object> caffeineCache() {
        return Caffeine.newBuilder()
                // 设置最后一次写入或访问后经过固定时间过期
                .expireAfterWrite(60, TimeUnit.SECONDS)
                // 初始的缓存空间大小
                .initialCapacity(100)
                // 缓存的最大条数
                .maximumSize(100)
                .build();
    }
}
```

### 实体对象和服务接口

实体对象：

```java
package com.example.entity;

import lombok.Data;
import lombok.ToString;

@Data
@ToString
public class UserInfo {
    private Integer id;
    private String name;
    private String sexual;
    private Integer age;
}
```

服务接口：

```java
package com.example.service;

import com.example.entity.UserInfo;

public interface UserInfoService {
    void addUserInfo(UserInfo userInfo);

    UserInfo getUserInfoById(Integer id);

    UserInfo updateUserInfo(UserInfo userInfo);

    void deleteUserInfoById(Integer id);
}
```

### 服务接口实现类

```java
package com.example.service.impl;

import com.example.entity.UserInfo;
import com.example.service.UserInfoService;
import com.github.benmanes.caffeine.cache.Cache;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

import java.util.HashMap;

@Slf4j
@Service
public class UserInfoServiceImpl implements UserInfoService {

    /**
     * 模拟数据库存储数据
     */
    private HashMap<Integer, UserInfo> userInfoHashMap = new HashMap<>();

    /**
     * 使用 caffeineCache bean
     */
    @Autowired
    private Cache<String, Object> caffeineCache;

    @Override
    public void addUserInfo(UserInfo userInfo) {
        log.info("addUserInfo");
        userInfoHashMap.put(userInfo.getId(), userInfo);
        // 加入缓存中
        caffeineCache.put(String.valueOf(userInfo.getId()), userInfo);
    }

    @Override
    public UserInfo getUserInfoById(Integer id) {
        log.info("试图先从缓存中读取");
        // Object ifPresent = caffeineCache.getIfPresent(String.valueOf(id));
        UserInfo userInfo = (UserInfo) caffeineCache.asMap().get(String.valueOf(id));
        if (userInfo != null) {
            return userInfo;
        }

        // 如果缓存不存在，则从数据库中获取
        log.info("缓存中没有，则从数据库中获取");
        userInfo = userInfoHashMap.get(id);
        // 加入缓存中
        if (userInfo != null) {
            caffeineCache.put(String.valueOf(userInfo.getId()), userInfo);
        }
        return userInfo;
    }

    @Override
    public UserInfo updateUserInfo(UserInfo userInfo) {
        log.info("updateUserInfo");
        if (userInfoHashMap.containsKey(userInfo.getId())) {
            return null;
        }

        UserInfo oldUserInfo = userInfoHashMap.get(userInfo.getId());
        oldUserInfo.setAge(userInfo.getAge());
        oldUserInfo.setName(userInfo.getName());
        oldUserInfo.setSexual(userInfo.getSexual());
        userInfoHashMap.put(oldUserInfo.getId(), oldUserInfo);
        caffeineCache.put(String.valueOf(oldUserInfo.getId()), oldUserInfo);
        return oldUserInfo;
    }

    @Override
    public void deleteUserInfoById(Integer id) {
        log.info("deleteUserInfoById");
        if (caffeineCache.getIfPresent(String.valueOf(id)) != null) {
            userInfoHashMap.remove(id);
            // 从缓存中删除
            caffeineCache.asMap().remove(String.valueOf(id));
        }
    }
}
```

### Controller 和启动类

Controller 类：

```java
package com.example.controller;

import com.example.entity.UserInfo;
import com.example.service.UserInfoService;
import io.swagger.annotations.Api;
import io.swagger.annotations.ApiOperation;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.*;

@Api(tags = "caffeine controller")
@RestController
@RequestMapping("/caffeine")
public class UserInfoController {
    @Autowired
    private UserInfoService userInfoService;

    @ApiOperation("get")
    @GetMapping("/userInfo")
    public UserInfo getUserInfoById(@RequestParam Integer id) {
        UserInfo userInfo = userInfoService.getUserInfoById(id);
        return userInfo;
    }

    @ApiOperation("add")
    @PostMapping("/add")
    public String addUserInfo(@RequestBody UserInfo userInfo) {
        userInfoService.addUserInfo(userInfo);
        return "Success";
    }

    @ApiOperation("update")
    @PostMapping("/update")
    public UserInfo updateUserInfo(@RequestBody UserInfo userInfo) {
        return userInfoService.updateUserInfo(userInfo);
    }

    @ApiOperation("delete")
    @GetMapping("/delete")
    public String deleteUserInfoById(@RequestParam Integer id) {
        userInfoService.deleteUserInfoById(id);
        return "SUCCESS";
    }
}
```

启动类：

```java
package com.example;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class CaffeineApplication {
    public static void main(String[] args) {
        SpringApplication.run(CaffeineApplication.class, args);
    }
}
```

# 使用 SpringCache 注解实现缓存

## 实现步骤

### 依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-cache</artifactId>
</dependency>

<dependency>
    <groupId>com.github.ben-manes.caffeine</groupId>
    <artifactId>caffeine</artifactId>
</dependency>
```

完整依赖：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.3.7.RELEASE</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>

    <groupId>org.example</groupId>
    <artifactId>SpringCaffeineCache</artifactId>
    <version>1.0-SNAPSHOT</version>
    
    <properties>
        <java.version>1.8</java.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-cache</artifactId>
        </dependency>

        <dependency>
            <groupId>com.github.ben-manes.caffeine</groupId>
            <artifactId>caffeine</artifactId>
        </dependency>

        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
        </dependency>

        <!-- https://mvnrepository.com/artifact/io.swagger.core.v3/swagger-models -->
        <dependency>
            <groupId>io.springfox</groupId>
            <artifactId>springfox-swagger2</artifactId>
            <version>2.9.2</version>
        </dependency>
        <!-- https://mvnrepository.com/artifact/io.springfox/springfox-swagger-ui -->
        <dependency>
            <groupId>io.springfox</groupId>
            <artifactId>springfox-swagger-ui</artifactId>
            <version>2.9.2</version>
        </dependency>
        <!-- https://mvnrepository.com/artifact/com.github.xiaoymin/knife4j-spring-boot-starter -->
        <dependency>
            <groupId>com.github.xiaoymin</groupId>
            <artifactId>knife4j-spring-boot-starter</artifactId>
            <version>2.0.9</version>
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

### 缓存配置类

可以直接在配置文件中进行配置，也可以使用`JavaConfig`的配置方式来代替配置文件：

```java
package com.example.config;

import com.github.benmanes.caffeine.cache.Caffeine;
import org.springframework.cache.CacheManager;
import org.springframework.cache.caffeine.CaffeineCacheManager;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.util.concurrent.TimeUnit;

@Configuration
public class CaffeineCacheConfig {
    @Bean("caffeineCacheManager")
    public CacheManager caffeineCacheManager() {
        CaffeineCacheManager caffeineCacheManager = new CaffeineCacheManager();
        caffeineCacheManager.setCaffeine(Caffeine.newBuilder()
                // 设置最后一次写入或访问后经过固定时间过期
                .expireAfterAccess(60, TimeUnit.SECONDS)
                // 初始的缓存空间大小
                .initialCapacity(100)
                // 缓存的最大条数
                .maximumSize(100));
        return caffeineCacheManager;
    }
}
```

### 实体对象和服务接口

实体对象：

```java
package com.example.entity;

import lombok.Data;
import lombok.ToString;

@Data
@ToString
public class UserInfo {
    private Integer id;
    private String name;
    private String sexual;
    private Integer age;
}
```

服务接口：

```java
package com.example.service;

import com.example.entity.UserInfo;

public interface UserInfoService {
    UserInfo addUserInfo(UserInfo userInfo);

    UserInfo getUserInfoById(Integer id);

    UserInfo updateUserInfo(UserInfo userInfo);

    void deleteUserInfoById(Integer id);
}
```

### 接口实现类：注解使用

```java
package com.example.service.impl;

import com.example.entity.UserInfo;
import com.example.service.UserInfoService;
import lombok.extern.slf4j.Slf4j;
import org.springframework.cache.annotation.CacheEvict;
import org.springframework.cache.annotation.CachePut;
import org.springframework.cache.annotation.Cacheable;
import org.springframework.stereotype.Service;

import java.util.HashMap;

@Slf4j
@Service
public class UserInfoServiceImpl implements UserInfoService {

    /**
     * 模拟数据库存储数据
     */
    private HashMap<Integer, UserInfo> userInfoHashMap = new HashMap<>();


    @Override
    @CachePut(key = "#userInfo.id", value = "userInfo")
    public UserInfo addUserInfo(UserInfo userInfo) {
        log.info("addUserInfo");
        userInfoHashMap.put(userInfo.getId(), userInfo);
        return userInfo;
    }

    @Override
    @Cacheable(key = "#id", value = "userInfo")
    public UserInfo getUserInfoById(Integer id) {
        log.info("缓存中没有，则从数据库中获取");
        return userInfoHashMap.get(id);

    }

    @Override
    @CachePut(key = "#userInfo.id", value = "userInfo")
    public UserInfo updateUserInfo(UserInfo userInfo) {
        log.info("updateUserInfo");
        if (userInfoHashMap.containsKey(userInfo.getId())) {
            return null;
        }

        UserInfo oldUserInfo = userInfoHashMap.get(userInfo.getId());
        oldUserInfo.setAge(userInfo.getAge());
        oldUserInfo.setName(userInfo.getName());
        oldUserInfo.setSexual(userInfo.getSexual());
        userInfoHashMap.put(oldUserInfo.getId(), oldUserInfo);
        return oldUserInfo;
    }

    @Override
    @CacheEvict(key = "#id", value = "userInfo")
    public void deleteUserInfoById(Integer id) {
        log.info("deleteUserInfoById");
        userInfoHashMap.remove(id);
    }
}
```

### controller 层

```java
package com.example.controller;

import com.example.entity.UserInfo;
import com.example.service.UserInfoService;
import io.swagger.annotations.Api;
import io.swagger.annotations.ApiOperation;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.*;

@Api(tags = "SpringCacheCaffeine controller")
@RestController
@RequestMapping("/springCacheCaffeine")
public class UserInfoController {
    @Autowired
    private UserInfoService userInfoService;

    @ApiOperation("get")
    @GetMapping("/userInfo")
    public UserInfo getUserInfoById(@RequestParam Integer id) {
        UserInfo userInfo = userInfoService.getUserInfoById(id);
        return userInfo;
    }

    @ApiOperation("add")
    @PostMapping("/add")
    public UserInfo addUserInfo(@RequestBody UserInfo userInfo) {
        return userInfoService.addUserInfo(userInfo);
    }

    @ApiOperation("update")
    @PostMapping("/update")
    public UserInfo updateUserInfo(@RequestBody UserInfo userInfo) {
        return userInfoService.updateUserInfo(userInfo);
    }

    @ApiOperation("delete")
    @GetMapping("/delete")
    public String deleteUserInfoById(@RequestParam Integer id) {
        userInfoService.deleteUserInfoById(id);
        return "SUCCESS";
    }
}
```

### 启动类

使用`@EnableCaching`注解让 Spring Boot 开启对缓存的支持：

```java
package com.example;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cache.annotation.EnableCaching;

@SpringBootApplication
@EnableCaching
public class SpringCacheCaffeineApplication {
    public static void main(String[] args) {
        SpringApplication.run(SpringCacheCaffeineApplication.class, args);
    }
}
```





# 参考资料

[SpringBoot 使用 Caffeine 本地缓存](http://www.mydlq.club/article/56/)

[Caffeine Cache-高性能Java本地缓存组件](https://www.cnblogs.com/rickiyang/p/11074158.html)：原理



Caffeine 详解 —— Caffeine 使用：https://zhuanlan.zhihu.com/p/329684099——参数

Caffeine缓存：https://www.jianshu.com/p/9a80c662dac4——策略等