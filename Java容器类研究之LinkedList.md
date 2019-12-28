
java.util.LinkedList

## Java中有现成的队列可以用吗

有，就是LinkedList。LinkedList实现的接口如下，其实也可以当做stack使用：

```java
public class LinkedList<E>
    extends AbstractSequentialList<E>
    implements List<E>, Deque<E>, Cloneable, java.io.Serializable
```

`Deque`是一个双端队列的接口，而Deque又继承了接口`Queue`。所以LinkedList可以作为队列、双端队列来使用。

`AbstractSequentialList`提供了顺序访问的方法，当然，大部分方法都依赖于ListIterator来实现，所以将锅甩给了子类。

## LinkedList用什么结构来实现

同样很简单，是一个Node的双向链表结构。

```java
    private static class Node<E> {
        E item;
        Node<E> next;
        Node<E> prev;

        Node(Node<E> prev, E element, Node<E> next) {
            this.item = element;
            this.next = next;
            this.prev = prev;
        }
    }
```

有属性first和last记录着链表的开始和结束节点。

## 在链表中通过下标取值是低效的   

在LinkedList中，大量的方法需要先获得指定下标的节点，具体实现如下：   

```java
    Node<E> node(int index) {
        // assert isElementIndex(index);

        if (index < (size >> 1)) {
            Node<E> x = first;
            for (int i = 0; i < index; i++)
                x = x.next;
            return x;
        } else {
            Node<E> x = last;
            for (int i = size - 1; i > index; i--)
                x = x.prev;
            return x;
        }
    }
```

可以看出，开发者已经尽力优化，根据index大小决定从何处开始遍历。   

## LinkedList实现了自己的ListIterator   

遍历方法利用了链表结构的特性，进行遍历。其中有如下属性记录遍历状态。

```java
    private Node<E> lastReturned; //记录最近一次返回的节点
    private Node<E> next;    //记录下一个要返回的节点
    private int nextIndex;   //记录下一个要返回的位置
    private int expectedModCount = modCount;  //记录期望的修改次数
```


## LinkedList的Spliterator采用什么样的分割策略

LinkedList每次划分，不是采用的1/2策略，而是每次分裂出来的一块数据都增加一个`BATCH_UNIT(1024)`的大小。比较有趣的是，每次分裂出来的Spliterator并不是LinkedList自己实现的Spliterator，而是一个ArraySpliterator，ArraySpliterator采用的是1/2的分裂策略。所以LinkedList每次分裂都想尽可能快的分裂出更多的元素，并且分裂过程中将链表结构转化为数组结构，这样做可能是出于性能的考虑，具体什么原因还是比较纳闷的。

```java
        //该方法位于class LLSpliterator中
        public Spliterator<E> trySplit() {
            Node<E> p;
            int s = getEst();
            if (s > 1 && (p = current) != null) {
                int n = batch + BATCH_UNIT;
                if (n > s)
                    n = s;
                if (n > MAX_BATCH)
                    n = MAX_BATCH;
                //copy到数组中
                Object[] a = new Object[n];
                int j = 0;
                do { a[j++] = p.item; } while ((p = p.next) != null && j < n);
                current = p;
                batch = j;
                est = s - j;
                //这里返回的不是LLSpliterator，其实是ArraySpliterator
                return Spliterators.spliterator(a, 0, j, Spliterator.ORDERED);
            }
            return null;
        }
```