### Condition

如果说AQS实现了synchronized的功能，那么Condition则实现了Object中wait和notify的功能。

#### Condition接口定义

```java
public interface Condition {
    void await() throws InterruptedException;
    void awaitUninterruptibly();
    long awaitNanos(long nanosTimeout) throws InterruptedException;
    boolean await(long time, TimeUnit unit) throws InterruptedException;
    boolean awaitUntil(Date deadline) throws InterruptedException;

    void signal();
    void signalAll();
}
```
#### AQS中的Condition
```java
public class ConditionObject implements Condition, java.io.Serializable {
    // Node 就是AQS中通用的Node
    private transient Node firstWaiter;
    private transient Node lastWaiter;
}


static final class Node {
    static final int CANCELLED =  1;
    static final int SIGNAL    = -1;
    static final int CONDITION = -2;
    static final int PROPAGATE = -3;
    volatile int waitStatus;
    volatile Node prev;
    volatile Node next;
    volatile Thread thread;
    // 标记节状态
    Node nextWaiter;
    static final Node EXCLUSIVE = null;

    Node(Thread thread, Node mode) {     // Used by addWaiter
        this.nextWaiter = mode;
        this.thread = thread;
    }
}
```

#### 唤醒signal

```java
// AQS.ConditionObject
public final void signal() {
    // isHeldExclusively方法由子类实现
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    // 与AQS一样Condition也是一个双向链表
    Node first = firstWaiter;
    // 只有不是空链表,才会执行唤醒操作
    if (first != null)
        doSignal(first);
}

// ReentrantLock,在ReentrantLock中只有用有锁的线程才能执行唤醒操作
protected final boolean isHeldExclusively() {
    return getExclusiveOwnerThread() == Thread.currentThread();
}

// AQS.ConditionObject
private void doSignal(Node first) {
    do {
        // 赋值头结点为first的后继节点,其实就是旧头节点出队,如果已经遍历到队尾,这个if是true
        if ( (firstWaiter = first.nextWaiter) == null)
            // 遍历到队尾,则把尾结点的指针也置为null
            lastWaiter = null;
        // 断开旧的头结点的后驱指针,(help gc)
        first.nextWaiter = null;
        
        // 如果节点入队成功则doSingle结束
        // 否则,判断一下firstWaiter是否为null,为null当然也没必要继续循环了,因为队列空了,我去唤醒设呢？同时注意first的赋值,
    } while (!transferForSignal(first) &&
             (first = firstWaiter) != null);
}
// 整个方法的逻辑就是 我去遍历Condition的条件队列,从头到尾的唤醒,成功唤醒一个节点就结束循环,同时会出队,这里的唤醒不是说抢锁成功,而是给你一个抢锁的资格(PS:加入AQS的等待队列,并将前驱节点设置为SIGNAL,让其前驱负责唤醒自己去抢锁)

// AQS.ConditionObject
final boolean transferForSignal(Node node) {
    // 将节点的状态设置为CONDITION,期望值-2,修改为0(初始状态)
    // 修改失则直接返回false,然后尝试唤醒下一个节点
    if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
        return false;
    // 修改成功,则将节点加入AQS的等待队列,注意节点的状态被修改为了默认值0
    // enq返回的是节点的前驱节点
    Node p = enq(node);
    
    int ws = p.waitStatus;
    // 若前驱节点的状态>0也就是CANCELLED状态,或者将前驱节点ws改为-1失败,则会直接唤醒节点,尝试抢锁
    // (ps 回忆一下AQS,前驱节点负责去唤醒后继,所以才需要修改前驱的状态为SIGNAL)
    if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
        LockSupport.unpark(node.thread);
    // 最后只要是节点进入AQS的等待队列了,就返回true
    return true;
}
```

#### 条件等待wait

```java
// AQS.ConditionObject
public final void await() throws InterruptedException {
    // 如果线程中断标记位为true,有中断,则在这里处理中断(抛出异常,为什么?避免因为wait导致wait前的中断被吞)
    if (Thread.interrupted())
        throw new InterruptedException();

    // 节点入队操作 会清理条件队列里状态不为-2的节点
    Node node = addConditionWaiter();

    // 保存一下释放了多少个锁
    int savedState = fullyRelease(node);
    int interruptMode = 0;

    while (!isOnSyncQueue(node)) {
        // 锁释放之后就睡吧,等待被唤醒
        LockSupport.park(this);

        // ps 如果是unpark被唤醒,那么节点已经被加入aqs等待队列了,并且状态被改为0初始态,上述signal有分析

        // 被唤醒后判断是如何被唤醒后的,中断唤醒才会跳出循环
        // 没有发生中断 就是unpark唤醒的,unpark就是signal(),会将节点状态设置为初始值,此时isOnSyncQueue 就返回true,循环结束
        // 发生中断,状态修改为初始值成功 返回-1 THROW_IE (会将节点加入到aqs的等待队列,状态被修改成功意味着,不是被unpark唤醒的,但是只要被唤醒了我就给你加入到aqs等待队列,在这我姑且定义为异常唤醒)
        // 发生中断,状态修改为初始值失败 返回 1  REINTERRUPT 线程调用yield 
        // 其实就是transferAfterCancelledWait 通知取消等待状态,因为被唤醒后,就是条件得到了满足
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;
    }

    // 把释放的锁在抢回来, 且不是异常唤醒会将异常状态设置为REINTERRUPT
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    // 节点的后驱不是空,则将节点从条件队列移出,并清理其他需要被清除的节点
    if (node.nextWaiter != null) // clean up if cancelled
        unlinkCancelledWaiters();
    // 判断是抛出中断还是只设置中断标记 THROW_IE -1 抛出异常(毕竟是异常唤醒), REINTERRUPT 只标记
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
}


// AQS.ConditionObject
private Node addConditionWaiter() {
    // 拿到条件队列尾指针
    Node t = lastWaiter;
    // 若尾结点存在,且状态不为CONDITION-2,则清理队列中所有状态不为-2的节点
    if (t != null && t.waitStatus != Node.CONDITION) {
        unlinkCancelledWaiters();
        // 清理完毕后,更新临时变量t指向的尾节点(因为尾节点会在上一步被清除,尾结点就发生变化了)
        t = lastWaiter;
    }

    // 创建节点,初始状态为-2
    Node node = new Node(Thread.currentThread(), Node.CONDITION);

    // 若队列为空
    if (t == null)
        // 头结点指针指向node,入队
        firstWaiter = node;
    else
        // 否则就是尾结点的后继指针指向node
        t.nextWaiter = node;

    // 最后都有更新尾结点指针为node,尾插法
    lastWaiter = node;
    return node;
}


// AQS.ConditionObject 清理所有链表中状态不为CONDITION的节点
private void unlinkCancelledWaiters() {
    // 拿到头结点
    Node t = firstWaiter;
    // 这个指针 指向当前节点的前驱节点
    Node trail = null;

    while (t != null) {
        // 节点的后继节点
        Node next = t.nextWaiter;

        // 判断,节点的等待状态只要不是-2(CONDITION)就执行清理操作,将当前节点从队列移出
        if (t.waitStatus != Node.CONDITION) {
            // help gc,断开后继指针,上文已经暂存了后继节点
            t.nextWaiter = null;
            // 前驱节点为null
            if (trail == null)
                // 修改ConditionObject的头结点为当前节点的后继节点
                firstWaiter = next;
            else
                // 否则,将前驱节点的后继指针指向当前节点的后继节点,单向联表删除节点的操作
                trail.nextWaiter = next;
            // 如果当前节点的后继为空,则修改ConditionObject的尾结点指针的为null
            if (next == null)
                lastWaiter = trail;
        }
        else
            // 否则 更新前驱指针为当前节点,遍历整个链表,清理所有状态不是-2(CONDITION)状态的节点
            trail = t;
        t = next;
    }
}


// AQS.ConditionObject  // 哦豁,线程主动wait,会释放线程拿到的所有锁
final int fullyRelease(Node node) {
    boolean failed = true;
    try {
        // 拿到锁的状态,ps可重入锁这里可不一定是1
        int savedState = getState();

        // 释放所有锁,回忆一下release 的返回值是是否完全释放了锁,并且会唤醒等待队列的第一个可用节点去抢锁
        if (release(savedState)) {
            失败表示设置为false,成功了当时就是没有失败
                failed = false;
            // 返回值是释放的锁的数量
            return savedState;
        } else {
            throw new IllegalMonitorStateException();
        }
    } finally {
        // 如果释放锁失败,则更改节点的状态为等待
        if (failed)
            node.waitStatus = Node.CANCELLED;
    }
}


// 是否加入了等待队列，加入等待队列就返回true,否则返回false
final boolean isOnSyncQueue(Node node) {
    // 状态是等待,前驱是null,则是没有加入等待队列
    if (node.waitStatus == Node.CONDITION || node.prev == null)
        return false;
    // 如果有后继节点,那么一定在等待队列
    if (node.next != null) // If has successor, it must be on queue
        return true;

    // 上述条件不满足,从等待队列尾部查找节点
    // 理论上讲, 上述情况都不满足,只有一种可能,节点是尾节点,或者我在判断节点状态时,有新的节点入队,但是新节点没完全连上队列,因此无论是那种,从队尾遍历都是最优的路径
    return findNodeFromTail(node);
}

// 从等待队列尾部查找节点
private boolean findNodeFromTail(Node node) {
    Node t = tail;
    for (;;) {
        if (t == node)
            return true;
        if (t == null)
            return false;
        t = t.prev;
    }
}
```

#### awaitUninterruptibly 不响应中断的等待

```java
public final void awaitUninterruptibly() {
    // 加入条件队列
    Node node = addConditionWaiter();
    // 释放锁
    int savedState = fullyRelease(node);
    // 设置中断状态
    boolean interrupted = false;
    
    // 直接睡
    while (!isOnSyncQueue(node)) {
        LockSupport.park(this);
        // 睡醒第一件事就是看看是不是被中断唤醒的
        if (Thread.interrupted())
            interrupted = true;
    }
    // 尝试抢锁,抢到锁发生中断,或者wait是被中断唤醒的,就设置一下终端状态,而不是抛出异常
    if (acquireQueued(node, savedState) || interrupted)
        selfInterrupt();
}
```

#### signalAll唤醒所有

```java
public final void signalAll() {
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    Node first = firstWaiter;
    if (first != null)
        doSignalAll(first);
}

// 比较简单
private void doSignalAll(Node first) {
    lastWaiter = firstWaiter = null;
    do {
        Node next = first.nextWaiter;
        first.nextWaiter = null;
        transferForSignal(first);
        first = next;
    } while (first != null);
}

// 唤醒节点
final boolean transferForSignal(Node node) {
    // 一般来讲只要状态还是CONDITION,就会唤醒成功(多线程并发,也最少一个线程会唤醒该node)
    if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
        return false;

    // 加入等待队列
    Node p = enq(node);
    // 前驱的ws
    int ws = p.waitStatus;
    // 前驱负责唤醒node
    if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
        // 唤醒线程,ps(被wait的线程,被唤醒后第一件事就是检查中断状态,随后去抢锁)
        LockSupport.unpark(node.thread);
    return true;
}
```

#### await(long, TimeUnit) 等待超时时间

```java
// 到这之后就比较简单了
public final boolean await(long time, TimeUnit unit)
    throws InterruptedException {
    // 需要等待的时间
    long nanosTimeout = unit.toNanos(time);
    if (Thread.interrupted())
        throw new InterruptedException();
    // 加入条件队列
    Node node = addConditionWaiter();
    // 释放锁
    int savedState = fullyRelease(node);
    // 计算出deadLine
    final long deadline = System.nanoTime() + nanosTimeout;
    // 设置超时标记
    boolean timedout = false;
    // 设置终端标记
    int interruptMode = 0;

    // 死循环
    while (!isOnSyncQueue(node)) {
        // 等待时间小于0则尝试将节点的状态改为初始状态,并且加入aqs等待队列
        if (nanosTimeout <= 0L) {
            timedout = transferAfterCancelledWait(node);
            // 跳出循环,注意哦,这里的条件队列并没有移出该节点,所以我们看到在别的操作里,会去遍历条件队列,
            // 移出那些状态不是CONDITION的节点(ps 都是设置了超时时间的节点,超时之后的取消了等待状态)
            break;
        }
        // 等待时间大于阈值1000L纳秒则睡觉
        if (nanosTimeout >= spinForTimeoutThreshold)
            LockSupport.parkNanos(this, nanosTimeout);


        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;

        // 更新需要等待的时间
        nanosTimeout = deadline - System.nanoTime();
    }

    // 没啥区别了
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    if (node.nextWaiter != null)
        unlinkCancelledWaiters();
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
    return !timedout;
}
```

