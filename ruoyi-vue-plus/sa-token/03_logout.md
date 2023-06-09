# Sa-Token（三）退出登录流程
- - -
## 前言
前两篇文章简单分析了一下登录以及通过注解校验权限的流程，有登录就有登出，所以这篇文章就来分析一下登出的流程源码。

## 参考目录
- [Sa-Token 官方文档](https://sa-token.dev33.cn/doc/index.html#/)

## 代码分析
请求接口：`SysLoginController#logout`<br>
![在这里插入图片描述](img03/3e409fe907fd422081ae29020af3aef1.png)<br>

方法很简单，就一行代码。看看官方文档的说法：<br>
> ![这里是引用](img03/7a56c37fa06e40388596089b70add1df.png)<br>
强制注销等价于对方主动调用了注销方法，再次访问会提示：Token无效。

### `StpLogic#logout`
![在这里插入图片描述](img03/f387fee8559c4702b5b19379f17e2f38.png)

方法的步骤主要有以下几点：
- 如果连token都没有，那么无需执行任何操作
- 从当前 [storage存储器] 里删除
- 如果打开了Cookie模式，则把cookie清除掉
- 清除这个token的相关信息

下面来对每一步骤进行 Debug 分析。
#### 1、获取当前用户 token，判断是否存在
![在这里插入图片描述](img03/8a4aa39a27f241969d2dd033deacf720.png)
###### `StpLogic#getTokenValue`
![在这里插入图片描述](img03/f849187815044113adaa44f92c96544f.png)

获取 token 的方法，在登录流程里面也有用到这个方法。
###### `StpLogic#getTokenValueNotCut`
![在这里插入图片描述](img03/2609e27079b14709943a383d8f640854.png)<br>

从请求头中获取到 token (未裁剪前缀) 并返回。<br>
![在这里插入图片描述](img03/59debdb96a494fadbdfc28694023f7a7.png)<br>

去掉前缀，得到最终的 token 并返回到上一级。<br>
![在这里插入图片描述](img03/c6454981088a4e68a4444d87f27e1459.png)<br>

![在这里插入图片描述](img03/6c617cdf207644ff9120585f7c25b750.png)<br>

token 不为空，继续执行下面的代码。
#### 2、从当前【storage 存储器】里删除
![在这里插入图片描述](img03/6cce347f3595482a8d34070ef7d77b6e.png)<br>

先获取 storage 存储器。<br>
![在这里插入图片描述](img03/0c3a6885178744e3af6299339af2b5dd.png)
###### `SaTokenContextForSpring#getStorage`

实际上就是根据 `request` 请求数据 new 了一个 `SaStorageForServlet`。<br>
![在这里插入图片描述](img03/c270fab0fa05479ebd4b60d0d46eb362.png)<br>

![在这里插入图片描述](img03/873f1e039f084079b746942c1b383d34.png)<br>
###### `StpLogic#splicingKeyJustCreatedSave`
![在这里插入图片描述](img03/56201117fce645068d8d14e5cd2801eb.png)
###### `SaStorageForServlet#delete`
![在这里插入图片描述](img03/ef40a8973240448fa746f28b8f5d6fb1.png)

#### 3、判断配置，把 cookie 清除掉
配置文件：<br>
![在这里插入图片描述](img03/1313c4de676c469cb505026f234b05d1.png)<br>

![在这里插入图片描述](img03/2819a4b1c31e44c3a62e90b8fb419ec5.png)<br>

没有配置 Cookie 模式，继续执行下面的代码。

#### 4、清除 token 的相关信息
###### `StpLogic#logoutByTokenValue`
![在这里插入图片描述](img03/e11330d502e94c14aa306a0611f7a9bb.png)
##### 4.1、清理 token-last-activity
###### `StpLogic#clearLastActivity`
![在这里插入图片描述](img03/dfddf518fc874584a9fb405fde85a649.png)<br>

判断是否是用不过期：false（配置时间是 1800 秒）。<br>

删除最后操作时间：<br>
![在这里插入图片描述](img03/fd484454c3e24fa0a252b4ad9d3bf244.png)<br>

清除标记，调用 storage 存储器删除 `request` 中的值。<br>
![在这里插入图片描述](img03/cd6fb920b33d43af872acd504a50ef47.png)<br>

完成操作返回上一级。
##### 4.2、注销 Token-Session
###### `StpLogic#deleteTokenSession`
![在这里插入图片描述](img03/3d2f75a1c14b4165aa501fe688dd66be.png)

##### 4.3、根据 token 获取 loginId，清理 token-id 索引
###### `StpLogic#getLoginIdNotHandle`
![在这里插入图片描述](img03/d010ed1b8ff044fda455f55e768aa4f1.png)<br>

loginId 值为 `sys_user:1`。<br>
![在这里插入图片描述](img03/ca74cd83d9ac494d893ec510fb6ed5df.png)<br>

删除 token-id 映射。<br>
![在这里插入图片描述](img03/53ef877d57354a58ad668a652e23ab73.png)<br>
##### 4.4、通知监听器，某某 Token 注销下线了
###### `UserActionListener#doLogout`
![在这里插入图片描述](img03/2a4e66162f1b4bf280d3819652bbc67b.png)<br>

控制台输出：<br>
![在这里插入图片描述](img03/60d72c4f747340cf8e019c51e71e3276.png)<br>
##### 4.5、清理 User-Session 上的 token 签名 & 尝试注销 User-Session
![在这里插入图片描述](img03/8c710a0ada1a47bdb8322b7b1d644d16.png)

移除 session 中的相关信息：<br>
![在这里插入图片描述](img03/22f95a0fce034579b478def574ea72d1.png)<br>

至此 `StpLogic#logout` 方法全部执行完成。用户退出成功：<br>
![在这里插入图片描述](img03/ab2f976899084f13a2f17e86b0a79af2.png)
