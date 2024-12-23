---
title: Hello Silverblue!
description: Silverblue 的魅力所在
date: 2024-11-06
slug: silverblue
image: 
categories:
    - Linux
---

> *平衡 忠诚 不息*

## “银蓝”
首先名字就很好听，哈哈... 

## 隔离变化与不变
在操作系统的日常使用中，底层内核往往保持稳定，上层应用变化较多。  

一般的系统中，上层应用在变化中可能破坏底层操作系统，接连影响到其他应用。最坏的情况是，上层应用导致操作系统内核损坏，从而整个系统崩溃。显然，原因是紧耦合。  

因此，解决方案是对变化与不变的隔离。Silverblue 具有不可变特性：操作系统的基础部分是只读的，保持稳定；应用的变化仅发生在上层，无法改变底层系统的核心，由此减少了系统损坏或崩溃的风险。  

## 容器化应用
除了少数最必要的包使用 `rpm-ostree install` 进行安装，大部分应用都使用容器化技术安装，例如 Flatpak, Toolbx, Podman，容器化技术在不可变的 Silverblue 系统上创建了可变、可管理的开发环境，并保证了应用之间的进一步隔离。  

另外，Toolbx 使各种工具能够整齐分类摆放，使工作空间保持整洁。  

## 无缝衔接的体验
除了不可变部分，Silverblue 的使用体验和 Workstation 极为相似。  

在 Toolbx 中，可以使用 dnf 安装应用，这一点应该能打消很多转变的疑虑。  

## 不要停止前进
在传统的操作系统中，面对前方的未知，前进由勇气驱动。Silverblue 则为勇气提供了保障。  

rpm-ostree 的版本控制机制能够切换和回滚到不同的系统版本，使用户能够大胆尝试新的版本；容器化应用则让用户不必担心对主系统产生不良影响。        

## 最后
Silverblue 的设计充满魅力，不要犹豫去使用！  
