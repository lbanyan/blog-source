---
title: Git 使用总结
tags:
    - git
---

### 参考书籍
[Pro Git 中文版](http://iissnan.com/progit "Pro Git 中文版")

### Git 思想
1. Git 只关心文件数据的整体是否发生变化，而不关心文件各版本差异变化的细节。Git 使用指纹、快照、索引的方式来进行版本控制。
2. Git 维持了一个本地版本库，这样绝大多数的操作都可以本地完成，而无需网络。速度上有了很大提升。
3. Git 使用 SHA-1 算法计算文件数据的校验和，这便是文件的指纹。
4. 常用的 Git 操作大多仅仅是把数据添加到数据库。例如对文件的删除操作，实际只是添加了一条删除记录到数据库，并没有删除该文件。

<!--more-->

### 项目初始化到 Git
``` bash
git init
git add *
git commit -m 'initial project version'
```
commit message：主题 + 修改原因

### 从现有仓库克隆
``` bash
git clone git://github.com/schacon/grit.git
```
clone相较于checkout的不同点是，clone后客户端会完全获得仓库中项目的所有历史变迁数据。这样避免了服务端的单点故障导致历史变迁数据丢失，也加速了客户端后期查看历史数据的速度。

### 文件状态变化
![文件状态变化](http://iissnan.com/progit/book_src/figures/18333fig0201-tn.png)
文件分为已跟踪文件（纳入版本控制的文件）和未跟踪文件（未纳入版本控制中的文件）
文件的状态：未修改、已修改、暂存。
工作目录：本地（客户端）开发环境；仓库：服务端 Git 的仓库；暂存区：本地存放暂存文件的区域。

### 检查当前文件状态
``` bash
git status
```
提示内容很详尽，包括了所属分支、未跟踪文件、已跟踪文件的各种状态等。

``` bash
git add *
```
该命令有两种作用：一是将未跟踪文件添加未跟踪文件，二是将已修改的跟踪文件提交未暂存文件。

### 忽略某些文件
``` bash
cat .gitignore
```
具体的规则参看[忽略某些文件](http://iissnan.com/progit/html/zh/ch2_2.html "忽略某些文件")。

### 查看已暂存和未暂存的更新
尚未暂存和已暂存的文件对比：
``` bash
git diff
```
已暂存和已提交的文件对比：
``` bash
git diff --cached
```

### 提交更新
``` bash
git commit
```

### 跳过使用暂存区域
``` bash
git commit -a
```

### 移除文件
工作空间rm掉该文件后，使用以下命令提交：
``` bash
git rm *
```
文件已修改且提交到暂存区后再删除，应加**-f**，以防误删除文件后丢失修改的内容。
保留工作目录文件，仅删除仓库文件：
``` bash
git rm --cached *
```
批量操作请参考[移除文件](http://iissnan.com/progit/html/zh/ch2_2.html "移除文件")。

### 移动文件
``` bash
git mv file_from file_to
```

### 查看提交历史
``` bash
git log
```
具体参考[查看提交历史](http://iissnan.com/progit/html/zh/ch2_3.html "查看提交历史")。

### 撤销最后一次提交
``` bash
git commit --amend
```

### 撤销已经暂存
``` bash
git reset HEAD *
```

### 取消对文件的修改
``` bash
git checkout -- *
```
会直接覆盖掉之前修改过的文件，导致之前的修改丢失。

### 查看当前的远程库
``` bash
git remote -v
```
-v：--verbose

### 添加远程仓库
TODO

### 从远程仓库抓取数据
TODO

### 推送数据到远程仓库
TODO

### 查看远程仓库信息
TODO

### 远程仓库的删除和重命名
``` bash
git remote rename pb paul
git remote rm paul
```

### 远程仓库版本回退方法
可参考 [远程仓库版本回退方法](http://blog.csdn.net/fuchaosz/article/details/52170105)

### 列显已有的标签
``` bash
git tag
git tag -l 'v1.*'
```

### 新建标签
``` bash
git tag -a v1.4 -m 'my version 1.4'
git show v1.4
```

### 签署标签
``` bash
git tag -s v1.5 -m 'my signed 1.5 tag'
```

### 验证标签
``` bash
git tag -v v1.4
```

### 后期加注标签
展示之前的提交
``` bash
git log --pretty=oneline
```

### 分享标签
只有通过显式命令才能分享标签到远端仓库
```
git push origin v1.5
git push origin --tags // push所有标签
```

### 标签高级使用
可参考 [通过Tag标签回退版本修复bug](http://blog.csdn.net/fuchaosz/article/details/51698896)，我认为这种方式略为复杂，如果需要控制发布的版本，建议使用以release开头为分支名的分支来简单控制。

### 自动补全和 Git 命令别名
可参考[技巧和窍门](http://iissnan.com/progit/html/zh/ch2_7.html "技巧和窍门")。

### 新建分支与切换
``` bash
git checkout -b iss53
等价于：
git branch iss53
git checkout iss53
```
切换分支到master
``` bash
git checkout master
```

### 合并分支
``` bash
git merge hotfix
```

### 删除分支
``` bash
git branch -d iss53
```

### 合并冲突
合并冲突，人工修改后，使用：
``` bash
git add *
```
暂存，再提交。

### 查看分支
``` bash
git branch
git branch -v
git branch --merged // 查看哪些分支已被并入当前分支
git branch --no-merged
git branch -d testing // 分支有修改尚未合并，便会出错
git branch -D testing // 强制删除分支
```

### 远程分支
TODO
``` bash
git checkout -b test // 创建本地test分支
git push origin test:test // 根据本地test分支创建一个新的远程分支test
git push origin :test // 删除test远程分支
git branch -D test // 删除本地test分支
```

### 衍合（rebase）
把在一个分支里提交的改变移到另一个分支里重放一遍。
衍合能产生一个更为整洁的提交历史。如果视察一个衍合过的分支的历史记录，看起来会更清楚：仿佛所有修改都是在一根线上先后进行的，尽管实际上它们原本是同时并行发生的。
``` bash
git checkout experiment
git rebase master
```
多分支的衍合，请参考[有趣的衍合](http://iissnan.com/progit/html/zh/ch3_6.html "有趣的衍合")。
**一旦分支中的提交对象发布到公共仓库，就千万不要对该分支进行衍合操作。**对于这点的解释详见[衍合的风险](http://iissnan.com/progit/html/zh/ch3_6.html "衍合的风险")。TODO
TODO 衍合之后并没有发现相关代码也更新到master了

### 采纳来自邮件的补丁
``` bash
git apply --check /tmp/patch-ruby-client.patch // 检测
git apply /tmp/patch-ruby-client.patch
```
apply原子操作，不会像patch那样可以打成功部分补丁。

TODO
Git 查看历史版本功能是怎么做到的

