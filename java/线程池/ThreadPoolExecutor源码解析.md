# ThreadPoolExecutor源码解析

## 构造方法

```java
    public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler) {
        if (corePoolSize < 0 ||
            maximumPoolSize <= 0 ||
            maximumPoolSize < corePoolSize ||
            keepAliveTime < 0)
            throw new IllegalArgumentException();
        if (workQueue == null || threadFactory == null || handler == null)
            throw new NullPointerException();
        this.corePoolSize = corePoolSize;
        this.maximumPoolSize = maximumPoolSize;
        this.workQueue = workQueue;
        this.keepAliveTime = unit.toNanos(keepAliveTime);
        this.threadFactory = threadFactory;
        this.handler = handler;
    }
```

### 参数解析

* corePoolSize：核心线程数量，线程池中保留的线程数量，即使空闲，除非allowCoreThreadTimeOut属性为true，即设置了允许核心线程超时，会在keepAliveTime时间之后关闭，如果为false，则空闲线程会一直保持。
* maximumPoolSize：最大线程数量，线程池中线程的最大数量。
* keepAliveTime：当线程数大于内核数时，多余的空闲线程在终止之前等待的最长时间。
* unit：keepAliveTime参数的单位。
* workQueue：在执行任务之前保留任务的队列，只保存execute方法提交的任务。
* threadFactory：创建新线程时使用的线程工厂。
* handler：当达到线程容量和workQueue队列容量的时候，对新提交的任务执行的拒绝策略。

## 核心方法

### execute(Runnable command)

```java
    public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
        int c = ctl.get();
        // 如果工作线程数量小于核心线程数,则尝试添加到核心任务
        if (workerCountOf(c) < corePoolSize) {
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }
        //	如果当前线程池为运行状态，则尝试添加到工作队列，添加成功后进入if代码块
        if (isRunning(c) && workQueue.offer(command)) {
        	//再次获取线程池状态
            int recheck = ctl.get();
            // 如果不是运行状态,尝试从任务队列中移除该方法，移除成功后执行拒绝策略
            if (! isRunning(recheck) && remove(command))
                reject(command);
            else if (workerCountOf(recheck) == 0)
                //如果workerCount为0，则添加一个新worker从workQueue获取任务执行
                addWorker(null, false);
        }
        // 如果添加到工作队列失败而且大于最大线程数量，则执行拒绝策略
        else if (!addWorker(command, false))
            reject(command);
    }
```
大概总结一下这个方法：

1. 如果线程数量小于corePoolSize，则会新建线程执行任务。
2. 如果线程数量大于等于corePoolSize且workQueue不满，且当前线程池为RUNNING状态，则添加到workQueue任务缓存队列中，等待空闲线程取出任务执行；如果当前工作线程数为0，则新建一个空闲线程从队列中取出任务执行。
3. 若步骤2workQueue添加worker失败，则尝试直接线程执行，如果线程数量达到maximumPoolSize，则执行拒绝策略。

### addWorker(Runnable firstTask, boolean core)

```java
private boolean addWorker(Runnable firstTask, boolean core) {
        retry：//该for循环目的是使worker数量+1
        for (;;) {
            int c = ctl.get();
            // 线程运行状态
            int rs = runStateOf(c);
            //判断线程池状态是否允许添加worker，不符合条件直接返回false
            if (rs >= SHUTDOWN &&
                ! (rs == SHUTDOWN &&
                   firstTask == null &&
                   ! workQueue.isEmpty()))
                return false;
            //如果到这里说明满足以下条件(上面条件取反)：
            //rs<SHUTDOWN||(rs == SHUTDOWN &&firstTask == null &&! workQueue.isEmpty())
            //上面条件的含义是：当前线程池状态为RUNNING或者状态为SHUTDOWN且在工作队列不为空的情况下添加了一个空的task
            for (;;) {
            	//当前正在工作的线程数量
                int wc = workerCountOf(c);
                // 如果当前工作线程数量达到或者超过最大容量,则返回false
                if (wc >= CAPACITY ||
                    wc >= (core ? corePoolSize ： maximumPoolSize))
                    return false;
               	//如果CAS增加线程数量成功,则跳出循环，准备新建线程执行任务
                if (compareAndIncrementWorkerCount(c))
                    break retry;
                c = ctl.get();  // Re-read ctl
                // CAS失败,说明有其他线程改变了workerCount,重新执行retry循环
                if (runStateOf(c) != rs)
                    continue retry;
            }
        }
		//到这里workerCount已经+1成功,接下来就要创建然后启动线程执行任务
        boolean workerStarted = false;
        boolean workerAdded = false;
        Worker w = null;
        try {
            w = new Worker(firstTask);
            final Thread t = w.thread;
            if (t != null) {
                final ReentrantLock mainLock = this.mainLock;
                // 加锁
                mainLock.lock();
                try {
                    // Recheck while holding lock.
                    // Back out on ThreadFactory failure or if
                    // shut down before lock acquired.
                    int rs = runStateOf(ctl.get());
                    // 如果当前线程池状态为RUNNING或者状态SHUTDOWN而且firstTask为null,添加该worker到works集合中
                    if (rs < SHUTDOWN ||
                        (rs == SHUTDOWN && firstTask == null)) {
                        if (t.isAlive()) // precheck that t is startable
                            throw new IllegalThreadStateException();
                        workers.add(w);
                        int s = workers.size();
                        //更新largestPoolSize
                        if (s > largestPoolSize)
                            largestPoolSize = s;
                        workerAdded = true;
                    }
                } finally {
                    mainLock.unlock();
                }
                // 添加成功后启动线程
                if (workerAdded) {
                    t.start();
                    workerStarted = true;
                }
            }
        } finally {
            if (! workerStarted)
                addWorkerFailed(w);
        }
        return workerStarted;
    }
```
该方法主要任务是**创建并启动一个新线程**，主要分为两个阶段：
1. 判断当前线程池是否能够增加workCount,符合条件循环CAS+1，不符合则返回false。
下面这段代码用于判断是否能够添加新的任务：
```java
 if (rs >= SHUTDOWN &&
                ! (rs == SHUTDOWN &&
                   firstTask == null &&
                   ! workQueue.isEmpty()))
                return false;
```
我们解析一下：
* 条件1： **rs>=SHUTDOWN** 即当前线程的状态为SHUTDOWN,STOP,TIDYING,TERMINATED。
* 条件2： **! (rs == SHUTDOWN && firstTask == null &&! workQueue.isEmpty())**等效为 **rs!=SHUTDOWN | | firstTask!=null | | workQueue.isEmpty()**。
  此时我们将 条件1与条件2合并一下即**rs>=SHUTDOWN&&(rs!=SHUTDOWN | | firstTask!=null | | workQueue.isEmpty()**。

最终等效为以下三种情况：
* **rs>SHUTDOWN**：当前线程池状态为STOP,TIDYING或者TERMINATED,这两张状态为当前线程池即将终止或者已经终止的状态,自然不允许添加新worker
* **rs == SHUTDOWN | | firstTask != null**：当前线程为SHUTDOWN状态，而且firstTask不为null，显然SHUTDOWN状态规定不允许添加新的任务
* **rs == SHUTDOWN | | firstTask == null && workQueue.isEmpty()**：当前线程为SHUTDOWN状态，而且当前工作队列为空,添加的firstTask为null，首先这里我们要知道**添加一个空的firstTask的目的就是新建一个线程从workQueue中获取任务去执行**，而此时workQueue为空，因此此时添加一个空的firstTask便没有意义。

2. 创建并启动线程。

   该过程主要分为以下步骤：

* 创建worker，分两种情况：
  * 当firstTask为null时，该worker会从workQueue中获取任务执行。
  * 当firstTask不为null时，该worker会新建线程执行任务。
  
* 添加worker，需要满足线程池状态为RUNNING或者状态为SHUTDOWN时firstTask为null。

* 执行任务，如果worker添加成功，则会启动worker中的线程执行任务。如果添加失败，则会执行addWorkerFailed()方法。
  
### addWorkerFailed(Worker w)

addWorkerFailed方法源代码如下：
```java
    private void addWorkerFailed(Worker w) {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            if (w != null)
                workers.remove(w);
            decrementWorkerCount();
            tryTerminate();
        } finally {
            mainLock.unlock();
        }
    }
```
该方法主要用于添加worker失败之后的回滚，具体包括以下三步：
  1. 如果workers中包含worker则移除worker。
  2. 在添加worker失败时回滚workCount，将workCount减1。
  3. 尝试终止线程池

### tryTerminate() 

tryTerminate方法源码如下：

```java
final void tryTerminate() {
        for (;;) {
            int c = ctl.get();
            // 如果当前线程池状态为RUNNING，TIDYING,TERMINATED或者为状态为SHUTDOWN且workQueue不为空时，返回。不符合结束条件
            if (isRunning(c) ||
                runStateAtLeast(c, TIDYING) ||
                (runStateOf(c) == SHUTDOWN && ! workQueue.isEmpty()))
                return;
            // 到这里，线程池状态为RUNNING或者状态为SHUTDOWN且workQueue为空。
            // 如果workerCount不为0，则中断1个空闲线程，然后返回。
            if (workerCountOf(c) != 0) { // Eligible to terminate
                interruptIdleWorkers(ONLY_ONE);
                return;
            }

            final ReentrantLock mainLock = this.mainLock;
            mainLock.lock();
            try {
                //尝试使用CAS设置线程池状态为TIDYING,workCount为0
                if (ctl.compareAndSet(c, ctlOf(TIDYING, 0))) {
                    try {
                        //终止线程
                        terminated();
                    } finally {
                        // 设置线程池状态为TERMINATED,workCount为0
                        ctl.set(ctlOf(TERMINATED, 0));
                        termination.signalAll();
                    }
                    return;
                }
            } finally {
                mainLock.unlock();
            }
            // CAS失败则重试
        }
    }
```

该方法主要用于尝试在符合终止条件的情况下终止线程池。

根据源码可以得出，线程池变为TERMINATED状态的前提条件有以下几个：
* 条件1：workCount为0，即当前工作线程数为0
* 条件2：线程池状态为RUNNING或者状态为SHUTDOWN且workQueue为空。

### getTask()

源码如下

```java
private Runnable getTask() {
        boolean timedOut = false; // Did the last poll() time out?

        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            // 检查是否有必要往下执行
            // 如果当前线程为STOP,TIDYING,TERMINATED状态，或者为SHUTDOWN状态但是workQueue为空
            // 则workerCount-1,返回null
            if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
                decrementWorkerCount();
                return null;
            }
			// 获取线程数量
            int wc = workerCountOf(c);
			// 是否允许超时
            boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;
            if ((wc > maximumPoolSize || (timed && timedOut))
                && (wc > 1 || workQueue.isEmpty())) {
                if (compareAndDecrementWorkerCount(c))
                    return null;
                continue;
            }
			// 从缓存队列中获取任务，根据timed调用不同的方法           
            try {
                Runnable r = timed ?
                    workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                    workQueue.take();
                if (r != null)
                    return r;
                timedOut = true;
            } catch (InterruptedException retry) {
                timedOut = false;
            }
        }
    }
```

该方法主要作用就是从任务缓存队列中获取任务。具体分析见注释。