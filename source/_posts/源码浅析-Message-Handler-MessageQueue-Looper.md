title: 源码浅析: Message/Handler/MessageQueue/Looper
date: 2012-11-13 00:40:56
tags: [Android, 消息循环, Message, Handler, MessageQueue, Looper]
categories: [Android]
---

项目终于没那么忙了！闲下来几天，想想应该学点什么，总结点什么。总体上来，要学的东西实在太多了，看了看自己写的代码，结果发现连最基本的消息机制都没有了解清楚，虽然一直在用Handler发消息（Message），但一直没有去探究它们内部是如何运作的。于是花了一天的时间仔细分析了一下几个基本类的源码，略有所悟，浅析一下。
<!--more-->
**在探究源码之前，我觉得有必要有一个温习一下Windows的消息机制，以示对比。**
**以消息为基础，以事件驱动之（Messagebased, event driven）**
（注：以下部分摘自 侯捷----《深入浅出MFC》）

熟悉**Win32/MFC**编程的人都知道，Windows应用程序的都是**以事件来驱动**。换句话说，程序不断等待（利用一个while回路），等待任何可能的输入，然后做判断，然后再做适当的处理。这个「输入」就是以消息（Message）的形式表现出来的。这个过程可以用下图简单的表示。如果把应用程序获得的各种「输入」分类，可以分为由硬件装置所产生的消息（如鼠标移动或键盘被按下），放在系统队列（system queue）中，以及由Windows系统或其它Windows程序传送过来的消息，放在程序队列（application queue）中（在这个过程中，使用SendMessage(…)和PostMessage(…)这两个API来发送消息）。以应用程序的眼光来看，消息就是消息，来自哪里或放在哪里其实并没有太大区别，反正程序调用GetMessage API就取得一个消息，程序的生命靠它来推动。
可想而知，每一个Windows程序都应该有一个回路如下：
```C++
MSG msg;
while (GetMessage(&msg, NULL, NULL, NULL)) {
    TranslateMessage(&msg);
    DispatchMessage(&msg);
}
```

对于接受并处理消息的主角就是窗口。每一个窗口都应该有一个函数负责处理消息，程序员必须负责设计这个所谓的「窗口函数」（window procedure）。如果窗口获得一个消息，这个窗口函数必须判断消息的类别，决定处理的方式。
```C++
WndProc(hwnd, msg, wParam, lParam) {
    switch (msg) {
        case WM_CREATE: ...// do something, break
        case WM_COMMAND: ...
        case WM_LBUTTONDOWN: ...
        case WM_PAINT: ...
        case WM_CLOSE: ...
        case WM_DESTROY: ...
        default: return DefWindowProc(...);
    }
    return(0);
}
```

所以程序一旦运行起来，就会不停的：**获取消息-->分发消息-->处理消息-->获取消息-->….**
消息从哪里来？嗯。上面提到过了，从消息队列（消息泵）里来。如果没有消息怎么办，那就在获取消息（GetMessage）的这个函数这里阻塞住，直到有新的消息发送到消息队列里。
![windows-message-loop](/image/source-analyze/windows-message-loop.png)

如果你了解上面所描述的概念，再看Android的消息循环应该会发现真的很简单，只不过多了一些封装，多了一些类而已，整体思路都是大同小异的。

## 相关概念

在看源码前，我们先需要熟悉一下它们的概念及作用。
 - **Message：**用于封装消息的简单数据结构。里面包含消息的ID、数据对象、处理消息的`Handler`引用和`Runnable`等。
 - **Handler：**消息的发送者和最终消息处理者。
 - **MessageQueue：**消息队列，提供消息的添加、删除、获取等操作来管理消息队列。
 - **Looper：**用于建立消息循环并管理消息队列（`MessageQueue`），不停的从消息队列中抽取消息，分发下去并执行。

注：以下分析均以 **android 2.3.3** 源码为基础。

## Message源码分析

### 成员变量

我们先看一下它的成员变量。
```java
public final class Message implements Parcelable {
    public int what;
    public int arg1; 
    public int arg2;
    public Object obj;
    public Messenger replyTo;
    /*package*/ long when;
    /*package*/ Bundle data;
    /*package*/ Handler target;     
    /*package*/ Runnable callback;   
    // sometimes we store linked lists of these things
    /*package*/ Message next;
    ......
}
```

简单参数就不需要解释了，重点在以下几个成员变量。
 - **long when：**该消息何时被处理的绝对时间戳。
 - **Handler target：**谁来处理该消息。如果它为空，那说明该消息可能被recycle掉了，存放在Message Pool中，或者，它代表一个QUIT消息。
 - **Runnable callback：**Runnable对象，如果为该Message设置了该对象，那么有优先执行它。这里需要看Handler的消息处理机制。在分析Handler时再提。
 - **Message next：**这个看起来有点奇怪，有种似曾相识的感觉，想想，到底什么情况。哦，想起来了，就是C语言里面链表的数据结构。

```C++
typedef struct LNode{
    ElemType data;
    Struct LNode* next;
}
```

由于JAVA是没有指针这个概念的，所以内部维护了一个**next的引用**。所以，实际上，Message本身不单纯是一个简单的只包含数据的类，**它实际上是一个链式结构的类，也就是说，一个Message本身就是一个消息队列，它通过next将所有消息串联起来。既然Message本身就是消息队列**，那MessageQueue又是如何建立消息队列的又是怎么回事？实际上，MessageQueue内部只有一个Message成员，它所要做的工作就是把Message实体串连起来，形成消息链。
接着再看静态成员变量：
```java
private static Object mPoolSync = new Object(); // 用于访问mPool时进行同步操作
private static Message mPool;                   // 全局的废弃消息池（链）（就是废品收购站）
private static int mPoolSize = 0;               // 消息池当前大小
private static final int MAX_POOL_SIZE = 10;    // 消息池上限值
```

从这上面能看出，有个叫`mPool`的Message对象，如果理解了Message本身就是链表结构，那么，应该就明白了为什么一个消息叫Pool（池），因为一个Message本身就代表着一群Message，通过next把一系列Message给串联起来。对于无数个message实体来说，他们共享同一个全局的消息池（链），里面存放废弃掉的message。很明显，这是在做`缓存机制`。
在该类中，核心函数有：

```java
public static Message obtain()及系列函数
public void recycle()
public void sendToTarget()
```

### obtain()及其系列函数
**obtain()**系列函数最核心的函数就只有obtain()方法，其它函数只不过提供了更多的可选参数，内部都是调用obtain()方法，因此，我们只需要关注核心函数的实现即可。
```java
/**
 * Return a new Message instance from the global pool. Allows us to
 * avoid allocating new objects in many cases.
 */
public static Message obtain() {
    synchronized (mPoolSync) {
        if (mPool != null) {
            Message m = mPool;
            mPool = m.next;
            m.next = null;
            return m;
        }
    }
    return new Message();
}
```

该函数内部首先是从全局的废弃消息池（链）中去取，看看有没有废弃掉的Message，如果有，那我们就获取消息链中第一个废弃掉的Message，这样，就无需再创建一个新的Message；如果消息池中没有，那就只能new一个新的消息出来。这样做的好处就是废物再利用，减少创建时间。实际上，这种思想很值得我们借鉴。对于其它重载版的obtain方法，内部都是先调用它，然后再使用其它额外的参数进行填充的。如：

```java
public static Message obtain(Handler h, int what, int arg1, int arg2, Object obj) {
    Message m = obtain();
    m.target = h;
    m.what = what;
    m.arg1 = arg1;
    m.arg2 = arg2;
    m.obj = obj;

    return m;
}
```

### recycle()函数

```java
/**
 * Return a Message instance to the global pool.  You MUST NOT touch
 * the Message after calling this function -- it has effectively been
 * freed.
 */
public void recycle() {
    synchronized (mPoolSync) {
        if (mPoolSize < MAX_POOL_SIZE) {
            clearForRecycle();
            
            next = mPool;
            mPool = this;
        }
    }
}
```

其中：clearForRecycle代码如下
```java
/*package*/ void clearForRecycle() {
    what = 0;
    arg1 = 0;
    arg2 = 0;
    obj = null;
    replyTo = null;
    when = 0;
    target = null;
    callback = null;
    data = null;
}
```

这个函数首先是看当前消息池中废弃个数已达上限（池子是不是满了），如果没有达到上限，则调用`clearForRecycle()`函数把当前消息的各种信息清空，然后添加到消息链的头部。注意：该函数的`if (mPoolSize < MAX_POOL_SIZE)`实际上是没有起到任何作用的，搜遍Message所有代码也没有发现`mPoolSize`的值有任何变化，始终为0，也就是说，这句话是恒成立的。**只要该Message被recycle掉，那他就会加入到废弃链中**。
可以用以下图示表示该过程：
![android-message-chain](/image/source-analyze/android-message-chain.png)

**值得说明一点的是，该recycle()函数何时被调用？有以下两个时机被调用：**
 - MessageQueue类中的removeMessages(...)及其系列函数，即当我们要从消息队列中干掉一个Message时，该Message被回收到废弃消息链。

```java
final boolean removeMessages(Handler h, int what, Object object, boolean doRemove) {  
    synchronized (this) {  
        Message p = mMessages;  
        boolean found = false;  
  
        // Remove all messages at front.  
        while (p != null && p.target == h && p.what == what  
               && (object == null || p.obj == object)) {  
            if (!doRemove) return true;  
            found = true;  
            Message n = p.next;  
            mMessages = n;  
            p.recycle();  
            p = n;  
        }  
        // Remove all messages after front 
        while (p != null) {  
            Message n = p.next;  
            if (n != null) {  
                if (n.target == h && n.what == what  
                    && (object == null || n.obj == object)) {  
                    if (!doRemove) return true;  
                    found = true;  
                    Message nn = n.next;  
                    n.recycle();  
                    p.next = nn;  
                    continue;  
                }  
            }  
            p = n;  
        }  
          
        return found;  
    }  
}  
```

 - Looper类中的loop函数。即当我们使用完了某个Message后，该Message被回收到废弃消息链。

```java
public static final void loop() {  
    Looper me = myLooper();  
    MessageQueue queue = me.mQueue;  
    while (true) {  
        Message msg = queue.next(); // might block  
        if (msg != null) {  
            if (msg.target == null) {  
                // No target is a magic identifier for the quit message.  
                return;  
            }  
            msg.target.dispatchMessage(msg);  
            msg.recycle();  
        }  
    }  
}  
```

### sendToTarget()函数
```java
public void sendToTarget() {
    target.sendMessage(this);
}
```

该函数比较简单，就是通过Message内部引用的Handler将消息发送出去。

## Handler源码分析

### 成员变量

我们先看一下它的成员变量：
```java
final MessageQueue mQueue;
final Looper mLooper;
final Callback mCallback;
```

**Handler作为一个管理者，其重要做用就是创建并发送消息，最后再处理消息。**
发送消息即为把指定的Message放入到消息队列中，等到合适的时机，消息泵从消息队列中抽取消息，再分发下去，进行处理。
因此，在Handler中，有必要维护当前线程的MessageQueue和Looper的引用。对于一个线程来说，MessageQueue和Looper都是唯一的，而多个handler是可以共享同一个线程的MessageQueue和Looper的引用。
Handler里面有以下几类核心函数共同完成上面的功能。
 - 构造函数
 - 创建消息函数
 - 发送消息函数
 - 移除消息函数
 - 消息分发及处理函数

### 构造函数

构造函数主要是对成员变量进行初始化，获取线程中的Looper、MessageQueue等对象。
```java
/**
 * Default constructor associates this handler with the queue for the
 * current thread.
 *
 * If there isn't one, this handler won't be able to receive messages.
 */
public Handler() {
    if (FIND_POTENTIAL_LEAKS) {
        final Class<? extends Handler> klass = getClass();
        if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &&
                (klass.getModifiers() & Modifier.STATIC) == 0) {
            Log.w(TAG, "The following Handler class should be static or leaks might occur: " +
                klass.getCanonicalName());
        }
    }

    mLooper = Looper.myLooper();
    if (mLooper == null) {
        throw new RuntimeException(
            "Can't create handler inside thread that has not called Looper.prepare()");
    }
    mQueue = mLooper.mQueue;
    mCallback = null;
}
```

默认构造函数通过`Looper.myLooper()`函数**从当前线程中获取Looper对象**，如果Looper对象不存在，那么Handler构造就**失败了，会抛出RuntimeException**，而MessageQueue是由Looper对象创建出来的，因此，mQueue直接便能从Looper中获取。对于UI线程，在程序初始化时，实际上looper对象就已被创建出来（通过调用Looper.prepare()进行创建，并把looper对象存放到一个静态的sThreadLocal中），因此，正常情况下，当我们new出来的Handler不指明任何参数时，实际上就是会默认关联到UI线程。但是，但如果该对象是在某个线程的Run方法中被创建出来，那么它会被关联到该后台线程。“关联到该线程”的意思实际上就是，当Handler关联到UI线程，那最终发送的消息是加到了UI线程的消息队列，如果它关联到后台线程，则发送的消息加到了后台线程的消息队列。下面的这种方式，mHandler会被直接关联到指定的线程。
```java
new Thread(new Runnable() {
    @Override
    public void run() {
        Looper.prepare();
        
        Handler mHandler = new Handler() {
            public void handleMessage(Message msg) {
                // process incoming messages here
            }
        };

        Looper.loop();
    }
});
```

### 创建消息函数：obtainMessage()及其系列函数

```java
public final Message obtainMessage(int what, int arg1, int arg2, Object obj) {
    return Message.obtain(this, what, arg1, arg2, obj);
}
```

这些函数都很简单，无非是通过Message.obtain(…)方法创建消息。obtain方法已在Message类中进行相关说明。还记得obtain方法的调用过程吗？忘记了的请回过头再看看。

### 发送消息函数

发送的过程是由post其系列函数和send系列函数进行的。如下：
```java
post(Runnable)
postAtTime(Runnable, long)
postAtTime(Runnable, Object, long)
postDelayed(Runnable, long)
postAtFrontOfQueue(Runnable)
sendMessage(Message)
sendEmptyMessage(int)
sendEmptyMessageDelayed(int, long)
sendEmptyMessageAtTime(int, long)
sendMessageDelayed(Message, long)
sendMessageAtTime(Message, long)
sendMessageAtFrontOfQueue(Message)
```

先说sendXXXMessage系列函数，这些函数提供了很多可选接口，主要是可使用delayMillis。如sendMessageDelayed(Message msg, long delayMillis)，**该函数并不是说把消息延迟xxx毫秒后发送，而是延迟将消息分发下来**。即消息加入消息队列后，message会隔xxx毫秒后从消息队列中被取出来执行。
这些函数最终都是调用的sendMessageAtTime函数。
```java
public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
    boolean sent = false;
    MessageQueue queue = mQueue;
    if (queue != null) {
        msg.target = this;
        sent = queue.enqueueMessage(msg, uptimeMillis);
    } else {
        RuntimeException e = new RuntimeException(
            this + " sendMessageAtTime() called with no mQueue");
        Log.w("Looper", e.getMessage(), e);
    }
    return sent;
}
```

该方法就是调用MessageQueue的enqueueMessage(…)方法将指定的Message插入到消息队列中去，即加入Message链，并指明何时应该从消息队列中取出来执行。其中uptimeMillis就是绝对时间戳，uptimeMillis = current time + delayMillis。
postXXX系列函数来说，需要指定一个Runnable对象，在合适的时间执行。
实际上，他们最终调用的还是sendMessageAtTime函数，只不过中间多了一步，即根据Runnable创建Message对象。如：
```java
public final boolean postAtTime(Runnable r, long uptimeMillis) {
   return  sendMessageAtTime(getPostMessage(r), uptimeMillis);
}
```

其中，getPostMessage代码如下：就是通过obtain方法获取一个Message，并设置其callback。
```java
private final Message getPostMessage(Runnable r) {
    Message m = Message.obtain();
    m.callback = r;
    return m;
}
```

### 移除消息系列函数
```java
removeCallbacks(Runnable)
removeCallbacks(Runnable, Object)
removeCallbacksAndMessages(Object)
removeMessages(int)
removeMessages(int, Object)
```

这些函数的作用就是从当前消息队列中移除掉所有指定ID或指定Runnable对象的Message。如：
```java
public final void removeCallbacks(Runnable r) {
    mQueue.removeMessages(this, r, null);
}
```

**上面的所有函数完成了第一个功能，即消息的发送，并加入到消息队列中去。
但发送完后，并不是立即就能执行，当消息从消息泵中被取出来后，才行执行。因此，消息的发送和处理实际上是一个异步的过程。**

### 消息分发及处理函数

当消息从消息泵中抽取出来后，就会进行消息的分发。
消息抽取的过程需要参见Looper的核心处理函数：
```java
public static final void loop() {
    Looper me = myLooper();
    MessageQueue queue = me.mQueue;
    while (true) {
        Message msg = queue.next(); // might block
        if (msg != null) {
            if (msg.target == null) {
                // No target is a magic identifier for the quit message.
                return;
            }
            msg.target.dispatchMessage(msg);
            msg.recycle();
        }
    }
}
```

从上面可以看出，loop函数本身就是一个回路（死循环），不停的调用queue.next()函数从消息链中取出消息（如果取不到消息就会被阻塞住）。然后通过MSG的target成员变量（Handler）来调用其dispatchMessage方法将消息分发下去，然后执行消息处理函数。
```java
public void dispatchMessage(Message msg) {
    if (msg.callback != null) {
        handleCallback(msg);
    } else {
        if (mCallback != null) {
            if (mCallback.handleMessage(msg)) {
                return;
            }
        }
        handleMessage(msg);
    }
}
```

在消息的分发的过程中，其执行是有**优先级**的：
 - 如果Message中包含callback（即通过post系列函数设置的Runnable对象），那么它会被优先执行。
 - 否则，如果给当前的Handler设置了mCallback，那么它会优先执行。如果该方法返回true，分发结束，处理完毕。如果返回false，那么他还有机会执行默认的handleMessage函数。

以下是三种情况的示例代码：
```java
1. 
    Handler mHandler = new Handler();
    mHandler.post(new Runnable() {
        @Override
        public void run() {
            // TODO something
        }
    });
2. 
    Handler mHandler = new Handler(new Handler.Callback() {
        
        @Override
        public boolean handleMessage(Message msg) {
            // TODO something
            return false;
        }
    });
3. 
    protected Handler m_handler = new Handler() {
        @Override
        public void handleMessage(Message msg) {
            // TODO something
        }
    });
```

## MessageQueue源码分析

### 成员变量

我们先看一下它的成员变量
```java
Message mMessages;
private final ArrayList<IdleHandler> mIdleHandlers = new ArrayList<IdleHandler>();
private IdleHandler[] mPendingIdleHandlers;
private boolean mQuiting;
boolean mQuitAllowed = true;
// Indicates whether next() is blocked waiting in pollOnce() with a non-zero timeout.
private boolean mBlocked;
```

**mMessages：**在最初不了解Message类时，以为MessageQueue里面，存放的是一个类似于LinkedList<Message>的数据结构。当了解Message数据结构后，才知道MessageQueue里面维护的只有一个Message，因为Message本身就能构建一个Message链。MessageQueue主要功能就是维护这个Message链，如插入和删除Message链中的元素，并提供获取Message链中next  Message的方法。这里，mMessages始终指向的是消息链的第一个节点，即头节点。
**mIdleHandlers：**外部注册的回调列表（listeners）。如果当前消息队列已没有新的Message能被取出来时，线程即将被阻塞前被调用。即线程处于空闲时间时，被调用。
**mPendingIdleHandlers：**该成员变量配合上面的mIdleHandlers使用，我觉得它没有必要保存起来，完全用一个临时变量即可。
**mQuiting：**当前消息队列是否已准备退出。实际上，如果它为True，也就表明当前线程将会立马结束掉。
**mQuitAllowed：**是否允许退出消息队列。对于主线程（UI线程），该标志量为true。
**mBlocked：**标志当前消息队列是否处于阻塞状态。
下面的接口定义了线程处于空闲状态时的回调函数。由此可以看出，当你想在线程不忙的时候干点其它事情的话，这个接口就能派得上用场了。
```java
public static interface IdleHandler {
    boolean queueIdle();
}
```

以下方法用来注册和移除线程处于空闲状态时的回调函数。
```java
public final void addIdleHandler(IdleHandler handler) {
    if (handler == null) {
        throw new NullPointerException("Can't add a null IdleHandler");
    }
    synchronized (this) {
        mIdleHandlers.add(handler);
    }
}

public final void removeIdleHandler(IdleHandler handler) {
    synchronized (this) {
        mIdleHandlers.remove(handler);
    }
}
```

再来看一看MessageQueue的两个核心函数：enqueueMessage(…)与next()函数。

### enqueueMessage(Message msg, long when)函数

该函数的目的就是把指定的Message按照绝对时间插入到当前的消息队列中去。还记得该函数是在哪里被调用的吗？请回过头看看Handler的源代码：sendMessageAtTime(Message msg, long uptimeMillis)
```java
final boolean enqueueMessage(Message msg, long when) {
    // 1. 如果MSG的绝对时间戳不为0（说明已被初始化，并已加入到队列中），抛出异常
    if (msg.when != 0) {
        throw new AndroidRuntimeException(msg
                + " This message is already in use.");
    }
    
    // 2. 如果MSG的target为null，说明该消息是QUIT消息。如果此时又发现mQuitAllowed为false，则抛出异常。
    // 实际上，如果你调用主线程的Looper.quit()方法，你会发现该异常会被抛出来。因为主线程的消息循环是不允许退出的。
    if (msg.target == null && !mQuitAllowed) {
        throw new RuntimeException("Main thread not allowed to quit");
    }
    final boolean needWake;
    synchronized (this) {
        // 3. 如果消息队列正在退出，则直接返回false；否则，查看当前要加入的MSG是否是要求退出消息队列的MSG
        //（QUIT MSG：判断依据是MSG的target是否为空），如果是，则将mQuiting设置为true，然后把该消息加入到消息
        // 队列的头部，以保证下一次通过next()函数取出的消息就是QUIT消息，能快速的终止掉线程。
        if (mQuiting) {
            RuntimeException e = new RuntimeException(
                msg.target + " sending message to a Handler on a dead thread");
            Log.w("MessageQueue", e.getMessage(), e);
            return false;
        } else if (msg.target == null) {
            mQuiting = true;
        }
        
        // 4. MSG最终从消息队列中取出来的绝对时间戳。NOTE：MSG的when字段只有在这一个地方被主动设置过。
        msg.when = when;
        //Log.d("MessageQueue", "Enqueing: " + msg);

        // NOTE：以下部分就是真正如何将消息插入消息链的过程。
        Message p = mMessages;
        // 5. 如果当前的消息链为空，或者要插入的MSG为QUIT消息，或者要插入的MSG时间小于消息链的第一个消息。
        // 那么，强势插入到消息链的头部。显示，消息链的头部被改变了，变成了新添加的消息。needWake需要根据
        // mBlocked的情况考虑是否触发。
        if (p == null || when == 0 || when < p.when) {
            msg.next = p;
            mMessages = msg;
            needWake = mBlocked; // new head, might need to wake up
        } else {
            //  6. 否则，我们需要遍历该消息链，将该MSG插入到合适的位置。整个消息链是按消息被取出的绝对时间戳
            // 由小到大链接起来的。时间轴为：msg1.when <= msg2.when <= msg3.when <= msg4.when <= msg5.when……
            Message prev = null;
            while (p != null && p.when <= when) {
                prev = p;
                p = p.next;
            }
            msg.next = prev.next;
            prev.next = msg;
            needWake = false; // still waiting on head, no need to wake up
        }
    }

    // 7. 对于needWake，如果该变量为true，说明mBlocked为true，即当前线程处于阻塞状，也即nativePollOnce处于阻塞状态。但此时，
    // 我们已通过这个enqueueMessage方法已经向消息链中添加了一个消息，也就是说，此时我们需要把阻塞状态变成非阻塞状态，
    // 让next()函数能够取到MSG。怎么办？通过执行nativeWake方法，便能触发nativePollOnce函数结束等待。实际上，
    // nativePollOnce和nativeWake内部是通过管道的机制来实现阻塞和接触阻塞的。我的理解是： nativePollOnce函数从管道中
    // 读数据，如果发现管道中有数据，则立即返回，否则，一直等待。而nativeWake就是向管道中写数据，只要往管道的另一端写数据，
    // 则nativePollOnce就能立马从管道中读出数据来，从而变成非阻塞状态。（请参考：http://book.51cto.com/art/201208/353352.htm）
    if (needWake) {
        nativeWake(mPtr);
    }
    return true;
}
```

### next()函数

我们先提前一步看看Looper.loop()函数（里面省掉了若干无用代码）。
该函数就是从消息队列中取出消息，然后把这个取出来的消息扔给Looper，Looper根据消息进行处理。
```java
public static final void loop() {
    Looper me = myLooper();
    MessageQueue queue = me.mQueue;
    while (true) {
        Message msg = queue.next(); // might block
        if (msg != null) {
            if (msg.target == null) {
                // No target is a magic identifier for the quit message.
                return;
            }
            msg.target.dispatchMessage(msg);
            msg.recycle();
        }
    }
}
```

从上面的代码来看，loop函数本身就是一个回路（死循环），不停的调用`queue.next()`函数从消息链中取出消息（如果取不到消息就会被阻塞住）。然后通过MSG的target成员变量（Handler）来调用其`dispatchMessage`方法将消息分发下去，然后执行消息处理函数。如果取出来的消息的target为null，那么说明该消息是退出消息，则Looper退出，线程即将结束。
```java
final Message next() {
    // 1. 取MSG前，先初始化状态
    // a) pendingIdleHandlerCount为-1时，说明是第一次循环，在当前没有消息队列中没有MSG的情况下，需要处理注册的Handler。
    // b) nextPollTimeoutMillis 超时时间。即等待xxx毫秒后，该函数返回。如果值为0，则无须等待立即返回。如果为-1，则进入无限等待，直到有事件发生为止。
    int pendingIdleHandlerCount = -1; // -1 only during first iteration
    int nextPollTimeoutMillis = 0;

    for (;;) {
           if (nextPollTimeoutMillis != 0) {
             /** 该函数作用暂时没有研究，拷贝基注释供参考
               * Flush any Binder commands pending in the current thread to the kernel
               * driver.  This can be useful to call before performing an operation that may block for a long
               * time, to ensure that any pending object references have been released
               * in order to prevent the process from holding on to objects longer than
               * it needs to.
               */
                Binder.flushPendingCommands();
             }
           
            // 该函数提供阻塞操作。如果nextPollTimeoutMillis为0，则该函数无须等待，立即返回。如果为-1，则进入无限等待，
            // 直到有事件发生为止。在第一次时，由于nextPollTimeoutMillis被初始化为0，所以该函数会立即返回，然后从消息链的头部获取消息。
            nativePollOnce(mPtr, nextPollTimeoutMillis);
            synchronized (this) {
                // Try to retrieve the next message.  Return if found.
                // 2. 获取消息链的头节点。
                // 2.1 如果头节点不为空，需要判断头节点所代表的MSG执行的时间是否小于当前时间，如果小于，则该MSG应该被扔出去，
                //     让loop()函数执行其分发过程。否则，需要让线程再次等待(when–now)毫秒。
                // 2.2 如果头节点为空，显然，消息链中无消息可能，我们需要设置nextPollTimeoutMillis为-1，让线程阻塞住，
                //      直到有消息投递（调用enqueueMessage方法），并利用nativeWake方法解除阻塞。
                final long now = SystemClock.uptimeMillis();
                final Message msg = mMessages;
                if (msg != null) {
                    final long when = msg.when;
                    if (now >= when) {
                        mBlocked = false;
                        mMessages = msg.next;
                        msg.next = null;
                        if (Config.LOGV) Log.v("MessageQueue", "Returning message: " + msg);
                        return msg;
                    } else {
                        nextPollTimeoutMillis = (int) Math.min(when - now, Integer.MAX_VALUE);
                    }
                } else {
                    nextPollTimeoutMillis = -1;
                }
                // 3. 如果走到这一步，说明当前无消息可用，或者当前的消息还需要等待一段时间才能够分发下去。
                // 所以，在这段时间之类，我们有时间告诉listeners，当前线程空闲了，给你们一个机会干点其它事情。比如说垃圾回收。
                // If first time, then get the number of idlers to run.
                if (pendingIdleHandlerCount < 0) {
                    pendingIdleHandlerCount = mIdleHandlers.size();
                }
                if (pendingIdleHandlerCount == 0) {
                    // No idle handlers to run.  Loop and wait some more.
                    mBlocked = true;
                    continue;
                }
                if (mPendingIdleHandlers == null) {
                    mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
                }
                mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
            }
            
            // 4. 通知listeners，执行其它事情。
            // Run the idle handlers.
            // We only ever reach this code block during the first iteration.
            for (int i = 0; i < pendingIdleHandlerCount; i++) {
                final IdleHandler idler = mPendingIdleHandlers[i];
                mPendingIdleHandlers[i] = null; // release the reference to the handler
                boolean keep = false;
                try {
                    // 如果该函数返回false，表明这个函数只想执行一次，我们应该把它从列表中删除。如果返回true，则表示下次空闲时，会再次执行。
                    keep = idler.queueIdle();
                } catch (Throwable t) {
                    Log.wtf("MessageQueue", "IdleHandler threw exception", t);
                }
                if (!keep) {
                    synchronized (this) {
                        mIdleHandlers.remove(idler);
                    }
                }
            }
            // 5. 重置状态
            // pendingIdleHandlerCount重置为0，是为了避免第二次循环时，再一次通知listeners，也就说是，如果想剩余的listeners再次被调用，
            // 那么只有等到下一次调用next()函数了。
            // nextPollTimeoutMillis重置为0，是为了避免在循环执行idler.queueIdle()时，有消息投递。所以重置它后，第二次循环在执行nativePollOnce时，
            // 会立即返回，然后再走其它逻辑。此时，如果还是消息链中还是没有消息，那么将会在continue;处执行完第二次循环，进行第三次循环，然后进入无限等待状态。
            // Reset the idle handler count to 0 so we do not run them again.
            pendingIdleHandlerCount = 0;
            // While calling an idle handler, a new message could have been delivered
            // so go back and look again for a pending message without waiting.
            nextPollTimeoutMillis = 0;
        }
    }
}
```

从上面的代码来看，next()函数主要有三个作用：
 - 如果消息链中有合适的消息，直接将MSG扔出去。
 - 如果没有，在消息循环进入阻塞状态。通过nativePollOnce进行阻塞。
 - 在阻塞前，会通过外界（用户）注册的IdleHandler接口通知外界（用户），线程即将处于block状态，外界可以处理一些其它的事情。如说垃圾回收。

在看这个函数时，最开始觉得这里的逻辑有点困惑，特别是`for(;;)`和`IdleHandler`调用的过程。看上去，这里的for(;;)看上去是是个死循环，但**实际上，每调用一次next()函数，这个循环最多只会被执行三次。**
**第一次循环**，正如上面的功能1，如果消息链中有合适的消息，直接将MSG扔出去。
如果没有，则会通知各listeners，线程空闲了。执行完后，为了避免在listners执行的过程中，有消息投递，那么此时重置nextPollTimeoutMillis，然后进行**第二次循环**，由于此时nextPollTimeoutMillis为0，则nativePollOnce不会阻塞，立即返回，取MSG，如果此时消息链中还是没有MSG，则会在将会在continue处结束第二次循环，此时nextPollTimeoutMillis已被设置为-1，最终，**第三次循环**时，nativePollOnce发现nextPollTimeoutMillis为-1，则进入无限等待状态，直到有新的MSG被投递到队列中来。当有新的MSG后，由于enqueueMessage中调用了nativeWake函数，nativePollOnce会从等待中恢复回来并返回，继续执行，然后将新的MSG扔出去，for循环结束。三次循环结束。
至于`nativePollOnce`函数是如何进行阻塞的，可以参考：http://book.51cto.com/art/201208/353352.htm

### 删除函数

```java
final boolean removeMessages(Handler h, int what, Objectobject, boolean doRemove);
final void removeMessages(Handler h,Runnable r, Object object)
final voidremoveCallbacksAndMessages(Handler h, Object object)
```

其内部运作基本一样，我们只需要搞明白一个即可：
```java
final boolean removeMessages(Handler h, int what, Object object, boolean doRemove) {
    synchronized (this) {
        Message p = mMessages;
        boolean found = false;
        // 以下两个过程均是简单的链表操作。看不懂的得要复习下数据结构第二章了。
        // Remove all messages at front.
        // 如果消息链的头节点就是所要找的MSG，则把该MSG从链表中断开，并把断开的节点加入废弃消息链中。
        // 然后头节点往下移动，直到头结点不为指定MSG为止。
        while (p != null && p.target == h && p.what == what
               && (object == null || p.obj == object)) {
            if (!doRemove) return true;
            found = true;
            Message n = p.next;
            mMessages = n;
            p.recycle();
            p = n;
        }
        // Remove all messages after front.
        // 如果消息链的头节点不是所要找的MSG，则通过辅助变量n，往下查找指定MSG，找到了，则把该MSG从链表中断开，
        // 并把断开的节点加入废弃消息链中。然后辅助变量n往下移动，直到链表尾部。
        while (p != null) {
            Message n = p.next;
            if (n != null) {
                if (n.target == h && n.what == what
                    && (object == null || n.obj == object)) {
                    if (!doRemove) return true;
                    found = true;
                    Message nn = n.next;
                    n.recycle();
                    p.next = nn;
                    continue;
                }
            }
            p = n;
        }
        
        return found;
    }
}
```

该函数有三个作用：
 - 查找功能：如果doRemove为false，则该函数中是从消息链中查找是否有对应的消息。有则返回true，否则返回false
 - 删除功能：从消息链中找到所有的同ID、同target、同object的消息，并把它从当前的消息链中断开。
 - 构建Message Pool：构建废弃消息链（池）。还记得Message的recycle()方法吗？该方法就会把当前的消息废弃掉，加入到废弃消息链中，以供废品再利用。

## Looper源码分析

Looper的主要功能是管理MessageQueue，不停的从MessageQueue里面抽取消息，然后分发下去，周而复始，直到抽取到的消息是退出消息，Looper结束，线程即将退出。
Looper有以下几点需要注意：
 - 一个线程只能有一个Looper对象。
 - 一个Looper对象只能有一个MessageQueue

我们先看一下它的重要成员变量及初始化函数：
```java
final MessageQueue mQueue;
volatile boolean mRun;
Thread mThread;

private Looper() {
    mQueue = new MessageQueue();
    mRun = true;
    mThread = Thread.currentThread();
}
```

显然，在创建一个Looper时，它就会顺便创建一个消息队列，初始化mRun，并关联到当然线程。由于构造函数是私有的，那如何创建Looper对象？通过prepare()函数。
```java
public static final void prepare() {
    if (sThreadLocal.get() != null) {
        throw new RuntimeException("Only one Looper may be created per thread");
    }
    sThreadLocal.set(new Looper());
}
```

对于每个线程来说，sThreadLocal存放着Looper的唯一实例，多次调用会直接导致异常。所以，一个线程只能调用一次prepare()函数。
另外，在该类中，维护了一个主线程的Looper对象，并提供了一系列方法可以访问它：
```java
private static Looper mMainLooper = null;
public static final void prepareMainLooper()
private synchronized static void setMainLooper(Looper looper)
public synchronized static final Looper getMainLooper()
```

顺便贴出主线程Looper对象生成的源代码：
frameworks/base/core/java/android/app/ActivityThread.java
```java
public static final void main(String[] args) {
    ……
    Looper.prepareMainLooper();

    ActivityThread thread = new ActivityThread();
    thread.attach(false);

    Looper.loop();

    ……
}
```

对于Looper类来说，最重要的莫过于loop()函数，不过该函数已被重复提过几次，这里不再重复描述了。
另外，再一个重要函数是quit()，它通过向消息队列中插入一条QUIT Message来退出Looper循环，从而达到退出线程的目的。其中，Quit Message的标志就是该Message的target为null。
```java
public void quit() {
    Message msg = Message.obtain();
    // NOTE: By enqueueing directly into the message queue, the
    // message is left with a null target.  This is how we know it is
    // a quit message.
    mQueue.enqueueMessage(msg, 0);
}
```



