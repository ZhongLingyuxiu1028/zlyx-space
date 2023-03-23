# XSS 过滤器以及 @Xss 注解简单分析

## 前言
之前在对接口进行传参时发现富文本包含的标签全部被过滤掉了，例如：`<h1>test</h1>` 变成了 `test`，但是框架自带的通知公告方法是正常的，请教了 [狮子大佬](https://blog.csdn.net/weixin_40461281) 后才知道原来是因为 XSS 过滤。这个其实是很早之前框架就带上的功能，一开始不知道是做什么的，后面就忘记了，所以本文来简单分析一下。

## 参考目录
### 关于 XSS 攻击
- [前端安全系列（一）：如何防止XSS攻击？](https://tech.meituan.com/2018/09/27/fe-security.html)
- [这一次，彻底理解XSS攻击](https://juejin.cn/post/6912030758404259854)


## 框架集成
### 配置说明
关于 XSS 过滤功能是通过过滤器来实现的，相关的配置如下表：

| 方法 / 类 名称                                                       | 功能                |
|-----------------------------------------------------------------|-------------------|
| com.ruoyi.common.filter.XssFilter                               | 防止 XSS 攻击的过滤器     |
| com.ruoyi.framework.config.properties.XssProperties             | XSS 过滤器配置对象       |
| com.ruoyi.framework.config.FilterConfig#xssFilterRegistration   | 注册 XSS 过滤器        |
| com.ruoyi.common.filter.XssHttpServletRequestWrapper            | XSS 过滤处理请求包装类     |
| com.ruoyi.common.xss.Xss                                        | 自定义 XSS 校验注解      |
| com.ruoyi.common.xss.XssValidator                               | 自定义 XSS 校验注解实现    |

详细的代码就不一一贴出来了，可以到相关类中进行查看。

框架中对于 XSS 的过滤有两种方式：

- 方式一：`application.yml` 配置
- 方式二：`@Xss` 注解

下面会通过 demo 的方式对这两种方式进行简单分析。

### 测试方法一：通过过滤器
`application.yml` 配置：
![在这里插入图片描述](img02/c2a135f946ca4363b76aef331cbe3623.png)

`TestDemoController` 测试接口：
![在这里插入图片描述](img02/f3e2a442a6e24c3fb37906a9d46298b8.png)

在 XSS 过滤处理请求包装类 `XssHttpServletRequestWrapper` 中对于不同类型的请求参数使用的过滤方法不同，所以此处进行分开。

`@SaIgnore` 注解是 Sa-Token 忽略鉴权注解，可以不需要登录获取 Token 直接请求接口。

`TestDemoController#testXss` 请求结果：
![在这里插入图片描述](img02/fefc58a2bb53465a95ced024bbb662b1.png)

`TestDemoController#testXss2` 请求结果：
![在这里插入图片描述](img02/aefc0752620e471ba4e91c20e76013e5.png)

### 测试方法二：通过 `@Xss` 注解
关于 `@Xss` 注解使用可以参考若依官方文档（[传送门](http://doc.ruoyi.vip/ruoyi/document/htsc.html#%E8%87%AA%E5%AE%9A%E4%B9%89%E6%B3%A8%E8%A7%A3%E6%A0%A1%E9%AA%8C)）。

文档中是对请求对象中的参数加上注解，可以参考框架 `SysUser` 对象。

测试接口简化了写法，直接对 Form 表单参数进行校验：
![在这里插入图片描述](img02/5f63d6ccb70047f6aa45c8cb1fdc6493.png)

需要注意的是，**对参数的校验会在 Xss 过滤器过滤之后**，进行注解校验测试时，不用打开 Xss 过滤器或者将配置文件中的过滤链接去掉。

`TestDemoController#testXss3` 请求结果：
![在这里插入图片描述](img02/56cd200ab67e4714a108de33b9651853.png)
## 功能调用流程分析
### XSS 过滤器启动初始化
首先根据配置中的条件注册过滤器：

`FilterConfig#xssFilterRegistration`<br>
![在这里插入图片描述](img02/6d694c79753c434283570722ce9e9733.png)

根据配置保存需要排除的接口：

`XssFilter#init`<br>
![在这里插入图片描述](img02/54ec4196e6754dbd868a03d62e66ce62.png)
### Form 表单请求过滤
测试方法一中的第一个方法：

请求接口为 `/demo/demo/testXss`：<br>
![在这里插入图片描述](img02/f089aae0108f4017bca875c3d715b78a.png)

`XssFilter#doFilter`<br>
![在这里插入图片描述](img02/efce1a5314424546b5491d4b44fe4635.png)

过滤器会匹配是否是需要排除的接口。

![在这里插入图片描述](img02/c0b41f1cfdb54856a7ba97c481aad703.png)

进入包装类过滤：<br>
`XssHttpServletRequestWrapper#getParameterValues`<br>
![在这里插入图片描述](img02/df9ae1808c0141a6b2fea332e7d42eeb.png)

过滤方法：<br>
![在这里插入图片描述](img02/5974f74033aa4cb9865822749e117da1.png)

返回过滤后的参数：<br>
![在这里插入图片描述](img02/ef1d883066b54f48b7a8c02dfe179bc7.png)
### JSON 对象请求过滤
测试方法一中的第二个方法：

请求接口为 `/demo/demo/testXss2`：<br>
![在这里插入图片描述](img02/9fae4a94c3e441ad9656bbb4a23b9171.png)

`XssFilter#doFilter`<br>
![在这里插入图片描述](img02/f22b4facb8454eafa3c64e16100bea8e.png)

同上，也会对接口进行匹配，看是否需要放行。

![在这里插入图片描述](img02/053df6a9cbdf4c988c81c979297a469f.png)

接着进入包装类方法过滤标签：<br>
`XssHttpServletRequestWrapper#getInputStream`<br>
![在这里插入图片描述](img02/0b843a9606b844efb9643b00e068587d.png)

过滤后：<br>
![在这里插入图片描述](img02/2c158c00a12148ea9eb9606ecf861937.png)

### `@Xss` 注解校验
测试方法二中的方法：

请求接口为 `/demo/demo/testXss3`：<br>
![在这里插入图片描述](img02/e48908ac491c43138344de8769862f76.png)

校验方法：

`XssValidator#isValid`<br>
![在这里插入图片描述](img02/cbbed362c8cf406f8d60efe9aa1ea5c1.png)

校验结果：

![在这里插入图片描述](img02/0c48d85ab6d94696916762b69ef2f6c6.png)

校验不通过，返回异常信息。

（完）