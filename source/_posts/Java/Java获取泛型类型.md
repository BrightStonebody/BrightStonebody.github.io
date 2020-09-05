---
title: Java获取泛型类型
date: 2020-07-23 20:05:38
categories:
- Java
tags:
- Java
---

## 获取泛型class的通用方法

```java
Type type = ((ParameterizedType)obj.getClass().getGenericSuperclass()).getActualTypeArguments()[0];

```

因为有泛型擦除, 并不是所有情况下都能获取到泛型实际类型. 只有在泛型类型明确时, 才能在运行时获取到泛型类型

## 无法获取到泛型类型

```java
// 1. 
class SuperClass<T>{

}
SuperClass<String> object = SuperClass<String>()

// 2.
SubClass<T> extends SuperClass<T>{

}
SubClass<String> object = SubClass<Strng>()
```

## 可以获取到泛型类型

```java
// 1. 子类继承父类时, 明确了泛型类型
SubClass extends SuperClass<String>{

}
// 2. 匿名内部类, 子类继承父类的特殊情况
abstract class SuperClass<T>{

}
SuperClass<String> object = new SuperClass<String>(){
    ...
}
```
