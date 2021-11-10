---
title: Healthcare Framework
date: 2021-11-08 14:04:35
---
# Healthcare Framework

---

<img style="display:inline;" src="https://img.shields.io/badge/author-lostred-red" alt="author"/> <img style="display:inline;" src="https://img.shields.io/badge/version-v3.2.0--SNAPSHOT-blue" alt="version"/>

<img style="box-shadow: 0 0 0;" src="https://cdn.jsdelivr.net/gh/LostRed/pic-repository@master/healthcare/healthcare-framework.dpimxsl0l1c.png" alt="healthcare"/>

Healthcare Framework框架是为了简化医保控费系统开发而编写，框架中的common模块包含了医保控费领域内的常用数据模型和基础编码数据，其目的是统一项目开发的数据实体结构和编码规范。 不仅如此，框架还提供了一些列常用的功能模块，如字典模块、质控模块、分组器、工作流引擎等，帮助开发者提高效率。

> git地址：http://110.83.51.29:9002/cost-containment/healthcare-framework

## 结构

---

```text
healthcare-framework     
├── healthcare-classify     //分组模块，逻辑代码：对给定的结算清单进行分组操作
├── healthcare-common       //公共模块，数据代码：领域定义的术语集，约定的数据传输对象dto，以及一些数据模型转换的工具类
├── healthcare-dict         //字典模块，数据代码：领域定义的字典数据，提供一个字典表的数据源
├── healthcare-data         //结算清单数据模块，数据代码：提供访问数据库的服务，实现对结算清单数据的查找、新增和修改
├── healthcare-filter       //质控模块，逻辑代码：对病案数据进行过滤和筛选
├── healthcare-web          //web模块，数据代码：基于国家基线版医药机构接口规范规定的报文输入和输出形式
└── healthcare-workflow     //工作流模块，逻辑代码：将系统的工作流程数据给予该模块托管
```

## 说明

---

- classify为分组模块，用于对病案进行分组，包括两套实现dip分组器和drg分组器
- common为基础模块，定义了医保控费系统中常用的数据模型，如结算清单数据模型、ICD10和ICD9CM3、病种等数据库模型，以及质控结果、分组结果等数据传输模型，依赖jackson、jpa、mybatis，支持mybatis-plus注解
- dict为字典模块，主要是提供字典数据库的查询服务
- data为结算清单基础数据模块，底层为JPA，目前只提供结算清单相关数据的查找、新增和修改功能
- filter为质控模块，对结算清单数据模型中的字段值进行过滤和筛选，可按照一定的规则记录不符合规则的字段
- filter-rule为质控模块规则的具体实现
- web为web应用模块，当需要对接国家局工程项目时，包内提供了国家基线版医药机构接口规范规定的报文输入和输出形式
- workflow为工作流模块，提供一套简单的工作流程管理API，实现对工作流程的管理

## 子项目

---

> [Healthcare Classify]()
> [Healthcare Filter](/posts/healthcare-filter)