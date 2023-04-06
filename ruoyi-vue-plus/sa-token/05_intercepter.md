# Sa-Token（五）登录验证拦截器之 Token 有效期及其续签
- - -

## 前言
前几天在群里面聊到关于 token 续签的问题，这个在 Sa-Token 官方文档也有相应的说明，这篇文章还是从源码角度去看看到底做了什么。

## 参考目录
- [Sa-Token 官方文档：Token有效期详解](https://sa-token.dev33.cn/doc/index.html#/fun/token-timeout)

## 框架集成
### 基于 Sa-Token 最新版本
![在这里插入图片描述](img05/744c56e3da7048c4b668fa87ca906d54.png)
### yaml 配置文件
这里需要关注的是两个 timeout 配置：<br>
![在这里插入图片描述](img05/5a88e5a96f9d4ed8b69dbc0e19eb987a.png)

### Sa-Token 配置 `SaTokenConfig` 注册拦截器
`SaTokenConfig#addInterceptors`<br>
![在这里插入图片描述](img05/81b5cce90439483aa39bb85ea966dfab.png)

这里重点关注 `StpUtil.checkLogin()` 这个方法，其他的会放在后面扩展去简单进行讨论。
## 功能调用流程分析
关于 `StpUtil.checkLogin()` 底层的调用过程比较简单容易理解，所以没有画流程图。

从上面 `SaTokenConfig#addInterceptors` 开始进入，到 `StpUtil#checkLogin` ：<br>
![在这里插入图片描述](img05/6fc1f964e6b7411aab55e5b97173b841.png)<br>

再深入一层，到 `StpLogic#checkLogin`：<br>
![在这里插入图片描述](img05/2141187c151046fe9f1f6f6c232979cb.png)<br>

然后到今天的主角 `StpLogic#getLoginId()`。
###  1、获取当前会话账号id `StpLogic#getLoginId()`
![在这里插入图片描述](img05/f0a09f822f3d418c962e79fb2795dfc8.png)

注释都写得很明白了，先是进行各种判断，如果全部通过了，来到最后两个方法：

1. 检查是否临时过期<br>
   这里的`checkActivityTimeout` 检查的值对应的是配置文件中的 `activity-timeout` 属性值。
2. 判断是否自动续签<br>
   续签的时间也是对应的是配置文件中的 `activity-timeout` 属性值。

### 1.1、检查 token 是否已经临时过期 `StpLogic#checkActivityTimeout`
![在这里插入图片描述](img05/dc317bb6be5047a8b115b6a0ce5764ec.png)
### 1.1.1、计算过期时间 `StpLogic#getTokenActivityTimeoutByToken`
![在这里插入图片描述](img05/1a1e540be256427ab22520ffcbbfdc15.png)
### 1.2、已过期抛出异常 `NotLoginException` （Token已过期）
![在这里插入图片描述](img05/272ee1830b9c493a942542a7d9221d05.png)

全局异常处理 `GlobalExceptionHandler#handleNotLoginException`：<br>
![在这里插入图片描述](img05/dc6d5270be754c9683cd9a5a8f253b63.png)<br>

控制台输出：<br>
![在这里插入图片描述](img05/41e8e6edc7c6420caceb1736187a5da1.png)
### 1.3、未过期，完成验证
未过期则打上检查标记，完成验证。<br>
![在这里插入图片描述](img05/1cd2376575e047af8288387c29ad4382.png)
### 2、续签 `StpLogic#updateLastActivityToNow`
![在这里插入图片描述](img05/c142eceeaac74fd89405c2d0e685d747.png)

续签实际上就是更新缓存中的最后操作时间为当前时间。<br>
![在这里插入图片描述](img05/2a8bc9306f1441e98a11263383d6c822.png)
### 3、完成所有步骤，返回 loginId
![在这里插入图片描述](img05/688c901fc679463293be25391bcd1d7e.png)

至此，拦截器登录检查完成。

## 扩展分析
### 扩展1、关于手动续签

>图源 Sa-Token 官方文档：<br>
> ![这里是引用](img05/78f2c860d8e64114a879320bf01c6bd5.png)

官方提供了手动续签的方法，底层也是调用了上面续签的方法 `StpLogic#updateLastActivityToNow` 。

值得注意的是，手动续签的时候如果判断 token 临时有效期过期了，依然可以续签成功，因为续签方法是单独的，并没有判断 token 的状态。换句话说，如果捕获到 `NotLoginException` 异常，那么可以在 catch 中进行手动续签。<br>
![在这里插入图片描述](img05/e604bab869924b5b98059838c0d3f2c3.png)<br>

这样的话依然可以返回正确结果，而非抛出 401 异常。
### 扩展2、为什么需要两个 timeout ？
关于这个问题，我询问了一下 [狮子大佬](https://blog.csdn.net/weixin_40461281?type=blog) ，他说得比较详细，我直接贴出来：

> ![这里是引用](img05/a603559f7a0648c08638c4d778ec4881.png)<br>
> ![在这里插入图片描述](img05/cb87f7679c7e457dab83e255d52270da.png)<br>
### 扩展3、关于 [存储器] 包装类 `SaStorage`
在前面 `1.1、检查 token 是否已经临时过期` 里面的方法有用到这个对象（存取检查标记），所以拿出来说一下。

通过查找源码可以得知这个对象是一开始被 Spring 加载到容器中的，本质上就是一个 `Servletcontext` 对象。

先从 1.1 方法 `StpLogic#checkActivityTimeout` 回溯：

调用方法 `SaStorage storage = SaHolder.getStorage();`<br>
![在这里插入图片描述](img05/bf0ae935a39b46868d95c4b5ee7387a6.png)<br>

`SaManager#getSaTokenContextOrSecond`<br>
![在这里插入图片描述](img05/f561a6a5bfd24435b1a72e87837afd64.png)<br>

Sa-Token 上下文处理器 `SaTokenContext`<br>
![在这里插入图片描述](img05/c40191bd1ea2439997a5f75cc28c8aba.png)<br>

关于 `SaTokenContext` [官方文档](https://sa-token.dev33.cn/doc/index.html#/fun/sa-token-context) 也有作介绍：<br>

> ![在这里插入图片描述](img05/9cb3d3ac39bf4ebb9d9250dc9770339a.png)<br>

最后一段话很重要，继续找源头：<br>
![在这里插入图片描述](img05/41fc82f6d50345f480a027c6e73a4851.png)

![在这里插入图片描述](img05/d7595420dd0043fb9df445a97afbb1bb.png)

![在这里插入图片描述](img05/18b56208c38740579cf54f0a08b6939b.png)

至此可知 `SaStorage` 的完整创建流程。
### 扩展4、登录接口中 `SaStorage` 使用
在之前登录接口中也有用到这个对象，因此可以在这里也整理一下。<br>

`StpLogic#login`<br>
![在这里插入图片描述](img05/e811f331aa544204994c34e2eb0d053a.png)<br>

`StpLogic#setTokenValueToStorage`<br>
![在这里插入图片描述](img05/5749e50cff8c4eb18a427408d6324076.png)
### 扩展5、设置注解允许匿名访问的url `ExcludeUrlProperties`
在配置拦截器时需要排除指定的路径，如下：<br>

![在这里插入图片描述](img05/008e38b2753d4fcfb5d3bd3642b6954f.png)

参考 [框架wiki](https://gitee.com/dromara/RuoYi-Vue-Plus/wikis/%E6%A1%86%E6%9E%B6%E5%8A%9F%E8%83%BD/%E6%8E%A5%E5%8F%A3%E6%94%BE%E8%A1%8C) ，主要有两个途径：

1. yaml 配置文件
   ![在这里插入图片描述](img05/5b3cce995b794ae6801b09e23af08001.png)
2. 配置 `@Anonymous` 注解
   ![在这里插入图片描述](img05/a4b21a10139e433983696f2bef9ce4b4.png)

第一种比较简单，直接读取配置文件，我们聊聊第二种。

`ExcludeUrlProperties`<br>
![在这里插入图片描述](img05/b8af745c139442559495a042d1a5ee17.png)<br>

这个类是懒加载方式，第一次请求的时候才会将所有需要过滤的请求放到数组 `excludes` 中（`RequestMappingHandlerMapping` 能获取到容器中所有请求的方法），并且只会加载一次。<br>

这里会对 `/demo/demo/{id}` 这种路径进行替换变成 `/demo/demo/*`。<br>

![在这里插入图片描述](img05/d204d2b5350f467c939a2a45a1fe5f5c.png)<br>

![在这里插入图片描述](img05/bb33368379d64d13812bcb0348514936.png)<br>

但是要注意路径要唯一，否则会把同级别其他路径一起进行过滤：<br>

> ![在这里插入图片描述](img05/c56affa253c14915909d4b91de6f512e.png)
