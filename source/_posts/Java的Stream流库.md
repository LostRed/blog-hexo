---
title: Java中的Stream流库
date: 2021-03-11 15:40:40
categories:
  - Java
tags:
  - Java
---

## 1. 什么是流

---

### 1.1 概念

Stream不是集合元素，它不是数据结构并不保存数据，它是有关算法和计算的，它更像一个高级版本的Iterator。原始版本的 Iterator，用户只能显式地一个一个遍历元素并对其执行某些操作；高级版本的Stream，用户只要给出需要对其包含的元素执行什么操作，比如 “过滤掉长度大于 10 的字符串”、“获取每个字符串的首字母”等，Stream会隐式地在内部进行遍历，做出相应的数据转换。

### 1.2 特点

- 流并不存储其元素
- 流的操作不会修改其数据源
- 流的操作是尽可能惰性执行的，就是说到了非要有结果的时候，它才会执行

## 2. 流的操作步骤

---

如Demo所示，这个工作流是操作流时的典型流程，其统计了“alice.txt”文件中单词字母长度大于10的单词数量。我们建立了一个包含三个阶段的操作管道：

1. 创建一个流
2. 指定初始流转换为其他流的中间操作，可能包含多个步骤
3. 应用终止操作，从而产生结果。这个操作会强制执行之前的惰性操作，从此之后这个流就再也不能用了。

```java
package example1;

import java.io.IOException;
import java.nio.charset.StandardCharsets;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.util.Arrays;
import java.util.List;

public class Demo {
    public static void main(String[] args) throws IOException {
        String contents = new String(Files.readAllBytes(
                Paths.get("alice.txt")), StandardCharsets.UTF_8);
        List<String> words = Arrays.asList(contents.split("\\PL+")); //以非字母为分隔符
        long count = words.parallelStream()     //创建流
                .filter(w -> w.length() > 10)   //转换操作(过滤长度大于10的字符)
                .count();                       //终止操作(统计数量)
        System.out.println(count);
    }
}
```

## 3. 流的创建

---

Collection接口的stream方法可以将任何集合转换成流。如果有一个数组就使用静态的Stream.of方法。

```java
//将集合转换成流
List<String> words = ...;
Stream<String> s1 = words.stream();
//将数组转换成流
String[] array = ...;
Stream<String> s2 = Stream.of(array);
//of方法还可以选择截取数组来创建流  of(array, from, to):包含from，不包含to
Stream<String> s3 = Stream.of(array, 0, 5);
//创建一个不包含任何元素的流
Stream<String> s4 = Stream.empty();
```

创建无限流有两种方法：

1. 调用Stream.generate()方法
2. 调用Stream.iterate()方法

```java
package example2;

import java.io.IOException;
import java.nio.charset.StandardCharsets;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.util.List;
import java.util.regex.Pattern;
import java.util.stream.Collectors;
import java.util.stream.Stream;

public class Create {
    public static void main(String[] args) throws IOException {
        //对于String类型的数组，用Stream.of方法来创建流
        Stream<String> words = Stream.of(new String(
                Files.readAllBytes(Paths.get("alice.txt")), StandardCharsets.UTF_8)
                .split("\\PL+"));
        show("words", words);

        Stream<String> song = Stream.of("gently", "down", "the", "stream");
        show("song", song);

        //创建一个不包含任何元素的流
        Stream<String> empty = Stream.empty();
        show("empty", empty);

        //创建一个String类型的无限流
        Stream<String> echos = Stream.generate(() -> "Echo");
        show("echos", echos);

        //创建一个产生随机数的无限流
        Stream<Double> randoms = Stream.generate(Math::random);
        show("randoms", randoms);

        //创建一个无限序列，开始为0，每次加1
        //iterate方法可以给出一个种子值，以及一个函数
        Stream<Integer> integers = Stream.iterate(0, n -> n + 1);
        show("integers", integers);

        //Pattern类有一个splitAsStream方法来按照给出的正则表达式分割一个CharSequence对象
        Stream<String> wordsAnotherWay = Pattern.compile("\\PL+")
                .splitAsStream(new String(
                        Files.readAllBytes(Paths.get("alice.txt")), StandardCharsets.UTF_8));
        show("wordsAnotherWay", wordsAnotherWay);

        //按照alice.txt的行来创建一个流
        Stream<String> lines = Files.lines(Paths.get("alice.txt"));
        show("lines", lines);
    }

    public static <T> void show(String title, Stream<T> stream) {
        final int SIZE = 10;
        List<T> firstElements = stream.limit(10).collect(Collectors.toList());
        System.out.print(title + ": ");
        for (int i = 0; i < firstElements.size(); i++) {
            if (i > 0) {
                System.out.print(",");
            }
            if (i < SIZE) {
                System.out.print(firstElements.get(i));
            } else {
                System.out.print("...");
            }
        }
        System.out.println();
    }
}
```

## 3. 流的转换

---

流的转换会产生一个新的流，它的元素派生自另外一个流中的元素。

### 3.1 filter、map、flatMap方法

```java
//返回由与此给定谓词匹配的此流的元素组成的流
Stream<T> filter(Predicate<? super T> predicate)
//返回由给定函数应用于此流的元素的结果组成的流
<R> Stream<R> map(Function<? super T,? extends R> mapper)
//返回由通过将提供的映射函数应用于每个元素而产生的映射流的内容来替换该流的每个元素的结果的流
<R> Stream<R> flatMap(Function<? super T,? extends Stream<? extends R>> mapper)
```

filter方法转换一个流，它的元素与给定的条件相匹配

map方法转换一个流，它的作用是按照给定的函数来转换流中的值

flatMap方法转换一个流，它的作用是将当前流中所有元素产生的结果连接到一起

### 3.2 抽取子流和连接流

抽取子流有两个方法：limit()和skip()

```java
//limit方法获取前n个元素 
//例如获取前100个元素的流
Stream<Double> randoms = Stream.generate(Math::random).limit(100);

//skip方法是丢弃前n个元素，保留n之后的
//获取100以后的元素
Stream<Integer> integers = Stream.iterate(0, x -> x + 1).skip(100);
```

连接流方法：concat()

```java
//contact方法连接两个流
//假设letters("Hello")的结果是["H","e","l","l","o"]的流
Stream<String> combined = Stream.concat(letters("Hello"), letters("World"));
```

注意：连接两个流时，前一个流不能是一个无限流

## 4. 流的终结

---

### 4.1 min和max

```java
//使用min方法终结流，得到流的最小值
Optional<String> smallest = words.min(String::compareToIgnoreCase);
System.out.println("smallest = " + smallest);
 
//使用max方法终结流，得到流的最大值
Optional<String> largest = words.max(String::compareToIgnoreCase);
System.out.println("largest = " + largest);
```

### 4.2 findFirst和findAny

```java
//findFirst方法返回与之匹配的非空集合的第一个值
//注意，如果没有任何值与之匹配就返回一个空的Optional值
Optional<String> first = words.filter(s -> s.startsWith("Q")).findFirst();
 
//findAny方法返回任意匹配的任意一个值
//注意，如果没有任何值与之匹配就返回一个空的Optional值
Optional<String> any = words.filter(s -> s.startsWith("Q")).findAny();
```

### 4.3 anyMatch, allMatch和noneMatch

```java
//如果想知道有任意元素与给出条件匹配用anymatch，返回的是boolean值
boolean anymatch = words.parallel().anymatch(s->s.contains("e"));
 
//查看是否所有元素都与给出条件匹配用allmatch，返回值是boolean值
boolean allmatch = words.parallel().allmatch(s->s.contains("e"));
 
//查看是否没有任何元素与给出条件匹配，返回值是boolean值
boolean nonematch = words.parallel().nonematch(s->s.contains("e"));
```

## 5. Optional类型

---

Optional<T>对象是一种包装器对象，要么包装了T类型对象，要么没有包装任何对象。Optional<T>类型被当作一种更安全的方式，用来代替类型T的引用，这种引用要么引用某个对象，要么为null。

### 5.1 如何使用Optional值

Optional在值存在的时候，才会使用这个值；在值不存在时，它会有一个可替代物。我们先看看Optional类的三个简单方法：

```java
//orElse()方法，如果存在值，就用原值，如果不存在值就使用一个值来替代它
//我们这里用空字符串来替代没有值的情况
String result = optionalString.orElse("");

//orElseGet()方法，后面可以接一个lambda表达式，表示如果不存在值的话就用表达式的值来替代它
String result = optionalString.orElseGet(() -> Locale.getDefault().getDisplayName());

//orElseThrow()方法，如果不存在值，就在方法中抛出对应的异常
String result = optionalString.orElseThrow(IllegalStateException::new);
```

还有一个ifPresent()方法，ifPresent(v->Process v)该方法会接收一个函数，如果在值存在的情况下，它会将值传递给该函数，但是如果在值不存在的情况下，它不会做任何事。

```java
//比如我们想要在值存在的情况下，就把值添加到一个集中
optionalValue.ifPresent(v->result.add(v));
//或者
optionalValue.ifPresent(result::add);
```

### 5.2 不适合使用Optional值的方式

不适合使用Optional值的情况有两种：

需要用到get()方法，因为Optional值在不存在的情况下，使用get方法会抛出NoSuchElementException异常。

```java
Optional<T> optionalValue = ...;
optionalValue.get().somemethod();
//这种方式并不比下面安全
T value = ...;
value.somemethod();
```

需要用到isPresent()方法作非空判断时。

```java
if(optionalValue.isPresent()){
    optionalValue.get().somemethod();
}
//这种方式也并不比下面容易处理
if(value != null){
    value.somemethod();
}
```

### 5.3 创建Optional值

三种方式创建Optional值：

```java
//创建一个空的Optional值
Optional<String> empty = Optional.empty();
//of(T value)方法：value不能为null，如果为null，会抛出NullPointerException
if(value != null){
    Optional<String> os = Optional.of(value);
}
//ofNullable(T value)方法：这个方法相当于empty方法和of方法的桥梁
//在value为null时，它会返回一个空Optional；在value不为null时，它会返回of方法的返回值
Optional<String> ofNullable = Optional.ofNullable(value);
```

### 5.4 用flatMap来构建Optional值的函数

假设你有一个可以产生Optional\<T> 对象的方法f，并且目标类型T有一个可以产生Optional\<U>的方法g，如果他们都是普通方法，那么你可以使用s.f().g()来调用。但是现在s.f()的返回类型是Optional\<T>，而不是类型T，无法调用g方法，这个时候我们就可以利用flatMap方法来组合这两个方法。

```java
Optional<U> result = s.f().flatMap(T::g);
```