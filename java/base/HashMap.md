# HashMap 源码学习
```$xslt
Map<String,Object> map = new HashMap<>();

map.put("test", 8888);
map.get("test");

```

## 初始化
### 默认参数
- DEFAULT_INITIAL_CAPACITY = 1 << 4; 默认初始化HashMap大小
- DEFAULT_LOAD_FACTOR = 0.75f; 默认负载因子
- TREEIFY_THRESHOLD = 8; 链表转红黑树的长度，触发条件 length（Node 链表）>= TREEIFY_THRESHOLD - 1
- UNTREEIFY_THRESHOLD = 6; 红黑树转链表的长度 
- MIN_TREEIFY_CAPACITY = 64; HashMap 触发链表转红黑树的最小数组长度 和 TREEIFY_THRESHOLD配合使用

### Init
```$xslt
/**
     * Constructs an empty <tt>HashMap</tt> with the specified initial
     * capacity and load factor.
     *
     * @param  initialCapacity the initial capacity
     * @param  loadFactor      the load factor
     * @throws IllegalArgumentException if the initial capacity is negative
     *         or the load factor is nonpositive
     */
    public HashMap(int initialCapacity, float loadFactor) {
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal initial capacity: " +
                                               initialCapacity);
        if (initialCapacity > MAXIMUM_CAPACITY)
            initialCapacity = MAXIMUM_CAPACITY;
        if (loadFactor <= 0 || Float.isNaN(loadFactor))
            throw new IllegalArgumentException("Illegal load factor: " +
                                               loadFactor);
        this.loadFactor = loadFactor;
        this.threshold = tableSizeFor(initialCapacity);
    }


    /**
     * 通过输入的cap 计算出最近的2的N次方数组的Size
     */
    static final int tableSizeFor(int cap) {
        int n = cap - 1;
        n |= n >>> 1;
        n |= n >>> 2;
        n |= n >>> 4;
        n |= n >>> 8;
        n |= n >>> 16;
        return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
    }

```


## Put操作
```$xslt
public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true);
    }
    
  //hash 扰动方法使用 hashCode 和HashCode无符号右移动16位后求 异或 获得最终的Hash值  
  // 这个方法进一步散列和Hash值，并且带入了高16位的信息，hash碰撞有所减少
 static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }   
    
    
 /**
      * Implements Map.put and related methods
      *
      * @param hash hash for key
      * @param key the key
      * @param value the value to put
      * @param onlyIfAbsent if true, don't change existing value
      * @param evict if false, the table is in creation mode.
      * @return previous value, or null if none
      */
     final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                    boolean evict) {
         Node<K,V>[] tab; Node<K,V> p; int n, i;
         if ((tab = table) == null || (n = tab.length) == 0)
             //判断table是否以及初始化，如果没有就执行resize()方法完成初始化
             n = (tab = resize()).length;
             
             //通过hash值和tab的size - 1 取 & 获得数组tab的下标
             //从这里 (n - 1) & hash 可以看到为什么 n = 2 的N次方 例如默认的n = 16 减去 1 = 15 后转成二进制 就是 ~ 0000 0000 0000 1111 
             // 和Hash 取 & 正好保留了 Hash 的后面几位，作为命中的数组的下标
         if ((p = tab[i = (n - 1) & hash]) == null)
             //如果没有数据则新建Node并赋值给tab
             tab[i] = newNode(hash, key, value, null);
         else {
             // 如果数组命中这已经有了数据
             Node<K,V> e; K k;
             if (p.hash == hash &&
                 ((k = p.key) == key || (key != null && key.equals(k))))
                 //如果这个命中的Node的 hash和key 与需要put的都能匹配上，将p 赋值给e 等待后续处理
                 e = p;
             else if (p instanceof TreeNode)
                 //如果上一步不能匹配上 且p是一个 Tree的情况下
                 e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
             else {
                 // 如果是链表的情况下 循环遍历
                 for (int binCount = 0; ; ++binCount) {
                     //当p.Next == null 时表面这个链表上没有Key存在则新建node，并插入都链表尾部
                     if ((e = p.next) == null) {
                         // jdk 1.8 变为尾插法，1.8之前是头插法
                         // 尾插法 变更为 头插法 保证了数据Put的顺序，也间接处理了resize过程中如果出现并发问题可能出现死循环的问题
                         // 我认为 HashMap用在并发场景就是不对的 所以上一步所说的问题更应该是使用不当的问题
                         // 1.8之前是头插法,可能是因为作者觉得新插入的数据被get的概率更高，尾插法 后新数据就放到尾部了
                         p.next = newNode(hash, key, value, null);
                         if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                         // 如果 binCount >= 7 则进入链表转 Tree的方法
                         // 这个方法里面会判断如果当前tab的长度 <64 则会先执行 resize 不会去执行链表转 Tree
                         // 链表转红黑树 需要 当前Map的 tab 的长度 > 64 且某个 tab上链表的长度 >= 7
                             treeifyBin(tab, hash);
                         break;
                     }
                     //如果匹配上 就跳出循环
                     if (e.hash == hash &&
                         ((k = e.key) == key || (key != null && key.equals(k))))
                         break;
                     //未匹配上继续向Next寻找    
                     p = e;
                 }
             }
             if (e != null) { // existing mapping for key
                 //如果是已经存在的Key
                 V oldValue = e.value;
                 if (!onlyIfAbsent || oldValue == null)
                     //如果允许覆盖value
                     e.value = value;
                 afterNodeAccess(e);
                 return oldValue;
             }
         }
         //如果新的K-V
         ++modCount;
         if (++size > threshold)
             resize();
         afterNodeInsertion(evict);
         return null;
     }   

```

## Resize扩容操作

```$xslt
/**
     * Initializes or doubles table size.  If null, allocates in
     * accord with initial capacity target held in field threshold.
     * Otherwise, because we are using power-of-two expansion, the
     * elements from each bin must either stay at same index, or move
     * with a power of two offset in the new table.
     *
     * @return the table
     */
    final Node<K,V>[] resize() {
        Node<K,V>[] oldTab = table;
        //获取当前tab的长度
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
        int oldThr = threshold;
        int newCap, newThr = 0;
        if (oldCap > 0) {
            if (oldCap >= MAXIMUM_CAPACITY) {
               //如果已经是最大长度了
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)
                     tab长度和 触发resize的值都 向左移动1 位 及 new = old * 2
                newThr = oldThr << 1; // double threshold
        }
        else if (oldThr > 0) // initial capacity was placed in threshold
            newCap = oldThr;
        else {               // zero initial threshold signifies using defaults
            //未初始化的时候
            //容量是 16
            newCap = DEFAULT_INITIAL_CAPACITY; 
            //resize的值是 16 * 负载因子 = 16 * 0.75 = 12
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }
        if (newThr == 0) {
            如果 计算出的触发resize的值 小于 MAXIMUM_CAPACITY 则返回 计算值，否则返回Integer.MAX_VALUE
            float ft = (float)newCap * loadFactor;
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }
        threshold = newThr;
        //新建 tab
        @SuppressWarnings({"rawtypes","unchecked"})
            Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
        table = newTab;
        //copy 老的tab 到新的tab
        if (oldTab != null) {
            for (int j = 0; j < oldCap; ++j) {
                Node<K,V> e;
                if ((e = oldTab[j]) != null) {
                    oldTab[j] = null;
                    if (e.next == null)
                        newTab[e.hash & (newCap - 1)] = e;
                    else if (e instanceof TreeNode)
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    else { // preserve order
                        Node<K,V> loHead = null, loTail = null;
                        Node<K,V> hiHead = null, hiTail = null;
                        Node<K,V> next;
                        do {
                            next = e.next;
                            if ((e.hash & oldCap) == 0) {
                                if (loTail == null)
                                    loHead = e;
                                else
                                    loTail.next = e;
                                loTail = e;
                            }
                            else {
                                if (hiTail == null)
                                    hiHead = e;
                                else
                                    hiTail.next = e;
                                hiTail = e;
                            }
                        } while ((e = next) != null);
                        if (loTail != null) {
                            loTail.next = null;
                            newTab[j] = loHead;
                        }
                        if (hiTail != null) {
                            hiTail.next = null;
                            newTab[j + oldCap] = hiHead;
                        }
                    }
                }
            }
        }
        return newTab;
    }

```

## Get过程

```$xslt
  /**
     * Implements Map.get and related methods
     *
     * @param hash hash for key
     * @param key the key
     * @return the node, or null if none
     */
    final Node<K,V> getNode(int hash, Object key) {
        Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
        //通过hash算在tab中的罗店
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (first = tab[(n - 1) & hash]) != null) {
            if (first.hash == hash && // always check first node
                ((k = first.key) == key || (key != null && key.equals(k))))
                return first;
            if ((e = first.next) != null) {
                if (first instanceof TreeNode)
                    //红黑树中查找
                    return ((TreeNode<K,V>)first).getTreeNode(hash, key);
                do {
                   //链表中查找
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        return e;
                } while ((e = e.next) != null);
            }
        }
        return null;
    }
```

## Remove过程
```$xslt

 /**
     * Implements Map.remove and related methods
     *
     * @param hash hash for key
     * @param key the key
     * @param value the value to match if matchValue, else ignored
     * @param matchValue if true only remove if value is equal
     * @param movable if false do not move other nodes while removing
     * @return the node, or null if none
     */
    final Node<K,V> removeNode(int hash, Object key, Object value,
                               boolean matchValue, boolean movable) {
        Node<K,V>[] tab; Node<K,V> p; int n, index;
        //查询key是否存在
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (p = tab[index = (n - 1) & hash]) != null) {
            Node<K,V> node = null, e; K k; V v;
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                node = p;
            else if ((e = p.next) != null) {
                if (p instanceof TreeNode)
                    node = ((TreeNode<K,V>)p).getTreeNode(hash, key);
                else {
                    do {
                        if (e.hash == hash &&
                            ((k = e.key) == key ||
                             (key != null && key.equals(k)))) {
                            node = e;
                            break;
                        }
                        p = e;
                    } while ((e = e.next) != null);
                }
            }
            //如果存在
            if (node != null && (!matchValue || (v = node.value) == value ||
                                 (value != null && value.equals(v)))) {
                if (node instanceof TreeNode)
                    //红黑树删除
                    ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);
                else if (node == p)
                    tab[index] = node.next;
                else
                    p.next = node.next;
                ++modCount;
                --size;
                afterNodeRemoval(node);
                return node;
            }
        }
        return null;
    }

```
