---
title: web游戏开发（二）
date: 2022-05-09T22:35:40+08:00
lastmod: 2022-05-09T22:35:40+08:00
author: Author Name
# avatar: /img/author.jpg
# authorlink: https://author.site
cover: /img/cover.jpg
# images:
#   - /img/cover.jpg
categories:
  - web游戏开发
tags:
  - Django
  - jquery
# nolastmod: true
draft: false
---

web游戏开发日志

<!--more-->

# 二、项目配置

## 1、 项目结构整理

为了防止`urls.py`、`views.py`和`models.py`过于冗长，将其拆分成模块，并添加`static`文件夹存放静态文件，python的特性，需要在文件夹下添加`__init__.py`才会被解释器认定为是`module`，添加后项目结构如下：

```
`-- webapp
    |-- game
    |   |-- __init__.py
    |   |-- admin.py
    |   |-- apps.py
    |   |-- migrations
    |   |   `-- __init__.py
    |   |-- models
    |   |   `-- __init__.py
    |   |-- static
    |   |-- tests.py
    |   |-- urls
    |   |   `-- __init__.py
    |   `-- views
    |       `-- __init__.py
    |-- manage.py
    `-- webapp
        |-- __init__.py
        |-- __pycache__
        |   |-- __init__.cpython-38.pyc
        |   `-- settings.cpython-38.pyc
        |-- asgi.py
        |-- settings.py
        |-- urls.py
        `-- wsgi.py
```

## 2、全局配置的修改

### 1） 时区调整，改为国内时区

```
TIME_ZONE = 'Asia/Shanghai'
```

### 2）将创建的`game`加入到settings中

在 `apps.py`的配置中添加`game`

```
from django.apps import AppConfig


class GameConfig(AppConfig):
    default_auto_field = 'django.db.models.BigAutoField'
    name = 'game'  # 加入game
```

在`settings.py`中修改相关配置

```
# Application definition

INSTALLED_APPS = [
    'game.apps.GameConfig',  # 加入game对应的路由
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
]
```

### 3）静态文件存储位置

```
# Static files (CSS, JavaScript, Images)
# https://docs.djangoproject.com/en/3.2/howto/static-files/

STATIC_ROOT = os.path.join(BASE_DIR, 'static') 
STATIC_URL = '/static/'  
```

其中`BASE_DIR`通过配置文件可知为最上层文件1`webapp`位置，在静态文件中可以存储`js`，`css`和图片等静态文件。分别创建`js`，`css`和`images`文件夹存储相应文件。并且整个游戏有菜单界面，游戏界面和设置界面三个部分，所以在每个对应的下面新建相应文件夹。

### 4）数据库配置

```
# Database
# https://docs.djangoproject.com/en/3.2/ref/settings/#databases

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.sqlite3',
        'NAME': BASE_DIR / 'db.sqlite3',
    }
}
```

当前采用django默认的sqlite3数据库，因为已经能够满足当前需求，后期可以考虑迁移到mysql数据库上。

## 3、js文件的打包

一个页面加载多个`js`文件的时候，加载速度会变慢。如果js文件过多会延长用户的等待时间，又消耗大量cpu，放在页面前面会影响页面渲染，造成用户体验差，因此需要将多个`js`文件合成一个，尽量减少客户端与服务端的交互。

为了便于调试，先手动实现一个`js`文件压缩的脚本命令，以后会改用已有的`js`压缩工具压缩，shell命令如下：

```
#! /bin/bash

JS_PATH=/home/aria/webapp/game/static/js/
JS_PATH_DIST=${JS_PATH}dist/
JS_PATH_SRC=${JS_PATH}src/

find $JS_PATH_SRC -type f -name '*.js' | sort | xargs cat > ${JS_PATH_DIST}game.js
```

并且将脚本文件统一存放到`webapp/scripts`目录下，并通过`chmod +x compress_game_js.sh`给其添加执行权限。

# 三、实现基础的菜单界面

## 1、html文件

在`templates/multiends/web.html`中：

```
{% load static %}

<head>
    <link rel="stylesheet" href="https://cdn.acwing.com/static/jquery-ui-dist/jquery-ui.min.css">
    <script src="https://cdn.acwing.com/static/jquery/js/jquery-3.3.1.min.js"></script>
    <link rel="stylesheet" href="{% static 'css/game.css' %}">
    <script src="{% static 'js/dist/game.js' %}"></script>
</head>

<body style="margin: 0">
    <div id="web_game_01"></div>
    <script>
        $(document).ready(function(){
            let web_game = new WebGame("web_game_01");  # 在WebGame中渲染界面
        });
    </script>
</body>
```

这里用了 Django 语法糖 `{% load static %}` 、 `{% static 'css/game.css' %}` 、 `{% static 'js/dist/game.js' %}` 加载静态文件
未来的界面都是在 `js` 中 `WebGame` 渲染的，通过jquery实现前后端分离，这样就在前端页面渲染，不会给后端服务器压力。

但是这里采用模块方式导入更规范，否则`js`文件会作为全局变量，需要再在`webgame`中将类`export`出来即可使用。

```
{% load static %}

<head>
    <link rel="stylesheet" href="https://cdn.acwing.com/static/jquery-ui-dist/jquery-ui.min.css">
    <script src="https://cdn.acwing.com/static/jquery/js/jquery-3.3.1.min.js"></script>
    <link rel="stylesheet" href="{% static 'css/game.css' %}">
</head>

<body style="margin: 0">
    <div id="web_game_01"></div>
    <script type="module">
        import { WebGame } from "{% static 'js/dist/game.js' %}";
        $(document).ready(function () {
            let web_game = new WebGame("web_game_01");
        });
    </script>
</body>
```



## 2、 js文件与css文件编写

在`static/js/src`中新建`zbase.js`文件（便于shell按照字典序打包，会放到最后打包）：

```
class WebGame {
    constructor(id) {
        this.id = id;
        this.$web_game = $('#' + id);
        this.menu = new WebGameMenu(this);  # 菜单界面
        this.playground = new WebGamePlayground(this);  # 游戏界面

        this.start();
    }

    start() {
    }
}
```

这里的WebGame对应了html文件中的内容，然后开始初步搭建我们的菜单界面和游戏界面，以下是菜单界面编写:

```
class WebGameMenu {
    constructor(root) {
        this.root = root;
        this.$menu = $(`
<div class="ac-game-menu">
    <div class="ac-game-menu-field">
        <div class="ac-game-menu-field-item ac-game-menu-field-item-single-mode">
            单人模式
        </div>
        <br>
        <div class="ac-game-menu-field-item ac-game-menu-field-item-multi-mode">
            多人模式
        </div>
        <br>
        <div class="ac-game-menu-field-item ac-game-menu-field-item-settings">
            设置
        </div>
    </div>
</div>
`);
        this.root.$web_game.append(this.$menu);
        this.$single_mode = this.$menu.find('.ac-game-menu-field-item-single-mode');
        this.$multi_mode = this.$menu.find('.ac-game-menu-field-item-multi-mode');
        this.$settings = this.$menu.find('.ac-game-menu-field-item-settings');

        this.start();
    }

    start() {
        this.add_listening_events();
    }

    add_listening_events() {
        let outer = this;
        this.$single_mode.click(function(){
            outer.hide();
            outer.root.playground.show();
        });
        this.$multi_mode.click(function(){
            console.log("click multi mode");
        });
        this.$settings.click(function(){
            console.log("click settings");
        });
    }

    show() {  // 显示menu界面
        this.$menu.show();
    }

    hide() {  // 关闭menu界面
        this.$menu.hide();
    }
}
```

`js`语言虽然不太熟，但是基本`api`学会就能上手使用，为了便于区分，用`$`符号代表当前文件下的变量。`constructor`就是默认的构造函数；`show()`与`hide()`是对应的页面`api`；每个`div`可以拥有多个变量名，便于后序渲染；`find`可以通过`div`名找到对应标签；可以利用`console.log()`函数在浏览器页面`F12`进行调试。

另外，`div`命名为了防止冲突，可以让名字长一些，可以起多个名字，并且通过同一个`div`名实现批量管理。

接下来，为`js`文件添加`css`样式,在对应的css文件目录下，`vim game.css`：

```
.ac-game-menu {
    width: 100%;
    height: 100%;
    background-image: url("/static/image/menu/background.gif");
    background-size: 100% 100%;
    user-select: none;
}

.ac-game-menu-field {
    width: 20vw;
    position: relative;
    top: 40vh;
    left: 19vw;
}

.ac-game-menu-field-item {
    color: white;
    height: 7vh;
    width: 18vw;
    font-size: 6vh;
    font-style: italic;
    padding: 2vh;
    text-align: center;
    background-color: rgba(39,21,28, 0.6);
    border-radius: 10px;  # 圆角特效
    letter-spacing: 0.5vw;
    cursor: pointer;  # 鼠标变成小手形状，这里可以找其他补丁DIY一下
}

.ac-game-menu-field-item:hover {  # 实现鼠标悬浮时，div变为之前的1.2倍大，并且在100ms内渐变
    transform: scale(1.2);  
    transition: 100ms;
}
```

`css`文件的格式更为简单，但是各种参数非常多，比如`vh`和`vw`就是相对于窗口的大小，会随窗口改变，而且各种参数也需要不断调整看效果，所以前端和算法的共同点之一就是调参了吧。

## 3、view视图函数编写

在`views`文件夹下，`vim index.py`，这里使用`django`下的`render`函数，查阅资料发现这里的位置是相对于`templates`文件夹的（但是初始生成的文件夹里没有这个文件夹，需要自己新建，这里倒是有点奇怪）:

```
from django.shortcuts import render


def index(request):
    return render(request, "multiends/web.html")  # 注意这里默认render函数的位置是相对于templates文件夹的
```

## 4、urls路由编写

`django`的`url`系统只需要在根目录下配置好相应路径，然后再在对应的路径下扩展路由即可，可以实现路由结构的层次化，便于后面的功能拓展：

```
"""webapp URL Configuration

The `urlpatterns` list routes URLs to views. For more information please see:
    https://docs.djangoproject.com/en/3.2/topics/http/urls/
Examples:
Function views
    1. Add an import:  from my_app import views
    2. Add a URL to urlpatterns:  path('', views.home, name='home')
Class-based views
    1. Add an import:  from other_app.views import Home
    2. Add a URL to urlpatterns:  path('', Home.as_view(), name='home')
Including another URLconf
    1. Import the include() function: from django.urls import include, path
    2. Add a URL to urlpatterns:  path('blog/', include('blog.urls'))
"""
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('', include('game.urls.index')),  #  这里直接将路径路由到game/urls/index.py文件中
    path('admin/', admin.site.urls),
]
```

再在对应的`urls`文件夹下对应所需的内容即可，这里不做赘述。

## 5、测试

以上，将我们的js文件通过自己写的压缩函数打包运行，就可以看到我们项目的雏形了(虽然只有菜单页面)，如果有问题，可以以在`F12`中调试错误。

这里记得补图