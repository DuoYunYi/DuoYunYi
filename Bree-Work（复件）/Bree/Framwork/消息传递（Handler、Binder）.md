
# 一、Handler

[【Android Framework系列】第1章 Handler消息传递机制_android binder通信,handler-CSDN博客](https://blog.csdn.net/u010687761/article/details/130915378?spm=1001.2014.3001.5502)

## 1.简介

消息传递机制，Handler主要用于同进程的**线程间**通信。而Binder/Socket用于**进程间**通信。


## 2.Handler运行机制

 主要会涉及到四个类：Handler、Looper、Message、MessageQueue

一句话理解：**Message通过Handler添加到当前线程的MessageQueue队列，当前线程的Looper不断轮询获取Message并通知对应Handler处理消息。**
 
```
Handler：消息处理器，通过obtainMessage()生成消息和handleMessage()处理消息；

Looper：循环器，用于取出消息，每个线程只能够一个Looper，管理MessageQueue，不断从中将Message分发并通知Handler处理消息；

Message：消息类，Handler的接收和处理对象通过obtain()方法可以生成一个Message对象，每个Message对象必须有一个Handler对象target，要执行该消息时通过target找到对应Handler执行

MessageQueue：消息队列，用于存储消息，通过enqueueMessage()放入消息，next()取出消息。
```



假设在线程 A 中创建了 Handler 对象，重写了 handler 的 handleMessage() 方法，并且将 Handler 对象传给了线程 B，则线程 B 和线程 A 间的通讯如下：

```
执行准备：线程A 中执行Looper.prepare() 准备线程消息循环和消息队列（MessageQueue），执行Looper.loop()开起无限循环处理消息队列里的任务

生成消息：线程 B 通过 handler.obtainMessage() 生成消息（内部会调用 Message.abtain() 方法，并且会把当前 handler 传给 message 对象）；

发送消息：线程 B 通过 handler.sendMessage() 发送消息，再调用 mQueue.enqueueMessage() 将消息放入消息队列；

取出消息：线程A 中 mLooper 对象的 loop() 方法一直循环执行（若 mQueue 中没数据会等待），发现MessageQueue 中有 Message 对象并且到达执行时间则内部会调用 mQueue 对象的 next() 方法，取出一个消息 msg；

执行任务：线程 A 中取出的 msg 对象在创建时已绑定 Handler，通过调用 msg.target.dispatchMessage() 方法，此处的 target 就是步骤 3 中创建的Handler 对象，通知 Handler 执行 handleMessage() 方法。dispatchMessage(msg) 根据 msg 对象是否有 callback 对象（Runnable 对象），有 callback 对象，就直接执行 callback.run()，无 callback 对象，就进行 Handler.handleMessage(msg) 方法进行处理

```


**MessageQueue没有消息时，便阻塞在looper的queue.next()中的nativePollOnce()方法里，此时主线程会释放CPU资源进入休眠状态，直到下个消息到达或者有事务发生nativeWake()**


epoll的通俗解释是一种**当文件描述符的内核缓冲区非空的时候，发出可读信号进行通知，当写缓冲区不满的时候，发出可写信号通知的机制**


## 3.Handler相关问答

Q1：用一句话概括Handler，并简述其原理。
Handler是Android系统的根本，在Android应用被启动的时候，会分配一个单独的虚拟机，虚拟机会执行ActivityThread中的main方法，在main方法中对主线程Looper进行了初始化，也就是几乎所有代码都执行在Handler内部。Handler也可以作为主线程和子线程通讯的桥梁。Handler通过sendMessage发送消息，将消息放入MessageQueue中，在MessageQueue中通过时间的维度来进行排序，Looper通过调用loop方法不断的从MessageQueue中获取消息，执行Handler的dispatchMessage，最后调用handleMessage方法。

Q2：为什么系统不建议在子线程访问UI？（为什么不能在子线程更新UI？）
在某些情况下，在子线程中是可以更新UI的。但是在ViewRootImpl中对UI操作进行了
checkThread，但是我们在OnCreate和onResume中可以使用子线程更新UI，由于我们在
ActivityThread中的performResumeActivity方法中通过addView创建了ViewRootImpl，这个行为是在onResume之后调用的，所以在OnCreate和onResume可以进行更新UI。
但是我们不能在子线程中更新UI，因为UI更新本身是线程不安全的，如果添加了耗时操作之后，一旦ViewRootImpl被创建将会抛出异常。一旦在子线程中更新UI，容易产生并发问题。

Q3：一个Thread可以有几个Looper？几个Handler？
一个线程只能有一个Looper，可以有多个Handler，
线程在初始化时候调用Looper.prepare()方法对将静态对象ThreadLocal作为key（整个进程全部线程的Looper共用一个sThreadLocal），创建Looper对象作为value存储在当前线程的ThreadLocalMap中（其实就只有一个key对应value，key为sThreadLocal，value为looper对象。这两玩意都TM只有一个，所以ThreadLocalMap只有一个键值对。。。）。
通过线程独有的ThreadLocal实现将Looper存储在ThreadLocalMap这样键值对的数据结构中。
为什么是一个Looper：
因为Looper需要循环，循环后面的代码无法执行了，所以一个线程只有一个Looper
为什么是多个Handler：
为了相互独立，互不干扰，各自处理各自的消息，谁发生的Message谁处理，也可以通过removeMessages清空掉自己的Message。

Q4：可以在子线程直接new一个Handler吗？那该怎么做？
可以在子线程中创建Handler，我们需要调用Looper.perpare和Looper.loop方法。或者通过获取主线程的looper来创建Handler。

Q5：Message可以如何创建？哪种效果更好，为什么？
Message.obtain来创建Message。通过这种方式创建的Message会被存放在一个大小为50的复用池中，这样会复用之前的Message的内存，不会频繁的创建对象，导致内存抖动。

Q6：主线程中Looper的轮询死循环为何没有阻塞主线程？
Looper轮询是死循环，但是当没有消息的时候他会block（阻塞），ANR是当我们处理点击事件的时候5s内没有响应，我们在处理点击事件的时候也是用的Handler，所以一定会有消息执行，并且ANR也会发送Handler消息，所以不会阻塞主线程。Looper是通过Linux系统的epoll实现的阻塞式等待消息执行（有消息执行无消息阻塞），而ANR是消息处理耗时太长导致无法执行剩下需要被执行的消息触发了ANR。

Q7：使用Hanlder的postDealy()后消息队列会发生什么变化？
Handler发送消息到消息队列，消息队列是一个时间优先级队列，内部是一个单向链表。发动postDelay之后会将该消息进行时间排序存放到消息队列中。

Q8：点击页面上的按钮后更新TextView的内容，谈谈你的理解？（阿里面试题）
点击按钮的时候会发送消息到Handler，但是为了保证优先执行，会加一个标记异步，同时会发送一个target为null的消息，这样在使用消息队列的next获取消息的时候，如果发现消息的target为null，那么会遍历消息队列将有异步标记的消息获取出来优先执行，执行完之后会将target为null的消息移除。(同步屏障：拦截同步消息执行，优先执行异步消息。 View 更新时，draw、requestLayout、invalidate 等很多地方都调用异步消息的方法)

同步屏障的设置可以方便地处理那些优先级较高的异步消息。当我们调用Handler.getLooper().getQueue().postSyncBarrier() 并设置消息的setAsynchronous(true)时，target 即为 null ，也就开启了同步屏障。当在消息轮询器 Looper 在loop()中循环处理消息时，如若开启了同步屏障，会优先处理其中的异步消息，而阻碍同步消息。

Q9：生产者-消费者设计模式懂不？
举个例子，面包店厨师不断在制作面包，客人来了之后就购买面包，这就是一个典型的生产者消费者设计模式。但是需要注意的是如果消费者消费能力大于生产者，或者生产者生产能力大于消费者，需要一个限制，在java里有一个blockingQueue。当目前容器内没有东西的时候，消费者来消费的时候会被阻塞，当容器满了的时候也会被阻塞。Handler.sendMessage相当于一个生产者,MessageQueue相当于容器，Looper相当于消费者。但我们的Handler为什么不使用java的blockingQueue呢？原因是除了我们上层需要使用到Handler，其实底层的消息都是需要传递给Handler处理，比如：驱动层需要发事件给APP、屏幕点击事件、底层刷新通知等，所以我们使用的是Native层提供的MessageQueue实现消息队列。

Q10：Handler是如何完成子线程和主线程通信的？
在主线程中创建Handler，在子线程中发送消息，放入到MessageQueue中,通过Looper.loop取出消息进行执行handleMessage，由于looper我们是在主线程初始化的，在初始化looper的时候会创建消息队列，所以消息是在主线程被执行的。

Q11：关于ThreadLocal，谈谈你的理解？
ThreadLocal实例进程内只有一个（静态实例），但其内部的set和get方法是获取的当前线程的ThreadLocalMap对象，ThreadLocalMap是每个线程有一个单独的内存空间，不共享，ThreadLocal在set的时候会将数据存入对应线程的ThreadLocalMap中，key=ThreadLocal，value=值

Q12：享元设计模式有用到吗？
享元设计模式就是重复利用内存空间，减少对象的创建，Message中使用到了享元设计模式。内部维护了一个链表，并且最大长度是50，当消息处理完之后会将消息内的属性设置为空，并且插入到链表的头部，使用obtain创建的Message会从头部获取空的Message

Q13: Handler内存泄漏问题及解决方案
内部类持有外部类的引用导致了内存泄漏，如果Activity退出的时候，MessageQueue中还有一个Message没有执行，这个Message持有了Handler的引用，而Handler持有了Activity的引用，导致Activity无法被回收，导致内存泄漏。使用static关键字修饰，在onDestory的时候将消息清除。
简单理解：
当Handler为非静态内部类时，其持有外部类Actviity对象，所以导致Looper->MessageQueue->Message->Handler->Activity，这个引用链中Message如果还在MessageQueue中等待执行，则会导致Activity一直无法被释放和回收。

导致的原因：
因为Looper需要循环，所以一个线程只有一个Looper，但一个线程中可有多个Handler，MessageQueue中消息Message执行时不知道要通知哪个Handler执行任务，
所以在Message创建时中存入了Handler对象target用于回调执行的消息。如果Handler是Activity这种短生命周期对象的非静态内部类时，则会让创建出来的Handler对象持有该外部类Activity的引用，Message还在队列中导致引用着Handler，而非静态内部类Handler引用外部类Activity导致Activity无法被回收，最终导致内存泄漏。

解决办法：
1.Handler不能是Activity这种短生命周期的对象类的内部类；
2.在Activity销毁时，将创建的Handler中的消息队列清空并结束所有任务

Q14: Handler异步消息处理（HandlerThread）
内部使用了Handler+Thread，并且处理了getLooper的并发问题。如果获取Looper的时候发现Looper还没创建，则wait，等待looper创建了之后在notify

Q15: 子线程中维护的Looper，消息队列无消息的时候处理方案是什么？有什么用？
子线程中创建了Looper，当没有消息的时候子线程将会被block，无法被回收，所以我们需要手动调用quit方法将消息删除并且唤醒looper，然后next方法返回null退出loop

Q16: 既然可以存在多个Handler往MessageQueue中添加数据(发消息是各个handler可能处于不同线程)，那他内部是如何确保线程安全的？
在添加数据和执行next的时候都加了this锁，这样可以保证添加的位置是正确的，获取的也会是最前面的。

Q17: 关于IntentService，谈谈你的理解
HandlerThread+Service实现，可以实现Service在子线程中执行耗时操作，并且执行完耗时操作时候会将自己stop。

Q18: Glide是如何维护生命周期的？
一般想问的应该都是这里

```
    @NonNull
    private RequestManagerFragment getRequestManagerFragment(
            @NonNull final android.app.FragmentManager fm,
            @Nullable android.app.Fragment parentHint,
            boolean isParentVisible) {
        RequestManagerFragment current = (RequestManagerFragment)
                fm.findFragmentByTag(FRAGMENT_TAG);
        if (current == null) {
            current = pendingRequestManagerFragments.get(fm);
            if (current == null) {
                current = new RequestManagerFragment();
                current.setParentFragmentHint(parentHint);
                if (isParentVisible) {
                    current.getGlideLifecycle().onStart();
                }
                pendingRequestManagerFragments.put(fm, current);
                fm.beginTransaction().add(current,
                        FRAGMENT_TAG).commitAllowingStateLoss();
                handler.obtainMessage(ID_REMOVE_FRAGMENT_MANAGER, fm).sendToTarget();
            }
        }
        return current;
    }
```

1.为什么会判断两次null，再多次调用with的时候，commitAllowingStateLoss会被执行两次，所以我们需要使用一个map集合来判断，如果map中已经有了证明已经添加过了
2.handler.obtainMessage(ID_REMOVE_FRAGMENT_MANAGER, fm).sendToTarget()，我们需要将map里面的记录删除。


Q19:handler 主线程阻塞了怎么办，阻塞怎么唤醒？
Android系统事件驱动系统，loop循环处理事件，如果不循环程序就结束了

Q20:Handler底层为什么用epoll，不用select、poll？
Socket非阻塞IO中select需要全部轮询不适合，poll也是需要遍历和copy，效率太低了。epoll非阻塞式IO，使用句柄获取APP对象，epoll无遍历，无拷贝。还使用了红黑树（解决查询慢的问题）



# 二、Binder

[【Android Framework系列】第2章 Binder机制大全_android binder-CSDN博客](https://blog.csdn.net/u010687761/article/details/131057192?spm=1001.2014.3001.5502)

## 1.简介

Binder是Android中主要的**跨进程**通信方式。

Android系统中，每个应用程序是由Android的Activity，Service，BroadCast，ContentProvider这四剑客中一个或多个组合而成，这四剑客所涉及的**多进程间的通信底层都是依赖于BinderIPC机制**。例如当进程A中的Activity要向进程B中的Service通信，这便需要依赖于BinderIPC。

Zygote通信便是采用socket。
 

## 2.为什么要使用Binder

传统Linux进程间通信方式：管道、信号量、socket、共享内存，而Binder是Android系统独有的通讯方式

```
Binder：只需要拷贝一次，基于C/S架构，易用性高，系统为每个APP分配UID同时支持实名和匿名更安全
共享内存：无需拷贝，控制复杂，易用性差，依赖上层协议，访问接入点是开放的不安全

Socket：需要拷贝两次，基于C/S架构，作为一款通用接口，其传输效率低，开销大，以来上层协议，访问接入点是开放的，不安全


管道：需要拷贝两次，非C/S架构，是一对一的通讯模型，把一个程序的输出直接链接另一个程序的输入。
Linux下的管道主要分两种：无名管道(pipe)和有名管道(fifo)。无名管道只能用于具有亲缘关系的进程之间的通信(也就是父子进程或兄弟进程之间)，是以半双工的一个通信方式，速度较慢，容量有限；有名管道是对无名管道的一种改进，可以让两个互补相干的进程之间进行通信。并且该管道在文件系统中可见，不过大小一直为0。

信号量：与其他的进程间通信方式不太相同，它主要提供对进程间共享资源的访问机制，进程会根据它判定是否能够访问某些共享资源，同时进程也可以修改该标志。除了用于访问控制外，还可以用于进程同步，主要是用来解决进程线程之间同步与互斥问题的一种通信机制。
```


Android系统需要一种**高效率、安全性高**的方式，因此Binder最合适不过。**Binder只需要拷贝一次，效率仅次于共享内存，而且采用传统的C/S结构**

![[Pasted image 20250611105208.png]]


## 3.Binder实现机制
