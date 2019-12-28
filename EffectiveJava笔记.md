## 对所有对象都通用的方法

### equals和hashCode方法的关系

1. 重写equals方法必须也要重写hashCode方法。   
2. equals用的属性没变，则多次调用hashCode返回值也必须保持不变。  
3. equals比较相等的对象，hashCode也必须相同。反之不然。   
4. 所处相同hash bucket的对象，hashCode可能不同，因为在求解bucket位置时会对hashCode进行截断，根据bucket大小采用后段值。

### clone方法的设计原则

1. Cloneable接口并不提供接口方法clone，clone是Object类实现的基本方法。   
2. 实现Cloneable接口的类应该提供一个public的clone函数覆盖原来protect的方法。  
3. clone方法首先调用super的clone方法，然后再处理需要深层拷贝的内部属性。   
4. 道行不深不要使用该接口。

### Comparable接口的设计原则   

1. 传递（a>b,b>c则a>c），对称（a>b则b<a）。   
2. 最好compareTo方法和equals方法保持一致。   

## 类和接口   

### 访问控制   

1. 顶层类（非嵌套）和接口只有两种访问级别，包私有和公有的。声明为public则类为公有的，无声明则默认包私有。   
2. 成员域、方法、嵌套类和嵌套接口有private、包私有（默认）、protected和public四种级别。可访问性依次增强。    
3. 子类在重写父类方法时，可访问性只能保持不变或者增强。因为通过超类的引用也可以正常的使用子类实例。   
4. 实例域一定不要设置为public。否则，会失去对该域的控制权。

### 公有类的暴露公有域    

公用类如果暴露了自己的域，则会导致客户端滥用该域，并且该公用类在以后的升级中无法灵活的改变属性的表达方式。

### 不可变对象的设计    

1. String、基本类型的包装类是不可变对象。     
2. 设计不可变类应该遵循以下准则：    
    1. 不要提供任何修改对象状态的方法。    
    2. 保证类不会被扩展。一般类可以加final。   
    3. 所有的域都是final。    
    4. 所有的域都是私有的。   
    5. 保证所有可变域无法被客户端获得。  
3. 不可变对象线程安全，可以被自由的共享。不可变类不应该提供clone和拷贝构造器，直接共享最好，但是String还是提供了拷贝构造器。   
4. 不可变类也有一定的劣势，因为一个操作可能涉及多个临时的不可变类，而导致大量对象的创建和销毁，所以此时应该采用不可变类配套的可变类。如String类对应的StringBuilder。   
5. 应该提供尽可能小的可变状态。

### 复合优先于继承    

1. 继承会破坏封装性，实现继承需要对父类的实现进行充分的了解。所以父类和子类应该实现在同一个包内，由同一个开发团队维护。  
2. 一个类在设计时应该明确指明是不是为了可被继承而设计的。    
3. 一个类如果没有考虑自己可能被继承，有些方法可能会被重写，则其内部调用这些可被重写方法的方法就可能会出现意想不到的异常行为。继承该方法的子类会调用父类的内部方法，而父类内部方法的更新可能会导致子类的异常。


### 接口优于抽象类    

1. 现有的类可以很容易的加入新的接口，因为接口可以实现多个。   
2. 类只允许有一个父类，如果用抽象类描述共性，则需要该共性的类必须为该抽象类的后代，即使这些子类并没有很显然的关系。    
3. 接口可以让我们实现非层次结构的类框架。如果采用抽象类，则属性组合可能导致子类的组合爆炸。       
4. 接口的缺点是扩展新方法时，所有实现该接口的类都要重新添加该方法，而抽象类可以提供默认实现。不过，现在Java8提供了default描述默认实现方法，似乎这种弊端可以避免。

### 接口中定义属性    

1. 接口中定义的属性默认为final static。    
2. 最好不要用接口来定义属性，因为实现该接口的类会引入这些属性，造成类的命名空间污染。接口应该用来描述类可以执行的动作。   

### 内部类的设计    

- 静态成员类：用static修饰的内部类，可以不依赖于外部实例进行创建。   
- 非静态成员类：创建时依赖于外部类的实例，并且创建后与外部实例绑定。无静态属性。   
- 匿名内部类：作为参数，只会被实例化一次。和非静态成员类似。在非静态环境中，会与外部类的实例绑定。   
- 局部类：声明局部变量的地方可以声明局部类，它有名字，可以在局部被多次实例化。在非静态环境中，会与外部类的实例绑定。   

## 泛型   

### 不要在代码中使用原生类型   

如下代码使用原生类型：   
```java
    ArrayList a=new ArrayList<String>();
    a.add(new Object());
```
以上代码编译和运行都可以通过，但是埋下了很多隐患。

List<String>是List的子类，而不是List<Object>的子类。List这种原生类型逃避了类型检查。

List中add任何对象都对。List<?>的变量可以引用任何参数化(非参数也可以)的List，但是无法通过该变量添加非null元素。

假设`Men extends Person, Boy extends Men`:    
1. `<? extends T>`表示上界，`<？ super T>`表示下界。   
2.  `ArrayList<? extends Men> ml=new ArrayList<Boy>();`，等号右边部分可以是Men与其子类的参数化ArrayList，或者是ArrayList<>()和ArrayList()。初始化时可以在`new ArrayList<Boy>()`中填入Boy对象，此后不能再往ml里存元素，从ml的视角，ml可能是ArrayList<Men>、ArrayList<Boy>等等，存入任何Men或者其子类对象都不合适，为了安全起见都不允许存。只能取元素，并且取出的元素只能赋值给Men或者其基类的引用，因为其中元素可能存了任何Men的子类，为了保险起见取出的值用Men或其基类表示。   
3. `ArrayList<? super Men> ml=new ArrayList<Person>();`，初始化时可以存Person对象，之后可以再存入Men（与其子类，这是默认的），但是再存入Person对象是错误的。从ml视角，等号右边可以是ArrayList<Person>()、ArrayList<Men>()等等，所以最高只能存入Men对象。取出的元素都是Object，因为等号右边可以是ArrayList<Object>()。    
4. 总结2和3条，可知 `<? extends T>`和`<？ super T>`是对等号右边实参数化ArrayList的限制，而不是对ArrayList中可存入元素的描述。因为从引用ml中无法得知其实际指向的是那种参数化的ArrayList实例，所以再往其中添加元素时会采用最谨慎的选择。

### 列表和数组的区别     

数组是协变的，也就是`Fruit[] fs= new Apple[5];`是合法的，因为Apple是Fruit的子类，则数组也成父子关系，而列表则不适用于该规则。数组的这种关系容易引发错误，如`fs[0]= new Banana()`，编译时没错，这在运行时报错。   

创建泛型数组是非法的，如`new E[]; new List<E>[]; new List<String>[] `。泛型参数在运行时会被擦除，List<String>[]数组可能存入List<Integer>对象，因为运行时两者都是List对象，这显然是错误的，所以泛型不允许用在数组中。   

如下代码在编译时不会出错，在运行时出错`java.lang.ClassCastException`。
```java
ArrayList<String> list=new ArrayList<String>();  
        for (int i = 0; i < 10; i++) {  
            list.add(""+i);  
        }
        //该行报错       
        String[] array= (String[]) list.toArray();  
}    
```
原因很迷，toArray返回的是Object[]数组，但是不能强制转化为String[]，明明元素实际类型是String。有的解释说，在运行时只有List的概念，而没有List<String>概念。我感觉事情没这么简单。


### 泛型和泛型方法   

泛型是在整个类上采用泛型，这样可以在类内部方便的使用泛型参数。泛型方法是更精细的利用参数类型，将泛型参数设定在每个方法上。

比较下面两个接口，体会其中不同：   

```java
public interface Comparable<T> {
    public int compareTo(T o);
}

public interface Comparable2 {
    public <T> int compareTo2(T o);
}

public class Apple implements Comparable<Apple>, Comparable2{
    @Override
    public int compareTo(Apple o) {
        return 0;
    }

    @Override
    public <T> int compareTo2(T o) {
        //T 可以为任何类，所以Apple可以和任何类比较
        return 0;
    }
}
```

### 递归类型限制   

有类：   

```java
class Apple implements Comparable<Apple>{

}

class RedApple extends Apple{

}
```

有方法：   

```java
public static  <T extends Comparable<T>> T get(T t){
    return t;
}
```

该方法就采用了递归的类型限制，因为泛型T被限制为 Comparable<T>的子类，而Comparable<T>中又包含了T，这形成一种递归的类型限制。Apple类可以调用该函数，RedApple则会出现错误，如下所示。  

```java
RedApple ra=new RedApple();
Apple a= get(ra); //正确   
RedApple b=get(ra); //错误
```
原因是在调用泛型函数时，会自动进行类型推断，第一个get函数根据左边参数，推断T为Apple，符合条件。在第二个get公式中，推断T为RedApple，不符合get函数的泛型限制条件。

### 一个比较复杂的泛型函数

```java
public static <T extends Comparable<? super T>> T max(List<? extends T> list)
```

其中`<T extends Comparable<? super T>>`描述了T实现了Comparable接口或者其基类实现了该接口，通过继承获得Comparable的情况比较常见，这增加了该函数的通用性。参数`List<? extends T>`表示List只要存的是T的子类就可以，这是显然合理的，同样增强了该函数的通用性。

### 异构容器   

一个容器如Set只有1个类型参数，Map只有2个类型参数，但是有时候需要多个类型参数。下面是一个设计巧妙、可以容纳多个类型参数的类。   

```java
public static void main(String[] args){
    Favorites f =new Favorites();
    f.putFavorite(String.class, "Java");
    f.putFavorite(Integer.class, 1111);
    f.putFavorite(Class.class, Favorites.class);
    int fi=f.getFavorite(Integer.class);
}

public class Favorites{
    private Map<Class<?>, Object> favorites=new HashMap<Class<?>, Object>();

    public <T> void putFavorite(Class<T> type, T instance){
        if(type==null)
            throw new NullPointerException("Type is null");
        favorites.put(type, instance);
    }

    public <T> T getFavorite(Class<T> type){
        return type.cast(favorites.get(type));
    }
}
```

String.class为Class<String>的实例，Integer.class为Class<Integer>的实例，这两者显然不是同一个类，但是却可以放在同一个Map中。Map中采用了通配符`?`，按理Map无法再加入任何元素，但是该通配符并不是直接表示Map的类型参数，而是Class<?>。因此Map的键值可以是Class<String>、Class<Integer>等不同的类型，因此成为一个异构容器。   

其中Map实例favorites并没有限制value一定是key描述的类的实例，而方法putFavorite通过类型参数T，巧妙的限制了两者的关系。


## 枚举和注解   

### 枚举类型举例   

枚举类型更像是一个不能new的类，只能在定义时就实例化好需要的固定数目的实例。如下所示：    
```java
public enum Planet {
    VENUS(2),
    EARTH(3),
    MARS(5);

    int data;

    Planet(int i){
        this.data=i;
    }

    public int getData(){
        return data;
    }
}
```
其构造函数默认是private，并且无法修改。在枚举中还可以定义抽象方法，如下：   

```java
public enum Operation{
    PLUS { double apply(double x, double y){return x+y;}},
    MINUS { double apply(double x, double y){return x-y}};

    abstract double apply(double x, double y);
}
```

### 自定义注解的例子   

定义一个用来测试方法是否能抛出目标异常的注解。   
```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface ExceptionTest{
    Class<? extends Exception>[] value();
}
```   
元注解指明了该注解在运行时保留，并且只适用于注解方法。   

使用如下：   
```java
@ExpectionTest({IndexOutOfBoundsException.class, NullPointerException.class})
public static void doublyBad(){
    List<String> list=new ArrayList<String>();
    //该方法会抛出IndexOutOfBoundsException
    list.addAll(5,null);
}
```   

测试过程实现如下：   
```java
public static void main(String[] args) throws Exception{
    int tests=0;
    int passed=0;
    Class testClass=Class.forName(args[0]);
    for(Method m : testClass.getDeclaredMethods()){
        if(m.isAnnotationPresent(ExceptionTest.class)){
            tests++;
            try{
                m.invoke(null);
                System.out.printf("Test failed: no exceptions");
            }catch( Throwable wrappedExc){
                Throwable exc = wrappedExc.getCause();
                Class<? extends Exception>[] excTypes=m.getAnnotation(ExceptionText.class).value();
                int oldPaassed=passed;
                for(Class<? extends Exception> excType:excTypes){
                    if(excType.isInstance(exc)){
                        passed++;
                        break;
                    }
                }

                if(passed==oldPassed)
                    System.out.printf("Test failed");
            }
        }
    }
}
```

## 方法   

### 必要时进行保护性拷贝   

1. 构造函数在接受外来参数时，必要时需要拷贝参数对象，而不是直接将参数对象赋值给自己的属性。因为直接采用外部传入的对象，外部可以任意的修改这些对象，从而导致你自己设计的类内部属性变化。如果不想让使用你设计的类的客户有修改其内部属性的权利，除了设置为private外，还应该注意采用拷贝的方式使用外部传入的数据。选择copy而不是clone，是因为传入对象可能是客户定制过的参数子类，该子类仍然可能将其暴露在外面。    

2. 需要保护内部属性不被修改，除了关注构造函数的参数，还需要关注get类似的方法。这些返回内部属性的方法，应该返回拷贝过的属性对象。


### 慎用重载   

重载方法是静态的，在编译时就已经选择好，根据参数的表面类型，如`Collection<String> c=new ArrayList<String>()`，有两个重载函数，`getMax(Collection<?> a)`和`getMax(ArrayList<?> a)`，在调用`getMax(c)`时，会选择`getMax(Collection<?> a)`方法，该选择在编译时就决定好了。当重载方法有多个参数时，情况会变得更复杂，选择结果可能出人意料。   

而方法的重写选择时动态的，在运行时根据调用者的实际类型决定哪个方法被调用。

### 慎用可变参数  

可变参数可以让用户灵活的填入不同数量的参数，但是该方法本质上是将参数组织成数组，所以每次调用这些方法时都会涉及数组的创建和销毁，开销较大。

## 并发

### 同步的意义

1. 保证数据从一个一致状态转一到另一个一致状态，任何时候读取该数据都是一致的。   
2. 保证对数据的修改，其它线程立即可见。

### 读写变量是原子性的   

除了double和long以外，读写变量是原子性的。但是Java无法保证一个线程的修改对另一个线程是可见的。

### 在同步模块中小心调用其它方法   

如果一个同步方法在其中调用了一个不由自己控制的方法，比如客户传入的方法，客户可能在实现方法时申请同步锁，或者启动新线程申请锁，这可能会导致死锁。

### 并发工具优先于wait和notify

java.util.concurrent包提供了执行框架、并发集合和同步器三种工具，应该尽量使用这些工具来实现并发功能，而不是使用wait、notify。   

如果使用wait，notify则应该采用如下模式：   
```java
    public void waitA(){
        synchronized(a){    //获得锁
            while(a>10)     //放在while循环中保证满足条件
                try {
                    a.wait(); //释放锁、如果被唤醒则需要重新获得锁
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
        }
    }

    //其它线程调用该方法唤醒等待线程
    public void notifyA(){
        a.notifyAll();
    }
```

notifyAll方法相较于notify方法更安全，它保证唤醒了所有等待a对象的线程，被唤醒不代表会被立即执行，因为还需要获得锁。

### 不要对线程调度器有任何期望

Tread yield（让步）即当前线程将资源归还给调度器，但是并不能保证当前线程下面一定不会被选中。线程的优先级设置也是不能保证按你预期的进行调度。

