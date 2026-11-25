# 链表、哈希与 LRU：指针与索引的组合拳

> 定位：链表题考的是指针操作与边界，哈希题考的是索引设计与冲突处理，LRU 则把两者组合成一个完整的工程结构。这一篇把链表常用操作、哈希表底层、LRU/LFU 的实现与线程安全一次讲清，代码可直接用于面试手写。

## 一、链表：先把边界想清楚

链表题的错误，八成不是算法错，而是边界漏了：

- 空链表；
- 只有一个节点；
- 两个节点；
- 删除头节点；
- 删除尾节点；
- 链表成环。

通用技巧：**加哑节点 `dummy`**，让头节点不再特殊；**画三格图**（prev / cur / next），每次改指针前先保存 next。

```java
class ListNode {
    int val;
    ListNode next;
    ListNode(int val) { this.val = val; }
}
```

## 二、五个基本操作

### 2.1 反转链表（迭代）

```java
ListNode reverse(ListNode head) {
    ListNode prev = null, cur = head;
    while (cur != null) {
        ListNode next = cur.next; // 先保存
        cur.next = prev;          // 改指针
        prev = cur;               // 前移
        cur = next;
    }
    return prev;
}
```

递归版：

```java
ListNode reverseRec(ListNode head) {
    if (head == null || head.next == null) return head;
    ListNode newHead = reverseRec(head.next);
    head.next.next = head;
    head.next = null;
    return newHead;
}
```

递归深度是 n，长链表会栈溢出；面试优先写迭代。

### 2.2 合并两个有序链表

```java
ListNode merge(ListNode a, ListNode b) {
    ListNode dummy = new ListNode(0), tail = dummy;
    while (a != null && b != null) {
        if (a.val <= b.val) { tail.next = a; a = a.next; }
        else { tail.next = b; b = b.next; }
        tail = tail.next;
    }
    tail.next = a != null ? a : b;
    return dummy.next;
}
```

`<=` 保证稳定；剩余部分直接接上，不需要再遍历。

### 2.3 删除倒数第 N 个

```java
ListNode removeNthFromEnd(ListNode head, int n) {
    ListNode dummy = new ListNode(0);
    dummy.next = head;
    ListNode fast = dummy, slow = dummy;
    for (int i = 0; i <= n; i++) fast = fast.next; // 走 n+1 步
    while (fast != null) {
        fast = fast.next;
        slow = slow.next;
    }
    slow.next = slow.next.next;
    return dummy.next;
}
```

快指针先走 `n+1` 步，慢指针最后停在待删节点的前一个；哑节点保证删头节点时也有前驱。

### 2.4 找中点与判环

```java
ListNode middle(ListNode head) {
    ListNode slow = head, fast = head;
    while (fast != null && fast.next != null) {
        slow = slow.next;
        fast = fast.next.next;
    }
    return slow; // 偶数个节点时返回后半段第一个
}
```

判环后找入口：相遇后，一个指针回头部，两个指针同速走，再次相遇即入口。证明用「慢指针走了 a+b，快指针走了 a+b+kL」，两者差 2 倍，可推出 a = kL - b。

### 2.5 判断回文链表

三步：找中点 → 反转后半段 → 逐个比较。比较完可以把后半段再反转回去（可选，保持原结构）。空间 O(1)，时间 O(n)。

## 三、哈希表：O(1) 是有条件的

### 3.1 底层结构

Java 8 的 `HashMap` 是「数组 + 链表 + 红黑树」：

- `hash(key)` 高低 16 位异或，减少碰撞；
- 下标 `(n - 1) & hash`，容量是 2 的幂；
- 链表长度 ≥ 8 且容量 ≥ 64 时转红黑树，降到 6 退回链表；
- 负载因子 0.75，超过阈值扩容为 2 倍。

### 3.2 为什么容量是 2 的幂

`n` 是 2 的幂时，`n-1` 的低位全 1，`(n-1) & hash` 等价于取模且更快；扩容时元素要么留在原位，要么移到「原位置 + oldCap」，不需要重新计算完整哈希。

### 3.3 工程坑

- **可变对象做 key**：修改字段后 `hashCode` 变了，`get` 不到；
- **只重写 `equals` 不重写 `hashCode`**：HashMap 直接失效；
- **哈希碰撞攻击**：大量同槽 key 退化为链表；Java 8 转红黑树，但仍要限制输入规模；
- **扩容**：负载因子 0.75，扩容 O(n)，大 Map 扩容会卡顿；预估容量 `(expected/0.75)+1`；
- **并发**：`HashMap` 非线程安全，JDK 7 头插法并发扩容会成环；JDK 8 改尾插仍会丢数据。并发用 `ConcurrentHashMap`。

### 3.4 ConcurrentHashMap 的关键点

- JDK 7：Segment 分段锁，并发度受段数限制；
- JDK 8：CAS + synchronized + 桶级锁，锁粒度是桶，并发度随数组长度增长；
- `size()` 是近似值，用 `baseCount + CounterCell[]` 分散热点；
- 扩容支持多线程协助迁移。

## 四、手写 LRU：HashMap + 双向链表

### 4.1 为什么需要双向链表

`get` 命中后要把节点移到头部，`put` 超容量要删尾节点。这两个操作都需要 O(1) 删除任意节点：单链表删节点需要前驱，做不到 O(1)；双向链表有 `prev`，可以。

### 4.2 完整实现

```java
class LRUCache {
    class Node {
        int k, v;
        Node prev, next;
        Node(int k, int v) { this.k = k; this.v = v; }
    }

    private final int cap;
    private final Map<Integer, Node> map = new HashMap<>();
    private final Node head = new Node(0, 0), tail = new Node(0, 0);

    public LRUCache(int capacity) {
        cap = capacity;
        head.next = tail;
        tail.prev = head;
    }

    private void remove(Node n) {
        n.prev.next = n.next;
        n.next.prev = n.prev;
    }

    private void addFirst(Node n) {
        n.next = head.next;
        n.prev = head;
        head.next.prev = n;
        head.next = n;
    }

    public int get(int key) {
        Node n = map.get(key);
        if (n == null) return -1;
        remove(n);
        addFirst(n);
        return n.v;
    }

    public void put(int key, int value) {
        Node n = map.get(key);
        if (n != null) {
            n.v = value;
            remove(n);
            addFirst(n);
            return;
        }
        if (map.size() == cap) {
            Node last = tail.prev;
            remove(last);
            map.remove(last.k);
        }
        Node node = new Node(key, value);
        map.put(key, node);
        addFirst(node);
    }
}
```

复杂度：`get/put` 均摊 O(1)，空间 O(capacity)。

### 4.3 常见追问

- **线程安全**：方法加锁、分段锁，或 `ConcurrentHashMap` + 同步链表；高并发下锁竞争会抵消 O(1) 优势。
- **Redis 的近似 LRU**：采样若干 key，淘汰最久未用的，不维护完整链表，省内存。
- **LFU**：按访问频率淘汰，需要频率桶 + 双向链表；比 LRU 更抗突发，但实现复杂。
- **过期与淘汰**：LRU 解决容量，TTL 解决时效，二者要配合。

## 五、链表与哈希的工程映射

| 场景 | 结构 | 原因 |
|---|---|---|
| 浏览器/本地缓存 | HashMap + 双向链表 | O(1) 读写与淘汰 |
| 消息去重 | HashSet/Bloom | 快速判断是否出现过 |
| 排行榜 | TreeSet/跳表 | 有序、范围、排名 |
| 路由表 | Trie | 前缀匹配 |
| 内存对象池 | 双向链表/队列 | 频繁分配释放 |
| 数据库索引 | B+ 树 | 磁盘页与范围查询 |

## 六、LFU：按频率淘汰

LRU 淘汰「最久未用」，LFU 淘汰「使用次数最少」。频次相同再按 LRU。

### 6.1 结构

- `Map<Integer, Node> keyMap`：key → 节点；
- `Map<Integer, DoublyList> freqMap`：频率 → 该频率的双向链表；
- `minFreq`：当前最小频率，淘汰时从 `freqMap.get(minFreq)` 的尾部删。

节点需要 `freq` 字段；`get/put` 命中后把节点从旧频率链表移到 `freq+1` 的链表头部。

### 6.2 关键操作

```java
class LFUCache {
    class Node {
        int k, v, freq;
        Node prev, next;
        Node(int k, int v) { this.k = k; this.v = v; this.freq = 1; }
    }
    class DList {
        Node head = new Node(0, 0), tail = new Node(0, 0);
        int size;
        DList() { head.next = tail; tail.prev = head; }
        void addFirst(Node n) {
            n.next = head.next; n.prev = head;
            head.next.prev = n; head.next = n; size++;
        }
        void remove(Node n) {
            n.prev.next = n.next; n.next.prev = n.prev; size--;
        }
        Node removeLast() {
            Node n = tail.prev;
            remove(n);
            return n;
        }
    }

    private final int cap;
    private int minFreq;
    private final Map<Integer, Node> keyMap = new HashMap<>();
    private final Map<Integer, DList> freqMap = new HashMap<>();

    public LFUCache(int capacity) { cap = capacity; }

    public int get(int key) {
        Node n = keyMap.get(key);
        if (n == null) return -1;
        touch(n);
        return n.v;
    }

    public void put(int key, int value) {
        if (cap == 0) return;
        Node n = keyMap.get(key);
        if (n != null) {
            n.v = value;
            touch(n);
            return;
        }
        if (keyMap.size() == cap) {
            DList list = freqMap.get(minFreq);
            Node old = list.removeLast();
            keyMap.remove(old.k);
            if (list.size == 0) freqMap.remove(minFreq);
        }
        Node node = new Node(key, value);
        keyMap.put(key, node);
        freqMap.computeIfAbsent(1, x -> new DList()).addFirst(node);
        minFreq = 1;
    }

    private void touch(Node n) {
        DList oldList = freqMap.get(n.freq);
        oldList.remove(n);
        if (oldList.size == 0 && n.freq == minFreq) minFreq++;
        n.freq++;
        freqMap.computeIfAbsent(n.freq, x -> new DList()).addFirst(n);
    }
}
```

复杂度：`get/put` O(1)，空间 O(capacity)。易错点：

- `minFreq` 只在「当前最小频率链表空了」时递增；
- 删除节点后要同步 `keyMap`；
- 频率链表的头部是最近使用，尾部是最久未用。

### 6.3 LFU 的局限

- 新数据容易饿死（频率低），需要「频率增长/老化」；
- 突发热点会把历史低频但常用的数据挤掉；
- 实现比 LRU 复杂，工程上常用 TinyLFU/Window-TinyLFU（Caffeine）。

## 七、LRU 的工程变体

### 7.1 2Q：区分「刚进来」和「证明过自己」

2Q 用两个队列：

- `A1`（probation）：新数据先进；
- `Am`（protected）：被再次访问的数据晋升。

淘汰优先从 `A1` 尾部开始。它能抵抗「一次性扫描」污染缓存。

### 7.2 LRU-K

记录每个 key 最近 K 次访问时间，只有访问次数达到 K 才进入缓存。适合「偶发访问不应污染缓存」的场景，但需要维护访问历史。

### 7.3 Redis 的近似 LRU

Redis 不维护完整双向链表，而是采样 `maxmemory-samples` 个 key，淘汰其中最久未用的。牺牲精确性换内存与 CPU。

Redis 还有 LFU 模式：用概率计数器记录访问频率，支持 `lfu-decay-time` 衰减。

### 7.4 选型

| 场景 | 方案 |
|---|---|
| 通用本地缓存 | LRU / Caffeine |
| 抗扫描污染 | 2Q / TinyLFU |
| 热点稳定、长尾少 | LFU |
| 分布式缓存 | Redis 近似 LRU/LFU |
| 强一致淘汰 | 自己实现 + 锁/分片 |

## 八、哈希契约与并发

### 8.1 equals/hashCode 契约

- `equals` 相等 → `hashCode` 必须相等；
- `hashCode` 相等 → `equals` 不一定相等；
- 重写 `equals` 必须重写 `hashCode`；
- `equals` 要满足自反、对称、传递、一致、非空。

违反契约的后果：HashSet 去重失效、HashMap get 不到、TreeMap 比较异常。

### 8.2 可变对象做 key

```java
List<Integer> key = new ArrayList<>(List.of(1, 2));
map.put(key, "x");
key.add(3); // hashCode 变了
map.get(key); // 找不到
```

key 的哈希值在放入后不能变；需要可变 key 时，用不可变快照或自定义稳定哈希。

### 8.3 哈希碰撞攻击

攻击者构造大量同槽 key，让 HashMap 退化为链表，CPU 被打满。防护：

- 限制请求体大小/参数个数；
- 用随机哈希种子（部分框架支持）；
- 升级到树化实现；
- 对不可信输入做长度/字符集限制。

### 8.4 ConcurrentHashMap 的并发语义

- `get` 无锁，依赖 `volatile` 读；
- `put` 桶级锁 + CAS；
- `size()` 是近似值，并发下不精确；
- 复合操作（先查后改）要用 `computeIfAbsent`/`merge` 保证原子；
- 不允许 null key/value。

## 面试高频问法

**Q1：手写 LRU，为什么需要双向链表？**

一句话结论：需要 O(1) 删除任意节点并把节点移到头部；单链表删节点要前驱，做不到 O(1)。

展开：`HashMap<key, Node>` 定位节点，双向链表维护顺序；`head` 最近使用，`tail` 最久未用；容量满删 `tail.prev`。

加分点：能讲线程安全方案与 Redis 近似 LRU 的差异。

**Q2：HashMap 为什么容量是 2 的幂？**

一句话结论：为了用 `(n-1) & hash` 代替取模，并且扩容时元素要么原位、要么移到「原位 + oldCap」。

展开：`n-1` 低位全 1，按位与等价于取模；扩容只看 hash 的第 log₂n 位。

加分点：能说出「树化阈值 8、退化阈值 6、最小树化容量 64」。

**Q3：HashMap 线程不安全会怎样？**

一句话结论：JDK 7 并发扩容可能成环导致死循环；JDK 8 改尾插仍会丢数据、size 不准。

展开：并发写用 `ConcurrentHashMap`；只读并发可以共享 `HashMap`，但发布后不能再改。

加分点：能说出 CHM 7 的分段锁与 8 的桶级锁差异。

**Q4：链表反转的递归和迭代怎么选？**

一句话结论：面试优先迭代，O(1) 空间、无栈溢出；递归代码短，但深度 n 有栈溢出风险。

展开：迭代用 prev/cur/next 三指针；递归先反转后续，再改 head.next。

加分点：能说出「递归深度等于链表长度，长链表会 StackOverflow」。

**Q5：如何判断链表有环并找入口？**

一句话结论：快慢指针相遇说明有环；相遇后一个指针回头部，同速再走，第二次相遇即入口。

展开：快指针每次两步、慢指针一步；无环时快指针先到 null。

加分点：能写出数学证明或直观解释「a = kL - b」。

## 小结

- **链表题先处理边界**：空、单节点、头尾删除；哑节点省特判，画图防断链。
- **五个基本操作**：反转、合并、删倒数第 N、找中点、判环，都是面试高频。
- **哈希 O(1) 有前提**：哈希函数均匀、容量足够、key 不可变；扩容是均摊 O(1)。
- **LRU = HashMap + 双向链表**：Map 定位、链表排序，双向是为了 O(1) 删除。
- **并发场景别用 HashMap**：JDK 7 成环、JDK 8 丢数据，用 `ConcurrentHashMap`。
