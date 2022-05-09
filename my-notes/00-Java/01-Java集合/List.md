### 一、ArrayList

1. 非线程安全的，多线程下会发生数据覆盖

2. 实现于 List、RandomAccess 接口。可以插入空数据，支持随机访问

  3. 有两个重要属性：elementData 数组和 size 大小。

  4. elementData 被 `transient`  关键词修饰，防止被自动序列化。ArrayList 自定义了序列化与反序列化，只序列化被使用的数据。

  5. 底层是动态数组，默认大小是 10，扩容是扩展 1.5 倍。

  6. 支持快速随机访问，即通过元素的序号快速获取元素对象。

  7. 插入和删除的时间复杂度受元素位置的影响，默认队尾插入O(1)。

  8. ArrayList 的空间花费 主要体现在列表的结尾会预留一定的容量空间

     > JDK 1.7 是头插法，resize 时，会造成死循环
     >
     > JDK 1.8 是尾插法，resize 时，不会造成死循环

#### 1. 成员变量

```java
public class ArrayList<E> extends AbstractList<E> implements List<E>, RandomAccess, Cloneable, java.io.Serializable{
    /** 默认初始容量 */
    private static final int DEFAULT_CAPACITY = 10;
    /** 
     * 为了对应不同的构造函数，ArrayList使用了不同的数组
     * 方便在第一次添加元素时知道该 elementData 是从空的构造函数还是有参构造函数被初始化的，以便确认如何扩容
     */
    private static final Object[] EMPTY_ELEMENTDATA = {};
    private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};
    /** 存储数组列表元素的数组缓冲区 */
    transient Object[] elementData; 
    /** 数组列表的大小(包含元素的数量)。 */
    private int size;
}
```

#### 2. 操作方法

##### 2.1 add(E e)

```java
public boolean add(E e) {
    // 确保数组已使用长度（size）加1之后足够存下 下一个数据
    ensureCapacityInternal(size + 1); 
    elementData[size++] = e;
    return true;
}
private void ensureCapacityInternal(int minCapacity) {
    if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
    	minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
    }
    ensureExplicitCapacity(minCapacity);
}
private void ensureExplicitCapacity(int minCapacity) {
    // 修改次数 modCount 标识自增1，如果当前数组已使用长度（size）加1后的大于当前的数组长度，则调用 grow 方法，增长数组，
    // grow 方法会将当前数组的长度变为原来容量的1.5倍。
    modCount++;
    if (minCapacity - elementData.length > 0)
    	grow(minCapacity);
}
private void grow(int minCapacity) {
    int oldCapacity = elementData.length;
    int newCapacity = oldCapacity + (oldCapacity >> 1);
    if (newCapacity - minCapacity < 0)
        newCapacity = minCapacity;
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity);
    // minCapacity is usually close to size, so this is a win:
    elementData = Arrays.copyOf(elementData, newCapacity);
}
```

##### 2.2 add(int index, E element)

```java
public void add(int index, E e) {
    rangeCheckForAdd(index);
    ensureCapacityInternal(size + 1);
    System.arraycopy(elementData, index, elementData, index + 1, size - index);
    elementData[index] = element;
    size++;
}
```

##### 2.3 addAll(int index, Collection<? extends E> c)

```java
public boolean addAll(int index, Collection<? extends E> c) {
    rangeCheckForAdd(index);

    Object[] a = c.toArray();
    int numNew = a.length;
    ensureCapacityInternal(size + numNew);  
    
    int numMoved = size - index;
    if (numMoved > 0)
    System.arraycopy(elementData, index, elementData, index + numNew, numMoved);

    System.arraycopy(a, 0, elementData, index, numNew);
    size += numNew;
    return numNew != 0;
}
```



### 二、LinkedList

#### 1. 成员变量

```java
public class LinkedList<E> extends AbstractSequentialList<E> implements List<E>, Deque<E>, Cloneable, java.io.Serializable {
    /** LinkedList 的大小 */
    transient int size = 0;
    /** LinkedList的头节点 */
    transient Node<E> first;
    /** LinkedList的尾节点 */
    transient Node<E> last;
    /**  */
    private static class Node<E> {
        /** 节点的值 */
        E item;
        /** 当前节点的后一个节点的引用 */
        Node<E> next;
        /** 当前节点的前一个节点的引用 */
        Node<E> prev;
    }
}
```

#### 2. 操作方法

##### 2.1 add(int index, E element)

```java
public void add(int index, E element) {
    checkPositionIndex(index);

    if (index == size)
        linkLast(element);
    else
        linkBefore(element, node(index));
}
void linkLast(E e) {
    final Node<E> l = last;
    final Node<E> newNode = new Node<>(l, e, null);
    last = newNode;
    if (l == null)
        first = newNode;
    else
        l.next = newNode;
    size++;
    modCount++;
}
void linkBefore(E e, Node<E> succ) {
    final Node<E> pred = succ.prev;
    final Node<E> newNode = new Node<>(pred, e, succ);
    succ.prev = newNode;
    if (pred == null)
        first = newNode;
    else
        pred.next = newNode;
    size++;
    modCount++;
}
```

##### 2.2 addAll(int index, Collection<? extends E> c)

```java
public boolean addAll(int index, Collection<? extends E> c) {
    // 判断传进来的参数是否合法
    rangeCheckForAdd(index);
    // 先把集合转化为数组，然后为该数组添加一个新的引用（Objext[] a）。
    Object[] a = c.toArray();
    // 新建一个变量存储数组的长度。
    int numNew = a.length;
    if (numNew == 0)
        return false;
	// Node<E> succ：待添加节点的位置。
	// Node<E> pred：待添加节点的前一个节点。
    Node<E> pred, succ;
    if (index == size) { // 此时需要添加 LinkedList 中的集合中的每一个元素都是在 LinkedList 最后面。
        // 所以把 succ 设置为空，pred 指向尾节点。
        succ = null;
        pred = last;
    } else {
        // succ 指向插入待插入位置的节点，node(int index)返回对应索引位置上的 Node
        // pred 指向 succ 节点的前一个节点
        succ = node(index);
        pred = succ.prev;
    }
	// 
    for (Object o : a) {
        @SuppressWarnings("unchecked") E e = (E) o;
        Node<E> newNode = new Node<>(pred, e, null);
        if (pred == null)
            first = newNode;
        else
            pred.next = newNode;
        pred = newNode;
    }
    // 当 succ==null，新添加的节点位于 LinkedList 集合的最后一个元素的后面
    if (succ == null) {
        // 此时 pred 指向的是 LinkedList 中的最后一个元素
        last = pred;
    } else { // 当不为空的时候，表明在 LinkedList 集合中添加的元素
        // 把 pred 的 next 指向 succ 上，succ 的 prev 指向 pred。
        pred.next = succ;
        succ.prev = pred;
    }
    // 最后把集合的大小设置为新的大小。
	// modCount（修改的次数）自增。
    size += numNew;
    modCount++;
    return true;
}
```



### 三、Vector

#### 1. 成员变量

```java
public class Vector<E> extends AbstractList<E> implements List<E>, RandomAccess, Cloneable, java.io.Serializable {
	/** 保存元素 */
    protected Object[] elementData;
    /** 动态数组的实际大小 */
    protected int elementCount;
    /** 动态数组的增长系数 */
    protected int capacityIncrement;
    
}
```

#### 2. 操作方法

##### 2.1 add(E e)

```java
public synchronized boolean add(E e) {
    modCount++;
    ensureCapacityHelper(elementCount + 1);
    elementData[elementCount++] = e;
    return true;
}
private void ensureCapacityHelper(int minCapacity) {
    if (minCapacity - elementData.length > 0)
        grow(minCapacity);
}
private void grow(int minCapacity) {
    int oldCapacity = elementData.length;
    int newCapacity = oldCapacity + ((capacityIncrement > 0) ? capacityIncrement : oldCapacity);
    if (newCapacity - minCapacity < 0)
        newCapacity = minCapacity;
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity);
    elementData = Arrays.copyOf(elementData, newCapacity);
}
```

