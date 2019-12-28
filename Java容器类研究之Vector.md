
java.util.Vector


## Vector与ArrayList的异同

相同点：随机存取，可通过位置序号直接获取数据。都是通过一个数组来存放元素。

不同点：Vector是线程安全的，方法有synchronized关键字修饰。   


## Vector容量增长策略    

Vector默认的增长策略是每次在原容量的基础上x2。

## Vector的ListIterator怎么做到线程安全的

Vector实现了自己的iterator，为了保证并发线程安全的共享一个Vector，开发者在next等方法中也加入了synchronized。

```java
    public E next() {
            synchronized (Vector.this) {
                checkForComodification();
                int i = cursor;
                if (i >= elementCount)
                    throw new NoSuchElementException();
                cursor = i + 1;
                return elementData(lastRet = i);
            }
    }
```

这里synchronized修饰的是Vector.this对象本身，而不是iterator自己，这样多个线程使用iterator操作Vector时，就可以保证线程的安全。

## Vector与ArrayList实现的Spliterator类似    

唯一的区别就是在使用自己的Vector时，加上了synchronized关键字。


## Stack与Vector

Stack类继承自Vector，stack的实现不止一种方式，比如LinkedList。java中在Vector基础上实现了一个Stack。实现的想法也很简单，就是在数组的末尾push和pop。

