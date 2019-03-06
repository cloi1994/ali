```java

 public final synchronized void join(long millis)
    throws InterruptedException {
        long base = System.currentTimeMillis();
        long now = 0;
 
        if (millis < 0) {
            throw new IllegalArgumentException("timeout value is negative");
        }
        if (millis == 0) {             //如果时执行的join(0)
            while (isAlive()) {        //如果线程是运行状态，就会执行下面的等待
                wait(0);               
            }
        } else {                       //如果是执行的join(time)
            while (isAlive()) {               //如果线程时运行状态
                long delay = millis - now;    
                if (delay <= 0) {
                    break;
                }
                wait(delay);                 //等待delay时间后自动返回继续执行
                now = System.currentTimeMillis() - base;
            }
        }
 }

```

---------------------
isAlive()方法下面会做详细的介绍，先看wait(0)，wait(0)是什么意思呢，查看下面wait()方法源码，其实wait()方法就是调用了wait(0)方法实现的，wait(0)就是让其一直等待。到这里会发现，其实join方法本质就是利用上面的线程实例作为对象锁的原理，当线程终止时，会调用线程自身的notifyAll()方法，通知所有等待在该线程对象上的线程的特征。

首先执行了t3.start(),在t3线程里的run方法里执行了t2.join()，此时有两种情况，可能还没有执行t2.start()，t2处于初始状态，也有可能执行了t2.start(),t2处于运行时状态，所以看到这里就明白了，join源码中while(isAlive())，其实相当于while(this.isAlive())就相当于判断这里的t2是不是已经是不是运行状态（有没有调用start方法）。这么设计是为了防止t2线程没有运行，此时t3线程如果直接执行wait(0)方法，那么将会一直等待下去，造成代码卡死



