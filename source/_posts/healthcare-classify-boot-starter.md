---
title: Healthcare Classify Boot Starter
date: 2021-11-06 17:52:54
categories:
  - 项目
tags: 
  - Java
  - Healthcare Boot
---

Healthcare Classify Boot Starter是分组启动器，程序启动时将根据配置在Spring容器中注册一个分组器对象，开箱即用。

## 特性

---

- 病案结算清单分组器有两套实现，一个是DIP分组器，一个是DRG分组器
- DIP分组器中分值修正部分需要根据不同地市的政策配置不同的策略Bean

## 快速开始

---

### 引入依赖

> 仓库地址：http://10.16.0.127:8081/nexus/repository/maven-public/

```xml
  <dependency>
      <groupId>com.ylzinfo.hf.boot</groupId>
      <artifactId>healthcare-classify-boot-starter</artifactId>
      <version>last-version</version>
  </dependency>
```

### 添加json文件

- 在resource/data下存放一个DIP目录库的json文件，对应类DipGroup的字段，示例如下：

  ```json
  [
      {
          "diagCode": "N20.0",
          "diagName": "肾结石",
          "oprnCode": "55.0300x005",
          "oprnName": "经皮肾造口术",
          "rw": 2.8214,
          "cv": 0.3007,
          "cal": 16,
          "total": 18,
          "mark": "核心病种",
          "dip": "2613",
          "base": false
      }
  ]
  ```

- 在resource/data下存放一个医药机构信息集合的json文件，对应类Fixmedins的字段，示例如下：

  ```json
  [
      {
          "area": "荆州",
          "fixmedinsCode": "42100230001",
          "fixmedinsName": "荆州市第一人民医院",
          "coefficient": "1",
          "grade": 3,
          "gradeName": "三级甲等"
      }
  ]
  ```

- 在resource/data下存放一个分值修正标准集合的json文件，对应类RectifyCriteria的字段，示例如下：

  ```json
  [
      {
          "index": "S3:P80-P83+2",
          "total": 1,
          "amount": 6679.62,
          "average": 6679.62,
          "upperLimit": 13359.24,
          "lowerLimit": 3339.81
      }
  ]
  ```

### 配置

```yaml
healthcare:
  # 配置分组器
  classify:
    json:
      # 地方dip目录库json文件名
      local-dip-grouping-database: dip-group-jingzhou
      # 地方医药机构信息集合json文件名
      fixmedins: fixmedins-jingzhou
      # 地方分值修正标准集合json文件名
      rectify-criteria: criteria-jingzhou
      # 范围内的手术操作编码集合json文件名
      # 如果使用了OprnPreProcessor，则需要在resource/data下提供范围内的手术操作编码集合的json文件
      # 版本号以-1或-2结尾，配置时不需要带上版本号
      included-oprn-code: included-oprn-code
```

### 编写配置类

```java
@Configuration
public class Jingzhou implements InitializingBean {
    //前置处理器
    @Bean
    public PreProcessor preProcessor(@Qualifier("includedOprnCodeSet") Set<String> includedOprnCodeSet) {
        return new DefaultPreProcessor(includedOprnCodeSet);
    }

    //核心病种分组器，需要传入手术类型集合和地方病种目录库
    @Bean
    public CoreClassifier coreClassifier(Map<String, Integer> oprnTypeMap, Map<String, DipGroup> localDipGroupingDatabaseMap) {
        return new DefaultCoreClassifier(oprnTypeMap, localDipGroupingDatabaseMap);
    }

    //综合病种分组器，需要传入手术类型集合和地方病种目录库
    @Bean
    public MixedClassifier mixedClassifier(Map<String, Integer> oprnTypeMap, Map<String, DipGroup> localDipGroupingDatabaseMap) {
        return new SecMixedClassifier(oprnTypeMap, localDipGroupingDatabaseMap);
    }

    //未入组后置处理器
    @Bean
    public NonePostProcessor nonePostProcessor() {
        return new NonePostProcessor() {
            //实现政策文件内容，将指标初始化到convertCriteria中
            @Override
            protected void initConvertCriteria() {
                super.convertCriteria = new ConvertCriteria();
                convertCriteria.setAverage(new BigDecimal("7955.52"));
                convertCriteria.setTotal(new BigDecimal(674366));
                convertCriteria.setAmount(new BigDecimal("5364931772.57"));
                convertCriteria.setCoefficient(new BigDecimal("0.8"));
            }

            //实现可能的总金额动态调整
            @Override
            protected void adjust(BigDecimal amount) {

            }
        };
    }

    //评估修正标准，需要注入定点医药机构集合和分值修正标准集合
    @Bean
    public Evaluator evaluator(Map<String, Fixmedins> fixmedinsMap, Map<String, RectifyCriteria> rectifyCriteriaMap) {
        return new LastYearWithGradeEvaluator(fixmedinsMap, rectifyCriteriaMap);
    }

    //分值修正器，需要注入定点医药机构集合
    @Bean
    public Rectifier rectifier(Map<String, Fixmedins> fixmedinsMap) {
        return new AutoRectifier(fixmedinsMap);
    }

    //设置上下限因子
    @Override
    public void afterPropertiesSet() {
        DipClassifierConstant.UPPER_LIMIT_FACTOR = new BigDecimal(2);
        DipClassifierConstant.DOWN_LIMIT_FACTOR = new BigDecimal("0.5");
    }
}
```

### 使用Classifier

```java
@RestController
@RequestMapping("/setlList")
public class CaseController {
    @Resource
    private Classifier classifier;

    @PostMapping("/classify")
    public ClassifyResult classify(@RequestBody NationSetlList setlList) {
        //根据政策文件要求输入金额(医疗总费用或医保总费用)  
        ClassifyResult classifyResult = classifier.handle(setlList, setlList.getSetlInfo().getMedfeeSumamt());
        return classifyResult;
    }
}
```

> NationSetlList是约定的结算清单数据传输对象
> Classifier是classify包中的一个分组器接口，实现对病案的分组
> ClassifyResult是分组结果的返回格式

