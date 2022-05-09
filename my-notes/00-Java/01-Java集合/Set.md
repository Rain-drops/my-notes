### 一、HashSet

#### 1. 成员变量

```java
public class HashSet<E> extends AbstractSet<E> implements Set<E>, Cloneable, java.io.Serializable {
	/** HashSet 的集合元素由 HashMap 的 Key 来保存 */
    private transient HashMap<E,Object> map;
    /** HashMap 的 value, 虚值 */
    private static final Object PRESENT = new Object();
}
```

#### 2. 操作方法

##### 2.1 add(E e)

```java
public boolean add(E e) {
    return map.put(e, PRESENT)==null;
}
```



### 二、LinkedHashSet

#### 1. 成员变量

```java
public class LinkedHashSet<E> extends HashSet<E> implements Set<E>, Cloneable, java.io.Serializable {	
}
```

#### 2. 操作方法

##### 2.1 add(E e)

```java

```



### 三、TreeSet