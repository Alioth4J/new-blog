---
title: 主备复制与故障容错系统设计
description: VMware - Fault Tolerance
date: 2024-07-01
slug: vm-ft
image: 
categories:
    - Distributed systems
    - 6.824
---

> *檐上踏月客，屏息听完莫入迷*
## 介绍
VMware 的论文《The Design of a Practical System for Fault-Tolerant Virtual Machines》中提出了一种以主备复制进行故障容错的系统设计，这个容错机制用于在出现单点故障等情况下保障系统的可用性
## 基础架构
```
               │                                                             
               │    ┌──────────────────┐                 ┌──────────────────┐
┌──────────┐   │    │  ┌────────────┐  │ logging channel │  ┌───────────┐   │
│          │   │    │  │ Primary VM ┼──┼─────────────────┼──► Backup VM │   │
│  Client  ┼───┼────►  └────────────┘  │                 │  └───────────┘   │
│          │   │    │                  │                 │                  │
└──────────┘   │    │ Physical server  │                 │ Physical server  │
               │    │                  │                 │                  │
               │    └────────┬─────────┘                 └─────────┬────────┘
               │             │                                     │         
               │             │                                     │         
               │             │                                     │         
               │             │                                     │         
               │             │                                     │         
               │             │          ┌─────────────────┐        │         
               │             │          │                 │        │         
               │             └──────────┤   Shared disk   ┼────────┘         
                                        │                 │                  
                                        └─────────────────┘                  
                                                                             
```
- 两台虚拟机，primary 和 backup，分别运行在**两台物理机器**上
- primary 和 backup 共享磁盘
- client 只与 primary 进行交互，目标是让 client 认为（就像是）只有一台机器，对容错毫不知情
## 状态机与确定性重放
- 如果初始状态相同，在确定性系统中以相同顺序执行相同的指令，那么结束状态也相同  
- primary 与 backup 初始状态相同，backup 执行与 primary 相同的指令，这能够保证 primary 与 backup 的一致性。
## 主备之间的通信
- primary 和 backup 分别有一块 log buffer
- primary 将指令写入自己的 log buffer，通过 logging channel 发送给 backup，backup 将指令从网络中接收到自己的 log buffer 中后回复 `ACK`
## 输出机制 
- primary 执行指令并产生输出； backup 只是执行指令，不产生输出
- 只有当 primary 接收到 backup 回复的 `ACK` 后，才向 client 输出
> 注意：性能问题  
> - 优化1：面对读请求，primary 是否无需等待？  
> - 优化2：应用层的处理机制  
## 故障转移
- 当 primary 出现故障，由 backup 接管 primary 的工作，而 client 毫不知情
## 防止脑裂
- 共享磁盘支持一个原子的 `test-and-set` 操作，在 backup 接管之前进行这个操作，如果两个服务器都存活，这个操作就会失败，避免了脑裂
## 更多
- 磁盘发生故障时系统将无法继续提供服务，一种可行的解决方案是备份磁盘
- 复制状态机方法对于多 CPU 主机有局限性，VM-FT 经过多年的发展，从复制状态机方法转向状态转移方法，以提供对多 CPU 的支持