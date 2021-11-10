---
title: Healthcare Filter
date: 2021-11-06 17:10:17
categories:
  - 项目
tags:
  - Java
  - Healthcare Framework
---

Healthcare Filter提供了一套针对结算清单字段过滤的解决方案。核心类RuleChain将所有单一职责的规则类连接到一起，形成责任链，对传入的结算清单参数进行校验，返回相应的结果。

## 特性

---

- 结算清单的整体质控实现模式是若干条校验规则组成的责任链模式，即校验规则链
- 校验规则链的规则执行器有两种模式可供选择，默认详细执行器会记录违规字段和违规值，而简单执行器则不会记录违规字段和违规值
- 校验规则链的规则迭代器有两种模式可供选择，默认全量迭代器会依次通过所有校验规则校验，除非在质控结果中包含"US"校验规则代号(即唯一校验)的错误记录，而跳出迭代器会在出现错误信息后直接结束校验
- 规则执行器每执行一个规则，且发现有违规项目时，会在校验结果中添加一个记录
- 经过所有校验规则校验后，会生成一个校验结果返回值，校验结果等级分为“合格”、“可疑”和“违规”

## 规则类

---

<img style="display:block; margin:0 auto;" src="https://cdn.jsdelivr.net/gh/LostRed/pic-repository@master/healthcare/Rule.a9uhliwzpew.png" alt="Rule校验规则"/>

Rule是一个核心类，它继承了Judgement和Record接口，分别定义了规则的责任方法、判定方法和记录结果方法。

## 规则加载原理

---

<img style="display:block; margin:0 auto;" src="https://cdn.jsdelivr.net/gh/LostRed/pic-repository@master/healthcare/RuleChain.6q9yh1zhg8s0.png" width = "400" alt="RuleChain继承树"/>

RuleChain是一个核心接口，AbstractRuleChain抽象类实现了它，我们直接进入AbstractRuleChain的代码：

```java
public abstract class AbstractRuleChain implements RuleChain, ApplicationContextAware, BeanNameAware, InitializingBean {
    protected final Logger logger = LoggerFactory.getLogger(this.getClass());
    protected RuleInfoService ruleInfoService;
    
    /**
     * 在所有spring的配置完成后，调用该方法
     */
    @Override
    public void afterPropertiesSet() {
        this.refresh(this.init());
    }

    @Override
    public void refresh(List<Rule> rules) {
        if (rules == null || rules.size() == 0) {
            return;
        }
        //将校验校验规则按配置的序列排序
        List<Rule> list = rules.stream()
            .sorted(Comparator.comparing(rule -> rule.getRuleInfo().getSeq()))
            .collect(Collectors.toList());
        //设置校验规则的先后顺序
        Iterator<Rule> iterator = list.iterator();
        Rule rule = iterator.next();
        head = rule;
        while (iterator.hasNext()) {
            Rule next = iterator.next();
            rule.setNext(next);
            rule = next;
        }
        logger.info("校验规则链({})已生成，共启用{}条规则", beanName, rules.size());
    }

    /**
     * 根据校验规则信息往spring容器中注册校验规则
     *
     * @param ruleInfo 校验规则信息
     * @return 校验规则
     */
    protected Rule register(RuleInfo ruleInfo) {
        try {
            Class.forName(ruleInfo.getClassName());
        } catch (ClassNotFoundException e) {
            logger.warn("没有找到校验规则类 ==> {}", ruleInfo.getClassName());
            return null;
        }
        BeanDefinition beanDefinition = BeanDefinitionBuilder.genericBeanDefinition().getBeanDefinition();
        beanDefinition.setBeanClassName(ruleInfo.getClassName());
        if (!applicationContext.containsBean(ruleInfo.beanName())) {
            applicationContext.registerBeanDefinition(ruleInfo.beanName(), beanDefinition);
            logger.debug("容器注册 <== {}", ruleInfo.getClassName());
        }
        Rule rule = applicationContext.getBean(ruleInfo.beanName(), Rule.class);
        rule.setRuleInfo(ruleInfo);
        return rule;
    }

    /**
     * 根据校验规则信息集合往spring容器中注册校验规则
     *
     * @param ruleInfoList 校验规则信息集合
     * @return 校验规则集合
     */
    protected List<Rule> register(List<RuleInfo> ruleInfoList) {
        List<Rule> rules = new ArrayList<>();
        for (RuleInfo ruleInfo : ruleInfoList) {
            Rule rule = this.register(ruleInfo);
            if (rule != null) {
                rules.add(rule);
            }
        }
        return rules;
    }

    /**
     * 根据数据库中的校验规则信息往spring容器中注册校验规则
     *
     * @return 校验规则集合
     */
    List<Rule> init();
}
```

可以看到AbstractRuleChain实现了InitializingBean接口，重写了afterPropertiesSet()方法，该方法会在所有spring的配置完成后执行。而该方法又调用了本类的refresh()和init()
的方法，其中refresh()方法仅仅是将所有规则组成了一个规则链。再来看看实现类的init()方法：

```java
public class QualityRuleChain extends AbstractRuleChain implements ApplicationListener<QualityRuleChangeEvent> {
    public QualityRuleChain(RuleInfoService ruleInfoService) {
        super(ruleInfoService);
    }

    @Override
    public void onApplicationEvent(QualityRuleChangeEvent qualityRuleChangeEvent) {
        qualityRuleChangeEvent.getChangeList().stream()
            .filter(e -> e.getEnable() == 0)
            .map(RuleInfo::getRuleCode)
            .forEach(super::removeRule);
        qualityRuleChangeEvent.getChangeList().stream()
            .filter(e -> e.getEnable() == 1)
            .map(super::register)
            .forEach(super::addRule);
    }

    @Override
    public List<Rule> init() {
        return super.register(ruleInfoService.findEnableByPhase(Phase.QUALITY.getCode()));
    }
}
```

init()方法又调用了父类的register()
方法，该方法的入参是从数据库查询出来的RuleInfo集合，同时该方法调用了重载方法，可以看到最终程序根据数据库查出的className字段信息，反射创建Rule实例对象，并将该对象注册到Spring容器中。

## 规则链校验流程

---

<img style="display:block; margin:0 auto;" src="https://cdn.jsdelivr.net/gh/LostRed/pic-repository@master/healthcare/校验流程.2f2wvchsxwsg.jpg" alt="规则链校验流程"/>

1. RuleChain的handle()方法将根据结算清单NationSetlList的唯一标识创建一个QualityResult对象
2. 将规则链首部规则取出，同结算清单NationSetlList和QualityResult交由规则执行器处理
3. 规则执行器结束执行后，将规则交由规则迭代器处理获取下一个规则
4. 当规则链走到链尾时，对QualityResult对象进行判断质控结果
5. 最终返回QualityResult对象

## 规则变化事件

---

<img style="display:block; margin:0 auto;" src="https://cdn.jsdelivr.net/gh/LostRed/pic-repository@master/healthcare/规则变化事件.2z932artkus0.jpg" width = "400" alt="规则变化事件"/>

QualityRuleChain和ScreenRuleChain还实现了ApplicationListener接口，它们分别监听了两个规则变化事件，QualityRuleChangeEvent和ScreenRuleChangeEvent。当使用Spring的ApplicationContext发布这些事件后，这两个规则链则会执行相应的方法，该法方法在onApplicationEvent()
中执行，通过这个监听模式，就能够实现在用户修改规则信息表中的启用字段后对规则链中规则的更新，实现RuleChain与规则管理类的解耦。
