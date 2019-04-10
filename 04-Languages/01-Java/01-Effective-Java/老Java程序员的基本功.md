# 老Java程序员的基本功
初学Java时，有两本Java基础语法书非常有名，一本是《Thinking in Java》，另一本是《Core Java》，都是大部头。当时前者是第四版，后者大概是第五六版。此外，还有一本没那么厚，名气却不小的书，那就是《Effective Java》，当年看的是第二版，去年更新到了第三版，口碑依然非常不错。今天，就重读了第三版并从中挑出个人觉得比较重要的点，通过这本书聊聊一名老Java程序员不应该忘记的基本功。


## 1.对象的生命周期

### 1.3 内存"泄露"
我们都知道Java不像C/C++，要自己去申请内存，明确指定大小，最后用完释放。既然一切工作都交给了垃圾回收器（GC），那为什么Java中还会有C/C++中的内存泄露问题呢？准备来说，Java中的内存问题应该叫做Unintentional Object Retention。不是很好翻译，大体就是说你在无意间一直拿着对象的引用，导致GC无法回收。下面就来看一个例子：

```java
public class Stack {
    private Object[] elements;
    private int size;

    // Here potential memory leak could happen..
    public Object pop() {
        if (size == 0)
            throw new EmptyStackException();
        return elements[--size];
    }
}
```

不知道你能否看出上面代码中方法`pop()`有什么问题？答案就是它只是简单的移动了栈顶的标记位`size`，而没有清空Stack中持有的对象引用。如果那个引用对应的对象已经没用了，但恰好Stack收缩后一直没有扩张，即那个位置一直没被其他值覆盖。这时内存泄露就发生了，因为GC没法知道其实Stack中的`size`以上持有的引用是没用的了。从GC的角度来看，一切都一视同仁。所以，下面才是正确的做法，所谓的**Null Out Object References**。

```java
    public Object pop() {
        if (size == 0)
            throw new EmptyStackException();
        Object result = elements[---size];
        elements[size] = null;
        return result;
    }
```

当然，这并不是说所有不用的对象，我们都要手动把它的引用置为`null`。这里讲到的问题和解决方法都应该是特例，只适用于下面几种特殊的应用场景：

1. **Class自己管理内存**：一个类中有一个成员变量是`List`，这种情况一般来说都不算，除非你是下面提到的两种情况。自己管理内存一般指用数组申请到一块内存，然后在这片内存中实现数据结构等，比如上面的Stack，也可能是循环数组实现的Buffer等。
2. **实现Cache**：也就是说你持有的引用都是其他对象的，你只是提供了缓存的实现。当这些对象已经没用了时，你可能无法知道而依然在缓存中保存着它们的引用。这种场景下，`WeakHashMap`和JDK提供的弱引用类们就要大显身手。所谓Weak，就是指当对象没有其他引用，只有在你这个缓存中被引用时，就会被回收。
3. Aaa