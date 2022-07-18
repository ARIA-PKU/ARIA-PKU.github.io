---
title: git的常用操作
date: 2022-05-02T15:12:22+08:00
lastmod: 2022-05-02T15:12:22+08:00

cover: http://oss.surfaroundtheworld.top/blog-pictures/cup_ice.jpg
# images:
#   - /img/cover.jpg
categories:
  - Linux常用知识
tags:
  - Linux
  - git
# nolastmod: true
draft: false
---

整理了git的命令常用的操作。封面来自pivix的[チャイ](https://www.pixiv.net/users/1096811)画师，内容整理自y总课程。

<!--more-->

## 1.1. git基本概念

工作区：仓库的目录。工作区是独立于各个分支的。

暂存区：数据暂时存放的区域，类似于工作区写入版本库前的缓存区。暂存区是独立于各个分支的。

版本库：存放所有已经提交到本地仓库的代码版本

版本结构：树结构，树中每个节点代表一个代码版本。

## 1.2 git常用命令

1. git config --global user.name xxx：设置全局用户名，信息记录在~/.gitconfig文件中
2. git config --global user.email xxx@xxx.com：设置全局邮箱地址，信息记录在~/.gitconfig文件中
3. git init：将当前目录配置成git仓库，信息记录在隐藏的.git文件夹中
4. git add XX：将XX文件添加到暂存区
5. git add .：将所有待加入暂存区的文件加入暂存区
6. git rm --cached XX：将文件从仓库索引目录中删掉
7. git commit -m "给自己看的备注信息"：将暂存区的内容提交到当前分支
8. git status：查看仓库状态
9. git diff XX：查看XX文件相对于暂存区修改了哪些内容
10. git log：查看当前分支的所有版本
11. git reflog：查看HEAD指针的移动历史（包括被回滚的版本）
12. git reset --hard HEAD^ 或 git reset --hard HEAD~：将代码库回滚到上一个版本
13. git reset --hard HEAD^^：往上回滚两次，以此类推
14. git reset --hard HEAD~100：往上回滚100个版本
15. git reset --hard 版本号：回滚到某一特定版本
16. git checkout — XX或git restore XX：将XX文件尚未加入暂存区的修改全部撤销
17. git remote add origin git@github.com:ARIA-PKU/webapp.git：将本地仓库关联到远程仓库
18. git push -u (第一次需要-u以后不需要)：将当前分支推送到远程仓库
19. git push origin branch_name：将本地的某个分支推送到远程仓库
20. git clone git@github.com:ARIA-PKU/webapp.git：将远程仓库XXX下载到当前目录下
21. git checkout -b branch_name：创建并切换到branch_name这个分支
22. git branch：查看所有分支和当前所处分支
23. git checkout branch_name：切换到branch_name这个分支
24. git merge branch_name：将分支branch_name合并到当前分支上
25. git branch -d branch_name：删除本地仓库的branch_name分支
26. git branch branch_name：创建新分支
27. git push --set-upstream origin branch_name：设置本地的branch_name分支对应远程仓库的branch_name分支
28. git push -d origin branch_name：删除远程仓库的branch_name分支
29. git pull：将远程仓库的当前分支与本地仓库的当前分支合并
30. git pull origin branch_name：将远程仓库的branch_name分支与本地仓库的当前分支合并
31. git branch --set-upstream-to=origin/branch_name1 branch_name2：将远程的branch_name1分支与本地的branch_name2分支对应
32. git checkout -t origin/branch_name 将远程的branch_name分支拉取到本地
33. git stash：将工作区和暂存区中尚未提交的修改存入栈中
34. git stash apply：将栈顶存储的修改恢复到当前分支，但不删除栈顶元素
35. git stash drop：删除栈顶存储的修改
36. git stash pop：将栈顶存储的修改恢复到当前分支，同时删除栈顶元素
37. git stash list：查看栈中所有元素
38. git rebase 多人协同同一个分支的时候会使用，会合并之前的commit历史。优点在于得到更简洁的项目历史，去掉了merge commit，但是需要手动处理版本冲突。
39. git restore --stage : 把不需要上传的文件pass掉，比如python中的.pyc文件，cpp中的.o文件以及linux中的.swp文件这种。

## 1.3 git其他知识

1. `.gitignore` 文件，可以设置不需要上传到git上的内容

2. git在上传的时候，把不需要上传的文件生成文件或者临时文件pass掉，比如python中的.pyc文件，cpp中的.o文件以及linux中的.swp文件这种，这样使得仓库更整洁更专业。

3. 连接git基本操作，最好先把本地公钥上传到git上，免得输密码麻烦。

   ```
   git init
   git add README.md
   git commit -m "first commit"
   git branch -M main
   git remote add origin git@github.com:xxx/xxx.git
   git push -u origin main
   ```