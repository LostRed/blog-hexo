---
title: Java中的lambda表达式
date: 2020-06-23 21:07:26
categories:
  - Java
tags:
  - Java
---

## 1. 什么是lambda表达式

---

lambda表达式是JDK8的一个新特性。lambda表达式采用一种简洁的语法定义代码块，该代码块可传递，可以在以后执行一次或多次。

### 1.1 为什么要引入lambda表达式

Java是一个面向对象的语言。在过去的旧版本中，传递一段代码必须先定义一个类(Class)
，然后在类中定义一个方法，该方法包含所需的代码，最后通过创建(new)
一个新的对象来调用方法实现所需的代码，这样往往增加了程序的代码量。而一些简单的方法往往不需要按照原有的特性来写，lambda表达式就能很好的定义一个简单的方法，而不需要去写一个类(
Class)。

### 1.2 lambda表达式的语法

lambda表达式的语法与方法类似，它不需要命名，但必须有一个“()”，之后用“->”连接后面的代码块。

有参数的lambda表达式

```java
(Type parameter1, Type parameter2) -> {
    //do some work
};
```

无参数的lambda表达式

```java
() -> {
    //do some work
};
```

如果可以推导出一个lambda表达式的参数类型，则可以忽略其类型。如果lambda表达式代码块中只有一个表达式时，无须指定返回类型，其返回类型由上下文推导得出。如下面这段代码，这个lambda表达式赋值给一个字符串比较器，所以编译器可以推导出first和second必然是字符串。

```java
Comparator<String> comp = (first, second) -> {
    first.length - second.length
};
```

如果方法只有一个参数，而且这个参数的类型可以推导得出，就可以省略小括号。如下面这段代码，ActionListener下*只有一个*
actionPerformed方法且该法方法的参数类型只有一个ActionEvent类型。

```java
ActionListener listener = event -> {
    //do some work
};
```

需要指出，如果lambda表达式中的代码块存在多个分支，其中有的分支返回一个值，另外的分支不返回值，这是不合法的。

## 2. lambda表达式的应用

---

### 2.1 lambda表达式在函数式接口的应用

只有一个抽象方法的接口，在创建一个变量时，可以指向一个lambda表达式，这种接口成为函数式接口。如上述提到的ActionListener。

### 2.2 lambda表达式改写成方法引用

当lambda表达式中的代码块仅仅只调用了一个方法而不做其他操作时，可以将lambda表达式改写成以下四种形式：

```java
1. object::instanceMethod	    //等同于 x -> object.Method(x)
2. Class::instanceMethod	    //等同于 (x, y) -> x.Method(y)，x为隐式参数，之后的参数传递到方法
3. Class::staticMethod		    //等同于 (x, y) -> Class.Method(x, y)，所有参数传递到静态方法
4. Class::new		            //等同于 x -> new Class(x)，参数由代码上下文决定
```

| 方法引用					| 等价的lambda表达式				     | 类型       |
| separator : : equals		| x -> separator.equals(x)		     | 第一种类型  |
| String : : trim           | x -> x.trim()						 | 第二种类型  |
| String : : concat			| (x, y) -> x.concat(y)				 | 第二种类型  |
| Integer  : : valueOf		| x -> Integer.valueOf(x)			 | 第三种类型  |
| Integer  : : sum			| (x, y)  -> Integer.sum(x, y)		 | 第三种类型  |
| Integer  : : new			| x -> new Integer(x)				 | 第四种类型  |
| Integer[]  : : new		| n -> new Integer[n]				 | 第四种类型  |


## 3.lambda表达式的变量作用域

---

lambda表达式能够捕获代码块作用域以外的变量，不过该变量必须是事实最终变量（即该变量不会再对它赋值）。这样lambda表达式就会将该变量的值复制到这个对象的实例变量中。

```java
public static void repeatMessage(String text, int delay){
    ActionListener listener = event -> {
        System.out.println(text);
        Toolkit.getDefaultToolkit().beep();
    };
    new Timer(delay, listener).start();
}
```

上述代码中的text变量将会被lambda表达式捕获并复制到listener中。
this关键字在lambda表达式中表示创建这个lambda表达式的方法的this参数。

```java
public class Application{
    public void init(){
        ActionListener listener = event -> {
            System.out.println(this.toString());
        }
    }
}
```

上述代码中的this是指Application的实例对象，而不是ActionListener实例的方法。

## 4.处理lambda表达式

---

使用lambda表达式的重点是延迟执行。如果希望立即执行代码，完成没必要将它包装在一个lambda表达式中。之所以希望以后再执行代码，有许多原因：

- 在一个单独的线程中运行代码；
- 多次运行代码；
- 在算法的适当位置运行代码（例如，排序中的比较操作）；
- 发生某种情况时执行代码（如，如点击了一个按钮，数据到达，等等）；
- 只在必要时才运行代码。

以上也是lambda表达式的应用场景。

```java
/**
 *  代码定义了一个函数式接口 IntConsumer，其中lambda表达式实现了accept方法
 *  等价于将 void accept(int value)方法重写为
 *	public void accept(int value){
 *		System.out.println("Countdown: " + (9 - value));
 *	}
 *  所以在循环中将会打印"Countdown: " + (9 - i)
 */
public class Test {
    public static void main(String[] args) {
        repeat(10, count -> System.out.println("Countdown: " + (9 - count)));
    }
    
    public static void repeat(int n, IntConsumer action) {
        for (int i = 0; i < n; i++) {
            action.accept(i);
        }
    }
    
    public interface IntConsumer {
        void accept(int value);
    }
}
```
