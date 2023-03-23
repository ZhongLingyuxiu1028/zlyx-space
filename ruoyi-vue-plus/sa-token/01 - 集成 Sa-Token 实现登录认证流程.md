# Sa-Token（一）集成 Sa-Token 实现登录认证流程

## 前言
之前在交流群里面进行了一次投票，关于框架分支的使用情况，结果显示 Sa-Token 分支的使用比例也很高，因此狮子大佬决定在下个版本就会把 Sa-Token 接入主分支中。这篇文章主要是基于框架现有 Sa-Token 分支进行分析。

现版本 V3.5.0 主分支使用的还是 Spring Security 作为权限认证框架，作为 Spring 全家桶的一员功能强大，但是对于一般人而言上手也不容易，如果想了解更多的关于框架中 Spring Security 的内容可以查看我之前整理的一篇文章[【若依】开源框架学习笔记 07 - 登录认证流程（Spring Security 源码）](https://blog.csdn.net/Michelle_Zhong/article/details/117483769)。

最开始接触若依框架的时候框架中使用的是 Shiro 作为权限框架，相对 Spring Security 而言上手简单一些，如果想要了解的话也可以参考我整理的另外一篇文章[【若依】开源框架学习笔记 02 - Shiro 权限框架 ](https://blog.csdn.net/Michelle_Zhong/article/details/116396364)。

因为是之前整理的文章，可能版本更新迭代之后可能会有不一样的地方，请见谅。

## 参考目录
- [Sa-Token 官方文档](https://sa-token.dev33.cn/doc/index.html#/)

## 代码分析
### 1、框架集成
| 类                  | 说明                                       |
|--------------------|------------------------------------------|
| SaTokenConfig      | Sa-Token 配置                              |
| SaInterfaceImpl    | Sa-Token 权限认证接口实现类                       |
| PlusSaTokenDao     | Sa-Token 持久层接口 (使用框架自带RedisUtils实现 协议统一) |
| UserActionListener | 用户行为 侦听器的实现                              |


依赖添加 `pom.xml` ：<br>

![在这里插入图片描述](img01/d177f6ee532547cc98327529eeddcc65.png)<br>

![在这里插入图片描述](img01/7411b4bb3374446eadc4e70b7dbcdfd6.png)<br>

配置文件 `application.yml` ：<br>
![在这里插入图片描述](img01/6ef5fd4a987c4c529f19a037842f50fc.png)

### 2、登录流程源码分析
登录验证方法：`SysLoginService#login()`<br>
![在这里插入图片描述](img01/c18a77cea78f4613aa643f84c95ceb3b.png)<br>

在该方法中前面验证通过后，会调用 Sa-Token 的方法生成 token，然后返回。这里主要分析生成 token 的过程。

`LoginUtils#loginByDevice()`<br>
![在这里插入图片描述](img01/8a715d149130401d9560c3bf537299ea.png)<br>

用户类型枚举：<br>

![在这里插入图片描述](img01/6e25b5a8a72844c5bd0f80acfad5455e.png)<br>

设备类型枚举：<br>

![在这里插入图片描述](img01/86117ab5c5944fd1a070de37cd0c3969.png)<br>
#### 2.1、`StpLogic#login()`
![在这里插入图片描述](img01/ca67e321bf0b4dd4867a4b676b403cc8.png)<br>

源码的注释挺清楚的，`login()` 方法的主要逻辑如下：<br>
##### 2.1.1、前置检查：如果此账号已被封禁
`StpLogic#isDisable`<br>
![在这里插入图片描述](img01/b67e6cda089845b5b016c41637980e73.png)<br>

`StpLogic#splicingKeyDisable`<br>
![在这里插入图片描述](img01/64271d9daa9a4d3d9e760de2d2c4e2c3.png)<br>

`PlusSaTokenDao#get`<br>
![在这里插入图片描述](img01/28f0a0d45533417483c22a4f9a3568b2.png)<br>

结果为 `null`，继续接下来的流程。

##### 2.1.2、初始化 loginModel
![在这里插入图片描述](img01/ee99e0de467841e198e7c2677783547a.png)
##### 2.1.3、生成一个token
先判断是否允许并发登录，配置文件里面为 true；再判断是否共享 token，配置为 false；直接生成一个 token。<br>
![在这里插入图片描述](img01/053a2d338039462d9a43b538f9e0d8d3.png)<br>

![在这里插入图片描述](img01/0a688385a9ec4df295aef0b145176dcd.png)<br>

`StpLogicJwtForStyle#createTokenValue()`<br>
![在这里插入图片描述](img01/1e24f68aafbc4c91b072e2813cc3fc37.png)<br>

根据配置文件中的 secretKey 生成一个 token。<br>
![在这里插入图片描述](img01/c8df9dbc9c9a4676b72d2579271e814a.png)
##### 2.1.4、获取 User-Session , 续期
![在这里插入图片描述](img01/23e06f35e7f6480fa2112355ed106bc3.png)

`StpLogic#getSessionByLoginId`<br>
![在这里插入图片描述](img01/a625a78575464cd5b966b0d329459f12.png)<br>

`StpLogic#splicingKeySession`<br>
![在这里插入图片描述](img01/e269650f2ca44cc583a9d80d774409be.png)<br>

`StpLogic#getSessionBySessionId`<br>
![在这里插入图片描述](img01/893dc54426314519bf409490dfbadf0e.png)<br>

返回 session，在 User-Session 上记录 token 签名。<br>
![在这里插入图片描述](img01/a37faf0f37b94a78a2490d81885a9b2e.png)<br>
##### 2.1.5、持久化其它数据
![在这里插入图片描述](img01/780f77f5309f49a2b004f64c974956a9.png)<br>

`StpLogic#saveTokenToIdMapping`<br>
![在这里插入图片描述](img01/79842df4006d460e9f399c1ab1b8b1d0.png)<br>

`StpLogic#setTokenValue`<br>
![在这里插入图片描述](img01/b7c61288ec044f128295bb9fc3f622fb.png)<br>

`StpLogic#setLastActivityToNow`<br>
![在这里插入图片描述](img01/8868dc34675848dd9e4ab2b48d8d6611.png)<br>
##### 2.1.6、通知监听器，账号xxx登录成功
![在这里插入图片描述](img01/6b667fa9dd054b0fa7a072ad0520894d.png)<br>
`UserActionListener#doLogin`<br>
![在这里插入图片描述](img01/5caee93c9d3f4f0590f62104b085c2ad.png)<br>
#### 2.2、`StpLogic#getTokenValue`
![在这里插入图片描述](img01/4594f909ff344ef1a31d237c32876a02.png)
##### 2.2.1、获取
![在这里插入图片描述](img01/b65f347713864e2982fcdc9bad686488.png)
##### 2.2.2、如果打开了前缀模式，则裁剪掉
![在这里插入图片描述](img01/aaa812eaf6ab49c588b8c5876eeb3767.png)
##### 2.2.3、返回
![在这里插入图片描述](img01/34fe0c4be0224fce95d96317a454fb56.png)
