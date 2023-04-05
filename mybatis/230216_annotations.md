# Java 元注解以及 MyBatis @Param 注解分析
- - -
## 前言
本文内容对应的是书本第 7 章的内容，主要是关于Java 元注解以及 `@Param` 注解的分析。

## 参考目录
- [《通用源码阅读指导书：MyBatis源码详解》](https://weread.qq.com/web/bookDetail/de732ba071f94a8ede7dc94)<br>
本文主要内容来自 ` 第7章 annotations包与lang包`。
- [《通用源码阅读指导书——MyBatis源码详解》配套示例](https://github.com/yeecode/MyBatisDemo)<br>
书中涉及的 Demo 示例，本文示例在 `Demo1` 的基础上进行了简单改造。

与上篇一样，需要说明的是，书中使用的框架版本和本文（本专栏）使用的版本不一样。

|名称  | 书中版本 | 专栏版本 |
|--|--|--|
| MyBatis | 3.5.2 | 3.5.11+ |
| Spring Boot| 2.X | 3.X |
| JDK | 8 | 17+ |

随着版本的升级迭代，会有一些内容不尽相同，需要结合着进行学习。

## 学习笔记
### 1、Java 注解
### 1.1、Java 元注解

> ![在这里插入图片描述](images/230216_annotations/f3e30a96ca4045fc8bd488b615ad6816.png)

两处划红色虚线的位置是我做了想法标注，由于作者使用的 JDK 版本并不是最新的，因此和现在的有差异，下面来说明一下。

Java中一共有 **七个** 元注解，分别是`@Documented`、`@Target`、`@Retention`、`@Inherited`、`@Repeatable`、`@Native`、`@ContentType`。

- `@Native`：这个注解用于标记一个方法是本地方法（native method）。本地方法是由非 Java 代码实现的方法，通常是用 C 或 C++ 等语言编写的。使用本地方法可以实现与Java虚拟机之外的底层系统或资源的交互。在声明本地方法时需要使用该注解，同时还需要在本地方法中使用 JNI（Java Native Interface）来和非 Java 代码交互。
- `@ContentType`：用于指定注解所表示的内容类型，例如时间跨度或频率。

![在这里插入图片描述](images/230216_annotations/031de26677b9475788f1a6a54db76aba.png)

![在这里插入图片描述](images/230216_annotations/1a8b8fedc5054fed9684c1523da2c7ba.png)

### 1.2、Java ElementType 枚举值

> ![这里是引用](images/230216_annotations/02b57e7f26694c3dba05e34aad99a276.png)

在 JDK 17 中，又多了两种枚举值 `MODULE` 以及 `RECORD_COMPONENT`。

具体查看源码可知：<br>
![在这里插入图片描述](images/230216_annotations/7978c0382679458083a55fc9ff7f939e.png)

![在这里插入图片描述](images/230216_annotations/86a99b8ef3ab4cf6ab4b9af6dcb92128.png)

和 ChatGPT 唠了一下关于这两者：<br>
![在这里插入图片描述](images/230216_annotations/c6644eaa50f94e7d88527c4b278a5284.png)

### 1.3、自定义注解
关于自定义注解，书中有进行举例说明。

结合前面学习的内容，本文以前几篇文章中分析 `RuoYi-Vue-Plus` 框架中的自定义注解 `@Translation` 为例对元注解的使用进行简单说明：<br>
![在这里插入图片描述](images/230216_annotations/fb5525ffcd674a25ba372acacc966b1e.png)

### 2、`@Param` 注解分析
完成了对 Java 注解的基本了解之后，书中对 MyBatis 自定义注解 `@Param` 注解进行了分析，并结合代码分析了关于 Mapper 接口中定义的参数进行解析的过程。

### 2.1、`@Param` 注解
![在这里插入图片描述](images/230216_annotations/382a02722a684d219a625bfd6f64c3b8.png)
### 2.2、测试方法
参照书中的举例，结合 `Demo1` 进行了一些改造，其他不变，重点是观察 Mapper 接口的参数解析过程。

![在这里插入图片描述](images/230216_annotations/a9563c5dc1c5420aa4633763092a58d3.png)
### 2.3、流程分析（重点：`ParamNameResolver`）
Debug 过程如下：
![在这里插入图片描述](images/230216_annotations/b03b19a57f8a4bdfac4da14c6ef594b8.png)

![在这里插入图片描述](images/230216_annotations/3f96aa93711c4a1386144f3371f9b14e.png)

`MapperProxy#invoke`<br>
![在这里插入图片描述](images/230216_annotations/b7f8d31161dd4915bf799c8a983aca6c.png)

此方法是最终执行 SQL 查询的主要方法。参数的解析方法在第一步 `cachedInvoker(method)` 时完成。

`MapperProxy#cachedInvoker`<br>
![在这里插入图片描述](images/230216_annotations/2e61f9d0f4af4c6d9888b4cc404dc2b5.png)

`MapperMethod#MapperMethod`<br>
![在这里插入图片描述](images/230216_annotations/567f3878ba2840f596363d7bd2ef8e9e.png)

创建映射方法，创建 SQL 命令以及方法签名 `MethodSignature`。

`MethodSignature#MethodSignature`<br>
![在这里插入图片描述](images/230216_annotations/a0f9f84ae1db467ab4c7e8ffc434807b.png)

该方法的最后会创建一个参数名称解析器 `ParamNameResolver`，也是`@Param` 注解能够生效的原因所在。

`ParamNameResolver#ParamNameResolver`<br>
![在这里插入图片描述](images/230216_annotations/8487595df5ca4f34a231ab3e85cc3a86.png)

> ![这里是引用](images/230216_annotations/2d3bebb19e6a42b7a94049cd3c941b01.png)

由于测试方法中第二个参数没有标注注解，来看下它的参数名实际上是什么：<br>
![在这里插入图片描述](images/230216_annotations/f48c3799db1e4a0fabbc45f5933909cc.png)

![在这里插入图片描述](images/230216_annotations/2ee855fc460b454596a1c4e2f62e61cd.png)

最终完成三个参数参数名称的解析：<br>
![在这里插入图片描述](images/230216_annotations/f6b3994d1aef4b748ee7a8d3a4006547.png)

所有的名称会被存在 names 中：<br>
![在这里插入图片描述](images/230216_annotations/1140a2eba82c4a83b424e22a01f2be4c.png)

回到上一级完成了方法签名的创建：<br>
![在这里插入图片描述](images/230216_annotations/e8e09523789d41148df8bbc1f41edd72.png)

最终返回到 `invoke` 方法执行 SQL 语句。<br>
![在这里插入图片描述](images/230216_annotations/66b63652d5b3400492d7d9a0f97755c8.png)

本章节的重点是分析参数名称解析器 `ParamNameResolver` 的执行过程，对于其他方法会在后续的章节中再展开说明。

（完）