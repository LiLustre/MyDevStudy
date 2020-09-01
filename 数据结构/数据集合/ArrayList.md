[原文](https://www.jianshu.com/p/192148ddab9f)

### 概述

ArrayList是一个动态数组，它的底层数据结构是数组，他实现了 `List<E>、RandomAccess、 Cloneable、java.io.Serializable`接口，其中`RandomAccess`代表了其拥有**随机快速访问**的能力，`ArrayList'可以以O(1)的时间复杂度去根据下标访问元素。

因为它底层数据结构是数组，所以可想而知，它是有一个容量的（数组的`length`），当集合中的元素超出这个容量，便会进行扩容操作。扩容操作也是`ArrayList` 的一个性能消耗比较大的地方，所以若**我们可以提前预知数据的规模**，应该通过`public ArrayList(int initialCapacity) {}`构造方法，指定集合的大小，去构建`ArrayList`实例**，以减少扩容次数，提高效率**。或者在需要扩容的时候，手动调用`public void ensureCapacity(int minCapacity) {}`方法扩容。
 不过该方法是`ArrayList`的API，不是`List`接口里的，所以使用时需要强转:
 `((ArrayList)list).ensureCapacity(30);`