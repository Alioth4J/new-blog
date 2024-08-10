---
title: Tess4J 基础演示
description: 一个 Java OCR 引擎的使用
date: 2024-05-10
slug: tess4j-demo
image: 
categories:
    - Java
    - OCR
---

## 简介
Tesseract 是一个开源 OCR 引擎，能将图像中的文本转换为可编辑的文本数据。  
Tess4J 是基于 Java 的 Tesseract 的封装库，使 Java 开发者能够方便地利用 Tesseract 进行文本识别。
## 环境配置
### Linux
不需要环境配置，大多数 Linux 发行版中都包含 Tesseract。

### Windows  
- 下载 Tesseract 安装包：[https://digi.bib.uni-mannheim.de/tesseract/](https://digi.bib.uni-mannheim.de/tesseract/)
- 傻瓜式安装
- 配置环境变量（在系统变量中配置）  
变量名：`TESSDATA_PREFIX`  
变量值：`downloadpath\Tesseract-OCR\tessdata`
## 下载训练数据
- 下载训练数据（中文）：[https://github.com/tesseract-ocr/tessdata_fast/blob/main/chi_sim.traineddata](https://github.com/tesseract-ocr/tessdata_fast/blob/main/chi_sim.traineddata)  
在这里也有其他版本：[https://github.com/tesseract-ocr/tessdoc/blob/main/Data-Files.md](https://github.com/tesseract-ocr/tessdoc/blob/main/Data-Files.md)
## 项目构建
- `pom.xml` 文件中引入 tess4j 以及日志依赖
- `log4j.properties` 作为 log4j 的配置文件
- 将训练数据文件复制到 `src/main/resources/tess4j` 文件夹中
- `src/main/resources` 中放入需要进行 OCR 操作的图片
- 新建 `Tess4JDemo` 类，编写关键代码
```
package com.alioth4j;

import net.sourceforge.tess4j.Tesseract;
import net.sourceforge.tess4j.TesseractException;

import java.io.File;

/**
 * Tess4j演示
 * @author Alioth4J
 */
public class Tess4JDemo {

    public static void main(String[] args) {
        // 创建 File 对象表示要识别的图像文件
        File imgFile = new File("src/main/resources/example.png");
        // 创建 Tesseract 对象
        Tesseract tess = new Tesseract();
        // 设置训练数据所在位置
        tess.setDatapath("src/main/resources/tess4j");
        // 设置语言：英文 -> eng; 简体中文 -> chi_sim
        tess.setLanguage("chi_sim");
        // 字符串对象用于接收结果
        String imgText = null;
        // 执行 OCR
        try {
            imgText = tess.doOCR(imgFile);
        } catch (TesseractException e) {
            e.printStackTrace();
        }
        // 输出结果
        System.out.println("OCR 结果：");
        System.out.println(imgText);
    }

}
```
## 运行项目
- 运行 `Tess4JDemo#main` 方法即可