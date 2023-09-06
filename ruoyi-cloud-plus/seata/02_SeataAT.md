# （二）Seata AT 模式 Demo 调用流程分析

---


## 前言
之前的文章介绍了 XA 模式的调用流程。但是 XA 模式也存在着一些问题：

> （截图自微信读书《阿里云云原生架构实践》）<br>
> ![这里是引用](https://img-blog.csdnimg.cn/82610ac287f74b65a600564b386479d0.png)

而为了能够解决 XA 模式存在的问题， Seata 演化出了 AT 模式，本文来介绍一下 AT 模式的底层调用流程。

## 参考目录
- [Seata 官方文档](https://seata.io/zh-cn/docs/overview/what-is-seata.html)
- [Seata 官方 Demo ：seata-xa](https://github.com/seata/seata-samples/tree/master/seata-xa)
- [Seata AT 模式](https://seata.io/zh-cn/docs/dev/mode/at-mode.html)
- [《阿里云云原生架构实践》](https://weread.qq.com/web/bookDetail/731327d07248f2dd7319578)<br>
书本章节 3.5 介绍了分布式事务模式的相关内容。

## 版本说明
由于官方 Demo 版本较低，本文使用的版本如下：
- `Seata`：`V1.7.0`
- `druid-spring-boot-starter`：`V1.2.16`


## 测试 Demo

这一部分和 XA 模式的内容大致相同，所以大部分内容就直接 copy 之前文章的。
### 0、Demo XA / AT 模式切换
两种模式业务代码相同，只是修改数据源配置即可进行模式切换。
> （截图自 GitHub README.md）<br>
> ![在这里插入图片描述](https://img-blog.csdnimg.cn/b7a67d4d80ec42cfa8acf4a77d2de81b.png)

建表 undo_log：

```sql
CREATE TABLE IF NOT EXISTS undo_log
(
    branch_id     BIGINT(20)   NOT NULL COMMENT 'branch transaction id',
    xid           VARCHAR(100) NOT NULL COMMENT 'global transaction id',
    context       VARCHAR(128) NOT NULL COMMENT 'undo_log context,such as serialization',
    rollback_info LONGBLOB     NOT NULL COMMENT 'rollback info',
    log_status    INT(11)      NOT NULL COMMENT '0:normal status,1:defense status',
    log_created   DATETIME(6)  NOT NULL COMMENT 'create datetime',
    log_modified  DATETIME(6)  NOT NULL COMMENT 'modify datetime',
    UNIQUE KEY ux_undo_log (xid, branch_id)
) ENGINE = InnoDB COMMENT ='AT transaction mode undo table';
```

### 1、模块说明
![在这里插入图片描述](https://img-blog.csdnimg.cn/4bac125836d348f59d75f499e0adefe3.png)

Demo 一共四个模块，是经典的下单流程。**按照如下顺序启动**：
- 账户模块（Port：8083） 
- 订单模块（Port：8082） 
- 库存模块（Port：8081） 
- 业务模块（Port：8084） 

模块间使用 openfeign 进行调用。

### 2、调用逻辑说明
入口：业务模块
```url
http://127.0.0.1:8084/purchase
```
业务逻辑：
1. 调用业务模块接口（business-xa）
2. 扣减商品库存（stock-xa）
3. 新增账户订单（order-xa）
4. 账户余额扣减（account-xa）


> （截图自 GitHub README.md）<br>
> ![在这里插入图片描述](https://img-blog.csdnimg.cn/1f9b3f57014044aa9669d232b94af4be.png)
### 3、分析流程说明
整个下单操作是一个全局事务，各个模块分属于各个分支事务。下面对源码进行分析时只对其中一个分支操作进行说明，其余分支操作都是相同的，就不再展开说明。

分析分为 Commit 和 Rollback 两种流程。

[官方文档](https://seata.io/zh-cn/docs/dev/mode/xa-mode.html) 对于 AT 模式有详细的说明，本文在此基础上进行展开。

### 4、注意事项
由于 XA 模式中出现了重试锁表的问题，所以本文的分析是基于控制台日志输出以及 Debug 堆栈信息。下文没有对每一个方法进行 Debug 截图，而是按照调用流程贴出了相应的主要方法。

为了完整体现整个全局事务流程，也在不断点的情况下对流程进行了截取分析并制作了流程图，详细的见下文。
## Seata AT 模式 Commit 调用流程分析
### 1、调用流程图
![在这里插入图片描述](https://img-blog.csdnimg.cn/787664ae618f412f9e2dba473962e7f2.png)

步骤前面由数字标识，根据各模块控制台输出整理，详细见附录。

### 2、全局事务开启 Global Begin
`io.seata.tm.api.DefaultGlobalTransaction#begin`

![在这里插入图片描述](https://img-blog.csdnimg.cn/e310b05c320d490e95adbbb120492984.png)

`io.seata.tm.DefaultTransactionManager#begin`

![在这里插入图片描述](https://img-blog.csdnimg.cn/840ed8e6c00f4977822d86438f32144f.png)

这一步骤向 Server 端（TC）发送了全局事务开启请求。下面来看下 Server 端处理逻辑：

`io.seata.core.rpc.processor.server.ServerOnRequestProcessor#onRequestMessage`

![在这里插入图片描述](https://img-blog.csdnimg.cn/3ad65d5ac60d447d9096133c1c4b7e85.png)

控制台打印：

```bash
11:40:09.610  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : receive msg[single]: GlobalBeginRequest{transactionName='purchase(java.lang.String, java.lang.String, int, boolean)', timeout=60000}, clientIp: 192.168.2.117, vgroup: my_test_tx_group
```

`io.seata.server.coordinator.DefaultCoordinator#doGlobalBegin`

![在这里插入图片描述](https://img-blog.csdnimg.cn/b25951ddae054a1cb2580780cbfcee58.png)

控制台打印：

```bash
11:40:09.624  INFO --- [verHandlerThread_1_11_500] [coordinator.DefaultCoordinator] [       doGlobalBegin]  [192.168.2.117:8091:2252230319988862988] : Begin new global transaction applicationId: business-xa,transactionServiceGroup: my_test_tx_group, transactionName: purchase(java.lang.String, java.lang.String, int, boolean),timeout:60000,xid:192.168.2.117:8091:2252230319988862988
```

`io.seata.server.coordinator.DefaultCore#begin`

![在这里插入图片描述](https://img-blog.csdnimg.cn/9a4e90eba1634b29b9050c9bbd27e3f3.png)

`io.seata.server.session.GlobalSession#GlobalSession`

![在这里插入图片描述](https://img-blog.csdnimg.cn/d01f25a6290f48c3880e85e8f8d6d729.png)

`io.seata.core.rpc.processor.server.ServerOnResponseProcessor#process`

![在这里插入图片描述](https://img-blog.csdnimg.cn/55a799b4d0f740c4bc4b7ad1bfbf8093.png)

控制台打印：

```bash
11:40:09.625  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : result msg[single]: GlobalBeginResponse{xid='192.168.2.117:8091:2252230319988862988', extraData='null', resultCode=Success, msg='null'}, clientIp: 192.168.2.117, vgroup: my_test_tx_group
```

Server 端处理完毕后，回到 Client 端主方法，事务开启。

```bash
11:40:09.635  INFO 17724 --- [nio-8084-exec-1] io.seata.sample.service.BusinessService  : New Transaction Begins: 192.168.2.117:8091:2252230319988862988
```

### 3、分支注册
分支还是以库存业务分支为例进行说明。

因为调用的方法很多，所以截了一张 Debug 堆栈信息图方便比对。

![在这里插入图片描述](https://img-blog.csdnimg.cn/3032359e4dee4456b54b86a93c91b12e.png)

从上图可以看出，先执行了 SQL 相关的一些操作，然后进行 Commit 操作，在提交之前进行分支注册操作。来看下一些关键的方法：

`io.seata.rm.datasource.PreparedStatementProxy#executeUpdate`

![在这里插入图片描述](https://img-blog.csdnimg.cn/5b167942925243399c76d90d74218d54.png)

`io.seata.rm.datasource.exec.ExecuteTemplate#execute`

![在这里插入图片描述](https://img-blog.csdnimg.cn/f2c21ee5f2674d76a60c7ada82042b4a.png)

`io.seata.rm.datasource.exec.BaseTransactionalExecutor#execute`

![在这里插入图片描述](https://img-blog.csdnimg.cn/608cb3e5179d49e2b889929473cef859.png)

`io.seata.rm.datasource.exec.AbstractDMLBaseExecutor#doExecute`

![在这里插入图片描述](https://img-blog.csdnimg.cn/846de7683c8a41cb8e9edd5de049a964.png)

这里获取到前后镜像，并执行了SQL语句。然后进行 Commit 操作。

`io.seata.rm.datasource.ConnectionProxy#commit`

![在这里插入图片描述](https://img-blog.csdnimg.cn/2a68d639831542b7857eb21a0c190462.png)

直到方法 `io.seata.rm.datasource.ConnectionProxy#processGlobalTransactionCommit` 终于到了注册逻辑。

`io.seata.rm.datasource.ConnectionProxy#register`

![在这里插入图片描述](https://img-blog.csdnimg.cn/a8d89b572a634664b853811877744e44.png)

`io.seata.rm.AbstractResourceManager#branchRegister`

![在这里插入图片描述](https://img-blog.csdnimg.cn/42181b7d12ae4a0e910db566434cd84f.png)

这里还是通过 Netty 向 Server 端（TC）进行分支注册请求。来看下 Server 端的处理逻辑：

`io.seata.core.rpc.processor.server.ServerOnRequestProcessor#handleRequestsByMergedWarpMessage`

![在这里插入图片描述](https://img-blog.csdnimg.cn/147e74b778ed455c93151d9fc55b5731.png)

控制台打印：

```bash
11:40:09.882  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : receive msg[merged]: BranchRegisterRequest{xid='192.168.2.117:8091:2252230319988862988', branchType=AT, resourceId='jdbc:mysql://192.168.2.158:3326/seata', lockKey='order_tbl:15', applicationData='null'}, clientIp: 192.168.2.117, vgroup: my_test_tx_group
```

`io.seata.server.coordinator.DefaultCoordinator#onRequest`

![在这里插入图片描述](https://img-blog.csdnimg.cn/cd58391d27a841679cbc25d429818271.png)

`io.seata.server.AbstractTCInboundHandler#handle`

![在这里插入图片描述](https://img-blog.csdnimg.cn/a80bc46aed5d4081bad48380b338fc4e.png)

`io.seata.server.coordinator.DefaultCoordinator#doBranchRegister`

![在这里插入图片描述](https://img-blog.csdnimg.cn/17f5ebb21cb3430fab57373e8ba74d19.png)

Server 端分支注册方法：

`io.seata.server.coordinator.AbstractCore#branchRegister`

![在这里插入图片描述](https://img-blog.csdnimg.cn/b91b3d5b074842e4902d172711a8fabe.png)

该方法的主要逻辑：

 1. 使用 xid 获取到全局会话。
 2. 锁定会话并在事务上下文进行操作。
 3. 全局会话状态检查。
 4. 全局会话添加会话生命周期监听器。
 5. 新建全局事务分支，获得分支会话对象。
 6. 锁定分支会话。
 7. 将分支会话添加到全局会话中，如果出现异常，解锁分支并抛出异常。
 8. 打印日志并返回分支id。

`io.seata.server.session.SessionHelper#newBranchByGlobal`

![在这里插入图片描述](https://img-blog.csdnimg.cn/5b1ee66023504765ba786ad65671f888.png)

控制台打印：

```bash
11:40:09.962  INFO --- [nPool.commonPool-worker-3] [erver.coordinator.AbstractCore] [bda$branchRegister$0]  [192.168.2.117:8091:2252230319988862988] : Register branch successfully, xid = 192.168.2.117:8091:2252230319988862988, branchId = 2252230319988862992, resourceId = jdbc:mysql://192.168.2.158:3326/seata ,lockKeys = order_tbl:15
11:40:09.962  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : result msg[merged]: BranchRegisterResponse{branchId=2252230319988862992, resultCode=Success, msg='null'}, clientIp: 192.168.2.117, vgroup: my_test_tx_group
```

Server 端分支注册完成，回到 Client 端打印日志并返回分支 id 。

控制台打印：

```bash
11:40:09.963  INFO 17280 --- [nio-8082-exec-4] io.seata.rm.AbstractResourceManager      : branch register success, xid:192.168.2.117:8091:2252230319988862988, branchId:2252230319988862992, lockKeys:order_tbl:15
```

### 4、UndoLog
我认为 AT 模式与 XA 模式最大的不同，在于 SQL commit 的时机：
- XA 模式在一阶段只是执行 SQL 语句，但是不提交，等到业务调用链中所有 SQL 执行无异常后，在二阶段才进行提交。
- AT 模式则是在一阶段执行 SQL 执行就会进行提交，然后把 SQL 执行前后的变化通过 undo_log 表进行记录，如果执行无异常，二阶段提交就把 undo_log 表记录进行删除，如果有异常，则根据 undo_log 表记录进行数据回滚操作。

SQL 执行前后的数据，Seata 称之为前后镜像。

`io.seata.rm.datasource.exec.AbstractDMLBaseExecutor#executeAutoCommitFalse`

![在这里插入图片描述](https://img-blog.csdnimg.cn/625e07124d374af6b999c98398c24c45.png)

`io.seata.rm.datasource.exec.BaseTransactionalExecutor#prepareUndoLog`

![在这里插入图片描述](https://img-blog.csdnimg.cn/8ac9fc7a7e4440989a5f4895b5bd2d87.png)

在上一步分支注册完成后，UndoLog 记录就会存入数据库中。

`io.seata.rm.datasource.ConnectionProxy#processGlobalTransactionCommit`

![在这里插入图片描述](https://img-blog.csdnimg.cn/e954b41271f84323afa2848901a07ce1.png)

`io.seata.rm.datasource.undo.AbstractUndoLogManager#flushUndoLogs`

![在这里插入图片描述](https://img-blog.csdnimg.cn/2e77f2fdd36f441c80fadfce05ac002a.png)

![在这里插入图片描述](https://img-blog.csdnimg.cn/886a5af3296f4f3f86e207560a561b71.png)

详细的信息都存在 rollback_info 字段中。三个事务分支的 undo_log 信息：

Stock 库存：

![在这里插入图片描述](https://img-blog.csdnimg.cn/7d8c903180064e17b60e5cd11ea41467.png)

Order 订单：

![在这里插入图片描述](https://img-blog.csdnimg.cn/9c809907fef7470394860ea49a2ca382.png)

Account 账户：

![在这里插入图片描述](https://img-blog.csdnimg.cn/7406e7db4d244fe686f40817c1a92b98.png)

### 5、全局事务提交 Commit
`io.seata.tm.api.DefaultGlobalTransaction#commit`

![在这里插入图片描述](https://img-blog.csdnimg.cn/0eff2c5a00964f4f9e71d13ab8c07712.png)

`io.seata.tm.DefaultTransactionManager#commit`

![在这里插入图片描述](https://img-blog.csdnimg.cn/f5626cff948747dc87908413c2e5b5a4.png)

向 Server 端发起 Commit 请求。

控制台打印：

```bash
11:40:10.294  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : receive msg[single]: GlobalCommitRequest{xid='192.168.2.117:8091:2252230319988862988', extraData='null'}, clientIp: 192.168.2.117, vgroup: my_test_tx_group
11:40:10.511  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : result msg[single]: GlobalCommitResponse{globalStatus=Committed, resultCode=Success, msg='null'}, clientIp: 192.168.2.117, vgroup: my_test_tx_group
```

全局事务提交成功。

控制台打印：

```bash
11:40:10.515  INFO 17724 --- [nio-8084-exec-1] i.seata.tm.api.DefaultGlobalTransaction  : transaction end, xid = 192.168.2.117:8091:2252230319988862988
11:40:10.515  INFO 17724 --- [nio-8084-exec-1] i.seata.tm.api.DefaultGlobalTransaction  : [192.168.2.117:8091:2252230319988862988] commit status: Committed
```

### 6、分支提交处理
`io.seata.core.rpc.processor.client.RmBranchCommitProcessor#process`

![在这里插入图片描述](https://img-blog.csdnimg.cn/1bf29bfde5034001bf42dd139e922318.png)

`io.seata.rm.AbstractRMHandler#doBranchCommit`

![在这里插入图片描述](https://img-blog.csdnimg.cn/cc8e74600be041c085aa80c4c5bde784.png)

因为一阶段 SQL 已经 Commit，因此二阶段分支提交主要是删除 UndoLog。

`io.seata.rm.datasource.AsyncWorker#dealWithGroupedContexts`

![在这里插入图片描述](https://img-blog.csdnimg.cn/be2fbe6d7d1745af958dc871871ee8de.png)

`io.seata.server.coordinator.DefaultCore#doGlobalCommit`

![在这里插入图片描述](https://img-blog.csdnimg.cn/86108007719b445ba1353cdba0cb6757.png)

Client 端控制台打印：

```bash
11:40:11.340  INFO 1296 --- [h_RMROLE_1_2_16] i.s.c.r.p.c.RmBranchCommitProcessor      : rm client handle branch commit process:BranchCommitRequest{xid='192.168.2.117:8091:2252230319988862988', branchId=2252230319988862990, branchType=AT, resourceId='jdbc:mysql://192.168.2.158:3326/seata', applicationData='null'}
11:40:11.340  INFO 1296 --- [h_RMROLE_1_2_16] io.seata.rm.AbstractRMHandler            : Branch committing: 192.168.2.117:8091:2252230319988862988 2252230319988862990 jdbc:mysql://192.168.2.158:3326/seata null
11:40:11.340  INFO 1296 --- [h_RMROLE_1_2_16] io.seata.rm.AbstractRMHandler            : Branch commit result: PhaseTwo_Committed
```

Server 端控制台打印：

```bash
11:40:11.342  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : receive msg[single]: BranchCommitResponse{xid='192.168.2.117:8091:2252230319988862988', branchId=2252230319988862990, branchStatus=PhaseTwo_Committed, resultCode=Success, msg='null'}, clientIp: 192.168.2.117, vgroup: my_test_tx_group
11:40:11.356  INFO --- [      AsyncCommitting_1_1] [server.coordinator.DefaultCore] [bda$doGlobalCommit$1]  [192.168.2.117:8091:2252230319988862988] : Commit branch transaction successfully, xid = 192.168.2.117:8091:2252230319988862988 branchId = 2252230319988862990
```

所有分支提交完成后，全局提交完成。

```bash
11:40:11.461  INFO --- [      AsyncCommitting_1_1] [server.coordinator.DefaultCore] [      doGlobalCommit]  [192.168.2.117:8091:2252230319988862988] : Committing global transaction is successfully done, xid = 192.168.2.117:8091:2252230319988862988.
```

## Seata AT 模式 Rollback 调用流程分析
### 1、调用流程图
![在这里插入图片描述](https://img-blog.csdnimg.cn/88cc5f8baf484141bde61973c3c7cc6c.png)


Rollback 业务流程中，在执行 Account 相关业务逻辑时抛出了运行时异常，然后各分支进行回滚操作。

### 2、全局事务回滚 Rollback
`io.seata.tm.api.DefaultGlobalTransaction#rollback`

![在这里插入图片描述](https://img-blog.csdnimg.cn/c2430f7696854d2a9285389a51fc51df.png)

`io.seata.tm.DefaultTransactionManager#rollback`

![在这里插入图片描述](https://img-blog.csdnimg.cn/82339cbfd4804d809dcb4e2da7323c2e.png)

与 Commit 操作类似，向 Server 端发起请求。

控制台打印：

```bash
11:20:12.756  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : receive msg[single]: GlobalRollbackRequest{xid='192.168.2.126:8091:2252231336169714223', extraData='null'}, clientIp: 192.168.2.126, vgroup: my_test_tx_group
```

### 3、分支回滚处理
`io.seata.core.rpc.processor.client.RmBranchRollbackProcessor#process`

![在这里插入图片描述](https://img-blog.csdnimg.cn/b7c8a27a6605478f831602a975170acc.png)

`io.seata.rm.AbstractRMHandler#doBranchRollback`

![在这里插入图片描述](https://img-blog.csdnimg.cn/a45d5007801b4120a0bb959683d8980d.png)

`io.seata.rm.datasource.DataSourceManager#branchRollback`

![在这里插入图片描述](https://img-blog.csdnimg.cn/14c2f8075d66419b8cbcffbc070d3d94.png)

在 `io.seata.rm.datasource.undo.AbstractUndoLogManager#undo` 这一步完成数据的回滚以及 undo_log 记录删除操作。

Client 端控制台打印：

```bash
11:20:12.789  INFO 7432 --- [h_RMROLE_1_4_16] i.s.c.r.p.c.RmBranchRollbackProcessor    : rm handle branch rollback process:BranchRollbackRequest{xid='192.168.2.126:8091:2252231336169714223', branchId=2252231336169714227, branchType=AT, resourceId='jdbc:mysql://192.168.2.158:3326/seata', applicationData='null'}
11:20:12.790  INFO 7432 --- [h_RMROLE_1_4_16] io.seata.rm.AbstractRMHandler            : Branch Rollbacking: 192.168.2.126:8091:2252231336169714223 2252231336169714227 jdbc:mysql://192.168.2.158:3326/seata
11:20:12.857  INFO 7432 --- [h_RMROLE_1_4_16] i.s.r.d.undo.AbstractUndoLogManager      : xid 192.168.2.126:8091:2252231336169714223 branch 2252231336169714227, undo_log deleted with GlobalFinished
11:20:12.859  INFO 7432 --- [h_RMROLE_1_4_16] i.seata.rm.datasource.DataSourceManager  : branch rollback success, xid:192.168.2.126:8091:2252231336169714223, branchId:2252231336169714227
11:20:12.859  INFO 7432 --- [h_RMROLE_1_4_16] io.seata.rm.AbstractRMHandler            : Branch Rollbacked result: PhaseTwo_Rollbacked
```

Server 端控制台打印：

```bash
11:20:12.861  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : receive msg[single]: BranchRollbackResponse{xid='192.168.2.126:8091:2252231336169714223', branchId=2252231336169714227, branchStatus=PhaseTwo_Rollbacked, resultCode=Success, msg='null'}, clientIp: 192.168.2.126, vgroup: my_test_tx_group
11:20:12.880  INFO --- [verHandlerThread_1_43_500] [server.coordinator.DefaultCore] [a$doGlobalRollback$3]  [192.168.2.126:8091:2252231336169714223] : Rollback branch transaction successfully, xid = 192.168.2.126:8091:2252231336169714223 branchId = 2252231336169714227
```

所有分支回滚完成后，全局回滚完成。

```bash
11:20:12.991  INFO 21468 --- [nio-8084-exec-5] i.seata.tm.api.DefaultGlobalTransaction  : transaction end, xid = 192.168.2.126:8091:2252231336169714223
11:20:12.992  INFO 21468 --- [nio-8084-exec-5] i.seata.tm.api.DefaultGlobalTransaction  : [192.168.2.126:8091:2252231336169714223] rollback status: Rollbacked

11:22:23.467  INFO --- [     RetryRollbacking_1_1] [server.coordinator.DefaultCore] [    doGlobalRollback]  [192.168.2.126:8091:2252231336169714223] : Rollback global transaction successfully, xid = 192.168.2.126:8091:2252231336169714223.
```
## 附录
### Commit 调用流程 Client 端控制台输出
#### 业务模块（business-xa）

```bash
2023-09-01 11:40:09.539  INFO 17724 --- [nio-8084-exec-1] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring DispatcherServlet 'dispatcherServlet'
2023-09-01 11:40:09.540  INFO 17724 --- [nio-8084-exec-1] o.s.web.servlet.DispatcherServlet        : Initializing Servlet 'dispatcherServlet'
2023-09-01 11:40:09.547  INFO 17724 --- [nio-8084-exec-1] o.s.web.servlet.DispatcherServlet        : Completed initialization in 7 ms
2023-09-01 11:40:09.603  INFO 17724 --- [nio-8084-exec-1] io.seata.tm.TransactionManagerHolder     : TransactionManager Singleton io.seata.tm.DefaultTransactionManager@76dc29e2
2023-09-01 11:40:09.630  INFO 17724 --- [nio-8084-exec-1] i.seata.tm.api.DefaultGlobalTransaction  : Begin new global transaction [192.168.2.117:8091:2252230319988862988]
2023-09-01 11:40:09.635  INFO 17724 --- [nio-8084-exec-1] io.seata.sample.service.BusinessService  : New Transaction Begins: 192.168.2.117:8091:2252230319988862988
2023-09-01 11:40:10.291  INFO 17724 --- [nio-8084-exec-1] i.seata.tm.api.DefaultGlobalTransaction  : transaction 192.168.2.117:8091:2252230319988862988 will be commit
2023-09-01 11:40:10.515  INFO 17724 --- [nio-8084-exec-1] i.seata.tm.api.DefaultGlobalTransaction  : transaction end, xid = 192.168.2.117:8091:2252230319988862988
2023-09-01 11:40:10.515  INFO 17724 --- [nio-8084-exec-1] i.seata.tm.api.DefaultGlobalTransaction  : [192.168.2.117:8091:2252230319988862988] commit status: Committed
```

#### 库存模块（stock-xa）

```bash
2023-09-01 11:40:09.651  INFO 1296 --- [nio-8081-exec-2] io.seata.sample.service.StockService     : deduct stock balance in transaction: 192.168.2.117:8091:2252230319988862988
2023-09-01 11:40:09.765  INFO 1296 --- [nio-8081-exec-2] io.seata.rm.AbstractResourceManager      : branch register success, xid:192.168.2.117:8091:2252230319988862988, branchId:2252230319988862990, lockKeys:stock_tbl:9
2023-09-01 11:40:09.818  WARN 1296 --- [nio-8081-exec-2] c.a.c.seata.web.SeataHandlerInterceptor  : xid in change during RPC from 192.168.2.117:8091:2252230319988862988 to null
2023-09-01 11:40:11.340  INFO 1296 --- [h_RMROLE_1_2_16] i.s.c.r.p.c.RmBranchCommitProcessor      : rm client handle branch commit process:BranchCommitRequest{xid='192.168.2.117:8091:2252230319988862988', branchId=2252230319988862990, branchType=AT, resourceId='jdbc:mysql://192.168.2.158:3326/seata', applicationData='null'}
2023-09-01 11:40:11.340  INFO 1296 --- [h_RMROLE_1_2_16] io.seata.rm.AbstractRMHandler            : Branch committing: 192.168.2.117:8091:2252230319988862988 2252230319988862990 jdbc:mysql://192.168.2.158:3326/seata null
2023-09-01 11:40:11.340  INFO 1296 --- [h_RMROLE_1_2_16] io.seata.rm.AbstractRMHandler            : Branch commit result: PhaseTwo_Committed
```

#### 订单模块（order-xa）

```bash
2023-09-01 11:40:09.839  INFO 17280 --- [nio-8082-exec-4] io.seata.sample.service.OrderService     : create order in transaction: 192.168.2.117:8091:2252230319988862988
2023-09-01 11:40:09.963  INFO 17280 --- [nio-8082-exec-4] io.seata.rm.AbstractResourceManager      : branch register success, xid:192.168.2.117:8091:2252230319988862988, branchId:2252230319988862992, lockKeys:order_tbl:15
2023-09-01 11:40:10.290  WARN 17280 --- [nio-8082-exec-4] c.a.c.seata.web.SeataHandlerInterceptor  : xid in change during RPC from 192.168.2.117:8091:2252230319988862988 to null
2023-09-01 11:40:11.356  INFO 17280 --- [h_RMROLE_1_2_16] i.s.c.r.p.c.RmBranchCommitProcessor      : rm client handle branch commit process:BranchCommitRequest{xid='192.168.2.117:8091:2252230319988862988', branchId=2252230319988862992, branchType=AT, resourceId='jdbc:mysql://192.168.2.158:3326/seata', applicationData='null'}
2023-09-01 11:40:11.357  INFO 17280 --- [h_RMROLE_1_2_16] io.seata.rm.AbstractRMHandler            : Branch committing: 192.168.2.117:8091:2252230319988862988 2252230319988862992 jdbc:mysql://192.168.2.158:3326/seata null
2023-09-01 11:40:11.357  INFO 17280 --- [h_RMROLE_1_2_16] io.seata.rm.AbstractRMHandler            : Branch commit result: PhaseTwo_Committed
```

#### 账户模块（account-xa）

```bash
2023-09-01 11:40:10.012  INFO 7696 --- [nio-8083-exec-3] io.seata.sample.service.AccountService   : reduce account balance in transaction: 192.168.2.117:8091:2252230319988862988
2023-09-01 11:40:10.093  INFO 7696 --- [nio-8083-exec-3] io.seata.sample.service.AccountService   : balance after transaction: 7000
2023-09-01 11:40:10.203  INFO 7696 --- [nio-8083-exec-3] io.seata.rm.AbstractResourceManager      : branch register success, xid:192.168.2.117:8091:2252230319988862988, branchId:2252230319988862994, lockKeys:account_tbl:9
2023-09-01 11:40:10.289  WARN 7696 --- [nio-8083-exec-3] c.a.c.seata.web.SeataHandlerInterceptor  : xid in change during RPC from 192.168.2.117:8091:2252230319988862988 to null
2023-09-01 11:40:11.384  INFO 7696 --- [h_RMROLE_1_2_16] i.s.c.r.p.c.RmBranchCommitProcessor      : rm client handle branch commit process:BranchCommitRequest{xid='192.168.2.117:8091:2252230319988862988', branchId=2252230319988862994, branchType=AT, resourceId='jdbc:mysql://192.168.2.158:3326/seata', applicationData='{"autoCommit":false}'}
2023-09-01 11:40:11.385  INFO 7696 --- [h_RMROLE_1_2_16] io.seata.rm.AbstractRMHandler            : Branch committing: 192.168.2.117:8091:2252230319988862988 2252230319988862994 jdbc:mysql://192.168.2.158:3326/seata {"autoCommit":false}
2023-09-01 11:40:11.385  INFO 7696 --- [h_RMROLE_1_2_16] io.seata.rm.AbstractRMHandler            : Branch commit result: PhaseTwo_Committed
```

### Commit 调用流程 Server 端控制台输出

```bash
11:40:09.610  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : receive msg[single]: GlobalBeginRequest{transactionName='purchase(java.lang.String, java.lang.String, int, boolean)', timeout=60000}, clientIp: 192.168.2.117, vgroup: my_test_tx_group
11:40:09.624  INFO --- [verHandlerThread_1_11_500] [coordinator.DefaultCoordinator] [       doGlobalBegin]  [192.168.2.117:8091:2252230319988862988] : Begin new global transaction applicationId: business-xa,transactionServiceGroup: my_test_tx_group, transactionName: purchase(java.lang.String, java.lang.String, int, boolean),timeout:60000,xid:192.168.2.117:8091:2252230319988862988
11:40:09.625  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : result msg[single]: GlobalBeginResponse{xid='192.168.2.117:8091:2252230319988862988', extraData='null', resultCode=Success, msg='null'}, clientIp: 192.168.2.117, vgroup: my_test_tx_group

11:40:09.697  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : receive msg[merged]: BranchRegisterRequest{xid='192.168.2.117:8091:2252230319988862988', branchType=AT, resourceId='jdbc:mysql://192.168.2.158:3326/seata', lockKey='stock_tbl:9', applicationData='null'}, clientIp: 192.168.2.117, vgroup: my_test_tx_group
11:40:09.764  INFO --- [nPool.commonPool-worker-3] [erver.coordinator.AbstractCore] [bda$branchRegister$0]  [192.168.2.117:8091:2252230319988862988] : Register branch successfully, xid = 192.168.2.117:8091:2252230319988862988, branchId = 2252230319988862990, resourceId = jdbc:mysql://192.168.2.158:3326/seata ,lockKeys = stock_tbl:9
11:40:09.764  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : result msg[merged]: BranchRegisterResponse{branchId=2252230319988862990, resultCode=Success, msg='null'}, clientIp: 192.168.2.117, vgroup: my_test_tx_group

11:40:09.882  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : receive msg[merged]: BranchRegisterRequest{xid='192.168.2.117:8091:2252230319988862988', branchType=AT, resourceId='jdbc:mysql://192.168.2.158:3326/seata', lockKey='order_tbl:15', applicationData='null'}, clientIp: 192.168.2.117, vgroup: my_test_tx_group
11:40:09.962  INFO --- [nPool.commonPool-worker-3] [erver.coordinator.AbstractCore] [bda$branchRegister$0]  [192.168.2.117:8091:2252230319988862988] : Register branch successfully, xid = 192.168.2.117:8091:2252230319988862988, branchId = 2252230319988862992, resourceId = jdbc:mysql://192.168.2.158:3326/seata ,lockKeys = order_tbl:15
11:40:09.962  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : result msg[merged]: BranchRegisterResponse{branchId=2252230319988862992, resultCode=Success, msg='null'}, clientIp: 192.168.2.117, vgroup: my_test_tx_group

11:40:10.096  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : receive msg[merged]: BranchRegisterRequest{xid='192.168.2.117:8091:2252230319988862988', branchType=AT, resourceId='jdbc:mysql://192.168.2.158:3326/seata', lockKey='account_tbl:9', applicationData='{"autoCommit":false}'}, clientIp: 192.168.2.117, vgroup: my_test_tx_group
11:40:10.202  INFO --- [nPool.commonPool-worker-3] [erver.coordinator.AbstractCore] [bda$branchRegister$0]  [192.168.2.117:8091:2252230319988862988] : Register branch successfully, xid = 192.168.2.117:8091:2252230319988862988, branchId = 2252230319988862994, resourceId = jdbc:mysql://192.168.2.158:3326/seata ,lockKeys = account_tbl:9
11:40:10.203  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : result msg[merged]: BranchRegisterResponse{branchId=2252230319988862994, resultCode=Success, msg='null'}, clientIp: 192.168.2.117, vgroup: my_test_tx_group

11:40:10.294  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : receive msg[single]: GlobalCommitRequest{xid='192.168.2.117:8091:2252230319988862988', extraData='null'}, clientIp: 192.168.2.117, vgroup: my_test_tx_group
11:40:10.511  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : result msg[single]: GlobalCommitResponse{globalStatus=Committed, resultCode=Success, msg='null'}, clientIp: 192.168.2.117, vgroup: my_test_tx_group

11:40:11.342  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : receive msg[single]: BranchCommitResponse{xid='192.168.2.117:8091:2252230319988862988', branchId=2252230319988862990, branchStatus=PhaseTwo_Committed, resultCode=Success, msg='null'}, clientIp: 192.168.2.117, vgroup: my_test_tx_group
11:40:11.356  INFO --- [      AsyncCommitting_1_1] [server.coordinator.DefaultCore] [bda$doGlobalCommit$1]  [192.168.2.117:8091:2252230319988862988] : Commit branch transaction successfully, xid = 192.168.2.117:8091:2252230319988862988 branchId = 2252230319988862990

11:40:11.359  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : receive msg[single]: BranchCommitResponse{xid='192.168.2.117:8091:2252230319988862988', branchId=2252230319988862992, branchStatus=PhaseTwo_Committed, resultCode=Success, msg='null'}, clientIp: 192.168.2.117, vgroup: my_test_tx_group
11:40:11.383  INFO --- [      AsyncCommitting_1_1] [server.coordinator.DefaultCore] [bda$doGlobalCommit$1]  [192.168.2.117:8091:2252230319988862988] : Commit branch transaction successfully, xid = 192.168.2.117:8091:2252230319988862988 branchId = 2252230319988862992

11:40:11.387  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : receive msg[single]: BranchCommitResponse{xid='192.168.2.117:8091:2252230319988862988', branchId=2252230319988862994, branchStatus=PhaseTwo_Committed, resultCode=Success, msg='null'}, clientIp: 192.168.2.117, vgroup: my_test_tx_group
11:40:11.402  INFO --- [      AsyncCommitting_1_1] [server.coordinator.DefaultCore] [bda$doGlobalCommit$1]  [192.168.2.117:8091:2252230319988862988] : Commit branch transaction successfully, xid = 192.168.2.117:8091:2252230319988862988 branchId = 2252230319988862994

11:40:11.461  INFO --- [      AsyncCommitting_1_1] [server.coordinator.DefaultCore] [      doGlobalCommit]  [192.168.2.117:8091:2252230319988862988] : Committing global transaction is successfully done, xid = 192.168.2.117:8091:2252230319988862988.
```

#### 按时序整理的完整流程输出（包含Client 端、Server 端）

```bash
11:40:09.603  INFO 17724 --- [nio-8084-exec-1] io.seata.tm.TransactionManagerHolder     : TransactionManager Singleton io.seata.tm.DefaultTransactionManager@76dc29e2

11:40:09.610  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : receive msg[single]: GlobalBeginRequest{transactionName='purchase(java.lang.String, java.lang.String, int, boolean)', timeout=60000}, clientIp: 192.168.2.117, vgroup: my_test_tx_group
11:40:09.624  INFO --- [verHandlerThread_1_11_500] [coordinator.DefaultCoordinator] [       doGlobalBegin]  [192.168.2.117:8091:2252230319988862988] : Begin new global transaction applicationId: business-xa,transactionServiceGroup: my_test_tx_group, transactionName: purchase(java.lang.String, java.lang.String, int, boolean),timeout:60000,xid:192.168.2.117:8091:2252230319988862988
11:40:09.625  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : result msg[single]: GlobalBeginResponse{xid='192.168.2.117:8091:2252230319988862988', extraData='null', resultCode=Success, msg='null'}, clientIp: 192.168.2.117, vgroup: my_test_tx_group

11:40:09.630  INFO 17724 --- [nio-8084-exec-1] i.seata.tm.api.DefaultGlobalTransaction  : Begin new global transaction [192.168.2.117:8091:2252230319988862988]
11:40:09.635  INFO 17724 --- [nio-8084-exec-1] io.seata.sample.service.BusinessService  : New Transaction Begins: 192.168.2.117:8091:2252230319988862988

11:40:09.651  INFO 1296 --- [nio-8081-exec-2] io.seata.sample.service.StockService     : deduct stock balance in transaction: 192.168.2.117:8091:2252230319988862988

11:40:09.697  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : receive msg[merged]: BranchRegisterRequest{xid='192.168.2.117:8091:2252230319988862988', branchType=AT, resourceId='jdbc:mysql://192.168.2.158:3326/seata', lockKey='stock_tbl:9', applicationData='null'}, clientIp: 192.168.2.117, vgroup: my_test_tx_group
11:40:09.764  INFO --- [nPool.commonPool-worker-3] [erver.coordinator.AbstractCore] [bda$branchRegister$0]  [192.168.2.117:8091:2252230319988862988] : Register branch successfully, xid = 192.168.2.117:8091:2252230319988862988, branchId = 2252230319988862990, resourceId = jdbc:mysql://192.168.2.158:3326/seata ,lockKeys = stock_tbl:9
11:40:09.764  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : result msg[merged]: BranchRegisterResponse{branchId=2252230319988862990, resultCode=Success, msg='null'}, clientIp: 192.168.2.117, vgroup: my_test_tx_group

11:40:09.765  INFO 1296 --- [nio-8081-exec-2] io.seata.rm.AbstractResourceManager      : branch register success, xid:192.168.2.117:8091:2252230319988862988, branchId:2252230319988862990, lockKeys:stock_tbl:9

11:40:09.818  WARN 1296 --- [nio-8081-exec-2] c.a.c.seata.web.SeataHandlerInterceptor  : xid in change during RPC from 192.168.2.117:8091:2252230319988862988 to null

11:40:09.839  INFO 17280 --- [nio-8082-exec-4] io.seata.sample.service.OrderService     : create order in transaction: 192.168.2.117:8091:2252230319988862988

11:40:09.882  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : receive msg[merged]: BranchRegisterRequest{xid='192.168.2.117:8091:2252230319988862988', branchType=AT, resourceId='jdbc:mysql://192.168.2.158:3326/seata', lockKey='order_tbl:15', applicationData='null'}, clientIp: 192.168.2.117, vgroup: my_test_tx_group
11:40:09.962  INFO --- [nPool.commonPool-worker-3] [erver.coordinator.AbstractCore] [bda$branchRegister$0]  [192.168.2.117:8091:2252230319988862988] : Register branch successfully, xid = 192.168.2.117:8091:2252230319988862988, branchId = 2252230319988862992, resourceId = jdbc:mysql://192.168.2.158:3326/seata ,lockKeys = order_tbl:15
11:40:09.962  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : result msg[merged]: BranchRegisterResponse{branchId=2252230319988862992, resultCode=Success, msg='null'}, clientIp: 192.168.2.117, vgroup: my_test_tx_group

11:40:09.963  INFO 17280 --- [nio-8082-exec-4] io.seata.rm.AbstractResourceManager      : branch register success, xid:192.168.2.117:8091:2252230319988862988, branchId:2252230319988862992, lockKeys:order_tbl:15

11:40:10.012  INFO 7696 --- [nio-8083-exec-3] io.seata.sample.service.AccountService   : reduce account balance in transaction: 192.168.2.117:8091:2252230319988862988
11:40:10.093  INFO 7696 --- [nio-8083-exec-3] io.seata.sample.service.AccountService   : balance after transaction: 7000

11:40:10.096  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : receive msg[merged]: BranchRegisterRequest{xid='192.168.2.117:8091:2252230319988862988', branchType=AT, resourceId='jdbc:mysql://192.168.2.158:3326/seata', lockKey='account_tbl:9', applicationData='{"autoCommit":false}'}, clientIp: 192.168.2.117, vgroup: my_test_tx_group
11:40:10.202  INFO --- [nPool.commonPool-worker-3] [erver.coordinator.AbstractCore] [bda$branchRegister$0]  [192.168.2.117:8091:2252230319988862988] : Register branch successfully, xid = 192.168.2.117:8091:2252230319988862988, branchId = 2252230319988862994, resourceId = jdbc:mysql://192.168.2.158:3326/seata ,lockKeys = account_tbl:9
11:40:10.203  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : result msg[merged]: BranchRegisterResponse{branchId=2252230319988862994, resultCode=Success, msg='null'}, clientIp: 192.168.2.117, vgroup: my_test_tx_group

11:40:10.203  INFO 7696 --- [nio-8083-exec-3] io.seata.rm.AbstractResourceManager      : branch register success, xid:192.168.2.117:8091:2252230319988862988, branchId:2252230319988862994, lockKeys:account_tbl:9

11:40:10.289  WARN 7696 --- [nio-8083-exec-3] c.a.c.seata.web.SeataHandlerInterceptor  : xid in change during RPC from 192.168.2.117:8091:2252230319988862988 to null
11:40:10.290  WARN 17280 --- [nio-8082-exec-4] c.a.c.seata.web.SeataHandlerInterceptor  : xid in change during RPC from 192.168.2.117:8091:2252230319988862988 to null

11:40:10.291  INFO 17724 --- [nio-8084-exec-1] i.seata.tm.api.DefaultGlobalTransaction  : transaction 192.168.2.117:8091:2252230319988862988 will be commit

11:40:10.294  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : receive msg[single]: GlobalCommitRequest{xid='192.168.2.117:8091:2252230319988862988', extraData='null'}, clientIp: 192.168.2.117, vgroup: my_test_tx_group
11:40:10.511  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : result msg[single]: GlobalCommitResponse{globalStatus=Committed, resultCode=Success, msg='null'}, clientIp: 192.168.2.117, vgroup: my_test_tx_group

11:40:10.515  INFO 17724 --- [nio-8084-exec-1] i.seata.tm.api.DefaultGlobalTransaction  : transaction end, xid = 192.168.2.117:8091:2252230319988862988
11:40:10.515  INFO 17724 --- [nio-8084-exec-1] i.seata.tm.api.DefaultGlobalTransaction  : [192.168.2.117:8091:2252230319988862988] commit status: Committed

11:40:11.340  INFO 1296 --- [h_RMROLE_1_2_16] i.s.c.r.p.c.RmBranchCommitProcessor      : rm client handle branch commit process:BranchCommitRequest{xid='192.168.2.117:8091:2252230319988862988', branchId=2252230319988862990, branchType=AT, resourceId='jdbc:mysql://192.168.2.158:3326/seata', applicationData='null'}
11:40:11.340  INFO 1296 --- [h_RMROLE_1_2_16] io.seata.rm.AbstractRMHandler            : Branch committing: 192.168.2.117:8091:2252230319988862988 2252230319988862990 jdbc:mysql://192.168.2.158:3326/seata null
11:40:11.340  INFO 1296 --- [h_RMROLE_1_2_16] io.seata.rm.AbstractRMHandler            : Branch commit result: PhaseTwo_Committed

11:40:11.342  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : receive msg[single]: BranchCommitResponse{xid='192.168.2.117:8091:2252230319988862988', branchId=2252230319988862990, branchStatus=PhaseTwo_Committed, resultCode=Success, msg='null'}, clientIp: 192.168.2.117, vgroup: my_test_tx_group
11:40:11.356  INFO --- [      AsyncCommitting_1_1] [server.coordinator.DefaultCore] [bda$doGlobalCommit$1]  [192.168.2.117:8091:2252230319988862988] : Commit branch transaction successfully, xid = 192.168.2.117:8091:2252230319988862988 branchId = 2252230319988862990

11:40:11.356  INFO 17280 --- [h_RMROLE_1_2_16] i.s.c.r.p.c.RmBranchCommitProcessor      : rm client handle branch commit process:BranchCommitRequest{xid='192.168.2.117:8091:2252230319988862988', branchId=2252230319988862992, branchType=AT, resourceId='jdbc:mysql://192.168.2.158:3326/seata', applicationData='null'}
11:40:11.357  INFO 17280 --- [h_RMROLE_1_2_16] io.seata.rm.AbstractRMHandler            : Branch committing: 192.168.2.117:8091:2252230319988862988 2252230319988862992 jdbc:mysql://192.168.2.158:3326/seata null
11:40:11.357  INFO 17280 --- [h_RMROLE_1_2_16] io.seata.rm.AbstractRMHandler            : Branch commit result: PhaseTwo_Committed

11:40:11.359  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : receive msg[single]: BranchCommitResponse{xid='192.168.2.117:8091:2252230319988862988', branchId=2252230319988862992, branchStatus=PhaseTwo_Committed, resultCode=Success, msg='null'}, clientIp: 192.168.2.117, vgroup: my_test_tx_group
11:40:11.383  INFO --- [      AsyncCommitting_1_1] [server.coordinator.DefaultCore] [bda$doGlobalCommit$1]  [192.168.2.117:8091:2252230319988862988] : Commit branch transaction successfully, xid = 192.168.2.117:8091:2252230319988862988 branchId = 2252230319988862992

11:40:11.384  INFO 7696 --- [h_RMROLE_1_2_16] i.s.c.r.p.c.RmBranchCommitProcessor      : rm client handle branch commit process:BranchCommitRequest{xid='192.168.2.117:8091:2252230319988862988', branchId=2252230319988862994, branchType=AT, resourceId='jdbc:mysql://192.168.2.158:3326/seata', applicationData='{"autoCommit":false}'}
11:40:11.385  INFO 7696 --- [h_RMROLE_1_2_16] io.seata.rm.AbstractRMHandler            : Branch committing: 192.168.2.117:8091:2252230319988862988 2252230319988862994 jdbc:mysql://192.168.2.158:3326/seata {"autoCommit":false}
11:40:11.385  INFO 7696 --- [h_RMROLE_1_2_16] io.seata.rm.AbstractRMHandler            : Branch commit result: PhaseTwo_Committed

11:40:11.387  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : receive msg[single]: BranchCommitResponse{xid='192.168.2.117:8091:2252230319988862988', branchId=2252230319988862994, branchStatus=PhaseTwo_Committed, resultCode=Success, msg='null'}, clientIp: 192.168.2.117, vgroup: my_test_tx_group
11:40:11.402  INFO --- [      AsyncCommitting_1_1] [server.coordinator.DefaultCore] [bda$doGlobalCommit$1]  [192.168.2.117:8091:2252230319988862988] : Commit branch transaction successfully, xid = 192.168.2.117:8091:2252230319988862988 branchId = 2252230319988862994

11:40:11.461  INFO --- [      AsyncCommitting_1_1] [server.coordinator.DefaultCore] [      doGlobalCommit]  [192.168.2.117:8091:2252230319988862988] : Committing global transaction is successfully done, xid = 192.168.2.117:8091:2252230319988862988.
```

### Rollback 调用流程 Client 端控制台输出
#### 业务模块（business-xa）

```bash
2023-09-04 11:20:12.590  INFO 21468 --- [nio-8084-exec-5] i.seata.tm.api.DefaultGlobalTransaction  : Begin new global transaction [192.168.2.126:8091:2252231336169714223]
2023-09-04 11:20:12.590  INFO 21468 --- [nio-8084-exec-5] io.seata.sample.service.BusinessService  : New Transaction Begins: 192.168.2.126:8091:2252231336169714223
2023-09-04 11:20:12.754  INFO 21468 --- [nio-8084-exec-5] i.seata.tm.api.DefaultGlobalTransaction  : transaction 192.168.2.126:8091:2252231336169714223 will be rollback
2023-09-04 11:20:12.991  INFO 21468 --- [nio-8084-exec-5] i.seata.tm.api.DefaultGlobalTransaction  : transaction end, xid = 192.168.2.126:8091:2252231336169714223
2023-09-04 11:20:12.992  INFO 21468 --- [nio-8084-exec-5] i.seata.tm.api.DefaultGlobalTransaction  : [192.168.2.126:8091:2252231336169714223] rollback status: Rollbacked
```

#### 库存模块（stock-xa）

```bash
2023-09-04 11:20:12.592  INFO 23144 --- [nio-8081-exec-4] io.seata.sample.service.StockService     : deduct stock balance in transaction: 192.168.2.126:8091:2252231336169714223
2023-09-04 11:20:12.635  INFO 23144 --- [nio-8081-exec-4] io.seata.rm.AbstractResourceManager      : branch register success, xid:192.168.2.126:8091:2252231336169714223, branchId:2252231336169714225, lockKeys:stock_tbl:13
2023-09-04 11:20:12.663  WARN 23144 --- [nio-8081-exec-4] c.a.c.seata.web.SeataHandlerInterceptor  : xid in change during RPC from 192.168.2.126:8091:2252231336169714223 to null
2023-09-04 11:20:12.883  INFO 23144 --- [h_RMROLE_1_4_16] i.s.c.r.p.c.RmBranchRollbackProcessor    : rm handle branch rollback process:BranchRollbackRequest{xid='192.168.2.126:8091:2252231336169714223', branchId=2252231336169714225, branchType=AT, resourceId='jdbc:mysql://192.168.2.158:3326/seata', applicationData='null'}
2023-09-04 11:20:12.885  INFO 23144 --- [h_RMROLE_1_4_16] io.seata.rm.AbstractRMHandler            : Branch Rollbacking: 192.168.2.126:8091:2252231336169714223 2252231336169714225 jdbc:mysql://192.168.2.158:3326/seata
2023-09-04 11:20:12.949  INFO 23144 --- [h_RMROLE_1_4_16] i.s.r.d.undo.AbstractUndoLogManager      : xid 192.168.2.126:8091:2252231336169714223 branch 2252231336169714225, undo_log deleted with GlobalFinished
2023-09-04 11:20:12.953  INFO 23144 --- [h_RMROLE_1_4_16] i.seata.rm.datasource.DataSourceManager  : branch rollback success, xid:192.168.2.126:8091:2252231336169714223, branchId:2252231336169714225
2023-09-04 11:20:12.953  INFO 23144 --- [h_RMROLE_1_4_16] io.seata.rm.AbstractRMHandler            : Branch Rollbacked result: PhaseTwo_Rollbacked
```

#### 订单模块（order-xa）

```bash
2023-09-04 11:20:12.667  INFO 7432 --- [nio-8082-exec-4] io.seata.sample.service.OrderService     : create order in transaction: 192.168.2.126:8091:2252231336169714223
2023-09-04 11:20:12.711  INFO 7432 --- [nio-8082-exec-4] io.seata.rm.AbstractResourceManager      : branch register success, xid:192.168.2.126:8091:2252231336169714223, branchId:2252231336169714227, lockKeys:order_tbl:28
java.lang.RuntimeException: Failed to call Account Service. 
	at io.seata.sample.service.OrderService.create(OrderService.java:37)
	at io.seata.sample.controller.OrderController.create(OrderController.java:21)
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:498)
	at org.springframework.web.method.support.InvocableHandlerMethod.doInvoke(InvocableHandlerMethod.java:190)
	at org.springframework.web.method.support.InvocableHandlerMethod.invokeForRequest(InvocableHandlerMethod.java:138)
	at org.springframework.web.servlet.mvc.method.annotation.ServletInvocableHandlerMethod.invokeAndHandle(ServletInvocableHandlerMethod.java:105)
	at org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter.invokeHandlerMethod(RequestMappingHandlerAdapter.java:879)
	at org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter.handleInternal(RequestMappingHandlerAdapter.java:793)
	at org.springframework.web.servlet.mvc.method.AbstractHandlerMethodAdapter.handle(AbstractHandlerMethodAdapter.java:87)
	at org.springframework.web.servlet.DispatcherServlet.doDispatch(DispatcherServlet.java:1040)
	at org.springframework.web.servlet.DispatcherServlet.doService(DispatcherServlet.java:943)
	at org.springframework.web.servlet.FrameworkServlet.processRequest(FrameworkServlet.java:1006)
	at org.springframework.web.servlet.FrameworkServlet.doGet(FrameworkServlet.java:898)
	at javax.servlet.http.HttpServlet.service(HttpServlet.java:634)
	at org.springframework.web.servlet.FrameworkServlet.service(FrameworkServlet.java:883)
	at javax.servlet.http.HttpServlet.service(HttpServlet.java:741)
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:231)
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:166)
	at org.apache.tomcat.websocket.server.WsFilter.doFilter(WsFilter.java:53)
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:193)
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:166)
	at org.springframework.web.filter.RequestContextFilter.doFilterInternal(RequestContextFilter.java:100)
	at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:119)
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:193)
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:166)
	at org.springframework.web.filter.FormContentFilter.doFilterInternal(FormContentFilter.java:93)
	at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:119)
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:193)
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:166)
	at org.springframework.web.filter.CharacterEncodingFilter.doFilterInternal(CharacterEncodingFilter.java:201)
	at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:119)
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:193)
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:166)
	at org.apache.catalina.core.StandardWrapperValve.invoke(StandardWrapperValve.java:202)
	at org.apache.catalina.core.StandardContextValve.invoke(StandardContextValve.java:96)
	at org.apache.catalina.authenticator.AuthenticatorBase.invoke(AuthenticatorBase.java:541)
	at org.apache.catalina.core.StandardHostValve.invoke(StandardHostValve.java:139)
	at org.apache.catalina.valves.ErrorReportValve.invoke(ErrorReportValve.java:92)
	at org.apache.catalina.core.StandardEngineValve.invoke(StandardEngineValve.java:74)
	at org.apache.catalina.connector.CoyoteAdapter.service(CoyoteAdapter.java:343)
	at org.apache.coyote.http11.Http11Processor.service(Http11Processor.java:373)
	at org.apache.coyote.AbstractProcessorLight.process(AbstractProcessorLight.java:65)
	at org.apache.coyote.AbstractProtocol$ConnectionHandler.process(AbstractProtocol.java:868)
	at org.apache.tomcat.util.net.NioEndpoint$SocketProcessor.doRun(NioEndpoint.java:1590)
	at org.apache.tomcat.util.net.SocketProcessorBase.run(SocketProcessorBase.java:49)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
	at org.apache.tomcat.util.threads.TaskThread$WrappingRunnable.run(TaskThread.java:61)
	at java.lang.Thread.run(Thread.java:748)
2023-09-04 11:20:12.753  WARN 7432 --- [nio-8082-exec-4] c.a.c.seata.web.SeataHandlerInterceptor  : xid in change during RPC from 192.168.2.126:8091:2252231336169714223 to null
2023-09-04 11:20:12.789  INFO 7432 --- [h_RMROLE_1_4_16] i.s.c.r.p.c.RmBranchRollbackProcessor    : rm handle branch rollback process:BranchRollbackRequest{xid='192.168.2.126:8091:2252231336169714223', branchId=2252231336169714227, branchType=AT, resourceId='jdbc:mysql://192.168.2.158:3326/seata', applicationData='null'}
2023-09-04 11:20:12.790  INFO 7432 --- [h_RMROLE_1_4_16] io.seata.rm.AbstractRMHandler            : Branch Rollbacking: 192.168.2.126:8091:2252231336169714223 2252231336169714227 jdbc:mysql://192.168.2.158:3326/seata
2023-09-04 11:20:12.857  INFO 7432 --- [h_RMROLE_1_4_16] i.s.r.d.undo.AbstractUndoLogManager      : xid 192.168.2.126:8091:2252231336169714223 branch 2252231336169714227, undo_log deleted with GlobalFinished
2023-09-04 11:20:12.859  INFO 7432 --- [h_RMROLE_1_4_16] i.seata.rm.datasource.DataSourceManager  : branch rollback success, xid:192.168.2.126:8091:2252231336169714223, branchId:2252231336169714227
2023-09-04 11:20:12.859  INFO 7432 --- [h_RMROLE_1_4_16] io.seata.rm.AbstractRMHandler            : Branch Rollbacked result: PhaseTwo_Rollbacked
```

#### 账户模块（account-xa）

```bash
2023-09-04 11:20:12.730  INFO 16208 --- [nio-8083-exec-4] io.seata.sample.service.AccountService   : reduce account balance in transaction: 192.168.2.126:8091:2252231336169714223
2023-09-04 11:20:12.740  INFO 16208 --- [nio-8083-exec-4] io.seata.sample.service.AccountService   : balance after transaction: -2000
2023-09-04 11:20:12.749  WARN 16208 --- [nio-8083-exec-4] c.a.c.seata.web.SeataHandlerInterceptor  : xid in change during RPC from 192.168.2.126:8091:2252231336169714223 to null
java.lang.RuntimeException: Not Enough Money ...
	at io.seata.sample.service.AccountService.reduce(AccountService.java:31)
	at io.seata.sample.service.AccountService$$FastClassBySpringCGLIB$$533284cb.invoke(<generated>)
	at org.springframework.cglib.proxy.MethodProxy.invoke(MethodProxy.java:218)
	at org.springframework.aop.framework.CglibAopProxy$CglibMethodInvocation.invokeJoinpoint(CglibAopProxy.java:771)
	at org.springframework.aop.framework.ReflectiveMethodInvocation.proceed(ReflectiveMethodInvocation.java:163)
	at org.springframework.aop.framework.CglibAopProxy$CglibMethodInvocation.proceed(CglibAopProxy.java:749)
	at org.springframework.transaction.interceptor.TransactionAspectSupport.invokeWithinTransaction(TransactionAspectSupport.java:367)
	at org.springframework.transaction.interceptor.TransactionInterceptor.invoke(TransactionInterceptor.java:118)
	at org.springframework.aop.framework.ReflectiveMethodInvocation.proceed(ReflectiveMethodInvocation.java:186)
	at org.springframework.aop.framework.CglibAopProxy$CglibMethodInvocation.proceed(CglibAopProxy.java:749)
	at org.springframework.aop.framework.CglibAopProxy$DynamicAdvisedInterceptor.intercept(CglibAopProxy.java:691)
	at io.seata.sample.service.AccountService$$EnhancerBySpringCGLIB$$a0f97bf3.reduce(<generated>)
	at io.seata.sample.controller.AccountController.reduce(AccountController.java:21)
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:498)
	at org.springframework.web.method.support.InvocableHandlerMethod.doInvoke(InvocableHandlerMethod.java:190)
	at org.springframework.web.method.support.InvocableHandlerMethod.invokeForRequest(InvocableHandlerMethod.java:138)
	at org.springframework.web.servlet.mvc.method.annotation.ServletInvocableHandlerMethod.invokeAndHandle(ServletInvocableHandlerMethod.java:105)
	at org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter.invokeHandlerMethod(RequestMappingHandlerAdapter.java:879)
	at org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter.handleInternal(RequestMappingHandlerAdapter.java:793)
	at org.springframework.web.servlet.mvc.method.AbstractHandlerMethodAdapter.handle(AbstractHandlerMethodAdapter.java:87)
	at org.springframework.web.servlet.DispatcherServlet.doDispatch(DispatcherServlet.java:1040)
	at org.springframework.web.servlet.DispatcherServlet.doService(DispatcherServlet.java:943)
	at org.springframework.web.servlet.FrameworkServlet.processRequest(FrameworkServlet.java:1006)
	at org.springframework.web.servlet.FrameworkServlet.doGet(FrameworkServlet.java:898)
	at javax.servlet.http.HttpServlet.service(HttpServlet.java:634)
	at org.springframework.web.servlet.FrameworkServlet.service(FrameworkServlet.java:883)
	at javax.servlet.http.HttpServlet.service(HttpServlet.java:741)
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:231)
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:166)
	at org.apache.tomcat.websocket.server.WsFilter.doFilter(WsFilter.java:53)
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:193)
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:166)
	at org.springframework.web.filter.RequestContextFilter.doFilterInternal(RequestContextFilter.java:100)
	at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:119)
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:193)
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:166)
	at org.springframework.web.filter.FormContentFilter.doFilterInternal(FormContentFilter.java:93)
	at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:119)
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:193)
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:166)
	at org.springframework.web.filter.CharacterEncodingFilter.doFilterInternal(CharacterEncodingFilter.java:201)
	at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:119)
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:193)
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:166)
	at org.apache.catalina.core.StandardWrapperValve.invoke(StandardWrapperValve.java:202)
	at org.apache.catalina.core.StandardContextValve.invoke(StandardContextValve.java:96)
	at org.apache.catalina.authenticator.AuthenticatorBase.invoke(AuthenticatorBase.java:541)
	at org.apache.catalina.core.StandardHostValve.invoke(StandardHostValve.java:139)
	at org.apache.catalina.valves.ErrorReportValve.invoke(ErrorReportValve.java:92)
	at org.apache.catalina.core.StandardEngineValve.invoke(StandardEngineValve.java:74)
	at org.apache.catalina.connector.CoyoteAdapter.service(CoyoteAdapter.java:343)
	at org.apache.coyote.http11.Http11Processor.service(Http11Processor.java:373)
	at org.apache.coyote.AbstractProcessorLight.process(AbstractProcessorLight.java:65)
	at org.apache.coyote.AbstractProtocol$ConnectionHandler.process(AbstractProtocol.java:868)
	at org.apache.tomcat.util.net.NioEndpoint$SocketProcessor.doRun(NioEndpoint.java:1590)
	at org.apache.tomcat.util.net.SocketProcessorBase.run(SocketProcessorBase.java:49)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
	at org.apache.tomcat.util.threads.TaskThread$WrappingRunnable.run(TaskThread.java:61)
	at java.lang.Thread.run(Thread.java:748)
```

### Rollback 调用流程 Server 端控制台输出

```bash
11:20:12.576  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : receive msg[single]: GlobalBeginRequest{transactionName='purchase(java.lang.String, java.lang.String, int, boolean)', timeout=60000}, clientIp: 192.168.2.126, vgroup: my_test_tx_group
11:20:12.589  INFO --- [verHandlerThread_1_40_500] [coordinator.DefaultCoordinator] [       doGlobalBegin]  [192.168.2.126:8091:2252231336169714223] : Begin new global transaction applicationId: business-xa,transactionServiceGroup: my_test_tx_group, transactionName: purchase(java.lang.String, java.lang.String, int, boolean),timeout:60000,xid:192.168.2.126:8091:2252231336169714223
11:20:12.590  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : result msg[single]: GlobalBeginResponse{xid='192.168.2.126:8091:2252231336169714223', extraData='null', resultCode=Success, msg='null'}, clientIp: 192.168.2.126, vgroup: my_test_tx_group
11:20:12.601  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : receive msg[merged]: BranchRegisterRequest{xid='192.168.2.126:8091:2252231336169714223', branchType=AT, resourceId='jdbc:mysql://192.168.2.158:3326/seata', lockKey='stock_tbl:13', applicationData='null'}, clientIp: 192.168.2.126, vgroup: my_test_tx_group
11:20:12.633  INFO --- [nPool.commonPool-worker-9] [erver.coordinator.AbstractCore] [bda$branchRegister$0]  [192.168.2.126:8091:2252231336169714223] : Register branch successfully, xid = 192.168.2.126:8091:2252231336169714223, branchId = 2252231336169714225, resourceId = jdbc:mysql://192.168.2.158:3326/seata ,lockKeys = stock_tbl:13
11:20:12.634  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : result msg[merged]: BranchRegisterResponse{branchId=2252231336169714225, resultCode=Success, msg='null'}, clientIp: 192.168.2.126, vgroup: my_test_tx_group
11:20:12.675  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : receive msg[merged]: BranchRegisterRequest{xid='192.168.2.126:8091:2252231336169714223', branchType=AT, resourceId='jdbc:mysql://192.168.2.158:3326/seata', lockKey='order_tbl:28', applicationData='null'}, clientIp: 192.168.2.126, vgroup: my_test_tx_group
11:20:12.709  INFO --- [nPool.commonPool-worker-9] [erver.coordinator.AbstractCore] [bda$branchRegister$0]  [192.168.2.126:8091:2252231336169714223] : Register branch successfully, xid = 192.168.2.126:8091:2252231336169714223, branchId = 2252231336169714227, resourceId = jdbc:mysql://192.168.2.158:3326/seata ,lockKeys = order_tbl:28
11:20:12.710  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : result msg[merged]: BranchRegisterResponse{branchId=2252231336169714227, resultCode=Success, msg='null'}, clientIp: 192.168.2.126, vgroup: my_test_tx_group
11:20:12.756  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : receive msg[single]: GlobalRollbackRequest{xid='192.168.2.126:8091:2252231336169714223', extraData='null'}, clientIp: 192.168.2.126, vgroup: my_test_tx_group
11:20:12.861  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : receive msg[single]: BranchRollbackResponse{xid='192.168.2.126:8091:2252231336169714223', branchId=2252231336169714227, branchStatus=PhaseTwo_Rollbacked, resultCode=Success, msg='null'}, clientIp: 192.168.2.126, vgroup: my_test_tx_group
11:20:12.880  INFO --- [verHandlerThread_1_43_500] [server.coordinator.DefaultCore] [a$doGlobalRollback$3]  [192.168.2.126:8091:2252231336169714223] : Rollback branch transaction successfully, xid = 192.168.2.126:8091:2252231336169714223 branchId = 2252231336169714227
11:20:12.957  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : receive msg[single]: BranchRollbackResponse{xid='192.168.2.126:8091:2252231336169714223', branchId=2252231336169714225, branchStatus=PhaseTwo_Rollbacked, resultCode=Success, msg='null'}, clientIp: 192.168.2.126, vgroup: my_test_tx_group
11:20:12.984  INFO --- [verHandlerThread_1_43_500] [server.coordinator.DefaultCore] [a$doGlobalRollback$3]  [192.168.2.126:8091:2252231336169714223] : Rollback branch transaction successfully, xid = 192.168.2.126:8091:2252231336169714223 branchId = 2252231336169714225
11:20:12.985  INFO --- [verHandlerThread_1_43_500] [server.coordinator.DefaultCore] [    doGlobalRollback]  [192.168.2.126:8091:2252231336169714223] : Rollback global transaction successfully, xid = 192.168.2.126:8091:2252231336169714223.
11:20:12.986  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : result msg[single]: GlobalRollbackResponse{globalStatus=Rollbacked, resultCode=Success, msg='null'}, clientIp: 192.168.2.126, vgroup: my_test_tx_group
11:22:23.467  INFO --- [     RetryRollbacking_1_1] [server.coordinator.DefaultCore] [    doGlobalRollback]  [192.168.2.126:8091:2252231336169714223] : Rollback global transaction successfully, xid = 192.168.2.126:8091:2252231336169714223.
```

#### 按时序整理的完整流程输出（包含Client 端、Server 端）

```bash
11:20:12.576  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : receive msg[single]: GlobalBeginRequest{transactionName='purchase(java.lang.String, java.lang.String, int, boolean)', timeout=60000}, clientIp: 192.168.2.126, vgroup: my_test_tx_group
11:20:12.589  INFO --- [verHandlerThread_1_40_500] [coordinator.DefaultCoordinator] [       doGlobalBegin]  [192.168.2.126:8091:2252231336169714223] : Begin new global transaction applicationId: business-xa,transactionServiceGroup: my_test_tx_group, transactionName: purchase(java.lang.String, java.lang.String, int, boolean),timeout:60000,xid:192.168.2.126:8091:2252231336169714223
11:20:12.590  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : result msg[single]: GlobalBeginResponse{xid='192.168.2.126:8091:2252231336169714223', extraData='null', resultCode=Success, msg='null'}, clientIp: 192.168.2.126, vgroup: my_test_tx_group

11:20:12.590  INFO 21468 --- [nio-8084-exec-5] i.seata.tm.api.DefaultGlobalTransaction  : Begin new global transaction [192.168.2.126:8091:2252231336169714223]
11:20:12.590  INFO 21468 --- [nio-8084-exec-5] io.seata.sample.service.BusinessService  : New Transaction Begins: 192.168.2.126:8091:2252231336169714223

11:20:12.592  INFO 23144 --- [nio-8081-exec-4] io.seata.sample.service.StockService     : deduct stock balance in transaction: 192.168.2.126:8091:2252231336169714223

11:20:12.601  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : receive msg[merged]: BranchRegisterRequest{xid='192.168.2.126:8091:2252231336169714223', branchType=AT, resourceId='jdbc:mysql://192.168.2.158:3326/seata', lockKey='stock_tbl:13', applicationData='null'}, clientIp: 192.168.2.126, vgroup: my_test_tx_group
11:20:12.633  INFO --- [nPool.commonPool-worker-9] [erver.coordinator.AbstractCore] [bda$branchRegister$0]  [192.168.2.126:8091:2252231336169714223] : Register branch successfully, xid = 192.168.2.126:8091:2252231336169714223, branchId = 2252231336169714225, resourceId = jdbc:mysql://192.168.2.158:3326/seata ,lockKeys = stock_tbl:13
11:20:12.634  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : result msg[merged]: BranchRegisterResponse{branchId=2252231336169714225, resultCode=Success, msg='null'}, clientIp: 192.168.2.126, vgroup: my_test_tx_group

11:20:12.635  INFO 23144 --- [nio-8081-exec-4] io.seata.rm.AbstractResourceManager      : branch register success, xid:192.168.2.126:8091:2252231336169714223, branchId:2252231336169714225, lockKeys:stock_tbl:13

11:20:12.667  INFO 7432 --- [nio-8082-exec-4] io.seata.sample.service.OrderService     : create order in transaction: 192.168.2.126:8091:2252231336169714223

11:20:12.675  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : receive msg[merged]: BranchRegisterRequest{xid='192.168.2.126:8091:2252231336169714223', branchType=AT, resourceId='jdbc:mysql://192.168.2.158:3326/seata', lockKey='order_tbl:28', applicationData='null'}, clientIp: 192.168.2.126, vgroup: my_test_tx_group
11:20:12.709  INFO --- [nPool.commonPool-worker-9] [erver.coordinator.AbstractCore] [bda$branchRegister$0]  [192.168.2.126:8091:2252231336169714223] : Register branch successfully, xid = 192.168.2.126:8091:2252231336169714223, branchId = 2252231336169714227, resourceId = jdbc:mysql://192.168.2.158:3326/seata ,lockKeys = order_tbl:28
11:20:12.710  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : result msg[merged]: BranchRegisterResponse{branchId=2252231336169714227, resultCode=Success, msg='null'}, clientIp: 192.168.2.126, vgroup: my_test_tx_group

11:20:12.711  INFO 7432 --- [nio-8082-exec-4] io.seata.rm.AbstractResourceManager      : branch register success, xid:192.168.2.126:8091:2252231336169714223, branchId:2252231336169714227, lockKeys:order_tbl:28

11:20:12.730  INFO 16208 --- [nio-8083-exec-4] io.seata.sample.service.AccountService   : reduce account balance in transaction: 192.168.2.126:8091:2252231336169714223
11:20:12.740  INFO 16208 --- [nio-8083-exec-4] io.seata.sample.service.AccountService   : balance after transaction: -2000
java.lang.RuntimeException: Not Enough Money ...

11:20:12.754  INFO 21468 --- [nio-8084-exec-5] i.seata.tm.api.DefaultGlobalTransaction  : transaction 192.168.2.126:8091:2252231336169714223 will be rollback

11:20:12.756  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : receive msg[single]: GlobalRollbackRequest{xid='192.168.2.126:8091:2252231336169714223', extraData='null'}, clientIp: 192.168.2.126, vgroup: my_test_tx_group

11:20:12.789  INFO 7432 --- [h_RMROLE_1_4_16] i.s.c.r.p.c.RmBranchRollbackProcessor    : rm handle branch rollback process:BranchRollbackRequest{xid='192.168.2.126:8091:2252231336169714223', branchId=2252231336169714227, branchType=AT, resourceId='jdbc:mysql://192.168.2.158:3326/seata', applicationData='null'}
11:20:12.790  INFO 7432 --- [h_RMROLE_1_4_16] io.seata.rm.AbstractRMHandler            : Branch Rollbacking: 192.168.2.126:8091:2252231336169714223 2252231336169714227 jdbc:mysql://192.168.2.158:3326/seata

11:20:12.857  INFO 7432 --- [h_RMROLE_1_4_16] i.s.r.d.undo.AbstractUndoLogManager      : xid 192.168.2.126:8091:2252231336169714223 branch 2252231336169714227, undo_log deleted with GlobalFinished
11:20:12.859  INFO 7432 --- [h_RMROLE_1_4_16] i.seata.rm.datasource.DataSourceManager  : branch rollback success, xid:192.168.2.126:8091:2252231336169714223, branchId:2252231336169714227
11:20:12.859  INFO 7432 --- [h_RMROLE_1_4_16] io.seata.rm.AbstractRMHandler            : Branch Rollbacked result: PhaseTwo_Rollbacked

11:20:12.861  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : receive msg[single]: BranchRollbackResponse{xid='192.168.2.126:8091:2252231336169714223', branchId=2252231336169714227, branchStatus=PhaseTwo_Rollbacked, resultCode=Success, msg='null'}, clientIp: 192.168.2.126, vgroup: my_test_tx_group
11:20:12.880  INFO --- [verHandlerThread_1_43_500] [server.coordinator.DefaultCore] [a$doGlobalRollback$3]  [192.168.2.126:8091:2252231336169714223] : Rollback branch transaction successfully, xid = 192.168.2.126:8091:2252231336169714223 branchId = 2252231336169714227

11:20:12.883  INFO 23144 --- [h_RMROLE_1_4_16] i.s.c.r.p.c.RmBranchRollbackProcessor    : rm handle branch rollback process:BranchRollbackRequest{xid='192.168.2.126:8091:2252231336169714223', branchId=2252231336169714225, branchType=AT, resourceId='jdbc:mysql://192.168.2.158:3326/seata', applicationData='null'}
11:20:12.885  INFO 23144 --- [h_RMROLE_1_4_16] io.seata.rm.AbstractRMHandler            : Branch Rollbacking: 192.168.2.126:8091:2252231336169714223 2252231336169714225 jdbc:mysql://192.168.2.158:3326/seata
11:20:12.949  INFO 23144 --- [h_RMROLE_1_4_16] i.s.r.d.undo.AbstractUndoLogManager      : xid 192.168.2.126:8091:2252231336169714223 branch 2252231336169714225, undo_log deleted with GlobalFinished
11:20:12.953  INFO 23144 --- [h_RMROLE_1_4_16] i.seata.rm.datasource.DataSourceManager  : branch rollback success, xid:192.168.2.126:8091:2252231336169714223, branchId:2252231336169714225
11:20:12.953  INFO 23144 --- [h_RMROLE_1_4_16] io.seata.rm.AbstractRMHandler            : Branch Rollbacked result: PhaseTwo_Rollbacked

11:20:12.957  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : receive msg[single]: BranchRollbackResponse{xid='192.168.2.126:8091:2252231336169714223', branchId=2252231336169714225, branchStatus=PhaseTwo_Rollbacked, resultCode=Success, msg='null'}, clientIp: 192.168.2.126, vgroup: my_test_tx_group
11:20:12.984  INFO --- [verHandlerThread_1_43_500] [server.coordinator.DefaultCore] [a$doGlobalRollback$3]  [192.168.2.126:8091:2252231336169714223] : Rollback branch transaction successfully, xid = 192.168.2.126:8091:2252231336169714223 branchId = 2252231336169714225
11:20:12.985  INFO --- [verHandlerThread_1_43_500] [server.coordinator.DefaultCore] [    doGlobalRollback]  [192.168.2.126:8091:2252231336169714223] : Rollback global transaction successfully, xid = 192.168.2.126:8091:2252231336169714223.
11:20:12.986  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : result msg[single]: GlobalRollbackResponse{globalStatus=Rollbacked, resultCode=Success, msg='null'}, clientIp: 192.168.2.126, vgroup: my_test_tx_group

11:20:12.991  INFO 21468 --- [nio-8084-exec-5] i.seata.tm.api.DefaultGlobalTransaction  : transaction end, xid = 192.168.2.126:8091:2252231336169714223
11:20:12.992  INFO 21468 --- [nio-8084-exec-5] i.seata.tm.api.DefaultGlobalTransaction  : [192.168.2.126:8091:2252231336169714223] rollback status: Rollbacked

11:22:23.467  INFO --- [     RetryRollbacking_1_1] [server.coordinator.DefaultCore] [    doGlobalRollback]  [192.168.2.126:8091:2252231336169714223] : Rollback global transaction successfully, xid = 192.168.2.126:8091:2252231336169714223.
```

（完）