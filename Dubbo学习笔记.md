
# Dubbo学习笔记

## Dubbo的设计目的  
1. 动态注册、发现服务，软负载均衡和服务降级。   
2. 描述服务依赖关系，描述整体服务架构。   
3. 统计服务负载，按需扩展容量。


## 简单实例   

该实例参考官方[实例](https://github.com/alibaba/dubbo/tree/master/dubbo-demo)，有所修改。


整个项目结构：

```
dubbo-hello
.
├── dubbo-hello-api
│   ├── pom.xml
│   ├── src
│   │   ├── main
│   │   │   ├── java
│   │   │   │   └── com
│   │   │   │       └── tc
│   │   │   │           └── dubbo
│   │   │   │               └── hello
│   │   │   │                   └── HelloService.java
│   │   │   └── resources
│   │   └── test
│   │       └── java
│   └── target
│
├── dubbo-hello-consumer
│   ├── pom.xml
│   ├── src
│   │   ├── main
│   │   │   ├── java
│   │   │   │   └── com
│   │   │   │       └── tc
│   │   │   │           └── dubbo
│   │   │   │               └── hello
│   │   │   │                   └── consumer
│   │   │   │                       └── Consumer.java
│   │   │   └── resources
│   │   │       ├── META-INF
│   │   │       │   └── spring
│   │   │       │       └── dubbo-hello-consumer.xml
│   │   │       ├── dubbo.properties
│   │   │       └── log4j.properties
│   │   └── test
│   │       └── java
│   └── target
│      
├── dubbo-hello-provider
│   ├── pom.xml
│   ├── src
│   │   ├── main
│   │   │   ├── java
│   │   │   │   └── com
│   │   │   │       └── tc
│   │   │   │           └── dubbo
│   │   │   │               └── hello
│   │   │   │                   └── provider
│   │   │   │                       ├── HelloServiceImpl.java
│   │   │   │                       └── Provider.java
│   │   │   └── resources
│   │   │       ├── META-INF
│   │   │       │   └── spring
│   │   │       │       └── dubbo-hello-provider.xml
│   │   │       ├── dubbo.properties
│   │   │       └── log4j.properties
│   │   └── test
│   │       └── java
│   └── target
└── pom.xml
```

整个项目分为三个模块，dubbo-hello-api、dubbo-hello-provider和dubbo-hello-consumer。一下是整个dubbo-hello工程的pom内容：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.tc</groupId>
    <artifactId>dubbo-hello</artifactId>
    <version>1.0-SNAPSHOT</version>
    <packaging>pom</packaging>

    <name>${project.artifactId}</name>

    <properties>
        <skip_maven_deploy>false</skip_maven_deploy>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <maven.compiler.source>1.8</maven.compiler.source>
        <maven.compiler.target>1.8</maven.compiler.target>
    </properties>

    <modules>
        <module>dubbo-hello-api</module>
        <module>dubbo-hello-provider</module>
        <module>dubbo-hello-consumer</module>
    </modules>

</project>
```

### 公共API定义

一般Provider和Consumer共享的接口应该单独一个包进行定义，方便双方的共享。这里新建了模块`dubbo-hello-api`来定义公共接口。其pom文件为基本配置，无特殊性，简单定义接口HelloService如下：    

```java
package com.tc.dubbo.hello;

/**
 * Created by tianchi on 2018/3/14.
 */
public interface HelloService {
    String sayHello(String name);
}
```

### Provider实现

服务提供者模块`dubbo-hello-provider`的pom文件内容如下：   
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <parent>
        <artifactId>dubbo-hello</artifactId>
        <groupId>com.tc</groupId>
        <version>1.0-SNAPSHOT</version>
    </parent>
    <modelVersion>4.0.0</modelVersion>

    <artifactId>dubbo-hello-provider</artifactId>
    <packaging>jar</packaging>
    <name>${project.artifactId}</name>
    <description>The demo provider module of dubbo project</description>
    <properties>
        <skip_maven_deploy>false</skip_maven_deploy>
    </properties>

    <dependencies>
        <dependency>
            <groupId>com.tc</groupId>
            <artifactId>dubbo-hello-api</artifactId>
            <version>${project.parent.version}</version>
        </dependency>

        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>dubbo</artifactId>
            <version>2.6.0</version>
        </dependency>

    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-assembly-plugin</artifactId>
                <version>2.3</version>
                <configuration>
                    <appendAssemblyId>false</appendAssemblyId>
                    <descriptorRefs>
                        <descriptorRef>jar-with-dependencies</descriptorRef>
                    </descriptorRefs>
                    <archive>
                        <manifest>
                            <mainClass>com.tc.dubbo.hello.provider.Provider</mainClass>
                        </manifest>
                    </archive>
                </configuration>
                <executions>
                    <execution>
                        <id>make-assembly</id>
                        <phase>package</phase>
                        <goals>
                            <goal>assembly</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>

</project>
```

其中声明了对`com.tc:dubbo-hello-api`接口的依赖，还有对`com.alibaba:dubbo`的依赖。插件plugin声明了采用加入依赖的打包方式，这样方便直接运行。


在文件`dubbo-hello-provider.xml`中声明了要暴露的服务：    

```xml
<?xml version="1.0" encoding="UTF-8"?>

<beans xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:dubbo="http://code.alibabatech.com/schema/dubbo"
       xmlns="http://www.springframework.org/schema/beans"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-4.3.xsd
       http://code.alibabatech.com/schema/dubbo http://code.alibabatech.com/schema/dubbo/dubbo.xsd">

    <!-- provider's application name, used for tracing dependency relationship -->
    <dubbo:application name="hello-provider"/>

    <!-- use multicast registry center to export service -->
    <dubbo:registry address="multicast://224.5.6.7:1234"/>

    <!-- use dubbo protocol to export service on port 20880 -->
    <dubbo:protocol name="dubbo" port="20880"/>

    <!-- service implementation, as same as regular local bean -->
    <bean id="helloService" class="com.tc.dubbo.hello.provider.HelloServiceImpl"/>

    <!-- declare the service interface to be exported -->
    <dubbo:service interface="com.tc.dubbo.hello.HelloService" ref="helloService"/>

</beans>
```    

其中说明了注册方式为multicast，也即利用广播，无注册中心的方式。同时声明了协议和端口号。


暴露的服务HelloServiceImpl实现如下：

```java
package com.tc.dubbo.hello.provider;

import com.alibaba.dubbo.rpc.RpcContext;
import com.tc.dubbo.hello.HelloService;

import java.text.SimpleDateFormat;
import java.util.Date;

/**
 * Created by tianchi on 2018/3/14.
 */
public class HelloServiceImpl implements HelloService{
    public String sayHello(String name) {
        System.out.println("[" + new SimpleDateFormat("HH:mm:ss").format(new Date()) + "] Hello " + name + ", request from consumer: " + RpcContext.getContext().getRemoteAddress());
        return "Hello " + name + ", response form provider: " + RpcContext.getContext().getLocalAddress();
    }
}
```

入口类Provider的实现如下：    
```java
package com.tc.dubbo.hello.provider;

import org.springframework.context.support.ClassPathXmlApplicationContext;

/**
 * Created by tianchi on 2018/3/14.
 */
public class Provider {

    public static void main(String[] args) throws Exception {
        //Prevent to get IPV6 address,this way only work in debug mode
        //But you can pass use -Djava.net.preferIPv4Stack=true,then it work well whether in debug mode or not
        System.setProperty("java.net.preferIPv4Stack", "true");
        ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext(new String[]{"META-INF/spring/dubbo-hello-provider.xml"});
        context.start();

        System.in.read(); // press any key to exit
    }

}
```

### Consumer实现

其pom文件和Provider类似：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <parent>
        <artifactId>dubbo-hello</artifactId>
        <groupId>com.tc</groupId>
        <version>1.0-SNAPSHOT</version>
    </parent>
    <modelVersion>4.0.0</modelVersion>

    <artifactId>dubbo-hello-consumer</artifactId>
    <packaging>jar</packaging>
    <name>${project.artifactId}</name>
    <description>The demo consumer module of dubbo project</description>

    <dependencies>
        <dependency>
            <groupId>com.tc</groupId>
            <artifactId>dubbo-hello-api</artifactId>
            <version>${project.parent.version}</version>
        </dependency>

        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>dubbo</artifactId>
            <version>2.6.0</version>
        </dependency>

    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-assembly-plugin</artifactId>
                <version>2.3</version>
                <configuration>
                    <appendAssemblyId>false</appendAssemblyId>
                    <descriptorRefs>
                        <descriptorRef>jar-with-dependencies</descriptorRef>
                    </descriptorRefs>
                    <archive>
                        <manifest>
                            <mainClass>com.tc.dubbo.hello.consumer.Consumer</mainClass>
                        </manifest>
                    </archive>
                </configuration>
                <executions>
                    <execution>
                        <id>make-assembly</id>
                        <phase>package</phase>
                        <goals>
                            <goal>assembly</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>

</project>
```

其dubbo-hello-consumer.xml配置如下，声明了对服务的引用。   
```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:dubbo="http://code.alibabatech.com/schema/dubbo"
       xmlns="http://www.springframework.org/schema/beans"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-4.3.xsd
       http://code.alibabatech.com/schema/dubbo http://code.alibabatech.com/schema/dubbo/dubbo.xsd">

    <!-- consumer's application name, used for tracing dependency relationship (not a matching criterion),
    don't set it same as provider -->
    <dubbo:application name="hello-consumer"/>

    <!-- use multicast registry center to discover service -->
    <dubbo:registry address="multicast://224.5.6.7:1234"/>

    <!-- generate proxy for the remote service, then demoService can be used in the same way as the
    local regular interface -->
    <dubbo:reference id="helloService" check="false" interface="com.tc.dubbo.hello.HelloService"/>

</beans>
```

入口类Consumer实现如下：   

```java
package com.tc.dubbo.hello.consumer;

import com.tc.dubbo.hello.HelloService;
import org.springframework.context.support.ClassPathXmlApplicationContext;

/**
 * Created by tianchi on 2018/3/14.
 */
public class Consumer {

    public static void main(String[] args) {
        //Prevent to get IPV6 address,this way only work in debug mode
        //But you can pass use -Djava.net.preferIPv4Stack=true,then it work well whether in debug mode or not
        System.setProperty("java.net.preferIPv4Stack", "true");
        ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext(new String[]{"META-INF/spring/dubbo-hello-consumer.xml"});
        context.start();
        HelloService helloService = (HelloService) context.getBean("helloService"); // get remote service proxy

        while (true) {
            try {
                Thread.sleep(1000);
                String hello = helloService.sayHello("world"); // call remote method
                System.out.println(hello); // get result

            } catch (Throwable throwable) {
                throwable.printStackTrace();
            }
        }

    }
}
```

