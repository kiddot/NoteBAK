# ArrayList和LinkedList



## 区别

1. ArrayList是实现了**基于动态数组**的数据结构，而LinkedList是**基于链表**的数据结构；
2. 对于**随机访问get和set，ArrayList要优于LinkedList**，因为LinkedList要移动指针；
3. 对于添加和删除操作add和remove，一般大家都会说LinkedList要比ArrayList快，因为ArrayList要移动数据。但是实际情况并非这样，对于添加或删除，LinkedList和ArrayList**并不能明确说明谁快谁慢**



#### get

​	ArrayList想要get(int index)元素时，直接返回index位置上的元素，而LinkedList需要通过for循环进行查找，虽然LinkedList已经在查找方法上做了优化，比如index < size / 2，则从左边开始查找，反之从右边开始查找，但是还是比ArrayList要慢。这点是毋庸置疑的。

#### insert or remove

​        ArrayList想要在指定位置插入或删除元素时，主要耗时的是System.arraycopy动作，会移动index后面所有的元素；LinkedList主耗时的是要先通过for循环找到index，然后直接插入或删除。这就导致了两者并非一定谁快谁慢。 **主要有两个因素决定他们的效率，插入的数据量和插入的位置。**



#### ArrayList扩容问题

​	ArrayList使用一个内置的数组来存储元素，这个数组的起始容量是10.当数组需要增长时，新的容量按如下公式获得：新容量=(旧容量*3)/2+1，也就是说每一次容量大概会增长50%。这就意味着，如果你有一个包含大量元素的ArrayList对象，那么最终将有很大的空间会被浪费掉，这个浪费是由ArrayList的工作方式本身造成的。如果没有足够的空间来存放新的元素，数组将不得不被重新进行分配以便能够增加新的元素。**对数组进行重新分配，将会导致性能急剧下降。**





# 回答梳理

## 相同点

都是java集合框架，都实现List接口



## 不同点

1. ArrayList是实现了**基于动态数组**的数据结构，它使用索引在数组中搜索和读取数据是很快的。Array获取数据的时间复杂度是O(1),但是要删除数据却是开销很大的，因为这需要重排数组中的所有数据。
2. LinkedList是**基于链表**的数据结构，对于添加和删除操作add和remove，一般大家都会说LinkedList要比ArrayList快，因为ArrayList要移动数据。但是实际情况并非这样，对于添加或删除，LinkedList和ArrayList**并不能明确说明谁快谁慢**
3.  LinkedList需要更多的内存，因为ArrayList的每个索引的位置是实际的数据，而LinkedList中的每个节点中存储的是实际的数据和前后节点的位置。



## 效率

ArrayList：

​	1.**内部的Object类型的影响** ，对于值类型来说，往ArrayList里面添加和修改元素，都会引起装箱和拆箱的操作，频繁的操作可能会影响一部分效率。

​	2.**数组扩容** ，扩容操作往往会导致不必要的空间浪费，尽量去评估自己需要的容量

​	3.**频繁的调用IndexOf、Contains等方法**（Sort、BinarySearch等方法经过优化，不在此列）引起的效率损失 



## 使用场景

使用LinkList的场景：

​	1.不会随机访问数据。因为如果你需要LinkedList中的第n个元素的时候，你需要从第一个元素顺序数到第n个数据，然后读取数据。

​	2.插入和删除元素的操作比较多，读取数据比较少的情况。因为插入和删除元素不涉及重排数据，所以它要比ArrayList要快。



## 扩展

多线问题



#### ArrayList中的操作不是线程安全的。所以，建议在单线程中才使用ArrayList，而在多线程中可以选择CopyOnWriteArrayList。

