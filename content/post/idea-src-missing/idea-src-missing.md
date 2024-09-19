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
`git clone` 一个项目后，使用 IDEA 打开，发现左侧项目目录中没有源代码所在目录，例如 src 缺失或者模块缺失
## 具体步骤
首先：  
**Project Structure -> modules -> + -> Import Module -> 选择项目根目录 -> (接着看文章)**

对于结构简单的项目，选择 Create module from existing sources 即可  
此时模块被导入，左侧目录中可以看到出现了各个模块  

但是，对于**结构较为复杂**（模块嵌套）的项目，上述做法会导致项目被拆散，即各个模块全部散出去  
于是，在选择根目录后，需要选择 Import module from external model，并选择项目对应的构建工具（例如 Maven），进行导入  

还没有结束，因为 `pom.xml` 变灰且被划横线  
解决方案：在 Settings -> Build, Execution, Deployment -> Build Tools -> Maven -> Ignored Files 中取消选择所有 `pom.xml`  

点击 `Apply`，发现点不动...  
解决方案：点击 `OK`，通过 File -> Invalidate Caches 删除缓存，重启后项目恢复正常  
## 结语
为什么会出现这样的问题呢？可能是 IDEA 的 bug  
这个解决方案也是我摸索出来的，希望能帮到他人，**不被业务以外的事情困扰**  