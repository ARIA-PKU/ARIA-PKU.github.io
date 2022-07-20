---
title: MIT6.828
date: 2022-07-20T13:08:09+08:00
lastmod: 2022-07-20T13:08:09+08:00
cover: https://aria9766.oss-cn-beijing.aliyuncs.com/blog-pictures/7_17/%E7%BA%A2%E5%8F%B6.jpg
# images:
#   - /img/cover.jpg
categories:
  - OS
tags:
  - MIT6.828

# nolastmod: true
draft: false
---

MIT6.828学习笔记

<!--more-->

对照课程完成lab中，结合lab记录各个知识点。

github地址：https://github.com/ARIA-PKU/MIT6.828

目前进度：lab1

## 第一章 操作系统接口

### fork()

fork()用于通过系统调用创建子进程，其在父子进程中都有返回值。在父进程中返回子进程的pid；在子进程中返回0，因此，可以根据返回值判断父子进程。

```
int pid = fork();
if(pid > 0) {  // 父进程
    printf("parent: child=%d\n", pid);
    pid = wait((int *) 0);
    printf("child %d is done\n", pid);
} else if(pid == 0) {  // 子进程
    printf("child: exiting\n");
    exit(0);
} else {  // error
    printf("fork error\n");
}
```

exit系统调用用于进程停止执行并释放资源，其接收一个整数状态参数，通常0表示成功，1表示失败。

wait系统调用返回当前进程已退出（或已杀死）的子进程pid，并将子进程的退出状态传递给wait地址；如果子进程没有退出，则wait等待一个子进程退出。如果调用者没有子进程，wait立即返回-1。如果父进程不关心子进程退出状态，可以传递0地址给wait。

父子进程拥有相同的内存内容，但是二者运行在不同的内存空间中，因此，变量不会相互影响。

### read()和write()

read和write都是以字节为单位读取指定fd（文件描述符）中的内容的。

read(fd, buf, n)表示从fd中读取最多n个字节，将它们复制到buf中并返回读取的字节数。read每次从文件偏移量读取数据，即后续会读取之前读取之后的字节。没有字节可读时，read返回0表示文件结束。

write(fd, buf, n)将buf中的n个字节写入fd，并返回写入字节数，发生错误时才会写入小于n字节的数据。其余与read类似。

### dup()

dup复制现有的文件描述符，返回一个引用自同一个底层对象的新fd。两个fd共享一个偏移量。

```
fd = dup(1);
write(1, "hello ", 6);
write(fd, "world\n", 6);
```

### 管道

管道是一对文件描述符提供给进程的小型内核缓冲区，为进程提供了一种通信方式。

以一个例子说明：

```
int p[2];
char *argv[2];
argv[0] = "wc";
argv[1] = 0;
pipe(p);
if (fork() == 0) {
    close(0);
    dup(p[0]);
    close(p[0]);
    close(p[1]);
    exec("/bin/wc", argv);
} else {
    close(p[0]);
    write(p[1], "hello world\n", 12);
    close(p[1]);
}
```

pipe，创建管道，在数组p中记录读写的fd。子进程调用close和dup使得fd 0指向读取端。关闭p中所存的fd，调用exec运行wc。

close是释放一个文件描述符，使其未来可被open、pipe或dup调用。新分配的文件描述符总是当前进程中编号最小的未使用描述符。

**TIPS：0对应的是标准输入， 1对应标准输出， 2对应标准错误输出。**

父进程关闭管道读取端，写入管道，然后关闭写入端。

在没有数据进来的时候，管道的写端会进入等待，直到有数据写入或所有指向写入端的fd都被关闭。

