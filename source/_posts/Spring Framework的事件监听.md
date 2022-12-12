---
title: Spring Framework的事件监听
date: 2022-12-12 09:50:00
categories:
  - Java
tags:
  - Java
  - Spring Framework
---

## 1. Spring中的观察者模式

---

Spring提供了ApplicationListener接口，利用该接口可以将原本bean之间的依赖关系进行解耦。这是一种发布订阅的模式，通过Spring容器发布事件，由订阅的监听器执行对应的代码逻辑。
需要注意的是，监听器执行与事件发布是同步执行的，如果需要异步执行，需要配合Spring的异步线程池进行处理。

## 2. 最佳实践

---

实现ApplicationListener接口需要先定义事件类，该事件类需要继承Spring的ApplicationEvent。

```java
public class TestEvent extends ApplicationEvent {
    private String value;

    public TestEvent(Object source) {
        super(source);
    }

    public String getValue() {
        return value;
    }

    public void setValue(String value) {
        this.value = value;
    }
}
```

接下来就是创建一个监听类，实现ApplicationListener的opApplicationEvent方法，这个方法就是在上述自定义事件被发布后，监听器会自动执行的代码逻辑。

```java
/**
 * 针对具体事件的监听器
 */
@Slf4j
@Component
@EnableAsync // 需要异步处理则可以考虑使用该注解
public class TestListener implements ApplicationListener<TestEvent> {
    @Async // 需要异步处理则可以考虑使用该注解
    @Override
    public void onApplicationEvent(TestEvent testEvent) {
        String value = testEvent.getValue();
        log.info("监听到事件{}, 事件的值为{}", testEvent.getClass().getSimpleName(), value);
    }
}
```

在原本需要依赖地方发布事件

```java
@RestController
@RequestMapping("/event")
public class EventController {
    private final ApplicationContext applicationContext;

    public EventController(ApplicationContext applicationContext) {
        this.applicationContext = applicationContext;
    }

    @PostMapping("/publish")
    public String publish(String value) {
        TestEvent testEvent = new TestEvent(this);
        testEvent.setValue(value);
        applicationContext.publishEvent(testEvent);
        return "success";
    }
}
```

这样，原本两个类之间的依赖关系就被解除了，改为调用者对容器与事件的依赖。而这些类都可以统一作为公共模块部分供其它业务类模块依赖，从而增加了系统代码的可移植性。

## 3. 其它相关接口

---

Spring还提供了一个SmartApplicationListener接口。该接口继承了ApplicationListener，这个接口增加排序功能。如果事件的监听器有多个，并且有执行顺序的需求，可以使用该接口，重写getOrder()方法即可。
该接口的另一个方法supportsEventType()是对事件类型的支持判断，只有该方法的返回值为true的情况下，监听器才会生效。

```java
@Slf4j
@Component
public class TestSmartListener1 implements SmartApplicationListener {
    /**
     * 监听条件
     *
     * @param eventType the event type (never {@code null})
     * @return 返回true才生效
     */
    @Override
    public boolean supportsEventType(Class<? extends ApplicationEvent> eventType) {
        return eventType == TestEvent.class;
    }

    /**
     * 对事件进行处理
     *
     * @param event the event to respond to
     */
    @Override
    public void onApplicationEvent(ApplicationEvent event) {
        String value = ((TestEvent) event).getValue();
        log.info("监听到事件{}, 事件的值为{}", event.getClass().getSimpleName(), value);
    }

    /**
     * 返回值小的优先执行
     *
     * @return 执行顺序号
     */
    @Override
    public int getOrder() {
        return 1;
    }
}
```

```java
@Slf4j
@Component
public class TestSmartListener2 implements SmartApplicationListener {
    /**
     * 监听条件
     *
     * @param eventType the event type (never {@code null})
     * @return 返回true才生效
     */
    @Override
    public boolean supportsEventType(Class<? extends ApplicationEvent> eventType) {
        return eventType == TestEvent.class;
    }

    /**
     * 对事件进行处理
     *
     * @param event the event to respond to
     */
    @Override
    public void onApplicationEvent(ApplicationEvent event) {
        String value = ((TestEvent) event).getValue();
        log.info("监听到事件{}, 事件的值为{}", event.getClass().getSimpleName(), value);
    }

    /**
     * 返回值小的优先执行
     *
     * @return 执行顺序号
     */
    @Override
    public int getOrder() {
        return 0;
    }
}
```