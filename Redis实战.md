## memcached和Redis比较   

1. 都使用内存来加速存取。    
2. Redis是非关系型数据库，支持部分事务，memcached主要提供缓存。    
3. memcached存储的键值关系中值主要为字符串，而Redis提供了字符串、列表、集合等数据结构。   

## Redis提供的数据结构   

这里的数据结构其实是键-值中的值的部分。

1. STRING：可以是字符串、整数和浮点数。   
2. LIST：一个链表结构，每个节点是一个字符串。    
3. SET：集合结构。   
4. HASH：hash表结构。   
5. ZSET：有序集合，存储的也为键值对，值表示该成员的score，按该score排序。   

### 在Spring中使用Redis

参照Spring官方[示例](https://projects.spring.io/spring-data-redis/)，是根本无法正常运行的，需要自己一步步填完其中的坑才能真正的在Spring项目中使用Redis。   

先看项目pom.xml，主要依赖于spring-data-redis和jedis：   

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.tc.redis</groupId>
    <artifactId>hello-redis</artifactId>
    <version>1.0-SNAPSHOT</version>


    <dependencies>
        <dependency>
            <groupId>org.springframework.data</groupId>
            <artifactId>spring-data-redis</artifactId>
            <version>2.0.7.RELEASE</version>
        </dependency>


        <dependency>
            <groupId>redis.clients</groupId>
            <artifactId>jedis</artifactId>
            <version>2.9.0</version>
        </dependency>

    </dependencies>

    <repositories>
        <repository>
            <id>spring-libs-release</id>
            <name>Spring Releases</name>
            <url>https://repo.spring.io/libs-release</url>
            <snapshots>
                <enabled>false</enabled>
            </snapshots>
        </repository>
    </repositories>
</project>
```

再看Spring配置文件，远程调用redis服务需要配置ip和port，创建连接Factory和redisTemplate两个bean，最后启动了自动扫描来方便的用注解自动的装配bean：    
```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/context
        http://www.springframework.org/schema/context/spring-context.xsd">

    <bean id="jedisConnFactory" class="org.springframework.data.redis.connection.jedis.JedisConnectionFactory">
        <property name="usePool" value="true"/>
        <property name="hostName" value="114.212.86.100"/>
        <property name="port" value="6379"/>
    </bean>

    <!-- redis template definition -->
    <bean id="redisTemplate" class="org.springframework.data.redis.core.StringRedisTemplate">
        <property name="connectionFactory" ref="jedisConnFactory"/>
    </bean>

    <context:component-scan base-package="com.tc.redis" />

</beans>
```
RedisTempLate选择了`StringRedisTemplate`，因为该类可以将Java中的String直接映射为redis中key，而不是进行编码转换，所以redis中的存储的key，value就是在Java代码中表达的样子。这便于通过命令行工具redis-cli进行查看。

创建一个redis调用服务：   
```java
package com.tc.redis;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.redis.core.BoundValueOperations;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.stereotype.Component;

import javax.annotation.Resource;

@Component("redisServer")
public class RedisServer {
    // inject the actual template
    @Autowired
    private RedisTemplate<String, String> redisTemplate;


    public void addLink(String key, String value) {
        BoundValueOperations<String, String> bound = redisTemplate.boundValueOps(key);
        bound.set(value);

    }
}
```

在main函数中启动spring容器：   
```java
package com.tc.redis;

import org.springframework.context.support.ClassPathXmlApplicationContext;

/**
 * Created by tianchi on 2018/5/17.
 */
public class Main {

    public static void main(String[] args){
        ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext(new String[]{"META-INF/spring/hello-redis.xml"});
        context.start();
        RedisServer rs=(RedisServer) context.getBean("redisServer");
        rs.addLink("tianchi","hello");
    }
}
```

最后注意，在redis服务端，需要在redis.conf中进行特殊配置以满足远程访问的需要：

```shell
bind 127.0.0.1 #注释掉改行   
protected-mode no # 改为no
```

### Redis的字符串操作   

Redis提供了丰富的方法对存储的字符串灵活地操作。

```java
package com.tc.redis;

import org.springframework.beans.factory.annotation.Autowired;

import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.core.ValueOperations;
import org.springframework.stereotype.Component;

@Component("redisServer")
public class RedisServer {
    // inject the actual template
    @Autowired
    private RedisTemplate<String, String> redisTemplate;


    public void addLink(String key, String value) {

        ValueOperations<String, String> valueOp = redisTemplate.opsForValue();
        valueOp.set("im","good");
        System.out.println(valueOp.get("im"));

        //整型自增
        valueOp.set("long","1");
        valueOp.increment("long",1);
        valueOp.increment("long",-5);
        System.out.println(valueOp.get("long"));
        //浮点数自增
        valueOp.increment("double",1.1);
        System.out.println(valueOp.get("double"));

        //添加
        valueOp.append("im"," job");
        System.out.println(valueOp.get("im"));
        //打印字符串子串
        System.out.println(valueOp.get("im",1,6));

        //设置子串值
        valueOp.set("im","bad",0);
        System.out.println(valueOp.get("im"));

        //位，00111111
        valueOp.set("bit","?");
        System.out.println(valueOp.getBit("bit",1));
        //设置位
        valueOp.setBit("bit",1,true);
        System.out.println(valueOp.getBit("bit",1));

    }
}

```

结果为：   
```shell
good
-3
3.3
good job
ood jo
badd job
false
true
```

### Redis的链表操作    


```java
package com.tc.redis;

import org.springframework.beans.factory.annotation.Autowired;

import org.springframework.data.redis.core.ListOperations;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.stereotype.Component;

/**
 * Created by tianchi on 2018/5/17.
 */

@Component("redisServer")
public class RedisServer {
    // inject the actual template
    @Autowired
    private RedisTemplate<String, String> redisTemplate;


    public void addLink(String key, String value) {
        ListOperations<String, String> ops = redisTemplate.opsForList();
        //添加元素的操作
        ops.rightPush("mlist", "two");
        ops.rightPush("mlist", "three");
        ops.leftPush("mlist", "one");
        //获取范围内的元素
        System.out.println(ops.range("mlist", 0, -1));
        System.out.println(ops.range("mlist", 0, 1));
        ops.leftPush("mlist", "zero");
        //修剪
        ops.trim("mlist", 0, -2);
        System.out.println(ops.range("mlist", 0, -1));

        //转移元素
        System.out.println(ops.rightPopAndLeftPush("mlist", "mlist2"));
        System.out.println(ops.range("mlist2", 0, -1));
        redisTemplate.delete("mlist");
        redisTemplate.delete("mlist2");
    }
}
```

输出结果：   

```
[one, two, three]
[one, two]
[zero, one, two]
two
[two]
```

### Redis的集合操作

```java
package com.tc.redis;

import org.springframework.beans.factory.annotation.Autowired;

import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.core.SetOperations;
import org.springframework.stereotype.Component;

/**
 * Created by tianchi on 2018/5/17.
 */

@Component("redisServer")
public class RedisServer {
    // inject the actual template
    @Autowired
    private RedisTemplate<String, String> redisTemplate;


    public void addLink(String key, String value) {
        SetOperations<String, String> ops = redisTemplate.opsForSet();
        ops.add("mSet", "one");
        ops.add("mSet", "two");
        ops.add("mSet", "three");
        ops.add("mSet", "four");
        ops.add("mSet", "five");
        ops.add("mSet", "six");

        ops.remove("mSet", "two");
        System.out.println(ops.members("mSet"));
        System.out.println(ops.isMember("mSet", "one"));
        System.out.println(ops.size("mSet"));

        //随机获得其中的几个元素
        System.out.println(ops.randomMembers("mSet", 3));

        //随机的移除元素，并返回
        System.out.println(ops.pop("mSet"));

        //转移元素
        ops.move("mSet", "one", "mSet2");
        System.out.println(ops.members("mSet2"));

        ops.add("mSet2", "five", "ten");
        //差集
        System.out.println(ops.difference("mSet", "mSet2"));
        //交集
        System.out.println(ops.intersect("mSet", "mSet2"));

        System.out.println(ops.members("mSet"));
        System.out.println(ops.members("mSet2"));
        //求并集，并存入第三个集合
        ops.unionAndStore("mSet", "mSet2", "mSet3");
        System.out.println(ops.members("mSet3"));

        redisTemplate.delete("mSet");
        redisTemplate.delete("mSet2");
        redisTemplate.delete("mSet3");
    }

}

```

输入结果：   

```
[three, one, four, five, six]
true
5
[six, six, five]
six
[one]
[four, three]
[five]
[three, four, five]
[five, ten, one]
[four, five, ten, three, one]
```

### Redis的散列表操作   

```java
package com.tc.redis;

import org.springframework.beans.factory.annotation.Autowired;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.data.redis.core.HashOperations;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.core.ValueOperations;
import org.springframework.stereotype.Component;

/**
 * Created by tianchi on 2018/5/17.
 */

@Component("redisServer")
public class RedisServer {
    // inject the actual template
    @Autowired
    private RedisTemplate<String, String> redisTemplate;


    public void addLink(String key, String value) {
        HashOperations<String, String, String> ops = redisTemplate.opsForHash();

        ops.put("mHash", "one", "1");
        ops.put("mHash", "two", "2");
        ops.put("mHash", "three", "3");

        System.out.println(ops.get("mHash", "three"));
        ops.delete("mHash", "one");
        System.out.println(ops.size("mHash"));

        System.out.println(ops.hasKey("mHash", "two"));
        System.out.println(ops.keys("mHash"));
        System.out.println(ops.values("mHash"));
        //获取所有元素
        System.out.println(ops.entries("mHash"));

        //自增
        ops.increment("mHash", "two", 1L);
        System.out.println(ops.get("mHash", "two"));


        redisTemplate.delete("mHash");

    }

}
```

输出结果：   
```
3
2
true
[two, three]
[2, 3]
{two=2, three=3}
3
```