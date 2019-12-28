## IoC容器的实现 

控制反转（Inversion of Control，缩写为IoC），是面向对象编程中的一种设计原则，可以用来减低计算机代码之间的耦合度。

### 依赖反转是什么

依赖反转在Spring中的体现是依赖注入。   

类通过引用来进行合作，这种引用形成了类之间的依赖，如果这种依赖关系需要对象自己管理，那么代码会产生高度的耦合，导致开发和测试上的困难。如果这种依赖的管理交给框架来做，将简化面向对象系统的复杂性，这就是依赖的反转。   

### Spring中的容器是什么

容器用来管理应用中的对象和其之间的依赖关系。**BeanFactory**是对容器的一种抽象，ApplicationContext是其高级实现。  

### IoC的初始化过程    

整体过程分三步：   
1. Resourece定位，也就是定位定义Bean资源的位置。Resourece的位置是多样的，使用比较多的是FileSystemResourece。   
2. BeanDefinition载入。读取定义的内容，将用户定义的Bean表示成BeanDefinition，BeanDefinition是在容器中对POJO的抽象，记录了配置的bean信息。

3. 向IoC容器注册BeanDefinition，IoC用HashMap结构来维护注册的BeanDefinition。

注意这里只是有Bean的载入，而不是依赖注入，依赖注入发生在第一次通过`getBean`向容器获取Bean的时候。

### 依赖注入过程   

在发起`getBean`方法时，才会进行依赖注入，创建响应的bean。   

依赖注入过程如下：  
1. 判断单例bean是否已经创建，创建过的无需重复创建。   
2. 递归创建依赖的bean。    
3. 最后创建目标bean实例。   
    - 实例化Java对象。采用了CGLIB或者JVM反射机制来生成对象。CGLIB可以在运行期动态的生成新的class。
    - 注入属性依赖。   

### Bean的生命周期   

在IoC初始化后，Bean并没有被实际的创建。在`getBean`后，才会有bean的实例化，bean的生命周期如下：   

创建bean实例->设置bean的属性->调用bean的初始化方法（initialization）->应用可以使用bean->容器关闭，销毁bean

### Bean的lazy init   

上述可知，bean会在使用时才会真正的创建，这防止了创建大量用不到的bean实例。

当然，可以通过设置参数，让bean在IoC容器初始化完毕后就创建。   

### IoC感知的Bean   

Bean一般情况下不需要知道IoC容器的存在，有时候则需要。Bean可以实现一些aware接口来获得想要的容器属性。   

如ApplicationContextAware，可以在Bean中获得上下文，从而在Bean中使用上下文。

## AOP的实现   

### AOP的作用   

分离关注点使解决特定领域问题的代码从业务逻辑中独立出来。

### 基本概念   

**Advice通知**：描述方法调用注入的切面行为。    
**Pointcut切点**：用来描述需要增强的方法集合。    
**Advisor通知器**：将Advice和Pointcut结合起来。 

### JVM的动态代理   

在代理模式中，会设计一个和目标对象有着一致接口的代理对象Proxy，客户端对目标对象的请求都会发送给代理对象，而客户端对此毫无察觉。

![](pic/Proxy_pattern_diagram.png)

该模式使得Proxy有机会对原始的对象的行为进行一定的修改，可以在执行前和执行后分别执行一定的动作，从而增强原始行为。

### AopProxy的实现原理

首先需要定义一些Advisor，这些Advisor定义了需要织入的增强功能，也就是涵盖了通知的内容。然后定义ProxyFactoryBean，它将会生成目标的代理对象。在配置中，ProxyFactoryBean需要知道其代理的目标是谁，代理的接口是哪个，有哪些Advisor需要添加。

ProxyFactoryBean生成AopProxy过程：   
1. 初始化通知器链。通过getBean获取通知器的bean，然后加入。ProxyFactoryBean只在第一次获得AopProxy时初始化。    
2. 生成代理对象。
    - JDK生成，需要设置ClassLoader和代理接口。
    - CGLIB生成。   

AopProxy工作过程：   
1. 当目标对象被调用时，会触发Proxy的回调函数进行拦截。   
2. 首先获取拦截器。从上述中初始化的通知器链中，遍历并获取匹配目标方法的通知器，然后获取通知器对应的拦截器，该结果会被缓存。
3. 调用器会依次迭代调用拦截器（interceptor）进行增强，最后调用目标方法。   


通知器封装为拦截器：   
1. 拦截器默认分为三类：MethodBeforeAdviceIntercepter、AfterReturn...、Throws...。   
2. 一个Advice会可能是MethodBeforeAdvice、AfterReturn...、Throws...其中的一个或者多个。   
3. 根据Advice的种类，适配器AdviceAdapter将其包装为相应的intercepter，intercepter中的`invoke`方法会根据before、after或者throw将advice的增强行为放置到适当位置。适配器默认是上述的三种，可以自定义adapter注册进来，以生成自己定制的intercepter。    
4. 调用器执行拦截器链的时候，递归过程：process（调用器）->invoke(intercepter)->process（调用器）->invoke(下一个intercepter)...->目标方法。所以intercepter会根据自己的种类在递归调用前或者后执行advice的方法。
