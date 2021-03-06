---
layout: post
title: "ReentrantLock"
subtitle: 'Some tips on leraning ReentrantLock'
author: "FengJiaWen"
header-style: text
tags:
  - 可重入锁
  - 自旋
  - 锁
  - AQS
  - park
---

Update: ReentrantLock

---

**ReentrantLock**

ReentrantLock构造方法```new ReentrantLock()```为非公平锁;```new ReentrantLock(true)```表示公平锁.

1.lock()加锁过程:

**非公平锁**

{% highlight java %}
        final void lock() {
            if (compareAndSetState(0, 1))
                setExclusiveOwnerThread(Thread.currentThread());
            else
                acquire(1);
        }
{% endhighlight %}
**公平锁**
{% highlight java %}
        final void lock() {
            acquire(1);
        }
{% endhighlight %}
1.```若compareAndSetState()```方法返回ture,表明CAS成功,```setExclusiveOwnerThread()```方法将持有锁的线程设置为当前线程.
2.acquire(1)方法:第一次自旋
若state=0,则加锁成功,将持有锁的线程标记为当前线程.
若加锁失败,调用acquire(1)进行一次自旋.参数意义:假如被重入,计数器每次增加1,方便在解锁的时候-1,直到state=0标志锁为自由状态,之前被park的线程被唤醒,继续执行业务逻辑.

**acquire(arg)方法**
{% highlight java %}
        public final void acquire(int arg) {
            if (!tryAcquire(arg) &&
                acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
                selfInterrupt();
        }
{% endhighlight %}

**非公平锁尝试获取锁:nonfairTryAcquire(1)**
{% highlight java %}
        final boolean nonfairTryAcquire(int acquires) {
            //获取当前线程
            final Thread current = Thread.currentThread();
            //获取锁状态
            int c = getState();
            if (c == 0) {
                //如果锁为自由,执行CAS
                if (compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {
                //如果尝试自旋的线程为当前持有锁的线程,则计数器+1,重入1次
                int nextc = c + acquires;
                if (nextc < 0) // overflow
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            //如果锁状态不是0,直接返回
            return false;
        }
{% endhighlight %}
**addWaiter(Node mode)方法**
{% highlight java %}
    private Node addWaiter(Node mode) {
        //初始化一个为Thread为当前线程的Node的独占锁(nextwaiter的值为Node.EXCLUSIVE为独占锁,nextwaiter的值为Node.SHARED为共享锁.
        Node node = new Node(Thread.currentThread(), mode);
        // Try the fast path of enq; backup to full enq on failure
        Node pred = tail;
        //如果队列尾部不为null
        if (pred != null) {
            //把准备入队Node的前驱节点指向队列尾节点
            node.prev = pred;
            //CAS将队尾指向新建Node
            if (compareAndSetTail(pred, node)) {
                //将老队尾元素的后驱节点指向准备入队Node
                pred.next = node;
                return node;
            }
        }
        //如果队列尾部为null
        enq(node);
        return node;
    }
{% endhighlight %}
**enq(final Node node)方法**
{% highlight java %}
    private Node enq(final Node node) {
        for (;;) {
            Node t = tail;
                //如果队尾指向null,表示AQS队列没有初始化,开始初始化AQS队列
            if (t == null) { // Must initialize
                if (compareAndSetHead(new Node()))
                    //初始化一个头部为null的Node,再将头部赋给尾部,头尾为null
                    tail = head;
            } else {
                //如果队尾指向不为null,表示AQS队列已经初始化过,准备入队Node的前驱节点指向队尾元素
                node.prev = t;
                if (compareAndSetTail(t, node)) {
                    //CAS尾部,将队尾指向准备入队的Node,将队列尾部元素的后驱节点指向准备入队元素Node,入队完成.
                    t.next = node;
                    return t;
                }
            }
        }
    }
{% endhighlight %}
**acquireQueued(final Node node, int arg)方法**
{% highlight java %}
    final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                //取出新入队Node的前驱Node
                final Node p = node.predecessor();
                //如果前驱节点是头结点,自旋一次(有可能在入队过程持有锁的线程释放了锁,所以在入队完成且自己为第一个排队的人时自旋一次)
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return interrupted;
                }
                //判断获取锁失败之后是否需要休眠并且休眠成功
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
{% endhighlight %}
**公平锁尝试索取锁:tryAcquire(1)**
{% highlight java %}
        protected final boolean tryAcquire(int acquires) {
            //获取当前线程
            final Thread current = Thread.currentThread();
            //获取锁状态
            int c = getState();
            if (c == 0) {
                if (!hasQueuedPredecessors() &&
                    compareAndSetState(0, acquires)) {
                    //hasQueuedPredecessors方法判断队列是否有人正在排队.若无人排队且CAS成功则继续执行
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0)
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            //如果锁状态不是0,直接返回
            return false;
        }
    }
{% endhighlight %}
**hasQueuedPredecessors**
{% highlight java %}
    public final boolean hasQueuedPredecessors() {
        //队尾赋值给t
        Node t = tail; 
        //队头赋值给h
        Node h = head;
        Node s;
        return h != t &&
            ((s = h.next) == null || s.thread != Thread.currentThread());
    }
{% endhighlight %}
hasQueuedPredecessors方法理解:
1. ```h!=t```判断队头和队尾指向的Node不同,因为头节点始终是null,所以含义为队列中除了头结点 **(头结点thread值为null,不参与排队)** 还有线程在排队.(若队列已经初始化:返回false,队列未初始化队头队尾都指向null,返回false).
2. ```(s = h.next) == null```判断队头指向的头节点的后驱节点非null(下一个排队的人不是null); ```s.thread != Thread.currentThread()```判断下一个节点的线程不等于当前尝试索取锁的线程.这里个人理解作者判断非常严谨,因为整个return为非原子操作,判断了```h!=t```之后可能第一个线程释放了锁,第二个线程获取了锁,队列中无人排队了,所以再进行一次判断,保证队列中头结点的下一个节点不为null.

 **lock()与trylock()**
    lock():
        (1)如果锁的状态state为0,CAS加锁,将state置为1,返回true.(立即获取锁).
        (2)如果锁已经被当前线程持有,则重入锁,将state+1,返回true.
        (3)如果锁被其他线程持有,**则最后调用UNSAFE.park()方法阻塞睡眠.**
    trylock():
        (1)如果锁的状态state为0,CAS加锁,将state置为1,返回true.(立即获取锁).
        (2)如果锁已经被当前线程持有,则重入锁,将state+1,返回true.
        (3)如果锁已经被其他线程持有,**返回false.**
**2.unlock()解锁过程:**

{% highlight java %}
    public void unlock() {
        sync.release(1);
    }
{% endhighlight %}
1.此处参数"1"解释了为什么acquire(1)方法参数是1.(每次对锁的计数器加减1)

{% highlight java %}
    public final boolean release(int arg) {
        if (tryRelease(arg)) {
            //如果锁状态被释放,处于自由状态
            Node h = head;
            if (h != null && h.waitStatus != 0)
                //如果头结点非空,或者头结点有下一个节点(h.waitStatus只会被下一个节点改为!=0)则唤醒队列中的下一个线程
                unparkSuccessor(h);
            return true;
        }
        return false;
    }
{% endhighlight %}

{% highlight java %}
    protected final boolean tryRelease(int releases) {
            int c = getState() - releases;
            if (Thread.currentThread() != getExclusiveOwnerThread())
            //如果尝试释放锁的线程不等于持有锁的线程
                throw new IllegalMonitorStateException();
            boolean free = false;
            if (c == 0) {
                //若变量c为0,更改变量free为真,设置持有锁的线程为空
                free = true;
                setExclusiveOwnerThread(null);
            }
            //将锁的状态设置为自由,返回true,release(1)继续执行.
            setState(c);
            return free;
        }
{% endhighlight %}
