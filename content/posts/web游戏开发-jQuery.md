---
title: web游戏开发(三)
date: 2022-05-10T14:59:42+08:00
lastmod: 2022-05-10T14:59:42+08:00
# author: Author Name
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
draft: true
---

web游戏开发日志-jquery的使用

<!--more-->

# 实现基础的游戏界面

## 1、结构与思路

游戏的基本原理比较简单，图片的刷新的频率超过人眼所能识别的范围，画面看起来就是连续的，因此只需要不停地刷新页面即可，这个在`js`中也有现成的`api`可以调用。

在项目中我们将实现单人模式和联机对战两种模式，这里先实现简单的单人模式。首先是`playground`展示界面：

```
class AcGamePlayground {
    constructor(root) {
        this.root = root;
        this.$playground = $(`<div class="ac-game-playground"></div>`);

        // this.hide();
        this.root.$ac_game.append(this.$playground);
        this.width = this.$playground.width();
        this.height = this.$playground.height();
        this.game_map = new GameMap(this);  // 新建地图
        this.players = [];  // 存放加入的玩家
        this.players.push(new Player(this, this.width / 2, this.height / 2, this.height * 0.05, "white", this.height * 0.15, true));

        for (let i = 0; i < 5; i++) {  // 单机模式增加五个其余的玩家
            this.players.push(new Player(this, this.width / 2, this.height / 2, this.height * 0.05, this.get_random_color(), this.height * 0.15, false));
        }

        this.start();
    }

    get_random_color() {
        let colors = ["blue", "red", "pink", "grey", "green"];
        return colors[Math.floor(Math.random() * 5)];
    }

    start() {
    }

    show() {  // 打开playground界面
        this.$playground.show();
    }

    hide() {  // 关闭playground界面
        this.$playground.hide();
    }
}
```

接下来需要确定整体的框架，我们在游戏过程中需要不停渲染的目前有地图，玩家，玩家的各种攻击特效，因此可以创建渲染的基类，所有需要动态渲染的都继承自此基类：

```
let AC_GAME_OBJECTS = []; // 储存所有可以“动”的元素的全局数组

class AcGameObject {
    constructor() // 构造函数
    {
        AC_GAME_OBJECTS.push(this);  // 将这个对象加入到储存动元素的全局数组里

        this.has_call_start = false; // 记录这个对象是否已经调用了start函数
        this.timedelta = 0; // 当前距离上一帧的时间间隔，相等于时间微分，用来防止因为不同浏览器不同的帧数，物体移动若按帧算会不同，所以要用统一的标准，就要用时间来衡量
    }

    start() {
        // 只会在第一帧执行一次的过程
    }

    update() {
        // 每一帧都会执行的过程
    }

    destroy() {
        this.on_destroy();
        // 删除这个元素
        for (let i = 0; i < AC_GAME_OBJECTS.length; ++i) {
            if (AC_GAME_OBJECTS[i] === this) {
                AC_GAME_OBJECTS.splice(i, 1); // 从数组中删除元素的函数splice()
                break;
            }
        }
    }

    on_destroy() {
        // 被删除之前的过程，“临终遗言”
    }
}

let last_timestp; // 上一帧的时间
let AC_GAME_ANIMATION = function (timestp) // timestp 是传入的一个参数，就是当前调用的时间
{
    for (let i = 0; i < AC_GAME_OBJECTS.length; ++i) // 所有动的元素都进行更新。
    {
        let obj = AC_GAME_OBJECTS[i];
        if (!obj.has_called_start) {
            obj.start(); // 调用start()
            obj.has_called_start = true; // 表示已经调用过start()了
        }
        else {
            obj.timedelta = timestp - last_timestp; // 时间微分
            obj.update(); // 不断调用
        }
    }
    last_timestp = timestp; // 进入下一帧时当前时间戳就是这一帧的时间戳

    requestAnimationFrame(AC_GAME_ANIMATION); // 不断递归调用
}

requestAnimationFrame(AC_GAME_ANIMATION); // JS的API，可以调用1帧里面的函数。(有些浏览器的一秒帧数不一定相等)
```

这里思路上也不难理解，只是`api`中调用了`requestAnimationFrame`这个，用来刷新界面，并且为了确保每个浏览器的效果相同，我们采用了时间作为其继承类中的距离的统一计算单位。

## 2、渲染地图

地图的渲染使用了`html5`中新增的`canvas`，`<canvas>` 会创建一个固定大小的画布，会公开一个或多个**渲染上下文**(画笔)，使用**渲染上下文**来绘制和处理要展示的内容。

```
class GameMap extends AcGameObject {
    constructor(playground) {
        super(); // 调用基类的构造函数

        this.playground = playground; // 这个Map是属于这个playground的
        this.$canvas = $(`<canvas></canvas>`); // canvas是画布
        this.ctx = this.$canvas[0].getContext('2d'); // 用ctx操作画布canvas

        this.ctx.canvas.width = this.playground.width; // 设置画布的宽度
        this.ctx.canvas.height = this.playground.height; // 设置画布的高度

        this.playground.$playground.append(this.$canvas); // 将这个画布加入到这个playground
    }

    render() {
        this.ctx.fillStyle = "rgba(0, 0, 0, 0.2)"; // 填充颜色设置为透明的黑色，透明度是为了出现拖尾效果
        this.ctx.fillRect(0, 0, this.ctx.canvas.width, this.ctx.canvas.height); // 画上给定的坐标的矩形
    }

    start() {

    }

    update() {
        this.render(); // 这个地图要一直画一直画（动画的基本原理）
    }

}
```

## 3、渲染玩家及技能特效

在玩家渲染的部分需要考虑更多的细节。

首先，需要考虑玩家大小，为了统一等比例显示大小，选用页面高度作为基准，后续的伤害，技能效果也以此作为标准；还有时间的确定，获得的时间单位是毫秒，注意单位转换。

其次，要考虑到其技能效果，基础版本先实现火球攻击特效与被攻击后的粒子四散特效。

最后，我们把玩家被攻击的判断放到玩家的类中实现，并且添加被击中后会失控一段时间。

```
class Player extends AcGameObject {
    constructor(playground, x, y, radius, color, speed, is_me) {
        super();
        this.playground = playground;
        this.ctx = this.playground.game_map.ctx;
        this.x = x;
        this.y = y;
        this.vx = 0;
        this.vy = 0;
        this.damage_x = 0;
        this.damage_y = 0;
        this.damage_speed = 0;
        this.move_length = 0;
        this.radius = radius;
        this.color = color;
        this.speed = speed;
        this.is_me = is_me;
        this.eps = 0.1;
        this.friction = 0.9;
        this.spent_time = 0;

        this.cur_skill = null;
    }

    start() {
        if (this.is_me) {  // 如果是玩家自己则需要增加监听函数
            this.add_listening_events();
        } else {  // 否则为电脑控制玩家，随机移动即可
            let tx = Math.random() * this.playground.width;
            let ty = Math.random() * this.playground.height;
            this.move_to(tx, ty);
        }
    }

    add_listening_events() {
        let outer = this;
        this.playground.game_map.$canvas.on("contextmenu", function () {  // 禁止页面右键出现菜单
            return false;
        });
        this.playground.game_map.$canvas.mousedown(function (e) {
            if (e.which === 3) {  // 3是右键
                outer.move_to(e.clientX, e.clientY);
            } else if (e.which === 1) {  // 1是左键
                if (outer.cur_skill === "fireball") {
                    outer.shoot_fireball(e.clientX, e.clientY);
                }

                outer.cur_skill = null;
            }
        });

        $(window).keydown(function (e) {
            if (e.which === 81) {  // 81为q键
                outer.cur_skill = "fireball";
                return false;
            }
        });
    }

    shoot_fireball(tx, ty) {  // 发射火球操作
        let x = this.x, y = this.y;
        let radius = this.playground.height * 0.01;
        let angle = Math.atan2(ty - this.y, tx - this.x);
        let vx = Math.cos(angle), vy = Math.sin(angle);
        let color = "orange";
        let speed = this.playground.height * 0.5;
        let move_length = this.playground.height * 1;
        new FireBall(this.playground, this, x, y, radius, vx, vy, color, speed, move_length, this.playground.height * 0.01);
    }

    get_dist(x1, y1, x2, y2) {
        let dx = x1 - x2;
        let dy = y1 - y2;
        return Math.sqrt(dx * dx + dy * dy);
    }

    move_to(tx, ty) {  // 移动到指定位置
        this.move_length = this.get_dist(this.x, this.y, tx, ty);
        let angle = Math.atan2(ty - this.y, tx - this.x);
        this.vx = Math.cos(angle);
        this.vy = Math.sin(angle);
    }

    is_attacked(angle, damage) {  // 被攻击判断
        for (let i = 0; i < 20 + Math.random() * 10; i++) {
            let x = this.x, y = this.y;
            let radius = this.radius * Math.random() * 0.1;
            let angle = Math.PI * 2 * Math.random();  // 被击中后方向失控
            let vx = Math.cos(angle), vy = Math.sin(angle);
            let color = this.color;
            let speed = this.speed * 10;  // 速度增加
            let move_length = this.radius * Math.random() * 5;
            new Particle(this.playground, x, y, radius, vx, vy, color, speed, move_length);  // 攻击后的粒子效果
        }
        this.radius -= damage;
        if (this.radius < 10) {
            this.destroy();
            return false;
        }
        this.damage_x = Math.cos(angle);
        this.damage_y = Math.sin(angle);
        this.damage_speed = damage * 100;
        this.speed *= 0.8;  // 被攻击后减速
    }

    update() {
        this.spent_time += this.timedelta / 1000;
        if (!this.is_me && this.spent_time > 4 && Math.random() < 1 / 300.0) {
            let player = this.playground.players[Math.floor(Math.random() * this.playground.players.length)];
            let tx = player.x + player.speed * this.vx * this.timedelta / 1000 * 0.3;
            let ty = player.y + player.speed * this.vy * this.timedelta / 1000 * 0.3;
            this.shoot_fireball(tx, ty);
        }

        if (this.damage_speed > 10) {
            this.vx = this.vy = 0;
            this.move_length = 0;
            this.x += this.damage_x * this.damage_speed * this.timedelta / 1000;
            this.y += this.damage_y * this.damage_speed * this.timedelta / 1000;
            this.damage_speed *= this.friction;
        } else {
            if (this.move_length < this.eps) {
                this.move_length = 0;
                this.vx = this.vy = 0;
                if (!this.is_me) {
                    let tx = Math.random() * this.playground.width;
                    let ty = Math.random() * this.playground.height;
                    this.move_to(tx, ty);
                }
            } else {
                let moved = Math.min(this.move_length, this.speed * this.timedelta / 1000);
                this.x += this.vx * moved;
                this.y += this.vy * moved;
                this.move_length -= moved;
            }
        }
        this.render();
    }

    render() {  // 在画布上渲染玩家，canvas的api函数
        this.ctx.beginPath();
        this.ctx.arc(this.x, this.y, this.radius, 0, Math.PI * 2, false);
        this.ctx.fillStyle = this.color;
        this.ctx.fill();
    }

    on_destroy() {
        for (let i = 0; i < this.playground.players.length; i++) {
            if (this.playground.players[i] === this) {
                this.playground.players.splice(i, 1);
            }
        }
    }
}
```

火球与粒子特效的实现，需要注意给其加上最长可以移动的距离效果会更符合游戏效果。

火球类的实现：

```
class FireBall extends AcGameObject {
    constructor(playground, player, x, y, radius, vx, vy, color, speed, move_length, damage) {
        super();
        this.playground = playground;
        this.player = player;
        this.ctx = this.playground.game_map.ctx;
        this.x = x;
        this.y = y;
        this.vx = vx;
        this.vy = vy;
        this.radius = radius;
        this.color = color;
        this.speed = speed;
        this.move_length = move_length;
        this.damage = damage;
        this.eps = 0.1;
    }

    start() {
    }

    update() {
        if (this.move_length < this.eps) {
            this.destroy();
            return false;
        }

        let moved = Math.min(this.move_length, this.speed * this.timedelta / 1000);
        this.x += this.vx * moved;
        this.y += this.vy * moved;
        this.move_length -= moved;

        for (let i = 0; i < this.playground.players.length; i++) {
            let player = this.playground.players[i];
            if (this.player !== player && this.is_collision(player)) {
                this.attack(player);
            }
        }

        this.render();
    }

    get_dist(x1, y1, x2, y2) {
        let dx = x1 - x2;
        let dy = y1 - y2;
        return Math.sqrt(dx * dx + dy * dy);
    }

    is_collision(player) {  // 判断是否相撞
        let distance = this.get_dist(this.x, this.y, player.x, player.y);
        if (distance < this.radius + player.radius)
            return true;
        return false;
    }

    attack(player) {
        let angle = Math.atan2(player.y - this.y, player.x - this.x);
        player.is_attacked(angle, this.damage);
        this.destroy();
    }

    render() {
        this.ctx.beginPath();
        this.ctx.arc(this.x, this.y, this.radius, 0, Math.PI * 2, false);
        this.ctx.fillStyle = this.color;
        this.ctx.fill();
    }
}
```

粒子特效实现：

```
class Particle extends AcGameObject {
    constructor(playground, x, y, radius, vx, vy, color, speed, move_length) {
        super();
        this.playground = playground;
        this.ctx = this.playground.game_map.ctx;
        this.x = x;
        this.y = y;
        this.radius = radius;
        this.vx = vx;
        this.vy = vy;
        this.color = color;
        this.speed = speed;
        this.move_length = move_length;
        this.friction = 0.9;
        this.eps = 1;
    }

    start() {
    }

    update() {
        if (this.move_length < this.eps || this.speed < this.eps) {
            this.destroy();
            return false;
        }

        let moved = Math.min(this.move_length, this.speed * this.timedelta / 1000);
        this.x += this.vx * moved;
        this.y += this.vy * moved;
        this.speed *= this.friction;
        this.move_length -= moved;
        this.render();
    }

    render() {
        this.ctx.beginPath();
        this.ctx.arc(this.x, this.y, this.radius, 0, Math.PI * 2, false);
        this.ctx.fillStyle = this.color;
        this.ctx.fill();
    }
}
```

## 4、terser 代码压缩工具

terser 不仅支持文件输入，也支持标准输入。结果会输出到标准输出中。

```
# 全局安装terser命令行工具
sudo apt-get update
sudo apt-get install npm
sudo npm install terser -g
# 使用terser
terser ./foo.js -c pure_funcs=[console.log],toplevel=true -m -o bar.js
# -c即compress表示普通的压缩代码
# pure_funcs表示删除代码中的console.log方法
# toplevel为true表示只在顶级作用域压缩清理变量
# -m即mangle会压缩变量名等等
# -o代表输出路径
```

