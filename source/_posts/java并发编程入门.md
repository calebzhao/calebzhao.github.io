---
title: java并发编程入门
date: 2019-12-29 16:06:54
tags: 
- java
- 并发
- juc
categories: java基础
---


# 1、入门介绍
## 1.1、实现线程的2种方式
```
package chapter2;

/**
 * @author calebzhao<9 3 9 3 4 7 5 0 7 @ qq.com>
 * 2019/6/29 14:05
 */
public class MyThreadDemo1 {
    public static void main(String[] args) {
        new Thread1().start();
        
        new Thread(new Thread2()).start();
    }
}

class Thread1 extends Thread{
    @Override
    public void run() {
        System.out.println(Thread.currentThread().getName());
    }
}

class Thread2 implements Runnable{
    @Override
    public void run() {
        System.out.println(Thread.currentThread().getName());
    }
}
```

## 1.2、线程的状态（生命周期）
### 1.2.1、线程的5种状态
1.&nbsp;**新建(NEW)**：新创建了一个线程对象。

2.&nbsp;**可运行(RUNNABLE)**：线程对象创建后，其他线程(比如main线程）调用了该对象的start()方法。该状态的线程位于可运行线程池中，等待被线程调度选中，获取cpu 的使用权 。

3.&nbsp;**运行(RUNNING)**：可运行状态(runnable)的线程获得了cpu 时间片（timeslice） ，执行程序代码。

4.&nbsp;**阻塞(BLOCKED)**：阻塞状态是指线程因为某种原因放弃了cpu 使用权，也即让出了cpu timeslice，暂时停止运行。直到线程进入可运行(runnable)状态，才有机会再次获得cpu timeslice 转到运行(running)状态。阻塞的情况分三种：&nbsp;

```
(一). 等待阻塞：运行(running)的线程执行o.wait()方法，JVM会把该线程放入等待队列(waitting queue)中。
(二). 同步阻塞：运行(running)的线程在获取对象的同步锁时，若该同步锁被别的线程占用，则JVM会把该线程放入锁池(lock pool)中。
(三). 其他阻塞：运行(running)的线程执行Thread.sleep(long ms)或t.join()方法，或者发出了I/O请求时，JVM会把该线程置为阻塞状态。当sleep()状态超时、join()等待线程终止或者超时、或者I/O处理完毕时，线程重新转入可运行(runnable)状态。
```
5.&nbsp;**死亡(DEAD)**：线程run()、main() 方法执行结束，或者因异常退出了run()方法，则该线程结束生命周期。死亡的线程不可再次复生。

### 1.2.2、线程运行状态图
![](https://oscimg.oschina.net/oscnet/up-eb5d2db11964516af51c019671105b82e47.png)

**1). 初始状态**

1. 实现Runnable接口和继承Thread可以得到一个线程类，new一个实例出来，线程就进入了初始状态

**2). 可运行状态**

1. 可运行状态只是说你资格运行，调度程序没有挑选到你，你就永远是可运行状态。
2. 调用线程的start()方法，此线程进入可运行状态。
3. 当前线程sleep()方法结束，其他线程join()结束，等待用户输入完毕，某个线程拿到对象锁，这些线程也将进入可运行状态。
4. 当前线程时间片用完了，调用当前线程的yield()方法，当前线程进入可运行状态。
5. 锁池里的线程拿到对象锁后，进入可运行状态。

**3). 运行状态**

1. 线程调度程序从可运行池中选择一个线程作为当前线程时线程所处的状态。这也是线程进入运行状态的唯一一种方式。

**4). 死亡状态**

1. 当线程的run()方法完成时，或者主线程的main()方法完成时，我们就认为它死去。这个线程对象也许是活的，但是，它已经不是一个单独执行的线程。线程一旦死亡，就不能复生。
2. 在一个死去的线程上调用start()方法，会抛出java.lang.IllegalThreadStateException异常。

**5). 阻塞状态**

1. 当前线程T调用Thread.sleep()方法，当前线程进入阻塞状态。
2. 运行在当前线程里的其它线程t2调用join()方法，当前线程进入阻塞状态。
3. 等待用户输入的时候，当前线程进入阻塞状态。

**6). 等待队列(是Object里的方法，但影响了线程)**

1. 调用obj的wait(), notify()方法前，必须获得obj锁，也就是必须写在synchronized(obj) 代码段内。
2. 与等待队列相关的步骤和图
* 线程1获取对象A的锁，正在使用对象A。
* 线程1调用对象A的wait()方法。
* 线程1释放对象A的锁，并马上进入等待队列。
* 锁池里面的对象争抢对象A的锁。
* 线程5获得对象A的锁，进入synchronized块，使用对象A。
* 线程5调用对象A的notifyAll()方法，唤醒所有线程，所有线程进入锁池。||||| 线程5调用对象A的notify()方法，唤醒一个线程，不知道会唤醒谁，被唤醒的那个线程进入锁池。
* notifyAll()方法所在synchronized结束，线程5释放对象A的锁。
* 锁池里面的线程争抢对象锁，但线程1什么时候能抢到就不知道了。||||| 原本锁池+第6步被唤醒的线程一起争抢对象锁。

![](https://oscimg.oschina.net/oscnet/up-69a8d075ff73baa13a29f1d937c2e346c06.png)

**7). 锁池状态**

1. 当前线程想调用对象A的同步方法时，发现对象A的锁被别的线程占有，此时当前线程进入锁池状态。简言之，锁池里面放的都是想争夺对象锁的线程。
2. 当一个线程1被另外一个线程2唤醒时，1线程进入锁池状态，去争夺对象锁。
3. 锁池是在同步的环境下才有的概念，一个对象对应一个锁池。
## **1.3. **sleep、yield、join、wait**的比较**
1. Thread.sleep(long millis)，一定是当前线程调用此方法，当前线程**进入阻塞**，但**不释放对象锁**，millis后线程**自动苏醒进入可运行状态**。作用：给其它线程执行机会的最佳方式。
2. Thread.yield()，一定是当前线程调用此方法，当前线程放弃获取的cpu时间片，由运行状态变会可运行状态，让OS再次选择线程。作用：让相同优先级的线程轮流执行，但并不保证一定会轮流执行。实际中无法保证yield()达到让步目的，因为让步的线程还有可能被线程调度程序再次选中。Thread.yield()不会导致阻塞。
3. t.join()/t.join(long millis)，当前线程里调用其它线程1的join方法，当前线程阻塞，但不释放对象锁，直到线程1执行完毕或者millis时间到，当前线程进入可运行状态。
4. obj.wait()，当前线程调用对象的wait()方法，当前线程释放对象锁，进入等待队列。依靠notify()/notifyAll()唤醒或者wait(long timeout)timeout时间到自动唤醒。
5. obj.notify()唤醒在此对象监视器上等待的单个线程，选择是任意性的。notifyAll()唤醒在此对象监视器上等待的所有线程。
## 1.4、Thread.sleep(long time)
Thread.sleep(long time)方法的作用是让当前正在执行的线程休眠(让出CPU时间片)指定的毫秒数。

需要注意的是：

* 调用sleep()方法时，如果当前线程持有锁**不会导致当前线程释放锁**。
* sleep不释放锁 线程是进入阻塞状态还是就绪状态？

 答案是**进入阻塞状态**，确切的说**Thread在Java的TIMED_WAITING状态**（但这个状态其实并没那么重要，可以认为是java的内部细节，用户不用太操心）。往下一层，在不同OS上底层的sleep的实现细节不太一样。但是大体上就是挂起当前的线程，然后**设置一个信号或者时钟中断到时候唤醒**。sleep后的的Thread在被唤醒前是不会消耗任何CPU的（确切的说，大部分OS都会这么实现，除非某个OS的实现偷懒了）。这点上，wait对当前线程的效果差不多是一样的，也会暂停调度，等着notify或者一个超时的时间。期间CPU也不会被消耗。

# 2、多线程实现银行叫号排队
## 2.1 版本1
```
package chapter2.version1;
/**
 * @author calebzhao<9 3 9 3 4 7 5 0 7 @ qq.com>
 * 2019/6/29 8:58
 */
public class Bank {
    public static void main(String[] args) {
        TicketWindow ticketWindow = new TicketWindow("1号窗口");
        TicketWindow ticketWindow2 = new TicketWindow("2号窗口");
        TicketWindow ticketWindow3 = new TicketWindow("3号窗口");
        ticketWindow.start();
        ticketWindow2.start();
        ticketWindow3.start();
    }
}
```


```
package chapter2.version1;
/**
 * @author calebzhao<9 3 9 3 4 7 5 0 7 @ qq.com>
 * 2019/6/29 8:58
 */
public class TicketWindow extends Thread {
    private static int MAX = 50;

    private static int current = 1;
    private String name;
    public TicketWindow(String name){
        this.name = name;
    }


    @Override
    public void run() {

        while (current < MAX){
            System.out.println("窗口：" + name +" 当前叫号:" + current);
            current++;

            try {
                Thread.sleep(100);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}
```

此版本中注意TicketWindow 类的成员变量current 为static, 表示无论实例化这个类多少次，都共享同一份变量，因为这个变量是在类加载的时候就已经创建好的。为了让多个线程消耗同一个current 所以才定义为static的，不然每个线程的current都是各自的，与其他线程不相关。
    
以上代码把业务与线程紧密掺杂在一起，为了让多个线程访问同一份current把他定义为static明显是不合适的。

## 2.2 版本2
```
package chapter2.version2;

/**
 * @author calebzhao<9 3 9 3 4 7 5 0 7 @ qq.com>
 * 2019/6/29 8:58
 */
public class Bank {

    public static void main(String[] args) {
        // 有线程安全问题， 多个线程同时访问到current变量
        TicketWindow ticketWindow = new TicketWindow();

        Thread windowThread1 = new Thread(ticketWindow, "1号窗口");
        Thread windowThread2 = new Thread(ticketWindow, "2号窗口");
        Thread windowThread3 = new Thread(ticketWindow, "3号窗口");

        windowThread1.start();
        windowThread2.start();
        windowThread3.start();
    }
```
}


```
package chapter2.version2;

/**
 * @author calebzhao<9 3 9 3 4 7 5 0 7 @ qq.com>
 * 2019/6/29 9:02
 */
public class TicketWindow implements Runnable {

    private int MAX = 50;

    private int current = 1;


    @Override
    public void run() {

        while (current < MAX){
            System.out.println("窗口：" + Thread.currentThread().getName() +" 当前叫号:" + current);
            current++;

            try {
                Thread.sleep(100);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
```
}

这里TicketWindow 并不是继承Thread, 这样启动线程多次线程可以共享同一份TicketWindow 实例， 把业务逻辑与线程启动拆分清晰明了。
注：以上2个版本均存在线程安全性问题，这是由于current变量在run()方法可能被多个线程同时访问， 可能多个线程同时执行到

```
System.out.println("窗口：" + Thread.currentThread().getName() +" 当前叫号:" + current);
```
这行代码，导致不同的线程打印出了一样的叫号值，比如运行以上程序会输出如下结果：
![](https://oscimg.oschina.net/oscnet/up-8fcf7e22ed3d11bc74d176d1e8564bb2760.png)

# 3、守护线程
要点：**守护线程会随着父线程的退出而退出**， 守护线程适宜一些**辅助性**的工作，而**不能**把核心工作的线程设置为守护线程。

```
package chapter2;

/**
 * @author calebzhao<9 3 9 3 4 7 5 0 7 @ qq.com>
 * 2019/6/29 14:07
 */
public class DeemonThreadDemo {

    public static void main(String[] args) {
        Thread thread = new Thread(){
            @Override
            public void run() {
                while (true){
                    System.out.println("hello");
                }
            }
        };

        // 设置为守护线程
        thread.setDaemon(true);

        thread.start();

        System.out.println("主线程退出");

    }
}
```

![](https://oscimg.oschina.net/oscnet/up-fc85a47ea7391a6b16833ee05dd676bdbb6.png)

从代码中可以看到thread 中的代码有while(true){  }语句， 在我们的映像中while(true)是死循环应该永远不退出，永远打印hello, 但是从输出结果可以看到运行一段时间后就没有再打印了，说明父线程结束执行了，子线程也随机退出了。

# 4、Thread.join()
Thread.join() 作用：调用join()方法的线程会等到他自己执行结束才会继续往后执行

Thread.join(time) 作用：调用join(time)方法的线程会等到他自己执行指定时间后就会继续往后执行，

## 4.1 无限等待
```
package chapter2;

/**
 * @author calebzhao<9 3 9 3 4 7 5 0 7 @ qq.com>
 * 2019/6/29 14:32
 */
public class ThreadJoinDemo {

    public static void main(String[] args) throws InterruptedException {
        Thread thread1 = new Thread(() -> {
            for (int i =0; i < 500; i++){
                System.out.println(Thread.currentThread().getName() + ":" + i);
            }
        });

        Thread thread2 = new Thread(() -> {
            for (int i =0; i < 500; i++){
                System.out.println(Thread.currentThread().getName() + ":" + i);
            }
        });

        thread1.start();
        thread2.start();
        thread1.join();
        thread2.join();

        System.out.println("所有线程已结束执行");

    }
}
```

![](https://oscimg.oschina.net/oscnet/up-6c25e6adc13c34ee3c2bcf73460b942021b.png)![](https://oscimg.oschina.net/oscnet/up-9659c2379ef2e59b476b021b69dc6cf36ff.png)

thread1 和thread2交互执行， 而main线程的打印结果**一定是在2个线程全部执行完后**才打印结果。

## 4.2 等待指定时间
```
package chapter2;

/**
 * @author calebzhao<9 3 9 3 4 7 5 0 7 @ qq.com>
 * 2019/6/29 14:32
 */
public class ThreadJoinDemo2 {

    public static void main(String[] args) throws InterruptedException {
        Thread thread1 = new Thread(() -> {
            System.out.println(Thread.currentThread().getName() +"开始执行");
            try {
                Thread.sleep(3000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println(Thread.currentThread().getName() +"结束执行");
        });

        thread1.start();
        
        // 虽然join了， 但是只会等待指定的时间，而不会无限等待
        thread1.join(1000);

        System.out.println("主线程等到1000毫秒就执行到了，而无需等到thread1执行完毕");

    }
}
```

![](https://oscimg.oschina.net/oscnet/up-f5f42565688249377afbecc3c3e345ebaeb.png)

# 5、中断线程的几种方式
## 5.1、介绍
```
package chapter2;

import org.junit.Test;

/**
 * @author calebzhao<9 3 9 3 4 7 5 0 7 @ qq.com>
 * 2019/6/29 14:53
 */
public class ThreadInterrupt {

    public static void main(String[] args) {
        Thread thread = new Thread(() ->{
            while (true){
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    System.out.println("线程被打断");
                    e.printStackTrace();

                    // 中断后让其退出while循环， 这样来控制线程的退出
                    break;
                }
            }
        });

        thread.start();

        System.out.println(thread.isInterrupted());
        // 打断thread线程的sleep, 让thread1不再继续sleep
        thread.interrupt();
        System.out.println(thread.isInterrupted());

        System.out.println("主线程退出");
    }

    @Test
    public void test2(){
        Object MONITOR = new Object();

        Thread thread = new Thread(() ->{
            while (true){
                synchronized (MONITOR){
                    try {
                        MONITOR.wait(100);
                    } catch (InterruptedException e) {
                        System.out.println("线程被打断" + Thread.interrupted());
                        e.printStackTrace();
                    }
                }
            }
        });

        thread.start();

        System.out.println(thread.isInterrupted());
        thread.interrupt();
        System.out.println(thread.isInterrupted());

        System.out.println("主线程退出");
    }

    @Test
    public void test3(){
        Thread thread1 = new Thread(() -> {
            System.out.println(Thread.currentThread().getName() +"开始执行");
            while (true){

            }
        });

        Thread main = Thread.currentThread();
        Thread thread2 = new Thread(() -> {
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }

            // 目前是main线程在等待thread1线程，所以需要打断main线程，才能让main线程不再继续等待而是继续往后执行
            main.interrupt();
            System.out.println("interrupt");
        });

        thread1.start();
        thread2.start();

        try {
            // main线程会等待thread1线程执行完毕才能继续往后执行
            thread1.join();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        System.out.println("主线程执行到了");

    }
}
```
## 5.2、通过whilte(running) { }变量控制
```
package chapter2;

import org.junit.Test;

/**
 * @author calebzhao<9 3 9 3 4 7 5 0 7 @ qq.com>
 * 2019/6/29 15:34
 */
public class ThreadCloseDemo {

    class Worker implements Runnable{
        private volatile boolean running = true;

        @Override
        public void run() {
            // 这里有问题， 假如doSomething()非常耗时， 下一次执行while (running)没有机会得到执行，没法立即中断
            while (running){
                // 假如执行doSomething()方法
            }

            System.out.println("退出死循环");
        }

        public void shutdown(){
            this.running = false;
        }
    }

    @Test
    public void close1(){
        Worker worker = new Worker();
        Thread thread = new Thread(worker);
        thread.start();

        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        worker.shutdown();

        System.out.println("主线程执行完毕");
    }
}
```



test1()的执行结果：

![](https://oscimg.oschina.net/oscnet/up-4cf535f5c40645ca6c10871df66b369df36.png)

## 5.3、通过Thread.interrupt()控制
```
package chapter2;

import org.junit.Test;

/**
 * @author calebzhao<9 3 9 3 4 7 5 0 7 @ qq.com>
 * 2019/6/29 15:34
 */
public class ThreadCloseDemo2 {

    class Worker2 implements Runnable{
        private volatile boolean running = true;

        @Override
        public void run() {
            while (true){
                if (Thread.currentThread().isInterrupted()){
                    break;
                }

                // 这里有问题， 假如doSomething()非常耗时， 上面的代码根本没有机会得到执行，没法立即中断
                // doSomething()


            }

            System.out.println("退出死循环");
        }
    }

    @Test
    public void close2(){
        Worker2 worker = new Worker2();
        Thread thread = new Thread(worker);
        thread.start();

        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        thread.interrupt();

        System.out.println("主线程执行完毕");
    }
}
```

![](https://oscimg.oschina.net/oscnet/up-cd01ac61f21bcd2cfd77f33fe4be6257a37.png)

## 5.4、暴力终止
```
package chapter2;

import org.junit.Test;

/**
 * @author calebzhao<9 3 9 3 4 7 5 0 7 @ qq.com>
 * 2019/6/29 16:06
 */
public class ThreadCloseDemo2 {

    class ThreadService{

        private Thread executeThread;

        private boolean running = true;

        public void execute(Runnable task){

            executeThread = new Thread(() -> {
                Thread workerThread = new Thread(task);
                workerThread.setDaemon(true);
                workerThread.start();

                //
                try {
                    // 让executeThread等待workerThread结束执行才往后执行
                    // 假如workerThread比较耗时，由于守护线程是依附于父线程的，假如这里不join，
                    // 那么可能会产生workerThread还没有真正执行完，executeThread线程就执行完了，导致workerThread也跟着退出了
                    workerThread.join();
                } catch (InterruptedException e) {
                    // e.printStackTrace();
                }

                //executeThread线程被打断等待 或者 workerThread正常执行结束
                this.running = false;

            });

            executeThread.start();
        }

        public void shutdown(long timeout){
            long beginTime = System.currentTimeMillis();

            // workerThread还在执行
            while (running){
                // 判断是否等待超时
                if (System.currentTimeMillis() - beginTime >= timeout){
                    // 等待超时了， 中断executeThread的等待会使this.running = false;执行到，
                    // executeThread执行完毕导致workerThread守护线程跟着退出
                    executeThread.interrupt();

                    //退出while循环
                    break;
                }

                try {
                    Thread.sleep(1);
                }
                catch (InterruptedException e) {
                    // 说明main线程被打断了
                    e.printStackTrace();

                    break;
                }
            }
        }

        public boolean isRunning(){
            return this.running;
        }

    }

    /**
     * 必须等待3秒后才结束执行
     */
    @Test
    public void test1() {
        ThreadService threadService = new ThreadService();
        threadService.execute(() -> {
            // 这是一个非常非常耗时的操作
           while (true){

           }
        });

        //让线程2秒后立即中断执行
        long begin = System.currentTimeMillis();
        threadService.shutdown(3000);
        long end = System.currentTimeMillis();

        System.out.println("耗时： " + (end - begin));

        System.out.println("主线程执行结果");
    }

    /**
     * 跟随threadService的执行而结束，无需等待3秒（1秒后就可以执行主线程后续的代码）
     */
    @Test
    public void test2() {
        ThreadService threadService = new ThreadService();
        threadService.execute(() -> {
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        });

        //让线程2秒后立即中断执行
        long begin = System.currentTimeMillis();
        threadService.shutdown(3000);
        long end = System.currentTimeMillis();

        System.out.println("耗时： " + (end - begin));

        System.out.println("主线程执行结果");
    }
}
```




## 5.5、interrupt()、interrupted()、isInterrupted()区别与原理

### 5.5.1、结论
* interrupt()方法：用于中断线程的，调用该方法的线程的状态将被置为"中断"状态。**注意：线程中断仅仅是设置线程的中断状态位，不会停止线程。需要用户自己去监视线程的状态并做处理**。支持线程中断的方法（也就是线程中断后会抛出InterruptedException的方法，比如Thread.sleep，以及Object.wait等方法）就是在监视线程的中断状态，一旦线程的中断状态被置为“中断状态”，就会抛出中断异常,并将线程的中断状态为设置为false。
* interrupted()：返回线程是否处于已中断状态并清除中断状态
* isInterrupted()：返回线程是否处于已中断状态

### 5.5.2、原理
以下内容来源于 [https://my.oschina.net/itblog/blog/787024](https://my.oschina.net/itblog/blog/787024)

```
public class Interrupt {
	public static void main(String[] args) throws Exception {
		Thread t = new Thread(new Worker());
		t.start();
		
		Thread.sleep(200);
		t.interrupt();
		
		System.out.println("Main thread stopped.");
	}
	
	public static class Worker implements Runnable {
		public void run() {
			System.out.println("Worker started.");
			
			try {
				Thread.sleep(500);
			} catch (InterruptedException e) {
				System.out.println("Worker IsInterrupted: " + 
						Thread.currentThread().isInterrupted());
			}
			
			System.out.println("Worker stopped.");
		}
	}
}
```
内容很简单：主线程main启动了一个子线程Worker，然后让worker睡500ms，而main睡200ms，之后main调用worker线程的interrupt方法去中断worker，worker被中断后打印中断的状态。
下面是执行结果：

```
Worker started.
Main thread stopped.
Worker IsInterrupted: false
Worker stopped.
```
**Worker明明已经被中断，而isInterrupted()方法竟然返回了false，为什么呢？**

在stackoverflow上搜索了一圈之后，发现有网友提到：可以查看抛出InterruptedException方法的JavaDoc（或源代码），于是我查看了Thread.sleep方法的文档，doc中是这样描述这个InterruptedException异常的：

>InterruptedException - if any thread has interrupted the current thread.&nbsp;The&nbsp;interrupted status&nbsp;of the current thread is cleared when this exception is thrown.

注意到后面这句“**当抛出这个异常的时候，中断状态已被清除**”。所以isInterrupted()方法应该返回false。可是有的时候，我们需要isInterrupted这个方法返回true，怎么办呢？这里就要先说说interrupt, interrupted和isInterrupted的区别了：

interrupt方法是用于中断线程的，调用该方法的线程的状态将被置为"中断"状态。注意：**线程中断仅仅是设置线程的中断状态位，不会停止线程**。需要用户自己去监视线程的状态为并做处理。支持线程中断的方法（也就是线程中断后会抛出InterruptedException的方法，比如这里的sleep，以及Object.wait等方法）就是**在监视线程的中断状态，一旦线程的中断状态被置为“中断状态”，就会抛出中断异常**。这个观点可以通过[这篇文章](http://www.ibm.com/developerworks/library/j-jtp05236/)证实：

再来看看interrupted方法的实现：

```
public static boolean interrupted() {
    return currentThread().isInterrupted(true);
}
```
和isInterrupted的实现：
```
public boolean isInterrupted() {
    return isInterrupted(false);
}
```
这两个方法一个是static的，一个不是，但实际上都是在调用同一个方法，只是interrupted方法传入的参数为true，而iInterrupted传入的参数为false。那么这个参数到底是什么意思呢？来看下这个isInterrupted(boolean)方法的实现：
```
/**
 * Tests if some Thread has been interrupted.  The interrupted state
 * is reset or not based on the value of ClearInterrupted that is
 * passed.
 */
private native boolean isInterrupted(boolean ClearInterrupted);
```
这是一个native方法，看不到源码没有关系，参数名字ClearInterrupted已经清楚的表达了该参数的作用----是否清除中断状态。方法的注释也清晰的表达了“中断状态将会根据传入的ClearInterrupted参数值确定是否重置”。所以，**静态方法interrupted将会清除中断状态**（传入的参数ClearInterrupted为true），而**实例方法isInterrupted则不会**（传入的参数ClearInterrupted为false）。
回到刚刚的问题：很明显，如果要isInterrupted这个方法返回true，通过**在调用isInterrupted方法之前再次调用interrupt()方法**来恢复这个中断的状态即可：

```
public class Interrupt  {
	public static void main(String[] args) throws Exception {
		Thread t = new Thread(new Worker());
		t.start();
		
		Thread.sleep(200);
		t.interrupt();
		
		System.out.println("Main thread stopped.");
	}
	
	public static class Worker implements Runnable {
		public void run() {
			System.out.println("Worker started.");
			
			try {
				Thread.sleep(500);
			} catch (InterruptedException e) {
				Thread curr = Thread.currentThread();
				//再次调用interrupt方法中断自己，将中断状态设置为“中断”
				curr.interrupt();
				System.out.println("Worker IsInterrupted: " + curr.isInterrupted());
				System.out.println("Worker IsInterrupted: " + curr.isInterrupted());
				System.out.println("Static Call: " + Thread.interrupted());//clear status
				System.out.println("---------After Interrupt Status Cleared----------");
				System.out.println("Static Call: " + Thread.interrupted());
				System.out.println("Worker IsInterrupted: " + curr.isInterrupted());
				System.out.println("Worker IsInterrupted: " + curr.isInterrupted());
			}
			
			System.out.println("Worker stopped.");
		}
	}
}
```
执行结果：
```
Worker started.
Main thread stopped.
Worker IsInterrupted: true
Worker IsInterrupted: true
Static Call: true
---------After Interrupt Status Cleared----------
Static Call: false
Worker IsInterrupted: false
Worker IsInterrupted: false
Worker stopped.
```
从执行结果也可以看到，前两次调用isInterrupted方法都返回true，说明isInterrupted方法不会改变线程的中断状态，而接下来调用静态的interrupted()方法，第一次返回了true，表示线程被中断，第二次则返回了false，因为第一次调用的时候已经清除了中断状态。最后两次调用isInterrupted()方法就肯定返回false了。

那么，**在什么场景下，我们需要在catch块里面中断线程（重置中断状态）呢？**

答案是：**如果不能抛出InterruptedException**（就像这里的Thread.sleep语句放在了Runnable的run方法中，这个方法不允许抛出任何受检查的异常），但又**想告诉上层调用者这里发生了中断**的时候，就只能在catch里面重置中断状态了。

```
public class TaskRunner implements Runnable {
    private BlockingQueue<task> queue;
 
    public TaskRunner(BlockingQueue<task> queue) { 
        this.queue = queue; 
    }
 
    public void run() { 
        try {
             while (true) {
                 Task task = queue.take(10, TimeUnit.SECONDS);
                 task.execute();
             }
         } catch (InterruptedException e) { 
             // Restore the interrupted status
             Thread.currentThread().interrupt();
         }
    }
}
```
那么问题来了：**为什么要在抛出InterruptedException的时候清除掉中断状态呢？**

这个问题没有找到官方的解释，估计只有Java设计者们才能回答了。但[这里](http://stackoverflow.com/questions/2523721/why-do-interruptedexceptions-clear-a-threads-interrupted-status)的解释似乎比较合理：**一个中断应该只被处理一次**（你catch了这个InterruptedException，说明你能处理这个异常，你不希望上层调用者看到这个中断）。

参考地址：

[https://my.oschina.net/itblog/blog/787024](https://my.oschina.net/itblog/blog/787024)

[http://stackoverflow.com/questions/7142665/why-does-thread-isinterrupted-always-return-false](http://stackoverflow.com/questions/7142665/why-does-thread-isinterrupted-always-return-false)

[http://stackoverflow.com/questions/2523721/why-do-interruptedexceptions-clear-a-threads-interrupted-status](http://stackoverflow.com/questions/2523721/why-do-interruptedexceptions-clear-a-threads-interrupted-status)

[http://www.ibm.com/developerworks/library/j-jtp05236/](http://www.ibm.com/developerworks/library/j-jtp05236/)

[http://blog.csdn.net/z69183787/article/details/25076033](http://blog.csdn.net/z69183787/article/details/25076033)


# 6、synchronized
## 6.1、示例代码
```
package chapter2;

/**
 * @author calebzhao<9 3 9 3 4 7 5 0 7 @ qq.com>
 * 2019/6/29 17:11
 */
public class SynchronizedDemo {

    private static final Object LOCK = new Object();

    public static void main(String[] args) {
        Runnable runnable = () -> {
            synchronized (LOCK){
                System.out.println(Thread.currentThread().getName() +"正在执行");
                try {
                    Thread.sleep(60_000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }

                System.out.println(Thread.currentThread().getName() +"执行结束");
            }
        };
        Thread thread1 = new Thread(runnable);
        Thread thread2 = new Thread(runnable);
        Thread thread3 = new Thread(runnable);

        thread1.start();
        thread2.start();
        thread3.start();

        System.out.println("主线程执行结束");
    }
}
```


##  6.2、通过jconsole查看
通过在cmd输入**jconsole**命令显示如下界面：

![](https://oscimg.oschina.net/oscnet/up-18779b62effbfca55ebe3ebb94a738bed72.png)

选中本地进程中刚刚运行的java程序， 进入主界面

![](https://oscimg.oschina.net/oscnet/up-d0e71af143f7185b5e4a400f33848378075.png)

可以看到 Thread-0 、Thread-1、Thread-2运行的三个线程，点击对应线程右边可以看到“拥有者 Thread-0”

## 6.3、通过jps、jstack查看
![](https://oscimg.oschina.net/oscnet/up-24a22e5653805abe5649f5e7f33cc191478.png)

启动程序后：

第1步：输入jps命令， 查看当前正在运行的java进程

第2步: 输入jstack [pid]  可以看到有三个线程 Thread-1、Thread-2均处于BLOCKED(on object nonitor)状态、而Thread-0处于TIMED_WATING(sleepint)状态（因为Thread.sleep()的作用）

## 6.4、通过javap -c [xxxx.class]命令查看汇编指令
![](https://oscimg.oschina.net/oscnet/up-1bcee94d08fc8fb733825e53c804b8ffb57.png)

## 6.5 this锁
含义：指在方法上面加上synchronized 关键字

```
package chapter2;
/**
 * @author calebzhao<9 3 9 3 4 7 5 0 7 @ qq.com>
 * 2019/6/29 17:56
 */
public class ThisLockDemo {

    public static void main(String[] args) {
        ThisLock thisLock = new ThisLock();

        Thread thread1 = new Thread(() -> {
           thisLock.m1();
        });

        Thread thread2 = new Thread(() -> {
            thisLock.m2();
        });

        thread1.start();
        thread2.start();

        System.out.println("主线程结束执行");
    }
}

class ThisLock{

    public synchronized void m1(){
        System.out.println(Thread.currentThread().getName() +" m1开始执行");

        try {
            Thread.sleep(10_000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        System.out.println(Thread.currentThread().getName() +" m1结束执行");
    }

    public synchronized void m2(){
        System.out.println(Thread.currentThread().getName() +" m2开始执行");

        try {
            Thread.sleep(10_000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        System.out.println(Thread.currentThread().getName() +" m2结束执行");
    }
}
```


运行结果：

![](https://oscimg.oschina.net/oscnet/up-b34888626a3ec62de366651f808033aa2a3.png)

可以看到Thread-1 m2的执行必须要等待Thread-0 m1执行完毕后才能执行， 说明同一个类在方法上面加上synchronized关键字所使用的的锁是this锁

## 6.6 class锁
介绍：在static方法上加synchronized关键所使用的的锁是class锁

```
package chapter2;

/**
 * @author calebzhao<9 3 9 3 4 7 5 0 7 @ qq.com>
 * 2019/6/29 18:29
 */
public class ClassLockDemo {

    public static void main(String[] args) {
        ClassLock classLock1 = new ClassLock();
        ClassLock classLock2 = new ClassLock();

        Thread thread1 = new Thread(() -> {
            classLock1.m1();
        });

        Thread thread2 = new Thread(() -> {
            classLock2.m2();
        });

        thread1.start();
        thread2.start();

        System.out.println("主线程结束执行");
    }
}


class ClassLock{

    public static synchronized void m1(){
        System.out.println(Thread.currentThread().getName() +" m1开始执行");

        try {
            Thread.sleep(10_000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        System.out.println(Thread.currentThread().getName() +" m1结束执行");
    }

    public static synchronized void m2(){
        System.out.println(Thread.currentThread().getName() +" m2开始执行");

        try {
            Thread.sleep(10_000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        System.out.println(Thread.currentThread().getName() +" m2结束执行");
    }
}
```

![](https://oscimg.oschina.net/oscnet/up-ec0c3b735ecc412e329b50e288744a1c017.png)


可以看到Thread-1 m2的执行必须要等待Thread-0 m1执行完毕后才能执行， 说明同一个类的多个实例在static方法上面加上synchronized关键字所使用的的锁是class锁


## 6.7 死锁
```
package chapter2;

/**
 * @author calebzhao<9 3 9 3 4 7 5 0 7 @ qq.com>
 * 2019/6/29 18:33
 */
public class DieLockDemo {



    public static void main(String[] args) {
        DieLock dieLock = new DieLock();

        Thread thread1 = new Thread(() ->{
            dieLock.doSomething1();
        });

        Thread thread2 = new Thread(() ->{
            dieLock.doSomething2();
        });

        thread1.start();
        thread2.start();

        System.out.println("主线程执行完毕");
    }
}

class DieLock{

    private final Object LOCK1 = new Object();
    private final Object LOCK2 = new Object();

    public void doSomething1(){
        synchronized (LOCK1){
            try {
                // 代表一个耗时操作
                Thread.sleep(100);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }

            System.out.println(Thread.currentThread().getName() + "获取到LOCK1， 等待LOCK2");
            synchronized (LOCK2){
                System.out.println("doSomething1");
            }
        }
    }

    public void doSomething2(){
        synchronized (LOCK2){
            try {
                // 代表一个耗时操作
                Thread.sleep(100);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }

            System.out.println(Thread.currentThread().getName() + "获取到LOCK2, 等待LOCK1");

            synchronized (LOCK1){
                System.out.println("doSomething2");
            }
        }
    }
}
```

![](https://oscimg.oschina.net/oscnet/up-d626b06c234cd44ea6235cd0646a5807953.png)

## 6.8、synchronzied锁重入
关键字synchronized拥有锁重入的功能，也就是在使用synchronized时，当一个线程获得一个对象锁，再次请求该对象锁时是可以获得该对象的锁的。表现在一个synchronized方法 / 块的内部调用本类的其他synchronized方法 / 块时，是永远可以得到锁的。

```
package chapter2;

/**
 * @author calebzhao<9 3 9 3 4 7 5 0 7 @ qq.com>
 * 2019/9/1 15:43
 */
public class SynchronizedReentrantDemo {

    public synchronized void methodA(){
        System.out.println("methodA invoked ");

        this.methodB();
    }

    public synchronized void methodB(){
        System.out.println("methodB invoked ");
    }

    public static void main(String[] args) {
        SynchronizedReentrantDemo demo = new SynchronizedReentrantDemo();

        demo.methodA();
    }
}
```

![](https://oscimg.oschina.net/oscnet/up-e67bb7a8f62ad5aee7f7f5a13a115e7bebb.png)

可以看到在synchronized方法methodsA中调用被synchronized修饰的methodB()也是可以的，即使methodA()还没有释放this锁，这证明了synchronzied锁是可以重入的


## 6.9、synchronized的原理
Java中synchronized的锁实现包括：偏向锁、轻量级锁、重量级锁（等待时间长）

1个对象的实例包括3部分：对象头、实例变量、填充数据

![](https://oscimg.oschina.net/oscnet/up-0e9fc60ad797ac4622198fb58e8edf03958.png)

* 对象头：实现synchronized的锁对象的基础
* 实例变量: 存放类的属性数据信息，包括父类的属性信息，如果是数组的实例部分还包括数组的长度，这部分内存按4字节对齐。
* 填充数据：由于虚拟机要求对象起始地址必须是8字节的整数倍。填充数据不是必须存在的，仅仅是为了字节对齐，这点了解即可。

synchronized使用的锁对象是存储在Java对象头里的，jvm中采用2个字来存储对象头(如果对象是数组则会分配3个字，多出来的1个字记录的是数组长度)，其主要结构是由Mark Word 和 Class Metadata Address 组成，其结构说明如下表：



其中Mark Word在默认情况下存储着对象的HashCode、分代年龄、锁标记位等以下是32位JVM的Mark Word默认存储结构

![](https://oscimg.oschina.net/oscnet/up-9b2a8700065fefb92d4cd536579f385a77d.png)

由于对象头的信息是与对象自身定义的数据没有关系的额外存储成本，因此考虑到JVM的空间效率，Mark Word 被设计成为一个非固定的数据结构，以便存储更多有效的数据，它会根据对象本身的状态复用自己的存储空间，如32位JVM下，除了上述列出的Mark Word默认存储结构外，还有如下可能变化的结构：

![](https://oscimg.oschina.net/oscnet/up-dc79047288acf631946f6f6141c544d4726.png)

其中轻量级锁和偏向锁是Java 6 对 synchronized 锁进行优化后新增加的，稍后我们会简要分析。这里我们主要分析一下重量级锁也就是通常说synchronized的对象锁，锁标识位为10，其中指针指向的是monitor对象（也称为管程或监视器锁）的起始地址。每个对象都存在着一个 monitor 与之关联，对象与其 monitor 之间的关系有存在多种实现方式，如monitor可以与对象一起创建销毁或当线程试图获取对象锁时自动生成，但当一个 monitor 被某个线程持有后，它便处于锁定状态。在Java虚拟机(HotSpot)中，monitor是由ObjectMonitor实现的，其主要数据结构如下（位于HotSpot虚拟机源码ObjectMonitor.hpp文件，C++实现的）

```
ObjectMonitor() {
    _header       = NULL;
    _count        = 0; //记录个数
    _waiters      = 0,
    _recursions   = 0;
    _object       = NULL;
    _owner        = NULL;
    _WaitSet      = NULL; //处于wait状态的线程，会被加入到_WaitSet
    _WaitSetLock  = 0 ;
    _Responsible  = NULL ;
    _succ         = NULL ;
    _cxq          = NULL ;
    FreeNext      = NULL ;
    _EntryList    = NULL ; //处于等待锁block状态的线程，会被加入到该列表
    _SpinFreq     = 0 ;
    _SpinClock    = 0 ;
    OwnerIsThread = 0 ;
  }
```
ObjectMonitor中有两个队列，_WaitSet 和 _EntryList，用来保存ObjectWaiter对象列表( 每个等待锁的线程都会被封装成ObjectWaiter对象)，_owner指向持有ObjectMonitor对象的线程，当多个线程同时访问一段同步代码时，首先会进入 _EntryList 集合，当线程获取到对象的monitor 后进入 _Owner 区域并把monitor中的owner变量设置为当前线程同时monitor中的计数器count加1，若线程调用 wait() 方法，将释放当前持有的monitor，owner变量恢复为null，count自减1，同时该线程进入 WaitSe t集合中等待被唤醒。若当前线程执行完毕也将释放monitor(锁)并复位变量的值，以便其他线程进入获取monitor(锁)。如下图所示
![图片](https://uploader.shimo.im/f/keLX2CdvjYkxTCJp.png!thumbnail)

由此看来，monitor对象存在于每个Java对象的对象头中(存储的指针的指向)，synchronized锁便是通过这种方式获取锁的，也是为什么Java中任意对象可以作为锁的原因，同时也是notify/notifyAll/wait等方法存在于顶级对象Object中的原因(关于这点稍后还会进行分析)，ok~，有了上述知识基础后，下面我们将进一步分析synchronized在字节码层面的具体语义实现。

## 6.10、synchronized底层实现原理
为了分析底层原理，我们编写1个最简单的类，代码如下：

```
package chapter2;
/**
 * @author calebzhao
 * @date 2019/11/30 18:41
 */
public class SynchronizedDemo2 {
    public synchronized static void test1(){
        System.out.println("test1");
    }
    public synchronized void test2(){
        System.out.println("test2");
    }
    public void test3(){
        synchronized (this){
            System.out.println("test4");
        }
    }
}
```

### 6.10.1、synchronized方法底层原理：
方法级的同步是隐式，即无需通过字节码指令来控制的，它实现在方法调用和返回操作之中。JVM可以从方法常量池中的方法表结构(method_info Structure) 中的 ACC_SYNCHRONIZED 访问标志区分一个方法是否同步方法。当方法调用时，调用指令将会 检查方法的 ACC_SYNCHRONIZED 访问标志是否被设置，如果设置了，执行线程将先持有monitor（虚拟机规范中用的是管程一词）， 然后再执行方法，最后再方法完成(无论是正常完成还是非正常完成)时释放monitor。在方法执行期间，执行线程持有了monitor，其他任何线程都无法再获得同一个monitor。如果一个同步方法执行期间抛 出了异常，并且在方法内部无法处理此异常，那这个同步方法所持有的monitor将在异常抛到同步方法之外时自动释放。下面我们看看字节码层面如何实现：

通过执行javap -v SynchroizedDemo2.class会输出如下字节码指令：

test1()方法对应的字节码指令：

![](https://oscimg.oschina.net/oscnet/up-1519102cf94401fa287e2890d1b24487052.png)

 

test2()方法对应的字节码指令：

![](https://oscimg.oschina.net/oscnet/up-839f858397a43f5dda0433571b02f1f487e.png)


从字节码中可以看出，synchronized修饰的方法并没有monitorenter指令和monitorexit指令，取得代之的确实是ACC_SYNCHRONIZED标识，该标识指明了该方法是一个同步方法，JVM通过该ACC_SYNCHRONIZED访问标志来辨别一个方法是否声明为同步方法，从而执行相应的同步调用。这便是synchronized锁在同步代码块和同步方法上实现的基本原理。同时我们还必须注意到的是在Java早期版本中，synchronized属于重量级锁，效率低下，因为监视器锁（monitor）是依赖于底层的操作系统的Mutex Lock来实现的，而操作系统实现线程之间的切换时需要从用户态转换到核心态，这个状态之间的转换需要相对比较长的时间，时间成本相对较高，这也是为什么早期的synchronized效率低的原因。庆幸的是在Java 6之后Java官方对从JVM层面对synchronized较大优化，所以现在的synchronized锁效率也优化得很不错了，Java 6之后，为了减少获得锁和释放锁所带来的性能消耗，引入了轻量级锁和偏向锁，接下来我们将简单了解一下Java官方在JVM层面对synchronized锁的优化。

### 6.10.2、synchonized代码块底层实现原理：
test3()方法对应的字节码指令：

![](https://oscimg.oschina.net/oscnet/up-0598217e1334e81de8a67ca10186041caf7.png)

**JSR133的解释：**

> synchronized 语句需要一个对象的引用；随后会尝试在该对象的管程上执行 lock 动 作，如果 lock 动作未能成功完成，将一直等待。当 lock 动作执行成功，就会运行synchronized 语句块中的代码。一旦语句块中的代码执行结束，不管是正常还是异 常结束，都会在之前执行 lock 动作的那个管程上自动执行一个 unlock 动作。
>
>>synchronized 方法在调用时会自动执行一个 lock 动作。在 lock 动作成功完成之前， 都不会执行方法体。如果是实例方法，锁的是调用该方法的实例（即，方法体执行 期间的 this）相关联的管程。如果是静态方法，锁的是定义该方法的类所对应的 Class 对象。一旦方法体执行结束，不管是正常还是异常结束，都会在之前执行 lock 动作的那个管程上自动执行一个 unlock 动作。

从字节码中可知同步语句块的实现使用的是monitorenter 和 monitorexit 指令，其中monitorenter指令指向同步代码块的开始位置，monitorexit指令则指明同步代码块的结束位置，当执行monitorenter指令时，当前线程将试图获取 objectref(即对象锁) 所对应的 monitor 的持有权，当 objectref 的 monitor 的进入计数器为 0，那线程可以成功取得 monitor，并将计数器值设置为 1，取锁成功。如果当前线程已经拥有 objectref 的 monitor 的持有权，那它可以重入这个 monitor (关于重入性稍后会分析)，重入时计数器的值也会加 1。倘若其他线程已经拥有 objectref 的 monitor 的所有权，那当前线程将被阻塞，直到正在执行线程执行完毕，即monitorexit指令被执行，执行线程将释放 monitor(锁)并设置计数器值为0 ，其他线程将有机会持有 monitor 。值得注意的是编译器将会确保无论方法通过何种方式完成，方法中调用过的每条 monitorenter 指令都有执行其对应 monitorexit 指令，而无论这个方法是正常结束还是异常结束。为了保证在方法异常完成时 monitorenter 和 monitorexit 指令依然可以正确配对执行，编译器会自动产生一个异常处理器，这个异常处理器声明可处理所有的异常，它的目的就是用来执行 monitorexit 指令。从字节码中也可以看出多了一个monitorexit指令，它就是异常结束时被执行的释放monitor 的指令。

## 6.11、Java虚拟机对synchronized的优化
锁的状态总共有四种，无锁状态、偏向锁、轻量级锁和重量级锁。随着锁的竞争，锁可以从偏向锁升级到轻量级锁，再升级的重量级锁，但是锁的升级是单向的，也就是说只能从低到高升级，不会出现锁的降级，关于重量级锁，前面我们已详细分析过，下面我们将介绍偏向锁和轻量级锁以及JVM的其他优化手段，这里并不打算深入到每个锁的实现和转换过程更多地是阐述Java虚拟机所提供的每个锁的核心优化思想，毕竟涉及到具体过程比较繁琐，如需了解详细过程可以查阅《深入理解Java虚拟机原理》。

### 6.11.1、偏向锁
偏向锁是Java 6之后加入的新锁，它是一种针对加锁操作的优化手段，经过研究发现，在大多数情况下，锁不仅不存在多线程竞争，而且总是由同一线程多次获得，因此为了减少同一线程获取锁(会涉及到一些CAS操作,耗时)的代价而引入偏向锁。偏向锁的核心思想是，如果一个线程获得了锁，那么锁就进入偏向模式，此时Mark Word 的结构也变为偏向锁结构，当这个线程再次请求锁时，无需再做任何同步操作，即获取锁的过程，这样就省去了大量有关锁申请的操作，从而也就提供程序的性能。所以，对于没有锁竞争的场合，偏向锁有很好的优化效果，毕竟极有可能连续多次是同一个线程申请相同的锁。但是对于锁竞争比较激烈的场合，偏向锁就失效了，因为这样场合极有可能每次申请锁的线程都是不相同的，因此这种场合下不应该使用偏向锁，否则会得不偿失，需要注意的是，偏向锁失败后，并不会立即膨胀为重量级锁，而是先升级为轻量级锁。下面我们接着了解轻量级锁。

### 6.11.2、轻量级锁
倘若偏向锁失败，虚拟机并不会立即升级为重量级锁，它还会尝试使用一种称为轻量级锁的优化手段(1.6之后加入的)，此时Mark Word 的结构也变为轻量级锁的结构。轻量级锁能够提升程序性能的依据是“对绝大部分的锁，在整个同步周期内都不存在竞争”，注意这是经验数据。需要了解的是，轻量级锁所适应的场景是线程交替执行同步块的场合，如果存在同一时间访问同一锁的场合，就会导致轻量级锁膨胀为重量级锁。

### 6.11.3、自旋锁
轻量级锁失败后，虚拟机为了避免线程真实地在操作系统层面挂起，还会进行一项称为自旋锁的优化手段。这是基于在大多数情况下，线程持有锁的时间都不会太长，如果直接挂起操作系统层面的线程可能会得不偿失，毕竟操作系统实现线程之间的切换时需要从用户态转换到核心态，这个状态之间的转换需要相对比较长的时间，时间成本相对较高，因此自旋锁会假设在不久将来，当前的线程可以获得锁，因此虚拟机会让当前想要获取锁的线程做几个空循环(这也是称为自旋的原因)，一般不会太久，可能是50个循环或100循环，在经过若干次循环后，如果得到锁，就顺利进入临界区。如果还不能获得锁，那就会将线程在操作系统层面挂起，这就是自旋锁的优化方式，这种方式确实也是可以提升效率的。最后没办法也就只能升级为重量级锁了。

### 6.11.4、锁消除
消除锁是虚拟机另外一种锁的优化，这种优化更彻底，Java虚拟机在JIT编译时(可以简单理解为当某段代码即将第一次被执行时进行编译，又称即时编译)，通过对运行上下文的扫描，去除不可能存在共享资源竞争的锁，通过这种方式消除没有必要的锁，可以节省毫无意义的请求锁时间，如下StringBuffer的append是一个同步方法，但是在add方法中的StringBuffer属于一个局部变量，并且不会被其他线程所使用，因此StringBuffer不可能存在共享资源竞争的情景，JVM会自动将其锁消除。

```java
public class StringBufferRemoveSync {

	public void add(String str1, String str2) {
		//StringBuffer是线程安全,由于sb只会在append方法中使用,不可能被其他线程引用
		//因此sb属于不可能共享的资源,JVM会自动消除内部的锁
		StringBuffer sb = new StringBuffer();
		sb.append(str1).append(str2);
	}

	public static void main(String[] args) {
		StringBufferRemoveSync rmsync = new StringBufferRemoveSync();
		for (int i = 0; i < 10000000; i++) {
			rmsync.add("abc", "123");
		}
	}
}
```
# 7、wait、notify方法
## 7.1、含义（需仔细研读）
1. wait()、notify()是Object类的方法， 调用这2个方法前，执行线程必须已经获得改对象的对象锁，即**只能在同步方法或同步代码块中调用wait() 或者notify()方法**，如果调用这2个方法时没有获得对象锁将会抛出IllegalMonitorStateException异常
1. 调用wait()方法后，当前线程立即释放已经获得的锁，并且**将当前线程置入“预执行队列(WaitSet)中”， 并且在wait()所在的代码处停止执行**，**必须直到收到notify()方法的通知**或者被中断执行当前线程才能被唤醒继续往wait()方法后面的代码执行
1. **notitify()方法用来通知那些等待获取该对象锁的线程， 如果有多个线程等待，则由线程规划器随机挑选出一个处于wait状态的线程B，对其发出Notify通知，使B退出等待队列，处于就绪状态，被重新唤醒的线程B会尝试获取临界区的对象锁，被唤醒线程B在真正获取到锁后就会继续执行wait()后面的代码。需要说明的是，在执行notify()方法后，并不会使当前线程A马上释放对象锁，处于wait状态的线程B也不能马上获取对象锁，要等到执行notify()方法的线程A将程序执行完，也就是退出synchronized代码块后，当前线程A才会释放对象锁，但是释放之后并不代表线程B就一定会获取到对象锁，只是说此时A、B都有机会竞争获取到对象锁**
2. 如果notify()方法执行时，此时并没有任何线程处于wait状态，那么执行该方法相当于无效操作
3. notify()与notifyAll()的区别是：notify()方法每次调用时都只是从所有处于wait状态的线程中**随机选择一个**线程进入就绪状态，而notifyAll()则是使所有处于wait状态的线程**全部退出**等待队列，**全部进入**就绪状态，此处唤醒不等于所有线程都获得该对象的monitor，此时优先级最高的那个线程优先执行(获得对象锁)，但也有可能是随机执行(获得对象锁)，这要取决于jvm实现

## 7.2、生产者消费者模式
### 7.2.1、 版本1产生假死(全部进入等待wait状态)
```java
package chapter2;

import java.util.stream.Stream;

/**
 * @author calebzhao<9 3 9 3 4 7 5 0 7 @ qq.com>
 * 2019/6/30 7:44
 */
public class ProducerAndConsumerVersion1 {

    private final Object LOCK = new Object();

    private boolean isProduced;

    private int num = 0;

    public static void main(String[] args) {
        ProducerAndConsumerVersion1 version1 = new ProducerAndConsumerVersion1();
        Stream.of("P1", "P2", "P3", "P4").forEach((item) ->{
            new Thread(() ->{
                while (true) {
                    version1.produce();
                }

            }, item).start();
        });
        Stream.of("C1").forEach(item ->{
            new Thread(() ->{
                while (true) {
                    version1.consumer();
                }

            }, item).start();
        });


        System.out.println("主线程执行结束");
    }

    public void produce(){
        synchronized (LOCK){
            if (isProduced){
                try {
                    System.out.println("[" + Thread.currentThread().getName() +"] produce wait");
                    LOCK.wait();

                    System.out.println("[" + Thread.currentThread().getName() +"] produce wait after");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
            else {
                num++;
                System.out.println("[" + Thread.currentThread().getName() + "] P ==>"  + num);
                isProduced=true;
                LOCK.notify();
                System.out.println("[" + Thread.currentThread().getName() + "] notify after");
            }
        }
    }

    public void consumer(){
        synchronized (LOCK){

            if (isProduced){
               
                System.out.println("[" + Thread.currentThread().getName() + "] C ==>" + num);
                isProduced = false;

                LOCK.notify();

                System.out.println("[" + Thread.currentThread().getName() + "] notify after");
            }
            else {
                try {
                    System.out.println("[" + Thread.currentThread().getName() +"] consumer wait");
                    LOCK.wait();

                    System.out.println("[" + Thread.currentThread().getName() +"] consumer wait after");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }
    }

}
```

![](https://oscimg.oschina.net/oscnet/up-61ffa8e9956cd9e5d35d8924f1e114cb606.png)

以上代码执行永远不会结束，所有线程最终都变为wait状态（没有发生死锁）

### 7.2.2、多消费者、多生产者正确版本
```
package chapter2;

import java.util.stream.Stream;

/**
 * @author calebzhao<9 3 9 3 4 7 5 0 7 @ qq.com>
 * 2019/6/30 7:44
 */
public class ProducerAndConsumerVersion2 {

    private final Object LOCK = new Object();

    private boolean isProduced;

    private int num = 0;

    public static void main(String[] args) {
        ProducerAndConsumerVersion2 version1 = new ProducerAndConsumerVersion2();
        Stream.of("P1", "P2", "P3", "P4").forEach((item) ->{
            new Thread(() ->{
                while (true) {
                    version1.produce();
                }

            }, item).start();
        });
        Stream.of("C1", "C2", "C3", "C4").forEach(item ->{
            new Thread(() ->{
                while (true) {
                    version1.consumer();
                }

            }, item).start();
        });


        System.out.println("主线程执行结束");
    }

    public void produce(){
        synchronized (LOCK){
            while (isProduced){
                try {
                    System.out.println("[" + Thread.currentThread().getName() +"] produce wait");
                    LOCK.wait();

                    System.out.println("[" + Thread.currentThread().getName() +"] produce wait after");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }

            num++;
            System.out.println("[" + Thread.currentThread().getName() + "] P ==>"  + num);
            isProduced=true;
            LOCK.notifyAll();
            System.out.println("[" + Thread.currentThread().getName() + "] notify after");
        }
    }

    public void consumer(){
        synchronized (LOCK){
            while (!isProduced){
                try {
                    System.out.println("[" + Thread.currentThread().getName() +"] consumer wait");
                    LOCK.wait();

                    System.out.println("[" + Thread.currentThread().getName() +"] consumer wait after");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
            
            System.out.println("[" + Thread.currentThread().getName() + "] C ==>" + num);
            isProduced = false;

            LOCK.notifyAll();

            System.out.println("[" + Thread.currentThread().getName() + "] notify after");

        }
    }

}
```

### 7.2.3、多生产者、多消费者， 阻塞, 错误版本，浪费执行机会
```
package chapter2;

import java.util.LinkedList;
import java.util.stream.Stream;

/**
 * @author calebzhao<9 3 9 3 4 7 5 0 7 @ qq.com>
 * 2019/6/30 7:44
 */
public class ProducerAndConsumerVersion3 {

    private final Object LOCK = new Object();

    private boolean isProduced;

    private int num = 0;

    public static void main(String[] args) {
        Container container = new Container(10);

        Producer producer = new Producer(container);
        Consumer consumer = new Consumer(container);
        Stream.of("P1", "P2", "P3", "P4").forEach((item) ->{
            new Thread(() ->{
                while (true) {
                    producer.produce();
                }

            }, item).start();
        });

        Stream.of("C1", "C2", "C3", "C4").forEach(item ->{
            new Thread(() ->{
                while (true) {
                    consumer.consume();
                }

            }, item).start();
        });


        System.out.println("主线程执行结束");
    }
}

class Producer{

    private Container container;
    private int i = 0;

    public Producer(Container container){
        this.container = container;
    }

    public void produce(){
        while (true){
            synchronized (container){
                if (container.isOverflow()){
                    try {
                        container.wait();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
                else{
                    container.push(++i);
                    System.out.println("[" + Thread.currentThread().getName() +"] produce " + i);

                    container.notifyAll();
                }
            }

            try {
                Thread.sleep(100);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}

class Consumer{
    private Container container;

    public Consumer(Container container){
        this.container = container;
    }

    public void consume(){
        while (true){
            synchronized (container){
                if (container.isEmpty()){
                    try {
                        container.wait();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
                // 这里其实是有问题的， 假设A、B线程都执行到container.wait();处于wait状态， 此时生产者执行了container.notifyAll();
              // A线程获得了对象锁， 从container.wait();处往后执行，但是由于这里用的是if..else结构，
              // 所以会导致A线程刚被唤醒获得了对象锁，又什么都不做，马上又释放了对象锁，假设A线程的CPU时间片刚好用完又让B获得了对象锁，
            // 可能出现后续都一直是B获得CPU实现片获得对象锁，而A明明之前获得过一次对象锁却啥事也不干，白白浪费了一次执行机会

                else {
                    Object value = container.pop();
                    System.out.println("[" + Thread.currentThread().getName() +"]  consume " + value);

                    container.notifyAll();
                }
            }
        }
    }
}

class Container{

    private LinkedList<object> storage;

    private int capticy;


    public Container(int capticy){
        this.storage = new LinkedList<>();

        this.capticy = capticy;
    }

    public void push(Object obj){
        this.storage.addLast(obj);
    }

    public Object pop(){
        return this.storage.removeFirst();
    }

    public int size(){
        return this.storage.size();
    }

    public boolean isOverflow(){
        return size() >= capticy;
    }

    public boolean isEmpty(){
        return this.storage.isEmpty();
    }
}
```

![](https://oscimg.oschina.net/oscnet/up-7042a92981b617b191f6ce2711936b2b0a8.png)

### 7.2.4、完全正确版本
```
package 生产者消费者模式;

import java.util.LinkedList;
import java.util.stream.Stream;

/**
 * @author calebzhao<9 3 9 3 4 7 5 0 7 @ qq.com>
 * 2019/6/30 7:44
 */
@SuppressWarnings("ALL")
public class ProducerAndConsumerVersion4 {

    private final Object LOCK = new Object();

    private boolean isProduced;

    private int num = 0;

    public static void main(String[] args) {
        Container2 container = new Container2(10);

        Producer2 producer = new Producer2(container);
        Consumer2 consumer = new Consumer2(container);
        Stream.of("P1", "P2", "P3", "P4").forEach((item) -> {
            new Thread(() ->{
                while (true) {
                    producer.produce();
                }

            }, item).start();
        });

        Stream.of("C1", "C2", "C3", "C4").forEach(item ->{
            new Thread(() -> {
                while (true) {
                    consumer.consume();
                }

            }, item).start();
        });


        System.out.println("主线程执行结束");
    }
}

@SuppressWarnings("ALL")
class Producer2 {

    private Container2 container;
    private int i = 0;

    public Producer2(Container2 container){
        this.container = container;
    }

    public void produce(){
        
        synchronized (container){
            // 这里的while不能换成if
            // 假设container现在是满的，C线程消费了一个个，然后调用container.notifyAll();通知A、B线程唤醒
            // A、B线程从container.wait();这一行代码处唤醒后处于就绪状态， A获得锁， 往container.wait();后面执行，
            // A执行 container.push(++i); 执行完后container满了，A执行完synchronized代码块释放锁， 紧接着B获取到锁一样从
            // container.wait();往后执行，假如这里while换成if, 那么B就会又执行container.push(++i);
            // 导致容器满了仍然向container push数据，这样就出现错误数据了
            while (container.isOverflow()) {
                try {
                    container.wait();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
            
            container.push(++i);
            System.out.println("[" + Thread.currentThread().getName() +"] produce " + i);

            container.notifyAll();
        }

        try {
            Thread.sleep(100);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}

@SuppressWarnings("ALL")
class Consumer2{
    private Container2 container;

    public Consumer2(Container2 container){
        this.container = container;
    }

    public void consume(){
        synchronized (container){
            // 这里的while不能换成if
            
            // 假设A、B线程都执行到container.wait();这行代码处，A、B处于wait状态， 此时生产者执行了container.notifyAll();
            // 然后A线程获得了对象锁， 从container.wait();处往后执行，A调用container.pop()后container变为空的
            // 假设这里while换成if, 那么会出现紧接着B获得了对象锁，一样地从从container.wait();处往后执行，但是container已经是空的了
            // 任然调用container.pop()就会报出ArrayIndeOutOfBoundExeption了
            while (container.isEmpty()){
                try {
                    container.wait();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
            
           
            Object value = container.pop();
            System.out.println("[" + Thread.currentThread().getName() +"]  consume " + value);

            container.notifyAll();
        }
    }
}

class Container2{

    private LinkedList<object> storage;

    private int capticy;


    public Container2(int capticy){
        this.storage = new LinkedList<>();

        this.capticy = capticy;
    }

    public void push(Object obj){
        this.storage.addLast(obj);
    }

    public Object pop(){
        return this.storage.removeFirst();
    }

    public int size(){
        return this.storage.size();
    }

    public boolean isOverflow(){
        return size() >= capticy;
    }

    public boolean isEmpty(){
        return this.storage.isEmpty();
    }
}
```

# 8 、捕获线程运行期间的异常
```
package chapter2;

/**
 * @author calebzhao<9 3 9 3 4 7 5 0 7 @ qq.com>
 * 2019/7/1 22:03
 */
public class ExceptionCaught {

    public static void main(String[] args) {

        Thread mythread = new Thread(() ->{
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }, "mythread");

        // 捕获异常
        mythread.setUncaughtExceptionHandler((thread, throwable) ->{
            System.out.println(thread.getName());

            throwable.printStackTrace();
        });

        mythread.start();

        mythread.interrupt();
    }
}
```

![](https://oscimg.oschina.net/oscnet/up-6361eaa7ef6389561d072da8d0ccbf10187.png)

# 9、ThreadGroup
```
package chapter2;

import java.util.stream.Stream;

/**
 * @author calebzhao<9 3 9 3 4 7 5 0 7 @ qq.com>
 * 2019/7/1 20:27
 */
public class ThreadGroupDemo {

    public static void main(String[] args) {
        // main方法也是一个线程， 其名称为main
        System.out.println(Thread.currentThread().getName());

        // main线程的线程组的名称是main
        System.out.println(Thread.currentThread().getThreadGroup().getName());

        // 创建线程组tg1
        ThreadGroup tg1 = new ThreadGroup("tg1");

        new Thread(tg1, "t1"){
            @Override
            public void run() {
                    try {
                        Thread.sleep(1000);
                        //获取当前线程组的名称
                        System.out.println(this.getThreadGroup().getName());
                        //获取当前线程组的父线程组
                        System.out.println(this.getThreadGroup().getParent());
                        //获取当前线程组的父线程组的名称
                        System.out.println(this.getThreadGroup().getParent().getName());
                        //评估当前线程组的父线程组及子级的线程数量
                        System.out.println(this.getThreadGroup().getParent().activeCount());
                        System.out.println("--------------");
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
            }
        }.start();

        // 创建线程组tg2
        ThreadGroup tg2 = new ThreadGroup("tg2");
        tg2.setDaemon(true);
        Stream.of("T1", "T2", "T3").forEach(name ->{
            new Thread(tg2, name){
                @Override
                public void run() {
                    System.out.println(this.getThreadGroup().getName());
                    System.out.println(tg1.getParent());
                    System.out.println(this.getThreadGroup().getParent().getName());
                    // 评估父线程组下的线程数量
                    System.out.println(this.getThreadGroup().getParent().activeCount());

                    Thread[] threads = new Thread[tg1.activeCount()];
                    this.getThreadGroup().enumerate(threads);
                    Stream.of(threads).forEach(System.out::println);
                    System.out.println("***************");

                    // 测试某个线程是parentOf(group)参数中group的父线程（直接或间接）
                    System.out.println("parentOf: " + this.getThreadGroup().getParent().parentOf(this.getThreadGroup()));
                    System.out.println(Thread.currentThread().getName() +" isDaemon:" + this.isDaemon());

                    try {
                        Thread.sleep(100);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }.start();
        });

        // 打断整个线程组下的所有线程
        tg2.interrupt();
    }
}
```

![](https://oscimg.oschina.net/oscnet/up-558774719105228a7681fd0f52957613efb.png)

# 10、单例设计模式
## version1 （不好）
```
 package chapter3.singleton;

/**
 * 问题是不能懒加载， 在类加载的时候就会初始化instance
 *
 * @author calebzhao<9 3 9 3 4 7 5 0 7 @ qq.com>
 * 2019/7/4 19:47
 */
public class SingletonVersion1 {

    private static final SingletonVersion1 instance = new SingletonVersion1();

    public static SingletonVersion1 getInstance(){
        return instance;
    }
}
```


## version2 (错误)
```
package chapter3.singleton;

/**
 * @author calebzhao<9 3 9 3 4 7 5 0 7 @ qq.com>
 * 2019/7/4 19:47
 */
public class SingletonVersion2 {
    private static SingletonVersion2 instance ;

    private SingletonVersion2(){

    }

    /**
     * 多线程并发访问时可能同时进入if代码里，造成多次实例化
     * @return
     */
    public static SingletonVersion2 getInstance(){
        if (instance == null){
           instance = new SingletonVersion2();
        }
        return instance;
    }

 }
```

## version3 (不好)
```
package chapter3.singleton;

/**
 * @author calebzhao<9 3 9 3 4 7 5 0 7 @ qq.com>
 * 2019/7/4 19:47
 */
public class SingletonVersion3 {
    private static SingletonVersion3 instance ;

    private SingletonVersion3(){

    }

    /**
     * 会有性能问题， 实例化后每次读取都要同步
     *
     * @return
     */
    public static synchronized SingletonVersion3 getInstance(){
        if (instance == null){
           instance = new SingletonVersion3();
        }

        return instance;
    }
}
```

## version4 (不好)
jvm指令重排序可能NullPointerException

```
package chapter3.singleton;

/**
 * @author calebzhao<9 3 9 3 4 7 5 0 7 @ qq.com>
 * 2019/7/4 19:47
 */
public class SingletonVersion4 {
    private static SingletonVersion4 instance ;

    private static Object LOCK = new Object();

    private boolean init;

    private SomeObject someObject = null;

    private SingletonVersion4(){
        init = true;
//        try {
//            Thread.sleep(3000);
//        } catch (InterruptedException e) {
//            e.printStackTrace();
//        }
        someObject = new SomeObject();
    }

    /**
     * 貌似没问题，但是极端情况会有NPL空指针异常问题， 比如这个类里有一些变量， 这些变量在构造方法里初始化
     * 当有多个线程调用getInstance时， 第一个线程访问时instance为null会进行实例化， 这是会在堆内存分配内存空间
     * 分配完内存空间，但是构造方法并没有执行完， 此时第二个线程访问时instance不为null返回Instance实例，直接调用里面的方法
     * new SingletonVersion4()
     *
     * @return
     */
    public static SingletonVersion4 getInstance(){
        if (instance == null){
            synchronized (LOCK){
                if (instance == null){
                    instance = new SingletonVersion4();
                }
            }

        }

        return instance;
    }

    public void print(){
        this.someObject.doSomething();
    }

    public static void main(String[] args) {
        Thread t1 = new Thread(() ->{
            SingletonVersion4.getInstance();
        }, "t1");

        Thread t2 = new Thread(() ->{
            SingletonVersion4.getInstance().print();
        }, "t2");

        t1.start();
        t2.start();
    }
}

class SomeObject{

    public void doSomething(){
        System.out.println(Thread.currentThread().getName() + " doSomething...");
    }
}
```

![](https://oscimg.oschina.net/oscnet/up-2c7d4a93a462c085f91b01792dbd97171a4.png)
#### version5 (正确、推荐)
```
package chapter3.singleton;

/**
 * @author calebzhao<9 3 9 3 4 7 5 0 7 @ qq.com>
 * 2019/7/4 19:47
 */
public class SingletonVersion5 {

    private SingletonVersion5(){
    }

    private static class Singleton{
        private static final SingletonVersion5 INSTANCE = new SingletonVersion5();
    }

    /**
     * 貌似没问题，但是极端情况会有NPL空指针异常问题， 比如这个类里有一些变量， 这些变量在构造方法里初始化
     * 当有多个线程调用getInstance时， 第一个线程访问时instance为null会进行实例化， 这是会在堆内存分配内存空间
     * 分配完内存空间，但是构造方法并没有执行完， 此时第二个线程访问时instance不为null返回Instance实例，直接调用里面的方法
     * new SingletonVersion4()
     *
     * @return
     */
    public static SingletonVersion5 getInstance(){
        return Singleton.INSTANCE;
    }
}
```

#### version 6 （正确）
```
package chapter3.singleton;

/**
 * @author calebzhao<9 3 9 3 4 7 5 0 7 @ qq.com>
 * 2019/7/4 19:47
 */
public class SingletonVersion6 {

    private SingletonVersion6(){
    }

    enum Singleton{
        INSTANCE;

        private SingletonVersion6 singletonVersion6 = new SingletonVersion6();

        public SingletonVersion6 getInstance(){
            return singletonVersion6;
        }
    }

   
    public static SingletonVersion6 getInstance(){
        return Singleton.INSTANCE.getInstance();
    }
}
```


# 11、volatile关键字
## 11.1 高并发的三个特性
1. **原子性**

例如 i = 9;  在16位计算机中 可能是16 16位分2次赋值的，比如低16位赋值成功、高16位赋值失败。

在一个或多个操作中，要么全部成功，要么全部失败，不能有中间状态。

a = 1;         保证原子性

a++;           **非原子性**，执行步骤：1：读取a; 2: 对a加1；  3：将结果赋值给a

a = 1+2      保证原子性， 编译期会确定值

a = a+1     ** 非原子性**，执行步骤：1：读取a; 2: 对a加1；  3：将结果赋值给a

**证明 ++value 操作不是原子性的**

```
package volatileDemo;

/**
 * @author calebzhao<9 3 9 3 4 7 5 0 7 @ qq.com>
 * 2019/7/8 22:01
 */
public class AtomDemo {

    private static int initValue = 1;

    private static int maxValue = 500;

    /**
     * 证明++initValue操作不是原子性的，
     *
     * 假设如下2种情况：
     * 1、如果 ++initValue操作是原子性的， 那么输出一定不会有重复的
     * 2、如果 ++initValue操作不熟原子性的，而是拆分成1、读取initValue; 2其他操作，那么可能多个线程读取到一样的initValue
     *
     * 那么可能出现如下情况：
     * t1  -> 读取 initValue值为10， cpu执行权被切换到t2
     * t2  -> 读取 initValue值为10,
     * t2  -> 11 = 10 +1
     * t2  -> initValue = 11
     * t2  -> System.out.printf("t2 执行后结果 [%d] \n", 11);
     * t1  -> 11 = 10 +1
     * t1  -> initValue = 11
     * t1  -> System.out.printf("t1 执行后结果 [%d] \n", 11);
     *
     * @param args
     */
    public static void main(String[] args) {
        new Thread(() ->{
            while (initValue < maxValue){
                System.out.printf("t1 执行后结果 [%d] \n", ++initValue);
                try {
                    Thread.sleep(100);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }).start();

        new Thread(() ->{
            while (initValue < maxValue){
                System.out.printf("t2 执行后结果 [%d] \n", ++initValue);
                try {
                    Thread.sleep(100);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }).start();
    }
}
```


![](https://oscimg.oschina.net/oscnet/up-645cefb328584fe0e3435ef04978f461e22.png)


2. **可见性**

**volatile关键字会保证多线程下的内存可见性及指令执行的有序性**

[https://www.cnblogs.com/yanlong300/p/8986041.html](https://www.cnblogs.com/yanlong300/p/8986041.html)

例子：

```
package volatileDemo;

/**
 * @author calebzhao<9 3 9 3 4 7 5 0 7 @ qq.com>
 * 2019/7/8 21:25
 */
public class VolatileDemo {

    private static int  initValue = 1;

    private static int MAX_VALUE = 50;

    public static void main(String[] args) {
        new Thread(() ->{
            int localValue = initValue;
            while (localValue < MAX_VALUE){
//                System.out.printf("t1线程读取到initValue的值为 [%d] localValue：[%d]\n", initValue, localValue);
                if (localValue != initValue){
                    localValue = initValue;
                    System.out.printf("initValue的值已被更新为 [%d]\n", initValue);
                }
//                else{
//                    System.out.printf("t1线程读取到initValue的值为 [%d] localValue：[%d]\n", initValue, localValue);
//                }
            }

            System.out.println("t1线程结束执行");
        }, "t1").start();

        new Thread(() ->{
            int localValue = initValue;
            while (initValue < MAX_VALUE){
                localValue++;
                initValue = localValue;
                System.out.printf("更新initValue的值为 [%d]\n", initValue);

                try {
                    Thread.sleep(500);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }

            System.out.println("t2线程结束执行");
        }, "t2").start();
    }
}
```


![](https://oscimg.oschina.net/oscnet/up-ff3e395f5ddb81be58c8535fcdc5faa180b.png)


当private static volatile int   *initValue *= 1; 这句代码加上volatile 关键字就是正确的结果了

![](https://oscimg.oschina.net/oscnet/up-c69faeca7d8a784c112d97a38928b55b5bb.png)


3. **有序性**
```
value = 3；

void exeToCPUA(){
  value = 10;   # 可能发生重排序 value的赋值发生在isFinsh之后
  isFinsh = true;
}

void exeToCPUB(){
  if(isFinsh){
    //value一定等于10？！
    assert value == 10;
  }
}
```
试想一下开始执行时，CPU A保存着finished在E(独享)状态，而value并没有保存在它的缓存中。（例如，Invalid）。在这种情况下，value会比finished更迟地抛弃存储缓存。完全有可能CPU B读取finished的值为true，而value的值不等于10。
**    即isFinsh的赋值在value赋值之前。**

这种在可识别的行为中发生的变化称为重排序（reordings）。注意，这不意味着你的指令的位置被恶意（或者好意）地更改。

它只是意味着其他的CPU会读到跟程序中写入的顺序不一样的结果。

为什么会有指令的重排序？

答案：因为为了使缓存能够得到更加合理地利用。

```
int a=1
int b=2
```
## 省略一万行代码...

int c=a+b

最后一句放第三行就能让缓存更合理, 原因：cpu将a=1读入CPU高速缓存，然后将b=2读入高速缓存，由于CPU高速缓存的容量很小，所以当执行后面的一万行代码时CPU高速缓存满了，那么就会把a=1、b=2这2个缓存行覆盖掉，当真正执行int c= a+ b时由于CPU高速缓存里面没有数据那么CPU就要重新从主存读取数据然后计算，这样就出现了不必须的重复读取主存的操作，浪费CPU，通过重指令排序让int c=a+b放到第三行则可以缓存能够立即得到利用，将c=a+b的结果计算后可以立即回写主内存，避免后续a=1、b=2的缓存行被其他指令的缓存行覆盖

11.2、volatile的内存语义
# 12、比较并交换（CAS)
## 12.1、使用CAS与使用锁相比的好处
与锁相比，使用比较并交换（CAS）会使程序看起来更复杂，但由于其非阻塞性，它对死锁问题天生免疫，并且线程间的影响也远远比基于锁的方式小的多。更为重要的是，**使用无锁的方式**完全没有锁竞争带来的开销，也没有线程间频繁调度调来的开销，因此它比基于锁的方式拥有更优越的性能。

## 12.2、CAS原理
CAS算法的过程是：它包含3个参数CAS(realValue,  expectValue, newValue), 其中realValue表示要更新的变量，expectValue表示预期值，newValue表示新值。仅当realValue等与expectValue值时，才将realValue的值变更为newValue，如果realValue与expectValue不相等，说明有其他线程已经更新做了更新，则当前线程什么也不做，最后CAS返回当前realValue的真实值。CAS是抱着乐观的态度去进行的，它总是认为自己可以完成操作，当多个线程同时使用CAS操作一个变量时，只有一个会胜出，并成功更新，其余均会失败。失败的线程不会被挂起，仅是被告知失败，并且允许再次尝试，当然允许失败的线程放弃操作。
    
简单的说，CAS需要你额外给出一个预期值，也就是你认为现在这个变量应该是什么样子的。如果变量不是你想象的那样，则说明已经被别人修改过了。你就重新读取，再次尝试修改就好了。
    
**在硬件层面大部分的处理器都已支持原子化的CAS指令**。在JDK5后，虚拟机便可以使用这个指令来进行原子化操作。

## 13.3、CAS实现计数器
```java
package 自旋锁计数;

import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.atomic.AtomicInteger;

public class Counter<psvm> {

	/**
     * 线程安全方式计数， 内部的value子段声明为 private volatile int value;所以保证了内存可见性
     */
	private AtomicInteger atomic = new AtomicInteger(0);

	/**
     * 线程不安全，非原子性操作计数
     */
	private int i = 0;

	public void count(){
		i++;
	}

	public void safeCount(){

		while (true){
			// 1、多核处理器可能会同时运行到这行代码，单核处理器由于时间片分配算法T1执行到这行代码后CPU执行权被T2获取了， 线程T1、T2均通过get()方法返回0
			// 2、假如T1先执行atomic.compareAndSet(currentValue, ++currentValue)这行代码，
			//    由于currentValue和atomic的值一致，cas操作成功，atomic变成1，退出循环,
			// 3、然后T2继续执行atomic.compareAndSet(currentValue, ++currentValue);
			//    这行代码会发现atomic内部维护的value值1已经与currentValue的值0不相等，不会进行设置值操作
			//    T2继续下次循环, 又执行atomic.get();获取到的currentValue为1， 再次执行compareAndSet时，
			//    atomic为1和currentValue为1相等，成功进行cas操作，然后退出循环
			int currentValue = atomic.get();
			boolean success = atomic.compareAndSet(currentValue, ++currentValue);
			if (success){
				break;
			}
		}

	}


	public static void main(String[] args) {
		Counter counter = new Counter();

		List<thread> threadList = new ArrayList<>(500);
		for (int j = 0; j < 500; j++){
			Thread thread = new Thread(() ->{
				try {
					Thread.sleep(100);
				} catch (InterruptedException e) {
					e.printStackTrace();
				}
				counter.count();

				counter.safeCount();
			});

			threadList.add(thread);
		}

		threadList.stream().forEach(thread -> thread.start());

		threadList.forEach(thread -> {
			try {
				thread.join();
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
		});

		System.out.println("count: " + counter.i);
		System.out.println("safeCount:" + counter.atomic);

	}
}
```

##    13.3、使用CAS实现无锁同步
```
package atomic;

import java.util.concurrent.atomic.AtomicInteger;

/**
 * 使用cas实现无锁同步
 *
 * @author calebzhao<9 3 9 3 4 7 5 0 7 @ qq.com>
 * 2019/8/23 19:46
 */
public class CasLock {

    private AtomicInteger lock = new AtomicInteger(0);

    //记录当前获取到锁的线程ID
    private Long getLockThreadId;


    public void tryLock(){
        while (true){
            boolean success = lock.compareAndSet(0, 1);
            if (success){
                getLockThreadId = Thread.currentThread().getId();
                break;
            }

            try {
                Thread.sleep(1);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }

    public void unLock(){
        long currentThreadId = Thread.currentThread().getId();
        if (currentThreadId != getLockThreadId){
            throw new IllegalStateException("未获取到锁，无需解锁");
        }

        int value = lock.get();
        if (value == 1){
            lock.set(0);
        }
    }
}
```


测试示例：

```
package atomic;

import java.util.ArrayList;
import java.util.BitSet;
import java.util.List;
import java.util.Random;

/**
 * @author calebzhao<9 3 9 3 4 7 5 0 7 @ qq.com>
 * 2019/8/23 19:50
 */
public class CasLockDemo {

    private static int i = 0;

    public static void main(String[] args) {
        BitSet bitSet = new BitSet();

        CasLock lock = new CasLock();

        List<thread> threadList = new ArrayList<>(500);
        Random random = new Random();

        for (int  j =0 ; j < 50000; j++){
            Thread t = new Thread(() -> {
                try {
                    Thread.sleep(random.nextInt(20) + 200);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }

                lock.tryLock();
                i++;
                if(bitSet.get(i)){
                    throw new RuntimeException("lock有问题");
                }

                bitSet.set(i);
                System.out.println(Thread.currentThread().getName() + " get lock, i=" + i);

                lock.unLock();
            }, "thread-" + j);

            threadList.add(t);
        }

        threadList.forEach(thread -> thread.start());
    }
}
```


# 13、atomic包原子操作类（无锁CAS）
## 13.1、AtomicInteger
主要方法：

![](https://oscimg.oschina.net/oscnet/up-751f793bc5ec254a0652d2f92b45fceecbb.png)

示例：

```
package atomic;

import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.atomic.AtomicInteger;

/**
 * @author calebzhao<9 3 9 3 4 7 5 0 7 @ qq.com>
 * 2019/8/25 15:07
 */
public class AtomicIntegerDemo {

    /**
     * 线程安全方式计数， 内部的value子段声明为 private volatile int value;所以保证了内存可见性
     */
    private AtomicInteger atomic = new AtomicInteger(0);

    /**
     * 线程不安全，非原子性操作计数
     */
    private int i = 0;

    public void count(){
        i++;
    }

    public void safeCount(){
        atomic.incrementAndGet();
    }

    public static void main(String[] args) {
        AtomicIntegerDemo counter = new AtomicIntegerDemo();

        List<thread> threadList = new ArrayList<>(500);

        for (int j = 0; j < 500; j++){
            Thread thread = new Thread(() ->{
                try {
                    Thread.sleep(100);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                counter.count();

                counter.safeCount();
            });

            threadList.add(thread);
        }

        threadList.stream().forEach(thread -> thread.start());

        threadList.forEach(thread -> {
            try {
                thread.join();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        });

        System.out.println("count: " + counter.i);
        System.out.println("safeCount:" + counter.atomic);
    }
}
```
![](https://oscimg.oschina.net/oscnet/up-2abc872153fcd998e98bea20be041400743.png)

## 13.2、AtomicReference
AtomicReference，顾名思义，就是以原子方式更新对象引用

可以看到，AtomicReference持有一个对象的引用——**value**，并通过Unsafe类来操作该引用:

![](https://oscimg.oschina.net/oscnet/up-9420164fbbd4a44d61338e10a1828a74d88.png)

>为什么需要AtomicReference？难道多个线程同时对一个引用变量赋值也会出现并发问题？
>引用变量的赋值本身没有并发问题，也就是说对于引用变量var ，类似下面的赋值操作本身就是原子操作:
>Foo var = ... ;
>**AtomicReference的引入是为了可以用一种类似乐观锁的方式操作共享资源，在某些情景下以提升性能。**

我们知道，当多个线程同时访问共享资源时，一般需要以加锁的方式控制并发：

```
volatile Foo sharedValue = value;
Lock lock = new ReentrantLock();

lock.lock();
try{
    // 操作共享资源sharedValue
}
finally{
    lock.unlock();
}
```

上述访问方式其实是一种对共享资源加**悲观锁**的访问方式。
而AtomicReference提供了**以无锁方式访问共享资源**的能力，看看如何通过AtomicReference保证线程安全，来看个具体的例子：

```
package atomic;

import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.atomic.AtomicReference;

/**
 * @author calebzhao<9 3 9 3 4 7 5 0 7 @ qq.com>
 * 2019/8/24 20:50
 */
public class AtomicReferenceCounter {

    public static void main(String[] args) throws InterruptedException {
        AtomicReference<integer> ref = new AtomicReference<>(new Integer(0));

        List<thread> list = new ArrayList<>();
        for (int i = 0; i < 1000; i++) {
            Thread t = new Thread(new Task(ref), "Thread-" + i);
            list.add(t);
        }

        for (Thread t : list) {
            t.start();
            t.join();
        }

        // 打印2000
        System.out.println(ref.get());
    }
}

class Task implements Runnable{

    private AtomicReference<integer> reference;

    public Task(AtomicReference<integer> reference){
        this.reference = reference;
    }

    @Override
    public void run() {
        while (true){
            Integer oldValue = reference.get();
            boolean success = reference.compareAndSet(oldValue, oldValue + 1);
            if (success){
                break;
            }
        }
    }
}
```

![](https://oscimg.oschina.net/oscnet/up-259bb67ab8c852ac92d4bcfa8306222fb9f.png)

该示例并没有使用锁，而是使用**自旋+CAS**的无锁操作保证共享变量的线程安全。1000个线程，每个线程对金额增加1，最终结果为2000，如果线程不安全，最终结果应该会小于2000。

通过示例，可以总结出AtomicReference的一般使用模式如下

```
AtomicReference<object> ref = new AtomicReference<>(new Object());
Object oldCache = ref.get();

// 对缓存oldCache做一些操作
Object newCache  =  someFunctionOfOld(oldCache); 

// 如果期间没有其它线程改变了缓存值，则更新
boolean success = ref.compareAndSet(oldCache , newCache);
```
上面的代码模板就是AtomicReference的常见使用方式，看下**compareAndSet**方法：
![](https://oscimg.oschina.net/oscnet/up-2d62cec526978d26a7758591f4bae7eafcf.png)

该方法会将入参的**expect**变量所指向的对象和AtomicReference中的引用对象进行比较，如果两者指向同一个对象，则将AtomicReference中的引用对象重新置为**update**，修改成功返回true，失败则返回false。也就是说，***AtomicReference其实是比较对象的引用***。

## 13.2、CAS操作可能存在的ABA问题
### 13.2.1、介绍
>CAS操作可能存在ABA的问题，就是说：
>假如一个值原来是A，变成了B，又变成了A，那么CAS检查时会发现它的值没有发生变化，但是实际上却变化了。

一般来讲这并不是什么问题，比如数值运算，线程其实根本不关心变量中途如何变化，只要最终的状态和预期值一样即可。

但是，有些操作会依赖于对象的变化过程，此时的解决思路一般就是使用版本号。在变量前面追加上版本号，每次变量更新的时候把版本号加一，那么A－B－A 就会变成1A - 2B - 3A。

### 13.2.3、贵宾充值卡问题（ABA示例）
**举例**：有一家蛋糕店为了挽留客户，决定为贵宾卡里小于20元的客户一次性充值20元，刺激客户充值和消费，但条件是：每位客户只能被赠送一次。

```
package atomic;

import java.util.concurrent.atomic.AtomicReference;

/**
 * cas ABA问题
 * @author calebzhao<9 3 9 3 4 7 5 0 7 @ qq.com>
 * 2019/8/25 15:30
 */
public class CasABAPromblem {

    private static AtomicReference<integer> money = new AtomicReference<>(19);

    public static void main(String[] args) {

        ChargeMoneyWorker chargeMoneyWorker = new ChargeMoneyWorker(money);
        for (int i = 0; i < 3; i++){
            new Thread(chargeMoneyWorker).start();
        }

        ConsumeMoneyWorker consumeMoneyWorker = new ConsumeMoneyWorker(money);
        new Thread(consumeMoneyWorker).start();
    }

}

class ChargeMoneyWorker implements Runnable{

    private AtomicReference<integer> money;

    public ChargeMoneyWorker(AtomicReference<integer> money){
        this.money = money;
    }

    @Override
    public void run() {
        while (true){

            while (true){
                Integer m = money.get();
                if (m < 20){
                    boolean success  = money.compareAndSet(m, m +20);
                    if (success){
                        System.out.println("余额小于20元，充值成功， 充值后余额：" + money.get());
                        break;
                    }
                }
                else {
//                    System.out.println("余额大于20元，无需充值");
                    break;
                }
            }

        }

    }
}

class ConsumeMoneyWorker implements Runnable{
    private AtomicReference<integer> money;

    public ConsumeMoneyWorker(AtomicReference<integer> money){
        this.money = money;
    }

    @Override
    public void run() {
        while (true){

            while (true){
                Integer m = money.get();
                if (m > 10){
                    boolean success  = money.compareAndSet(m, m - 10);
                    if (success){
                        System.out.println("成功消费10元， 余额：" + money.get());
                        break;
                    }
                }
                else {
                    System.out.println("余额不足10元");
                }
            }

            try {
                Thread.sleep(500);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }

        }

    }
 }
```

![](https://oscimg.oschina.net/oscnet/up-4dcaed85c641a37d145edfc18d4d84b9ed7.png)

从上面的输出可以到用户的账户被先后反复充值，其原因是用户的账户余额被反复修改，导致修改后又满足的原充值条件，使得充值线程无法正确判断该用户是否已经充值过。

虽然这种情况出现的概率不大，但是依然也是由可能出现的，因此当业务中确实出现这种问题，我们需要注意是否是我们的业务本身就不合理。JDK为我们考虑到了这种情况，使用AtomicStampedReference可以很好的解决这个问题。

## 13.3、AtomicStampedReference
AtomicStampedReference就是上面所说的加了版本号的AtomicReference。

### 13.3.1、AtomicStampedReference原理
先来看下如何构造一个AtomicStampedReference对象，AtomicStampedReference只有一个构造器：

![](https://oscimg.oschina.net/oscnet/up-058d51af7f8886b2a7dc793d8f5db984ae2.png)

可以看到，除了传入一个初始的引用变量**initialRef**外，还有一个**initialStamp**变量，**initialStamp**其实就是版本号（或者说时间戳），用来唯一标识引用变量。

在构造器内部，实例化了一个**Pair**对象，**Pair**对象记录了对象引用和时间戳信息，采用int作为时间戳，实际使用的时候，要保证时间戳唯一（一般做成自增的），如果时间戳如果重复，还会出现**ABA**的问题。

>AtomicStampedReference的所有方法，其实就是Unsafe类针对这个Pair对象的操作。
>和AtomicReference相比，AtomicStampedReference中的每个引用变量都带上了pair.stamp这个版本号，这样就可以解决CAS中的ABA问题了。
### 13.3.2、AtomicStampedReference使用示例
```
// 创建AtomicStampedReference对象，持有Foo对象的引用，初始为null，版本为0
AtomicStampedReference<foo>  asr = new AtomicStampedReference<>(null,0);  

int[] stamp=new  int[1];
Foo  oldRef = asr.get(stamp);   // 调用get方法获取引用对象和对应的版本号
```
int oldStamp=stamp[0];          // stamp[0]保存版本号

asr.compareAndSet(oldRef, null, oldStamp, oldStamp + 1)   //尝试以CAS方式更新引用对象，并将版本号+1

上述模板就是AtomicStampedReference的一般使用方式，注意下***compareAndSet***方法：
![](https://oscimg.oschina.net/oscnet/up-6024fc1db40a826c6add3fa37148921f26b.png)

我们知道，AtomicStampedReference内部保存了一个pair对象，该方法的逻辑如下：

1. 如果AtomicStampedReference内部pair的引用变量、时间戳 与 入参**expectedReference**、**expectedStamp**都一样，说明期间没有其它线程修改过AtomicStampedReference，可以进行修改。此时，会创建一个新的Pair对象（casPair方法，因为Pair是Immutable类）。

但这里有段优化逻辑，就是如果&nbsp;newReference == current.reference &amp;&amp; newStamp == current.stamp，说明用户修改的新值和AtomicStampedReference中目前持有的值完全一致，那么其实不需要修改，直接返回true即可。


### **13.3.3、**AtomicStampedReference**解决贵宾卡多次充值问题**
```
package atomic;

import java.util.concurrent.atomic.AtomicStampedReference;

/**
 * cas ABA问题解决方案
 * @author calebzhao<9 3 9 3 4 7 5 0 7 @ qq.com>
 * 2019/8/25 15:30
 */
public class ResolveCasABAProblem {

    private static AtomicStampedReference<integer> money = new AtomicStampedReference<>(19, 0);

    public static void main(String[] args) {

        ResolveChargeMoneyWorker chargeMoneyWorker = new ResolveChargeMoneyWorker(money);
        for (int i = 0; i < 3; i++){
            new Thread(chargeMoneyWorker).start();
        }

        ResolveConsumeMoneyWorker consumeMoneyWorker = new ResolveConsumeMoneyWorker(money);
        new Thread(consumeMoneyWorker).start();
    }

}

class ResolveChargeMoneyWorker implements Runnable{

    private AtomicStampedReference<integer> money;

    public ResolveChargeMoneyWorker(AtomicStampedReference<integer> money){
        this.money = money;
    }

    @Override
    public void run() {
        while (true){
            try {
                Thread.sleep(400);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            while (true){
                Integer m = money.getReference();
                if (m < 20){
                    boolean success  = money.compareAndSet(m, m +20, 0 , 1);
                    if (success){
                        System.out.println("余额小于20元，充值成功， 充值后余额：" + money.getReference());
                        break;
                    }
                }
                else {
//                    System.out.println("余额大于20元，无需充值");
                    break;
                }
            }

        }

    }
}

class ResolveConsumeMoneyWorker implements Runnable{
    private AtomicStampedReference<integer> money;

    public ResolveConsumeMoneyWorker(AtomicStampedReference<integer> money){
        this.money = money;
    }

    @Override
    public void run() {
        while (true){

            while (true){
                Integer m = money.getReference();
                int stamp = money.getStamp();
                if (m > 10){
                    // 这里为什么不是给版本号加1呢？
                    // 假如这里变成  boolean success  = money.compareAndSet(m, m - 10, stamp, stamp + 1);
                    // 考虑一种情况，用户账户本身有19元钱， 初始充值状态为0表示为充值, 用户先消费了10元，stamp变为1
                    // 这时用户还没有充值，账户金额也确实少于20元，这会导致充值线程扫描是发现stamp已变为1，就不会充值了
                    // 根本原因是用户是否消费与是否充值过无关，充值的状态不能由于其他因素改变
                    // 这个例子是《实战高并发程序设计》一书中的例子，书中例子没有考虑到这点，
                    // 书中假想的情况是先充值后消费，但如果是先消费再充值就有问题了
                    boolean success  = money.compareAndSet(m, m - 10, stamp, stamp);
                    if (success){
                        System.out.println("成功消费10元， 余额：" + money.getReference());
                        break;
                    }
                }
                else {
                    System.out.println("余额不足10元");
                }
            }

            try {
                Thread.sleep(500);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }

        }

    }
}
```


**情况1：先充值后消费**

![](https://oscimg.oschina.net/oscnet/up-a206538fda31fbac36ebf648e963233601b.png)

**情况2：先消费，后充值**

![](https://oscimg.oschina.net/oscnet/up-405901c6ff52d8dda3aa7cefff4a99d936c.png)

## 13.4、AtomicMarkableReference
AtomicMarkableReference是AtomicStampedReference的特殊化形式**AtomicMarkableReference用于无需知道数据目前具体是哪个版本，只需要知道数据是否被更改过。**

前面的客户账户充值例子ABA问题使用AtomicMarkableReference解决的代码如下：

```
package atomic;

/**
 * @author calebzhao<9 3 9 3 4 7 5 0 7 @ qq.com>
 * 2019/8/25 17:45
 */

import java.util.concurrent.atomic.AtomicMarkableReference;

/**
 * cas ABA问题
 * @author calebzhao<9 3 9 3 4 7 5 0 7 @ qq.com>
 * 2019/8/25 15:30
 */
public class AtomicMarkableReferenceDemo {

    private static AtomicMarkableReference<integer> money = new AtomicMarkableReference<>(19, false);

    public static void main(String[] args) {

        MarkableChargeMoneyWorker chargeMoneyWorker = new MarkableChargeMoneyWorker(money);
        for (int i = 0; i < 3; i++){
            new Thread(chargeMoneyWorker).start();
        }

        MarkableConsumeMoneyWorker consumeMoneyWorker = new MarkableConsumeMoneyWorker(money);
        new Thread(consumeMoneyWorker).start();
    }

}

class MarkableChargeMoneyWorker implements Runnable{

    private AtomicMarkableReference<integer> money;

    public MarkableChargeMoneyWorker(AtomicMarkableReference<integer> money){
        this.money = money;
    }

    @Override
    public void run() {
        while (true){
            try {
                Thread.sleep(400);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }

            while (true){
                Integer m = money.getReference();
                if (m < 20){
                    boolean success  = money.compareAndSet(m, m +20, false , true);
                    if (success){
                        System.out.println("余额小于20元，充值成功， 充值后余额：" + money.getReference());
                        break;
                    }
                }
                else {
//                    System.out.println("余额大于20元，无需充值");
                    break;
                }
            }

        }

    }
}

class MarkableConsumeMoneyWorker implements Runnable{
    private AtomicMarkableReference<integer> money;

    public MarkableConsumeMoneyWorker(AtomicMarkableReference<integer> money){
        this.money = money;
    }

    @Override
    public void run() {
        while (true){

            while (true){
                Integer m = money.getReference();
                boolean isMarked = money.isMarked();
                if (m > 10){
                    // 消费不更改充值标识
                    boolean success  = money.compareAndSet(m, m - 10, isMarked, isMarked);
                    if (success){
                        System.out.println("成功消费10元， 余额：" + money.getReference());
                        break;
                    }
                }
                else {
                    System.out.println("余额不足10元");
                }
            }

            try {
                Thread.sleep(500);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }

        }

    }
}
```

![](https://oscimg.oschina.net/oscnet/up-a516122ec60a6795845b7a9dc818e324e5f.png)

## 13.4、AtomicIntegerArray
AtomicIntegerArray本质上是对int[]类型的封装，使用Unsafe类通过CAS的方式控制int[]在多线程下的安全性，它提供了以下几个核心API：

![](https://oscimg.oschina.net/oscnet/up-80e1c38cf5e8568fd7c040effcab4575837.png)

### 13.4.1、如果没有AtomicIntegerArray的错误示例
```
package atomic;

/**
 * Increment任务：这个类使用vector[i]++方法增加数组中所有元素的值
 * Decrement任务：这个类使用vector[i]--方法减少数组中所有元素的值
 *
 * 在main方法中创建了1000个元素的int[]数组，
 * 执行了1000个Increment任务和1000个Decrement任务，在任务的结尾如果没有不一致的错误，
 * 数组中所有元素的值不全为0，执行程序后会看到程序输出了一些不全为0的数值
 * @author calebzhao<9 3 9 3 4 7 5 0 7 @ qq.com>
 * 2019/8/25 12:22
 */
public class BadAtomiceIntegerArrayDemo {

    public static void main(String[] args) throws InterruptedException {
        int[] vector = new int[1000];
        for (int  i = 0; i < vector.length; i++){
            vector[i] = 0;
        }

        BadIncrement increment = new BadIncrement(vector);
        BadDecrement decrement = new BadDecrement(vector);

        Thread[] badThreadIncrements = new Thread[1000];
        Thread[] badThreadDecrements = new Thread[1000];

        for (int i =0 ; i < badThreadIncrements.length; i++){
            badThreadIncrements[i] = new Thread(increment);
            badThreadDecrements[i] = new Thread(decrement);
        }

        for (int i =0 ; i < badThreadIncrements.length; i++){
            badThreadIncrements[i].start();
            badThreadDecrements[i].start();
        }

        for (int i =0 ; i < badThreadIncrements.length; i++){
            badThreadIncrements[i].join();
            badThreadDecrements[i].join();
        }

        for (int i =0 ; i < vector.length; i++){
           if (vector[i] != 0){
               System.out.println("Vector["+i+"] : " + vector[i]);
           }
        }

        System.out.println("main end");
    }
}

class BadIncrement implements Runnable{

    private int[] vector;

    public BadIncrement(int[] vector){
        this.vector = vector;
    }

    @Override
    public void run() {
        for (int i = 0; i <vector.length; i++){
            vector[i]++;
        }
    }
}

class BadDecrement implements Runnable{

    private int[] vector;

    public BadDecrement(int[] vector){
        this.vector = vector;
    }

    @Override
    public void run() {
        for (int i = 0; i <vector.length; i++){
            vector[i]--;
        }
    }
}

```
![](https://oscimg.oschina.net/oscnet/up-61d97c25f3f953b628729704a39df7590d0.png)


### 13.4.2、atomicatintegerarray的正确使用 
```java
package atomic;

import java.util.concurrent.atomic.AtomicIntegerArray;

public class AtomiceIntegerArrayDemo {

    public static void main(String[] args) throws InterruptedException {
        AtomicIntegerArray vector = new AtomicIntegerArray(1000);

        Increment increment = new Increment(vector);
        Decrement decrement = new Decrement(vector);

        Thread[] threadIncrements = new Thread[1000];
        Thread[] threadDecrements = new Thread[1000];

        for (int i =0 ; i < threadIncrements.length; i++){
            threadIncrements[i] = new Thread(increment);
            threadDecrements[i] = new Thread(decrement);
        }

        for (int i =0 ; i < threadIncrements.length; i++){
            threadIncrements[i].start();
            threadDecrements[i].start();
        }

        for (int i =0 ; i < threadIncrements.length; i++){
            threadIncrements[i].join();
            threadDecrements[i].join();
        }

        for (int i =0 ; i < vector.length(); i++){
           if (vector.get(i) != 0){
               System.out.println("Vector["+i+"] : " + vector.get(i));
           }
        }

        System.out.println("main end");
    }
}

class Increment implements Runnable{

    private AtomicIntegerArray vector;

    public Increment(AtomicIntegerArray vector){
        this.vector = vector;
    }

    @Override
    public void run() {
        for (int i = 0; i <vector.length(); i++){
            vector.getAndIncrement(i);
        }
    }
}

class Decrement implements Runnable{

    private AtomicIntegerArray vector;

    public Decrement(AtomicIntegerArray vector){
        this.vector = vector;
    }

    @Override
    public void run() {
        for (int i = 0; i <vector.length(); i++){
            vector.getAndDecrement(i);
        }
    }
}

```

工作原理：
Increment任务：这个类使用getAndIncrement方法增加数组中所有元素的值
  Decrement任务：这个类使用getAndDecrement方法减少数组中所有元素的值

  在main方法中创建了1000个元素的AtomicIntegerArray数组，
  执行了1000个Increment任务和1000个Decrement任务，在任务的结尾如果没有不一致的错误，
  数组中所有元素的值都应该是0，执行程序后会看到程序只将最后的main end消息打印到控制台，因为所有元素值为0

## 13.5、atomicreferencearray
**atomicreferencearray是针对普通自定义对象的数组的原子更新类的通用形式** 
下面的类实现了atomicintegerarray的功能 
```java
package atomic;

import java.util.concurrent.atomic.AtomicReferenceArray;

/**
 * @author calebzhao<9 3 9 3 4 7 5 0 7 @ qq.com>
 * 2019/8/25 17:26
 */
public class AtomicReferenceArrayDemo {


    public static void main(String[] args) throws InterruptedException {
        AtomicReferenceArray<Integer> vector = new AtomicReferenceArray(1000);
        for (int  i =0; i < vector.length(); i++){
            vector.set(i, 0);
        }

        ReferenceIncrement increment = new ReferenceIncrement(vector);
        ReferenceDecrement decrement = new ReferenceDecrement(vector);

        Thread[] threadIncrements = new Thread[1000];
        Thread[] threadDecrements = new Thread[1000];

        for (int i =0 ; i < threadIncrements.length; i++){
            threadIncrements[i] = new Thread(increment);
            threadDecrements[i] = new Thread(decrement);
        }

        for (int i =0 ; i < threadIncrements.length; i++){
            threadIncrements[i].start();
            threadDecrements[i].start();
        }

        for (int i =0 ; i < threadIncrements.length; i++){
            threadIncrements[i].join();
            threadDecrements[i].join();
        }

        for (int i =0 ; i < vector.length(); i++){
            if (vector.get(i) != 0){
                System.out.println("Vector["+i+"] : " + vector.get(i));
            }
        }

        System.out.println("main end");
    }
}


class ReferenceIncrement implements Runnable{

    private AtomicReferenceArray<Integer> vector;

    public ReferenceIncrement(AtomicReferenceArray<Integer> vector){
        this.vector = vector;
    }

    @Override
    public void run() {
        for (int i = 0; i <vector.length(); i++){


            // 不能用set， 用了set后就是不管内存现有真实值是多少，直接设置为新值， 在cpu时间片切换时会有问题
            // 正确的应该是在原有值基础上加1
//            vector.set(i, current + 1);

            while (true){
                int current = vector.get(i);
                if(vector.compareAndSet(i, current, current + 1)){
                    break;
                }
            }

        }
    }
}

class ReferenceDecrement implements Runnable{

    private AtomicReferenceArray<Integer> vector;

    public ReferenceDecrement(AtomicReferenceArray<Integer> vector){
        this.vector = vector;
    }

    @Override
    public void run() {
        for (int i = 0; i <vector.length(); i++){
            while (true){
                int current = vector.get(i);
                if(vector.compareAndSet(i, current, current - 1)){
                    break;
                }
            }
        }
    }
}
```
![](https://oscimg.oschina.net/oscnet/up-3ea4ba16dc446b821f36372f0653a85ea3e.png)

## 13.6、atomicintegerfieldupdater 
根据数据类型不同， updater有3种， 分别是atomicintegerfieldupdater、atomiclongfieldupdater、atomicreferencefieldupdater 

**示例场景:** 
假设某地要进行一次选举。现在模拟这个投票场景，如果选民投了候选人1票，就记为1，否则记为0，最终就是要统计某个选择被投票的次数。

```
package atomic; 
import java.util.arrays; 
java.util.concurrent.atomic.atomicinteger;
java.util.concurrent.atomic.atomicintegerfieldupdater; 
/** 
* @author calebzhao<9 3 9 4 7 5 0 @ qq.com>
 * 2019/8/25 18:01
 */
public class AtomicIntegerFieldUpdaterDemo {

    private static AtomicIntegerFieldUpdater<candidate> scoreUpdater = AtomicIntegerFieldUpdater.newUpdater(Candidate.class, "score");

    //作用是为了来验证AtomicIntegerFieldUpdater计算的正确性
    private static AtomicInteger allScore = new AtomicInteger(0);

    public static void main(String[] args) throws InterruptedException {
        Candidate candidate = new Candidate();

        Thread[] threads = new Thread[1000];

        for (int i = 0; i < 1000; i++){
            threads[i] = new Thread(() -> {
                // 模拟投票过程随机
                if (Math.random() > 0.5){
                    scoreUpdater.incrementAndGet(candidate);
                    allScore.incrementAndGet();
                }
            });
        }

        Arrays.stream(threads).forEach(thread -> thread.start());

        for (int i =0 ; i < threads.length; i++){
            threads[i].join();
        }

        System.out.println("scoreUpdater:" + scoreUpdater.get(candidate));
        System.out.println("allScore:" +allScore.get());
    }
}

class Candidate{

    int id;

     volatile int score;
}
```

![](https://oscimg.oschina.net/oscnet/up-38be9d644b8863e160e1755f5364e798bef.png)

上述代码模拟了这个场景，候选人的得票数量记录在Candidate.score中，注意它是一个普通的volatile 变量，而volatile 变量并不会保证线程安全性，只会保证内存可见性和禁止重排序，代码中通过Math.random()来模拟随机投票过程，&nbsp;AtomicInteger allScore = new AtomicInteger(0);这一行代码用来验证&nbsp;AtomicIntegerFieldUpdater计算的正确性， 运行这段程序会发现 allScore的值总是和scoreUpdater的值相等。

**AtomicIntegerFieldUpdater使用注意事项：**

1. Updater只能修改它可见范围内的变量，因为Updater使用反射获得这个变量，如果变量不可见就会报错，比如score声明为private的，就不行。
2. 为了保证变量在多线程环境被正确的读取，它必须是volatile修饰的，如果不修饰也会报错。
1. 由于CAS操作通过对象实例中的偏移量直接进行赋值，因此它不支持static字段（*U*.objectFieldOffset(field)不支持静态变量），如果被static修饰了也会报错

# 14、Unsafe
## 14.1、获取unsafe
```
package 自旋锁计数;

import sun.misc.Unsafe;

import java.lang.reflect.Field;

/**
 * @author calebzhao<9 3 9 3 4 7 5 0 7 @ qq.com>
 * 2019/8/26 20:17
 */
public class UnsafeUtil
{
    public static Unsafe getUnsafe(){
        try {
            Field field = Unsafe.class.getDeclaredField("theUnsafe");
            field.setAccessible(true);
            
            // 这里是null是因为theUnsafe属性是static静态属性
            return (Unsafe) field.get(null);
        } catch (NoSuchFieldException | IllegalAccessException e) {
            throw new RuntimeException();
        }
    }
}
```

15、几种Couter计数的性能比较
```
package 自旋锁计数;

import sun.misc.Unsafe;

import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.atomic.AtomicInteger;

public class CounterDemo {

    private  final int THREAD_COUNT = 1000;

    public static void main(String[] args) {
        CounterDemo counterDemo =  new CounterDemo();

        counterDemo.atomicCouterTest();

        counterDemo.casCouterTest();

        counterDemo.synchronizedCouterTest();

        counterDemo.badCouterTest();
    }

    public void atomicCouterTest(){
        try{
            AtomicIntegerCouter atomicIntegerCouter = new AtomicIntegerCouter();
            CouterThread couterThread = new CouterThread(atomicIntegerCouter);
            List<thread> threadList = new ArrayList<>();
            for (int j = 0; j < THREAD_COUNT; j++){
                Thread thread = new Thread(couterThread);
                threadList.add(thread);
            }

            long start = System.currentTimeMillis();
            threadList.stream().forEach(thread -> thread.start());

            threadList.forEach(thread -> {
                try {
                    thread.join();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            });

            long time = System.currentTimeMillis() - start;

            System.out.println("atomicIntegerCouter: " + atomicIntegerCouter.getValue() + " 耗时： " + time);
        }
        catch (Exception e){
            e.printStackTrace();
        }
    }

    public void casCouterTest(){
        try{
            CasCounter casCounter = new CasCounter();
            CouterThread couterThread = new CouterThread(casCounter);
            List<thread> threadList = new ArrayList<>();
            for (int j = 0; j < THREAD_COUNT; j++){
                Thread thread = new Thread(couterThread);
                threadList.add(thread);
            }

            long start = System.currentTimeMillis();
            threadList.stream().forEach(thread -> thread.start());

            threadList.forEach(thread -> {
                try {
                    thread.join();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            });

            long time = System.currentTimeMillis() - start;
            System.out.println("casCounter: " + casCounter.getValue() + " 耗时： " + time);
        }
        catch (Error e){
            e.printStackTrace();
        }
    }

    public void synchronizedCouterTest(){
        try{
            SynchronizedCouter synchronizedCouter = new SynchronizedCouter();
            CouterThread couterThread = new CouterThread(synchronizedCouter);
            List<thread> threadList = new ArrayList<>();
            for (int j = 0; j < THREAD_COUNT; j++){
                Thread thread = new Thread(couterThread);
                threadList.add(thread);
            }

            long start = System.currentTimeMillis();
            threadList.stream().forEach(thread -> thread.start());

            threadList.forEach(thread -> {
                try {
                    thread.join();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            });

            long time = System.currentTimeMillis() - start;

            System.out.println("synchronizedCounter: " + synchronizedCouter.getValue() + " 耗时： " + time);
        }
        catch (Exception e){
            e.printStackTrace();
        }
    }

    public void badCouterTest(){
        try{
            BadCouter badCouter = new BadCouter();
            CouterThread couterThread = new CouterThread(badCouter);
            List<thread> threadList = new ArrayList<>();
            for (int j = 0; j < THREAD_COUNT; j++){
                Thread thread = new Thread(couterThread);
                threadList.add(thread);
            }

            long start = System.currentTimeMillis();
            threadList.stream().forEach(thread -> thread.start());

            threadList.forEach(thread -> {
                try {
                    thread.join();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            });

            long time = System.currentTimeMillis() - start;
            System.out.println("badCouter: " + badCouter.getValue() + " 耗时： " + time);
        }
        catch (Exception e){
            e.printStackTrace();
        }
    }

}

class CouterThread implements Runnable{

    private Counter counter;

    public CouterThread(Counter counter){
        this.counter = counter;
    }

    @Override
    public void run() {
        try {
            Thread.sleep(100);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        counter.incrementAndGet();
    }
}

interface Counter{

    int incrementAndGet();

    int getAndIncrement();

    int getValue();
}

class AtomicIntegerCouter implements Counter{
    /**
     * 线程安全方式计数， 内部的value子段声明为 private volatile int value;所以保证了内存可见性
     */
    private AtomicInteger atomic = new AtomicInteger(0);

    @Override
    public int incrementAndGet(){

        while (true){
            // 1、多核处理器可能会同时运行到这行代码，单核处理器由于时间片分配算法T1执行到这行代码后CPU执行权被T2获取了， 线程T1、T2均通过get()方法返回0
            // 2、假如T1先执行atomic.compareAndSet(currentValue, ++currentValue)这行代码，
            //    由于currentValue和atomic的值一致，cas操作成功，atomic变成1，退出循环,
            // 3、然后T2继续执行atomic.compareAndSet(currentValue, ++currentValue);
            //    这行代码会发现atomic内部维护的value值1已经与currentValue的值0不相等，不会进行设置值操作
            //    T2继续下次循环, 又执行atomic.get();获取到的currentValue为1， 再次执行compareAndSet时，
            //    atomic为1和currentValue为1相等，成功进行cas操作，然后退出循环
            int currentValue = atomic.get();
            boolean success = atomic.compareAndSet(currentValue, ++currentValue);
            if (success){
                return atomic.get();
            }
        }
    }

    @Override
    public int getAndIncrement() {
        while (true){
            int currentValue = atomic.get();
            boolean success = atomic.compareAndSet(currentValue, ++currentValue);
            if (success){
                return currentValue;
            }
        }
    }

    @Override
    public int getValue() {
        return atomic.get();
    }
}

class CasCounter implements Counter{

    private volatile int count = 0;

    private static final Unsafe unsafe = UnsafeUtil.getUnsafe();

    private static long valueOffset;

    static {
        try {
            valueOffset = unsafe.objectFieldOffset(CasCounter.class.getDeclaredField("count"));
        }
        catch (NoSuchFieldException e) {
            throw new Error();
        }
    }

    @Override
    public int incrementAndGet() {
        while (true){
            boolean success = unsafe.compareAndSwapInt(this, valueOffset, count, count + 1);
            if (success){
                return count;
            }
        }
    }

    @Override
    public int getAndIncrement() {
        while (true){
            boolean success = unsafe.compareAndSwapInt(this, valueOffset, count, count + 1);
            if (success){
                return count;
            }
        }
    }

    @Override
    public int getValue() {
        return count;
    }
}

class BadCouter implements Counter{
    private volatile int count = 0;

    @Override
    public int incrementAndGet() {
        return ++count;
    }

    @Override
    public int getAndIncrement() {
        return count++;
    }

    @Override
    public int getValue() {
        return count;
    }
}

class SynchronizedCouter implements Counter{
    private volatile int count = 0;

    @Override
    public synchronized int incrementAndGet() {
        return ++count;
    }

    @Override
    public synchronized int getAndIncrement() {
        return count++;
    }

    @Override
    public int getValue() {
        return count;
    }
}
```
![](https://oscimg.oschina.net/oscnet/up-8dab2724597e9856028fb66370a4020241a.png)

多次运行示例代码，会发现基本上每次synchronizedCouter的耗时最短，这也说明了synchronized使用锁的方式性能并不一定低。

# 16、JUC
## 16.1、CountDownLatch
### 16.1.1、介绍
CountDownLatch简单理解为倒计数器，这个类通常用来控制线程等待，它可以让一个线程等待直到倒计数器的值为0，再开始执行

调用await()方法会调用调用线程被阻塞，直到CountDownLatch的值为0才会继续往后执行

调用countdown()方法会使计数器的值减1.
### 16.1.2、入门示例
```java
package countdown;

import java.util.stream.IntStream;

/**
 * @author calebzhao&lt;9 3 9 3 4 7 5 0 7 @ qq.com&gt;
 * 2019/8/7 20:28
 */
public class MyCountDownLatchDemo {

    private static MyCountDownLatch countDownLatch = new MyCountDownLatch(4);

    public static void main(String[] args) throws InterruptedException {
        IntStream.of(1,2,3,4).forEach((i) -&gt; {
            new Thread(() -&gt; {
                try {
                    Thread.sleep(300);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println(Thread.currentThread().getName() + " finished");

                countDownLatch.countDown();
            }, "Thread-" + i).start();
        });

        countDownLatch.await();
        System.out.println("main finished");
    }
}
```

![](https://oscimg.oschina.net/oscnet/up-e1b6773ada7f6283081e27fdb69bcb8f874.png)

main线程的输出一定会等到所有线程执行完毕后才会执行（类似thread.join()的作用）

## 16.2、CyclicBarrer
### 16.2.1、简介
CyclicBarrer的作用和```CountDownLatch```非常类似，它也可以**实现线程间的计数等待**，但它的作用比CountDownLatch更加强大和复杂。  
    
```CyclicBarrier ```的字面意思是可循环使用（Cyclic）的屏障（Barrier）。它要做的事情是，**让一组线程到达一个屏障（也可以叫同步点）时被阻塞，直到最后一个线程到达屏障时，屏障才会开门，所有被屏障拦截的线程才会继续干活**。CyclicBarrier默认的构造方法是CyclicBarrier(int parties)，其参数表示屏障拦截的线程数量，每个线程调用await方法告诉CyclicBarrier我已经到达了屏障，然后当前线程被阻塞。
    
```CyclicBarrer```可以理解为循环栅栏，栅栏就是一种障碍物，比如通常在私人府邸的周围围上一圈栅栏，阻止闲杂人等入内。这里当然是用来阻止线程继续执行，要求线程在栅栏外等待。前面Cycli意为循环，也就是说这个计数器可以反复使用，比如我们将计数器的值设置10，那么当第一批执行线程报道(调用await()方法)后，计数就会归零，接着下一批执行线程报道，这是Cycli的含义。

### 16.2.2、入门示例
```
package cyclicBarrier;

import java.util.concurrent.BrokenBarrierException;
import java.util.concurrent.CyclicBarrier;

/**
 * @author calebzhao&lt;9 3 9 3 4 7 5 0 7 @ qq.com&gt;
 * 2019/8/27 22:18
 */
public class CycliBarrerDemo2 {

    public static void main(String[] args) throws BrokenBarrierException, InterruptedException {
        CyclicBarrier cyclicBarrier = new CyclicBarrier(2);

        new Thread(){
            @Override
            public void run() {
                try {
                    cyclicBarrier.await();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } catch (BrokenBarrierException e) {
                    e.printStackTrace();
                }

                System.out.println(1);
            }
        }.start();

        cyclicBarrier.await();
        System.out.println(2);
    }
}
```


![](https://oscimg.oschina.net/oscnet/up-d8dd855c66b695a99895493fdad3e1bcd66.png)
![](https://oscimg.oschina.net/oscnet/up-54fbf8910c445c6eb6927df7386519b09ed.png)

上面的示例有时输出12，有时输出21,

如果把new CyclicBarrier(2)修改成new CyclicBarrier(3)则主线程和子线程会永远等待，因为没有第三个线程执行await方法，即没有第三个线程到达屏障，所以之前到达屏障的两个线程都不会继续执行。

### 16.2.3、CyclicBarrier的构造函数，执行完成hook
```CyclicBarrier```还提供一个更高级的构造函数```CyclicBarrier(int parties, Runnable barrierAction)```，用于在线程到达屏障时，优先执行barrierAction，方便处理更复杂的业务场景。代码如下：

```java
package cyclicBarrier;

import java.util.concurrent.BrokenBarrierException;
import java.util.concurrent.CyclicBarrier;

/**
 * @author calebzhao&lt;9 3 9 3 4 7 5 0 7 @ qq.com&gt;
 * 2019/8/27 22:18
 */
public class CycliBarrerDemo2 {

    public static void main(String[] args) throws BrokenBarrierException, InterruptedException {
        CyclicBarrier cyclicBarrier = new CyclicBarrier(2, new BarrierAction());

        new Thread(){
            @Override
            public void run() {
                try {
                    cyclicBarrier.await();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } catch (BrokenBarrierException e) {
                    e.printStackTrace();
                }

                System.out.println(1);
            }
        }.start();

        cyclicBarrier.await();
        System.out.println(2);
    }

    static class BarrierAction implements Runnable{

        @Override
        public void run() {
            System.out.println("执行完毕");
        }
    }
}
```
![](https://oscimg.oschina.net/oscnet/up-1b1e9651615b35ee0f8bfa320d89e03b64f.png)

### 16.2.4、示例3
```java
package cyclicBarrier;

import java.util.concurrent.BrokenBarrierException;
import java.util.concurrent.CyclicBarrier;
import java.util.concurrent.TimeUnit;

/**
 * @author calebzhao&lt;9 3 9 3 4 7 5 0 7 @ qq.com&gt;
 * 2019/8/27 21:34
 */
public class CycliBarrierDemo {

    public static void main(String[] args) {
        CyclicBarrier cyclicBarrier = new CyclicBarrier(2);

        new Thread(){
            @Override
            public void run() {
                try {
                    TimeUnit.SECONDS.sleep(5);

                    System.out.println("t1 finished");
                    cyclicBarrier.await();

                    System.out.println("t1 the other finished too");
                }
                catch (InterruptedException | BrokenBarrierException e) {
                    e.printStackTrace();
                }
            }
        }.start();

        new Thread(){
            @Override
            public void run() {
                try {
                    TimeUnit.SECONDS.sleep(1);

                    System.out.println("t2 finished");
                    cyclicBarrier.await();
                    System.out.println("t2 the other finished too");
                }
                catch (InterruptedException | BrokenBarrierException e) {
                    e.printStackTrace();
                }
            }
        }.start();
        
          TimeUnit.SECONDS.sleep(2);
          System.out.println(cyclicBarrier.getNumberWaiting());
          System.out.println(cyclicBarrier.getParties());
          System.out.println(cyclicBarrier.isBroken());
          
          TimeUnit.SECONDS.sleep(4);
          System.out.println(cyclicBarrier.getNumberWaiting());
          System.out.println(cyclicBarrier.getParties());
          System.out.println(cyclicBarrier.isBroken());

    }
}
```
![](https://oscimg.oschina.net/oscnet/up-8f87f400c85078a0ce21bbfd40a502dfe2f.png)

### 16.2.5、士兵报道示例
考虑如下场景：司令下达命令，要求10个士兵一起去完成一项任务，这是会要求10个士兵先集合报道，接着，一起雄赳赳，气昂昂地去执行任务。当10个士兵把自己手头上的任务全部执行完了，那么司令才能对外宣布，任务完成。

```java
package cyclicBarrier;

import java.util.concurrent.BrokenBarrierException;
import java.util.concurrent.CyclicBarrier;
import java.util.concurrent.TimeUnit;

/**
 * @author calebzhao&lt;9 3 9 3 4 7 5 0 7 @ qq.com&gt;
 * 2019/8/27 23:17
 */
public class CyclicBarrierDemo4 {

    public static void main(String[] args) {
        CyclicBarrier cyclicBarrier = new CyclicBarrier(10, new BarrierAction());

        for (int i = 1; i &lt; 11; i++){
            new Thread(new Soldier("士兵" +i, cyclicBarrier)).start();
        }
    }

    static class BarrierAction implements Runnable{

        private boolean wokrFinshed;

        @Override
        public void run() {
            if (wokrFinshed){
                System.out.println("\n全部士兵完成工作\n");
            }
            else{
                System.out.println("\n全部士兵报道完成\n");
                wokrFinshed = true;
            }
        }
    }

    static class Soldier implements Runnable{

        private String soldier;

        private CyclicBarrier cyclicBarrier;

        public Soldier(String soldier, CyclicBarrier cyclicBarrier) {
            this.soldier = soldier;
            this.cyclicBarrier = cyclicBarrier;
        }

        @Override
        public void run() {
            try {
                //等待所有士兵报道完成
                System.out.println(soldier +"报道");
                cyclicBarrier.await();

                // 所有士兵报道后开始做真正的工作
                this.doWork();
                System.out.println(soldier +"完成工作");

                // 等待所有士兵完成工作
                cyclicBarrier.await();
                System.out.println(soldier +"等待其他士兵完成工作");
            }
            catch (InterruptedException | BrokenBarrierException e) {
                e.printStackTrace();
            }
        }

        public void doWork(){
            try {
                TimeUnit.MILLISECONDS.sleep(200);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}	
```

![](https://oscimg.oschina.net/oscnet/up-09d3395a35a86cc96f2f2d961def1b27be9.png)

### 16.2.5、CyclicBarrier的应用场景
CyclicBarrier可以用于多线程计算数据，最后合并计算结果的应用场景。比如我们用一个Excel保存了用户所有银行流水，每个Sheet保存一个帐户近一年的每笔银行流水，现在需要统计用户的日均银行流水，先用多线程处理每个sheet里的银行流水，都执行完之后，得到每个sheet的日均银行流水，最后，再用barrierAction用这些线程的计算结果，计算出整个Excel的日均银行流水。 

### 16.2.6、CyclicBarrier和CountDownLatch的区别
* CountDownLatch的计数器只能使用一次。而CyclicBarrier的计数器可以使用reset() 方法重置。所以CyclicBarrier能处理更为复杂的业务场景，比如如果计算发生错误，可以重置计数器，并让线程们重新执行一次。
* CyclicBarrier还提供其他有用的方法，比如getNumberWaiting方法可以获得CyclicBarrier阻塞的线程数量。isBroken方法用来知道**阻塞的线程是否被中断**。比如以下代码执行完之后会返回true
```java
import java.util.concurrent.BrokenBarrierException;
import java.util.concurrent.CyclicBarrier;

public class CyclicBarrierTest3 {

    static CyclicBarrier c = new CyclicBarrier(2);

    public static void main(String[] args) throws InterruptedException, BrokenBarrierException {
        Thread thread = new Thread(new Runnable() {

            @Override
            public void run() {
                try {
                    c.await();
                } catch (Exception e) {
                }
            }
        });
        thread.start();
        thread.interrupt();
        try {
            c.await();
        } catch (Exception e) {
            System.out.println(c.isBroken());
        }
    }
}
```

输出 true

## 16.3、Exchanger
### 16.3.1、介绍
```Exchanger```（交换者）是一个用于线程间协作的工具类。```Exchanger```用于进行线程间的数据交换。它提供一个同步点，在这个同步点，两个线程可以交换彼此的数据。这两个线程通过

exchange方法交换数据，如果第一个线程先执行```exchange()```方法，它会一直等待第二个线程也

执行exchange方法，当两个线程都到达同步点时，这两个线程就可以交换数据，将本线程生产

出来的数据传递给对方。

### 16.3.2、Exchanger的应用场景
Exchanger可以用于遗传算法，遗传算法里需要选出两个人作为交配对象，这时候会交换两人的数据，并使用交叉规则得出2个交配结果。Exchanger也可以用于校对工作，比如我们需要将纸制银行流水通过人工的方式录入成电子银行流水，为了避免错误，采用AB岗两人进行录入，录入到Excel之后，系统需要加载这两个Excel，并对两个Excel数据进行校对，看看是否录入一致，代码如代码清单8-8所示。

```java
package juc.exchanger;

import java.util.concurrent.Exchanger;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.TimeoutException;

public class ExchangerDemo {

    public static void main(String[] args) {
        Exchanger<string> exchanger = new Exchanger&lt;&gt;();

        new Thread(){
            @Override
            public void run() {
                System.out.println("银行流水A before exchange....");

                String value = null;
                try {
                    value = exchanger.exchange("1111111");
                    // value = exchanger.exchange("the value is from t1", 1, TimeUnit.SECONDS);
                } catch (Exception e) {
                    e.printStackTrace();
                }

                System.out.println("银行流水A，校验收到的银行流水B: " + value);
            }
        }.start();

        new Thread(){
            @Override
            public void run() {
                System.out.println("银行流水B before exchange....");

                try {
                    TimeUnit.SECONDS.sleep(2);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }

                String value = null;
                try {
                    value = exchanger.exchange("2222222");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }

                System.out.println("银行流水B，校验收到的银行流水A: " + value);
            }
        }.start();

        System.out.println("main done");
    }
}
```

![](https://oscimg.oschina.net/oscnet/up-162554e02d7b939b77d897a8ce15d6e8351.png)

### 16.3.3、exchage()超时控制
如果两个线程有一个没有执行exchange()方法，则会一直等待，如果担心有特殊情况发生，避免一直等待，**可以使用exchange（V x，longtimeout，TimeUnit unit）设置最大等待时长**。

当把银行流水A线程的exchage这一行代码改为如下超时参数，会导致A等待B超时，银行流水A线程抛出TimeoutException(因为银行流水B的exchage在2秒后才会执行，而银行流水A最多等待1秒)，而银行流水B线程在睡眠结束后执行exchage()方法后又等待银行流水A的数据，但是此时银行流水A线程早已结束，导致银行流水B线程由于一直交换不了数据而一直阻塞在exchage()方法这一行。

```
String value = exchanger.exchange("the value is from t1", 1, TimeUnit.SECONDS);
```
![](https://oscimg.oschina.net/oscnet/up-66adea3471e5101a8abfdee64bcc241dab8.png)

## 16.4、Semaphore
### 16.4.1、介绍
Semaphore的中文意思是信号、信号系统，在java中一般称为信号量，此类的主要作用是限制线程并发的数量，如果不限制线程并发的数据，则当有一大批任务要执行时（考虑Tomcat高并发请求的场景），如果不加以限制，为每个任务分配一个线程，那么CPU的资源（线程资源、CPU时间片）很快会被耗尽（系统资源被这些线程长期占据着，导致系统其他程序分配不到线程或时间片而无法执行或者非常卡顿），另外由于每个请求对应一个线程，导致操作系统中的线程数量非常大，而CPU时间片非常短，那么会导致CPU在成千上万个线程之间不断快速切换线程上下文，这样导致CPU的时间片大部分用于切换线程上下文了，而CPU时间片并没有用于线程实际的任务执行，这样就导致系统整体变得非常卡慢，另外由于系统中的线程已经耗尽，那么当有其他地方需要开启线程时，由于系统没有多余的可用线程，会导致其他程序，甚至是操作系统也挂掉（其实hystrix熔断的目的就是为了隔离线程池，控制并发量）。

Semaphore可以用于做流量控制，特别是公用资源有限的应用场景，比如数据库连接。假  如有一个需求，要读取几万个文件的数据，因为都是IO密集型任务，我们可以启动几十个线程并发地读取，但是如果读到内存后，还需要存储到数据库中，而数据库的连接数只有10个，这时我们必须控制只有10个线程同时获取数据库连接保存数据，否则会报错无法获取数据库接。这个时候，就可以使用Semaphore来做流量控制。

### 16.4.2、Semaphore实现同步锁
```java
package juc.semaphore;

import java.util.concurrent.Semaphore;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicReference;

/**
 * @author calebzhao&lt;9 3 9 3 4 7 5 0 7 @ qq.com&gt;
 * 2019/8/28 21:09
 */
public class SemaphoreDemo1 {

    public static void main(String[] args) throws InterruptedException {
        AtomicReference<integer> counter = new AtomicReference&lt;&gt;(0);
        SemaphoreLock lock = new SemaphoreLock();


        for (int i = 0; i &lt; 10; i++){
            new Thread("t" + i){
                @Override
                public void run() {
                    try{
                        System.out.println(Thread.currentThread().getName() + " is running");
                        lock.lock();
                        System.out.println(Thread.currentThread().getName() + " get lock");
                        try {
                            TimeUnit.SECONDS.sleep(2);
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                        }
                    }
                    finally {
                        lock.unLock();
                        System.out.println(Thread.currentThread().getName() + " release lock");
                    }
                }
            }.start();
        }
    }
}

class SemaphoreLock{
    private Semaphore semaphore = new Semaphore(1);

    private Thread current;

    public void lock(){
        try {
            semaphore.acquire();

            current = Thread.currentThread();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    public void unLock(){
        if (current == Thread.currentThread()){
            semaphore.release();
        }

    }
}
```

![](https://oscimg.oschina.net/oscnet/up-fc4e2d30bd448c42f4fc51894e38a321029.png)

从输出可以看到一个线程必须要等待另一个线程释放锁后，获取到了锁才能往下执行

### 16.4.3、构造方法permits参数作用
数permits的作用是设置许可的个数，前面已经使用过代码：

```
Semaphore semaphore = new Semaphore(1)
```
来进行程序的设计，使同一时间内最多只有1个线程可以获取到锁，执行lock()和unLock()之间的代码，其实还可以传入大于1的数值，代表同一时间内最多运行有x个线程可以执行accquire()和release()之间的代码。
同一时间内最多允许运行2个线程同时运行代码示例：

```java
package juc.semaphore;

import java.util.concurrent.Semaphore;
import java.util.concurrent.TimeUnit;

/**
 * @author calebzhao&lt;9 3 9 3 4 7 5 0 7 @ qq.com&gt;
 * 2019/8/28 21:09
 */
public class SemaphoreDemo2 {

    public static void main(String[] args) throws InterruptedException {
        Semaphore semaphore = new Semaphore(2);


        for (int i = 0; i &lt; 3; i++){
            new Thread("t" + i){
                @Override
                public void run() {
                    try{
                        System.out.println(Thread.currentThread().getName() + " is running");
                        semaphore.acquire();
                        System.out.println(Thread.currentThread().getName() + " get lock");
                        try {
                            TimeUnit.SECONDS.sleep(2);
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                        }
                    }
                    catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    finally {
                        semaphore.release();
                        System.out.println(Thread.currentThread().getName() + " release lock");
                    }
                }
            }.start();
        }
    }
}
```


![](https://oscimg.oschina.net/oscnet/up-f286050c05c8bf9e13fe1eccbc17e64e7f1.png)

注意：当Semaphore的构造方法的参数permits的值大于1时，该类并不能保证线程安全性，因为可能会出现多个线程共同访问示例变量，导致出现脏数据的情况。

### 16.4.4、acquire(permits)的作用
有参方法acquire(int permits)的作用是每调用1次此方法，就使用X个许可

```
package juc.semaphore;

import java.util.concurrent.Semaphore;
import java.util.concurrent.TimeUnit;

/**
 * @author calebzhao&lt;9 3 9 3 4 7 5 0 7 @ qq.com&gt;
 * 2019/8/28 21:09
 */
public class SemaphoreDemo3 {

    public static void main(String[] args) throws InterruptedException {
        Semaphore semaphore = new Semaphore(3);


        for (int i = 0; i &lt; 2; i++){
            new Thread("t" + i){
                @Override
                public void run() {
                    try{
                           System.out.println(Thread.currentThread().getName() + " is running");

                        semaphore.acquire(2);
                        System.out.println("\n" +
Thread.currentThread().getName() + " get lock" + " , availablePermits:" + semaphore.availablePermits());
                        System.out.println();
                        try {
                            TimeUnit.SECONDS.sleep(10);
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                        }
                    }
                    catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    finally {
                        semaphore.release(2);
                        System.out.println(Thread.currentThread().getName() + " release lock" + " , availablePermits:" + semaphore.availablePermits());

                    }
                }
            }.start();
        }
    }
}
```
![](https://oscimg.oschina.net/oscnet/up-2342902b27eea0ae7d0c04d32b4bd8d33aa.png)

从输出可以看到t0线程获取到锁了，然后一直不释放，虽然t0只用了2个许可，还剩余1个许可，但是t1任务得不到继续执行，必须要等到过了10秒钟t0释放锁后t1才能得到继续执行。说明acquire(int permits)方法每次调用**必须要获取到足够的许可才能继续往后执行**，线程只获取到部分许可并不能让其继续往后执行。

### 16.4.5、动态增加许可数量（release()）及drainPermits()
**每调用一次release(int permits)可增加x个许可，可一直重复调用**

- semaphore.availablePermits()获取当前可用的许可数量

- semaphore.drainPermits()返回当前可用的许可数量，**并将信号量的可用许可数量置为0**，后续调用acquire()就无法获取到许可（除非调用release()增加许可）

```java
package juc.semaphore;

import java.util.concurrent.Semaphore;

/**
 * @author calebzhao&lt;9 3 9 3 4 7 5 0 7 @ qq.com&gt;
 * 2019/8/28 22:23
 */
public class SemaphoreDemo4 {

    public static void main(String[] args) throws InterruptedException {
        Semaphore semaphore = new Semaphore(2);

        semaphore.acquire();
        semaphore.acquire();
        System.out.println("availablePermits:" +semaphore.availablePermits());

        semaphore.release();
        semaphore.release();
        semaphore.release();
        System.out.println("availablePermits:" +semaphore.availablePermits());

        semaphore.release(2);
        System.out.println("availablePermits:" +semaphore.availablePermits());
        System.out.println(semaphore.drainPermits() +", " + semaphore.availablePermits());

        semaphore.release();
        semaphore.release();
        System.out.println(semaphore.drainPermits() +", " + semaphore.availablePermits());
        semaphore.drainPermits();
        System.out.println( semaphore.availablePermits());
    }
}
```

![](https://oscimg.oschina.net/oscnet/up-2b3303d5f4b5909ea0f88c8c2cf3f77f3db.png)

**实验说明构造函数new Semaphore(2)的参数值2并不是最终的许可值，仅仅是初始的状态值，每调用一次release(int permits)可增加x个许可。**

### 16.4.6、acquireUninterruptibly()的作用
方法acquireUninterruptibly(）是使进入acquire()方法的线程，**不允许**被中断。

- 1、线程执行可以被中断

```java
package juc.semaphore;

import java.util.concurrent.Semaphore;
import java.util.concurrent.TimeUnit;

/**
 * @author calebzhao&lt;9 3 9 3 4 7 5 0 7 @ qq.com&gt;
 * 2019/8/28 22:57
 */
public class InterruptSemaphore {

    public static void main(String[] args) throws InterruptedException {
        Semaphore semaphore = new Semaphore(1);

        Thread t1 = new Thread(() -&gt; {
            try {
                System.out.println("t1 wait permit");
                semaphore.acquire();

                System.out.println("t1 get permit");
                for (int i = 0; i &lt; Integer.MAX_VALUE / 50; i++){
                    Math.random();
                }

                semaphore.release();
                System.out.println("t1 release permit");
            }
            catch (Exception e) {
                System.out.print("t1 被中断");
                e.printStackTrace();
            }
        }, "t1");

        Thread t2 = new Thread(() -&gt; {
            try {
                System.out.println("t2 wait permit");
                semaphore.acquire();

                System.out.println("t2 get permit");
                for (int i = 0; i &lt; Integer.MAX_VALUE / 50; i++){
                    Math.random();
                }

                semaphore.release();
                System.out.println("t2 release permit");
            }
            catch (Exception e) {
                System.out.print("t2 被中断");
                e.printStackTrace();
            }
        }, "t2");

        t1.start();
        t2.start();

        TimeUnit.MILLISECONDS.sleep(100);

        t2.interrupt();

        System.out.println("main中断了");

    }
}
```

![](https://oscimg.oschina.net/oscnet/up-ebf9398948833140f1220a8b098a8f84526.png)

线程t1、t2都处于运行中， t1获得许可， t2等待许可，t2线程调用interrupt()方法，导致t2的等待被中断。

- 2、线程执行不能被中断

```java
package juc.semaphore;

import java.util.concurrent.Semaphore;
import java.util.concurrent.TimeUnit;

/**
 * @author calebzhao&lt;9 3 9 3 4 7 5 0 7 @ qq.com&gt;
 * 2019/8/28 22:57
 */
public class UnInterruptSemaphore {

    public static void main(String[] args) throws InterruptedException {
        Semaphore semaphore = new Semaphore(1);

        Thread t1 = new Thread(() -&gt; {
            try {
                System.out.println("t1 wait permit");
                semaphore.acquire();

                System.out.println("t1 get permit");
                for (int i = 0; i &lt; Integer.MAX_VALUE / 50; i++){
                    Math.random();
                }

                semaphore.release();
                System.out.println("t1 release permit");
            }
            catch (Exception e) {
                System.out.print("t1 被中断");
                e.printStackTrace();
            }
        }, "t1");

        Thread t2 = new Thread(() -&gt; {
            try {
                System.out.println("t2 wait permit");
                semaphore.acquireUninterruptibly();

                System.out.println("t2 get permit");
                for (int i = 0; i &lt; Integer.MAX_VALUE / 50; i++){
                    Math.random();
                }

                semaphore.release();
                System.out.println("t2 release permit");
            }
            catch (Exception e) {
                System.out.print("t2 被中断");
                e.printStackTrace();
            }
        }, "t2");

        t1.start();
        t2.start();

        TimeUnit.MILLISECONDS.sleep(100);

        t2.interrupt();

        System.out.println("main中断了");

    }
}
```


![](https://oscimg.oschina.net/oscnet/up-2abc58686114903b3030dc75f1b4aa2ee55.png)

线程t2是调用semaphore.acquireUninterruptibly();获取许可，在线程t2上调用interrupt()方法并没有导致线程t2被中断进入catch(Exception e){}代码块.

acquireUninterruptibly()还有重载方法**acquireUninterruptibly(int permits)**; 此方法的作用时等待指定数量的许可时不能被中断，如果成功获取锁，则获得指定数量的许可。

### 16.4.7、getQueueLength()和hasQueueThreads()
getQueueLength()作用是获取当前正在等待获取许可的线程个数

hasQueueThread()作用是判断当前还有没有线程在等待获取许可。

这2个方法通常是用在判断当前有没有等待许可的线程信息时使用

```
package juc.semaphore;

import java.util.concurrent.Semaphore;
import java.util.concurrent.TimeUnit;

/**
 * @author calebzhao&lt;9 3 9 3 4 7 5 0 7 @ qq.com&gt;
 * 2019/8/29 20:06
 */
public class GetQueueLengthDemo {

    public static void main(String[] args) {
        Semaphore semaphore = new Semaphore(1);

        for (int i = 0; i &lt; 3; i++){
            new Thread(){
                @Override
                public void run() {
                    try {

                        semaphore.acquire();
                        System.out.println(Thread.currentThread().getName() +"获取到许可");
                        TimeUnit.SECONDS.sleep(1);
                        System.out.println("还有" + semaphore.getQueueLength() + "个线程在等待");
                        System.out.println("是否还有线程在等待" + semaphore.hasQueuedThreads());
                        semaphore.release();
                        System.out.println(Thread.currentThread().getName() +"释放许可\n");
                    }
                    catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }.start();
        }
    }
}
```

![](https://oscimg.oschina.net/oscnet/up-654bc52af76519285e7748c0ed9e97912c0.png)

### 16.4.8、获取许可的公平与非公平测试
有时候获取许可的顺序与线程启动的顺序有关，这时就要分为公平与非公平，new Semaphore(1, false);构造函数的第2个参数用于指定公平还是非公平获取许可， true表示公平获取许可，false表示非公平获取许可 。

**公平**：线程获得许可的顺序与线程启动的顺序有关，先启动的线程优先获得许可，但不代表100%优先获得许可，仅仅是在概率上得到保障。

**非公平**：线程获得许可的顺序与线程启动顺序无关，随机获取。

1、非公平获取许可

```
package juc.semaphore;

import java.util.concurrent.Semaphore;

/**
 * @author calebzhao&lt;9 3 9 3 4 7 5 0 7 @ qq.com&gt;
 * 2019/8/29 20:06
 */
public class FairDemo {

    static class WorkerThread implements Runnable{
        Semaphore semaphore = new Semaphore(1, false);

        @Override
        public void run() {
            try {
                System.out.println(Thread.currentThread().getName() +" is running");
                semaphore.acquire();
                System.out.println(Thread.currentThread().getName() +"获取到许可");

                semaphore.release();
            }
            catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }

    public static void main(String[] args) {
        WorkerThread workerThread = new WorkerThread();

        new Thread(workerThread).start();

        for (int i = 0; i &lt; 3; i++){
            new Thread(workerThread, "线程" + i).start();
        }
    }
}
```

![](https://oscimg.oschina.net/oscnet/up-957e56d103d9bc4f85ad7d4f2f890685b4f.png)

可以看到线程是最后启动的，但是却优于线程1优先获取到了许可。

2、公平获取许可

当把构造方法的第二个参数new Semaphore(1, true)**改为true**时就是代表公平获取锁，此时再运行上面程序输出如下：

![](https://oscimg.oschina.net/oscnet/up-fecc237ed70d8346b8018b352a657a8768e.png)

如果反复运行程序会发现绝大部分情况下，线程获取许可的顺序是按照器启动的顺序获取到许可的

### 16.4.9、tryAcquire()
tryAcquire() 以非阻塞方式尝试获取许可，如果获取不到则立即返回false

tryAcquire(int permits) 同上，只是一次尝试获取多个许可

tryAcquire(long timeout, TimeUnit unit) 在指定的时间内尝试获取1个许可，如果没到超时时间则等待获取许可，如果到达超时时间则立即返回true或false

tryAcquire(int permits, long timeout, TimeUnit unit)  同上，一次获取多个许可

## 16.5、ReentrantLock
### 16.5.1、介绍
在java多线程中可以使用synchronized关键字来实现线程同步互斥执行，但在JDK1.5中增加了ReentrantLock可达到同样的目的，并且在扩展功能上也更加强大，比如具有公平与非公平、嗅探锁定（tryLock()非阻塞）、多路通知(一个锁产生多个Condition)等功能，而且在使用上比synchonrized更加灵活。

### 16.5.2、线程间同步
```
package juc.lock;

import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.ReentrantLock;

/**
 * @author calebzhao&lt;9 3 9 3 4 7 5 0 7 @ qq.com&gt;
 * 2019/8/29 21:29
 */
public class ReentrantLockDemo {

    public static void main(String[] args) {
        ReentrantLock reentrantLock = new ReentrantLock(false);

        for (int  i = 0; i &lt; 5; i++){
            new Thread("thread-" + i){
                @Override
                public void run() {
                    try{
                        System.out.println(Thread.currentThread().getName() + " is running");
                        reentrantLock.lock();
                        System.out.println(Thread.currentThread().getName() + " get lock");
                        try {
                            TimeUnit.MILLISECONDS.sleep(200);
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                        }
                    }
                    finally {
                        reentrantLock.unlock();
                        System.out.println(Thread.currentThread().getName() + " release lock");
                    }
                }
            }.start();
        }
    }
}
```
![](https://oscimg.oschina.net/oscnet/up-c338c507497a92f02bae823a4bbc7a2f32f.png)

可以看到多个线程都在运行中，一个线程必须要等待另一个线程释放后获得了锁才能打印

### 16.5.2、公平与非公平获取锁、isFair()、isLock()
new ReentrantLock(true)的构造方法的参数可以控制是线程获取锁的顺序是公平的还是非公平的获取锁，

设置为true时表示使用公平锁，多个线程获得许可的顺序与线程启动的顺序有关，先启动的线程优先获得许可，但不代表100%优先获取到锁，只是从概率上保证
false表示非公平地获取锁，即多个线程随机获取锁
lock.isFair()返回是否使用的是公平锁 
返回true表示使用的是公平锁
返回false表示使用的是非公平锁

lock.isLock()返回是否有任意线程正持有锁

lock.isHeldByCurrentThread()返回当前线程是否正持有此锁定

```java
package juc.lock;

import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.ReentrantLock;

/**
 * @author calebzhao&lt;9 3 9 3 4 7 5 0 7 @ qq.com&gt;
 * 2019/8/30 21:41
 */
public class IsLockDemo {
    public static void main(String[] args) throws InterruptedException {
        ReentrantLock lock = new ReentrantLock();
        System.out.println("是否有任意线程获取此锁定：" + lock.isLocked());
        System.out.println("main是否持有此锁定: " + lock.isHeldByCurrentThread());

        lock.lock();
        System.out.println("是否有任意线程获取此锁定：" + lock.isLocked());
        System.out.println("main是否持有此锁定: " + lock.isHeldByCurrentThread());


        lock.unlock();

        new Thread(){
            @Override
            public void run() {
                System.out.println("t is running");
                lock.lock();
            }
        }.start();

        TimeUnit.SECONDS.sleep(1);
        System.out.println("main是否持有此锁定: " + lock.isHeldByCurrentThread());
        System.out.println("是否有任意线程获取此锁定：" + lock.isLocked());
    }
}
```

![](https://oscimg.oschina.net/oscnet/up-186951e406a0f74d8c222f3c3609349a7eb.png)

### 16.5.3、Condition
任意一个Java对象，都拥有一组监视器方法（定义在java.lang.Object上），主要包括wait()、   wait(long timeout)、notify()以及notifyAll()方法，这些方法与synchronized同步关键字配合，可以实现等待/通知模式。Condition接口也提供了类似Object的监视器方法，与Lock配合可以实现等待/通知模式,但是它比synchonrized的wait、notiify更加灵活，可以实现多路通知功能，也就是在一个Lock对象里创建多个Condition(即对象监视器)实例，线程对象可以注册在指定的Condition中，从而可以有选择的进行线程通知，在调度线程上更加灵活。
    
在使用notify() / notifyAll()方法进行通知时时，被通知的线程是由jvm随机选择的，但使用ReentrantLock可以实现前面介绍过的“选择性通知”，这个功能在Condition中默认提供的。
    
而synchronized相当于整个Lock对象中只有一个单一的Condition对象，所有线程都注册在它一个对象的身上。线程开始notifyAll时，会通知所有处于WAITING状态的线程，没有选择权，会出现相当大的效率问题。
    
注意：condition.signal() / signalAll()方法只是让调用await()方法的线程退出等待队列，调用signal的线程不释放锁，虽然await()方法所在线程被唤醒，但是由于获得锁，也不会得到执行（阻塞在获取锁的临界区处）。

- 示例1：

```java
package juc.lock;

import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.ReentrantLock;

/**
 * @author calebzhao&lt;9 3 9 3 4 7 5 0 7 @ qq.com&gt;
 * 2019/9/1 12:39
 */
public class ConditionLockDemo {

    public static void main(String[] args) throws InterruptedException {
        ReentrantLock lock = new ReentrantLock();
        Condition condition = lock.newCondition();

        Thread t1 = new Thread("t1"){
            @Override
            public void run() {
                lock.lock();
                try {
                    System.out.println("t1 await...");
                    condition.await();
                    System.out.println("t1 get lock");
                }
                catch (InterruptedException e) {
                    e.printStackTrace();
                }
                finally {
                    lock.unlock();
                    System.out.println("t1 unlock");
                }
            }
        };

        t1.start();


        TimeUnit.SECONDS.sleep(1);

        new Thread(){
            @Override
            public void run() {
                lock.lock();
                System.out.println("main locked");

                condition.signalAll();
                System.out.println("main signal");
//                lock.unlock();
            }
        }.start();

    }
}
```

![](https://oscimg.oschina.net/oscnet/up-b5637bd628069d90bc367cfcb223c185027.png)

可以看到虽然线程t2调用了signalAll()方法唤醒线程t1, 但是线程t1并没有从await()处唤醒继续向后执行

- 示例2、交替输出数字

```java
package juc.lock;

import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

/**
 * @author calebzhao&lt;9 3 9 3 4 7 5 0 7 @ qq.com&gt;
 * 2019/8/29 22:17
 */
public class ConditionDemo {

    static class ReadWrite {

        private Lock lock = new ReentrantLock(false);

        private Condition condition = lock.newCondition();

        private boolean hasIncrement = false;

        private int count = 0;

        public void read() {
            while (true){
                lock.lock();

                try{
                    while (!hasIncrement){
                        condition.await();
                    }

                    // 代表一些耗时的操作
                    TimeUnit.MILLISECONDS.sleep(100);
                    System.out.println(Thread.currentThread().getName()  + " read: " + count);

                    hasIncrement = false;
                    condition.signalAll();
                }
                catch (Exception e){
                    e.printStackTrace();
                }
                finally {
                    lock.unlock();

                }
            }


        }

        public void write(){
           while (true){
               lock.lock();
               try{
                   while (hasIncrement){
                       condition.await();
                   }

                   // 代表一些耗时的操作
                   TimeUnit.MILLISECONDS.sleep(100);
                   count++;
                   hasIncrement = true;
                   System.out.println(Thread.currentThread().getName()  + " write: " + count);
                   condition.signalAll();
               }
               catch (Exception e){
                   e.printStackTrace();
               }
               finally {
                   lock.unlock();
               }
           }
        }
    }

    public static void main(String[] args) {
        ReadWrite readWrite = new ReadWrite();

        for (int i = 0; i &lt; 50; i++){
            new Thread(){
                @Override
                public void run() {
                    readWrite.read();
                }
            }.start();
        }

        for (int i = 0; i &lt; 50; i++){
            new Thread(){
                @Override
                public void run() {
                    readWrite.write();
                }
            }.start();
        }
    }
}
```
![](https://oscimg.oschina.net/oscnet/up-7331f3f4ca30a6468f4ee8b40f4aee14e23.png)

- 3、多生产者、多消费者示例：锁实现有界阻塞队列

```java
package juc.lock;

import java.util.LinkedList;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicInteger;
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.ReentrantLock;

/**
 * @author calebzhao&lt;9 3 9 3 4 7 5 0 7 @ qq.com&gt;
 * 2019/8/29 22:54
 */
public class ProduerAndConsumerDemo {

    public static void main(String[] args) {
        ArrayBlockingContainer<integer> container = new ArrayBlockingContainer&lt;&gt;();

        Producer producer = new Producer(container);
        Consumer consumer = new Consumer(container);
        for (int i =0; i &lt; 5; i++){
            new Thread(producer).start();
            new Thread(consumer).start();
        }


    }

    static class ArrayBlockingContainer<e>{

        private ReentrantLock lock = new ReentrantLock();

        private Condition condition = lock.newCondition();

        private LinkedList<e> storage;

        private int capticy;

        public ArrayBlockingContainer(){
            this(16);
        }

        public ArrayBlockingContainer(int capticy){
            this.storage = new LinkedList&lt;&gt;();

            this.capticy = capticy;
        }

        public void brpush(E value){
            lock.lock();
            try{
                while (this.isOverflow()){
                    condition.await();
                }

                this.storage.addLast(value);
                System.out.println(Thread.currentThread().getName() + " brpush value: " + value);
                condition.signalAll();
            }
            catch (Exception e){

            }
            finally {
                lock.unlock();
            }
        }

        public E blpop(){
            lock.lock();

            try{
                while (this.isEmpty()){
                    condition.await();
                }

                E value = this.storage.removeFirst();
                condition.signalAll();
                System.out.println(Thread.currentThread().getName() + " blpop value: " +value);
                return value;
            }
            catch (InterruptedException e) {
                e.printStackTrace();
                return null;
            }
            finally {
                lock.unlock();
            }

        }

        public int size(){
            return this.storage.size();
        }

        public boolean isOverflow(){
            return size() &gt;= capticy;
        }

        public boolean isEmpty(){
            return this.storage.isEmpty();
        }
    }

    static class Producer implements Runnable{
        ArrayBlockingContainer container;

        private AtomicInteger count = new AtomicInteger(0);

        public Producer(ArrayBlockingContainer container){
            this.container = container;
        }

        @Override
        public void run() {
            while (true){
                try {
                    TimeUnit.SECONDS.sleep(2);
                    int value = count.incrementAndGet();
                    this.container.brpush(value);

                }
                catch (InterruptedException e) {
                    e.printStackTrace();
                }

            }
        }
    }

    static class Consumer implements Runnable{
        ArrayBlockingContainer container;

        public Consumer(ArrayBlockingContainer container){
            this.container = container;
        }

        @Override
        public void run() {
            while (true){
                try {
                    TimeUnit.SECONDS.sleep(1);
                    Object value = this.container.blpop();

                }
                catch (InterruptedException e) {
                    e.printStackTrace();
                }

            }
        }
    }
}
```

![](https://oscimg.oschina.net/oscnet/up-eb7ae5c1e0f1a503c8ccb0aa864ce5d26b9.png)

从输出可以看到读写是有顺序的（pop的一定是之前被push进去的，读出来的没有出现重复的），但是读读、写写是没有顺序的，因为只保证了容器的读写正确性，并没有线程保证线程读写的有序性。

### 16.5.4、getHolderCount()
查询当前线程持有此锁定的次数，也就是lock()方法被调用的次数

```
package juc.lock;

import java.util.concurrent.locks.ReentrantLock;

/**
 * @author calebzhao&lt;9 3 9 3 4 7 5 0 7 @ qq.com&gt;
 * 2019/8/30 20:00
 */
public class ReentrantLockHolderDemo {

    public static void main(String[] args) {
        ReentrantLock lock = new ReentrantLock();

        lock.lock();
        System.out.println(lock.getHoldCount());
        lock.lock();
        System.out.println(lock.getHoldCount());
        
        lock.unlock();
        System.out.println(lock.getHoldCount());
        lock.unlock();
        System.out.println(lock.getHoldCount());
    }
}
```
![](https://oscimg.oschina.net/oscnet/up-3f5df07bd8cd8bf6f738a2d788ceb307143.png)

### 16.5.5、getQueuedLength()、hasQueueuedThreads()
lock.getQueuedLength() 返回当前正在等待获取此锁定的线程的个数

lock.hasQueueuedThreads() 返回当前是否还有线程等待获取此锁定

lock.hasQueuedThread(Thread t) 返回指定的线程t是否正在等待获取此锁定

```
package juc.lock;

import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.ReentrantLock;
/**
 * @author calebzhao&lt;9 3 9 3 4 7 5 0 7 @ qq.com&gt;
 * 2019/8/30 20:03
 */
public class GetQueueLegth {

    public static void main(String[] args) throws InterruptedException {
        ReentrantLock lock = new ReentrantLock();
        Thread[] threads = new Thread[5];
        for (int  i = 0; i &lt; 3; i++){
            threads[i] = new Thread(){
                @Override
                public void run() {
                    System.out.println(Thread.currentThread().getName() + " is running");
                    lock.lock();
                    System.out.println(Thread.currentThread().getName() +" get lock");
                    try {
                        TimeUnit.SECONDS.sleep(3);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    System.out.println(Thread.currentThread().getName() +" unlock");
                    lock.unlock();
                }
            };
            threads[i].start();
        }

        TimeUnit.SECONDS.sleep(2);
        System.out.println("Thread-0 wait lock: " +lock.hasQueuedThread(threads[0]));
        System.out.println("Thread-1 wait lock: " + lock.hasQueuedThread(threads[1]));
        System.out.println("lock wait: " + lock.hasQueuedThreads());

        TimeUnit.SECONDS.sleep(10);
        System.out.println("lock wait: " + lock.hasQueuedThreads());
    }

}
```
![](https://oscimg.oschina.net/oscnet/up-369ff3a523f8f0461d47dff48b2da2fe42f.png)

### 16.5.6、getWaitQueueLength(Condition c)、hasWaiters(Condition c)
* lock.getWaitQueueLength(Condition c)：

返回在给定Condition上等待的线程估计数（即有多少个线程在这个Condition上正处于WAITING等待状态），比如3个线程在同一个Condition上都调用了await()方法，此时另一个线程获取锁后，再调用getWaitQueueLength(Condition condition)则返回3。

- hasWaiters(Condition c) 
返回是否有线程正在此Condition上处于WAITING等待状态

```java
package juc.lock;

import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.ReentrantLock;

/**
 * @author calebzhao&lt;9 3 9 3 4 7 5 0 7 @ qq.com&gt;
 * 2019/8/30 20:24
 */
public class GetWaitQueueLengt {

    static class ServiceRunnable {
        private ReentrantLock lock = new ReentrantLock();
        private Condition condition = lock.newCondition();


        public void waitMethod(){
            System.out.println(Thread.currentThread().getName() + " is running");
            lock.lock();
            try {
                System.out.println(Thread.currentThread().getName() + " await");
                condition.await();
                System.out.println(Thread.currentThread().getName() + " 被唤醒");
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            finally {
                System.out.println(Thread.currentThread().getName() + " unlock");
                lock.unlock();

            }
        }

        public void notifyMethod(){
            System.out.println("\nnotifyMethod is running");
            lock.lock();
            System.out.println("has waiter:" + lock.hasWaiters(condition)  + ", wait length:" + lock.getWaitQueueLength(condition));
            condition.signal();
            System.out.println("notifyMethod signalAll");
            lock.unlock();
            System.out.println("notifyMethod unlock\n");
        }
    }



    public static void main(String[] args) throws InterruptedException {
        ServiceRunnable serviceRunnable = new ServiceRunnable();

        for (int i = 0; i &lt; 3; i++){
            new Thread(){
                @Override
                public void run() {
                    serviceRunnable.waitMethod();
                }
            }.start();
        }

        // 确保所有线程都已经处于运行中
        Thread.sleep(1000);
        serviceRunnable.notifyMethod();

        Thread.sleep(3000);
        serviceRunnable.notifyMethod();
    }
}
```
![](https://oscimg.oschina.net/oscnet/up-9cb2fc1a46847da1311bf9c219ea15efc00.png)

**第一次调用**
```
// 确保所有线程都已经处于运行中
Thread.sleep(1000);
serviceRunnable.notifyMethod();
```
时由于有3个线程在这个Condition上正处于WAITING状态，所以getWaitQueueLength(Condition c)返回数字3，此时由于notifyMethod()中调用了一次signal(),所以会导致3个线程中的某个线程被唤醒（这里是Thread-0被唤醒）

**第二次调用**

```
Thread.sleep(3000);
serviceRunnable.notifyMethod();
```
时，由于之前Thread-0已经被唤醒，此时在Conition上只有2个线程处于WAITING状态，所以notifyMethod()方法中getWaitQueueLength(Condition c)返回数字2，此时由于notifyMethod()中调用了一次signal(),所以会导致2个线程中的某个线程被唤醒（这里是Thread-2被唤醒），此时Thread-1还处于WAITING状态，但是由于没有被signal()通知，所以导致Thread-1一直处于等待状态，导致看到的结果就是控制台一直不停止。

### 16.5.7、lock()、lockInterruptibly()
1. lockInterruptibly()的作用是当前线程等待获取锁定的过程中，如果被中断则抛出异常InterruptedException， lock()方法所在的线程等待获取锁定的过程中，如果被中断也不会抛出异常，而是正常等待直到获取锁定。
```java
package juc.lock;

import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.ReentrantLock;

/**
 * @author calebzhao&lt;9 3 9 3 4 7 5 0 7 @ qq.com&gt;
 * 2019/9/1 10:50
 */
public class LockInterruptDemo {

    public static void main(String[] args) throws InterruptedException {
        ReentrantLock lock = new ReentrantLock();

        Thread thread1 = new Thread("thread1"){
            @Override
            public void run() {
                System.out.println("thread1 is running");
                lock.lock();
                System.out.println("thread1 get lock");
                for (int i =0; i &lt; Integer.MAX_VALUE / 50; i ++){
                    Math.random();
                }
                lock.unlock();
                System.out.println("thread1 unlock");
            }
        };

        thread1.start();

        // 确保thread1先运行
        TimeUnit.MILLISECONDS.sleep(100);

        Thread thread2 = new Thread("thread2"){
            @Override
            public void run() {
                System.out.println("thread2 is running");
                // 即使被中断，也不抛出异常，继续执行
                lock.lock();
                System.out.println("thread2 get lock");
                for (int i =0; i &lt; Integer.MAX_VALUE / 50; i ++){
                    Math.random();
                }
                lock.unlock();
                System.out.println("thread2 unlock");
            }
        };

        thread2.start();

        thread2.interrupt();

        Thread thread3 = new Thread("thread3"){
            @Override
            public void run() {
                System.out.println("thread3 is running");
                try {
                    // 在等待锁的过程中，如果被中断则抛出异常
                    lock.lockInterruptibly();
                    &nbsp;System.out.println("thread3 get lock");

                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
               
                for (int i =0; i &lt; Integer.MAX_VALUE / 50; i ++){
                    Math.random();
                }
                // 由于线程被中断，抛出了异常，但是异常被捕获了，只是打印了，而这里打印出调用lock.unlock()
                // 抛出IllegalMonitorStateException，说明线程被中断没有获取到锁
                lock.unlock();
                System.out.println("thread3 unlock");
            }
        };

        thread3.start();
        thread3.interrupt();
    }
}
```

![](https://oscimg.oschina.net/oscnet/up-4f9afffe9e660c2766acb1acb9a06a79c7e.png)

从输出可以看到：

线程t1先启动，持有锁定还没有释放的时候，线程t2调用lock()方法等待获取锁定时，在main线程调用thread2.interrupt()方法并没有导致线程thread2抛出异常，而是等到线程thread1释放锁后线程thread2依然获得了锁，说明**thead.interrupt()方法并不能打断lock.lock()方法。
线程t3调用lock.lockInterruptibly();方法在等待获取锁定的过程中，在main线程中调用thread3.interrupt()方法，可以看到抛出了异常，但是由于异常被捕获了，所以导致thread3的代码继续往后执行，当执行到lock.unlock()方法时，抛出了IllegalMonitorStateException，说明thread3并没有获取到锁。

### 16.5.8、tryLock()
* tryLock()是lock()的非阻塞版本， tryLock()调用后会立即返回true或false，表示是否获取到锁
* tryLock(long timeout, TimeUnit unit)是tryLock()的非阻塞超时版本，可以指定在没有获取到锁时最多等待多长时间，如果到了指定超时时间还没有获取到锁则返回false, 如果在等待过程中线程被中断，则抛出异常。
```java
package juc.lock;

import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.ReentrantLock;

/**
 * @author calebzhao&lt;9 3 9 3 4 7 5 0 7 @ qq.com&gt;
 * 2019/9/1 11:34
 */
public class TryLockDemo {

    public static void main(String[] args) throws InterruptedException {
        ReentrantLock lock = new ReentrantLock();

        Thread thread1 = new Thread("thread1"){
            @Override
            public void run() {
                System.out.println("thread1 is running");
                lock.lock();
                System.out.println("thread1 get lock");
                try {
                    TimeUnit.SECONDS.sleep(2);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                lock.unlock();
                System.out.println("thread1 unlock");
            }
        };

        thread1.start();

        // 确保thread1先运行,已经持有锁
        TimeUnit.MILLISECONDS.sleep(100);

        Thread thread2 = new Thread("thread2"){
            @Override
            public void run() {
                System.out.println("thread2 is running");
                if (lock.tryLock()){
                    System.out.println("thread2 get lock");
                    for (int i =0; i &lt; Integer.MAX_VALUE / 50; i ++){
                        Math.random();
                    }
                    lock.unlock();
                    System.out.println("thread2 unlock");
                }
                else{
                    System.out.println("thread2 没有获取到锁");
                }

            }
        };

        thread2.start();

        Thread thread3 = new Thread("thread3"){
            @Override
            public void run() {
                System.out.println("thread3 is running");
                try {
                    if (lock.tryLock(3, TimeUnit.SECONDS)){
                        System.out.println("thread3 get lock");
                        for (int i =0; i &lt; Integer.MAX_VALUE / 50; i ++){
                            Math.random();
                        }


                        lock.unlock();
                        System.out.println("thread3 unlock");
                    }
                    else{
                        System.out.println("thread3 等待3秒仍然没有获取到锁");
                    }
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        };

        thread3.start();

    }
}
```

![](https://oscimg.oschina.net/oscnet/up-9a6d2538f7ff1d70612ec6421ab3bf37074.png)

如果把线程thread3的lock.tryLock(3, TimeUnit.SECONDS)的超时时间改为1，则会导致thread3到了超时时间仍然获取不到锁，修改后的输出如下：

![](https://oscimg.oschina.net/oscnet/up-32fdd175414b64b89f9265bc219b5434eb1.png)

### 16.5.8、Contition.awaitUninterrupyibly()
Contition.await()方法等待过程中**可以**被thread.interrupt()中断

Contition.awaitUninterrupyibly()等待过程中**不能**被thread.interrupt()中断

```java
package juc.lock;

import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.ReentrantLock;

/**
 * @author calebzhao&lt;9 3 9 3 4 7 5 0 7 @ qq.com&gt;
 * 2019/9/1 12:39
 */
public class ConditionInterruptDemo {

    public static void main(String[] args) throws InterruptedException {
        ReentrantLock lock = new ReentrantLock();
        Condition condition = lock.newCondition();

        Thread t1 = new Thread("t1"){
            @Override
            public void run() {
                lock.lock();
                try {
                    System.out.println("t1 await...");
                    condition.await();
                }
                catch (InterruptedException e) {
                    System.out.println("t1被中断");
                    e.printStackTrace();
                }
                finally {
                    lock.unlock();
                    System.out.println("t1 unlock");
                }
            }
        };

        t1.start();
        t1.interrupt();

        Thread t2 = new Thread("t2"){
            @Override
            public void run() {
                System.out.println("t2 is running...");
                lock.lock();
                try {
                    System.out.println("t2 await...");
                    condition.awaitUninterruptibly();
                    System.out.println("t2 get lock");
                }
                catch (Exception e){
                    e.printStackTrace();
                }
                finally {
                    lock.unlock();
                    System.out.println("t2 unlock");
                }
            }
        };

        t2.start();
        t2.interrupt();

        TimeUnit.SECONDS.sleep(1);

        new Thread(){
            @Override
            public void run() {
                lock.lock();
                System.out.println("main locked");

                condition.signalAll();
                System.out.println("main signal");
                lock.unlock();
            }
        }.start();

    }
}
```

![](https://oscimg.oschina.net/oscnet/up-c8a9ccc09ebe912b14b3b44c38de0cb2245.png)

从输出可以看到：

1. Condition.await()可以被thread.interrupt()中断。
2. 线程t2调用awaitUninterruptibly()处于等待状态，然后线程在main线程中调用t2.interrupt()方法并没有导致线程t2被中断，当线程t3调用signalAll()并释放锁后，线程t2正常获得了锁。
### 16.5.9、condition.awaitUntil(Date deadlineDate)
condition.awaitUntil(Date deadlineDate) 是await()的等待超时版本，表示线程最多等待指定的时间，如果到了指定的超时时间则还没有被唤醒则立即返回false, 如果未到超时时间就被唤醒了则立即返回true

```java
package juc.lock;

import java.time.Instant;
import java.time.LocalDateTime;
import java.time.ZoneId;
import java.util.Date;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.ReentrantLock;

/**
 * @author calebzhao&lt;9 3 9 3 4 7 5 0 7 @ qq.com&gt;
 * 2019/9/1 13:26
 */
public class AwaitUntil {

    public static void main(String[] args) throws InterruptedException {
        ReentrantLock lock = new ReentrantLock();
        Condition condition = lock.newCondition();

        Thread t1 = new Thread("t1"){
            @Override
            public void run() {
                lock.lock();
                try {
                    System.out.println("t1 await...");
                    LocalDateTime deadlineDate = LocalDateTime.now().plusSeconds(1);
                    Instant instant = deadlineDate.atZone(ZoneId.systemDefault()).toInstant();
                    boolean success = condition.awaitUntil(Date.from(instant));
                    if (success){
                        System.out.println("t1 do something...");
                    }
                    else{
                        System.out.println("t1 wait timeout");
                    }

                }
                catch (InterruptedException e) {
                    System.out.println("t1被中断");
                    e.printStackTrace();
                }
                finally {
                    lock.unlock();
                    System.out.println("t1 unlock");
                }
            }
        };

        t1.start();

        System.out.println("main sleep");
        TimeUnit.SECONDS.sleep(2);
        lock.lock();
        System.out.println("main before signal");
        condition.signal();
        lock.unlock();
        System.out.println("main signal");
    }
}
```
![](https://oscimg.oschina.net/oscnet/up-3f39b63b50f9baa9f01d287b3a159d70c9e.png)

### 16.6.1、可重入功能
ReentrantLock拥有锁重入的功能，也就是在使用lock()方法时，当一个线程获得一个锁，再次请求该锁时是可以获得该锁的。表现在一个lock()及unlock()方法 包围的代码块的内部调用本类的其他lock()及unlock()方法块时，是永远可以得到锁的。

```java
package juc.lock;

import java.util.concurrent.locks.ReentrantLock;

/**
 * @author calebzhao&lt;9 3 9 3 4 7 5 0 7 @ qq.com&gt;
 * 2019/9/1 15:49
 */
public class ReentrantDemo {

    ReentrantLock lock = new ReentrantLock();

    public void methodA(){
        lock.lock();
        System.out.println("methodA invoked ");

        this.methodB();

        lock.unlock();
    }

    public void methodB(){
        lock.lock();
        System.out.println("methodB invoked ");
        lock.unlock();
    }


    public static void main(String[] args) {
        ReentrantDemo demo = new ReentrantDemo();

        demo.methodA();
    }
}
```

![](https://oscimg.oschina.net/oscnet/up-c013940cb6ee6b6a11eb8c67ee7e9d7b757.png)

## 16.6、ReentrantReadWriteLock

**ReentrantReadWriteLock表示可重入的读写锁，他有2个锁，一个是读操作相关的共享锁，另一个是写相关的排他锁。多个读锁之间不互斥，读锁与写锁互斥，写锁与写锁互斥。**
    
在没有线程进行进行写入操作时，进行读取操作的多个Thread都可以获取到读锁，而进行写入操作的Thread只有在获取到写锁后才能进行写入操作，即多个Thread可以同时进行读操作，但是同一时刻只允许一个线程进行写操作。

### 16.6.1、读读共享
```java
package juc.lock.readWriteLock;

import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.ReentrantReadWriteLock;

/**
 * @author calebzhao&lt;9 3 9 3 4 7 5 0 7 @ qq.com&gt;
 * 2019/9/1 15:02
 */
public class ReadReadLockDemo {

    public static void main(String[] args) {
        ReentrantReadWriteLock lock = new ReentrantReadWriteLock();

        for (int i = 0; i &lt; 5; i++){
            new Thread(){
                @Override
                public void run() {
                    lock.readLock().lock();
                    System.out.println(Thread.currentThread().getName() + " read");
                    try {
                            TimeUnit.SECONDS.sleep(1);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }

                    lock.readLock().unlock();
                }
            }.start();
        }
    }
}
```

![](https://oscimg.oschina.net/oscnet/up-7f435a3360ae11a994eba3f004878d29ad3.png)

可以每个线程运行时获取到锁后，都睡眠了1秒钟，1秒钟后才释放锁，但是控制台的输出却是一瞬间全部输出完成的，说明多个线程之间并没有互斥执行（即读读共享）。

### 16.6.2、写写互斥
```java
package juc.lock.readWriteLock;

import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.ReentrantReadWriteLock;

/**
 * @author calebzhao&lt;9 3 9 3 4 7 5 0 7 @ qq.com&gt;
 * 2019/9/1 15:02
 */
public class WriteWriteLockDemo {

    public static void main(String[] args) {
        ReentrantReadWriteLock lock = new ReentrantReadWriteLock();

        for (int i = 0; i &lt; 5; i++){
            new Thread(){
                @Override
                public void run() {
                    System.out.println(Thread.currentThread().getName() + " wait write lock");
                    lock.writeLock().lock();
                    System.out.println(Thread.currentThread().getName() + " write");
                    try {
                            TimeUnit.SECONDS.sleep(1);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }

                    lock.writeLock().unlock();
                    System.out.println(Thread.currentThread().getName() + " release write lock");
                }
            }.start();
        }
    }
}
```

![](https://oscimg.oschina.net/oscnet/up-8385d953e8dd64fcb2a3970d5e9063d5d4a.png)

从输出可以看到多个线程几乎同时执行到线程内的第一行打印，等待获取锁，而多个线程之间互斥获取锁（必须要等待前一个线程释放了锁，下一个线程才能获取到锁）

### #16.6.3、读写互斥(先读后写)
```java
package juc.lock.readWriteLock;

import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.ReentrantReadWriteLock;

/**
 * @author calebzhao&lt;9 3 9 3 4 7 5 0 7 @ qq.com&gt;
 * 2019/9/1 15:02
 */
public class ReadWriteLockDemo {

    public static void main(String[] args) throws InterruptedException {
        ReentrantReadWriteLock lock = new ReentrantReadWriteLock();

        new Thread(){
            @Override
            public void run() {
                System.out.println(Thread.currentThread().getName() + " wait read lock");
                lock.readLock().lock();
                System.out.println(Thread.currentThread().getName() + " read");
                try {
                    TimeUnit.SECONDS.sleep(3);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }

                lock.readLock().unlock();
                System.out.println(Thread.currentThread().getName() + " release read lock");
            }
        }.start();

        // 确保读锁已经被占据
        TimeUnit.SECONDS.sleep(1);

        new Thread(){
            @Override
            public void run() {
                System.out.println(Thread.currentThread().getName() + " wait write lock");
                lock.writeLock().lock();
                System.out.println(Thread.currentThread().getName() + " write");
                try {
                    TimeUnit.SECONDS.sleep(1);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }

                lock.writeLock().unlock();
                System.out.println(Thread.currentThread().getName() + " release write lock");
            }
        }.start();
    }
}
```

![](https://oscimg.oschina.net/oscnet/up-07cde93123d6ddf2a26a7b14d20d03b8896.png)

### 16.6.4、写读互斥（先写后读）
```java
package juc.lock.readWriteLock;

import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.ReentrantReadWriteLock;

/**
 * @author calebzhao&lt;9 3 9 3 4 7 5 0 7 @ qq.com&gt;
 * 2019/9/1 15:02
 */
public class WriteReadLockDemo {

    public static void main(String[] args) throws InterruptedException {
        ReentrantReadWriteLock lock = new ReentrantReadWriteLock();

        new Thread(){
            @Override
            public void run() {
                System.out.println(Thread.currentThread().getName() + " wait write lock");
                lock.writeLock().lock();
                System.out.println(Thread.currentThread().getName() + " write");
                try {
                    TimeUnit.SECONDS.sleep(3);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }

                lock.writeLock().unlock();
                System.out.println(Thread.currentThread().getName() + " release write lock");
            }
        }.start();

        // 确保写锁已经被占据
        TimeUnit.SECONDS.sleep(1);

        new Thread(){
            @Override
            public void run() {
                System.out.println(Thread.currentThread().getName() + " wait read lock");
                lock.readLock().lock();
                System.out.println(Thread.currentThread().getName() + " read");
                try {
                    TimeUnit.SECONDS.sleep(1);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }

                lock.readLock().unlock();
                System.out.println(Thread.currentThread().getName() + " release read lock");
            }
        }.start();
    }

}
```

![](https://oscimg.oschina.net/oscnet/up-b4fb138b10a6100aa06b095adc94e8a56d6.png)

## 16.7、StampedLock
### 16.7.1、介绍
StampedLock类，在JDK1.8时引入，是对读写锁ReentrantReadWriteLock的增强，该类提供了一些功能，优化了读锁、写锁的访问，同时使读写锁之间可以互相转换，更细粒度控制并发。

首先明确下，该类的设计初衷是作为一个内部工具类，用于辅助开发其它线程安全组件，用得好，该类可以提升系统性能，用不好，容易产生死锁和其它莫名其妙的问题。

**先来看下，为什么有了ReentrantReadWriteLock，还要引入StampedLock？**

ReentrantReadWriteLock使得多个读线程同时持有读锁（只要写锁未被占用），而写锁是独占的。

但是，读写锁如果使用不当，很容易产生“饥饿”问题：

比如在读线程非常多，写线程很少的情况下，很容易导致写线程“饥饿”，虽然使用“公平”策略可以一定程度上缓解这个问题，但是“公平”策略是以牺牲系统吞吐量为代价的。

### 16.7.2、 StampedLock的特点
StampedLock的主要特点概括一下，有以下几点：

1. 所有获取锁的方法，都返回一个邮戳（Stamp），Stamp为0表示获取失败，其余都表示成功；
2. 所有释放锁的方法，都需要一个邮戳（Stamp），这个Stamp必须是和成功获取锁时得到的Stamp一致；
3. StampedLock是**不可重入的**；（如果一个线程已经持有了写锁，再去获取写锁的话就会造成死锁）
4. StampedLock有三种访问模式：
①Reading（读模式）：功能和ReentrantReadWriteLock的读锁类似
②Writing（写模式）：功能和ReentrantReadWriteLock的写锁类似
③Optimistic reading（乐观读模式）：这是一种优化的读模式。
5. StampedLock支持读锁和写锁的相互转换
我们知道RRW中，当线程获取到写锁后，可以降级为读锁，但是读锁是不能直接升级为写锁的。
StampedLock提供了读锁和写锁相互转换的功能，使得该类支持更多的应用场景。
6. 无论写锁还是读锁，都不支持Conditon等待

### 16.7.3、oracle官方示例
```java
package juc.lock.stampedLock;

import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicInteger;
import java.util.concurrent.locks.StampedLock;

/**
 * @author calebzhao&lt;9 3 9 3 4 7 5 0 7 @ qq.com&gt;
 * 2019/9/1 16:34
 */
public class StampedLockDemo {

    static class Point{
        private StampedLock stampedLock = new StampedLock();

        private int x;
        private int y;

        public void write(int deltaX, int deltaY){
            long stamp = stampedLock.writeLock();
            this.x += deltaX;
            this.y += deltaY;

            System.out.println("write: (" + this.x +"," + this.y +") = (" + (this.x - deltaX) + " + " +deltaX+", " + (this.y - deltaY) + " + " + deltaY +")");
            stampedLock.unlockWrite(stamp);
        }

        public double read(){
            long stamp = stampedLock.tryOptimisticRead();
            int currentX = x;
            int currentY = y;
            // 判断这个时间戳的数据是否被修改过，如果发生过修改，则说明读取到的x和y不一致
            // 有可能读取完x后， y被另一个线程修改了，导致读取到的x和y不是成对的，出现脏数据，validate()返回true说明数据没有被另一个线程修改过，如果返回true说读取过程中，数据被另一个线程修改了，出现了数据不一致的情况
            if (!stampedLock.validate(stamp)){
                // 数据被修改了，升级锁为悲观锁
                stamp = stampedLock.readLock();
                currentX = x;
                currentY = y;
                stampedLock.unlockRead(stamp);
            }
      
            return Math.sqrt(currentX * currentX + currentY * currentY);
        }
    }

    public static void main(String[] args) {
        Point point = new Point();

        for (int  i = 0; i &lt; 100; i++){
            new Thread(){
                @Override
                public void run() {
                    while (true){
                        try {
                            TimeUnit.SECONDS.sleep(1);
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                        }
                        double value = point.read();
                        System.out.println("read: " +value);
                    }

                }
            }.start();
        }

        AtomicInteger counter = new AtomicInteger(0);

        for (int  i = 0; i &lt; 2; i++){
            new Thread(){
                @Override
                public void run() {
                    while (true){
                        int c = counter.incrementAndGet();
                        point.write(c, c);
                        try {
                            TimeUnit.SECONDS.sleep(1);
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                        }
                    }

                }
            }.start();
        }

    }

}
```

![](https://oscimg.oschina.net/oscnet/up-cdf01af501f36b6ebf577f646ca322acff8.png)

可以看到，上述示例最特殊的其实是**distanceFromOrigin**方法，这个方法中使用了“Optimistic reading”乐观读锁，使得读写可以并发执行，但是```Optimistic reading```的使用必须遵循以下模式：

```java
long stamp = lock.tryOptimisticRead();  // 非阻塞获取版本信息
copyVaraibale2ThreadMemory();           // 拷贝变量到线程本地堆栈
if(!lock.validate(stamp)){              // 校验
    long stamp = lock.readLock();       // 获取读锁
    try {
        copyVaraibale2ThreadMemory();   // 拷贝变量到线程本地堆栈
     } finally {
       lock.unlock(stamp);              // 释放悲观锁
    }

}
useThreadMemoryVarables();              // 使用线程本地堆栈里面的数据进行操作
```
### 16.7.4、实现原理

```
fg
```

