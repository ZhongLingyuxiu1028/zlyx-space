# （四）Seata TCC 模式全局锁相关知识简单整理
---


## 前言
前面的博客分别整理了 XA 模式与 AT 模式，它们都属于两阶段提交模型，这篇文章也是讲述另外一个两阶段模型—— TCC 模式。对于最常用的 AT 模型，只需要加入相关的配置信息、添加 undo_log 数据库表，并编写业务代码即可，用户无需关心底层数据操作；而相对于自动挡的 AT 模式，TCC 模式除了业务逻辑之外还需要管理自定义的 commit 或者 rollback 操作，官方也给出了以下的说明：

> (截图自官方文档）<br>
> ![在这里插入图片描述](images/04_SeataTCC/28db3b6e025243e18898284f34588ec1.png)

## 参考目录
- [Seata 官方文档](https://seata.io/zh-cn/docs/overview/what-is-seata.html)
- [Seata TCC 模式](https://seata.io/zh-cn/docs/dev/mode/tcc-mode.html)
- [《阿里云云原生架构实践》](https://weread.qq.com/web/bookDetail/731327d07248f2dd7319578)<br>
书本章节 3.5 介绍了分布式事务模式的相关内容。

## 版本说明
本文使用的版本如下：
- `Seata`：`V1.7.0`
- `druid-spring-boot-starter`：`V1.2.16`

## 前置知识
### 1、TCC 模式预留资源
TCC 模式在官方文档中的讲解比较简单，所以在此补充说明一下。

> TCC 实际上是 Try、Confirm 和 Cancel 的简称，将事务的提交过程分为 try-confirm cancel 三个阶段。try 阶段完成业务检查、预留业务资源；confirm 阶段使用预留的资源执行业务操作；cancel 阶段取消执行业务操作，释放预留的资源。<br>
> （摘自《阿里云云原生架构实践》3.5.3 TCC）

在前言部分也有官方文档的截图是关于 AT 模式与 TCC 模式的对比。上面的文字反复提及了一个概念——预留资源，这个概念实际上就是充当了 AT 模式中前后镜像的功能。

举个栗子来说明一下：
- 用户 A 和用户 B，各有 ￥100。
- A 向 B 转账 ￥50。（**这个变化的 ￥50 即预留资源**）
- 转账完成，A 剩余￥50，B 则有￥150。

可以简单点理解成，预留资源就是用来做回滚用的。假如 A 向 B 转账过程中出现了异常，那么就会根据这个预留资源把 ￥50 加回给 A。

因此，在下面的 Demo 中，我给原本的三张业务表分别加上了各自的预留资源的字段，便于进行后续操作。

### 2、TCC 模式可能会出现的问题
因为 TCC 模式实际上在生产环境用得比较少，所以在此简单带过一下可能会出现的一些问题，方便以后回滚复习。
#### 2.1、幂等性问题
这个比较好理解，在 TCC 模式中，一个全局事务 try-comfirm 或者是 try-cancel 只能执行一次。但是可能会由于网络问题等造成的执行了两次相同的方法而导致数据前后不一致。
#### 2.2、空回滚问题
这个问题和预留资源相关。

首先需要知道分别定义好了 try、comfirm、cancel 的实现方法。假设在 try 阶段，业务代码还没有执行到与数据库相关的操作时就已经报错了，此时会进入 cancel 方法，因为没有预留资源，当回滚时会出现数据不一致的情况，就是空回滚。

例如：A 转账给 B，执行了 `A - 50` 的操作，但是还没执行到到 `B + 50` 的 SQL 操作，此时报错了，此时进入了 cancel 阶段，执行 rollback 方法，正常来说结果应该是 A 和 B 各 100，但是实际上得到的结果却是 A 变回 100，B 变成了 150。
#### 2.3、悬挂问题
还是举个栗子来说明：
- 全局事务有 A 和 B 两个事务分支属于两个不同模块，相互之间通过 RPC 调用。
- A 调用 B 时，由于网络延迟导致超时了，此时 A 进行了回滚操作。
- 回滚操作之后，RPC 请求又到达了 B 并执行了 try 方法。
- 由于 A 执行了 cancel，导致了 B 预留的资源无法释放。

这就是悬挂。

Seata TCC事务模式在 `V1.5.0` 增加防悬挂功能，需要新增一张表 `tcc_fence_log`，在 Demo 中也进行了相应配置。
## 测试 Demo
官方也提供了关于 TCC 模式的 Demo，不过我觉得不太好用，所以就在原本 XA 模式 Demo 的基础上重新写了一个 TCC Demo，能复用的代码几乎都复用了，Demo 以及使用的 Server 打成了压缩包（在文章开头）可以自行下载使用。

### 1、数据库表结构
因为业务流程没有变化，所以业务表使用的还是最开始的三张业务表，只是在此基础上加了数据留存的字段，以及加了一张 TCC 模式需要使用的表 `tcc_fence_log`。

相关的 SQL：
```sql
-- -------------------------------- The script use tcc fence  --------------------------------
CREATE TABLE IF NOT EXISTS `tcc_fence_log`
(
    `xid`           VARCHAR(128)  NOT NULL COMMENT 'global id',
    `branch_id`     BIGINT        NOT NULL COMMENT 'branch id',
    `action_name`   VARCHAR(64)   NOT NULL COMMENT 'action name',
    `status`        TINYINT       NOT NULL COMMENT 'status(tried:1;committed:2;rollbacked:3;suspended:4)',
    `gmt_create`    DATETIME(3)   NOT NULL COMMENT 'create time',
    `gmt_modified`  DATETIME(3)   NOT NULL COMMENT 'update time',
    PRIMARY KEY (`xid`, `branch_id`),
    KEY `idx_gmt_modified` (`gmt_modified`),
    KEY `idx_status` (`status`)
) ENGINE = InnoDB
DEFAULT CHARSET = utf8mb4;


-- stock --
ALTER TABLE `seata`.`stock_tbl` 
ADD COLUMN `reduce` int(0) NULL DEFAULT 0 AFTER `count`;

-- order --
ALTER TABLE `seata`.`order_tbl` 
ADD COLUMN `order_no` varchar(255) NULL AFTER `money`;

-- account --
ALTER TABLE `seata`.`account_tbl` 
ADD COLUMN `reduce` int(0) NULL DEFAULT 0 AFTER `money`;
```

`tcc_fence_log` 是 TCC 模式防悬挂功能的表。关于悬挂问题后面再展开。

库存表 `stock_tbl` 新增预留资源字段 `reduce` 记录库存变化。

订单表 `order_tbl` 新增预留资源字段 `order_no` 记录订单数据。

账户表 `account_tbl` 新增预留资源字段 `reduce` 记录余额变化。

### 2、模块说明
![在这里插入图片描述](images/04_SeataTCC/f9a58ff696be47d8800feef89418ad6c.png)

Demo 一共四个模块，是经典的下单流程。**按照如下顺序启动**：
- 账户模块（Port：8083） 
- 订单模块（Port：8082） 
- 库存模块（Port：8081） 
- 业务模块（Port：8084） 

模块间使用 openfeign 进行调用。

### 3、调用逻辑说明

入口：业务模块
```url
http://127.0.0.1:8084/purchase
```
业务逻辑：
1. 调用业务模块接口（business-tcc）
2. 扣减商品库存（stock-tcc）
3. 新增账户订单（order-tcc）
4. 账户余额扣减（account-tcc）

> （截图自 GitHub README.md）<br>
> ![在这里插入图片描述](images/04_SeataTCC/1f9b3f57014044aa9669d232b94af4be.png)

与 XA、AT 模式不同，TCC 模式需要自定义分支事务逻辑，所以 Demo 中对业务模块进行了拆解，新增了 action 接口及其实现类来编写相关逻辑。

以 Stock 库存模块为例：

![在这里插入图片描述](images/04_SeataTCC/d3c2cbd9a35446ca9a0ac624a8a82708.png)

![在这里插入图片描述](images/04_SeataTCC/773c8861766f4f84a184a5074ba5d7c9.png)

### 4、分析流程说明
整个下单操作是一个全局事务，各个模块分属于各个分支事务。下面对源码进行分析时只对其中一个分支操作进行说明，其余分支操作都是相同的，就不再展开说明。

分析分为 Commit 和 Rollback 两种流程。

**注：本文的流程分析主要还是依照控制台输出对源码进行分析，要点集中在 RM 与 TC 之间的操作，即事务分支的操作，对于开启全局事务等方法将不再详细说明，可参考 AT 模式。**


## Seata TCC 模式 Commit 调用流程
### 1、调用流程图
![在这里插入图片描述](images/04_SeataTCC/020e9e6d186c429793d783b370f126c6.png)

步骤前面有数字标识，也标记了主要的类，根据各模块控制台输出整理，详细见附录。

### 2、TCC 动作拦截器（事务分支注册）
`io.seata.spring.tcc.TccActionInterceptor#invoke`

![在这里插入图片描述](images/04_SeataTCC/51363cd1eabe48a794c8b4136443b322.png)

`io.seata.rm.tcc.interceptor.ActionInterceptorHandler#proceed`

![在这里插入图片描述](images/04_SeataTCC/7d668f059acd425e966a465c52fdf1c5.png)

`io.seata.rm.tcc.interceptor.ActionInterceptorHandler#doTccActionLogStore`

![在这里插入图片描述](images/04_SeataTCC/250fa7b1d19749858567fda5a46191ec.png)

`io.seata.rm.tcc.interceptor.ActionInterceptorHandler#fetchActionRequestContext`

![在这里插入图片描述](images/04_SeataTCC/1da6cf50feae4e34833494e69c21c5dc.png)

`io.seata.rm.tcc.interceptor.ActionInterceptorHandler#initBusinessContext`

![在这里插入图片描述](images/04_SeataTCC/fb11c49beb3d46a194cdf254abd071d9.png)

注册分支方法和 AT 模式一致，向 Server 端（TC）发送分支注册请求：

`io.seata.rm.AbstractResourceManager#branchRegister`

![在这里插入图片描述](images/04_SeataTCC/3871fbe5307b48cba761f9b1c5f48ae9.png)

分支注册完成，返回 `ActionInterceptorHandler` 拦截器处理器继续执行。

`io.seata.rm.tcc.interceptor.ActionInterceptorHandler#proceed`

![在这里插入图片描述](images/04_SeataTCC/52da87ed25804891a9713efb9a230c9e.png)

`io.seata.rm.tcc.TCCFenceHandler#prepareFence`

![在这里插入图片描述](images/04_SeataTCC/050dccb121ed4917894c4e47a0bf7d25.png)

`io.seata.rm.tcc.TCCFenceHandler#insertTCCFenceLog`

![在这里插入图片描述](images/04_SeataTCC/8657ad747bab4b5ea15843a60a57b718.png)

`io.seata.rm.tcc.store.db.TCCFenceStoreDataBaseDAO#insertTCCFenceDO`

![在这里插入图片描述](images/04_SeataTCC/eb205d9ad40642b6bf152f8bc06bd416.png)

### 3、事务分支提交
`io.seata.core.rpc.processor.client.RmBranchCommitProcessor`

![在这里插入图片描述](images/04_SeataTCC/cfde60b1f8d64b76afcc1c01474f2430.png)

`io.seata.rm.AbstractRMHandler#doBranchCommit`

![在这里插入图片描述](images/04_SeataTCC/6e03af2e7c244821b31e3b0407dc8c6f.png)

`io.seata.rm.tcc.TCCResourceManager#branchCommit`

![在这里插入图片描述](images/04_SeataTCC/8896c7c1058b44708e6d729dcace0c45.png)

`io.seata.rm.tcc.TCCFenceHandler#commitFence`

![在这里插入图片描述](images/04_SeataTCC/45fce43e1eab4bf2b252f91fadcb8035.png)

`io.seata.rm.tcc.TCCFenceHandler#updateStatusAndInvokeTargetMethod`

![在这里插入图片描述](images/04_SeataTCC/b05fbf89fd614fc4b8e4a9ebe7b06ec4.png)

`io.seata.rm.tcc.TCCFenceHandler#updateStatusAndInvokeTargetMethod`

![在这里插入图片描述](images/04_SeataTCC/9884bb07cbd1417fafae2285518085f5.png)

## Seata TCC 模式 Rollback 调用流程
### 1、调用流程图
![在这里插入图片描述](images/04_SeataTCC/37a8e054eeac4f3fae83367e2f122410.png)

步骤前面有数字标识，也标记了主要的类，根据各模块控制台输出整理，详细见附录。

### 2、事务分支回滚
执行三次业务之后，数据如下：

![在这里插入图片描述](images/04_SeataTCC/df7cd1bb9d9f4b99afde7a7cb1412a80.png)

第四次执行会抛出异常，并进行回滚。

`io.seata.core.rpc.processor.client.RmBranchRollbackProcessor`

![在这里插入图片描述](images/04_SeataTCC/de11490ec565440f9601c4e7579fbb13.png)

`io.seata.rm.AbstractRMHandler#handle`

![在这里插入图片描述](images/04_SeataTCC/5dbe32e626244d41b83b86dd8bac9ece.png)

`io.seata.rm.AbstractRMHandler#doBranchRollback`

![在这里插入图片描述](images/04_SeataTCC/b94996923f3d431aad12502d60a38097.png)

`io.seata.rm.tcc.TCCResourceManager#branchRollback`

![在这里插入图片描述](images/04_SeataTCC/3b00e11f5476421ea5fc169dd717d66d.png)

`io.seata.rm.tcc.TCCFenceHandler#rollbackFence`

![在这里插入图片描述](images/04_SeataTCC/1356a928abea47a6bf47a1b574972bff.png)

## 附录
### Commit 调用流程 Client 端控制台输出
#### 业务模块（business-tcc）
```console
2023-09-11 16:07:42.610  INFO 24792 --- [nio-8484-exec-1] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring DispatcherServlet 'dispatcherServlet'
2023-09-11 16:07:42.611  INFO 24792 --- [nio-8484-exec-1] o.s.web.servlet.DispatcherServlet        : Initializing Servlet 'dispatcherServlet'
2023-09-11 16:07:42.622  INFO 24792 --- [nio-8484-exec-1] o.s.web.servlet.DispatcherServlet        : Completed initialization in 11 ms
2023-09-11 16:07:42.676  INFO 24792 --- [nio-8484-exec-1] io.seata.tm.TransactionManagerHolder     : TransactionManager Singleton io.seata.tm.DefaultTransactionManager@3b4a9591
2023-09-11 16:07:42.713  INFO 24792 --- [nio-8484-exec-1] i.seata.tm.api.DefaultGlobalTransaction  : Begin new global transaction [192.168.2.126:8091:2252233925503221763]
2023-09-11 16:07:42.717  INFO 24792 --- [nio-8484-exec-1] s.zlyx.example.service.BusinessService   : New Transaction Begins: 192.168.2.126:8091:2252233925503221763
2023-09-11 16:07:43.590  INFO 24792 --- [nio-8484-exec-1] i.seata.tm.api.DefaultGlobalTransaction  : transaction 192.168.2.126:8091:2252233925503221763 will be commit
2023-09-11 16:07:43.854  INFO 24792 --- [nio-8484-exec-1] i.seata.tm.api.DefaultGlobalTransaction  : transaction end, xid = 192.168.2.126:8091:2252233925503221763
2023-09-11 16:07:43.854  INFO 24792 --- [nio-8484-exec-1] i.seata.tm.api.DefaultGlobalTransaction  : [192.168.2.126:8091:2252233925503221763] commit status: Committed
```
#### 库存模块（stock-tcc）
```console
2023-09-11 16:07:42.815  INFO 24000 --- [nio-8181-exec-1] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring DispatcherServlet 'dispatcherServlet'
2023-09-11 16:07:42.815  INFO 24000 --- [nio-8181-exec-1] o.s.web.servlet.DispatcherServlet        : Initializing Servlet 'dispatcherServlet'
2023-09-11 16:07:42.822  INFO 24000 --- [nio-8181-exec-1] o.s.web.servlet.DispatcherServlet        : Completed initialization in 7 ms
2023-09-11 16:07:42.960  INFO 24000 --- [nio-8181-exec-1] io.seata.rm.AbstractResourceManager      : branch register success, xid:192.168.2.126:8091:2252233925503221763, branchId:2252233925503221765, lockKeys:null
2023-09-11 16:07:42.977  INFO 24000 --- [nio-8181-exec-1] io.seata.rm.tcc.TCCFenceHandler          : TCC fence prepare result: true. xid: 192.168.2.126:8091:2252233925503221763, branchId: 2252233925503221765
2023-09-11 16:07:42.980  INFO 24000 --- [nio-8181-exec-1] s.z.e.tccaction.impl.StockActionImpl     : deduct stock balance in transaction: 192.168.2.126:8091:2252233925503221763
2023-09-11 16:07:42.992  INFO 24000 --- [nio-8181-exec-1] s.z.e.tccaction.impl.StockActionImpl     : stock after transaction: 70
2023-09-11 16:07:43.023  WARN 24000 --- [nio-8181-exec-1] c.a.c.seata.web.SeataHandlerInterceptor  : xid in change during RPC from 192.168.2.126:8091:2252233925503221763 to null
2023-09-11 16:07:43.641  INFO 24000 --- [h_RMROLE_1_1_16] i.s.c.r.p.c.RmBranchCommitProcessor      : rm client handle branch commit process:BranchCommitRequest{xid='192.168.2.126:8091:2252233925503221763', branchId=2252233925503221765, branchType=TCC, resourceId='tryDeduct', applicationData='{"actionContext":{"action-start-time":1694419662860,"useTCCFence":true,"sys::prepare":"tryDeduct","commodityCode":"C100000","count":30,"sys::rollback":"rollback","sys::commit":"commit","host-name":"192.168.2.126","actionName":"tryDeduct"}}'}
2023-09-11 16:07:43.645  INFO 24000 --- [h_RMROLE_1_1_16] io.seata.rm.AbstractRMHandler            : Branch committing: 192.168.2.126:8091:2252233925503221763 2252233925503221765 tryDeduct {"actionContext":{"action-start-time":1694419662860,"useTCCFence":true,"sys::prepare":"tryDeduct","commodityCode":"C100000","count":30,"sys::rollback":"rollback","sys::commit":"commit","host-name":"192.168.2.126","actionName":"tryDeduct"}}
2023-09-11 16:07:43.689  INFO 24000 --- [h_RMROLE_1_1_16] io.seata.rm.AbstractResourceManager      : TCC resource commit result : true, xid: 192.168.2.126:8091:2252233925503221763, branchId: 2252233925503221765, resourceId: tryDeduct
2023-09-11 16:07:43.690  INFO 24000 --- [h_RMROLE_1_1_16] io.seata.rm.AbstractRMHandler            : Branch commit result: PhaseTwo_Committed
```
#### 订单模块（order-tcc）
```console
2023-09-11 16:07:43.100  INFO 24332 --- [nio-8282-exec-1] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring DispatcherServlet 'dispatcherServlet'
2023-09-11 16:07:43.101  INFO 24332 --- [nio-8282-exec-1] o.s.web.servlet.DispatcherServlet        : Initializing Servlet 'dispatcherServlet'
2023-09-11 16:07:43.109  INFO 24332 --- [nio-8282-exec-1] o.s.web.servlet.DispatcherServlet        : Completed initialization in 8 ms
2023-09-11 16:07:43.253  INFO 24332 --- [nio-8282-exec-1] io.seata.rm.AbstractResourceManager      : branch register success, xid:192.168.2.126:8091:2252233925503221763, branchId:2252233925503221767, lockKeys:null
2023-09-11 16:07:43.267  INFO 24332 --- [nio-8282-exec-1] io.seata.rm.tcc.TCCFenceHandler          : TCC fence prepare result: true. xid: 192.168.2.126:8091:2252233925503221763, branchId: 2252233925503221767
2023-09-11 16:07:43.272  INFO 24332 --- [nio-8282-exec-1] s.z.e.tccaction.impl.OrderActionImpl     : create order in transaction: 192.168.2.126:8091:2252233925503221763
2023-09-11 16:07:43.591  WARN 24332 --- [nio-8282-exec-1] c.a.c.seata.web.SeataHandlerInterceptor  : xid in change during RPC from 192.168.2.126:8091:2252233925503221763 to null
2023-09-11 16:07:43.710  INFO 24332 --- [h_RMROLE_1_1_16] i.s.c.r.p.c.RmBranchCommitProcessor      : rm client handle branch commit process:BranchCommitRequest{xid='192.168.2.126:8091:2252233925503221763', branchId=2252233925503221767, branchType=TCC, resourceId='tryCreate', applicationData='{"actionContext":{"action-start-time":1694419663154,"useTCCFence":true,"sys::prepare":"tryCreate","commodityCode":"C100000","count":30,"sys::rollback":"rollback","sys::commit":"commit","host-name":"192.168.2.126","userId":"U100000","actionName":"tryCreate"}}'}
2023-09-11 16:07:43.712  INFO 24332 --- [h_RMROLE_1_1_16] io.seata.rm.AbstractRMHandler            : Branch committing: 192.168.2.126:8091:2252233925503221763 2252233925503221767 tryCreate {"actionContext":{"action-start-time":1694419663154,"useTCCFence":true,"sys::prepare":"tryCreate","commodityCode":"C100000","count":30,"sys::rollback":"rollback","sys::commit":"commit","host-name":"192.168.2.126","userId":"U100000","actionName":"tryCreate"}}
2023-09-11 16:07:43.749  INFO 24332 --- [h_RMROLE_1_1_16] io.seata.rm.AbstractResourceManager      : TCC resource commit result : true, xid: 192.168.2.126:8091:2252233925503221763, branchId: 2252233925503221767, resourceId: tryCreate
2023-09-11 16:07:43.751  INFO 24332 --- [h_RMROLE_1_1_16] io.seata.rm.AbstractRMHandler            : Branch commit result: PhaseTwo_Committed
```
#### 账户模块（account-tcc）
```console
2023-09-11 16:07:43.350  INFO 21608 --- [nio-8383-exec-2] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring DispatcherServlet 'dispatcherServlet'
2023-09-11 16:07:43.350  INFO 21608 --- [nio-8383-exec-2] o.s.web.servlet.DispatcherServlet        : Initializing Servlet 'dispatcherServlet'
2023-09-11 16:07:43.356  INFO 21608 --- [nio-8383-exec-2] o.s.web.servlet.DispatcherServlet        : Completed initialization in 6 ms
2023-09-11 16:07:43.485  INFO 21608 --- [nio-8383-exec-2] io.seata.rm.AbstractResourceManager      : branch register success, xid:192.168.2.126:8091:2252233925503221763, branchId:2252233925503221769, lockKeys:null
2023-09-11 16:07:43.498  INFO 21608 --- [nio-8383-exec-2] io.seata.rm.tcc.TCCFenceHandler          : TCC fence prepare result: true. xid: 192.168.2.126:8091:2252233925503221763, branchId: 2252233925503221769
2023-09-11 16:07:43.503  INFO 21608 --- [nio-8383-exec-2] s.z.e.tccaction.impl.AccountActionImpl   : reduce account balance in transaction: 192.168.2.126:8091:2252233925503221763
2023-09-11 16:07:43.513  INFO 21608 --- [nio-8383-exec-2] s.z.e.tccaction.impl.AccountActionImpl   : balance after transaction: 7000
2023-09-11 16:07:43.544  WARN 21608 --- [nio-8383-exec-2] c.a.c.seata.web.SeataHandlerInterceptor  : xid in change during RPC from 192.168.2.126:8091:2252233925503221763 to null
2023-09-11 16:07:43.772  INFO 21608 --- [h_RMROLE_1_1_16] i.s.c.r.p.c.RmBranchCommitProcessor      : rm client handle branch commit process:BranchCommitRequest{xid='192.168.2.126:8091:2252233925503221763', branchId=2252233925503221769, branchType=TCC, resourceId='tryReduce', applicationData='{"actionContext":{"action-start-time":1694419663400,"useTCCFence":true,"money":3000,"sys::prepare":"tryReduce","sys::rollback":"rollback","sys::commit":"commit","host-name":"192.168.2.126","userId":"U100000","actionName":"tryReduce"}}'}
2023-09-11 16:07:43.773  INFO 21608 --- [h_RMROLE_1_1_16] io.seata.rm.AbstractRMHandler            : Branch committing: 192.168.2.126:8091:2252233925503221763 2252233925503221769 tryReduce {"actionContext":{"action-start-time":1694419663400,"useTCCFence":true,"money":3000,"sys::prepare":"tryReduce","sys::rollback":"rollback","sys::commit":"commit","host-name":"192.168.2.126","userId":"U100000","actionName":"tryReduce"}}
2023-09-11 16:07:43.833  INFO 21608 --- [h_RMROLE_1_1_16] io.seata.rm.AbstractResourceManager      : TCC resource commit result : true, xid: 192.168.2.126:8091:2252233925503221763, branchId: 2252233925503221769, resourceId: tryReduce
2023-09-11 16:07:43.834  INFO 21608 --- [h_RMROLE_1_1_16] io.seata.rm.AbstractRMHandler            : Branch commit result: PhaseTwo_Committed
```
### Commit 调用流程 Server 端控制台输出
```console
16:07:42.684  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : receive msg[single]: GlobalBeginRequest{transactionName='purchase(java.lang.String, java.lang.String, int, boolean)', timeout=60000}, clientIp: 192.168.2.126, vgroup: my_test_tx_group
16:07:42.706  INFO --- [rverHandlerThread_1_4_500] [coordinator.DefaultCoordinator] [       doGlobalBegin]  [192.168.2.126:8091:2252233925503221763] : Begin new global transaction applicationId: business-tcc,transactionServiceGroup: my_test_tx_group, transactionName: purchase(java.lang.String, java.lang.String, int, boolean),timeout:60000,xid:192.168.2.126:8091:2252233925503221763
16:07:42.708  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : result msg[single]: GlobalBeginResponse{xid='192.168.2.126:8091:2252233925503221763', extraData='null', resultCode=Success, msg='null'}, clientIp: 192.168.2.126, vgroup: my_test_tx_group
16:07:42.924  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : receive msg[merged]: BranchRegisterRequest{xid='192.168.2.126:8091:2252233925503221763', branchType=TCC, resourceId='tryDeduct', lockKey='null', applicationData='{"actionContext":{"action-start-time":1694419662860,"useTCCFence":true,"sys::prepare":"tryDeduct","commodityCode":"C100000","count":30,"sys::rollback":"rollback","sys::commit":"commit","host-name":"192.168.2.126","actionName":"tryDeduct"}}'}, clientIp: 192.168.2.126, vgroup: my_test_tx_group
16:07:42.954  INFO --- [nPool.commonPool-worker-2] [erver.coordinator.AbstractCore] [bda$branchRegister$0]  [192.168.2.126:8091:2252233925503221763] : Register branch successfully, xid = 192.168.2.126:8091:2252233925503221763, branchId = 2252233925503221765, resourceId = tryDeduct ,lockKeys = null
16:07:42.955  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : result msg[merged]: BranchRegisterResponse{branchId=2252233925503221765, resultCode=Success, msg='null'}, clientIp: 192.168.2.126, vgroup: my_test_tx_group
16:07:43.208  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : receive msg[merged]: BranchRegisterRequest{xid='192.168.2.126:8091:2252233925503221763', branchType=TCC, resourceId='tryCreate', lockKey='null', applicationData='{"actionContext":{"action-start-time":1694419663154,"useTCCFence":true,"sys::prepare":"tryCreate","commodityCode":"C100000","count":30,"sys::rollback":"rollback","sys::commit":"commit","host-name":"192.168.2.126","userId":"U100000","actionName":"tryCreate"}}'}, clientIp: 192.168.2.126, vgroup: my_test_tx_group
16:07:43.227  INFO --- [nPool.commonPool-worker-2] [erver.coordinator.AbstractCore] [bda$branchRegister$0]  [192.168.2.126:8091:2252233925503221763] : Register branch successfully, xid = 192.168.2.126:8091:2252233925503221763, branchId = 2252233925503221767, resourceId = tryCreate ,lockKeys = null
16:07:43.227  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : result msg[merged]: BranchRegisterResponse{branchId=2252233925503221767, resultCode=Success, msg='null'}, clientIp: 192.168.2.126, vgroup: my_test_tx_group
16:07:43.460  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : receive msg[merged]: BranchRegisterRequest{xid='192.168.2.126:8091:2252233925503221763', branchType=TCC, resourceId='tryReduce', lockKey='null', applicationData='{"actionContext":{"action-start-time":1694419663400,"useTCCFence":true,"money":3000,"sys::prepare":"tryReduce","sys::rollback":"rollback","sys::commit":"commit","host-name":"192.168.2.126","userId":"U100000","actionName":"tryReduce"}}'}, clientIp: 192.168.2.126, vgroup: my_test_tx_group
16:07:43.479  INFO --- [nPool.commonPool-worker-2] [erver.coordinator.AbstractCore] [bda$branchRegister$0]  [192.168.2.126:8091:2252233925503221763] : Register branch successfully, xid = 192.168.2.126:8091:2252233925503221763, branchId = 2252233925503221769, resourceId = tryReduce ,lockKeys = null
16:07:43.480  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : result msg[merged]: BranchRegisterResponse{branchId=2252233925503221769, resultCode=Success, msg='null'}, clientIp: 192.168.2.126, vgroup: my_test_tx_group
16:07:43.594  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : receive msg[single]: GlobalCommitRequest{xid='192.168.2.126:8091:2252233925503221763', extraData='null'}, clientIp: 192.168.2.126, vgroup: my_test_tx_group
16:07:43.694  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : receive msg[single]: BranchCommitResponse{xid='192.168.2.126:8091:2252233925503221763', branchId=2252233925503221765, branchStatus=PhaseTwo_Committed, resultCode=Success, msg='null'}, clientIp: 192.168.2.126, vgroup: my_test_tx_group
16:07:43.705  INFO --- [rverHandlerThread_1_8_500] [server.coordinator.DefaultCore] [bda$doGlobalCommit$1]  [192.168.2.126:8091:2252233925503221763] : Commit branch transaction successfully, xid = 192.168.2.126:8091:2252233925503221763 branchId = 2252233925503221765
16:07:43.755  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : receive msg[single]: BranchCommitResponse{xid='192.168.2.126:8091:2252233925503221763', branchId=2252233925503221767, branchStatus=PhaseTwo_Committed, resultCode=Success, msg='null'}, clientIp: 192.168.2.126, vgroup: my_test_tx_group
16:07:43.766  INFO --- [rverHandlerThread_1_8_500] [server.coordinator.DefaultCore] [bda$doGlobalCommit$1]  [192.168.2.126:8091:2252233925503221763] : Commit branch transaction successfully, xid = 192.168.2.126:8091:2252233925503221763 branchId = 2252233925503221767
16:07:43.837  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : receive msg[single]: BranchCommitResponse{xid='192.168.2.126:8091:2252233925503221763', branchId=2252233925503221769, branchStatus=PhaseTwo_Committed, resultCode=Success, msg='null'}, clientIp: 192.168.2.126, vgroup: my_test_tx_group
16:07:43.850  INFO --- [rverHandlerThread_1_8_500] [server.coordinator.DefaultCore] [bda$doGlobalCommit$1]  [192.168.2.126:8091:2252233925503221763] : Commit branch transaction successfully, xid = 192.168.2.126:8091:2252233925503221763 branchId = 2252233925503221769
16:07:43.850  INFO --- [rverHandlerThread_1_8_500] [server.coordinator.DefaultCore] [      doGlobalCommit]  [192.168.2.126:8091:2252233925503221763] : Committing global transaction is successfully done, xid = 192.168.2.126:8091:2252233925503221763.
16:07:43.851  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : result msg[single]: GlobalCommitResponse{globalStatus=Committed, resultCode=Success, msg='null'}, clientIp: 192.168.2.126, vgroup: my_test_tx_group
```
#### 按时序整理的完整流程输出（包含Client 端、Server 端）
```console
16:07:42.684  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : receive msg[single]: GlobalBeginRequest{transactionName='purchase(java.lang.String, java.lang.String, int, boolean)', timeout=60000}, clientIp: 192.168.2.126, vgroup: my_test_tx_group
16:07:42.706  INFO --- [rverHandlerThread_1_4_500] [coordinator.DefaultCoordinator] [       doGlobalBegin]  [192.168.2.126:8091:2252233925503221763] : Begin new global transaction applicationId: business-tcc,transactionServiceGroup: my_test_tx_group, transactionName: purchase(java.lang.String, java.lang.String, int, boolean),timeout:60000,xid:192.168.2.126:8091:2252233925503221763
16:07:42.708  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : result msg[single]: GlobalBeginResponse{xid='192.168.2.126:8091:2252233925503221763', extraData='null', resultCode=Success, msg='null'}, clientIp: 192.168.2.126, vgroup: my_test_tx_group

16:07:42.713  INFO 24792 --- [nio-8484-exec-1] i.seata.tm.api.DefaultGlobalTransaction  : Begin new global transaction [192.168.2.126:8091:2252233925503221763]
16:07:42.717  INFO 24792 --- [nio-8484-exec-1] s.zlyx.example.service.BusinessService   : New Transaction Begins: 192.168.2.126:8091:2252233925503221763

16:07:42.924  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : receive msg[merged]: BranchRegisterRequest{xid='192.168.2.126:8091:2252233925503221763', branchType=TCC, resourceId='tryDeduct', lockKey='null', applicationData='{"actionContext":{"action-start-time":1694419662860,"useTCCFence":true,"sys::prepare":"tryDeduct","commodityCode":"C100000","count":30,"sys::rollback":"rollback","sys::commit":"commit","host-name":"192.168.2.126","actionName":"tryDeduct"}}'}, clientIp: 192.168.2.126, vgroup: my_test_tx_group
16:07:42.954  INFO --- [nPool.commonPool-worker-2] [erver.coordinator.AbstractCore] [bda$branchRegister$0]  [192.168.2.126:8091:2252233925503221763] : Register branch successfully, xid = 192.168.2.126:8091:2252233925503221763, branchId = 2252233925503221765, resourceId = tryDeduct ,lockKeys = null
16:07:42.955  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : result msg[merged]: BranchRegisterResponse{branchId=2252233925503221765, resultCode=Success, msg='null'}, clientIp: 192.168.2.126, vgroup: my_test_tx_group

16:07:42.960  INFO 24000 --- [nio-8181-exec-1] io.seata.rm.AbstractResourceManager      : branch register success, xid:192.168.2.126:8091:2252233925503221763, branchId:2252233925503221765, lockKeys:null
16:07:42.977  INFO 24000 --- [nio-8181-exec-1] io.seata.rm.tcc.TCCFenceHandler          : TCC fence prepare result: true. xid: 192.168.2.126:8091:2252233925503221763, branchId: 2252233925503221765
16:07:42.980  INFO 24000 --- [nio-8181-exec-1] s.z.e.tccaction.impl.StockActionImpl     : deduct stock balance in transaction: 192.168.2.126:8091:2252233925503221763
16:07:42.992  INFO 24000 --- [nio-8181-exec-1] s.z.e.tccaction.impl.StockActionImpl     : stock after transaction: 70

16:07:43.208  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : receive msg[merged]: BranchRegisterRequest{xid='192.168.2.126:8091:2252233925503221763', branchType=TCC, resourceId='tryCreate', lockKey='null', applicationData='{"actionContext":{"action-start-time":1694419663154,"useTCCFence":true,"sys::prepare":"tryCreate","commodityCode":"C100000","count":30,"sys::rollback":"rollback","sys::commit":"commit","host-name":"192.168.2.126","userId":"U100000","actionName":"tryCreate"}}'}, clientIp: 192.168.2.126, vgroup: my_test_tx_group
16:07:43.227  INFO --- [nPool.commonPool-worker-2] [erver.coordinator.AbstractCore] [bda$branchRegister$0]  [192.168.2.126:8091:2252233925503221763] : Register branch successfully, xid = 192.168.2.126:8091:2252233925503221763, branchId = 2252233925503221767, resourceId = tryCreate ,lockKeys = null
16:07:43.227  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : result msg[merged]: BranchRegisterResponse{branchId=2252233925503221767, resultCode=Success, msg='null'}, clientIp: 192.168.2.126, vgroup: my_test_tx_group

16:07:43.253  INFO 24332 --- [nio-8282-exec-1] io.seata.rm.AbstractResourceManager      : branch register success, xid:192.168.2.126:8091:2252233925503221763, branchId:2252233925503221767, lockKeys:null
16:07:43.267  INFO 24332 --- [nio-8282-exec-1] io.seata.rm.tcc.TCCFenceHandler          : TCC fence prepare result: true. xid: 192.168.2.126:8091:2252233925503221763, branchId: 2252233925503221767
16:07:43.272  INFO 24332 --- [nio-8282-exec-1] s.z.e.tccaction.impl.OrderActionImpl     : create order in transaction: 192.168.2.126:8091:2252233925503221763

16:07:43.460  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : receive msg[merged]: BranchRegisterRequest{xid='192.168.2.126:8091:2252233925503221763', branchType=TCC, resourceId='tryReduce', lockKey='null', applicationData='{"actionContext":{"action-start-time":1694419663400,"useTCCFence":true,"money":3000,"sys::prepare":"tryReduce","sys::rollback":"rollback","sys::commit":"commit","host-name":"192.168.2.126","userId":"U100000","actionName":"tryReduce"}}'}, clientIp: 192.168.2.126, vgroup: my_test_tx_group
16:07:43.479  INFO --- [nPool.commonPool-worker-2] [erver.coordinator.AbstractCore] [bda$branchRegister$0]  [192.168.2.126:8091:2252233925503221763] : Register branch successfully, xid = 192.168.2.126:8091:2252233925503221763, branchId = 2252233925503221769, resourceId = tryReduce ,lockKeys = null
16:07:43.480  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : result msg[merged]: BranchRegisterResponse{branchId=2252233925503221769, resultCode=Success, msg='null'}, clientIp: 192.168.2.126, vgroup: my_test_tx_group

16:07:43.485  INFO 21608 --- [nio-8383-exec-2] io.seata.rm.AbstractResourceManager      : branch register success, xid:192.168.2.126:8091:2252233925503221763, branchId:2252233925503221769, lockKeys:null
16:07:43.498  INFO 21608 --- [nio-8383-exec-2] io.seata.rm.tcc.TCCFenceHandler          : TCC fence prepare result: true. xid: 192.168.2.126:8091:2252233925503221763, branchId: 2252233925503221769
16:07:43.503  INFO 21608 --- [nio-8383-exec-2] s.z.e.tccaction.impl.AccountActionImpl   : reduce account balance in transaction: 192.168.2.126:8091:2252233925503221763
16:07:43.513  INFO 21608 --- [nio-8383-exec-2] s.z.e.tccaction.impl.AccountActionImpl   : balance after transaction: 7000

16:07:43.590  INFO 24792 --- [nio-8484-exec-1] i.seata.tm.api.DefaultGlobalTransaction  : transaction 192.168.2.126:8091:2252233925503221763 will be commit

16:07:43.594  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : receive msg[single]: GlobalCommitRequest{xid='192.168.2.126:8091:2252233925503221763', extraData='null'}, clientIp: 192.168.2.126, vgroup: my_test_tx_group

16:07:43.641  INFO 24000 --- [h_RMROLE_1_1_16] i.s.c.r.p.c.RmBranchCommitProcessor      : rm client handle branch commit process:BranchCommitRequest{xid='192.168.2.126:8091:2252233925503221763', branchId=2252233925503221765, branchType=TCC, resourceId='tryDeduct', applicationData='{"actionContext":{"action-start-time":1694419662860,"useTCCFence":true,"sys::prepare":"tryDeduct","commodityCode":"C100000","count":30,"sys::rollback":"rollback","sys::commit":"commit","host-name":"192.168.2.126","actionName":"tryDeduct"}}'}
16:07:43.645  INFO 24000 --- [h_RMROLE_1_1_16] io.seata.rm.AbstractRMHandler            : Branch committing: 192.168.2.126:8091:2252233925503221763 2252233925503221765 tryDeduct {"actionContext":{"action-start-time":1694419662860,"useTCCFence":true,"sys::prepare":"tryDeduct","commodityCode":"C100000","count":30,"sys::rollback":"rollback","sys::commit":"commit","host-name":"192.168.2.126","actionName":"tryDeduct"}}
16:07:43.689  INFO 24000 --- [h_RMROLE_1_1_16] io.seata.rm.AbstractResourceManager      : TCC resource commit result : true, xid: 192.168.2.126:8091:2252233925503221763, branchId: 2252233925503221765, resourceId: tryDeduct
16:07:43.690  INFO 24000 --- [h_RMROLE_1_1_16] io.seata.rm.AbstractRMHandler            : Branch commit result: PhaseTwo_Committed

16:07:43.694  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : receive msg[single]: BranchCommitResponse{xid='192.168.2.126:8091:2252233925503221763', branchId=2252233925503221765, branchStatus=PhaseTwo_Committed, resultCode=Success, msg='null'}, clientIp: 192.168.2.126, vgroup: my_test_tx_group
16:07:43.705  INFO --- [rverHandlerThread_1_8_500] [server.coordinator.DefaultCore] [bda$doGlobalCommit$1]  [192.168.2.126:8091:2252233925503221763] : Commit branch transaction successfully, xid = 192.168.2.126:8091:2252233925503221763 branchId = 2252233925503221765

16:07:43.710  INFO 24332 --- [h_RMROLE_1_1_16] i.s.c.r.p.c.RmBranchCommitProcessor      : rm client handle branch commit process:BranchCommitRequest{xid='192.168.2.126:8091:2252233925503221763', branchId=2252233925503221767, branchType=TCC, resourceId='tryCreate', applicationData='{"actionContext":{"action-start-time":1694419663154,"useTCCFence":true,"sys::prepare":"tryCreate","commodityCode":"C100000","count":30,"sys::rollback":"rollback","sys::commit":"commit","host-name":"192.168.2.126","userId":"U100000","actionName":"tryCreate"}}'}
16:07:43.712  INFO 24332 --- [h_RMROLE_1_1_16] io.seata.rm.AbstractRMHandler            : Branch committing: 192.168.2.126:8091:2252233925503221763 2252233925503221767 tryCreate {"actionContext":{"action-start-time":1694419663154,"useTCCFence":true,"sys::prepare":"tryCreate","commodityCode":"C100000","count":30,"sys::rollback":"rollback","sys::commit":"commit","host-name":"192.168.2.126","userId":"U100000","actionName":"tryCreate"}}
16:07:43.749  INFO 24332 --- [h_RMROLE_1_1_16] io.seata.rm.AbstractResourceManager      : TCC resource commit result : true, xid: 192.168.2.126:8091:2252233925503221763, branchId: 2252233925503221767, resourceId: tryCreate
16:07:43.751  INFO 24332 --- [h_RMROLE_1_1_16] io.seata.rm.AbstractRMHandler            : Branch commit result: PhaseTwo_Committed

16:07:43.755  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : receive msg[single]: BranchCommitResponse{xid='192.168.2.126:8091:2252233925503221763', branchId=2252233925503221767, branchStatus=PhaseTwo_Committed, resultCode=Success, msg='null'}, clientIp: 192.168.2.126, vgroup: my_test_tx_group
16:07:43.766  INFO --- [rverHandlerThread_1_8_500] [server.coordinator.DefaultCore] [bda$doGlobalCommit$1]  [192.168.2.126:8091:2252233925503221763] : Commit branch transaction successfully, xid = 192.168.2.126:8091:2252233925503221763 branchId = 2252233925503221767

16:07:43.772  INFO 21608 --- [h_RMROLE_1_1_16] i.s.c.r.p.c.RmBranchCommitProcessor      : rm client handle branch commit process:BranchCommitRequest{xid='192.168.2.126:8091:2252233925503221763', branchId=2252233925503221769, branchType=TCC, resourceId='tryReduce', applicationData='{"actionContext":{"action-start-time":1694419663400,"useTCCFence":true,"money":3000,"sys::prepare":"tryReduce","sys::rollback":"rollback","sys::commit":"commit","host-name":"192.168.2.126","userId":"U100000","actionName":"tryReduce"}}'}
16:07:43.773  INFO 21608 --- [h_RMROLE_1_1_16] io.seata.rm.AbstractRMHandler            : Branch committing: 192.168.2.126:8091:2252233925503221763 2252233925503221769 tryReduce {"actionContext":{"action-start-time":1694419663400,"useTCCFence":true,"money":3000,"sys::prepare":"tryReduce","sys::rollback":"rollback","sys::commit":"commit","host-name":"192.168.2.126","userId":"U100000","actionName":"tryReduce"}}
16:07:43.833  INFO 21608 --- [h_RMROLE_1_1_16] io.seata.rm.AbstractResourceManager      : TCC resource commit result : true, xid: 192.168.2.126:8091:2252233925503221763, branchId: 2252233925503221769, resourceId: tryReduce
16:07:43.834  INFO 21608 --- [h_RMROLE_1_1_16] io.seata.rm.AbstractRMHandler            : Branch commit result: PhaseTwo_Committed

16:07:43.837  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : receive msg[single]: BranchCommitResponse{xid='192.168.2.126:8091:2252233925503221763', branchId=2252233925503221769, branchStatus=PhaseTwo_Committed, resultCode=Success, msg='null'}, clientIp: 192.168.2.126, vgroup: my_test_tx_group
16:07:43.850  INFO --- [rverHandlerThread_1_8_500] [server.coordinator.DefaultCore] [bda$doGlobalCommit$1]  [192.168.2.126:8091:2252233925503221763] : Commit branch transaction successfully, xid = 192.168.2.126:8091:2252233925503221763 branchId = 2252233925503221769

16:07:43.850  INFO --- [rverHandlerThread_1_8_500] [server.coordinator.DefaultCore] [      doGlobalCommit]  [192.168.2.126:8091:2252233925503221763] : Committing global transaction is successfully done, xid = 192.168.2.126:8091:2252233925503221763.
16:07:43.851  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : result msg[single]: GlobalCommitResponse{globalStatus=Committed, resultCode=Success, msg='null'}, clientIp: 192.168.2.126, vgroup: my_test_tx_group

16:07:43.854  INFO 24792 --- [nio-8484-exec-1] i.seata.tm.api.DefaultGlobalTransaction  : transaction end, xid = 192.168.2.126:8091:2252233925503221763
16:07:43.854  INFO 24792 --- [nio-8484-exec-1] i.seata.tm.api.DefaultGlobalTransaction  : [192.168.2.126:8091:2252233925503221763] commit status: Committed
```
### Rollback 调用流程 Client 端控制台输出
#### 业务模块（business-tcc）
```console
2023-09-11 17:47:06.443  INFO 24792 --- [nio-8484-exec-7] i.seata.tm.api.DefaultGlobalTransaction  : Begin new global transaction [192.168.2.126:8091:2252233925503221807]
2023-09-11 17:47:06.443  INFO 24792 --- [nio-8484-exec-7] s.zlyx.example.service.BusinessService   : New Transaction Begins: 192.168.2.126:8091:2252233925503221807
2023-09-11 17:47:06.878  INFO 24792 --- [nio-8484-exec-7] i.seata.tm.api.DefaultGlobalTransaction  : transaction 192.168.2.126:8091:2252233925503221807 will be rollback
2023-09-11 17:47:07.105  INFO 24792 --- [nio-8484-exec-7] i.seata.tm.api.DefaultGlobalTransaction  : transaction end, xid = 192.168.2.126:8091:2252233925503221807
2023-09-11 17:47:07.105  INFO 24792 --- [nio-8484-exec-7] i.seata.tm.api.DefaultGlobalTransaction  : [192.168.2.126:8091:2252233925503221807] rollback status: Rollbacked
```
#### 库存模块（stock-tcc）
```console
2023-09-11 17:47:06.676  INFO 22560 --- [nio-8181-exec-1] io.seata.rm.AbstractResourceManager      : branch register success, xid:192.168.2.126:8091:2252233925503221807, branchId:2252233925503221809, lockKeys:null
2023-09-11 17:47:06.690  INFO 22560 --- [nio-8181-exec-1] io.seata.rm.tcc.TCCFenceHandler          : TCC fence prepare result: true. xid: 192.168.2.126:8091:2252233925503221807, branchId: 2252233925503221809
2023-09-11 17:47:06.694  INFO 22560 --- [nio-8181-exec-1] s.z.e.tccaction.impl.StockActionImpl     : deduct stock balance in transaction: 192.168.2.126:8091:2252233925503221807
2023-09-11 17:47:06.739  WARN 22560 --- [nio-8181-exec-1] c.a.c.seata.web.SeataHandlerInterceptor  : xid in change during RPC from 192.168.2.126:8091:2252233925503221807 to null
2023-09-11 17:47:07.035  INFO 22560 --- [h_RMROLE_1_1_16] i.s.c.r.p.c.RmBranchRollbackProcessor    : rm handle branch rollback process:BranchRollbackRequest{xid='192.168.2.126:8091:2252233925503221807', branchId=2252233925503221809, branchType=TCC, resourceId='tryDeduct', applicationData='{"actionContext":{"action-start-time":1694425626577,"useTCCFence":true,"sys::prepare":"tryDeduct","commodityCode":"C100000","count":30,"sys::rollback":"rollback","sys::commit":"commit","host-name":"192.168.2.126","actionName":"tryDeduct"}}'}
2023-09-11 17:47:07.036  INFO 22560 --- [h_RMROLE_1_1_16] io.seata.rm.AbstractRMHandler            : Branch Rollbacking: 192.168.2.126:8091:2252233925503221807 2252233925503221809 tryDeduct
2023-09-11 17:47:07.090  INFO 22560 --- [h_RMROLE_1_1_16] io.seata.rm.AbstractResourceManager      : TCC resource rollback result : true, xid: 192.168.2.126:8091:2252233925503221807, branchId: 2252233925503221809, resourceId: tryDeduct
2023-09-11 17:47:07.091  INFO 22560 --- [h_RMROLE_1_1_16] io.seata.rm.AbstractRMHandler            : Branch Rollbacked result: PhaseTwo_Rollbacked
```
#### 订单模块（order-tcc）
```console
2023-09-11 17:47:06.771  INFO 24332 --- [nio-8282-exec-6] io.seata.rm.AbstractResourceManager      : branch register success, xid:192.168.2.126:8091:2252233925503221807, branchId:2252233925503221811, lockKeys:null
2023-09-11 17:47:06.779  INFO 24332 --- [nio-8282-exec-6] io.seata.rm.tcc.TCCFenceHandler          : TCC fence prepare result: true. xid: 192.168.2.126:8091:2252233925503221807, branchId: 2252233925503221811
2023-09-11 17:47:06.779  INFO 24332 --- [nio-8282-exec-6] s.z.e.tccaction.impl.OrderActionImpl     : create order in transaction: 192.168.2.126:8091:2252233925503221807
java.lang.RuntimeException: Failed to call Account Service. 
	at space.zlyx.example.tccaction.impl.OrderActionImpl.tryCreate(OrderActionImpl.java:48)
	at space.zlyx.example.tccaction.impl.OrderActionImpl$$FastClassBySpringCGLIB$$180ddab3.invoke(<generated>)
	at org.springframework.cglib.proxy.MethodProxy.invoke(MethodProxy.java:218)
	at org.springframework.aop.framework.CglibAopProxy$CglibMethodInvocation.invokeJoinpoint(CglibAopProxy.java:771)
	at org.springframework.aop.framework.ReflectiveMethodInvocation.proceed(ReflectiveMethodInvocation.java:163)
	at org.springframework.aop.framework.CglibAopProxy$CglibMethodInvocation.proceed(CglibAopProxy.java:749)
	at io.seata.rm.tcc.TCCFenceHandler.lambda$prepareFence$0(TCCFenceHandler.java:115)
	at org.springframework.transaction.support.TransactionTemplate.execute(TransactionTemplate.java:140)
	at io.seata.rm.tcc.TCCFenceHandler.prepareFence(TCCFenceHandler.java:109)
	at io.seata.rm.tcc.interceptor.ActionInterceptorHandler.proceed(ActionInterceptorHandler.java:94)
	at io.seata.spring.tcc.TccActionInterceptor.invoke(TccActionInterceptor.java:100)
	at org.springframework.aop.framework.ReflectiveMethodInvocation.proceed(ReflectiveMethodInvocation.java:186)
	at org.springframework.aop.framework.CglibAopProxy$CglibMethodInvocation.proceed(CglibAopProxy.java:749)
	at org.springframework.aop.framework.CglibAopProxy$DynamicAdvisedInterceptor.intercept(CglibAopProxy.java:691)
	at space.zlyx.example.tccaction.impl.OrderActionImpl$$EnhancerBySpringCGLIB$$a5c94c21.tryCreate(<generated>)
	at space.zlyx.example.service.OrderService.create(OrderService.java:21)
	at space.zlyx.example.controller.OrderController.create(OrderController.java:22)
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
2023-09-11 17:47:06.877  WARN 24332 --- [nio-8282-exec-6] c.a.c.seata.web.SeataHandlerInterceptor  : xid in change during RPC from 192.168.2.126:8091:2252233925503221807 to null
2023-09-11 17:47:06.980  INFO 24332 --- [h_RMROLE_1_5_16] i.s.c.r.p.c.RmBranchRollbackProcessor    : rm handle branch rollback process:BranchRollbackRequest{xid='192.168.2.126:8091:2252233925503221807', branchId=2252233925503221811, branchType=TCC, resourceId='tryCreate', applicationData='{"actionContext":{"action-start-time":1694425626742,"useTCCFence":true,"sys::prepare":"tryCreate","commodityCode":"C100000","count":30,"sys::rollback":"rollback","sys::commit":"commit","host-name":"192.168.2.126","userId":"U100000","actionName":"tryCreate"}}'}
2023-09-11 17:47:06.982  INFO 24332 --- [h_RMROLE_1_5_16] io.seata.rm.AbstractRMHandler            : Branch Rollbacking: 192.168.2.126:8091:2252233925503221807 2252233925503221811 tryCreate
2023-09-11 17:47:07.006  INFO 24332 --- [h_RMROLE_1_5_16] io.seata.rm.tcc.TCCFenceHandler          : Insert tcc fence record result: true. xid: 192.168.2.126:8091:2252233925503221807, branchId: 2252233925503221811
2023-09-11 17:47:07.017  INFO 24332 --- [h_RMROLE_1_5_16] io.seata.rm.AbstractResourceManager      : TCC resource rollback result : true, xid: 192.168.2.126:8091:2252233925503221807, branchId: 2252233925503221811, resourceId: tryCreate
2023-09-11 17:47:07.017  INFO 24332 --- [h_RMROLE_1_5_16] io.seata.rm.AbstractRMHandler            : Branch Rollbacked result: PhaseTwo_Rollbacked
```
#### 账户模块（account-tcc）
```console
2023-09-11 17:47:06.805  INFO 21608 --- [nio-8383-exec-6] io.seata.rm.AbstractResourceManager      : branch register success, xid:192.168.2.126:8091:2252233925503221807, branchId:2252233925503221813, lockKeys:null
2023-09-11 17:47:06.829  INFO 21608 --- [nio-8383-exec-6] io.seata.rm.tcc.TCCFenceHandler          : TCC fence prepare result: true. xid: 192.168.2.126:8091:2252233925503221807, branchId: 2252233925503221813
2023-09-11 17:47:06.829  INFO 21608 --- [nio-8383-exec-6] s.z.e.tccaction.impl.AccountActionImpl   : reduce account balance in transaction: 192.168.2.126:8091:2252233925503221807
2023-09-11 17:47:06.833  INFO 21608 --- [nio-8383-exec-6] s.z.e.tccaction.impl.AccountActionImpl   : balance after transaction: -2000
java.lang.RuntimeException: Not Enough Money ...
	at space.zlyx.example.tccaction.impl.AccountActionImpl.tryReduce(AccountActionImpl.java:36)
	at space.zlyx.example.tccaction.impl.AccountActionImpl$$FastClassBySpringCGLIB$$486c7b12.invoke(<generated>)
	at org.springframework.cglib.proxy.MethodProxy.invoke(MethodProxy.java:218)
	at org.springframework.aop.framework.CglibAopProxy$CglibMethodInvocation.invokeJoinpoint(CglibAopProxy.java:771)
	at org.springframework.aop.framework.ReflectiveMethodInvocation.proceed(ReflectiveMethodInvocation.java:163)
	at org.springframework.aop.framework.CglibAopProxy$CglibMethodInvocation.proceed(CglibAopProxy.java:749)
	at io.seata.rm.tcc.TCCFenceHandler.lambda$prepareFence$0(TCCFenceHandler.java:115)
	at org.springframework.transaction.support.TransactionTemplate.execute(TransactionTemplate.java:140)
	at io.seata.rm.tcc.TCCFenceHandler.prepareFence(TCCFenceHandler.java:109)
	at io.seata.rm.tcc.interceptor.ActionInterceptorHandler.proceed(ActionInterceptorHandler.java:94)
	at io.seata.spring.tcc.TccActionInterceptor.invoke(TccActionInterceptor.java:100)
	at org.springframework.aop.framework.ReflectiveMethodInvocation.proceed(ReflectiveMethodInvocation.java:186)
	at org.springframework.aop.framework.CglibAopProxy$CglibMethodInvocation.proceed(CglibAopProxy.java:749)
	at org.springframework.aop.framework.CglibAopProxy$DynamicAdvisedInterceptor.intercept(CglibAopProxy.java:691)
	at space.zlyx.example.tccaction.impl.AccountActionImpl$$EnhancerBySpringCGLIB$$86f696ad.tryReduce(<generated>)
	at space.zlyx.example.service.AccountService.reduce(AccountService.java:23)
	at space.zlyx.example.controller.AccountController.reduce(AccountController.java:22)
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
2023-09-11 17:47:06.855  WARN 21608 --- [nio-8383-exec-6] c.a.c.seata.web.SeataHandlerInterceptor  : xid in change during RPC from 192.168.2.126:8091:2252233925503221807 to null
2023-09-11 17:47:06.922  INFO 21608 --- [h_RMROLE_1_5_16] i.s.c.r.p.c.RmBranchRollbackProcessor    : rm handle branch rollback process:BranchRollbackRequest{xid='192.168.2.126:8091:2252233925503221807', branchId=2252233925503221813, branchType=TCC, resourceId='tryReduce', applicationData='{"actionContext":{"action-start-time":1694425626784,"useTCCFence":true,"money":3000,"sys::prepare":"tryReduce","sys::rollback":"rollback","sys::commit":"commit","host-name":"192.168.2.126","userId":"U100000","actionName":"tryReduce"}}'}
2023-09-11 17:47:06.924  INFO 21608 --- [h_RMROLE_1_5_16] io.seata.rm.AbstractRMHandler            : Branch Rollbacking: 192.168.2.126:8091:2252233925503221807 2252233925503221813 tryReduce
2023-09-11 17:47:06.937  INFO 21608 --- [h_RMROLE_1_5_16] io.seata.rm.tcc.TCCFenceHandler          : Insert tcc fence record result: true. xid: 192.168.2.126:8091:2252233925503221807, branchId: 2252233925503221813
2023-09-11 17:47:06.960  INFO 21608 --- [h_RMROLE_1_5_16] io.seata.rm.AbstractResourceManager      : TCC resource rollback result : true, xid: 192.168.2.126:8091:2252233925503221807, branchId: 2252233925503221813, resourceId: tryReduce
2023-09-11 17:47:06.960  INFO 21608 --- [h_RMROLE_1_5_16] io.seata.rm.AbstractRMHandler            : Branch Rollbacked result: PhaseTwo_Rollbacked
```
### Rollback 调用流程 Server 端控制台输出
```console
17:47:06.423  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : receive msg[single]: GlobalBeginRequest{transactionName='purchase(java.lang.String, java.lang.String, int, boolean)', timeout=60000}, clientIp: 192.168.2.126, vgroup: my_test_tx_group
17:47:06.442  INFO --- [verHandlerThread_1_24_500] [coordinator.DefaultCoordinator] [       doGlobalBegin]  [192.168.2.126:8091:2252233925503221807] : Begin new global transaction applicationId: business-tcc,transactionServiceGroup: my_test_tx_group, transactionName: purchase(java.lang.String, java.lang.String, int, boolean),timeout:60000,xid:192.168.2.126:8091:2252233925503221807
17:47:06.442  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : result msg[single]: GlobalBeginResponse{xid='192.168.2.126:8091:2252233925503221807', extraData='null', resultCode=Success, msg='null'}, clientIp: 192.168.2.126, vgroup: my_test_tx_group
17:47:06.648  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : receive msg[merged]: BranchRegisterRequest{xid='192.168.2.126:8091:2252233925503221807', branchType=TCC, resourceId='tryDeduct', lockKey='null', applicationData='{"actionContext":{"action-start-time":1694425626577,"useTCCFence":true,"sys::prepare":"tryDeduct","commodityCode":"C100000","count":30,"sys::rollback":"rollback","sys::commit":"commit","host-name":"192.168.2.126","actionName":"tryDeduct"}}'}, clientIp: 192.168.2.126, vgroup: my_test_tx_group
17:47:06.669  INFO --- [nPool.commonPool-worker-4] [erver.coordinator.AbstractCore] [bda$branchRegister$0]  [192.168.2.126:8091:2252233925503221807] : Register branch successfully, xid = 192.168.2.126:8091:2252233925503221807, branchId = 2252233925503221809, resourceId = tryDeduct ,lockKeys = null
17:47:06.670  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : result msg[merged]: BranchRegisterResponse{branchId=2252233925503221809, resultCode=Success, msg='null'}, clientIp: 192.168.2.126, vgroup: my_test_tx_group
17:47:06.743  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : receive msg[merged]: BranchRegisterRequest{xid='192.168.2.126:8091:2252233925503221807', branchType=TCC, resourceId='tryCreate', lockKey='null', applicationData='{"actionContext":{"action-start-time":1694425626742,"useTCCFence":true,"sys::prepare":"tryCreate","commodityCode":"C100000","count":30,"sys::rollback":"rollback","sys::commit":"commit","host-name":"192.168.2.126","userId":"U100000","actionName":"tryCreate"}}'}, clientIp: 192.168.2.126, vgroup: my_test_tx_group
17:47:06.770  INFO --- [nPool.commonPool-worker-4] [erver.coordinator.AbstractCore] [bda$branchRegister$0]  [192.168.2.126:8091:2252233925503221807] : Register branch successfully, xid = 192.168.2.126:8091:2252233925503221807, branchId = 2252233925503221811, resourceId = tryCreate ,lockKeys = null
17:47:06.770  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : result msg[merged]: BranchRegisterResponse{branchId=2252233925503221811, resultCode=Success, msg='null'}, clientIp: 192.168.2.126, vgroup: my_test_tx_group
17:47:06.787  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : receive msg[merged]: BranchRegisterRequest{xid='192.168.2.126:8091:2252233925503221807', branchType=TCC, resourceId='tryReduce', lockKey='null', applicationData='{"actionContext":{"action-start-time":1694425626784,"useTCCFence":true,"money":3000,"sys::prepare":"tryReduce","sys::rollback":"rollback","sys::commit":"commit","host-name":"192.168.2.126","userId":"U100000","actionName":"tryReduce"}}'}, clientIp: 192.168.2.126, vgroup: my_test_tx_group
17:47:06.804  INFO --- [nPool.commonPool-worker-4] [erver.coordinator.AbstractCore] [bda$branchRegister$0]  [192.168.2.126:8091:2252233925503221807] : Register branch successfully, xid = 192.168.2.126:8091:2252233925503221807, branchId = 2252233925503221813, resourceId = tryReduce ,lockKeys = null
17:47:06.804  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : result msg[merged]: BranchRegisterResponse{branchId=2252233925503221813, resultCode=Success, msg='null'}, clientIp: 192.168.2.126, vgroup: my_test_tx_group
17:47:06.879  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : receive msg[single]: GlobalRollbackRequest{xid='192.168.2.126:8091:2252233925503221807', extraData='null'}, clientIp: 192.168.2.126, vgroup: my_test_tx_group
17:47:06.963  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : receive msg[single]: BranchRollbackResponse{xid='192.168.2.126:8091:2252233925503221807', branchId=2252233925503221813, branchStatus=PhaseTwo_Rollbacked, resultCode=Success, msg='null'}, clientIp: 192.168.2.126, vgroup: my_test_tx_group
17:47:06.977  INFO --- [verHandlerThread_1_28_500] [server.coordinator.DefaultCore] [a$doGlobalRollback$3]  [192.168.2.126:8091:2252233925503221807] : Rollback branch transaction successfully, xid = 192.168.2.126:8091:2252233925503221807 branchId = 2252233925503221813
17:47:07.020  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : receive msg[single]: BranchRollbackResponse{xid='192.168.2.126:8091:2252233925503221807', branchId=2252233925503221811, branchStatus=PhaseTwo_Rollbacked, resultCode=Success, msg='null'}, clientIp: 192.168.2.126, vgroup: my_test_tx_group
17:47:07.031  INFO --- [verHandlerThread_1_28_500] [server.coordinator.DefaultCore] [a$doGlobalRollback$3]  [192.168.2.126:8091:2252233925503221807] : Rollback branch transaction successfully, xid = 192.168.2.126:8091:2252233925503221807 branchId = 2252233925503221811
17:47:07.094  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : receive msg[single]: BranchRollbackResponse{xid='192.168.2.126:8091:2252233925503221807', branchId=2252233925503221809, branchStatus=PhaseTwo_Rollbacked, resultCode=Success, msg='null'}, clientIp: 192.168.2.126, vgroup: my_test_tx_group
17:47:07.104  INFO --- [verHandlerThread_1_28_500] [server.coordinator.DefaultCore] [a$doGlobalRollback$3]  [192.168.2.126:8091:2252233925503221807] : Rollback branch transaction successfully, xid = 192.168.2.126:8091:2252233925503221807 branchId = 2252233925503221809
17:47:07.104  INFO --- [verHandlerThread_1_28_500] [server.coordinator.DefaultCore] [    doGlobalRollback]  [192.168.2.126:8091:2252233925503221807] : Rollback global transaction successfully, xid = 192.168.2.126:8091:2252233925503221807.
17:47:07.105  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : result msg[single]: GlobalRollbackResponse{globalStatus=Rollbacked, resultCode=Success, msg='null'}, clientIp: 192.168.2.126, vgroup: my_test_tx_group
```
#### 按时序整理的完整流程输出（包含Client 端、Server 端）
```console
17:47:06.423  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : receive msg[single]: GlobalBeginRequest{transactionName='purchase(java.lang.String, java.lang.String, int, boolean)', timeout=60000}, clientIp: 192.168.2.126, vgroup: my_test_tx_group
17:47:06.442  INFO --- [verHandlerThread_1_24_500] [coordinator.DefaultCoordinator] [       doGlobalBegin]  [192.168.2.126:8091:2252233925503221807] : Begin new global transaction applicationId: business-tcc,transactionServiceGroup: my_test_tx_group, transactionName: purchase(java.lang.String, java.lang.String, int, boolean),timeout:60000,xid:192.168.2.126:8091:2252233925503221807
17:47:06.442  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : result msg[single]: GlobalBeginResponse{xid='192.168.2.126:8091:2252233925503221807', extraData='null', resultCode=Success, msg='null'}, clientIp: 192.168.2.126, vgroup: my_test_tx_group

17:47:06.443  INFO 24792 --- [nio-8484-exec-7] i.seata.tm.api.DefaultGlobalTransaction  : Begin new global transaction [192.168.2.126:8091:2252233925503221807]
17:47:06.443  INFO 24792 --- [nio-8484-exec-7] s.zlyx.example.service.BusinessService   : New Transaction Begins: 192.168.2.126:8091:2252233925503221807

17:47:06.648  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : receive msg[merged]: BranchRegisterRequest{xid='192.168.2.126:8091:2252233925503221807', branchType=TCC, resourceId='tryDeduct', lockKey='null', applicationData='{"actionContext":{"action-start-time":1694425626577,"useTCCFence":true,"sys::prepare":"tryDeduct","commodityCode":"C100000","count":30,"sys::rollback":"rollback","sys::commit":"commit","host-name":"192.168.2.126","actionName":"tryDeduct"}}'}, clientIp: 192.168.2.126, vgroup: my_test_tx_group
17:47:06.669  INFO --- [nPool.commonPool-worker-4] [erver.coordinator.AbstractCore] [bda$branchRegister$0]  [192.168.2.126:8091:2252233925503221807] : Register branch successfully, xid = 192.168.2.126:8091:2252233925503221807, branchId = 2252233925503221809, resourceId = tryDeduct ,lockKeys = null
17:47:06.670  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : result msg[merged]: BranchRegisterResponse{branchId=2252233925503221809, resultCode=Success, msg='null'}, clientIp: 192.168.2.126, vgroup: my_test_tx_group

17:47:06.676  INFO 22560 --- [nio-8181-exec-1] io.seata.rm.AbstractResourceManager      : branch register success, xid:192.168.2.126:8091:2252233925503221807, branchId:2252233925503221809, lockKeys:null
17:47:06.690  INFO 22560 --- [nio-8181-exec-1] io.seata.rm.tcc.TCCFenceHandler          : TCC fence prepare result: true. xid: 192.168.2.126:8091:2252233925503221807, branchId: 2252233925503221809
17:47:06.694  INFO 22560 --- [nio-8181-exec-1] s.z.e.tccaction.impl.StockActionImpl     : deduct stock balance in transaction: 192.168.2.126:8091:2252233925503221807

17:47:06.743  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : receive msg[merged]: BranchRegisterRequest{xid='192.168.2.126:8091:2252233925503221807', branchType=TCC, resourceId='tryCreate', lockKey='null', applicationData='{"actionContext":{"action-start-time":1694425626742,"useTCCFence":true,"sys::prepare":"tryCreate","commodityCode":"C100000","count":30,"sys::rollback":"rollback","sys::commit":"commit","host-name":"192.168.2.126","userId":"U100000","actionName":"tryCreate"}}'}, clientIp: 192.168.2.126, vgroup: my_test_tx_group
17:47:06.770  INFO --- [nPool.commonPool-worker-4] [erver.coordinator.AbstractCore] [bda$branchRegister$0]  [192.168.2.126:8091:2252233925503221807] : Register branch successfully, xid = 192.168.2.126:8091:2252233925503221807, branchId = 2252233925503221811, resourceId = tryCreate ,lockKeys = null
17:47:06.770  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : result msg[merged]: BranchRegisterResponse{branchId=2252233925503221811, resultCode=Success, msg='null'}, clientIp: 192.168.2.126, vgroup: my_test_tx_group

17:47:06.771  INFO 24332 --- [nio-8282-exec-6] io.seata.rm.AbstractResourceManager      : branch register success, xid:192.168.2.126:8091:2252233925503221807, branchId:2252233925503221811, lockKeys:null
17:47:06.779  INFO 24332 --- [nio-8282-exec-6] io.seata.rm.tcc.TCCFenceHandler          : TCC fence prepare result: true. xid: 192.168.2.126:8091:2252233925503221807, branchId: 2252233925503221811
17:47:06.779  INFO 24332 --- [nio-8282-exec-6] s.z.e.tccaction.impl.OrderActionImpl     : create order in transaction: 192.168.2.126:8091:2252233925503221807

17:47:06.787  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : receive msg[merged]: BranchRegisterRequest{xid='192.168.2.126:8091:2252233925503221807', branchType=TCC, resourceId='tryReduce', lockKey='null', applicationData='{"actionContext":{"action-start-time":1694425626784,"useTCCFence":true,"money":3000,"sys::prepare":"tryReduce","sys::rollback":"rollback","sys::commit":"commit","host-name":"192.168.2.126","userId":"U100000","actionName":"tryReduce"}}'}, clientIp: 192.168.2.126, vgroup: my_test_tx_group
17:47:06.804  INFO --- [nPool.commonPool-worker-4] [erver.coordinator.AbstractCore] [bda$branchRegister$0]  [192.168.2.126:8091:2252233925503221807] : Register branch successfully, xid = 192.168.2.126:8091:2252233925503221807, branchId = 2252233925503221813, resourceId = tryReduce ,lockKeys = null
17:47:06.804  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : result msg[merged]: BranchRegisterResponse{branchId=2252233925503221813, resultCode=Success, msg='null'}, clientIp: 192.168.2.126, vgroup: my_test_tx_group

17:47:06.805  INFO 21608 --- [nio-8383-exec-6] io.seata.rm.AbstractResourceManager      : branch register success, xid:192.168.2.126:8091:2252233925503221807, branchId:2252233925503221813, lockKeys:null
17:47:06.829  INFO 21608 --- [nio-8383-exec-6] io.seata.rm.tcc.TCCFenceHandler          : TCC fence prepare result: true. xid: 192.168.2.126:8091:2252233925503221807, branchId: 2252233925503221813
17:47:06.829  INFO 21608 --- [nio-8383-exec-6] s.z.e.tccaction.impl.AccountActionImpl   : reduce account balance in transaction: 192.168.2.126:8091:2252233925503221807
17:47:06.833  INFO 21608 --- [nio-8383-exec-6] s.z.e.tccaction.impl.AccountActionImpl   : balance after transaction: -2000

17:47:06.878  INFO 24792 --- [nio-8484-exec-7] i.seata.tm.api.DefaultGlobalTransaction  : transaction 192.168.2.126:8091:2252233925503221807 will be rollback

17:47:06.879  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : receive msg[single]: GlobalRollbackRequest{xid='192.168.2.126:8091:2252233925503221807', extraData='null'}, clientIp: 192.168.2.126, vgroup: my_test_tx_group

17:47:06.922  INFO 21608 --- [h_RMROLE_1_5_16] i.s.c.r.p.c.RmBranchRollbackProcessor    : rm handle branch rollback process:BranchRollbackRequest{xid='192.168.2.126:8091:2252233925503221807', branchId=2252233925503221813, branchType=TCC, resourceId='tryReduce', applicationData='{"actionContext":{"action-start-time":1694425626784,"useTCCFence":true,"money":3000,"sys::prepare":"tryReduce","sys::rollback":"rollback","sys::commit":"commit","host-name":"192.168.2.126","userId":"U100000","actionName":"tryReduce"}}'}
17:47:06.924  INFO 21608 --- [h_RMROLE_1_5_16] io.seata.rm.AbstractRMHandler            : Branch Rollbacking: 192.168.2.126:8091:2252233925503221807 2252233925503221813 tryReduce
17:47:06.937  INFO 21608 --- [h_RMROLE_1_5_16] io.seata.rm.tcc.TCCFenceHandler          : Insert tcc fence record result: true. xid: 192.168.2.126:8091:2252233925503221807, branchId: 2252233925503221813
17:47:06.960  INFO 21608 --- [h_RMROLE_1_5_16] io.seata.rm.AbstractResourceManager      : TCC resource rollback result : true, xid: 192.168.2.126:8091:2252233925503221807, branchId: 2252233925503221813, resourceId: tryReduce
17:47:06.960  INFO 21608 --- [h_RMROLE_1_5_16] io.seata.rm.AbstractRMHandler            : Branch Rollbacked result: PhaseTwo_Rollbacked

17:47:06.963  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : receive msg[single]: BranchRollbackResponse{xid='192.168.2.126:8091:2252233925503221807', branchId=2252233925503221813, branchStatus=PhaseTwo_Rollbacked, resultCode=Success, msg='null'}, clientIp: 192.168.2.126, vgroup: my_test_tx_group
17:47:06.977  INFO --- [verHandlerThread_1_28_500] [server.coordinator.DefaultCore] [a$doGlobalRollback$3]  [192.168.2.126:8091:2252233925503221807] : Rollback branch transaction successfully, xid = 192.168.2.126:8091:2252233925503221807 branchId = 2252233925503221813

17:47:06.980  INFO 24332 --- [h_RMROLE_1_5_16] i.s.c.r.p.c.RmBranchRollbackProcessor    : rm handle branch rollback process:BranchRollbackRequest{xid='192.168.2.126:8091:2252233925503221807', branchId=2252233925503221811, branchType=TCC, resourceId='tryCreate', applicationData='{"actionContext":{"action-start-time":1694425626742,"useTCCFence":true,"sys::prepare":"tryCreate","commodityCode":"C100000","count":30,"sys::rollback":"rollback","sys::commit":"commit","host-name":"192.168.2.126","userId":"U100000","actionName":"tryCreate"}}'}
17:47:06.982  INFO 24332 --- [h_RMROLE_1_5_16] io.seata.rm.AbstractRMHandler            : Branch Rollbacking: 192.168.2.126:8091:2252233925503221807 2252233925503221811 tryCreate
17:47:07.006  INFO 24332 --- [h_RMROLE_1_5_16] io.seata.rm.tcc.TCCFenceHandler          : Insert tcc fence record result: true. xid: 192.168.2.126:8091:2252233925503221807, branchId: 2252233925503221811
17:47:07.017  INFO 24332 --- [h_RMROLE_1_5_16] io.seata.rm.AbstractResourceManager      : TCC resource rollback result : true, xid: 192.168.2.126:8091:2252233925503221807, branchId: 2252233925503221811, resourceId: tryCreate
17:47:07.017  INFO 24332 --- [h_RMROLE_1_5_16] io.seata.rm.AbstractRMHandler            : Branch Rollbacked result: PhaseTwo_Rollbacked

17:47:07.020  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : receive msg[single]: BranchRollbackResponse{xid='192.168.2.126:8091:2252233925503221807', branchId=2252233925503221811, branchStatus=PhaseTwo_Rollbacked, resultCode=Success, msg='null'}, clientIp: 192.168.2.126, vgroup: my_test_tx_group
17:47:07.031  INFO --- [verHandlerThread_1_28_500] [server.coordinator.DefaultCore] [a$doGlobalRollback$3]  [192.168.2.126:8091:2252233925503221807] : Rollback branch transaction successfully, xid = 192.168.2.126:8091:2252233925503221807 branchId = 2252233925503221811

17:47:07.035  INFO 22560 --- [h_RMROLE_1_1_16] i.s.c.r.p.c.RmBranchRollbackProcessor    : rm handle branch rollback process:BranchRollbackRequest{xid='192.168.2.126:8091:2252233925503221807', branchId=2252233925503221809, branchType=TCC, resourceId='tryDeduct', applicationData='{"actionContext":{"action-start-time":1694425626577,"useTCCFence":true,"sys::prepare":"tryDeduct","commodityCode":"C100000","count":30,"sys::rollback":"rollback","sys::commit":"commit","host-name":"192.168.2.126","actionName":"tryDeduct"}}'}
17:47:07.036  INFO 22560 --- [h_RMROLE_1_1_16] io.seata.rm.AbstractRMHandler            : Branch Rollbacking: 192.168.2.126:8091:2252233925503221807 2252233925503221809 tryDeduct
17:47:07.090  INFO 22560 --- [h_RMROLE_1_1_16] io.seata.rm.AbstractResourceManager      : TCC resource rollback result : true, xid: 192.168.2.126:8091:2252233925503221807, branchId: 2252233925503221809, resourceId: tryDeduct
17:47:07.091  INFO 22560 --- [h_RMROLE_1_1_16] io.seata.rm.AbstractRMHandler            : Branch Rollbacked result: PhaseTwo_Rollbacked

17:47:07.094  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : receive msg[single]: BranchRollbackResponse{xid='192.168.2.126:8091:2252233925503221807', branchId=2252233925503221809, branchStatus=PhaseTwo_Rollbacked, resultCode=Success, msg='null'}, clientIp: 192.168.2.126, vgroup: my_test_tx_group

17:47:07.104  INFO --- [verHandlerThread_1_28_500] [server.coordinator.DefaultCore] [a$doGlobalRollback$3]  [192.168.2.126:8091:2252233925503221807] : Rollback branch transaction successfully, xid = 192.168.2.126:8091:2252233925503221807 branchId = 2252233925503221809
17:47:07.104  INFO --- [verHandlerThread_1_28_500] [server.coordinator.DefaultCore] [    doGlobalRollback]  [192.168.2.126:8091:2252233925503221807] : Rollback global transaction successfully, xid = 192.168.2.126:8091:2252233925503221807.

17:47:07.105  INFO --- [     batchLoggerPrint_1_1] [ocessor.server.BatchLogHandler] [                 run]  [] : result msg[single]: GlobalRollbackResponse{globalStatus=Rollbacked, resultCode=Success, msg='null'}, clientIp: 192.168.2.126, vgroup: my_test_tx_group

17:47:07.105  INFO 24792 --- [nio-8484-exec-7] i.seata.tm.api.DefaultGlobalTransaction  : transaction end, xid = 192.168.2.126:8091:2252233925503221807
17:47:07.105  INFO 24792 --- [nio-8484-exec-7] i.seata.tm.api.DefaultGlobalTransaction  : [192.168.2.126:8091:2252233925503221807] rollback status: Rollbacked
```

（完）
