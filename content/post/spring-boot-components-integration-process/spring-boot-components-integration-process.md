---
title: Spring Boot 组件集成流程
description: 组件的通用集成过程
date: 2024-08-16
slug: spring-boot-components-integration-process
image: 
categories:
    - Java
    - Spring
---

## “通用”的来源——模板方法模式
模板方法模式（Template Method Pattern）定义了算法的骨架，并将一些步骤的具体实现延迟到子类中，以允许子类在不改变算法结构的情况下重新定义算法的某些步骤。  
在 Spring Boot 中集成组件有一个通用过程，其中每一个步骤由用户自己实现，这就是模板方法模式的应用。
## "通用"的骨架—— IoC 容器
- IoC 容器提供了模板方法模式中“算法的骨架”
- 控制反转是 Spring Boot 集成其它组件的核心
## 通用集成过程
### 引入依赖
`pom.xml`（Maven）或 `build.gradle`（Gradle）文件中添加所需组件的依赖
### 环境配置
四种方式：
- application.properties
- application.yml
- 环境变量
- 启动选项（命令行参数）
### 自定义配置类
`@Configuration`
### 自定义组件
// 此“组件”指：所引入依赖中的一个部件  
`@Component` + 继承原有抽象组件，实现自定义组件  
### 注入组件，调用 API
集成完成，可以开始使用了
## 总结
IoC 容器既提供对组件集成的支持，又为通用的集成流程制定了骨架。  
对不同组件的集成能够有一个通用的集成过程，得益于模板方法模式。  