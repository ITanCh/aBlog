
## HashTable和HashMap的区别   

HashTable和HashMap的区别是，HashTable在操作table的方法上加入synchronized关键字，使得线程安全。实现方式和HashMap类似，但是HashTable没有处理在某个槽下的值太多，链表过长的情况。


## TreeMap和HashMap的区别   

有序，二叉搜索树，使用红黑树。使用key进行比较，或者传入的比较器。方法containsKey、get、put和remove的时间复杂度是log(n)。


## LinkedHashMap与HashMap的区别   

LinkedHashMap保存了元素的插入顺序，在遍历时会遵循插入的顺序。而HashMap遍历时，顺序是按照table的顺序，依次遍历每一个槽中的链表，所以顺序和插入顺序完全不同。    

LinkedHashMap在Node上加上了`before`和`after`属性用以构建保持原顺序的双向链表。   

LinkeHashMap可以设置参数，使其从插入顺序变为访问顺序。   

LinkedHashMap基于HashMap，它自己的任务主要是维护保持顺序的双向链表。   

## ConcurrentHashMap

### ConcurrentHashMap和HashMap的区别

java.util.concurrent;

ConcurrentHashMap提供了一个高效的、线程安全的访问和更新HashMap的方式。较之HashTable，它没有提供在整个table上的锁，同步方式粒度更细。

### 如何实现并发访问的安全   

看网上资料，旧版本的ConcurrentHashMap采用了分段的策略进行同步。Hash步骤如下：
0. ConcurrentHashMap维护了一个segment数组，segment是一个锁。   
1. 将key先hash到segment位置，每个segment存储了一个真正的table。    
2. 再次hash，找到在table中的位置。      
3. 在table中的过程和HashMap类似。  

所有并发访问ConcurrentHashMap的线程会在每个segment上同步，所以线程可以并行的访问不同的segment，减小了同步的粒度。    

但是，在1.8中，使用的是`sun.misc.Unsafe`工具。它的作用有：   
1. 对变量和数组内容的原子访问，自定义内存屏障   
2. 对序列化的支持
3. 自定义内存管理/高效的内存布局   
4. 与原生代码和其他JVM进行互操作   
5. 对高级锁的支持   

ConcurrentHashMap对table的操作全部由sun.misc.Unsafe来完成：   

```java
    static final <K,V> Node<K,V> tabAt(Node<K,V>[] tab, int i) {
        return (Node<K,V>)U.getObjectVolatile(tab, ((long)i << ASHIFT) + ABASE);
    }

    static final <K,V> boolean casTabAt(Node<K,V>[] tab, int i,
                                        Node<K,V> c, Node<K,V> v) {
        //c表示旧值
        //v表示新值
        //更新前会验证旧值是否变化，如果变化说明期间被其它线程修改，则修改失败
        //外部使用该方法一般会使用一个循环，以多次重试
        return U.compareAndSwapObject(tab, ((long)i << ASHIFT) + ABASE, c, v);
    }

    static final <K,V> void setTabAt(Node<K,V>[] tab, int i, Node<K,V> v) {
        U.putObjectVolatile(tab, ((long)i << ASHIFT) + ABASE, v);
    }
```

可以简单的理解为上述方法是原子性的，对目标数据所在的内存位置进行了加锁，整个操作过程中，对该部分内存的读、写不会受其它线程的影响。可见，该方法的并发实现粒度更细。  

还有一点需要注意，在并发操作时，可能会出现有的线程开始扩张或缩小table，但是有的线程却试图操作table的情况。因为搬迁table上元素的过程比较耗时，所以，其它线程发现该table正在重建，会先将操作table的事情往后放一放，而是转头去帮助搬迁table，等table搬迁完毕再继续自己的活。   