
java.util.Set

HashSet继承自AbstractSet，继承接口Set，Set接口继承自Collection。

## null是否可以放在Set中

null可以放在Set中，并且最多只能有一个。null的hashcode是0。

## HashSet用什么结构实现的

HashSet用了一个HashMap。。。该HashMap默认的负载因子是0.75。

需要存入的对象作为HashMap的key存入，而value使用了一个公共静态的Object`PRESENT`来填充。

HashSet的iterator也是使用的HashMap的key set的iterator。

HashSet中add、remove和contains方法理论上是**常数时间**的复杂度，前提是元素在bucket中分散比较均匀。迭代访问其中元素的时间，则是元素数量+bucket容量，所以在迭代访问操作比较重要时，不要将set的容量初始为很大。

## HashSet不是线程安全的

将其变为一个线程安全set的官方方法是：

```java
Set s = Collections.synchronizedSet(new HashSet(...)
```

## TreeSet是用什么结构实现的

不出所料，是用TreeMap实现的。并且这些元素在结构中，根据comparator进行了排序，所以用iterator遍历时可以按照递增、递减顺序。

comparator在TreeSet中是比较关键的，这些元素要么是comparable，要么自己提供一个定制的comparator。并且comparator的实现一定要和equals方法一致。

对于add、remove和contains方法，理论上是**log(n)**时间复杂度。

## LinkedHashSet与HashSet的区别    

LinkedHashSet继承自HashSet，但是记录了元素的插入顺序，仍然具有原来HashSet常数时间复杂度操作的优势，但是这些操作会比原来慢一些，用来维护这个顺序。遍历访问元素的操作比HashSet更为高效，时间由元素数量决定。    

一样，LinkedHashSet用LinkedHashMap来实现。    






