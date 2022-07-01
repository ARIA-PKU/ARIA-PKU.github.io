---
title: React
date: 2022-06-03T22:23:09+08:00
lastmod: 2022-06-03T22:23:09+08:00

cover: https://oss.surfaroundtheworld.top/blog-pictures/sakura_tree.jpg
# images:
#   - /img/cover.jpg
categories:
  - 前端知识
tags:
  - React
# nolastmod: true
draft: false
---

React框架

<!--more-->

# 一、React小知识

## DOM树

虚拟DOM树，每次会重新修改虚拟DOM树，但是只会重新渲染修改过的实际DOM树。

##JSX
React中的一种语言，会被Babel编译成标准JavaScript。

# 二、环境配置

用的是windows环境，mac和linux也差不太多（因为都用的命令行处理）

**1、安装nodejs** 

​	官网：https://nodejs.org/en/

**2、安装create-react-app**

```
npm i -g create-react-app
```

**3、创建React App**

在目标目录下打开Git Bash或Powershell，在终端中执行：

```
create-react-app react-app  # 可以替换为其他app名称

cd react-app
npm start  # 启动应用
```

# 三、ES6语法糖

## 使用bind()函数绑定this取值

在JavaScript中，函数里的this指向的是执行时的调用者，而非定义时所在的对象。

例如：

```
const person = {
  name: "aria",
  talk: function() {
    console.log(this);
  }
}

person.talk();

const talk = person.talk;
talk();
```


运行结果：

```
{name: 'aria', talk: ƒ}
Window
```


bind()函数，可以绑定this的取值。例如：

```
const talk = person.talk.bind(person);
```

## 箭头函数的简写方式

```
const f = (x) => {
  return x * x;
};
```

可以简写为：

```
const f = x => x * x;
```

## 箭头函数不重新绑定this的取值

例如：

```
const person = {
  talk: function() {
    setTimeout(function() {
      console.log(this);
    }, 1000);
  }
};

person.talk();  // 输出Window
```

```
const person = {
  talk: function() {
    setTimeout(() => {
      console.log(this);
    }, 1000);
  }
};

person.talk();  // 输出 {talk: f}
```

## 对象的解构

例如：

```
const person = {
  name: "aria",
  age: 18,
  height: 180,
};

const {name : nm, age} = person;  // nm是name的别名
```

## 数组和对象的展开

例如：

```
let a = [1, 2, 3];
let b = [...a];  // b是a的复制
let c = [...a, 4, 5, 6];
const a = {name: "yxc"};
const b = {age: 18};
const c = {...a, ...b, height: 180};
```

## Named 与 Default exports

Named Export：可以export多个，import的时候需要加大括号，名称需要匹配
Default Export：最多export一个，import的时候不需要加大括号，可以直接定义别名

# 四、Component

## 项目创建

创建项目box-app：

```
create-react-app box-app
cd box-app
npm start
```


安装bootstrap库：

```
npm i bootstrap
```


bootstrap的引入方式：

```
import 'bootstrap/dist/css/bootstrap.css';
```

快捷补全技巧：

`imrc` + `tab` 自动补全引用

`cc` + `tab` 类组件

## 创建多个元素

当子节点数量大于1时，可以用`<div>`或`<React.Fragment>`将其括起来。后者不会在浏览器中显示出来，只是使得其合法。

## 内嵌表达式

JSX中使用`{}`嵌入表达式。类似js中用`${}`写入文本。

```
import React, { Component } from 'react';
class Box extends Component {
    state = { 
        x: 0,
     } 
    render() { 
        return (
            <React.Fragment>
                <div>{this.state.x}</div>  // 引入外部变量
                <div>{this.toString()}</div>  // 引入外部函数
                <h1>Hello World</h1>
                <button>left</button>
                <button>right</button>
            </React.Fragment>
        );
    }

    toString() {
        const {x} = this.state;  // 解构函数
        return `x: ${x}`
    }
}
 
export default Box;
```

## 设置属性

**class -> className**

CSS属性：**background-color -> backgroundColor**，其它属性类似，蛇形命名法改驼峰命名法

## 数据驱动改变style

可以通过参数变化，改变style

## 渲染列表

使用map函数

每个元素需要具有唯一的key属性，用来帮助React快速找到被修改的DOM元素。这个key属性不会在前端代码中出现。

```
{this.state.colors.map(color => (
     <div key={color}>{color}</div>
))}
```

## Conditional Rendering

利用逻辑表达式的短路原则。

与表达式中 `expr1 && expr2`，当expr1为假时返回expr1的值，否则返回expr2的值
或表达式中 `expr1 || expr2`，当expr1为真时返回expr1的值，否则返回expr2的值

## 绑定事件

注意妥善处理好绑定事件函数的this

```
// 两种处理方式，一种使用bind()函数，一种使用箭头函数

	handleClickLeft = () => {  //箭头函数，不会修改this的绑定，推荐
        console.log("click left", this);
    }

    handleClickRight() {
        console.log("click right", this);
    }

    render() { 
        return (
            <React.Fragment>
                <div>{this.state.x}</div>
                <div style={this.styles}>{this.toString()}</div>
                <h1>Hello World</h1>
                
                <button onClick={this.handleClickLeft} className="btn btn-primary m-2">left</button>
                <button onClick={this.handleClickRight.bind(this)} className="btn btn-success m-2">right</button>
                {this.state.colors.map(color => (
                    <div key={color}>{color}</div>
                ))}
            </React.Fragment>
        );
    }

```

## 修改state

需要使用this.setState()函数

每次调用this.setState()函数后，会重新调用this.render()函数，用来修改虚拟DOM树。React只会修改不同步的实际DOM树节点。

## 给事件函数添加参数

括号内添加参数即可

# 五、组合Components

## 创建Boxes组件

Boxes组件中包含一系列Box组件。

## 从上往下传递数据

通过this.props属性可以从上到下传递数据。

**传递子节点：**

通过this.props.children属性传递子节点

## 从下往上调用函数

注意：每个组件的this.state只能在组件内部修改，不能在其他组件内修改。可以理解为私有变量。

## 每个维护的数据仅能保存在一个this.state中

不要直接修改this.state的值，因为setState函数可能会将修改覆盖掉。只存一份，否则会存在无法同步修改的问题。

JS中传递对象都是传递的对象的引用。

## 创建App组件

包含：

```
导航栏组件
Boxes组件
```


注意：

要将多个组件共用的数据存放到最近公共祖先的this.state中。

## 无状态函数组件

当组件中没有用到this.state时，可以简写为无状态的函数组件。

函数的传入参数为props对象

## 组件的生命周期

**Mount周期**，执行顺序：constructor() -> render() -> componentDidMount()

Mount周期中的componentDidMount()函数在执行时表明，当前页面元素加载完毕，因此可以通过ajax请求获取数据填充上去



**Update周期**，执行顺序：render() -> componentDidUpdate()

Update周期中的componentDidUpdate(prevProps, prevState)可以填两个参数，应用在：对比 this.state 与 prevState的区别，从而去更新数据库



**Unmount周期**，执行顺序：componentWillUnmount()

Unmount周期中的componentWillUnmount()是在删除前执行函数，因此可以在此函数更新全局状态

# 六、路由

## Web分类

静态页面：页面里的数据是写死的

动态页面：页面里的数据是动态填充的

- 后端渲染：数据在后端填充，每次返回当前完整页面
- 前端渲染：数据在前端填充，每次返回模板，用到什么数据再向后端请求什么数据。切换页面的时候，不会向后端发出请求，而是通过JS渲染新的前端页面。

## 安装环境

VSCODE安装插件：Auto Import - ES6, TS, JSX, TSX

安装Route组件：npm i react-router-dom

## Route组件介绍

BrowserRouter：所有需要路由的组件，都要包裹在BrowserRouter组件内

Link：跳转到某个链接，to属性表示跳转到的链接 

Routes：类似于C++中的switch，匹配第一个路径

Route：路由，path属性表示路径，element属性表示路由到的内容

## URL中传递参数

解析URL：

```
<Route path="/linux/:chapter_id/:section_id/" element={<Linux />} />
```


获取参数，类组件写法：

```
import React, { Component } from 'react';
import { useParams } from 'react-router-dom';

class Linux extends Component {
    state = {  } 
    render() {
        console.log(this.props.params);
        return <h1>Linux</h1>;
    }
}

export default (props) => (
    <Linux
        {...props}
        params={useParams()}
    />
)
```


函数组件写法：

```
import React, { Component } from 'react';
import { useParams } from 'react-router-dom';

const Linux = () => {
    console.log(useParams());
    return (<h1>Linux</h1>);
}

export default Linux;
```

## Search Params传递参数

写在路由地址里面，获取`?`后面的内容。

类组件写法：

```
import React, { Component } from 'react';
import { useSearchParams } from 'react-router-dom';

class Django extends Component {
    state = {
        searchParams: this.props.params[0],  // 获取某个参数
        setSearchParams: this.props.params[1],  // 设置链接中的参数，然后重新渲染当前页面
    }

    handleClick = () => {
        this.state.setSearchParams({
            name: "abc",
            age: 20,
        })
    }
    
    render() {
        console.log(this.state.searchParams.get('age'));
        return <h1 onClick={this.handleClick}>Django</h1>;
    }

}

export default (props) => (
    <Django
        {...props}
        params={useSearchParams()}
    />
);
```


函数组件写法：

```
import React, { Component } from 'react';
import { useSearchParams } from 'react-router-dom';

const Django = () => {
    let [searchParams, setSearchParams] = useSearchParams();
    console.log(searchParams.get('age'));
    return (<h1>Django</h1>);
}

export default Django;
```

## 重定向

使用Navigate组件可以重定向。

```
<Route path="*" element={ <Navigate replace to="/404" /> } />
```

嵌套路由

```
<Route path="/web" element={<Web />}>
    <Route index path="a" element={<h1>a</h1>} />
    <Route index path="b" element={<h1>b</h1>} />
    <Route index path="c" element={<h1>c</h1>} />
</Route>
```

注意：需要在父组件中添加`<Outlet />`组件，用来填充子组件的内容。

注意：需要在父组件中添加`<Outlet />`组件，用来填充子组件的内容。

# 七、Redux

redux将所有数据存储到树中，且树是唯一的。使得各个模块的数据与全局数据交互。

## Redux基本概念

store：存储树结构。

state：维护的数据，一般维护成树的结构。

reducer：对state进行更新的函数，每个state绑定一个reducer。传入两个参数：当前state和action，返回新state。

action：一个普通对象，存储reducer的传入参数，一般描述对state的更新类型。

dispatch：传入一个参数action，对整棵state树操作一遍。

subscribe：订阅

## React-Redux基本概念

Provider组件：用来包裹整个项目，其store属性用来存储redux的store对象。一定要传store否则报错。

connect(mapStateToProps, mapDispatchToProps)函数：用来将store与组件关联起来。每次调用完dispatch函数重新渲染。

- mapStateToProps：每次store中的状态更新后调用一次，用来更新组件中的值。
- mapDispatchToProps：组件创建时调用一次，用来将store的dispatch函数传入组件。

## 安装

```
npm i redux react-redux @reduxjs/toolkit
```

# 八、部署到云端

```
npm run build
tar -zcvf build.tar.gz build/
docker cp build.tar.gz django_server:/home/aria
tar -zxvf build.tar.gz
```

