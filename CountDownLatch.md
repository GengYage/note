### CountDownLatch

CountDownLatch是juc停工的并发工具。帮助一个或多个线程等待一组操作完成后再继续执行

注意一致强调的，AQS的state具体表示什么意思，只有在其子类才能确定，并不是一成不变的，在ReentrantLock中是独占锁模式，在CountDownLatch中则是共享锁模式。

### 继承结构

```java
// CDL内部的Sync类继承了AQS,只有有参构造器
// CDL的方法都是委托给Sync子类操作的
private static final class Sync extends AbstractQueuedSynchronizer {
    private static final long serialVersionUID = 4982264981922014374L;

    // 有参构造
    Sync(int count) {
        setState(count);
    }

    // 获取State
    int getCount() {
        return getState();
    }

    protected int tryAcquireShared(int acquires) {
        return (getState() == 0) ? 1 : -1;
    }

    // 释放共享锁,分析在下文
    protected boolean tryReleaseShared(int releases) {
        // Decrement count; signal when transition to zero
        for (;;) {
            int c = getState();
            if (c == 0)
                return false;
            int nextc = c-1;
            if (compareAndSetState(c, nextc))
                return nextc == 0;
        }
    }
}
```

### countDown

```java
// CDL
public void countDown() {
    sync.releaseShared(1);
}

// AQS
public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)) {
        doReleaseShared();
        return true;
    }
    return false;
}

// CDL.Sync
protected boolean tryReleaseShared(int releases) {
    // Decrement count; signal when transition to zero
    for (;;) {
        // 获取当前state的值
        int c = getState();
        // 已经被释放完,则返回false
        if (c == 0)
            return false;
        // -1
        int nextc = c-1;
        // cas设置state,设置失败则一次for循环,直到设置成功
        if (compareAndSetState(c, nextc))
            // 返回值是 state是否被减到0
            return nextc == 0;
    }
}

// AQS 唤醒等待队列上的节点
// 在CDL里 作用是唤醒等待队列的所有节点
private void doReleaseShared() {
    for (;;) {
        Node h = head;
        // 头结点不为null,且头结点不等于尾结点(等待队列有数据)
        if (h != null && h != tail) {
            int ws = h.waitStatus;

            // ws是-1
            if (ws == Node.SIGNAL) {
                // 设置状态为初始值
                if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                    continue;            // loop to recheck cases
                // 然后唤醒头结点的后驱
                unparkSuccessor(h);
            }
            // ws 是0,则设置状态为-3 CDL用不到
            else if (ws == 0 &&
                     !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                continue;                // loop on failed CAS
        }

        // 头结点有变更,就去唤醒下一个
        if (h == head)                   // loop if head changed
            break;
    }
}
```

### await

```java
// CDL
public void await() throws InterruptedException {
    sync.acquireSharedInterruptibly(1);
}

// AQS
public final void acquireSharedInterruptibly(int arg)
    throws InterruptedException {
    // 线程是中断唤醒,直接抛异常
    if (Thread.interrupted())
        throw new InterruptedException();
    // 如果返回值是1,则返回await,无须阻塞线程
    // 否则
    if (tryAcquireShared(arg) < 0)
        doAcquireSharedInterruptibly(arg);
}

// CDL.Sync
// 和参数就没有关系了,只有states是0,才返回1,否则就返回-1, 也就是说只有states被countDown到0才会返回1
protected int tryAcquireShared(int acquires) {
    return (getState() == 0) ? 1 : -1;
}

// AQS
private void doAcquireSharedInterruptibly(int arg)
    throws InterruptedException {
    // 熟悉的addWaiter,只不过节点状态是SHARED,将节点加入等待队列,不会尝试抢锁,之加入队列
    final Node node = addWaiter(Node.SHARED);
    // 获取锁失败标记 
    boolean failed = true;

    // 解锁逻辑,类似于ReentrantLock的非公平锁
    try {
        for (;;) {
            final Node p = node.predecessor();

            // 同样头结点的后驱才有资格唤醒别的线程
            // 在CDL里,头结点的后驱会一直自旋,直到state减为0
            // 高效着勒,多个线程await,只会有一个线程自旋,其他线程park
            if (p == head) {
                // 查看当前状态
                int r = tryAcquireShared(arg);
                // 是1,state=0的情况,先看state不是0的情况比较好理解
                if (r >= 0) {
                    // state是0,设置当前节点为头结点
                    setHeadAndPropagate(node, r);
                    // 头结点的后驱指针设置为null
                    p.next = null; // help GC
                    failed = false;
                    // 返回
                    return;
                }
            }
            
            // 睡觉相关处理 shouldParkAfterFailedAcquire 方法入其名,获取失败是否是要park?
            // 只有多个节点都在await才会出现
            // 在CDL里,会将前驱的ws设置为-1,返回值是false
            // 但是第二次循环到这就返回true了,因为前驱ws是-1 SIGNAL
            // 所以最简单的用法错误来了,只有一个线程,先调用await后调用countDown就一直睡,或者多线程,先调用await后调用countDown都会导致调用await的线程一直睡死
            if (shouldParkAfterFailedAcquire(p, node) &&
                // 线程睡觉,响应中断,有中断直接异常
                parkAndCheckInterrupt())
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}


// AQS 在CDL中propagate 为1
private void setHeadAndPropagate(Node node, int propagate) {
    Node h = head; // Record old head for check below
    // 将节点设置为头结点,其实就是老头结点出队
    setHead(node);
    // CDL一定会进来因为propagate是1
    if (propagate > 0 || h == null || h.waitStatus < 0 ||
        (h = head) == null || h.waitStatus < 0) {

        Node s = node.next;
        // 后继节点为null或者是共享锁模式
        // CDL一定会进来,因为是共享锁
        if (s == null || s.isShared())
            doReleaseShared();
    }
}

// 再讲一遍
private void doReleaseShared() {
    for (;;) {
        Node h = head;
        // 等待队列有节点
        if (h != null && h != tail) {
            // 在CDL中ws会被设置为-1 SIGNAL
            int ws = h.waitStatus;
            
            // CDL
            if (ws == Node.SIGNAL) {
                // 设置节点ws为0,设置失败则继续设置下一个,否则唤醒节点,CDL 会存在LockSupport.unpark(null)
                if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                    // cas设置状态失败则自选尝试
                    continue;            // loop to recheck cases
                // 唤醒后继节点,注意节点在哪睡的,是在doAcquireSharedInterruptibly:shouldParkAfterFailedAcquire park的,唤醒后,头结点的后驱 会先设置头结点是自己, ws为-1或者0,然后会再次走到doReleaseShared方法。是-1就唤醒下一个节点。在CDL里不存在-3的状态,因为如果是最后一个节点,在外层的if里不满足h != null && h != tail, 最后一个节点时h==tail
                unparkSuccessor(h);
            }
            else if (ws == 0 &&
                     !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                continue;                // loop on failed CAS
        }
        // CDL不存在头结点变更,因为先设置状态,后出队头结点
        // 头结点
        if (h == head)                   // loop if head changed
            break;
    }
}
```

