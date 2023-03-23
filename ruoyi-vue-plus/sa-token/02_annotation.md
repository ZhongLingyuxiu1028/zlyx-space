# Sa-Token（二）通过注解校验用户权限
- - -
## 前言
上一篇文章主要介绍了 Sa-Token 登录认证流程，本篇文章是对于权限注解的校验流程分析。

## 参考目录
- [Sa-Token 官方文档](https://sa-token.dev33.cn/doc/index.html#/)

## 代码分析
请求接口：`TestDemoController#list`<br>
![在这里插入图片描述](img02/028cf3c7f08c47aeb47085c214b474bc.png)<br>

每次请求之前触发的方法：`SaAnnotationInterceptor#preHandle`<br>
![在这里插入图片描述](img02/40f2d2b103004348a9e2990463f61776.png)<br>

首先被注解拦截器进行拦截，如果验证通过，则拦截器放行。

查看验证方法：`SaStrategy#checkMethodAnnotation`<br>
![在这里插入图片描述](img02/65359d6704d44c18a7b18aa1e91d7fd9.png)<br>

先校验 Method 所属 Class 上的注解，类上没有标注注解，所以校验 Method 上的注解。

`SaTokenActionDefaultImpl#validateAnnotation`<br>
![在这里插入图片描述](img02/4e44d93107cf4b7bab158866699e478c.png)<br>

获取注解信息：`SaStrategy#getAnnotation`<br>
![在这里插入图片描述](img02/c54f3faee25b4a59bf2810e1529fac8f.png)<br>

![在这里插入图片描述](img02/32dab881bf714b7ba8dd259552ef66bc.png)<br>

根据注解 (`@SaCheckPermission`) 鉴权：`StpLogic#checkByAnnotation`<br>
![在这里插入图片描述](img02/77433cfd7e234268960057af2b143bd2.png)<br>

`StpLogic#checkPermissionAnd`<br>
![在这里插入图片描述](img02/7a64d6c92f94414b8757d38ad41c3048.png)<br>

对所有权限进行校验，循环判断登录用户是否拥有该权限参数（用户登录时会把所有权限集合保存到指定对象中），如果没有则抛出没有权限的异常。

所有校验通过后则校验完成，返回 true。<br>
![在这里插入图片描述](img02/ba74b753996847c29dc2c9e4122ded36.png)<br>

至此，Sa-Token 完成了关于权限注解的校验。