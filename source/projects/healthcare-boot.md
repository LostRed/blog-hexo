---
title: Healthcare Boot
date: 2021-11-08 14:04:35
---

# Healthcare Boot

---

<img style="display:inline;" src="https://img.shields.io/badge/author-lostred-red" alt="author"/> <img style="display:inline;" src="https://img.shields.io/badge/version-v3.2.0--SNAPSHOT-blue" alt="version"/>

Healthcare Boot是构建在Healthcare Framework之上的一个项目，使用Spring Boot自动配置机制，将Healthcare Framework包中的一些类配置好注入Spring容器。

> git地址：http://110.83.51.29:9002/cost-containment/healthcare-boot

## 结构

---

```text
healthcare-boot
├── healthcare-boot-dependencies         //healthcare-boot的依赖管理
├── healthcare-boot-autoconfigure        //healthcare-boot自动配置类
├── healthcare-classify-boot-starter     //分组器启动器，自动配置好分组器的参数，并将分组器实现类注入spring的ApplicationContext中
├── healthcare-context-boot-starter      //术语集启动器，将常用的术语集以java集合的形式注入spring的ApplicationContext中
├── healthcare-dict-boot-starter         //字典启动器，将字典服务的实现类注入spring的ApplicationContext中
├── healthcare-data-boot-starter         //结算清单数据启动器，将数据库访问服务的实现类注入spring的ApplicationContext中
├── healthcare-filter-boot-starter       //质控启动器，自动配置好规则链的参数，并将质控规则实现类注入spring的ApplicationContext中
├── healthcare-workflow-boot-starter     //工作流引擎启动器
└── integration-tests                    //测试模块，测试以上模块用
```

## 特性

---

- 框架内所有模块都是基于spring-boot自动配置原理，利用spring.factories的加载机制将其他应用程序可能需要的依赖注入spring容器
- 定制化开发不需要引入框架内所有的jar包，按需引入即可。自己代码中需要依赖jar包中的术语集或关键类，通过@Resource或@Autowired注解注入即可

## 快速开始

---

- 引入依赖管理

    ```xml
    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>com.ylzinfo.hf.boot</groupId>
                <artifactId>healthcare-boot-dependencies</artifactId>
                <version>last-version</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>
    ```

- 根据使用场景引入相应的boot-starter启动器

## 子项目

---

> [Healthcare Classify Boot Starter](/posts/healthcare-classify-boot-starter)
> [Healthcare Filter Boot Starter](/posts/healthcare-filter-boot-starter)