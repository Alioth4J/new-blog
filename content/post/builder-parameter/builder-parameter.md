---
title: 建造者模式构建方法参数
description: 将方法与方法参数解耦
date: 2024-05-25
slug: builder-parameter
image: 
categories:
    - Java
    - Design patterns
---

## MinIO中使用建造者模式构建方法的参数
例如：  
```
PutObjectArgs putObjectArgs = PutObjectArgs.builder()
                    .bucket(BUCKET_NAME)
                    .object(objectName)
                    .contentType(file.getContentType())
                    .stream(file.getInputStream(), file.getSize(), ObjectWriteArgs.MIN_MULTIPART_SIZE).build();
minioClient.putObject(putObjectArgs);
```
### 将多个参数传入`()`中的问题
- 可读性较差
一行代码过长，分多行也不够明确  
参数对应的是什么，需要记住或者ide提示  
- 可能导致错误重载
见《Effective Java》第52条
- 违反开闭原则
当需要新增、删除、修改参数时，需要修改方法签名
## 使用建造者模式构建方法参数的优点
### 提供了一种规范
方法入参唯一，是`methodNameArgs`
### 便于参数构建
- 链式调用清晰明了（编码、阅读、维护）
- 各个参数的设置可选择，提供了灵活性
### 符合开闭原则
需要新增、删除、修改参数时，不需要修改方法签名，只需要修改参数对应的类
## 好处来源于解耦
方法与方法参数之间解耦