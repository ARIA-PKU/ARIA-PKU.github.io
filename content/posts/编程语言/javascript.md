---
title: Javascript
date: 2022-06-15T23:22:22+08:00
lastmod: 2022-06-15T23:22:22+08:00

cover: http://oss.surfaroundtheworld.top/blog-pictures/6_22/js.jpg
# images:
#   - /img/cover.jpg
categories:
  - 编程语言
tags:
  - JavaScript
# nolastmod: true
draft: false
---

写项目中用到的一些JS语法

<!--more-->

# 一、高阶函数

这些在python中也有用过，基本各种编程语言之间的函数都是相互借鉴。

## 1、map()方法

`map()`方法定义在JavaScript的`Array`中，我们调用`Array`的`map()`方法，传入我们自己的函数，就得到了一个新的`Array`作为结果：

```
function pow(x) {
    return x * x;
}

var arr = [1, 2, 3, 4, 5, 6, 7, 8, 9];
var results = arr.map(pow); // [1, 4, 9, 16, 25, 36, 49, 64, 81]
console.log(results);
```

注意：`map()`传入的参数是`pow`，即函数对象本身。

## 2、reduce()方法

Array的`reduce()`把一个函数作用在这个`Array`的`[x1, x2, x3...]`上，这个函数必须接收两个参数，`reduce()`把结果继续和序列的下一个元素做累积计算，其效果就是：

```
[x1, x2, x3, x4].reduce(f) = f(f(f(x1, x2), x3), x4)
```

比方说对一个`Array`求和，就可以用`reduce`实现：

```
var arr = [1, 3, 5, 7, 9];
arr.reduce(function (x, y) {
    return x + y;
}); // 25
```

## 3、 filler()方法

------

filter也是一个常用的操作，它用于把`Array`的某些元素过滤掉，然后返回剩下的元素。

和`map()`类似，`Array`的`filter()`也接收一个函数。和`map()`不同的是，`filter()`把传入的函数依次作用于每个元素，然后根据返回值是`true`还是`false`决定保留还是丢弃该元素。

例如，在一个`Array`中，删掉偶数，只保留奇数，可以这么写：

```
var arr = [1, 2, 4, 5, 6, 9, 10, 15];
var r = arr.filter(function (x) {
    return x % 2 !== 0;
});
r; // [1, 5, 9, 15]
```

利用`filter`，可以巧妙地去除`Array`的重复元素：

```
var
    r,
    arr = ['apple', 'strawberry', 'banana', 'pear', 'apple', 'orange', 'orange', 'strawberry'];
    r = arr.filter(function (element, index, self) {
    return self.indexOf(element) === index;
});
```

去除重复元素依靠的是`indexOf`总是返回第一个元素的位置，后续的重复元素位置与`indexOf`返回的位置不相等，因此被`filter`滤掉了。

# 二、闭包

一个简单的概括：函数 + 引用环境 = 闭包。

其作用是在没有class机制机制的函数语言中，可以封装私有变量，并且它的状态可以完全对外隐藏起来。

```
function lazy_sum(arr) {
    var sum = function () {
        return arr.reduce(function (x, y) {
            return x + y;
        });
    }
    return sum;
}
```

当我们调用`lazy_sum()`时，返回的并不是求和结果，而是求和函数：

```
var f = lazy_sum([1, 2, 3, 4, 5]); // function sum()
```

调用函数`f`时，才真正计算求和的结果：

```
f(); // 15
```

在这个例子中，我们在函数`lazy_sum`中又定义了函数`sum`，并且，内部函数`sum`可以引用外部函数`lazy_sum`的参数和局部变量，当`lazy_sum`返回函数`sum`时，相关参数和变量都保存在返回的函数中，这种称为“闭包（Closure）”的程序结构拥有极大的威力。

# 三、箭头函数

ES6标准新增了一种新的函数：Arrow Function（箭头函数）。

```
x => x * x
```

上面的箭头函数相当于：

```
function (x) {
    return x * x;
}
```

箭头函数相当于匿名函数，并且简化了函数定义。箭头函数有两种格式，一种像上面的，只包含一个表达式，连`{ ... }`和`return`都省略掉了。还有一种可以包含多条语句，这时候就不能省略`{ ... }`和`return`：

```
x => {
    if (x > 0) {
        return x * x;
    }
    else {
        return - x * x;
    }
}
```

箭头函数看上去是匿名函数的一种简写，但实际上，箭头函数和匿名函数有个明显的区别**：箭头函数内部的`this`是词法作用域，由上下文确定。**

# 四、JavaScript中var、let、const区别？

使用var声明的变量，其作用域为该语句所在的函数内，且存在变量提升现象，提升指变量声明语句位置的改变；

使用let声明的变量，其作用域为该语句所在的代码块内，不存在变量提升；

使用const声明的是常量，在后面出现的代码中不能再修改该常量的值，const要求的是指针地址不变而不是指针指向对象内部发生了什么。

# 参考文章

1、[廖雪峰的官方网站](https://www.liaoxuefeng.com/)