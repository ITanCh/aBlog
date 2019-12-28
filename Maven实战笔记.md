
## 坐标和依赖    

### 依赖冲突的调节    

当包的依赖产生冲突，如A->B->X(1.0)和A->D-X(2.0)，应该引入X的哪一个版本？消解冲突的法则如下：    
1. 路径最近者优先。    
2. 如路径长度一样，第一声明者优先。   

### 排除不想要的依赖    

在引入第三方库时，会自动的引入它们的依赖，有时候传递的依赖并不是我们想要的，可以用`exclusion`标签排除不想要的库，并且自己在依赖中直接引入想要的库。

```xml
    <dependencies>
        <dependency>
            <groupId>com.tc</groupId>
            <artifactId>a</artifactId>
            <version>1.0.0</version>
            <exclusions>
                <exclusion>
                    <groupId>com.tt</groupId>
                    <artifactId>n</artifactId>
                </exclusion>
            </exclusions>
        </dependency>

        <dependency>
            <groupId>com.tt</groupId>
            <artifactId>n</artifactId>
            <version>4.0</version>
        </dependency>
    </dependencies>       
```

### 使用未声明的依赖    

在项目中，可以不在pom中直接声明你需要的依赖，因为在其它引入的库中可能已经包含了这个依赖。但是，这种行为是危险的，所以最佳实践应该是显示声明任何项目中直接用到的依赖。    

### SNAPSHOT的作用    

依赖模块的快照版本，会保持该模块的定期更新，因为在发布snapshot的模块时，仓库会为其打上时间戳。在用户使用该模块时，无需更改pom中模块的版本号，maven会自动的向仓库对比时间戳是否变化，然后现在最新的版本。

SNAPSHOT版本应该只在组织内部模块之间调试和使用，在真正发布项目时，应该保证所有的依赖的模块都是发布版本，否则可能因为SNAPSHOT版本的不断变化而引入bug。

## 生命周期和插件   

### Maven的生命周期   

clean生命周期：清理项目。   
default生命周期：构建的主要步骤，如compile，是核心部分。   
site生命周期：建立和发布站点，分享项目信息。 

不同的生命周期又有多个阶段，比如clean有pre-clean、clean和post-clean的阶段。   

生命周期互相独立，但是步骤之间会有依赖关系，比如default周期中的test就依赖于test-compile等步骤。    

## 插件目标与阶段之间的绑定    

生命周期有多个阶段，一个插件也有多个目标。比如`maven-compiler-plugin`插件有`compile`、`testCompile`等目标。    

阶段会和插件目标绑定来执行自己的功能。比如default周期的complie阶段会和`maven-compiler-plugin:compile`插件目标绑定。

## Maven的聚合和继承   

多个模块可以放在一个总项目下，然后通过在总项目下一次构建所有的子模块。在总模块pom中加入如下元素：    

```xml
<modules>
    <module>A</module>
    <module>B</module>
</modules>
```   

子项目可以继承父项目的pom中配置的参数，如依赖、插件配置等。可以避免重复，还可以统一配置的依赖、插件版本。子项目需要在pom中添加如下部分：    

```xml   
<parent>
    <groupId>parent.group</groupId>
    <artifactId>parent.artifact</artifactId>
    <version>parent.version</version>
</parent> 
```

聚合和继承在maven中是两个不同的概念，但是，可以统一在一个pom文件中。    



