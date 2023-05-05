### ConcurrentHashMap

### putVal

```java
final V putVal(K key, V value, boolean onlyIfAbsent) {
    // 与hashmap的第一个区别,不允许key value为null
    if (key == null || value == null) throw new NullPointerException();

    // 两次hash,重点看spread,与HashMap是一样的,只不过多做了一次与运算 0x7fffffff 31个1,也就是说,做完这个与运算,最高位一定是0,也就是说hash一定是正数
    int hash = spread(key.hashCode());
    // 链表转红黑树的计数器
    int binCount = 0;
    // table是volatile修饰的数组
    for (Node<K,V>[] tab = table;;) {
        // 临时变量
        Node<K,V> f; int n, i, fh;
        // table的懒加载,注意n的赋值
        if (tab == null || (n = tab.length) == 0)
            // 创建数组,下一次循环就不会到这里了
            tab = initTable();

        // 下一次循环会到这,关键方法tabAt
        // 计算索引的方式与HashMap一致,都是容量-1 & hash
        // 是null 就是没有hash冲突
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            // 开始cas尝试设置值
            // tab 对象指针 i索引位置(和tabAt一样会计算出偏移地址)
            // null 期望值
            // 期望修改成 new Node<K,V>(hash, key, value, null)
            if (casTabAt(tab, i, null,
                         new Node<K,V>(hash, key, value, null)))
                // 设置成功直接返回,put结束
                break;                   // no lock when adding to empty bin
        }
        // 有hash冲突,且hash 被表集成了MOVED -1
        else if ((fh = f.hash) == MOVED)
            // 则当前线程,帮助扩容
            tab = helpTransfer(tab, f);
        // 仅仅是hash冲突,没有扩容
        else {
            V oldVal = null;
            // 锁node 或者说锁了树的根节点,或者链表的根节点,而不是锁整个tab
            synchronized (f) {
                // double check, 检测加锁前 (防止扩容,链变树等等操作导致这个头结点被更改)
                // 双重检测失败，会继续自旋去增加这个key
                if (tabAt(tab, i) == f) {
                    // fh 的 hash>=0 表示这是一个链表
                    if (fh >= 0) {
                        // 链转树阈值的计算
                        binCount = 1;
                        // 遍历链表
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;
                            // key 一样的情况,换value
                            if (e.hash == hash &&
                                ((ek = e.key) == key ||
                                 (ek != null && key.equals(ek)))) {
                                oldVal = e.val;
                                if (!onlyIfAbsent)
                                    e.val = value;
                                break;
                            }
                            // 否则就是key不一致的逻辑
                            // 前驱指针
                            Node<K,V> pred = e;
                            // 注意e的赋值,for循环
                            // 遍历到链表尾 key都没有一致的,那么就是新增
                            if ((e = e.next) == null) {
                                // 前驱指针就是干这个的 新节点加到链上
                                pred.next = new Node<K,V>(hash, key,
                                                          value, null);
                                break;
                            }
                        }
                    }
                    // 树节点的增加,看不懂忽略
                    else if (f instanceof TreeBin) {
                        Node<K,V> p;
                        binCount = 2;
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                              value)) != null) {
                            oldVal = p.val;
                            if (!onlyIfAbsent)
                                p.val = value;
                        }
                    }
                }
            }
           
            // binCount = 0 只有一种情况就是
            if (binCount != 0) {
                 // 添加成功,判断是否需要转化成树
                if (binCount >= TREEIFY_THRESHOLD)
                    treeifyBin(tab, i);
                // why? 上述分析中，走到这oldVal一定被赋值
                if (oldVal != null)
                    return oldVal;
                break;
            }
        }
    }
    
    // 关键方法
    addCount(1L, binCount);
    return null;
}


```

### initTable

```java
// 初始化table
private final Node<K,V>[] initTable() {
    Node<K,V>[] tab; int sc;

    // 自旋
    while ((tab = table) == null || tab.length == 0) {
        // sizeCtl volatile修饰,用于table状态的控制 默认值0; 等于-1时,标识正在创建
        if ((sc = sizeCtl) < 0)
            // 别的线程正在创建table,就只需要自旋等待别的线程创建完成
            Thread.yield(); // lost initialization race; just spin
        // 否则,没有线程正在创建,那么当前线程创建  cas 先把sizeCtl设置为-1
        // 解释一下compareAndSwapInt 对象指针,字段偏移量,期望值,要修改的值
        else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
            // 状态设置成功,才开始创建表,否则就是被别的线程捷足先登了,自旋等待去吧
            try {
                // 注意tab的赋值
                if ((tab = table) == null || tab.length == 0) {
                    // 计算出来容量
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                    // 创建数组,泛型数组强转警告忽略
                    @SuppressWarnings("unchecked")
                    Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                    // 赋值table(问:为什么这里不需要cas或者加锁?)
                    table = tab = nt;
                    // sc = 3/4 容量
                    sc = n - (n >>> 2);
                }
            } finally {
                // 将sizeCtl设置为 3/4容量(问:为什么不需要cas或者加锁)
                sizeCtl = sc;
            }
            // 修改成功,跳出自旋
            break;
        }
    }
    // 返回新的数组
    return tab;
}
```

### tabAt

```java
// static 块
private static final long ABASE;
private static final int ASHIFT;
static {
    // tab 就是一个Node数组
    Class<?> ak = Node[].class;
    // ABASE是第一个元素的偏移量
    ABASE = U.arrayBaseOffset(ak);
    // 获取数组中元素的增量值(一个元素的大小)
    int scale = U.arrayIndexScale(ak);
    // 元素大小必须是2的整数次幂,scale 是2的整数次幂,则二进制只有一个1,剩下全为0,scale-1 则最高位的1变成0,低位全是1,与运算之后就是0了
    if ((scale & (scale - 1)) != 0)
        throw new Error("data type scale not a power of two");
    // numberOfLeadingZeros 简单的讲,就是获取从高位开始连续的0的个数
    ASHIFT = 31 - Integer.numberOfLeadingZeros(scale);
}

// tabAt
static final <K,V> Node<K,V> tabAt(Node<K,V>[] tab, int i) {
    // 获取对象中,指定偏移大小的值,支持volatile(有内存屏障)
    // 思考一下为什么要这么复杂?(Java的volatile仅支持成员变量,不支持数组里面的元素级别,这里是把数组当成了对象进行读写,曲线救国,让数组支持了volatile语义)

    // 偏移量计算公式
    // i << ASHIFT + ABASE
    // 正常的公式 i * eleSize + ABASE
    // 正向推到出  i << ASHIFT == i * eleSize
    // so why?
    // 首先static块保证了 eleSize都是2的整数次幂
    // 那么一个数乘一个2的整数次幂就可以转化为左移运算
    // 左移多少位, 假如eleSize是0b0100 需要左移2位, 0b0100的numberOfLeadingZeros是29
    // ASHIFT = 31 - 29 = 2


    // 整数的numberOfLeadingZeros 最大是32,也就是0,32位都是0, 那么31-32  = -1(左移负数会整数溢出)
    // 那么为什么这里没有溢出检测? 大概是因为Node的大小肯定不是0吧
    return (Node<K,V>)U.getObjectVolatile(tab, ((long)i << ASHIFT) + ABASE);
}
```

### casTabAt

```java
static final <K,V> boolean casTabAt(Node<K,V>[] tab, int i,
                                    Node<K,V> c, Node<K,V> v) {
    return U.compareAndSwapObject(tab, ((long)i << ASHIFT) + ABASE, c, v);
}
```

### addCount

```java
private final void addCount(long x, int check) {
    CounterCell[] as; long b, s;
    // 计数表不为null 或者 尝试cas将值累加到baseCount失败
    if ((as = counterCells) != null ||
        !U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)) {

        CounterCell a; long v; int m;

        boolean uncontended = true;

        // 如果计数表是空,或者计数表的长度小于1,或者当前线程的计数器为null,或者当前线程在当前线程的计数器中累加值失败(|| 条件,只有前一个返回false,才会去执行后一个,所以不会空指针)
        if (as == null || (m = as.length - 1) < 0 ||
            // as[ThreadLocalRandom.getProbe() & m] 分析 as = CounterCell[] counterCells 是ConcurrentHashMap维护的一个计数表
            // ThreadLocalRandom.getProbe() 获取线程的Probe在一定线程数量范围内,这个值不会重复,可以认为&上计数表长度-1后也不会重复.到这里就可以倒推出为什么计数表的长度也要是2的倍数
            (a = as[ThreadLocalRandom.getProbe() & m]) == null ||
            !(uncontended =
              U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))) 
        {
            // 计数,走到意味着,尝试在计数表里计数失败了
            fullAddCount(x, uncontended);
            return;
        }

        // 元素加入联表的第一个位置,或者没有hash冲突,这里才会小于1
        // 走到只有上面if的cas修改成功,才不会提前return
        if (check <= 1)
            return;
        // 遍历计数表求和
        s = sumCount();
    }

    // 再次检查check,走到这两个可能 
    // 1.该线程一次cas计数表计数就成功了,且有hash冲突. 此时会去统计计数表的值
    // 2.计数表未初始化,且一次cas去更新baseCount就成功了 这种情况下s是baseCount+x
    if (check >= 0) {
        Node<K,V>[] tab, nt; int n, sc;
        // 判断容量是否超过阈值, sizeCtl保存的是阈值,初始值是0,用于新建tab,-1表示正在初始化
        // 当前元素数已经大于阈值, 问为什么要加(tab = table) != null ? (ps:防熊 sizeCtl=-1表示正在初始化,假如初始化tab的线程还没有走到finally块,把sizeCtl设置成下一次扩容的阈值,此时一堆线程来put,走到了这,想扩容就会出问题)
        while (s >= (long)(sc = sizeCtl) && (tab = table) != null &&
               // tab的长度没有超过最大长度限制
               (n = tab.length) < MAXIMUM_CAPACITY) {
            // resizeStamp 
            int rs = resizeStamp(n);

            // sc小于0只有两种情况
            // 1.初始化tab(基本不可能)  2.正在扩容
            // 建议先看 sc>0,开始扩容的逻辑
            // 正在扩容,则协助扩容
            if (sc < 0) {
                // (sc >>> RESIZE_STAMP_SHIFT) != rs 什么时候相等
                // 走到这，sc的低16位表示有多少个线程扩容,右移16位后,高位又回来了,如果和现在计算的rs相等,表示sizeCtl的高16位表示的tab长度和现在的tab长度一样,也就是说扩容未完成。综上扩容未完成时,这个条件相等
                if ((sc >>> RESIZE_STAMP_SHIFT) != rs 
                    // sc 必然是负数,rs必然是整数 永远不可能相等,bug连接https://bugs.openjdk.org/browse/JDK-8214427
                    // https://bugs.java.com/bugdatabase/view_bug?bug_id=8214427
                    || sc == rs + 1 
                    // 这个也是同样的道理,必然false
                    || sc == rs + MAX_RESIZERS 
                    // double check,确保nextTable不为null
                    || (nt = nextTable) == null 
                    // transferIndex <= 0 表示已经扩容完毕,扩容是从右向左分配给扩容线程的	
                    || transferIndex <= 0)
                    // 上述无须协助扩容的条件检查完毕,若无须协助,直接跳出
                    break;
                // 否则,sc+1 回忆一下，低16位表示扩容的线程数 则sc扩容线程数+1
                if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                    // 然后协助扩容
                    transfer(tab, nt);
            }
            // 未在扩容,则进入扩容状态,rs左移RESIZE_STAMP_SHIFT必然为负数, 且低16位为0
            // cas 设置sizeCtl 加上2之后,用低16位表示有两个线程在扩容
            else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                         (rs << RESIZE_STAMP_SHIFT) + 2))
                // 扩容,会初始化nextTable表,扩容后sizeCtl会被设置成下一次扩容的阈值,具体逻辑后续分析
                transfer(tab, null);
            s = sumCount();
        }
    }
}
```

### fullAddCount

```java
/**
  * Spinlock (locked via CAS) used when resizing and/or creating CounterCells.
   */
private transient volatile int cellsBusy;

private final void fullAddCount(long x, boolean wasUncontended) {
    int h;
    // 如果线程没有使用过ThreadLocalRandom的api,那么会返回0
    if ((h = ThreadLocalRandom.getProbe()) == 0) {
        // 初始化线程的探针
        ThreadLocalRandom.localInit();      // force initialization
        // 再次获取
        h = ThreadLocalRandom.getProbe();
        wasUncontended = true;
    }

    boolean collide = false;                // True if last slot nonempty
    for (;;) {
        // 定义临时变量
        CounterCell[] as; CounterCell a; int n; long v;

        // 计数表存在,且长度大于0
        if ((as = counterCells) != null && (n = as.length) > 0) {
            // 计算当前线程,在计数表的位置如果是null,就创建
            if ((a = as[(n - 1) & h]) == null) {
                // 标记计数表是否空闲
                if (cellsBusy == 0) {            // Try to attach new Cell
                    // 空闲就创建
                    CounterCell r = new CounterCell(x); // Optimistic create
                    
                    // 创建完成将空闲标志设置为1,设置成功,才进行下面的逻辑,若已经被别的线程设置,则也走不进if
                    if (cellsBusy == 0 &&
                        U.compareAndSwapInt(this, CELLSBUSY, 0, 1)) {
                        // 设置空闲标志成功
                        
                        boolean created = false;
                        // double chekc,再次检查
                        try {               // Recheck under lock
                            CounterCell[] rs; int m, j;
                            if ((rs = counterCells) != null &&
                                (m = rs.length) > 0 &&
                                rs[j = (m - 1) & h] == null) {
                                // double check 检查通过,才最终赋值
                                rs[j] = r;
                               // 新增技术cell标记为true
                                created = true;
                            }
                        } finally {
                            // 因为拿到的有锁,或者说当前线程将标记设置为1,别的线程就无法设置成别的,于是可以认为只有当前线程持有计数表的修改权 或者说拥有锁
                            
                            //有锁的线程,当然可以释放锁
                            cellsBusy = 0;
                        }
                        // 是新增的cell 则跳出循环,否则是别的线程抢先一步,则继续循环,因为,别的线程创建的时候,cell装的是别的线程值,因此需要累加本线程的值到cell里
                        if (created)
                            break;
                        continue;           // Slot is now non-empty
                    }
                }
                collide = false;
            }
            // 进入fullAddCount之前的cas尝试中失败了,则设置这个wasUncontended为true,继续尝试
            else if (!wasUncontended)       // CAS already known to fail
                wasUncontended = true;      // Continue after rehash
            // 下次循环,就无须新建cell了 直接尝试cas累加,累加成功直接返回,否则继续循环
            else if (U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))
                break;
            
            // 再次cas又失败
            // counterCells 已经不等于as(发生了扩容) 或者计数表长度大于cpu个数 计数表最大只能是cpu核心个数,因为最多这么多个线程同事走到这计数,一个线程拿到一个cell刚刚好
            else if (counterCells != as || n >= NCPU)
                collide = false;            // At max size or stale
            // 又尝试cas又失败了,并且计数表未达cup个数
            else if (!collide)
                // 设置为true,让下次再次cas失败能走到扩容逻辑
                collide = true;
            // 尝试扩容
            else if (cellsBusy == 0 &&
                     U.compareAndSwapInt(this, CELLSBUSY, 0, 1)) {
                try {
                    if (counterCells == as) {// Expand table unless stale
                        CounterCell[] rs = new CounterCell[n << 1];
                        for (int i = 0; i < n; ++i)
                            rs[i] = as[i];
                        counterCells = rs;
                    }
                } finally {
                    cellsBusy = 0;
                }
                collide = false;
                // 扩容成功,继续尝试cas，直接开启下次循环
                continue;                   // Retry with expanded table
            }
            
           // 上述没有brek 和continue的分支,到这后刷新probe值,cas抢不过我就换个cell
            h = ThreadLocalRandom.advanceProbe(h);
        }
        // 初始化 计数表
        else if (cellsBusy == 0 && counterCells == as &&
                 U.compareAndSwapInt(this, CELLSBUSY, 0, 1)) {
            boolean init = false;
            try {                           // Initialize table
                if (counterCells == as) {
                    // 注意容量2,后续扩容也都是2的整数次幂
                    CounterCell[] rs = new CounterCell[2];
                    rs[h & 1] = new CounterCell(x);
                    counterCells = rs;
                    init = true;
                }
            } finally {
                cellsBusy = 0;
            }
            // 初始化成功,也意味着计数成功,因为新建的过程就创建了一个cell则跳出循环,计数完毕
            if (init)
                break;
        }
        // 别的线程正在初始化计数表,那我也不闲着,我去尝试累加到baseCount上,哼！累加失败继续回来循环cas
        else if (U.compareAndSwapLong(this, BASECOUNT, v = baseCount, v + x))
            break;                          // Fall back on using base
    }
}
```

### resizeStamp

```java
// int RESIZE_STAMP_BITS = 16
// int RESIZE_STAMP_SHIFT = 32 - RESIZE_STAMP_BITS = 16
private static int RESIZE_STAMP_BITS = 16;

// Returns the stamp bits for resizing a table of size n. Must be negative when shifted left by RESIZE_STAMP_SHIFT.
static final int resizeStamp(int n) {
    // 1 左移15位,是一个第16位为1,其余全0的数,官方的注释,返回值 左移RESIZE_STAMP_BITS位,必然为负数
    // 废话,因为是或运算,返回值的第16位必然是1,java的int都是有符号数,左移16位后,最高位为1,必然是负数
    // numberOfLeadingZeros 逻辑也比较简单,就是返回一个数从左开始有多少个连续的0
    // 传入的n是tab的长度,是一个2的整数次幂,则numberOfLeadingZeros返回的最大值为32 全0,二进制最大6位
    // 因此整个方法就是生成了一个第16位为1,低6位和现在容量强相关,高16位全0的正整数
    return Integer.numberOfLeadingZeros(n) | (1 << (RESIZE_STAMP_BITS - 1));
}

// >>> 无符号右移,高位补0低位抛弃
```

### openjdk 无bug版本 oracle修复版本8u361

```java
if (check >= 0) {
    Node<K,V>[] tab, nt; int n, sc;
    while (s >= (long)(sc = sizeCtl) && (tab = table) != null &&
           (n = tab.length) < MAXIMUM_CAPACITY) {
        // resizeStamp 方法没变,直接左移16位,直接将低16位置0,高16位,为容量
        int rs = resizeStamp(n) << RESIZE_STAMP_SHIFT;

        // 有线程在扩容
        if (sc < 0) {
            // 协助扩容判断
            // sc还是仍然是高16位是容量(tab长度),低16位是协助扩容线程数
            //  MAX_RESIZERS 65535 2的15次方-1个线程
            // sc 与 rs + MAX_RESIZERS相等只有一种情况 1.tab长度没变(扩容进行中),2.已经达到最大协助扩容线程数 65535,此时当然不需要协助扩容
            if (sc == rs + MAX_RESIZERS 
                // sc == rs + 1 只有一种情况 1.tab长度没变(扩容进行中) 2.扩容线程数为1(因为rs 低16位全0)
                // 为什么只有一个线程在扩容就不需要协助呢?(答案在扩容逻辑,扩容一开始将线程数置为2,其实只有一个线程在扩容,多一个线程协助,数量+1,同样的扩容完成后,每个线程再去-1,也就是说所有线程扩容完毕后,线程数为1)
                || sc == rs + 1 
                // double check 
                || (nt = nextTable) == null 
                // check transferIndex
                || transferIndex <= 0)
                break;
            if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                transfer(tab, nt);
        }
        else if (U.compareAndSwapInt(this, SIZECTL, sc, rs + 2))
            transfer(tab, null);
        s = sumCount();
    }
}
```

### transfer

```java
static final int NCPU = Runtime.getRuntime().availableProcessors();
private static final int MIN_TRANSFER_STRIDE = 16;

private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
    int n = tab.length, stride;
    // 步长stride
    // NCPU是cpu核心数大于1的话,则stride = n/(8*cpu), 否则stride=n
    // 如果算出来的步长小于16,则赋值为16
    if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
        stride = MIN_TRANSFER_STRIDE; // subdivide range
    // nextTab为null表示nextTable还没有创建,则初始化nextTable
    if (nextTab == null) {            // initiating
        try {
            // 创建新tab 容量扩容为旧容量二倍
            @SuppressWarnings("unchecked")
            Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
            // 赋值
            nextTab = nt;
            // 会有什么异常？OOM
        } catch (Throwable ex) {      // try to cope with OOME
            sizeCtl = Integer.MAX_VALUE;
            return;
        }
        // 赋值给 this.nextTable
        nextTable = nextTab;
        // 扩容索引设置为旧数组长度,从右边开始扩容
        transferIndex = n;
    }

    // 获取新数组的长度
    int nextn = nextTab.length;

    // ForwardingNode继承自Node,增加了find函数
    ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);

    // 标记是否继续倒腾数据
    boolean advance = true;
    boolean finishing = false; // to ensure sweep before committing nextTab

    // 初始两个变量,bound会被设置为下次需要倒腾的索引 注意是从右往左开始倒腾的
    for (int i = 0, bound = 0;;) {
        Node<K,V> f; int fh;

        // 会去一直尝试cas直到cas成功,获取了正确的bound和i
        // bound 是本线程倒腾tab的下界索引,i是上一个transferIndex-1,若无本线程无须倒腾,则bound为默认值0,为负数
        // 可以认为这是一个任务分发的过程,而且是个滑动窗口
        while (advance) {
            int nextIndex, nextBound;
            // --i比bound大或者已经完成
            if (--i >= bound || finishing)
                // 不需要继续倒腾
                advance = false;
            // 判断tab是否倒腾完,倒腾完也无须继续倒腾
            else if ((nextIndex = transferIndex) <= 0) {
                i = -1;
                advance = false;
            }
            // cas设置transferIndex,设置为nextIndex-stride(如果nextIndex>stride)
            else if (U.compareAndSwapInt
                     (this, TRANSFERINDEX, nextIndex,
                      nextBound = (nextIndex > stride ?
                                   nextIndex - stride : 0))) {
                bound = nextBound;
                // 一次倒腾一个槽
                i = nextIndex - 1;
                advance = false;
            }
        }

        // i < 0说明无须协助扩容,i >= n (不合法的索引值)或者 i + n>=nextn(n是旧数组长度,理论上nextn是n的二倍,则i+n>=nextn就说明i依旧是无效索引,n大于等于旧数组长度)
        // 索引无效的情况默认认为扩容完成。
        // 当i=n-1的时候, 每个条件都不符合,会去走下面的if,检查每个槽是不是都被设置成了MOVED,此时i每次都会-1,直到检查完旧tab的所有槽,然后i-1称为负数,走进这个逻辑,开始commit
        if (i < 0 || i >= n || i + n >= nextn) {
            int sc;
            // 判断是否扩容成功,当一个线程来协助扩容,发现任务已经被其他线程领取完了,由于finishing默认值false不会走到这,走到这只有一个可能,那就是本线程参与了扩容,且扩容完成
            if (finishing) {
                // 清空nextTable
                nextTable = null;
                // 替换table
                table = nextTab;
                // 设置sizeCtl为新容量的3/4,扩容完成sizeCtl为正数,扩容进行时,sizeCtl为负数
                sizeCtl = (n << 1) - (n >>> 1);
                return;
            }

            // 尝试cas减少协助扩容计数
            if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                // 	判断扩容线程数是否为1(注意sc的赋值,sc被赋值为cas前的sizeCtl)
                // 这就是为什么commmit逻辑里不需要再次减少sizeCtl sizeCtl 线程数是1表示扩容完成,或者扩容进入了commit前的检查工作,不需要新的线程进入协助
                if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                    return;

                // 到这说明,除了只剩下自己一个线程在扩容,则将finishing和advance都设置成true,在下一次for循环里,再次检查能否领到扩容任务(大概率领不到),设计的很巧妙,最后一个完成扩容的线程进行commit操作
                finishing = advance = true;
                // 设置i为n,很关键，因为还有一个线程在扩容,我不能下次直接进行commit操作,需要等待,直到所有线程扩容完毕,当i被设置成n,在while里领取不到新任务,且由于while的第一个if i会被减少1变成了n-1
                i = n; // recheck before commit
            }
        }

        // 旧tab的i位置为null
        else if ((f = tabAt(tab, i)) == null)
            // 则将旧位置替换成ForwardingNode,什么意思?，扩容的时候我也允许你新增哦,万一我扩容的时候你是null,结果把null移动到了新的tab上,然后扩容没结束,你又给我put来一个怎么办?,于是我将旧的tab设置成一个hash为MOVED的node,让你遇到就来协助扩容
            advance = casTabAt(tab, i, null, fwd);
        // 如果f旧数组的位置不是null,但是这个节点的hash是MOVED(-1)//代表已经被转移到新的数组位置
        else if ((fh = f.hash) == MOVED)
            // advance标记为true,我需要新的任务!
            advance = true; // already processed
        // 否则就是就tab[i]没有被移动,需要执行移动逻辑
        else {
            // 上锁,锁一个槽
            synchronized (f) {
                // double check
                if (tabAt(tab, i) == f) {
                    Node<K,V> ln, hn;
                    // fh 必然大于0(普通节点,或者链表),具体看concurrenhashMap的spread方法
                    if (fh >= 0) {
                        // 节点应该在原位置还是新位置？(原理和HashMap一致)
                        int runBit = fh & n;
                        // 旧节点
                        Node<K,V> lastRun = f;

                        // 遍历节点,查找链表最后一个节点(或者最后连续的一段需要在同一位置的节点)
                        for (Node<K,V> p = f.next; p != null; p = p.next) {
                            // 计算没个节点的旧的索引位置
                            int b = p.hash & n;
                            // 不相等,则设置成相等的,存在ABA问题?但这段代码的逻辑对ABA没要求
                            if (b != runBit) {
                                runBit = b;
                                // 同时更新节点
                                lastRun = p;
                            }
                        }

                        // runBit == 0 说明最后一个节点需要在原位置
                        if (runBit == 0) {
                            // 将ln设置成lastRun lowNode
                            ln = lastRun;
                            hn = null;
                        }
                        // 否则,说明最后一个节点要在新位置highNode
                        else {
                            // 设置hn
                            hn = lastRun;
                            // 同时将ln置为null,必须的,否则新链会多出来一些节点
                            ln = null;
                        }

                        // 再次遍历,节点p不是是最后一段在同一位置的连续节点的起始节点
                        // 构建新链,会反转之前的链
                        for (Node<K,V> p = f; p != lastRun; p = p.next) {
                            int ph = p.hash; K pk = p.key; V pv = p.val;
                            // 会不会疑惑lastRun后面的节点会不会丢了？
                            // 不会丢哦,因为lastRun是最后一段链的起始节点,在new Node的时候会加入到新链(这也优化,牛逼,这就解释了为什么上面即使出现aba也没事，因为我就是找最后一段在新链同一位置的短链)

                            // 不需要移动位置的
                            if ((ph & n) == 0)
                                // 构建新链，头插法
                                ln = new Node<K,V>(ph, pk, pv, ln);
                            // 需要移动位置
                            else
                                // 构建新链，头插法
                                hn = new Node<K,V>(ph, pk, pv, hn);
                        }

                        // 设置在新链原位置的链 且volatile
                        setTabAt(nextTab, i, ln);
                        // 设置在新链原位置的链cas 且volatile
                        setTabAt(nextTab, i + n, hn);
                        // 将旧tab的位置设置成hash为MOVED ForwardingNode节点
                        setTabAt(tab, i, fwd);
                        // 需要继续领取任务
                        advance = true;
                    }
                    // 树节点的扩容,不看了难度太大,后续等看完红黑树的实现在回来研究
                    else if (f instanceof TreeBin) {
                        TreeBin<K,V> t = (TreeBin<K,V>)f;
                        TreeNode<K,V> lo = null, loTail = null;
                        TreeNode<K,V> hi = null, hiTail = null;
                        int lc = 0, hc = 0;
                        for (Node<K,V> e = t.first; e != null; e = e.next) {
                            int h = e.hash;
                            TreeNode<K,V> p = new TreeNode<K,V>
                                (h, e.key, e.val, null, null);
                            if ((h & n) == 0) {
                                if ((p.prev = loTail) == null)
                                    lo = p;
                                else
                                    loTail.next = p;
                                loTail = p;
                                ++lc;
                            }
                            else {
                                if ((p.prev = hiTail) == null)
                                    hi = p;
                                else
                                    hiTail.next = p;
                                hiTail = p;
                                ++hc;
                            }
                        }
                        ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :
                        (hc != 0) ? new TreeBin<K,V>(lo) : t;
                        hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :
                        (lc != 0) ? new TreeBin<K,V>(hi) : t;
                        setTabAt(nextTab, i, ln);
                        setTabAt(nextTab, i + n, hn);
                        setTabAt(tab, i, fwd);
                        advance = true;
                    }
                }
            }
        }
    }
}
```

