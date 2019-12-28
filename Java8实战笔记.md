
## 函数式编程
### 函数式编程特点     

函数式编程的两个要点：   
1. 方法可以作为参数传递。   
2. 方法是无状态的（无副作用的）。

### 行为参数化   

行为参数化可以将行为代码作为参数传递给方法，从而增加方法的灵活性。    

行为参数化的方式有以下几种：  
1. 定义抽象行为接口，实现多种具体类，将类的实例作为参数。需要实现大量的实现类，有时候这些类只创建了一个实例。   
2. 定义接口，使用匿名内部类。代码不够清楚，不易理解。   
3. 定义接口，采用Lambda表达式。清楚、简洁。    
4. 定义接口，传入方法引用。   

### lambda表达式的书写规则    

Lambda表达式可以作为方法参数，也可以赋值给变量。    

Lambda表达式主要有两种形式：   
1. `(parameters)->expression`   
2. `(parameters)->{statements;}`   

一下Lambda表达式的使用都是正确的：   

```java  
()->{} //无参数，返回值为void
()->42
(Apple a1, Apple a2)->a1.getWeight()>a2.getWeight()
(int x)->{
    System.out.println(x);
    return x+1;
}
```

### 函数式接口   

函数式接口就是仅仅声明了一个方法的接口。这种接口便于用Lambda表达式进行表示。

如果接口有多个默认方法，仅有一个抽象方法，也为函数式接口。


### Lambda表达式使用外部局部变量   

Lambda表达式可以在方法中使用传入的参数是大家所熟知的，同时Lambda表达式也可以访问其上下文中的变量，如声明Lambda表达式的实例中的变量和静态变量，Lambda也可以访问其宿主方法体中的局部变量，但是这些局部变量必须是final的。因为局部变量是保存在线程的栈中，而作为参数的Lambda表达式在运行时可能会运行在其它线程中，所以Lambda表达式访问的局部变量其实是原始局部变量的副本，因为局部变量的不可修改，所以可以保证变量一致性，避免线程之间的不安全。同时局部变量也可能被所在线程提前回收，所以Lambda使用局部变量的副本。

这种使用外部作用域变量的lambda叫做捕获Lambda。   


### 方法引用   

方法应用的方式可以替代Lambda表达式来实现行为的参数化，并且便于重复的使用。方法引用有以下三种方式：   
1. `(args) -> ClassName.staticMethod(args)`等价于`ClassName::staticMethod`。   
2. `(arg0, rest) -> arg0.instanceMethod(rest)`等价于`ClassName::instanceMethod`。这里的Lambda表达式中第一个参数是ClassName类的一个实例，   
3. `(args)->expr.instanceMethod(args)`等价于`expr::instanceMethod`。这里expr并不在参数中。

构造函数也可以引用。例子如下：   
```java
//无参构造函数
Supplier<Apple> a=Apple::new;
//等同于
Supplier<Apple> a=()->new Apple();

//单个参数
Function<Integer, Apple> a= Apple::new;
//等同于
Function<Integer, Apple> a= (weight)->new Apple(weight);

//两个参数   
BiFunction<String, Integer, Apple> a=Apple::new;
//等同于
BiFunction<String, Integer, Apple> a= (color, weight)->new Apple(color, weight);
```

对于更多参数的情况，java没有提供现成的函数接口，可以自行构造。    

### 复合Lambda表达式    

得益于一些函数接口实现的默认方法，可以将多个Lambda表达式复合起来使用。

比较器复合：    
```java
Comparator<Apple> c=(a, b)-> a.getWeight().compareTo(b.getWeight()); 
//先逆序，再进一步比较颜色
appleList.sort(c.reversed().thenComparing(Apple:getColor))
```

谓词复合：   
```java
//可以用来判断是否为大于150的红苹果，或者为绿苹果
Predicate<Apple> rha= redApple.and(a->a.getWeight>150).or(a->"green".equals(a.getColor()));
```

redApple为Predicate的实例，由谓词组成的链为左结合。

函数复合：   
```java
Function<Integer, Integer> f= x->x+1;
Function<Integer, Integer> g= x*2;
//h(x)=g(f(x))
Function<Integer, Integer> h=f.andThen(g);
//t(x)=f(g(x))
Function<Integer, Integer> t=f.compose(g);
```

这些常用的函数接口提供的默认复合函数，特点是返回值仍为该函数接口的实例，其本质是将原来的实例按照需求重新包装进了一个新的实例中返回。

## 流   

### 什么是流    

流：从支持数据处理操作的源生成的元素序列。   

流只能被消费一次，类似迭代器。

流是对数据的计算，集合是对数据的存储和访问。流只获取需要的值，可以一段一段的获取，而集合是将数据都存储在内存中。流的迭代过程被封装在内部，而集合的迭代访问需要开发者自己实现。


### 流操作 

```java
List<String> names= menu.stream().filter(d->d.getCalories()>300)
                    .map(Dish::getName)
                    .limit(3)
                    .collect(toList());
```  

中间操作：filter、map、limit等。其实中间操作并不会真正的执行任何处理。返回一个流。   
终端操作：collect、foreach。合并中间操作，执行所有操作。不返回流。

filter和map的循环被合并为一个循环（循环合并），limit会执行短路操作，所以当发现三个满足条件的元素后执行会立即结束。

流操作更像是设计模式中的`Builder`。


### 流提供的操作   

筛选和分片：   
- filter：过滤，以Predicate作为参数。   
- distinct：去重。   
- limit：截断，保留前几个。   
- skip：跳过某几个。   

映射：   
- map：映射，以Function作为参数。   
- flagMap：流的扁平化。在使用map时可能产生流的流，也就是Stream<Stream<T>>，但这不是你想要的，所以可以用flagMap来避免这种问题，也就是将多个Stream合并为一个流。   

查找和匹配：   
- anyMatch：至少有一个元素匹配，终端操作，返回一个boolean型。   
- allMatch：所有的都匹配。   
- noneMatch：没有任何元素匹配。   
- findAny：找到任何一个，返回值为Optional<T>。   
- findFirst：找到第一个，和findAny类似，但是findAny没有顺序要求，更好并行化。    

规约：   
- reduce：将流规约成一个值。终端操作。

### 流操作的有、无状态   

大部分流操作是无状态的，比如filter，不用记录任何东西。   

有状态的操作如下：   
1. distinct：需要缓存所有的元素以去重，并且缓存的数据长度是无界的，所以处理过长的流可能会有问题。   
2. skip、limit：缓存元素的数量，只需简单计数，是有界的状态。   
3. sorted：缓存所有元素，无界。   
4. reduce：缓存一个中间结果，有界。


### 构建流   

除了通过集合生成流，还可以通过如下方式产生流：   
```java
//用元素直接生成流
Stream stream1= Stream.of("java", "lambda", "action");

//迭代生成无限的数据流，生成偶数序列  
Stream stream2= Stream.iterate(0, n->n+2);

//生成随机序列   
Stram stream3= Stream.generate(Math:random);
```

### 收集器    

Stream流操作的最后一步可以通过收集器来汇总结果，如：`stream.collect(toList())`。其中`collect`操作是一个终端操作，参数是一个收集器（collector）。Java 8提供了一些常用的收集器工厂方法：   
1. toList()。将结果收集为一个列表。   
2. counting()。统计数目。   
3. maxBy(Comparator<T> c)。最大值。   
4. summingInt(ToIntFunction<? super T> mapper)。汇总int类型的值。  
5. joining()。连接字符串。 
6. groupingBy(Function<? super T, ? extends K> classifier)。分组，生成一个以分组保准为key的map。支持嵌套，实现多级分组。     
7. partitioningBy(Predicate<? super T)。二分，groupingBy的特例。

### 实现自己的收集器   

实现定制的收集器需要实现接口：   
```java
public interface Collector<T, A, R> {
    /**
     * A function that creates and returns a new mutable result container.
     * 提供一个中间结果的容器
     * @return a function which returns a new, mutable result container
     */
    Supplier<A> supplier();

    /**
     * A function that folds a value into a mutable result container.
     * 将流中的项集成进中间结果容器
     * @return a function which folds a value into a mutable result container
     */
    BiConsumer<A, T> accumulator();

    /**
     * A function that accepts two partial results and merges them.  The
     * combiner function may fold state from one argument into the other and
     * return that, or may return a new result container.
     * 将多个中间结果合并为一个，主要在并行时使用
     * @return a function which combines two partial results into a combined
     * result
     */
    BinaryOperator<A> combiner();

    /**
     * Perform the final transformation from the intermediate accumulation type
     * {@code A} to the final result type {@code R}.
     *
     * <p>If the characteristic {@code IDENTITY_TRANSFORM} is
     * set, this function may be presumed to be an identity transform with an
     * unchecked cast from {@code A} to {@code R}.
     * 将中间结果转化为目标结果
     * @return a function which transforms the intermediate result to the final
     * result
     */
    Function<A, R> finisher();

    /**
     * Returns a {@code Set} of {@code Collector.Characteristics} indicating
     * the characteristics of this Collector.  This set should be immutable.
     * 描述该收集器的特征：无序的、可并行的、结果一致的（中间结果即为最后结果）
     * @return an immutable set of collector characteristics
     */
    Set<Characteristics> characteristics();
}
```

下面是实现收集质数的样例collector：   
```java
import java.util.*;
import java.util.function.BiConsumer;
import java.util.function.BinaryOperator;
import java.util.function.Function;
import java.util.function.Supplier;
import java.util.stream.Collector;

/**
 * Created by tianchi on 2018/6/19.
 *
 * 收集质数
 */
public class PrimeNumbersCollector implements Collector<Integer, Map<Boolean, List<Integer>>, Map<Boolean, List<Integer>>> {

    @Override
    public Supplier<Map<Boolean, List<Integer>>> supplier() {
        //用map保存中间结果，质数和非质数分别放入两个list
        return () -> new HashMap<Boolean, List<Integer>>() {
            {
                put(true, new ArrayList<Integer>());
                put(false, new ArrayList<Integer>());
            }
        };
    }

    @Override
    public BiConsumer<Map<Boolean, List<Integer>>, Integer> accumulator() {
        return (Map<Boolean, List<Integer>> acc, Integer candidate) -> {
            //判断数据流中的元素是否为质数，并且放入相应的list
            acc.get(isPrime(candidate)).add(candidate);
        };
    }

    @Override
    public BinaryOperator<Map<Boolean, List<Integer>>> combiner() {
        //合并两个map
        return (Map<Boolean, List<Integer>> map1, Map<Boolean, List<Integer>> map2) -> {
            map1.get(true).addAll(map2.get(true));
            map1.get(false).addAll(map2.get(false));
            return map1;
        };
    }

    @Override
    public Function<Map<Boolean, List<Integer>>, Map<Boolean, List<Integer>>> finisher() {
        //结果不变
        return Function.identity();
    }

    @Override
    public Set<Characteristics> characteristics() {
        //该collector中间结果和最后结果相一致，可以省略finisher这一步。
        return Collections.unmodifiableSet(EnumSet.of(Characteristics.IDENTITY_FINISH));
    }
}

```

### 并行流的使用建议  

1. 一定要测量，实践是检验真理的唯一标准。并行后的实际效果有时候会违反直觉。    
2. 注意自动装箱和拆箱导致的消耗。   
3. 有些操作不适合采用并行，比如limit和findFirst这些依赖于顺序的操作。    
4. 考虑数据量和计算成本，小数据量采用并行可能带来更多的损耗，对每个元素的计算比较耗时的操作适合采用并行化。   
5. 流背后的数据结构是否容易分解。ArrayList可以快速分解，而LinkedList需要遍历才能分解。  
6. 注意最后合并流时是否需要很大代价。

### 分支/合并框架   

并行流实现的背后原理是Fork/Join框架的使用。通过实现RecursiveTask将任务递归的划分，子任务被分配到新的线程中执行，最后合并子任务的结果。    

### 自定义划分方式   

可以自己实现Spliterator来定制特殊的划分方法，具体如下：   
```java
public interface Spliterator<T> {
    /**
     * 处理下一元素
     *
     * @param action The action
     * @return {@code false} if no remaining elements existed
     * upon entry to this method, else {@code true}.
     * @throws NullPointerException if the specified action is null
     */
    boolean tryAdvance(Consumer<? super T> action);
    /**
     * 进行分裂，自定义停止分裂的阈值，返回一个新的Spliterator，剩余的元素由原
     * Spliterator继续处理
     * @return a {@code Spliterator} covering some portion of the
     * elements, or {@code null} if this spliterator cannot be split
     */
    Spliterator<T> trySplit();

    //返回一个大致数量
    long estimateSize();

    //返回该分裂器的特征
    int characteristics();
```

## 高效Java 8编程   

### 匿名内部类和Lambda表达式的关系   

Lambda表达式大多数情况下可以替换匿名内部类，特别是函数接口的匿名类。但是，Lambda表达式不完全等价于一个匿名内部类。   

匿名内部类中，this表示该匿名内部类对象，而Lambda中则是声明该表达式的对象。匿名内部类的局部变量不会和外部类冲突，而Lambda表达式中的变量不能与外部重复。  

匿名内部类声明的对象，类型是确定的，所以作为参数时可以明确的表示它的类型。而Lambda表达式则可能同时匹配多个目标类型，导致混淆。

### 接口和抽象类   

Java8中，接口可以提供默认的方法实现，所以在形式上和抽象类越来越相近，但两者还是存在区别。一个类只能继承一个抽象类，而可以实现多个接口。抽象类可以有实例变量，而接口没有。  

### 多继承冲突的解决   

在实现多个接口或继承某个类时，可能出现一个方法有多个来源的情况，这时候需判断优先选用哪个来源的方法。解决规则如下： 
1. 类或父类中声明的方法（包括抽象方法）的优先级高于接口中的方法。如果父类中没有实现该方法，而是从某个接口继承而来，则根据第2条继续判断。   
2. 上条无法判断，则子接口（最具体）的声明的方法（包括没具体实现的方法）优先高。   
3. 如果仍然无法判断，则应该显示的指明调用的哪个来源的方法。

### 新的时间和日期API   

Java8提供了新的时间、日期API，使用也更为方便。需要注意各种时间表示和时区的关系。这里对时间对象的修改都是返回一个新的时间对象，而不改变旧的对象。

```javaimport java.time.*;

/**
 * Created by tianchi on 2018/6/26.
 */
public class Main {
    public static void main(String[] args) {
        //获取本机时间，也就是计算机设定的当前时间，此处为北京时间
        LocalDateTime dateTime = LocalDateTime.now();
        System.out.println(dateTime);

        ZoneId zd = ZoneId.of("Africa/Harare");
        //设定时区，强行设置为非洲，并不会自动调整为正确的当地时间
        ZonedDateTime zDateTime = dateTime.atZone(zd);
        System.out.println(zDateTime);
        //设定为上海
        zDateTime = dateTime.atZone(ZoneId.of("Asia/Shanghai"));
        System.out.println(zDateTime);

        //只看日期
        LocalDate date = LocalDate.now();
        zDateTime = date.atStartOfDay(zd);
        System.out.println(zDateTime);
        
        //机器时间，从1970年1月1日凌晨开始
        Instant instant=Instant.now();
        //这个输出的就是格林尼治时间
        System.out.println(instant);
        //调整时区，时间也变为当地时间
        zDateTime = instant.atZone(zd);
        System.out.println(zDateTime);

    }
}

```

代码输出的结果。   
```
2018-06-28T18:51:21.692
2018-06-28T18:51:21.692+02:00[Africa/Harare]
2018-06-28T18:51:21.692+08:00[Asia/Shanghai]
2018-06-28T00:00+02:00[Africa/Harare]
2018-06-28T10:51:21.694Z
2018-06-28T12:51:21.694+02:00[Africa/Harare]
```
