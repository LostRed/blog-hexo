---
title: Ruler
date: 2022-07-27 00:00:00
categories:
  - 项目
tags:
  - Java
---

Ruler框架是一个基于SpEL表达式的规则引擎，通过扩展核心AbstractRule抽象类并定义实现核心RulesEngine接口，可以让开发者轻松的实现规则引擎的功能。

## 特性

- 框架侵入性低，可扩展性强
- 一处配置，处处使用
- 可根据需求配置不同的引擎类型(simple, incomplete, complete)

## 核心概念

### DomainFactory领域模型工厂

- 根据配置解析规则引擎需要校验类的属性
- 统一管理这些需要校验的类

### RulesEngineFactory规则引擎工厂

- 根据配置构建规则引擎
- 统一管理不同业务类型的规则引擎

### RuleFactory规则工厂

- 根据配置构建规则
- 统一管理规则

### RulesEngine接口

- simple类型的实现类只关心结果，有违规则直接返回结果
- incomplete类型的实现类会输出报告，有违规也会直接返回结果
- complete类型的实现类会输出报告，并且会执行完所有的规则

### AbstractRule抽象类

定义了规则的主要方法，开发者可扩展该类，实现其他特殊的规则。

## 开始使用

推荐使用ruler-spring-boot-starter启动器，该启动器将ruler框架整合进spring，并且能够在应用启动时，根据配置文件在spring容器中配置好规则引擎组件。
[Rule Spring Boot Starter](ruler-spring-boot-starter.html)
