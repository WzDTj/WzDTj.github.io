---
layout: post
title:  "Swift inout 修饰符"
date:   2017-06-02 10:30:00 +0800
categories: 
---
## 描述

函数参数一般是常量不允许修改的。但是如果想要修改其传入的参数的话 `inout` 修饰符就派上用场了,有点 C++ 中 `&` 的味道。

## 例子

```
func foo(input: Int) {
  input = 1 // error
}

func foo2(input: inout Int) {
  input = 0
}

var aNumber = 10
print(aNumber) // 10

foo2(x: &aNumber)
print(aNumber) // 0
```

## 其他  

在 SE-0031 提案之前，`inout` 修饰符在参数名之前的。

```
func foo(inout x: Int)
```

在 SE-0031 提案之后，`inout` 修饰符在类型之前，参数名之后。

```
func foo(x: inout Int)
```

这样做的目的是为了和外部标签进行区分。

## 参考

- [inout 关键字位置的变化](https://www.swiftcafe.io/2016/05/14/swfit3-inout)
- [The Swift Programming Language (Swift 3.1)](https://developer.apple.com/library/content/documentation/Swift/Conceptual/Swift_Programming_Language/Functions.html#//apple_ref/doc/uid/TP40014097-CH10-ID158)
