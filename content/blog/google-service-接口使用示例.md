---
title: Google Service 接口使用示例
date: 2024-09-27T07:30:52.934Z
---

Google Service 接口使用示例
 

```java
import com.google.common.util.concurrent.Service;

import java.time.Duration;
import java.util.concurrent.Executor;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.TimeoutException;
import java.util.concurrent.atomic.AtomicBoolean;
import java.util.concurrent.atomic.AtomicInteger;

public class ServiceDemo implements Service {
    private final AtomicBoolean running = new AtomicBoolean(false);
    private final AtomicInteger count = new AtomicInteger(0);
    private Thread workerThread;

    @Override
    public Service startAsync() {
        if (running.compareAndSet(false, true)) {
            workerThread = new Thread(() -> {
                while (running.get()) {
                    System.out.println("Count: " + count.incrementAndGet());
                    try {
                        Thread.sleep(1000);
                    } catch (InterruptedException e) {
                        Thread.currentThread().interrupt();
                        break;
                    }
                }
            });
            workerThread.start();
        }
        return this;
    }

    @Override
    public boolean isRunning() {
        return running.get();
    }

    @Override
    public State state() {
        if (running.get()) {
            return State.RUNNING;
        } else if (count.get() > 0) {
            return State.TERMINATED;
        } else {
            return State.NEW;
        }
    }

    @Override
    public Service stopAsync() {
        if (running.compareAndSet(true, false)) {
            workerThread.interrupt();
        }
        return this;
    }

    @Override
    public void awaitRunning() {
        while (!isRunning()) {
            Thread.onSpinWait();
        }
    }

    @Override
    public void awaitRunning(Duration timeout) throws TimeoutException {
        Service.super.awaitRunning(timeout);
    }

    @Override
    public void awaitRunning(long timeout, TimeUnit unit) throws TimeoutException {
        long endTime = System.currentTimeMillis() + unit.toMillis(timeout);
        while (!isRunning() && System.currentTimeMillis() < endTime) {
            Thread.onSpinWait();
        }
        if (!isRunning()) {
            throw new TimeoutException("Service did not start within the timeout period");
        }
    }

    @Override
    public void awaitTerminated() {
        try {
            if (workerThread != null) {
                workerThread.join();
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }

    @Override
    public void awaitTerminated(Duration timeout) throws TimeoutException {
        Service.super.awaitTerminated(timeout);
    }

    @Override
    public void awaitTerminated(long timeout, TimeUnit unit) throws TimeoutException {
        try {
            if (workerThread != null) {
                workerThread.join(unit.toMillis(timeout));
            }
            if (isRunning()) {
                throw new TimeoutException("Service did not terminate within the timeout period");
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new TimeoutException("Interrupted while waiting for service to terminate");
        }
    }

    @Override
    public Throwable failureCause() {
        return null; // 简化实现，不处理失败情况
    }

    @Override
    public void addListener(Listener listener, Executor executor) {
        // 简化实现，不处理监听器
    }

    public static void main(String[] args) throws Exception {
        ServiceDemo service = new ServiceDemo();
        
        System.out.println("Starting service...");
        service.startAsync().awaitRunning();
        System.out.println("Service started.");

        Thread.sleep(10000); // 运行10秒

        System.out.println("Stopping service...");
        service.stopAsync().awaitTerminated();
        System.out.println("Service stopped.");
    }
}
```

 

1. 使用 `AtomicBoolean` 来安全地控制服务的运行状态。
2. 使用 `AtomicInteger` 作为计数器，确保线程安全。
3. `startAsync()` 方法创建并启动一个新线程来执行计数任务。
4. `stopAsync()` 方法通过设置 `running` 为 false 并中断工作线程来停止服务。
5. `awaitRunning()` 和 `awaitTerminated()` 方法使用自旋等待或线程 join 来等待服务状态变化。
6. `state()` 方法根据服务的当前状态返回相应的 `State` 枚举值。
7. 主方法 (`main()`) 演示了如何使用这个服务：启动、运行10秒、然后停止。

可以启动、运行、停止，并且可以等待服务的状态变化。实现了计数器功能，每秒增加计数并打印。
