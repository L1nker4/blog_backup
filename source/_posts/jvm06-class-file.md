---
title: JVM之类文件结构（六）
date: 2020-03-26 17:30:40
categories:
- Java
- JVM
tags:
- Java
- JVM
---

## 前言
实现语言无关性的基础是虚拟机和字节码存储格式。Java虚拟机不与任何程序语言绑定，它只与Class文件这种特定的二进制文件格式所关联。Class文件中包含了Java虚拟机指令集，符号表以及若干其他辅助信息。

## Class文件结构

任何一个Class文件对应着唯一的一个类或者接口的定义信息，Class文件是一组以八个字节为基础单位的**二进制流**，各个数据项目严格按照顺序紧凑的排列，中间没有任何分隔符，Class文件只有两种类型：**无符号数**、**表**。
- 无符号数属于基本数据类型，以u1,u2,u4,u8分别代表1/2/4/8个字节的无符号数，无符号数用来描述数字，索引引用，数量值。
- 表：由多个无符号数或者其他表作为数据项构成的复合数据类型。

### 魔数
Class文件的头4个字节被称为魔数，它的唯一作用就是确定这个文件是否为一个能被JVM接受的Class文件，魔数的十六进制表示为`0xCAFEBABE`。

### 版本信息

紧接着魔数的4个字节的是版本信息，第5-6字节是次版本号，7-8字节是主版本号，表示Class文件使用的JDK版本。
高版本的JDK能向下兼容，但是不能向上兼容。

### 常量池

版本信息之后是常量池入口，常量池主要存放两大类常量：**字面量**、**符号引用**。

常量池中常量的数量是不固定的，所以在入口需要放置一项u2类型的数据，代表常量池容量计数值。

字面量接近于常量概念，如字符串、被声明为final的值。
符号引用属于编译原理方面的概念：主要包括：

- 被模块导出或开放的包
- 类和接口的全限定名
- 字段的名称和描述符
- 方法的名称和描述符
- 方法句柄和方法类型
- 动态调用点和动态产量

常量池的每个常量都是一个表，共有17钟不同类型的常量。表开始的第一位是个u1类型的标志位，代表当前常量属于哪种常量类型。

17种常量类型：

| 类型                             | tag  | 描述                   |
| -------------------------------- | ---- | ---------------------- |
| CONSTANT_utf8_info               | 1    | UTF-8编码的字符串      |
| CONSTANT_Integer_info            | 3    | 整型字面量             |
| CONSTANT_Float_info              | 4    | 浮点型字面量           |
| CONSTANT_Long_info               | 5    | 长整型字面量           |
| CONSTANT_Double_info             | 6    | 双精度浮点型字面量     |
| CONSTANT_Class_info              | 7    | 类或接口的符号引用     |
| CONSTANT_String_info             | 8    | 字符串类型字面量       |
| CONSTANT_Fieldref_info           | 9    | 字段的符号引用         |
| CONSTANT_Methodref_info          | 10   | 类中方法的符号引用     |
| CONSTANT_InterfaceMethodref_info | 11   | 接口中方法的符号引用   |
| CONSTANT_NameAndType_info        | 12   | 字段或方法的符号引用   |
| CONSTANT_MethodHandle_info       | 15   | 表示方法句柄           |
| CONSTANT_MethodType_info         | 16   | 标识方法类型           |
| CONSTANT_InvokeDynamic_info      | 18   | 表示一个动态方法调用点 |

<br>
对于 CONSTANT_Class_info（此类型的常量代表一个类或者接口的符号引用），它的二维表结构如下：

| 类型 | 名称        | 数量 |
| ---- | ----------- | ---- |
| u1   | tag         | 1    |
| u2   | name\_index | 1    |

tag 是标志位，用于区分常量类型；name_index 是一个索引值，它指向常量池中一个 CONSTANT_Utf8_info 类型常量，此常量代表这个类（或接口）的全限定名，这里 name_index 值若为 0x0002，也即是指向了常量池中的第二项常量。

<br>
CONSTANT_Utf8_info 型常量的结构如下：

| 类型 | 名称   | 数量   |
| ---- | ------ | ------ |
| u1   | tag    | 1      |
| u2   | length | 1      |
| u1   | bytes  | length |

tag 是当前常量的类型；length表示这个字符串的长度；bytes 是这个字符串的内容（采用缩略的 UTF8 编码）


### 访问标志
常量池结束后，紧接着的两个字节表示访问标志，用来识别一些类或者接口层次的访问信息，包括：这个Class是类还是接口，是否定义为public类型，是否定义为abstract类型，是否被final修饰。


### 类索引，父类索引，接口索引集合
类索引和父类索引都是一个u2类型的数据，而接口索引集合是一组u2类型的数据集合，Class文件由这三项数据来确定该类的继承关系，类索引用来确定这个类的父类的全限定名。，父类索引只有一个，除了`java.lang.Object`之外，所有的Java类都有父类。接口索引集合用来描述这个类实现了哪些接口，这些接口按`implements`关键字后面的顺序从左到右排列在接口索引集合内。接口索引集合的第一项为u2类型的数据，表示索引表的容量，接下来就是接口的名字索引。

类索引，父类索引引用两个u2类型的索引值标识，它们各自指向一个类型为 CONSTANT_Class_info 的类描述符常量，通过该常量总的索引值可以找到定义在 CONSTANT_Utf8_info 类型的常量中的全限定名字符串。

### 字段表集合
字段表用来描述接口或者类种声明的变量，包括实例变量和类变量，但是不包括方法中的局部变量。
每个字段表只表示一个变量，本类中所有的成员变量构成字段表集合，字段表结构如下所示：



| 类型 | 名称              | 数量              | 说明                                                         |
| ---- | ----------------- | ----------------- | ------------------------------------------------------------ |
| u2   | access\_flags     | 1                 | 字段的访问标志，与类稍有不同                                 |
| u2   | name\_index       | 1                 | 字段名字的索引                                               |
| u2   | descriptor\_index | 1                 | 描述符，用于描述字段的数据类型。 基本数据类型用大写字母表示； 对象类型用“L 对象类型的全限定名”表示。 |
| u2   | attributes\_count | 1                 | 属性表集合的长度                                             |
| u2   | attributes        | attributes\_count | 属性表集合，用于存放属性的额外信息，如属性的值。             |



### 方法表集合

对方法的描述和对字段的描述采取了几乎完全一致的方式。依次包括访问标志，名称索引，描述符索引，属性表集合。

因为volatile关键字和transient关键字不能修饰方法，所以方法表的访问标志中没有这两项。

### 属性表集合
字段表，方法表都可以携带自己的属性表集合，以描述某些场景专有的信息。

属性表结构如下：



| 类型 | 名称                   | 数量              |
| ---- | ---------------------- | ----------------- |
| u2   | attribute\_name\_index | 1                 |
| u4   | attribute\_length      | 1                 |
| u1   | info                   | attribute\_length |