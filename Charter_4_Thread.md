### 线程



线程的状态

![1705242079817](https://raw.githubusercontent.com/tanxingwei/bolgImg/master/2024/01/14/20240114-222128.png)

## 启动和终止线程

### 构造线程

```java
private void init(ThreadGroup g, Runnable target, String name,
                      long stackSize, AccessControlContext acc,
                      boolean inheritThreadLocals) {
        if (name == null) {
            throw new NullPointerException("name cannot be null");
        }

        this.name = name;
    	// 父线程是当前的线程, 
        Thread parent = currentThread();
        this.group = g;
    	// 根据父线程设置是否为守护线程
        this.daemon = parent.isDaemon();
    	// 根据父线程设置优先级
        this.priority = parent.getPriority();
    	// 类加载器
        if (security == null || isCCLOverridden(parent.getClass()))
            this.contextClassLoader = parent.getContextClassLoader();
        else
            this.contextClassLoader = parent.contextClassLoader;
        this.inheritedAccessControlContext =
                acc != null ? acc : AccessController.getContext();
        this.target = target;
        setPriority(priority);
    	// 这里是判断是否使用threadLocal
        if (inheritThreadLocals && parent.inheritableThreadLocals != null)
            this.inheritableThreadLocals =
                ThreadLocal.createInheritedMap(parent.inheritableThreadLocals);
        /* 指定线程的栈大小*/
        this.stackSize = stackSize;

        /* 指定线程ID*/
        tid = nextThreadID();
    }
```

### 启动线程

通过thread的start实例方法启动线程, 下面是start方法的源码. 最主要的就是调用**start0()**的native方法

```java
/**
     使该线程开始执行；Java 虚拟机会调用该线程的运行方法。
	结果是两个线程同时运行：当前线程（调用 start 方法后返回）和另一个线程（执行其 run 方法）。
	一个线程启动超过一次是不合法的。尤其是，线程执行完毕后不得重新启动。
     */
    public synchronized void start() {
        
        if (threadStatus != 0)
            throw new IllegalThreadStateException();
        group.add(this);

        boolean started = false;
        try {
            // 执行一个native方法, 创建系统线程, 并且回调线程的Run方法
            start0();
            started = true;
        } finally {
            try {
                if (!started) {
                    group.threadStartFailed(this);
                }
            } catch (Throwable ignore) {
            }
        }
    }
```

### 理解中断

**中断相关的API**

```java
public void interrupt() {
    if (this != Thread.currentThread())
        checkAccess();
    synchronized (blockerLock) {
        // 如果一个线程正处于由java.nio.channels包中的某个通道的IO操作所阻塞的状态，那么该线程的Interruptible字段会被设置为一个非空的实例
        Interruptible b = blocker;
        if (b != null) {
            interrupt0();           // 这是一个native方法 仅仅只是设置线程的中断标志
            b.interrupt(this);
            return;
        }
    }
    interrupt0();
}
```

```java
/**
	测试此线程是否已中断。此方法不影响线程的中断状态。
	由于线程在中断时不是活动的而忽略的线程中断将通过此方法返回false来反映
*/
public boolean isInterrupted() {
    	// 这个也是一个native方法, 根据传入的布尔值决定是否要重置中断位
        return isInterrupted(false);
}
```

```java
/**
测试当前线程是否已中断。此方法清除线程的中断状态。换句话说，如果连续调用此方法两次，则第二次调用将返回false(除非当前线程再次中断，在第一次调用清除其中断状态之后，在第二次调用检查它之前)
*/
public static boolean interrupted() {
        return currentThread().isInterrupted(true);
}
```

应该如何去理解中断呢?

首先中断是一种线程间的通信方式, 由线程A发起中断信号, 线程B接收中断信号之后, 对中断信号做出响应.
在抛开InterruptedException 相关内容后, 其实也是可以把中断操作理解成一个volatile变量, A线程通过API接口interrupt()修改对应中断标志的值,,并且这个操作还需要保证可见性.当B线程读到这个volatile变量后, 根据最新的value做出某些操作(例如退出当前线程)

```java
doSomeThing();
boolean isInterrupted =   currentThread.isInterrupted();
if (isInterrupted){
   throws XxxException;
}else{
   doSomeThing();
}
```

再回到InterruptedException, 实际上也可以认为这是一种特定于线程阻塞的情况下的中断响应. 只是这个响应是由JVM替我们完成的.(让线程退出阻塞状态, 并复位响应标志位, 响应开发者).然后显式的给开发者提供一个提示.
再回到中断的设置于复位上来说:
是不是也很相似?都是响应其他线程对中断状态的修改操作, 既而对被中断线程施加预期的影响.

```java
public void response(){
    try{
		doSomeThing();
        // 阻塞操作
		waitOpration();
    }
    catch (InterruptedException e){
       // 响应异常复位
       thread.currentThread().interrupt(); 
    }
}
```

```java
public void response2){
    while(condition1 && !hread.isInterrupted()){
        doSomeThing();
        if (hread.isInterrupted()){
            sendMsg()
        }
    }
}
```

