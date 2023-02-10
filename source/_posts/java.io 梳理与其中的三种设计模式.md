---
title: java.io 梳理与其中的三种设计模式
categories:
- Design Patterns
date: 2019-8-2
---

# java.io 梳理与其中的三种设计模式

以 `Reader` 抽象类下的类为例。它们的主要构造函数如下：

```java
FilterReader(Reader)
BufferedReader(Reader)

FileReader(File)
StringReader(String)
InputStreamReader(InputStream [, Charset])
```

![Reader UML](/content/images/2019/ReaderUML.png)

大致上，`java.io` 可以分为：

1. 字节流操作：`InputStream`、`OutputStream`
2. 字符操作：`Reader`、`Writer`
3. 针对磁盘操作：`File`

## 适配器模式 Adapter pattern

`InputStreamReader` 是字节流与字符流之间的桥梁，它的实现使用了适配器模式。适配器模式通过聚合将某个接口适配成为另一个接口来方便使用，类图如下：

![InputStreamReader Adapter Pattern](/content/images/2019/InputStreamReaderAdapterPattern.png)

在开头的代码可以看到，`InputStreamReader` 的构造器需要传入一个字节流 `InputSream`，以及可选的字符集。如果没有传入字符集就会使用默认的。通过持有一个 `InputStream`，`InputStreamReader` 拥有了该接口的功能。同时，`InputStreamReader` 还是一个实现了 `Reader` 类，这样一来它就把 `InputStream` 适配成了一个 `Reader` 类。

## 装饰器模式 Decorator pattern

这个应该是 `java.io` 更为人熟知的，装饰器模式与适配器模式都可以看作包装模式 Wrapper。通过对类的层层包裹，装饰器模式增强了目标类的功能。类图示意如下：

![IO Decorator Pattern](/content/images/2019/IODecoratorPattern.png)

`Decorator` 是 `Component` 的实现类，但是其内部也持有一个 `Decorator` 类。不同的 `Decorator` 类下的 `ConcreteDecorator` 子类包含了需要增强的功能。

在 `java.io` 中的典型使用：

```java
FilterReader pbr = new PushbackReader(
        new BufferedReader(new InputStreamReader(System.in)))
```

### 二者的不同

装饰器模式更多的是对目标类的功能上的增强；适配器没有对功能进行增进，只是为了使目标类能适应调用方的接口，方便被调用。

## 模板方法

在模板方法模式中，大的算法框架在抽象父类中被定义好，某些业务相关的实现则延迟到子类。

在 `Reader` 抽象类中，`read` 方法的默认实现都留给了被重载的 `read(char[], int, int)`，如下：

```java
int read()
int read(char[] cbuf)

abstract int read(char[] cbuf, int offset, int len)
```

具体的 `read` 的实现留给子类。

