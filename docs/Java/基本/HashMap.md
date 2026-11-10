# what

HashMap 是基于哈希表实现的键值对容器：`key` 唯一、`value` 可重复，平均 O(1) 查找/插入，不保证顺序，非线程安全。

一句话记住 JDK8 的结构：**数组 + 链表 + 红黑树**。

![.hashMap.png](hashMap.png)

# 一篇背完：高频八股

## 1. 为什么数组长度必须是 2 的幂？

下标靠 `index = (n - 1) & hash` 计算。n 是 2 的幂时，`n - 1` 的低位全是 1，位与等价于 `hash % n`，但更快。

同时扩容时可以用 `hash & oldCap` 判断元素留在原位还是移到 `原下标 + oldCap`，不必重新计算 hash。

## 2. 扰动函数是干什么的？

```java
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

`hashCode` 的高 16 位在取模时会被丢掉，`h ^ (h >>> 16)` 把高位混进低位，让高位也参与下标计算，减少冲突。

## 3. put 的流程（背这一句）

计算 hash 和下标 → 桶为空直接放 → key 相同（`==` 或 `equals`）则覆盖 → 树节点走树插入 → 否则尾插链表 → 链表 ≥ 8 且容量 ≥ 64 时树化 → `++size > threshold` 时扩容。

## 4. 扩容怎么做的？

- 容量翻倍，阈值也翻倍（`threshold = capacity × 0.75`）；
- JDK8：用 `(e.hash & oldCap) == 0` 把原链表拆成**低位链**和**高位链**：低位留在原下标，高位移到 `原下标 + oldCap`，不用重新算 hash；
- JDK7：重新计算 hash（rehash）+ 头插。

## 5. 为什么负载因子是 0.75？

时间与空间的折中：太高冲突多、链表长；太低浪费空间。0.75 下，一个桶里放 8 个节点的概率约 `6e-8`（泊松分布），所以树化阈值取 8。

## 6. 树化和退化的阈值

| 参数 | 值 | 含义 |
|---|---|---|
| `TREEIFY_THRESHOLD` | 8 | 链表长度达到 8 尝试树化 |
| `MIN_TREEIFY_CAPACITY` | 64 | 容量不足 64 时先扩容，不树化 |
| `UNTREEIFY_THRESHOLD` | 6 | 节点减少到 6 退化回链表 |

6 和 8 之间留了缓冲，避免在边界反复转换。

## 7. HashMap 为什么线程不安全？

- JDK7：扩容用**头插**，并发 transfer 可能形成环形链表，`get` 时死循环，还会丢数据；
- JDK8：改成**尾插**，不会成环，但并发 put 仍会互相覆盖、丢数据，`size` 也不准。

## 8. 想要线程安全怎么办？

Hashtable（方法级 synchronized，性能差）、`Collections.synchronizedMap`（mutex 加锁）、**ConcurrentHashMap（首选）**：JDK8 用 CAS + synchronized 锁单个桶。

## 9. 什么是 fail-fast？

迭代器内部记录 `modCount`，迭代过程中若结构被修改（put/remove），会抛 `ConcurrentModificationException`。用 `iterator.remove()` 是安全的。

## 10. 允许 null 吗？

HashMap 允许 **1 个 null key**（hash 记 0，放在下标 0）和**多个 null value**；Hashtable 两者都不允许。

## 11. 对 key 有什么要求？

必须正确重写 `hashCode()` 和 `equals()`（契约：equals 相等，hashCode 必须相等），并且尽量用不可变对象——key 的 hashCode 变了就再也定位不到了。

## 12. HashMap / Hashtable / synchronizedMap

| | HashMap | Hashtable | synchronizedMap |
|---|---|---|---|
| 线程安全 | 否 | 是（方法 synchronized） | 是（mutex） |
| null key/value | 允许 | 不允许 | 取决于被包装的 Map |
| 迭代器 | fail-fast | 非 fail-fast | fail-fast |
| 性能 | 高 | 低 | 中 |

## 13. 为什么 Hashtable 不允许 null？

源码里直接判空并抛 `NullPointerException`；常见解释是并发场景下无法区分「key 不存在」和「value 为 null」。

## 14. JDK7 和 JDK8 的区别

| | JDK7 | JDK8 |
|---|---|---|
| 结构 | 数组 + 链表 | 数组 + 链表 + 红黑树 |
| 插入 | 头插 | 尾插 |
| hash | 4 次移位异或 | 一次 `h ^ (h >>> 16)` |
| 扩容 | rehash | 高低位拆分，不重算 |
| 并发 | 可能成环死循环 | 不成环，但会覆盖/丢数据 |

## 15. 冲突有哪些解决方式？

- 链地址法：冲突的桶挂链表（HashMap 用的就是它）；
- 开放地址法：从冲突位置往后找空位（线性探测、二次探测等）；
- 再哈希法：换一个哈希函数再算；
- 公共溢出区：冲突的放另一块区域。

# 原理实现（了解即可）

## 散列表与冲突

不用 HashMap 时，自己用数组做散列大致是这样：

```java
String[] tab = new String[16];          // 容量取 2 的幂
for (String s : new String[]{"abc", "def", "xiao"}) {
    int idx = s.hashCode() & (tab.length - 1);
    tab[idx] = (tab[idx] == null) ? s : tab[idx] + "->" + s;
}
```

不同 key 可能落到同一下标，冲突的元素用链表串起来，这就是链地址法。注意数组长度必须取 2 的幂，否则 `& (length - 1)` 不能当取模用。

## 扩容为什么不用重算 hash

扩容到两倍后，多出来的那一位正好是 `oldCap`。所以只需要看 `hash & oldCap`：

- 为 0：下标不变；
- 不为 0：新下标 = 旧下标 + oldCap。

一条链表因此被拆成「低位链」和「高位链」，这就是 JDK8 的扩容优化。

## 树化

链表长度 ≥ 8 且 `table.length ≥ 64` 时转红黑树；节点数 ≤ 6 时退化回链表。红黑树通过左旋、右旋、染色维持平衡，把最坏查找从 O(n) 降到 O(log n)。

# 参考

- JDK8 `HashMap` 源码
