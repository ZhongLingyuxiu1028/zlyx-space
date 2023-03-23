# Easy Excel（一）Excel 2003（*.xls）导入流程分析

## 前言
因为分析 Validator 校验器而给自己挖坑说要分析 Easy Excel 导入流程，今天终于要来填坑了。

因为内容比较多，所以分成了 `Excel 2003（*.xls）` 以及 `Excel 2007（*.xlsx）` 两种类型分析，并且分开两篇来讲，这篇先从 xls 下手。

## 参考目录
- [Easy Excel 官方文档](https://easyexcel.opensource.alibaba.com/docs/current/)

## 框架集成
### 1、Maven
总工程 `pom.xml`， Easy Excel 版本 `V3.1.1`，poi 版本 `V5.2.2`<br>
![在这里插入图片描述](img01/0b4a1e7d4f8e4f2e9b9d5f59a60b9a7b.png)

框架这里自行引入了 poi，可以参考 [官网描述](https://easyexcel.opensource.alibaba.com/qa/#%E5%85%B3%E4%BA%8E%E7%89%88%E6%9C%AC%E9%80%89%E6%8B%A9) 修改：<br>
![在这里插入图片描述](img01/e0c11c297f5e4e1d9dd6b74e44295f49.png)
### 2、框架集成公共部分
### 2.1、Excel 操作工具类 `ExcelUtil`
框架封装了工具类 `ExcelUtil`，可以直接进行调用。<br>
![在这里插入图片描述](img01/d1d51492a2fe40f895a8c88b775e228b.png)
### 2.2、导入监听接口 `ExcelListener`
![在这里插入图片描述](img01/7d2b87f088f04c5a8a1d38238abfd1ed.png)
### 2.3、默认监听器 `DefaultExcelListener`
![在这里插入图片描述](img01/0c209580c76a44688bc7feb495b37bf4.png)
### 2.3.1、数据处理方法 `DefaultExcelListener#invoke`
![在这里插入图片描述](img01/ee4cb6736f834719a47072ffcc8be409.png)

此处只校验 `ImportGroup` 分组。

### 2.3.2、异常处理方法 `DefaultExcelListener#onException`
![在这里插入图片描述](img01/9c628498bb1345e7af4fb094238aa56c.png)
### 2.4、Excel 结果接口 `ExcelResult`
![在这里插入图片描述](img01/de9a959ba37b4df0ac0ee70b0aac7e80.png)
### 2.5、默认 Excel 结果对象 `DefautExcelResult`
![在这里插入图片描述](img01/2e4c14cca4cc495b9a29cd8ec8f51d9f.png)
### 2.5.1、导入结果 `DefautExcelResult#getAnalysis`
![在这里插入图片描述](img01/7eff51a91e9f4039912f9c6e351fcda2.png)
### 3、导入测试方法
### 3.1、导入测试接口 `TestDemoController#importData`
![在这里插入图片描述](img01/a31a28307a564a639dced0ff113b8d5e.png)
### 3.2、导入对象 `TestDemoImportVo`
![在这里插入图片描述](img01/bd4435ff4ab847fba32fe1c1f407a0fd.png)

导入数据：<br>
![在这里插入图片描述](img01/d23e0df629bc4383949fbc35dee7da58.png)

先说明一下调用分析流程：

1. 调用总共分两次，用到的数据是上图（**同样的文件，同样的内容**）。
2. 第一次是正常的调用分析，即不进行数据校验（会把相关字段注解注释），目的是先理顺请求流程。
3. 第二次是有异常的调用分析，即校验有标注组信息 `groups = {ImportGroup.class}` 的属性字段，其他默认分组不进行校验。

### 4、接口测试
### 4.1、导入成功
![在这里插入图片描述](img01/2551c3a29a5940a5b6bbcea96daf0409.png)
### 4.2、导入存在异常
![在这里插入图片描述](img01/72d8161f4c72417a9e5cea534ce75b34.png)

## 执行流程分析
### 1、流程简图（非常重要）
为了便于理解和对比，所以我把两种类型文件的导入流程画在了一张图里面，**很多流程是相同的，不同的地方用不同的颜色进行了区分**，但是为了简洁和突出重点，并不是每一层都详细列了出来（中间省略了一些不太重要的深入调用，在下面 Debug 分析里面会把截图放出来）。<br>
![在这里插入图片描述](img01/5245770abdd7427e8c9b5e474c793cd5.png)

**温馨提醒，由于流程步骤较多，结合这张图走不容易迷路。**

之前每次画图都可能需要 Debug 好几次才能把所有流程记录下来，不过这次想分享一个小技巧，但 **注意不是所有方法都适用**。

在分析异常报错时，抛出的异常如下图：<br>
![在这里插入图片描述](img01/1dbccd91b4c84221aec022d42e418019.png)

由下往上就是所有的调用流程。

好了，废话不多说，下面进入正题。
### 2、（#1）`TestDemoController#importData`
![在这里插入图片描述](img01/ec3c8121827646a1afdeb774282fba4a.png)
### 3、（#2）`ExcelUtil#importExcel`
![在这里插入图片描述](img01/6690dc689707418f94a35ff8c19418f8.png)
### 4、（#4）阅读器生成器 `EasyExcelFactory#read`
![在这里插入图片描述](img01/f565ed6ce7cd4af4b2c7c9c9e4ceba88.png)
### 5、（#5）工作表 `ExcelReaderBuilder#sheet`
![在这里插入图片描述](img01/edc3d2ecf712408bbb59d54a25e363a8.png)
### 6、（#6）`ExcelReaderBuilder#build`
![在这里插入图片描述](img01/eb88200cc0044a4ca38a11f991e85498.png)
### 7、（#7）`ExcelReader#ExcelReader`
![在这里插入图片描述](img01/f4f2c5324392437f9fa551a75a530161.png)

`ExcelAnalyserImpl#ExcelAnalyserImpl`<br>
![在这里插入图片描述](img01/db1a4cdce8984e27a79b6ef1e0798c94.png)
### 8、（#8）根据类型选择执行器 `ExcelAnalyserImpl#choiceExcelExecutor`
![在这里插入图片描述](img01/7c9c989347864932b90ae6b271edd101.png)
### 8.1、获取 Excel 文件类型
`ExcelTypeEnum#valueOf`<br>
![在这里插入图片描述](img01/cb557dc669fb476885c39ef617bf3b8f.png)

![在这里插入图片描述](img01/cbeddb13aa2b48528f2f2753b6e80806.png)

`ExcelTypeEnum#recognitionExcelType`<br>
![在这里插入图片描述](img01/fe92d677d2c247a58e0e7f7579cf668c.png)

得到文件类型为 `xls`，回到主方法 `ExcelAnalyserImpl#choiceExcelExecutor` 继续执行。<br>
![在这里插入图片描述](img01/141dc755ad8340faba2484d7ada2e4ae.png)

工作表 `ExcelReaderSheetBuilder` 构建完成：<br>
![在这里插入图片描述](img01/0b196b884b074d4f9d9171e061f642ba.png)
### 9、（#9）读操作 `ExcelReaderSheetBuilder#doRead`
![在这里插入图片描述](img01/233c6f87c1dc4d88968ff3743d77e3b3.png)
### 10、（#10）读取工作表 `ExcelReader#read`
![在这里插入图片描述](img01/efec7d2fcfd54a41992e6a5ac51433bf.png)

### 11、（#11）解析 `ExcelAnalyserImpl#analysis`
![在这里插入图片描述](img01/f08d7f0893894cf594fb5f23227d3541.png)

在前面步骤 `#9` 已经通过文件类型确定了是 xls 解析器。
### 12、（03#1）xls 解析 `XlsSaxAnalyser#execute`
![在这里插入图片描述](img01/970a2ea6b4494f3eaf802b8302c78e27.png)

### 13、（03#2）处理工作表事件 `HSSFEventFactory#processWorkbookEvents`
![在这里插入图片描述](img01/37f0f36c3a664bee978672fb0c3f15f2.png)

![在这里插入图片描述](img01/3759a43a7a46480d8c53c6bd581e5cd2.png)

`HSSFEventFactory#processEvents`<br>
![在这里插入图片描述](img01/53c77f21aae54c76933584fb3ea46afe.png)

### 14、循环解析
`HSSFEventFactory#genericProcessEvents`<br>
![在这里插入图片描述](img01/e9486b52a0224418b51434c52f1e099b.png)

**注：此方法是循环工作表中的数据，因为循环很多，所以我只截取其中两次（表头、行数据）来进行说明。**

`HSSFRequest#processRecord`<br>
![在这里插入图片描述](img01/8d987f3b95794733840478b73703f490.png)

`FormatTrackingHSSFListener#processRecord`<br>
![在这里插入图片描述](img01/aee7f9bdadd54e68ab7d969168e0d836.png)

`MissingRecordAwareHSSFListener#processRecord`<br>
![在这里插入图片描述](img01/9be44ef9550546b3ad6329f1ac534a2a.png)

![在这里插入图片描述](img01/1852fbf16d8b4120885c3386e52a916d.png)

![在这里插入图片描述](img01/dd01b8e0ced543269447d72e2df6bc53.png)

### 15、（03#3）处理工作表数据记录 `XlsSaxAnalyser#processRecord`
![在这里插入图片描述](img01/131f35a3465945d19ecd93b4ad6928aa.png)

`BofRecordHandler#processRecord`<br>
![在这里插入图片描述](img01/1a4376c60a944fe0a49b083ad39e1d2f.png)

这里是第一次循环处理完毕返回。

当所有表头解析完成之后，会调用 `DummyRecordHandler#processRecord`。

### 16、（03#4）处理工作表数据记录 `DummyRecordHandler#processRecord`
![在这里插入图片描述](img01/da6cafc7aff44eb4899a466c3b498227.png)

### 17、（#12）结束行 `DefaultAnalysisEventProcessor#endRow`
![在这里插入图片描述](img01/d62c6fe7c599494b8300a4f0def97436.png)

### 18、（#13）数据处理 `DefaultAnalysisEventProcessor#dealData`
### 18.1、构造表头
![在这里插入图片描述](img01/a36d68e0eba14195a5032200a7a2ae7f.png)

`DefaultAnalysisEventProcessor#buildHead`<br>
![在这里插入图片描述](img01/12dba139eeac494896dbf1e284726609.png)

![在这里插入图片描述](img01/cb435a121d51478bae91ddf24b968c2b.png)

`ModelBuildEventListener#invokeHead`<br>
![在这里插入图片描述](img01/cad9ad539d014116953c803f967b23d3.png)

### 18.2、（#14）行数据处理 `DefaultExcelListener#invoke`
`DefaultAnalysisEventProcessor#dealData`<br>
![在这里插入图片描述](img01/06c88ef17b2c4b92aa9ce27bf357992b.png)

`DefaultExcelListener#invoke`<br>
![在这里插入图片描述](img01/a3a7e148c34847eba36c6bcf688610c2.png)

![在这里插入图片描述](img01/bb475499fdbe4cd68093602bcbb78b65.png)
### 19、（#14）校验异常 `ValidatorUtils#validate`
在前一个步骤中如果校验不通过，则会抛出异常 `ConstraintViolationException`<br>
![在这里插入图片描述](img01/78b53da5b27e4e5ba2e78195ec8278a6.png)
### 20、（#15）捕获异常 `DefaultAnalysisEventProcessor#dealData`
![在这里插入图片描述](img01/354e6befda604f4aa0e578e5dc222352.png)
### 21、（#16）自定义异常处理 `onException`
![在这里插入图片描述](img01/d1ea98f599c04fad904030c768fdfb7f.png)

`DefaultAnalysisEventProcessor#onException`<br>
![在这里插入图片描述](img01/64f62d5a20a14bc6b70924988f46b282.png)

`DefaultExcelListener#onException`<br>
![在这里插入图片描述](img01/c55b573c3f23430bb7e611d30ea74168.png)

![在这里插入图片描述](img01/8974edefd71f4d588e020bef34c94508.png)
### 22、解析完成（循环结束）
`HSSFEventFactory#genericProcessEvents`<br>
![在这里插入图片描述](img01/eefc46a50f4344949da17fc427c9f580.png)

### 23、（#17）完成读取
`ExcelReaderSheetBuilder#doRead`<br>
![在这里插入图片描述](img01/ec960980e265473cab7011a70cbff83e.png)

`ExcelReader#finish`<br>
![在这里插入图片描述](img01/ad5d5f47e49e46baa92037ed8e7cddb5.png)

`ExcelAnalyserImpl#finish`<br>
![在这里插入图片描述](img01/de45c1a8b9144b3da1ddc82fe49a7a13.png)

![在这里插入图片描述](img01/dba67a69ed2c4a2c9645dd83b5452221.png)

### 24、（#18）返回结果
![在这里插入图片描述](img01/9e970d5e3de044fbb7c3180267a4318a.png)

![在这里插入图片描述](img01/4d5792a8e3534136b267139374b7cfed.png)
### 25、完成读取（有异常信息）
`ExcelUtil#importExcel`<br>
![在这里插入图片描述](img01/977609e9628d45dbb5d06083f73d7f96.png)

控制台只输出了校验通过的结果列表<br>
![在这里插入图片描述](img01/704319bdf11b47eb8f93d79c3683879b.png)

返回自定义信息（抛出异常）`DefautExcelResult#getAnalysis`<br>
![在这里插入图片描述](img01/9a8c48b488d143a28d9365acf5ac72b1.png)

控制台错误信息<br>
![在这里插入图片描述](img01/99d5d9e003a04296824ed2b728788dfc.png)

![在这里插入图片描述](img01/8228f5b6770242aea6d4c6188471a048.png)

至此所有流程解析完毕。