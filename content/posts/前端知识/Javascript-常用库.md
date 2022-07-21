---
title: Javascript 常用库
date: 2022-06-02T00:42:12+08:00
lastmod: 2022-06-02T00:42:12+08:00
cover: https://aria9766.oss-cn-beijing.aliyuncs.com/blog-pictures/cup_blue.jpg
# images:
#   - /img/cover.jpg
categories:
  - 前端知识
tags:
  - Js常用库
# nolastmod: true
draft: false
---

常用的Js库

<!--more-->

# 一、jQuery

## 使用方式

按jQuery官网提示下载

## 选择器

$(selector)，例如：

```
$('div');
$('.big-div');
$('div > p')
```

**selector类似于CSS选择器。**

## 事件

`$(selector).on(event, func)`绑定事件，例如：

```
$('div').on('click', function (e) {
    console.log("click div");
})
$(selector).off(event, func)
```

删除事件，例如：

```
$('div').on('click', function (e) {
    console.log("click div");

    $('div').off('click');

});
```


当存在多个相同类型的事件触发函数时，可以通过click.name来区分，例如：

```
$('div').on('click.first', function (e) {  // 这里是给第一次点击命名first
    console.log("click div");

    $('div').off('click.first');

});
```

**在事件触发的函数中的return false等价于同时执行：**

1、e.stopPropagation()：阻止事件**向上传递**。
2、e.preventDefault()：阻止事件的默认行为，只阻止了当前的，事件还会向上传递。

## 元素的隐藏、展现

```
$A.hide()：隐藏，可以添加参数，表示消失时间
$A.show()：展现，可以添加参数，表示出现时间
$A.fadeOut()：慢慢消失，可以添加参数，表示消失时间
$A.fadeIn()：慢慢出现，可以添加参数，表示出现时间
```

## 元素的添加、删除

```
$('<div class="mydiv"><span>Hello World</span></div>')：构造一个jQuery对象
$A.append($B)：将$B添加到$A的末尾
$A.prepend($B)：将$B添加到$A的开头
$A.remove()：删除元素$A
$A.empty()：清空元素$A的所有儿子
```

## 对类的操作

```
$A.addClass(class_name)：添加某个类
$A.removeClass(class_name)：删除某个类
$A.hasClass(class_name)：判断某个类是否存在
```

## 对CSS的操作

```
$("div").css("background-color")：获取某个CSS的属性
$("div").css("background-color","yellow")：设置某个CSS的属性
同时设置多个CSS的属性：
$('div').css({
    width: "200px",
    height: "200px",
    "background-color": "orange",
});
```

## 对标签属性的操作

```
('div').attr('id')：获取属性
$('div').attr('id', 'ID')：设置属性
```

## 对HTML内容、文本的操作

不需要背每个标签该用哪种，用到的时候Google或者百度即可。

```
$A.html()：获取、修改HTML内容，可以添加参数
$A.text()：获取、修改文本信息
$A.val()：获取、修改文本的值
```

## 查找

```
$(selector).parent(filter)：查找父元素
$(selector).parents(filter)：查找所有祖先元素
$(selector).children(filter)：在所有子元素中查找
$(selector).find(filter)：在所有后代元素中查找
```

## ajax

不刷新整个页面，只刷新局部内容。通过获取的后端返回值进行拼接。

```
GET方法：

$.ajax({
    url: url,
    type: "GET",
    data: {
    },
    dataType: "json",
    success: function (resp) {

    },

});
POST方法：

$.ajax({
    url: url,
    type: "POST",
    data: {
    },
    dataType: "json",
    success: function (resp) {

    },

});
```

# 二、setTimeout与setInterval

## setTimeout(func, delay)

延时执行函数，delay毫秒后，执行函数func()。

## clearTimeout()

关闭定时器，例如：

```
let timeout_id = setTimeout(() => {  //timeotut_id是唯一id
    console.log("Hello World!")
}, 2000);  // 2秒后在控制台输出"Hello World"

clearTimeout(timeout_id);  // 清除定时器
```

## setInterval(func, delay)

每隔delay毫秒，执行一次函数func()。
第一次在第delay毫秒后执行。

## clearInterval()

关闭周期执行的函数，例如：

```
let interval_id = setInterval(() => {
    console.log("Hello World!")
}, 2000);  // 每隔2秒，输出一次"Hello World"

clearTimeout(interval_id);  // 清除周期执行的函数
```

# 三、requestAnimationFrame

**requestAnimationFrame(func)**

该函数会在下次浏览器刷新页面之前执行一次，通常会用递归写法使其每秒执行60次func函数。调用时会传入一个参数，表示函数执行的时间戳，单位为毫秒。

例如：

```
let step = (timestamp) => {  // 每帧将div的宽度增加1像素
    let div = document.querySelector('div');
    div.style.width = div.clientWidth + 1 + 'px';
    requestAnimationFrame(step);
};

requestAnimationFrame(step);
```

**与setTimeout和setInterval的区别：**

1、requestAnimationFrame渲染动画的效果更好，性能更佳。

该函数可以保证每两次调用之间的时间间隔相同，但setTimeout与setInterval不能保证这点。**setTmeout两次调用之间的间隔包含回调函数的执行时间**；setInterval只能保证按固定时间间隔将回调函数压入栈中，但**具体的执行时间间隔仍然受回调函数的执行时间影响**，因为可能还会有其他函数被压入栈中。

2、**当页面在后台时，因为页面不再渲染，因此requestAnimationFrame不再执行**。但setTimeout与setInterval函数会继续执行。

# 四、Map与Set

## Map

Map 对象保存键值对。

1、用for...of或者forEach可以按插入顺序遍历。

2、键值可以为任意值，包括函数、对象或任意基本类型。

常用API：

```
set(key, value)：插入键值对，如果key已存在，则会覆盖原有的value
get(key)：查找关键字，如果不存在，返回undefined
size：返回键值对数量
has(key)：返回是否包含关键字key
delete(key)：删除关键字key
clear()：删除所有元素
```

## Set

Set 对象允许你存储任何类型的唯一值，无论是原始值或者是对象引用。

用for...of或者forEach可以按插入顺序遍历。

常用API：

```
add()：添加元素
has()：返回是否包含某个元素
size：返回元素数量
delete()：删除某个元素
clear()：删除所有元素
```

# 五、localStorage

可以在用户的浏览器上存储键值对。

常用API：

```
setItem(key, value)：插入
getItem(key)：查找
removeItem(key)：删除
clear()：清空
```

# 六、Json

JSON对象用于序列化对象、数组、数值、字符串、布尔值和null。

常用API：

```
JSON.parse()：将字符串解析成对象
JSON.stringify()：将对象转化为字符串
```

# 七、日期

减法可以直接得到毫秒数的时间差。

返回值为整数的API，数值为1970-1-1 00:00:00 UTC（世界标准时间）到某个时刻所经过的毫秒数：

```
Date.now()：返回现在时刻。
Date.parse("2022-04-15T15:30:00.000+08:00")：返回北京时间2022年4月15日 15:30:00的时刻。
```

与Date对象的实例相关的API：

```
new Date()：返回现在时刻。
new Date("2022-04-15T15:30:00.000+08:00")：返回北京时间2022年4月15日 15:30:00的时刻。
两个Date对象实例的差值为毫秒数
getDay()：返回星期，0表示星期日，1-6表示星期一至星期六
getDate()：返回日，数值为1-31
getMonth()：返回月，数值为0-11
getFullYear()：返回年份
getHours()：返回小时
getMinutes()：返回分钟
getSeconds()：返回秒
getMilliseconds()：返回毫秒
```

# 八、WebSocket

与服务器建立全双工连接。

常用API：

```
new WebSocket('ws://localhost:8080');：建立ws连接。
send()：向服务器端发送一个字符串。一般用JSON将传入的对象序列化为字符串。
onopen：类似于onclick，当连接建立时触发。
onmessage：当从服务器端接收到消息时触发。
close()：关闭连接。
onclose：当连接关闭后触发。
```

# 九、window

```
window.open("https://www.acwing.com")在新标签栏中打开页面。
location.reload()刷新页面。
location.href = "https://www.acwing.com"：在当前标签栏中打开页面。
```

# 十、canvas

https://developer.mozilla.org/zh-CN/doBaseObjectcs/Web/API/Canvas_API/Tutorial