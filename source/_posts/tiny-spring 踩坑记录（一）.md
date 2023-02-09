---
title: tiny-spring 踩坑记录（一）可变长泛型参数与堆污染
categories:
- Java
tags:
- Generics
- 泛型
---

## 可变长泛型参数与堆污染

### 定义

> 什么是堆污染

一个类型为泛型 `T` 的变量被指向一个类型不是 `T` 的对象，这就是堆污染。典型的情况是

```java
List<T> listOfTs = new ArrayList<>(Arrays.asList(t));
List<E> listOfEs = (List<E>)(Object)listOfTs; // 此时指向了含有 T 的 List, 但是不会抛出异常，只是在 Object -> List<E> 的转型中抛出警告

E e = listOfEs.get(0);  // 运行时错误，java.lang.ClassCastException
```

之所以编译器无法在转型时检查出类型转换错误，是因为 Java 泛型是通过类型擦除实现的，在泛型代码内部无法获得任何有关泛型参数类型的信息。

一个更好一点的关于可变长泛型参数和堆污染一起的例子，来自 [Oracle 官方文档](<https://docs.oracle.com/javase/tutorial/java/generics/nonReifiableVarargsType.html>)：

```java
public static void faultyMethod(List<String>... l) {
    Object[] objectArray = l;     // Valid
    objectArray[0] = Arrays.asList(42);
    String s = l[0].get(0);       // ClassCastException thrown here
}
```

关于泛型的[几大限制](https://docs.oracle.com/javase/tutorial/java/generics/restrictions.html)中，最明显的一个就是不能创造泛型数组，如

```java
List<Integer>[] arrayOfLists = new List<Integer>[2];  // compile-time error
```

但是这一限制在可变长的泛型参数中被打破了。在可变长参数中，一个数组会在函数调用时建立（Effective Java，Item 53），每次调用都会引起一次数组分配和初始化。当可变长参数是泛型时，一个泛型数组就被创造出来了。

### 实操时的问题

我在 tiny-spring 中写了一个方法，给定注解参数，获取所有带有该注解的类。

```java
public static Set<Class<?>> getClassSetByAnnotation(Class<? extends Annotation>... cls) {
    Set<Class<?>> classSet = new HashSet<>();

    for (Class<?> c : getClassSet(basePkg)) {
        for (Class<? extends Annotation> annotationCls : cls) {
            if (c.isAnnotationPresent(annotationCls)) {
                classSet.add(c);
            }
        }
    }
    return classSet;
}

```

这里就出现了一个可变长泛型参数的问题。另一个隐藏的问题是，这个方法并没有对传入的注解参数的数量检查参数为零的情况，而可变长参数允许传入零个参数。一个优美一点的解决方案是：

```java
@SafeVarargs
public static Set<Class<?>> getClassSetByAnnotation(Class<? extends Annotation> firstCls, Class<? extends Annotation>... cls) {
    Set<Class<?>> classSet = new HashSet<>();

    for (Class<?> c : getClassSet(basePkg)) {
        if (c.isAnnotationPresent(firstCls)) {
            classSet.add(c);
        }
        for (Class<? extends Annotation> annotationCls : cls) {
            if (c.isAnnotationPresent(annotationCls)) {
                classSet.add(c);
            }
        }
    }
    return classSet;
}
```

这样既省去明显的检查参数数量，又能在调用处强制调用者传入至少一个参数，这也是这个方法的本意。

### @SafeVarargs 的使用标准

如上所见，可以将报警用 `@SafeVarargs` 注解掉。但是我们知道，这个注解仅仅是消除编译警告，完全不能保证消除这个错误。那么什么时候应该使用这个注解呢？

在 StackOverflow 有一个票的[回答](https://stackoverflow.com/a/14252221)：

If your method has an argument of type `T...` (where T is any type parameter), then:

- Safe: If your method only depends on the fact that the elements of the array are instances of `T`
- Unsafe: If it depends on the fact that the array is an instance of `T[]`

Effective Java 也表示了类似的意思：如果你的方法使用可变长泛型参数的时候，只是用来一次传入多个参数，并且实际使用的时候只是把它们当作多个参数来看，那么就没有问题。如果你的方法体中把可变长参数当作数组来使用，那就会有问题，不要使用这个注解。