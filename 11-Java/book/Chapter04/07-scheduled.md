ScheduledThreadPoolExecutor
===

### 相比 Timer 优势

- Timer 基于绝对时间，而 ScheduledThreadPoolExecutor 支持相对时间。
- Timer使用单线程执行所有 TimerTask，如果某个 TimerTask 很耗时则会影响到其他 TimerTask 的执行；
而 ScheduledThreadPoolExecutor 可以构造一个固定大小的线程池来执行任务。
- Timer 不会捕获由 TimerTask 抛出的异常，有异常时 Timer 会终止，未执行完的 TimerTask 不再执行；
ScheduledThreadPoolExecutor 可 catch 异常，即使有异常，周期性时间也不会终止。


### 调用方法

```
schedule(Callable callable, long delay, TimeUnit unit);
延迟 delay 时间后执行一次 callable

scheduleAtFixedRate(Runnable runnable, long initialDelay, long period, TimeUnit unit);
延迟 initialDelay 时间后执行 runnable，按照 period 时间固定频率周期性调用
注意，period 时间包括 runnable 运行时间，
若 period 时间比 runnable 运行时间短，则 runnable 运行完毕后，立刻重复运行

scheduleWithFixedDelay(Runnable runnable, long initialDelay, long delay, TimeUnit unit);
延迟 initialDelay 时间后开始执行 runnable，并按照 period 延迟时间周期性调用
注意，在上次运行完后才会执行下一次，period 时间不包括 runnable 运行时间
```

### 全局线程方法参数

使用 shutdown() 方法终止线程池中的线程，是否立刻终止每个线程，由如下方法决定：

```
setContinueExistingPeriodicTasksAfterShutdownPolicy(boolean value)
针对 schedule 和 scheduleAtFixedRate
value 为 true，shutdown 之后已存在的延迟仍会执行，反之则不会执行，直接退出

2） setExecuteExistingDelayedTasksAfterShutdownPolicy(boolean value)
针对 scheduleWithFixedDelay
value 为 true，shutdown 之后已存在的延迟仍会执行，反之则不会执行，直接退出
```

### 示例

```java
public class Test {

    private static final int THREAD_NUMBER = 1;
    private static final int DELAY_TIME = 1000;
    private static final int PERIODIC_TIME = 2000;

    public static void main(String[] args) {
        ScheduledExecutorService service = new ScheduledThreadPoolExecutor(THREAD_NUMBER);
        BusinessTask task = new BusinessTask();
        // 1秒后开始执行任务，之后每隔2秒执行一次
        service.scheduleWithFixedDelay(task, DELAY_TIME, PERIODIC_TIME, TimeUnit.MILLISECONDS);
    }

    private static class BusinessTask implements Runnable {
        @Override
        public void run() {
            // 捕获所有异常，保证定时任务周期性执行
            try {
                System.out.println("begin: " + getTimes());
                // doBusiness
                System.out.println("end  : " + getTimes());
            } catch (Exception e) {
                // handle
            }
        }
    }

    private static String getTimes() {
        SimpleDateFormat format = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss E");
        Date date = new Date();
        date.setTime(System.currentTimeMillis());
        return format.format(date);
    }
}
```
