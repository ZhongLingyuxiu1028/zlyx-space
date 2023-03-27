# 数据权限（一）数据权限调用流程分析（参照 Mybatis Plus 数据权限插件）
- - -
## 前言
之前写过一篇关于数据权限的文章【[【若依】开源框架学习笔记 06 - 数据权限](/ruoyi-vue-plus/data-permission/00_permission.md) 】，在原版若依和 3.4.0 及其以下版本的 RuoYi-Vue-Plus 中使用的都是基于切面方式实现的数据权限功能。

在 RuoYi-Vue-Plus 3.5.0 中，[狮子大佬](https://blog.csdn.net/weixin_40461281) 重写了数据权限的实现。

对于新的数据权限的使用方法也在框架wiki中有说明，所以本文只是在此基础上做简单的分析，仅作为学习之用，如有错误也请大佬们指出。

## 参考目录
- [框架 wiki - 数据权限](https://gitee.com/JavaLionLi/RuoYi-Vue-Plus/wikis/%E6%A1%86%E6%9E%B6%E5%8A%9F%E8%83%BD/%E6%95%B0%E6%8D%AE%E6%9D%83%E9%99%90)
- [Mybatis Plus 文档 - 数据权限](https://baomidou.com/pages/1864e1/#%E6%95%B0%E6%8D%AE%E8%8C%83%E5%9B%B4-%E6%95%B0%E6%8D%AE%E6%9D%83%E9%99%90)

框架中没有直接使用 Mybatis Plus 原生的数据权限插件，但是从写法来看应该对此有所借鉴，所以也可以参考一下 Mybatis Plus 的数据权限插件源码实现自己的数据权限功能。

## 代码分析

> 框架wiki中关于数据权限的简单说明：<br>
![这里是引用](img01/113b1aae75dd4f709c8f3f5e5014d66f.png)
### 1、数据权限配置 `MybatisPlusConfig`
![在这里插入图片描述](img01/39f6498c4189419fa37cde1e72d79582.png)<br>
在 `MybatisPlusInterceptor` 拦截器中加入了自定义的数据权限拦截器组件 `PlusDataPermissionInterceptor`。

### 2、数据权限拦截器 `PlusDataPermissionInterceptor`
![在这里插入图片描述](img01/ddd59466797641529cd36e7843a1339a.png)

数据权限拦截器继承了 `JsqlParserSupport` Jsql 解析器，实现了 `InnerInterceptor` 拦截器接口：<br>
- `beforeQuery` ：`Executor.query ` 操作前置处理 - `InnerInterceptor`
  ![在这里插入图片描述](img01/c74c246cfe034f7595ff4a26e23ce90a.png)
- `beforePrepare` ：`StatementHandler.prepare` 操作前置处理 - `InnerInterceptor`
  ![在这里插入图片描述](img01/8caaf9f4961948c3ada595ef0f0905a8.png)
- `processSelect` ：处理查询 - `JsqlParserSupport`
- `processUpdate` ：处理更新 - `JsqlParserSupport`
- `processDelete` ：处理删除 - `JsqlParserSupport`
  ![在这里插入图片描述](img01/14b011f11c5246789ba01286a1606a3d.png)

在 Mybatis Plus 数据权限插件源码中，也有类似的拦截器 `DataPermissionInterceptor`。不过只重写了两个方法 `beforeQuery` 和 `processSelect`。<br>
![在这里插入图片描述](img01/0157e7c6d34042a5a4d2985a67a257f3.png)
### 3、数据权限处理器 `PlusDataPermissionHandler`
在拦截器处理查询的方法中有调用 `setWhere` 方法进行 SQL 语句 Where 条件的设置。主要的逻辑就是调用处理器获取 SQL 的方法 `getSqlSegment`。

`PlusDataPermissionHandler#getSqlSegment`<br>
![在这里插入图片描述](img01/1167efcb97214cb49024fbcf64032145.png)

当然这个方法也是根据框架进行了扩展，源码中只是一个简单的接口。<br>
![在这里插入图片描述](img01/71ad307d1d424b1aba1fb0eb356823bd.png)
## 方法调用流程
### 1、测试方法
`TestDemoController#list()`<br>
![在这里插入图片描述](img01/10e1a2df12b14f8b81dcfa458ba00b6e.png)

这是一个新增的方法，原来 Demo 中有一个加上了分页的测试方法，这里暂时排除分页进行测试。

`TestDemoServiceImpl#queryList()`<br>
![在这里插入图片描述](img01/cddb07bd91f044569a989aebf3d719b8.png)

`buildQueryWrapper` 是根据请求参数构建条件构造器，测试方法中不传参数，则会查询全部数据。

`ServicePlusImpl#listVo()`<br>
![在这里插入图片描述](img01/4086bcc58c6c49c0817405aada9d2c8b.png)

`ServicePlusImpl` 是封装的通用实现类。

`BaseMapperPlus#selectVoList()`<br>
![在这里插入图片描述](img01/cfa1f7d9c3ae487f85ff21d23dda218e.png)

`this.selectList(wrapper)` 就是调用对应的 Mapper 的查询列表方法。

`TestDemoMapper#selectList`<br>
![在这里插入图片描述](img01/e04ae27de0264b46b0fe945c31277ef9.png)

这里是对原生方法的重写，加入了自定义数据权限注解 `@DataPermission` 以及 `@DataColumn`。

如果 SQL 是自定义方法也是在对应方法上加上注解即可。<br>
![在这里插入图片描述](img01/b8f7726f57c94c0fac8276444d813c58.png)

![在这里插入图片描述](img01/79fc2553e9a2423ea0eb39af4eb65439.png)
### 2、超级管理员测试
#### 2.1、`beforeQuery()`
###### `PlusDataPermissionInterceptor#beforeQuery()`
![在这里插入图片描述](img01/1bb35ec0cff044d2975f11d12eef551b.png)

首先判断忽略注解。<br>
`InterceptorIgnoreHelper#willIgnoreDataPermission()`<br>
![在这里插入图片描述](img01/ac008d2db2214143975eadd721017d70.png)

`InterceptorIgnoreHelper#willIgnore()`<br>
![在这里插入图片描述](img01/137671b550e04e93a4f4c088bd036f06.png)

![在这里插入图片描述](img01/c96dbc95d48f4afebb42550c83422118.png)

结果为 `false`。

回到 `beforeQuery` 中，继续判断注解是否有效。<br>
![在这里插入图片描述](img01/4685e7131d574784b775297cdc7b28bc.png)

`PlusDataPermissionHandler#isInvalid()`<br>
![在这里插入图片描述](img01/ad585b8cb7a3402fa7cd2faf2f260606.png)

![在这里插入图片描述](img01/2e48e61962ae4037ad8d303dd0b66961.png)

结果为 `false`。

最终得到 SQL 语句。<br>
![在这里插入图片描述](img01/923a47bb63174a58a028449d011209be.png)

![在这里插入图片描述](img01/436b87b93c4d4f989ace2e50d6ef6d20.png)

SQL 解析器 `JsqlParserSupport#parserSingle`<br>
![在这里插入图片描述](img01/fe2255d7340d42968f41eda6b2ebee57.png)

执行SQL解析。
###### `JsqlParserSupport#processParser()`
![在这里插入图片描述](img01/ab88170328ce4dee8d4afde9e5572345.png)

调用处理查询的方法。
###### `PlusDataPermissionInterceptor#processSelect()`
![在这里插入图片描述](img01/976b7cfaa85e4af4a2874a0232f32ae1.png)

设置 Where 条件。<br>
`PlusDataPermissionInterceptor#setWhere()`<br>
![在这里插入图片描述](img01/0276e2ba242940fa974d8e2a0ab6a1ae.png)
###### `PlusDataPermissionHandler#getSqlSegment()`
根据注解以及权限获取 SQL 片段。<br>
![在这里插入图片描述](img01/6f9cb4d496a043feb92e0c38bd55e71b.png)<br>
- 1、通过方法 `findAnnotation(mappedStatementId)` 获取到注解内容。<br>
  `PlusDataPermissionHandler#findAnnotation()`<br>
  首先从 dataPermissionCacheMap 中获取，没有则通过 `AnnotationUtil.getAnnotation()` 方法获取，并存到 dataPermissionCacheMap 中。<br>
  ![在这里插入图片描述](img01/b4e53c50595d46108c25b95d2a004f8b.png)<br>
  然后返回得到的注解内容。

- 2、获取当前用户并判断权限。<br>
  首先通过 `DataPermissionHelper.getVariable("user")` 方法获取用户信息，如果没有，则通过查询数据库获取，并把用户信息通过 `DataPermissionHelper` 存到上下文中。<br>
  ![在这里插入图片描述](img01/8223721f48224d548354a1a47c3ba630.png)<br>
  ![在这里插入图片描述](img01/63aebdf630dd4c45bb484f6a5075a728.png)<br>
  ![在这里插入图片描述](img01/d505281d10dd4cca9e458f6223bb0c30.png)<br>
  此处为超级管理员直接返回 where 语句。<br>
  ![在这里插入图片描述](img01/eb89ba38590543e6845b38b4f48cf8d2.png)

至此完成了 `beforeQuery()` 整个方法的流程。

#### 2.2、`beforePrepare()`
![在这里插入图片描述](img01/46c07b38f04a46c7b73891f64d2ed1b7.png)
#### 2.3、执行输出
![在这里插入图片描述](img01/313a253957c64d7a981272c256356a33.png)<br>
![在这里插入图片描述](img01/8bac464f0e384bbb95865138feb97109.png)
### 3、非超级管理员测试
![在这里插入图片描述](img01/5a86c5896ace4f9c86e2b578c74bdc42.png)
**注：非超级管理员测试的流程和超级管理员基本相同，下面只分析不同的部分（主要是在where条件的获取不一样），如果有不了解的地方可以多 Debug。**

`PlusDataPermissionHandler#getSqlSegment()`<br>
![在这里插入图片描述](img01/67a39b3474c644fba3b4680623db7785.png)
非超级管理员用户，通过 `buildDataFilter()` 方法构造 SQL 查询条件。
#### 3.1、`buildDataFilter()`
###### `PlusDataPermissionHandler#buildDataFilter()`
![在这里插入图片描述](img01/4dddb04cbd544d0190f63a5b991713f3.png)
- 1、获取拼接字符
  ![在这里插入图片描述](img01/632b4fc8d7204a25864ae8d36b580286.png)
- 2、从 `DataPermissionHelper` 获取用户信息并保存到上下文中。
  ![在这里插入图片描述](img01/d0a92f8664c049b7a52daef42953e877.png)
- 3、获取用户角色信息并循环进行操作
  ![在这里插入图片描述](img01/ad7b1ee245b34d49a004f103e46fa38d.png)<br>
  获取用户角色权限泛型。<br>
  ![在这里插入图片描述](img01/130fbd5268d743d3b5c3ced3e7057a20.png)<br>
  循环数据权限注解信息并操作。<br>
  ![在这里插入图片描述](img01/0b63b01354c34c3a9babbd3c05c0ce1c.png)<br>
  `@sdss.getDeptAndChild` 对应的是方法 `SysDataScopeServiceImpl#getDeptAndChild()`。<br>
  ![在这里插入图片描述](img01/1b05108002ae476aab0854635052a89f.png)<br>
  ![在这里插入图片描述](img01/9529e1cbfa8a4007abb7ca393df7a88d.png)<br>
  该方法获取用户权限部门id。解析得到 SQL 语句。<br>
  ![在这里插入图片描述](img01/3fef498f36d04286a29458a8b4083539.png)<br>
- 4、处理 SQL 并返回
  ![在这里插入图片描述](img01/4e191ef818d54e6eb60e740659a41550.png)<br>
  此处需要截掉 OR 是因为只有一个条件时不需要 OR，否则 SQL 就变成了 `where OR ...`。

SQL 查询条件构造完成，返回 `PlusDataPermissionHandler#getSqlSegment()` 方法。<br>
![在这里插入图片描述](img01/8df150fe39be49f99413fb362fd8b499.png)

由上面可得到最终的 SQL 语句查询条件。其余流程和上面超级管理员基本一致。


#### 3.2、执行输出
![在这里插入图片描述](img01/61bb059d3df0478ab95947d13b141fc6.png)

![在这里插入图片描述](img01/5e7500ede495475f9b3b0aac7d680b6e.png)
