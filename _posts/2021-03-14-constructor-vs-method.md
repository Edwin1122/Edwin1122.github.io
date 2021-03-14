---
title: Java constructor VS method
author: Edwin
date: 2020-03-14 19:52:00 +0800
categories: [Java]
tags: [constructor]
---

 构造函数（构造器）是一种特殊的函数。其主要功能是用来在创建对象时初始化对象， 即为对象成员变量赋初始值，总与new运算符一起使用在创建对象的语句中。构造函数与类名相同，可重载多个不同的构造函数。在JAVA语言中，构造函数与C++语言中的构造函数相同，JAVA语言中普遍称之为构造方法。

构造器特性：

1. 如果我们的类当中没有定义任何构造器，系统会给我们默认提供一个无参的构造器。
2. 如果我们的类当中定义了构造器，那么系统就不会再给我们提供默认的无参构造器。

作用：构建创造一个对象。同时可以给我们的属性做一个初始化操作。

下面主要学习构造器和方法的区别：

我们说构造器是一种方法，就象讲澳大利亚的鸭嘴兽是一种哺育动物。（按：老外喜欢打比喻，我也就照着翻译）。要理解鸭嘴兽，那么先必须理解它和其他哺育动物的区别。同样地，要理解构造器，那么就要了解构造器和方法的区别。所有学习java的人，尤其是对那些要认证考试的，理解构造器是非常重要的。下面将简单介绍一下 ，最后用一个表作了些简单的总结。

### 1. 功能和作用的不同

构造器是为了创建一个类的实例。用来创建一个对象，同时可以给属性做初始化。这个过程也可以在创建一个对象的时候用到：Platypus p1 = new Platypus(); 

相反，方法的作用是仅仅是功能函数，为了执行java代码。

### 2. 修饰符，返回值和命名的不同

       构造器和方法在下面三个方便的区别：修饰符，返回值，命名。

      和方法一样，构造器可以有任何访问的修饰： public, protected, private或者没有修饰（通常被package 和 friendly调用）. 不同于方法的是，构造器不能有以下非访问性质的修饰： abstract, final, native, static, 或者 synchronized。

### 3. 返回类型

      方法必须要有返回值，能返回任何类型的值或者无返回值（void），构造器没有返回值，也不需要void。

### 4. 命名

      构造器使用和类相同的名字，而方法则不同。按照习惯，方法通常用小写字母开始，而构造器通常用大写字母开始。

      构造器通常是一个名词，因为它和类名相同；而方法通常更接近动词，因为它说明一个操作。

### 5. 调用：

      构造方法：只有在对象创建的时候才会去调用，而且只会调用一次。

     一般方法：在对象创建之后才可以调用，并且可以调用多次。

### 6. "this"的用法

       构造器和方法使用关键字this有很大的区别。方法引用this指向正在执行方法的类的实例。静态方法不能使用this关键字，因为静态方法不属于类的实例，所以this也就没有什么东西去指向。构造器的this指向同一个类中，不同参数列表的另外一个构造器，我们看看下面的代码：

``` JAVA

public class Platypus { 
    String name; 
    
    Platypus(String input) { 
        name = input; 
    } 
    
    Platypus() { 
        this("John/Mary Doe"); 
    } 
    
    public static void main(String args[]) { 
        Platypus p1 = new Platypus("digger"); 
        Platypus p2 = new Platypus(); 
    } 
} 

```

在上面的代码中，有2个不同参数列表的构造器。第一个构造器，给类的成员name赋值，第二个构造器，调用第一个构造器，给成员变量name一个初始值 "John/Mary Doe".

在构造器中，如果要使用关键字this, 那么，必须放在第一行，如果不这样，将导致一个编译错误。

### 7. "super"的用法

构造器和方法，都用关键字super指向超类，但是用的方法不一样。方法用这个关键字去执行被重载的超类中的方法。看下面的例子：

``` JAVA
class Mammal { 
    void getBirthInfo() { 
        System.out.println("born alive."); 
    } 
} 
 
class Platypus extends Mammal { 
    void getBirthInfo() { 
        System.out.println("hatch from eggs"); 
        System.out.print("a mammal normally is "); 
        super.getBirthInfo(); 
    } 
} 
```

 
在上面的例子中，使用super.getBirthInfo()去调用超类Mammal中被重载的方法。

构造器使用super去调用超类中的构造器。而且这行代码必须放在第一行，否则编译将出错。看下面的例子：

``` JAVA

public class SuperClassDemo { 
    SuperClassDemo() {} 
} 
 
class Child extends SuperClassDemo { 
    Child() { 
        super(); 
    } 
} 
```

 
在上面这个没有什么实际意义的例子中，构造器 Child()包含了 super, 它的作用就是将超类中的构造器SuperClassDemo实例化，并加到 Child类中。

编译器自动加入代码
编译器自动加入代码到构造器，对于这个，java程序员新手可能比较混淆。当我们写一个没有构造器的类，编译的时候，编译器会自动加上一个不带参数的构造器，例如：public class Example {}
编译后将如下代码：

``` JAVA

public class Example { 
    Example() {} 
} 
在构造器的第一行，没有使用super，那么编译器也会自动加上，例如：
public class TestConstructors { 
    TestConstructors() {} 
} 
编译器会加上代码，如下：

public class TestConstructors { 
    TestConstructors() { 
        super; 
    } 
} 
仔细想一下，就知道下面的代码
public class Example {} 
经过会被编译器加代码形如：

public class Example { 
    Example() { 
        super; 
    } 
} 
```

 

### 8. 继承

构造器是不能被继承的。子类可以继承超类的任何方法。看看下面的代码：

``` JAVA

public class Example { 
    public void sayHi { 
    system.out.println("Hi"); 
} 
 
Example() {} 
} 
 
public class SubClass extends Example { 
} 
```

类 SubClass 自动继承了父类中的sayHi方法，但是，父类中的构造器 Example()却不能被继承。
