---
title: Vue3
date: 2022-06-10T15:44:37+08:00
lastmod: 2022-06-10T15:44:37+08:00

cover: https://oss.surfaroundtheworld.top/blog-pictures/sakura_tree.jpg
# images:
#   - /img/cover.jpg
categories:
  - 前端知识
tags:
  - vue3
# nolastmod: true
draft: false
---

Vue3的使用心得

<!--more-->

前端渲染：在第一次打开页面的时候，会返回所有页面的js代码，打开新页面的时候可以无须向后端发送请求。



## script部分

export default对象的属性：

- name：组件的名称
- components：存储`<template>`中用到的所有组件
- props：存储父组件传递给子组件的数据
- watch()：当某个数据发生变化时触发
- computed：动态计算某个数据
- setup(props, context)：初始化变量、函数
  - ref定义变量，可以用.value属性重新赋值
  - reactive定义对象，不可重新赋值
  - props存储父组件传递过来的数据
  - context.emit()：触发父组件绑定的函数

## template部分

- `<slot></slot>`：存放父组件传过来的children。
- v-on:click或@click属性：绑定事件
- v-if、v-else、v-else-if属性：判断
- v-for属性：循环，:key循环的每个元素需要有唯一的key
- v-bind:或:：绑定属性
- v-model：实现表单元素和数据的双向绑定

## style部分

`<style>`标签添加scope属性后，不同组件间的css不会相互影响。

## 第三方组件

- view-router包：实现路由功能。
- vuex：存储全局状态，全局唯一。
  - state: 存储所有数据，可以用modules属性划分成若干模块
  - getters：根据state中的值计算新的值
  - mutations：所有对state的修改操作都需要定义在这里，不支持异步，可以通过`$store.commit()`触发
  - actions：定义对state的复杂修改操作，支持异步，可以通过`$store.dispatch()`触发。注意不能直接修改state，只能通过mutations修改state。
  - modules：定义state的子模块