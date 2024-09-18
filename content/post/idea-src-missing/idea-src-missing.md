---
title: IDEA 左侧项目目录消失问题 解决方案
description: src 快回来
date: 2024-09-18
slug: idea-src-missing
image: 
categories:
    - Java
---

## 问题描述
`git clone` 一个项目后，使用 IDEA 打开，发现左侧项目目录中没有源代码所在目录，例如 src 或者模块缺失
## 一些可能有用的解决方案
- File -> Invalidate Caches
- 删除 .idea 文件
## 正片开始
**Project Structure -> modules -> + -> Import Module -> 选择项目根目录 -> Create module from existing sources**  
此时模块被导入，左侧目录中可以看到出现了各个模块  

但是可能还没完，对于**结构较为复杂**（模块嵌套）的项目，上述做法会导致项目被拆散，即各个模块全都散出来了  

于是，上述步骤中选择根目录后，需要**先**选择 Import modole from external model，并选择项目对应的构建工具（例如 Maven），开始导入后**取消**。此时会发现右下角仍在下载 Maven 相关的包。**再**选择 Create module from existing sources 导入模块，会发现项目目录恢复正常了  
## 结语
为什么会出现这样的问题呢？可能是 IDEA 的一个 bug  
这个解决方案也是我摸索出来的，希望能帮到他人，**不被业务以外的事情困扰**  