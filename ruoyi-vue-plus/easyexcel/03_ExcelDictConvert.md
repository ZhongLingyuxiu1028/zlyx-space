# Easy Excel（三）自定义转换器 ExcelDictConvert 源码分析
- - -
## 前言
在前两天测试新的 Easy Excel 自定义转换器 `ExcelEnumConvert` 时才发现之前没有分析过这一部分的功能，所以本篇来介绍一下框架中另外一个比较有代表性的自定义转换器 `ExcelDictConvert`。

自定义转换器在实际应用中也比较常见。举个栗子，某些字段值在数据库中使用诸如 `0`，`1`，`2` 这样的枚举值进行存储，而在 Excel 导入导出功能中，则需要对这些特定的字段进行转换，这时就需要使用自定义转换器进行转换，解析出所需要的值。

下面本文就以导入导出为例对这一功能进行说明，其他的自定义转换器也是类似的实现方式，感兴趣的朋友可以自行研究。

## 参考目录
- [Easy Excel 官方文档 - 自定义转换器](https://easyexcel.opensource.alibaba.com/docs/current/quickstart/read#%E8%87%AA%E5%AE%9A%E4%B9%89%E8%BD%AC%E6%8D%A2%E5%99%A8)

## 框架集成
### 1、Maven
- 框架分支：`5.X`
  （注：在框架 `4.X` （`dev`）分支也有该功能，只是文件目录不同）
-  Easy Excel 版本：`V3.2.0`
- poi 版本：`V5.2.3`

### 2、框架集成模块 `ruoyi-common-excel`
由于在框架 `5.X` 分支中对框架目录进行了重构，所以与 Excel 相关的功能被单独抽成了一个公共模块，方便使用。

![在这里插入图片描述](img03/4786f95dedd54766994ed5ccac472e3c.png)

关于本文自定义转换器功能所涉及的类分别是：
- 核心实现类 `ExcelDictConvert`
- 注解类 `ExcelDictFormat`
- 工具类 `ExcelUtil`

### 2.1、自定义转换器 `ExcelDictConvert`
![在这里插入图片描述](img03/052d9ebc578e4fdab2885882f8b02b46.png)

实现字典值转换功能的核心类，实现了 Easy Excel 的 `com.alibaba.excel.converters.Converter` 接口。

按照官方文档，要实现自定义转换器的功能很简单，只需要实现 `Converter` 接口并实现指定类即可。

![在这里插入图片描述](img03/4081f38d1c6540499e50d29cfe617b93.png)

`com.alibaba.excel.converters.Converter`
![](img03/3fd9aa31f1d04e09b19d06cc55c529a1.png)

由上图源码，右侧黄色背景的类都是 Easy Excel 已经实现的转换器，都是常见类型的转换器。

需要实现的方法如下：

`com.alibaba.excel.converters.Converter#convertToJavaData`<br>
![在这里插入图片描述](img03/f4e4acd4e62942f19d7278f2c32ce9f5.png)

`com.alibaba.excel.converters.Converter#convertToExcelData`<br>
![在这里插入图片描述](img03/d0fb6f842fcb4a9cb3ce4c42d506ac0f.png)

框架中的实现方法如下：

![在这里插入图片描述](img03/c2b27a977bba4ff3880e1b573e4f9eab.png)

### 2.2、自定义转换器注解 `ExcelDictFormat`
![在这里插入图片描述](img03/373e4ae5e0964ec8a0a0d55507c91660.png)

通过在对象字段上标注该注解实现转换功能，框架中定义了两种解析方式：`字典类型解析` 以及 `表达式解析`，后者可以自定义分隔符。

### 2.3、Excel 工具类 `ExcelUtil`
在之前的导入分析文章中也有涉及到这个类，这个类封装了常用的 Excel 处理方法，本文涉及到的方法有两个：`convertByExp` 以及 `reverseByExp`。这两个方法逻辑类似，理解了其中一个，另一个也能够理解。

`ExcelUtil#convertByExp`<br>
![在这里插入图片描述](img03/6a653f012cd94c93b926aa6c28f233a2.png)

`ExcelUtil#reverseByExp`<br>
![在这里插入图片描述](img03/3cf2e66e1f28452ab397d384f6deb077.png)
### 3、测试方法
### 3.1、用户导入
`SysUserController#importData`<br>
![在这里插入图片描述](img03/fa375645a92d46bdae0caae42ef20aa0.png)

`SysUserImportVo`<br>
![在这里插入图片描述](img03/2ef5eb5a3abf40479e067cc596927678.png)

### 3.2、用户导出
`SysUserController#export`<br>
![在这里插入图片描述](img03/051019c4a9ce4a5daa511a14e2c07f5a.png)

`SysUserExportVo`<br>
![在这里插入图片描述](img03/2cedb704ca6f4c519bc14be65564d4c6.png)

### 3.3、测试调用流程说明
本文的测试流程如下：

1. 由于转换器的实现方法分别对应 `读 Excel` （导入）和 `写 Excel`（导出），所以分两次进行调用。
2. 调用导出方法。
3. 使用导出方法得到的 Excel 表格结果进行修改后，调用导入方法。
4. 为了更好地说明不同的解析方法，所以对于相关转换字段的导入导出方法都进行了分析。
5. 导入导出方法的转换功能差异不大，所以只要理解了一个，另一个也能够理解。

## 执行流程分析
本文的分析重点集中在实现方法的核心类，对于中间的调用流程不会展开详细说明。

### 1、用户导出流程分析
![在这里插入图片描述](img03/38b94250db074028a3780b0870ae3868.png)

`SysUserController#export`<br>
![在这里插入图片描述](img03/6f0d6472522d4d9199fad4774448a220.png)

`ExcelUtil#exportExcel`<br>
![在这里插入图片描述](img03/33010494728c4f8ab25ff44f551f3f54.png)

`ExcelUtil#exportExcel`<br>
![在这里插入图片描述](img03/fac9b6f7d28742b48f7797b9bc4eb0fc.png)

`ExcelWriterSheetBuilder#doWrite`<br>
![在这里插入图片描述](img03/eaf2fa933ea448aba00d1149fa51bc6c.png)

### 1.1、方法调用链

从框架工具类到转换器经过的调用链如下：

![在这里插入图片描述](img03/a93acfdaaedc444990653e405990747c.png)

然后就是核心方法 `ExcelDictConvert#convertToExcelData`。

### 1.2、性别字段转换分析（使用字典解析）
`ExcelDictConvert#convertToExcelData`<br>
![在这里插入图片描述](img03/377c947c9e5b44d6bda7b9ee9d0dcf8d.png)

该方法的主要逻辑如下图：

![在这里插入图片描述](img03/49aab435916248e29d4812e814b7d7e3.png)

`SysDictTypeServiceImpl#getDictLabel`<br>
![在这里插入图片描述](img03/217020b3303342e89d3e731168909a66.png)

### 1.3、用户状态字段转换分析（使用表达式解析）
`ExcelDictConvert#convertToExcelData`<br>
![在这里插入图片描述](img03/eb69134929204846910fee45b5b5d733.png)

`ExcelUtil#convertByExp`<br>
![在这里插入图片描述](img03/47a5bdcea93c45ee86066f8eb3e15706.png)

解析的结果：<br>
![在这里插入图片描述](img03/37289979d40c44e88d36a7689380ef44.png)

### 1.4、导出结果
![在这里插入图片描述](img03/13ea7555e4ad4322befe959e58dea688.png)

### 2、用户导入流程分析
P.S. 对于导入流程，在之前的文章中分别以 Excel [2003 版本](01_import_2003.md) 以及 [2007 版本](02_import_2007.md) 为例进行了详细说明，感兴趣的朋友可以回头看看。

导入的数据：（根据前面导出的结果简单修改）

![在这里插入图片描述](img03/a442b6096d28465381be3b6a7816140e.png)

导入操作：<br>
![在这里插入图片描述](img03/5018ce3a2844498a8046f43a29a89c3c.png)

![在这里插入图片描述](img03/561ccd0bade2420fb6fd2adf87942e36.png)

`SysUserController#importData`<br>
![在这里插入图片描述](img03/d8fe908cfb8943caa1b6c0cfe72aa780.png)

`ExcelUtil#importExcel`<br>
![在这里插入图片描述](img03/62b9e78b59a847b8baea3a689065db65.png)

### 2.1、方法调用链
![在这里插入图片描述](img03/c8d65cb783c9422082416a29bbd48d1a.png)
### 2.2、性别字段转换分析（使用字典解析）
`ExcelDictConvert#convertToJavaData`<br>
![在这里插入图片描述](img03/493d7d603e25407da583dd544a399b55.png)

`SysDictTypeServiceImpl#getDictValue`<br>
![在这里插入图片描述](img03/506a47e1202649009ba2f9ba937cd5a8.png)

最终转换结果：

![在这里插入图片描述](img03/488747ba9ad94652b5d442d739e8fe5b.png)

### 2.3、用户状态字段转换分析（使用表达式解析）
`ExcelDictConvert#convertToJavaData`<br>
![在这里插入图片描述](img03/2b981837476e43c9b03f3a9f499cb3a3.png)

`ExcelUtil#reverseByExp`<br>
![在这里插入图片描述](img03/2c9ff4183d3c461592b4fc33e66e3953.png)

![在这里插入图片描述](img03/ce7d0219c9814b129e346135a7625610.png)

### 1.5、导入结果
![在这里插入图片描述](img03/e8de69e36f09484ba3fb7e0af9268eb9.png)

用户列表：

![在这里插入图片描述](img03/ef10acd7469549f884ac03c03c2079cf.png)

用户详情：

![在这里插入图片描述](img03/1829151bb3864883ad57c847699204ca.png)

以上是关于自定义转换器的流程分析。

（完）