---
layout:     note
title:      "Java 开发规范"
subtitle:   " \"Java 开发规范\""
author:     "tablesheep"
catalog: true
hide-in-nav: true
header-style: text
tags:
- note
- Java
---



# 命名规范

> 见名知意，不推荐使用缩写，除非是一些业界公认的缩写（减少沟通成本）



## 类/接口命名规范

### 通用

类/接口一律使用大驼峰（**UpperCamelCase**）方式命名

**命名方式：**

- 名词 
- 多名词
- 形容词 + 名词

**eg：**

- 名词：`java.lang.String`
- 多名词：`java.lang.StringBuffer`
- 形容词 + 名词：`java.util.LinkedList`



### 抽象类/接口

抽象类/接口可以使用**形容词**的方式进行命名

**eg：**

- 抽象类：`java.lang.reflect.Executable`
- 接口：`java.io.Serializable`



抽象类更加明了的方式是以**`Abstract`**开头

**eg：**`java.lang.AbstractStringBuilder`



#### 子类/实现类

- `xxx` + 抽象类/接口名

  **eg：** `java.io.InputStream`与其实现类

  > 默认实现 / 模板实现：Default、Generic、Common、Basic ... 开头，eg：`org.springframework.context.support.GenericApplicationContext`

  

- 抽象类/接口名 + `Impl`

  **eg：** `org.springframework.beans.BeanWrapperImpl`

  

### 枚举

枚举统一以`Enum`后缀结尾

**eg:** `HandleStateEnum`



### 异常类

异常类命名使用 Exception 结尾

**eg:** `java.lang.RuntimeException`



### 测试类

测试类以它要测试的类的名称开始，以 Test 结尾





## 方法命名规范

方法名统一使用小驼峰（ **lowerCamelCase** ）风格

**eg:** `getFieldName()`、`selectNameById()`



## 变量命名规范

### 常量

常量命名全部用大写，单词间用下划线隔开

**eg:** `CACHE_MAX_CAPACITY`



### 参数名、成员变量、局部变量

统一使用小驼峰（ **lowerCamelCase** ）风格

- 布尔类型不要以`is`开头





# 框架使用注意点

## lombok

- 使用`@Builder`时要注意构造方法，否则序列化时可能会出现异常，推荐直接加上`@AllArgsConstructor`、`@NoArgsConstructor`

  

# References

- <https://developer.aliyun.com/article/769408>
