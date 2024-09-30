---
title: 访问权限控制情形：外来的对象回到了自己的类中
description: 同一类中具有直接访问私有字段的权限
date: 2024-09-30
slug: access-private-field
image: 
categories:
    - Java
---

> *笑问客从何处来*
## 代码
```java
import java.util.Objects;

public class ExampleClass {

    private String privateField;

    public ExampleClass(String privateField) {
        this.privateField = privateField;
    }

    @Override
    public boolean equals(Object other) {
        return ((this == other) 
                || (other instanceof ExampleClass that 
                    && Objects.equals(this.privateField, that.privateField)));
    }

}
```
## 这段代码能通过编译吗？  
`equals` 方法中，使用 `that.privateField` 访问私有字段  

乍一看，`Object other` 通过方法参数传来，是外部对象，不能使用 `.` 直接访问私有字段，而应该使用 `getter` 方法  

实际上，此时 `that` 的类型就是 `ExampleClass`，**在同一类中具有直接访问私有字段的权限**，所以代码是正确的，可以通过编译  
## 致谢
感谢 [**@Sam Brannen**](https://github.com/sbrannen) 和 [**Spring 社区**](https://github.com/spring-projects)  