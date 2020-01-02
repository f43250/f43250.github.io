---
layout: post
title: "ReentrantReadWriteLock"
subtitle: 'Some tips on leraning ReentrantReadWriteLock'
author: "FengJiaWen"
header-style: text
tags:
  - 读锁
  - 写锁
  - 共享锁
  - 独占锁
---

Update: ReentrantReadWriteLock

---

**ReentrantReadWriteLock**

**读锁(共享锁)ReadLock**
{% highlight java %}
    public void lock() {
        sync.acquireShared(1);
    }
{% endhighlight %}

{% highlight java %}
    public final void acquireShared(int arg) {
        if (tryAcquireShared(arg) < 0)
            /*tryAcquireShared方法返回值:
              1.小于0表示获取锁失败;
              2.等于0表示获取锁成功,但后续线程会获取锁失败.
              3.大于0表示获取锁成功,且后续线程很可能获取锁成功.
            */
            doAcquireShared(arg);
    }
{% endhighlight %}
**doAcquireShared(arg)**
{% highlight java %}
    private void doAcquireShared(int arg) {
        final Node node = addWaiter(Node.SHARED);
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                final Node p = node.predecessor();
                if (p == head) {
                    //如果入队节点的前驱节点是队列头部(含义为自己是第一个排队的人,持有线程的头结点不算排队的人),则自旋一次
                    int r = tryAcquireShared(arg);
                    if (r >= 0) {
                        setHeadAndPropagate(node, r);
                        p.next = null; // help GC
                        if (interrupted)
                            selfInterrupt();
                        failed = false;
                        return;
                    }
                }
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
**setHeadAndPropagate(node, r)**
{% highlight java %}
    private void setHeadAndPropagate(Node node, int propagate) {
        Node h = head; 
        setHead(node);
        if (propagate > 0 || h == null || h.waitStatus < 0 ||
            (h = head) == null || h.waitStatus < 0) {
            /*如果传播属性大于0或者头结点为null或者头节点的等待状态为小于0(有后续线程排队)或者成为新头节点的Node为null,或者新头结点的等待状态为-1.
            */
            Node s = node.next;
            //如果成为新头结点的Node的下个节点为null或者下个节点为共享属性
            if (s == null || s.isShared())
                //如果后续节点为空或者或许节点是共享属性的,CAS将头结点的ws设置为Node.PROPAGATE(-3).
                doReleaseShared();
        }
    }
{% endhighlight %}
**doReleaseShared()**
{% highlight java %}
    private void doReleaseShared() {
        for (;;) {
            Node h = head;
            if (h != null && h != tail) {
                int ws = h.waitStatus;
                if (ws == Node.SIGNAL) {
                    if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                        continue;            // loop to recheck cases
                    unparkSuccessor(h);
                }
                else if (ws == 0 &&
                         !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                    continue;                // loop on failed CAS
            }
            if (h == head)                   // loop if head changed
                break;
        }
    }
{% endhighlight %}
**写锁(独占锁)WriteLock**
{% highlight java %}
    public void lock() {
        sync.acquire(1);
    }
{% endhighlight %}
1.与ReentrantLock中的调用的lock方法为同一个.

**未完待续....**
