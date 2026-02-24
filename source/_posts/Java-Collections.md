---
title: Java集合原理和底層數據結構總結
date: 2024-08-25 23:05:58
tags:
  - Java
  - Java Collections
categories:
  - [Java SE, Collections]
cover: https://pics.findfuns.org/java-collections.png
---

# 先放一張集合的層級圖

<img src="https://pics.findfuns.org/Collections-hierarchy.png" alt="collectionsHierarchy" style="zoom:50%;" />

# List

List的特點是順序性，可重複性。

## `ArrayList`

允許任何元素包括`null`。和`Vector`相比**不是線程安全的**。在必要時可以用`Collections.synchronizedList()`來將`ArrayList`轉換成線程安全的List

```java
List list = Collections.synchronizedList(new ArrayList());
```

底層維護了一個`Object`數組用來存儲元素。

```Java
transient Object[] elementData;
```

定義了一個`grow()`方法，當調用一繫列重載的`add`方法時，會根據是否到達`capacity`來調用`grow()`以保証`ArrayList`的容量滿足需求。

## `PriorityQueue`

PriorityQueue通過維護了一個小頂堆來保証首元素是所有元素中最小的元素，最小取決於元素實現的`Comparator`或者`Comparable`。它也提供了一個構造函數以便傳入自定義的`Comparator`來決定排序的順序。

```java
public PriorityQueue(Comparator<? super E> comparator)
```

```java
transient Object[] queue;

int size;

private final Comparator<? super E> comparator;

transient int modCount; 
```

這裡的modCount是一個用來記錄restructure（添加，刪除或改變集合中元素的排列順序）次數的變量，當創建了iterator之後如果modCount髮生了變化就會拋出`ConcurrentModificationException`異常。

`ProrityQueue`不允許插入`null`值。

插入，刪除頭部元素操作的時間複雜度爲`O(logn)`。查找和刪除指定元素的時間複雜度爲`O(n)`。

插入元素的操作

- 超出`capacity`，需要無論在什麼位置插入，都需要`O(n)`將原數組複製到更大的數組中，再用`O(1)`進行插入操作，所以時間複雜度爲`O(n)`。
- 沒有超出`capacity`
    - 在尾部插入`O(1)`
    - 在其他位置`O(n)`

刪除元素的操作

- 刪除尾部元素`O(1)`
- 刪除其他位置`O(n)`

### `RandomAccess`

`ArrayList`實現了`RandomAccess`接口，表示可以隨機訪問，即可以根據下標訪問元素，因爲底層是一個數組，數組天生具備隨機訪問的能力。

## `LinkedList`

`LinkedList`在底層維護了一個**雙向鏈表**來存儲數據

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

每一個`LinkedList`包含兩個指針分別指向鏈表的頭節點和末節點。

`RandomAccess`

顯然，`LinkedList`並不支持隨機訪問，因爲鏈表在查找元素時需要線性遍曆整個鏈表，時間複雜度爲`O(n)`，不能做到像數組那樣`O(1)`的直接訪問。

### Methods

對於插入和刪除操作主要有兩個繫列

1. add and remove

- `addFirst()`, `addLast()`
- `removeFirst()`, `removeLast()`

2. offer and poll

- `offerFirst()`, `offerLast()`
- `pollFirst()`, `pollLast()`

主要的區別的在於1在remove的時候如果list爲空會拋出異常而2的poll隻會返回`false`。

其實當調用2繫列方法的時候實際上也會調用1的對應方法，如

```java
public boolean offerLast(E e) {
    addLast(e);
    return true;
}
```

在鏈表頭和尾插入或刪除的時間複雜度爲`O(1)`。在其他位置插入或刪除的時間複雜度均爲`O(n)`。

# Queue

隊列的最大特點是實現了FIFO（First In First Out）的數據結構，所以很適合在有順序需求的場景下應用。

在`Queue`接口下還有一個子接口`Deque`，在`Queue`的基礎上實現了雙端隊列的邏輯，同時可以用來模擬`Stack`。

可以用`ArrayDeque`或者`LinkedList`來實現`Queue`接口，但相比之下ArrayDeque不允許null值。值得一提的是當用來模擬`Stack`的時候使用`ArrayDeque`比`Stack`效率高，同時在用作隊列時效率比`LinkedList`高。

## `ArrayDeque`

ArrayDeque的底層維護着一個**循環數組**，這就是爲什麼他可以被用作雙端隊列。除了`remove`一繫列的方法和`contains`，其他的方法都保持在常數級別的複雜度。

```java
transient Object[] elements;

transient int head;

transient int tail;
```

在執行添加操作的時候，隻需要移動頭指針或者尾指針即可。



# Set

Set主要應用場景是去重。

## `HashSet`

`HashSet`的底層其實維護着一個`HashMap`，隻不過插入`HashSet`中的所有元素的Value全部都被設置爲了一個`Object`

```java
private static final Object PRESENT = new Object();
```

當執行`add()`操作時，就會講value設置爲上麵這個`PRESENT

```java
 public boolean add(E e) {
        return map.put(e, PRESENT)==null;
 }
```

## `TreeSet`

TreeSet底層維護的是一個紅黑樹，可以實現插入元素的自動排序。

### `Comparator` and `Comparable`

自動排序依賴的是元素實現的`Comparator`或者`Comparable`接口，在構造方法中也可以將實現的`Comparator`接口作爲參數傳入，並按照實現的接口的方式對`TreeSet`內的元素進行排序。

```java
TreeSet(Comparator<? super E> comparator)
```

假如TreeSet的元素必須實現`Comparator`或者`Comparable`接口，或者在構造方法中傳入一個`Comparator`的實現類。否則在向TreeSet中插入元素的時候會髮生`ClassCastException`異常。

當在構造方法中傳入`Comparator`的時候會覆蓋原有方法中的`Comparable`或`Comparator`。

下麵是一個例子

```java
// 自定義Person類，實現了Comparable接口，先比較age再比較name
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
                "name=''" + name + ''\'''' +
                ", age=" + age +
                ''}'';
    }
}
```

```java
// 測試
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
Person{name=''Jack'', age=23}
Person{name=''Sam'', age=23}
Person{name=''Bob'', age=24}
Person{name=''Amy'', age=25}
Person{name=''Cathy'', age=26}
```

```java
// 重冩Comparator， 先按照相反的順序比較age，再比較name
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
Person{name=''Cathy'', age=26}
Person{name=''Amy'', age=25}
Person{name=''Bob'', age=24}
Person{name=''Jack'', age=23}
Person{name=''Sam'', age=23}
```

# Map

Map主要是存放類似`<key, value>`這樣的鍵值對的數據結構。

`HashMap`的底層數據結構在Java8之前採用的是數組+鏈表的形式，數組的每一個元素即爲哈希桶，同一個桶中存在多個元素時（髮生哈希衝突）將同一個桶中的元素存儲在鏈表中。

Java8之後對鏈表進行了改進，當鏈表長度超過閾值（默認值8）之後鏈表會被轉化成紅黑樹，從而提高查詢效率（從`O(n)`提昇至`O(logn)`）

## `HashMap`

### `null`值的處理

`HashMap`的Key和Value都可以是`null`（Key隻可以有一個`null`，Value可以有多個`null`）。

### 線程安全性

`HashMap`是Map接口的實現類之一。和`HashTable`相比`HashMap`**不是線程安全的**，並且爲了避免多線程問題帶來的問題，當遇到多線程問題時，應當使用`Collections.synchronizedMap`把`HashMap`包裝成線程安全的Map或者直接使用`ConcurrentHashMap`

```java
Map m = Collections.synchronizedMap(new HashMap(...));
ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();
```

下麵是一個存在線程安全問題的例子

```java
Map<String, Integer> map = new HashMap<>();
// 第一個線程
Thread thread1 = new Thread() {
    @Override
    public void run() {
        for(int i = 0; i < 1000; i ++) {
            map.put(String.valueOf(i), i);
        }
    }
};
// 第二個線程
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
1992 （總是小於2000）
```

因爲在單獨使用`HashMap`的情況下，不能保証線程安全。

使用`Collections.synchronizedMap`包裝HashMap之後

```java
Map<String, Integer> map = Collections.synchronizedMap(new HashMap<>());
// 第一個線程
Thread thread1 = new Thread() {
    @Override
    public void run() {
        for(int i = 0; i < 1000; i ++) {
            map.put(String.valueOf(i), i);
        }
    }
};
// 第二個線程
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

總結：Collections.synchronizedMap包裝之後的HashMap可以保証線程安全，即在同一時間內隻能有一個對map的操作生效，不會髮生衝突，或者説所有對map的操作都是同步的，有嚴格的先後順序的。

### 遍曆順序可變

此外，HashMap不能保証遍曆時的順序不變（因爲Key的存儲取決於哈希值，當進行插入，刪除或者`Rehashing`等操作的時候會改變Key之間的相對位置，從而改變遍曆的順序）。

### `initial capacity` 和 `load factor`

HashMap有兩個參數`initial capacity`和`load factor`,可以在構造函數中進行設置。

```java
HashMap() // 默認initial capacity爲16， load factor爲0.75
HashMap(int initialCapacity) // 隻設置initial capacity
HashMap(int initialCapacity, float loadFactor) // 兩個參數都設置
HashMap(Map<? extends K,? extends V> m) // 用另一個map構造HashMap
```

`Initial capacity`是哈希桶的初始數量。

`load factor`是負載因子，用來衡量哈希表的密度。
$$
Load \ Factor(\alpha) = \frac{Number \ of \ Elements}{Capacity \ of \ Hash \ Table}
$$
當負載因子較小時説明哈希表比較稀疏，髮生哈希碰撞的可能較小，但需要更多的內存空間。相反，負載因子過大時説明哈希表過於稠密，很容易髮生哈希碰撞，降低了查找，插入和刪除的效率，但減少了內存消耗。

在HashMap中，`load factor`的默認值爲0.75，意味着當HashMap的元素達到當前`capacity`的75%時HashMap就會自動進行`Rehashing`並且**自動擴容**爲原來的兩倍。

HashMap的基本操作（`get()`, `put()`等）的時間複雜度都是`O(1)`，但這依賴於對兩個基本參數`initial capacity`和`load factor`的合理設置。

### Fieids

HashMap的底層是一個`Node<K, V>`類型的數組，這個數組其實就是哈希桶，每一個桶可以存放一個`Node`，實際上就是鏈表（或紅黑樹）

```java
transient Node<K,V>[] table;
```

Node是HashMap的一個內部類

```java
static class Node<K,V> implements Map.Entry<K,V> {
        final int hash; // 哈希值
        final K key; // Key
        V value; // Value
        Node<K,V> next; // 後繼節點
```

## `LinkedHashMap`

`LinkedHashMap`是`HashMap`的子類，在`HashMap`的基礎上添加了**雙向鏈表**來保証遍曆的順序。當在需要保証遍曆順序和插入順序相同的場景下`LinkedHashMap`非常合適。

### `accessOrder`

`LinkedHashMap`和`HashMap`相比多了一個特別的構造方法

```java
LinkedHashMap(int initialCapacity, float loadFactor, boolean accessOrder)
```

新參數`accessOrder`是一個佈爾值，默認爲`false`，表示按照**插入順序**進行遍曆，爲`true`時表示按照**訪問順序**排序（訪問過的元素排在末尾）。可見當按照訪問順序遍曆時非常適合用來模擬LRU緩存。

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

`removeEldestEntry`會在`put`方法被調用時被執行，如果返回值爲true就會移除首個元素。

下麵是測試代碼

```java
LRUCache<Integer, Integer> cache = new LRUCache<>(5);

for(int i = 0; i < 5; i ++) {
  cache.put(i, i);
} // 此時的順序爲 0 1 2 3 4
cache.get(0); // 由於設置accessOrder爲true， 所以0被放到了末尾，順序變爲 1 2 3 4 0
cache.put(5, 5); // 1 2 3 4 0 5 此時超過了capacity觸髮removeEldestEntry，移除了首個元素1
for(Integer key: cache.keySet()) {
  System.out.println(key);
}
```

```java
Output:
2 3 4 0 5
```

此外，和`HashMap`不同的是`LinkedHashMap`在遍曆的時候不受`capacity`的影響，因爲`LinkedHashMap`按照雙向鏈表遍曆而不是遍曆哈希桶。

同樣的，`LinkedHashMap`也不是線性安全的，在多線程並髮場景下需要使用`Collections.synchronizedMap()`進行包裝。
