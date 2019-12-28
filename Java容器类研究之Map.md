
java.util.Map

## Map中的自我引用   

需要小心用易变的对象作为Map的key，这会导致Map的行为无法预测。Map也不可以将自己作为key，可以作为value，但是会导致equals和hashCode方法不是well defined.   

Map中有些操作涉及递归的遍历，如果Map自我引用，则有可能出现异常。这些方法包括：clone，equals，hashCode和toString。   

## Map可以将null作为key和value

这是Map接口自己默认实现的一个获取指定key的value，当没有该key时可以获取一个默认值得方法。这就给用户提供了更灵活的选择来处理map中没有存这个key的情况。这里比较有意思的一点是在用`v = get(key)`判断一次后，为什么又用`containsKey(key)`再判断一次，因为有的map中是允许存null作为value的，所以有key在Map中，但是value为null的情况。

```java
    default V getOrDefault(Object key, V defaultValue) {
        V v;
        return (((v = get(key)) != null) || containsKey(key))
            ? v
            : defaultValue;
    }
```

## HashMap

### 如何求int型数的最近2次幂   

HashMap中，用户可以初始化容量，但是HashMap中容量皆为2的次幂，所以会把用户预设的值先转为最近的2的次幂。tableSizeFor方法求出的结果总是大于或等于cap，且最接近cap的一个2的次幂。当然，对于大于`2^30`的数，会返回-2147483648，所以这里会判断n为负数的情况，同时，设置了HashMap的最大容量应该为`2^30`（MAXIMUM_CAPACITY）。

有意思的是经过如下几步就能求得一个2的次幂，下面方法所做的是先求得一个`2^n-1`，然后+1。获得一个`2^n-1`的过程也很巧妙，就是不停的拷贝`1`。`n |= n >>> x;`将前x位的`1`拷贝到了接着的x位。通过几步位操作就获得了目标值，不得不赞叹开发者的聪明。

```java
    static final int tableSizeFor(int cap) {
        int n = cap - 1;
        n |= n >>> 1;
        n |= n >>> 2;
        n |= n >>> 4;
        n |= n >>> 8;
        n |= n >>> 16;
        return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
    }
```

### HashMap的table   

HashMap的table就是一个Node的数组，大小一致保持2的次幂。Node就是HashMap中存储的元素，它有哈希值、key、value和存储下一个Node的next属性。处理冲突的方法是闭哈希方法，也就是有相同的hash值的Node会用链表串起来。

### HashMap中的hash值如何计算    

```java
    static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }
```

将key的hashCode的高位和低位部分通过异或合并，使得低位和高位的值都能在hash时发挥作用。因为哈希表会用一个掩码来取hashCode的后面一段，如不采用上述方法，前面一部分的hashCode就被忽略了。如果一组key恰巧只在前段的hash值有所差异，而后段都是相同的，则会出现大量的冲突，这种情况在实际中是很可能存在的。


### 如何将Node哈希到table中    

公式如：`table[node.hash & (capacity - 1)] = node`。node的hash值是key在上述`hash()`方法中处理过的值，通过与当前容量-1进行mask，直接获取到哈希表的位置。

### HashMap的table的扩张    

如上所述，table大小保持2的次幂，扩张的步骤：   
1. 申请一个容量**乘以2**的新Node数组。   
2. 遍历table，将原来的元素依次copy到新数组上，这里原来的链表会进行分裂。因为table的扩大，Node的hash值会被多mask一位，所以Node被Hash到的位置也会变化，Node要么保持原位置，要么就是在相对于原位置有一个旧容量大小的偏移。

这里不会将原来的元素重新进行一次hash的过程，重建原来的hash table，这样代价是比较大的。这里就利用了Node位置变化的规律，直接将原来的链表分裂为两个。    

虽然已经进行了优化，但是该过程代价还是比较高的，时间复杂度为**原table大小+元素数量**，

### table中链表太长如何处理    

当Node发生多次冲突，在一个hash值下建了一个很长的链表，这会导致查询的代价越来越大，这里采用了树结构来减轻这种问题。

具体过程是，当某个hash值下的链表超过了阈值，则会采取策略对其长度进行消解。   
1. 如果table还不算大，那么直接对table扩容，链表自然会被分裂到两地。   
2. 策略二，如果table已经很大了，扩容table已经不可取，那么就采用红黑树结构转化链表。

红黑树的创建不再详述。需要知道，这里采用的是二叉搜索树，进行比较的是每个hash值，不是key的直接比较。红黑树的根就是table中第一个节点。原始链表会先变成双向链表，以保存前后关系，然后再变成树结构。虽然由链表转化成了树结构，但是每个节点仍然保存了链表的前后关系，所以可以迅速的从树结构退化为链表结构。从TreeNode的属性中可以看出其既有树结构的left、right、parent关系，又有prev、next（继承）的双向链表关系。

```java
        TreeNode<K,V> parent;  // red-black tree links
        TreeNode<K,V> left;
        TreeNode<K,V> right;
        TreeNode<K,V> prev;    // needed to unlink next upon deletion
```

### HashMap的KeySet、Values和EntrySet  

HashMap用KeySet来提供一个key的集合视角，KeySet并没有再申请一个空间来存储这些key，而是将所有的方法建立于HashMap的方法之上。KeySet提供了删除key的操作，该操作会映射到其HashMap之上，最后就是在HashMap中删除了该元素。KeySet没有提供添加元素的方法，因为HashMap需要的是key和value，而KeySet只能添加key，方法不能映射到HashMap上。    

同样，HashMap提供了value的集合视角Values，EntrySet提供了Node集合的视角，原理类似。

### HashMap的Spliterator   

在HashMap中，分为key、value和node三种Spliterator，实现原理都是类似的。HashMap的Spliterator的划分不是针对元素的直接划分，而是对table这个数组的划分，这样更为简单。划分的策略也很简单，采用的二分法。    



