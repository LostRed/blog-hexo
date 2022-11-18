---
title: Spring Framework的扩展接口
date: 2021-11-23 22:13
categories:
  - Java
tags:
  - Java
  - Spring Framework
---

## 操作bean的相关接口

### InitializingBean接口和DisposableBean接口

InitializingBean是由需要在BeanFactory设置所有属性后做出反应的 bean 实现的接口：例如，执行自定义初始化，或仅检查是否已设置所有必需属性。其afterPropertiesSet()方法在设置所有 bean 属性并满足BeanFactoryAware ， ApplicationContextAware等之后，由包含的BeanFactory调用。

DisposableBean是由想要在销毁时释放资源的 bean 实现的接口。 BeanFactory将在单独销毁作用域 bean 时调用 destroy 方法。

```java
@Component
public class User implements InitializingBean, DisposableBean {
    private String id;
    private String name;

    public String getId() {
        return id;
    }

    public void setId(String id) {
        this.id = id;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    @Override
    public String toString() {
        return "User{" +
                "id='" + id + '\'' +
                ", name='" + name + '\'' +
                '}';
    }

    @Override
    public void afterPropertiesSet() throws Exception {
        this.id = "123";
        this.name = "zhangsan";
    }

    @Override
    public void destroy() throws Exception {
        this.id = null;
        this.name = null;
    }
}

```

以上代码在Spring的ApplicationContext运行之后容器中的User的bean对象的id和name会被初始化并赋值，销毁时id和name会被置空。

```java
@SpringBootApplication
public class TestApplication {
    public static void main(String[] args) {
        ConfigurableApplicationContext run = SpringApplication.run(TestApplication.class, args);
        User bean = run.getBean(User.class);
        System.out.println(bean);
        ConfigurableListableBeanFactory beanFactory = run.getBeanFactory();
        beanFactory.destroyBean(bean);
        System.out.println(bean);
    }
}
```

主程序运行打印结果：

```
User{id='123', name='zhangsan'}
User{id='null', name='null'}
```

###  BeanPostProcessor接口

BeanPostProcessor接口提供了两个方法，分别是postProcessBeforeInitialization()和postProcessAfterInitialization()，分别在bean的初始化回调（如 InitializingBean 的afterPropertiesSet或自定义初始化方法）之前和初始化回调（如 InitializingBean 的afterPropertiesSet或自定义初始化方法）之后调用。

ApplicationContext可以在其 bean 定义中自动检测BeanPostProcessor bean，并将这些后处理器应用于随后创建的任何 bean。 普通的BeanFactory允许以编程方式注册后处理器，将它们应用于通过 bean 工厂创建的所有 bean。

现在新建一个类实现BeanPostProcessor接口，并对User类进行处理：

```java
@Component
public class TestPostProcessor implements BeanPostProcessor {
    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        if (bean instanceof User) {
            ((User) bean).setId("321");
            ((User) bean).setName("lisi");
            System.out.println("postProcessBeforeInitialization: " + bean);
        }
        return bean;
    }

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        if (bean instanceof User) {
            ((User) bean).setId("000");
            ((User) bean).setName("wangwu");
            System.out.println("postProcessAfterInitialization: " + bean);
        }
        return bean;
    }
}

```

主程序运行打印结果：

```
postProcessBeforeInitialization: User{id='321', name='lisi'}
postProcessAfterInitialization: User{id='000', name='wangwu'}
User{id='000', name='wangwu'}
User{id='null', name='null'}
```

可以看出postProcessBeforeInitialization()方法在afterPropertiesSet()之前调用，而postProcessAfterInitialization()在afterPropertiesSet()之后调用，最终bean的name的值变成了`wangwu`。



###  Aware接口

实现Aware接口，能获取增强方法，而处理Aware接口方法的就是上面的BeanPostProcessor接口。

下面以BeanNameAware接口为例，让User实现BeanNameAware接口：

```java
@Component
public class User implements InitializingBean, DisposableBean, BeanNameAware {
    private String id;
    private String name;

    public String getId() {
        return id;
    }

    public void setId(String id) {
        this.id = id;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    @Override
    public String toString() {
        return "User{" +
                "id='" + id + '\'' +
                ", name='" + name + '\'' +
                '}';
    }

    @Override
    public void afterPropertiesSet() throws Exception {
        this.id = "123";
        this.name = "zhangsan";
    }

    @Override
    public void destroy() throws Exception {
        this.id = null;
        this.name = null;
    }

    @Override
    public void setBeanName(String name) {
        this.name = name;
        System.out.println("setBeanName: " + this);
    }
}

```

主程序运行打印结果：

```
setBeanName: User{id='null', name='user'}
postProcessBeforeInitialization: User{id='321', name='lisi'}
postProcessAfterInitialization: User{id='000', name='wangwu'}
User{id='000', name='wangwu'}
User{id='null', name='null'}
```

## SpringMVC的相关接口

SpringMVC的核心组件是DispatcherServlet，DispatcherServlet下还驱动着9大策略组件，分别是：

```
MultipartResolver				 //文件解析器
LocaleResolver					//当前环境解析器
ThemeResolver					//主题解析器
HandlerMapping					//处理器的映射器
HandlerAdapter					//处理器的适配器
HandlerExceptionResolver		    //处理器的异常解析器
RequestToViewNameTranslator		    //当前环境处理器
ViewResolver					//视图解析器
FlashMapManager					//参数传递管理器
```

最重要的两个接口是HandlerMapping和HandlerAdapter，一个用于扫描web请求的处理器，一个用于对DispatcherServlet接收到的请求进行处理，DispatcherServlet的获取到的参数仅仅是HttpServletRequest和HttpServletResponse

### HandlerMapping接口

HandlerMapping的主要职责是根据HttpServletRequest请求找到适合的Handler，其中getHandler方法返回的是HandlerExecutionChain，该类包装了Handler处理器和HandlerInterceptor处理器拦截器。
HandlerMapping底下的抽象类AbstractHandlerMapping实现类分支有两个，一个是AbstractUrlHandlerMapping，最终映射到一个实例对象上；另一个是AbstractHandlerMethodMapping，最终映射到一个方法上。

### HandlerAdapter接口

HandlerAdapter的主要职责是根据匹配到的Handler处理具体的业务请求，返回模型视图。
