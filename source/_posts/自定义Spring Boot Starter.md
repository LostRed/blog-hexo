---
title: 自定义Spring Boot Starter
date: 2021-11-08 08:49:52
categories:
  - Java
tags:
  - Java
  - Spring Boot
---

Spring Boot Starter是Spring Boot中的一个重要机制，它借鉴了Java中的SPI(Service Provider Interface)，它通常由AutoConfiguration和Properties类组成，在程序加载时Spring容器启动后，读取资源类路径下特定文件中配置好的类，将它们注册到容器中。其实Spring Boot Starter的任务就是在Spring Boot应用启动前完成自动配置。

> 注意：Spring官方提供的Starter包命名方式为spring-boot-starter-xxx，而第三方Starter包的命名方式为xxx-spring-boot-starter。

下面我们来看看如何自定义一个Spring Boot Starter。

## 1. 创建Maven工程

---

创建一个maven工程，artifactId起名为xxx-spring-boot-starter，groupId不可在org.springframework路径下。

## 2. 引入依赖

---

引入spring-boot-dependencies下的autoconfigure和configuration-processor，autoconfigure提供自动配置支持，而configuration-processor是给开发人员在IDE环境配置`application.properties`或`application.yaml`时，提供上下文帮助、提示和跳转支持。

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-autoconfigure</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-configuration-processor</artifactId>
        <optional>true</optional>
    </dependency>
</dependencies>
```

## 3. 编写自动配置类

---

在编写配置类之前，我们需要先了解一下[Spring Boot的自动配置原理](spring-boot-autoconfigure-principle.html)。

```java
@Configuration
@EnableConfigurationProperties(MyProperties.class)
public class MyAutoConfiguration { 
    @Resource
    private MyProperties properties;
    
    // 编写自己注册Bean的方法
}
```

在resources目录下创建`META-INF/spring.factories`文件，并在org.springframework.boot.autoconfigure.EnableAutoConfiguration下添加自动配置类的全限定类名。

```properties
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
info.lostred.autoconfigure.MyAutoConfiguration
```

## 4. 编写配置绑定

---

创建Properties类并指定配置项前缀。

```java

@ConfigurationProperties(prefix = "my")
public class MyProperties {
    private String name;
    
    public String getName(){
        return name;
    }
    
    public void setName(String name){
        this.name = name;
    }
}
```

在`application.properties`中配置配置项，可以看到IDE有配置项的提示。

```properties
my.name=lostred
```

## 5. 编写配置元数据

---

> 如果用Properties绑定了配置，这步可以省略。

在resources目录下创建`META-INF/additional-spring-configuration-metadata.json`文件。

```json
{
    "properties": [
        {
            "name": "my.name",
            "type": "java.lang.Boolean",
            "sourceType": "info.lostred.autoconfigure.MyAutoConfiguration",
            "description": "我的名字.",
            "defaultValue": "lostred"
        }
    ]
}
```

## 6. 打包发布Starter

---

将项目打包后发布到maven仓库，这样其它工程就可以依赖该xxx-spring-boot-starter实现自动配置。
