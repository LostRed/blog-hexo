---
title: Java中DelayQueue的使用
date: 2023-02-24 14:56:00
categories:
  - Java
tags:
  - Java
---

## 1. 什么是DelayQueue

DelayQueue是JDK concurrent包下提供的一个类，实现了Queue接口。其本质是一个队列数据结构。
DelayQueue的元素必须是Delayed接口，该接口继承Comparable。
接口提供getDelay的方法，返回延时剩余时间，当返回值为0时，才能取出元素。

## 2. DelayQueue能用在什么地方

DelayQueue一般用于生产者消费者模式，典型案例就是订单系统中的超时支付。

## 3. 最佳实践

使用DelayQueue给订单超时支付进行一个建模。首先，在订单中定义一个超时时间，这个在创建订单时就应该确定。
并且在创建订单后，将订单加入一个DelayQueue，之后程序循环从队列取出订单执行关单处理的逻辑。

### 3.1 创建一个数据库实体类

实体类需要实现Delayed接口，重写getDelay方法，返回订单的过期时间减去当前系统时间来确定订单是否过期，也就是能否从队列中取出。
以下代码使用了mybatis-plus和lombok依赖。

```java
@Data
@TableName("`order`")
public class Order implements Delayed {
    @TableId
    private String id;
    private String orderStatus; //0表示未过期，1表示已过期
    private Date expiredTime;

    @Override
    public long getDelay(TimeUnit unit) {
        return this.expiredTime.getTime() - System.currentTimeMillis();
    }

    @Override
    public int compareTo(Delayed o) {
        if (this == o) {
            return 0;
        }
        Order order = (Order) o;
        long time = this.expiredTime.getTime() - order.getExpiredTime().getTime();
        return time == 0 ? 0 : time < 0 ? -1 : 1;
    }
}
```

### 3.2 定义订单服务

由于循环从队列取出的任务必须在后台执行，所以在订单服务中需使用到线程池，并将这个任务提交给线程池处理。
由于这一任务在应用关闭后就会停止，所以在应用启动时，就应该将数据中的订单数据读出并加入队列。

```java
@Slf4j
@Service
public class OrderService implements InitializingBean {
    @Autowired
    private OrderMapper orderMapper;
    private final DelayQueue<Order> delayQueue = new DelayQueue<>();
    private final AtomicBoolean start = new AtomicBoolean(false);// 任务启动的状态标识，这里可以使用boolean类型
    // 线程池
    private final ExecutorService executorService = new ThreadPoolExecutor(
            3, // 核心线程池大小
            6, // 最大线程池大小
            60L, // 线程最大空闲时间
            TimeUnit.MILLISECONDS, // 时间单位(毫秒)
            new LinkedBlockingQueue<>(),  // 线程等待队列
            Executors.defaultThreadFactory(), // 线程创建工厂
            new ThreadPoolExecutor.CallerRunsPolicy() // 拒绝策略
    );

    //InitializingBean接口方法，在spring配置完bean后会执行
    @Override
    public void afterPropertiesSet() {
        start();
    }

    public void start() {
        if (start.get()) {
            return;
        }
        start.set(true);
        //查询所有未过期的订单
        List<Order> list = orderMapper.selectList(Wrappers.lambdaQuery(Order.class)
                .eq(Order::getOrderStatus, "0"));
        //底层会自动排序
        delayQueue.addAll(list);
        //向线程池提交任务
        executorService.submit(() -> {
            // 取消订单
            while (start.get()) {
                try {
                    //从队列中取出订单，take会阻塞等待直到有订单被取出
                    Order order = delayQueue.take();
                    orderMapper.update(null, Wrappers.lambdaUpdate(Order.class)
                            .set(Order::getOrderStatus, "1")
                            .eq(Order::getId, order.getId()));
                    log.info("订单id={}，已过期", order.getId());
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        });
    }
}
```
