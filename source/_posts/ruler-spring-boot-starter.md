---
title: Ruler Spring Boot Starter
date: 2022-07-27 00:00:00
categories:
  - 项目
tags:
  - Java
  - Spring Boot
---

Ruler Spring Boot Starter是一个spring boot启动器，该项目构建在ruler-core上，整合了spring boot，在项目中直接引入该依赖，能够帮助开发者在spring容器中快速配置ruler框架组件，达到开箱即用的效果。

## 快速开始

### 引入依赖

```xml
<dependency>
    <groupId>info.lostred.ruler</groupId>
    <artifactId>ruler-spring-boot-starter</artifactId>
    <version>{ruler.version}</version>
</dependency>
```

### 配置application.yaml

框架默认只会根据application.yaml配置单实例规则引擎，项目中需要使用到多类规则引擎时，需要自己编写配置类。

```yaml
ruler:
  business-type: person #业务类型
  engine-type: complete #上述提到的规则引擎类型，默认为simple
  rule-default-scope: info.lostred.ruler.test.rule #规则类包扫描路径，与注解@RulerScan定义的路径会取并集并一起扫描
  domain-default-scope: info.lostred.ruler.test.domain #领域模型类包扫描路径，与注解@DomainScan定义的路径会取并集并一起扫描
```

### 编写配置类(可选)

使用注解初始化方式必须配置Configuration，单实例规则引擎不能满足项目时，可自定义规则引擎。

```java
@Configuration
@RuleScan("info.lostred.ruler.test.rule")
@DomainScan("info.lostred.ruler.test.domain")
public class RulerConfig {
    //注册全局函数，spEl表达式可以使用#methodName调用全局函数
    @Bean
    public List<Method> globalFunctions() {
        return Arrays.asList(DateTimeUtils.class.getMethods());
    }
}
```

以上，info.lostred.ruler.test.domain为需要校验类的包名路径，下面是需要校验类的示例代码。

```java
@Data
public class Person {
    private String certNo;
    private String name;
    private String gender;
    private Integer age;
    @JsonFormat(pattern = "yyyy-MM-dd", timezone = "GMT+8")
    private Date birthday;
    private Area area;
    private List<Contact> contacts;
}

@Data
public class Area {
    private String continent;
    private String country;
    private String province;
    private String city;
    private String district;
    private String town;
}

@Data
public class Contact {
    private String type;
    private String account;
    private String password;
    private Area area;
}
```

### 规则引擎依赖注入

以下是单元测试案例。

```java
@SpringBootTest
class ApplicationTest {
    static String businessType = "person";
    static Person person;
    @Autowired
    RulesEngineFactory rulesEngineFactory;
    @Autowired
    ObjectMapper objectMapper;

    String toJson(Object object) throws JsonProcessingException {
        return objectMapper.writerWithDefaultPrettyPrinter().writeValueAsString(object);
    }

    void printResult(Object result, long startTime, long endTime) throws JsonProcessingException {
        System.out.println(toJson(result));
        System.out.println("执行时间: " + (endTime - startTime) + " ms");
    }

    @BeforeAll
    static void init() throws ParseException {
        person = new Person();
        person.setCertNo("12312");
        person.setGender("男");
        person.setAge(10);
        SimpleDateFormat simpleDateFormat = new SimpleDateFormat("yyyy-MM-dd");
        Date parse = simpleDateFormat.parse("2020-01-01");
        person.setBirthday(parse);
        Area area = new Area();
        person.setArea(area);
        Contact contact1 = new Contact();
        contact1.setArea(area);
        contact1.setPassword("1234");
        Contact contact2 = new Contact();
        contact2.setPassword("1234");
        person.setContacts(Arrays.asList(contact1, contact2));
    }

    @Test
    void executeTest() throws JsonProcessingException {
        RulesEngine rulesEngine = rulesEngineFactory.getEngine(businessType);
        long s = System.currentTimeMillis();
        Result result = rulesEngine.execute(person);
        long e = System.currentTimeMillis();
        printResult(result, s, e);
    }
}
```

这里注入的是RulesEngineFactory接口，使用该接口的getEngine()方法获取业务类型对应的规则引擎接口。当然也可以直接注入自己配置规则引擎的实现类。

## 二次开发

继承AbstractRule，并在类上添加@Rule注解。以下提供了开发规则两种方式。

1. 使用注解直接配置表达式
```java
@Rule(ruleCode = "rule_01",
        businessType = "person", //自定义的业务类型
        description = "身份证号码长度必须为18位",
        parameterExp = "certNo",
        conditionExp = "certNo!=null",
        predicateExp = "certNo.length()!=18")
public class CertNoLengthRule extends AbstractRule {
    public CertNoLengthRule(RuleDefinition ruleDefinition, ExpressionParser parser) {
        super(ruleDefinition, parser);
    }
}
```

2. 重写Judgement和Collector的接口方法
```java
@Rule(ruleCode = "rule_01",
        businessType = "person", //自定义的业务类型
        description = "身份证号码长度必须为18位")
public class CertNoLengthRule extends AbstractRule {
    public CertNoLengthRule(RuleDefinition ruleDefinition) {
        super(ruleDefinition);
    }

    @Override
    public boolean isSupported(EvaluationContext context, ExpressionParser parser, Object object) {
        return object instanceof Person && ((Person) object).getCertNo() != null;
    }

    @Override
    public boolean judge(EvaluationContext context, ExpressionParser parser, Object object) {
        return ((Person) object).getCertNo().length() != 18;
    }

    @Override
    public Map<String, Object> collectMappings(EvaluationContext context, ExpressionParser parser, Object object) {
        String certNo = ((Person) object).getCertNo();
        Map<String, Object> map = new HashMap<>(1);
        map.put("certNo", certNo);
        return map;
    }
}
```

### Rule注解

在类上标记该注解，规则工厂在扫描包时，会将其放入RuleFactory的单例池，统一管理。

### RuleScan注解

在配置类上标记该注解，规则工厂会扫描其value指定的包路径。当使用spring时，需将该配置类注册到spring容器。

### DomainScan注解

在配置类上标记该注解，规则工厂会扫描其value指定的包路径。当使用spring时，需将该配置类注册到spring容器。

```java
@Configuration
@RuleScan("info.lostred.ruler.test.rule")
@DomainScan("info.lostred.ruler.test.domain")
public class RulerConfig {
}
```
