---
title: Java集合原理和底层数据结构总结
date: 2024-08-25 23:05:58
tags: 
  - Java
  - Java Collections
categories:
  - [Java SE, Collections]
cover: https://pics.findfuns.org/java-collections.png
---

# 先放一张集合的层级图

<img src="https://pics.findfuns.org/Collections-hierarchy.png" alt="collectionsHierarchy" style="zoom:50%;" />

# List

List的特点是顺序性，可重复性。

## `ArrayList`

允许任何元素包括`null`。和`Vector`相比**不是线程安全的**。在必要时可以用`Collections.synchronizedList()`来将`ArrayList`转换成线程安全的List

```java
List list = Collections.synchronizedList(new ArrayList());
```

底层维护了一个`Object`数组用来存储元素。

```Java
transient Object[] elementData;
```

定义了一个`grow()`方法，当调用一系列重载的`add`方法时，会根据是否到达`capacity`来调用`grow()`以保证`ArrayList`的容量满足需求。

## `PriorityQueue`

PriorityQueue通过维护了一个小顶堆来保证首元素是所有元素中最小的元素，最小取决于元素实现的`Comparator`或者`Comparable`。它也提供了一个构造函数以便传入自定义的`Comparator`来决定排序的顺序。

```java
public PriorityQueue(Comparator<? super E> comparator)
```

```java
transient Object[] queue;

int size;

private final Comparator<? super E> comparator;

transient int modCount; 
```

这里的modCount是一个用来记录restructure（添加，删除或改变集合中元素的排列顺序）次数的变量，当创建了iterator之后如果modCount发生了变化就会抛出`ConcurrentModificationException`异常。

`ProrityQueue`不允许插入`null`值。

插入，删除头部元素操作的时间复杂度为`O(logn)`。查找和删除指定元素的时间复杂度为`O(n)`。

插入元素的操作

- 超出`capacity`，需要无论在什么位置插入，都需要`O(n)`将原数组复制到更大的数组中，再用`O(1)`进行插入操作，所以时间复杂度为`O(n)`。
- 没有超出`capacity`
  - 在尾部插入`O(1)`
  - 在其他位置`O(n)`

删除元素的操作

- 删除尾部元素`O(1)`
- 删除其他位置`O(n)`

### `RandomAccess`

`ArrayList`实现了`RandomAccess`接口，表示可以随机访问，即可以根据下标访问元素，因为底层是一个数组，数组天生具备随机访问的能力。

## `LinkedList`

`LinkedList`在底层维护了一个**双向链表**来存储数据

```java
transient int size = 0;

transient Node<E> first;

transient Node<E> last;
```

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

每一个`LinkedList`包含两个指针分别指向链表的头节点和末节点。

`RandomAccess`

显然，`LinkedList`并不支持随机访问，因为链表在查找元素时需要线性遍历整个链表，时间复杂度为`O(n)`，不能做到像数组那样`O(1)`的直接访问。

### Methods

对于插入和删除操作主要有两个系列

1. add and remove

- `addFirst()`, `addLast()`
- `removeFirst()`, `removeLast()`

2. offer and poll

- `offerFirst()`, `offerLast()`
- `pollFirst()`, `pollLast()`

主要的区别的在于1在remove的时候如果list为空会抛出异常而2的poll只会返回`false`。

其实当调用2系列方法的时候实际上也会调用1的对应方法，如

```java
public boolean offerLast(E e) {
    addLast(e);
    return true;
}
```

在链表头和尾插入或删除的时间复杂度为`O(1)`。在其他位置插入或删除的时间复杂度均为`O(n)`。

# Queue

队列的最大特点是实现了FIFO（First In First Out）的数据结构，所以很适合在有顺序需求的场景下应用。

在`Queue`接口下还有一个子接口`Deque`，在`Queue`的基础上实现了双端队列的逻辑，同时可以用来模拟`Stack`。

可以用`ArrayDeque`或者`LinkedList`来实现`Queue`接口，但相比之下ArrayDeque不允许null值。值得一提的是当用来模拟`Stack`的时候使用`ArrayDeque`比`Stack`效率高，同时在用作队列时效率比`LinkedList`高。

## `ArrayDeque`

ArrayDeque的底层维护着一个**循环数组**，这就是为什么他可以被用作双端队列。除了`remove`一系列的方法和`contains`，其他的方法都保持在常数级别的复杂度。

```java
transient Object[] elements;

transient int head;

transient int tail;
```

在执行添加操作的时候，只需要移动头指针或者尾指针即可。



# Set

Set主要应用场景是去重。

## `HashSet`

`HashSet`的底层其实维护着一个`HashMap`，只不过插入`HashSet`中的所有元素的Value全部都被设置为了一个`Object`

```java
private static final Object PRESENT = new Object();
```

当执行`add()`操作时，就会讲value设置为上面这个`PRESENT

```java
 public boolean add(E e) {
        return map.put(e, PRESENT)==null;
 }
```

## `TreeSet`

TreeSet底层维护的是一个红黑树，可以实现插入元素的自动排序。

### `Comparator` and `Comparable`

自动排序依赖的是元素实现的`Comparator`或者`Comparable`接口，在构造方法中也可以将实现的`Comparator`接口作为参数传入，并按照实现的接口的方式对`TreeSet`内的元素进行排序。

```java
TreeSet(Comparator<? super E> comparator)
```

假如TreeSet的元素必须实现`Comparator`或者`Comparable`接口，或者在构造方法中传入一个`Comparator`的实现类。否则在向TreeSet中插入元素的时候会发生`ClassCastException`异常。

当在构造方法中传入`Comparator`的时候会覆盖原有方法中的`Comparable`或`Comparator`。

下面是一个例子

```java
// 自定义Person类，实现了Comparable接口，先比较age再比较name
class Person implements Comparable<Person>{
		private String name;

    private Integer age;

    public Person(String name, Integer age) {
        this.name = name;
        this.age = age;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public Integer getAge() {
        return age;
    }

    public void setAge(Integer age) {
        this.age = age;
    }

    @Override
    public int compareTo(Person o) {
        if (this == o) {
            return 0;
        }

        int ageComparison = this.age.compareTo(o.age);
        if (ageComparison != 0) {
            return ageComparison;
        }

        return this.name.compareTo(o.name);
    }

    @Override
    public String toString() {
        return "Person{" +
                "name='" + name + '\'' +
                ", age=" + age +
                '}';
    }
}
```

```java
// 测试
Set<Person> set = new TreeSet<>();
set.add(new Person("Jack", 23));
set.add(new Person("Bob", 24));
set.add(new Person("Amy", 25));
set.add(new Person("Cathy", 26));
set.add(new Person("Sam", 23));
for(Person p: set) {
    System.out.println(p);
}
```

```java
Output:
Person{name='Jack', age=23}
Person{name='Sam', age=23}
Person{name='Bob', age=24}
Person{name='Amy', age=25}
Person{name='Cathy', age=26}
```

```java
// 重写Comparator， 先按照相反的顺序比较age，再比较name
Set<Person> set = new TreeSet<>(new Comparator<Person>() {
    @Override
    public int compare(Person o1, Person o2) {
        if (o1 == o2) {
            return 0;
        }

        int ageComparison = o1.getAge().compareTo(o2.getAge());
        if (ageComparison > 0) {
            return -1;
        } else if (ageComparison < 0){
            return 1;
        }

        return o1.getName().compareTo(o2.getName());
    }
});
```

```java
Output:
Person{name='Cathy', age=26}
Person{name='Amy', age=25}
Person{name='Bob', age=24}
Person{name='Jack', age=23}
Person{name='Sam', age=23}
```

# Map

Map主要是存放类似`<key, value>`这样的键值对的数据结构。

`HashMap`的底层数据结构在Java8之前采用的是数组+链表的形式，数组的每一个元素即为哈希桶，同一个桶中存在多个元素时（发生哈希冲突）将同一个桶中的元素存储在链表中。

Java8之后对链表进行了改进，当链表长度超过阈值（默认值8）之后链表会被转化成红黑树，从而提高查询效率（从`O(n)`提升至`O(logn)`）

## `HashMap`

### `null`值的处理

`HashMap`的Key和Value都可以是`null`（Key只可以有一个`null`，Value可以有多个`null`）。

### 线程安全性

`HashMap`是Map接口的实现类之一。和`HashTable`相比`HashMap`**不是线程安全的**，并且为了避免多线程问题带来的问题，当遇到多线程问题时，应当使用`Collections.synchronizedMap`把`HashMap`包装成线程安全的Map或者直接使用`ConcurrentHashMap`

```java
Map m = Collections.synchronizedMap(new HashMap(...));
ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();
```

下面是一个存在线程安全问题的例子

```java
Map<String, Integer> map = new HashMap<>();
// 第一个线程
Thread thread1 = new Thread() {
    @Override
    public void run() {
        for(int i = 0; i < 1000; i ++) {
            map.put(String.valueOf(i), i);
        }
    }
};
// 第二个线程
Thread thread2 = new Thread() {
    @Override
    public void run() {
        for(int i = 0; i < 1000; i ++) {
            map.put(String.valueOf(i + 1000), i);
        }
    }
};
thread1.start();
thread2.start();
thread1.join();
thread2.join();
System.out.println(map.size());
```

```java
Output:
1992 （总是小于2000）
```

因为在单独使用`HashMap`的情况下，不能保证线程安全。

使用`Collections.synchronizedMap`包装HashMap之后

```java
Map<String, Integer> map = Collections.synchronizedMap(new HashMap<>());
// 第一个线程
Thread thread1 = new Thread() {
    @Override
    public void run() {
        for(int i = 0; i < 1000; i ++) {
            map.put(String.valueOf(i), i);
        }
    }
};
// 第二个线程
Thread thread2 = new Thread() {
    @Override
    public void run() {
        for(int i = 0; i < 1000; i ++) {
            map.put(String.valueOf(i + 1000), i);
        }
    }
};
thread1.start();
thread2.start();
thread1.join();
thread2.join();
System.out.println(map.size());
```

```java
output:
2000
```

总结：Collections.synchronizedMap包装之后的HashMap可以保证线程安全，即在同一时间内只能有一个对map的操作生效，不会发生冲突，或者说所有对map的操作都是同步的，有严格的先后顺序的。

### 遍历顺序可变

此外，HashMap不能保证遍历时的顺序不变（因为Key的存储取决于哈希值，当进行插入，删除或者`Rehashing`等操作的时候会改变Key之间的相对位置，从而改变遍历的顺序）。

### `initial capacity` 和 `load factor`

HashMap有两个参数`initial capacity`和`load factor`,可以在构造函数中进行设置。

```java
HashMap() // 默认initial capacity为16， load factor为0.75
HashMap(int initialCapacity) // 只设置initial capacity
HashMap(int initialCapacity, float loadFactor) // 两个参数都设置
HashMap(Map<? extends K,? extends V> m) // 用另一个map构造HashMap
```

`Initial capacity`是哈希桶的初始数量。

`load factor`是负载因子，用来衡量哈希表的密度。
$$
Load \ Factor(\alpha) = \frac{Number \ of \ Elements}{Capacity \ of \ Hash \ Table}
$$
当负载因子较小时说明哈希表比较稀疏，发生哈希碰撞的可能较小，但需要更多的内存空间。相反，负载因子过大时说明哈希表过于稠密，很容易发生哈希碰撞，降低了查找，插入和删除的效率，但减少了内存消耗。

在HashMap中，`load factor`的默认值为0.75，意味着当HashMap的元素达到当前`capacity`的75%时HashMap就会自动进行`Rehashing`并且**自动扩容**为原来的两倍。

HashMap的基本操作（`get()`, `put()`等）的时间复杂度都是`O(1)`，但这依赖于对两个基本参数`initial capacity`和`load factor`的合理设置。

### Fieids

HashMap的底层是一个`Node<K, V>`类型的数组，这个数组其实就是哈希桶，每一个桶可以存放一个`Node`，实际上就是链表（或红黑树）

```java
transient Node<K,V>[] table;
```

Node是HashMap的一个内部类

```java
static class Node<K,V> implements Map.Entry<K,V> {
        final int hash; // 哈希值
        final K key; // Key
        V value; // Value
        Node<K,V> next; // 后继节点
```

## `LinkedHashMap`

`LinkedHashMap`是`HashMap`的子类，在`HashMap`的基础上添加了**双向链表**来保证遍历的顺序。当在需要保证遍历顺序和插入顺序相同的场景下`LinkedHashMap`非常合适。

### `accessOrder`

`LinkedHashMap`和`HashMap`相比多了一个特别的构造方法

```java
LinkedHashMap(int initialCapacity, float loadFactor, boolean accessOrder)
```

新参数`accessOrder`是一个布尔值，默认为`false`，表示按照**插入顺序**进行遍历，为`true`时表示按照**访问顺序**排序（访问过的元素排在末尾）。可见当按照访问顺序遍历时非常适合用来模拟LRU缓存。

```java
import java.util.LinkedHashMap;
import java.util.Map;

public class LRUCache<K, V> extends LinkedHashMap<K, V> {
    private final Integer capacity;
    public LRUCache(Integer capacity) {
        super(capacity, 0.75f, true);
        this.capacity = capacity;
    }

    @Override
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        return this.size() > capacity;
    }
}
```

`removeEldestEntry`会在`put`方法被调用时被执行，如果返回值为true就会移除首个元素。

下面是测试代码

```java
LRUCache<Integer, Integer> cache = new LRUCache<>(5);

for(int i = 0; i < 5; i ++) {
  cache.put(i, i);
} // 此时的顺序为 0 1 2 3 4
cache.get(0); // 由于设置accessOrder为true， 所以0被放到了末尾，顺序变为 1 2 3 4 0
cache.put(5, 5); // 1 2 3 4 0 5 此时超过了capacity触发removeEldestEntry，移除了首个元素1
for(Integer key: cache.keySet()) {
  System.out.println(key);
}
```

```java
Output:
2 3 4 0 5
```

此外，和`HashMap`不同的是`LinkedHashMap`在遍历的时候不受`capacity`的影响，因为`LinkedHashMap`按照双向链表遍历而不是遍历哈希桶。

同样的，`LinkedHashMap`也不是线性安全的，在多线程并发场景下需要使用`Collections.synchronizedMap()`进行包装。
