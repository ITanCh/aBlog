
> java.util.ArrayList


ArrayList继承自AbstractList，AbstractList为随机访问数据的结构，如数组提供了基本实现，并且提供了Iterator。首先看AbstractList实现了什么方法。

## AbstractList

### AbstractList里可以存储null吗

null可以作为一项存储在ArrayList中。ListIterator是遍历访问AbstractList中元素较好的方式，需要注意获取元素序号的方法是previousIndex，而不是nextIndex，因为之前有next方法的调用。还有，这里判断两个元素是否相等采用的是equals方法，而不是`==`。

```java
 public int indexOf(Object o) {
        ListIterator<E> it = listIterator();
        if (o==null) {
            while (it.hasNext())
                if (it.next()==null)
                    return it.previousIndex();
        } else {
            while (it.hasNext())
                if (o.equals(it.next()))
                    return it.previousIndex();
        }
        return -1;
    }
```

### ListIterator可以逆序访问元素

从后向前遍历AbstractList，过程恰恰相反。

```java
 public int lastIndexOf(Object o) {
        ListIterator<E> it = listIterator(size());
        if (o==null) {
            while (it.hasPrevious())
                if (it.previous()==null)
                    return it.nextIndex();
        } else {
            while (it.hasPrevious())
                if (o.equals(it.previous()))
                    return it.nextIndex();
        }
        return -1;
    }
```
### Iterator遍历元素时，被其它线程修改了怎么办

如果在使用Iterator时，有其他线程尝试去修改List的大小，会被发现且立即抛出异常。

可以看`class Itr implements Iterator<E>`中，有属性`int expectedModCount = modCount;`记录着期望的数组大小，如果不一致，会抛出`ConcurrentModificationException`。

### Iterator在AbstractList中如何实现的

有两个游标分别记录当前指向的位置和上一次指向的位置。

```java
       /**
         * Index of element to be returned by subsequent call to next.
         */
        int cursor = 0;

        /**
         * Index of element returned by most recent call to next or
         * previous.  Reset to -1 if this element is deleted by a call
         * to remove.
         */
        int lastRet = -1;
```

### 如何处理遍历过程中，删除list中的元素导致的序号偏移

Iterator可以正确的删除AbstractList中的元素，并且保证访问的顺序的正确性。

```java
        public void remove() {
            if (lastRet < 0)
                throw new IllegalStateException();
            checkForComodification();

            try {
                AbstractList.this.remove(lastRet);
                if (lastRet < cursor)
                    cursor--;
                lastRet = -1;
                expectedModCount = modCount;
            } catch (IndexOutOfBoundsException e) {
                throw new ConcurrentModificationException();
            }
        }
```

删除的元素是上一个next调用后的元素，并且连续调用两次remove会抛出异常，因为元素只能删除一次，上次指向的元素已经没有了。

### ListIterator可以添加元素

在`class ListItr extends Itr implements ListIterator<E>`中实现了向list添加元素的方法。添加的位置是上一次next返回的元素之后，下一个next之前。

```java
        public void add(E e) {
            checkForComodification();

            try {
                int i = cursor;
                AbstractList.this.add(i, e);
                lastRet = -1;
                cursor = i + 1;
                expectedModCount = modCount;
            } catch (IndexOutOfBoundsException ex) {
                throw new ConcurrentModificationException();
            }
        }
```

## ArrayList

### ArrayList用什么来存储元素

就是用了一个简单的数组。。。

```java
/**
     * The array buffer into which the elements of the ArrayList are stored.
     * The capacity of the ArrayList is the length of this array buffer. Any
     * empty ArrayList with elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA
     * will be expanded to DEFAULT_CAPACITY when the first element is added.
     */
    transient Object[] elementData; // non-private to simplify nested class access
```

### 如何节约ArrayList占用的空间

使用trimToSize函数，将原始数组copy到适合大小的数组中。

```java
/**
     * Trims the capacity of this <tt>ArrayList</tt> instance to be the
     * list's current size.  An application can use this operation to minimize
     * the storage of an <tt>ArrayList</tt> instance.
     */
    public void trimToSize() {
        modCount++;
        if (size < elementData.length) {
            elementData = (size == 0)
              ? EMPTY_ELEMENTDATA
              : Arrays.copyOf(elementData, size);
        }
    }
```

### 用Iterator遍历ArrayList真的高效吗

不可否认，Iterator为遍历一个集合提供了统一的接口，用户可以忽略集合内部的具体实现，但是过多的封装会导致效率的降低。显然，Java开发人员认为通过下标遍历ArrayList的数组结构更加高效。所以重写了indexOf和lastIndexOf。

```java
    public int indexOf(Object o) {
        if (o == null) {
            for (int i = 0; i < size; i++)
                if (elementData[i]==null)
                    return i;
        } else {
            for (int i = 0; i < size; i++)
                if (o.equals(elementData[i]))
                    return i;
        }
        return -1;
    }
```

### ArrayList的clone是浅copy

ArrayList只是新建了数组，copy了元素的引用，元素本身没有进行copy。clone方法为什么不new一个ArrayList对象，而是调用了Object类的clone？因为Object的clone函数是native的，更高效。

```java
/**
     * Returns a shallow copy of this <tt>ArrayList</tt> instance.  (The
     * elements themselves are not copied.)
     *
     * @return a clone of this <tt>ArrayList</tt> instance
     */
    public Object clone() {
        try {
            //使用Object的clone获得一个对象
            ArrayList<?> v = (ArrayList<?>) super.clone();
            v.elementData = Arrays.copyOf(elementData, size);
            v.modCount = 0;
            return v;
        } catch (CloneNotSupportedException e) {
            // this shouldn't happen, since we are Cloneable
            throw new InternalError(e);
        }
    }
```

### ArrayList存储空间的扩展策略

如果list初始没有元素，使用一个静态的空数组。元素增加时，空间扩展为默认的10。在后续的扩展过程中，容量会每次增加原始大小的1/2。


### ArrayList实现了自己的iterator

虽然AbstractList实现了iterator，但ArrayList似乎不太满意，又重新实现了一遍。主要区别就是在获取元素时，利用了数组结构的优势，可以直接通过下标获取元素，而不必通过调用方法。

### 用Spliterator分割遍历ArrayList

ArrayList理所当然的实现了自己的Spliterator，也就是`ArrayListSpliterator`。分割的策略简而言之为：二分+延迟初始化。

ArrayListSpliterator有如下属性:

```java
this.list = list; // OK if null unless traversed 保存目标list
this.index = origin; //起始位置
this.fence = fence;  //终止位置
this.expectedModCount = expectedModCount;  //期望修改次数，用来判断运行时是否有其它线程修改
```

每次从中间开始分裂。在进行分裂时，原始spliterator保留中部至末尾的元素，新的spliterator保留原起始位置到中部的元素。

```java
        public ArrayListSpliterator<E> trySplit() {
            int hi = getFence(), lo = index, mid = (lo + hi) >>> 1;
            return (lo >= mid) ? null : // divide range in half unless too small
                new ArrayListSpliterator<E>(list, lo, index = mid,
                                            expectedModCount);
        }
```

在第一次创建spliterator时，fence被初始为-1，所以在实际使用spliterator时，getFence才会被调用，从而fence才会被赋值为数组大小。同样，expectedModCount也会进行重新赋值，使得spliterator在使用前与list保持一致。

```java
         private int getFence() { // initialize fence to size on first use
            int hi; // (a specialized variant appears in method forEach)
            ArrayList<E> lst;
            if ((hi = fence) < 0) {
                if ((lst = list) == null)
                    hi = fence = 0;
                else {
                    expectedModCount = lst.modCount;
                    hi = fence = lst.size;
                }
            }
            return hi;
        }
```

在遍历list时，会调用方法`tryAdvance`，将动作施加于元素之上。该方法会检查是否有其它线程的修改，在ArrayList中，有大量方法都会使用`modCount`来记录修改，或者判断是否在方法执行时有其它线程的修改，目前看来，用这个方法来检测是否有并行修改使得list结构变化是有效的，可以避免并行修改带来的一些问题。

```java
        public boolean tryAdvance(Consumer<? super E> action) {
            if (action == null)
                throw new NullPointerException();
            int hi = getFence(), i = index;
            if (i < hi) {
                index = i + 1;
                @SuppressWarnings("unchecked") E e = (E)list.elementData[i];
                action.accept(e);
                if (list.modCount != expectedModCount)
                    throw new ConcurrentModificationException();
                return true;
            }
            return false;
        }
```



