#rxjava与juc源码分析
##一、rxjava的源码导读
```java
 public void test() {
        Observable.create(new ObservableOnSubscribe<String>() {
            @Override
            public void subscribe(ObservableEmitter<String> e) throws Exception {
                try {
                    e.onNext("123");
                    e.onComplete();
                } catch (Exception exception) {
                    e.onError(exception);
                }
            }
        }).map(new Function<String, Integer>() {
            @Override
            public Integer apply(String s) throws Exception {
                return Integer.valueOf(s);
            }
        }).subscribeOn(Schedulers.io())
                .observeOn(Schedulers.io())
                .subscribe(new DisposableObserver<Integer>() {
                    @Override
                    public void onNext(Integer value) {

                    }

                    @Override
                    public void onError(Throwable e) {

                    }

                    @Override
                    public void onComplete() {

                    }
                });
    }
```
代码执行的流程图：
![](https://github.com/skyskiper/java-juc-note/blob/master/rxjava%E7%9A%84%E6%89%A7%E8%A1%8C%E6%B5%81%E7%A8%8B.png?raw=true)

###重点说以下几种操作符：
- 1.map与flatmap
- 2.concat与merge
- 3.observeOn与subscribeOn
- 4.zip与first
- ...

##二、ThreadPoolExecutor线程池执行原理

###ThreadPoolExecutor的执行流程图：
![](https://blog.dreamtobe.cn/img/thread-pool-executor.png)

###ScheduledThreadPoolExecutor的执行流程图：
scheduleAtFixedRate：基于初始延迟（initialDelay）后固定间隔（period）进行任务调度；
scheduleWithFixedDelay：基于上次任务完成后固定的延迟时间来进行任务调度。
![](https://blog.dreamtobe.cn/img/scheduled-thread-pool-executor.png)

###Timer的执行流程：
schedule(TimerTask task, long delay, long period) ：安排指定的任务从指定的延迟后开始进行重复的固定延迟执行；
scheduleAtFixedRate(TimerTask task, long delay, long period)：安排指定的任务在指定的延迟后开始进行重复的固定速率执行。

###ScheduledThreadPoolExecutor与Timer比较：
- timer只有一个线程，任务的执行方式是串行的，如果前一个任务发生延迟会影响到后续任务的执行。其实ScheduledThreadPoolExecutor也一样，如果他的所有线程都耗时过长，也会影响后续任务的执行。
- Timer中的scheduleAtFixedRate(TimerTask task, Date firstTime,long period)具有时间追回功能：例如：制定一个昨天现在的时间开始执行任务，每2个小时执行一次，timer会立刻连续执行12次（24小时除以2）
- 建议最好使用ScheduledThreadPoolExecutor

###Executors工具类中的推荐线程池比较：
- newFixedThreadPool：线程最多n个，无界队列
- newSingleThreadExecutor：1个线程，无界队列
- newCachedThreadPool：线程数最多Integer.MAX_VALUE,无界队列（SynchronousQueue有点与众不同：在某次添加元素后必须等待其他线程取走后才能继续添加）
- newSingleThreadScheduledExecutor：1个线程，无界队列
- newScheduledThreadPool：线程数最多Integer.MAX_VALUE,无界队列
- newWorkStealingPool（ForkJoinPool留待后续）

##三、Lock与AQS原理

以ReentrantLock为例子分析独占锁原理:

```java
class Test {
        ReentrantLock lock = new ReentrantLock();

        public void testA() {
            //thread A
            try {
                lock.lock();
                for (int i = 0; i < 1000000; i++) {
                    System.out.println(i);
                }
            } finally {
                lock.unlock();
            }
        }

        public void testB() {
            //thread B
            try {
                lock.lock();
                for (int i = 0; i < 1000000; i++) {
                    System.out.println(i);
                }
            } finally {
                lock.unlock();
            }
        }

        public void testC() {
            //thread C
            try {
                lock.lock();
                for (int i = 0; i < 1000000; i++) {
                    System.out.println(i);
                }
            } finally {
                lock.unlock();
            }
        }
    }
```

假设在公平锁的情况下执行顺序为： 
##C:lock->A:lock->B:lock->C:unlock->A:lock->B:unlock

lock的流程图:
![lock的流程图](http://images2015.cnblogs.com/blog/616953/201604/616953-20160404112240734-653491814.png)

unlock的流程图:
![unlock的流程图](http://images2015.cnblogs.com/blog/616953/201604/616953-20160407192749047-2004134902.png)

在a和b线程lock时的内存图:
![](https://github.com/skyskiper/java-juc-note/blob/master/AQS%E4%B8%ADlock%E5%86%85%E5%AD%98%E5%9B%BE.png?raw=true)

在Condition的情况下：

```
  class Test {
        ReentrantLock lock = new ReentrantLock();
        Condition condition = lock.newCondition();

        public void testA() {
            //thread A
            try {
                lock.lock();
                for (int i = 0; i < 1000000; i++) {
                    System.out.println(i);
                }
                condition.await();
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                lock.unlock();
            }
        }

        public void testB() {
            //thread B
            try {
                lock.lock();
                for (int i = 0; i < 1000000; i++) {
                    System.out.println(i);
                }
                condition.await();
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                lock.unlock();
            }
        }

        public void testC() {
            //thread C
            try {
                lock.lock();
                for (int i = 0; i < 1000000; i++) {
                    System.out.println(i);
                }
            } finally {
                lock.unlock();
            }
        }

        public void testD() {
            //thread D
            try {
                lock.lock();
                for (int i = 0; i < 1000000; i++) {
                    System.out.println(i);
                }
                condition.signalAll();
            } finally {
                lock.unlock();
            }
        }
    }
```
假设在公平锁的情况下执行顺序为： 
##C:lock->A:lock->B:lock->C:unlock->A:await->B:await->D:lock->D:signall->D:unlock->A:unlock->B:unlock

D:signall会将condition queue中的a和b移动到sync queue，D:unlock会唤醒sync queue的a和b

a和b线程await时的内存图：
![](https://github.com/skyskiper/java-juc-note/blob/master/aqs%E4%B8%ADcondition%E5%86%85%E5%AD%98%E6%83%85%E5%86%B5.png?raw=true)

##ReentrantLock与Synchronized的异同：
Synchronized来说，它是java语言的关键字，是原生语法层面的互斥，需要jvm实现。ReentrantLock它是JDK 1.5之后提供的API层面的互斥锁，需要lock()和unlock()方法配合try/finally语句块来完成。

ReentrantLock优势:灵活使用，可以公平锁和不公平锁（Synchronized锁非公平锁）；可以同时绑定对个对象（Condition）

分析共享锁原理：(与独占锁的主流程类似，主要区别是共享锁会在清除头结点时唤醒下一个共享锁节点)

``` java
private void setHeadAndPropagate(Node node, long propagate) {
        Node h = head; // Record old head for check below
        setHead(node);
        /*
         * Try to signal next queued node if:
         *   Propagation was indicated by caller,
         *     or was recorded (as h.waitStatus either before
         *     or after setHead) by a previous operation
         *     (note: this uses sign-check of waitStatus because
         *      PROPAGATE status may transition to SIGNAL.)
         * and
         *   The next node is waiting in shared mode,
         *     or we don't know, because it appears null
         *
         * The conservatism in both of these checks may cause
         * unnecessary wake-ups, but only when there are multiple
         * racing acquires/releases, so most need signals now or soon
         * anyway.
         */
        if (propagate > 0 || h == null || h.waitStatus < 0 ||
            (h = head) == null || h.waitStatus < 0) {
            Node s = node.next;
            if (s == null || s.isShared())
                doReleaseShared();
        }
    }
```
###AQS的典型实现：
- ReentrantLock
- Semaphore
- ReentrantReadWriteLock
- CountDownLatch
- CyclicBarrier

