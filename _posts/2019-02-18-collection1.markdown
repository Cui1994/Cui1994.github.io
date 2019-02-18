---
layout: post
category: "Java"
title:  "Java核心类库学习（二）Collection"
tags: [Java, 源码, Collection]
---

* content
{:toc}

![](https://picsum.photos/800/300/?image=743)

Java中的集合分为三类，Set、Map 和 List，它们都处于`java.util`包中，每个接口都有各自实现。这其中 Set 和 List 继承自 Collection，本文将介绍 Collection 接口以及一些常用 Collection 的实现。





## Collection

### Iterable

要让一个 Java 对象提供`foreach`循环，只要实现`Iterable`接口，并调用它的`iterator`生成一个`Iterator`即可。
```java
public interface Iterable<T> {
    Iterator<T> iterator();
}
```

### Iterator
提供了遍历 Collection 集合元素的统一编程接口，Collection 没有`get()`方法获取指定位置的元素，只能通过 Iterator 遍历。
```java
public interface Iterator<E> {
    boolean hasNext();
    E next();
    default void remove() {
        throw new UnsupportedOperationException("remove");
    }
}
```

### Collection
继承自`Iterable`，提供一些通用的集合方法。`List`、`Queue`和`Set`接口均继承自`Collection`接口。
```java
public interface Collection<E> extends Iterable<E> {
    int size();
    boolean isEmpty();
    boolean contains(Object o);
    boolean add(E, a)
    boolean remove(Object o);
    void clear();
    boolean addAll(Collection<? extends E> c);
    boolean removeAll(Collection<?> c);
    public <T> T[] toArray(T[] a) {}
}
```

有一个公共的抽象实现类`AbstractCollection`，实现了如`isEmpty()`、`contains()`、`clear()`、`remove()`等方法。集合的实现类一般会继承自这个抽象类，根据其可变还是不可变实现对应的方法：
- 不可变集合只需实现`size()`和`iterator()`接口，并且返回的 Iterator 需要实现`hashNext()`和`next()`方法。
- 可变集合则还需要实现`add()`方法，返回的 Iterator 需要实现`remove()`方法。

`toArray()`方法也在这里实现，具体流程如下：
1. 根据集合`size()`返回的大小创建一个新的数组
2. 通过`next()`将集合内的元素加入数组
3. 数组大小够用，则将数组已经填充元素的部分复制一个新的数组返回；若不够用则进行`finishToArray`动态扩容，扩容为1.5倍大小`int newCap = cap + (cap >> 1) + 1`。

## List
是`Collection`对线性表的扩充，提供了针对指定位置（index）元素操作的方法。

```java
    E get(int index);
    E set(int index, E element);
    void add(int index, E element);
    E remove(int index);
    int indexOf(Object o);
    int lastIndexOf(Object o);
    ListIterator<E> listIterator(int index);
    List<E> subList(int fromIndex, int toIndex);
    default void sort(Comparator<? super E> c) {};
```

### AbstractList

`AbstarctList`是`List`的一个抽象实现类，继承自`AbstractCollection`并实现了`List`接口。同`AbstractCollection`，针对可变与不可变集合，只需要实现相应的方法即可：
- 不可变集合只需要实现`size()`和`get()`方法
- 可变集合则需要实现`set()`方法，如果支持动态扩容则需要实现`add()`和`remove()`方法。

`AbstractList`对`iterator`方法的实现很简单，在内部定义了一个`Iterator`类，在方法中将其实例化并返回。
```java
public Iterator<E> iterator() {
    return new Itr();
}
```
此外，它还定义了`ListIterator`类，继承自`Iterator`，增加了向前遍历的功能，在`indexOf()`实现中用到了它。

`subList()`的实现也同样采用内部类`SubList`，它继承自`AbstractList`，增加了一些 index 范围的校验。
```java
public List<E> subList(int fromIndex, int toIndex) {
    subListRangeCheck(fromIndex, toIndex, size);
    return new SubList(this, 0, fromIndex, toIndex);
}
```

### modCount
`AbstractList`中维护者一个`整型的modCount`字段，在对集合进行变更时会伴随`modCount`的增加，在对集合进行遍历时是不允许有其他的变更动作的，因此会在遍历开始之前拿到`modCount`的值写入到`expectedModCount`变量中，在遍历过程中会检查这个值和当前的`modCount`值相等，否则会抛出`ConcurrentModificationException`。

## ArrayList
`ArrayList`继承自`AbstractList`，并实现了三个标识接口，实现该类接口不许实现任何方法即拥有了对应功能
- `RandomAccess`：支持快速随机访问
- `Cloneable`：可克隆（`clone()`方法）
- `java.io.Serializable`：可序列化的（`writeObject()`和`readObject()`方法）

`ArrayList`对数组操作进行了封装和优化，支持动态扩容。

### 初始化
内部维护了一个数组来存储元素，一个整型`size`来维护自身元素个数。
```java
transient Object[] elementData;
private int size;
```

它提供了三个构造方法，参数分别是空、制定大小和指定集合，当参数为空时，会直接返回`EMPTY_ELEMENTDATA`即空的数组。而后面两种规则会指定`ArrayList`的大小。`ArrayList`有一个默认大小`DEFAULT_CAPACITY`为 10，但只有当默认创建的数组添加一个新的元素时，这个数组的大小才会变成默认大小。
```java
public ArrayList() {
    this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
}
```

### 重写
`ArrayList`重写了`AbstractList`和`AbstractCollection`中的方法，修改了其相应实现。`toArray()`不再借助`Iterator`，而是直接截取 size 大小的数组返回（调用`Arrays.copyOf`方法）。

### 动态扩容
`ArrayList`会在添加元素时检查当前数组的大小是否满足要求，若不满足要求则会进行动态扩容，通过下面几步进行。
```java
// 只针在默认数组中对初次添加元素，会将数组大小增至DEFAULT_CAPACITY
private static int calculateCapacity(Object[] elementData, int minCapacity) {
    if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
        return Math.max(DEFAULT_CAPACITY, minCapacity);
    }
    return minCapacity;
}

private void ensureCapacityInternal(int minCapacity) {
    ensureExplicitCapacity(calculateCapacity(elementData, minCapacity));
}

// 若当前数组大小不满足条件，则会进行动态扩容
private void ensureExplicitCapacity(int minCapacity) {
    modCount++;

    // overflow-conscious code
    if (minCapacity - elementData.length > 0)
        grow(minCapacity);
}

// 扩容策略
private void grow(int minCapacity) {
    // overflow-conscious code
    int oldCapacity = elementData.length;

    // 扩容至1.5倍
    int newCapacity = oldCapacity + (oldCapacity >> 1);

    // 若仍不满足要求，直接使用minCapacity
    if (newCapacity - minCapacity < 0)
        newCapacity = minCapacity;

    // 保证newCapacity不会超过整数上限，MAX_ARRAY_SIZE的值为Integer.MAX_VALUE - 8
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity);
    // minCapacity is usually close to size, so this is a win:
    elementData = Arrays.copyOf(elementData, newCapacity);
}

private static int hugeCapacity(int minCapacity) {
    if (minCapacity < 0) // overflow
        throw new OutOfMemoryError();
    return (minCapacity > MAX_ARRAY_SIZE) ?
        Integer.MAX_VALUE :
        MAX_ARRAY_SIZE;
}
```

动态扩容会将数组进行拷贝，存在开销。为了避免频繁的扩容，在初始化时就应该为`ArrayList`指定初始大小，但这样在实际使用前也会一直占据大块的连续内存，最好的思路是**仍然使用无参构造器创建`ArrayList`，只在使用前为其进行一次扩容，通过调用`ensureCapacity()`方法。**

### Vector
`Vector`与`ArrayList`的功能和实现都类似，最大的区别在于**Vector 是线程安全的**，它锁实现的大部分方法带有`synchronized`关键字，但开销也较大。`Stack`是它的一个子类，在单线程环境中不推荐使用`Vector`和`Stack`。

## Queue
实现了先入先出的队列功能
```java
public interface Queue<E> extends Collection<E> {
    boolean add(E e);
    boolean offer(E e);
    E remove();
    E poll();
    E element();
    E peek();
}
```

有两个地方需要注意：
- `add()`与`offer()`的区别：`add()`对满的队列使用时会抛出异常，而`offer()`则会返回`false`。
- `remove()`与`poll()`的区别：`remove()`在对空的队列使用时会抛出异常，而`poll()`则会返回`null`。

### AbstractQueue
是`Queue`的超级实现类，其中`add`与`remove`方法的实现如下：
```java
public boolean add(E e) {
    if (offer(e))
        return true;
    else
        throw new IllegalStateException("Queue full");
}

public E remove() {
    E x = poll();
    if (x != null)
        return x;
    else
        throw new NoSuchElementException();
}
```
### Deque
`Deque`继承自`Queue`，定义了双向队列。
```java
public interface Deque<E> extends Queue<E> {
    void addFirst(E e);
    void addLast(E e);
    boolean offerFirst(E e);
    boolean offerLast(E e);
    E removeFirst();
    E removeLast();
    E pollFirst();
    E pollLast();
    E getFirst();
    E getLast();
    E peekFirst();
    E peekLast();
    boolean removeFirstOccurrence(Object o);
    boolean removeLastOccurrence(Object o);
}
```

## ArrayDeque
一个基于数组构建的循环队列，是`Deque`的实现类，不但实现了`Deque`的全部方法，还实现了栈相关的方法，因此它**既可以作为队列使用，又能作为栈使用。**
```java
// *** Stack methods ***

public void push(E e) {
    addFirst(e);
}

public E pop() {
    return removeFirst();
}
```

### 初始化
定义了几个成员变量，用来存储元素和首末位置的引用。定义了一个常量，表示数组的最小大小，`ArrayDeque`内部维护的数组大小总是 2 的幂次方。
```java
    transient Object[] elements; // non-private to simplify nested class access
    transient int head;
    transient int tail;
    private static final int MIN_INITIAL_CAPACITY = 8;
```

有三个构造方法，无参构造器默认数组大小为 16
```java
public ArrayDeque() {
    elements = new Object[16];
}

public ArrayDeque(int numElements) {
    allocateElements(numElements);
}

public ArrayDeque(Collection<? extends E> c) {
    allocateElements(c.size());
    addAll(c);
}
```
其余构造方法需要调用`allocateElements()`方法找到最近的 2 的幂次方，方式为不断和自己的右移 1/2/4/8/16 位之后的值进行或运算，使自己的最高位以下的值均为 1，之后进行自加操作即可。
```java
private static int calculateSize(int numElements) {
    int initialCapacity = MIN_INITIAL_CAPACITY;
    // Find the best power of two to hold elements.
    // Tests "<=" because arrays aren't kept full.
    if (numElements >= initialCapacity) {
        initialCapacity = numElements;
        initialCapacity |= (initialCapacity >>>  1);
        initialCapacity |= (initialCapacity >>>  2);
        initialCapacity |= (initialCapacity >>>  4);
        initialCapacity |= (initialCapacity >>>  8);
        initialCapacity |= (initialCapacity >>> 16);
        initialCapacity++;

        if (initialCapacity < 0)   // Too many elements, must back off
            initialCapacity >>>= 1;// Good luck allocating 2 ^ 30 elements
    }
    return initialCapacity;
}
```

### add
增加元素要判断数组是否已满，判断方式为 head 与 tail 指向同一个位置，head是储存元素的，而 tail 是不存储元素的。这里`head = (head - 1) & (elements.length - 1)`是当 head 的值为 0 时，`head-1`为 -1，二进制全为 1，与`elements.length - 1`进行与运算时只会保留`elements.length - 1`的部分，即将元素添加到`elements.length - 1`的位置，这也是`ArrayDeque`的大小永远是 2 的幂次方的原因。
```java
public void addFirst(E e) {
    if (e == null)
        throw new NullPointerException();
    elements[head = (head - 1) & (elements.length - 1)] = e;
    if (head == tail)
        doubleCapacity();
}

public void addLast(E e) {
    if (e == null)
        throw new NullPointerException();
    elements[tail] = e;
    if ( (tail = (tail + 1) & (elements.length - 1)) == head)
        doubleCapacity();
}
```

### 动态扩容
扩容方式很简单，直接扩容至原大小的 2 倍。
```java
private void doubleCapacity() {
    assert head == tail;
    int p = head;
    int n = elements.length;
    int r = n - p; // number of elements to the right of p
    int newCapacity = n << 1;
    if (newCapacity < 0)
        throw new IllegalStateException("Sorry, deque too big");
    Object[] a = new Object[newCapacity];
    System.arraycopy(elements, p, a, 0, r);
    System.arraycopy(elements, 0, a, r, p);
    elements = a;
    head = 0;
    tail = n;
}
```

## LinkedList
同时实现了`List`和`Deque`接口，同时继承自`AbstractSequentialList`。`AbstractSequentialList`继承自`AbstractList`，提供了`itrator`接口的默认实现，返回内部`ListIterator`对象。

### 初始化
LinkedList为基于链表的集合，内部维护着节点类。
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

三个成员变量，维护着头尾节点的引用和链表大小。
```java
transient int size = 0;
transient Node<E> first;
transient Node<E> last;
```

两个构造方法，无参和集合。
```java
public LinkedList() {
}

public LinkedList(Collection<? extends E> c) {
    this();
    addAll(c);
}
```

### 内部方法
定义了一组由 link 和 unlink 开头的方法，维护着链表特有的逻辑，其他从`List`和`Deque`继承来的方法均在这些内部方法的协助下实现。
```java
// 举个例子，将元素 link 到首位的逻辑
private void linkFirst(E e) {
    final Node<E> f = first;
    final Node<E> newNode = new Node<>(null, e, f);
    first = newNode;
    if (f == null)
        last = newNode;
    else
        f.prev = newNode;
    size++;
    modCount++;
}
```

### 性能
`LinkedList`在进行带有 index 的增删查时，通常需要对链表进行遍历或对折遍历。同`ArrayList`相比，`LinkedList`的插入很快，但查询无力。
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



