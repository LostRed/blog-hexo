---
title: Healthcare Filter Boot Starter
date: 2021-11-06 17:10:17
categories:
  - 项目
tags: 
  - Java
  - Healthcare Boot
---

Healthcare Filter Boot Starter是质控启动器，程序启动时将在Spring容器中注册一个RuleChain实现类对象，开箱即用。

## 特性

---

- 结算清单的整体质控实现模式是若干条校验规则组成的责任链模式，即校验规则链
- 校验规则链的规则执行器有两种模式可供选择，默认详细执行器会记录违规字段和违规值，而简单执行器则不会记录违规字段和违规值
- 校验规则链的规则迭代器有两种模式可供选择，默认全量迭代器会依次通过所有校验规则校验，除非在质控结果中包含"US"校验规则代号(即唯一校验)的错误记录，而跳出迭代器会在出现错误信息后直接结束校验
- 规则执行器每执行一个规则，且发现有违规项目时，会在校验结果中添加一个记录
- 经过所有校验规则校验后，会生成一个校验结果返回值，校验结果等级分为“合格”、“可疑”和“违规”

## 快速开始

---

###  引入依赖

> 仓库地址：http://10.16.0.127:8081/nexus/repository/maven-public/

```xml
<dependency>
    <groupId>com.ylzinfo.hf.boot</groupId>
    <artifactId>healthcare-filter-boot-starter</artifactId>
    <version>last-version</version>
</dependency>
```

### 配置

```yaml
# 配置Spring
spring:
  # 数据源
  datasource:
    driver-class-name: com.mysql.cj.jdbc.Driver
    url: jdbc:mysql://ip:port/db?useUnicode=true&characterEncoding=utf8
    username: username
    password: password
  # JPA
  jpa:
    open-in-view: false

# 配置校验规则
healthcare:
  # 配置自定义表名
  table-name:
    rule-info: rule_info
  # 配置编码版本
  context:
    icd10:
      enable: true # 是否启用《ICD-10》
      version: 2   # 版本
    icd9cm3:
      enable: true # 是否启用《ICD-9-CM-3》
      version: 2   # 版本
    quality:
      enable: true # 是否启用质控相关术语集
    tcm:
      enable: true # 是否启用中医编码
  # 配置质控模块
  filter:
    # 配置校验规则链模式
    rule-chain:
      rule-executor: detailed     # 配置执行器，不配置默认为detailed
      rule-iterator: full         # 配置迭代器，不配置默认为full
    # 唯一字段(参考系统要求的接口文档)
    unique:
      setl-info-fields:
        - rid
    # 必填字段(参考系统要求的接口文档)
    required:
      setl-info-fields:
        - mdtrtId
        - setlId
```

> 首次运行程序，程序会检测校验规则信息表，当表不存在时，会重新初始化表格(配置非默认自定义表名则失效)，目前只支持mysql数据库

### 自定义规则

> 无二次开发可跳过此步骤

- 创建一个Java类继承Rule类，实现其方法

  ```java
  public class TestRule extends Rule implements ErrorRecord {
      // 必须重写父类的isResponsible()方法，定义校验规则的校验条件
      @Override
      public boolean isResponsible(SetlList setlList) {
          return true;
      }
  
      // 简单执行器会执行的判断方法，返回true表示违规，false表示未违规
      // 该方法在判断出违规后直接返回true，可提升校验的执行速度
      @Override
      public boolean judge(SetlList setlList) {
          return false;
      }
  
      // 详细执行器会执行的判断方法，返回true表示违规，false表示未违规
      // 请将违规字段和违规值用QualityPoint类包装后加入集合
      @Override
      public boolean judge(SetlList setlList, List<QualityPoint> qualityPoints) {
          return false;
      }
  }
  ```

  > 可疑类规则可直接实现WarnRecord接口，违规类规则可直接实现ErrorRecord，省去重写Record接口的write()方法	
  
  ```
  可继承的基类：
  com.ylzinfo.hf.filter.Rule		        //结算清单抽象校验规则
  com.ylzinfo.hf.filter.QualityRule		//结算清单质量校验规则
  com.ylzinfo.hf.filter.LogicRule			//结算清单逻辑校验规则
  com.ylzinfo.hf.filter.DictRule			//结算清单字典校验规则
  com.ylzinfo.hf.filter.ObjectDictRule	        //结算清单对象类节点字典校验规则
  com.ylzinfo.hf.filter.ListDictRule		//结算清单集合类节点字典校验规则
  com.ylzinfo.hf.filter.RequiredRule		//结算清单必填校验规则
  com.ylzinfo.hf.filter.ObjectRequiredRule	//结算清单对象类节点必填校验规则
  com.ylzinfo.hf.filter.ListRequiredRule		//结算清单集合类节点必填校验规则
  com.ylzinfo.hf.filter.UniqueRule		//结算清单唯一校验规则
  com.ylzinfo.hf.filter.ObjectUniqueRule		//结算清单对象类节点唯一校验规则
  com.ylzinfo.hf.filter.ListUniqueRule		//结算清单集合类节点唯一校验规则
  ```
  
- 在规则信息表中添加相关的规则信息

    ```sql
    insert into rule_info (rule_code, rule_type, item_type, item, grade, rule_desc, class_name, seq, phase, must, enable) values ('DS10', '规则类型', '质控类型', '质控项目', '违规', '规则描述', 'Java类名', 1, 0, 0, 1);
    ```

    > seq字段是指规则执行的先后顺序，数字小的优先执行
    > phase字段表示规则生效环节，可使用数字代表不同的环节
    > must字段表示是否为必选规则
    > enable字段表示该规则是否启用

### 使用RuleChain

```java
@RestController
@RequestMapping("/setlList")
public class CaseController {
    @Resource
    private RuleChain ruleChain;

    @PostMapping("/validateDetails")
    public QualityResult validate(@RequestBody NationSetlList setlList) {
        // 调用handle()方法进行校验            
        return ruleChain.handle(setlList);                             
    }
}
```

> NationSetlList是约定的国家基线版结算清单数据报文
> RuleChain是filter包中的一个核心类，实现对病案数据的校验
> QualityResult是校验结果的统一返回格式

## 高级功能

---

### 忽略规则

忽略的规则在会被RuleChain直接跳过，通过addIgnoredRules()和removeIgnoredRules()，可以对忽略规则进行添加和移除。

### 设置执行器

目前规则执行器有两种，通过RuleExecutor接口可以扩展新的执行器实现类，并设置为RuleChain的执行器。

### 设置迭代器

目前规则迭代器有两种，通过RuleIterator接口可以扩展新的迭代器实现类，并设置为RuleChain的迭代器。

```java
@RestController
@RequestMapping("/setlList")
public class CaseController {
    @Resource
    private RuleChain ruleChain;

    @PostMapping("/validateDetails")
    public QualityResult validate(@RequestBody NationSetlList setlList) {
        // 可手动设置执行器
        ruleChain.setRuleExecutor(new DetailedRuleExecutor());
        // 可手动设置迭代器           
        ruleChain.setRuleIterator(new FullRuleIterator());
        // 添加忽略的规则
        ruleChain.addIgnoredRules(Arrays.asList(AcpInsRule.class));
        // 调用handle()方法进行校验      
        QualityResult qualityResult = ruleChain.handle(setlList);
        // 移除忽略的规则
        ruleChain.removeIgnoredRules(Arrays.asList(AcpInsRule.class));
        return qualityResult;
    }
}
```

> 更多内容请移步：[Healthcare Filter](/posts/healthcare-filter)
