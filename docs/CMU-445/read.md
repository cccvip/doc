# 准备工作
-  开发环境：windows,vscode,c++
-  工作背景：6年开发经验
-  擅长语言：java,golang,python(优先级排序)
-  学习原因：好奇驱动

## 安装docker
解决WSL 2的问题
[WSL2](https://learn.microsoft.com/zh-cn/windows/wsl/install-manual#step-4---download-the-linux-kernel-update-package)


## 第一节课
andy Pavlo有点东西,第一节课讲了些计算的细节,以及数据库的重要性。真不错,挺有意思的 

## 第二节课
讲了一些SQL的高级用法,以及使用,最终留下了一份家庭作业。SQL的高级使用, 还是比较简单的SQL题目,第一次接触SQL的,有兴趣可以搞一搞

## 第三节课
[参考PDF](https://15445.courses.cs.cmu.edu/fall2022/notes/03-storage1.pdf)

### 我们有很多不同的层面有页这个概念：

- 硬件存储页（Hardware Page）：通常是 4KB。也就是如果你写的数据在 4KB 以内，那么就能保证原子性，不会发生写了一部分掉电导致只写入了一部分成功的情况，只会要么全成功或者全写入失败
- 操作系统页（OS Page）：通常也是 4KB
- 数据库页（Database Page）：通常是 512~16KB，例如 MySQL InnoDB 页大小默认 16KB，目前 8.0 可以配置成 4KB 8KB 16KB 32KB 64KB。在一些比较高端的系统，你甚至可以对于不同类型的页设置不同的页大小，例如元组数据页大小比较小，索引页大小调整的比较大。

这一节课讲到了大部分的数据库的原理部分,例如 page header。页的选择相对论等等。

推荐阅读：https://cloud.tencent.com/developer/article/2006282


