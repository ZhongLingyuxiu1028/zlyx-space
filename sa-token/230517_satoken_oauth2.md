## 前言

好久没有更新，MyBatis 源码的书上周也看完了，但是没有想好要写什么内容（书里面讲得比较全面，感觉不知道该写什么哈哈哈） 。

框架【`RuoYi-Vue-Plus`】准备发布 `5.X` 版本了，基于`JDK17` 和 `Spring Boot 3.X`，项目结构做了很大调整，也加入了简单的多租户功能，感兴趣的朋友可以去体验一下。

`5.X` 版本有考虑接入 `OAuth 2` 功能，仓库 Issue 也有相关说明。因为之前没有接触过相关的功能开发，因此以 `Sa-Token` 的 `OAuth 2.0` 的 Demo 为主进行学习，希望能够对这一功能有更深入的了解。

## 参考目录 

- [Sa-Token 官方文档：Sa-Token-OAuth2.0 模块 ](https://sa-token.cc/doc.html#/oauth2/readme)
- [OAuth 2.0 的四种方式 (by 阮一峰老师)](https://www.ruanyifeng.com/blog/2019/04/oauth-grant-types.html)

## 测试 Demo

本文使用的是 `Sa-Token` 最新版本 `V1.34.0` 的官方 Demo。

- [OAuth2-Server端](https://gitee.com/dromara/sa-token/tree/master/sa-token-demo/sa-token-demo-oauth2-server)： `/sa-token-demo/sa-token-demo-oauth2-server/`
- [OAuth2-Client端](https://gitee.com/dromara/sa-token/tree/master/sa-token-demo/sa-token-demo-oauth2-client)： `/sa-token-demo/sa-token-demo-oauth2-client/`

Demo 目录结构如下：

![在这里插入图片描述](images/230517_satoken_oauth2/b19836ba3bfe4c05983ab0912860d090.png)

需要注意的是，Demo 中的数据是没有对数据库进行交互的，换句话说就是写死的，实际使用中需要根据需要从数据库中获取相关数据，在 Demo 中也有相关的注释说明。

## 调用流程分析

### 调用流程说明

`OAuth 2.0` 一共有四种方式，本文分析的是最常用的 **授权码方式**。

如果看了参考目录里面的说明的话应该对下面这张图不陌生：

> （截图自 Sa-Token 官方文档）
> ![在这里插入图片描述](images/230517_satoken_oauth2/b95074fb8522453692ff9bf752146cc1.png)

这是授权码方式的流程简图，经过 Debug 分析之后，我画了一张更贴近代码调用流程的图，可以根据这张图来分析 `OAuth2` 授权码模式。

![在这里插入图片描述](images/230517_satoken_oauth2/15a210549af646658812d70fbfd0a473.png)

先简单说明一下，调用流程可以划分为四个步骤，右边服务端入口方法都是同一个 `SaOAuth2Handle#serverRequest`，在方法内部进行了路由分发，根据不同的请求调用不同的处理方法，下面来看看 Debug 流程。

### 源码分析

### 0、启动项目

按照官方文档步骤启动项目，并打开测试页面。

![在这里插入图片描述](images/230517_satoken_oauth2/12a2bf3686c646dda3298074ac79b0e3.png)

这里面有不同的操作，上面流程图所描绘的是第一种静默授权的流程，理解了这个流程之后，下面三种方式都比较好理解，文本暂不展开。

### 1、步骤1：点击授权登录

这一步骤的请求是：

```html
http://sa-oauth-server.com:8001/oauth2/authorize?response_type=code&client_id=1001&redirect_uri=http://sa-oauth-client.com:8002/ 
```

`SaOAuth2ServerController#request`
![在这里插入图片描述](images/230517_satoken_oauth2/023efb81339f4a02a6191fa25d1e71bb.png)

后续的步骤也是进入此方法处理。

`SaOAuth2Handle#serverRequest`
![在这里插入图片描述](images/230517_satoken_oauth2/8c72b1677cc94884898375cfc5703964.png)

### 1.1、获取变量以及配置（路由分发的实现）

首先从 `SaHolder` 获取请求 `Request` 以及 `Response` 信息，当然也是经过了封装的增强对象。

`SaHolder` 是 `Sa-Token` 上下文持有类，将请求的信息封装在这个类里面。

![在这里插入图片描述](images/230517_satoken_oauth2/8eebd224521842f1b3598de1569469fb.png)

`SaOAuth2Config` 是 `Sa-Token` 的 `OAuth2` 配置对象，Demo 里面对此进行了定制化的配置，在项目启动时就已经将配置信息初始化到了容器里面。

`SaOAuth2ServerController#setSaOAuth2Config`
![在这里插入图片描述](images/230517_satoken_oauth2/3694d8f63d5443ddba3a128c0d3aff74.png)

除此之外还有默认配置。

![在这里插入图片描述](images/230517_satoken_oauth2/31726cf5e7d941cf97bd23797d7815fc.png)

不同的请求都封装在常量类 `SaOAuth2Consts` 里面。

![在这里插入图片描述](images/230517_satoken_oauth2/9cf79ed006dd45199c4ce56f3438bb65.png)

判断请求信息，根据不同的请求做不同的处理，这就完成了路由分发的操作，配置信息则是用来进行授权。

`SaOAuth2Handle#serverRequest` 这个方法是将子方法归拢到了一个方法里面，如果想要分开处理的话，也可以在 Controller 里面根据不同的请求写不同的入口，并调用 `SaOAuth2Handle` 的方法进行处理。

### 1.2、获取客户端对象

根据上一步的判断，接下来是要获取 `SaClientModel` 对象。这一步需要根据请求中的参数 `client_id` 来获取，在 Demo 中给了一个简单的实现。

`SaOAuth2Handle#currClientModel`
![在这里插入图片描述](images/230517_satoken_oauth2/5a2e7d4438b043f7a4dcdb4bb1acac1f.png)

`SaOAuth2Util#checkClientModel`
![在这里插入图片描述](images/230517_satoken_oauth2/55a0a3385ece475eafa6d397d71eb118.png)

`SaOAuth2Template#checkClientModel`
![在这里插入图片描述](images/230517_satoken_oauth2/5009e19f3d0e4fccb1484714b9f024fc.png)

`SaOAuth2TemplateImpl#getClientModel`
![在这里插入图片描述](images/230517_satoken_oauth2/5f78277c323f4ba09566731c9185b4f8.png)

![在这里插入图片描述](images/230517_satoken_oauth2/a4fad41c243045ec9d59fe81d90c79be.png)

### 1.3、尝试授权（未登录）

![在这里插入图片描述](images/230517_satoken_oauth2/c3d0d986cdd7428f90f26b48d848ac56.png)

`SaOAuth2Handle#authorize`
![在这里插入图片描述](images/230517_satoken_oauth2/76aa1dc462544abf986571004b6ae3e9.png)

未登录页面的配置：

![在这里插入图片描述](images/230517_satoken_oauth2/648f6769b727497cbf58bd1b54b03384.png)

前端跳转至未登录页面。

### 2、步骤2：输入账号密码登录

前端登录页面：

![在这里插入图片描述](images/230517_satoken_oauth2/aaf81e86f87245888e102c20f3c9a2c5.png)

输入账号密码后，请求的是：

```html
http://sa-oauth-server.com:8001/oauth2/doLogin
```

### 2.1、登录请求校验

同样进入到 `SaOAuth2Handle#serverRequest` 方法。

![在这里插入图片描述](images/230517_satoken_oauth2/85b737b8cdb740369a0551d9c1f830b2.png)

`SaOAuth2Handle#doLogin`
![在这里插入图片描述](images/230517_satoken_oauth2/afc2cc99bd034b6689f60813de3a4546.png)

获取登录函数 `DoLoginHandle` 对账号密码进行校验。

登录函数的配置如下：
![在这里插入图片描述](images/230517_satoken_oauth2/7632b56b70514b59847e884d4641268b.png)

![在这里插入图片描述](images/230517_satoken_oauth2/393e041de8ca4f02ad754d52a6582a2b.png)

登录成功之后，会再次请求 Code，也就是再次进入步骤 1 中的方法。

### 3、步骤3：再次向服务端请求授权并获取 Code

`SaOAuth2Handle#authorize`
![在这里插入图片描述](images/230517_satoken_oauth2/1f7dde160873456a81a7507a126d85c2.png)

在获取 Code 之前需要进行一系列的判断。前面未登录判断已经校验通过了，下面来简单说明一下其他判断步骤。

### 3.1、构建请求对象 `RequestAuthModel`

`SaOAuth2Util#generateRequestAuth`
![在这里插入图片描述](images/230517_satoken_oauth2/c866d5e2c7ca46fba0f6a14e100415de.png)

`SaOAuth2Template#generateRequestAuth`
![在这里插入图片描述](images/230517_satoken_oauth2/5ebd01c07c37413fad6135290e8ef60d.png)

### 3.2、重定向路径校验

![在这里插入图片描述](images/230517_satoken_oauth2/f80b72f6924a4514a046fea11e19b2ee.png)

`SaOAuth2Util#checkRightUrl`
![在这里插入图片描述](images/230517_satoken_oauth2/41afbb8209be497e84aec3c9d3bfb31c.png)

`SaOAuth2Template#checkRightUrl`
![在这里插入图片描述](images/230517_satoken_oauth2/f50aebd7a92041a5bf61886fd62f2626.png)

`SaStrategy#hasElement`
![在这里插入图片描述](images/230517_satoken_oauth2/32f6f0f180064d5cbbffcf68ad7c8df5.png)

### 3.3、获取 Code

由于不是显式授权，也没有设置 `Scope`，因此判断步骤 4和5 暂且不进行说明。

![在这里插入图片描述](images/230517_satoken_oauth2/5976b0a19ef34a8ca7567596ab4c01ab.png)

然后来到获取 Code 步骤：
![在这里插入图片描述](images/230517_satoken_oauth2/b2194f6a1be84e14b237e6ac251fde51.png)

`SaOAuth2Util#generateCode`
![在这里插入图片描述](images/230517_satoken_oauth2/24f97199489141a480a1549a58dc6add.png)

`SaOAuth2Template#generateCode`
![在这里插入图片描述](images/230517_satoken_oauth2/d8ac3f2872374aaa9d748e4916f3fdd7.png)

![在这里插入图片描述](images/230517_satoken_oauth2/46ecdb1246a6413ab94e8fd30784a5c2.png)

引用一下官方文档对于 Code 的说明：

> Code授权码具有以下特点：
> 1.每次授权产生的Code码都不一样
> 2.Code码用完即废，不能二次使用
> 3.一个Code的有效期默认为五分钟，超时自动作废
> 4.每次授权产生新Code码，会导致旧Code码立即作废，即使旧Code码尚未使用

### 3.4、组装重定向路径

`SaOAuth2Util#buildRedirectUri`
![在这里插入图片描述](images/230517_satoken_oauth2/5fb228adb181489eb04db47311619f1e.png)

`SaOAuth2Template#buildRedirectUri`
![在这里插入图片描述](images/230517_satoken_oauth2/f419cd2d84454220bfdc7e396f46a326.png)

组装好路径之后就会进行页面重定向。

![在这里插入图片描述](images/230517_satoken_oauth2/15cbbc3fbb3a4be491df4966f26fc366.png)

然后前端回到首页，并且使用 Code 码进行授权登录。

### 4、步骤4：使用 Code 获取 Access-Token

经过上一步骤之后获取了 Code，然后前端再根据 Code 请求获取 Access-Token。如果是显示授权，可以使用 Access-Token 可以获取到系统相关的用户信息等。

`SaOAuthClientController#codeLogin`
![在这里插入图片描述](images/230517_satoken_oauth2/3533930dc5a5425b95513af6ea967746.png)

`SaOAuth2Handle#serverRequest`
![在这里插入图片描述](images/230517_satoken_oauth2/7860ffcd323e412594d2dbe6bd8e07c3.png)

`SaOAuth2Handle#token`
![在这里插入图片描述](images/230517_satoken_oauth2/2cf5686467464b54bcfe7afe16e21fb1.png)

### 4.1、校验参数

`SaOAuth2Util#checkGainTokenParam`
![在这里插入图片描述](images/230517_satoken_oauth2/31c9d52714874320aec229fbd143504d.png)

`SaOAuth2Template#checkGainTokenParam`
![在这里插入图片描述](images/230517_satoken_oauth2/90d1f83055ee43328dd785274a259e9a.png)

### 4.2、构建 Access-Token

（Debug 时间太长导致 Code 失效了，所以这里 Code 和前面的不太一样，是重新请求之后的结果）

`SaOAuth2Util#generateAccessToken`
![在这里插入图片描述](images/230517_satoken_oauth2/b7f88f7ae4724fda97d1eb0be3c58f52.png)

`SaOAuth2Template#generateAccessToken`
![在这里插入图片描述](images/230517_satoken_oauth2/c93a9cd9234b49fc87c665ba2a516b0d.png)

`AccessTokenModel` 对象：
![在这里插入图片描述](images/230517_satoken_oauth2/76f85542a3c14bd59c95aaf837db9abd.png)

构建完成，将 Access-Token 对象返回到前端。

![在这里插入图片描述](images/230517_satoken_oauth2/245813b3946f45b2acfea58675e4b027.png)

### 5、请求成功

请求完成之后前端展示的内容：

![在这里插入图片描述](images/230517_satoken_oauth2/935816f2cd714324b730000d226246a4.png)

（Debug 时间太长异步请求异常了，所以这里 Token 和前面的不太一样，是重新请求之后的结果）

至此完成了OAuth2 授权码模式（静默授权）调用流程的分析。

（完）