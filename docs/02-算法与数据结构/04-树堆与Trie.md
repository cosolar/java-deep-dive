# 树、堆与 Trie：有序结构与 TopK

> 定位：树解决「有序与范围」，堆解决「最值与 TopK」，Trie 解决「前缀」。这三类结构在工程里比链表更常见：数据库索引是 B+ 树，排行榜是堆/跳表，敏感词与路由是 Trie。这一篇把遍历、平衡、堆操作与前缀树讲透，代码以 Java 为主。

## 一、二叉树：遍历是基本功

```java
class TreeNode {
    int val;
    TreeNode left, right;
    TreeNode(int val) { this.val = val; }
}
```

### 1.1 递归遍历

```java
void preorder(TreeNode root, List<Integer> out) {
    if (root == null) return;
    out.add(root.val);
    preorder(root.left, out);
    preorder(root.right, out);
}
```

中序、后序只是 `add` 的位置不同。递归代码短，但深度等于树高；退化成链表时深度 n，可能栈溢出。

### 1.2 迭代中序

```java
List<Integer> inorder(TreeNode root) {
    List<Integer> res = new ArrayList<>();
    Deque<TreeNode> st = new ArrayDeque<>();
    TreeNode cur = root;
    while (cur != null || !st.isEmpty()) {
        while (cur != null) {
            st.push(cur);
            cur = cur.left;
        }
        cur = st.pop();
        res.add(cur.val);
        cur = cur.right;
    }
    return res;
}
```

迭代的本质是用显式栈模拟递归；前序只需调整入栈顺序，后序可以用「根右左」再反转。

### 1.3 Morris 遍历：O(1) 空间

Morris 中序利用叶子节点的空右指针临时指向后继，遍历完再恢复。时间 O(n)，空间 O(1)，适合内存敏感场景；代价是遍历过程中会临时修改树结构，并发读不安全。

### 1.4 层序遍历

```java
List<List<Integer>> levelOrder(TreeNode root) {
    List<List<Integer>> res = new ArrayList<>();
    if (root == null) return res;
    Deque<TreeNode> q = new ArrayDeque<>();
    q.offer(root);
    while (!q.isEmpty()) {
        int size = q.size();
        List<Integer> level = new ArrayList<>(size);
        for (int i = 0; i < size; i++) {
            TreeNode cur = q.poll();
            level.add(cur.val);
            if (cur.left != null) q.offer(cur.left);
            if (cur.right != null) q.offer(cur.right);
        }
        res.add(level);
    }
    return res;
}
```

`size` 必须在循环前固定，否则会把下一层节点混进当前层。

## 二、二叉搜索树与平衡

### 2.1 BST 的定义

对每个节点，左子树所有值 < 根 < 右子树所有值。注意不是「左子节点 < 根 < 右子节点」——只比较直接子节点是错的，必须传上下界。

```java
boolean isValidBST(TreeNode root) {
    return valid(root, Long.MIN_VALUE, Long.MAX_VALUE);
}

boolean valid(TreeNode node, long low, long high) {
    if (node == null) return true;
    if (node.val <= low || node.val >= high) return false;
    return valid(node.left, low, node.val) && valid(node.right, node.val, high);
}
```

用 `long` 是因为节点值可能是 `Integer.MIN_VALUE` / `MAX_VALUE`。

### 2.2 查找、插入、删除

- 查找：比较后走左/右，平均 O(log n)，退化成链表 O(n)；
- 插入：走到空位插入，保持 BST 性质；
- 删除：叶子直接删；只有一个孩子用孩子替代；有两个孩子用后继（右子树最小）或前驱替代，再删后继。

### 2.3 AVL 与红黑树

| 维度 | AVL | 红黑树 |
|---|---|---|
| 平衡严格度 | 高度差 ≤ 1 | 最长路径 ≤ 2×最短 |
| 查找 | 略快 | 略慢 |
| 插入/删除 | 旋转多 | 旋转少 |
| 适用 | 查找密集 | 插入删除混合 |

HashMap 用红黑树，是因为冲突链表化后插入删除频繁；TreeMap 也用红黑树，因为要维护有序。数据库索引用 B+ 树，不是红黑树，因为磁盘 IO 按页算，树高决定 IO 次数。

## 三、堆：最值与 TopK

### 3.1 堆的性质

- 完全二叉树，用数组存储；
- 大顶堆：父 ≥ 子；小顶堆：父 ≤ 子；
- 取顶 O(1)，插入/删除 O(log n)，建堆 O(n)。

### 3.2 上浮与下沉

```java
void siftUp(int[] h, int i) {
    while (i > 0) {
        int p = (i - 1) / 2;
        if (h[p] >= h[i]) break;
        swap(h, p, i);
        i = p;
    }
}

void siftDown(int[] h, int i, int n) {
    while (true) {
        int l = 2 * i + 1, r = l + 1, max = i;
        if (l < n && h[l] > h[max]) max = l;
        if (r < n && h[r] > h[max]) max = r;
        if (max == i) break;
        swap(h, i, max);
        i = max;
    }
}
```

建堆从最后一个非叶子节点 `n/2 - 1` 开始下沉，总复杂度 O(n)，不是 O(n log n)。

### 3.3 TopK：求第 k 小用大顶堆

```java
int kthSmallest(int[] a, int k) {
    PriorityQueue<Integer> pq = new PriorityQueue<>(Comparator.reverseOrder());
    for (int x : a) {
        pq.offer(x);
        if (pq.size() > k) pq.poll();
    }
    return pq.peek();
}
```

复杂度 O(n log k)，空间 O(k)。方向别记反：**求第 k 小，堆里留最小的 k 个，堆顶是其中最大**；求第 k 大用小顶堆。

数据量超大时，先分块各取 TopK，再归并；不要全量加载。

### 3.4 Java PriorityQueue 的坑

- 默认小顶堆；大顶堆要传 `Comparator.reverseOrder()`；
- 不支持 decrease-key，Dijkstra 里常用「重复入堆 + 出堆时跳过旧距离」；
- 非线程安全，并发用 `PriorityBlockingQueue`；
- 遍历顺序不是有序的，只有 `poll` 才按优先级。

## 四、Trie：前缀树

### 4.1 结构

```java
class Trie {
    class Node {
        Node[] child = new Node[26];
        boolean end;
    }
    private final Node root = new Node();

    public void insert(String word) {
        Node p = root;
        for (char c : word.toCharArray()) {
            int i = c - 'a';
            if (p.child[i] == null) p.child[i] = new Node();
            p = p.child[i];
        }
        p.end = true;
    }

    public boolean search(String word) {
        Node p = find(word);
        return p != null && p.end;
    }

    public boolean startsWith(String prefix) {
        return find(prefix) != null;
    }

    private Node find(String s) {
        Node p = root;
        for (char c : s.toCharArray()) {
            int i = c - 'a';
            if (p.child[i] == null) return null;
            p = p.child[i];
        }
        return p;
    }
}
```

插入/查询 O(L)，L 是字符串长度。

### 4.2 工程优化

- 26 叉数组每个节点 26 个引用，内存浪费；稀疏时用 `HashMap<Character, Node>`；
- 双数组 Trie 把结构压到两个数组，适合词典；
- 敏感词过滤可在 Trie 节点加 `fail` 指针，变成 AC 自动机；
- 路由表用压缩 Trie（Patricia Trie），按前缀最长匹配。

### 4.3 Trie 的适用边界

- 适合前缀查询、自动补全、敏感词、IP 路由；
- 不适合纯精确查找，哈希更省内存；
- 字符集大（中文、Unicode）时，数组方案要改成 Map；
- 大量短串时，Trie 的节点开销可能超过收益。

## 五、树结构怎么选

| 需求 | 结构 | 原因 |
|---|---|---|
| 内存有序映射 | 红黑树（TreeMap） | O(log n) 查找/插入，支持范围 |
| 磁盘索引 | B+ 树 | 扇出大、树高低、叶子链表 |
| TopK/最值 | 堆 | O(1) 取顶，O(log n) 调整 |
| 前缀匹配 | Trie | 按字符逐层 |
| 排行榜 | 跳表/TreeSet | 有序、范围、排名 |
| 并查集 | 父指针 + 路径压缩 | 连通性近 O(1) |

## 六、AVL 旋转：平衡是怎么恢复的

AVL 要求每个节点左右子树高度差 ≤ 1。插入/删除后，从下往上更新高度，找到第一个失衡节点，按四种情况旋转：

| 失衡形态 | 操作 |
|---|---|
| LL（左左） | 右旋 |
| RR（右右） | 左旋 |
| LR（左右） | 先左旋左子，再右旋 |
| RL（右左） | 先右旋右子，再左旋 |

```java
class AVLNode {
    int val, height = 1;
    AVLNode left, right;
    AVLNode(int val) { this.val = val; }
}

int height(AVLNode n) { return n == null ? 0 : n.height; }
void update(AVLNode n) { n.height = 1 + Math.max(height(n.left), height(n.right)); }
int balance(AVLNode n) { return n == null ? 0 : height(n.left) - height(n.right); }

AVLNode rotateRight(AVLNode y) {
    AVLNode x = y.left;
    AVLNode t = x.right;
    x.right = y;
    y.left = t;
    update(y);
    update(x);
    return x;
}

AVLNode rotateLeft(AVLNode x) {
    AVLNode y = x.right;
    AVLNode t = y.left;
    y.left = x;
    x.right = t;
    update(x);
    update(y);
    return y;
}

AVLNode insert(AVLNode node, int val) {
    if (node == null) return new AVLNode(val);
    if (val < node.val) node.left = insert(node.left, val);
    else if (val > node.val) node.right = insert(node.right, val);
    else return node;
    update(node);
    int b = balance(node);
    if (b > 1 && val < node.left.val) return rotateRight(node);       // LL
    if (b < -1 && val > node.right.val) return rotateLeft(node);      // RR
    if (b > 1 && val > node.left.val) {                               // LR
        node.left = rotateLeft(node.left);
        return rotateRight(node);
    }
    if (b < -1 && val < node.right.val) {                             // RL
        node.right = rotateRight(node.right);
        return rotateLeft(node);
    }
    return node;
}
```

AVL 查找严格 O(log n)，但插入/删除旋转多；读多写少、对查找延迟敏感的场景更适合。

## 七、红黑树：五条规则与工程取舍

红黑树不追求严格平衡，而用颜色约束保证「最长路径 ≤ 2×最短路径」：

1. 每个节点是红或黑；
2. 根是黑；
3. 叶子（NIL）是黑；
4. 红节点的子节点必须是黑（不能连续红）；
5. 从任一节点到其叶子，黑节点数相同。

这五条保证树高 O(log n)，插入/删除最多常数次旋转。相比 AVL：

- 查找略慢（树可能更高）；
- 插入/删除旋转少，综合更快；
- 实现复杂，工程上直接用 `TreeMap`/`TreeSet`。

什么时候需要自己实现？需要定制节点数据、范围查询、持久化版本，或面试要求手写。否则不要重复造轮子。

## 八、B 树与 B+ 树：磁盘友好的索引

### 8.1 B 树

B 树是多叉平衡搜索树，每个节点存多个 key 和多个孩子。阶为 m 的 B 树：

- 每个节点最多 m 个孩子、m-1 个 key；
- 除根外至少 ceil(m/2) 个孩子；
- 所有叶子在同一层。

查找/插入/删除 O(log n)，但每次节点访问是一次磁盘页 IO。

### 8.2 B+ 树

B+ 树是 B 树的变体：

- 内部节点只存索引 key，不存数据；
- 数据全部在叶子；
- 叶子之间有链表，范围查询顺序扫描；
- 扇出更大，树高更低。

MySQL InnoDB 的聚簇索引：叶子存整行；二级索引：叶子存主键，查非索引列要回表。

### 8.3 为什么不用二叉树做磁盘索引

二叉树每层只比较一个 key，树高约 log₂n；B+ 树每层几百个 key，树高 3–4 层。磁盘 IO 次数差一个量级，这就是「磁盘结构」与「内存结构」的根本区别。

## 九、堆的进阶用法

### 9.1 数据流中位数：两个堆

- 大顶堆存较小的一半；
- 小顶堆存较大的一半；
- 保持两堆大小差 ≤ 1；
- 中位数 = 大顶堆顶（奇数）或两堆顶平均（偶数）。

```java
class MedianFinder {
    PriorityQueue<Integer> low = new PriorityQueue<>(Comparator.reverseOrder());
    PriorityQueue<Integer> high = new PriorityQueue<>();

    void add(int x) {
        low.offer(x);
        high.offer(low.poll());
        if (low.size() < high.size()) low.offer(high.poll());
    }

    double median() {
        return low.size() == high.size()
            ? (low.peek() + high.peek()) / 2.0
            : low.peek();
    }
}
```

复杂度：插入 O(log n)，取中位数 O(1)。

### 9.2 延迟队列

`DelayQueue` 用堆按到期时间排序，适合单机延迟任务；分布式用 Redis ZSet、RocketMQ 延迟消息、时间轮。堆的 `peek` 只看到期最早的任务，避免全量扫描。

### 9.3 TopK 与堆的边界

- k 很小、n 很大：堆 O(n log k)；
- k 接近 n：直接排序 O(n log n)；
- 数据流：维护堆；
- 内存放不下：分块 TopK + 归并。

## 面试高频问法

**Q1：为什么数据库索引用 B+ 树，不用红黑树？**

一句话结论：磁盘 IO 以页为单位，B+ 树一个节点存几百个 key，树高 3–4 层就能索引亿级数据；红黑树每层只存一个 key，树高 log₂n，IO 次数多。

展开：B+ 树叶子链表支持范围查询；数据都在叶子，内部节点只存索引，扇出大。

加分点：能联系「页大小 16KB、树高与磁盘 IO 次数」。

**Q2：堆建堆为什么是 O(n)，不是 O(n log n)？**

一句话结论：从最后一个非叶子节点下沉，越靠近底层的节点越多但下沉高度越小，求和后是 O(n)。

展开：大约一半节点是叶子，不需要下沉；总代价 Σ(节点数 × 高度) 收敛到 O(n)。

加分点：能说出「插入建堆是 O(n log n)，原地建堆是 O(n)」。

**Q3：TopK 为什么用堆，不用排序？**

一句话结论：排序 O(n log n)，堆 O(n log k)；k 远小于 n 时堆更省时间与空间。

展开：求第 k 大用小顶堆，堆里保留最大的 k 个；求第 k 小用大顶堆。数据量超大时先分治再归并。

加分点：能说出「堆顶是当前 k 个里的最值，所以能被替换」。

**Q4：BST 校验为什么不能只比较父子节点？**

一句话结论：BST 要求整个左子树都小于根、整个右子树都大于根，只比较直接子节点会漏掉跨层违规。

展开：递归传 `(low, high)` 上下界；用 `long` 防 `Integer.MIN_VALUE` 边界。

加分点：能举出「右子树的左子节点小于根」的反例。

**Q5：Trie 的优缺点是什么？**

一句话结论：前缀查询 O(L)、结构清晰；但节点开销大，字符集大时内存膨胀，纯精确查找不如哈希。

展开：稀疏用 Map、词典用双数组、敏感词用 AC 自动机；中文/Unicode 要避免 26 叉数组。

加分点：能说出「Trie 适合前缀，哈希适合精确」。

## 小结

- **遍历是树题的地基**：递归简单但有栈深风险，迭代用显式栈，Morris 可 O(1) 空间。
- **BST 校验传上下界**，不是只比父子；删除有两个孩子时用后继/前驱替代。
- **AVL 查找快、红黑树插入删除稳**；磁盘索引用 B+ 树，不是红黑树。
- **堆的核心是上浮/下沉**；建堆 O(n)，TopK O(n log k)，方向别记反。
- **Trie 适合前缀**，工程上要解决内存与字符集问题。
