# Mybatis-Plus（四）批量插入功能（中篇）saveBatch 实现与分析

## 前言
承接上篇，继续聊聊批量插入功能。

其实这个功能本身不算太难，所以本文会对比一下框架 V3.5.0 以及 V4.0.1 两个版本之间的差异。

## 参考目录
- [MP官方文档 - Service CRUD 接口](https://baomidou.com/pages/49cc81/#service-crud-%E6%8E%A5%E5%8F%A3)

## 批量插入功能的代码实现
参照官方文档：<br>
> ![这里是引用](img04/a85ea7232bd84fc2a486b7c2f06c284e.png)

以框架中 `TestDemo` 为例。
### V3.X 版本使用

1. 实现类中继承 `ServicePlusImpl`（`IServicePlus` 实现类）<br>
   ![在这里插入图片描述](img04/a8efd42b9fca46c180704f45f497b4d0.png)<br>
   这一步一般是框架中使用代码生成功能时就会做好，如果是自己添加的实现类也是继承相应类即可。
2. 调用方法 `service.saveBatch`
   ![在这里插入图片描述](img04/3763cb829024418784b614c94c0d597d.png)
### V4.X 版本使用
1. Mapper 继承 `BaseMapperPlus`
   ![在这里插入图片描述](img04/531790e96ad44cf4890e4005a40f682a.png)<br>
   这一步同样是框架中使用代码生成功能时就会做好。
2. 调用方法 `mapper.insertBatch`
   ![在这里插入图片描述](img04/2635d219af294c349b358d8c06c93346.png)

### Debug & 测试结果（V4.X）
`BaseMapperPlus#insertBatch`<br>
![在这里插入图片描述](img04/fb7e756c06df477089db668f7224f886.png)

`SqlHelper#executeBatch`<br>
![在这里插入图片描述](img04/a79242bae22045eeb208999b6ebdfd76.png)

可以看到控制台的输出是单条的，注意这里是一个 sqlSession 完成所有插入，并不是 1000 个sqlSession 来执行。

数据库表数据：<br>
![在这里插入图片描述](img04/4d4657501c8c469fabe2edacf1ab1b85.png)

至此功能测试完成。

因为底层执行流程一样，所以 V3.X 版本就不再赘述了。

## 批量插入功能的调用流程分析
### ##、流程简图（重点）
因为是不同版本对比进行说明，所以画了简图方便说明。<br>
![在这里插入图片描述](img04/39fbf5a345b84fa9b529cbf80f88f9cc.png)
### ##、版本差异及其原因
`Service` 差异：<br>
![在这里插入图片描述](img04/227c9065dc29497d98cd11234e850494.png)

![在这里插入图片描述](img04/9bc7f3d191894718b7513352c2ea5388.png)

差异原因：<br>

从上面简图可以看出来，底层的调用流程并不复杂，并且底层方法都是一样的，在群里 [狮子大佬](https://blog.csdn.net/weixin_40461281) 也说过：

> ![在这里插入图片描述](img04/2bd8dfd294b34602818aa4a2b66865a2.png)

至于为什么不用 Service 的而要重新写到 Mapper 中，在框架更新日志中可以找到答案：

> ![这里是引用](img04/26597748fec44145bc06112f9003167b.png)


### V3.#2、`ServicePlusImpl#saveBatch`
![在这里插入图片描述](img04/e56302dc1c634132b55bbe4abb015577.png)
### V3.#3、`ServiceImpl#executeBatch`
![在这里插入图片描述](img04/8bfe3df69b97499791264ca1f4c29940.png)

### V4.#2、`BaseMapperPlus#insertBatch`
![在这里插入图片描述](img04/21926b5f4163477390aa7a0d07af8c93.png)

在下面的 `insertBatch` 方法其实和 `ServiceImpl#executeBatch`（V3.#3）调用的方法就是同一个了。
### #4、`SqlHelper#executeBatch`
![在这里插入图片描述](img04/c08dd6f861c9426395d5c178bc27ad0e.png)
### #5、`SqlHelper#executeBatch`
![在这里插入图片描述](img04/00dbcbc3aace462fbe6c0b4d8d6db0ef.png)

所有的流程到这里就结束了。

## 需要注意的点（下篇主要内容）

1. 要使用批处理功能，需要在配置文件中（`spring.datasource.dynamic.datasource.master.url`）添加以下参数：`rewriteBatchedStatements=true`
2. 相比起 SQL 注入器，两种插入方式哪个速度更优？

以上两点，我会放到下篇去进行说明。